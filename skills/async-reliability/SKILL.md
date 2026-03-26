---
name: async-reliability
description: Use when implementing asyncio tasks, background workers, or any concurrent code in trading systems, or when tasks fail silently or exceptions return None
---

# Async Reliability for Trading Systems

Prevents: silent task failures (incident #4), task invisibility (incident #12),
fail-open behavior (incident #13).

In a trading bot, async tasks are not background jobs you can retry later.
They are the heartbeat of the system: market data processing, order monitoring,
position reconciliation, risk checking. When a task dies silently, the system
appears healthy while it is actually broken. Positions drift. Orders go
unmonitored. Risk checks stop running. By the time someone notices, the
damage is done.

---

## The Iron Law

> **EVERY ASYNCIO TASK MUST HAVE A done_callback.**
> **EVERY EXCEPTION MUST PROPAGATE OR ALERT.**
> **NO except:pass. EVER.**
>
> Origin: Incident #4 -- Silent Task Failure. A background task responsible
> for monitoring open orders raised an unhandled exception. The task had no
> done_callback. The exception was stored on the Task object but never
> retrieved. The event loop logged a "Task exception was never retrieved"
> warning to stderr, which was not monitored. The order monitor appeared
> healthy (no errors in application logs). For 6 hours, open orders went
> unmonitored. A partial fill was never detected. The bot opened a new
> position on top of the existing one, doubling exposure.

---

## The Three Rules

### Rule 1: Every Task Gets a done_callback

```python
# BAD: Fire and forget -- exception is silently swallowed
asyncio.create_task(monitor_orders())

# BAD: Storing the task but never checking it
self._task = asyncio.create_task(monitor_orders())

# GOOD: done_callback ensures exceptions are always handled
task = asyncio.create_task(monitor_orders(), name="order-monitor")
task.add_done_callback(self._on_task_done)

def _on_task_done(self, task: asyncio.Task) -> None:
    """Called when ANY task completes. NEVER ignore exceptions here."""
    if task.cancelled():
        logger.info(f"Task {task.get_name()} was cancelled")
        return

    exception = task.exception()
    if exception is not None:
        logger.critical(
            f"Task {task.get_name()} DIED with exception: {exception}",
            exc_info=exception,
        )
        # Choose ONE response based on task criticality:

        # For critical tasks (order monitor, risk checker):
        asyncio.create_task(self._emergency_shutdown(
            reason=f"Critical task {task.get_name()} died: {exception}"
        ))

        # For important tasks (analytics, non-critical logging):
        asyncio.create_task(self._restart_task(task.get_name()))

        # For nice-to-have tasks (metrics export):
        asyncio.create_task(alert_monitoring(
            f"Task {task.get_name()} died: {exception}"
        ))
```

### Rule 2: No Bare Except. No except:pass.

```python
# LETHAL: Silences ALL errors including SystemExit and KeyboardInterrupt
try:
    await process_order()
except:
    pass

# BAD: Catches everything, logs nothing, continues with corrupted state
try:
    position = await get_position(symbol)
except Exception:
    position = None  # Silently returns None -- caller has no idea

# BAD: Logs but continues as if nothing happened
try:
    await update_position(fill)
except Exception as e:
    logger.error(f"Failed to update position: {e}")
    # Continues trading with stale position data!

# GOOD: Specific exceptions, explicit handling, re-raise or alert
try:
    await update_position(fill)
except StaleDataError:
    logger.warning(f"Stale data for {fill.symbol}, triggering reconciliation")
    await reconcile_position(fill.symbol)
    await update_position(fill)  # Retry after reconciliation
except asyncpg.PostgresError as e:
    logger.exception(f"Database error updating position for {fill.symbol}")
    await alert_monitoring("position_update_db_error", fill.symbol, str(e))
    raise  # Re-raise: caller must know this failed
```

### Rule 3: Never Return None as a Failure Signal

```python
# BAD: Caller cannot distinguish "no position" from "error getting position"
async def get_position(symbol: str) -> Optional[Position]:
    try:
        return await db.fetch_position(symbol)
    except Exception:
        return None  # Was this "not found" or "database is on fire"?

# GOOD: Explicit error types
async def get_position(symbol: str) -> Position:
    """Returns position or raises. Never returns None for errors."""
    try:
        row = await db.fetch_position(symbol)
    except asyncpg.PostgresError as e:
        raise DatabaseError(f"Cannot fetch position for {symbol}: {e}") from e

    if row is None:
        return Position(symbol=symbol, qty=Decimal("0"))  # Explicit zero

    return Position(symbol=symbol, qty=row["qty"])


# ALSO GOOD: Use Result type for expected failures
from dataclasses import dataclass

@dataclass
class PositionResult:
    position: Position | None = None
    error: str | None = None

    @property
    def is_ok(self) -> bool:
        return self.error is None

async def get_position_safe(symbol: str) -> PositionResult:
    try:
        pos = await get_position(symbol)
        return PositionResult(position=pos)
    except DatabaseError as e:
        return PositionResult(error=str(e))
```

---

## Task Registry Pattern

Track ALL running tasks. Know what is alive, what died, and when.

```python
import asyncio
import logging
from dataclasses import dataclass, field
from datetime import datetime, timezone
from enum import Enum
from typing import Coroutine, Callable

logger = logging.getLogger(__name__)


class TaskCriticality(Enum):
    CRITICAL = "critical"       # Death triggers emergency shutdown
    IMPORTANT = "important"     # Death triggers restart
    BACKGROUND = "background"   # Death triggers alert only


@dataclass
class TaskInfo:
    name: str
    criticality: TaskCriticality
    task: asyncio.Task
    started_at: datetime = field(
        default_factory=lambda: datetime.now(timezone.utc)
    )
    restart_count: int = 0
    max_restarts: int = 3


class TaskRegistry:
    """
    Central registry for all async tasks in the trading system.

    Rules:
    - ALL tasks are created through this registry.
    - ALL tasks have done_callbacks.
    - Task deaths are handled based on criticality.
    - Registry can report system health at any time.
    """

    def __init__(self) -> None:
        self._tasks: dict[str, TaskInfo] = {}
        self._factories: dict[str, Callable[[], Coroutine]] = {}
        self._shutdown_event = asyncio.Event()

    def register(
        self,
        name: str,
        coro_factory: Callable[[], Coroutine],
        criticality: TaskCriticality = TaskCriticality.IMPORTANT,
    ) -> asyncio.Task:
        """Create and register a task."""
        if name in self._tasks and not self._tasks[name].task.done():
            raise RuntimeError(f"Task {name} is already running")

        self._factories[name] = coro_factory
        task = asyncio.create_task(coro_factory(), name=name)
        task.add_done_callback(lambda t: self._on_task_done(t, name))

        self._tasks[name] = TaskInfo(
            name=name, criticality=criticality, task=task,
        )
        logger.info(
            f"Task registered: {name} (criticality={criticality.value})"
        )
        return task

    def _on_task_done(self, task: asyncio.Task, name: str) -> None:
        """Handle task completion based on criticality."""
        if task.cancelled():
            logger.info(f"Task {name} cancelled")
            return

        exception = task.exception()
        if exception is None:
            logger.info(f"Task {name} completed normally")
            return

        info = self._tasks.get(name)
        if info is None:
            logger.error(f"Unknown task {name} died: {exception}")
            return

        logger.error(f"Task {name} died: {exception}", exc_info=exception)

        if info.criticality == TaskCriticality.CRITICAL:
            logger.critical(
                f"CRITICAL task {name} died. Emergency shutdown."
            )
            self._shutdown_event.set()

        elif info.criticality == TaskCriticality.IMPORTANT:
            if info.restart_count < info.max_restarts:
                info.restart_count += 1
                logger.warning(
                    f"Restarting {name} "
                    f"(attempt {info.restart_count}/{info.max_restarts})"
                )
                asyncio.get_event_loop().call_soon(
                    lambda: asyncio.create_task(
                        self._restart_task(name)
                    )
                )
            else:
                logger.critical(
                    f"Task {name} exceeded max restarts. Shutdown."
                )
                self._shutdown_event.set()

        else:  # BACKGROUND
            logger.warning(
                f"Background task {name} died. Not restarting."
            )

    async def _restart_task(self, name: str) -> None:
        """Restart a failed task."""
        factory = self._factories.get(name)
        if factory is None:
            logger.error(f"No factory for task {name}, cannot restart")
            return

        info = self._tasks[name]
        new_task = asyncio.create_task(factory(), name=name)
        new_task.add_done_callback(
            lambda t: self._on_task_done(t, name)
        )
        info.task = new_task
        logger.info(f"Task {name} restarted")

    def health_check(self) -> dict[str, str]:
        """Report health of all registered tasks."""
        report = {}
        for name, info in self._tasks.items():
            if info.task.done():
                if info.task.cancelled():
                    report[name] = "cancelled"
                elif info.task.exception():
                    report[name] = f"dead: {info.task.exception()}"
                else:
                    report[name] = "completed"
            else:
                report[name] = "running"
        return report

    @property
    def shutdown_requested(self) -> asyncio.Event:
        return self._shutdown_event

    async def shutdown_all(self) -> None:
        """Cancel all tasks and wait for completion."""
        for info in self._tasks.values():
            if not info.task.done():
                info.task.cancel()
        tasks = [
            info.task for info in self._tasks.values()
            if not info.task.done()
        ]
        if tasks:
            await asyncio.gather(*tasks, return_exceptions=True)
        logger.info("All tasks shut down")
```

### Usage

```python
registry = TaskRegistry()

# Register tasks with appropriate criticality
registry.register(
    "order-monitor", order_monitor.run, TaskCriticality.CRITICAL
)
registry.register(
    "risk-checker", risk_checker.run, TaskCriticality.CRITICAL
)
registry.register(
    "position-sync", position_sync.run, TaskCriticality.IMPORTANT
)
registry.register(
    "metrics-export", metrics.export_loop, TaskCriticality.BACKGROUND
)

# Monitor for shutdown
await registry.shutdown_requested.wait()
await registry.shutdown_all()
```

---

## Heartbeat Pattern

Long-running loops MUST emit heartbeats. A loop that stops looping is
indistinguishable from a dead task -- unless it has heartbeats.

```python
from datetime import datetime, timezone, timedelta

HEARTBEAT_INTERVAL = timedelta(seconds=30)
HEARTBEAT_TIMEOUT = timedelta(seconds=90)  # 3x interval


class HeartbeatMonitor:
    """Tracks heartbeats from long-running tasks."""

    def __init__(self) -> None:
        self._heartbeats: dict[str, datetime] = {}

    def beat(self, task_name: str) -> None:
        """Record a heartbeat."""
        self._heartbeats[task_name] = datetime.now(timezone.utc)

    def check_all(self) -> list[str]:
        """Return names of tasks that have missed their heartbeat."""
        now = datetime.now(timezone.utc)
        stale = []
        for name, last_beat in self._heartbeats.items():
            if now - last_beat > HEARTBEAT_TIMEOUT:
                stale.append(name)
        return stale


# In a long-running loop:
async def order_monitor_loop(heartbeat: HeartbeatMonitor):
    """Monitor open orders. Emits heartbeat every iteration."""
    while True:
        try:
            heartbeat.beat("order-monitor")

            open_orders = await gateway.get_open_orders()
            for order in open_orders:
                await check_order_status(order)

            await asyncio.sleep(1.0)

        except asyncio.CancelledError:
            raise  # Always let cancellation propagate
        except Exception:
            logger.exception("Error in order monitor loop")
            raise  # Do NOT continue with a broken monitor


# Separate watchdog task checks heartbeats:
async def heartbeat_watchdog(
    heartbeat: HeartbeatMonitor,
    registry: TaskRegistry,
):
    """Watchdog that detects stuck tasks via missing heartbeats."""
    while True:
        await asyncio.sleep(HEARTBEAT_INTERVAL.total_seconds())
        stale = heartbeat.check_all()
        if stale:
            logger.critical(
                f"HEARTBEAT TIMEOUT: Tasks {stale} appear stuck"
            )
            for task_name in stale:
                await alert_monitoring("heartbeat_timeout", task_name)
```

---

## Graceful Shutdown

```python
async def graceful_shutdown(
    registry: TaskRegistry,
    bus: IntentBus,
    gateway: ExecutionGateway,
    timeout: float = 30.0,
):
    """
    Graceful shutdown sequence for async trading system.

    Order matters:
    1. Stop accepting new work
    2. Drain pending work
    3. Cancel remaining tasks
    4. Wait for cleanup
    """
    logger.info("Initiating graceful shutdown...")

    # 1. Stop generating new signals
    await registry.signal_stop("signal-engine")

    # 2. Drain the intent bus
    try:
        async with asyncio.timeout(timeout / 3):
            await bus.drain()
            logger.info("Intent bus drained")
    except TimeoutError:
        logger.warning("Bus drain timed out, continuing shutdown")

    # 3. Wait for pending orders to fill or timeout
    try:
        async with asyncio.timeout(timeout / 3):
            await gateway.wait_for_pending()
            logger.info("All pending orders settled")
    except TimeoutError:
        logger.warning("Pending orders did not settle, cancelling")
        await gateway.cancel_remaining()

    # 4. Cancel all remaining tasks
    await registry.shutdown_all()

    logger.info("Graceful shutdown complete")
```

---

## Common Async Patterns for Trading

### Timeout-Protected External Calls

```python
# BAD: No timeout -- hangs forever if broker is unresponsive
result = await broker.get_positions()

# GOOD: Explicit timeout with clear error
try:
    async with asyncio.timeout(5.0):
        result = await broker.get_positions()
except TimeoutError:
    logger.error("Broker get_positions timed out after 5s")
    raise BrokerTimeoutError("get_positions") from None
```

### Structured Concurrency for Parallel Operations

```python
# BAD: Unstructured -- if one fails, others are orphaned
tasks = [asyncio.create_task(check_position(s)) for s in symbols]
results = await asyncio.gather(*tasks)

# GOOD: TaskGroup for structured concurrency (Python 3.11+)
async def check_all_positions(symbols: list[str]) -> list[Position]:
    results = []
    async with asyncio.TaskGroup() as tg:
        for symbol in symbols:
            tg.create_task(check_position(symbol))
    # If ANY task fails, ALL are cancelled and the exception propagates
```

### Periodic Tasks with Error Boundaries

```python
async def periodic_reconciliation(
    interval: float = 60.0,
    heartbeat: HeartbeatMonitor | None = None,
):
    """Run reconciliation periodically with proper error handling."""
    consecutive_failures = 0
    MAX_CONSECUTIVE_FAILURES = 3

    while True:
        try:
            if heartbeat:
                heartbeat.beat("reconciliation")

            await reconcile_with_broker()
            consecutive_failures = 0

        except asyncio.CancelledError:
            raise  # ALWAYS propagate cancellation

        except ReconciliationError as e:
            consecutive_failures += 1
            logger.error(
                f"Reconciliation failed "
                f"({consecutive_failures}/{MAX_CONSECUTIVE_FAILURES}): {e}"
            )
            if consecutive_failures >= MAX_CONSECUTIVE_FAILURES:
                logger.critical("Reconciliation failed too many times.")
                raise

        except Exception:
            logger.exception("Unexpected error in reconciliation")
            raise  # Unknown errors always propagate

        await asyncio.sleep(interval)
```

---

## Red Flags

| Red Flag                                          | Why It Is Dangerous                                             | Correct Pattern                                |
|---------------------------------------------------|-----------------------------------------------------------------|------------------------------------------------|
| `asyncio.create_task()` without `add_done_callback`| Exception stored on Task, never retrieved, silently lost        | Always add done_callback                       |
| `except Exception: pass` or `except: pass`        | Silences ALL errors; task continues with corrupted state        | Specific exceptions, log, re-raise or alert    |
| Function returning `None` on error                | Caller cannot distinguish "no data" from "error"                | Raise exception or use Result type             |
| `await asyncio.gather(*tasks)` without handling   | First exception cancels gather but orphans other tasks          | Use `TaskGroup` or handle exceptions explicitly|
| Long-running loop without heartbeat               | Stuck loop is invisible; appears healthy                        | Emit heartbeat every iteration                 |
| Catching `CancelledError` without re-raising      | Task ignores shutdown signals                                   | Let CancelledError propagate (never swallow)   |
| No timeout on external calls (broker, DB)         | Hangs forever on network issues                                 | `async with asyncio.timeout(N):`               |
| Background task accessing shared mutable state    | Race conditions; inconsistent reads                             | Single owner pattern (see architecture skill)  |
| `task.result()` without checking `task.done()`    | Blocks or raises `InvalidStateError`                            | Use done_callback or check `done()` first      |

---

## Testing Async Reliability

```python
import asyncio
import pytest


@pytest.mark.asyncio
async def test_task_failure_triggers_callback():
    """Verify that task failures are never silent."""
    callback_called = asyncio.Event()
    captured_exception = None

    def on_done(task):
        nonlocal captured_exception
        if not task.cancelled():
            captured_exception = task.exception()
        callback_called.set()

    async def failing_task():
        raise RuntimeError("Simulated failure")

    task = asyncio.create_task(failing_task(), name="test-failure")
    task.add_done_callback(on_done)

    await callback_called.wait()
    assert captured_exception is not None
    assert "Simulated failure" in str(captured_exception)


@pytest.mark.asyncio
async def test_no_except_pass_in_codebase():
    """Scan for except:pass -- the silent killer."""
    import ast
    import pathlib

    violations = []
    for py_file in pathlib.Path("src/").rglob("*.py"):
        tree = ast.parse(py_file.read_text())
        for node in ast.walk(tree):
            if isinstance(node, ast.ExceptHandler):
                if (
                    len(node.body) == 1
                    and isinstance(node.body[0], ast.Pass)
                ):
                    violations.append(f"{py_file}:{node.lineno}")

    assert violations == [], (
        f"Found except:pass (the silent killer):\n"
        + "\n".join(violations)
    )


@pytest.mark.asyncio
async def test_heartbeat_detects_stuck_task():
    """Verify heartbeat monitor catches stuck tasks."""
    monitor = HeartbeatMonitor()
    monitor.beat("test-task")

    # Simulate time passing beyond timeout
    monitor._heartbeats["test-task"] = (
        datetime.now(timezone.utc) - timedelta(seconds=120)
    )

    stale = monitor.check_all()
    assert "test-task" in stale
```

---

## Integration

| Scenario                                  | Invoke Skill                           |
|-------------------------------------------|----------------------------------------|
| Setting up component architecture         | `trading-bot-architecture`             |
| Task handles database operations          | `database-safety-for-trading`          |
| Task monitors order execution             | `order-execution-integrity`            |
| Task performs risk calculations            | `risk-management-gates`                |
| WebSocket connection management           | `websocket-lifecycle`                  |
| Testing async behavior                    | `trading-tdd`                          |
| Monitoring task health                    | `monitoring-and-alerting`              |
| Task death triggers emergency procedures  | `disaster-recovery`                    |
