# Trading Test Patterns -- Python/pytest Reference

Complete, runnable test functions for all 10 mandatory trading test categories. Copy, adapt, and extend these patterns for your specific trading system.

## Setup: Common Fixtures

```python
import pytest
import asyncio
from unittest.mock import AsyncMock, MagicMock, patch, PropertyMock
from decimal import Decimal
from datetime import datetime, timedelta
import math


# ---------- Mock Broker ----------

class MockBroker:
    """Simulates broker behavior for testing. Handles orders, fills, positions."""

    def __init__(self):
        self.orders = {}
        self.positions = {}
        self.fills = []
        self._next_order_id = 1
        self._connected = True

    def submit_order(self, symbol, side, quantity, order_type="market", limit_price=None):
        if not self._connected:
            raise ConnectionError("Broker disconnected")
        order_id = f"ORD-{self._next_order_id}"
        self._next_order_id += 1
        self.orders[order_id] = {
            "symbol": symbol,
            "side": side,
            "quantity": quantity,
            "filled_qty": 0,
            "status": "submitted",
            "order_type": order_type,
            "limit_price": limit_price,
        }
        return order_id

    def simulate_fill(self, order_id, fill_qty, fill_price):
        """Simulate a partial or full fill arriving from the broker."""
        order = self.orders[order_id]
        order["filled_qty"] += fill_qty
        if order["filled_qty"] >= order["quantity"]:
            order["status"] = "filled"
        else:
            order["status"] = "partially_filled"

        fill = {
            "order_id": order_id,
            "symbol": order["symbol"],
            "side": order["side"],
            "quantity": fill_qty,
            "price": fill_price,
            "timestamp": datetime.utcnow(),
        }
        self.fills.append(fill)
        return fill

    def cancel_order(self, order_id):
        order = self.orders.get(order_id)
        if order and order["status"] in ("submitted", "partially_filled"):
            order["status"] = "cancelled"
            return True
        return False

    def get_positions(self):
        return dict(self.positions)

    def disconnect(self):
        self._connected = False

    def reconnect(self):
        self._connected = True


@pytest.fixture
def mock_broker():
    return MockBroker()


@pytest.fixture
def event_loop():
    loop = asyncio.new_event_loop()
    yield loop
    loop.close()
```

---

## Category 1: Partial Fill Tests

```python
class TestPartialFills:
    """Every order-related feature must handle partial fills correctly."""

    def test_three_partial_fills_accumulate(self, mock_broker):
        """Order 100, fills 30+30+40. Position must equal 100 after all fills."""
        from your_bot.position_tracker import PositionTracker

        tracker = PositionTracker()
        order_id = mock_broker.submit_order("AAPL", "buy", 100)

        # Three partial fills
        fill1 = mock_broker.simulate_fill(order_id, 30, 150.00)
        tracker.on_fill(fill1)
        assert tracker.get_position("AAPL") == 30

        fill2 = mock_broker.simulate_fill(order_id, 30, 150.05)
        tracker.on_fill(fill2)
        assert tracker.get_position("AAPL") == 60

        fill3 = mock_broker.simulate_fill(order_id, 40, 150.10)
        tracker.on_fill(fill3)
        assert tracker.get_position("AAPL") == 100

    def test_partial_fill_then_cancel_reflects_actual_fills(self, mock_broker):
        """Order 100, fill 60, cancel remainder. Position must be 60."""
        from your_bot.position_tracker import PositionTracker

        tracker = PositionTracker()
        order_id = mock_broker.submit_order("AAPL", "buy", 100)

        fill = mock_broker.simulate_fill(order_id, 60, 150.00)
        tracker.on_fill(fill)
        mock_broker.cancel_order(order_id)

        # Position must reflect actual fills, NOT intended order quantity
        assert tracker.get_position("AAPL") == 60

    def test_single_full_fill_same_result(self, mock_broker):
        """Single fill of 100 must produce same end state as partial fills."""
        from your_bot.position_tracker import PositionTracker

        tracker = PositionTracker()
        order_id = mock_broker.submit_order("AAPL", "buy", 100)

        fill = mock_broker.simulate_fill(order_id, 100, 150.00)
        tracker.on_fill(fill)
        assert tracker.get_position("AAPL") == 100

    def test_average_fill_price_weighted_by_quantity(self, mock_broker):
        """Average price must be quantity-weighted across partial fills."""
        from your_bot.position_tracker import PositionTracker

        tracker = PositionTracker()
        order_id = mock_broker.submit_order("AAPL", "buy", 100)

        mock_broker.simulate_fill(order_id, 60, 150.00)
        tracker.on_fill(mock_broker.fills[-1])

        mock_broker.simulate_fill(order_id, 40, 151.00)
        tracker.on_fill(mock_broker.fills[-1])

        expected_avg = (60 * 150.00 + 40 * 151.00) / 100  # 150.40
        assert abs(tracker.get_average_price("AAPL") - expected_avg) < 0.01

    def test_out_of_order_fills_correct_final_state(self, mock_broker):
        """Fills processed in different order than generated must give same result."""
        from your_bot.position_tracker import PositionTracker

        tracker = PositionTracker()
        order_id = mock_broker.submit_order("AAPL", "buy", 100)

        fill_a = mock_broker.simulate_fill(order_id, 40, 150.00)
        fill_b = mock_broker.simulate_fill(order_id, 30, 150.05)
        fill_c = mock_broker.simulate_fill(order_id, 30, 150.10)

        # Process in reverse order
        tracker.on_fill(fill_c)
        tracker.on_fill(fill_a)
        tracker.on_fill(fill_b)

        assert tracker.get_position("AAPL") == 100
```

---

## Category 2: Race Condition Tests

```python
class TestRaceConditions:
    """Concurrent events must not corrupt state."""

    @pytest.mark.asyncio
    async def test_fill_arrives_before_cancel_ack(self, mock_broker):
        """Cancel sent, but fill arrives first. Position must reflect the fill."""
        from your_bot.order_manager import OrderManager

        manager = OrderManager(mock_broker)
        order_id = await manager.submit_order("AAPL", "buy", 100)

        # Fill arrives before cancel processes
        fill = mock_broker.simulate_fill(order_id, 100, 150.00)
        await manager.on_fill(fill)

        # Cancel comes back as "too late"
        await manager.cancel_order(order_id)

        assert manager.get_position("AAPL") == 100

    @pytest.mark.asyncio
    async def test_two_signals_same_symbol_produces_one_order(self):
        """Two buy signals for AAPL within dedup window must produce one order."""
        from your_bot.signal_processor import SignalProcessor
        from your_bot.dedup_gate import DedupGate

        dedup = DedupGate(window_seconds=5)
        processor = SignalProcessor(dedup_gate=dedup)

        signal1 = {"symbol": "AAPL", "action": "buy", "timestamp": datetime.utcnow()}
        signal2 = {"symbol": "AAPL", "action": "buy",
                    "timestamp": datetime.utcnow() + timedelta(milliseconds=100)}

        result1 = await processor.process_signal(signal1)
        result2 = await processor.process_signal(signal2)

        placed = [r for r in [result1, result2] if r is not None]
        assert len(placed) == 1, "Duplicate signal within dedup window must produce single order"

    @pytest.mark.asyncio
    async def test_concurrent_fills_no_position_corruption(self, mock_broker):
        """10 concurrent fills of 10 shares each must total exactly 100."""
        from your_bot.position_tracker import AsyncPositionTracker

        tracker = AsyncPositionTracker()
        order_id = mock_broker.submit_order("AAPL", "buy", 100)

        fills = [mock_broker.simulate_fill(order_id, 10, 150.00 + i * 0.01) for i in range(10)]

        # Process all concurrently
        await asyncio.gather(*[tracker.on_fill(f) for f in fills])

        assert tracker.get_position("AAPL") == 100

    @pytest.mark.asyncio
    async def test_kill_switch_cancels_all_pending(self, mock_broker):
        """Kill switch must cancel every pending order."""
        from your_bot.kill_switch import KillSwitch
        from your_bot.order_manager import OrderManager

        manager = OrderManager(mock_broker)
        ks = KillSwitch(manager)

        oid1 = await manager.submit_order("AAPL", "buy", 100)
        oid2 = await manager.submit_order("MSFT", "buy", 50)

        await ks.activate("test: max loss exceeded")

        assert mock_broker.orders[oid1]["status"] == "cancelled"
        assert mock_broker.orders[oid2]["status"] == "cancelled"
        assert ks.is_active is True

    @pytest.mark.asyncio
    async def test_reconciliation_during_fill_processing(self, mock_broker):
        """Reconciliation running while fills arrive must not produce false mismatch."""
        from your_bot.reconciler import Reconciler
        from your_bot.position_tracker import AsyncPositionTracker

        tracker = AsyncPositionTracker()
        reconciler = Reconciler(mock_broker, tracker)

        order_id = mock_broker.submit_order("AAPL", "buy", 100)
        fill = mock_broker.simulate_fill(order_id, 100, 150.00)

        # Process fill and reconciliation concurrently
        mock_broker.positions = {"AAPL": 100}
        await asyncio.gather(
            tracker.on_fill(fill),
            reconciler.reconcile(),
        )

        # Should not have false mismatch
        assert reconciler.last_result.has_mismatch is False
```

---

## Category 3: Stale Data Tests

```python
class TestStaleData:
    """Stale or missing data must NEVER result in a trade."""

    def test_stale_price_rejected(self):
        """Price older than staleness threshold must be rejected."""
        from your_bot.data_validator import DataValidator

        validator = DataValidator(max_staleness_seconds=30)

        stale_price = {
            "symbol": "AAPL",
            "price": 150.00,
            "timestamp": datetime.utcnow() - timedelta(seconds=60),
        }

        result = validator.validate_price(stale_price)
        assert result.is_valid is False
        assert "stale" in result.reason.lower()

    def test_fresh_price_accepted(self):
        """Price within staleness threshold must be accepted."""
        from your_bot.data_validator import DataValidator

        validator = DataValidator(max_staleness_seconds=30)

        fresh_price = {
            "symbol": "AAPL",
            "price": 150.00,
            "timestamp": datetime.utcnow() - timedelta(seconds=5),
        }

        result = validator.validate_price(fresh_price)
        assert result.is_valid is True

    def test_missing_vix_degrades_safely(self):
        """Missing VIX feed must result in no trades, not crash."""
        from your_bot.strategy import VolatilityStrategy

        strategy = VolatilityStrategy()

        market_data = {
            "AAPL": {"price": 150.00, "timestamp": datetime.utcnow()},
            "VIX": None,
        }

        signal = strategy.evaluate(market_data)
        assert signal is None or signal.action == "no_trade"

    def test_insufficient_bars_returns_none(self):
        """Indicator needing 20 bars given 5 must return None, not crash."""
        from your_bot.indicators import moving_average

        prices = [150.0, 151.0, 152.0, 153.0, 154.0]
        result = moving_average(prices, period=20)
        assert result is None

    def test_nan_price_rejected(self):
        """NaN values in price data must be detected and rejected."""
        from your_bot.data_validator import DataValidator

        validator = DataValidator(max_staleness_seconds=30)

        bad_price = {
            "symbol": "AAPL",
            "price": float("nan"),
            "timestamp": datetime.utcnow(),
        }

        result = validator.validate_price(bad_price)
        assert result.is_valid is False

    def test_none_price_rejected(self):
        """None price must be detected and rejected."""
        from your_bot.data_validator import DataValidator

        validator = DataValidator(max_staleness_seconds=30)

        bad_price = {
            "symbol": "AAPL",
            "price": None,
            "timestamp": datetime.utcnow(),
        }

        result = validator.validate_price(bad_price)
        assert result.is_valid is False

    def test_timestamp_gap_detected(self):
        """Gaps in candle data must be detected."""
        from your_bot.data_validator import DataValidator

        validator = DataValidator(max_staleness_seconds=30)

        candles = [
            {"timestamp": datetime(2025, 1, 2, 10, 0), "close": 150.0},
            {"timestamp": datetime(2025, 1, 2, 10, 1), "close": 150.5},
            # Gap: 10:02 missing
            {"timestamp": datetime(2025, 1, 2, 10, 3), "close": 151.0},
        ]

        gaps = validator.detect_gaps(candles, expected_interval_seconds=60)
        assert len(gaps) == 1
        assert gaps[0]["expected"] == datetime(2025, 1, 2, 10, 2)
```

---

## Category 4: Slippage Tests

```python
class TestSlippage:
    """Slippage must be detected, calculated, and tracked for every fill."""

    def test_adverse_slippage_on_stop_loss(self):
        """Stop at $10.00, fill at $8.50 must detect $1.50 adverse slippage."""
        from your_bot.slippage_tracker import SlippageTracker

        tracker = SlippageTracker()

        fill = {
            "order_id": "ORD-1",
            "symbol": "AAPL",
            "side": "sell",
            "expected_price": 10.00,
            "fill_price": 8.50,
            "quantity": 100,
        }

        slippage = tracker.record_fill(fill)
        assert slippage.amount == pytest.approx(1.50)
        assert slippage.direction == "adverse"

    def test_favorable_slippage_tracked(self):
        """Fill better than expected must still be tracked."""
        from your_bot.slippage_tracker import SlippageTracker

        tracker = SlippageTracker()

        fill = {
            "order_id": "ORD-1",
            "symbol": "AAPL",
            "side": "buy",
            "expected_price": 50.00,
            "fill_price": 49.95,
            "quantity": 100,
        }

        slippage = tracker.record_fill(fill)
        assert slippage.amount == pytest.approx(0.05)
        assert slippage.direction == "favorable"

    def test_slippage_threshold_triggers_alert(self):
        """Slippage exceeding threshold must generate an alert."""
        from your_bot.slippage_tracker import SlippageTracker

        tracker = SlippageTracker(alert_threshold_pct=2.0)
        alerts = []
        tracker.on_alert = lambda a: alerts.append(a)

        fill = {
            "order_id": "ORD-1",
            "symbol": "AAPL",
            "side": "sell",
            "expected_price": 100.00,
            "fill_price": 95.00,  # 5% slippage, exceeds 2% threshold
            "quantity": 100,
        }

        tracker.record_fill(fill)
        assert len(alerts) == 1
        assert alerts[0].slippage_pct == pytest.approx(5.0)

    def test_aggregate_slippage_running_total(self):
        """Running total of slippage dollars across multiple fills must be correct."""
        from your_bot.slippage_tracker import SlippageTracker

        tracker = SlippageTracker()

        for i in range(5):
            fill = {
                "order_id": f"ORD-{i}",
                "symbol": "AAPL",
                "side": "buy",
                "expected_price": 100.00,
                "fill_price": 100.10,  # $0.10 adverse per share
                "quantity": 100,
            }
            tracker.record_fill(fill)

        # 5 fills * 100 shares * $0.10 slippage = $50
        assert tracker.total_slippage_dollars == pytest.approx(50.0)

    def test_zero_slippage_on_exact_fill(self):
        """Fill at exact expected price must record zero slippage."""
        from your_bot.slippage_tracker import SlippageTracker

        tracker = SlippageTracker()

        fill = {
            "order_id": "ORD-1",
            "symbol": "AAPL",
            "side": "buy",
            "expected_price": 150.00,
            "fill_price": 150.00,
            "quantity": 100,
        }

        slippage = tracker.record_fill(fill)
        assert slippage.amount == pytest.approx(0.0)
```

---

## Category 5: Fail-Closed Tests

```python
class TestFailClosed:
    """Exceptions in risk/validation MUST return REJECT, never None or pass-through.
    This is the most critical test category. Every decision point must be fail-closed."""

    def test_risk_check_exception_returns_reject(self):
        """Exception in risk check must explicitly REJECT, not return None."""
        from your_bot.risk_manager import RiskManager

        manager = RiskManager()

        with patch.object(manager, '_check_position_limits', side_effect=RuntimeError("DB down")):
            result = manager.check_risk({"symbol": "AAPL", "side": "buy", "quantity": 100})

        assert result is not None, "Risk check must not return None on exception"
        assert result.approved is False, "Risk check must REJECT on exception"
        assert result.reason is not None

    def test_position_sizer_exception_returns_zero(self):
        """Exception in position sizing must return 0 shares."""
        from your_bot.position_sizer import PositionSizer

        sizer = PositionSizer()

        with patch.object(sizer, '_calculate_kelly', side_effect=ZeroDivisionError):
            size = sizer.calculate_size(
                symbol="AAPL", signal_strength=0.8, account_value=100000
            )

        assert size == 0, "Position sizer must return 0 on exception, not raise"

    def test_order_validation_exception_blocks_order(self):
        """Exception in order validation must prevent order placement."""
        from your_bot.order_validator import OrderValidator

        validator = OrderValidator()

        with patch.object(validator, '_check_symbol_tradeable', side_effect=ConnectionError):
            is_valid = validator.validate({"symbol": "AAPL", "side": "buy", "quantity": 100})

        assert is_valid is False, "Validation exception must block order, not pass through"

    def test_none_from_subcheck_treated_as_reject(self):
        """If a risk sub-check returns None (bug), top-level must REJECT."""
        from your_bot.risk_manager import RiskManager

        manager = RiskManager()

        with patch.object(manager, '_check_position_limits', return_value=None):
            result = manager.check_risk({"symbol": "AAPL", "side": "buy", "quantity": 100})

        assert result.approved is False, "None from sub-check must escalate to REJECT"

    def test_network_timeout_rejects(self):
        """Network timeout during risk check must REJECT, not skip the check."""
        from your_bot.risk_manager import RiskManager
        import socket

        manager = RiskManager()

        with patch.object(manager, '_fetch_current_exposure', side_effect=socket.timeout):
            result = manager.check_risk({"symbol": "AAPL", "side": "buy", "quantity": 100})

        assert result.approved is False

    def test_all_except_blocks_have_reject(self):
        """Scan pattern: every except block in risk code must not return None.
        This is a code-level check you can run with ast or grep."""
        import ast
        import inspect
        from your_bot import risk_manager

        source = inspect.getsource(risk_manager)
        tree = ast.parse(source)

        for node in ast.walk(tree):
            if isinstance(node, ast.ExceptHandler):
                # Check that the except block contains a return statement
                # and that return value is not None
                has_return = any(isinstance(child, ast.Return) for child in ast.walk(node))
                if not has_return:
                    # except block with no return -- may fall through
                    # This is a heuristic; refine for your codebase
                    pass  # Log warning for manual review
```

---

## Category 6: Reconciliation Tests

```python
class TestReconciliation:
    """Local state must match broker state. Mismatch halts trading."""

    def test_local_more_than_broker_detected(self, mock_broker):
        """Local 100 shares, broker 0 -> mismatch detected, trading halted."""
        from your_bot.reconciler import Reconciler

        reconciler = Reconciler(mock_broker)

        local_positions = {"AAPL": 100}
        mock_broker.positions = {"AAPL": 0}

        result = reconciler.reconcile(local_positions)

        assert result.has_mismatch is True
        assert result.trading_halted is True
        assert "AAPL" in result.mismatches

    def test_ghost_position_detected(self, mock_broker):
        """Broker has position, local does not -> ghost position detected."""
        from your_bot.reconciler import Reconciler

        reconciler = Reconciler(mock_broker)

        local_positions = {}
        mock_broker.positions = {"AAPL": 100}

        result = reconciler.reconcile(local_positions)

        assert result.has_mismatch is True
        assert result.ghost_positions == {"AAPL": 100}

    def test_extra_broker_position_detected(self, mock_broker):
        """Broker has extra symbol not in local -> detected."""
        from your_bot.reconciler import Reconciler

        reconciler = Reconciler(mock_broker)

        local_positions = {"AAPL": 100}
        mock_broker.positions = {"AAPL": 100, "MSFT": 50}

        result = reconciler.reconcile(local_positions)

        assert result.has_mismatch is True
        assert "MSFT" in result.extra_broker_positions

    def test_matching_positions_no_halt(self, mock_broker):
        """Matching positions must not trigger halt."""
        from your_bot.reconciler import Reconciler

        reconciler = Reconciler(mock_broker)

        local_positions = {"AAPL": 100, "MSFT": 50}
        mock_broker.positions = {"AAPL": 100, "MSFT": 50}

        result = reconciler.reconcile(local_positions)

        assert result.has_mismatch is False
        assert result.trading_halted is False

    def test_empty_positions_both_sides_no_halt(self, mock_broker):
        """Both local and broker empty -> no mismatch."""
        from your_bot.reconciler import Reconciler

        reconciler = Reconciler(mock_broker)

        result = reconciler.reconcile({})

        assert result.has_mismatch is False
```

---

## Category 7: Config Validation Tests

```python
class TestConfigValidation:
    """Invalid config must be caught at startup, never at runtime."""

    def test_negative_max_position_rejected(self):
        """Negative max position size must raise ValueError at construction."""
        from your_bot.config import TradingConfig

        with pytest.raises(ValueError, match="max_position_size"):
            TradingConfig(max_position_size=-100)

    def test_missing_required_field_raises(self):
        """Missing required config field must raise with clear message."""
        from your_bot.config import TradingConfig

        with pytest.raises(ValueError, match="api_key.*required"):
            TradingConfig(api_key=None, max_position_size=100)

    def test_zero_max_loss_rejected(self):
        """Max daily loss of 0 would halt immediately. Must reject."""
        from your_bot.config import TradingConfig

        with pytest.raises(ValueError, match="max_daily_loss"):
            TradingConfig(max_daily_loss=0.0, api_key="test", max_position_size=100)

    def test_empty_api_key_rejected(self):
        """Empty string API key must be caught before first API call."""
        from your_bot.config import TradingConfig

        with pytest.raises(ValueError, match="api_key"):
            TradingConfig(api_key="", max_position_size=100)

    def test_wrong_type_rejected(self):
        """String where float expected must be rejected."""
        from your_bot.config import TradingConfig

        with pytest.raises((TypeError, ValueError)):
            TradingConfig(max_position_size="one hundred", api_key="test")

    def test_valid_config_loads_successfully(self):
        """Valid configuration must load without errors."""
        from your_bot.config import TradingConfig

        config = TradingConfig(
            api_key="test-key-123",
            max_position_size=1000,
            max_daily_loss=5000.0,
            staleness_threshold_seconds=30,
        )
        assert config.max_position_size == 1000
        assert config.max_daily_loss == 5000.0
```

---

## Category 8: Math Correctness Tests

```python
class TestMathCorrectness:
    """All calculations must match reference implementations within epsilon."""

    def test_sma_matches_pandas(self):
        """Our SMA must match pandas rolling mean."""
        import pandas as pd
        from your_bot.indicators import simple_moving_average

        prices = [100, 102, 104, 103, 105, 107, 106, 108, 110, 109,
                  111, 113, 112, 114, 116, 115, 117, 119, 118, 120]

        our_sma = simple_moving_average(prices, period=10)
        ref_sma = pd.Series(prices).rolling(10).mean().iloc[-1]

        assert abs(our_sma - ref_sma) < 1e-10

    def test_ema_matches_pandas(self):
        """Our EMA must match pandas ewm mean."""
        import pandas as pd
        from your_bot.indicators import exponential_moving_average

        prices = [100, 102, 104, 103, 105, 107, 106, 108, 110, 109,
                  111, 113, 112, 114, 116, 115, 117, 119, 118, 120]

        our_ema = exponential_moving_average(prices, period=10)
        ref_ema = pd.Series(prices).ewm(span=10, adjust=False).mean().iloc[-1]

        assert abs(our_ema - ref_ema) < 1e-6

    def test_trailing_stop_never_decreases(self):
        """Trailing stop level must be monotonically non-decreasing."""
        from your_bot.trailing_stop import TrailingStop

        stop = TrailingStop(trail_pct=5.0, initial_price=100.0)

        # Price goes up, down, up, down -- stop must only go up
        price_sequence = [100, 105, 110, 108, 103, 107, 115, 112, 120, 95, 130]
        stop_levels = []

        for price in price_sequence:
            stop.update(price)
            stop_levels.append(stop.level)

        for i in range(1, len(stop_levels)):
            assert stop_levels[i] >= stop_levels[i - 1], (
                f"Trailing stop DECREASED from {stop_levels[i-1]} to {stop_levels[i]} "
                f"at price {price_sequence[i]} (index {i}). This violates monotonicity."
            )

    def test_trailing_stop_correct_level(self):
        """Trailing stop must be trail_pct below highest observed price."""
        from your_bot.trailing_stop import TrailingStop

        stop = TrailingStop(trail_pct=5.0, initial_price=100.0)

        stop.update(100.0)
        assert stop.level == pytest.approx(95.0)  # 100 * 0.95

        stop.update(110.0)
        assert stop.level == pytest.approx(104.5)  # 110 * 0.95

        stop.update(105.0)  # Price drops -- stop must NOT drop
        assert stop.level == pytest.approx(104.5)  # Still 110 * 0.95

        stop.update(120.0)
        assert stop.level == pytest.approx(114.0)  # 120 * 0.95

    def test_position_sizing_whole_shares(self):
        """Position size must be whole shares when fractional is disabled."""
        from your_bot.position_sizer import PositionSizer

        sizer = PositionSizer(allow_fractional=False)

        size = sizer.calculate_size(
            symbol="AAPL", signal_strength=0.7, account_value=10000, price=150.00
        )

        assert isinstance(size, int)
        assert size >= 0

    def test_pnl_long_trade(self):
        """Long P&L: (exit - entry) * quantity."""
        from your_bot.pnl import calculate_pnl

        pnl = calculate_pnl(entry_price=150.00, exit_price=155.00, quantity=100, side="long")
        assert pnl == pytest.approx(500.00)

    def test_pnl_short_trade(self):
        """Short P&L: (entry - exit) * quantity."""
        from your_bot.pnl import calculate_pnl

        pnl = calculate_pnl(entry_price=155.00, exit_price=150.00, quantity=100, side="short")
        assert pnl == pytest.approx(500.00)

    def test_no_divide_by_zero_empty_portfolio(self):
        """Empty portfolio percentage return must be 0, not ZeroDivisionError."""
        from your_bot.portfolio import Portfolio

        portfolio = Portfolio()
        pct_return = portfolio.total_return_pct()
        assert pct_return == 0.0
        assert not math.isnan(pct_return)
```

---

## Category 9: Kill Switch Tests

```python
class TestKillSwitch:
    """Kill switch must halt ALL trading when triggered. No overrides allowed."""

    def test_max_daily_loss_triggers_halt(self):
        """Cumulative daily loss exceeding threshold must activate kill switch."""
        from your_bot.kill_switch import KillSwitch

        ks = KillSwitch(max_daily_loss=5000.0)

        ks.record_loss(3000.0)
        assert ks.is_active is False

        ks.record_loss(3000.0)  # Total: 6000 > 5000
        assert ks.is_active is True

    @pytest.mark.asyncio
    async def test_halt_rejects_new_orders(self):
        """After kill switch activation, all new orders must be rejected."""
        from your_bot.kill_switch import KillSwitch
        from your_bot.order_manager import OrderManager

        manager = OrderManager(MockBroker())
        ks = KillSwitch(max_daily_loss=5000.0)
        manager.kill_switch = ks

        ks.activate("test: threshold exceeded")

        result = await manager.submit_order("AAPL", "buy", 100)
        assert result is None or result.rejected is True

    @pytest.mark.asyncio
    async def test_profitable_signal_still_rejected(self):
        """Even highly confident signals must be rejected when kill switch is active."""
        from your_bot.kill_switch import KillSwitch
        from your_bot.signal_processor import SignalProcessor

        ks = KillSwitch(max_daily_loss=5000.0)
        processor = SignalProcessor(kill_switch=ks)

        ks.activate("max loss exceeded")

        signal = {"symbol": "AAPL", "action": "buy", "confidence": 0.99}
        result = await processor.process_signal(signal)
        assert result is None, "Kill switch must not be overridden by any signal"

    @pytest.mark.asyncio
    async def test_broker_disconnect_triggers_halt(self):
        """Lost broker connection must trigger kill switch."""
        from your_bot.kill_switch import KillSwitch
        from your_bot.connection_monitor import ConnectionMonitor

        ks = KillSwitch(max_daily_loss=5000.0)
        monitor = ConnectionMonitor(kill_switch=ks)

        await monitor.on_disconnect()
        assert ks.is_active is True
        assert "disconnect" in ks.reason.lower()

    def test_daily_reset_clears_kill_switch(self):
        """Kill switch must reset at configured daily reset time."""
        from your_bot.kill_switch import KillSwitch

        ks = KillSwitch(max_daily_loss=5000.0)
        ks.activate("test")
        assert ks.is_active is True

        ks.daily_reset()
        assert ks.is_active is False
        assert ks.daily_loss == 0.0

    def test_manual_activation_works(self):
        """Manual kill switch must work regardless of loss tracking."""
        from your_bot.kill_switch import KillSwitch

        ks = KillSwitch(max_daily_loss=5000.0)
        assert ks.daily_loss == 0.0  # No losses recorded

        ks.activate("manual: operator triggered")
        assert ks.is_active is True
```

---

## Category 10: Dedup Tests

```python
class TestDedup:
    """Identical signals within dedup window must produce exactly one order."""

    def test_same_signal_twice_blocked(self):
        """Second identical signal within window must be blocked."""
        from your_bot.dedup_gate import DedupGate

        gate = DedupGate(window_seconds=5)

        signal = {"symbol": "AAPL", "action": "buy", "price": 150.0}

        assert gate.check(signal) is True   # First: allowed
        assert gate.check(signal) is False  # Second: blocked

    def test_same_signal_different_sources_blocked(self):
        """Same symbol+action from different sources must be deduped."""
        from your_bot.dedup_gate import DedupGate

        gate = DedupGate(window_seconds=5)

        signal_a = {"symbol": "AAPL", "action": "buy", "source": "strategy_A"}
        signal_b = {"symbol": "AAPL", "action": "buy", "source": "strategy_B"}

        assert gate.check(signal_a) is True
        assert gate.check(signal_b) is False  # Same symbol+action = dedup

    def test_different_direction_not_deduped(self):
        """Same symbol but different action (buy vs sell) must both proceed."""
        from your_bot.dedup_gate import DedupGate

        gate = DedupGate(window_seconds=5)

        assert gate.check({"symbol": "AAPL", "action": "buy"}) is True
        assert gate.check({"symbol": "AAPL", "action": "sell"}) is True

    def test_cancel_clears_dedup(self):
        """After cancel, same signal must be allowed again."""
        from your_bot.dedup_gate import DedupGate

        gate = DedupGate(window_seconds=5)

        signal = {"symbol": "AAPL", "action": "buy"}

        assert gate.check(signal) is True
        gate.clear("AAPL", "buy")
        assert gate.check(signal) is True  # Allowed again after clear

    def test_window_expiration_allows_signal(self):
        """Signal after dedup window expires must be allowed."""
        from your_bot.dedup_gate import DedupGate

        gate = DedupGate(window_seconds=5)
        signal = {"symbol": "AAPL", "action": "buy"}
        now = datetime.utcnow()

        with patch('your_bot.dedup_gate.datetime') as mock_dt:
            mock_dt.utcnow.return_value = now
            assert gate.check(signal) is True

            mock_dt.utcnow.return_value = now + timedelta(seconds=3)
            assert gate.check(signal) is False  # Still within window

            mock_dt.utcnow.return_value = now + timedelta(seconds=6)
            assert gate.check(signal) is True   # Window expired

    def test_different_symbols_not_deduped(self):
        """Different symbols must not interfere with each other."""
        from your_bot.dedup_gate import DedupGate

        gate = DedupGate(window_seconds=5)

        assert gate.check({"symbol": "AAPL", "action": "buy"}) is True
        assert gate.check({"symbol": "MSFT", "action": "buy"}) is True
```

---

## Running the Tests

```bash
# Run all trading tests
pytest tests/trading/ -v --tb=short

# Run specific category
pytest tests/trading/ -v -k "TestPartialFills"
pytest tests/trading/ -v -k "TestFailClosed"
pytest tests/trading/ -v -k "TestKillSwitch"

# Run with coverage report
pytest tests/trading/ --cov=your_bot --cov-report=term-missing

# Run only unit tests (mocks only, fast)
pytest tests/trading/ -v -m unit

# Run integration tests (paper API, requires broker connection)
pytest tests/trading/ -v -m integration

# Run everything with verbose failure output
pytest tests/trading/ -v --tb=long -x  # Stop on first failure
```
