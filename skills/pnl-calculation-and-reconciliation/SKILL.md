---
name: pnl-calculation-and-reconciliation
description: "You MUST use this when calculating P&L, tracking partial fills, handling options contract multipliers, reconciling DB totals against broker equity, or aggregating daily P&L from multiple trade tables"
---

# P&L Calculation and Reconciliation

P&L calculation in multi-strategy trading systems is deceptively complex. Partial fills create fractional positions with changing average costs. Options have a x100 contract multiplier that's easy to forget. Multiple trade tables (equities, options, 0DTE, pre-close) must be UNION'd with dedup — not summed separately. And the broker's equity change is the ONLY source of truth.

---

## The Iron Law

> **BROKER EQUITY IS TRUTH. YOUR DB P&L IS AN APPROXIMATION. WHEN THEY DISAGREE, THE BROKER IS RIGHT.**
>
> Origin: emabot's dashboard showed the wrong P&L for weeks. The "hero P&L" metric aggregated closed trades from the DB, but partial fills, ghost positions, and missing multipliers corrupted the total. 28+ commits to fix. The final solution: `hero_pnl = broker_current_equity - broker_opening_equity`.

---

## Core Patterns

### Pattern 1: Broker Equity as Source of Truth

```python
# BAD: Aggregate P&L from DB — fragile, drifts from reality
hero_pnl = sum(t.pnl for t in await fetch_all_closed_trades_today())

# GOOD: Use broker's actual equity change
async def get_daily_pnl(broker_client) -> float:
    account = await broker_client.get_account()
    return float(account.equity) - float(account.last_equity)
    # last_equity = previous close equity; equity = current
```

### Pattern 2: Options Contract Multiplier

```python
# BAD: Missing x100 multiplier — reports $5 gain instead of $500
pnl = (exit_premium - entry_premium) * quantity

# GOOD: Options P&L includes contract multiplier
OPTION_MULTIPLIER = 100  # Each options contract = 100 shares

def calculate_options_pnl(
    entry_premium: float,
    exit_premium: float,
    quantity: int,
    side: str = "long",
) -> float:
    """Calculate P&L for options position."""
    raw_pnl = (exit_premium - entry_premium) * quantity * OPTION_MULTIPLIER
    return raw_pnl if side == "long" else -raw_pnl
```

### Pattern 3: Partial Fill Tracking

When an order is partially filled, the average cost must be updated incrementally.

```python
# BAD: Use original order price as entry — ignores fill price variation
entry_price = order.limit_price  # Wrong if filled at different prices

# GOOD: Weighted average cost basis
def update_cost_basis(
    current_qty: int,
    current_avg_cost: float,
    fill_qty: int,
    fill_price: float,
) -> tuple[int, float]:
    """Update position after a partial fill. Returns (new_qty, new_avg_cost)."""
    new_qty = current_qty + fill_qty
    if new_qty == 0:
        return 0, 0.0
    new_avg_cost = (
        (current_qty * current_avg_cost) + (fill_qty * fill_price)
    ) / new_qty
    return new_qty, new_avg_cost

# Track each fill as it arrives
position_qty, position_avg = 0, 0.0
for fill in order.fills:
    position_qty, position_avg = update_cost_basis(
        position_qty, position_avg, fill.qty, fill.price
    )
```

### Pattern 4: Multi-Table UNION with Dedup

When trades live in multiple tables, aggregate with UNION and dedup — never sum separate queries.

```python
# BAD: Sum separate queries — double-counts if a trade appears in both tables
equity_pnl = await db.fetchval("SELECT SUM(pnl) FROM trades WHERE ...")
options_pnl = await db.fetchval("SELECT SUM(pnl) FROM options_trades WHERE ...")
spx_pnl = await db.fetchval("SELECT SUM(pnl) FROM spx_0dte_trades WHERE ...")
total = equity_pnl + options_pnl + spx_pnl  # DOUBLE-COUNT RISK!

# GOOD: Single UNION query with dedup
DAILY_PNL_QUERY = """
    SELECT COALESCE(SUM(pnl), 0) as total_pnl FROM (
        SELECT pnl, idempotency_key FROM trades
        WHERE closed_at >= $1 AND status = 'CLOSED'
        UNION
        SELECT pnl, idempotency_key FROM options_trades
        WHERE closed_at >= $1 AND status = 'CLOSED'
        UNION
        SELECT pnl, idempotency_key FROM spx_0dte_trades
        WHERE closed_at >= $1 AND status = 'CLOSED'
        UNION
        SELECT pnl, idempotency_key FROM spx_preclose_trades
        WHERE closed_at >= $1 AND status = 'CLOSED'
    ) combined
"""
# UNION (not UNION ALL) deduplicates rows with same idempotency_key
today_start = datetime.now(ET).replace(hour=0, minute=0, second=0)
total_pnl = await db.fetchval(DAILY_PNL_QUERY, today_start)
```

### Pattern 5: Ghost Position P&L Estimation

When a position exists at broker but not in DB (ghost), estimate worst-case P&L.

```python
async def estimate_ghost_pnl(
    broker_position,
    max_adverse_pct: float = 0.10,  # Assume 10% adverse move
) -> float:
    """Estimate P&L for a ghost position (no DB record).
    Use worst-case assumptions — better to overstate loss than understate.
    """
    qty = abs(float(broker_position.qty))
    avg_entry = float(broker_position.avg_entry_price)
    current_price = float(broker_position.current_price)

    # Use actual unrealized P&L if available
    if broker_position.unrealized_pl is not None:
        return float(broker_position.unrealized_pl)

    # Worst case: assume we entered at the worst possible price
    estimated_pnl = (current_price - avg_entry) * qty
    if broker_position.side == "short":
        estimated_pnl = -estimated_pnl

    return estimated_pnl
```

### Pattern 6: Reconciliation Tolerance

Broker and DB will never match exactly due to timing, rounding, and fees.

```python
RECONCILIATION_TOLERANCE = 0.50  # $0.50 tolerance for rounding/fees

async def reconcile_daily_pnl(broker_client, db) -> dict:
    broker_pnl = await get_daily_pnl(broker_client)
    db_pnl = await db.fetchval(DAILY_PNL_QUERY, today_start)

    diff = abs(broker_pnl - db_pnl)
    is_reconciled = diff <= RECONCILIATION_TOLERANCE

    if not is_reconciled:
        logger.warning(
            f"P&L mismatch: broker=${broker_pnl:.2f} vs DB=${db_pnl:.2f} "
            f"(diff=${diff:.2f}, tolerance=${RECONCILIATION_TOLERANCE:.2f})"
        )
        # Patch DB with broker truth
        await db.execute(
            "INSERT INTO pnl_reconciliation (date, broker_pnl, db_pnl, diff) "
            "VALUES ($1, $2, $3, $4)",
            date.today(), broker_pnl, db_pnl, diff,
        )

    return {"broker": broker_pnl, "db": db_pnl, "reconciled": is_reconciled}
```

### Pattern 7: Quantity Validation Invariants

```python
def validate_position_quantities(position: dict) -> list[str]:
    """Check position quantity invariants. Returns list of violations."""
    errors = []

    if position["qty"] < 0:
        errors.append(f"Negative qty: {position['qty']}")

    if position["status"] == "CLOSED" and position["qty"] != 0:
        errors.append(f"Closed position has qty={position['qty']}, expected 0")

    if position.get("filled_qty", 0) > position.get("requested_qty", 0):
        errors.append(
            f"filled_qty ({position['filled_qty']}) > "
            f"requested_qty ({position['requested_qty']})"
        )

    if position.get("contracts_running", 0) > position.get("qty", 0):
        errors.append(
            f"contracts_running ({position['contracts_running']}) > "
            f"qty ({position['qty']})"
        )

    return errors
```

---

## Red Flags

| Red Flag | Why It's Dangerous | Correct Pattern |
|----------|-------------------|-----------------|
| `pnl = (exit - entry) * qty` for options | Missing x100 multiplier — 100x underreporting | `* OPTION_MULTIPLIER` for all options P&L |
| Separate SUM queries per trade table | Double-counting if trade appears in 2 tables | Single `UNION` query with dedup key |
| DB aggregate as "hero P&L" on dashboard | Drifts from broker reality due to fills, ghosts | `broker.equity - broker.last_equity` |
| `entry_price = order.limit_price` | Partial fills execute at different prices | Weighted average from actual fill prices |
| Ghost position P&L = $0 | Hides real losses — position is unmanaged | Estimate worst-case using broker's avg_entry |
| Exact equality check in reconciliation | Rounding, fees, timing → always mismatches | Tolerance-based comparison ($0.50 typical) |
| No quantity validation on positions | Negative qty, closed with nonzero qty go undetected | Validate invariants after every position update |

---

## Integration

| Scenario | Related Skill |
|----------|--------------|
| Position tracking and sync | `trading-bot-skills:position-reconciliation` |
| Options-specific P&L gotchas | `trading-bot-skills:options-trading-safety` |
| Database transaction safety for P&L updates | `trading-bot-skills:database-transaction-patterns` |
| Numeric None vs zero handling | `trading-bot-skills:falsy-zero-and-sentinel-values` |
| Fill handling and order lifecycle | `trading-bot-skills:order-execution-integrity` |
| Daily loss limit using P&L | `trading-bot-skills:risk-management-gates` |
