---
name: distributed-trading-patterns
description: Use when scaling a trading bot beyond single-process architecture, implementing event streaming with Kafka or NATS, decomposing into microservices, or building multi-bot coordination systems
---

# Distributed Trading Patterns

Companion to `trading-bot-architecture` (~467 lines), which covers single-process,
local event-driven design. This skill covers scaling BEYOND single-process: event
streaming, microservices decomposition, multi-bot coordination, and distributed
safety mechanisms.

The single-process architecture is correct for most trading bots. Do not distribute
prematurely. Distribution adds latency, failure modes, and operational complexity.
Only cross this boundary when you have a concrete scaling trigger.

---

## When to Go Distributed

Not every trading bot needs microservices. Most do not. Distribute only when you
hit one or more of these triggers:

| Trigger | Threshold | Why It Forces Distribution |
|---|---|---|
| Strategy count | >5 active strategies | CPU contention in signal computation starves other strategies |
| Symbol coverage | >50 symbols | Market data ingestion saturates single-process I/O |
| Latency requirement | <10ms signal-to-order | GC pauses and event loop contention break latency SLAs |
| Asset/broker diversity | Multi-asset, multi-broker | Broker API differences pollute shared execution code |
| Team size | >3 developers | Merge conflicts and deployment coupling slow iteration |

If you are not hitting these triggers, stay single-process. Reference
`trading-bot-architecture` for the correct single-process event-driven design.

### Single-Process vs Distributed Tradeoffs

| Dimension | Single-Process | Distributed |
|---|---|---|
| Latency | Microseconds (in-process events) | Milliseconds (network serialization) |
| Failure modes | Process crash = total failure, simple restart | Partial failures, split-brain, message loss |
| Debugging | Single stack trace, single log file | Distributed tracing, correlated logs, span trees |
| Deployment | One binary/container | Orchestration (K8s), service mesh, config sync |
| State consistency | In-memory, trivially consistent | Eventual consistency, distributed locks |
| Kill switch | Single flag, instant propagation | Heartbeat-based, independent service required |
| Operational cost | Low: one process to monitor | High: N services, message bus, observability stack |

> **Rule: Distribute the component, not the logic.** Each service still follows
> the same event-driven pipeline from `trading-bot-architecture`. The Intent Bus
> becomes a message queue. The Risk Gate becomes a service. The pipeline shape
> does not change -- only the transport between components changes.

---

## Event Streaming: Kafka/Pulsar

For high-throughput, durable event streaming with replay capability, strict
ordering guarantees per symbol, and audit-grade retention.

### Topic Topology

| Topic | Partition Key | Retention | Purpose |
|---|---|---|---|
| `market-data.{symbol}` | symbol | 24 hours | Normalized price/quote events |
| `order-intents` | symbol | 7 days | Strategy-emitted order intents |
| `fills` | order_id | Forever | Execution confirmations from broker |
| `risk-events` | strategy_id | 90 days | Risk gate decisions, circuit breaker trips |
| `kill-switch-commands` | None (single partition) | 30 days | Emergency halt commands |

**Partition by symbol** to guarantee ordering within a symbol. All events for AAPL
go to the same partition, processed by the same consumer.

**Never partition order-intents by strategy.** Two strategies trading the same
symbol must have their intents ordered relative to each other for the risk gate
to reason about aggregate exposure.

### KafkaTradeEventProducer

```python
from dataclasses import dataclass, asdict
from confluent_kafka import Producer

@dataclass
class TradeEvent:
    event_type: str          # "order_intent" | "fill" | "risk_event"
    symbol: str
    strategy_id: str
    timestamp_ns: int
    trace_id: str
    payload: dict

class KafkaTradeEventProducer:
    """Produces trade events to Kafka with delivery confirmation."""

    def __init__(self, bootstrap_servers: str, client_id: str) -> None:
        self._producer = Producer({
            "bootstrap.servers": bootstrap_servers,
            "client.id": client_id,
            "acks": "all",               # Wait for all replicas
            "enable.idempotence": True,   # Exactly-once per partition
            "max.in.flight.requests.per.connection": 5,
            "retries": 3,
            "linger.ms": 5,              # Batch for throughput
        })
        self._delivery_failures: int = 0

    def _on_delivery(self, err, msg) -> None:
        if err is not None:
            self._delivery_failures += 1
            logger.error("Delivery failed: topic=%s err=%s", msg.topic(), err)
            if self._delivery_failures > 10:
                logger.critical("Excessive delivery failures -- trigger kill switch")

    def send_event(self, topic: str, event: TradeEvent) -> None:
        """Send event with symbol as partition key for ordering."""
        self._producer.produce(
            topic=topic,
            key=event.symbol.encode("utf-8"),
            value=json.dumps(asdict(event)).encode("utf-8"),
            headers={"trace_id": event.trace_id.encode("utf-8")},
            callback=self._on_delivery,
        )
        self._producer.poll(0)  # Trigger delivery callbacks
```

### KafkaTradeEventConsumer

```python
from confluent_kafka import Consumer, KafkaError, KafkaException

class KafkaTradeEventConsumer:
    """Consumes trade events with at-least-once delivery guarantees."""

    def __init__(
        self,
        bootstrap_servers: str,
        group_id: str,
        topics: list[str],
        handler: Callable[[TradeEvent], None],
    ) -> None:
        self._consumer = Consumer({
            "bootstrap.servers": bootstrap_servers,
            "group.id": group_id,
            "auto.offset.reset": "latest",
            "enable.auto.commit": False,  # Manual commit after processing
            "max.poll.interval.ms": 30000,
        })
        self._consumer.subscribe(topics)
        self._handler = handler
        self._running = True

    def run(self) -> None:
        """Main consume loop. Call from a dedicated thread."""
        try:
            while self._running:
                msg = self._consumer.poll(timeout=1.0)
                if msg is None:
                    continue
                if msg.error():
                    if msg.error().code() == KafkaError._PARTITION_EOF:
                        continue
                    raise KafkaException(msg.error())
                event = TradeEvent(**json.loads(msg.value().decode("utf-8")))
                try:
                    self._handler(event)
                    self._consumer.commit(msg)  # Commit AFTER successful processing
                except Exception:
                    logger.exception("Handler failed for trace_id=%s", event.trace_id)
                    # Do NOT commit. Message will be redelivered.
        finally:
            self._consumer.close()
```

**Consumer groups:** One group per service (`risk-gate-group`, `execution-group`).
Each group independently tracks offsets. Scale consumers within a group up to the
number of partitions.

---

## Lightweight Pub/Sub: NATS

When Kafka is overkill. Use NATS for low-latency signal sharing between bots,
request-reply risk checks, and lightweight coordination.

| Criterion | NATS | Kafka |
|---|---|---|
| Message durability | Optional (JetStream) | Always durable |
| Ordering guarantees | Per-subject (JetStream) | Per-partition |
| Operational complexity | Single binary, zero config | Broker cluster + controller |
| Use case | Signal sharing, health checks | Audit trail, replay, high-durability |

### Subject Hierarchy

```
trading.signals.{strategy}.{symbol}    # Signal emissions
trading.risk.check                     # Request-reply risk checks
trading.risk.events                    # Risk gate decisions
trading.killswitch.{level}             # Kill switch commands
trading.heartbeat.{service}            # Service liveness
```

Wildcard subscriptions: `trading.signals.>` receives all signals across all
strategies and symbols.

### NATSSignalBroadcaster

```python
import nats
from nats.aio.client import Client as NATSClient

@dataclass
class TradingSignal:
    strategy_id: str
    symbol: str
    direction: str         # "long" | "short" | "flat"
    strength: float        # 0.0 to 1.0
    timestamp_ns: int
    trace_id: str
    metadata: dict

class NATSSignalBroadcaster:
    """Broadcasts trading signals over NATS for multi-bot coordination."""

    def __init__(self, servers: str = "nats://localhost:4222") -> None:
        self._nc: Optional[NATSClient] = None
        self._js = None  # JetStream context

    async def connect(self) -> None:
        self._nc = await nats.connect(
            self._servers,
            reconnected_cb=lambda: logger.warning("NATS reconnected"),
            disconnected_cb=lambda: logger.error("NATS disconnected"),
            max_reconnect_attempts=10,
        )
        self._js = self._nc.jetstream()
        await self._js.add_stream(
            name="SIGNALS", subjects=["trading.signals.>"],
            retention="limits", max_age=3600,
        )

    async def broadcast_signal(self, signal: TradingSignal) -> None:
        subject = f"trading.signals.{signal.strategy_id}.{signal.symbol}"
        ack = await self._js.publish(subject, json.dumps(asdict(signal)).encode())
        logger.debug("Signal published: stream=%s seq=%d", ack.stream, ack.seq)

    async def request_risk_check(
        self, signal: TradingSignal, timeout: float = 0.5
    ) -> dict:
        """Synchronous request-reply risk check. Timeout = reject."""
        try:
            response = await self._nc.request(
                "trading.risk.check",
                json.dumps(asdict(signal)).encode(), timeout=timeout,
            )
            return json.loads(response.data.decode())
        except asyncio.TimeoutError:
            logger.warning("Risk check timeout for %s -- REJECTING", signal.symbol)
            return {"approved": False, "reason": "risk_check_timeout"}
```

**Multi-bot signal sharing:** Bot A publishes to `trading.signals.momentum.AAPL`.
Bot B subscribes to `trading.signals.>` and uses signals as confirming indicators.
Each bot maintains its own risk gates -- shared signals never bypass local risk checks.

---

## Actor Model

Each trading component is an actor with its own mailbox (asyncio queue). No shared
mutable state. Supervision trees restart failed actors automatically.

### TradingActor Base Class

```python
from abc import ABC, abstractmethod
from enum import Enum

class ActorState(Enum):
    RUNNING = "running"
    STOPPED = "stopped"
    FAILED = "failed"

@dataclass
class ActorMessage:
    sender: str
    msg_type: str
    payload: Any
    trace_id: str

class TradingActor(ABC):
    """Base actor with mailbox, lifecycle, and supervision support."""

    def __init__(self, name: str, mailbox_size: int = 1000,
                 parent: Optional["TradingActor"] = None) -> None:
        self.name = name
        self._mailbox: asyncio.Queue[ActorMessage] = asyncio.Queue(maxsize=mailbox_size)
        self._state = ActorState.STOPPED
        self._parent = parent
        self._children: dict[str, "TradingActor"] = {}
        self._restart_count: int = 0
        self._max_restarts: int = 3

    async def start(self) -> None:
        self._state = ActorState.RUNNING
        self._task = asyncio.create_task(self._run_loop(), name=f"actor-{self.name}")
        self._task.add_done_callback(self._on_task_done)

    async def _run_loop(self) -> None:
        while self._state == ActorState.RUNNING:
            try:
                msg = await asyncio.wait_for(self._mailbox.get(), timeout=5.0)
                await self.handle_message(msg)
            except asyncio.TimeoutError:
                continue  # Idle -- override on_idle() for health checks
            except Exception as e:
                self._state = ActorState.FAILED
                if self._parent:
                    await self._parent.on_child_failure(self, e)
                raise

    @abstractmethod
    async def handle_message(self, msg: ActorMessage) -> None: ...

    async def send(self, msg: ActorMessage) -> None:
        try:
            self._mailbox.put_nowait(msg)
        except asyncio.QueueFull:
            logger.error("Actor %s mailbox full -- dropping %s", self.name, msg.msg_type)

    async def on_child_failure(self, child: "TradingActor", error: Exception) -> None:
        """Supervision: restart failed child if under restart limit."""
        child._restart_count += 1
        if child._restart_count <= child._max_restarts:
            logger.warning("Restarting %s (attempt %d/%d)",
                           child.name, child._restart_count, child._max_restarts)
            await child.stop()
            await child.start()
        else:
            logger.critical("Child %s exceeded max restarts -- escalating", child.name)
            self._state = ActorState.FAILED
            if self._parent:
                await self._parent.on_child_failure(self, error)

    async def stop(self) -> None:
        self._state = ActorState.STOPPED
        for child in self._children.values():
            await child.stop()
        if hasattr(self, "_task") and not self._task.done():
            self._task.cancel()

    def _on_task_done(self, task: asyncio.Task) -> None:
        if not task.cancelled() and task.exception():
            logger.error("Actor %s died: %s", self.name, task.exception())
```

---

## In-Memory Data Grids

Shared state across services for positions and risk limits.

### RedisPositionCache

```python
import redis.asyncio as redis

@dataclass
class PositionState:
    symbol: str
    quantity: int
    avg_entry_price: float
    unrealized_pnl: float
    strategy_id: str
    last_updated_ns: int

class RedisPositionCache:
    """Shared position state with TTL-based staleness detection."""

    POSITION_TTL_SECONDS: int = 30  # Must be refreshed within 30s

    def __init__(self, redis_url: str = "redis://localhost:6379") -> None:
        self._redis = redis.from_url(redis_url, decode_responses=True)

    async def update_position(self, pos: PositionState) -> None:
        """Update position with TTL. If not refreshed, key expires = stale."""
        key = f"positions:{pos.strategy_id}:{pos.symbol}"
        await self._redis.setex(key, self.POSITION_TTL_SECONDS, json.dumps(asdict(pos)))
        await self._redis.publish(f"position_updates:{pos.strategy_id}", json.dumps(asdict(pos)))

    async def get_position(self, strategy_id: str, symbol: str) -> Optional[PositionState]:
        """Returns None if expired (stale)."""
        data = await self._redis.get(f"positions:{strategy_id}:{symbol}")
        return PositionState(**json.loads(data)) if data else None

    async def get_total_exposure(self) -> float:
        total = 0.0
        async for key in self._redis.scan_iter(match="positions:*"):
            data = await self._redis.get(key)
            if data:
                pos = PositionState(**json.loads(data))
                total += abs(pos.quantity * pos.avg_entry_price)
        return total
```

### DistributedRiskLimits

```python
class DistributedRiskLimits:
    """Atomic risk limit enforcement across concurrent services."""

    # Lua script: atomic check-and-increment
    _CHECK_AND_INCREMENT = """
    local current = tonumber(redis.call('GET', KEYS[1]) or '0')
    if current + tonumber(ARGV[1]) <= tonumber(ARGV[2]) then
        redis.call('INCRBYFLOAT', KEYS[1], ARGV[1])
        return 1
    end
    return 0
    """

    def __init__(self, redis_url: str = "redis://localhost:6379") -> None:
        self._redis = redis.from_url(redis_url, decode_responses=True)

    async def try_allocate_exposure(
        self, strategy_id: str, amount: float, max_exposure: float
    ) -> bool:
        """Atomically check and allocate exposure. Returns True if within limits."""
        key = f"risk_limits:exposure:{strategy_id}"
        result = await self._redis.eval(self._CHECK_AND_INCREMENT, 1, key, str(amount), str(max_exposure))
        if result != 1:
            logger.warning("Exposure rejected: strategy=%s amount=%.2f limit=%.2f", strategy_id, amount, max_exposure)
        return result == 1

    async def release_exposure(self, strategy_id: str, amount: float) -> None:
        await self._redis.incrbyfloat(f"risk_limits:exposure:{strategy_id}", -amount)

    async def record_loss(self, strategy_id: str, loss_amount: float) -> bool:
        """Record a loss and return True if daily limit is breached."""
        new_total = await self._redis.incrbyfloat(f"risk_limits:daily_loss_total:{strategy_id}", loss_amount)
        limit = await self._redis.get(f"risk_limits:daily_loss:{strategy_id}")
        if limit and new_total >= float(limit):
            logger.critical("Daily loss limit breached: strategy=%s total=%.2f", strategy_id, new_total)
            return True
        return False
```

---

## Microservices Decomposition

Split in this order. Each step is independently valuable:

1. **Signal Engine** -- Scales independently per strategy. CPU-intensive indicator
   computation does not block execution or risk checking.
2. **Risk Gate** -- Single source of truth for all strategies. Every order intent
   from every signal engine passes through one risk service.
3. **Execution Gateway** -- Single point of broker contact. Consolidates connection
   management, rate limiting, and order deduplication.

### Service Boundary Interfaces

```python
from abc import ABC, abstractmethod
from enum import Enum

class RiskDecision(Enum):
    APPROVED = "approved"
    REJECTED = "rejected"
    THROTTLED = "throttled"

@dataclass
class OrderIntent:
    intent_id: str
    strategy_id: str
    symbol: str
    side: str              # "buy" | "sell"
    quantity: int
    order_type: str        # "market" | "limit"
    limit_price: Optional[float]
    trace_id: str
    timestamp_ns: int

@dataclass
class RiskCheckResult:
    intent_id: str
    decision: RiskDecision
    reason: str
    exposure_after: float
    trace_id: str

@dataclass
class ExecutionReport:
    intent_id: str
    broker_order_id: str
    fill_price: float
    fill_quantity: int
    status: str            # "filled" | "partial" | "rejected"
    trace_id: str
    timestamp_ns: int

class RiskGateService(ABC):
    @abstractmethod
    async def check_intent(self, intent: OrderIntent) -> RiskCheckResult: ...

class ExecutionGatewayService(ABC):
    @abstractmethod
    async def execute(self, intent: OrderIntent) -> ExecutionReport: ...
    @abstractmethod
    async def cancel_all(self) -> int: ...
```

### Inter-Service Communication

| Path | Protocol | Why |
|---|---|---|
| Signal Engine -> Risk Gate | gRPC (sync) | Risk check must complete before execution proceeds |
| Signal Engine -> Execution Gateway | Message queue (async) | Decoupled, buffered, retryable |
| Execution Gateway -> Signal Engine | Message queue (async) | Fill notifications, non-blocking |
| Kill Switch -> Execution Gateway | gRPC + direct TCP fallback | Must work even if message bus is down |

---

## Kill Switch Across Services

> **The kill switch MUST be independent of all other services.**
>
> If the signal engine crashes, the kill switch still works. If the message bus
> goes down, the kill switch has a direct line to the execution gateway. If the
> risk gate is unresponsive, the kill switch does not depend on it.

### DistributedKillSwitch

```python
from enum import IntEnum

class KillLevel(IntEnum):
    NORMAL = 0
    PAUSE = 1       # Stop new orders
    CANCEL = 2      # Cancel all open orders + pause
    FLATTEN = 3     # Close all positions + cancel + pause

@dataclass
class Heartbeat:
    service_name: str
    timestamp_ns: int
    kill_level: KillLevel
    sequence: int

class DistributedKillSwitch:
    """Independent kill switch with heartbeat-based dead-man's switch.

    Runs as its own service. Does NOT depend on message bus, risk gate,
    or signal engine. Communicates with execution gateway via direct
    gRPC connection (not through message queue).

    Dead-man's switch: if this service dies, execution gateway has not
    received a heartbeat and auto-escalates to PAUSE after timeout.
    """

    HEARTBEAT_INTERVAL_S: float = 1.0
    DEAD_MAN_TIMEOUT_S: float = 5.0  # No heartbeat for 5s = auto-pause

    def __init__(self, execution_gateway_addr: str) -> None:
        self._exec_addr = execution_gateway_addr
        self._current_level = KillLevel.NORMAL
        self._sequence: int = 0
        self._running = False

    async def run(self) -> None:
        """Main loop: send heartbeats to execution gateway."""
        self._running = True
        while self._running:
            self._sequence += 1
            heartbeat = Heartbeat("kill-switch", time.time_ns(),
                                  self._current_level, self._sequence)
            try:
                await self._send_heartbeat(heartbeat)  # Direct gRPC call
            except Exception:
                logger.exception("Heartbeat failed -- gateway should auto-pause")
            await asyncio.sleep(self.HEARTBEAT_INTERVAL_S)

    async def trigger(self, level: KillLevel, reason: str) -> None:
        if level <= self._current_level:
            return
        logger.critical("KILL SWITCH: level=%s reason=%s", level.name, reason)
        self._current_level = level
        # Direct gRPC to execution gateway -- NOT through message bus
        if level >= KillLevel.CANCEL:
            await self._direct_cancel_all_orders()
        if level >= KillLevel.FLATTEN:
            await self._direct_flatten_all_positions()
```

**Dead-man's switch:** The execution gateway tracks last heartbeat timestamp.
If `DEAD_MAN_TIMEOUT_S` elapses without a heartbeat, the gateway auto-escalates
to `PAUSE`. Kill switch failure is itself a safety trigger -- the system fails
safe, not open.

---

## Distributed Tracing

Every event carries a `trace_id` generated at signal creation time. This ID
follows the order through every service boundary.

```
Signal Engine          Risk Gate            Execution Gateway        Broker
    |                      |                      |                    |
    |-- OrderIntent ------>|                      |                    |
    |   trace_id=abc123    |-- RiskCheck -------->|                    |
    |                      |   trace_id=abc123    |-- BrokerOrder ---->|
    |                      |                      |   trace_id=abc123  |
    |                      |                      |<-- Fill -----------|
    |<-- FillNotification --|<-- FillEvent --------|   trace_id=abc123  |
```

### OpenTelemetry Integration

```python
from contextvars import ContextVar

_current_trace: ContextVar[str] = ContextVar("current_trace", default="")
_current_span: ContextVar[str] = ContextVar("current_span", default="")

@dataclass
class TradeSpan:
    trace_id: str
    span_id: str
    parent_span_id: str
    service: str
    operation: str
    start_ns: int
    end_ns: int = 0
    attributes: dict = field(default_factory=dict)

    @property
    def duration_ms(self) -> float:
        return (self.end_ns - self.start_ns) / 1_000_000 if self.end_ns else 0.0

class TradingTracer:
    """Lightweight tracer for trade lifecycle spans."""

    def __init__(self, service_name: str, exporter: Callable) -> None:
        self._service = service_name
        self._exporter = exporter
        self._span_counter = 0

    def start_span(self, trace_id: str, operation: str,
                   parent_span_id: str = "") -> TradeSpan:
        self._span_counter += 1
        span = TradeSpan(trace_id=trace_id, span_id=f"{self._service}-{self._span_counter}",
                         parent_span_id=parent_span_id, service=self._service,
                         operation=operation, start_ns=time.time_ns())
        _current_trace.set(trace_id)
        _current_span.set(span.span_id)
        return span

    def end_span(self, span: TradeSpan, status: str = "ok") -> None:
        span.end_ns = time.time_ns()
        span.attributes["status"] = status
        self._exporter(span)

    def trace(self, operation: str) -> Callable:
        """Decorator for tracing async functions."""
        def decorator(func: Callable) -> Callable:
            @functools.wraps(func)
            async def wrapper(*args: Any, **kwargs: Any) -> Any:
                span = self.start_span(_current_trace.get() or "unknown",
                                       operation, _current_span.get())
                try:
                    result = await func(*args, **kwargs)
                    self.end_span(span, "ok")
                    return result
                except Exception as e:
                    span.attributes["error"] = str(e)
                    self.end_span(span, "error")
                    raise
            return wrapper
        return decorator
```

**Key latency spans to instrument:**

| Span | Expected | Alert If |
|---|---|---|
| Signal computation | <5ms | >20ms |
| Risk check (gRPC round-trip) | <2ms | >10ms |
| Order submission to broker | <50ms | >200ms |
| Fill confirmation | <100ms | >500ms |
| End-to-end signal-to-fill | <200ms | >1000ms |

---

## Red Flags

| Red Flag | What Is Wrong | Fix |
|---|---|---|
| Kill switch goes through the message bus | Message bus failure = kill switch failure | Direct gRPC + dead-man's switch |
| No trace IDs in events | Cannot correlate events across services | Generate trace_id at signal origin, propagate everywhere |
| Shared database for all services | Database becomes bottleneck and SPOF | Each service owns its data; share via events |
| Synchronous calls between all services | One slow service blocks the entire pipeline | gRPC only for risk checks; async messaging for everything else |
| No partition key strategy | Out-of-order events per symbol | Partition by symbol for ordering guarantees |
| Consumer auto-commit enabled | Message loss on consumer crash | Manual commit after successful processing |
| No dead-man's switch | Kill switch failure goes undetected | Heartbeat timeout triggers automatic safety escalation |
| Position state without TTL | Stale positions treated as current | TTL on position cache; missing key = stale = block trading |
| Risk limits in application memory only | Restart resets limits; concurrent services diverge | Atomic operations in Redis or equivalent shared store |
| Distributing before hitting scale triggers | Unnecessary complexity, more failure modes | Stay single-process until triggers from the table above are hit |

---

## Integration Points

- **trading-bot-architecture** -- Single-process event-driven design. Start here.
  Distributed patterns extend this architecture across service boundaries.
- **kill-switch-and-circuit-breakers** -- Single-process kill switch patterns.
  `DistributedKillSwitch` extends those with heartbeat-based dead-man's switch.
- **async-reliability** -- Task lifecycle, done_callbacks, fail-closed behavior.
  Every async task in every distributed service must follow these rules.
- **trading-monitoring-and-alerts** -- Distributed systems require correlated
  metrics across services using the trace IDs from this skill.
