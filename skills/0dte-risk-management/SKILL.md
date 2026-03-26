---
name: 0dte-risk-management
description: Use when trading zero-days-to-expiration options, managing 0DTE gamma risk, implementing auto-exit rules near expiration, or handling SPY/SPX settlement differences
---

# 0DTE Risk Management

**Iron Law: "NO 0DTE POSITION WITHOUT AUTOMATED EXIT RULES. GAMMA IS INFINITE AT EXPIRATION."**

0DTE options are a fundamentally different instrument from options with days or weeks remaining. Theta is non-linear and accelerates violently. Liquidity evaporates in the final minutes. Gamma makes delta meaningless as a hedge metric. This skill covers the mechanics that make 0DTE uniquely dangerous and the automated rules required to survive them.

> **What this skill does NOT cover** (handled elsewhere):
> - Gamma explosion math -- see `options-trading-safety`
> - Assignment detection -- see `options-trading-safety`
> - DTE=0 emergency thresholds -- see `options-trading-safety`
> - Pin risk near max OI strikes -- see `spy-vix-regime-trading`

---

## 1. Time-Decay Acceleration Mechanics

Theta is not linear. On expiration day, theta accelerates dramatically in the final hours. A position that decays $0.02/hour at 10am can decay $0.15/hour at 3:30pm.

### Theta Decay Table by Time Remaining

| Time to Expiry | Approx. Theta (ATM, per hour) | Cumulative Decay (% of remaining value) | Risk Level |
|----------------|-------------------------------|------------------------------------------|------------|
| 4 hours        | 1x baseline                   | ~15%                                     | ELEVATED   |
| 2 hours        | 2x baseline                   | ~30%                                     | HIGH       |
| 1 hour         | 4x baseline                   | ~50%                                     | CRITICAL   |
| 30 min         | 8x baseline                   | ~70%                                     | EXTREME    |
| 15 min         | 16x baseline                  | ~85%                                     | EMERGENCY  |
| 5 min          | 30x+ baseline                 | ~95%                                     | EXIT NOW   |

The "baseline" is approximate ATM theta at market open for a 0DTE option. Exact values depend on IV and underlying price.

```python
from dataclasses import dataclass
from enum import Enum
from datetime import datetime, time as dtime
from zoneinfo import ZoneInfo
from typing import Optional

ET = ZoneInfo("America/New_York")


class RiskLevel(Enum):
    NORMAL = "NORMAL"
    ELEVATED = "ELEVATED"
    HIGH = "HIGH"
    CRITICAL = "CRITICAL"
    EXTREME = "EXTREME"
    EMERGENCY = "EMERGENCY"


@dataclass(frozen=True)
class RiskDecision:
    allow: bool
    reason: str
    risk_level: RiskLevel
    max_position_pct: float  # 0.0 to 1.0, fraction of normal limit


@dataclass
class ThetaAccelerationMonitor:
    """Adjusts position limits based on time remaining to expiration.

    Fail-closed: any error returns a deny decision with zero position allowance.
    """

    market_close: dtime = dtime(16, 0)
    # Thresholds: (minutes_remaining, max_position_pct, risk_level)
    THRESHOLDS: list[tuple[int, float, RiskLevel]] = None

    def __post_init__(self):
        if self.THRESHOLDS is None:
            self.THRESHOLDS = [
                (240, 1.0, RiskLevel.ELEVATED),
                (120, 0.75, RiskLevel.HIGH),
                (60,  0.50, RiskLevel.CRITICAL),
                (30,  0.25, RiskLevel.EXTREME),
                (15,  0.0,  RiskLevel.EMERGENCY),  # No new positions
            ]

    def minutes_to_close(self) -> float:
        now = datetime.now(ET)
        close_dt = now.replace(
            hour=self.market_close.hour,
            minute=self.market_close.minute,
            second=0, microsecond=0,
        )
        return (close_dt - now).total_seconds() / 60.0

    def evaluate(self) -> RiskDecision:
        try:
            mins = self.minutes_to_close()
            if mins <= 0:
                return RiskDecision(
                    allow=False, reason="Market closed",
                    risk_level=RiskLevel.EMERGENCY, max_position_pct=0.0,
                )
            for threshold_mins, max_pct, level in self.THRESHOLDS:
                if mins <= threshold_mins:
                    return RiskDecision(
                        allow=max_pct > 0,
                        reason=f"{mins:.0f}min to close, limit={max_pct*100:.0f}%",
                        risk_level=level,
                        max_position_pct=max_pct,
                    )
            return RiskDecision(
                allow=True, reason="Sufficient time remaining",
                risk_level=RiskLevel.NORMAL, max_position_pct=1.0,
            )
        except Exception as e:
            return RiskDecision(
                allow=False, reason=f"ThetaAccelerationMonitor error: {e}",
                risk_level=RiskLevel.EMERGENCY, max_position_pct=0.0,
            )
```

---

## 2. Liquidity Degradation Near Expiration

In the final 30 minutes of trading, bid-ask spreads on 0DTE options widen dramatically -- often 2-5x normal. OTM options may have no real bids at all. This means your "limit order to close" may never fill.

### Exit Priority Rules

1. **Close ATM positions first** -- they still have liquidity and the most gamma risk.
2. **Close ITM positions second** -- intrinsic value provides a floor.
3. **Never hold OTM positions into the final 15 minutes** -- they are illiquid, near-worthless, but can explode on a late-day move.
4. **Use market orders in the final 5 minutes** -- paying the spread beats holding into close.

```python
@dataclass(frozen=True)
class SpreadSnapshot:
    symbol: str
    bid: float
    ask: float
    mid: float
    minutes_to_close: float


@dataclass
class LiquidityMonitor:
    """Tracks bid-ask spread degradation and triggers forced exits.

    Fail-closed: errors force exit recommendation.
    """

    # If spread exceeds this multiple of the mid price, trigger exit
    max_spread_pct: float = 0.10  # 10% of mid
    # If spread exceeds this multiple of the morning baseline, trigger exit
    max_spread_ratio_vs_baseline: float = 3.0
    # Absolute cutoff: force exit regardless of spread
    force_exit_minutes: float = 15.0

    def __init__(self, max_spread_pct: float = 0.10,
                 max_spread_ratio_vs_baseline: float = 3.0,
                 force_exit_minutes: float = 15.0):
        self.max_spread_pct = max_spread_pct
        self.max_spread_ratio_vs_baseline = max_spread_ratio_vs_baseline
        self.force_exit_minutes = force_exit_minutes
        self._baselines: dict[str, float] = {}

    def record_baseline(self, symbol: str, spread: float) -> None:
        """Call at market open to establish normal spread."""
        self._baselines[symbol] = spread

    def evaluate(self, snap: SpreadSnapshot) -> RiskDecision:
        try:
            spread = snap.ask - snap.bid
            if snap.minutes_to_close <= self.force_exit_minutes:
                return RiskDecision(
                    allow=False,
                    reason=f"<{self.force_exit_minutes}min to close, force exit",
                    risk_level=RiskLevel.EMERGENCY, max_position_pct=0.0,
                )
            if snap.mid <= 0:
                return RiskDecision(
                    allow=False, reason="Mid price <= 0, no valid quote",
                    risk_level=RiskLevel.EMERGENCY, max_position_pct=0.0,
                )
            spread_pct = spread / snap.mid
            if spread_pct > self.max_spread_pct:
                return RiskDecision(
                    allow=False,
                    reason=f"Spread {spread_pct:.1%} exceeds {self.max_spread_pct:.1%} limit",
                    risk_level=RiskLevel.EXTREME, max_position_pct=0.0,
                )
            baseline = self._baselines.get(snap.symbol)
            if baseline and baseline > 0:
                ratio = spread / baseline
                if ratio > self.max_spread_ratio_vs_baseline:
                    return RiskDecision(
                        allow=False,
                        reason=f"Spread {ratio:.1f}x vs baseline (limit {self.max_spread_ratio_vs_baseline}x)",
                        risk_level=RiskLevel.CRITICAL, max_position_pct=0.0,
                    )
            return RiskDecision(
                allow=True, reason="Liquidity acceptable",
                risk_level=RiskLevel.NORMAL, max_position_pct=1.0,
            )
        except Exception as e:
            return RiskDecision(
                allow=False, reason=f"LiquidityMonitor error: {e}",
                risk_level=RiskLevel.EMERGENCY, max_position_pct=0.0,
            )
```

---

## 3. 0DTE Spread Management

### Partially-Filled Spreads

A partially-filled 0DTE spread is a **naked position**. If you sell a put spread and only the short leg fills, you are naked short a put with hours to go. Rules:

- **If <90 minutes to expiry and only one leg fills**: cancel the unfilled leg and immediately close the filled leg. Do not wait for the other side.
- **If spread is fully filled but one leg is now deep ITM**: evaluate early close of the entire spread rather than holding to expiry.

### Close Rules: 0DTE vs 30DTE

| Decision Factor          | 30DTE Spread              | 0DTE Spread                   |
|--------------------------|---------------------------|-------------------------------|
| Profit target            | 50-65% of max profit      | 25-40% of max profit          |
| Stop loss                | 2x credit received        | 1.5x credit received          |
| Time-based close         | Optional                  | MANDATORY before final 15min  |
| Partial fill handling    | Wait up to 5 min          | Close immediately             |
| Roll decision            | Roll to next month        | Do not roll 0DTE -- just close|

### Early Close vs Hold-to-Expiry Decision

Hold to expiry ONLY if ALL of the following are true:
1. Position is >80% of max profit already captured.
2. Underlying is >2 standard deviations from short strike.
3. More than 30 minutes remain.
4. Bid-ask spread on closing trade is <5% of remaining credit.

If any condition fails, close early. The marginal gain from holding is almost never worth the tail risk.

---

## 4. SPY Physical vs SPX Cash Settlement

This is the single most common source of 0DTE settlement surprises.

### Comparison Table

| Feature               | SPY                              | SPX                              |
|-----------------------|----------------------------------|----------------------------------|
| Settlement type       | **Physical** (shares delivered)  | **Cash** (cash difference paid)  |
| Exercise style        | **American** (any time)          | **European** (at expiry only)    |
| Assignment risk       | YES -- can be assigned early     | NO -- cannot be assigned early   |
| Settlement timing     | PM settlement (closing price)    | **AM settlement** (Mon/Wed/Fri standard) or PM (0DTE specific) |
| After-hours risk      | YES -- shares held after assign  | NO -- cash settles at close      |
| Weekend gap exposure  | YES -- Friday assignment = hold shares over weekend | NO |
| Tax treatment         | Standard capital gains           | **60/40** (60% long-term, 40% short-term) |
| Contract multiplier   | 100 shares                       | $100 x index                     |

### Critical SPY Danger: Friday Assignment

If you sell a SPY 0DTE put spread on Friday and the short put is ITM at close:
1. You get assigned 100 shares per contract **after market close**.
2. You hold those shares over the weekend with **no hedge**.
3. Monday gap risk is entirely unhedged.

SPX avoids this entirely because it cash-settles.

```python
@dataclass(frozen=True)
class SettlementProfile:
    is_physical: bool
    is_american: bool
    has_assignment_risk: bool
    has_weekend_gap_risk: bool
    settlement_time: str  # "AM" or "PM"


@dataclass
class SettlementRiskClassifier:
    """Classifies settlement risk for 0DTE positions.

    Fail-closed: unknown symbols are treated as physical/American.
    """

    PROFILES: dict[str, SettlementProfile] = None

    def __post_init__(self):
        if self.PROFILES is None:
            self.PROFILES = {
                "SPY": SettlementProfile(
                    is_physical=True, is_american=True,
                    has_assignment_risk=True, has_weekend_gap_risk=True,
                    settlement_time="PM",
                ),
                "SPX": SettlementProfile(
                    is_physical=False, is_american=False,
                    has_assignment_risk=False, has_weekend_gap_risk=False,
                    settlement_time="PM",  # For 0DTE specifically
                ),
                "QQQ": SettlementProfile(
                    is_physical=True, is_american=True,
                    has_assignment_risk=True, has_weekend_gap_risk=True,
                    settlement_time="PM",
                ),
                "XSP": SettlementProfile(
                    is_physical=False, is_american=False,
                    has_assignment_risk=False, has_weekend_gap_risk=False,
                    settlement_time="PM",
                ),
            }

    def classify(self, symbol: str, is_friday: bool = False) -> RiskDecision:
        try:
            profile = self.PROFILES.get(symbol.upper())
            if profile is None:
                # Fail-closed: treat unknown as worst case
                return RiskDecision(
                    allow=False,
                    reason=f"Unknown symbol {symbol}, assuming physical settlement",
                    risk_level=RiskLevel.CRITICAL, max_position_pct=0.25,
                )
            if profile.has_weekend_gap_risk and is_friday:
                return RiskDecision(
                    allow=False,
                    reason=f"{symbol} is physically settled -- Friday 0DTE creates weekend gap risk",
                    risk_level=RiskLevel.EXTREME, max_position_pct=0.0,
                )
            if profile.has_assignment_risk:
                return RiskDecision(
                    allow=True,
                    reason=f"{symbol} has assignment risk -- monitor closely",
                    risk_level=RiskLevel.HIGH, max_position_pct=0.50,
                )
            return RiskDecision(
                allow=True,
                reason=f"{symbol} is cash-settled, no assignment risk",
                risk_level=RiskLevel.NORMAL, max_position_pct=1.0,
            )
        except Exception as e:
            return RiskDecision(
                allow=False, reason=f"SettlementRiskClassifier error: {e}",
                risk_level=RiskLevel.EMERGENCY, max_position_pct=0.0,
            )
```

---

## 5. Auto-Exit Proximity Rules

If the underlying is within a configurable distance of your short strike, you MUST auto-close before the bell. Near expiry, gamma makes delta useless -- a $0.50 move can flip a position from safe to max loss.

```python
@dataclass
class ProximityExitMonitor:
    """Monitors distance between underlying and short strikes.

    Triggers auto-close when underlying is dangerously close to short strike.
    Fail-closed: errors trigger exit.
    """

    # Proximity thresholds: (minutes_remaining, max_distance_dollars)
    THRESHOLDS: list[tuple[int, float]] = None

    def __post_init__(self):
        if self.THRESHOLDS is None:
            self.THRESHOLDS = [
                (120, 1.00),  # >2hrs: alert if within $1.00
                (60,  1.50),  # >1hr: alert if within $1.50
                (30,  2.00),  # >30min: alert if within $2.00
                (15,  3.00),  # >15min: exit if within $3.00
                (5,   5.00),  # >5min: exit if within $5.00
                (0,   999.0), # At close: exit everything
            ]

    def evaluate(
        self, underlying_price: float, short_strike: float,
        minutes_to_close: float,
    ) -> RiskDecision:
        try:
            distance = abs(underlying_price - short_strike)

            # Walk thresholds from tightest time to widest
            for min_minutes, max_distance in self.THRESHOLDS:
                if minutes_to_close <= min_minutes:
                    continue
                # Found the matching bucket
                if distance <= max_distance:
                    return RiskDecision(
                        allow=False,
                        reason=(
                            f"Underlying ${underlying_price:.2f} is ${distance:.2f} "
                            f"from short strike ${short_strike:.2f} with "
                            f"{minutes_to_close:.0f}min remaining (limit ${max_distance:.2f})"
                        ),
                        risk_level=RiskLevel.EXTREME if minutes_to_close < 30
                            else RiskLevel.CRITICAL,
                        max_position_pct=0.0,
                    )
                return RiskDecision(
                    allow=True,
                    reason=f"Distance ${distance:.2f} > ${max_distance:.2f} threshold",
                    risk_level=RiskLevel.NORMAL, max_position_pct=1.0,
                )

            # Fallback: at or past close
            return RiskDecision(
                allow=False, reason="At or past close -- exit all",
                risk_level=RiskLevel.EMERGENCY, max_position_pct=0.0,
            )
        except Exception as e:
            return RiskDecision(
                allow=False, reason=f"ProximityExitMonitor error: {e}",
                risk_level=RiskLevel.EMERGENCY, max_position_pct=0.0,
            )
```

---

## 6. Expected Value Per Minute Framework

Every 0DTE position has an implicit cost: theta bleed per minute. If your edge (expected profit per minute from favorable delta moves) is less than theta cost, the position is -EV and should be closed.

**Formula:** `EV/min = (prob_win * avg_win - prob_loss * avg_loss) / holding_minutes - theta_per_minute`

When `EV/min < 0`, exit immediately.

```python
@dataclass(frozen=True)
class EVSnapshot:
    prob_win: float          # 0.0 to 1.0
    avg_win_dollars: float   # Expected gain if trade wins
    avg_loss_dollars: float  # Expected loss if trade loses (positive number)
    theta_per_minute: float  # Dollars lost per minute to decay (positive number)
    minutes_remaining: float


@dataclass
class EVPerMinuteCalculator:
    """Calculates whether holding a 0DTE position is +EV.

    Fail-closed: errors return negative EV (exit).
    """

    min_ev_per_minute: float = 0.0  # Minimum EV/min to justify holding

    def evaluate(self, snap: EVSnapshot) -> RiskDecision:
        try:
            if snap.minutes_remaining <= 0:
                return RiskDecision(
                    allow=False, reason="No time remaining",
                    risk_level=RiskLevel.EMERGENCY, max_position_pct=0.0,
                )
            raw_edge = (
                snap.prob_win * snap.avg_win_dollars
                - (1 - snap.prob_win) * snap.avg_loss_dollars
            )
            ev_per_minute = (raw_edge / snap.minutes_remaining) - snap.theta_per_minute

            if ev_per_minute < self.min_ev_per_minute:
                return RiskDecision(
                    allow=False,
                    reason=(
                        f"EV/min=${ev_per_minute:.4f} < ${self.min_ev_per_minute:.4f} "
                        f"(theta bleeds ${snap.theta_per_minute:.4f}/min, "
                        f"edge=${raw_edge:.2f} over {snap.minutes_remaining:.0f}min)"
                    ),
                    risk_level=RiskLevel.HIGH, max_position_pct=0.0,
                )
            return RiskDecision(
                allow=True,
                reason=f"EV/min=${ev_per_minute:.4f}, position is +EV",
                risk_level=RiskLevel.NORMAL, max_position_pct=1.0,
            )
        except Exception as e:
            return RiskDecision(
                allow=False, reason=f"EVPerMinuteCalculator error: {e}",
                risk_level=RiskLevel.EMERGENCY, max_position_pct=0.0,
            )
```

---

## 7. Volatility Surface Dynamics on Expiration Day

The volatility surface behaves anomalously on 0DTE:

- **Skew compression**: The normally steep put skew flattens because OTM puts have almost no time value left. There is no extrinsic value to distribute asymmetrically.
- **IV crush timing**: IV does not drop smoothly to zero. It can spike intraday on sudden moves and then collapse in the final hour. The "crush" is not a single event -- it is a step function tied to actual realized volatility.
- **Straddle pricing anomalies**: ATM straddles on 0DTE can appear cheap because they price only the remaining intraday move. But realized vol intraday often exceeds implied, especially around 2pm-3pm ET. This creates a repeatable pattern where straddles are underpriced in the 1-2pm window on high-vol days.
- **Gamma-vega inversion**: On 0DTE, gamma dominates vega entirely. A 1% IV change moves the option price far less than a $0.50 move in the underlying. Greeks models that weight vega heavily will misjudge risk.

**Practical rule**: Do not rely on IV-based signals for 0DTE entry/exit. Use delta and proximity to strike, not implied volatility.

---

## Red Flags

| Red Flag | Why It's Dangerous |
|----------|-------------------|
| Holding 0DTE past 3:45pm ET without auto-exit rules | Gamma makes any position a coin flip in the final 15 minutes |
| Selling 0DTE SPY spreads on Friday | Physical settlement + weekend gap = unhedged equity position |
| Widening bid-ask spread ignored | You may not be able to exit at any reasonable price |
| Partially filled 0DTE spread held >5 minutes | You are naked on one leg with hours to expiry |
| Using 30DTE profit targets on 0DTE | 50% profit target on 0DTE means holding through the danger zone |
| Relying on IV signals for 0DTE decisions | Gamma dominates vega on 0DTE; IV changes barely move price |
| No EV/min calculation before holding through final hour | Theta bleed may exceed any realistic edge |
| Opening new 0DTE positions after 2:30pm ET | Insufficient time to manage, theta acceleration is severe |
| Treating SPY and SPX as interchangeable | Completely different settlement, assignment, and tax treatment |
| Manual close plan instead of automated rules | "I'll close it if it moves" fails when the move is instant |

---

## Integration Points

- **trading-bot-skills:options-trading-safety** -- Gamma explosion math, assignment detection, DTE=0 emergency thresholds. This skill extends those foundations with 0DTE-specific time mechanics.
- **trading-bot-skills:risk-management-gates** -- Position sizing gates should incorporate `ThetaAccelerationMonitor` output to reduce limits as expiry approaches.
- **trading-bot-skills:kill-switch-and-circuit-breakers** -- Circuit breakers must trigger faster on 0DTE. A 2% portfolio loss in 5 minutes on 0DTE is a kill-switch event, not a warning.
- **trading-bot-skills:spy-vix-regime-trading** -- VIX regime determines baseline 0DTE risk. In VIX>25 regimes, 0DTE position sizes should be halved or eliminated.
- **trading-bot-skills:eod-liquidation** -- EOD flatten for 0DTE must trigger earlier (30min before close vs 15min for equities). `ProximityExitMonitor` feeds into EOD liquidation priority.
