# Alpaca Broker API Reference

This is a companion reference to the `broker-api-integration` skill, specific to the
Alpaca Markets API.

---

## API Architecture: REST vs WebSocket

Alpaca provides two communication channels. You need BOTH.

| Channel | Use For | Endpoint (Paper) | Endpoint (Live) |
|---|---|---|---|
| REST (Trading) | Order submission, cancellation, account info, positions | `https://paper-api.alpaca.markets` | `https://api.alpaca.markets` |
| REST (Data) | Historical bars, quotes, snapshots | `https://data.alpaca.markets` | `https://data.alpaca.markets` |
| WebSocket (Trading) | Real-time order updates (fills, cancellations) | `wss://paper-api.alpaca.markets/stream` | `wss://api.alpaca.markets/stream` |
| WebSocket (Data) | Real-time quotes, trades, bars | `wss://stream.data.alpaca.markets/v2/sip` (paid) or `wss://stream.data.alpaca.markets/v2/iex` (free) | Same |

**Critical:** The trading WebSocket (`/stream`) is how you receive fill notifications.
If this connection drops and you don't detect it, you will miss fills and your local
state will diverge from the broker.

---

## API Key Management

```python
import os

# Paper trading
PAPER_API_KEY = os.environ["ALPACA_PAPER_API_KEY"]
PAPER_API_SECRET = os.environ["ALPACA_PAPER_API_SECRET"]
PAPER_BASE_URL = "https://paper-api.alpaca.markets"

# Live trading
LIVE_API_KEY = os.environ["ALPACA_LIVE_API_KEY"]
LIVE_API_SECRET = os.environ["ALPACA_LIVE_API_SECRET"]
LIVE_BASE_URL = "https://api.alpaca.markets"

# NEVER hardcode keys. NEVER use the same keys for paper and live.
# ALWAYS verify the base URL matches the intended mode before startup.
```

### Paper vs Live Keys

- Paper keys start with `PK` (key) and `PS` (secret) -- but do NOT rely on this prefix
  for safety checks. Always verify the URL.
- Live keys start with `AK` / `AS` -- again, verify the URL.
- Paper and live are completely separate accounts. Positions, orders, and balances are
  independent.

---

## Order Lifecycle Events

When you submit an order, it moves through states. Alpaca sends events via WebSocket.

```
         submit
           |
           v
         [new] --------> [rejected]
           |
           v
      [accepted] ------> [cancelled]
           |                  ^
           v                  |
    [partially_filled] -------+
           |
           v
        [filled]
           |
           v
       [expired]  (if TIF=DAY and market closes)
```

### Event Stream Format (Trade Updates)

```json
{
  "stream": "trade_updates",
  "data": {
    "event": "fill",
    "order": {
      "id": "broker-uuid",
      "client_order_id": "SPY-buy-a3f8c1d9e7b24510",
      "symbol": "SPY",
      "side": "buy",
      "type": "market",
      "qty": "10",
      "filled_qty": "10",
      "filled_avg_price": "452.35",
      "status": "filled",
      "created_at": "2025-01-15T14:30:00Z",
      "updated_at": "2025-01-15T14:30:01Z"
    },
    "timestamp": "2025-01-15T14:30:01Z",
    "price": "452.35",
    "qty": "10",
    "position_qty": "10"
  }
}
```

### Key Events to Handle

| Event | Meaning | Action |
|---|---|---|
| `new` | Order accepted by Alpaca | Update local state to SUBMITTED |
| `partial_fill` | Some shares filled | Update `filled_qty`, check if stop-loss needs adjustment |
| `fill` | Order fully filled | Update position, set stop-loss, record entry |
| `canceled` | Order cancelled (by you or broker) | Clean up pending state, check if partial was filled |
| `rejected` | Order rejected (insufficient funds, etc.) | Log reason, alert, clean up pending state |
| `expired` | DAY order expired at market close | Clean up, decide if resubmit tomorrow |
| `replaced` | Order was modified (qty or price change) | Update local order record |

---

## Partial Fill Handling

This is where most bots break. A partial fill means your order is BOTH filled AND pending.

```python
async def handle_trade_update(event: dict):
    """Handle Alpaca trade update events."""
    data = event.get("data", {})
    event_type = data.get("event")
    order_data = data.get("order", {})

    client_order_id = order_data.get("client_order_id")
    symbol = order_data.get("symbol")
    filled_qty = int(order_data.get("filled_qty", 0))
    filled_avg_price = float(order_data.get("filled_avg_price", 0)) if order_data.get("filled_avg_price") else None

    if event_type == "partial_fill":
        # CRITICAL: Update position with ACTUAL filled qty, not requested qty
        fill_qty_this_event = int(data.get("qty", 0))
        fill_price_this_event = float(data.get("price", 0))

        logger.info(
            f"Partial fill: {symbol} +{fill_qty_this_event} @ {fill_price_this_event} "
            f"(total filled: {filled_qty})"
        )

        # Update local position tracking
        await update_position_from_fill(
            symbol=symbol,
            filled_qty=filled_qty,  # cumulative from broker
            avg_price=filled_avg_price,
            client_order_id=client_order_id,
            is_final=False,
        )

    elif event_type == "fill":
        logger.info(f"Full fill: {symbol} {filled_qty} @ {filled_avg_price}")
        await update_position_from_fill(
            symbol=symbol,
            filled_qty=filled_qty,
            avg_price=filled_avg_price,
            client_order_id=client_order_id,
            is_final=True,
        )

    elif event_type == "canceled":
        # CHECK: was there a partial fill before cancellation?
        if filled_qty > 0:
            logger.warning(
                f"Order cancelled with partial fill! {symbol} has {filled_qty} shares. "
                f"This position MUST be managed."
            )
            await update_position_from_fill(
                symbol=symbol,
                filled_qty=filled_qty,
                avg_price=filled_avg_price,
                client_order_id=client_order_id,
                is_final=True,  # treat as final since order is done
            )
        else:
            await cleanup_pending_order(client_order_id)

    elif event_type == "rejected":
        reason = order_data.get("reject_reason", "unknown")
        logger.error(f"Order rejected: {symbol} - {reason}")
        await cleanup_pending_order(client_order_id)
        await send_alert(f"ORDER REJECTED: {symbol} - {reason}")
```

---

## alpaca-py SDK Patterns

The official `alpaca-py` SDK. Use it instead of raw HTTP when possible.

### Setup

```python
from alpaca.trading.client import TradingClient
from alpaca.trading.requests import MarketOrderRequest, LimitOrderRequest
from alpaca.trading.enums import OrderSide, TimeInForce
from alpaca.trading.stream import TradingStream

# Paper client
paper_client = TradingClient(
    api_key=PAPER_API_KEY,
    secret_key=PAPER_API_SECRET,
    paper=True,  # CRITICAL: set this flag
)

# Live client -- SEPARATE INSTANCE
live_client = TradingClient(
    api_key=LIVE_API_KEY,
    secret_key=LIVE_API_SECRET,
    paper=False,
)
```

### Submitting Orders

```python
from decimal import Decimal

# Market order with client_order_id (MANDATORY)
request = MarketOrderRequest(
    symbol="SPY",
    qty=10,
    side=OrderSide.BUY,
    time_in_force=TimeInForce.DAY,
    client_order_id=generate_client_order_id("sig_001", "SPY", "buy", bucket),
)

# Always wrap in protected_broker_call (see broker-api-integration skill)
order = await protected_broker_call(
    paper_client.submit_order,
    request,
    circuit_breaker=order_circuit_breaker,
    rate_limiter=order_rate_limiter,
    timeout=10.0,
)
```

### Streaming Trade Updates

```python
stream = TradingStream(
    api_key=PAPER_API_KEY,
    secret_key=PAPER_API_SECRET,
    paper=True,
)

@stream.on("trade_updates")
async def on_trade_update(data):
    """Called for every order lifecycle event."""
    event = data.event
    order = data.order

    logger.info(f"Trade update: {event} for {order.symbol} ({order.client_order_id})")

    if event == "fill":
        await handle_fill(order)
    elif event == "partial_fill":
        await handle_partial_fill(order, data)
    elif event == "canceled":
        await handle_cancel(order)
    elif event == "rejected":
        await handle_rejection(order)

# Run in background -- but wrap with reconnection logic!
# The alpaca-py stream does NOT auto-reconnect reliably.
# You MUST add your own stale detection and reconnect wrapper.
```

---

## Common Pitfalls

### 1. Rate Limits

- **Trading API:** 200 requests/minute
- **Data API:** Varies by plan (free IEX: 200/min, SIP: higher)
- Alpaca returns HTTP 429 when exceeded. Back off immediately.
- Your rate limiter should use a buffer (e.g., cap at 180/min not 200/min).

### 2. Market Hours

- Regular: 9:30 AM - 4:00 PM ET
- Extended: 4:00 AM - 8:00 PM ET (if enabled on account)
- DAY orders are only active during regular hours
- If you submit a DAY market order at 4:01 PM, it will be **queued for next open**
  (not rejected!). This catches many bot developers off guard.
- Use `TimeInForce.DAY` and check `clock.is_open` before submitting.

```python
clock = paper_client.get_clock()
if not clock.is_open:
    logger.warning("Market is closed. DAY orders will queue for next open.")
    # Decide: skip, or use GTC, or wait
```

### 3. Pattern Day Trader (PDT) Rule

- Accounts under $25,000 equity: limited to 3 day trades per 5 rolling business days
- A "day trade" = buy and sell (or sell and buy) same security same day
- Alpaca tracks this and will **reject** orders that would violate PDT
- Check `account.pattern_day_trader` and `account.daytrade_count`
- Your bot MUST track day trade count and refuse entries that would trigger PDT

```python
account = paper_client.get_account()
if float(account.equity) < 25000:
    # PDT applies -- check day trade count
    if account.daytrade_count >= 3:
        logger.error("PDT limit reached. No more day trades until count resets.")
        # DO NOT submit the order
```

### 4. Fractional Shares

- Alpaca supports fractional shares for some symbols
- Fractional orders must be `market` + `day` -- no limit, no GTC
- Not all symbols support fractional. Check before submitting.
- Fractional fills come as a single fill (no partial fills)

### 5. The Alpaca WebSocket "Silent Death"

The Alpaca trading WebSocket (`/stream`) can go stale without sending a close frame.
Your connection looks alive but no events arrive. This is catastrophic for a trading
bot because you miss all fill notifications.

**Solution:** Implement heartbeat/stale detection. If no message received in 30 seconds
during market hours, force disconnect and reconnect. See the `BrokerWebSocket` class in
the parent `broker-api-integration` skill.

### 6. Order ID Collisions

Alpaca requires `client_order_id` to be unique across the lifetime of the account (or
at least within a large window). If you reuse an ID, the order is rejected.

Your deterministic ID generator (from `broker-api-integration`) handles this by
including the timestamp bucket. But if you ever reset your signal IDs or change the
hashing scheme, verify uniqueness.

### 7. Overnight Positions and Gaps

- If you hold positions overnight, the opening price can gap significantly
- Stop-loss orders placed as stop-market will execute at the gap price, which may be
  far from your stop level
- Consider: stop-limit orders (risk of no fill), or reducing position size for
  overnight holds

---

## Alpaca API Endpoint Quick Reference

### Trading (v2)

| Method | Endpoint | Description |
|---|---|---|
| POST | `/v2/orders` | Submit order |
| GET | `/v2/orders/{id}` | Get order by ID |
| DELETE | `/v2/orders/{id}` | Cancel order |
| DELETE | `/v2/orders` | Cancel all orders |
| GET | `/v2/positions` | List all positions |
| DELETE | `/v2/positions/{symbol}` | Close position |
| GET | `/v2/account` | Account info |
| GET | `/v2/clock` | Market clock |
| GET | `/v2/calendar` | Market calendar |

### Headers

```
APCA-API-KEY-ID: {your_key}
APCA-API-SECRET-KEY: {your_secret}
Content-Type: application/json
```

---

## Integration

This reference supports the **trading-bot-skills:broker-api-integration** skill. All
patterns here (circuit breaker, idempotent IDs, rate limiting) are defined in the
parent skill and applied to Alpaca-specific calls.
