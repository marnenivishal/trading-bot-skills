# Event-Driven Trading Bot: Reference Implementation

Complete reference implementation showing the event-driven architecture
described in the `trading-bot-architecture` skill. This is a structural
guide, not production-ready code -- adapt to your broker SDK and requirements.

---

## Project Structure

```
trading_bot/
    __init__.py
    main.py                    # Entry point, startup/shutdown
    config.py                  # Configuration loading
    events.py                  # Typed event definitions
    bus.py                     # Message bus implementation
    components/
        __init__.py
        base.py                # TradingComponent base class
        market_data.py         # Market data feed
        signal_engine.py       # Strategy evaluation
        dedup_gate.py          # Duplicate intent rejection
        risk_gate.py           # Risk limit enforcement
        execution_gateway.py   # Order execution (SINGLE entry point)
        position_manager.py    # Position state ownership
        reconciler.py          # Broker-local state sync
    strategies/
        __init__.py
        base.py                # Strategy base class
        ema_crossover.py       # Example strategy
    broker/
        __init__.py
        interface.py           # Broker ABC
        alpaca.py              # Broker implementation example
```

---

## Typed Events

```python
"""events.py -- All events in the system are defined here."""

from __future__ import annotations

import uuid
from dataclasses import dataclass, field
from datetime import datetime, timezone
from decimal import Decimal
from enum import Enum
from typing import Optional


class EventType(Enum):
    PRICE_UPDATE = "price_update"
    ORDER_INTENT = "order_intent"
    FLATTEN_INTENT = "flatten_intent"
    ORDER_SUBMITTED = "order_submitted"
    ORDER_FILLED = "order_filled"
    ORDER_REJECTED = "order_rejected"
    ORDER_CANCELLED = "order_cancelled"
    POSITION_CHANGED = "position_changed"
    RISK_BREACH = "risk_breach"
    HEARTBEAT = "heartbeat"


class Side(Enum):
    BUY = "buy"
    SELL = "sell"


class OrderType(Enum):
    MARKET = "market"
    LIMIT = "limit"
    STOP = "stop"
    STOP_LIMIT = "stop_limit"


@dataclass(frozen=True)
class Event:
    """Base event. All events are immutable."""
    event_id: str = field(default_factory=lambda: str(uuid.uuid4()))
    timestamp: datetime = field(default_factory=lambda: datetime.now(timezone.utc))
    event_type: EventType = field(init=False)


@dataclass(frozen=True)
class PriceUpdate(Event):
    event_type: EventType = field(default=EventType.PRICE_UPDATE, init=False)
    symbol: str = ""
    bid: Decimal = Decimal("0")
    ask: Decimal = Decimal("0")
    last: Decimal = Decimal("0")
    volume: int = 0


@dataclass(frozen=True)
class OrderIntent(Event):
    """
    An ORDER INTENT, not an order. Strategies emit intents.
    Only the ExecutionGateway converts intents to actual orders.
    """
    event_type: EventType = field(default=EventType.ORDER_INTENT, init=False)
    source: str = ""                    # Which strategy/component emitted this
    symbol: str = ""
    side: Side = Side.BUY
    qty: Decimal = Decimal("0")
    order_type: OrderType = OrderType.MARKET
    limit_price: Optional[Decimal] = None
    stop_price: Optional[Decimal] = None
    idempotency_key: str = ""           # REQUIRED. Must be unique per logical intent.
    reason: str = ""                    # Human-readable reason for audit trail


@dataclass(frozen=True)
class FlattenIntent(Event):
    """Intent to close all positions. Emergency or controlled."""
    event_type: EventType = field(default=EventType.FLATTEN_INTENT, init=False)
    source: str = ""
    reason: str = ""
    idempotency_key: str = ""
    symbols: tuple[str, ...] = ()       # Empty = flatten ALL


@dataclass(frozen=True)
class OrderFilled(Event):
    event_type: EventType = field(default=EventType.ORDER_FILLED, init=False)
    broker_order_id: str = ""
    symbol: str = ""
    side: Side = Side.BUY
    qty: Decimal = Decimal("0")
    fill_price: Decimal = Decimal("0")
    idempotency_key: str = ""


@dataclass(frozen=True)
class PositionChanged(Event):
    event_type: EventType = field(default=EventType.POSITION_CHANGED, init=False)
    symbol: str = ""
    old_qty: Decimal = Decimal("0")
    new_qty: Decimal = Decimal("0")
    cause_event_id: str = ""            # The fill/reconciliation that caused this
```

---

## Message Bus

```python
"""bus.py -- Central message bus. All inter-component communication goes here."""

from __future__ import annotations

import asyncio
import logging
from collections import defaultdict
from typing import Awaitable, Callable

from .events import Event, EventType

logger = logging.getLogger(__name__)

Handler = Callable[[Event], Awaitable[None]]


class IntentBus:
    """
    Central event bus for the trading system.

    Rules:
    - All inter-component communication goes through this bus.
    - Handlers are async and must not block.
    - Handlers MUST NOT swallow exceptions (see async-reliability skill).
    - Events are immutable; handlers cannot modify them.
    """

    def __init__(self) -> None:
        self._handlers: dict[EventType, list[Handler]] = defaultdict(list)
        self._queue: asyncio.Queue[Event] = asyncio.Queue()
        self._running = False
        self._process_task: asyncio.Task | None = None

    def subscribe(self, event_type: EventType, handler: Handler) -> None:
        """Register a handler for an event type."""
        self._handlers[event_type].append(handler)
        logger.info(
            f"Subscribed {handler.__qualname__} to {event_type.value}"
        )

    async def emit(self, event: Event) -> None:
        """Emit an event onto the bus. Non-blocking."""
        if not event.idempotency_key and hasattr(event, "idempotency_key"):
            raise ValueError(
                f"Event {event.event_type.value} missing idempotency_key. "
                f"Every intent MUST have an idempotency key."
            )
        await self._queue.put(event)

    async def start(self) -> None:
        """Start processing events from the queue."""
        self._running = True
        self._process_task = asyncio.create_task(
            self._process_loop(), name="intent-bus-processor"
        )
        self._process_task.add_done_callback(self._on_processor_done)

    async def stop(self) -> None:
        """Stop the bus. Drain remaining events first."""
        self._running = False
        if self._process_task:
            self._process_task.cancel()
            try:
                await self._process_task
            except asyncio.CancelledError:
                pass

    async def drain(self, timeout: float = 10.0) -> None:
        """Process all remaining events in the queue."""
        try:
            async with asyncio.timeout(timeout):
                while not self._queue.empty():
                    event = self._queue.get_nowait()
                    await self._dispatch(event)
        except TimeoutError:
            remaining = self._queue.qsize()
            logger.warning(f"Bus drain timed out. {remaining} events remaining.")

    async def _process_loop(self) -> None:
        """Main event processing loop."""
        while self._running:
            try:
                event = await asyncio.wait_for(self._queue.get(), timeout=1.0)
                await self._dispatch(event)
            except TimeoutError:
                continue  # Check self._running
            except asyncio.CancelledError:
                raise
            except Exception:
                logger.exception("Unhandled error in bus processor")
                raise  # Do NOT swallow. See async-reliability skill.

    async def _dispatch(self, event: Event) -> None:
        """Dispatch event to all registered handlers."""
        handlers = self._handlers.get(event.event_type, [])
        for handler in handlers:
            try:
                await handler(event)
            except Exception:
                logger.exception(
                    f"Handler {handler.__qualname__} failed on "
                    f"{event.event_type.value} ({event.event_id})"
                )
                raise  # Handler failure must propagate

    def _on_processor_done(self, task: asyncio.Task) -> None:
        """Callback when processor task completes."""
        if not task.cancelled() and task.exception():
            logger.critical(
                f"Intent bus processor died: {task.exception()}"
            )
            # In production: trigger emergency shutdown
```

---

## Component Base Class

```python
"""components/base.py -- Base class for all trading components."""

from __future__ import annotations

import asyncio
import logging
from abc import ABC, abstractmethod

from ..bus import IntentBus

logger = logging.getLogger(__name__)


class TradingComponent(ABC):
    """
    Base class enforcing component boundaries.

    Every component MUST:
    - Implement start() and stop()
    - Use _create_task() for all background tasks
    - Communicate only via the IntentBus
    - Own its state or query another component (never both)
    """

    def __init__(self, bus: IntentBus, *, name: str = "") -> None:
        self._bus = bus
        self._name = name or self.__class__.__name__
        self._tasks: set[asyncio.Task] = set()
        self._logger = logging.getLogger(f"trading.{self._name}")

    @abstractmethod
    async def start(self) -> None:
        """Initialize and begin processing."""
        ...

    @abstractmethod
    async def stop(self) -> None:
        """Graceful shutdown."""
        for task in list(self._tasks):
            task.cancel()
        if self._tasks:
            await asyncio.gather(*self._tasks, return_exceptions=True)
        self._tasks.clear()

    def _create_task(self, coro, *, name: str) -> asyncio.Task:
        """Create a tracked task with mandatory done_callback."""
        task_name = f"{self._name}.{name}"
        task = asyncio.create_task(coro, name=task_name)
        self._tasks.add(task)
        task.add_done_callback(self._tasks.discard)
        task.add_done_callback(self._on_task_done)
        return task

    def _on_task_done(self, task: asyncio.Task) -> None:
        """Handle task completion. NEVER silently ignore failures."""
        if task.cancelled():
            self._logger.info(f"Task {task.get_name()} cancelled")
            return
        exc = task.exception()
        if exc:
            self._logger.error(
                f"Task {task.get_name()} failed: {exc}",
                exc_info=exc,
            )
            # Subclasses can override to trigger emergency shutdown
```

---

## Dedup Gate

```python
"""components/dedup_gate.py -- Rejects duplicate order intents."""

from __future__ import annotations

import logging
import time
from collections import OrderedDict

from ..bus import IntentBus
from ..events import EventType, OrderIntent
from .base import TradingComponent

logger = logging.getLogger(__name__)


class DedupGate(TradingComponent):
    """
    Rejects duplicate OrderIntents by idempotency_key.

    Maintains a time-bounded cache of recently seen keys.
    If the same key appears within the TTL window, the intent is dropped.
    """

    def __init__(
        self,
        bus: IntentBus,
        *,
        ttl_seconds: float = 300.0,  # 5 minute dedup window
        max_cache_size: int = 10_000,
    ) -> None:
        super().__init__(bus, name="DedupGate")
        self._ttl = ttl_seconds
        self._max_cache = max_cache_size
        self._seen: OrderedDict[str, float] = OrderedDict()

    async def start(self) -> None:
        self._bus.subscribe(EventType.ORDER_INTENT, self._handle_intent)

    async def stop(self) -> None:
        await super().stop()

    async def _handle_intent(self, event: OrderIntent) -> None:
        key = event.idempotency_key
        now = time.monotonic()

        # Evict expired entries
        while self._seen:
            oldest_key, oldest_time = next(iter(self._seen.items()))
            if now - oldest_time > self._ttl:
                self._seen.pop(oldest_key)
            else:
                break

        # Check for duplicate
        if key in self._seen:
            self._logger.warning(
                f"DEDUP: Rejected duplicate intent {key} "
                f"from {event.source} for {event.symbol}"
            )
            return  # Drop the duplicate. Do NOT re-emit.

        # Record and forward
        self._seen[key] = now
        if len(self._seen) > self._max_cache:
            self._seen.popitem(last=False)

        # Re-emit as a deduplicated event for the risk gate
        # (In practice, you might use a separate event type or channel)
        self._logger.info(
            f"DEDUP: Passed intent {key} from {event.source} for {event.symbol}"
        )
```

---

## Example Strategy

```python
"""strategies/ema_crossover.py -- Example strategy showing correct boundaries."""

from __future__ import annotations

from decimal import Decimal

from ..bus import IntentBus
from ..components.base import TradingComponent
from ..components.position_manager import PositionManager
from ..events import EventType, OrderIntent, PriceUpdate, Side


class EmaCrossoverStrategy(TradingComponent):
    """
    Example EMA crossover strategy.

    NOTE: This strategy NEVER imports the broker module.
    It ONLY emits OrderIntents onto the bus.
    The execution gateway handles everything downstream.
    """

    def __init__(
        self,
        bus: IntentBus,
        position_manager: PositionManager,
        *,
        fast_period: int = 9,
        slow_period: int = 21,
        symbol: str = "SPY",
        position_size: Decimal = Decimal("100"),
    ) -> None:
        super().__init__(bus, name="EmaCrossover")
        self._pm = position_manager       # Query positions, never track our own
        self._fast_period = fast_period
        self._slow_period = slow_period
        self._symbol = symbol
        self._position_size = position_size
        self._prices: list[Decimal] = []
        self._signal_counter = 0

    async def start(self) -> None:
        self._bus.subscribe(EventType.PRICE_UPDATE, self._on_price)

    async def stop(self) -> None:
        await super().stop()

    async def _on_price(self, event: PriceUpdate) -> None:
        if event.symbol != self._symbol:
            return

        self._prices.append(event.last)

        if len(self._prices) < self._slow_period:
            return

        fast_ema = self._calc_ema(self._prices, self._fast_period)
        slow_ema = self._calc_ema(self._prices, self._slow_period)

        current_position = self._pm.get(self._symbol)

        # Bullish crossover: fast > slow and not already long
        if fast_ema > slow_ema and current_position.qty <= 0:
            self._signal_counter += 1
            await self._bus.emit(OrderIntent(
                source="ema_crossover",
                symbol=self._symbol,
                side=Side.BUY,
                qty=self._position_size,
                idempotency_key=f"ema-cross-buy-{self._signal_counter}",
                reason=f"Bullish crossover: fast={fast_ema:.2f} > slow={slow_ema:.2f}",
            ))

        # Bearish crossover: fast < slow and not already short/flat
        elif fast_ema < slow_ema and current_position.qty > 0:
            self._signal_counter += 1
            await self._bus.emit(OrderIntent(
                source="ema_crossover",
                symbol=self._symbol,
                side=Side.SELL,
                qty=current_position.qty,
                idempotency_key=f"ema-cross-sell-{self._signal_counter}",
                reason=f"Bearish crossover: fast={fast_ema:.2f} < slow={slow_ema:.2f}",
            ))

    @staticmethod
    def _calc_ema(prices: list[Decimal], period: int) -> Decimal:
        """Simple EMA calculation."""
        if len(prices) < period:
            return Decimal("0")
        multiplier = Decimal(2) / (Decimal(period) + 1)
        ema = sum(prices[-period:]) / Decimal(period)
        for price in prices[-period:]:
            ema = (price - ema) * multiplier + ema
        return ema
```

---

## Main Entry Point

```python
"""main.py -- Application entry point with correct startup/shutdown."""

import asyncio
import logging
import signal

from .bus import IntentBus
from .components.dedup_gate import DedupGate
from .components.execution_gateway import ExecutionGateway
from .components.market_data import MarketDataFeed
from .components.position_manager import PositionManager
from .components.reconciler import Reconciler
from .components.risk_gate import RiskGate
from .strategies.ema_crossover import EmaCrossoverStrategy

logger = logging.getLogger(__name__)


async def main() -> None:
    bus = IntentBus()

    # Build components (dependency injection, no global state)
    position_manager = PositionManager(bus)
    reconciler = Reconciler(bus, position_manager)
    dedup_gate = DedupGate(bus, ttl_seconds=300.0)
    risk_gate = RiskGate(bus, position_manager)
    execution_gateway = ExecutionGateway(bus, position_manager)
    market_data = MarketDataFeed(bus)
    strategy = EmaCrossoverStrategy(bus, position_manager)

    # Startup sequence (ORDER MATTERS -- see trading-bot-architecture skill)
    await bus.start()
    await execution_gateway.start()     # Ready to execute before signals
    await risk_gate.start()             # Ready to check before signals
    await dedup_gate.start()            # Ready to dedup before signals
    await position_manager.start()

    # Reconcile BEFORE processing signals
    await reconciler.reconcile_with_broker()

    await market_data.start()           # Start data feed
    await strategy.start()              # Start generating signals LAST

    logger.info("Trading bot started. All systems active.")

    # Wait for shutdown signal
    shutdown_event = asyncio.Event()
    loop = asyncio.get_event_loop()
    for sig in (signal.SIGINT, signal.SIGTERM):
        loop.add_signal_handler(sig, shutdown_event.set)

    await shutdown_event.wait()

    # Shutdown sequence (REVERSE ORDER)
    logger.info("Shutting down...")
    await strategy.stop()               # Stop signals FIRST
    await market_data.stop()
    await bus.drain(timeout=10.0)       # Process remaining events
    await execution_gateway.stop()
    await risk_gate.stop()
    await dedup_gate.stop()
    await position_manager.stop()
    await reconciler.final_check()      # Final reconciliation
    await bus.stop()

    logger.info("Trading bot stopped cleanly.")


if __name__ == "__main__":
    logging.basicConfig(level=logging.INFO)
    asyncio.run(main())
```

---

## Key Principles Demonstrated

1. **No broker imports in strategies** -- EmaCrossoverStrategy only uses IntentBus
2. **Typed events** -- All events are frozen dataclasses with explicit fields
3. **Single entry point** -- Only ExecutionGateway talks to the broker
4. **State ownership** -- PositionManager is THE source of truth
5. **Startup order** -- Reconcile before processing signals
6. **Shutdown order** -- Stop signals first, drain bus, then infrastructure
7. **Task tracking** -- All tasks created via _create_task with done_callback
8. **Dedup at the gate** -- Before risk checks, before execution
9. **Idempotency keys required** -- Bus rejects intents without them
10. **No except:pass** -- All exceptions logged and propagated
