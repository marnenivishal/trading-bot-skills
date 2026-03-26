# CEP Engine and Broker Adapter Reference

Complete reference implementation for Complex Event Processing (CEP) engines with temporal windowing and multi-broker adapter patterns.

---

## Full CEP Engine with Temporal Windowing

```python
from dataclasses import dataclass, field
from collections import defaultdict, deque
from datetime import datetime, timezone
from typing import Callable
import logging

logger = logging.getLogger(__name__)


@dataclass(frozen=True)
class MarketEvent:
    symbol: str
    timestamp: datetime
    event_type: str  # "trade", "quote", "bar", "volume_spike"
    price: float
    volume: int = 0
    metadata: dict = field(default_factory=dict)


@dataclass(frozen=True)
class Signal:
    source: str
    symbol: str
    timestamp: datetime = field(default_factory=lambda: datetime.now(timezone.utc))
    strength: float = 1.0
    metadata: dict = field(default_factory=dict)


@dataclass(frozen=True)
class EventRule:
    """Declarative rule for pattern detection.

    name: unique identifier for the rule
    window_seconds: how far back to look for events
    min_events: minimum number of events required in window
    condition: function that evaluates whether the pattern is present
    cooldown_seconds: minimum time between signal emissions for this rule
    """
    name: str
    window_seconds: float
    min_events: int
    condition: Callable[[list[MarketEvent]], bool]
    cooldown_seconds: float = 0.0


class CEPEngine:
    """Complex Event Processing engine with temporal windowing.

    Processes a stream of market events and evaluates declarative rules
    to detect patterns and emit trading signals.
    """

    def __init__(self, rules: list[EventRule], max_buffer_size: int = 10000):
        self.rules = rules
        self.max_buffer_size = max_buffer_size
        self.event_buffer: dict[str, deque[MarketEvent]] = defaultdict(
            lambda: deque(maxlen=max_buffer_size)
        )
        self._last_signal_time: dict[str, datetime] = {}

    async def process_event(self, event: MarketEvent) -> list[Signal]:
        """Process a single market event and return any triggered signals."""
        self.event_buffer[event.symbol].append(event)
        self._prune_expired(event.symbol)

        signals = []
        for rule in self.rules:
            if self._in_cooldown(rule, event.timestamp):
                continue

            events_in_window = self._get_window(event.symbol, rule.window_seconds)

            if len(events_in_window) >= rule.min_events:
                try:
                    if rule.condition(events_in_window):
                        signal = Signal(source=rule.name, symbol=event.symbol)
                        signals.append(signal)
                        self._last_signal_time[rule.name] = event.timestamp
                        logger.info(
                            f"CEP rule '{rule.name}' triggered for {event.symbol} "
                            f"({len(events_in_window)} events in window)"
                        )
                except Exception as e:
                    logger.error(f"CEP rule '{rule.name}' evaluation failed: {e}")

        return signals

    def _get_window(self, symbol: str, window_seconds: float) -> list[MarketEvent]:
        """Get events within the specified time window."""
        cutoff = datetime.now(timezone.utc).timestamp() - window_seconds
        return [
            e for e in self.event_buffer[symbol]
            if e.timestamp.timestamp() >= cutoff
        ]

    def _prune_expired(self, symbol: str) -> None:
        """Remove events older than the longest rule window."""
        if not self.rules:
            return
        max_window = max(r.window_seconds for r in self.rules)
        cutoff = datetime.now(timezone.utc).timestamp() - max_window * 2
        buf = self.event_buffer[symbol]
        while buf and buf[0].timestamp.timestamp() < cutoff:
            buf.popleft()

    def _in_cooldown(self, rule: EventRule, current_time: datetime) -> bool:
        """Check if rule is in cooldown period."""
        if rule.cooldown_seconds <= 0:
            return False
        last = self._last_signal_time.get(rule.name)
        if last is None:
            return False
        elapsed = (current_time - last).total_seconds()
        return elapsed < rule.cooldown_seconds
```

### Pattern Matching Examples

```python
# Volume spike: 5+ high-volume events in 30 seconds
volume_spike_rule = EventRule(
    name="volume_spike",
    window_seconds=30.0,
    min_events=5,
    condition=lambda events: all(e.volume > 10000 for e in events[-5:]),
    cooldown_seconds=60.0,
)

# Price momentum: price moved 1%+ in 60 seconds
def price_momentum(events: list[MarketEvent]) -> bool:
    if len(events) < 2:
        return False
    first_price = events[0].price
    last_price = events[-1].price
    pct_change = abs(last_price - first_price) / first_price
    return pct_change >= 0.01

momentum_rule = EventRule(
    name="price_momentum_1pct",
    window_seconds=60.0,
    min_events=2,
    condition=price_momentum,
    cooldown_seconds=120.0,
)

# Rapid trade burst: 20+ trades in 10 seconds (unusual activity)
trade_burst_rule = EventRule(
    name="trade_burst",
    window_seconds=10.0,
    min_events=20,
    condition=lambda events: len([e for e in events if e.event_type == "trade"]) >= 20,
    cooldown_seconds=30.0,
)

# Initialize the engine with all rules
engine = CEPEngine(
    rules=[volume_spike_rule, momentum_rule, trade_burst_rule]
)
```

---

## Broker Adapter Protocol

```python
from typing import Protocol, Callable, runtime_checkable
from dataclasses import dataclass
from decimal import Decimal
from enum import Enum


class OrderSide(Enum):
    BUY = "buy"
    SELL = "sell"


class OrderType(Enum):
    MARKET = "market"
    LIMIT = "limit"
    STOP = "stop"
    STOP_LIMIT = "stop_limit"


@dataclass
class Order:
    symbol: str
    side: OrderSide
    qty: Decimal
    order_type: OrderType
    limit_price: Decimal | None = None
    stop_price: Decimal | None = None
    client_order_id: str | None = None


@dataclass
class OrderResult:
    order_id: str
    client_order_id: str
    status: str
    filled_qty: Decimal
    filled_avg_price: Decimal | None


@dataclass
class CancelResult:
    order_id: str
    status: str
    message: str


@dataclass
class Position:
    symbol: str
    qty: Decimal
    avg_entry_price: Decimal
    market_value: Decimal
    unrealized_pl: Decimal


@dataclass
class AccountInfo:
    equity: Decimal
    buying_power: Decimal
    cash: Decimal
    pattern_day_trader: bool
    day_trade_count: int


@runtime_checkable
class BrokerAdapter(Protocol):
    """Protocol that all broker adapters must implement."""
    async def submit_order(self, order: Order) -> OrderResult: ...
    async def cancel_order(self, order_id: str) -> CancelResult: ...
    async def get_positions(self) -> list[Position]: ...
    async def get_account(self) -> AccountInfo: ...
    async def subscribe_fills(self, callback: Callable) -> None: ...


class BrokerNotConfigured(Exception):
    pass


class BrokerAdapterRegistry:
    """Central registry for broker adapters.

    Broker selection becomes a configuration decision -- strategy code
    never references a specific broker implementation.
    """

    def __init__(self):
        self._adapters: dict[str, BrokerAdapter] = {}

    def register(self, name: str, adapter: BrokerAdapter) -> None:
        if name in self._adapters:
            raise ValueError(f"Adapter already registered for broker: {name}")
        self._adapters[name] = adapter

    def get(self, name: str) -> BrokerAdapter:
        if name not in self._adapters:
            raise BrokerNotConfigured(
                f"No adapter registered for broker: {name}. "
                f"Available: {list(self._adapters.keys())}"
            )
        return self._adapters[name]

    def list_brokers(self) -> list[str]:
        return list(self._adapters.keys())
```

---

## Example: Alpaca Adapter

```python
import httpx
from decimal import Decimal


class AlpacaAdapter:
    """Alpaca broker adapter implementing BrokerAdapter protocol."""

    def __init__(self, api_key: str, api_secret: str, base_url: str):
        self.base_url = base_url
        self.headers = {
            "APCA-API-KEY-ID": api_key,
            "APCA-API-SECRET-KEY": api_secret,
        }
        self._client = httpx.AsyncClient(
            base_url=base_url,
            headers=self.headers,
            timeout=10.0,
        )
        self._fill_callback = None

    async def submit_order(self, order: Order) -> OrderResult:
        payload = {
            "symbol": order.symbol,
            "qty": str(order.qty),
            "side": order.side.value,
            "type": order.order_type.value,
            "time_in_force": "day",
        }
        if order.limit_price is not None:
            payload["limit_price"] = str(order.limit_price)
        if order.stop_price is not None:
            payload["stop_price"] = str(order.stop_price)
        if order.client_order_id is not None:
            payload["client_order_id"] = order.client_order_id

        resp = await self._client.post("/v2/orders", json=payload)
        resp.raise_for_status()
        data = resp.json()

        return OrderResult(
            order_id=data["id"],
            client_order_id=data.get("client_order_id", ""),
            status=data["status"],
            filled_qty=Decimal(data.get("filled_qty", "0")),
            filled_avg_price=(
                Decimal(data["filled_avg_price"])
                if data.get("filled_avg_price")
                else None
            ),
        )

    async def cancel_order(self, order_id: str) -> CancelResult:
        resp = await self._client.delete(f"/v2/orders/{order_id}")
        if resp.status_code == 204:
            return CancelResult(order_id=order_id, status="cancelled", message="OK")
        resp.raise_for_status()
        return CancelResult(
            order_id=order_id,
            status="error",
            message=resp.text,
        )

    async def get_positions(self) -> list[Position]:
        resp = await self._client.get("/v2/positions")
        resp.raise_for_status()
        return [
            Position(
                symbol=p["symbol"],
                qty=Decimal(p["qty"]),
                avg_entry_price=Decimal(p["avg_entry_price"]),
                market_value=Decimal(p["market_value"]),
                unrealized_pl=Decimal(p["unrealized_pl"]),
            )
            for p in resp.json()
        ]

    async def get_account(self) -> AccountInfo:
        resp = await self._client.get("/v2/account")
        resp.raise_for_status()
        data = resp.json()
        return AccountInfo(
            equity=Decimal(data["equity"]),
            buying_power=Decimal(data["buying_power"]),
            cash=Decimal(data["cash"]),
            pattern_day_trader=data.get("pattern_day_trader", False),
            day_trade_count=int(data.get("daytrade_count", 0)),
        )

    async def subscribe_fills(self, callback) -> None:
        self._fill_callback = callback
        # In production, this would connect to Alpaca's streaming API
        # and invoke callback on each fill event.
```

---

## Example: IBKR Adapter

```python
from ib_insync import IB, MarketOrder, LimitOrder, StopOrder, Trade


class IBKRAdapter:
    """Interactive Brokers adapter implementing BrokerAdapter protocol.

    Uses ib_insync for TWS/Gateway connectivity.
    """

    def __init__(self, host: str = "127.0.0.1", port: int = 7497, client_id: int = 1):
        self.ib = IB()
        self._host = host
        self._port = port
        self._client_id = client_id
        self._fill_callback = None

    async def connect(self) -> None:
        """Must be called before any other method."""
        await self.ib.connectAsync(self._host, self._port, clientId=self._client_id)

    async def submit_order(self, order: Order) -> OrderResult:
        contract = Stock(order.symbol, "SMART", "USD")
        await self.ib.qualifyContractsAsync(contract)

        if order.order_type == OrderType.MARKET:
            ib_order = MarketOrder(
                order.side.value.upper(),
                float(order.qty),
            )
        elif order.order_type == OrderType.LIMIT:
            ib_order = LimitOrder(
                order.side.value.upper(),
                float(order.qty),
                float(order.limit_price),
            )
        elif order.order_type == OrderType.STOP:
            ib_order = StopOrder(
                order.side.value.upper(),
                float(order.qty),
                float(order.stop_price),
            )
        else:
            raise ValueError(f"Unsupported order type: {order.order_type}")

        trade: Trade = self.ib.placeOrder(contract, ib_order)
        # Wait briefly for order acknowledgment
        await self.ib.sleep(0.5)

        return OrderResult(
            order_id=str(trade.order.orderId),
            client_order_id=order.client_order_id or str(trade.order.orderId),
            status=trade.orderStatus.status,
            filled_qty=Decimal(str(trade.orderStatus.filled)),
            filled_avg_price=(
                Decimal(str(trade.orderStatus.avgFillPrice))
                if trade.orderStatus.avgFillPrice
                else None
            ),
        )

    async def cancel_order(self, order_id: str) -> CancelResult:
        for trade in self.ib.openTrades():
            if str(trade.order.orderId) == order_id:
                self.ib.cancelOrder(trade.order)
                await self.ib.sleep(0.5)
                return CancelResult(
                    order_id=order_id,
                    status="cancelled",
                    message="OK",
                )
        return CancelResult(
            order_id=order_id,
            status="not_found",
            message=f"Order {order_id} not found in open trades",
        )

    async def get_positions(self) -> list[Position]:
        ib_positions = self.ib.positions()
        return [
            Position(
                symbol=pos.contract.symbol,
                qty=Decimal(str(pos.position)),
                avg_entry_price=Decimal(str(pos.avgCost)),
                market_value=Decimal(str(pos.position * pos.avgCost)),
                unrealized_pl=Decimal(str(pos.unrealizedPNL or 0)),
            )
            for pos in ib_positions
        ]

    async def get_account(self) -> AccountInfo:
        account_values = self.ib.accountSummary()
        values = {v.tag: v.value for v in account_values}
        return AccountInfo(
            equity=Decimal(values.get("NetLiquidation", "0")),
            buying_power=Decimal(values.get("BuyingPower", "0")),
            cash=Decimal(values.get("TotalCashValue", "0")),
            pattern_day_trader=False,  # IBKR handles PDT internally
            day_trade_count=0,
        )

    async def subscribe_fills(self, callback) -> None:
        self._fill_callback = callback
        self.ib.orderStatusEvent += self._on_order_status

    def _on_order_status(self, trade: Trade) -> None:
        if self._fill_callback and trade.orderStatus.status == "Filled":
            self._fill_callback(trade)
```

---

## Registry Initialization Pattern

```python
import os
import logging

logger = logging.getLogger(__name__)


def initialize_broker_registry() -> BrokerAdapterRegistry:
    """Initialize the broker registry from environment configuration.

    The active broker is selected via BROKER_NAME env var.
    Each broker's credentials come from broker-specific env vars.
    """
    registry = BrokerAdapterRegistry()

    # Register Alpaca if configured
    alpaca_key = os.getenv("ALPACA_API_KEY")
    alpaca_secret = os.getenv("ALPACA_API_SECRET")
    alpaca_url = os.getenv("ALPACA_BASE_URL", "https://paper-api.alpaca.markets")
    if alpaca_key and alpaca_secret:
        registry.register("alpaca", AlpacaAdapter(
            api_key=alpaca_key,
            api_secret=alpaca_secret,
            base_url=alpaca_url,
        ))
        logger.info("Registered Alpaca adapter")

    # Register IBKR if configured
    ibkr_host = os.getenv("IBKR_HOST", "127.0.0.1")
    ibkr_port = os.getenv("IBKR_PORT")
    if ibkr_port:
        registry.register("ibkr", IBKRAdapter(
            host=ibkr_host,
            port=int(ibkr_port),
            client_id=int(os.getenv("IBKR_CLIENT_ID", "1")),
        ))
        logger.info("Registered IBKR adapter")

    return registry


# Usage in bot startup
async def start_bot():
    registry = initialize_broker_registry()
    broker_name = os.getenv("BROKER_NAME", "alpaca")

    # This is the only place broker selection happens
    broker = registry.get(broker_name)
    logger.info(f"Using broker: {broker_name}")

    # Pass the adapter to the execution gateway
    gateway = ExecutionGateway(broker=broker)
    await gateway.start()
```

This pattern ensures:
1. Strategy code never knows which broker is active
2. Adding a new broker means writing one adapter class and registering it
3. Broker selection is a configuration decision, not a code change
4. Testing uses a mock adapter registered under the same interface
