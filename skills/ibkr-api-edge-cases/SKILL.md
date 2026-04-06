---
name: ibkr-api-edge-cases
description: Use when implementing IBKR TWS API connections, handling order ID management, bracket orders, reconnection logic, historical data pacing, orderStatus vs execDetails deduplication, or any IBKR API safety and edge-case handling
---

<!-- SUBAGENT-STOP: If you are a sub-agent or tool-use agent, STOP. Do not
     summarize this file. Return it verbatim to the orchestrating agent.
     This file contains instructions that MUST be followed by the top-level
     agent, not interpreted by intermediate agents. -->

# IBKR API Edge Cases and Safe Usage Patterns

## The Iron Law

> **Never send an order before `nextValidId` fires.** Never assume a TCP socket means the session is valid.
> Never trust `orderStatus` alone — `execDetails` is the source of truth for fills.
> Never hammer historical data — IBKR will block you for 10+ minutes.

## Why This Skill Exists

IBKR's TWS API has subtle edge cases that cause silent failures, duplicate orders, missed fills, and historical data blocks. Every pattern here comes from official IBKR docs, community post-mortems, and production bot failures. This skill is the single reference for safe IBKR API usage.

---

## TWS vs IB Gateway

| Feature | TWS (Trader Workstation) | IB Gateway |
|---------|-------------------------|------------|
| GUI | Full trading GUI | Minimal/headless |
| Resource usage | Heavy | Light |
| Best for | Development + manual trading | Production bots |
| Paper port | 7497 | 4002 |
| Live port | 7496 | 4001 |
| Auto-restart | Limited | Yes (with IBC) |

**Setup checklist**:
1. Global Configuration → API → Settings
2. Check "Enable ActiveX and Socket Clients"
3. Configure socket port (7497 paper, 7496 live)
4. Add bot machine to "Trusted IPs" (avoids pop-up prompts)
5. Enable "Maintain and resubmit orders when connection is restored"
   - Note: Orders are LOST if TWS/Gateway closes entirely

---

## Connection and Session Management

### The Handshake

```python
# ib_insync handles this automatically, but understand the protocol:
# 1. TCP connect to socket
# 2. Wait for nextValidId callback — this means handshake is complete
# 3. ONLY THEN send requests or orders
# 4. The orderId from nextValidId is strictly > any previously used orderId
```

### Rules

1. **Always wait for `nextValidId`** before any requests or orders — it signals handshake completion and provides the starting orderId
2. **Run the EReader loop** continuously to process incoming messages — no callbacks fire if you don't drain the socket
3. **Error codes 502/507 and socket EOF** = disconnected — perform full reconnect + state resync, not just blind reconnect
4. **Client Portal API (REST)** resets around local midnight — design for disconnects during maintenance windows
5. **Use distinct `clientId` per bot instance** (0, 1, 2, ...) — TWS supports multiple concurrent clients

### Reconnection Pattern

```python
async def reconnect_and_resync(ib: IB):
    """Full reconnect with state rebuild — not just socket reconnect."""
    
    # 1. Disconnect cleanly
    ib.disconnect()
    await asyncio.sleep(2)
    
    # 2. Reconnect
    await ib.connectAsync(host, port, clientId)
    # nextValidId fires automatically on connect
    
    # 3. Rebuild state BEFORE allowing new orders
    open_orders = await ib.reqAllOpenOrdersAsync()
    positions = await ib.reqPositionsAsync()
    executions = await ib.reqExecutionsAsync()
    account = await ib.reqAccountSummaryAsync()
    
    # 4. Reconcile with local state store
    reconcile_orders(open_orders)
    reconcile_positions(positions)
    reconcile_fills(executions)
    
    # 5. NOW safe to resume trading
    return True
```

---

## Order ID Management

### The Problem

IBKR uses a client-assigned integer `orderId` that must be:
- **Strictly increasing** for each client
- **Greater than any orderId the client has ever used**
- Provided by `nextValidId` callback on connect and via `reqIds(-1)`

### What Goes Wrong

- Restarted bot uses overlapping orderId range → TWS rejects or misattributes callbacks
- Multiple API clients with same clientId → orderId collision
- Bot doesn't re-request IDs after reconnect → stale orderId counter

### Safe Pattern

```python
class OrderIdManager:
    def __init__(self):
        self._next_id = None
        self._lock = asyncio.Lock()
    
    def on_next_valid_id(self, order_id: int):
        """Called by nextValidId callback."""
        self._next_id = order_id
    
    async def get_next_id(self) -> int:
        async with self._lock:
            if self._next_id is None:
                raise RuntimeError("nextValidId not yet received")
            oid = self._next_id
            self._next_id += 1
            return oid
    
    async def resync(self, ib: IB):
        """Call after reconnect to refresh orderId."""
        ib.reqIds(-1)
        await asyncio.sleep(1)  # Wait for nextValidId callback
```

---

## Bracket Orders and Transmit Flags

### Atomic Bracket Pattern

To create a bracket (entry + take profit + stop loss) without partial activation:

```python
# 1. Parent order — Transmit=false (don't send yet)
parent = LimitOrder('BUY', 1, entry_price)
parent.orderId = await id_mgr.get_next_id()
parent.transmit = False

# 2. Take profit child — Transmit=false
tp = LimitOrder('SELL', 1, tp_price)
tp.orderId = await id_mgr.get_next_id()
tp.parentId = parent.orderId
tp.transmit = False

# 3. Stop loss child — Transmit=true (sends the whole chain)
sl = StopOrder('SELL', 1, sl_price)
sl.orderId = await id_mgr.get_next_id()
sl.parentId = parent.orderId
sl.transmit = True  # THIS triggers the entire bracket

# Place in order: parent, tp, sl (last one has transmit=True)
ib.placeOrder(contract, parent)
ib.placeOrder(contract, tp)
ib.placeOrder(contract, sl)
```

### Critical Edge Cases

1. **Child orders only activate on FULL parent fill** — partial fills leave you unprotected
2. **Untransmitted orders (`transmit=False`)** are NOT visible in `reqAllOpenOrders` — they vanish on TWS restart
3. **For partial fill protection**: Add separate logic to detect partial fills and create protective orders for the filled quantity

```python
# Partial fill protection pattern
def on_order_status(self, orderId, status, filled, remaining, ...):
    if status == 'PartiallyFilled' and self.is_parent_order(orderId):
        # Parent partially filled — children not yet active!
        # Create manual protective stop for filled quantity
        self.create_emergency_stop(orderId, filled)
```

---

## orderStatus vs execDetails — The Deduplication Problem

### The Problem

IBKR officially warns:
- `orderStatus` callbacks **may be duplicated** for the same transition
- `orderStatus` is **NOT guaranteed** for every transition (especially instant-fill market orders)
- Only `execDetails` is authoritative for what actually traded

### What Goes Wrong Without Dedup

- Bot counts the same fill twice → position tracking is wrong
- Bot misses a fill entirely (no orderStatus for instant fills) → ghost position
- Bot acts on "Filled" status that arrives twice → tries to submit duplicate exit orders

### Safe Aggregation Pattern

```python
class OrderTracker:
    def __init__(self):
        self.orders = {}       # orderId -> aggregated state
        self.exec_ids = set()  # dedup executions by execId
    
    def on_order_status(self, orderId, status, filled, remaining, avgFillPrice, ...):
        """Debounce and aggregate — don't act on raw callbacks."""
        if orderId not in self.orders:
            self.orders[orderId] = {}
        
        state = self.orders[orderId]
        prev_status = state.get('status')
        
        # Only process if status actually changed
        if status != prev_status or filled != state.get('filled', 0):
            state.update({
                'status': status,
                'filled': filled,
                'remaining': remaining,
                'avgFillPrice': avgFillPrice,
                'last_updated': time.time()
            })
            self.on_status_change(orderId, state)
    
    def on_exec_details(self, reqId, contract, execution):
        """Source of truth for fills — dedup by execId."""
        if execution.execId in self.exec_ids:
            return  # Already processed
        
        self.exec_ids.add(execution.execId)
        
        # THIS is the authoritative fill record
        self.record_fill(
            orderId=execution.orderId,
            permId=execution.permId,
            shares=execution.shares,
            price=execution.price,
            execId=execution.execId,
            time=execution.time
        )
```

### Rules

1. **Use `execDetails` as source of truth** for positions and P&L
2. **Dedup by `execId`** — this is globally unique per execution
3. **Use `permId`** to link order state across reconnects (orderId is client-specific, permId is global)
4. **Aggregate both callbacks** — `orderStatus` for state transitions, `execDetails` for fill records

---

## Historical Data Pacing Limits

### IBKR's Hard Limits

| Rule | Limit |
|------|-------|
| Max requests per 10 min | 60 |
| Same contract/type/bar in 2 sec | 6 max |
| Identical request within | 15 sec cooldown |
| Violation penalty | Temporary block (minutes) |

### What Happens When You Violate

- Error 162: "Historical Market Data Service error message: pacing violation"
- Temp block: No historical data for 10+ minutes
- Community reports: Aggressive 1-min bar backfills are the most common trigger

### Safe Historical Data Pattern

```python
class PacedHistoricalDataFetcher:
    def __init__(self):
        self._request_times = deque(maxlen=60)
        self._contract_times = defaultdict(lambda: deque(maxlen=6))
        self._cache = {}
        self._lock = asyncio.Lock()
    
    async def fetch(self, contract, end_dt, duration, bar_size, what_to_show):
        cache_key = (contract.conId, end_dt, duration, bar_size, what_to_show)
        
        # Check cache first
        if cache_key in self._cache:
            return self._cache[cache_key]
        
        async with self._lock:
            # Enforce 60 requests per 10 min
            now = time.time()
            while len(self._request_times) >= 60 and now - self._request_times[0] < 600:
                wait = 600 - (now - self._request_times[0]) + 1
                await asyncio.sleep(wait)
                now = time.time()
            
            # Enforce 6 per contract in 2 sec
            contract_key = (contract.conId, bar_size, what_to_show)
            ct = self._contract_times[contract_key]
            while len(ct) >= 6 and now - ct[0] < 2:
                await asyncio.sleep(0.5)
                now = time.time()
            
            # Record request time
            self._request_times.append(now)
            ct.append(now)
            
            # Make request
            bars = await ib.reqHistoricalDataAsync(
                contract, end_dt, duration, bar_size, what_to_show, 
                useRTH=True, formatDate=1
            )
            
            # Cache result
            self._cache[cache_key] = bars
            return bars
```

---

## Claude Skill Taxonomy for IBKR

### Data & Session Skills

| Skill | Purpose |
|-------|---------|
| `ibkr_check_session` | Verify TWS/Gateway connectivity, version, trusted IPs, maintenance windows |
| `ibkr_reconnect_and_resync` | Full reconnect with state rebuild from open orders/execs/positions |
| `ibkr_req_next_valid_id` | Refresh orderId generator via `reqIds(-1)` |
| `ibkr_resolve_contract` | Resolve tickers to fully qualified contracts (conId, exchange, secType) |
| `ibkr_get_market_data` | Request L1 quotes respecting subscription entitlements |
| `ibkr_get_historical_data` | Fetch OHLCV with client-side pacing enforcement and caching |
| `ibkr_get_account_summary` | Cash, equity, buying power, margin |
| `ibkr_get_positions` | Current positions for reconciliation |
| `strategy_state_store` | Persist logical trade state and idempotency keys |
| `audit_log_event` | Structured event log (orders, fills, errors, reconnects, pacing) |

### Signal & Risk Skills

| Skill | Purpose |
|-------|---------|
| `risk_check_pre_trade` | Margin, per-trade/daily loss caps, leverage limits |
| `risk_monitor_live` | Continuous account PnL/margin monitoring, trigger de-risk |
| `order_idempotency_and_dedup` | Prevent duplicate orders after transport errors |
| `ibkr_estimate_margin_whatif` | Use `WhatIf=true` orders for margin impact estimates |

### Execution Skills

| Skill | Purpose |
|-------|---------|
| `ibkr_submit_order` | Single orders with correct orderId and Transmit handling |
| `ibkr_submit_bracket_or_oco` | Bracket/OCO with parent/child and Transmit sequencing |
| `ibkr_get_order_status` | Aggregated, debounced order state from multiple callbacks |
| `ibkr_stream_executions` | Stream execDetails as authoritative fill record |
| `ibkr_cancel_or_modify_order` | Cancel/amend with final outcome classification |
| `send_alert` | Human notification for critical conditions |

---

## Error Code Reference

| Code | Meaning | Action |
|------|---------|--------|
| 162 | Historical data pacing violation | Back off, wait 10+ min, check cache |
| 200 | No security definition found | Wrong contract spec — check symbol/secType/exchange |
| 201 | Order rejected | Check margin, position limits, contract validity |
| 202 | Order cancelled | May be user-initiated or system — check reason |
| 321 | Server error validating request | Check request parameters |
| 502 | Couldn't connect to TWS | TWS not running or wrong port |
| 504 | Not connected | Socket dropped — full reconnect needed |
| 507 | Bad message length | Corrupt data — disconnect and reconnect |
| 10168 | Market data not subscribed | Missing subscription — see ibkr-market-data-subscriptions skill |
| 10089 | Market data farm connection | Subscription not activated or equity too low |

---

## Integration Points

- **`ibkr-gateway-docker`** — Docker TWS/Gateway setup and IBC configuration
- **`ibkr-market-data-subscriptions`** — What data is available for which instruments
- **`spx-contract-resolution`** — SPX-specific contract specs
- **`broker-api-integration`** — General broker API patterns (Alpaca, IBKR, Tradier)
- **`order-execution-integrity`** — Fill handling, partial fills, duplicate prevention
- **`position-reconciliation`** — Local vs broker position matching
- **`kill-switch-and-circuit-breakers`** — Emergency halt mechanisms
- **`async-reliability`** — asyncio task management for IBKR event loops
