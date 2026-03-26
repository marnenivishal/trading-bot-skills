---
name: trading-tdd
description: Use when implementing any trading feature or bugfix, before writing implementation code - extends TDD with trading-specific test cases and patterns
---

# Trading Test-Driven Development

## Purpose

This skill extends standard TDD with trading-specific test categories that catch the failure modes unique to automated trading systems. Every trading feature or bugfix MUST have tests written BEFORE implementation code. No exceptions.

## REQUIRED BACKGROUND

This skill extends `trading-bot-skills:test-driven-development`. All standard TDD rules apply (red-green-refactor cycle, test-first, etc.). This skill adds the 10 mandatory trading test categories on top of the base TDD workflow.

## The Iron Rule

**Write the test. Watch it fail. Write the code. Watch it pass. Commit.**

No implementation code is written until a failing test exists. This is not optional. This is not aspirational. This is the process.

## 10 Mandatory Trading Test Categories

Every trading feature MUST include tests from ALL applicable categories below. If a category does not apply, you must explicitly document WHY it does not apply in a comment.

### Category 1: Partial Fill Tests

Orders rarely fill completely in one shot. Your code MUST handle partial fills correctly.

**What to test:**
- Order for 100 shares, fills arrive as 30 + 30 + 40. Verify position tracking after each fill.
- Order for 100 shares, fills arrive as 100 (single fill). Verify same end result.
- Order for 100 shares, only 60 fills, then order cancelled. Verify position is 60, not 100.
- Partial fill on one leg of a spread. Verify state consistency.
- Rapid partial fills arriving out of order. Verify final state is correct regardless of arrival order.

**Key assertion:** After all partial fills, `local_position == sum(all_fills)` and NOT `order_quantity`.

### Category 2: Race Condition Tests

Trading systems are inherently concurrent. Signals, fills, cancels, and data all arrive asynchronously.

**What to test:**
- Cancel request sent, but fill arrives before cancel confirmation. Verify position updated correctly.
- Two signals for same symbol arrive within milliseconds. Verify only one order placed (dedup).
- Price update arrives during order submission. Verify order uses intended price.
- Kill switch triggers while orders are in flight. Verify all pending orders cancelled.
- Reconciliation runs while fills are processing. Verify no false mismatches.

**Key assertion:** System reaches correct final state regardless of event arrival order.

### Category 3: Stale Data Tests

Data feeds fail. Prices go stale. Indicators need minimum bars. Your system must detect and reject stale data.

**What to test:**
- Price feed returns data older than staleness threshold. Verify REJECT, not trade on old price.
- VIX feed unavailable. Verify strategy degrades safely (no trades, not crash).
- Insufficient historical bars for indicator calculation. Verify graceful skip, not index error.
- Data feed returns NaN or None values. Verify detection and rejection.
- Timestamp gap in candle data. Verify detection.

**Key assertion:** Stale or missing data NEVER results in a trade. Always results in explicit rejection or skip.

### Category 4: Slippage Tests

Real fills happen at prices different from expected. Your system must detect and track slippage.

**What to test:**
- Stop loss at $10.00, fill at $8.50. Verify slippage detected and logged.
- Limit order at $50.00, fill at $49.95 (favorable). Verify tracking.
- Slippage exceeds configurable threshold. Verify alert generated.
- Market order during high volatility. Verify slippage calculation correct.
- Aggregate slippage tracking across multiple fills. Verify running total accurate.

**Key assertion:** `slippage = abs(expected_price - fill_price)` is calculated and tracked for every fill.

### Category 5: Fail-Closed Tests

When something goes wrong, the system must REJECT/HALT, never silently continue. This is the most critical category.

**What to test:**
- Inject exception in risk check function. Verify returns REJECT, not None.
- Inject exception in position sizing. Verify returns 0 shares, not unhandled error.
- Inject exception in order validation. Verify order NOT placed.
- Risk function returns None instead of True/False. Verify treated as REJECT.
- Network timeout during risk check. Verify REJECT, not skip-check.

**Key assertion:** Every exception path in risk/validation code results in explicit REJECT. NEVER `None`, NEVER silent pass-through.

### Category 6: Reconciliation Tests

Local state MUST match broker state. When they diverge, the system must detect it and halt.

**What to test:**
- Local shows 100 shares, broker shows 0. Verify mismatch detected and trading halted.
- Local shows 0 shares, broker shows 100 (ghost position). Verify detection.
- Local shows 100 AAPL, broker shows 100 AAPL + 50 MSFT. Verify extra position detected.
- Reconciliation during market hours. Verify it runs without blocking fills.
- Reconciliation finds mismatch, then next run finds match. Verify halt is cleared appropriately.

**Key assertion:** `local_positions == broker_positions` is checked periodically and mismatches halt trading.

### Category 7: Config Validation Tests

Bad configuration must be caught at startup, not during trading.

**What to test:**
- Max position size set to negative number. Verify rejected at load time.
- Required config field missing. Verify clear error message at startup.
- Max loss threshold set to 0 (would halt immediately). Verify rejected or warned.
- API key set to empty string. Verify caught before first API call.
- Config values with wrong types (string where float expected). Verify rejection.

**Key assertion:** Invalid configuration NEVER reaches runtime. All validation happens at startup.

### Category 8: Math Correctness Tests

Indicators, position sizing, and risk calculations must be mathematically correct.

**What to test:**
- Your SMA/EMA/RSI vs a reference library (e.g., TA-Lib, pandas_ta). Verify within epsilon.
- Trailing stop: once raised, NEVER lowered. Test with price sequence that rises then falls.
- Position sizing: verify rounding to valid lot sizes (no fractional shares unless intended).
- P&L calculation: verify against manual calculation for known trade sequences.
- Percentage calculations: verify no divide-by-zero on empty portfolios.

**Key assertion:** All math matches reference implementations within acceptable epsilon. Trailing stops are monotonically non-decreasing.

### Category 9: Kill Switch Tests

When things go catastrophically wrong, the kill switch must work. Every time. No exceptions.

**What to test:**
- Daily loss exceeds max threshold. Verify all trading halted immediately.
- Broker connection drops. Verify trading halted, reconnect attempted.
- N consecutive losses hit threshold. Verify halt.
- Kill switch triggers, then profitable signal arrives. Verify signal REJECTED (no override).
- Manual kill switch activation. Verify immediate halt of all activity.
- Kill switch resets at configured time (e.g., next day). Verify reset works correctly.

**Key assertion:** Once kill switch activates, NO trades execute until explicit reset condition is met.

### Category 10: Dedup Tests

The same trade must never execute twice. Deduplication must be airtight.

**What to test:**
- Same signal emitted twice within dedup window. Verify single order.
- Same signal from two different sources. Verify single order.
- Similar but different signals (same symbol, different direction). Verify both processed.
- Signal, then cancel, then same signal again. Verify second signal processed (cancel clears dedup).
- Dedup window expiration. Verify signal after window is processed correctly.

**Key assertion:** Identical signals within dedup window produce exactly one order.

## Test Environment Rules

| Test Type | Environment | Rationale |
|-----------|-------------|-----------|
| Unit tests | Mocks only | Fast, deterministic, no side effects |
| Integration tests | Paper API | Real API behavior, no real money |
| End-to-end tests | Paper API | Full system validation |
| Performance tests | Mock or Paper | Measure latency, throughput |
| **Live broker** | **NEVER for tests** | **Real money at risk. NEVER.** |

### Environment Setup

```python
# pytest markers for environment separation
# In conftest.py or pytest.ini:
# markers =
#     unit: Mocks only, runs in CI
#     integration: Paper API, runs nightly
#     e2e: Full system, runs before deploy

@pytest.mark.unit        # Mocks only, runs in CI
@pytest.mark.integration # Paper API, runs nightly
@pytest.mark.e2e         # Full system, runs before deploy
```

## Workflow: Adding a Trading Feature

1. **Identify applicable test categories** from the 10 above.
2. **Write tests for ALL applicable categories.** If a category does not apply, add a comment explaining why: `# Category 6 (Reconciliation): N/A - this feature does not affect position tracking`
3. **Run tests. ALL must fail** (red phase).
4. **Write implementation code.**
5. **Run tests. ALL must pass** (green phase).
6. **Refactor** if needed. Tests must still pass.
7. **Commit** with message referencing test categories covered.

## Red Flags -- Stop and Fix

If you see ANY of these, stop implementation and add the missing tests:

- **No partial fill tests** for any order-related feature.
- **No fail-closed tests** for any risk/validation feature.
- **Only happy-path tests** -- where are the error cases?
- **Live broker credentials in test config** -- NEVER.
- **Tests that depend on market hours** -- tests must run 24/7.
- **Tests with `time.sleep()` for synchronization** -- use proper async patterns.
- **Mocked risk checks that return True** -- test the REJECT path too.
- **No dedup tests** for any signal-processing feature.

## Coverage Requirements

- All 10 categories addressed (or explicitly documented as N/A with reasoning).
- Branch coverage > 90% for risk and order management code.
- Every `except` block has a test that triggers it.
- Every config value has a validation test.

## Quick Reference: Test Category Applicability

| Feature Area | Must-Have Categories |
|---|---|
| Order management | 1, 2, 5, 6, 10 |
| Risk management | 3, 5, 7, 8, 9 |
| Signal processing | 2, 3, 8, 10 |
| Position tracking | 1, 2, 6 |
| Configuration | 7 |
| Strategy logic | 3, 4, 8 |
| Kill switch | 5, 9 |
| Data pipeline | 3, 8 |

## See Also

- `trading-test-patterns.md` -- Complete runnable pytest examples for all 10 categories.
