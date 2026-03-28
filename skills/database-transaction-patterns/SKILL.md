---
name: database-transaction-patterns
description: "You MUST use this when encountering InFailedSqlTransaction errors, implementing nested database operations, using SAVEPOINT patterns, or designing transaction boundaries for concurrent trading systems"
---

# Database Transaction Patterns for Trading

A single failed query inside a PostgreSQL transaction "poisons" every subsequent query in that transaction — they all fail with `InFailedSqlTransaction`. In a trading bot, this means a failed timezone parse inside a dedup check can silently kill position management, reconciliation, and stop-loss updates for the entire session.

---

## The Iron Law

> **A FAILED QUERY POISONS THE ENTIRE TRANSACTION. EVERY SUB-QUERY THAT CAN FAIL INDEPENDENTLY MUST USE SAVEPOINT OR A SEPARATE CONNECTION.**
>
> Origin: emabot Failure #1 — A timezone conversion inside a dedup query failed. `except: pass` swallowed the error. ALL subsequent queries returned `InFailedSqlTransaction`. Bot crashed at 9:19 AM with 9 broker positions having ZERO DB records. No stops, no management, no kill switch visibility. 22+ commits to fix across db.py, analytics_db.py, and reconciliation modules.

---

## Core Patterns

### Pattern 1: SAVEPOINT for Optional Sub-Queries

Use SAVEPOINTs when a sub-operation can fail without invalidating the parent operation.

```python
# BAD: Optional dedup check poisons the entire reconciliation transaction
async def reconcile_positions(conn):
    async with conn.transaction():
        positions = await conn.fetch("SELECT * FROM positions WHERE status='OPEN'")
        for pos in positions:
            # This timezone parse can fail — poisons ALL subsequent queries
            try:
                await conn.execute(
                    "INSERT INTO dedup_log (hash) VALUES ($1)",
                    compute_hash(pos)  # If this fails → InFailedSqlTransaction
                )
            except Exception:
                pass  # SILENT! Transaction is now poisoned. Next query WILL fail.

        # This query FAILS with InFailedSqlTransaction even though it's correct
        await conn.execute("UPDATE positions SET last_reconciled = NOW()")

# GOOD: SAVEPOINT isolates the optional operation
async def reconcile_positions(conn):
    async with conn.transaction():
        positions = await conn.fetch("SELECT * FROM positions WHERE status='OPEN'")
        for pos in positions:
            sp = await conn.execute("SAVEPOINT sp_dedup")
            try:
                await conn.execute(
                    "INSERT INTO dedup_log (hash) VALUES ($1)",
                    compute_hash(pos)
                )
                await conn.execute("RELEASE SAVEPOINT sp_dedup")
            except Exception:
                await conn.execute("ROLLBACK TO SAVEPOINT sp_dedup")
                logger.warning(f"Dedup failed for {pos['symbol']} — skipped, not fatal")

        # This query succeeds — transaction was never poisoned
        await conn.execute("UPDATE positions SET last_reconciled = NOW()")
```

### Pattern 2: Separate Connection for Risky Operations

When a sub-operation is complex and failure-prone, use an entirely separate connection.

```python
# BAD: Dedup check shares connection with critical update
async def update_position(conn, position_id, new_stop):
    async with conn.transaction():
        # Complex dedup check — multiple joins, string parsing, timezone conversion
        is_dup = await check_complex_dedup(conn, position_id)  # Can fail!
        if not is_dup:
            await conn.execute(
                "UPDATE positions SET stop_loss = $1 WHERE id = $2 AND status = 'OPEN'",
                new_stop, position_id
            )

# GOOD: Separate connection for risky dedup check
async def update_position(pool, position_id, new_stop):
    # Risky check on separate connection — failure can't poison main transaction
    async with pool.acquire() as dedup_conn:
        try:
            is_dup = await check_complex_dedup(dedup_conn, position_id)
        except Exception:
            is_dup = False  # Fail-open for dedup is OK (better to check than miss update)
            logger.warning("Dedup check failed — proceeding with update")

    # Critical update on clean connection — guaranteed unpoisoned
    async with pool.acquire() as conn:
        async with conn.transaction():
            await conn.execute(
                "UPDATE positions SET stop_loss = $1 WHERE id = $2 AND status = 'OPEN'",
                new_stop, position_id
            )
```

### Pattern 3: UPDATE WHERE Status Guard

Every UPDATE on a trade/position table must include a status guard to prevent race conditions.

```python
# BAD: Concurrent close and management can overwrite each other
await conn.execute(
    "UPDATE positions SET stop_loss = $1 WHERE id = $2",
    new_stop, position_id
)
# Race: if another coroutine just closed this position, we've
# overwritten the closed_at, exit_price, and status fields!

# GOOD: Status guard prevents overwriting closed positions
result = await conn.execute(
    "UPDATE positions SET stop_loss = $1 WHERE id = $2 AND status = 'OPEN'",
    new_stop, position_id
)
if result == "UPDATE 0":
    logger.info(f"Position {position_id} already closed — skip stop update")
```

### Pattern 4: Explicit Cursor (No Tuple Unpacking)

```python
# BAD: Assumes connect() returns (conn, cursor) tuple
with connect() as (conn, cur):  # TypeError if connect() returns conn only
    cur.execute("SELECT ...")

# GOOD: Get cursor explicitly from connection
with connect() as conn:
    cur = conn.cursor()
    cur.execute("SELECT ...")
    rows = cur.fetchall()
```

### Pattern 5: Connection Pool Recovery

Broken connections must be discarded, not returned to the pool.

```python
# BAD: Return broken connection to pool — next borrower gets poisoned connection
try:
    conn.execute("SELECT 1")
except Exception:
    conn.rollback()
    pool.putconn(conn)  # Broken connection goes back to pool!

# GOOD: Discard broken connection
try:
    conn.execute("SELECT 1")
except Exception:
    try:
        conn.rollback()
        pool.putconn(conn)
    except Exception:
        pool.putconn(conn, close=True)  # Discard — don't return broken conn
        logger.error("Discarded broken connection from pool")
```

### Pattern 6: Transaction Scope Boundaries

```python
# BAD: One giant transaction for everything — any failure kills all
async with conn.transaction():
    await fetch_broker_positions()      # API call inside transaction!
    await reconcile_positions()         # Complex logic
    await update_analytics()            # Optional
    await send_telegram_alert()         # External service inside transaction!

# GOOD: Separate transactions for independent operations
# Step 1: Fetch data (no transaction needed)
broker_positions = await fetch_broker_positions()

# Step 2: Critical update (own transaction)
async with conn.transaction():
    await reconcile_positions(broker_positions)

# Step 3: Optional analytics (own transaction with SAVEPOINT)
async with conn.transaction():
    try:
        await update_analytics(broker_positions)
    except Exception:
        logger.warning("Analytics update failed — non-critical")

# Step 4: External notification (no transaction)
await send_telegram_alert(broker_positions)
```

---

## Detecting Transaction Poisoning

```python
# How to detect InFailedSqlTransaction before it cascades
import psycopg2

def is_transaction_healthy(conn) -> bool:
    """Check if connection's transaction is in a usable state."""
    return conn.info.transaction_status != psycopg2.extensions.TRANSACTION_STATUS_INERROR

# Use before critical operations
if not is_transaction_healthy(conn):
    conn.rollback()  # Reset the transaction state
    logger.error("Transaction was poisoned — rolled back before critical operation")
```

---

## Red Flags

| Red Flag | Why It's Dangerous | Correct Pattern |
|----------|-------------------|-----------------|
| `except: pass` inside a transaction | Swallows the error but transaction is STILL poisoned | SAVEPOINT + ROLLBACK TO, or separate connection |
| API calls inside transactions | Network timeout holds transaction lock open | Fetch data first, then open transaction for DB ops only |
| `UPDATE` without `WHERE status = 'OPEN'` | Race condition: concurrent close + management overwrite | Always include status guard on trade/position UPDATEs |
| Single transaction for critical + optional ops | Optional failure kills critical operation | Separate transactions per concern |
| `with connect() as (conn, cur):` | Tuple unpacking breaks if API changes | `with connect() as conn: cur = conn.cursor()` |
| Returning broken connections to pool | Next borrower gets poisoned connection | `pool.putconn(conn, close=True)` to discard |
| No `TRANSACTION_STATUS_INERROR` check | Poisoned transaction discovered too late | Check `conn.info.transaction_status` before critical ops |

---

## Integration

| Scenario | Related Skill |
|----------|--------------|
| Basic database safety patterns | `trading-bot-skills:database-safety-for-trading` |
| Position reconciliation queries | `trading-bot-skills:position-reconciliation` |
| Numeric NULL handling in queries | `trading-bot-skills:falsy-zero-and-sentinel-values` |
| Async database operations | `trading-bot-skills:async-reliability` |
| Order insert dedup | `trading-bot-skills:order-execution-integrity` |
