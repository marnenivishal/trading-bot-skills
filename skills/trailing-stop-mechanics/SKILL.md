---
name: trailing-stop-mechanics
description: Use when implementing trailing stops, ratchet stops, or any dynamic stop-loss adjustment, or when stops are loosening when they should only tighten
---

# Trailing Stop Mechanics

## Iron Law

**new_stop = max(current_stop, computed_stop) FOR LONGS.**
**new_stop = min(current_stop, computed_stop) FOR SHORTS.**
**ALWAYS. NO EXCEPTIONS. STOPS ONLY TIGHTEN, NEVER LOOSEN.**

A trailing stop that can move backward is not a stop -- it is a suggestion. The entire purpose
of a trailing stop is to lock in gains as price moves favorably. If your code allows the stop
to retreat, you have a bug that will cost real money.

## The Ratchet Bug (Exact Emabot Failure)

In emabot, the trailing stop was implemented like this:

```python
# EMABOT BUG - DO NOT COPY
def update_trailing_stop(position, current_price, trail_pct):
    trail_amount = current_price * trail_pct
    new_stop = current_price - trail_amount  # For longs
    position.stop_price = new_stop  # <-- THE BUG
```

The problem: when price pulls back, `current_price` drops, so `current_price - trail_amount`
also drops. The stop moves DOWN with falling price. The trailing stop was supposed to only
go up, but it went both directions -- it was just a floating offset from current price, not
a ratchet at all.

Price moved: 100 -> 110 -> 105 -> 95. Stop moved: 95 -> 104.5 -> 99.75 -> 90.25.
The stop retreated from 104.5 to 90.25. The position was finally stopped out at 90.25
instead of 104.5, losing an extra $14.25 per share.

### The Fix

```python
# CORRECT IMPLEMENTATION
def update_trailing_stop(position, current_price, trail_pct):
    # Track the high water mark
    position.high_water_mark = max(position.high_water_mark, current_price)

    # Compute stop from high water mark, not current price
    trail_amount = position.high_water_mark * trail_pct
    computed_stop = position.high_water_mark - trail_amount

    # RATCHET: stop can only go UP for longs
    position.stop_price = max(position.stop_price, computed_stop)
```

The key differences:
1. Track `high_water_mark` separately -- it only goes up
2. Compute trail from `high_water_mark`, not `current_price`
3. Apply `max()` so `stop_price` can only increase for longs

## High Water Mark Tracking

The high water mark is the foundation of a correct trailing stop. Without it, you cannot
compute a correct trail.

```python
from dataclasses import dataclass, field


@dataclass
class TrailingStopState:
    """
    Complete state for a trailing stop. All fields are required.
    high_water_mark and stop_price only move in the favorable direction.
    """
    symbol: str
    side: str  # "long" or "short"
    entry_price: float
    initial_stop: float
    high_water_mark: float  # Highest price seen (longs) or lowest (shorts)
    stop_price: float       # Current stop level -- only tightens

    def __post_init__(self):
        """Validate invariants on construction."""
        if self.side == "long":
            assert self.stop_price < self.entry_price, "long stop must be below entry"
            assert self.high_water_mark >= self.entry_price, "HWM must be >= entry"
        elif self.side == "short":
            assert self.stop_price > self.entry_price, "short stop must be above entry"
            assert self.high_water_mark <= self.entry_price, "HWM must be <= entry for shorts"

    def update(self, current_price: float, trail_pct: float) -> float:
        """
        Update trailing stop with new price. Returns new stop price.
        Stop can only tighten (move toward current price), never loosen.
        """
        if self.side == "long":
            return self._update_long(current_price, trail_pct)
        elif self.side == "short":
            return self._update_short(current_price, trail_pct)
        else:
            raise ValueError(f"Invalid side: {self.side}")

    def _update_long(self, current_price: float, trail_pct: float) -> float:
        """For longs: HWM goes up, stop goes up, never down."""
        self.high_water_mark = max(self.high_water_mark, current_price)
        computed_stop = self.high_water_mark * (1.0 - trail_pct)
        self.stop_price = max(self.stop_price, computed_stop)
        return self.stop_price

    def _update_short(self, current_price: float, trail_pct: float) -> float:
        """For shorts: HWM (low water mark) goes down, stop goes down, never up."""
        self.high_water_mark = min(self.high_water_mark, current_price)
        computed_stop = self.high_water_mark * (1.0 + trail_pct)
        self.stop_price = min(self.stop_price, computed_stop)
        return self.stop_price

    def is_triggered(self, current_price: float) -> bool:
        """Check if stop has been triggered."""
        if self.side == "long":
            return current_price <= self.stop_price
        else:
            return current_price >= self.stop_price
```

## Implementation Patterns

### Pattern A: Percentage Trailing Stop

The simplest form. Trail a fixed percentage below the high water mark.

```python
def percentage_trail(hwm: float, trail_pct: float, current_stop: float, side: str) -> float:
    """
    Percentage trailing stop.
    trail_pct: e.g., 0.05 for 5% trail.
    """
    if side == "long":
        computed = hwm * (1.0 - trail_pct)
        return max(current_stop, computed)
    else:
        computed = hwm * (1.0 + trail_pct)
        return min(current_stop, computed)
```

### Pattern B: ATR Trailing Stop

Trail by a multiple of Average True Range. Adapts to volatility.

```python
def atr_trail(
    hwm: float,
    atr_value: float,
    atr_multiplier: float,
    current_stop: float,
    side: str,
) -> float:
    """
    ATR-based trailing stop.
    atr_value: current ATR value for the symbol.
    atr_multiplier: e.g., 2.0 means trail 2x ATR from HWM.
    """
    trail_distance = atr_value * atr_multiplier

    if side == "long":
        computed = hwm - trail_distance
        return max(current_stop, computed)
    else:
        computed = hwm + trail_distance
        return min(current_stop, computed)
```

### Pattern C: Stepped / Ratchet Stop

Stop moves in discrete steps as price crosses thresholds. Useful for locking in
specific profit levels.

```python
from typing import List, Tuple


def stepped_trail(
    entry_price: float,
    current_price: float,
    current_stop: float,
    steps: List[Tuple[float, float]],
    side: str,
) -> float:
    """
    Stepped trailing stop. Moves stop to specific levels at profit thresholds.

    steps: List of (profit_pct_threshold, stop_pct_from_entry).
           e.g., [(0.02, 0.0), (0.05, 0.02), (0.10, 0.05)]
           means: at 2% profit move stop to breakeven,
                  at 5% profit move stop to +2%,
                  at 10% profit move stop to +5%.
    Steps MUST be sorted by profit threshold ascending.
    """
    computed_stop = current_stop

    for profit_threshold, stop_level in steps:
        if side == "long":
            threshold_price = entry_price * (1.0 + profit_threshold)
            if current_price >= threshold_price:
                stop_candidate = entry_price * (1.0 + stop_level)
                computed_stop = max(computed_stop, stop_candidate)
        else:
            threshold_price = entry_price * (1.0 - profit_threshold)
            if current_price <= threshold_price:
                stop_candidate = entry_price * (1.0 - stop_level)
                computed_stop = min(computed_stop, stop_candidate)

    # RATCHET GUARD: stop can only tighten
    if side == "long":
        return max(current_stop, computed_stop)
    else:
        return min(current_stop, computed_stop)
```

## Verification Test

This test MUST pass for any trailing stop implementation. It is the minimum bar.

```python
def test_trailing_stop_never_decreases_for_longs():
    """
    Price sequence: [100, 105, 110, 108, 103, 115, 112]
    Trail: 5%
    Entry: 100, Initial stop: 95

    Expected behavior:
      Price 100 -> HWM 100, computed 95.00, stop = max(95, 95.00)    = 95.00
      Price 105 -> HWM 105, computed 99.75, stop = max(95.00, 99.75) = 99.75
      Price 110 -> HWM 110, computed 104.50, stop = max(99.75, 104.50) = 104.50
      Price 108 -> HWM 110, computed 104.50, stop = max(104.50, 104.50) = 104.50  (no change!)
      Price 103 -> HWM 110, computed 104.50, stop = max(104.50, 104.50) = 104.50  (no change!)
      Price 115 -> HWM 115, computed 109.25, stop = max(104.50, 109.25) = 109.25
      Price 112 -> HWM 115, computed 109.25, stop = max(109.25, 109.25) = 109.25  (no change!)

    CRITICAL: stop must NEVER decrease. At no point does the stop go below a previous value.
    """
    state = TrailingStopState(
        symbol="TEST",
        side="long",
        entry_price=100.0,
        initial_stop=95.0,
        high_water_mark=100.0,
        stop_price=95.0,
    )

    prices = [100, 105, 110, 108, 103, 115, 112]
    expected_stops = [95.0, 99.75, 104.50, 104.50, 104.50, 109.25, 109.25]
    trail_pct = 0.05

    previous_stop = 0.0
    for i, price in enumerate(prices):
        new_stop = state.update(price, trail_pct)

        # THE CRITICAL ASSERTION: stop never decreases
        assert new_stop >= previous_stop, (
            f"RATCHET BUG: stop decreased from {previous_stop} to {new_stop} "
            f"at price {price} (index {i})"
        )

        # Verify expected values
        assert abs(new_stop - expected_stops[i]) < 0.01, (
            f"Stop mismatch at index {i}: expected {expected_stops[i]}, got {new_stop}"
        )

        previous_stop = new_stop


def test_trailing_stop_never_increases_for_shorts():
    """
    Mirror test for short positions. Stop must NEVER increase.
    Price sequence: [100, 95, 90, 92, 97, 85, 88]
    Trail: 5%
    Entry: 100, Initial stop: 105
    """
    state = TrailingStopState(
        symbol="TEST",
        side="short",
        entry_price=100.0,
        initial_stop=105.0,
        high_water_mark=100.0,
        stop_price=105.0,
    )

    prices = [100, 95, 90, 92, 97, 85, 88]
    expected_stops = [105.0, 99.75, 94.50, 94.50, 94.50, 89.25, 89.25]
    trail_pct = 0.05

    previous_stop = float("inf")
    for i, price in enumerate(prices):
        new_stop = state.update(price, trail_pct)

        # THE CRITICAL ASSERTION: stop never increases for shorts
        assert new_stop <= previous_stop, (
            f"RATCHET BUG (short): stop increased from {previous_stop} to {new_stop} "
            f"at price {price} (index {i})"
        )

        assert abs(new_stop - expected_stops[i]) < 0.01, (
            f"Stop mismatch at index {i}: expected {expected_stops[i]}, got {new_stop}"
        )

        previous_stop = new_stop
```

## Trailing Stop State Flow

```
digraph trailing_stop {
    rankdir=LR;
    node [shape=box, style=rounded];

    tick [label="New Price Tick"];
    hwm [label="Update High Water Mark\nHWM = max(HWM, price)"];
    compute [label="Compute Trail Stop\ncomputed = HWM * (1 - trail%)"];
    ratchet [label="Ratchet Guard\nstop = max(stop, computed)", style="filled", fillcolor="lightyellow"];
    check [label="Check Triggered?\nprice <= stop?", shape=diamond];
    hold [label="Hold Position"];
    exit [label="EXIT POSITION", style="filled", fillcolor="salmon"];

    tick -> hwm;
    hwm -> compute;
    compute -> ratchet;
    ratchet -> check;
    check -> hold [label="No"];
    check -> exit [label="Yes"];
    hold -> tick [label="Next tick"];
}
```

## Short Position Handling

The mirror image of long trailing stops. Everything is inverted:

| Concept | Long | Short |
|---|---|---|
| High water mark | `max(hwm, price)` -- tracks highest | `min(hwm, price)` -- tracks lowest |
| Computed stop | `hwm * (1 - trail%)` -- below HWM | `hwm * (1 + trail%)` -- above HWM |
| Ratchet guard | `max(current_stop, computed)` -- only up | `min(current_stop, computed)` -- only down |
| Triggered when | `price <= stop` | `price >= stop` |

**The most common bug is using max() for shorts.** If your code has `max(current_stop, computed)`
for a short position, it will ratchet the WRONG direction -- the stop will only increase,
meaning it moves AWAY from the favorable direction and your stop gets worse over time.

## Red Flags

- **No high_water_mark field:** Without HWM tracking, the trailing stop is computed from current
  price, which means it retreats when price retreats. This is the emabot bug.
- **Stop computed from current price without max/min guard:** `stop = price - trail` without
  comparing to previous stop. The stop will go up AND down with price.
- **Using max() for short trailing stops:** The stop ratchets upward (away from profit), making
  the stop worse as the trade moves in your favor.
- **Using min() for long trailing stops:** Mirror of the above -- stop retreats with pullbacks.
- **No verification test with pullback prices:** If your test only has monotonically increasing
  prices, it will not catch the ratchet bug. ALWAYS test with pullbacks.
- **Mutable stop state without audit trail:** If you overwrite stop_price without logging the
  previous value, you cannot detect the ratchet bug in production. Log every stop change.
- **Trail percentage applied to entry instead of HWM:** `entry_price * (1 - trail%)` is a fixed
  stop, not a trailing stop. The trail must be from the high water mark.

## Integration

- **trading-bot-skills:risk-management-gates** -- The initial stop-loss is set by risk management
  gates. The trailing stop takes over after entry and only tightens from there.
- **trading-bot-skills:indicator-math-verification** -- ATR-based trailing stops depend on correct
  ATR calculation. Verify ATR with known values before using for stop placement.
- **trading-bot-skills:trading-tdd** -- The verification test in this skill should be included in
  your test suite. Run it on every change to trailing stop logic. It catches the ratchet bug
  immediately.
