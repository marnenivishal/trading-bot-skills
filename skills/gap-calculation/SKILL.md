---
name: gap-calculation
description: Use when calculating market gaps, classifying gap types, implementing gap-based trading strategies, or when gap calculations produce incorrect results due to corporate actions
---

# Gap Calculation

## Iron Law

**EVERY GAP CALCULATION USES ADJUSTED CLOSE PRICES. UNADJUSTED PRICES PRODUCE PHANTOM GAPS.**

Raw close prices do not account for corporate actions. A stock that splits 2:1 from $200
to $100 looks like a 50% gap down on raw data. Adjusted close normalizes the historical
series so price changes reflect real market movement only. If your gap scanner is not
using adjusted close, it is generating phantom signals that will cost you real money.

---

## 1. The Phantom Gap Problem

### Case Study: The Split That Looked Like a Short Signal

A bot scanning for mean-reversion setups detected a "5% gap up" on XYZ at open. It went
short, expecting the gap to fill within the session.

**What actually happened:** XYZ had executed a 2:1 stock split. The previous raw close
was $200, the post-split open was $102. Raw data showed $200 -> $102, but the adjusted
previous close was $100. The real gap was only 2% up -- below the bot's threshold.

**The result:** The bot shorted into a stock genuinely trending up on a catalyst. The
"gap" was an artifact of unadjusted data. Real money lost chasing a phantom signal.

**Root cause:** Data pipeline delivered raw close prices. No corporate action filter
existed. The gap calculator treated every price discontinuity as a tradeable gap.

**The fix:** Every gap calculation must use adjusted close, and every large gap must be
cross-checked against a corporate action calendar before generating a signal.

---

## 2. Adjusted Close Prices

### What Adjusted Close Accounts For

| Corporate Action     | Effect on Raw Close        | Adjusted Close Handles It? |
|----------------------|----------------------------|----------------------------|
| Cash dividends       | Price drops by dividend    | Yes                        |
| Stock splits         | Price divided by ratio     | Yes                        |
| Reverse splits       | Price multiplied by ratio  | Yes                        |
| Secondary offerings  | Dilution pressure          | Yes                        |
| Spin-offs            | Price reduced by spin-off  | Yes                        |

### Why Raw Close Creates False Signals

Raw close preserves the literal traded price. A $100 stock paying a $2 dividend opens
at $99 -- looks like a 1% gap down on raw data. On adjusted data, the previous close
is retroactively set to $98, showing the open at $99 is actually a 1% gap *up*. The
raw signal is not just wrong -- it points in the opposite direction.

### Gap Percentage Formula

```
Gap% = (Open - AdjPrevClose) / AdjPrevClose * 100
```

### Code: AdjustedPriceCalculator

```python
from dataclasses import dataclass

@dataclass
class AdjustedPriceCalculator:
    """Calculates gap percentages using adjusted close prices only."""
    raw_close: float
    adjusted_close: float
    current_open: float

    def gap_percent(self) -> float:
        if self.adjusted_close <= 0:
            raise ValueError("Adjusted close must be positive")
        return ((self.current_open - self.adjusted_close) / self.adjusted_close) * 100

    def has_corporate_action(self, tolerance: float = 0.01) -> bool:
        """Detect corporate action by comparing raw vs adjusted."""
        if self.raw_close <= 0:
            return False
        ratio = abs(self.raw_close - self.adjusted_close) / self.raw_close
        return ratio > tolerance
```

---

## 3. Volume Multiplier Validation

A gap without volume is noise. Without volume confirmation, a gap is just a wide
spread in a thin market.

**Rule:** Opening volume must be >= 1.5x the 20-period average volume to confirm
institutional participation. Gaps below this threshold are downgraded or ignored.

### Code: VolumeMultiplierFilter

```python
from dataclasses import dataclass, field

@dataclass
class VolumeMultiplierFilter:
    """Filters gaps by volume confirmation."""
    volume_history: list[int] = field(default_factory=list)
    lookback_period: int = 20
    min_multiplier: float = 1.5

    def average_volume(self) -> float:
        if not self.volume_history:
            return 0.0
        window = self.volume_history[-self.lookback_period:]
        return sum(window) / len(window)

    def is_confirmed(self, opening_volume: int) -> bool:
        avg = self.average_volume()
        if avg <= 0:
            return False
        return opening_volume >= (avg * self.min_multiplier)

    def volume_multiplier(self, opening_volume: int) -> float:
        avg = self.average_volume()
        if avg <= 0:
            return 0.0
        return opening_volume / avg
```

---

## 4. Gap Classification

Classification determines whether a gap is actionable, requires confirmation, or
should be ignored entirely.

### Four Gap Types

| Type           | Size     | Volume       | Interpretation                              |
|----------------|----------|--------------|---------------------------------------------|
| Full Gap       | > 1%     | High (>1.5x) | Institutional move, likely continuation      |
| Partial Gap    | 0.5-1%   | Moderate     | Mixed signal, wait for confirmation bar      |
| Noise Gap      | < 0.5%   | Low          | Ignore entirely, not tradeable               |
| Exhaustion Gap | > 1%     | High (>2x)   | At end of extended move, likely reversal     |

### Exhaustion Gap Detection

An exhaustion gap occurs after an extended directional move -- high volume but marks
the final push before reversal. Key signal: price moved >10% in the same direction
over the prior 10 sessions before the gap.

### Code: GapClassifier

```python
from dataclasses import dataclass
from enum import Enum

class GapType(Enum):
    FULL_GAP_UP = "full_gap_up"
    FULL_GAP_DOWN = "full_gap_down"
    PARTIAL_GAP_UP = "partial_gap_up"
    PARTIAL_GAP_DOWN = "partial_gap_down"
    NOISE = "noise"
    EXHAUSTION_UP = "exhaustion_up"
    EXHAUSTION_DOWN = "exhaustion_down"

@dataclass
class GapClassifier:
    """Classifies gaps by size, volume, and context."""
    gap_percent: float
    volume_multiplier: float
    prior_trend_percent: float = 0.0
    exhaustion_trend_threshold: float = 10.0

    def classify(self) -> GapType:
        abs_gap = abs(self.gap_percent)
        direction_up = self.gap_percent > 0

        if abs_gap < 0.5:
            return GapType.NOISE

        # Exhaustion: large gap + high volume at end of extended move
        if abs_gap >= 1.0 and self.volume_multiplier >= 2.0:
            if abs(self.prior_trend_percent) >= self.exhaustion_trend_threshold:
                trend_up = self.prior_trend_percent > 0
                if trend_up == direction_up:
                    return GapType.EXHAUSTION_UP if direction_up else GapType.EXHAUSTION_DOWN

        if abs_gap >= 1.0 and self.volume_multiplier >= 1.5:
            return GapType.FULL_GAP_UP if direction_up else GapType.FULL_GAP_DOWN

        if 0.5 <= abs_gap < 1.0:
            return GapType.PARTIAL_GAP_UP if direction_up else GapType.PARTIAL_GAP_DOWN

        return GapType.NOISE  # Large gap but low volume

    def is_actionable(self) -> bool:
        gap_type = self.classify()
        return gap_type in (
            GapType.FULL_GAP_UP, GapType.FULL_GAP_DOWN,
            GapType.EXHAUSTION_UP, GapType.EXHAUSTION_DOWN,
        )
```

---

## 5. Gap Fill Probability

Probability depends on gap type and current market regime. These are empirical
reference values -- use them as priors, not certainties.

### Gap Fill Probabilities by Type and Regime

| Gap Type       | Trending Market (same day) | Ranging Market (same day) | Within 5 Days |
|----------------|----------------------------|---------------------------|---------------|
| Full Gap Up    | 25-30%                     | 55-65%                    | 70-80%        |
| Full Gap Down  | 30-35%                     | 60-70%                    | 75-85%        |
| Partial Gap    | 50-60%                     | 70-80%                    | 85-90%        |
| Exhaustion Gap | 60-70%                     | 75-85%                    | 90-95%        |
| Noise Gap      | N/A (do not trade)         | N/A (do not trade)        | N/A           |

### Key Observations

- **Full gaps in trends rarely fill same-day.** Continuation entries have higher
  expectancy than fading.
- **Exhaustion gaps fill at high rates.** They mark the end of a move -- fading with
  tight stops is valid.
- **Ranging markets fill gaps more often.** Mean reversion works without directional
  bias.
- **Regime detection is mandatory.** Same gap in two regimes demands opposite
  strategies. See `spy-vix-regime-trading` for regime classification.

---

## 6. Corporate Action Detection

Before trading any gap, verify it is not caused by a corporate action. Real gaps and
corporate-action gaps are indistinguishable on raw price data.

### Checks Before Trading a Gap

1. **Ex-dividend date:** Does today match an ex-dividend date for this symbol?
2. **Split announcement:** Has a split been announced effective near today?
3. **Secondary offering:** Has a secondary offering been priced overnight?
4. **Spin-off:** Is a spin-off effective today?

### Code: CorporateActionFilter

```python
from dataclasses import dataclass
from datetime import date
from typing import Optional

@dataclass
class CorporateAction:
    symbol: str
    action_type: str  # "dividend", "split", "secondary", "spinoff"
    effective_date: date
    adjustment_factor: Optional[float] = None

@dataclass
class CorporateActionFilter:
    """Flags gaps that may be caused by corporate actions."""
    actions: list[CorporateAction]
    suspicious_gap_threshold: float = 3.0

    def has_action_on_date(self, symbol: str, check_date: date) -> bool:
        return any(
            a.symbol == symbol and a.effective_date == check_date
            for a in self.actions
        )

    def get_actions(self, symbol: str, check_date: date) -> list[CorporateAction]:
        return [a for a in self.actions
                if a.symbol == symbol and a.effective_date == check_date]

    def is_suspicious(self, symbol: str, check_date: date,
                      gap_percent: float) -> bool:
        if self.has_action_on_date(symbol, check_date):
            return True
        if abs(gap_percent) >= self.suspicious_gap_threshold:
            return True  # Large gap with no known cause -- flag for review
        return False
```

---

## Red Flags

| Red Flag                                        | What It Means                                      | Action                                   |
|-------------------------------------------------|----------------------------------------------------|------------------------------------------|
| Gap > 5% with no news catalyst                  | Likely corporate action or data error              | Do not trade until verified              |
| Gap direction opposes adjusted-close direction   | Raw vs adjusted mismatch, phantom gap              | Recalculate with adjusted close          |
| High gap % but low volume multiplier (< 1.0x)   | Thin market, unreliable signal                     | Ignore the gap                           |
| Gap coincides with ex-dividend date              | Gap is dividend artifact, not real momentum         | Subtract dividend from gap calculation   |
| Multiple large gaps in same session across symbols | Possible data feed issue                          | Halt gap-based trading, check data source|
| Exhaustion gap after < 5% prior trend            | Misclassified, likely a continuation gap           | Reclassify and re-evaluate               |

---

## Integration Points

| Skill                            | Relationship                                                             |
|----------------------------------|--------------------------------------------------------------------------|
| `spy-vix-regime-trading`         | Regime determines gap fill probability and strategy direction            |
| `market-data-pipeline`           | Must deliver adjusted close prices; raw close alone is insufficient      |
| `indicator-math-verification`    | Gap calculations must be verified against reference implementations      |
| `strategy-signal-validation`     | Gap signals must pass validation gates before generating orders          |
