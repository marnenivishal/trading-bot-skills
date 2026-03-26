---
name: spy-vix-regime-trading
description: Use when building SPY or index trading strategies, implementing VIX-based regime detection, gap trading, opening range breakout, institutional order flow analysis, or volatility-adjusted position sizing using the Rule of 16
---

# SPY/VIX Regime Trading

## Overview

The VIX is not just a risk gate — it is the primary context for every trading decision. The -0.8 inverse correlation between SPY and VIX means that volatility regime dictates position sizing, stop placement, strategy selection, and expected daily range. This skill covers the complete institutional framework for SPY day trading.

**Core principle:** Every trading decision must account for the current volatility regime. A strategy that works at VIX 12 will destroy capital at VIX 35.

## The Iron Law

```
EVERY TRADING DECISION MUST ACCOUNT FOR THE CURRENT VOLATILITY REGIME.
THE VIX IS NOT JUST A RISK GATE — IT IS THE PRIMARY CONTEXT FOR
POSITION SIZING, STOP PLACEMENT, AND STRATEGY SELECTION.
```

## When to Use

- Building any SPY, QQQ, or index-based trading strategy
- Implementing gap trading or opening range breakout
- Using VIX for more than just a halt gate
- Analyzing institutional order flow
- Sizing positions based on expected volatility
- Trading 0DTE options on SPY/SPX

---

## 1. VIX/SPY Inverse Correlation

The VIX and SPY have a historical correlation of approximately **-0.8**. In ~80% of sessions, they move in opposite directions.

**Why:** VIX measures demand for protective SPX put options. When SPY drops, institutions buy puts to hedge → implied volatility rises → VIX rises. When SPY rallies, hedging demand drops → VIX falls.

**Key property:** VIX is **mean-reverting** with **positive skew**. Fear spikes fast (VIX can double in days), but subsides slowly (takes weeks to normalize). This asymmetry creates high-probability contrarian entries at VIX extremes.

```python
from dataclasses import dataclass
from decimal import Decimal
import numpy as np

@dataclass
class VIXSPYCorrelation:
    lookback_days: int = 20
    spy_returns: list[float] = None
    vix_changes: list[float] = None

    def rolling_correlation(self) -> float:
        """Calculate rolling correlation between SPY returns and VIX changes."""
        if len(self.spy_returns) < self.lookback_days:
            return -0.8  # Default assumption
        spy = np.array(self.spy_returns[-self.lookback_days:])
        vix = np.array(self.vix_changes[-self.lookback_days:])
        corr = np.corrcoef(spy, vix)[0, 1]
        return round(corr, 3)

    def is_correlation_normal(self) -> bool:
        """Alert if correlation deviates significantly from historical."""
        corr = self.rolling_correlation()
        return -0.95 < corr < -0.5  # Normal range
```

**Red Flag:** If VIX/SPY correlation breaks above -0.5 (moving in same direction), something unusual is happening — investigate before trading.

---

## 2. Volatility Regime Detection

The absolute VIX level defines the "trading weather." Strategies and parameters must adapt.

| VIX Level | Regime | Character | Strategy Bias | Size Multiplier |
|-----------|--------|-----------|---------------|----------------|
| 0–15 | Complacency | Tight ranges, slow trends | Trend following, long calls | 1.0x |
| 15–25 | Normal | Standard swings, levels respected | Gap and Go, ORB | 0.8x |
| 25–30 | Elevated | Wide swings, sharp reversals | Scalping, mean reversion | 0.5x |
| 30+ | Panic | Massive ranges, stophunting | Contrarian bounces only | 0.25x |

```python
from enum import Enum

class VolatilityRegime(Enum):
    COMPLACENCY = "complacency"   # VIX 0-15
    NORMAL = "normal"             # VIX 15-25
    ELEVATED = "elevated"         # VIX 25-30
    PANIC = "panic"               # VIX 30+

@dataclass
class RegimeDetector:
    thresholds: list[float] = None  # [15, 25, 30]
    multipliers: list[float] = None  # [1.0, 0.8, 0.5, 0.25]

    def __post_init__(self):
        self.thresholds = self.thresholds or [15.0, 25.0, 30.0]
        self.multipliers = self.multipliers or [1.0, 0.8, 0.5, 0.25]

    def detect(self, vix: float) -> VolatilityRegime:
        if vix < self.thresholds[0]:
            return VolatilityRegime.COMPLACENCY
        elif vix < self.thresholds[1]:
            return VolatilityRegime.NORMAL
        elif vix < self.thresholds[2]:
            return VolatilityRegime.ELEVATED
        else:
            return VolatilityRegime.PANIC

    def position_multiplier(self, vix: float) -> float:
        regime = self.detect(vix)
        return self.multipliers[list(VolatilityRegime).index(regime)]

    def on_regime_change(self, old: VolatilityRegime, new: VolatilityRegime):
        """Trigger parameter updates when regime transitions."""
        logger.warning(
            f"REGIME CHANGE: {old.value} -> {new.value}. "
            f"Adjusting position sizes, stops, and strategy selection."
        )
        # Update all active strategies with new regime parameters
```

**Red Flag:** Same position size regardless of VIX regime. At VIX 35, a position sized for VIX 12 will blow through stops on normal intraday noise.

---

## 3. The Rule of 16: Expected Daily Move

The VIX is annualized. To get the **expected daily move**, divide by 16 (the square root of ~256 trading days).

```
expected_daily_move_pct = VIX / 16
```

| VIX | Expected Daily Move | SPY at $600 | Stop Must Be > |
|-----|-------------------|-------------|----------------|
| 12 | 0.75% | $4.50 | $4.50 |
| 20 | 1.25% | $7.50 | $7.50 |
| 32 | 2.00% | $12.00 | $12.00 |
| 50 | 3.13% | $18.75 | $18.75 |

```python
@dataclass
class ExpectedMoveCalculator:
    rule_of_16_divisor: float = 16.0

    def daily_move_pct(self, vix: float) -> float:
        """Expected daily move as a percentage."""
        return vix / self.rule_of_16_divisor

    def daily_move_dollars(self, vix: float, spy_price: float) -> float:
        """Expected daily move in dollar terms."""
        return spy_price * (self.daily_move_pct(vix) / 100)

    def minimum_stop_distance(self, vix: float, spy_price: float,
                               stop_multiple: float = 1.0) -> float:
        """Stop must be at least this far from entry to survive daily noise."""
        return self.daily_move_dollars(vix, spy_price) * stop_multiple


class VolatilityAdjustedStop:
    def __init__(self, calc: ExpectedMoveCalculator):
        self.calc = calc

    def validate_stop(self, entry: float, stop: float,
                      vix: float, spy_price: float) -> bool:
        """Reject stops that are inside expected daily noise."""
        min_distance = self.calc.minimum_stop_distance(vix, spy_price)
        actual_distance = abs(entry - stop)
        if actual_distance < min_distance:
            logger.warning(
                f"Stop too tight: {actual_distance:.2f} < "
                f"expected daily move {min_distance:.2f} (VIX={vix})"
            )
            return False
        return True
```

**Iron Rule:** A stop-loss placed inside the expected daily noise (VIX/16) is a guaranteed stop-out. It's not risk management — it's donating money to the market.

---

## 4. Gap Trading Mechanics

A gap is a price discontinuity between yesterday's close and today's open. Gaps provide the day's liquidity roadmap.

### Gap Classification

```python
@dataclass
class GapClassification:
    gap_pct: float
    gap_type: str  # "full_up", "partial_up", "full_down", "partial_down"
    fill_probability: float
    has_catalyst: bool

def classify_gap(
    today_open: float,
    yesterday_close: float,
    yesterday_high: float,
    yesterday_low: float,
) -> GapClassification:
    gap_pct = ((today_open - yesterday_close) / yesterday_close) * 100

    if today_open > yesterday_high:
        gap_type = "full_up"
    elif today_open > yesterday_close:
        gap_type = "partial_up"
    elif today_open < yesterday_low:
        gap_type = "full_down"
    else:
        gap_type = "partial_down"

    # Fill probability inversely proportional to gap size
    abs_gap = abs(gap_pct)
    if abs_gap < 1.0:
        fill_prob = 0.60
    elif abs_gap < 2.0:
        fill_prob = 0.46  # ~45-47% historically
    else:
        fill_prob = 0.32  # ~30-33% for large gaps

    return GapClassification(
        gap_pct=gap_pct,
        gap_type=gap_type,
        fill_probability=fill_prob,
        has_catalyst=False,  # Set by caller after news check
    )
```

### Gap Trading Rules

| Gap Type | Fill Probability | Strategy |
|----------|-----------------|----------|
| Full Gap Up (> yesterday high) | Lower (~32%) | Gap and Go momentum — don't fade |
| Partial Gap Up | Higher (~46%) | Likely fills before trending — wait for fill or breakout |
| Full Gap Down (< yesterday low) | Lower (~32%) | Fear-driven — potential bounce or continuation |
| Partial Gap Down | Higher (~46%) | Mean reversion to previous close likely |

**Critical:** Gaps without a catalyst (earnings, news, Fed) are "exhaustion gaps" — prone to reversal. Always verify the catalyst before trading a gap.

---

## 5. Opening Range Breakout (ORB)

### The 5-Minute ORB Protocol

```python
from datetime import time
from zoneinfo import ZoneInfo

ET = ZoneInfo("America/New_York")

@dataclass
class ORBSetup:
    window_minutes: int = 5  # 5, 15, or 30
    orb_high: float = None
    orb_low: float = None
    volume_threshold: float = 1.5  # 150% of average

    def mark_range(self, bars: list[Bar]) -> None:
        """Mark the opening range from the first N minutes."""
        open_time = time(9, 30)
        end_time = time(9, 30 + self.window_minutes)  # Simplified
        orb_bars = [b for b in bars if open_time <= b.time_et <= end_time]
        self.orb_high = max(b.high for b in orb_bars)
        self.orb_low = min(b.low for b in orb_bars)

    def check_breakout(self, price: float, volume: float,
                        avg_volume: float, vix_direction: str) -> str | None:
        """Check for ORB breakout with volume and VIX confirmation."""
        if volume < avg_volume * self.volume_threshold:
            return None  # No volume confirmation

        if price > self.orb_high:
            if vix_direction == "falling":
                return "LONG"  # VIX falling + break up = bullish
            return "LONG_WEAK"  # Break up but VIX not confirming
        elif price < self.orb_low:
            if vix_direction == "rising":
                return "SHORT"  # VIX rising + break down = bearish
            return "SHORT_WEAK"

        return None  # Inside range
```

### ORB Rules

1. Mark high/low of first 5 minutes (9:30–9:35 ET)
2. Entry: break above high OR below low with **150%+ volume** of 10-day average
3. VIX confirmation: falling VIX + upside break = strong; rising VIX + downside break = strong
4. Stop: just inside the opening range (ORB low for longs, ORB high for shorts)
5. Target: 2:1 reward-to-risk ratio
6. Time limit: resolve within **92 minutes** to avoid 0DTE theta acceleration
7. Abort: if no breakout by 10:30 AM, the ORB has failed — move on

**Backtest reference:** 303-trade 0DTE SPY study (2023-2025): 41.3% win rate, 2:1 payoff, 59% net profit on $25k account.

---

## 6. Institutional Order Flow Analysis

Institutions break large orders into child orders to hide their intent. Detecting these reveals "smart money" positioning.

```python
@dataclass
class FlowSignal:
    timestamp: datetime
    symbol: str
    side: str  # "buy_aggression" or "sell_aggression"
    size: int  # Reconstructed institutional size
    price: float
    level_type: str  # "support", "resistance", "vwap", "gap_fill"

class OrderFlowAnalyzer:
    def __init__(self, big_trade_threshold: int = 10000):
        self.threshold = big_trade_threshold
        self.flow_buffer: list[FlowSignal] = []

    def detect_absorption(self, trades: list[Trade]) -> str | None:
        """Detect iceberg/absorption patterns.

        Massive selling but price refuses to drop = hidden buyer absorbing.
        Massive buying but price refuses to rise = hidden seller distributing.
        """
        sell_volume = sum(t.size for t in trades if t.side == "sell")
        buy_volume = sum(t.size for t in trades if t.side == "buy")
        price_change = trades[-1].price - trades[0].price

        if sell_volume > buy_volume * 2 and price_change >= 0:
            return "BULLISH_ABSORPTION"  # Hidden buyer
        if buy_volume > sell_volume * 2 and price_change <= 0:
            return "BEARISH_DISTRIBUTION"  # Hidden seller
        return None

    def detect_big_trades(self, trades: list[Trade]) -> list[FlowSignal]:
        """Reconstruct institutional orders from child order clusters."""
        signals = []
        # Group trades by price level within a time window
        # Aggregate child orders back into institutional blocks
        # Flag as buy_aggression (hitting ask) or sell_aggression (hitting bid)
        return signals
```

### Flow Interpretation

| Pattern | Meaning | Action |
|---------|---------|--------|
| Buy aggression at support | Institutions defending level | Buy calls |
| Sell aggression at resistance | Institutions defending ceiling | Buy puts |
| Absorption (selling absorbed) | Hidden buyer accumulating | Strong buy signal |
| Distribution (buying absorbed) | Hidden seller distributing | Strong sell signal |
| VWAP reclaim with volume | Institutions resetting fair value | Trend continuation |

**Integration:** Flow signals feed into `strategy-signal-validation` confluence gate as the **"Institutional"** indicator family.

---

## 7. VIX Futures Term Structure

VIX futures term structure reveals institutional fear positioning.

```python
@dataclass
class VIXTermStructure:
    spot_vix: float
    front_month: float
    back_month: float

    @property
    def is_contango(self) -> bool:
        """Normal: futures > spot (institutions paying premium for protection)."""
        return self.front_month > self.spot_vix

    @property
    def is_backwardation(self) -> bool:
        """Fear regime: futures < spot (immediate fear exceeds future fear)."""
        return self.front_month < self.spot_vix

    @property
    def premium_pct(self) -> float:
        """How much institutions are over/under-paying for protection."""
        return ((self.front_month - self.spot_vix) / self.spot_vix) * 100

    def edge_signal(self) -> str | None:
        prem = self.premium_pct
        if prem > 15:
            return "BEARISH_EDGE"  # Institutions over-paying for protection
        elif prem < -10:
            return "BULLISH_EDGE"  # Fear capitulation, VIX likely to drop
        return None
```

| Structure | Meaning | SPY Bias |
|-----------|---------|----------|
| Large contango (>15% premium) | Institutions over-hedging | Bearish — buy puts |
| Normal contango (5-15%) | Standard hedging demand | Neutral |
| Flat (0-5%) | Low conviction | Neutral |
| Backwardation (<0%) | Fear capitulation | Bullish — buy calls |

---

## 8. RSI-VIX Predictive Signals

Cross-market RSI analysis predicts volatility regime changes.

```python
@dataclass
class RSIVIXPredictor:
    rsi_period: int = 2  # Short-term RSI for extremes
    spy_rsi: float = 50.0
    vix_ppo: float = 0.0  # VIX Percentage Price Oscillator

    def signal(self) -> str | None:
        # SPY RSI(2) extreme oversold -> VIX contraction likely -> buy calls
        if self.spy_rsi < 5:
            return "BUY_CALLS"  # Extreme oversold, mean reversion likely

        # VIX PPO < 5 after spike -> fear peaked -> enter long SPY
        if self.vix_ppo < 5 and self.vix_ppo > 0:
            return "ENTER_LONG"  # Fear subsiding

        # VIX PPO > -15 after being deeply negative -> vol bottoming
        if -15 < self.vix_ppo < -5:
            return "EXIT_LONGS"  # Complacency ending, volatility returning

        return None
```

| Condition | Inference | Action |
|-----------|-----------|--------|
| SPY RSI(2) < 5 | Extreme oversold | Buy SPY calls at close |
| VIX PPO < 5 (after spike) | Fear has peaked | Enter long SPY positions |
| VIX PPO > -15 (after low) | Volatility bottoming | Exit longs, consider puts |

---

## 9. Pin Risk for 0DTE

Near expiration, SPY price gravitates toward strikes with maximum open interest — "pinning."

```python
@dataclass
class PinRiskDetector:
    buffer_strikes: int = 2  # Alert within 2 strikes of max OI

    def detect_pin_risk(
        self, current_price: float, strike_oi: dict[float, int]
    ) -> float | None:
        """Identify the max OI strike that price may pin to."""
        if not strike_oi:
            return None
        max_oi_strike = max(strike_oi, key=strike_oi.get)
        distance = abs(current_price - max_oi_strike)
        strike_width = 1.0  # SPY has $1 strikes

        if distance <= strike_width * self.buffer_strikes:
            return max_oi_strike  # Price is near the pin
        return None
```

**Danger:** If you're short a 0DTE option and SPY closes AT your strike, you face assignment → overnight stock position → gap risk the next morning. Close 0DTE positions 30+ minutes before close (see `eod-liquidation` skill).

---

## 10. VIX/SPY Hedged Strategy

A long-volatility play using simultaneous SPY + VIX options.

**When to deploy:** VIX term structure at extreme premium (>15%) or extreme discount (<-10%).

| Market Move | SPY Calls | VIX Calls | Net Result |
|-------------|-----------|-----------|------------|
| SPY rallies big | Large gain | Small loss (premium) | Net profit |
| SPY crashes big | Loss | Massive gain (VIX spikes 2-3x) | Net profit |
| SPY flat | Theta loss | Theta loss | Net loss |

**Key insight:** VIX has positive skew — it spikes harder than it drops. A VIX call can gain 200-500% on a crash, more than offsetting SPY call losses. The strategy loses only when nothing moves.

---

## 11. Volatility-Adjusted Position Sizing

Replace discrete tiers with continuous sizing based on Rule of 16.

```python
@dataclass
class VolatilityAdjustedSizer:
    account_equity: Decimal
    risk_per_trade_pct: Decimal = Decimal("0.01")  # 1%
    rule_of_16: float = 16.0

    def calculate_shares(
        self, vix: float, spy_price: float, stop_multiple: float = 1.0
    ) -> int:
        """Size position so max loss = risk_pct of account."""
        expected_move = spy_price * (vix / self.rule_of_16 / 100)
        stop_distance = expected_move * stop_multiple

        if stop_distance <= 0:
            return 0

        risk_dollars = float(self.account_equity * self.risk_per_trade_pct)
        shares = int(risk_dollars / stop_distance)
        return max(shares, 0)

    def calculate_contracts(
        self, vix: float, option_price: float, multiplier: int = 100
    ) -> int:
        """For options: ensure total premium <= risk budget."""
        risk_dollars = float(self.account_equity * self.risk_per_trade_pct)
        max_premium = risk_dollars
        contracts = int(max_premium / (option_price * multiplier))
        return max(contracts, 0)
```

| VIX | Expected Move | Shares (on $100k, 1% risk, $600 SPY) | Logic |
|-----|---------------|--------------------------------------|-------|
| 12 | $4.50 | 222 shares | Tight range → more shares OK |
| 20 | $7.50 | 133 shares | Normal → standard size |
| 32 | $12.00 | 83 shares | Wide range → fewer shares |
| 50 | $18.75 | 53 shares | Extreme → small position |

---

## 12. Configuration

```toml
[spy_vix_regime]
# Regime detection
regime_thresholds = [15, 25, 30]
position_multipliers = [1.0, 0.8, 0.5, 0.25]

# Rule of 16
rule_of_16_divisor = 16
stop_multiple = 1.0  # Stop at 1x expected daily move

# Gap trading
gap_min_pct = 0.5
gap_catalyst_required = true

# ORB
orb_window_minutes = 5
orb_volume_threshold = 1.5  # 150% of average
orb_max_hold_minutes = 92

# RSI-VIX
rsi_extreme_oversold = 5
vix_ppo_fear_peak = 5
vix_ppo_vol_bottom = -15

# Pin risk
pin_risk_buffer_strikes = 2
pin_risk_exit_minutes_before_close = 30

# Flow analysis
big_trade_threshold_shares = 10000
absorption_volume_ratio = 2.0
```

---

## Red Flags - STOP

| Red Flag | Why It's Dangerous |
|----------|-------------------|
| Stop-loss inside VIX/16 expected move | Guaranteed stop-out by normal noise |
| Same position size at VIX 12 and VIX 35 | 3x more volatile but same exposure |
| Trading gap without catalyst verification | Exhaustion gaps reverse violently |
| ORB entry without volume confirmation | False breakouts trap traders |
| Ignoring VIX regime for strategy selection | Trend-following in VIX 35 = disaster |
| No regime transition parameter update | Old parameters in new regime = wrong sizing |
| Holding 0DTE at pin risk strike near close | Assignment → overnight gap exposure |
| Fading a full gap without statistical edge | Full gaps fill only 30% of the time |
| VIX term structure ignored for hedging | Missing the institutional fear signal |

## Common Rationalizations

| Excuse | Reality |
|--------|---------|
| "VIX is just for risk management" | VIX is the primary context for ALL decisions |
| "I don't need regime detection" | Same strategy in different regimes = inconsistent P&L |
| "Gaps always fill" | Only 30-47% fill. Size and type matter. |
| "ORB works every day" | 41% win rate. Need 2:1 payoff to profit. |
| "I'll just use fixed position sizes" | Fixed sizes in variable volatility = variable risk |

---

## Integration

| Scenario | Related Skill |
|----------|---------------|
| ORB/flow signals need confirmation | `trading-bot-skills:strategy-signal-validation` (confluence gate) |
| Position sizing with volatility | `trading-bot-skills:risk-management-gates` |
| VIX data freshness | `trading-bot-skills:market-data-pipeline` |
| 0DTE pin risk near close | `trading-bot-skills:eod-liquidation` |
| 0DTE gamma at expiration | `trading-bot-skills:options-trading-safety` |
| RSI/VWAP math correctness | `trading-bot-skills:indicator-math-verification` |
| Market hours for ORB timing | `trading-bot-skills:pre-trade-validation` |
| Flow signals as "Institutional" family | `trading-bot-skills:strategy-signal-validation` (confluence families) |
