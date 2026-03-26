---
name: trade-audit-and-replay
description: Use when implementing trade logging, audit trails, trade replay validation, or when comparing bot trades against broker records to detect discrepancies
---

# Trade Audit and Replay

**Iron Law:** EVERY TRADE DECISION, SIGNAL, ORDER, AND FILL MUST BE LOGGED TO AN IMMUTABLE, QUERYABLE AUDIT TRAIL. BOT STATE MUST BE REPLAYABLE FROM LOGS.

If you cannot reconstruct exactly what happened and why from your logs alone, your trading bot is a black box. Black boxes lose money in ways you cannot diagnose, fix, or prevent from recurring.

## Immutable Audit Trail

The audit trail is append-only. You never update or delete entries. Every event gets a timestamp, a trace ID for correlation, and structured data.

```python
import json
import time
import uuid
from dataclasses import dataclass, field, asdict
from enum import Enum
from pathlib import Path
from typing import Any, Optional


class EventType(Enum):
    SIGNAL_RECEIVED = "signal_received"
    SIGNAL_PARSED = "signal_parsed"
    SIGNAL_REJECTED = "signal_rejected"
    VALIDATION_PASSED = "validation_passed"
    VALIDATION_FAILED = "validation_failed"
    RISK_APPROVED = "risk_approved"
    RISK_REJECTED = "risk_rejected"
    ORDER_SUBMITTED = "order_submitted"
    ORDER_FILLED = "order_filled"
    ORDER_PARTIALLY_FILLED = "order_partially_filled"
    ORDER_CANCELLED = "order_cancelled"
    ORDER_REJECTED_BROKER = "order_rejected_broker"
    POSITION_OPENED = "position_opened"
    POSITION_CLOSED = "position_closed"
    POSITION_CHANGED = "position_changed"
    CONFIG_CHANGED = "config_changed"
    KILL_SWITCH_ACTIVATED = "kill_switch_activated"
    ERROR = "error"
    EOD_FLATTEN_STARTED = "eod_flatten_started"
    EOD_FLATTEN_COMPLETED = "eod_flatten_completed"


@dataclass
class AuditEntry:
    timestamp: float
    event_type: str
    component: str
    symbol: Optional[str]
    data: dict[str, Any]
    trace_id: str

    def to_json(self) -> str:
        return json.dumps(asdict(self), default=str, separators=(",", ":"))


class AuditLogger:
    """Append-only structured audit logger. Writes JSONL to file."""

    def __init__(self, log_dir: str, max_file_size_mb: int = 100):
        self._log_dir = Path(log_dir)
        self._log_dir.mkdir(parents=True, exist_ok=True)
        self._max_size = max_file_size_mb * 1024 * 1024
        self._current_file = self._open_log_file()

    def _open_log_file(self):
        date_str = time.strftime("%Y-%m-%d")
        path = self._log_dir / f"audit_{date_str}.jsonl"
        return open(path, "a", encoding="utf-8")

    def _rotate_if_needed(self):
        if self._current_file.tell() > self._max_size:
            self._current_file.close()
            self._current_file = self._open_log_file()

    def log(
        self,
        event_type: EventType,
        component: str,
        symbol: Optional[str] = None,
        data: Optional[dict] = None,
        trace_id: Optional[str] = None,
    ) -> AuditEntry:
        entry = AuditEntry(
            timestamp=time.time(),
            event_type=event_type.value,
            component=component,
            symbol=symbol,
            data=data or {},
            trace_id=trace_id or str(uuid.uuid4()),
        )
        self._rotate_if_needed()
        self._current_file.write(entry.to_json() + "\n")
        self._current_file.flush()  # ensure durability
        return entry

    def close(self):
        self._current_file.close()
```

## What to Log (Mandatory)

Every event below MUST appear in the audit trail. If any event type is missing, you have an audit gap.

| Event | When | Key Data |
|---|---|---|
| Signal received | External or internal signal arrives | source, raw text, trace_id |
| Validation result | Pre-trade validation pass/fail | checks passed, checks failed, reasons |
| Risk decision | Risk gate approve/reject | gate name, reason, current exposure |
| Order submitted | Order sent to broker | symbol, side, qty, type, limit_price, order_id |
| Fill received | Broker confirms fill | fill_qty, fill_price, slippage vs expected |
| Position change | Position opens/closes/changes | symbol, qty_before, qty_after, avg_price |
| Config change | Any config parameter changes | param_name, old_value, new_value, who changed |
| Error/exception | Any caught exception | component, error_type, message, stack trace |
| Kill switch | Emergency shutdown triggered | reason, positions at time of activation |

```python
# Example: logging a complete order lifecycle

trace_id = str(uuid.uuid4())

# Signal arrives
audit.log(EventType.SIGNAL_RECEIVED, "webhook", symbol="AAPL",
          data={"raw": "BTO AAPL 150C 1.50", "source": "discord"},
          trace_id=trace_id)

# Validation passes
audit.log(EventType.VALIDATION_PASSED, "pre_trade_validator", symbol="AAPL",
          data={"checks": ["symbol", "market_hours", "liquidity", "price", "dedup", "buying_power"]},
          trace_id=trace_id)

# Risk gate approves
audit.log(EventType.RISK_APPROVED, "risk_gate", symbol="AAPL",
          data={"position_size_pct": 2.0, "max_allowed": 5.0, "total_exposure": 15.0},
          trace_id=trace_id)

# Order submitted
audit.log(EventType.ORDER_SUBMITTED, "order_executor", symbol="AAPL",
          data={"order_id": "abc123", "side": "buy", "qty": 10, "type": "limit", "limit_price": 1.50},
          trace_id=trace_id)

# Fill received
audit.log(EventType.ORDER_FILLED, "order_executor", symbol="AAPL",
          data={"order_id": "abc123", "fill_qty": 10, "fill_price": 1.52, "slippage": 0.02},
          trace_id=trace_id)
```

## Trade Replay Validation

Every night, compare what the bot logged against what the broker recorded. Any mismatch is a critical discrepancy.

```python
from dataclasses import dataclass


@dataclass
class BrokerTrade:
    order_id: str
    symbol: str
    side: str
    qty: float
    avg_price: float
    filled_at: float  # epoch


@dataclass
class BotTrade:
    order_id: str
    symbol: str
    side: str
    qty: float
    expected_price: float
    fill_price: Optional[float]
    submitted_at: float
    filled_at: Optional[float]


@dataclass
class Discrepancy:
    discrepancy_type: str  # "missing_in_broker", "missing_in_bot", "price_mismatch", "qty_mismatch"
    order_id: Optional[str]
    symbol: str
    detail: str


class TradeReplayValidator:
    """Nightly comparison of bot trade log vs broker trade history."""

    def __init__(self, audit_logger: AuditLogger, broker_client, price_tolerance: float = 0.01):
        self._audit = audit_logger
        self._broker = broker_client
        self._price_tol = price_tolerance

    def load_bot_trades(self, date: str) -> dict[str, BotTrade]:
        """Load bot trades from audit log for a given date."""
        trades = {}
        log_path = Path(self._audit._log_dir) / f"audit_{date}.jsonl"
        if not log_path.exists():
            return trades

        with open(log_path, "r") as f:
            for line in f:
                entry = json.loads(line)
                if entry["event_type"] == EventType.ORDER_FILLED.value:
                    d = entry["data"]
                    trades[d["order_id"]] = BotTrade(
                        order_id=d["order_id"],
                        symbol=entry["symbol"],
                        side=d["side"],
                        qty=d["fill_qty"],
                        expected_price=d.get("expected_price", 0),
                        fill_price=d["fill_price"],
                        submitted_at=entry["timestamp"],
                        filled_at=entry["timestamp"],
                    )
        return trades

    def load_broker_trades(self, date: str) -> dict[str, BrokerTrade]:
        """Fetch broker trade history for a given date."""
        orders = self._broker.list_orders(
            status="filled", after=f"{date}T00:00:00Z", until=f"{date}T23:59:59Z"
        )
        trades = {}
        for o in orders:
            trades[o.id] = BrokerTrade(
                order_id=o.id,
                symbol=o.symbol,
                side=o.side,
                qty=float(o.filled_qty),
                avg_price=float(o.filled_avg_price),
                filled_at=o.filled_at.timestamp(),
            )
        return trades

    def compare(self, date: str) -> list[Discrepancy]:
        bot = self.load_bot_trades(date)
        broker = self.load_broker_trades(date)
        discrepancies = []

        # Check for trades bot logged but broker does not have
        for oid, bt in bot.items():
            if oid not in broker:
                discrepancies.append(Discrepancy(
                    discrepancy_type="missing_in_broker",
                    order_id=oid,
                    symbol=bt.symbol,
                    detail=f"Bot logged fill for {oid} but broker has no record",
                ))
                continue

            bk = broker[oid]
            # Quantity mismatch
            if bt.qty != bk.qty:
                discrepancies.append(Discrepancy(
                    discrepancy_type="qty_mismatch",
                    order_id=oid,
                    symbol=bt.symbol,
                    detail=f"Bot qty={bt.qty}, broker qty={bk.qty}",
                ))
            # Price mismatch
            if bt.fill_price and abs(bt.fill_price - bk.avg_price) > self._price_tol:
                discrepancies.append(Discrepancy(
                    discrepancy_type="price_mismatch",
                    order_id=oid,
                    symbol=bt.symbol,
                    detail=f"Bot price={bt.fill_price}, broker price={bk.avg_price}",
                ))

        # Check for trades broker has but bot did not log
        for oid, bk in broker.items():
            if oid not in bot:
                discrepancies.append(Discrepancy(
                    discrepancy_type="missing_in_bot",
                    order_id=oid,
                    symbol=bk.symbol,
                    detail=f"Broker has fill for {oid} but bot has no record -- PHANTOM TRADE",
                ))

        return discrepancies
```

## Discrepancy Detector (Automated Nightly)

```python
import time


class DiscrepancyDetector:
    """Runs nightly to compare bot vs broker and alert on mismatches."""

    def __init__(self, validator: TradeReplayValidator, alerter, audit: AuditLogger):
        self._validator = validator
        self._alerter = alerter
        self._audit = audit

    def run_nightly_check(self, date: str) -> None:
        discrepancies = self._validator.compare(date)

        if not discrepancies:
            self._audit.log(
                EventType.SIGNAL_RECEIVED,  # reuse as "system_event"
                component="discrepancy_detector",
                data={"date": date, "result": "clean", "count": 0},
            )
            return

        # Log every discrepancy
        for d in discrepancies:
            self._audit.log(
                EventType.ERROR,
                component="discrepancy_detector",
                symbol=d.symbol,
                data={
                    "date": date,
                    "type": d.discrepancy_type,
                    "order_id": d.order_id,
                    "detail": d.detail,
                },
            )

        # Alert operator
        summary = f"{len(discrepancies)} trade discrepancies for {date}:\n"
        for d in discrepancies:
            summary += f"  [{d.discrepancy_type}] {d.symbol}: {d.detail}\n"

        self._alerter.send_critical(
            subject=f"TRADE DISCREPANCY ALERT: {date}",
            body=summary,
        )
```

## Replay Engine

Feed recorded market data through your strategy offline. If the same input produces different output, your strategy has a non-deterministic bug.

```python
class ReplayEngine:
    """Replay historical market data through strategy to verify determinism."""

    def __init__(self, strategy, audit_log_dir: str):
        self._strategy = strategy
        self._log_dir = Path(audit_log_dir)

    def replay(self, date: str) -> list[dict]:
        """Replay signals and compare strategy output to recorded decisions."""
        log_path = self._log_dir / f"audit_{date}.jsonl"
        signals, decisions = [], {}
        with open(log_path, "r") as f:
            for line in f:
                entry = json.loads(line)
                if entry["event_type"] == EventType.SIGNAL_RECEIVED.value:
                    signals.append(entry)
                elif entry["event_type"] in (EventType.RISK_APPROVED.value, EventType.RISK_REJECTED.value):
                    decisions[entry["trace_id"]] = entry["data"]

        mismatches = []
        for sig in signals:
            tid = sig["trace_id"]
            replay_result = self._strategy.evaluate(sig["data"])
            original = decisions.get(tid)
            if original and replay_result != original:
                mismatches.append({"trace_id": tid, "symbol": sig.get("symbol"),
                                   "original": original, "replay": replay_result})
        return mismatches
```

## Audit Log Storage

| Approach | Pros | Cons |
|---|---|---|
| JSONL files | Simple, portable, no dependencies | Hard to query at scale |
| SQLite | Queryable, single file, no server | Write contention under load |
| PostgreSQL | Full SQL, concurrent writes, scalable | Operational overhead |

**Recommendation:** Start with JSONL files. Move to SQLite or PostgreSQL when you need complex queries or retention management. Retention policy: keep 90 days minimum.

## Red Flags

| Red Flag | Why It Matters |
|---|---|
| `print()` statements instead of structured audit | Cannot query, filter, or replay from print output |
| Mutable log entries (update/delete) | Destroys audit integrity, cannot trust the trail |
| No bot-vs-broker comparison | Phantom trades and missing fills go undetected |
| Non-deterministic strategy | Replay produces different results, impossible to debug |
| Audit gaps (missing event types) | Cannot reconstruct full decision chain |
| No trace_id correlation | Cannot follow a signal through the entire pipeline |
| Audit log on same disk as trading bot | Disk failure loses both bot state and audit trail |
| No log rotation or retention policy | Disk fills up, bot crashes |

## Integration

- **position-reconciliation** -- position-level comparison complements the trade-level audit. Reconciliation catches drift; audit catches individual trade issues.
- **trading-monitoring-and-alerts** -- discrepancy detector feeds into the alerting system. Any mismatch triggers an operator alert.
- **signal-source-integration** -- every signal received from external sources is logged with its source identifier and trace_id.
- **order-execution-integrity** -- every order submission and fill is logged. The audit trail is the source of truth for execution quality analysis.
