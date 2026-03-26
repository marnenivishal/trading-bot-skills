---
name: trading-monitoring-and-alerts
description: Use when implementing dashboards, alerting, health checks, audit trails, or when failures go unnoticed or audit data masks real problems
---

# Trading Monitoring and Alerts

## Iron Law

**EVERY FAILURE MUST BE VISIBLE WITHIN 60 SECONDS. EVERY TRADE MUST BE AUDITABLE. EVERY COMPONENT MUST PROVE IT IS ALIVE.**

A trading bot that fails silently is worse than one that crashes. A crash stops trading.
A silent failure continues trading with broken logic. You will discover it when you
check your P&L at end of day and find unexplainable losses.

## Prevents

- **Silent failures (#4):** Exceptions caught and swallowed, components dying without
  notification, data feeds dropping without alert.
- **Stale audit (#5):** Dashboard showing data from hours ago, health checks passing
  because they query cached values, audit logs that stop updating.
- **Slippage undetected (#6):** Orders filling at worse prices than expected, accumulating
  hidden costs that destroy edge over weeks.
- **Complexity explosion (#14):** 58 audit rules where 10 would suffice, each bug
  spawning a new rule until the audit system itself becomes a source of bugs.

---

## Alert Tiers

Not all alerts are equal. Three tiers, three response times, three channels.

### CRITICAL -- Immediate Action Required

Delivery: Telegram push notification with sound. Response time: < 1 minute.

- Kill switch triggered
- Position/broker mismatch (reconciliation failure)
- Broker disconnection during market hours
- All data feeds down
- Daily loss limit breached
- Unhandled exception in core loop

### WARNING -- Investigate Within 15 Minutes

Delivery: Telegram message (silent). Response time: < 15 minutes.

- Single data feed stale
- Slippage exceeds threshold on single order
- Order rejected by broker
- Config hot reload failed
- Component heartbeat missed (single occurrence)

### INFO -- Audit Trail

Delivery: Structured log + database. Review: end of day.

- Order placed, filled, cancelled
- Position opened, closed
- Signal generated, confirmed, rejected
- Config loaded, reloaded
- Daily P&L summary
- System startup, shutdown

```python
from dataclasses import dataclass, field
from datetime import datetime, timezone
from enum import Enum
from typing import Optional
import json
import asyncio


class AlertLevel(Enum):
    CRITICAL = "critical"
    WARNING = "warning"
    INFO = "info"


@dataclass
class Alert:
    level: AlertLevel
    component: str
    message: str
    timestamp: datetime = field(default_factory=lambda: datetime.now(timezone.utc))
    symbol: Optional[str] = None
    order_id: Optional[str] = None
    metadata: dict = field(default_factory=dict)

    def to_dict(self) -> dict:
        return {
            "level": self.level.value,
            "component": self.component,
            "message": self.message,
            "timestamp": self.timestamp.isoformat(),
            "symbol": self.symbol,
            "order_id": self.order_id,
            **self.metadata,
        }


class AlertRouter:
    """
    Routes alerts to appropriate channels based on severity.

    CRITICAL -> Telegram (loud) + log + DB
    WARNING  -> Telegram (silent) + log + DB
    INFO     -> log + DB only
    """

    def __init__(
        self,
        telegram_client: "TelegramClient",
        critical_channel: str,
        warning_channel: str,
        logger: "StructuredLogger",
        db: "AuditDB",
    ):
        self.telegram = telegram_client
        self.critical_channel = critical_channel
        self.warning_channel = warning_channel
        self.logger = logger
        self.db = db
        self._alert_counts: dict[str, int] = {}
        self._suppressed_until: dict[str, datetime] = {}

    async def send(self, alert: Alert) -> None:
        """Route alert to appropriate channels."""
        # Always log and persist
        self.logger.log(alert)
        await self.db.store_alert(alert)

        # Rate limit check (prevent alert storms)
        if self._is_suppressed(alert):
            return

        self._alert_counts[alert.component] = (
            self._alert_counts.get(alert.component, 0) + 1
        )

        if alert.level == AlertLevel.CRITICAL:
            await self.telegram.send_urgent(
                channel=self.critical_channel,
                text=self._format_critical(alert),
            )
        elif alert.level == AlertLevel.WARNING:
            await self.telegram.send_silent(
                channel=self.warning_channel,
                text=self._format_warning(alert),
            )

    def _is_suppressed(self, alert: Alert) -> bool:
        """Suppress duplicate alerts within a window to prevent storms."""
        key = f"{alert.component}:{alert.level.value}:{alert.message[:50]}"
        suppressed_until = self._suppressed_until.get(key)
        now = datetime.now(timezone.utc)

        if suppressed_until and now < suppressed_until:
            return True

        # Suppress duplicates for 60s (critical) or 300s (warning)
        from datetime import timedelta
        if alert.level == AlertLevel.CRITICAL:
            self._suppressed_until[key] = now + timedelta(seconds=60)
        elif alert.level == AlertLevel.WARNING:
            self._suppressed_until[key] = now + timedelta(seconds=300)

        return False

    def _format_critical(self, alert: Alert) -> str:
        return (
            f"CRITICAL [{alert.component}]\n"
            f"{alert.message}\n"
            f"Symbol: {alert.symbol or 'N/A'}\n"
            f"Time: {alert.timestamp.strftime('%H:%M:%S UTC')}"
        )

    def _format_warning(self, alert: Alert) -> str:
        return (
            f"WARNING [{alert.component}]\n"
            f"{alert.message}\n"
            f"Time: {alert.timestamp.strftime('%H:%M:%S UTC')}"
        )
```

---

## Health Check Endpoints

Every component exposes a health check. No exceptions.

```python
from dataclasses import dataclass, field
from datetime import datetime, timedelta, timezone
from typing import Optional


@dataclass
class ComponentHealth:
    component: str
    status: str                     # "healthy", "degraded", "dead"
    last_heartbeat: datetime
    error_count_last_hour: int
    positions_count: Optional[int] = None
    daily_pnl: Optional[float] = None
    data_age_seconds: Optional[float] = None
    details: dict = field(default_factory=dict)

    @property
    def is_healthy(self) -> bool:
        return self.status == "healthy"

    @property
    def heartbeat_age(self) -> timedelta:
        return datetime.now(timezone.utc) - self.last_heartbeat


class HealthMonitor:
    """
    Centralized health monitoring for all components.

    Every component registers and sends heartbeats.
    Missed heartbeat -> WARNING. Two missed -> CRITICAL.
    """

    def __init__(
        self,
        alert_router: AlertRouter,
        heartbeat_interval: timedelta = timedelta(seconds=30),
        check_interval: timedelta = timedelta(seconds=10),
    ):
        self.alert_router = alert_router
        self.heartbeat_interval = heartbeat_interval
        self.check_interval = check_interval
        self._components: dict[str, ComponentHealth] = {}
        self._missed_beats: dict[str, int] = {}

    def register(self, component: str) -> None:
        """Register a component for health monitoring."""
        self._components[component] = ComponentHealth(
            component=component,
            status="healthy",
            last_heartbeat=datetime.now(timezone.utc),
            error_count_last_hour=0,
        )
        self._missed_beats[component] = 0

    def heartbeat(self, component: str, **kwargs) -> None:
        """Record heartbeat from a component."""
        if component not in self._components:
            self.register(component)

        health = self._components[component]
        health.last_heartbeat = datetime.now(timezone.utc)
        health.status = "healthy"
        self._missed_beats[component] = 0

        # Update optional fields
        for key, value in kwargs.items():
            if hasattr(health, key):
                setattr(health, key, value)

    async def check_all(self) -> dict[str, ComponentHealth]:
        """Check health of all registered components."""
        now = datetime.now(timezone.utc)

        for name, health in self._components.items():
            age = now - health.last_heartbeat

            if age > self.heartbeat_interval * 2:
                health.status = "dead"
                self._missed_beats[name] = self._missed_beats.get(name, 0) + 1

                if self._missed_beats[name] >= 2:
                    await self.alert_router.send(Alert(
                        level=AlertLevel.CRITICAL,
                        component="health_monitor",
                        message=f"Component '{name}' is DEAD. "
                                f"No heartbeat for {age.total_seconds():.0f}s.",
                    ))

            elif age > self.heartbeat_interval:
                health.status = "degraded"
                self._missed_beats[name] = self._missed_beats.get(name, 0) + 1

                if self._missed_beats[name] == 1:
                    await self.alert_router.send(Alert(
                        level=AlertLevel.WARNING,
                        component="health_monitor",
                        message=f"Component '{name}' missed heartbeat. "
                                f"Last seen {age.total_seconds():.0f}s ago.",
                    ))

        return dict(self._components)

    def get_system_status(self) -> dict:
        """Aggregate system health for dashboard."""
        components = list(self._components.values())
        return {
            "overall": "healthy" if all(c.is_healthy for c in components) else "degraded",
            "components": {c.component: c.status for c in components},
            "total_positions": sum(c.positions_count or 0 for c in components),
            "total_daily_pnl": sum(c.daily_pnl or 0 for c in components),
            "checked_at": datetime.now(timezone.utc).isoformat(),
        }

    async def run(self) -> None:
        """Main health check loop."""
        while True:
            await self.check_all()
            await asyncio.sleep(self.check_interval.total_seconds())
```

---

## Core Metrics

Track these metrics. They are not optional.

### 1. Per-Order Slippage

```python
@dataclass
class SlippageTracker:
    """
    Track slippage per order and alert when it exceeds threshold.

    Slippage = actual_fill_price - expected_price
    For buys: positive slippage = paid more than expected (bad)
    For sells: negative slippage = received less than expected (bad)
    """
    warning_threshold_pct: float = 0.1   # 0.1% = 10 bps
    critical_threshold_pct: float = 0.5  # 0.5% = 50 bps
    _history: list = field(default_factory=list)

    def record(
        self,
        order_id: str,
        symbol: str,
        side: str,
        expected_price: float,
        fill_price: float,
        quantity: int,
    ) -> Alert:
        """Record fill and return appropriate alert."""
        slippage_pct = (fill_price - expected_price) / expected_price * 100
        # For sells, flip sign (negative fill diff = bad slippage)
        if side == "sell":
            slippage_pct = -slippage_pct

        slippage_dollars = abs(fill_price - expected_price) * quantity

        record = {
            "order_id": order_id,
            "symbol": symbol,
            "side": side,
            "expected": expected_price,
            "filled": fill_price,
            "slippage_pct": slippage_pct,
            "slippage_dollars": slippage_dollars,
            "timestamp": datetime.now(timezone.utc),
        }
        self._history.append(record)

        if abs(slippage_pct) > self.critical_threshold_pct:
            return Alert(
                level=AlertLevel.CRITICAL,
                component="slippage_tracker",
                message=(
                    f"CRITICAL SLIPPAGE on {symbol} {side}: "
                    f"{slippage_pct:+.3f}% (${slippage_dollars:.2f})"
                ),
                symbol=symbol,
                order_id=order_id,
                metadata=record,
            )
        elif abs(slippage_pct) > self.warning_threshold_pct:
            return Alert(
                level=AlertLevel.WARNING,
                component="slippage_tracker",
                message=(
                    f"Slippage on {symbol} {side}: "
                    f"{slippage_pct:+.3f}% (${slippage_dollars:.2f})"
                ),
                symbol=symbol,
                order_id=order_id,
                metadata=record,
            )
        else:
            return Alert(
                level=AlertLevel.INFO,
                component="slippage_tracker",
                message=f"Fill {symbol} {side}: slippage {slippage_pct:+.3f}%",
                symbol=symbol,
                order_id=order_id,
                metadata=record,
            )

    def daily_summary(self) -> dict:
        """Compute daily slippage statistics."""
        if not self._history:
            return {"total_slippage_dollars": 0, "avg_slippage_pct": 0, "order_count": 0}

        total_dollars = sum(r["slippage_dollars"] for r in self._history)
        avg_pct = sum(r["slippage_pct"] for r in self._history) / len(self._history)

        return {
            "total_slippage_dollars": total_dollars,
            "avg_slippage_pct": avg_pct,
            "max_slippage_pct": max(r["slippage_pct"] for r in self._history),
            "order_count": len(self._history),
        }
```

### 2. Data Freshness Metric

```python
class DataFreshnessMonitor:
    """Track data freshness across all symbols and sources."""

    def __init__(self, max_staleness: timedelta, alert_router: AlertRouter):
        self.max_staleness = max_staleness
        self.alert_router = alert_router
        self._last_update: dict[str, datetime] = {}

    def record_update(self, symbol: str, source: str) -> None:
        key = f"{symbol}:{source}"
        self._last_update[key] = datetime.now(timezone.utc)

    async def check_freshness(self) -> dict[str, float]:
        """Check freshness of all tracked symbols. Returns age in seconds."""
        now = datetime.now(timezone.utc)
        ages = {}

        for key, last in self._last_update.items():
            age = (now - last).total_seconds()
            ages[key] = age

            if age > self.max_staleness.total_seconds():
                await self.alert_router.send(Alert(
                    level=AlertLevel.WARNING,
                    component="data_freshness",
                    message=f"Stale data: {key} last updated {age:.0f}s ago",
                    symbol=key.split(":")[0],
                ))

        return ages
```

### 3. Reconciliation Delta

```python
class ReconciliationMonitor:
    """
    Compare internal position state vs broker state.

    ANY mismatch is CRITICAL. No tolerance. No "it'll sync eventually."
    """

    async def reconcile(
        self,
        internal_positions: dict[str, int],   # symbol -> quantity
        broker_positions: dict[str, int],
        alert_router: AlertRouter,
    ) -> bool:
        """
        Compare positions. Returns True if reconciled, False if mismatch.
        """
        all_symbols = set(internal_positions.keys()) | set(broker_positions.keys())
        mismatches = []

        for symbol in all_symbols:
            internal = internal_positions.get(symbol, 0)
            broker = broker_positions.get(symbol, 0)

            if internal != broker:
                mismatches.append({
                    "symbol": symbol,
                    "internal": internal,
                    "broker": broker,
                    "delta": internal - broker,
                })

        if mismatches:
            mismatch_str = "\n".join(
                f"  {m['symbol']}: internal={m['internal']}, broker={m['broker']}, "
                f"delta={m['delta']}"
                for m in mismatches
            )
            await alert_router.send(Alert(
                level=AlertLevel.CRITICAL,
                component="reconciliation",
                message=f"POSITION MISMATCH DETECTED:\n{mismatch_str}",
                metadata={"mismatches": mismatches},
            ))
            return False

        return True
```

---

## Audit Rule Simplification

### The Emabot Audit Problem (Case Study)

Emabot accumulated 58 audit rules over its lifetime. Each bug fix added a new audit
rule to "catch this if it happens again." The result:

1. **Alert fatigue.** 58 rules meant dozens of alerts per day. Operators stopped reading them.
2. **False positives.** Many rules were too specific, triggering on normal market conditions.
3. **Maintenance burden.** Each rule needed updating when strategy changed. Many became stale.
4. **Masking real problems.** Important alerts were buried in noise from minor rules.
5. **The audit system itself had bugs.** Rule #43 contradicted rule #17. Rule #29 was
   checking a field that had been renamed. Nobody noticed.

### The Fix: 5-10 Core Invariants

Instead of 58 specific rules, define 5-10 invariants that MUST always be true:

```python
CORE_INVARIANTS = [
    # 1. Positions match broker
    "internal_positions == broker_positions",

    # 2. No position exceeds max risk
    "all(pos.risk_pct <= config.risk.max_single_loss_pct for pos in positions)",

    # 3. Total risk within limits
    "sum(pos.risk_pct for pos in positions) <= config.risk.max_portfolio_risk_pct",

    # 4. All data fresh
    "all(source.age < config.data.max_staleness for source in data_sources)",

    # 5. Daily loss within limit
    "daily_pnl >= -config.risk.max_daily_loss_pct * account_value",

    # 6. No orphaned orders (open orders must have matching position intent)
    "all(order.symbol in intended_positions for order in open_orders)",

    # 7. Kill switch accessible
    "kill_switch.is_responsive()",

    # 8. Trailing stops monotonic
    "all(stop.current >= stop.previous for stop in long_stops)",

    # 9. No duplicate positions (one position per symbol per strategy)
    "len(positions) == len(set((p.symbol, p.strategy) for p in positions))",

    # 10. Heartbeats current
    "all(component.heartbeat_age < 2 * heartbeat_interval for component in components)",
]


class CoreInvariantChecker:
    """
    Check core invariants on a schedule.

    ANY violation is CRITICAL. No warnings. No "probably fine."
    """

    def __init__(self, alert_router: AlertRouter):
        self.alert_router = alert_router
        self._violation_count: int = 0

    async def check_positions_match_broker(
        self, internal: dict, broker: dict
    ) -> bool:
        if internal != broker:
            self._violation_count += 1
            await self.alert_router.send(Alert(
                level=AlertLevel.CRITICAL,
                component="invariant_checker",
                message="INVARIANT VIOLATION: Positions do not match broker",
                metadata={"internal": internal, "broker": broker},
            ))
            return False
        return True

    async def check_risk_limits(
        self, positions: list, config: "RiskConfig", account_value: float
    ) -> bool:
        violations = []
        total_risk = 0.0

        for pos in positions:
            risk_pct = abs(pos.unrealized_pnl) / account_value
            total_risk += risk_pct

            if risk_pct > config.max_single_loss_pct:
                violations.append(
                    f"{pos.symbol}: risk {risk_pct:.2%} > max {config.max_single_loss_pct:.2%}"
                )

        if total_risk > config.max_portfolio_risk_pct:
            violations.append(
                f"Total risk {total_risk:.2%} > max {config.max_portfolio_risk_pct:.2%}"
            )

        if violations:
            self._violation_count += 1
            await self.alert_router.send(Alert(
                level=AlertLevel.CRITICAL,
                component="invariant_checker",
                message="INVARIANT VIOLATION: Risk limits exceeded\n"
                        + "\n".join(f"  - {v}" for v in violations),
            ))
            return False
        return True

    async def check_trailing_stop_monotonicity(
        self, stop_history: dict[str, list[float]]
    ) -> bool:
        violations = []
        for symbol, stops in stop_history.items():
            for i in range(1, len(stops)):
                if stops[i] < stops[i - 1]:
                    violations.append(
                        f"{symbol}: stop decreased {stops[i-1]:.2f} -> {stops[i]:.2f}"
                    )
                    break  # One violation per symbol is enough

        if violations:
            self._violation_count += 1
            await self.alert_router.send(Alert(
                level=AlertLevel.CRITICAL,
                component="invariant_checker",
                message="INVARIANT VIOLATION: Trailing stop moved backward\n"
                        + "\n".join(f"  - {v}" for v in violations),
            ))
            return False
        return True
```

---

## Structured Logging

Every log entry must be structured JSON. No `print()`. No f-string log messages
without structure.

```python
import json
import logging
from datetime import datetime, timezone
from typing import Optional


class StructuredLogger:
    """
    JSON structured logger for trading systems.

    Every log entry has: timestamp, component, level, event_type.
    Additional fields vary by event type.
    """

    def __init__(self, component: str, log_file: str):
        self.component = component
        self._logger = logging.getLogger(component)
        handler = logging.FileHandler(log_file)
        handler.setFormatter(logging.Formatter("%(message)s"))
        self._logger.addHandler(handler)
        self._logger.setLevel(logging.DEBUG)

    def log(self, alert: Alert) -> None:
        """Log an alert as structured JSON."""
        entry = {
            "timestamp": datetime.now(timezone.utc).isoformat(),
            "component": alert.component,
            "level": alert.level.value,
            "event_type": "alert",
            "message": alert.message,
            "symbol": alert.symbol,
            "order_id": alert.order_id,
            **alert.metadata,
        }
        self._logger.info(json.dumps(entry))

    def log_order(
        self,
        event: str,
        symbol: str,
        side: str,
        quantity: int,
        price: float,
        order_id: str,
        **kwargs,
    ) -> None:
        entry = {
            "timestamp": datetime.now(timezone.utc).isoformat(),
            "component": self.component,
            "level": "info",
            "event_type": f"order_{event}",
            "symbol": symbol,
            "side": side,
            "quantity": quantity,
            "price": price,
            "order_id": order_id,
            **kwargs,
        }
        self._logger.info(json.dumps(entry))

    def log_signal(
        self,
        symbol: str,
        direction: str,
        strategy: str,
        status: str,
        strength: float,
        **kwargs,
    ) -> None:
        entry = {
            "timestamp": datetime.now(timezone.utc).isoformat(),
            "component": self.component,
            "level": "info",
            "event_type": "signal",
            "symbol": symbol,
            "direction": direction,
            "strategy": strategy,
            "status": status,
            "strength": strength,
            **kwargs,
        }
        self._logger.info(json.dumps(entry))

    def log_health(self, status: dict) -> None:
        entry = {
            "timestamp": datetime.now(timezone.utc).isoformat(),
            "component": self.component,
            "level": "info",
            "event_type": "health_check",
            **status,
        }
        self._logger.info(json.dumps(entry))
```

```python
# BAD: Unstructured logging
print(f"Order filled: AAPL 100 shares at $150.25")
logger.info(f"Signal: LONG AAPL EMA crossover strength=1.2")

# GOOD: Structured logging
structured_log.log_order(
    event="filled",
    symbol="AAPL",
    side="buy",
    quantity=100,
    price=150.25,
    order_id="ord_abc123",
    slippage_pct=0.03,
)

structured_log.log_signal(
    symbol="AAPL",
    direction="long",
    strategy="ema_crossover",
    status="confirmed",
    strength=1.2,
    confirmation_bars=4,
)
```

---

## Dashboard Anti-Patterns

### Stale Dashboard Problem

```python
# BAD: Dashboard queries DB directly -- may show stale data
@app.get("/dashboard")
def dashboard():
    positions = db.query("SELECT * FROM positions")  # When was this last updated?
    pnl = db.query("SELECT sum(pnl) FROM trades WHERE date=today()")
    return render(positions=positions, pnl=pnl)

# GOOD: Dashboard shows data age and health status
@app.get("/dashboard")
async def dashboard():
    health = await health_monitor.check_all()
    system_status = health_monitor.get_system_status()

    return render(
        system_status=system_status,
        components=health,
        data_ages=await freshness_monitor.check_freshness(),
        last_reconciliation=reconciliation_monitor.last_check_time,
        # Show HOW OLD the data is, not just the data
        data_staleness_warning=(
            any(c.status != "healthy" for c in health.values())
        ),
    )
```

---

## Putting It All Together

```python
class MonitoringSystem:
    """
    Complete monitoring system tying together all components.

    Initializes: alert routing, health monitoring, slippage tracking,
    data freshness, reconciliation, and invariant checking.
    """

    def __init__(self, config: "TradingConfig"):
        self.logger = StructuredLogger(
            component="monitoring",
            log_file=config.logging.file,
        )
        self.alert_router = AlertRouter(
            telegram_client=TelegramClient(config.alerts),
            critical_channel=config.alerts.critical_channel,
            warning_channel=config.alerts.warning_channel,
            logger=self.logger,
            db=AuditDB(config.db),
        )
        self.health_monitor = HealthMonitor(
            alert_router=self.alert_router,
        )
        self.slippage_tracker = SlippageTracker(
            warning_threshold_pct=0.1,
            critical_threshold_pct=0.5,
        )
        self.freshness_monitor = DataFreshnessMonitor(
            max_staleness=timedelta(seconds=config.data.max_staleness_seconds),
            alert_router=self.alert_router,
        )
        self.reconciliation = ReconciliationMonitor()
        self.invariant_checker = CoreInvariantChecker(
            alert_router=self.alert_router,
        )

    async def run(self) -> None:
        """Start all monitoring loops."""
        await asyncio.gather(
            self.health_monitor.run(),
            self._reconciliation_loop(),
            self._freshness_loop(),
            self._invariant_loop(),
        )

    async def _reconciliation_loop(self) -> None:
        while True:
            internal = await get_internal_positions()
            broker = await get_broker_positions()
            await self.reconciliation.reconcile(
                internal, broker, self.alert_router
            )
            await asyncio.sleep(30)

    async def _freshness_loop(self) -> None:
        while True:
            await self.freshness_monitor.check_freshness()
            await asyncio.sleep(5)

    async def _invariant_loop(self) -> None:
        while True:
            await self.invariant_checker.check_positions_match_broker(
                await get_internal_positions(),
                await get_broker_positions(),
            )
            await asyncio.sleep(60)
```

---

## Red Flags

Stop and fix immediately if you see ANY of these:

- **`except: pass` or `except Exception: pass`.** This is the #1 cause of silent
  failures. Every exception must be logged and alerted on.
- **Email-only alerts.** Email is too slow for trading. Use Telegram, SMS, or push
  notifications for anything CRITICAL.
- **Stale DB queries in dashboard.** If the dashboard queries the DB but the DB writer
  died an hour ago, you are looking at hour-old data and thinking it is live.
- **20+ audit rules.** You have alert fatigue. Consolidate to 5-10 core invariants.
- **`print()` statements.** These are unstructured, unsearchable, and disappear when
  stdout is not captured. Use structured logging.
- **Health check that only checks "is process running."** A process can be running but
  broken. Health checks must verify functionality (heartbeats, data freshness, position
  reconciliation).
- **No slippage tracking.** If you do not measure slippage, you do not know your true
  execution costs. Your backtested edge may be entirely consumed by slippage.
- **Alerts without suppression.** A broken data feed can generate thousands of
  "data stale" alerts per minute. Rate limit your alerts.

---

## Testing Monitoring

```python
class TestAlertRouter:
    @pytest.mark.asyncio
    async def test_critical_sends_telegram(self):
        telegram = MockTelegramClient()
        router = AlertRouter(
            telegram_client=telegram,
            critical_channel="test-critical",
            warning_channel="test-warning",
            logger=MockLogger(),
            db=MockDB(),
        )
        await router.send(Alert(
            level=AlertLevel.CRITICAL,
            component="test",
            message="Kill switch triggered",
        ))
        assert telegram.last_channel == "test-critical"
        assert "Kill switch" in telegram.last_message

    @pytest.mark.asyncio
    async def test_info_does_not_send_telegram(self):
        telegram = MockTelegramClient()
        router = AlertRouter(
            telegram_client=telegram,
            critical_channel="test-critical",
            warning_channel="test-warning",
            logger=MockLogger(),
            db=MockDB(),
        )
        await router.send(Alert(
            level=AlertLevel.INFO,
            component="test",
            message="Order filled",
        ))
        assert telegram.last_message is None  # No Telegram for INFO

    @pytest.mark.asyncio
    async def test_alert_suppression_prevents_storms(self):
        telegram = MockTelegramClient()
        router = AlertRouter(
            telegram_client=telegram,
            critical_channel="test-critical",
            warning_channel="test-warning",
            logger=MockLogger(),
            db=MockDB(),
        )
        # Send same alert 10 times
        for _ in range(10):
            await router.send(Alert(
                level=AlertLevel.CRITICAL,
                component="test",
                message="Same alert repeated",
            ))
        # Telegram should only receive 1 (rest suppressed)
        assert telegram.send_count == 1


class TestSlippageTracker:
    def test_normal_slippage_is_info(self):
        tracker = SlippageTracker()
        alert = tracker.record("ord1", "AAPL", "buy", 150.00, 150.01, 100)
        assert alert.level == AlertLevel.INFO

    def test_high_slippage_is_warning(self):
        tracker = SlippageTracker(warning_threshold_pct=0.1)
        alert = tracker.record("ord1", "AAPL", "buy", 150.00, 150.20, 100)
        assert alert.level == AlertLevel.WARNING

    def test_extreme_slippage_is_critical(self):
        tracker = SlippageTracker(critical_threshold_pct=0.5)
        alert = tracker.record("ord1", "AAPL", "buy", 150.00, 151.00, 100)
        assert alert.level == AlertLevel.CRITICAL
```

---

## Integration

- **trading-bot-skills:kill-switch-and-circuit-breakers** -- Kill switch events are
  the highest priority alert. The monitoring system must immediately surface kill
  switch activations.
- **trading-bot-skills:position-reconciliation** -- Reconciliation mismatches are
  CRITICAL alerts. The monitoring system runs reconciliation checks on a schedule
  and alerts on any discrepancy.
