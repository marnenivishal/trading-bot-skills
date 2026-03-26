---
name: pre-trade-validation
description: Use when implementing order submission logic, adding pre-trade checks, or when orders are rejected by the broker for invalid symbols, closed markets, or insufficient liquidity
---

# Pre-Trade Validation

**Iron Law:** EVERY ORDER MUST PASS ALL PRE-TRADE CHECKS BEFORE REACHING THE BROKER. BROKER REJECTION IS NOT A SAFETY NET.

Broker rejections are noisy, slow, and unreliable as a validation layer. They burn rate limits, trigger account flags, and provide inconsistent error messages across brokers. Your bot must catch every invalid order locally before it ever touches the wire.

## Pre-Trade Validation Gate

The validation gate runs BEFORE dedup and risk gates. Every order candidate must pass all six checks in sequence. A failure at any step short-circuits the pipeline and logs the rejection reason.

### The 6 Checks (in order)

1. **Symbol exists and is tradeable** -- validate against broker API + internal symbol cache
2. **Market is open** -- regular hours, extended hours, half-days, holidays
3. **Liquidity sufficient** -- bid-ask spread below threshold, volume above minimum
4. **Price within acceptable range** -- no stale quotes, no zero prices, no absurd deviations
5. **No duplicate pending orders** -- same symbol + same side already in flight
6. **Account has sufficient buying power** -- including T+1 settlement for cash accounts

```python
from dataclasses import dataclass, field
from enum import Enum
from typing import Optional


class ValidationFailure(Enum):
    SYMBOL_NOT_FOUND = "symbol_not_found"
    SYMBOL_NOT_TRADEABLE = "symbol_not_tradeable"
    MARKET_CLOSED = "market_closed"
    INSUFFICIENT_LIQUIDITY = "insufficient_liquidity"
    STALE_QUOTE = "stale_quote"
    ZERO_PRICE = "zero_price"
    PRICE_OUT_OF_RANGE = "price_out_of_range"
    DUPLICATE_PENDING = "duplicate_pending"
    INSUFFICIENT_BUYING_POWER = "insufficient_buying_power"


@dataclass
class ValidationResult:
    passed: bool
    failures: list[str] = field(default_factory=list)

    def add_failure(self, failure: ValidationFailure, detail: str = "") -> None:
        self.passed = False
        msg = failure.value
        if detail:
            msg += f": {detail}"
        self.failures.append(msg)
```

## Symbol Registry Cache

Hitting the broker API for symbol validation on every order is wasteful and slow. Maintain a local cache with a TTL, refresh daily, and fall back to the broker API on cache miss.

```python
import time
from dataclasses import dataclass


@dataclass
class SymbolInfo:
    symbol: str
    tradeable: bool
    asset_class: str  # "equity", "option", "crypto"
    exchange: str
    last_verified: float  # epoch timestamp


class SymbolRegistryCache:
    """Local symbol cache with TTL and broker API fallback."""

    def __init__(self, broker_client, ttl_seconds: int = 86400):
        self._cache: dict[str, SymbolInfo] = {}
        self._broker = broker_client
        self._ttl = ttl_seconds

    def is_valid_and_tradeable(self, symbol: str) -> tuple[bool, str]:
        info = self._cache.get(symbol)
        now = time.time()

        # Cache hit and still fresh
        if info and (now - info.last_verified) < self._ttl:
            if not info.tradeable:
                return False, f"{symbol} exists but is not tradeable"
            return True, ""

        # Cache miss or stale -- call broker API
        try:
            asset = self._broker.get_asset(symbol)
        except Exception:
            return False, f"{symbol} not found in broker or cache"

        info = SymbolInfo(
            symbol=asset.symbol,
            tradeable=asset.tradeable,
            asset_class=asset.asset_class,
            exchange=asset.exchange,
            last_verified=now,
        )
        self._cache[symbol] = info

        if not info.tradeable:
            return False, f"{symbol} exists but is not tradeable"
        return True, ""

    def refresh_all(self) -> None:
        """Daily full refresh. Call from a scheduled job."""
        assets = self._broker.list_assets(status="active")
        now = time.time()
        for asset in assets:
            self._cache[asset.symbol] = SymbolInfo(
                symbol=asset.symbol,
                tradeable=asset.tradeable,
                asset_class=asset.asset_class,
                exchange=asset.exchange,
                last_verified=now,
            )
```

## Market Hours Engine

Your bot must know when the market is open. This is not optional. Sending orders while the market is closed wastes rate limits and creates confusing error states.

```python
from datetime import datetime, time as dtime
from zoneinfo import ZoneInfo

ET = ZoneInfo("America/New_York")

# Half-days: market closes at 1:00 PM ET
HALF_DAYS_2026 = {
    "2026-11-27",  # Day after Thanksgiving
    "2026-12-24",  # Christmas Eve
}

HOLIDAYS_2026 = {
    "2026-01-01", "2026-01-19", "2026-02-16", "2026-04-03",
    "2026-05-25", "2026-07-03", "2026-09-07", "2026-11-26",
    "2026-12-25",
}


class MarketHoursChecker:
    """Determines whether the US equity market is currently open."""

    REGULAR_OPEN = dtime(9, 30)
    REGULAR_CLOSE = dtime(16, 0)
    HALF_DAY_CLOSE = dtime(13, 0)
    EXTENDED_PRE_OPEN = dtime(4, 0)
    EXTENDED_POST_CLOSE = dtime(20, 0)

    def __init__(
        self,
        holidays: set[str] = HOLIDAYS_2026,
        half_days: set[str] = HALF_DAYS_2026,
    ):
        self._holidays = holidays
        self._half_days = half_days

    def is_market_open(self, allow_extended: bool = False) -> tuple[bool, str]:
        now = datetime.now(ET)
        date_str = now.strftime("%Y-%m-%d")

        # Weekend check
        if now.weekday() >= 5:
            return False, "market closed: weekend"

        # Holiday check
        if date_str in self._holidays:
            return False, f"market closed: holiday ({date_str})"

        current_time = now.time()

        # Half-day check
        if date_str in self._half_days:
            close = self.HALF_DAY_CLOSE
        else:
            close = self.REGULAR_CLOSE

        # Regular hours
        if self.REGULAR_OPEN <= current_time <= close:
            return True, "regular hours"

        # Extended hours (if allowed)
        if allow_extended:
            if self.EXTENDED_PRE_OPEN <= current_time < self.REGULAR_OPEN:
                return True, "pre-market extended hours"
            if close < current_time <= self.EXTENDED_POST_CLOSE:
                return True, "post-market extended hours"

        return False, f"market closed: outside hours (current={current_time})"
```

## Liquidity Score

Thin liquidity means terrible fills. Gate orders on spread and volume before they reach the broker.

```python
from dataclasses import dataclass


@dataclass
class LiquidityScore:
    spread_pct: float   # (ask - bid) / mid * 100
    volume: int         # shares traded today
    depth: float        # average size at top of book (bid + ask qty) / 2

    @staticmethod
    def from_quote(bid: float, ask: float, volume: int, bid_size: int, ask_size: int):
        if bid <= 0 or ask <= 0:
            return LiquidityScore(spread_pct=999.0, volume=0, depth=0)
        mid = (bid + ask) / 2
        spread_pct = ((ask - bid) / mid) * 100
        depth = (bid_size + ask_size) / 2
        return LiquidityScore(spread_pct=spread_pct, volume=volume, depth=depth)


def check_liquidity(
    score: LiquidityScore,
    max_spread_pct: float = 0.5,
    min_volume: int = 10_000,
) -> tuple[bool, str]:
    if score.spread_pct > max_spread_pct:
        return False, (
            f"spread {score.spread_pct:.2f}% exceeds max {max_spread_pct}%"
        )
    if score.volume < min_volume:
        return False, (
            f"volume {score.volume} below minimum {min_volume}"
        )
    return True, ""
```

## Full PreTradeValidator

This class orchestrates all six checks and returns a single `ValidationResult`.

```python
class PreTradeValidator:
    """Orchestrates all pre-trade checks. Must pass before order reaches broker."""

    def __init__(
        self,
        symbol_cache: SymbolRegistryCache,
        market_hours: MarketHoursChecker,
        broker_client,
        max_spread_pct: float = 0.5,
        min_volume: int = 10_000,
        max_quote_age_seconds: float = 30.0,
        max_price_deviation_pct: float = 10.0,
    ):
        self._symbols = symbol_cache
        self._market = market_hours
        self._broker = broker_client
        self._max_spread_pct = max_spread_pct
        self._min_volume = min_volume
        self._max_quote_age = max_quote_age_seconds
        self._max_deviation = max_price_deviation_pct
        self._pending_orders: dict[str, str] = {}  # key: "AAPL:buy"

    def validate(self, symbol: str, side: str, qty: float, limit_price: float | None = None, allow_extended: bool = False) -> ValidationResult:
        result = ValidationResult(passed=True)

        # 1. Symbol valid and tradeable
        valid, reason = self._symbols.is_valid_and_tradeable(symbol)
        if not valid:
            result.add_failure(ValidationFailure.SYMBOL_NOT_FOUND, reason)
            return result  # short-circuit: no point checking further

        # 2. Market open
        open_, reason = self._market.is_market_open(allow_extended=allow_extended)
        if not open_:
            result.add_failure(ValidationFailure.MARKET_CLOSED, reason)

        # 3. Liquidity check
        quote = self._broker.get_latest_quote(symbol)
        score = LiquidityScore.from_quote(
            bid=quote.bid, ask=quote.ask,
            volume=quote.volume,
            bid_size=quote.bid_size, ask_size=quote.ask_size,
        )
        liq_ok, liq_reason = check_liquidity(score, self._max_spread_pct, self._min_volume)
        if not liq_ok:
            result.add_failure(ValidationFailure.INSUFFICIENT_LIQUIDITY, liq_reason)

        # 4. Price sanity
        if quote.bid <= 0 or quote.ask <= 0:
            result.add_failure(ValidationFailure.ZERO_PRICE, f"bid={quote.bid} ask={quote.ask}")
        if hasattr(quote, "timestamp"):
            age = time.time() - quote.timestamp
            if age > self._max_quote_age:
                result.add_failure(ValidationFailure.STALE_QUOTE, f"quote age {age:.1f}s > {self._max_quote_age}s")
        if limit_price is not None and quote.bid > 0:
            mid = (quote.bid + quote.ask) / 2
            deviation = abs(limit_price - mid) / mid * 100
            if deviation > self._max_deviation:
                result.add_failure(ValidationFailure.PRICE_OUT_OF_RANGE, f"limit {limit_price} deviates {deviation:.1f}% from mid {mid:.2f}")

        # 5. Duplicate pending check
        key = f"{symbol}:{side}"
        if key in self._pending_orders:
            result.add_failure(ValidationFailure.DUPLICATE_PENDING, f"order already pending for {key}")

        # 6. Buying power check
        account = self._broker.get_account()
        est_cost = (limit_price or quote.ask) * qty
        if est_cost > float(account.buying_power):
            result.add_failure(
                ValidationFailure.INSUFFICIENT_BUYING_POWER,
                f"need ${est_cost:.2f}, have ${float(account.buying_power):.2f}",
            )

        return result
```

## BAD vs GOOD

```python
# --- BAD: Rely on broker rejection as validation ---
def submit_order_bad(symbol, qty, side):
    try:
        broker.submit_order(symbol=symbol, qty=qty, side=side)
    except BrokerError as e:
        logger.error(f"Broker rejected: {e}")  # This is NOT validation

# --- GOOD: Proactive validation before submission ---
def submit_order_good(symbol, qty, side, limit_price=None):
    result = pre_trade_validator.validate(symbol, side, qty, limit_price)
    if not result.passed:
        audit_log.log("order_rejected_pretrade", symbol=symbol, failures=result.failures)
        return None

    # Only now submit to broker
    order = broker.submit_order(symbol=symbol, qty=qty, side=side, limit_price=limit_price)
    audit_log.log("order_submitted", symbol=symbol, order_id=order.id)
    return order
```

## Red Flags

| Red Flag | Why It Matters |
|---|---|
| Relying on broker to reject invalid orders | Burns rate limits, inconsistent errors, not a safety net |
| No market hours check before submission | Orders queue or reject unpredictably outside hours |
| No liquidity check | Wide spreads cause terrible fills, especially on options |
| No symbol validation / no symbol cache | Typos and delisted symbols reach the broker |
| No stale quote detection | Acting on minutes-old data leads to bad fills |
| No buying power check | Cash account violations trigger PDT or margin calls |
| Validation only on some order paths | Every path to the broker must go through the gate |

## Integration

- **Runs BEFORE** `order-execution-integrity` -- dedup and execution logic assume the order is already validated
- **Runs BEFORE** `risk-management-gates` -- risk gates check position sizing and exposure; pre-trade checks symbol/market/liquidity fundamentals
- **Uses** `trading-bot-architecture` Intent Bus -- validation is a stage in the intent pipeline
- **Feeds** `trade-audit-and-replay` -- every validation result (pass or fail) is logged to the audit trail
