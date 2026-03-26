---
name: kill-switch-and-circuit-breakers
description: Use when implementing emergency stop mechanisms, daily loss limits, error-rate circuit breakers, or when the bot needs to halt trading safely under adverse conditions
---

# Kill Switch and Circuit Breakers

## Iron Law

**A KILL SWITCH MUST WORK EVEN WHEN THE REST OF THE SYSTEM IS BROKEN.**

If your kill switch runs inside the same event loop as your trading logic, it will die when
the trading logic hangs. If your kill switch depends on the same database that is down, it
will not fire. The kill switch is the last line of defense -- it must be independent of
everything it is protecting.

## Kill Switch Levels

Three escalation levels, each more aggressive than the last:

| Level | Name | Actions | When |
|---|---|---|---|
| 1 | Pause | Stop opening new positions. Existing positions and stops remain active. | Approaching daily loss limit, elevated error rate, unusual volatility. |
| 2 | Cancel + Pause | Cancel all open orders, then Level 1. No new orders, no pending orders. | Daily loss limit hit, reconciliation mismatch detected, broker API degraded. |
| 3 | Flatten | Close all positions at market, then Level 2. Total exit. | Max drawdown breached, system integrity compromised, manual emergency. |

```python
import enum
import time
import logging
from dataclasses import dataclass, field
from typing import Optional

logger = logging.getLogger(__name__)


class KillSwitchLevel(enum.IntEnum):
    NONE = 0
    PAUSE = 1       # No new orders
    CANCEL = 2      # Cancel all open orders + pause
    FLATTEN = 3     # Flatten all positions + cancel + pause


@dataclass
class KillSwitchState:
    level: KillSwitchLevel = KillSwitchLevel.NONE
    reason: str = ""
    triggered_at: Optional[float] = None
    triggered_by: str = ""  # "auto" or "manual"

    @property
    def is_active(self) -> bool:
        return self.level > KillSwitchLevel.NONE


class KillSwitch:
    """
    Emergency stop mechanism for trading bot.
    Thread-safe. Independent of main trading loop.
    """

    def __init__(self, broker, portfolio):
        self._state = KillSwitchState()
        self._broker = broker
        self._portfolio = portfolio
        self._lock = threading.Lock()

    @property
    def state(self) -> KillSwitchState:
        return self._state

    def trigger(self, level: int, reason: str, triggered_by: str = "auto") -> None:
        """
        Trigger kill switch. Only escalates, never de-escalates.
        Level can only go UP, never DOWN. To reset, use explicit reset().
        """
        with self._lock:
            new_level = KillSwitchLevel(level)
            if new_level <= self._state.level:
                logger.warning(
                    f"Kill switch already at level {self._state.level}, "
                    f"ignoring level {new_level} trigger"
                )
                return

            logger.critical(
                f"KILL SWITCH TRIGGERED: Level {new_level.name} | "
                f"Reason: {reason} | By: {triggered_by}"
            )

            self._state = KillSwitchState(
                level=new_level,
                reason=reason,
                triggered_at=time.time(),
                triggered_by=triggered_by,
            )

            # Execute kill switch actions
            self._execute(new_level, reason)

    def _execute(self, level: KillSwitchLevel, reason: str) -> None:
        """Execute kill switch actions for the given level."""
        try:
            if level >= KillSwitchLevel.FLATTEN:
                self._flatten_all(reason)

            if level >= KillSwitchLevel.CANCEL:
                self._cancel_all_orders(reason)

            # PAUSE is implicit: the check_allowed() gate blocks new orders
            logger.critical(f"Kill switch level {level.name} executed successfully")

        except Exception as e:
            # Kill switch execution itself failed -- log and keep state active
            logger.critical(
                f"KILL SWITCH EXECUTION ERROR at level {level.name}: {e}. "
                f"State remains ACTIVE. Manual intervention required."
            )

    def _cancel_all_orders(self, reason: str) -> None:
        """Cancel all open orders. Best effort -- log failures but continue."""
        try:
            open_orders = self._broker.get_open_orders()
            for order in open_orders:
                try:
                    self._broker.cancel_order(order.id)
                    logger.info(f"Cancelled order {order.id}")
                except Exception as e:
                    logger.error(f"Failed to cancel order {order.id}: {e}")
        except Exception as e:
            logger.critical(f"Failed to retrieve open orders for cancellation: {e}")

    def _flatten_all(self, reason: str) -> None:
        """
        Close all positions at market. Verify each fill.
        Use market orders with slippage protection.
        """
        try:
            positions = self._portfolio.get_all_positions()
            for pos in positions:
                try:
                    close_side = "sell" if pos.side == "long" else "buy"
                    order = self._broker.place_market_order(
                        symbol=pos.symbol,
                        qty=abs(pos.qty),
                        side=close_side,
                    )
                    # Verify fill
                    fill = self._wait_for_fill(order.id, timeout=10.0)
                    if fill:
                        logger.info(
                            f"Flattened {pos.symbol}: {pos.qty} @ {fill.price}"
                        )
                    else:
                        logger.critical(
                            f"FLATTEN FILL NOT CONFIRMED for {pos.symbol}. "
                            f"MANUAL INTERVENTION REQUIRED."
                        )
                except Exception as e:
                    logger.critical(
                        f"Failed to flatten {pos.symbol}: {e}. "
                        f"MANUAL INTERVENTION REQUIRED."
                    )
        except Exception as e:
            logger.critical(f"Failed to retrieve positions for flattening: {e}")

    def _wait_for_fill(self, order_id: str, timeout: float) -> Optional[object]:
        """Wait for order fill confirmation with timeout."""
        start = time.monotonic()
        while time.monotonic() - start < timeout:
            status = self._broker.get_order_status(order_id)
            if status.filled:
                return status
            if status.rejected or status.cancelled:
                return None
            time.sleep(0.25)
        return None

    def check_allowed(self) -> bool:
        """
        Call this before every new order. Returns False if kill switch is active.
        This is the gate that enforces the pause.
        """
        return not self._state.is_active

    def reset(self, authorized_by: str) -> None:
        """
        Reset kill switch. REQUIRES human authorization.
        Never call this automatically. Never auto-resume.
        """
        with self._lock:
            logger.critical(
                f"KILL SWITCH RESET by {authorized_by}. "
                f"Previous state: level={self._state.level.name}, "
                f"reason={self._state.reason}"
            )
            self._state = KillSwitchState()
```

## Trigger Conditions

Each of these conditions should independently trigger the kill switch:

```python
import threading


class KillSwitchMonitor:
    """
    Watchdog that monitors trigger conditions independently of main trading loop.
    Runs in a separate thread so it fires even if the main loop is frozen.
    """

    def __init__(self, kill_switch: KillSwitch, config: dict):
        self.kill_switch = kill_switch
        self.config = config
        self._consecutive_rejections = 0
        self._running = True

    def check_all_triggers(self, portfolio, broker_status) -> None:
        """Run all trigger checks. Any one can fire the kill switch."""

        # 1. Daily P&L limit
        daily_pnl = portfolio.get_daily_pnl()
        if daily_pnl <= -abs(self.config["max_daily_loss"]):
            self.kill_switch.trigger(
                level=2,
                reason=f"daily loss ${daily_pnl:.2f} exceeds limit ${self.config['max_daily_loss']:.2f}"
            )

        # 2. Reconciliation mismatch
        local_positions = portfolio.get_all_positions()
        broker_positions = broker_status.get_positions()
        if not self._positions_match(local_positions, broker_positions):
            self.kill_switch.trigger(
                level=2,
                reason="position reconciliation mismatch detected"
            )

        # 3. Broker API circuit open
        if broker_status.circuit_state == "OPEN":
            self.kill_switch.trigger(
                level=1,
                reason="broker API circuit breaker is OPEN"
            )

        # 4. Consecutive order rejections
        if self._consecutive_rejections >= self.config["max_consecutive_rejections"]:
            self.kill_switch.trigger(
                level=2,
                reason=f"{self._consecutive_rejections} consecutive order rejections"
            )

        # 5. Max drawdown
        drawdown = portfolio.get_current_drawdown()
        if drawdown >= self.config["max_drawdown_pct"]:
            self.kill_switch.trigger(
                level=3,
                reason=f"drawdown {drawdown:.2%} exceeds limit {self.config['max_drawdown_pct']:.2%}"
            )

    def on_order_rejected(self):
        self._consecutive_rejections += 1

    def on_order_filled(self):
        self._consecutive_rejections = 0

    def _positions_match(self, local, broker) -> bool:
        """Compare local and broker positions. Any mismatch returns False."""
        local_map = {p.symbol: p.qty for p in local}
        broker_map = {p.symbol: p.qty for p in broker}
        return local_map == broker_map
```

## Independence from Main Loop

The kill switch watchdog MUST run independently. If the main trading loop hangs, deadlocks,
or enters an infinite loop, the watchdog must still be able to trigger.

```python
import threading
import time


class WatchdogThread(threading.Thread):
    """
    Independent watchdog that monitors the kill switch triggers.
    Runs in its own thread. If main loop stops sending heartbeats,
    the watchdog triggers the kill switch.
    """

    def __init__(
        self,
        kill_switch: KillSwitch,
        monitor: KillSwitchMonitor,
        portfolio,
        broker,
        heartbeat_timeout: float = 30.0,
        check_interval: float = 5.0,
    ):
        super().__init__(daemon=True, name="kill-switch-watchdog")
        self.kill_switch = kill_switch
        self.monitor = monitor
        self.portfolio = portfolio
        self.broker = broker
        self.heartbeat_timeout = heartbeat_timeout
        self.check_interval = check_interval
        self._last_heartbeat = time.monotonic()
        self._lock = threading.Lock()
        self._running = True

    def heartbeat(self) -> None:
        """Called by main loop to signal it is alive."""
        with self._lock:
            self._last_heartbeat = time.monotonic()

    def run(self) -> None:
        """Watchdog loop. Runs independently of main trading loop."""
        logger.info("Kill switch watchdog started")
        while self._running:
            try:
                # Check heartbeat from main loop
                with self._lock:
                    elapsed = time.monotonic() - self._last_heartbeat

                if elapsed > self.heartbeat_timeout:
                    self.kill_switch.trigger(
                        level=2,
                        reason=f"main loop heartbeat missing for {elapsed:.1f}s "
                               f"(timeout: {self.heartbeat_timeout}s)"
                    )

                # Run all other trigger checks
                broker_status = self.broker.get_status()
                self.monitor.check_all_triggers(self.portfolio, broker_status)

            except Exception as e:
                logger.error(f"Watchdog check error: {e}")
                # Watchdog errors are concerning but do not trigger kill switch
                # unless they are persistent (that would be caught by heartbeat timeout)

            time.sleep(self.check_interval)

    def stop(self) -> None:
        self._running = False
```

### Emabot Failure: Kill Switch in Main Loop

In an early emabot version, the kill switch check was inside the main `asyncio` event loop:

```python
# EMABOT BUG - DO NOT COPY
async def main_loop():
    while True:
        await process_signals()
        await execute_orders()
        await check_kill_switch()  # <-- Never reached if execute_orders() hangs
        await asyncio.sleep(1)
```

When `execute_orders()` hung waiting for a broker response that never came, `check_kill_switch()`
never executed. The bot sat with open positions and no risk monitoring for hours. The fix was
moving the kill switch to an independent watchdog thread with heartbeat monitoring.

## Recovery Protocol

After a kill switch fires, recovery MUST follow this protocol:

```
1. VERIFY:  Run full position reconciliation (local vs broker).
2. REVIEW:  Human reviews the trigger reason and all logs.
3. RESOLVE: Fix the root cause that triggered the kill switch.
4. TEST:    Verify the fix (paper trade or unit test).
5. RESET:   Human manually resets the kill switch with their name logged.
6. MONITOR: Elevated monitoring for the next trading session.
```

**NEVER auto-resume.** If the kill switch fired, something went wrong. Automatic recovery
means automatic repetition of the same failure.

```python
# WRONG - DO NOT COPY
if kill_switch.is_active and time.time() - kill_switch.triggered_at > 3600:
    kill_switch.reset("auto-resume after 1 hour")  # NEVER DO THIS

# CORRECT
# Kill switch stays active until a human explicitly resets it.
# The reset is done via CLI command or admin API, never in trading code.
```

## Circuit Breaker Pattern

For broker API calls and other external dependencies, use a circuit breaker to prevent
cascading failures.

```python
import time
import enum
from dataclasses import dataclass


class CircuitState(enum.Enum):
    CLOSED = "closed"          # Normal operation
    OPEN = "open"              # Failing, reject all calls
    HALF_OPEN = "half_open"    # Testing if service recovered


@dataclass
class CircuitBreakerConfig:
    failure_threshold: int = 5       # Failures before opening
    recovery_timeout: float = 60.0   # Seconds before trying half-open
    half_open_max_calls: int = 1     # Test calls in half-open state


class CircuitBreaker:
    """
    Circuit breaker for external service calls (broker API, data feeds, etc).
    Prevents hammering a failing service and allows graceful degradation.
    """

    def __init__(self, name: str, config: CircuitBreakerConfig):
        self.name = name
        self.config = config
        self.state = CircuitState.CLOSED
        self.failure_count = 0
        self.last_failure_time = 0.0
        self.half_open_calls = 0

    def can_execute(self) -> bool:
        """Check if a call is allowed through the circuit breaker."""
        if self.state == CircuitState.CLOSED:
            return True

        if self.state == CircuitState.OPEN:
            # Check if recovery timeout has elapsed
            elapsed = time.monotonic() - self.last_failure_time
            if elapsed >= self.config.recovery_timeout:
                self.state = CircuitState.HALF_OPEN
                self.half_open_calls = 0
                logger.info(f"Circuit breaker '{self.name}' -> HALF_OPEN")
                return True
            return False

        if self.state == CircuitState.HALF_OPEN:
            return self.half_open_calls < self.config.half_open_max_calls

        return False

    def record_success(self) -> None:
        """Record a successful call."""
        if self.state == CircuitState.HALF_OPEN:
            self.state = CircuitState.CLOSED
            self.failure_count = 0
            logger.info(f"Circuit breaker '{self.name}' -> CLOSED (recovered)")
        elif self.state == CircuitState.CLOSED:
            self.failure_count = 0

    def record_failure(self) -> None:
        """Record a failed call."""
        self.failure_count += 1
        self.last_failure_time = time.monotonic()

        if self.state == CircuitState.HALF_OPEN:
            self.state = CircuitState.OPEN
            logger.warning(f"Circuit breaker '{self.name}' -> OPEN (half-open test failed)")

        elif self.state == CircuitState.CLOSED:
            if self.failure_count >= self.config.failure_threshold:
                self.state = CircuitState.OPEN
                logger.warning(
                    f"Circuit breaker '{self.name}' -> OPEN "
                    f"({self.failure_count} failures)"
                )

    def execute(self, func, *args, **kwargs):
        """Execute a function through the circuit breaker."""
        if not self.can_execute():
            raise CircuitOpenError(
                f"Circuit breaker '{self.name}' is {self.state.value}. "
                f"Call rejected."
            )

        try:
            result = func(*args, **kwargs)
            self.record_success()
            return result
        except Exception as e:
            self.record_failure()
            raise


class CircuitOpenError(Exception):
    pass
```

## Circuit Breaker State Diagram

```
digraph circuit_breaker {
    rankdir=LR;
    node [shape=circle, style=filled];

    CLOSED [fillcolor="lightgreen", label="CLOSED\n(normal)"];
    OPEN [fillcolor="salmon", label="OPEN\n(rejecting)"];
    HALF_OPEN [fillcolor="lightyellow", label="HALF_OPEN\n(testing)"];

    CLOSED -> OPEN [label="N failures\nreached"];
    OPEN -> HALF_OPEN [label="timeout\nelapsed"];
    HALF_OPEN -> CLOSED [label="test call\nsucceeded"];
    HALF_OPEN -> OPEN [label="test call\nfailed"];
    CLOSED -> CLOSED [label="success\n(reset count)"];
}
```

## Manual Kill Switch

Always provide a way to manually trigger the kill switch. This is not optional.

```python
# CLI command for manual kill switch
# Usage: python manage.py kill-switch --level 3 --reason "unusual market conditions"

import argparse


def manual_kill_switch_cli():
    parser = argparse.ArgumentParser(description="Manual kill switch trigger")
    parser.add_argument("--level", type=int, required=True, choices=[1, 2, 3])
    parser.add_argument("--reason", type=str, required=True)
    parser.add_argument("--reset", action="store_true", help="Reset kill switch")
    parser.add_argument("--who", type=str, required=True, help="Your name for audit trail")
    args = parser.parse_args()

    kill_switch = get_kill_switch_instance()  # Connect to running instance

    if args.reset:
        kill_switch.reset(authorized_by=args.who)
        print(f"Kill switch RESET by {args.who}")
    else:
        kill_switch.trigger(
            level=args.level,
            reason=f"MANUAL: {args.reason}",
            triggered_by=f"manual:{args.who}",
        )
        print(f"Kill switch TRIGGERED at level {args.level} by {args.who}: {args.reason}")
```

## Red Flags

- **Kill switch logic inside the main event loop:** If the main loop hangs, the kill switch
  never fires. It must be in an independent thread or process.
- **No manual trigger mechanism:** You MUST be able to trigger the kill switch by hand without
  modifying code. A CLI command, API endpoint, or even a file-based trigger.
- **Auto-resume after kill switch fires:** The kill switch exists for emergencies. If it
  auto-resets, you will repeat the same emergency. Recovery requires human judgment.
- **Flatten without verifying fills:** Sending market orders and assuming they filled. Each
  flatten order must be verified. Unconfirmed flattens require manual intervention alerts.
- **Kill switch depends on the same infrastructure it protects:** If the kill switch state is
  stored in the same database that is failing, the kill switch cannot read its own state.
  Use a simple file, environment variable, or separate lightweight store.
- **No heartbeat monitoring:** Without heartbeat checks, a frozen main loop is undetectable.
  The watchdog must know the main loop is alive.
- **Circuit breaker without half-open state:** A circuit that opens and never closes requires
  manual intervention for every transient failure. Half-open with test calls enables
  automatic recovery for transient issues while staying open for persistent failures.

## Integration

- **trading-bot-skills:position-reconciliation** -- Reconciliation mismatches are a Level 2
  kill switch trigger. The reconciliation system feeds data to the kill switch monitor.
- **trading-bot-skills:trading-monitoring-and-alerts** -- All kill switch events must generate
  alerts. Level 2+ should page on-call. The monitoring system is the eyes; the kill switch
  is the hands.
- **trading-bot-skills:async-reliability** -- Async code has unique failure modes (hung awaits,
  unhandled task exceptions, event loop death) that the watchdog must detect via heartbeat
  monitoring. The circuit breaker pattern applies to all async broker API calls.
