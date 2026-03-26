---
name: systematic-debugging
description: Use when encountering any bug, test failure, or unexpected behavior in trading systems, before proposing fixes
---

# Systematic Debugging

## Purpose

This skill enforces a structured, evidence-based approach to debugging trading systems. No fix is proposed until the root cause is identified through systematic investigation. In trading systems, guessing at fixes is how you turn a small bug into a catastrophic financial loss.

## The Rule

**NEVER propose a fix before completing the investigation phases.** A premature fix that masks the symptom can leave the root cause active, silently corrupting positions or placing incorrect orders during live trading.

## When to Use This Skill

- Any test failure.
- Any unexpected behavior in the trading bot.
- Any position mismatch or reconciliation failure.
- Any kill switch activation you do not fully understand.
- Any log entry that seems wrong or unexpected.
- Any divergence between backtest and live performance.
- Any "it works sometimes" intermittent issue.

---

## Phase 1: Reproduce

Before investigating, confirm you can reproduce the issue reliably.

### Steps

1. **Capture the exact symptoms**: What happened? What was expected? What was observed instead? Be precise -- "it didn't work" is not a symptom description.
2. **Capture the context**: What time? What market conditions? What was the system state (positions, pending orders, kill switch status)?
3. **Find the trigger**: What input, event, or sequence of events causes this behavior?
4. **Write a failing test**: If possible, write a pytest test that reproduces the bug. This test will later verify the fix.
5. **Isolate**: Can you reproduce in a test environment (mock/paper) without live broker? If yes, debug there. If no, the bug may be environment-specific.

### Trading-Specific Reproduction Checks

- [ ] Can you reproduce with mock broker? (If not, may be broker API specific.)
- [ ] Can you reproduce outside market hours? (If not, may depend on real-time data or market state.)
- [ ] Can you reproduce with captured data replay? (Capture and replay the exact data that triggered the issue.)
- [ ] Is the issue timing-dependent? (Race condition -- see Phase 2 async checks.)

---

## Phase 2: Gather Evidence

Collect ALL relevant data before forming any hypothesis. Do not jump to conclusions.

### Standard Evidence Collection

1. **Logs**: Pull log entries from the time of the incident. Include entries from 5-10 minutes BEFORE the symptom (the cause precedes the symptom).
2. **State snapshots**: Position state, order state, kill switch state, config values at incident time.
3. **Data inputs**: What market data was being processed? Prices, timestamps, indicators.
4. **Stack traces**: If an exception occurred, capture the full trace with local variables.
5. **Timeline**: Reconstruct the exact chronological sequence of events leading to the symptom.
6. **Diffs**: What code or config changed since the last known-good state?

### Trading Domain Checks

Run EVERY one of these checks, even if you believe the issue is unrelated. Trading bugs frequently have surprising root causes that cross component boundaries.

#### Check Position State

- [ ] Do local positions match broker positions right now?
- [ ] Did they match at the time of the incident?
- [ ] When was the last successful reconciliation?
- [ ] Are there any ghost positions (broker has, local does not)?
- [ ] Are there any phantom positions (local has, broker does not)?
- [ ] Were there any options that expired recently (grace period check)?

#### Check Data Freshness

- [ ] Are all price feeds returning current data right now?
- [ ] What is the timestamp on the most recent price for each affected symbol?
- [ ] Is the staleness threshold being enforced correctly?
- [ ] Are any feeds returning NaN, None, or zero?
- [ ] Were there any data gaps (missing candles) around the time of the incident?

#### Check Order State Machine

- [ ] Are any orders stuck in invalid states (e.g., "submitted" for an unusually long time)?
- [ ] Are there orders that were partially filled but tracked incorrectly?
- [ ] Did any cancel requests fail silently (sent but never acknowledged)?
- [ ] Are fill notifications being received and processed in order?
- [ ] Are there orphaned orders (in broker system but not in local tracking)?

#### Check Fail-Closed

- [ ] Did any exception handler return None instead of an explicit REJECT?
- [ ] Is there a bare `except:` or `except Exception:` that swallows errors silently?
- [ ] Did a risk check pass when it should have rejected (fail-open)?
- [ ] Search the logs for exceptions that were caught but not properly handled.
- [ ] Run: `grep -rn "except:" src/ | grep -v "REJECT\|raise\|log"` to find suspicious handlers.

#### Check Dedup

- [ ] Were duplicate orders placed for the same signal?
- [ ] Is the dedup gate processing signals correctly?
- [ ] Was there a new code path that bypassed the dedup gate?
- [ ] Check the dedup log for rejected duplicates -- are they expected?

#### Check Config

- [ ] Did any config values change recently (diff against last known-good config)?
- [ ] Are all config values within their valid ranges?
- [ ] Is the system using the expected config file (check the loaded path, not just the expected path)?
- [ ] Were any config values overridden by environment variables?

#### Check Async Tasks

- [ ] Did any async task fail silently (started but never completed, no done_callback fired)?
- [ ] Is the task registry showing all expected tasks as healthy?
- [ ] Are there zombie tasks (running but not making progress)?
- [ ] Check done_callback logs -- are failures being reported?
- [ ] Were any tasks cancelled or timed out?

---

## Phase 3: Analyze

With evidence collected, now form and test hypotheses systematically.

### Hypothesis Formation

1. Based on evidence, form 2-3 candidate hypotheses for the root cause.
2. For each hypothesis, identify:
   - What evidence would **confirm** it.
   - What evidence would **refute** it.
3. Check each hypothesis against evidence already collected.
4. If no hypothesis is confirmed, return to Phase 2 and gather more specific evidence.

### Backward Tracing

Use the backward tracing technique (see `root-cause-tracing.md` for full details):

1. Start from the **symptom** (the observable wrong behavior).
2. Trace backward through the call chain: what function produced this output?
3. For that function: was its input correct? If yes, the bug is in this function. If no, trace backward further.
4. Continue until you find the **first point of divergence** between expected and actual behavior.
5. That divergence point is the root cause.

### Common Trading Bug Patterns

Check if the bug matches any of these known patterns:

| Pattern | Typical Symptom | Typical Root Cause |
|---------|----------------|-------------------|
| **Fail-open** | Unexpected order placed during error condition | Exception handler returned None instead of REJECT |
| **Ghost position** | Position mismatch after options expiration | No grace period tracking, option removed before exercise confirmed |
| **Stale trade** | Trade at obviously wrong price | Price staleness check missing or threshold too generous |
| **Double order** | Duplicate positions, double intended size | Signal bypassed dedup gate through new code path |
| **Silent task death** | Feature stopped working, no errors | Async task died without done_callback, no alert |
| **Config drift** | Behavior changed after deploy | Config value changed, validation did not catch edge case |
| **Partial fill gap** | Position quantity does not match expected | Code tracked order quantity instead of sum of actual fills |
| **Race condition** | Intermittent wrong state | Two concurrent operations modifying shared state without lock |
| **Monotonicity violation** | Trailing stop lowered | Stop level updated on price drop instead of only on new highs |

---

## Phase 4: Fix and Verify

Only after the root cause is identified AND confirmed by evidence do you propose a fix.

### Fix Process

1. **Write a reproducing test**: A test that demonstrates the exact bug. Must fail before fix, pass after.
2. **Propose the fix**: Explain WHY this fix addresses the ROOT CAUSE, not just the symptom.
3. **Check for the same bug elsewhere**: If this pattern exists in one place, search for it in all similar code. Fix every instance.
4. **Implement the minimum fix**: Smallest change that addresses the root cause. Do not refactor unrelated code in the same fix.
5. **Verify the fix**:
   - [ ] Reproducing test now passes.
   - [ ] All existing tests still pass (no regressions).
   - [ ] Fix addresses ROOT CAUSE, not symptom.
6. **Add preventive tests**: Tests that catch this CLASS of bug, not just this specific instance.
7. **Update documentation**: If the fix changes behavior, interfaces, or requirements.

### Trading-Specific Fix Verification

After any fix to trading code, verify these concerns:

- [ ] All 10 trading-tdd test categories still passing.
- [ ] No new fail-open patterns introduced (search for bare `except:` returning None).
- [ ] All order paths still route through unified dedup gate.
- [ ] All async tasks still have done_callbacks registered.
- [ ] Kill switch still functions correctly (test activation and blocking).
- [ ] Reconciliation still functions correctly (test match and mismatch detection).
- [ ] No new config values introduced without validation.

### Fix Categorization

| Root Cause Category | Fix Approach | Audit Scope |
|---|---|---|
| Fail-open exception handler | Add explicit REJECT return | All exception handlers in risk/validation code |
| Missing done_callback | Add callback + registry entry | All async task creation points |
| Dedup bypass | Route through unified gate | All signal/order submission paths |
| Stale data accepted | Add freshness check | All data consumption points |
| Config validation gap | Add startup validation | All config values |
| Partial fill tracking | Track fill sum, not order qty | All position update code |
| Race condition | Add synchronization (lock/queue) | All shared mutable state access |
| Monotonicity violation | Add max() guard | All trailing stop update code |

---

## Debugging Quick Reference Checklist

```
REPRODUCE
[ ] Can I describe the exact symptom precisely?
[ ] Can I reproduce it reliably?
[ ] Do I have a failing test?

GATHER EVIDENCE
[ ] Relevant logs collected (including BEFORE the symptom)?
[ ] Position state checked (local vs broker)?
[ ] Data freshness verified?
[ ] Order state machine checked?
[ ] Fail-closed behavior verified?
[ ] Dedup gate checked?
[ ] Config values verified?
[ ] Async tasks checked?

ANALYZE
[ ] 2-3 hypotheses formed?
[ ] Each hypothesis tested against evidence?
[ ] Root cause identified (not just symptom)?
[ ] Root cause confirmed by evidence?

FIX AND VERIFY
[ ] Reproducing test written?
[ ] Fix addresses root cause?
[ ] Same pattern checked elsewhere?
[ ] All existing tests pass?
[ ] Trading-specific verification complete?
[ ] Preventive tests added?
```

---

## Red Flags -- Stop and Reassess

- **"Let me just try this fix"**: STOP. Follow the phases. Guessing in trading systems costs real money.
- **Fixing the symptom, not the cause**: The bug will return, probably during live trading. Find the root cause.
- **"It works now" without understanding why it failed**: You do not know if the fix is correct or if the bug is intermittent.
- **No reproducing test**: If you cannot test for the bug, you cannot verify the fix. Write the test.
- **Fix that changes many files**: Large fixes suggest you may be treating symptoms. Reconsider the root cause.
- **"This can't happen"**: It did happen. The evidence says so. Trust evidence over assumptions.
- **Silencing an alert instead of fixing the cause**: Alerts are symptoms. Fix what triggered them.
- **"It's probably a race condition"**: Maybe, but prove it. Race conditions are also a common excuse for not investigating thoroughly.
