---
name: risk-management-gates
description: Use when implementing position sizing, stop-losses, max drawdown limits, or any pre-trade risk checks, or when risk checks are being bypassed or returning None on failure
---

# Risk Management Gates

## Iron Law

**EVERY RISK CHECK THAT FAILS OR ERRORS RETURNS REJECT, NEVER NONE, NEVER "SAFE BY DEFAULT". FAIL-CLOSED, NOT FAIL-OPEN.**

This is non-negotiable. A risk gate that returns None on exception is worse than no risk gate
at all, because it creates the illusion of safety. The emabot disaster proved this with real money.

## The Emabot Fail-Open Disaster (Case Study)

In emabot, risk check functions were written like this:

```python
# CATASTROPHIC BUG - DO NOT COPY
def check_position_limit(symbol: str, qty: float) -> bool | None:
    try:
        current = get_position(symbol)
        return current.qty + qty <= MAX_POSITION
    except Exception as e:
        logger.error(f"Risk check failed: {e}")
        return None  # <-- THE BUG: None is not False
```

The caller did:

```python
# CATASTROPHIC BUG - DO NOT COPY
if check_position_limit(symbol, qty):
    place_order(symbol, qty)
```

When the broker API timed out, `check_position_limit` returned `None`. In Python, `if None:`
is falsy, so the order was blocked... sometimes. But other checks used `if not check_...():` or
`if check_...() is False:`, and `not None` is `True`, and `None is False` is `False`. The
result: exceptions silently bypassed ALL risk gates. Real money was lost before anyone noticed.

**Root cause:** Using `None` as a return value in risk-critical code. The system failed OPEN
instead of CLOSED.

## The Fail-Closed Pattern

Every risk function MUST return a structured decision, never a bare bool, never None.

```python
from dataclasses import dataclass
from typing import List


@dataclass(frozen=True)
class RiskDecision:
    """Immutable risk decision. Cannot be None. Cannot be ambiguous."""
    allowed: bool
    reason: str

    def __bool__(self):
        return self.allowed


REJECT = RiskDecision(allowed=False, reason="default reject")


def check_position_limit(symbol: str, qty: float, max_pos: float) -> RiskDecision:
    """Check if adding qty would exceed position limit for symbol."""
    try:
        current = get_current_position(symbol)
        proposed = abs(current.qty + qty)
        if proposed > max_pos:
            return RiskDecision(
                allowed=False,
                reason=f"position limit: {proposed:.2f} > {max_pos:.2f} for {symbol}"
            )
        return RiskDecision(allowed=True, reason=f"position OK: {proposed:.2f} <= {max_pos:.2f}")
    except Exception as e:
        # FAIL CLOSED. Exception = REJECT. Always.
        return RiskDecision(
            allowed=False,
            reason=f"risk check error (position limit): {e}"
        )
```

## Pre-Trade Risk Gate

Before EVERY order, run ALL checks. ANY failure or error rejects the order.

```python
def pre_trade_risk_gate(
    symbol: str,
    side: str,
    qty: float,
    price: float,
    portfolio: Portfolio,
    config: RiskConfig,
) -> RiskDecision:
    """
    Run all pre-trade risk checks. ALL must pass. ANY failure = REJECT.
    ANY exception = REJECT. No exceptions. No shortcuts.
    """
    checks: List[RiskDecision] = []

    # 1. Max position size per symbol
    checks.append(check_position_limit(symbol, qty, config.max_position_size))

    # 2. Max portfolio exposure (sum of all positions as % of equity)
    checks.append(check_portfolio_exposure(portfolio, symbol, qty, price, config.max_exposure))

    # 3. Max daily loss
    checks.append(check_daily_loss(portfolio, config.max_daily_loss))

    # 4. Max concurrent positions
    checks.append(check_concurrent_positions(portfolio, config.max_concurrent))

    # 5. Symbol-level limits (sector, correlation, etc.)
    checks.append(check_symbol_limits(symbol, portfolio, config))

    # 6. Order rate limit (prevent runaway loops)
    checks.append(check_order_rate(symbol, config.max_orders_per_minute))

    # Evaluate: ALL must be allowed
    for decision in checks:
        if not decision.allowed:
            return RiskDecision(
                allowed=False,
                reason=f"BLOCKED: {decision.reason}"
            )

    return RiskDecision(allowed=True, reason="all risk checks passed")
```

**Critical:** Notice there is no `if decision is None` check because `RiskDecision` can never
be None. The type system enforces correctness.

## Position Sizing Rules

Never risk more than a fixed fraction of equity per trade. Size based on stop distance.

```python
def calculate_position_size(
    equity: float,
    risk_per_trade: float,  # e.g., 0.01 = 1% of equity
    entry_price: float,
    stop_price: float,
) -> RiskDecision:
    """
    Calculate position size based on stop distance.
    risk_per_trade: fraction of equity to risk (e.g., 0.01 for 1%).
    """
    try:
        if entry_price <= 0 or stop_price <= 0:
            return RiskDecision(allowed=False, reason="invalid prices for sizing")

        stop_distance = abs(entry_price - stop_price)
        if stop_distance < 1e-9:
            return RiskDecision(allowed=False, reason="stop distance is zero")

        risk_amount = equity * risk_per_trade
        raw_size = risk_amount / stop_distance

        # Round down to avoid exceeding risk
        import math
        size = math.floor(raw_size * 100) / 100  # 2 decimal places, floor

        if size <= 0:
            return RiskDecision(allowed=False, reason="calculated size is zero or negative")

        return RiskDecision(
            allowed=True,
            reason=f"size={size:.2f}, risking ${risk_amount:.2f} with stop distance ${stop_distance:.2f}"
        )
    except Exception as e:
        return RiskDecision(allowed=False, reason=f"sizing error: {e}")
```

### Kelly Criterion Variant

```python
def kelly_position_size(
    equity: float,
    win_rate: float,
    avg_win: float,
    avg_loss: float,
    kelly_fraction: float = 0.25,  # Use quarter-Kelly for safety
) -> float:
    """
    Kelly criterion position sizing with fractional Kelly for safety.
    Returns fraction of equity to allocate.
    """
    if avg_loss == 0 or avg_win == 0:
        return 0.0

    win_loss_ratio = avg_win / avg_loss
    kelly = win_rate - ((1 - win_rate) / win_loss_ratio)

    # Clamp: never negative, never more than max
    kelly = max(0.0, min(kelly, 0.25))

    # Apply fractional Kelly (quarter-Kelly is conservative)
    return kelly * kelly_fraction
```

## Stop-Loss Rules

Every position MUST have a stop-loss set within N seconds of fill confirmation.

```python
import time
from typing import Optional


STOP_LOSS_DEADLINE_SECONDS = 5


async def ensure_stop_loss(
    position: Position,
    stop_price: float,
    deadline_seconds: float = STOP_LOSS_DEADLINE_SECONDS,
) -> RiskDecision:
    """
    Set stop-loss for position. If stop is not confirmed within deadline,
    issue market close. A position without a stop is an emergency.
    """
    try:
        if stop_price <= 0:
            return RiskDecision(allowed=False, reason="invalid stop price")

        # Validate stop is on the correct side
        if position.side == "long" and stop_price >= position.entry_price:
            return RiskDecision(allowed=False, reason="long stop must be below entry")
        if position.side == "short" and stop_price <= position.entry_price:
            return RiskDecision(allowed=False, reason="short stop must be above entry")

        # Attempt to place stop order
        stop_order = await broker.place_stop_order(
            symbol=position.symbol,
            qty=position.qty,
            stop_price=stop_price,
            side="sell" if position.side == "long" else "buy",
        )

        # Verify stop was accepted
        start = time.monotonic()
        while time.monotonic() - start < deadline_seconds:
            status = await broker.get_order_status(stop_order.id)
            if status == "accepted":
                return RiskDecision(allowed=True, reason=f"stop set at {stop_price}")
            if status in ("rejected", "cancelled"):
                break
            await asyncio.sleep(0.25)

        # Deadline exceeded or rejected -> EMERGENCY: flatten the position
        logger.critical(f"STOP-LOSS NOT CONFIRMED for {position.symbol} - EMERGENCY FLATTEN")
        await emergency_flatten(position)
        return RiskDecision(
            allowed=False,
            reason=f"stop not confirmed within {deadline_seconds}s - position flattened"
        )
    except Exception as e:
        logger.critical(f"Stop-loss placement error: {e} - EMERGENCY FLATTEN")
        await emergency_flatten(position)
        return RiskDecision(allowed=False, reason=f"stop placement error: {e}")
```

## Max Drawdown Kill Switch

```python
class DrawdownMonitor:
    """Track equity curve and trigger kill switch on max drawdown breach."""

    def __init__(self, max_drawdown_pct: float, kill_switch: KillSwitch):
        self.max_drawdown_pct = max_drawdown_pct
        self.peak_equity = 0.0
        self.kill_switch = kill_switch

    def update(self, current_equity: float) -> RiskDecision:
        try:
            if current_equity <= 0:
                self.kill_switch.trigger(level=3, reason="equity <= 0")
                return RiskDecision(allowed=False, reason="equity is zero or negative")

            self.peak_equity = max(self.peak_equity, current_equity)
            drawdown = (self.peak_equity - current_equity) / self.peak_equity

            if drawdown >= self.max_drawdown_pct:
                self.kill_switch.trigger(
                    level=3,
                    reason=f"drawdown {drawdown:.2%} >= limit {self.max_drawdown_pct:.2%}"
                )
                return RiskDecision(
                    allowed=False,
                    reason=f"max drawdown breached: {drawdown:.2%}"
                )

            return RiskDecision(
                allowed=True,
                reason=f"drawdown OK: {drawdown:.2%} < {self.max_drawdown_pct:.2%}"
            )
        except Exception as e:
            self.kill_switch.trigger(level=3, reason=f"drawdown monitor error: {e}")
            return RiskDecision(allowed=False, reason=f"drawdown monitor error: {e}")
```

## Risk Gate Flow

```
digraph risk_gate_flow {
    rankdir=TB;
    node [shape=box, style=rounded];

    signal [label="Trade Signal"];
    gate [label="Pre-Trade Risk Gate", shape=diamond];
    pos_size [label="Position Size Check"];
    exposure [label="Portfolio Exposure Check"];
    daily_loss [label="Daily Loss Check"];
    concurrent [label="Concurrent Positions Check"];
    symbol [label="Symbol Limits Check"];
    rate [label="Order Rate Check"];
    allow [label="ORDER ALLOWED", style="filled", fillcolor="lightgreen"];
    reject [label="ORDER REJECTED", style="filled", fillcolor="salmon"];

    signal -> gate;
    gate -> pos_size;
    gate -> exposure;
    gate -> daily_loss;
    gate -> concurrent;
    gate -> symbol;
    gate -> rate;

    pos_size -> allow [label="all pass"];
    pos_size -> reject [label="any fail OR error"];
    exposure -> allow;
    exposure -> reject;
    daily_loss -> allow;
    daily_loss -> reject;
    concurrent -> allow;
    concurrent -> reject;
    symbol -> allow;
    symbol -> reject;
    rate -> allow;
    rate -> reject;
}
```

## Common Rationalizations (And Why They Are Wrong)

| Rationalization | Why It Is Wrong |
|---|---|
| "The exception is transient, just retry" | Retry is fine. Returning None/True while retrying is not. Reject until retry succeeds. |
| "We log the error so we will notice" | You will not notice at 3 AM when the bot is running unattended. |
| "This check is redundant anyway" | Redundancy is the point. Defense in depth. Remove nothing. |
| "None means unknown, the caller handles it" | The caller does `if result:` and None is falsy... sometimes. Ambiguity kills. |
| "We can add proper error handling later" | Risk code ships correct or does not ship. There is no "later" when money is on the line. |
| "The position is small, risk is minimal" | Small positions become large positions when risk gates are broken in a loop. |
| "It works in backtesting" | Backtesting never has broker timeouts, API errors, or network failures. |

## Red Flags

Watch for these patterns in code review. Any one of these is a potential fail-open bug:

- A risk function that can return `None` (check all code paths including exceptions)
- `if risk_check():` where the function might return None (None is falsy but not False)
- `if not risk_check():` where the function might return None (`not None` is `True` -- fail open!)
- `if risk_check() is False:` -- None is not False, so None passes this check (fail open!)
- No stop-loss set on position entry
- Position sizing that does not account for stop distance
- Daily loss check that resets on restart (must persist to disk/database)
- Risk config loaded once at startup with no way to update without restart
- `except: pass` or `except: continue` anywhere in risk code

## Integration

- **trading-bot-skills:order-execution-integrity** -- Risk gates must run BEFORE order execution. Order execution must verify risk gate returned `allowed=True`.
- **trading-bot-skills:kill-switch-and-circuit-breakers** -- Max drawdown and daily loss triggers feed into the kill switch. Kill switch is the last line of defense when risk gates fail.
- **trading-bot-skills:trailing-stop-mechanics** -- Stop-loss rules here define the initial stop. Trailing stop mechanics handle dynamic adjustment after entry.
