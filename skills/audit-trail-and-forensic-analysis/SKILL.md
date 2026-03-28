---
name: audit-trail-and-forensic-analysis
description: "Use when implementing audit rules for trading systems, building forensic analysis for broker-vs-DB discrepancies, designing health score aggregation, or detecting silent failures across trading system components"
---

# Audit Trail and Forensic Analysis

Trading bots fail silently. Positions drift from the broker, P&L calculations go stale, management loops hang, and orders fill at unexpected prices — all without crashing the process. Audit rules are the immune system: they detect problems that code can't prevent. But 66 unstructured rules create alert fatigue worse than no auditing at all.

---

## The Iron Law

> **AUDIT RULES MUST BE DETERMINISTIC, ZONE-ISOLATED, AND SEVERITY-RANKED. MORE THAN 10 RULES PER ZONE MEANS YOU ARE AUDITING NOISE, NOT PROBLEMS.**
>
> Origin: emabot grew to 66 audit rules across 5 zones. Each bug fix added a new rule. Rules were not deterministic — same data produced different results depending on timing. Alert fatigue made operators ignore real problems. Commit 7b719e3 added the first structured audit with 34 deterministic rules and a health score. Subsequent commits (aad4b70, 28501ae) added forensic cross-checks.

---

## Core Patterns

### Pattern 1: Five-Zone Audit Model

Divide the trading system into independent audit zones. Each zone has its own rules and health score.

```python
from enum import Enum

class AuditZone(Enum):
    ENTRY = "entry"          # Signal → order placement
    POSITION = "position"    # Open positions → management
    EXIT = "exit"            # Exit logic → fill confirmation
    PNL = "pnl"             # P&L tracking → reconciliation
    SYSTEM = "system"        # Health, connectivity, scheduling

# Each zone: max 10 rules, independently scored
ZONE_RULES = {
    AuditZone.ENTRY: [
        "E1: Every order has an idempotency key",
        "E2: No duplicate open positions for same symbol+direction",
        "E3: All entries within market hours",
        "E4: Entry confidence >= threshold",
        "E5: Position count <= max at time of entry",
    ],
    AuditZone.POSITION: [
        "P1: Every open position has a stop-loss",
        "P2: No position older than max hold time",
        "P3: DB position count matches broker position count",
        "P4: Management loop heartbeat < 120 seconds",
        "P5: No negative quantity positions",
    ],
    AuditZone.EXIT: [
        "X1: Every closed position has exit_price and closed_at",
        "X2: Exit fill price within acceptable slippage",
        "X3: No OPEN positions at broker for CLOSED DB records",
        "X4: All 0DTE positions closed by 15:50 ET",
    ],
    AuditZone.PNL: [
        "L1: DB daily P&L within $1.00 of broker daily P&L",
        "L2: No closed trades with NULL P&L",
        "L3: Sum of partial fills = total filled quantity",
        "L4: No trades with entry_price = 0",
    ],
    AuditZone.SYSTEM: [
        "S1: Bot heartbeat < 120 seconds",
        "S2: DB connection pool healthy",
        "S3: Broker API responsive (< 5s latency)",
        "S4: All scheduled jobs ran today",
    ],
}
```

### Pattern 2: Severity Tiers

```python
class AuditSeverity(Enum):
    CRITICAL = "critical"  # Halt trading immediately
    WARNING = "warning"    # Alert human, continue trading
    INFO = "info"          # Log only, no action

@dataclass
class AuditResult:
    rule_id: str
    zone: AuditZone
    severity: AuditSeverity
    passed: bool
    message: str
    details: dict | None = None
    auto_fix: bool = False  # Can this be safely auto-fixed?

# CRITICAL rules are VETO — any CRITICAL failure = health score 0
def calculate_zone_health(results: list[AuditResult]) -> float:
    """Zone health score: 0-100. CRITICAL failure = instant 0."""
    if any(r.severity == AuditSeverity.CRITICAL and not r.passed for r in results):
        return 0.0

    total = len(results)
    if total == 0:
        return 100.0

    passed = sum(1 for r in results if r.passed)
    return (passed / total) * 100.0
```

### Pattern 3: Deterministic Rules (No External State)

```python
# BAD: Non-deterministic — depends on wall clock time
def check_position_age(position):
    age = datetime.now() - position["opened_at"]  # Which timezone?!
    return age.total_seconds() < MAX_HOLD_SECONDS

# GOOD: Deterministic — takes reference time as parameter
def check_position_age(
    position: dict,
    reference_time: datetime,  # Explicitly passed, not computed inside
    max_hold_seconds: int,
) -> AuditResult:
    opened_at = position["opened_at"]
    if opened_at.tzinfo is None or reference_time.tzinfo is None:
        return AuditResult(
            rule_id="P2", zone=AuditZone.POSITION,
            severity=AuditSeverity.WARNING, passed=False,
            message=f"Naive datetime in position {position['id']} — can't compute age",
        )
    age = (reference_time - opened_at).total_seconds()
    return AuditResult(
        rule_id="P2", zone=AuditZone.POSITION,
        severity=AuditSeverity.WARNING,
        passed=age <= max_hold_seconds,
        message=f"Position {position['symbol']} age: {age/3600:.1f}h (max: {max_hold_seconds/3600:.1f}h)",
    )
```

### Pattern 4: Broker-vs-DB Forensic Cross-Check

```python
async def forensic_cross_check(broker_client, db) -> list[AuditResult]:
    """Compare broker state with DB state. Find ghosts and orphans."""
    results = []

    broker_positions = {
        p.symbol: p for p in await broker_client.get_all_positions()
    }
    db_positions = {
        r["symbol"]: r for r in await db.fetch(
            "SELECT * FROM all_open_positions"
        )
    }

    # Check 1: DB orphans (in DB but not at broker)
    for symbol, db_pos in db_positions.items():
        if symbol not in broker_positions:
            results.append(AuditResult(
                rule_id="X3", zone=AuditZone.EXIT,
                severity=AuditSeverity.CRITICAL, passed=False,
                message=f"DB ORPHAN: {symbol} OPEN in DB but gone from broker",
                details={"position_id": db_pos["id"], "symbol": symbol},
                auto_fix=False,  # NOT safe to auto-fix — investigate first
            ))

    # Check 2: Broker orphans (at broker but not in DB)
    for symbol, broker_pos in broker_positions.items():
        if symbol not in db_positions:
            results.append(AuditResult(
                rule_id="P3", zone=AuditZone.POSITION,
                severity=AuditSeverity.CRITICAL, passed=False,
                message=f"BROKER ORPHAN: {symbol} at broker but not in DB — unmanaged!",
                details={
                    "symbol": symbol,
                    "qty": broker_pos.qty,
                    "avg_entry": broker_pos.avg_entry_price,
                },
                auto_fix=False,  # Must investigate before inserting
            ))

    # Check 3: Quantity mismatch
    for symbol in set(db_positions) & set(broker_positions):
        db_qty = abs(db_positions[symbol].get("qty", 0))
        broker_qty = abs(int(broker_positions[symbol].qty))
        if db_qty != broker_qty:
            results.append(AuditResult(
                rule_id="P3", zone=AuditZone.POSITION,
                severity=AuditSeverity.WARNING, passed=False,
                message=f"QTY MISMATCH: {symbol} DB={db_qty} vs broker={broker_qty}",
            ))

    if not results:
        results.append(AuditResult(
            rule_id="P3", zone=AuditZone.POSITION,
            severity=AuditSeverity.INFO, passed=True,
            message="Broker-DB positions reconciled — no discrepancies",
        ))

    return results
```

### Pattern 5: Silent Failure Detection (Absence Rules)

```python
async def check_expected_events(db, reference_time: datetime) -> list[AuditResult]:
    """Detect silent failures by checking for ABSENCE of expected events."""
    results = []

    # S1: Bot heartbeat — should update every 60s
    last_hb = await db.fetchval(
        "SELECT MAX(timestamp) FROM system_events WHERE event_type = 'heartbeat'"
    )
    if last_hb is None or (reference_time - last_hb).total_seconds() > 120:
        results.append(AuditResult(
            rule_id="S1", zone=AuditZone.SYSTEM,
            severity=AuditSeverity.CRITICAL, passed=False,
            message=f"Bot heartbeat stale — last: {last_hb}, now: {reference_time}",
        ))

    # S4: Scheduled jobs — did EOD flatten run today?
    today = reference_time.date()
    eod_ran = await db.fetchval(
        "SELECT EXISTS(SELECT 1 FROM system_events "
        "WHERE event_type = 'eod_flatten' AND timestamp::date = $1)",
        today,
    )
    if reference_time.hour >= 16 and not eod_ran:
        results.append(AuditResult(
            rule_id="S4", zone=AuditZone.SYSTEM,
            severity=AuditSeverity.CRITICAL, passed=False,
            message="EOD flatten did NOT run today — positions may be held overnight!",
        ))

    return results
```

### Pattern 6: Health Score Aggregation

```python
async def compute_system_health(all_results: list[AuditResult]) -> dict:
    """Aggregate audit results into zone scores and overall health."""
    zone_scores = {}

    for zone in AuditZone:
        zone_results = [r for r in all_results if r.zone == zone]
        zone_scores[zone.value] = calculate_zone_health(zone_results)

    # Overall: weighted average with CRITICAL veto
    weights = {
        "entry": 0.15,
        "position": 0.25,
        "exit": 0.20,
        "pnl": 0.20,
        "system": 0.20,
    }

    # Any CRITICAL failure in any zone = overall health 0
    has_critical = any(
        r.severity == AuditSeverity.CRITICAL and not r.passed
        for r in all_results
    )
    if has_critical:
        overall = 0.0
    else:
        overall = sum(
            zone_scores[zone] * weight
            for zone, weight in weights.items()
        )

    return {
        "overall": round(overall, 1),
        "zones": zone_scores,
        "critical_failures": [
            r.message for r in all_results
            if r.severity == AuditSeverity.CRITICAL and not r.passed
        ],
        "warnings": [
            r.message for r in all_results
            if r.severity == AuditSeverity.WARNING and not r.passed
        ],
    }
```

### Pattern 7: Safe Auto-Fix vs Investigation Required

```python
SAFE_AUTO_FIXES = {
    "stale_cache": lambda: cache.clear(),
    "missing_heartbeat": lambda: update_heartbeat(),
    "null_pnl_closed_trade": lambda trade_id: recalculate_pnl(trade_id),
}

# NEVER auto-fix these — require human investigation
NEVER_AUTO_FIX = {
    "broker_orphan",      # Could be a real position — investigate
    "db_orphan",          # Could be pending fill — investigate
    "qty_mismatch",       # Could indicate partial fill issue
    "negative_pnl_entry", # Could indicate wrong entry price
}

async def apply_auto_fixes(results: list[AuditResult]):
    for result in results:
        if result.passed or not result.auto_fix:
            continue
        fix_key = result.rule_id.lower()
        if fix_key in SAFE_AUTO_FIXES:
            logger.info(f"Auto-fixing: {result.message}")
            await SAFE_AUTO_FIXES[fix_key]()
        elif fix_key in NEVER_AUTO_FIX:
            logger.warning(f"MANUAL FIX REQUIRED: {result.message}")
            await send_alert(f"Audit rule {result.rule_id} failed — manual investigation needed")
```

### Pattern 8: Rule Sunset Policy

```python
async def sunset_stale_rules(db, days_since_last_fire: int = 30):
    """Disable rules that haven't detected issues in 30+ days.
    Stale rules = noise that masks real problems.
    """
    stale_rules = await db.fetch("""
        SELECT rule_id, MAX(fired_at) as last_fired
        FROM audit_events
        WHERE passed = FALSE
        GROUP BY rule_id
        HAVING MAX(fired_at) < NOW() - INTERVAL '%s days'
    """, days_since_last_fire)

    for rule in stale_rules:
        logger.info(
            f"Sunsetting rule {rule['rule_id']} — "
            f"last fired {rule['last_fired']}, no issues in {days_since_last_fire}d"
        )
        await db.execute(
            "UPDATE audit_rules SET enabled = FALSE WHERE rule_id = $1",
            rule["rule_id"],
        )
```

---

## Red Flags

| Red Flag | Why It's Dangerous | Correct Pattern |
|----------|-------------------|-----------------|
| > 10 rules per zone | Alert fatigue — operators ignore everything | Max 10 rules per zone, sunset stale rules |
| Non-deterministic rules (uses `datetime.now()`) | Same data produces different results at different times | Pass `reference_time` as parameter |
| Auto-fix for position mismatches | May hide real problems — positions could be pending | Manual investigation for position-related anomalies |
| No severity tiers | All failures treated equally — CRITICAL buried in noise | CRITICAL (halt) / WARNING (alert) / INFO (log) |
| No rule sunset policy | Dead rules accumulate, adding noise | Disable rules with no failures in 30 days |
| Overall health = simple average | One CRITICAL failure hidden by passing rules | CRITICAL veto: any CRITICAL = health 0 |
| Audit runs inside trading transaction | Audit failure poisons trading operations | Separate connection/transaction for audit |

---

## Integration

| Scenario | Related Skill |
|----------|--------------|
| P&L reconciliation rules | `trading-bot-skills:pnl-calculation-and-reconciliation` |
| Database transaction safety for audit queries | `trading-bot-skills:database-transaction-patterns` |
| Trade replay for investigation | `trading-bot-skills:trade-audit-and-replay` |
| Position mismatch detection | `trading-bot-skills:position-reconciliation` |
| Alert routing and monitoring | `trading-bot-skills:trading-monitoring-and-alerts` |
| Tuning change audit trail | `trading-bot-skills:self-tuning-and-learning-systems` |
