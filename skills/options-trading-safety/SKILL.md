---
name: options-trading-safety
description: Use when trading options, handling expiration, Greeks, DTE management, or when options positions expire without proper handling
---

# Options Trading Safety

## Purpose

Options have failure modes that equity-only systems never encounter. Expiring options create ghost positions. Assignments happen unexpectedly. Spread legs mismatch. This skill prevents the most dangerous options-specific failures that can result in unhedged risk and catastrophic loss.

## The Core Problem: Ghost Positions

When options expire, several things happen silently:
- **In-the-money options auto-exercise**, creating equity positions you did not explicitly request.
- **Short options get assigned**, creating equity positions at the worst possible time.
- **Exercise/assignment settles over the weekend**, so your tracker removes the option but the equity has not appeared yet.

If your system does not handle these cases, you will have **ghost positions**: real positions at your broker that your system does not know about. Ghost positions mean unhedged risk, incorrect P&L, false reconciliation, and potential catastrophic loss.

---

## DTE Management

### Track DTE for Every Options Position

Every options position MUST have its expiration date tracked and its DTE (Days to Expiration) recalculated on every evaluation cycle.

```
Position: AAPL 150C 2025-03-21
Current Date: 2025-03-16
DTE: 5 days
Status: APPROACHING EXPIRATION -- action required
```

### DTE Alert Thresholds

| DTE | Alert Level | Required Action |
|-----|-------------|-----------------|
| <= 21 days | INFO | Log: position approaching expiration |
| <= 7 days | WARNING | Alert operator: expiration week, review position |
| <= 3 days | URGENT | Auto-close or roll (configurable). Block new trades if not addressed. |
| <= 1 day | CRITICAL | Close immediately unless operator has explicitly approved hold-to-expiration |
| 0 (expiration day) | EMERGENCY | Position MUST be explicitly managed. No silent expiration. |

### Auto-Close or Roll Policy

At the minimum DTE threshold (configurable, default 3 days), the system MUST take one of these actions:

1. **Auto-close**: Submit market or limit closing order for the position.
2. **Auto-roll**: Close current position and open new position at a later expiration date.
3. **Alert and block**: Alert the operator immediately and block ALL new trades until the expiring position is addressed.

The system MUST NOT silently allow options to expire without explicit action. Every expiration must be a deliberate decision, not a passive default.

```python
# Required: DTE check on every evaluation cycle
def check_expiring_positions(options_positions, config):
    for position in options_positions:
        dte = (position.expiration_date - date.today()).days

        if dte <= 0:
            alert_operator(f"EMERGENCY: {position} EXPIRES TODAY. Immediate action required.")
            block_new_trades()
        elif dte <= config.min_dte_threshold:
            if config.auto_close_enabled:
                submit_closing_order(position)
                log(f"Auto-closing {position} at DTE={dte}")
            else:
                alert_operator(f"URGENT: {position} expires in {dte} days. Manual action required.")
                block_new_trades()
        elif dte <= 7:
            alert_operator(f"WARNING: {position} enters expiration week. DTE={dte}")
        elif dte <= 21:
            log(f"INFO: {position} approaching expiration. DTE={dte}")
```

---

## Expiration Grace Period

### The Weekend Problem

Options expiring on Friday may exercise or be assigned, but:
- Exercise/assignment notification may not arrive until Saturday.
- Resulting equity position may not settle until Monday or Tuesday (T+1).
- Your position tracker removes the option on Friday close, but the equity does not appear until Monday.

This creates a gap where your system has no record of a real position.

### Grace Period Rules

1. **Do NOT remove expired options from tracking until T+1 after expiration.** Keep them in a "grace period" state.
2. After expiration, actively check for new equity positions that match expired options.
3. If an expired option was ITM (in-the-money) at expiration, ASSUME exercise/assignment until confirmed otherwise.
4. Run reconciliation first thing Monday after any Friday expiration.

### Timeline Example

```
Friday 4:00 PM:  AAPL 150C expires. AAPL closing price: $152 (ITM).
Friday 4:01 PM:  Option disappears from broker positions.
                 Your system: option in GRACE_PERIOD state. Assumed exercised.
Saturday:        OCC processes exercise. Notification generated.
Monday AM:       100 shares AAPL appears in broker equity positions.
Monday cycle:    Your system detects 100 shares AAPL, matches against grace period.
                 Grace period resolved: confirmed exercise. Position converted to equity.
```

### Grace Period Implementation

```python
class ExpirationTracker:
    def __init__(self, grace_period_days=3):
        self.grace_positions = []  # Options in grace period
        self.grace_period_days = grace_period_days

    def on_expiration(self, option_position, underlying_price):
        """Option expired. Enter grace period. Do NOT remove from tracking."""
        itm = self._is_in_the_money(option_position, underlying_price)

        option_position.status = "EXPIRED_GRACE_PERIOD"
        option_position.assumed_exercised = itm
        option_position.grace_end = option_position.expiration_date + timedelta(
            days=self.grace_period_days
        )
        self.grace_positions.append(option_position)

        if itm:
            expected = self._expected_equity_position(option_position)
            log(f"GRACE PERIOD: {option_position} expired ITM. "
                f"Expecting {expected} to appear by {option_position.grace_end}")

    def check_grace_period(self, broker_equity_positions):
        """Run on every cycle. Check if expired options resulted in equity."""
        resolved = []
        for opt in self.grace_positions:
            expected = self._expected_equity_position(opt)

            if expected and expected.symbol in broker_equity_positions:
                # Exercise/assignment confirmed
                self._convert_to_equity(opt, broker_equity_positions[expected.symbol])
                resolved.append(opt)
                log(f"GRACE RESOLVED: {opt} exercised/assigned -> {expected}")

            elif datetime.now().date() > opt.grace_end:
                if opt.assumed_exercised:
                    # Expected exercise but no equity appeared -- ALERT
                    alert_operator(
                        f"ALERT: {opt} was ITM at expiration but no equity position appeared. "
                        f"Manual investigation required."
                    )
                else:
                    # OTM option, grace period over, expired worthless
                    log(f"GRACE RESOLVED: {opt} expired worthless. Removing from tracking.")
                resolved.append(opt)

        for r in resolved:
            self.grace_positions.remove(r)
```

---

## Greeks Monitoring

### Portfolio-Level Greeks

Track these Greeks at the portfolio level, updated every evaluation cycle:

| Greek | Measures | Risk If Unmonitored |
|-------|----------|---------------------|
| **Delta** | Directional exposure (equivalent shares) | Unknown directional risk; surprise large moves |
| **Gamma** | Rate of delta change | Sudden delta shifts near expiration |
| **Theta** | Time decay cost per day | Bleeding capital daily without noticing |
| **Vega** | Sensitivity to implied volatility | IV crush after events destroys position value |

### Portfolio Greeks Thresholds

Configure maximum acceptable portfolio-level Greeks:

```python
# Example: configure per account size
MAX_PORTFOLIO_DELTA = 500      # Max equivalent share exposure
MAX_PORTFOLIO_GAMMA = 100      # Max gamma
MAX_PORTFOLIO_THETA = -200     # Max daily theta bleed (dollars)
MAX_PORTFOLIO_VEGA = 1000      # Max vega exposure
```

When any threshold is breached:
1. **Alert operator** with current exposure vs threshold.
2. **Block new positions** that would increase the breached Greek.
3. **Suggest hedging trades** that would reduce exposure back within limits.

### Gamma Risk Near Expiration

Gamma increases dramatically for ATM (at-the-money) options as expiration approaches. A position that was manageable at 30 DTE can become extremely dangerous at 1 DTE.

**Rule**: Any position with `gamma > gamma_threshold AND DTE < 3` MUST be closed or actively hedged. Do not carry high-gamma positions into expiration unless:
- The operator explicitly approves.
- Delta hedging is active and monitored.
- Position size is small enough to absorb worst-case gamma-driven P&L swing.

### Theta Decay Awareness

Theta accelerates near expiration (theta curve is not linear). For long options:
- Track cumulative theta bleed daily.
- Alert if theta bleed exceeds a percentage of position value.
- Consider rolling positions with high theta relative to remaining value.

---

## Assignment Risk

### The Assignment Problem

Short American-style options can be assigned at ANY time before expiration. Assignment is especially likely when:
- Option is deep in-the-money.
- Option has little extrinsic (time) value remaining.
- Dividend ex-date is approaching (for short calls on dividend-paying stocks).
- Option is ITM and within 1 week of expiration.

### Assignment Detection

Your system MUST detect unexpected equity positions resulting from assignment:

```python
def detect_assignment(current_equity, previous_equity, short_options):
    """Check for new equity positions that match short option assignments."""
    for symbol in current_equity:
        current_qty = current_equity[symbol]
        previous_qty = previous_equity.get(symbol, 0)
        delta = current_qty - previous_qty

        if delta != 0:
            # Check if this matches a short option assignment
            matching_options = [
                opt for opt in short_options
                if opt.underlying == symbol
                and abs(delta) == opt.contract_size * opt.quantity
            ]
            if matching_options:
                alert_operator(
                    f"PROBABLE ASSIGNMENT: {delta:+d} shares {symbol} appeared. "
                    f"Matches short {matching_options[0]}. "
                    f"Review and confirm. DO NOT auto-trade until confirmed."
                )
                return AssignmentEvent(symbol=symbol, quantity=delta, options=matching_options)
    return None
```

### Assignment Response Protocol

1. **Alert immediately** -- do not wait for next scheduled cycle.
2. **Reconcile** -- update local position tracking with the new equity.
3. **Assess risk** -- the equity position may need hedging or may change portfolio Greeks significantly.
4. **Do NOT auto-close** the assigned equity without operator approval. The assignment may be part of an intentional strategy (e.g., covered call assigned is often acceptable).
5. **Update options tracking** -- mark the short option as assigned, not as expired.

---

## Spread Safety

### The Leg Mismatch Problem

When trading multi-leg strategies (verticals, iron condors, butterflies, calendars), ALL legs must fill or the resulting position has unintended naked exposure.

### Spread Execution Rules

1. **Use native spread/combo orders** when the broker supports them. This is the safest approach -- the broker handles atomicity.

2. **If legging in manually, the protective leg MUST fill first:**
   - Bull call spread: Buy long call FIRST, then sell short call.
   - Bear put spread: Buy long put FIRST, then sell short put.
   - Iron condor: Buy wings FIRST, then sell short strikes.
   - **Rationale**: If only one leg fills, you have a defined-risk position (long option), not a naked short.

3. **Leg mismatch timeout**: Configure maximum time between leg fills (default: 30 seconds). If exceeded, close the filled leg and abort the spread.

4. **Alert on any mismatch**: If one leg fills and the other does not fill within timeout, alert operator IMMEDIATELY.

### Leg Mismatch Detection

```python
class SpreadMonitor:
    def __init__(self, max_leg_delay_seconds=30):
        self.pending_spreads = {}
        self.max_delay = max_leg_delay_seconds

    def on_spread_initiated(self, spread_id, total_legs):
        self.pending_spreads[spread_id] = {
            "total_legs": total_legs,
            "filled_legs": [],
            "initiated_at": datetime.utcnow(),
            "timeout_at": None,
        }

    def on_leg_fill(self, spread_id, leg_number, fill):
        spread = self.pending_spreads.get(spread_id)
        if not spread:
            return

        spread["filled_legs"].append({"leg": leg_number, "fill": fill})

        if len(spread["filled_legs"]) == spread["total_legs"]:
            # All legs filled -- spread complete
            del self.pending_spreads[spread_id]
            log(f"Spread {spread_id} complete: all {spread['total_legs']} legs filled.")
        elif spread["timeout_at"] is None:
            # First leg filled, start countdown for remaining legs
            spread["timeout_at"] = datetime.utcnow() + timedelta(seconds=self.max_delay)

    def check_timeouts(self):
        """Call on every cycle. Handle spreads with missing legs."""
        now = datetime.utcnow()
        timed_out = []

        for spread_id, spread in self.pending_spreads.items():
            if spread["timeout_at"] and now > spread["timeout_at"]:
                filled = len(spread["filled_legs"])
                total = spread["total_legs"]
                alert_operator(
                    f"LEG MISMATCH: Spread {spread_id} has {filled}/{total} legs filled. "
                    f"Remaining legs timed out after {self.max_delay}s. "
                    f"CLOSING filled legs to eliminate naked exposure."
                )
                for leg_info in spread["filled_legs"]:
                    self._submit_closing_order(leg_info["fill"])
                timed_out.append(spread_id)

        for sid in timed_out:
            del self.pending_spreads[sid]
```

---

## 0DTE Options: Gamma Explosion Risk

### Gamma Explosion Math

As time to expiration tau approaches 0, ATM gamma approaches infinity:

```
gamma = N'(d1) / (S * sigma * sqrt(tau))
```

As tau approaches 0, the denominator approaches 0, and gamma approaches infinity. This means ATM 0DTE options can swing from $0 to deep ITM on a few cents of underlying movement.

### Concrete Example

SPY ATM 0DTE call at 2:00 PM: SPY moves $1, and the option swings 50-80% in value. At 3:30 PM, the same $1 move causes an 80-100% swing. The closer to expiration, the more violent the gamma effect.

### Dealer Positioning and Gamma Exposure

Market makers holding large open interest must delta-hedge continuously:

- **Put Wall**: Price level with maximum put open interest. Acts as support because dealers buy the underlying to hedge.
- **Call Wall**: Price level with maximum call open interest. Acts as resistance because dealers sell the underlying to hedge.
- **Gamma Flip Level**: When price crosses this level, dealer hedging flips from stabilizing to destabilizing.

**GEX (Gamma Exposure) Data**:
- **Positive GEX** = dealers buy dips and sell rallies (mean-reverting market behavior).
- **Negative GEX** = dealers sell dips and buy rallies (trend-amplifying market behavior).

### 0DTE-Specific Risk Gates

```python
def check_zero_dte_risk(self, signal: Signal) -> RiskDecision:
    if signal.dte == 0:
        # Reduce position size to 25% of normal
        max_size = self.config.zero_dte_max_position_pct * self.normal_max_size
        if signal.quantity > max_size:
            return RiskDecision(allowed=False, reason=f"0DTE size {signal.quantity} exceeds cap {max_size}")
        # Check portfolio-level 0DTE gamma exposure
        portfolio_gamma = self.calculate_portfolio_gamma(dte_filter=0)
        if abs(portfolio_gamma) > self.config.zero_dte_portfolio_gamma_cap:
            return RiskDecision(allowed=False, reason=f"Portfolio 0DTE gamma {portfolio_gamma} exceeds cap")
    return RiskDecision(allowed=True, reason="0DTE checks passed")
```

### 0DTE Red Flags

- Using the same position sizing for 0DTE and 30DTE options.
- No 0DTE-specific portfolio exposure cap.
- Holding 0DTE positions past 3:30 PM without an explicit exit plan.

---

## Credit Spread Risk Management

### Defined-Risk vs Undefined-Risk Classification

- **Credit spreads** (bull put spread, bear call spread, iron condor) are **defined-risk**: max loss = spread width - credit received.
- **Naked shorts** are **undefined-risk**: max loss is theoretically unlimited for calls, and strike price minus credit for puts.

### Multiple-of-Credit Stop-Loss Pattern

For credit spreads, percentage-based stops do not work well because the option value can swing dramatically relative to credit received. Instead, use the "multiple-of-credit" stop:

If you received $1.00 credit, exit when the spread value reaches $3.00 (2x credit = 200% loss on the credit received).

```python
@dataclass
class CreditSpreadStop:
    credit_received: Decimal
    stop_multiple: Decimal = Decimal("2.0")  # Exit at 2x credit

    @property
    def stop_debit(self) -> Decimal:
        return self.credit_received * (1 + self.stop_multiple)

    def should_exit(self, current_value: Decimal) -> bool:
        return current_value >= self.stop_debit
```

### Max Loss Verification Before Entry

Before entering any credit spread, calculate and verify:

```
max_loss = spread_width - credit_received
```

Verify that `max_loss` fits within the per-trade risk budget defined in risk-management-gates. If it does not fit, reduce position size or widen the spread.

### Iron Condor Management

Both sides of an iron condor are managed independently:
- If one side breaches its stop, close that side.
- The other side may remain open as a winner.
- Do NOT close the entire iron condor just because one side was stopped out.

### Configuration

```yaml
options_safety:
  # Credit Spread Settings
  credit_spread_stop_multiple: 2.0       # Exit at 2x credit received
  require_defined_risk: true             # Block naked short options
  zero_dte_max_position_pct: 0.25        # 0DTE positions capped at 25% of normal size
```

---

## Options Position Lifecycle

```
+----------+     +----------+     +------------------+     +----------+
|  Open    | --> | Monitor  | --> | Approaching Exp  | --> |  Close/  |
|  Position|     | Greeks & |     | DTE <= threshold |     |  Roll/   |
|          |     | DTE      |     | Action required  |     |  Expire  |
+----------+     +----------+     +------------------+     +----+-----+
                                                                |
                                                    +-----------v-----------+
                                                    | Expiration Grace      |
                                                    | Period (T+1 to T+3)  |
                                                    | Monitor for exercise/ |
                                                    | assignment results    |
                                                    +-----------+-----------+
                                                                |
                                                    +-----------v-----------+
                                                    | Confirmed Resolution: |
                                                    | - Exercised -> equity |
                                                    | - Assigned -> equity  |
                                                    | - Expired worthless   |
                                                    +-----------------------+
```

---

## Configuration Requirements

All options safety settings must be in the single config file with validation at startup:

```yaml
options_safety:
  # DTE Management
  min_dte_threshold: 3                    # Days before expiration to force action
  auto_close_at_min_dte: true             # true = auto-close, false = alert-only
  dte_warning_threshold: 7                # Days for WARNING alert
  dte_info_threshold: 21                  # Days for INFO log

  # Expiration Grace Period
  grace_period_days: 3                    # Days after expiration to monitor for exercise
  assume_itm_exercised: true              # Assume ITM options will exercise

  # Greeks Limits (portfolio level)
  max_portfolio_delta: 500
  max_portfolio_gamma: 100
  max_portfolio_theta: -200               # Negative = max daily bleed in dollars
  max_portfolio_vega: 1000

  # Spread Safety
  spread_max_leg_delay_seconds: 30        # Max time between leg fills
  protective_leg_first: true              # Enforce protective leg fills first

  # Assignment
  check_assignment_every_cycle: true      # Check for assignment on every reconciliation
  alert_on_deep_itm_short: true           # Alert when short option goes deep ITM
```

---

## Red Flags -- Stop and Investigate

- **No DTE tracking**: Options positions without expiration dates. Fix immediately -- this is how ghost positions happen.
- **Options expiring without explicit action**: System allowed silent expiration. Add DTE management with mandatory action.
- **No assignment detection**: Short options without checking for assignment. Add detection now.
- **Legging into spreads short-leg-first**: Selling naked exposure before protection is in place. ALWAYS buy protection first.
- **No Greeks monitoring**: Flying blind on portfolio risk. Add portfolio-level Greeks tracking.
- **Removing expired options from tracking immediately**: Must wait for grace period. Ghost positions WILL result.
- **No reconciliation after expiration weekend**: Monday positions MUST be checked against Friday expirations.
- **Gamma exposure ignored near expiration**: ATM options at 1 DTE have extreme gamma. Monitor and manage or close.
- **No spread leg mismatch detection**: Partial spread fills create unintended naked exposure.
- **Short options without assignment monitoring**: You can be assigned any time. Monitor continuously.
- **Same position sizing for 0DTE and 30DTE options**: 0DTE gamma is exponentially higher and requires reduced position sizes.
- **Credit spread without multiple-of-credit stop**: Percentage stops do not work for credit strategies. Use multiple-of-credit stops.
- **No defined-risk vs undefined-risk classification**: Different risk profiles need different controls. Classify every options position.
