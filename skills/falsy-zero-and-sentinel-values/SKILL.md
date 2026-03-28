---
name: falsy-zero-and-sentinel-values
description: "You MUST use this when writing conditional checks on numeric trading data, handling missing market data values, or designing sentinel/default values for prices, VIX, premium, or equity"
---

# Falsy-Zero and Sentinel Values

Python's falsy semantics silently corrupt trading decisions. `0`, `0.0`, and `Decimal("0")` are all `False` in boolean context — but they are valid trading values (zero premium, zero VIX, flat P&L). Using `value or default` on numeric fields conflates "value is zero" with "value is missing."

---

## The Iron Law

> **NEVER USE `value or default` ON NUMERIC TRADING DATA. ZERO IS A VALID VALUE.**
>
> Origin: emabot Failure — VIX=0.0 was treated as missing (falsy), triggering the 99.0 sentinel, which activated extreme-fear position tightening on ALL open positions. 13+ commits to fix across options_manager.py, flow_simulator.py, ema_cloud.py, and broadcast routes.

---

## Core Patterns

### Pattern 1: Explicit None Checks

```python
# BAD: Zero VIX treated as "missing" — triggers 99.0 sentinel
vix = get_vix_value() or 99.0  # VIX=0.0 → returns 99.0!

# GOOD: Explicit None check preserves zero as a valid value
vix = get_vix_value()
if vix is None:
    vix = 99.0  # Fail-closed sentinel: assume extreme fear
```

```python
# BAD: Zero premium treated as "missing"
entry_premium = row.get("premium") or 0.0  # $0.00 premium → 0.0 (looks ok but hides None)

# GOOD: Distinguish between "zero" and "absent"
entry_premium = row.get("premium")
if entry_premium is None:
    raise MissingFieldError("premium is required for P&L calculation")
```

```python
# BAD: Zero P&L treated as falsy
if not trade.pnl:  # Skips trades with exactly $0.00 P&L!
    logger.info("No P&L data")

# GOOD: Explicit None check
if trade.pnl is None:
    logger.info("No P&L data")
```

### Pattern 2: Fail-Closed Sentinel Design

When data is truly missing (API failure, network timeout), sentinels must make the system MORE conservative:

```python
# Sentinel values — each makes the system RESTRICT, not PERMIT
SENTINEL_VIX = 99.0          # Extreme fear → blocks most entries
SENTINEL_EQUITY = -99999.0   # Impossible equity → blocks all entries
SENTINEL_SPREAD = 1.0        # 100% spread → blocks options entry

def get_vix_safe() -> float:
    """Fail-closed VIX fetch. Missing data = assume worst case."""
    try:
        vix = await fetch_vix()
        if vix is None:
            return SENTINEL_VIX
        return vix
    except Exception:
        return SENTINEL_VIX  # API failure = assume extreme fear
```

### Pattern 3: Numeric Field Validation

```python
# BAD: Using truthiness for numeric validation
def validate_order(qty, price, stop_loss):
    if not qty or not price:  # qty=0 or price=0.0 → treated as invalid!
        raise ValidationError("Missing fields")

# GOOD: Explicit type + None checks
def validate_order(qty: int | None, price: float | None, stop_loss: float | None):
    if qty is None or price is None:
        raise ValidationError("Missing fields")
    if qty <= 0:
        raise ValidationError(f"Invalid qty: {qty}")
    if price < 0:
        raise ValidationError(f"Invalid price: {price}")
```

### Pattern 4: Database Row Handling

```python
# BAD: Truthy check on DB row values
row = cursor.fetchone()
entry_vix = row["entry_vix"] or 0.0
peak_premium = row["peak_premium"] or 0.0
realized_pnl = row["realized_pnl"] or 0.0

# GOOD: Explicit None coalescing
entry_vix = row["entry_vix"] if row["entry_vix"] is not None else SENTINEL_VIX
peak_premium = row["peak_premium"] if row["peak_premium"] is not None else 0.0
realized_pnl = row["realized_pnl"] if row["realized_pnl"] is not None else 0.0
```

---

## The Falsy Trap Reference

| Value | `bool()` | `or default` Result | Trading Impact |
|-------|----------|---------------------|----------------|
| `0` | `False` | Returns default | Zero shares = "no order" — skips valid zero-fill check |
| `0.0` | `False` | Returns default | $0.00 premium = "missing" — breaks P&L calc |
| `Decimal("0")` | `False` | Returns default | Zero cost basis = "missing" — breaks reconciliation |
| `""` | `False` | Returns default | Empty ticker — should raise, not use default |
| `None` | `False` | Returns default | Actually missing — this is the ONLY case you want |
| `[]` | `False` | Returns default | Empty positions list — valid, should not trigger default |

---

## Red Flags

| Red Flag | Why It's Dangerous | Correct Pattern |
|----------|-------------------|-----------------|
| `value or 0` on any numeric field | Zero is a valid trading value; conflates zero with None | `value if value is not None else 0` |
| `value or default` for VIX/premium/price | Disables safety gates when value is exactly zero | Explicit `if value is None: value = SENTINEL` |
| `if not value:` on numeric | Catches zero, empty string, None, False — too broad | `if value is None:` for missing check |
| `bool(quantity)` in conditionals | `quantity = 0` is valid (fully filled) — fails bool check | `if quantity is not None and quantity > 0:` |
| Sentinel values that ENABLE trading | Missing data should restrict, not permit | Sentinels should be extreme values that trigger safety gates |
| No centralized sentinel constants | Same sentinel defined differently in different files | Single `constants.py` with all sentinel values |

---

## Integration

| Scenario | Related Skill |
|----------|--------------|
| VIX gate using sentinel values | `trading-bot-skills:market-data-pipeline` |
| Risk checks with missing data | `trading-bot-skills:risk-management-gates` |
| Database NULL handling | `trading-bot-skills:database-safety-for-trading` |
| Options premium calculations | `trading-bot-skills:options-trading-safety` |
| Config defaults for numeric settings | `trading-bot-skills:trading-config-management` |
