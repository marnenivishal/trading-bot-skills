---
name: database-safety-for-trading
description: Use when implementing database operations for trading systems, encountering transaction failures or stale data, or designing position and order persistence
---

# Database Safety for Trading

Prevents transaction poisoning (incident #1) and stale audit data (incident #5).
In trading systems, a corrupted transaction doesn't just cause an error -- it
causes incorrect position data, which causes incorrect risk calculations, which
causes incorrect order sizing. A single poisoned transaction can cascade through
the entire system.

---

## The Iron Law

> **EVERY DATABASE OPERATION MUST HANDLE TRANSACTION FAILURE INDEPENDENTLY.**
>
> Never batch unrelated operations in a single transaction. Never assume
> a transaction will succeed. Never silently swallow a database error.
>
> Origin: Incident #1 -- Transaction Poisoning. A single database transaction
> contained: (a) update position, (b) log to audit trail, (c) update P&L
> cache. The audit log insert failed due to a constraint violation. The
> ENTIRE transaction rolled back, including the position update. The bot
> continued trading with stale position data. By the time the error was
> noticed, actual exposure was 5x the tracked exposure.

---

## Transaction Isolation Rules

### Rule 1: One Concern Per Transaction

```python
# BAD: Multiple concerns in one transaction
async def process_fill(pool, fill):
    async with pool.acquire() as conn:
        async with conn.transaction():
            # If ANY of these fail, ALL roll back
            await conn.execute("UPDATE positions SET qty = qty + $1 WHERE symbol = $2",
                               fill.qty, fill.symbol)
            await conn.execute("INSERT INTO audit_log (event, data) VALUES ($1, $2)",
                               'fill', json.dumps(fill))
            await conn.execute("UPDATE pnl_cache SET realized = realized + $1",
                               fill.pnl)

# GOOD: Each concern is independent
async def process_fill(pool, fill):
    # Critical: Position update (must succeed)
    await update_position(pool, fill)
    # Important: Audit log (should succeed, but don't block on failure)
    await log_fill_to_audit(pool, fill)
    # Nice-to-have: Cache update (can be recalculated)
    await update_pnl_cache(pool, fill)

async def update_position(pool, fill):
    async with pool.acquire() as conn:
        async with conn.transaction():
            await conn.execute(
                "UPDATE positions SET qty = qty + $1, updated_at = NOW() "
                "WHERE symbol = $2",
                fill.qty, fill.symbol,
            )

async def log_fill_to_audit(pool, fill):
    try:
        async with pool.acquire() as conn:
            async with conn.transaction():
                await conn.execute(
                    "INSERT INTO audit_log (event_type, data, created_at) "
                    "VALUES ($1, $2, NOW())",
                    'fill', json.dumps(fill.to_dict()),
                )
    except Exception:
        logger.exception("Audit log write failed -- ALERT but don't block trading")
        # Queue for retry, alert monitoring, but do NOT re-raise
        await alert_monitoring("audit_log_write_failed", fill)
```

### Rule 2: Use SAVEPOINTs for Partial Rollback

When operations within a logical group have different criticality:

```python
async def process_order_result(conn, order_result):
    """Process an order result with savepoints for partial rollback."""
    async with conn.transaction():
        # Critical: record the order state
        await conn.execute(
            "UPDATE orders SET status = $1, broker_id = $2, updated_at = NOW() "
            "WHERE idempotency_key = $3",
            order_result.status, order_result.broker_id, order_result.idempotency_key,
        )

        # Non-critical: update analytics (use savepoint)
        savepoint = await conn.execute("SAVEPOINT analytics_update")
        try:
            await conn.execute(
                "INSERT INTO order_analytics (symbol, latency_ms, slippage) "
                "VALUES ($1, $2, $3)",
                order_result.symbol, order_result.latency_ms, order_result.slippage,
            )
        except Exception:
            await conn.execute("ROLLBACK TO SAVEPOINT analytics_update")
            logger.warning("Analytics insert failed, continuing without it")
        else:
            await conn.execute("RELEASE SAVEPOINT analytics_update")
```

### Rule 3: Explicit Timeouts on Every Transaction

```python
# BAD: No timeout -- can hold a connection indefinitely
async with conn.transaction():
    await conn.execute(long_running_query)

# GOOD: Explicit timeout
async with asyncio.timeout(5.0):
    async with conn.transaction():
        await conn.execute(
            "SET LOCAL statement_timeout = '5000';"
        )
        await conn.execute(query)
```

---

## Freshness Guarantees

### Rule: Every Query That Drives Trading Decisions Must Check Freshness

```python
from datetime import datetime, timezone, timedelta

MAX_STALENESS = timedelta(seconds=30)

async def get_position(conn, symbol: str) -> Position:
    """Get position with freshness guarantee."""
    row = await conn.fetchrow(
        "SELECT symbol, qty, updated_at FROM positions WHERE symbol = $1",
        symbol,
    )
    if row is None:
        return Position(symbol=symbol, qty=Decimal("0"), is_fresh=True)

    age = datetime.now(timezone.utc) - row["updated_at"]
    if age > MAX_STALENESS:
        logger.warning(
            f"STALE POSITION DATA: {symbol} last updated {age.total_seconds():.1f}s ago"
        )
        # Do NOT silently return stale data
        raise StaleDataError(
            f"Position for {symbol} is {age.total_seconds():.1f}s stale "
            f"(max {MAX_STALENESS.total_seconds():.0f}s). "
            f"Reconcile before trading."
        )

    return Position(
        symbol=symbol,
        qty=row["qty"],
        updated_at=row["updated_at"],
        is_fresh=True,
    )


class StaleDataError(Exception):
    """Raised when data is too old to make trading decisions."""
    pass
```

### Freshness for Aggregates

```python
async def get_total_exposure(conn) -> Decimal:
    """Get total portfolio exposure with freshness check on ALL positions."""
    rows = await conn.fetch(
        "SELECT symbol, qty, updated_at FROM positions WHERE qty != 0"
    )
    now = datetime.now(timezone.utc)
    stale_symbols = []

    for row in rows:
        age = now - row["updated_at"]
        if age > MAX_STALENESS:
            stale_symbols.append((row["symbol"], age.total_seconds()))

    if stale_symbols:
        raise StaleDataError(
            f"Cannot calculate exposure: stale positions: "
            + ", ".join(f"{s} ({a:.0f}s)" for s, a in stale_symbols)
        )

    return sum(abs(row["qty"]) for row in rows)
```

---

## Idempotent Writes

### Rule: Every Write That Can Be Retried Must Be Idempotent

Trading systems retry operations. Network failures, timeouts, and process
restarts all cause retries. If a write is not idempotent, retries create
duplicates.

```python
# BAD: Non-idempotent -- retry creates duplicate
async def record_fill(conn, fill):
    await conn.execute(
        "INSERT INTO fills (symbol, qty, price, filled_at) "
        "VALUES ($1, $2, $3, $4)",
        fill.symbol, fill.qty, fill.price, fill.filled_at,
    )

# GOOD: Idempotent with ON CONFLICT
async def record_fill(conn, fill):
    await conn.execute(
        "INSERT INTO fills (idempotency_key, symbol, qty, price, filled_at) "
        "VALUES ($1, $2, $3, $4, $5) "
        "ON CONFLICT (idempotency_key) DO NOTHING",
        fill.idempotency_key, fill.symbol, fill.qty, fill.price, fill.filled_at,
    )
```

### Idempotent Position Updates

```python
# BAD: Non-idempotent -- retry double-counts
async def apply_fill_to_position(conn, fill):
    await conn.execute(
        "UPDATE positions SET qty = qty + $1 WHERE symbol = $2",
        fill.qty, fill.symbol,
    )

# GOOD: Idempotent -- track which fills have been applied
async def apply_fill_to_position(conn, fill):
    """Apply fill to position idempotently using fill tracking."""
    async with conn.transaction():
        # Try to record this fill (will fail if already applied)
        result = await conn.execute(
            "INSERT INTO applied_fills (idempotency_key, symbol, qty) "
            "VALUES ($1, $2, $3) "
            "ON CONFLICT (idempotency_key) DO NOTHING",
            fill.idempotency_key, fill.symbol, fill.qty,
        )

        # If the insert did nothing, this fill was already applied
        if result == "INSERT 0":
            logger.info(f"Fill {fill.idempotency_key} already applied, skipping")
            return

        # Apply the fill
        await conn.execute(
            "UPDATE positions SET qty = qty + $1, updated_at = NOW() "
            "WHERE symbol = $2",
            fill.qty, fill.symbol,
        )
```

---

## Connection Pool Safety

### Rule: Never Hold a Connection Across Async Boundaries

```python
# BAD: Holds connection while waiting for external service
async def process_and_record(pool, fill):
    async with pool.acquire() as conn:
        await conn.execute("INSERT INTO fills ...")  # DB operation
        await broker.confirm_fill(fill)              # EXTERNAL CALL while holding connection!
        await conn.execute("UPDATE positions ...")   # More DB while connection was held

# GOOD: Separate DB operations from external calls
async def process_and_record(pool, fill):
    # Step 1: Record fill (quick, releases connection)
    async with pool.acquire() as conn:
        await conn.execute("INSERT INTO fills ...")

    # Step 2: External call (no connection held)
    confirmation = await broker.confirm_fill(fill)

    # Step 3: Update position (quick, new connection)
    async with pool.acquire() as conn:
        await conn.execute("UPDATE positions ...")
```

### Pool Configuration for Trading

```python
import asyncpg

async def create_trading_pool() -> asyncpg.Pool:
    """Create a connection pool with trading-appropriate settings."""
    pool = await asyncpg.create_pool(
        dsn=config.database_url,

        # Size: enough for concurrent operations, not so many
        # that you overwhelm the database
        min_size=2,
        max_size=10,

        # Timeouts: fail fast, don't hang
        command_timeout=10.0,        # Individual query timeout
        timeout=5.0,                 # Connection acquisition timeout

        # Statements: avoid prepared statement leaks
        max_cached_statement_lifetime=300,
        max_cacheable_statement_size=1024 * 16,

        # Health checks
        setup=_setup_connection,
    )
    return pool


async def _setup_connection(conn: asyncpg.Connection) -> None:
    """Configure each connection with trading-safe defaults."""
    # Set statement timeout at connection level
    await conn.execute("SET statement_timeout = '10000'")
    # Prevent accidental full table locks
    await conn.execute("SET lock_timeout = '5000'")
```

---

## Schema Design Patterns

### Positions Table

```sql
CREATE TABLE positions (
    symbol          TEXT PRIMARY KEY,
    qty             DECIMAL NOT NULL DEFAULT 0,
    avg_cost        DECIMAL NOT NULL DEFAULT 0,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    version         INTEGER NOT NULL DEFAULT 1  -- Optimistic locking
);

-- Index for freshness queries
CREATE INDEX idx_positions_updated ON positions(updated_at);
```

### Orders Table

```sql
CREATE TABLE orders (
    id              BIGSERIAL PRIMARY KEY,
    idempotency_key TEXT UNIQUE NOT NULL,       -- THE dedup mechanism
    symbol          TEXT NOT NULL,
    side            TEXT NOT NULL CHECK (side IN ('buy', 'sell')),
    qty             DECIMAL NOT NULL,
    order_type      TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'pending',
    broker_order_id TEXT,
    source          TEXT NOT NULL,              -- Which component created this
    reason          TEXT,                       -- Human-readable audit trail
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_orders_status ON orders(status) WHERE status IN ('pending', 'open');
CREATE INDEX idx_orders_symbol ON orders(symbol, created_at DESC);
```

### Fills Table

```sql
CREATE TABLE fills (
    id              BIGSERIAL PRIMARY KEY,
    idempotency_key TEXT UNIQUE NOT NULL,
    order_id        BIGINT REFERENCES orders(id),
    symbol          TEXT NOT NULL,
    side            TEXT NOT NULL,
    qty             DECIMAL NOT NULL,
    price           DECIMAL NOT NULL,
    broker_fill_id  TEXT,
    filled_at       TIMESTAMPTZ NOT NULL,
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### Applied Fills Tracking (for Idempotent Position Updates)

```sql
CREATE TABLE applied_fills (
    idempotency_key TEXT PRIMARY KEY,
    symbol          TEXT NOT NULL,
    qty             DECIMAL NOT NULL,
    applied_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

## Optimistic Locking for Positions

```python
async def update_position_optimistic(conn, symbol: str, fill) -> bool:
    """Update position with optimistic locking. Returns True if successful."""
    result = await conn.execute(
        "UPDATE positions "
        "SET qty = qty + $1, updated_at = NOW(), version = version + 1 "
        "WHERE symbol = $2 AND version = $3",
        fill.qty, symbol, fill.expected_version,
    )

    if result == "UPDATE 0":
        # Version mismatch -- someone else updated first
        logger.warning(
            f"Optimistic lock conflict on {symbol} "
            f"(expected version {fill.expected_version})"
        )
        return False

    return True
```

---

## Red Flags

| Red Flag                                          | Why It's Dangerous                                              | Correct Pattern                                |
|---------------------------------------------------|-----------------------------------------------------------------|------------------------------------------------|
| Single transaction wrapping unrelated operations  | One failure rolls back everything, corrupting critical state    | One transaction per concern                    |
| `except Exception: pass` after a DB operation     | Silences transaction failures; bot continues with stale state   | Log, alert, and handle explicitly              |
| `INSERT` without `ON CONFLICT` for retryable ops  | Retries create duplicates: double fills, double position updates| `ON CONFLICT (idempotency_key) DO NOTHING`     |
| Query driving trading decisions without freshness check | Stale data causes wrong position sizing and risk calculations | Check `updated_at` against MAX_STALENESS       |
| Holding DB connection during external API calls   | Connection pool exhaustion during broker latency spikes         | Separate DB ops from external calls            |
| No `statement_timeout` on queries                 | Runaway query blocks connection, starves other operations       | `SET statement_timeout = '10000'`              |
| `UPDATE positions SET qty = qty + $1` without dedup | Retry double-counts the fill                                  | Track applied fills, check before applying     |
| Shared connection across async tasks              | Transaction state leaks between tasks                           | Acquire per-operation, release immediately     |
| No `version` column on mutable trading tables     | Lost updates when concurrent operations modify same row         | Optimistic locking with version check          |

---

## Integration

| Scenario                                  | Invoke Skill                           |
|-------------------------------------------|----------------------------------------|
| Designing the order execution layer       | `order-execution-integrity`            |
| Implementing position state management    | `trading-bot-architecture`             |
| Setting up async DB operations            | `async-reliability`                    |
| Reconciling with broker state             | `state-reconciliation`                 |
| Building audit trail                      | `audit-trail-design`                   |
| Testing database operations               | `trading-tdd`                          |
| Managing connection secrets               | `secrets-and-api-key-management`       |
