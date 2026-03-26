---
name: market-data-pipeline
description: Use when implementing market data feeds, price subscriptions, indicator computation, or when encountering stale prices, missing data, or sentinel values being treated as real data
---

# Market Data Pipeline

## Iron Law

**EVERY PRICE CHECK MUST VERIFY FRESHNESS. SENTINEL VALUES MUST USE A DEDICATED TYPE, NOT MAGIC NUMBERS.**

Violations of this law have caused the single worst class of bugs in trading bot history.
If you remember nothing else: data has TWO components -- the value AND when it was observed.
A price without a timestamp is not a price. It is a landmine.

## Prevents

- **VIX sentinel misuse (#10):** Magic number 99.0 treated as real VIX reading, triggering
  aggressive risk tightening across all positions simultaneously.
- **Stale data (#5):** Cached prices from minutes or hours ago used for live trading decisions,
  leading to fills at wildly different prices than expected.

---

## Freshness Checks

Every data point must carry `(value, timestamp)`. Before ANY use, verify:

```
timestamp > now - max_staleness
```

If stale, **FAIL-CLOSED**. Do not use the data. Do not fall back to the stale value.
Do not log a warning and continue. STOP.

### Python Implementation

```python
from dataclasses import dataclass, field
from datetime import datetime, timedelta, timezone
from typing import Optional, Union
from enum import Enum


class DataStatus(Enum):
    LIVE = "live"
    STALE = "stale"
    UNAVAILABLE = "unavailable"


@dataclass(frozen=True)
class MarketData:
    """Immutable market data point with mandatory freshness metadata."""
    symbol: str
    price: float
    timestamp: datetime
    source: str
    status: DataStatus = DataStatus.LIVE

    def is_fresh(self, max_staleness: timedelta) -> bool:
        """Check if this data point is fresh enough to use."""
        age = datetime.now(timezone.utc) - self.timestamp
        return age <= max_staleness and self.status == DataStatus.LIVE

    @property
    def age_seconds(self) -> float:
        return (datetime.now(timezone.utc) - self.timestamp).total_seconds()


@dataclass(frozen=True)
class DataUnavailable:
    """Explicit representation of missing data. NOT a magic number."""
    symbol: str
    reason: str
    since: datetime
    last_known_value: Optional[float] = None
    last_known_timestamp: Optional[datetime] = None


# Tagged union: every consumer must handle both cases
MarketDataResult = Union[MarketData, DataUnavailable]


def get_price(symbol: str, max_staleness: timedelta) -> MarketDataResult:
    """
    Fetch price with mandatory freshness check.

    Returns MarketData if fresh, DataUnavailable if stale or missing.
    NEVER returns a magic number. NEVER returns stale data silently.
    """
    raw = _fetch_from_primary(symbol)

    if raw is None:
        raw = _fetch_from_backup(symbol)

    if raw is None:
        return DataUnavailable(
            symbol=symbol,
            reason="All data sources failed",
            since=datetime.now(timezone.utc),
        )

    if not raw.is_fresh(max_staleness):
        return DataUnavailable(
            symbol=symbol,
            reason=f"Data stale: age={raw.age_seconds:.1f}s, max={max_staleness.total_seconds():.1f}s",
            since=datetime.now(timezone.utc),
            last_known_value=raw.price,
            last_known_timestamp=raw.timestamp,
        )

    return raw
```

### Using MarketDataResult Safely

```python
def should_enter_trade(symbol: str) -> bool:
    result = get_price(symbol, max_staleness=timedelta(seconds=5))

    # MUST handle both branches. No shortcuts.
    match result:
        case MarketData() as data:
            return evaluate_entry_signal(data)
        case DataUnavailable() as unavail:
            log.warning(f"Cannot evaluate {symbol}: {unavail.reason}")
            return False  # FAIL-CLOSED
```

---

## The Emabot VIX Disaster (Case Study)

### What Happened

Emabot used VIX (volatility index) to gate position sizing and risk. When the VIX data
feed went down, the system used a sentinel value of `99.0` to mean "data unavailable."

The problem: downstream code did not distinguish between `VIX=99.0` (sentinel) and
`VIX=99` (real reading). A VIX of 99 is apocalyptic -- it means extreme market fear.
The risk module responded by:

1. Tightening all trailing stops to minimum distance
2. Reducing all position sizes to minimum
3. Blocking all new entries
4. Triggering stop-losses on existing positions due to tight trailing stops

The result: every position was stopped out during a period of normal market conditions
because a data feed went down. The sentinel value was indistinguishable from a real
VIX spike.

### Root Cause

```python
# BAD: The code that caused the disaster
VIX_UNAVAILABLE = 99.0  # Magic number sentinel

def get_vix() -> float:
    try:
        return fetch_vix_from_cboe()
    except Exception:
        return VIX_UNAVAILABLE  # Returns 99.0

def calculate_risk_multiplier(vix: float) -> float:
    # This code has NO IDEA if vix=99.0 is real or sentinel
    if vix > 30:
        return 0.5  # Reduce risk
    if vix > 50:
        return 0.25
    if vix > 80:
        return 0.1  # Minimum risk -- triggered by sentinel!
    return 1.0
```

### The Fix

```python
# GOOD: Sentinel replaced with typed result
def get_vix() -> MarketDataResult:
    try:
        value = fetch_vix_from_cboe()
        return MarketData(
            symbol="VIX",
            price=value,
            timestamp=datetime.now(timezone.utc),
            source="cboe",
        )
    except Exception as e:
        return DataUnavailable(
            symbol="VIX",
            reason=f"CBOE fetch failed: {e}",
            since=datetime.now(timezone.utc),
        )

def calculate_risk_multiplier(vix_result: MarketDataResult) -> Optional[float]:
    match vix_result:
        case DataUnavailable():
            return None  # Caller must handle -- fail-closed
        case MarketData(price=vix):
            if vix > 80:
                return 0.1
            if vix > 50:
                return 0.25
            if vix > 30:
                return 0.5
            return 1.0
```

---

## VIX Gate Pattern

A complete VIX gate checks three things in order. ANY failure halts entries.

```python
@dataclass
class VIXGate:
    """Three-check VIX gate. All must pass or entries are blocked."""
    max_staleness: timedelta = field(default_factory=lambda: timedelta(seconds=30))
    min_vix: float = 9.0    # Below this is suspicious
    max_vix: float = 80.0   # Above this, halt entries regardless

    def check(self, vix_result: MarketDataResult) -> tuple[bool, str]:
        # Check 1: Is data available?
        if isinstance(vix_result, DataUnavailable):
            return False, f"VIX unavailable: {vix_result.reason}"

        # Check 2: Is it fresh?
        if not vix_result.is_fresh(self.max_staleness):
            return False, f"VIX stale: age={vix_result.age_seconds:.1f}s"

        # Check 3: Is it within plausible range?
        if not (self.min_vix <= vix_result.price <= self.max_vix):
            return False, (
                f"VIX out of range: {vix_result.price} "
                f"(expected {self.min_vix}-{self.max_vix})"
            )

        return True, "VIX gate passed"
```

---

## Data Source Failover

### Architecture

```
Primary (WebSocket) --> Parser --> Freshness Check --> Consumer
    |                                   |
    v                                   v
Backup (REST Poll)               FAIL-CLOSED if stale
    |
    v
Both fail? --> HALT TRADING
```

### Implementation

```python
class MarketDataPipeline:
    """Multi-source market data with automatic failover and freshness enforcement."""

    def __init__(
        self,
        primary_source: DataSource,
        backup_source: DataSource,
        max_staleness: timedelta = timedelta(seconds=5),
    ):
        self.primary = primary_source
        self.backup = backup_source
        self.max_staleness = max_staleness
        self._last_failover: Optional[datetime] = None

    async def get(self, symbol: str) -> MarketDataResult:
        # Try primary
        result = await self.primary.fetch(symbol)
        if isinstance(result, MarketData) and result.is_fresh(self.max_staleness):
            return result

        # Primary failed or stale -- try backup
        log.warning(f"Primary failed for {symbol}, trying backup")
        result = await self.backup.fetch(symbol)
        if isinstance(result, MarketData) and result.is_fresh(self.max_staleness):
            self._last_failover = datetime.now(timezone.utc)
            return result

        # Both failed -- HALT (fail-closed)
        log.critical(f"ALL data sources failed for {symbol}")
        await self._trigger_trading_halt(symbol)
        return DataUnavailable(
            symbol=symbol,
            reason="All sources failed -- trading halted",
            since=datetime.now(timezone.utc),
        )

    async def _trigger_trading_halt(self, symbol: str) -> None:
        """Halt trading when all data sources fail. This is NOT optional."""
        await alert_critical(
            f"TRADING HALT: No fresh data for {symbol}. "
            f"Primary and backup both failed."
        )
```

---

## WebSocket vs Polling

Prefer WebSocket for primary feed. ALWAYS have a polling fallback for gap detection.

```python
class WebSocketWithPollingFallback:
    """
    WebSocket for speed, polling for gap detection.

    The polling loop runs ALONGSIDE the websocket, not instead of it.
    It detects if the websocket silently dropped messages.
    """

    def __init__(self, ws_url: str, rest_url: str, gap_threshold: timedelta):
        self.ws_url = ws_url
        self.rest_url = rest_url
        self.gap_threshold = gap_threshold
        self._last_ws_message: dict[str, datetime] = {}

    async def _poll_loop(self):
        """Runs continuously. Detects gaps in websocket feed."""
        while True:
            await asyncio.sleep(self.gap_threshold.total_seconds())
            for symbol, last_time in self._last_ws_message.items():
                age = datetime.now(timezone.utc) - last_time
                if age > self.gap_threshold:
                    log.warning(
                        f"WebSocket gap detected for {symbol}: "
                        f"no message in {age.total_seconds():.1f}s"
                    )
                    # Fetch via REST to fill gap
                    await self._fetch_rest_fallback(symbol)

    async def _on_ws_message(self, symbol: str, data: dict):
        """Process websocket message and update last-seen time."""
        self._last_ws_message[symbol] = datetime.now(timezone.utc)
        await self._process_update(symbol, data)
```

---

## Red Flags

Stop and fix immediately if you see ANY of these:

- **Prices without timestamps.** A price without a timestamp is not a price.
- **Magic number sentinels.** `VIX_UNAVAILABLE = 99.0`, `PRICE_NA = -1.0`, `NO_DATA = 0.0`.
  Use `Optional`, `None`, or a tagged union. Always.
- **No freshness check before use.** Every `get_price()` call must verify staleness.
- **Single data source.** If it goes down, you are blind and still trading.
- **Cached data without TTL.** A cache without expiry is a stale data generator.
- **`except Exception: return default_value`.** This silences failures and returns garbage.
- **Comparing floats with `==`.** Prices are floats. Use tolerance-based comparison.

### Detecting Red Flags in Code Review

```python
# RED FLAG: No timestamp
price = get_latest_price("AAPL")  # What if this is 5 minutes old?

# RED FLAG: Magic number sentinel
if vix_value == 99.0:
    print("VIX unavailable")

# RED FLAG: Silent failure with default
def get_price(symbol):
    try:
        return broker.quote(symbol)
    except:
        return 0.0  # This WILL be used as a real price downstream

# GREEN: Correct pattern
result = get_price("AAPL", max_staleness=timedelta(seconds=5))
match result:
    case MarketData() as data:
        process(data)
    case DataUnavailable() as unavail:
        halt_and_alert(unavail)
```

---

## Testing Market Data Pipelines

```python
import pytest
from unittest.mock import AsyncMock
from datetime import datetime, timedelta, timezone


class TestFreshnessCheck:
    def test_fresh_data_accepted(self):
        data = MarketData(
            symbol="AAPL",
            price=150.0,
            timestamp=datetime.now(timezone.utc),
            source="test",
        )
        assert data.is_fresh(timedelta(seconds=5))

    def test_stale_data_rejected(self):
        data = MarketData(
            symbol="AAPL",
            price=150.0,
            timestamp=datetime.now(timezone.utc) - timedelta(seconds=30),
            source="test",
        )
        assert not data.is_fresh(timedelta(seconds=5))

    def test_sentinel_value_never_created(self):
        """Ensure we never accidentally create sentinel values."""
        result = get_vix_when_feed_down()
        assert isinstance(result, DataUnavailable)
        # NOT a MarketData with price=99.0


class TestFailover:
    @pytest.mark.asyncio
    async def test_primary_fails_uses_backup(self):
        pipeline = MarketDataPipeline(
            primary_source=FailingSource(),
            backup_source=MockSource(price=150.0),
            max_staleness=timedelta(seconds=5),
        )
        result = await pipeline.get("AAPL")
        assert isinstance(result, MarketData)
        assert result.price == 150.0

    @pytest.mark.asyncio
    async def test_both_fail_halts_trading(self):
        pipeline = MarketDataPipeline(
            primary_source=FailingSource(),
            backup_source=FailingSource(),
            max_staleness=timedelta(seconds=5),
        )
        result = await pipeline.get("AAPL")
        assert isinstance(result, DataUnavailable)
        assert "halted" in result.reason.lower()


class TestVIXGate:
    def test_unavailable_vix_blocks_entries(self):
        gate = VIXGate()
        unavail = DataUnavailable(
            symbol="VIX", reason="feed down", since=datetime.now(timezone.utc)
        )
        passed, reason = gate.check(unavail)
        assert not passed

    def test_stale_vix_blocks_entries(self):
        gate = VIXGate()
        stale = MarketData(
            symbol="VIX",
            price=20.0,
            timestamp=datetime.now(timezone.utc) - timedelta(minutes=5),
            source="cboe",
        )
        passed, reason = gate.check(stale)
        assert not passed

    def test_out_of_range_vix_blocks_entries(self):
        gate = VIXGate()
        extreme = MarketData(
            symbol="VIX",
            price=99.0,  # This is the old sentinel value!
            timestamp=datetime.now(timezone.utc),
            source="cboe",
        )
        passed, reason = gate.check(extreme)
        assert not passed  # 99 > max_vix of 80
```

---

## Integration

- **trading-bot-skills:strategy-signal-validation** -- Signals consume market data.
  Every signal must receive `MarketDataResult`, not raw floats.
- **trading-bot-skills:trading-monitoring-and-alerts** -- Data freshness is a core
  health metric. Alert on staleness before it causes bad trades.
