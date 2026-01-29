Zoom Auto Login Scheduler

Python • Selenium • Automation • Security Engineering

Summary

A Python-based automation tool that securely logs into Zoom on a scheduled basis using Selenium. Built with retry logic, structured logging, and secure credential handling, this project demonstrates real-world automation patterns used in SOC and cloud security operations.

What This Demonstrates

Browser automation with Selenium WebDriver

Secure credential handling via environment variables

Explicit waits for dynamic web content

Resilient automation with retries and backoff

Scheduled execution and structured logging

Defensive coding practices (no hardcoded secrets)

Core Features

Headless Chrome execution

Exponential retry logic on failure

Explicit element waits (no fragile sleeps)

Scheduled weekly execution

File + console logging

Clean shutdown and error handling

Tech Stack

Python 3.10+

Selenium

webdriver-manager

schedule

pytz

Google Chrome

How It Works (High Level)

Launches a headless Chrome browser

Navigates to the Zoom login page

Authenticates using environment variables

Handles common post-login prompts

Verifies successful login via page indicators

Logs outcome and exits cleanly

Security Practices

No credentials stored in code or repo

Environment variable–based secrets

.gitignore excludes logs and virtual environments

Browser password managers disabled

No unsafe browser flags used
Scheduling Notes

Runs weekly using the schedule library.
Execution time is based on the machine’s local timezone.

Intended Use

Automation engineering examples

SOC / security workflow automation

Selenium best practices

Portfolio demonstration (sanitized)

Author

Terrance Robinson
Cybersecurity Engineer
GitHub: https://github.com/trob3
