---
name: multi-engine-coordination
description: "Use when running multiple trading strategies concurrently, implementing cross-strategy deduplication, managing position ownership across engines, or preventing duplicate entries from multiple signal sources"
---

# Multi-Engine Coordination

Running multiple trading engines concurrently (EMA Cloud, ORB, SPX 0DTE, Flow signals) creates coordination challenges that don't exist with a single strategy. Without a unified position view and shared dedup gate, engines can independently open duplicate positions, fight over closing logic, and corrupt each other's state.

---

## The Iron Law

> **ONE POSITION TABLE. ONE DEDUP GATE. ALL ENGINES CHECK THE SAME STATE. NO ENGINE OWNS A POSITION — THE SYSTEM OWNS POSITIONS.**
>
> Origin: emabot entered SPY 11 times in one session because 4 entry paths (Cloud event, Chat learning, Chat signal, Flow webhook) each maintained independent cooldown dicts. Each path thought it was making the first entry. Commit dfc582d added a cross-engine duplicate guard checking all 3 position tables.

---

## Core Patterns

### Pattern 1: Unified Position Table

```python
# BAD: Separate tables per engine — can't see each other's positions
# trades (for equities), options_trades (for options), spx_trades (for 0DTE)
# Engine A checks trades table → "no SPY position" → enters
# Engine B checks options_trades → "no SPY position" → enters
# Result: 2 SPY positions

# GOOD: Single positions table with engine column
CREATE TABLE positions (
    id SERIAL PRIMARY KEY,
    symbol VARCHAR(10) NOT NULL,
    direction VARCHAR(5) NOT NULL,  -- CALL, PUT, LONG, SHORT
    engine VARCHAR(20) NOT NULL,     -- 'ema_cloud', 'orb', 'spx_0dte', 'flow'
    status VARCHAR(20) NOT NULL DEFAULT 'OPEN',
    qty INTEGER NOT NULL,
    entry_price NUMERIC(12,4),
    opened_at TIMESTAMPTZ DEFAULT NOW(),
    -- ... other fields
    CONSTRAINT unique_open_per_symbol_direction
        UNIQUE (symbol, direction) WHERE status = 'OPEN'  -- Partial unique index
);
```

If you MUST use separate tables (legacy), create a unified view:

```sql
CREATE VIEW all_open_positions AS
    SELECT symbol, direction, 'equity' as engine, status FROM trades WHERE status = 'OPEN'
    UNION ALL
    SELECT symbol, direction, 'options' as engine, status FROM options_trades WHERE status = 'OPEN'
    UNION ALL
    SELECT symbol, direction, 'spx_0dte' as engine, status FROM spx_0dte_trades WHERE status = 'OPEN'
    UNION ALL
    SELECT symbol, direction, 'preclose' as engine, status FROM spx_preclose_trades WHERE status = 'OPEN';
```

### Pattern 2: Cross-Engine Dedup Gate

```python
async def has_open_position(db, symbol: str, direction: str | None = None) -> bool:
    """Check ALL position tables for open position. This is the UNIFIED dedup gate."""
    query = """
        SELECT EXISTS (
            SELECT 1 FROM all_open_positions
            WHERE symbol = $1
            AND ($2::text IS NULL OR direction = $2)
        )
    """
    return await db.fetchval(query, symbol.upper(), direction)

async def can_enter_position(
    db,
    symbol: str,
    direction: str,
    engine: str,
    cooldown: "UnifiedCooldown",
) -> tuple[bool, str]:
    """Unified entry gate — ALL engines must pass through this."""
    # Check 1: Already have open position?
    if await has_open_position(db, symbol, direction):
        return False, f"Open {direction} position exists for {symbol}"

    # Check 2: Global cooldown (shared across ALL engines)
    if not cooldown.check_and_set(symbol, direction):
        return False, f"{symbol} {direction} in cooldown"

    # Check 3: Max positions limit
    open_count = await db.fetchval(
        "SELECT COUNT(*) FROM all_open_positions"
    )
    if open_count >= MAX_OPEN_POSITIONS:
        return False, f"Max positions reached ({open_count}/{MAX_OPEN_POSITIONS})"

    return True, "cleared"
```

### Pattern 3: Global Cooldown (Shared Across All Engines)

```python
import time
import threading

class UnifiedCooldown:
    """ONE cooldown dict shared by ALL engines. Not per-engine."""

    def __init__(self, window_seconds: int = 300):
        self._cooldowns: dict[str, float] = {}
        self._lock = threading.Lock()
        self._window = window_seconds

    def check_and_set(self, symbol: str, direction: str) -> bool:
        """Returns True if allowed. Sets cooldown atomically."""
        key = f"{symbol.upper()}:{direction.upper()}"
        now = time.time()

        with self._lock:
            # Prune stale entries
            if len(self._cooldowns) > 1000:
                cutoff = now - self._window
                self._cooldowns = {
                    k: v for k, v in self._cooldowns.items() if v > cutoff
                }

            last = self._cooldowns.get(key)
            if last and (now - last) < self._window:
                return False

            self._cooldowns[key] = now
            return True

# CRITICAL: Single instance shared by ALL engines
unified_cooldown = UnifiedCooldown(window_seconds=300)
```

### Pattern 4: Nanosecond Client Order ID

```python
import time

def generate_client_order_id(engine: str, symbol: str, side: str) -> str:
    """Generate unique order ID. Nanosecond precision prevents collision
    when multiple engines submit within the same millisecond.
    """
    # BAD: Minute-level granularity — allows 60 duplicates/minute
    # f"{symbol}_{side}_{datetime.now().strftime('%H%M')}"

    # GOOD: Nanosecond + engine tag — globally unique
    ts_ns = time.time_ns()
    return f"{engine}_{symbol}_{side}_{ts_ns}"

# Alpaca returns 409 Conflict for duplicate client_order_id — this is SAFE
# It means "order already exists" which is the desired dedup behavior
```

### Pattern 5: Position Lifecycle and Ownership

```python
# Positions are owned by the SYSTEM, not by engines.
# Any engine can open, but closing follows rules:

class PositionManager:
    """Centralized position lifecycle. Engines request actions; manager executes."""

    async def request_close(
        self,
        position_id: int,
        reason: str,
        requesting_engine: str,
    ) -> bool:
        """Any engine can request close. Manager decides and executes."""
        position = await self.db.fetchrow(
            "SELECT * FROM positions WHERE id = $1 AND status = 'OPEN'",
            position_id,
        )
        if not position:
            return False  # Already closed by another engine

        # Log which engine requested the close
        await self.db.execute(
            "UPDATE positions SET status = 'CLOSED', closed_by = $1, "
            "close_reason = $2, closed_at = NOW() WHERE id = $3 AND status = 'OPEN'",
            requesting_engine, reason, position_id,
        )
        return True

    async def get_positions_for_engine(self, engine: str) -> list:
        """Engine can see its own positions, but dedup checks ALL."""
        return await self.db.fetch(
            "SELECT * FROM positions WHERE engine = $1 AND status = 'OPEN'",
            engine,
        )
```

### Pattern 6: Engine Registration and Health

```python
class EngineRegistry:
    """Track which engines are alive. Dead engines' positions need attention."""

    def __init__(self):
        self._engines: dict[str, float] = {}  # engine_name → last_heartbeat

    def heartbeat(self, engine: str):
        self._engines[engine] = time.time()

    def get_dead_engines(self, max_age: float = 120) -> list[str]:
        """Engines that haven't sent heartbeat in max_age seconds."""
        now = time.time()
        return [
            name for name, last_hb in self._engines.items()
            if (now - last_hb) > max_age
        ]

    async def handle_dead_engines(self, position_manager):
        """Transfer positions from dead engines to system management."""
        for engine in self.get_dead_engines():
            positions = await position_manager.get_positions_for_engine(engine)
            for pos in positions:
                logger.warning(
                    f"Engine {engine} dead — position {pos['symbol']} needs management"
                )
                # System takes over: apply default stop-loss
                await position_manager.apply_emergency_management(pos["id"])
```

---

## Red Flags

| Red Flag | Why It's Dangerous | Correct Pattern |
|----------|-------------------|-----------------|
| Per-engine cooldown dict | Same symbol entered by multiple engines | Single `UnifiedCooldown` shared by ALL engines |
| Checking only one trade table for dedup | Other engine's positions invisible | Query `all_open_positions` view (all tables) |
| Minute-level client_order_id | Multiple orders in same minute get same ID | Nanosecond `time.time_ns()` + engine prefix |
| Engine closes positions it doesn't "own" | Race: two engines close same position simultaneously | `UPDATE ... WHERE status = 'OPEN'` guard + `closed_by` tracking |
| No engine health monitoring | Dead engine's positions unmanaged indefinitely | Engine registry with heartbeat + dead-engine handler |
| Direct broker calls from strategy code | Bypasses unified dedup gate | All entries go through `can_enter_position()` |

---

## Integration

| Scenario | Related Skill |
|----------|--------------|
| Order dedup and idempotency | `trading-bot-skills:order-execution-integrity` |
| Signal dedup before engine routing | `trading-bot-skills:chat-signal-parsing-and-dedup` |
| Database atomic operations | `trading-bot-skills:database-transaction-patterns` |
| Position tracking and reconciliation | `trading-bot-skills:position-reconciliation` |
| Kill switch across all engines | `trading-bot-skills:kill-switch-and-circuit-breakers` |
| Distributed multi-bot coordination | `trading-bot-skills:distributed-trading-patterns` |
