---
name: broker-api-integration
description: Use when connecting to any broker API (Alpaca, IBKR, Tradier), implementing order submission, or handling broker events and webhooks
---

# Broker API Integration

## The Iron Law

**EVERY BROKER CALL MUST HAVE: TIMEOUT, RETRY LIMIT, CIRCUIT BREAKER, AND IDEMPOTENT CLIENT_ORDER_ID.**

No exceptions. No "I'll add it later." If a broker call is missing any of these four,
the code is not production-ready. Period.

## Why This Skill Exists

Broker APIs are unreliable by nature. Networks drop, APIs return 5xx, WebSockets go
stale, and orders get submitted twice. Without disciplined integration patterns, you
will experience:

- **Duplicate orders** from retries without idempotency
- **Hung positions** from undetected failed cancellations
- **Cascading failures** when a broker outage causes your bot to hammer a dead endpoint
- **Silent data loss** when WebSocket connections die without detection

---

## Circuit Breaker Pattern

After N consecutive failures, STOP sending requests. Alert. Wait for manual reset or
cooldown expiry.

### States

```
CLOSED  --[failure_count >= threshold]--> OPEN
OPEN    --[cooldown_elapsed]-----------> HALF_OPEN
HALF_OPEN --[success]-------------------> CLOSED
HALF_OPEN --[failure]-------------------> OPEN
```

### Implementation

```python
import time
from enum import Enum
from dataclasses import dataclass, field
from typing import Optional
import logging

logger = logging.getLogger(__name__)


class CircuitState(Enum):
    CLOSED = "closed"
    OPEN = "open"
    HALF_OPEN = "half_open"


@dataclass
class CircuitBreaker:
    """Circuit breaker for broker API calls.

    Opens after `failure_threshold` consecutive failures.
    Transitions to HALF_OPEN after `recovery_timeout` seconds.
    A single success in HALF_OPEN closes the circuit.
    A single failure in HALF_OPEN reopens immediately.
    """
    name: str
    failure_threshold: int = 5
    recovery_timeout: float = 60.0  # seconds
    state: CircuitState = CircuitState.CLOSED
    failure_count: int = 0
    last_failure_time: Optional[float] = None
    _listeners: list = field(default_factory=list)

    def can_execute(self) -> bool:
        if self.state == CircuitState.CLOSED:
            return True
        if self.state == CircuitState.OPEN:
            if time.time() - self.last_failure_time >= self.recovery_timeout:
                self.state = CircuitState.HALF_OPEN
                logger.warning(f"CircuitBreaker[{self.name}]: OPEN -> HALF_OPEN (attempting recovery)")
                return True
            return False
        if self.state == CircuitState.HALF_OPEN:
            return True  # allow one probe request
        return False

    def record_success(self):
        if self.state == CircuitState.HALF_OPEN:
            logger.info(f"CircuitBreaker[{self.name}]: HALF_OPEN -> CLOSED (recovered)")
        self.state = CircuitState.CLOSED
        self.failure_count = 0

    def record_failure(self):
        self.failure_count += 1
        self.last_failure_time = time.time()
        if self.state == CircuitState.HALF_OPEN:
            self.state = CircuitState.OPEN
            logger.error(f"CircuitBreaker[{self.name}]: HALF_OPEN -> OPEN (recovery failed)")
            self._notify_open()
        elif self.failure_count >= self.failure_threshold:
            self.state = CircuitState.OPEN
            logger.error(
                f"CircuitBreaker[{self.name}]: CLOSED -> OPEN "
                f"(after {self.failure_count} consecutive failures)"
            )
            self._notify_open()

    def on_open(self, callback):
        self._listeners.append(callback)

    def _notify_open(self):
        for cb in self._listeners:
            try:
                cb(self.name, self.failure_count)
            except Exception:
                pass
```

---

## Idempotent Order IDs

Generate `client_order_id` deterministically so retries do not create duplicate orders.

```python
import hashlib


def generate_client_order_id(
    signal_id: str,
    symbol: str,
    side: str,
    timestamp_bucket: int,
) -> str:
    """Deterministic order ID from signal parameters.

    timestamp_bucket should be floored to a window (e.g., 60s) so that
    retries within the same window produce the same ID.

    The broker rejects duplicate client_order_ids, preventing double-entry.
    """
    raw = f"{signal_id}|{symbol}|{side}|{timestamp_bucket}"
    hash_hex = hashlib.sha256(raw.encode()).hexdigest()[:16]
    return f"{symbol}-{side}-{hash_hex}"


# Example: bucket timestamps to 60-second windows
import time

bucket = int(time.time()) // 60
order_id = generate_client_order_id("ema_cross_001", "SPY", "buy", bucket)
# => "SPY-buy-a3f8c1d9e7b24510"
```

**Why timestamp bucketing?** If your signal fires at 09:31:12 and you retry at
09:31:45, both produce the same bucket (assuming 60s windows). The broker deduplicates
the second submission. Without bucketing, each retry gets a unique ID and the broker
happily accepts all of them.

---

## Rate Limiting

Brokers enforce rate limits. Exceeding them causes 429 errors or temporary bans.

```python
import time
import threading
from collections import deque


class RateLimiter:
    """Sliding window rate limiter.

    Alpaca: 200 requests/minute for orders.
    IBKR: varies by endpoint.
    """
    def __init__(self, max_requests: int, window_seconds: float):
        self.max_requests = max_requests
        self.window_seconds = window_seconds
        self._timestamps: deque = deque()
        self._lock = threading.Lock()

    def acquire(self, timeout: float = 30.0) -> bool:
        """Block until a slot is available. Returns False on timeout."""
        deadline = time.time() + timeout
        while time.time() < deadline:
            with self._lock:
                now = time.time()
                # purge timestamps outside the window
                while self._timestamps and self._timestamps[0] < now - self.window_seconds:
                    self._timestamps.popleft()
                if len(self._timestamps) < self.max_requests:
                    self._timestamps.append(now)
                    return True
            time.sleep(0.1)
        return False


# Usage
order_limiter = RateLimiter(max_requests=190, window_seconds=60)  # 10 req buffer

if not order_limiter.acquire(timeout=10):
    logger.error("Rate limit: could not acquire slot in 10s")
    raise RuntimeError("Rate limited")
```

---

## WebSocket Lifecycle Management

WebSocket connections are fragile. They go stale silently. You MUST:

1. **Detect stale connections** -- if no message received in N seconds, assume dead
2. **Reconnect with exponential backoff** -- do not hammer the server
3. **Resubscribe on reconnect** -- subscriptions are lost when connections drop
4. **Process messages idempotently** -- you may receive duplicates after reconnect

```python
import asyncio
import websockets
import json
import logging

logger = logging.getLogger(__name__)


class BrokerWebSocket:
    """Resilient WebSocket client with reconnection and stale detection."""

    def __init__(self, url: str, subscriptions: list[str], on_message, auth_msg: dict):
        self.url = url
        self.subscriptions = subscriptions
        self.on_message = on_message
        self.auth_msg = auth_msg
        self._ws = None
        self._reconnect_delay = 1.0
        self._max_reconnect_delay = 60.0
        self._stale_timeout = 30.0  # seconds with no message = stale
        self._running = False

    async def run(self):
        self._running = True
        while self._running:
            try:
                await self._connect_and_listen()
            except Exception as e:
                logger.error(f"WebSocket error: {e}")
            if self._running:
                logger.warning(
                    f"Reconnecting in {self._reconnect_delay:.1f}s..."
                )
                await asyncio.sleep(self._reconnect_delay)
                self._reconnect_delay = min(
                    self._reconnect_delay * 2,
                    self._max_reconnect_delay,
                )

    async def _connect_and_listen(self):
        async with websockets.connect(self.url) as ws:
            self._ws = ws
            self._reconnect_delay = 1.0  # reset on successful connect
            logger.info("WebSocket connected, authenticating...")

            await ws.send(json.dumps(self.auth_msg))
            auth_resp = await asyncio.wait_for(ws.recv(), timeout=10)
            logger.info(f"Auth response: {auth_resp}")

            # Resubscribe
            for sub in self.subscriptions:
                await ws.send(json.dumps(sub))
            logger.info(f"Subscribed to {len(self.subscriptions)} channels")

            # Listen with stale detection
            while True:
                try:
                    raw = await asyncio.wait_for(
                        ws.recv(), timeout=self._stale_timeout
                    )
                    data = json.loads(raw)
                    await self.on_message(data)
                except asyncio.TimeoutError:
                    logger.warning("WebSocket stale (no messages), forcing reconnect")
                    await ws.close()
                    return

    def stop(self):
        self._running = False
```

---

## Broker Abstraction Layer

**Never scatter direct broker SDK calls across your codebase.** Define a unified interface.

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass
from enum import Enum
from typing import Optional
from decimal import Decimal


class OrderSide(Enum):
    BUY = "buy"
    SELL = "sell"


class OrderType(Enum):
    MARKET = "market"
    LIMIT = "limit"
    STOP = "stop"
    STOP_LIMIT = "stop_limit"


class TimeInForce(Enum):
    DAY = "day"
    GTC = "gtc"
    IOC = "ioc"


@dataclass
class OrderRequest:
    symbol: str
    side: OrderSide
    qty: Decimal
    order_type: OrderType
    time_in_force: TimeInForce = TimeInForce.DAY
    limit_price: Optional[Decimal] = None
    stop_price: Optional[Decimal] = None
    client_order_id: Optional[str] = None


@dataclass
class OrderResult:
    broker_order_id: str
    client_order_id: str
    status: str
    filled_qty: Decimal
    filled_avg_price: Optional[Decimal]


@dataclass
class Position:
    symbol: str
    qty: Decimal
    avg_entry_price: Decimal
    market_value: Decimal
    unrealized_pl: Decimal
    side: str  # "long" or "short"


@dataclass
class AccountInfo:
    equity: Decimal
    buying_power: Decimal
    cash: Decimal
    portfolio_value: Decimal
    pattern_day_trader: bool


class BrokerInterface(ABC):
    """All broker implementations MUST implement this interface.

    Every method must:
    - Have a timeout
    - Use the circuit breaker
    - Be retryable (idempotent where applicable)
    """

    @abstractmethod
    async def submit_order(self, request: OrderRequest) -> OrderResult:
        ...

    @abstractmethod
    async def cancel_order(self, broker_order_id: str) -> bool:
        ...

    @abstractmethod
    async def get_positions(self) -> list[Position]:
        ...

    @abstractmethod
    async def get_account(self) -> AccountInfo:
        ...

    @abstractmethod
    async def get_order_status(self, broker_order_id: str) -> OrderResult:
        ...

    @abstractmethod
    async def subscribe_order_updates(self, callback) -> None:
        ...
```

---

## Paper / Live Separation

**Paper and live environments MUST be completely separate.** Different API keys,
different URLs, different database tables or schemas.

```python
from dataclasses import dataclass


@dataclass
class BrokerConfig:
    mode: str  # "paper" or "live"
    api_key: str
    api_secret: str
    base_url: str
    ws_url: str
    db_schema: str  # "paper" or "live" -- SEPARATE SCHEMAS

    def __post_init__(self):
        assert self.mode in ("paper", "live")
        # Safety: verify URLs match declared mode
        if self.mode == "paper":
            assert "paper" in self.base_url.lower(), \
                f"Paper mode but URL doesn't contain 'paper': {self.base_url}"
        if self.mode == "live":
            assert "paper" not in self.base_url.lower(), \
                f"Live mode but URL contains 'paper': {self.base_url}"


PAPER_CONFIG = BrokerConfig(
    mode="paper",
    api_key="PAPER_KEY_FROM_ENV",
    api_secret="PAPER_SECRET_FROM_ENV",
    base_url="https://paper-api.alpaca.markets",
    ws_url="wss://paper-api.alpaca.markets/stream",
    db_schema="paper",
)

LIVE_CONFIG = BrokerConfig(
    mode="live",
    api_key="LIVE_KEY_FROM_ENV",
    api_secret="LIVE_SECRET_FROM_ENV",
    base_url="https://api.alpaca.markets",
    ws_url="wss://api.alpaca.markets/stream",
    db_schema="live",
)
```

**Never** read API keys from code. Always from environment variables or a secrets manager.

---

## Putting It All Together: Protected Broker Call

```python
import asyncio
import logging
from typing import TypeVar, Callable, Awaitable

logger = logging.getLogger(__name__)
T = TypeVar("T")


async def protected_broker_call(
    func: Callable[..., Awaitable[T]],
    *args,
    circuit_breaker: CircuitBreaker,
    rate_limiter: RateLimiter,
    timeout: float = 10.0,
    max_retries: int = 3,
    retry_delay: float = 1.0,
    **kwargs,
) -> T:
    """Every broker call goes through this wrapper. No exceptions.

    Enforces: timeout, retry, circuit breaker, rate limiting.
    """
    for attempt in range(max_retries):
        # 1. Circuit breaker check
        if not circuit_breaker.can_execute():
            raise RuntimeError(
                f"Circuit breaker [{circuit_breaker.name}] is OPEN. "
                f"Broker calls blocked."
            )

        # 2. Rate limit
        if not rate_limiter.acquire(timeout=5):
            raise RuntimeError("Rate limit exceeded, could not acquire slot")

        try:
            # 3. Timeout-wrapped call
            result = await asyncio.wait_for(
                func(*args, **kwargs),
                timeout=timeout,
            )
            # 4. Record success
            circuit_breaker.record_success()
            return result

        except asyncio.TimeoutError:
            logger.warning(
                f"Broker call timeout (attempt {attempt + 1}/{max_retries})"
            )
            circuit_breaker.record_failure()
        except Exception as e:
            logger.error(
                f"Broker call failed (attempt {attempt + 1}/{max_retries}): {e}"
            )
            circuit_breaker.record_failure()

        if attempt < max_retries - 1:
            await asyncio.sleep(retry_delay * (2 ** attempt))

    raise RuntimeError(f"Broker call failed after {max_retries} retries")
```

---

## Red Flags -- Stop and Fix Immediately

| Red Flag | Why It's Dangerous |
|---|---|
| Direct broker SDK calls scattered across modules | No central timeout/retry/circuit-breaker enforcement |
| No `client_order_id` on order submissions | Retries create duplicate orders |
| No timeout on broker HTTP calls | Hung connection blocks the event loop or thread |
| No circuit breaker | Broker outage causes thousands of failed requests and possible rate-ban |
| WebSocket with no stale detection | Connection dies silently; you miss all fills and events |
| Same API keys for paper and live | One config mistake sends live orders during testing |
| Rate limiter missing or too generous | 429 errors, temporary bans, missed orders |

---

## Account Type Constraints and PDT Compliance

### Pattern Day Trader Rule

FINRA requires $25,000 minimum equity for accounts executing 4+ day trades in 5 rolling business days. If equity drops below $25k, the account is restricted to closing-only trades for 90 calendar days. This rule applies to margin accounts only.

### PDT as Pre-Trade Gate

Check BEFORE every order, not after the broker rejects. Relying on broker rejection as your safety net means you discover the problem after the order fails -- by then your strategy state may already assume the order was placed.

```python
@dataclass
class DayTradeTracker:
    trades: list[DayTrade] = field(default_factory=list)

    def count_rolling_5_days(self) -> int:
        cutoff = datetime.now(UTC) - timedelta(days=5)
        return sum(1 for t in self.trades if t.date >= cutoff and t.is_day_trade)

    def can_day_trade(self, account_equity: Decimal, is_margin: bool) -> PDTDecision:
        if not is_margin:
            return PDTDecision(allowed=True, reason="Cash account: PDT does not apply")
        if account_equity >= Decimal("25000"):
            return PDTDecision(allowed=True, reason="Equity >= $25k")
        if self.count_rolling_5_days() >= 3:
            return PDTDecision(allowed=False, reason=f"PDT limit: {self.count_rolling_5_days()}/3 day trades used, equity ${account_equity} < $25k")
        return PDTDecision(allowed=True, reason=f"Day trades remaining: {3 - self.count_rolling_5_days()}")
```

### Cash Account vs Margin

Cash accounts avoid PDT entirely but face T+1 settlement. Buying power equals settled cash only -- unsettled funds from recent sales cannot be used to open new positions.

```python
@dataclass
class SettlementTracker:
    pending_settlements: list[Settlement] = field(default_factory=list)

    def available_buying_power(self, cash_balance: Decimal) -> Decimal:
        unsettled = sum(s.amount for s in self.pending_settlements if s.settlement_date > date.today())
        return cash_balance - unsettled
```

### PDT Workarounds

- **Cash account cycling**: trade only with settled funds. After selling, wait T+1 for settlement before reusing those funds. Rotate across multiple positions to keep capital deployed.
- **Futures and index options**: CFTC-regulated instruments (e.g., /ES, /NQ, SPX options, NDX options) are exempt from FINRA PDT rules entirely.
- **Multiple broker accounts**: spread day trades across separate brokers to stay under 3 day trades per account per rolling 5-day window. Each account is tracked independently.

### Red Flags

| Red Flag | Why It's Dangerous |
|---|---|
| No PDT check before orders | Order rejected by broker mid-strategy, leaving inconsistent state |
| Relying on broker rejection as safety net | Discover the problem too late; strategy state already updated |
| No settlement tracking for cash accounts | Free-riding violation; broker may restrict the account |
| Mixing cash and margin logic | Wrong buying power calculation; orders fail or over-leverage |

See `pdt-compliance-reference.md` in this directory for detailed settlement timelines, account type decision matrix, and day trade counting rules.

---

## Integration Points

- **trading-bot-skills:order-execution-integrity** -- This skill provides the transport layer; order-execution-integrity provides the logic layer (dedup, state machine, fill tracking).
- **trading-bot-skills:trading-bot-architecture** -- The broker abstraction layer is a core component of the overall bot architecture. Architecture decisions (async vs sync, event-driven vs polling) affect broker integration deeply.
