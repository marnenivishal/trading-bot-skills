# Root Cause Tracing for Trading Systems

## The Technique

Root cause tracing is a backward analysis technique. You start from the observable symptom and trace backward through the system until you find the first point where actual behavior diverged from expected behavior. That divergence point is your root cause.

## Why Backward Tracing

Forward tracing ("let me read the code from the beginning") is inefficient for debugging because:
- The codebase is large. You do not know where the bug is.
- The bug may be in an unexpected component.
- Forward tracing biases you toward the "happy path" -- bugs live on the unhappy paths.

Backward tracing starts from the KNOWN bad output and follows the chain of causation directly to the source.

## The Trading System Trace Chain

Trading systems have a natural chain of components. When tracing backward from a symptom, follow this chain in reverse order:

```
Symptom (observable wrong behavior)
    |
    v
Fill Handler
    - Did the fill arrive correctly from the broker?
    - Was it processed correctly?
    - Was the position updated correctly?
    |
    v
Order Manager
    - Was the order submitted correctly?
    - Was the order the right type, size, price?
    - Did the order go through dedup and risk checks?
    |
    v
Execution Gateway
    - Was the order request correctly formatted?
    - Did it pass all validation checks?
    - Was the kill switch checked?
    |
    v
Risk Manager
    - Did the risk check pass when it should have?
    - Did the risk check fail-closed on error?
    - Were the inputs to the risk check correct?
    |
    v
Signal Engine
    - Was the signal generated correctly?
    - Were the signal parameters (symbol, side, size) correct?
    - Was the signal deduplicated?
    |
    v
Data Pipeline
    - Was the market data correct and fresh?
    - Were indicators calculated correctly?
    - Was the data complete (no gaps)?
    |
    v
External Source
    - Was the broker API returning correct data?
    - Was the data feed working correctly?
    - Was the config loaded correctly?
```

## Step-by-Step Backward Trace

### Step 1: Document the Symptom

Write down exactly what is wrong. Be specific.

**Bad**: "The order was wrong."
**Good**: "A BUY order for 200 shares of AAPL was placed at 10:15:03 UTC, but the signal called for 100 shares. The order was filled at $152.30. Expected position: 100 shares. Actual position: 200 shares."

### Step 2: Check the Fill Handler

Start at the point closest to the symptom.

Questions to answer:
- What fill(s) were received from the broker?
- Do the fill quantities match the order quantity?
- Were partial fills accumulated correctly?
- Was the position tracker updated correctly for each fill?
- Did any fill arrive out of order or duplicated?

```python
# Example: Check fill processing logs
# Look for entries like:
# [FILL] order_id=ORD-123 symbol=AAPL side=buy qty=200 price=152.30
# [POSITION] symbol=AAPL prev=0 new=200 source=fill

# If fill qty=200 but order was for 100, the problem is upstream (order manager).
# If fill qty=100 but position shows 200, the problem is in the fill handler.
```

**If the fill handler processed correctly, trace backward to the order manager.**

### Step 3: Check the Order Manager

Questions to answer:
- What order was submitted to the broker?
- What were the order parameters (symbol, side, quantity, type, price)?
- Do they match what the signal engine requested?
- Did the order go through the dedup gate?
- Was the order logged before submission?

```python
# Example: Check order submission logs
# Look for entries like:
# [ORDER_SUBMIT] order_id=ORD-123 symbol=AAPL side=buy qty=200 type=market
# [DEDUP_CHECK] symbol=AAPL action=buy result=ALLOWED
# [RISK_CHECK] symbol=AAPL qty=200 result=APPROVED

# If qty=200 here but signal was for 100, the problem is between signal and order.
# If qty=100 here but fill was for 200, the problem is at the broker or fill handler.
```

**If the order manager submitted correctly, trace backward to the execution gateway and risk manager.**

### Step 4: Check the Execution Gateway and Risk Manager

Questions to answer:
- Did the risk check approve this order?
- Were the inputs to the risk check correct?
- Did any risk check throw an exception? If so, was it handled fail-closed?
- Was the kill switch checked?
- Was the order validated (symbol tradeable, quantity valid, etc.)?

```python
# Example: Check risk manager logs
# Look for entries like:
# [RISK] check=position_limit symbol=AAPL current=0 requested=200 limit=1000 result=APPROVED
# [RISK] check=daily_loss current=-500 limit=-5000 result=APPROVED
# [KILL_SWITCH] status=INACTIVE

# If risk approved qty=200 but signal was for 100, the problem is upstream.
# If risk check threw an exception and returned None, that is a fail-open bug.
```

**If the risk checks were correct, trace backward to the signal engine.**

### Step 5: Check the Signal Engine

Questions to answer:
- What signal was generated?
- Were the signal parameters correct (symbol, action, quantity, price)?
- Was the signal based on correct, fresh data?
- Was the signal deduplicated correctly?
- Was there a duplicate signal that slipped through dedup?

```python
# Example: Check signal generation logs
# Look for entries like:
# [SIGNAL] symbol=AAPL action=buy quantity=100 source=mean_reversion
# [SIGNAL] symbol=AAPL action=buy quantity=100 source=momentum  <-- DUPLICATE?

# If two signals both generated qty=100 and both passed dedup,
# the total order would be 200. Root cause: dedup gate failure.
```

**If the signal was correct, trace backward to the data pipeline.**

### Step 6: Check the Data Pipeline

Questions to answer:
- What market data was fed to the signal engine?
- Was the data fresh (within staleness threshold)?
- Were indicators calculated correctly?
- Was there a data gap or anomaly?
- Were any data points NaN, None, or zero?

```python
# Example: Check data pipeline logs
# Look for entries like:
# [DATA] symbol=AAPL price=152.00 timestamp=2025-01-15T10:14:58Z age=5s status=FRESH
# [INDICATOR] symbol=AAPL sma_10=151.50 sma_50=149.80 rsi=65.3

# If the data was stale but passed staleness check,
# root cause: staleness threshold too generous or check bypassed.
```

### Step 7: Identify the Divergence Point

At some point in the trace, you will find where expected behavior and actual behavior diverge. That is your root cause.

**Example divergence points:**

| Trace Point | Expected | Actual | Root Cause |
|---|---|---|---|
| Signal quantity | 100 | 100 | Not here, trace further |
| Dedup check | Block second signal | Allowed both | Dedup key did not include source |
| Order quantity | 100 | 200 | Two signals reached order manager |
| Fill quantity | 200 | 200 | Correct given two orders |
| **Root cause** | | | Dedup gate used wrong key, allowed two signals through |

## Trading-Specific Trace Shortcuts

For common symptom categories, start the trace at the most likely point:

### Position Mismatch
Start at: **Reconciliation -> Fill Handler -> Order Manager**
Common causes: Partial fill not tracked, ghost position from options exercise, fill notification lost.

### Unexpected Order
Start at: **Order Manager -> Dedup Gate -> Signal Engine**
Common causes: Dedup bypass, duplicate signal sources, fail-open risk check.

### Wrong Order Size
Start at: **Order Manager -> Position Sizer -> Risk Manager**
Common causes: Position sizer rounding error, config value wrong, risk check returning wrong limit.

### Kill Switch Did Not Trigger
Start at: **Kill Switch -> Loss Tracker -> Fill Handler**
Common causes: Loss not recorded, threshold config wrong, kill switch check bypassed.

### Stale Data Trade
Start at: **Data Validator -> Data Pipeline -> External Feed**
Common causes: Staleness check missing, threshold too generous, timestamp in wrong timezone.

### Silent Feature Failure
Start at: **Task Registry -> Async Task -> Done Callback**
Common causes: Task exception with no done_callback, task cancelled silently, event loop saturated.

## Documenting the Trace

When you complete a backward trace, document it for future reference:

```markdown
## Bug Trace: [Brief description]

**Symptom**: [Exact observable behavior]
**Expected**: [What should have happened]

**Trace**:
1. Fill handler: [OK / Issue found] - [Details]
2. Order manager: [OK / Issue found] - [Details]
3. Execution gateway: [OK / Issue found] - [Details]
4. Risk manager: [OK / Issue found] - [Details]
5. Signal engine: [OK / Issue found] - [Details]
6. Data pipeline: [OK / Issue found] - [Details]

**Root cause**: [Component] - [Exact description of the bug]
**Divergence point**: [Where expected != actual]
**Fix**: [What was changed to fix it]
**Preventive test**: [Test added to catch this class of bug]
```

## Common Mistakes in Root Cause Tracing

1. **Stopping too early**: Finding a symptom and calling it the root cause. Keep tracing until you find the FIRST divergence.
2. **Assuming the root cause**: "It's probably X" without tracing. Trace the evidence.
3. **Skipping components**: Jumping from symptom to data pipeline without checking intermediate components.
4. **Not checking timing**: Many trading bugs are timing-dependent. Include timestamps in your trace.
5. **Ignoring concurrent events**: Check what else was happening at the same time (other orders, reconciliation, data updates).
