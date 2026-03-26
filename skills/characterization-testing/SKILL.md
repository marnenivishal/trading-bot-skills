---
name: characterization-testing
description: Use when refactoring legacy trading code, when existing code lacks tests and behavior is unknown, or when you need to lock current behavior before making changes
---

# Characterization Testing for Trading Systems

Iron Law: **"BEFORE REFACTORING LEGACY TRADING CODE, LOCK ITS CURRENT BEHAVIOR WITH CHARACTERIZATION TESTS. CHANGE NOTHING UNTIL THE HARNESS IS GREEN."**

A characterization test doesn't check whether code is *correct*. It checks what the code *actually does*. You capture existing behavior — warts and all — so that when you refactor, you know instantly if you broke something.

---

## 1. The Big-Bang Rewrite Disaster

**Case Study: The Gap Calculation Module**

A trading team had a messy `calculate_gap()` function — 200 lines, nested conditionals, magic numbers everywhere. Nobody understood why it worked, but it did. The team decided to rewrite it "from scratch" with clean code.

The new version looked beautiful. It passed code review. It shipped.

Two weeks later, someone noticed the bot was ignoring valid gap setups. The old code used `round(gap_pct, 1)` at a critical point, which meant a 0.3% gap stayed as `0.3`. The new "clean" code used full floating-point precision, and `0.2999999999999997` failed the `>= 0.3` threshold check.

The result: the bot silently skipped every borderline gap setup for 14 days. Lost opportunities worth thousands of dollars.

**The root cause was not the rounding.** The root cause was that nobody wrote tests capturing what the old code actually produced before replacing it.

If they had written one characterization test:

```python
def test_gap_calculation_borderline_case():
    # Characterization: captures what the OLD code actually returns
    result = calculate_gap(previous_close=100.0, current_open=100.3)
    assert result == 0.3  # Old code rounds to 1 decimal place
```

...the rewrite would have failed this test immediately, and the rounding behavior would have been preserved or consciously changed.

---

## 2. Michael Feathers' 5-Step Technique (Adapted for Trading Code)

From *Working Effectively with Legacy Code*, adapted for trading systems:

### Step 1: Identify the Boundary

Find everything the function touches:

- **Inputs**: parameters, global config, database state, broker API responses
- **Outputs**: return value, side effects, database writes, orders placed, logs emitted

```python
# Example: this function has hidden inputs and outputs
def calculate_fees(order):
    # Hidden input: reads from global config
    fee_rate = config.FEE_RATE
    # Hidden input: queries database
    volume_today = db.get_daily_volume(order.symbol)
    # Visible input: order parameter
    base_fee = order.quantity * order.price * fee_rate
    # Hidden output: mutates the order object
    order.fees = base_fee * (1 - min(volume_today / 1000000, 0.5))
    # Hidden output: writes to database
    db.record_fee(order.id, order.fees)
    # Visible output: return value
    return order.fees
```

### Step 2: Create the Test Harness

Wrap the function so you control every input and capture every output. Mock at the boundary.

```python
import pytest
from unittest.mock import patch, MagicMock

@pytest.fixture
def fee_harness():
    """Harness that controls all inputs to calculate_fees."""
    mock_db = MagicMock()
    mock_config = MagicMock()

    with patch("trading.fees.db", mock_db), \
         patch("trading.fees.config", mock_config):
        yield {
            "db": mock_db,
            "config": mock_config,
            "call": calculate_fees,
        }
```

### Step 3: Write a Failing Assertion (Guess the Output)

Don't read the code carefully. Just guess:

```python
def test_fees_normal_order(fee_harness):
    fee_harness["config"].FEE_RATE = 0.001
    fee_harness["db"].get_daily_volume.return_value = 0

    order = MagicMock(quantity=100, price=50.0, symbol="AAPL")
    result = fee_harness["call"](order)

    # Guess: 100 * 50 * 0.001 = 5.0? Let's see.
    assert result == 0  # Deliberately wrong — we want the failure message
```

### Step 4: Capture the Truth

The test fails:

```
AssertionError: assert 5.0 == 0
```

Now you know: `calculate_fees` returns `5.0` for these inputs. You didn't have to read every line of the function to learn this.

### Step 5: Lock the Behavior

```python
def test_fees_normal_order(fee_harness):
    fee_harness["config"].FEE_RATE = 0.001
    fee_harness["db"].get_daily_volume.return_value = 0

    order = MagicMock(quantity=100, price=50.0, symbol="AAPL")
    result = fee_harness["call"](order)

    assert result == 5.0  # LOCKED: this is what the code actually does
    assert order.fees == 5.0  # LOCKED: side effect captured
    fee_harness["db"].record_fee.assert_called_once()  # LOCKED: DB write captured
```

The test passes. The behavior is locked. Now you can refactor with confidence.

---

## 3. Trading-Specific Adaptations

Before refactoring any of these, write characterization tests first:

### Order Flow Paths

Before consolidating multiple order paths into a single execution gateway:

```python
class TestOrderFlowCharacterization:
    """Lock existing order flow behavior before refactoring."""

    @pytest.mark.asyncio
    async def test_market_order_flow(self):
        """Characterize: what actually happens when we send a market order?"""
        broker = MockBroker()
        broker.fill_price = 150.25

        engine = TradingEngine(broker=broker)
        result = await engine.execute_order(
            symbol="AAPL",
            side="buy",
            order_type="market",
            quantity=100,
        )

        # Lock every observable behavior
        assert result.status == "filled"
        assert result.fill_price == 150.25
        assert result.quantity == 100
        assert broker.orders_submitted == 1
        assert engine.position_manager.get_position("AAPL").size == 100

    @pytest.mark.asyncio
    async def test_limit_order_flow(self):
        """Characterize: limit orders have different side effects."""
        broker = MockBroker()
        broker.fill_price = None  # Not filled yet

        engine = TradingEngine(broker=broker)
        result = await engine.execute_order(
            symbol="AAPL",
            side="buy",
            order_type="limit",
            quantity=100,
            price=149.00,
        )

        # Lock: unfilled limit orders have different state
        assert result.status == "pending"
        assert result.fill_price is None
        assert engine.position_manager.get_position("AAPL").size == 0
        assert engine.pending_orders["AAPL"].price == 149.00

    @pytest.mark.asyncio
    async def test_order_with_risk_rejection(self):
        """Characterize: what happens when risk check fails?"""
        broker = MockBroker()
        risk_gate = MockRiskGate(allow=False)

        engine = TradingEngine(broker=broker, risk_gate=risk_gate)
        result = await engine.execute_order(
            symbol="AAPL", side="buy", order_type="market", quantity=100
        )

        # Lock: rejected orders never reach the broker
        assert result.status == "rejected"
        assert result.rejection_reason == "risk_check_failed"
        assert broker.orders_submitted == 0
```

### Risk Gate Return Values

Before refactoring to a `RiskDecision` pattern:

```python
def test_risk_gate_return_values():
    """Lock what the risk gate actually returns in each scenario."""
    gate = RiskGate(max_position=1000, max_daily_loss=500)

    # Normal case
    assert gate.check("AAPL", 100, 150.0) == True

    # At limit
    assert gate.check("AAPL", 1000, 150.0) == True

    # Over limit — does it return False or raise?
    assert gate.check("AAPL", 1001, 150.0) == False

    # Negative quantity — what does it actually do?
    assert gate.check("AAPL", -100, 150.0) == True  # Allows closing positions
```

### Position State Mutations

Before extracting a `PositionTracker`:

```python
def test_position_mutations():
    """Lock how position state changes through a trade lifecycle."""
    pm = PositionManager()

    pm.update_fill("AAPL", side="buy", qty=100, price=150.0)
    pos = pm.get_position("AAPL")
    assert pos.size == 100
    assert pos.avg_price == 150.0
    assert pos.unrealized_pnl(150.0) == 0.0

    pm.update_fill("AAPL", side="buy", qty=50, price=160.0)
    pos = pm.get_position("AAPL")
    assert pos.size == 150
    assert pos.avg_price == pytest.approx(153.33, abs=0.01)

    pm.update_fill("AAPL", side="sell", qty=150, price=155.0)
    pos = pm.get_position("AAPL")
    assert pos.size == 0
    assert pos.realized_pnl == pytest.approx(250.0, abs=1.0)
```

### Fee/Commission Calculations and Indicator Computations

Always characterize before simplifying fee structures or switching indicator libraries.

---

## 4. The Characterize-Then-Refactor Loop

Follow this loop exactly. Do not skip steps.

```
1. Write characterization tests  -->  Lock current behavior
2. Extract one small piece       -->  Single function, single responsibility
3. Run tests                     -->  MUST stay green
4. Commit                        -->  Structure change only
5. Repeat from step 2
```

**NEVER change behavior and structure in the same commit.**

A behavior change is: the function returns a different value, has different side effects, or handles an edge case differently.

A structure change is: moving code to a new function, renaming, splitting a class, changing how the same result is computed.

### Example: Extracting a Fee Calculator

**Before** (monolithic):

```python
def execute_order(symbol, side, quantity, price, order_type="market"):
    # ... 50 lines of validation ...

    # Fee calculation buried in the middle
    base_fee = quantity * price * 0.001
    if quantity > 1000:
        base_fee *= 0.9  # Volume discount
    if order_type == "limit":
        base_fee *= 0.8  # Limit order discount
    fee = round(base_fee, 2)

    # ... 80 more lines of order execution ...
    return {"status": "filled", "fee": fee, "net_cost": quantity * price + fee}
```

**Step 1: Characterize the fee behavior** (before touching anything):

```python
class TestFeeCharacterization:
    def test_basic_fee(self):
        result = execute_order("AAPL", "buy", 100, 150.0)
        assert result["fee"] == 15.0

    def test_volume_discount(self):
        result = execute_order("AAPL", "buy", 1500, 150.0)
        assert result["fee"] == 202.5  # 1500*150*0.001*0.9

    def test_limit_order_discount(self):
        result = execute_order("AAPL", "buy", 100, 150.0, order_type="limit")
        assert result["fee"] == 12.0  # 100*150*0.001*0.8

    def test_combined_discounts(self):
        result = execute_order("AAPL", "buy", 1500, 150.0, order_type="limit")
        assert result["fee"] == 162.0  # 1500*150*0.001*0.9*0.8
```

**Step 2: Extract** (structure-only change):

```python
def _calculate_fee(quantity, price, order_type):
    """Extracted from execute_order. Behavior unchanged."""
    base_fee = quantity * price * 0.001
    if quantity > 1000:
        base_fee *= 0.9
    if order_type == "limit":
        base_fee *= 0.8
    return round(base_fee, 2)


def execute_order(symbol, side, quantity, price, order_type="market"):
    # ... 50 lines of validation ...
    fee = _calculate_fee(quantity, price, order_type)
    # ... 80 more lines of order execution ...
    return {"status": "filled", "fee": fee, "net_cost": quantity * price + fee}
```

**Step 3: Run tests.** All green. Commit: `"refactor: extract _calculate_fee from execute_order"`.

**Step 4: Now** you can improve `_calculate_fee` in a separate commit if needed, with tests protecting you.

---

## 5. Edge Cases to Characterize

For every function you plan to refactor, test these categories:

```python
class CharacterizationTestBuilder:
    """Helper to systematically characterize a function's behavior."""

    def __init__(self, func, default_args=None, default_kwargs=None):
        self.func = func
        self.default_args = default_args or []
        self.default_kwargs = default_kwargs or {}
        self.cases = []

    def add_case(self, name, args=None, kwargs=None, expect_exception=None):
        self.cases.append({
            "name": name,
            "args": args or self.default_args,
            "kwargs": {**self.default_kwargs, **(kwargs or {})},
            "expect_exception": expect_exception,
        })
        return self

    def discover(self):
        """Run all cases and print actual results for locking."""
        for case in self.cases:
            try:
                result = self.func(*case["args"], **case["kwargs"])
                print(f"  {case['name']}: returned {result!r}")
            except Exception as e:
                print(f"  {case['name']}: raised {type(e).__name__}({e})")

    def build_test_cases(self):
        """Generate locked assertions after discovery."""
        locked = []
        for case in self.cases:
            try:
                result = self.func(*case["args"], **case["kwargs"])
                locked.append((case["name"], case["args"], case["kwargs"], result, None))
            except Exception as e:
                locked.append((case["name"], case["args"], case["kwargs"], None, type(e)))
        return locked


# Usage: systematically characterize calculate_gap
builder = CharacterizationTestBuilder(calculate_gap)
builder.add_case("normal_gap", args=[100.0, 105.0])
builder.add_case("zero_gap", args=[100.0, 100.0])
builder.add_case("negative_gap", args=[105.0, 100.0])
builder.add_case("zero_previous_close", args=[0.0, 100.0])
builder.add_case("none_input", args=[None, 100.0])
builder.add_case("both_zero", args=[0.0, 0.0])
builder.add_case("very_small_gap", args=[100.0, 100.001])

# Step 1: Discover actual behavior
builder.discover()
```

Output:

```
  normal_gap: returned 5.0
  zero_gap: returned 0.0
  negative_gap: returned -5.0
  zero_previous_close: raised ZeroDivisionError(division by zero)
  none_input: raised TypeError(unsupported operand type(s))
  both_zero: raised ZeroDivisionError(division by zero)
  very_small_gap: returned 0.0  # <-- Interesting! Rounding hides small gaps
```

Now lock every one of these:

```python
class TestCalculateGapCharacterization:
    def test_normal_gap(self):
        assert calculate_gap(100.0, 105.0) == 5.0

    def test_zero_gap(self):
        assert calculate_gap(100.0, 100.0) == 0.0

    def test_negative_gap(self):
        assert calculate_gap(105.0, 100.0) == -5.0

    def test_zero_previous_close_raises(self):
        with pytest.raises(ZeroDivisionError):
            calculate_gap(0.0, 100.0)

    def test_none_input_raises(self):
        with pytest.raises(TypeError):
            calculate_gap(None, 100.0)

    def test_both_zero_raises(self):
        with pytest.raises(ZeroDivisionError):
            calculate_gap(0.0, 0.0)

    def test_very_small_gap_rounds_to_zero(self):
        # NOTE: This is surprising behavior! The old code rounds
        # small gaps to zero. Preserve this during refactoring,
        # then decide separately whether to change it.
        assert calculate_gap(100.0, 100.001) == 0.0
```

---

## 6. When to Stop Characterizing

You don't need 100% coverage. Follow this rule:

**Characterize the code you're about to change, plus one layer of callers.**

If `calculate_fees()` is called by:
1. `execute_market_order()`
2. `execute_limit_order()`
3. `generate_daily_report()`

Then characterize:
- `calculate_fees()` directly (edge cases, normal cases)
- The fee-related behavior visible through each of the 3 callers

**Stop when you can answer YES to**: "If my refactoring changes the return value or side effects of this function, will at least one test fail?"

If yes, you have enough characterization. Move on to refactoring.

**Do NOT characterize**:
- Code you aren't going to change
- Internal implementation details (only observable behavior)
- Code that already has good unit tests

---

## Red Flags

| Red Flag | What It Means | What to Do |
|----------|--------------|------------|
| "I'll just rewrite this, it's not that complex" | You're about to introduce subtle bugs | Write characterization tests first, no exceptions |
| Refactoring changes test results | You changed behavior, not just structure | Revert and split into separate commits |
| "The old code was wrong anyway" | Maybe, but prove it with a test first | Lock the old behavior, then fix in a separate commit with a deliberate test update |
| Function has no return value, only side effects | Harder to characterize but more important to | Mock and assert on every side effect: DB writes, API calls, state mutations |
| "I'll write tests after the refactor" | You won't know if the tests match old behavior | Tests must be written against the OLD code first |
| Test uses `pytest.approx` everywhere | Floating-point behavior may be significant | First lock exact values, only use `approx` when you confirm precision doesn't matter |
| Characterization test seems to test a bug | The bug might be relied upon downstream | Lock it, document it, fix it separately with full impact analysis |

---

## Integration Points

- **test-driven-development**: Characterization tests transition into proper unit tests. Once behavior is locked and refactored, evolve characterization tests into specification tests that describe *intended* behavior.
- **trading-tdd**: Use the Red-Green-Refactor cycle *after* characterization. Characterization gets you to green; then TDD guides new behavior.
- **systematic-debugging**: When a characterization test fails after refactoring, use systematic debugging to find exactly where the behavior diverged.
- **verification-before-completion**: After refactoring, run the full characterization suite as your verification step. No green suite, no merge.
