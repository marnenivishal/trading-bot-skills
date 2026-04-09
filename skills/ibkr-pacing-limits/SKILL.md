---
name: ibkr-pacing-limits
description: Use when hitting IBKR API pacing violations, rate limits, disconnects from too many requests, order efficiency ratio warnings, or "max market data requests exceeded" errors
---

# IBKR Pacing Limits Screener

## Why This Skill Exists

IBKR enforces strict API pacing limits and order efficiency rules that are poorly
documented and silently enforced. Bots that spam orders, market data requests, or
historical data queries will get throttled, temporarily blocked, or disconnected
without clear error messages. This skill diagnoses pacing issues BEFORE assuming
code bugs or network problems.

---

## When to Activate

Trigger on ANY of:

- "Pacing violation" messages in TWS/API logs
- 429 or rate-limit HTTP errors (Web API / Client Portal)
- "Too many requests" / "max number of market data requests exceeded"
- "API disconnects when I send many orders or data requests"
- Unexplained disconnects during high-activity periods
- "Order efficiency" warnings or restrictions
- Historical data requests returning errors or empty results
- Market data subscriptions dropping silently

**First assumption: The bot is exceeding IBKR's pacing or efficiency limits.**

---

## IBKR Pacing & Rate Limits

### TWS API (Socket-Based)

| Limit | Value | Consequence |
|-------|-------|-------------|
| API messages per second | ~50 msgs/sec | Pacing violation, temporary block |
| Historical data requests | ~6 requests per 2 seconds (same instrument) | "Pacing violation" error, 10-minute cooldown |
| Identical historical requests | 1 per 15 seconds | Cached or rejected |
| Concurrent market data lines | 100 (default, varies by account) | "Max number of market data requests exceeded" |
| Market data request rate | ~1 per second recommended | Throttled or dropped |

### Web / Client Portal API

| Limit | Value | Consequence |
|-------|-------|-------------|
| Requests per second (session) | ~10 req/sec | 429 Too Many Requests |
| Endpoint-specific caps | Varies | Throttled or blocked |
| Session timeout | ~5 minutes idle | Re-authentication required |

### Order Efficiency Ratio

IBKR monitors the ratio of **orders submitted** to **orders executed**:

- Too many unfilled or canceled orders relative to fills triggers restrictions
- Cancel/replace spam (modify-cancel-resubmit loops) counts against this ratio
- Restrictions can include temporary order submission blocks or account warnings
- Particularly aggressive monitoring on options and futures

---

## Diagnostic Checklist

**Run this when pacing issues are suspected:**

### Step 1: Identify What's Being Spammed

Ask the user or check code for:

- [ ] **Order spam**: Rapid submit/cancel/modify cycles (scalping, trailing stop adjustments)
- [ ] **Market data overload**: Subscribing to too many tickers simultaneously
- [ ] **Historical data flooding**: Requesting same bars repeatedly or many instruments at once
- [ ] **Tick-by-tick requests**: Requesting tick data for many symbols concurrently

### Step 2: Measure Current Request Rate

Check the bot's actual request frequency:

```python
# Add logging to measure request rate
import time
_request_times = []

def log_request():
    now = time.time()
    _request_times.append(now)
    # Count requests in last second
    recent = [t for t in _request_times if now - t < 1.0]
    if len(recent) > 40:
        logger.warning("Approaching pacing limit: %d requests/sec", len(recent))
    # Prune old entries
    _request_times[:] = [t for t in _request_times if now - t < 60.0]
```

### Step 3: Check TWS Logs for Pacing Errors

Look for these patterns in TWS/API logs:

- `"Pacing violation"` — historical data request too fast
- `"Max number of market data lines"` — too many concurrent subscriptions
- `"Order held"` with timing correlation — order rate exceeded
- Connection drops correlated with burst activity

### Step 4: Check Order Efficiency

If order-related restrictions appear:

- Calculate: `fills / (submissions + modifications + cancellations)` over last hour
- If ratio is below ~1:10 (1 fill per 10 order actions), IBKR may flag the account
- Cancel/replace loops for trailing stops are a common trigger

---

## Throttling Solutions

### Token Bucket Rate Limiter

Implement in the bot to cap request rate:

```python
import asyncio
import time

class RateLimiter:
    """Token bucket rate limiter for IBKR API calls."""

    def __init__(self, max_per_second: float = 40.0):
        self._max = max_per_second
        self._tokens = max_per_second
        self._last_refill = time.monotonic()
        self._lock = asyncio.Lock()

    async def acquire(self):
        async with self._lock:
            now = time.monotonic()
            elapsed = now - self._last_refill
            self._tokens = min(self._max, self._tokens + elapsed * self._max)
            self._last_refill = now

            if self._tokens < 1.0:
                wait = (1.0 - self._tokens) / self._max
                await asyncio.sleep(wait)
                self._tokens = 0.0
            else:
                self._tokens -= 1.0
```

### Recommended Rate Limits by Request Type

| Request Type | Safe Rate | Burst Limit |
|-------------|-----------|-------------|
| Order submissions | 10/sec | 25/sec for 2 seconds |
| Order modifications | 5/sec | 10/sec briefly |
| Market data subscriptions | 1/sec | 5 in a burst |
| Historical data (same ticker) | 1 per 2 seconds | Never faster |
| Historical data (different tickers) | 3/sec | 6 in a burst |
| Account/portfolio queries | 1/sec | 2/sec briefly |

### Backoff Strategy for Pacing Errors

```python
async def request_with_backoff(func, *args, max_retries=3):
    """Exponential backoff on pacing violations."""
    for attempt in range(max_retries):
        try:
            return await func(*args)
        except PacingViolation:
            wait = min(2 ** attempt * 5, 60)  # 5s, 10s, 20s, max 60s
            logger.warning("Pacing violation, backing off %ds (attempt %d/%d)",
                          wait, attempt + 1, max_retries)
            await asyncio.sleep(wait)
    raise PacingViolation(f"Still throttled after {max_retries} retries")
```

---

## Order Efficiency Improvements

### Problem: Cancel/Replace Spam

Bad pattern (triggers efficiency warnings):
```
submit order -> price moves -> cancel -> submit new -> price moves -> cancel -> ...
```

Better pattern:
```
submit order -> wait for meaningful price change -> modify order (single action)
```

### Specific Recommendations

1. **Trailing stops**: Update stop price only when price moves by a meaningful increment (e.g., $0.10 for stocks, $0.05 for options), not on every tick
2. **Limit order chasing**: Use modify (not cancel+resubmit) and only adjust after significant spread change
3. **Batch market data**: Subscribe to needed symbols at startup, don't subscribe/unsubscribe repeatedly
4. **Historical data caching**: Cache bars locally, only request new data on interval (not per-tick)
5. **Combine signals**: Aggregate multiple signals into fewer, more meaningful order actions

---

## IBKR is NOT an HFT Venue

**Critical warning for scalping/high-frequency bots:**

- IBKR's retail API is designed for moderate-frequency trading (seconds to minutes between actions)
- Sub-second order cycling WILL trigger pacing violations and efficiency restrictions
- If the bot needs to modify orders more than ~5 times per second per instrument, the strategy is incompatible with IBKR's retail API
- Consider: consolidating signals, using wider trailing increments, or accepting slightly worse fills to stay within limits

---

## Common Misdiagnoses This Skill Prevents

| Symptom | Wrong Diagnosis | Actual Cause |
|---------|----------------|--------------|
| Random disconnects during trading | "Network issue" or "Docker problem" | Message rate exceeded ~50/sec |
| Historical data returns empty | "Wrong contract specification" | Pacing violation cooldown (10 min) |
| Market data stops updating | "Subscription dropped" | Max concurrent data lines exceeded |
| Orders rejected intermittently | "Invalid order parameters" | Order rate or efficiency ratio exceeded |
| Account restricted from trading | "Compliance issue" | Order efficiency ratio too low |

---

## Integration with Other Skills

- **ibkr-api-precautions**: Rule out precaution dialogs first, THEN check pacing
- **broker-api-integration**: Use circuit breaker pattern to stop hammering on pacing errors
- **async-reliability**: Ensure rate limiter works correctly in async context
- **kill-switch-and-circuit-breakers**: Auto-halt if pacing violations exceed threshold
