# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Multi-platform, multi-account automatic check-in script for NewAPI/OneAPI compatible platforms (primarily Any Router and Agent Router). Runs as a GitHub Actions workflow on Windows (`windows-2025`) every 6 hours, with optional manual trigger. Written in Python, uses `uv` as the package manager.

## Commands

```bash
# Install dependencies
uv sync --dev

# Install Playwright browser (required for WAF bypass)
uv run playwright install chromium

# Run the check-in script (requires ANYROUTER_ACCOUNTS env var)
uv run checkin.py

# Run tests
uv run pytest tests/

# Lint
uv run ruff check .

# Format
uv run ruff format .

# Pre-commit hooks (ruff + pre-commit-hooks)
uv run pre-commit run --all-files
```

## Code Style

- **Formatter**: ruff (single quotes, tab indentation, line length 120)
- **Linter**: ruff with `ASYNC`, `E`, `F`, `FAST`, `I`, `PLE` rules
- Pre-commit hooks enforce formatting and linting on commit

## Architecture

**Entry point**: `checkin.py` — async main loop that iterates over accounts, performs check-in for each, tracks balance changes via SHA-256 hash (`balance_hash.txt`), and sends notifications only on failures or balance changes.

**`utils/config.py`** — Configuration layer with three dataclasses:
- `ProviderConfig`: platform connection details (domain, API paths, WAF bypass settings). Built-in providers: `anyrouter` (WAF bypass required) and `agentrouter` (WAF bypass required, auto-check-in via user info query — `sign_in_path=None`).
- `AppConfig`: loads providers from built-in defaults + optional `PROVIDERS` env var (custom providers override built-in ones by name).
- `AccountConfig`: per-account credentials (cookies, api_user, provider name). Loaded from `ANYROUTER_ACCOUNTS` env var (JSON array).

**`utils/notify.py`** — `NotificationKit` singleton (`notify`) supporting 9 notification channels (Email/SMTP, PushPlus, ServerPush, DingTalk, Feishu, WeCom, Gotify, Telegram, Bark). All configured via env vars; unconfigured channels are silently skipped.

**WAF bypass flow**: For providers with `bypass_method="waf_cookies"`, Playwright launches a headless Chromium browser to visit the login page and extract WAF cookies (e.g., `acw_tc`, `cdn_sec_tc`) before making API requests with `httpx`.

## Key Design Decisions

- HTTP client: `httpx` with HTTP/2 support (not `requests`)
- `agentrouter` provider has `sign_in_path=None` — check-in happens automatically when querying user info, no explicit sign-in POST needed
- Notifications are only sent when there are failures or balance changes (hash comparison), not on every successful run
- GitHub Actions runs on Windows runner with UV caching for both dependencies and Playwright browsers
- Environment: Python >=3.11, managed by `uv` with `uv.lock`
