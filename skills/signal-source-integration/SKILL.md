---
name: signal-source-integration
description: Use when ingesting signals from external sources like Discord, Telegram, webhooks, or alert services, or when parsing alert messages into structured trade signals
---

# Signal Source Integration

**Iron Law:** EVERY EXTERNAL SIGNAL MUST BE AUTHENTICATED, VALIDATED, RATE-LIMITED, AND PARSED INTO A TYPED STRUCT BEFORE REACHING THE SIGNAL ENGINE. RAW TEXT NEVER REACHES STRATEGY CODE.

External signals are untrusted input. They arrive as unstructured text over unauthenticated channels. Treating them as anything other than hostile input is a fast path to rogue trades.

## Webhook Receiver Pattern

The webhook endpoint is the front door. Lock it down. Every request must prove its identity via HMAC-SHA256 before the body is even parsed.

```python
import hashlib
import hmac
import time
from dataclasses import dataclass, field
from enum import Enum
from typing import Optional

from fastapi import FastAPI, Request, HTTPException


WEBHOOK_SECRET = b"your-secret-key-from-env"  # load from env, never hardcode
ALLOWED_IPS = {"52.89.214.238", "34.212.75.30"}  # allowlist source IPs


app = FastAPI()


def verify_hmac(payload: bytes, signature: str, secret: bytes) -> bool:
    """Verify HMAC-SHA256 signature on incoming webhook."""
    expected = hmac.new(secret, payload, hashlib.sha256).hexdigest()
    return hmac.compare_digest(expected, signature)


@app.post("/webhook/signals")
async def receive_signal(request: Request):
    # Step 1: IP allowlist
    client_ip = request.client.host
    if client_ip not in ALLOWED_IPS:
        raise HTTPException(status_code=403, detail="IP not allowed")

    # Step 2: HMAC signature verification
    signature = request.headers.get("X-Signature-256", "")
    body = await request.body()
    if not verify_hmac(body, signature, WEBHOOK_SECRET):
        raise HTTPException(status_code=401, detail="Invalid signature")

    # Step 3: Parse and validate (never pass raw body downstream)
    try:
        signal = parse_alert_signal(body.decode("utf-8"))
    except ParseError as e:
        audit_log.log("signal_parse_error", raw=body.decode("utf-8"), error=str(e))
        # Return 200 to acknowledge receipt -- do not leak error details
        return {"status": "acknowledged", "acted": False}

    # Step 4: Rate limit check
    if not rate_limiter.allow(source="webhook", key=signal.symbol):
        audit_log.log("signal_rate_limited", signal=signal)
        return {"status": "acknowledged", "acted": False}

    # Step 5: Content hash dedup (prevent replay)
    if dedup_filter.is_duplicate(body):
        audit_log.log("signal_duplicate_rejected", signal=signal)
        return {"status": "acknowledged", "acted": False}

    # Step 6: Enqueue into Intent Bus as typed Signal
    intent_bus.submit(signal.to_intent())
    audit_log.log("signal_accepted", signal=signal)
    return {"status": "acknowledged", "acted": True}
```

Key design decisions:
- Return 200 for acknowledged, regardless of whether the bot acts. The caller should not know your internal decision logic.
- Parse errors are logged but never returned in the response body.
- HMAC comparison uses `hmac.compare_digest` to prevent timing attacks.

## NLP Alert Parsing

Alert services send messages in semi-structured text. A regex-based parser extracts fields into a typed struct. If parsing fails, the signal is rejected -- never silently dropped, always logged.

```python
import re
from dataclasses import dataclass
from typing import Optional


class ParseError(Exception):
    """Raised when an alert message cannot be parsed into a valid signal."""
    pass


class Action(Enum):
    BUY_TO_OPEN = "BTO"
    SELL_TO_CLOSE = "STC"
    BUY = "BUY"
    SELL = "SELL"


@dataclass
class AlertSignal:
    action: Action
    symbol: str
    strike: Optional[float]
    option_type: Optional[str]  # "C" or "P"
    price: Optional[float]
    raw_text: str

    def to_intent(self):
        """Convert to Intent Bus signal format."""
        return {
            "action": self.action.value,
            "symbol": self.symbol,
            "strike": self.strike,
            "option_type": self.option_type,
            "price": self.price,
            "source": "external_alert",
        }


# Patterns for common alert formats
# "BTO AAPL 150C 1.50"  ->  action=BTO, symbol=AAPL, strike=150, type=C, price=1.50
# "SELL SPY 450P"        ->  action=SELL, symbol=SPY, strike=450, type=P, price=None
# "BUY MSFT 320 @ 2.10" ->  action=BUY, symbol=MSFT, strike=320, price=2.10

OPTION_PATTERN = re.compile(
    r"^(BTO|STC|BUY|SELL)\s+"       # action
    r"([A-Z]{1,5})\s+"              # symbol (1-5 uppercase letters)
    r"(\d+(?:\.\d+)?)"              # strike price
    r"([CP])"                        # option type
    r"(?:\s+(?:@\s*)?(\d+(?:\.\d+)?))?$"  # optional price
)

EQUITY_PATTERN = re.compile(
    r"^(BUY|SELL)\s+"
    r"([A-Z]{1,5})"
    r"(?:\s+(?:@\s*)?(\d+(?:\.\d+)?))?$"
)


def parse_alert_signal(text: str) -> AlertSignal:
    """Parse alert text into a typed AlertSignal. Raises ParseError on failure."""
    text = text.strip().upper()

    # Try option pattern first
    m = OPTION_PATTERN.match(text)
    if m:
        action_str, symbol, strike, opt_type, price = m.groups()
        return AlertSignal(
            action=Action(action_str),
            symbol=symbol,
            strike=float(strike),
            option_type=opt_type,
            price=float(price) if price else None,
            raw_text=text,
        )

    # Try equity pattern
    m = EQUITY_PATTERN.match(text)
    if m:
        action_str, symbol, price = m.groups()
        return AlertSignal(
            action=Action(action_str),
            symbol=symbol,
            strike=None,
            option_type=None,
            price=float(price) if price else None,
            raw_text=text,
        )

    raise ParseError(f"Cannot parse alert: '{text}'")
```

## Rate Limiting

Per-source rate limiting prevents a compromised or malfunctioning source from flooding the bot with signals. Token bucket algorithm allows bursts while enforcing a sustained rate.

```python
import time
from dataclasses import dataclass, field


@dataclass
class TokenBucket:
    capacity: float
    refill_rate: float  # tokens per second
    tokens: float = 0.0
    last_refill: float = field(default_factory=time.time)

    def allow(self) -> bool:
        now = time.time()
        elapsed = now - self.last_refill
        self.tokens = min(self.capacity, self.tokens + elapsed * self.refill_rate)
        self.last_refill = now

        if self.tokens >= 1.0:
            self.tokens -= 1.0
            return True
        return False


class SignalRateLimiter:
    """Per-source, per-symbol rate limiter using token bucket."""

    def __init__(self, capacity: float = 5.0, refill_rate: float = 0.1):
        self._buckets: dict[str, TokenBucket] = {}
        self._capacity = capacity
        self._refill_rate = refill_rate

    def allow(self, source: str, key: str) -> bool:
        bucket_key = f"{source}:{key}"
        if bucket_key not in self._buckets:
            self._buckets[bucket_key] = TokenBucket(
                capacity=self._capacity,
                refill_rate=self._refill_rate,
                tokens=self._capacity,
            )
        return self._buckets[bucket_key].allow()
```

## Content Hash Dedup (Replay Prevention)

Replay attacks send the same webhook payload multiple times. Hash the content and reject duplicates within a time window.

```python
import hashlib
import time


class ContentHashDedup:
    """Reject duplicate payloads within a time window."""

    def __init__(self, window_seconds: int = 300):
        self._seen: dict[str, float] = {}
        self._window = window_seconds

    def is_duplicate(self, payload: bytes) -> bool:
        content_hash = hashlib.sha256(payload).hexdigest()
        now = time.time()

        # Prune expired entries
        expired = [k for k, t in self._seen.items() if now - t > self._window]
        for k in expired:
            del self._seen[k]

        if content_hash in self._seen:
            return True

        self._seen[content_hash] = now
        return False
```

## Source Authentication by Channel

Different sources require different authentication mechanisms.

```python
# --- BAD: Trust any incoming webhook ---
@app.post("/webhook/signals")
async def bad_webhook(request: Request):
    body = await request.json()
    process_signal(body)  # No auth, no validation, no rate limit
    return {"ok": True}

# --- GOOD: Full authentication pipeline ---
@app.post("/webhook/signals")
async def good_webhook(request: Request):
    verify_ip(request)          # IP allowlist
    verify_hmac(request)        # HMAC-SHA256 signature
    signal = parse_and_validate(request)  # Typed parsing
    check_rate_limit(signal)    # Per-source rate limit
    check_dedup(request)        # Replay prevention
    intent_bus.submit(signal)   # Only now enter the pipeline
```

## Signal Normalization

All external signals, regardless of source, must be normalized into a standard `Signal` dataclass before entering the Intent Bus. This ensures strategy code never handles source-specific formats.

```python
from dataclasses import dataclass
from datetime import datetime
from typing import Optional


@dataclass
class NormalizedSignal:
    """Standard signal format for the Intent Bus. All sources normalize to this."""
    source: str               # "webhook", "telegram", "discord", "manual"
    action: str               # "BTO", "STC", "BUY", "SELL"
    symbol: str
    quantity: Optional[int]
    limit_price: Optional[float]
    strike: Optional[float]
    option_type: Optional[str]
    received_at: datetime
    raw_text: str
    trace_id: str             # unique ID for audit trail correlation

    def validate(self) -> list[str]:
        """Return list of validation errors. Empty list means valid."""
        errors = []
        if not self.symbol or not self.symbol.isalpha():
            errors.append(f"invalid symbol: {self.symbol}")
        if self.action not in ("BTO", "STC", "BUY", "SELL"):
            errors.append(f"invalid action: {self.action}")
        if self.limit_price is not None and self.limit_price <= 0:
            errors.append(f"invalid price: {self.limit_price}")
        if self.strike is not None and self.strike <= 0:
            errors.append(f"invalid strike: {self.strike}")
        if self.option_type and self.option_type not in ("C", "P"):
            errors.append(f"invalid option type: {self.option_type}")
        return errors
```

## Source Health Monitoring

Detect when a source goes silent. If your Discord channel normally sends 10 signals per hour and suddenly sends zero for 30 minutes, something is wrong.

```python
import time
from dataclasses import dataclass


@dataclass
class SourceHealthConfig:
    source_name: str
    max_silence_seconds: int  # alert if no messages for this long
    expected_rate_per_hour: float  # for anomaly detection


class SourceHealthMonitor:
    """Detect when signal sources go silent or behave anomalously."""

    def __init__(self, configs: list[SourceHealthConfig]):
        self._configs = {c.source_name: c for c in configs}
        self._last_seen: dict[str, float] = {}

    def record_signal(self, source: str) -> None:
        self._last_seen[source] = time.time()

    def check_health(self) -> list[str]:
        alerts = []
        now = time.time()
        for name, config in self._configs.items():
            last = self._last_seen.get(name, 0)
            silence = now - last if last > 0 else float("inf")
            if silence > config.max_silence_seconds:
                alerts.append(
                    f"Source '{name}' silent for {silence:.0f}s "
                    f"(threshold: {config.max_silence_seconds}s)"
                )
        return alerts
```

## Red Flags

| Red Flag | Why It Matters |
|---|---|
| Raw webhook body passed to strategy code | Untrusted input reaches trading logic, injection risk |
| No HMAC validation on webhooks | Anyone can send fake signals to your bot |
| No rate limiting on signal sources | Compromised source floods bot with orders |
| Parser silently drops unparseable messages | You lose signals without knowing -- silent data loss |
| External signal bypasses Intent Bus | Skips dedup, risk gates, and audit logging |
| No replay prevention (content hash dedup) | Same signal processed multiple times |
| No source health monitoring | Source dies silently, bot stops trading without alerting |
| Webhook returns error details in response | Leaks internal logic to external callers |

## Integration

- **strategy-signal-validation** -- external signals pass through the same confirmation gate as internally generated signals. No signal, regardless of source, gets special treatment.
- **trading-bot-architecture** -- all normalized signals enter via the Intent Bus. The Intent Bus is the single entry point for all trade intents.
- **pre-trade-validation** -- after a signal becomes an intent, it still must pass pre-trade checks (symbol validity, market hours, liquidity).
- **trade-audit-and-replay** -- every signal received, parsed, rejected, or accepted is logged to the audit trail with its trace_id for full traceability.
