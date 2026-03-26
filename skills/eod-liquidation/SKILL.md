---
name: eod-liquidation
description: Use when implementing end-of-day position flattening, avoiding overnight risk, or when positions are held past market close unexpectedly
---

# EOD Liquidation

**Iron Law:** DAY TRADING BOTS MUST FLATTEN ALL POSITIONS BEFORE MARKET CLOSE. THE EOD FLATTEN IS A SCHEDULED RISK CONTROL, NOT AN AFTERTHOUGHT.

Overnight gaps can exceed any stop-loss. A stock that closes at $100 can open at $90 on bad earnings. That is a 10% loss that your 2% stop-loss never had a chance to catch. Day trading bots must never hold overnight.

## Time-Based Flatten Trigger

The EOD flatten is a scheduled event, not a reaction. It fires 10-15 minutes before market close. Never rely on broker `OnEndOfDay` events -- they fire AT or AFTER the close, which is too late.

```python
import time
from dataclasses import dataclass
from datetime import datetime, time as dtime
from zoneinfo import ZoneInfo
from typing import Callable, Optional

ET = ZoneInfo("America/New_York")


@dataclass
class EODConfig:
    eod_exit_minutes_before_close: int = 15
    walk_down_enabled: bool = True
    walk_down_interval_seconds: int = 60
    walk_down_step_cents: int = 5
    force_market_minutes_before_close: int = 2
    zero_dte_exit_minutes_before_close: int = 30


class EODScheduler:
    """Schedules and triggers end-of-day position flattening."""

    REGULAR_CLOSE = dtime(16, 0)
    HALF_DAY_CLOSE = dtime(13, 0)

    def __init__(self, config: EODConfig, half_days: set[str]):
        self.config = config
        self._half_days = half_days
        self._flatten_triggered = False

    def get_close_time(self) -> dtime:
        date_str = datetime.now(ET).strftime("%Y-%m-%d")
        if date_str in self._half_days:
            return self.HALF_DAY_CLOSE
        return self.REGULAR_CLOSE

    def minutes_until_close(self) -> float:
        now = datetime.now(ET)
        close = self.get_close_time()
        close_dt = now.replace(hour=close.hour, minute=close.minute, second=0, microsecond=0)
        delta = (close_dt - now).total_seconds() / 60.0
        return delta

    def should_flatten(self) -> bool:
        minutes_left = self.minutes_until_close()
        return minutes_left <= self.config.eod_exit_minutes_before_close

    def should_flatten_zero_dte(self) -> bool:
        minutes_left = self.minutes_until_close()
        return minutes_left <= self.config.zero_dte_exit_minutes_before_close

    def should_force_market_order(self) -> bool:
        minutes_left = self.minutes_until_close()
        return minutes_left <= self.config.force_market_minutes_before_close

    def check_and_trigger(self, flatten_callback: Callable, positions: list) -> None:
        """Called on each tick. Triggers flatten when time threshold is reached."""
        if self._flatten_triggered:
            return

        if not positions:
            return

        zero_dte_positions = [p for p in positions if self._is_zero_dte(p)]
        regular_positions = [p for p in positions if not self._is_zero_dte(p)]

        # 0DTE exits earlier
        if zero_dte_positions and self.should_flatten_zero_dte():
            self._flatten_triggered = True
            flatten_callback(zero_dte_positions, force_market=False)

        # Regular positions
        if regular_positions and self.should_flatten():
            self._flatten_triggered = True
            flatten_callback(regular_positions, force_market=False)

    def _is_zero_dte(self, position) -> bool:
        if not hasattr(position, "expiration"):
            return False
        today = datetime.now(ET).strftime("%Y-%m-%d")
        return position.expiration == today

    def reset_daily(self) -> None:
        """Call at start of each trading day."""
        self._flatten_triggered = False
```

## Walk-Down Limit Orders

For illiquid options, market orders get terrible fills. The walk-down pattern starts with a limit order at mid-price and gradually lowers the ask until filled or time forces a market order.

```python
import time
from dataclasses import dataclass
from typing import Optional


@dataclass
class WalkDownState:
    symbol: str
    qty: int
    side: str
    current_limit: float
    order_id: Optional[str]
    step_count: int = 0
    started_at: float = 0.0


class WalkDownLiquidator:
    """Liquidates positions using walk-down limit orders to minimize slippage."""

    def __init__(self, broker_client, config: EODConfig, audit_logger):
        self._broker = broker_client
        self._config = config
        self._audit = audit_logger
        self._active_walkdowns: dict[str, WalkDownState] = {}

    def start_liquidation(self, symbol: str, qty: int, side: str) -> None:
        """Begin walk-down liquidation for a position."""
        quote = self._broker.get_latest_quote(symbol)
        mid_price = (quote.bid + quote.ask) / 2.0

        # Start at mid-price
        initial_limit = round(mid_price, 2)

        order = self._broker.submit_order(
            symbol=symbol,
            qty=qty,
            side=side,
            type="limit",
            limit_price=initial_limit,
            time_in_force="day",
        )

        state = WalkDownState(
            symbol=symbol,
            qty=qty,
            side=side,
            current_limit=initial_limit,
            order_id=order.id,
            started_at=time.time(),
        )
        self._active_walkdowns[symbol] = state

        self._audit.log(
            event_type="eod_walkdown_started",
            component="eod_liquidator",
            symbol=symbol,
            data={
                "qty": qty,
                "side": side,
                "initial_limit": initial_limit,
                "mid_price": mid_price,
                "bid": quote.bid,
                "ask": quote.ask,
            },
        )

    def tick(self, eod_scheduler: EODScheduler) -> None:
        """Called periodically. Walks down limits or forces market orders."""
        completed = []

        for symbol, state in self._active_walkdowns.items():
            # Check if order is already filled
            order = self._broker.get_order(state.order_id)
            if order.status == "filled":
                self._audit.log(
                    event_type="eod_walkdown_filled",
                    component="eod_liquidator",
                    symbol=symbol,
                    data={
                        "fill_price": float(order.filled_avg_price),
                        "steps_taken": state.step_count,
                    },
                )
                completed.append(symbol)
                continue

            # Force market order if time is running out
            if eod_scheduler.should_force_market_order():
                self._cancel_and_market(state)
                completed.append(symbol)
                continue

            # Walk down: lower the limit price by step_cents
            elapsed = time.time() - state.started_at
            expected_steps = int(elapsed / self._config.walk_down_interval_seconds)

            if expected_steps > state.step_count:
                step_amount = self._config.walk_down_step_cents / 100.0
                new_limit = round(state.current_limit - step_amount, 2)
                new_limit = max(new_limit, 0.01)  # floor at 1 cent

                # Cancel old order and submit new one
                self._broker.cancel_order(state.order_id)
                new_order = self._broker.submit_order(
                    symbol=symbol,
                    qty=state.qty,
                    side=state.side,
                    type="limit",
                    limit_price=new_limit,
                    time_in_force="day",
                )

                state.order_id = new_order.id
                state.current_limit = new_limit
                state.step_count = expected_steps

                self._audit.log(
                    event_type="eod_walkdown_step",
                    component="eod_liquidator",
                    symbol=symbol,
                    data={
                        "new_limit": new_limit,
                        "step": state.step_count,
                    },
                )

        for symbol in completed:
            del self._active_walkdowns[symbol]

    def _cancel_and_market(self, state: WalkDownState) -> None:
        """Last resort: cancel limit order and submit market order."""
        try:
            self._broker.cancel_order(state.order_id)
        except Exception:
            pass  # order may already be filled or cancelled

        self._broker.submit_order(
            symbol=state.symbol,
            qty=state.qty,
            side=state.side,
            type="market",
            time_in_force="day",
        )

        self._audit.log(
            event_type="eod_force_market",
            component="eod_liquidator",
            symbol=state.symbol,
            data={
                "reason": "time_expired",
                "steps_taken": state.step_count,
                "last_limit": state.current_limit,
            },
        )

    @property
    def has_active_walkdowns(self) -> bool:
        return len(self._active_walkdowns) > 0
```

## Overnight Gap Risk

Gaps bypass stop-losses entirely. There is no order type that protects you from a gap.

**Example scenario:**
- Stock closes at $100. You hold 100 shares long with a stop-loss at $98 (2% risk).
- Overnight, the company reports terrible earnings.
- Stock opens at $90. Your stop-loss triggers at $90 (or worse), not at $98.
- Actual loss: 10%, not the 2% you planned for.

This is not a theoretical risk. It happens regularly around earnings, FDA announcements, geopolitical events, and macro data releases. Day trading bots eliminate this risk by being flat before close.

## OnEndOfDay Event Pitfalls

Many broker APIs provide an `OnEndOfDay` or `OnMarketClose` callback. These are dangerous for flattening because:

1. **They fire AT the close, not before it.** By the time your code runs, the closing auction may have started or completed.
2. **For options, exercise/assignment decisions happen after close.** If you hold ITM options at close, you may be auto-exercised or assigned before your flatten logic runs.
3. **Network latency and processing time mean your orders arrive AFTER the close.** The broker rejects them as "market closed."

```python
# --- BAD: Using broker OnEndOfDay event ---
def on_end_of_day(positions):
    # This fires AT or AFTER close -- too late!
    for pos in positions:
        broker.submit_order(symbol=pos.symbol, qty=pos.qty, side="sell", type="market")
        # Result: orders rejected because market is already closed

# --- GOOD: Scheduled flatten 15 minutes before close ---
def on_tick(positions, eod_scheduler):
    if eod_scheduler.should_flatten():
        for pos in positions:
            walk_down_liquidator.start_liquidation(pos.symbol, pos.qty, "sell")
        # Result: orders submitted while market is still open with time to fill
```

## Options-Specific EOD Handling

Options near expiration have unique risks at end of day:

- **Exercise/assignment risk:** ITM options may be auto-exercised, creating unexpected stock positions overnight.
- **0DTE options:** Same-day expiration options must exit even earlier (by 3:30 PM ET) because their value can swing wildly in the final minutes and exercise risk is immediate.
- **Pin risk:** Options near the strike may flip between ITM and OTM repeatedly near close, making exercise unpredictable.

```python
class OptionsEODHandler:
    """Handles options-specific EOD concerns."""

    def __init__(self, eod_scheduler: EODScheduler, liquidator: WalkDownLiquidator, audit_logger):
        self._scheduler = eod_scheduler
        self._liquidator = liquidator
        self._audit = audit_logger

    def process_positions(self, positions: list) -> None:
        for pos in positions:
            if not hasattr(pos, "expiration"):
                continue  # equity position, handled by regular EOD

            is_zero_dte = self._scheduler._is_zero_dte(pos)
            is_itm = self._is_in_the_money(pos)
            side = "sell" if pos.qty > 0 else "buy"
            qty = abs(pos.qty)

            # 0DTE: exit by 3:30 PM regardless
            if is_zero_dte and self._scheduler.should_flatten_zero_dte():
                self._audit.log(event_type="eod_zero_dte_exit", component="options_eod",
                                symbol=pos.symbol, data={"reason": "0DTE time limit", "is_itm": is_itm})
                self._liquidator.start_liquidation(pos.symbol, qty, side)
            # ITM options near close: exit to avoid exercise/assignment
            elif is_itm and self._scheduler.should_flatten():
                self._audit.log(event_type="eod_itm_exit", component="options_eod",
                                symbol=pos.symbol, data={"reason": "ITM exercise risk", "is_itm": is_itm})
                self._liquidator.start_liquidation(pos.symbol, qty, side)
            # Regular options near close
            elif self._scheduler.should_flatten():
                self._liquidator.start_liquidation(pos.symbol, qty, side)

    def _is_in_the_money(self, pos) -> bool:
        if not hasattr(pos, "strike") or not hasattr(pos, "option_type"):
            return False
        try:
            quote = self._liquidator._broker.get_latest_quote(pos.underlying_symbol)
            mid = (quote.bid + quote.ask) / 2
            if pos.option_type == "call":
                return mid > pos.strike
            else:
                return mid < pos.strike
        except Exception:
            return True  # assume ITM if we cannot determine, safer to exit
```

## Configuration Reference

| Parameter | Default | Description |
|---|---|---|
| `eod_exit_minutes_before_close` | 15 | When to start flattening regular positions |
| `walk_down_enabled` | True | Use walk-down limit orders instead of market orders |
| `walk_down_interval_seconds` | 60 | Time between each price step-down |
| `walk_down_step_cents` | 5 | Amount to lower limit price each step |
| `force_market_minutes_before_close` | 2 | When to abandon limits and force market orders |
| `zero_dte_exit_minutes_before_close` | 30 | When to start flattening 0DTE options |

## Red Flags

| Red Flag | Why It Matters |
|---|---|
| No EOD flatten configured | Positions held overnight are exposed to gap risk |
| Using broker `OnEndOfDay` as flatten trigger | Fires too late, orders arrive after market close |
| Market orders on illiquid options | Terrible fills, potentially losing more than the position is worth |
| No walk-down logic for options | Single limit order sits unfilled while time runs out |
| 0DTE options held past 3:30 PM ET | Exercise/assignment risk, extreme gamma risk |
| No half-day awareness | Half-days close at 1:00 PM ET, flatten must trigger earlier |
| Flatten logic only runs on manual trigger | Must be automated and scheduled, not dependent on operator |
| No audit logging of EOD flatten | Cannot verify flatten completed or diagnose failures |

## Integration

- **kill-switch-and-circuit-breakers** -- EOD flatten is a scheduled risk control. The kill switch is an emergency control. Both flatten positions, but EOD is routine and the kill switch is exceptional. They must not interfere with each other.
- **options-trading-safety** -- options-specific EOD handling (exercise/assignment risk, 0DTE early exit) builds on the options safety foundation.
- **pre-trade-validation** -- the market hours engine used by pre-trade validation is the same engine that drives EOD timing. Share the `MarketHoursChecker` instance to ensure consistent behavior.
- **trade-audit-and-replay** -- every EOD flatten event (start, walk-down step, force market, completion) is logged to the audit trail. Replay validation confirms the flatten completed successfully.
