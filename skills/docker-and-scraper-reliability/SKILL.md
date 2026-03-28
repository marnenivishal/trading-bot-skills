---
name: docker-and-scraper-reliability
description: "Use when running Playwright or Selenium scrapers in Docker, handling Cloudflare challenges, managing browser session persistence across restarts, or implementing health checks for headless Chrome"
---

# Docker and Scraper Reliability

A web scraper is not a service — it is a fragile process that WILL hang. Chrome leaks memory, Cloudflare blocks headless browsers, session cookies expire, and Docker containers restart. Every scraper needs a watchdog, a timeout, and a fallback signal source.

---

## The Iron Law

> **A SCRAPER IS NOT A SERVICE. IT WILL HANG. EVERY SCRAPER NEEDS A WATCHDOG, A TIMEOUT, AND A FALLBACK.**
>
> Origin: emabot's Playwright scraper hung for 30+ minutes on a Cloudflare challenge page — appearing "running" while producing zero output. OTP session cookies were lost on container restart. Chrome consumed all available memory. 3 extension versions and multiple scraper rewrites before stability.

---

## Core Patterns

### Pattern 1: Docker Compose with Health Check and Resource Limits

```yaml
# docker-compose.yml
services:
  scraper:
    build:
      context: .
      dockerfile: Dockerfile.scraper
    restart: unless-stopped
    shm_size: "2g"  # Chrome needs shared memory
    deploy:
      resources:
        limits:
          memory: 4g
          cpus: "2.0"
    healthcheck:
      test: ["CMD", "python", "scripts/healthcheck.py", "--key", "scraper_heartbeat"]
      interval: 60s
      timeout: 10s
      retries: 3
      start_period: 30s
    volumes:
      - scraper-data:/app/data  # Persist session cookies across restarts
    environment:
      - SCRAPER_TIMEOUT_SECONDS=300
      - CLOUDFLARE_MAX_RETRIES=3
```

### Pattern 2: Playwright Dockerfile

```dockerfile
# Use official Playwright base image — includes browser deps
FROM mcr.microsoft.com/playwright/python:v1.51.0-noble

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

# Chrome flags for Docker stability
ENV PLAYWRIGHT_CHROMIUM_ARGS="--no-sandbox --disable-dev-shm-usage --disable-gpu"

CMD ["python", "main_scraper.py"]
```

### Pattern 3: Scraper Heartbeat and Watchdog

The health check must verify the scraper is PRODUCING output, not just that the process is alive.

```python
import time
import asyncio

HEARTBEAT_KEY = "scraper_heartbeat"
HEARTBEAT_MAX_AGE = 180  # 3 minutes

async def scraper_loop():
    while True:
        try:
            messages = await scrape_chatroom()
            if messages:
                await process_messages(messages)
            # Update heartbeat on EVERY successful cycle
            await redis.set(HEARTBEAT_KEY, str(time.time()))
        except Exception as e:
            logger.error(f"Scraper cycle failed: {e}")
            # Still update heartbeat — we're alive, just erroring
            # Watchdog cares about "hung" not "erroring"
            await redis.set(HEARTBEAT_KEY, str(time.time()))

        await asyncio.sleep(5)  # Poll interval

# healthcheck.py — called by Docker healthcheck
async def check_health(key: str) -> bool:
    last_hb = await redis.get(key)
    if not last_hb:
        return False  # No heartbeat ever recorded
    age = time.time() - float(last_hb)
    return age < HEARTBEAT_MAX_AGE
```

### Pattern 4: Cloudflare Circuit Breaker

```python
class CloudflareCircuitBreaker:
    """Stop retrying when Cloudflare is blocking — switch to fallback."""

    def __init__(self, max_failures: int = 3, reset_seconds: int = 300):
        self.failures = 0
        self.max_failures = max_failures
        self.reset_seconds = reset_seconds
        self.last_failure: float = 0
        self.is_open = False

    def record_failure(self):
        self.failures += 1
        self.last_failure = time.time()
        if self.failures >= self.max_failures:
            self.is_open = True
            logger.warning(
                f"Cloudflare circuit breaker OPEN — {self.failures} failures. "
                f"Switching to fallback for {self.reset_seconds}s"
            )

    def record_success(self):
        self.failures = 0
        self.is_open = False

    def should_attempt(self) -> bool:
        if not self.is_open:
            return True
        # Check if reset period has elapsed
        if time.time() - self.last_failure > self.reset_seconds:
            self.is_open = False
            self.failures = 0
            return True  # Half-open: try once
        return False

async def scrape_with_circuit_breaker(page, cb: CloudflareCircuitBreaker):
    if not cb.should_attempt():
        logger.info("Circuit breaker open — using fallback")
        return await fallback_signal_source()

    try:
        content = await page.content()
        if "challenge" in content.lower() or "cloudflare" in content.lower():
            cb.record_failure()
            return await fallback_signal_source()
        cb.record_success()
        return parse_messages(content)
    except Exception as e:
        cb.record_failure()
        raise
```

### Pattern 5: Session Cookie Persistence

```python
import json
from pathlib import Path

COOKIE_FILE = Path("/app/data/session_cookies.json")

async def save_session(context):
    """Persist browser session to survive container restarts."""
    cookies = await context.cookies()
    storage = await context.storage_state()
    COOKIE_FILE.write_text(json.dumps({
        "cookies": cookies,
        "storage": storage,
    }))

async def restore_session(browser) -> "BrowserContext":
    """Restore previous session if available."""
    if COOKIE_FILE.exists():
        try:
            data = json.loads(COOKIE_FILE.read_text())
            context = await browser.new_context(storage_state=data["storage"])
            logger.info("Restored previous browser session")
            return context
        except Exception:
            logger.warning("Failed to restore session — starting fresh")
    return await browser.new_context()

# Save session periodically and after successful login
async def scraper_loop():
    browser = await playwright.chromium.launch()
    context = await restore_session(browser)
    page = await context.new_page()

    while True:
        messages = await scrape_chatroom(page)
        await save_session(context)  # Persist after each successful cycle
        await asyncio.sleep(5)
```

### Pattern 6: Auto-Fallback Chain

```python
class SignalSourceManager:
    """Manages fallback chain: Extension → Scraper → Webhook-only."""

    def __init__(self):
        self.active_source = "extension"
        self.extension_last_seen: float = 0
        self.scraper_circuit_breaker = CloudflareCircuitBreaker()

    async def get_signals(self) -> list[dict]:
        # Priority 1: Extension (most reliable when running)
        if self._extension_is_alive():
            self.active_source = "extension"
            return []  # Extension pushes via webhook — no polling needed

        # Priority 2: Scraper (backup)
        if self.scraper_circuit_breaker.should_attempt():
            try:
                signals = await run_scraper_cycle()
                self.scraper_circuit_breaker.record_success()
                self.active_source = "scraper"
                return signals
            except Exception:
                self.scraper_circuit_breaker.record_failure()

        # Priority 3: Webhook-only (degraded mode)
        self.active_source = "webhook-only"
        logger.warning("All scrapers down — webhook-only mode")
        return []  # Only process manually submitted webhooks

    def _extension_is_alive(self) -> bool:
        return (time.time() - self.extension_last_seen) < 180  # 3-min threshold
```

---

## Red Flags

| Red Flag | Why It's Dangerous | Correct Pattern |
|----------|-------------------|-----------------|
| No `shm_size` in Docker Compose | Chrome crashes with "out of memory" on shared memory | `shm_size: "2g"` or `--disable-dev-shm-usage` flag |
| Health check = "process is alive" | Hung scraper shows as healthy | Check last-successful-scrape timestamp |
| No memory limit on scraper container | Chrome leaks memory → OOM kills other containers | `deploy.resources.limits.memory: 4g` |
| Session cookies in memory only | Lost on container restart → requires re-login | Persist to volume-mounted file |
| No Cloudflare detection | Scraper loops on challenge page, burning CPU | Detect challenge page text, circuit breaker |
| No fallback signal source | Single scraper failure = total signal blackout | Extension → Scraper → Webhook-only chain |
| `playwright.chromium.launch(headless=False)` in Docker | No display server in Docker — crashes immediately | `headless=True` (default) or use Xvfb |

---

## Integration

| Scenario | Related Skill |
|----------|--------------|
| Extension as primary signal source | `trading-bot-skills:chrome-extension-signal-bridge` |
| Signal dedup after scraper ingestion | `trading-bot-skills:chat-signal-parsing-and-dedup` |
| Async scraper lifecycle management | `trading-bot-skills:async-reliability` |
| Webhook receiver for fallback mode | `trading-bot-skills:signal-source-integration` |
| Health monitoring and alerting | `trading-bot-skills:trading-monitoring-and-alerts` |
