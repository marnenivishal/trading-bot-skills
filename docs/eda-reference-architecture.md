# Event-Driven Architecture Reference for Trading Systems

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Technology Comparison](#technology-comparison)
3. [WebSocket vs Polling](#websocket-vs-polling)
4. [Event Bus Patterns](#event-bus-patterns)
5. [Microservices for Trading](#microservices-for-trading)
6. [Deployment Patterns](#deployment-patterns)

---

## Architecture Overview

An event-driven trading system processes market events asynchronously through
a message bus, enabling independent scaling and fault isolation of each
component.

### System Diagram

```
                         ┌─────────────────────────────────────────────┐
                         │              Event Bus / Message Queue       │
                         └──────┬──────────────┬──────────────┬────────┘
                                │              │              │
                    ┌───────────▼──┐  ┌────────▼─────┐  ┌────▼─────────┐
                    │   Signal     │  │    Risk      │  │  Dashboard   │
                    │   Engine     │  │   Monitor    │  │  / Alerting  │
                    └───────┬──────┘  └──────┬───────┘  └──────────────┘
                            │                │
                            ▼                ▼
                    ┌──────────────────────────────┐
                    │      Execution Gateway       │
                    └──────────────┬───────────────┘
                                   │
                    ┌──────────────▼───────────────┐
                    │         Broker API           │
                    └──────────────────────────────┘
```

**Data Flow:**

1. **Market Data Feed** publishes raw tick/bar data to the Event Bus.
2. **Event Bus** fans out events to all subscribed consumers simultaneously.
3. **Signal Engine** processes data, generates trade signals.
4. **Risk Monitor** validates signals against position limits, drawdown caps.
5. **Dashboard** displays real-time P&L, positions, and alerts.
6. **Execution Gateway** receives approved signals, routes orders to Broker API.

This is an **async fan-out pattern**: one event triggers multiple independent
consumers. A slow dashboard does not block trade execution. A failing risk
monitor does not crash the signal engine.

---

## Technology Comparison

| Technology | Best For | Latency | Persistence | Complexity | Throughput |
|------------|----------|---------|-------------|------------|------------|
| **Kafka** | High-throughput, multi-consumer replay | Medium (ms) | Yes (log-based) | High | Very High |
| **NATS** | Lightweight pub/sub, microservices | Low (us) | Optional (JetStream) | Low | High |
| **Redis Streams** | In-memory state + event streaming | Very Low (us) | Optional (RDB/AOF) | Medium | High |
| **ZeroMQ** | Direct process-to-process, no broker | Lowest | No | Medium | Very High |
| **RabbitMQ** | Reliable delivery, complex routing | Medium (ms) | Yes (queue-based) | Medium | Medium |

### When to Use Each

**Kafka** — Choose when you need event replay (reprocessing historical signals),
multi-consumer groups (multiple strategies reading the same feed), and durable
ordered logs. Overkill for single-strategy setups.

**NATS** — Choose for lightweight, low-latency pub/sub where operational
simplicity matters. NATS JetStream adds persistence when needed. Excellent for
small-to-medium trading systems.

**Redis Streams** — Choose when you already use Redis for caching/state and
want to add event streaming without another infrastructure component. Good for
combining real-time state (positions, balances) with event processing.

**ZeroMQ** — Choose for direct process-to-process communication with minimal
latency. No central broker means no single point of failure, but also no
built-in persistence or replay. Best for co-located services on the same
machine.

**RabbitMQ** — Choose when you need sophisticated routing (topic exchanges,
headers-based routing), guaranteed delivery with acknowledgments, and dead
letter queues. Good for systems where message loss is unacceptable.

---

## WebSocket vs Polling

### When to Use WebSocket

- **Real-time price feeds** — Tick-by-tick or second-by-second market data.
  WebSocket provides sub-second delivery as events occur.
- **Order status updates** — Immediate notification when an order fills,
  partially fills, or is rejected.
- **Live P&L streaming** — Continuous equity curve updates for monitoring.

WebSocket is **event-driven and low-latency**. The server pushes data to the
client the moment it is available.

### When to Use Polling

- **Account balance checks** — Balances change infrequently. Polling every
  30-60 seconds is sufficient.
- **Position reconciliation** — End-of-day or periodic checks to verify
  positions match expected state.
- **Daily reports and summaries** — Data that updates once per day.

Polling is appropriate when you have **tolerance for delay** and the data
changes infrequently.

### Anti-Patterns

| Anti-Pattern | Why It Is Wrong | Correct Approach |
|-------------|-----------------|------------------|
| Polling for price data | Wastes CPU cycles, introduces unnecessary latency, misses fast moves | Use WebSocket for real-time price feeds |
| WebSocket for daily reports | Maintains an unnecessary persistent connection for data that updates once a day | Use a scheduled HTTP request or cron job |
| Polling every 100ms | Effectively simulates real-time but consumes 10x the bandwidth | Use WebSocket instead |
| No reconnection logic on WebSocket | Connection drops silently, system goes blind | Implement exponential backoff reconnection |

### WebSocket Best Practices

1. **Implement heartbeat/ping-pong** to detect dead connections.
2. **Buffer messages** during reconnection to avoid data gaps.
3. **Use separate connections** for market data and order updates to isolate
   failures.
4. **Rate-limit outbound messages** to avoid broker throttling.

---

## Event Bus Patterns

### Fan-Out

One event is delivered to multiple consumers simultaneously. Each consumer
processes the event independently.

```
                    ┌── Signal Engine A (momentum)
Market Event ──────┼── Signal Engine B (mean-reversion)
                    ├── Risk Monitor
                    └── Dashboard
```

**Key property:** A slow consumer does not block the fast path. If the
dashboard takes 500ms to render, trade execution still happens in 1ms.

### Back-Pressure

When a consumer cannot keep up with the event rate, back-pressure prevents
memory exhaustion and data loss.

**Strategies:**
- **Bounded queues** — Reject or drop events when the queue is full. Suitable
  for price data where only the latest value matters.
- **Rate limiting** — Producer slows down to match consumer speed. Not ideal
  for market data (you cannot slow down the market).
- **Load shedding** — Skip intermediate events, process only the latest.
  Acceptable for non-critical consumers (dashboards).
- **Horizontal scaling** — Add more consumer instances via consumer groups
  (Kafka) or queue workers (RabbitMQ).

### Dead Letter Queues (DLQ)

Events that fail processing after N retries are routed to a dead letter queue
for manual inspection.

```
Event ──► Consumer ──[fail]──► Retry Queue ──[fail x3]──► Dead Letter Queue
                                                                │
                                                          Manual review
```

**For trading systems:** A DLQ failure on a trade signal means a signal was
generated but never executed. This must trigger an immediate alert.

### Event Replay

Kafka and similar log-based systems allow replaying historical events for:
- **Backtesting** — Replay a day of market data through the signal engine.
- **Debugging** — Reproduce a specific failure by replaying the exact event
  sequence.
- **New consumer onboarding** — A new strategy can process historical events
  to build initial state.

---

## Microservices for Trading

### Service Decomposition

```
┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐
│  Data Ingestion  │    │  Strategy        │    │  Risk            │
│  Service         │    │  Engine          │    │  Service         │
│                  │    │                  │    │                  │
│  - WebSocket     │    │  - Signal gen    │    │  - Position      │
│    connections   │    │  - Indicator     │    │    limits        │
│  - Data normali- │    │    computation   │    │  - Drawdown      │
│    zation        │    │  - Multi-strat   │    │    monitoring    │
│  - Gap detection │    │    aggregation   │    │  - Correlation   │
│                  │    │                  │    │    checks        │
└────────┬─────────┘    └────────┬─────────┘    └────────┬─────────┘
         │                       │                       │
         └───────────────────────┼───────────────────────┘
                                 │
                    ┌────────────▼─────────────┐
                    │  Execution Gateway       │
                    │                          │
                    │  - Order routing         │
                    │  - Fill tracking         │
                    │  - Retry logic           │
                    │  - Broker abstraction    │
                    └────────────┬─────────────┘
                                 │
                    ┌────────────▼─────────────┐
                    │  Kill Switch Service     │
                    │  (always-on, separate)   │
                    │                          │
                    │  - Emergency shutdown    │
                    │  - Position flattening   │
                    │  - Independent monitoring│
                    └──────────────────────────┘
```

### Service Communication

| Communication Type | Use Case | Protocol | Example |
|--------------------|----------|----------|---------|
| Synchronous | Risk check before order | gRPC / HTTP | Strategy asks Risk Service "can I open this position?" |
| Asynchronous | Signal broadcast | Message Queue | Strategy publishes signal; Execution Gateway consumes |
| Fire-and-forget | Logging, metrics | UDP / Message Queue | Services emit telemetry without waiting for ack |

**gRPC for synchronous risk checks:** When the Strategy Engine generates a
signal, it calls the Risk Service synchronously via gRPC before publishing to
the Execution Gateway. This ensures no order is placed without risk approval.
gRPC is preferred over REST for internal service communication due to lower
latency and strong typing via Protocol Buffers.

**Message queue for async signals:** After risk approval, the signal is
published to the message queue. The Execution Gateway picks it up
asynchronously. This decouples signal generation from order execution.

### Kill Switch as an Independent Service

The kill switch is the most critical component. It must:

- **Run independently** from all other services. If the Strategy Engine or
  Execution Gateway crashes, the kill switch must still function.
- **Monitor continuously** for drawdown breaches, anomalous behavior, or
  manual shutdown commands.
- **Flatten all positions** when triggered, regardless of strategy state.
- **Have direct broker API access** — it does not route through the Execution
  Gateway, which may be the component that is malfunctioning.
- **Be the simplest service** in the system. Minimal dependencies, minimal
  code, maximum reliability.

---

## Deployment Patterns

### Container Orchestration

Each microservice runs in its own container. Use Docker Compose for development
and Kubernetes for production.

```yaml
# docker-compose.yml (development)
services:
  data-ingestion:
    image: trading/data-ingestion:latest
    restart: always
    environment:
      - BROKER_WS_URL=wss://stream.broker.com
    depends_on:
      - message-queue

  strategy-engine:
    image: trading/strategy-engine:latest
    restart: always
    depends_on:
      - data-ingestion
      - risk-service

  risk-service:
    image: trading/risk-service:latest
    restart: always
    environment:
      - MAX_POSITION_SIZE=10000
      - MAX_DRAWDOWN_PCT=15

  execution-gateway:
    image: trading/execution-gateway:latest
    restart: always
    depends_on:
      - risk-service

  kill-switch:
    image: trading/kill-switch:latest
    restart: always
    deploy:
      resources:
        limits:
          memory: 128M
    # Kill switch has NO dependencies on other services

  message-queue:
    image: nats:latest
    restart: always
```

### Health Checks

Every service must expose a health endpoint. Kubernetes or Docker uses this
to restart unhealthy containers automatically.

| Service | Health Check | Failure Action |
|---------|-------------|----------------|
| Data Ingestion | WebSocket connected + receiving data | Restart, alert |
| Strategy Engine | Processing events within latency SLA | Restart, pause trading |
| Risk Service | Responding to gRPC health probe | Restart, BLOCK all new orders |
| Execution Gateway | Broker API reachable | Restart, queue pending orders |
| Kill Switch | Always healthy (self-monitoring) | Restart on SEPARATE node |

**Critical rule:** If the Risk Service health check fails, the Execution
Gateway must refuse all new orders until Risk Service recovers. Fail-closed,
not fail-open.

### Rolling Updates

Deploy new versions without downtime:

1. Start new container with updated code.
2. Run health checks on the new container.
3. Route traffic to the new container.
4. Drain and stop the old container.

**For trading systems:** Never perform rolling updates during market hours
unless it is an emergency fix. Schedule deployments for pre-market (before
9:30 AM ET) or post-market (after 4:00 PM ET).

### Canary Deployments

Route a small percentage of traffic to the new version before full rollout:

1. Deploy the new Strategy Engine as a canary (receives 5% of signals).
2. Monitor performance metrics: latency, error rate, P&L deviation.
3. If metrics are healthy after 1 hour, increase to 25%, then 50%, then 100%.
4. If any anomaly is detected, immediately route 100% back to the old version.

**For trading systems:** Canary deployments are particularly valuable because
a bug in a strategy engine can lose real money. Running the canary on paper
trading while the primary runs on live trading provides an additional safety
layer.

### Monitoring and Observability

| Layer | Tool | Purpose |
|-------|------|---------|
| Metrics | Prometheus + Grafana | Latency, throughput, error rates |
| Logging | ELK Stack or Loki | Structured event logs, debugging |
| Tracing | Jaeger or Zipkin | End-to-end request tracing across services |
| Alerting | PagerDuty or Opsgenie | Immediate notification for critical failures |

**Trading-specific metrics to monitor:**
- Order-to-fill latency (target < 100ms for market orders)
- Signal-to-execution latency (target < 500ms end-to-end)
- Event bus lag (consumer offset behind producer)
- Position drift (expected vs. actual positions)
- Drawdown percentage (real-time, triggers kill switch)

---

## Cross-References

- MCP server setup for broker connectivity: see `docs/mcp-configuration-guide.md`
- Risk management and kill switch details: see `skills/risk-management.md`
- Infrastructure and hosting: see `docs/infrastructure-guide.md`

---

*Event-driven architecture adds complexity. Start with the simplest viable
design (monolith with clear module boundaries) and decompose into services
only when scaling, team size, or reliability requirements demand it.*
