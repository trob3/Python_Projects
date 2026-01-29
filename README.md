# Python_Projects
import logging
import os
import signal
import sys
import time
from dataclasses import dataclass
from datetime import datetime
from typing import Optional

import pytz
import schedule
from selenium import webdriver
from selenium.common.exceptions import TimeoutException, StaleElementReferenceException
from selenium.webdriver.chrome.options import Options
from selenium.webdriver.chrome.service import Service
from selenium.webdriver.common.by import By
from selenium.webdriver.support import expected_conditions as EC
from selenium.webdriver.support.ui import WebDriverWait
from webdriver_manager.chrome import ChromeDriverManager


# =============================================================================
# CONFIG
# =============================================================================

@dataclass(frozen=True)
class Config:
    zoom_login_url: str = "https://zoom.us/signin"
    zoom_success_indicators: tuple[str, ...] = ("profile", "user", "dashboard", "meetings", "my account")
    max_retries: int = 3
    login_timeout: int = 30
    scheduler_check_interval: int = 60
    dialog_timeout: float = 2.0
    headless: bool = True


CONFIG = Config()


# =============================================================================
# LOGGING
# =============================================================================

def setup_logging() -> logging.Logger:
    logger = logging.getLogger("zoom_auto_login")
    logger.setLevel(logging.INFO)

    fmt = logging.Formatter("%(asctime)s - %(levelname)s - %(message)s")

    # Avoid duplicate handlers if re-run in VS Code interactive environments
    if logger.handlers:
        return logger

    try:
        fh = logging.FileHandler("zoom_login.log", encoding="utf-8")
        fh.setFormatter(fmt)
        logger.addHandler(fh)
    except Exception as e:
        # Fall back to console only if file handler fails
        print(f"Warning: Could not create log file: {e}")

    sh = logging.StreamHandler()
    sh.setFormatter(fmt)
    logger.addHandler(sh)

    return logger


logger = setup_logging()


# =============================================================================
# ZOOM AUTO LOGIN
# =============================================================================

class ZoomAutoLogin:
    def __init__(self, email: str, password: str, config: Config = CONFIG):
        self.email = email
        self.password = password
        self.config = config
        self.driver: Optional[webdriver.Chrome] = None

    def __enter__(self) -> "ZoomAutoLogin":
        self._setup_driver()
        return self

    def __exit__(self, exc_type, exc, tb) -> None:
        self._cleanup()

    def _setup_driver(self) -> None:
        chrome_options = Options()

        # Headless mode
        if self.config.headless:
            chrome_options.add_argument("--headless=new")
            chrome_options.add_argument("--disable-gpu")

        # Stability/perf flags
        chrome_options.add_argument("--disable-extensions")
        chrome_options.add_argument("--disable-infobars")
        chrome_options.add_argument("--no-sandbox")
        chrome_options.add_argument("--disable-dev-shm-usage")
        chrome_options.add_argument("--window-size=1280,900")

        # Suppress browser UI prompts
        prefs = {
            "profile.default_content_setting_values.notifications": 2,
            "profile.default_content_settings.popups": 0,
            "credentials_enable_service": False,
            "profile.password_manager_enabled": False,
            # Properly disable images (Chrome uses prefs, not a CLI flag)
            "profile.managed_default_content_settings.images": 2,
        }
        chrome_options.add_experimental_option("prefs", prefs)

        # Consistent UA (optional)
        chrome_options.add_argument(
            "user-agent=Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36"
        )

        try:
            self.driver = webdriver.Chrome(
                service=Service(ChromeDriverManager().install()),
                options=chrome_options,
            )
            self.driver.implicitly_wait(0)  # prefer explicit waits only (cleaner & predictable)
            logger.info("✓ WebDriver initialized")
        except Exception as e:
            logger.error(f"✗ Failed to initialize WebDriver: {e}")
            raise

    def _cleanup(self) -> None:
        if self.driver:
            try:
                self.driver.quit()
                logger.info("✓ Browser closed")
            except Exception as e:
                logger.warning(f"⚠ Error closing browser: {e}")
            finally:
                self.driver = None

    def _wait(self) -> WebDriverWait:
        if not self.driver:
            raise RuntimeError("Driver not initialized")
        return WebDriverWait(self.driver, self.config.login_timeout)

    def login(self) -> bool:
        if not self.driver:
            raise RuntimeError("Driver not initialized")

        try:
            logger.info("→ Starting Zoom login process...")
            self.driver.get(self.config.zoom_login_url)

            wait = self._wait()

            # Email
            email_field = wait.until(EC.presence_of_element_located((By.ID, "email")))
            email_field.clear()
            email_field.send_keys(self.email)
            logger.info("✓ Email entered")

            # Password
            password_field = wait.until(EC.presence_of_element_located((By.ID, "password")))
            password_field.clear()
            password_field.send_keys(self.password)
            logger.info("✓ Password entered")

            # Sign in
            sign_in_button = wait.until(EC.element_to_be_clickable((By.ID, "login")))
            sign_in_button.click()
            logger.info("✓ Sign-in clicked")

            self._handle_dialogs()

            self._wait_for_login_success()
            logger.info("✓ Successfully logged into Zoom")
            return True

        except TimeoutException:
            logger.error("✗ Timeout waiting for login page elements or success condition")
            return False
        except Exception as e:
            logger.error(f"✗ Login error: {e}")
            return False

    def _wait_for_login_success(self) -> None:
        if not self.driver:
            raise RuntimeError("Driver not initialized")

        def success_condition(d: webdriver.Chrome) -> bool:
            url = (d.current_url or "").lower()
            src = (d.page_source or "").lower()
            return any(k in url or k in src for k in self.config.zoom_success_indicators)

        self._wait().until(lambda d: success_condition(d))

    def _handle_dialogs(self) -> None:
        """Best-effort handling for post-login prompts/alerts."""
        if not self.driver:
            return

        time.sleep(self.config.dialog_timeout)

        # Common “continue” style buttons (best-effort)
        xpaths = [
            "//button[@type='submit']",
            "//button[contains(translate(normalize-space(.), 'ABCDEFGHIJKLMNOPQRSTUVWXYZ', 'abcdefghijklmnopqrstuvwxyz'), 'continue')]",
            "//button[contains(translate(normalize-space(.), 'ABCDEFGHIJKLMNOPQRSTUVWXYZ', 'abcdefghijklmnopqrstuvwxyz'), 'sign in')]",
            "//button[contains(translate(normalize-space(.), 'ABCDEFGHIJKLMNOPQRSTUVWXYZ', 'abcdefghijklmnopqrstuvwxyz'), 'ok')]",
        ]

        for xp in xpaths:
            try:
                btns = self.driver.find_elements(By.XPATH, xp)
                for b in btns[:1]:
                    if b.is_displayed() and b.is_enabled():
                        txt = (b.text or "").strip()
                        logger.info(f"✓ Clicking prompt button: {txt[:40] if txt else '[no text]'}")
                        b.click()
                        time.sleep(0.5)
                        break
            except StaleElementReferenceException:
                continue
            except Exception:
                continue

        # JS alert (rare)
        try:
            alert = self.driver.switch_to.alert
            logger.info(f"✓ Accepting alert: {(alert.text or '')[:60]}")
            alert.accept()
        except Exception:
            pass

    def run_with_retries(self) -> bool:
        for attempt in range(1, self.config.max_retries + 1):
            logger.info(f"→ Login attempt {attempt}/{self.config.max_retries}")

            try:
                with self:
                    if self.login():
                        return True
            except Exception as e:
                logger.error(f"✗ Attempt {attempt} failed: {e}")

            if attempt < self.config.max_retries:
                wait_time = min(2 ** attempt, 30)
                logger.info(f"⏳ Retrying in {wait_time}s...")
                time.sleep(wait_time)

        logger.error(f"✗ Login failed after {self.config.max_retries} attempts")
        return False


# =============================================================================
# SCHEDULER JOB
# =============================================================================

def scheduled_login() -> bool:
    central = pytz.timezone("US/Central")
    now = datetime.now(central)
    logger.info(f"→ Scheduled task triggered at {now.strftime('%Y-%m-%d %H:%M:%S %Z')}")

    email = os.getenv("ZOOM_EMAIL")
    password = os.getenv("ZOOM_PASSWORD")

    if not email or not password:
        logger.error("✗ Missing ZOOM_EMAIL or ZOOM_PASSWORD environment variables.")
        logger.info("  Windows (PowerShell):")
        logger.info("    setx ZOOM_EMAIL \"you@example.com\"")
        logger.info("    setx ZOOM_PASSWORD \"your-password\"")
        return False

    return ZoomAutoLogin(email, password).run_with_retries()


# =============================================================================
# SIGNAL HANDLING
# =============================================================================

def signal_handler(sig, frame) -> None:
    logger.info("✓ Shutdown signal received. Exiting...")
    sys.exit(0)


# =============================================================================
# MAIN
# =============================================================================

def main() -> None:
    signal.signal(signal.SIGINT, signal_handler)

    logger.info("=" * 60)
    logger.info("Zoom Auto-Login Script Started")
    logger.info("=" * 60)

    # NOTE: schedule uses your machine's LOCAL timezone for .at("HH:MM").
    # If your PC timezone is Central, you're good. If not, consider APScheduler.
    schedule.every().thursday.at("20:00").do(scheduled_login)
    logger.info("✓ Scheduled: Every Thursday at 8:00 PM (machine local time)")
    logger.info("  Press Ctrl+C to exit\n")

    while True:
        schedule.run_pending()
        time.sleep(CONFIG.scheduler_check_interval)


if __name__ == "__main__":
    main()
