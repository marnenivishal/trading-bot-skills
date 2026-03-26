---
name: strategy-signal-validation
description: Use when implementing trading signals, EMA crossovers, indicator-based entries or exits, or when signals fire too frequently, flip immediately, or generate false entries
---

# Strategy Signal Validation

## Iron Law

**A SIGNAL IS NOT CONFIRMED UNTIL IT HAS PERSISTED FOR N BARS, AGREES ACROSS TIMEFRAMES, EXCEEDS NOISE THRESHOLD, AND IS NOT CONTRADICTED BY HIGHER TIMEFRAME TREND.**

A single bar crossing a line is not a signal. It is noise that looks like a signal.

## Prevents

- **Cloud flip false exits (#9):** EMA cloud flips direction minutes after entry,
  triggering an immediate exit at a loss.
- **EMA curl unreliable (#15):** Curl detection too sensitive on short lookback,
  generating constant false signals in choppy markets.

---

## Signal Confirmation Checklist

Before acting on ANY signal, verify ALL of these:

1. **Persistence:** Signal has persisted for N consecutive bars (not just one crossing)
2. **Timeframe agreement:** Signal direction agrees across at least 2 timeframes
3. **Magnitude:** Signal magnitude exceeds noise threshold (typically ATR-normalized)
4. **No contradiction:** Higher timeframe trend does not contradict the signal
5. **Cooldown clear:** Sufficient time has passed since last signal in same direction

If ANY check fails, the signal is rejected. No exceptions. No "just this once."

```python
from dataclasses import dataclass, field
from datetime import datetime, timedelta, timezone
from typing import Optional
from enum import Enum


class SignalDirection(Enum):
    LONG = "long"
    SHORT = "short"
    FLAT = "flat"


class SignalStatus(Enum):
    PENDING = "pending"        # Detected, not yet confirmed
    CONFIRMED = "confirmed"    # All checks passed
    REJECTED = "rejected"      # Failed one or more checks
    COOLDOWN = "cooldown"      # Too soon after last signal


@dataclass
class Signal:
    symbol: str
    direction: SignalDirection
    strategy: str
    strength: float              # ATR-normalized magnitude
    timeframe: str
    bar_count: int = 0           # How many bars this signal has persisted
    first_seen: Optional[datetime] = None
    status: SignalStatus = SignalStatus.PENDING

    @property
    def key(self) -> tuple:
        """Unique key for dedup: (symbol, direction, strategy)."""
        return (self.symbol, self.direction, self.strategy)


@dataclass
class SignalConfirmationGate:
    """
    Multi-check signal confirmation gate.

    ALL checks must pass. Any failure -> signal rejected.
    """
    min_persistence_bars: int = 3
    min_strength_atr_multiple: float = 0.5
    cooldown_bars: int = 10
    require_higher_tf_agreement: bool = True

    # Cooldown tracking: keyed by (symbol, direction, strategy)
    _last_signal_bar: dict[tuple, int] = field(default_factory=dict)

    def evaluate(
        self,
        signal: Signal,
        current_bar: int,
        higher_tf_direction: SignalDirection,
    ) -> SignalStatus:
        """
        Evaluate a signal against all confirmation checks.

        Returns CONFIRMED only if ALL checks pass.
        """
        checks = [
            self._check_persistence(signal),
            self._check_magnitude(signal),
            self._check_cooldown(signal, current_bar),
            self._check_higher_tf(signal, higher_tf_direction),
        ]

        failures = [reason for passed, reason in checks if not passed]

        if failures:
            signal.status = SignalStatus.REJECTED
            log.info(
                f"Signal REJECTED for {signal.symbol} {signal.direction.value}: "
                f"{'; '.join(failures)}"
            )
            return SignalStatus.REJECTED

        # All checks passed
        signal.status = SignalStatus.CONFIRMED
        self._last_signal_bar[signal.key] = current_bar
        return SignalStatus.CONFIRMED

    def _check_persistence(self, signal: Signal) -> tuple[bool, str]:
        if signal.bar_count < self.min_persistence_bars:
            return False, (
                f"Persistence: {signal.bar_count}/{self.min_persistence_bars} bars"
            )
        return True, "Persistence OK"

    def _check_magnitude(self, signal: Signal) -> tuple[bool, str]:
        if signal.strength < self.min_strength_atr_multiple:
            return False, (
                f"Magnitude: {signal.strength:.2f} < "
                f"{self.min_strength_atr_multiple:.2f} ATR"
            )
        return True, "Magnitude OK"

    def _check_cooldown(
        self, signal: Signal, current_bar: int
    ) -> tuple[bool, str]:
        last = self._last_signal_bar.get(signal.key)
        if last is not None and (current_bar - last) < self.cooldown_bars:
            bars_remaining = self.cooldown_bars - (current_bar - last)
            return False, f"Cooldown: {bars_remaining} bars remaining"
        return True, "Cooldown OK"

    def _check_higher_tf(
        self, signal: Signal, higher_tf_direction: SignalDirection
    ) -> tuple[bool, str]:
        if not self.require_higher_tf_agreement:
            return True, "Higher TF check disabled"
        if higher_tf_direction == SignalDirection.FLAT:
            return True, "Higher TF flat (neutral)"
        if signal.direction != higher_tf_direction:
            return False, (
                f"Higher TF contradiction: signal={signal.direction.value}, "
                f"higher_tf={higher_tf_direction.value}"
            )
        return True, "Higher TF agrees"
```

---

## The EMA Cloud Flip Problem (Emabot Case Study)

### What Happened

Emabot used a 15-minute EMA cloud (fast EMA vs slow EMA) to determine trend direction
and generate entry signals. The system would enter a long position when the fast EMA
crossed above the slow EMA.

The problem: **5 minutes after entering long, the EMAs flipped back.** The exit logic
saw the cloud flip as a reversal signal and immediately closed the position at a loss.

This happened repeatedly in choppy, range-bound markets. The bot would:

1. Detect bullish EMA crossover -> Enter long
2. 1-3 bars later, EMAs flip back -> Exit at loss
3. 1-3 bars later, EMAs flip bullish again -> Enter long again
4. Repeat, accumulating small losses that added up to significant drawdown

### Root Cause

```python
# BAD: Emabot's original logic -- no confirmation, instant reaction
def check_exit_signal(self, fast_ema: float, slow_ema: float) -> bool:
    """Exit if cloud flips. No confirmation. Instant."""
    if self.position == "long" and fast_ema < slow_ema:
        return True  # EXIT IMMEDIATELY on single-bar flip
    return False
```

The exit had ZERO confirmation bars. A single bar where `fast_ema < slow_ema` by
0.001 would trigger a full position exit. In choppy markets where EMAs oscillate
around each other, this generated constant false exits.

### The Fix: Confirmation Bars + Minimum Spread

```python
@dataclass
class EMACloudExitSignal:
    """
    EMA cloud exit with confirmation bars and minimum spread.

    Requires the cloud to flip AND stay flipped for N bars AND the spread
    between EMAs must exceed a noise threshold.
    """
    confirmation_bars: int = 3
    min_spread_atr_multiple: float = 0.3
    _flip_count: dict[str, int] = field(default_factory=dict)

    def check_exit(
        self,
        symbol: str,
        fast_ema: float,
        slow_ema: float,
        atr: float,
        position_direction: str,
    ) -> tuple[bool, str]:
        is_flipped = (
            (position_direction == "long" and fast_ema < slow_ema) or
            (position_direction == "short" and fast_ema > slow_ema)
        )

        if not is_flipped:
            # Reset counter -- cloud is back in our favor
            self._flip_count[symbol] = 0
            return False, "Cloud not flipped"

        # Cloud is flipped -- increment counter
        self._flip_count[symbol] = self._flip_count.get(symbol, 0) + 1
        bars_flipped = self._flip_count[symbol]

        # Check 1: Persistence
        if bars_flipped < self.confirmation_bars:
            return False, (
                f"Cloud flipped but only {bars_flipped}/{self.confirmation_bars} bars"
            )

        # Check 2: Magnitude -- spread must exceed noise
        spread = abs(fast_ema - slow_ema)
        spread_atr = spread / atr if atr > 0 else 0
        if spread_atr < self.min_spread_atr_multiple:
            return False, (
                f"Cloud flipped {bars_flipped} bars but spread "
                f"({spread_atr:.2f} ATR) below threshold "
                f"({self.min_spread_atr_multiple:.2f} ATR)"
            )

        return True, f"CONFIRMED EXIT: flipped {bars_flipped} bars, spread {spread_atr:.2f} ATR"
```

---

## EMA Curl Detection Fix (Emabot Case Study)

### What Happened

Emabot used "EMA curl" -- the rate of change of an EMA -- to detect early momentum
shifts. The implementation used a 5-bar lookback on the 5-minute timeframe.

The problem: 5 bars on 5-minute = 25 minutes of data. This is far too short.
Normal price noise created constant small curls that the detector flagged as signals.
The bot entered and exited dozens of times per day based on meaningless noise.

### Root Cause

```python
# BAD: Emabot's original curl detection
def detect_curl(ema_values: list[float], lookback: int = 5) -> float:
    """Raw difference over lookback. No normalization. Too sensitive."""
    return ema_values[-1] - ema_values[-lookback]
```

A raw difference is meaningless without context. Is a curl of 0.5 significant?
It depends on the instrument. For SPY, 0.5 is noise. For a penny stock, 0.5 is huge.

### The Fix: ATR-Normalized Curl with Threshold

```python
@dataclass
class EMACurlDetector:
    """
    ATR-normalized EMA curl detection.

    The curl value is normalized by ATR so the threshold is consistent
    across instruments and volatility regimes.
    """
    lookback: int = 12            # Bars to measure curl over
    atr_threshold: float = 0.15   # Minimum curl in ATR units
    smoothing: int = 3            # Smooth curl to reduce noise

    def compute_curl(
        self,
        ema_values: list[float],
        atr: float,
    ) -> tuple[float, bool]:
        """
        Compute ATR-normalized EMA curl.

        Returns (curl_value, is_significant).
        """
        if len(ema_values) < self.lookback + self.smoothing:
            return 0.0, False

        if atr <= 0:
            return 0.0, False

        # Compute raw curl values over the smoothing window
        curls = []
        for i in range(-self.smoothing, 0):
            idx = len(ema_values) + i
            raw_curl = ema_values[idx] - ema_values[idx - self.lookback]
            normalized_curl = raw_curl / atr
            curls.append(normalized_curl)

        # Smoothed curl (average of recent curl values)
        smoothed_curl = sum(curls) / len(curls)

        is_significant = abs(smoothed_curl) > self.atr_threshold

        return smoothed_curl, is_significant
```

---

## Signal Cooldown: Unified Dedup Gate

### Critical Design Decision

Signal cooldown must be per `(symbol, direction, strategy)`, NOT per function instance.

```python
# BAD: Per-instance cooldown -- each function has its own dict
class EMAStrategy:
    def __init__(self):
        self._cooldowns = {}  # Only THIS instance tracks cooldowns

class RSIStrategy:
    def __init__(self):
        self._cooldowns = {}  # Different dict! No coordination!

# If EMAStrategy fires LONG on AAPL, RSIStrategy has no idea
# and can fire its own LONG on AAPL immediately.
```

```python
# GOOD: Unified cooldown gate shared across all strategies
class SignalDeduplicationGate:
    """
    Centralized signal dedup. ALL strategies must pass through this gate.

    Keyed by (symbol, direction, strategy) so:
    - Same strategy cannot re-fire same signal too quickly
    - Different strategies CAN fire independently (intended)
    - Same strategy on different symbols are independent (intended)
    """

    def __init__(self, default_cooldown_bars: int = 10):
        self._last_fired: dict[tuple, int] = {}
        self._default_cooldown = default_cooldown_bars

    def can_fire(
        self,
        symbol: str,
        direction: SignalDirection,
        strategy: str,
        current_bar: int,
        cooldown_bars: Optional[int] = None,
    ) -> bool:
        key = (symbol, direction.value, strategy)
        cooldown = cooldown_bars or self._default_cooldown
        last = self._last_fired.get(key)

        if last is not None and (current_bar - last) < cooldown:
            return False
        return True

    def record_fire(
        self,
        symbol: str,
        direction: SignalDirection,
        strategy: str,
        current_bar: int,
    ) -> None:
        key = (symbol, direction.value, strategy)
        self._last_fired[key] = current_bar


# Singleton -- ALL strategies use this ONE instance
signal_gate = SignalDeduplicationGate(default_cooldown_bars=10)
```

---

## Anti-Whipsaw Patterns

These patterns reduce false signals in choppy, range-bound markets.

### 1. EMA Spread Filter

```python
def ema_spread_filter(
    fast_ema: float, slow_ema: float, atr: float, min_spread_atr: float = 0.5
) -> bool:
    """Only act on EMA crossover if spread exceeds noise threshold."""
    spread = abs(fast_ema - slow_ema)
    return (spread / atr) >= min_spread_atr if atr > 0 else False
```

### 2. Heikin-Ashi Confirmation

```python
def heikin_ashi_confirms(
    ha_open: float, ha_close: float, direction: SignalDirection
) -> bool:
    """
    Heikin-Ashi candle must agree with signal direction.
    HA smooths price action, reducing whipsaw.
    """
    if direction == SignalDirection.LONG:
        return ha_close > ha_open  # Bullish HA candle
    elif direction == SignalDirection.SHORT:
        return ha_close < ha_open  # Bearish HA candle
    return False
```

### 3. Volume Confirmation

```python
def volume_confirms(
    current_volume: float,
    avg_volume: float,
    min_ratio: float = 1.2,
) -> bool:
    """Signal should be accompanied by above-average volume."""
    if avg_volume <= 0:
        return False
    return (current_volume / avg_volume) >= min_ratio
```

### 4. Higher Timeframe Trend Filter

```python
def higher_tf_filter(
    signal_direction: SignalDirection,
    daily_ema_fast: float,
    daily_ema_slow: float,
) -> bool:
    """
    Only take signals that agree with the daily trend.
    Long signals only when daily trend is bullish, and vice versa.
    """
    daily_bullish = daily_ema_fast > daily_ema_slow
    if signal_direction == SignalDirection.LONG:
        return daily_bullish
    elif signal_direction == SignalDirection.SHORT:
        return not daily_bullish
    return True
```

### Combining Anti-Whipsaw Filters

```python
def full_signal_validation(
    signal: Signal,
    fast_ema: float,
    slow_ema: float,
    atr: float,
    ha_open: float,
    ha_close: float,
    volume: float,
    avg_volume: float,
    daily_ema_fast: float,
    daily_ema_slow: float,
) -> tuple[bool, list[str]]:
    """
    Run ALL anti-whipsaw filters. ALL must pass.
    Returns (passed, list_of_reasons).
    """
    checks = {
        "EMA spread": ema_spread_filter(fast_ema, slow_ema, atr),
        "Heikin-Ashi": heikin_ashi_confirms(ha_open, ha_close, signal.direction),
        "Volume": volume_confirms(volume, avg_volume),
        "Higher TF": higher_tf_filter(signal.direction, daily_ema_fast, daily_ema_slow),
    }

    failures = [name for name, passed in checks.items() if not passed]

    if failures:
        return False, [f"Failed: {f}" for f in failures]
    return True, ["All filters passed"]
```

---

## Signal Confluence and Collinearity Detection

### The Confluence Requirement

Require 3 or more non-correlated indicator FAMILIES to agree before entry. Studies show that 3+ diverse, non-correlated signals achieve approximately 70% success rate versus 40% for single indicators. The key word is **non-correlated** -- using multiple indicators from the same family gives the illusion of confluence without the benefit.

### Analytical Family Classification

| Family | Indicators | What They Measure |
|--------|-----------|-------------------|
| Trend | EMA, SMA, MACD, ADX | Direction |
| Momentum | RSI, Stochastic, CCI, ROC | Speed of price change |
| Volume | OBV, VWAP, AD Line, MFI | Participation |
| Volatility | ATR, Bollinger Bands, Keltner | Range/expansion |
| Sentiment | External alerts, social, news | Crowd opinion |

### The Collinearity Trap

EMA + MACD + Signal Line looks like 3 indicators, but ALL are moving-average derived. This is 1 signal, not 3. Similarly, RSI + Stochastic both measure momentum and are collinear.

The danger: you think you have 3 independent confirmations, but you actually have 1 confirmation counted 3 times. This creates false confidence and oversized positions.

### Collinearity Detection

```python
from enum import Enum

class IndicatorFamily(Enum):
    TREND = "trend"
    MOMENTUM = "momentum"
    VOLUME = "volume"
    VOLATILITY = "volatility"
    SENTIMENT = "sentiment"

INDICATOR_FAMILIES = {
    "ema": IndicatorFamily.TREND, "sma": IndicatorFamily.TREND, "macd": IndicatorFamily.TREND,
    "rsi": IndicatorFamily.MOMENTUM, "stochastic": IndicatorFamily.MOMENTUM, "cci": IndicatorFamily.MOMENTUM,
    "obv": IndicatorFamily.VOLUME, "vwap": IndicatorFamily.VOLUME,
    "atr": IndicatorFamily.VOLATILITY, "bollinger": IndicatorFamily.VOLATILITY,
}

class ConfluenceGate:
    def __init__(self, min_families: int = 3, collinearity_threshold: float = 0.85):
        self.min_families = min_families
        self.collinearity_threshold = collinearity_threshold

    def check(self, signals: list[IndicatorSignal]) -> ConfluenceResult:
        families = set()
        for sig in signals:
            if sig.value_agrees:
                families.add(INDICATOR_FAMILIES.get(sig.name, IndicatorFamily.TREND))
        if len(families) < self.min_families:
            return ConfluenceResult(passed=False, reason=f"Only {len(families)} families agree, need {self.min_families}")
        return ConfluenceResult(passed=True, families_agreeing=families)
```

### Rolling Correlation for Collinearity Detection

Compute rolling Pearson correlation between indicator outputs over the last 50 bars. If correlation exceeds 0.85, those indicators are collinear and count as one signal, regardless of their names.

### Confluence Red Flags

- **"5 indicators all same family = confirmation"** -- it is collinearity, not confirmation.
- **"EMA + MACD + Signal Line = 3 signals"** -- it is 1 signal (all moving-average derived).
- **Confluence threshold below 3 families** -- minimum 3 non-correlated families for meaningful confluence.

### Configuration

```yaml
signal_validation:
  min_confluence_families: 3             # Minimum non-correlated families required
  collinearity_threshold: 0.85           # Pearson correlation above this = collinear
  collinearity_lookback_bars: 50         # Bars to compute rolling correlation over
```

---

## Red Flags

Stop and fix immediately if you see ANY of these:

- **Signal fires entry then exit on the same bar.** This means confirmation is missing.
  A signal that flips within one bar is noise, not a signal.
- **No confirmation bars.** Acting on a single crossover bar is the #1 source of
  false signals in EMA-based strategies.
- **Per-instance cooldown dicts.** If each strategy instance maintains its own cooldown
  dictionary, there is no coordination and the same signal can fire multiple times.
- **Raw (unnormalized) indicator thresholds.** A curl of 0.5 means nothing without
  ATR normalization. Thresholds must be relative to current volatility.
- **No higher timeframe filter.** Trading 5-minute signals against the daily trend
  is fighting the tide. Most losses come from counter-trend signals.
- **Cooldown measured in wall-clock time instead of bars.** Low-volume periods have
  fewer bars per hour. Wall-clock cooldown lets signals fire too frequently in slow
  markets and too rarely in fast markets.

---

## Testing Signal Validation

```python
class TestSignalConfirmation:
    def test_single_bar_signal_rejected(self):
        """A signal that has persisted for only 1 bar must be rejected."""
        gate = SignalConfirmationGate(min_persistence_bars=3)
        signal = Signal(
            symbol="AAPL",
            direction=SignalDirection.LONG,
            strategy="ema_cross",
            strength=1.0,
            timeframe="5min",
            bar_count=1,
        )
        result = gate.evaluate(signal, current_bar=100, higher_tf_direction=SignalDirection.LONG)
        assert result == SignalStatus.REJECTED

    def test_confirmed_after_n_bars(self):
        """Signal persisting for N bars with sufficient strength is confirmed."""
        gate = SignalConfirmationGate(min_persistence_bars=3)
        signal = Signal(
            symbol="AAPL",
            direction=SignalDirection.LONG,
            strategy="ema_cross",
            strength=1.0,
            timeframe="5min",
            bar_count=5,
        )
        result = gate.evaluate(signal, current_bar=100, higher_tf_direction=SignalDirection.LONG)
        assert result == SignalStatus.CONFIRMED

    def test_counter_trend_signal_rejected(self):
        """Long signal against bearish higher TF must be rejected."""
        gate = SignalConfirmationGate(require_higher_tf_agreement=True)
        signal = Signal(
            symbol="AAPL",
            direction=SignalDirection.LONG,
            strategy="ema_cross",
            strength=1.0,
            timeframe="5min",
            bar_count=10,
        )
        result = gate.evaluate(signal, current_bar=100, higher_tf_direction=SignalDirection.SHORT)
        assert result == SignalStatus.REJECTED

    def test_cooldown_prevents_rapid_reentry(self):
        """Cannot fire same signal again within cooldown period."""
        gate = SignalConfirmationGate(cooldown_bars=10)
        signal = Signal(
            symbol="AAPL",
            direction=SignalDirection.LONG,
            strategy="ema_cross",
            strength=1.0,
            timeframe="5min",
            bar_count=5,
        )
        # First signal confirmed
        gate.evaluate(signal, current_bar=100, higher_tf_direction=SignalDirection.LONG)
        # Second signal within cooldown rejected
        result = gate.evaluate(signal, current_bar=105, higher_tf_direction=SignalDirection.LONG)
        assert result == SignalStatus.REJECTED

    def test_cloud_flip_exit_requires_confirmation(self):
        """EMA cloud flip must persist before triggering exit."""
        exit_signal = EMACloudExitSignal(confirmation_bars=3, min_spread_atr_multiple=0.3)

        # Single bar flip -- should NOT trigger exit
        should_exit, _ = exit_signal.check_exit("AAPL", 99.5, 100.0, 2.0, "long")
        assert not should_exit

        # Second bar
        should_exit, _ = exit_signal.check_exit("AAPL", 99.3, 100.0, 2.0, "long")
        assert not should_exit

        # Third bar with sufficient spread -- NOW should trigger
        should_exit, _ = exit_signal.check_exit("AAPL", 99.0, 100.0, 2.0, "long")
        assert should_exit
```

---

## Integration

- **trading-bot-skills:indicator-math-verification** -- Signal validation depends on
  correct indicator math. Verify EMA, ATR, and curl computations against reference
  implementations before trusting signal outputs.
- **trading-bot-skills:backtesting-before-live** -- Every signal validation parameter
  (confirmation bars, ATR threshold, cooldown period) must be validated through
  backtesting before live deployment.
