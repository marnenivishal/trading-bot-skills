---
name: indicator-math-verification
description: Use when implementing any mathematical indicator (EMA, ATR, RSI, VWAP, Bollinger Bands), trailing stop formulas, or position sizing calculations
---

# Indicator Math Verification

## Iron Law

**EVERY INDICATOR FUNCTION MUST HAVE A TEST COMPARING OUTPUT TO AN INDEPENDENT REFERENCE IMPLEMENTATION. NO EXCEPTIONS.**

"I computed it by hand and it looks right" is not verification. "I eyeballed the chart
and it seems close" is not verification. You need a reference library (pandas-ta, TA-Lib,
or a known-good independent implementation), and you need to compare outputs
programmatically with a defined tolerance.

## Prevents

- **Trailing stop math bugs (#8):** Non-monotonic trailing stops that move AWAY from
  price, increasing risk instead of locking in profit.
- **Curl detection errors (#15):** Incorrect EMA computation feeding bad values into
  curl detection, generating false signals.

---

## Reference Testing Pattern

The gold standard for indicator verification:

1. Compute the indicator with YOUR code
2. Compute the same indicator with a REFERENCE library (pandas-ta, TA-Lib)
3. Compare outputs element-by-element
4. Maximum allowed divergence: 0.01% (1 basis point)

```python
import numpy as np
import pandas as pd
import pandas_ta as ta
import pytest


def custom_ema(prices: list[float], period: int) -> list[float]:
    """
    Custom EMA implementation.

    EMA formula:
        multiplier = 2 / (period + 1)
        ema[0] = prices[0]  (or SMA of first `period` values)
        ema[i] = prices[i] * multiplier + ema[i-1] * (1 - multiplier)
    """
    if len(prices) < period:
        return []

    multiplier = 2.0 / (period + 1)
    # Initialize with SMA of first `period` values
    ema_values = []
    sma = sum(prices[:period]) / period
    ema_values.append(sma)

    for i in range(period, len(prices)):
        ema = prices[i] * multiplier + ema_values[-1] * (1 - multiplier)
        ema_values.append(ema)

    return ema_values


class TestEMAAgainstReference:
    """Compare custom EMA to pandas-ta reference implementation."""

    @pytest.fixture
    def sample_prices(self) -> pd.Series:
        """Real-ish price data for testing."""
        np.random.seed(42)
        returns = np.random.normal(0.0005, 0.02, 200)
        prices = 100 * np.cumprod(1 + returns)
        return pd.Series(prices)

    def test_ema_matches_reference(self, sample_prices):
        period = 20
        # Our implementation
        custom = custom_ema(sample_prices.tolist(), period)

        # Reference (pandas-ta)
        reference = ta.ema(sample_prices, length=period).dropna().tolist()

        # Compare -- maximum 0.01% divergence
        assert len(custom) == len(reference), (
            f"Length mismatch: custom={len(custom)}, reference={len(reference)}"
        )
        for i, (c, r) in enumerate(zip(custom, reference)):
            if r != 0:
                pct_diff = abs(c - r) / abs(r) * 100
                assert pct_diff < 0.01, (
                    f"EMA divergence at index {i}: custom={c:.6f}, "
                    f"reference={r:.6f}, diff={pct_diff:.4f}%"
                )

    def test_ema_different_periods(self, sample_prices):
        """Test across multiple periods to catch off-by-one errors."""
        for period in [5, 10, 20, 50, 100]:
            custom = custom_ema(sample_prices.tolist(), period)
            reference = ta.ema(sample_prices, length=period).dropna().tolist()
            assert len(custom) == len(reference), f"Length mismatch for period={period}"

            max_divergence = max(
                abs(c - r) / abs(r) * 100
                for c, r in zip(custom, reference)
                if r != 0
            )
            assert max_divergence < 0.01, (
                f"EMA period={period}: max divergence={max_divergence:.4f}%"
            )
```

---

## Common Math Bugs

### 1. Wrong EMA Smoothing Factor

```python
# BUG: Using 1/period instead of 2/(period+1)
multiplier = 1.0 / period  # WRONG

# CORRECT:
multiplier = 2.0 / (period + 1)
```

This is the single most common EMA bug. The difference is subtle for large periods
but significant for small ones (e.g., EMA-5).

### 2. ATR Without True Range

```python
# BUG: Using high-low only (ignoring gaps)
def bad_atr(highs, lows, period):
    ranges = [h - l for h, l in zip(highs, lows)]
    return sum(ranges[-period:]) / period

# CORRECT: True Range accounts for gaps
def true_range(high: float, low: float, prev_close: float) -> float:
    return max(
        high - low,
        abs(high - prev_close),
        abs(low - prev_close),
    )

def atr(highs: list, lows: list, closes: list, period: int) -> list[float]:
    """ATR using true range (handles gaps correctly)."""
    tr_values = []
    for i in range(1, len(highs)):
        tr = true_range(highs[i], lows[i], closes[i - 1])
        tr_values.append(tr)

    # ATR is EMA of true range values
    atr_values = []
    if len(tr_values) >= period:
        first_atr = sum(tr_values[:period]) / period
        atr_values.append(first_atr)
        multiplier = 2.0 / (period + 1)
        for i in range(period, len(tr_values)):
            new_atr = tr_values[i] * multiplier + atr_values[-1] * (1 - multiplier)
            atr_values.append(new_atr)

    return atr_values
```

### 3. Trailing Stop Monotonicity Violation

This is the bug from Emabot issue #8. A trailing stop for a long position must
ONLY move UP, never down. If it moves down, it is increasing risk instead of
locking in profit.

```python
# BUG: Trailing stop recomputed from scratch each bar
def bad_trailing_stop(price: float, atr: float, multiplier: float) -> float:
    return price - (atr * multiplier)
    # If price drops, stop drops too -- DEFEATS THE PURPOSE

# CORRECT: Ratcheting trailing stop
@dataclass
class TrailingStop:
    """
    Trailing stop that can only move in the favorable direction.

    For longs: stop can only move UP (ratchet up).
    For shorts: stop can only move DOWN (ratchet down).
    """
    direction: str  # "long" or "short"
    atr_multiplier: float = 2.0
    _current_stop: Optional[float] = None

    def update(self, price: float, atr: float) -> float:
        """
        Update trailing stop. Returns new stop level.

        INVARIANT: For longs, new_stop >= old_stop (monotonically increasing).
        INVARIANT: For shorts, new_stop <= old_stop (monotonically decreasing).
        """
        candidate = self._compute_candidate(price, atr)

        if self._current_stop is None:
            self._current_stop = candidate
            return self._current_stop

        if self.direction == "long":
            # Long: stop can only go UP
            self._current_stop = max(self._current_stop, candidate)
        elif self.direction == "short":
            # Short: stop can only go DOWN
            self._current_stop = min(self._current_stop, candidate)

        return self._current_stop

    def _compute_candidate(self, price: float, atr: float) -> float:
        if self.direction == "long":
            return price - (atr * self.atr_multiplier)
        else:
            return price + (atr * self.atr_multiplier)

    @property
    def level(self) -> Optional[float]:
        return self._current_stop
```

### 4. Integer Division

```python
# BUG: Python 2 legacy or explicit floor division
lookback = period // 2 + 1  # Off by 1 for even periods?

# CAREFUL: Always verify with explicit test cases
assert lookback_for_period(10) == 6  # Document expected values
```

### 5. Off-by-One Lookback

```python
# BUG: prices[-5:] gives 5 elements, but lookback=5 means 5 bars BACK
#      which is 6 data points (current + 5 prior)
curl = prices[-1] - prices[-5]  # This is 4-bar lookback, not 5

# CORRECT:
curl = prices[-1] - prices[-(lookback + 1)]  # N bars back from current
```

---

## Position Sizing Math

### Common Bugs

```python
# BUG 1: Risk calculated on notional, not account
def bad_position_size(price, stop_distance, account_value, risk_pct):
    return (account_value * risk_pct) / price  # WRONG: ignores stop distance

# BUG 2: Stop distance as percentage instead of absolute
def also_bad(price, stop_pct, account_value, risk_pct):
    risk_amount = account_value * risk_pct
    # If stop_pct is 0.02 (2%), stop distance = price * stop_pct
    # But code uses stop_pct directly as dollar amount -- DISASTER
    shares = risk_amount / stop_pct  # Way too many shares!
    return shares

# CORRECT:
def position_size(
    account_value: float,
    risk_per_trade: float,  # e.g., 0.01 = 1% of account
    entry_price: float,
    stop_price: float,
) -> int:
    """
    Calculate position size based on account risk and stop distance.

    Risk amount = account_value * risk_per_trade
    Stop distance = abs(entry_price - stop_price)
    Shares = risk_amount / stop_distance

    Returns integer shares (rounded down -- never round up).
    """
    risk_amount = account_value * risk_per_trade
    stop_distance = abs(entry_price - stop_price)

    if stop_distance <= 0:
        raise ValueError(
            f"Invalid stop distance: entry={entry_price}, stop={stop_price}"
        )

    shares = risk_amount / stop_distance
    return int(shares)  # Floor -- NEVER round up


class TestPositionSizing:
    def test_basic_calculation(self):
        # $100k account, 1% risk, entry at $50, stop at $48
        # Risk = $1000, stop distance = $2, shares = 500
        assert position_size(100_000, 0.01, 50.0, 48.0) == 500

    def test_zero_stop_distance_raises(self):
        with pytest.raises(ValueError):
            position_size(100_000, 0.01, 50.0, 50.0)

    def test_rounds_down_not_up(self):
        # Risk=$1000, stop_distance=$3 -> 333.33 -> floor to 333
        assert position_size(100_000, 0.01, 50.0, 47.0) == 333

    def test_short_position(self):
        # Short: entry at $50, stop at $52, distance = $2
        assert position_size(100_000, 0.01, 50.0, 52.0) == 500
```

---

## RSI Verification

```python
def custom_rsi(prices: list[float], period: int = 14) -> list[float]:
    """
    RSI implementation using Wilder's smoothing method.

    RSI = 100 - (100 / (1 + RS))
    RS = avg_gain / avg_loss
    """
    if len(prices) < period + 1:
        return []

    deltas = [prices[i] - prices[i - 1] for i in range(1, len(prices))]
    gains = [max(d, 0) for d in deltas]
    losses = [abs(min(d, 0)) for d in deltas]

    # First average: simple mean of first `period` values
    avg_gain = sum(gains[:period]) / period
    avg_loss = sum(losses[:period]) / period

    rsi_values = []
    if avg_loss == 0:
        rsi_values.append(100.0)
    else:
        rs = avg_gain / avg_loss
        rsi_values.append(100.0 - (100.0 / (1.0 + rs)))

    # Subsequent: Wilder's smoothing
    for i in range(period, len(deltas)):
        avg_gain = (avg_gain * (period - 1) + gains[i]) / period
        avg_loss = (avg_loss * (period - 1) + losses[i]) / period

        if avg_loss == 0:
            rsi_values.append(100.0)
        else:
            rs = avg_gain / avg_loss
            rsi_values.append(100.0 - (100.0 / (1.0 + rs)))

    return rsi_values


class TestRSIAgainstReference:
    def test_rsi_matches_pandas_ta(self):
        np.random.seed(42)
        prices = pd.Series(100 * np.cumprod(1 + np.random.normal(0, 0.02, 200)))
        period = 14

        custom = custom_rsi(prices.tolist(), period)
        reference = ta.rsi(prices, length=period).dropna().tolist()

        # Allow slightly more tolerance for RSI due to initialization differences
        for i in range(5, len(custom)):  # Skip first few for initialization
            if reference[i] != 0:
                pct_diff = abs(custom[i] - reference[i]) / abs(reference[i]) * 100
                assert pct_diff < 0.1, (
                    f"RSI divergence at index {i}: "
                    f"custom={custom[i]:.4f}, ref={reference[i]:.4f}"
                )
```

---

## VWAP Verification

```python
def custom_vwap(
    highs: list[float],
    lows: list[float],
    closes: list[float],
    volumes: list[float],
) -> list[float]:
    """
    VWAP = cumulative(typical_price * volume) / cumulative(volume)
    Typical price = (high + low + close) / 3
    """
    vwap_values = []
    cumulative_tp_vol = 0.0
    cumulative_vol = 0.0

    for h, l, c, v in zip(highs, lows, closes, volumes):
        typical_price = (h + l + c) / 3.0
        cumulative_tp_vol += typical_price * v
        cumulative_vol += v

        if cumulative_vol > 0:
            vwap_values.append(cumulative_tp_vol / cumulative_vol)
        else:
            vwap_values.append(typical_price)

    return vwap_values
```

---

## Edge Cases to Test

Every indicator implementation MUST be tested with these edge cases:

### 1. First N Bars Incomplete

```python
def test_ema_insufficient_data():
    """EMA with fewer bars than period should return empty or partial."""
    result = custom_ema([100.0, 101.0, 102.0], period=20)
    assert result == []  # Not enough data
```

### 2. Zero Volume

```python
def test_vwap_zero_volume():
    """VWAP must handle zero-volume bars without division by zero."""
    result = custom_vwap(
        highs=[100.0], lows=[99.0], closes=[99.5], volumes=[0.0]
    )
    assert len(result) == 1
    assert not np.isnan(result[0])
    assert not np.isinf(result[0])
```

### 3. Price Gaps

```python
def test_atr_with_gap():
    """ATR true range must account for gap between prev close and current high/low."""
    # Previous close: 100. Current bar: open=105 (gap up), high=106, low=104
    tr = true_range(high=106.0, low=104.0, prev_close=100.0)
    # True range = max(106-104, |106-100|, |104-100|) = max(2, 6, 4) = 6
    assert tr == 6.0  # NOT 2.0 (high-low only would miss the gap)
```

### 4. Identical Prices (Flat Market)

```python
def test_ema_flat_prices():
    """EMA of constant prices should equal that constant."""
    flat = [50.0] * 100
    result = custom_ema(flat, period=20)
    for v in result:
        assert abs(v - 50.0) < 1e-10
```

### 5. Very Small and Very Large Numbers

```python
def test_position_size_penny_stock():
    """Position sizing must work for penny stocks."""
    shares = position_size(100_000, 0.01, 0.05, 0.04)
    # Risk=$1000, stop distance=$0.01, shares=100,000
    assert shares == 100_000

def test_position_size_high_price():
    """Position sizing must work for high-priced stocks."""
    shares = position_size(100_000, 0.01, 5000.0, 4900.0)
    # Risk=$1000, stop distance=$100, shares=10
    assert shares == 10
```

---

## Trailing Stop Testing (Emabot #8)

```python
class TestTrailingStopMonotonicity:
    """
    The trailing stop for a long position must ONLY move UP.
    This test catches the Emabot #8 bug where stops moved down with price.
    """

    def test_long_stop_never_decreases(self):
        stop = TrailingStop(direction="long", atr_multiplier=2.0)
        atr = 1.0

        prices = [100, 102, 104, 103, 101, 105, 103, 106]
        prev_stop = None

        for price in prices:
            current = stop.update(price, atr)
            if prev_stop is not None:
                assert current >= prev_stop, (
                    f"Long trailing stop decreased: {prev_stop} -> {current} "
                    f"at price={price}"
                )
            prev_stop = current

    def test_short_stop_never_increases(self):
        stop = TrailingStop(direction="short", atr_multiplier=2.0)
        atr = 1.0

        prices = [100, 98, 96, 97, 99, 95, 97, 94]
        prev_stop = None

        for price in prices:
            current = stop.update(price, atr)
            if prev_stop is not None:
                assert current <= prev_stop, (
                    f"Short trailing stop increased: {prev_stop} -> {current} "
                    f"at price={price}"
                )
            prev_stop = current

    def test_stop_moves_with_favorable_price(self):
        stop = TrailingStop(direction="long", atr_multiplier=2.0)
        atr = 1.0

        stop.update(100.0, atr)  # stop = 98.0
        stop.update(105.0, atr)  # stop should be 103.0

        assert stop.level == 103.0

    def test_stop_holds_on_adverse_price(self):
        stop = TrailingStop(direction="long", atr_multiplier=2.0)
        atr = 1.0

        stop.update(105.0, atr)  # stop = 103.0
        stop.update(100.0, atr)  # candidate=98.0, but stop stays at 103.0

        assert stop.level == 103.0
```

---

## Bollinger Bands Verification

```python
def custom_bollinger(
    prices: list[float], period: int = 20, std_dev: float = 2.0
) -> tuple[list[float], list[float], list[float]]:
    """
    Bollinger Bands:
        middle = SMA(period)
        upper = middle + std_dev * STDEV(period)
        lower = middle - std_dev * STDEV(period)

    COMMON BUG: Using population std dev instead of sample std dev.
    pandas uses ddof=0 (population) by default for rolling.std().
    Verify which your reference uses.
    """
    middle, upper, lower = [], [], []

    for i in range(period - 1, len(prices)):
        window = prices[i - period + 1: i + 1]
        sma = sum(window) / period
        variance = sum((p - sma) ** 2 for p in window) / period  # population
        std = variance ** 0.5

        middle.append(sma)
        upper.append(sma + std_dev * std)
        lower.append(sma - std_dev * std)

    return upper, middle, lower
```

---

## Red Flags

Stop and fix immediately if you see ANY of these:

- **No reference comparison.** "I checked it manually" is not a test. Compare against
  pandas-ta, TA-Lib, or another independent implementation.
- **"Computed by hand, looks right."** Hand computation is error-prone. Automate it.
- **Float equality without tolerance.** `assert ema == expected` will fail due to
  floating-point precision. Always use `abs(a - b) < tolerance` or `pytest.approx`.
- **Missing edge case tests.** If you have not tested zero volume, gaps, flat prices,
  and insufficient data, your implementation is not verified.
- **Trailing stop without monotonicity test.** This is the most dangerous math bug:
  a stop that moves backward INCREASES risk silently.
- **Hardcoded smoothing factors.** If you see `0.1` instead of `2/(period+1)`,
  the EMA formula is likely wrong.

---

## Integration

- **trading-bot-skills:trading-tdd** -- Every indicator function must be test-driven.
  Write the reference comparison test FIRST, then implement the indicator.
- **trading-bot-skills:trailing-stop-mechanics** -- Trailing stop math is a special
  case with critical monotonicity invariants. See that skill for complete patterns.
