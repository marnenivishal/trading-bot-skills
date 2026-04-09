---
name: verification-before-completion
description: Use when about to claim work is complete, fixed, or passing, before committing or creating PRs - requires running verification commands and confirming output
---

# Verification Before Completion

## Purpose

This skill prevents false claims of completion. Before declaring any work as "done," "fixed," or "passing," you MUST run verification commands and confirm the output with your own eyes. In trading systems, claiming something works when it does not can result in deploying broken code that loses real money.

## The Rule

**NEVER claim work is complete without running verification commands and confirming the output.** "I think it works" is not verification. "The tests pass" (without showing output) is not verification. Run the commands. Read the output. Confirm success.

## When to Use This Skill

- Before committing code.
- Before creating a pull request.
- Before claiming a bug is fixed.
- Before claiming tests pass.
- Before claiming a feature is complete.
- Before any status update that says "done."

---

## Verification Process

### Step 1: Run the Tests

Run the FULL test suite, not just the tests for your changes. Show the output.

```bash
# Run full test suite
pytest tests/ -v --tb=short

# Run with coverage to verify branch coverage
pytest tests/ --cov=src --cov-report=term-missing

# Run specific test categories if applicable
pytest tests/trading/ -v -k "TestFailClosed"
```

**You MUST see the output.** Do not assume tests pass because they passed last time.

### Step 2: Check for Regressions

Verify that your changes did not break anything that was previously working.

```bash
# Full suite must pass
pytest tests/ -v --tb=short 2>&1 | tail -20

# Check for any warnings that might indicate future failures
pytest tests/ -v --tb=short -W error 2>&1 | tail -20
```

### Step 3: Run the Trading Verification Checklist

For ANY change to trading code, verify ALL of the following. Check each box only after running the specified verification.

---

## Trading Verification Checklist

### Core Test Categories

- [ ] **All 10 trading-tdd test categories passing**
  ```bash
  pytest tests/ -v --tb=short
  ```
  Confirm: zero failures, zero errors.

### Fail-Closed Patterns

- [ ] **No fail-open patterns in new or modified code**
  Search for exception handlers that return None or silently continue:
  ```bash
  # Search for bare except or except that doesn't reject
  grep -rn "except:" src/ --include="*.py" | grep -v "REJECT\|raise\|log\|#"
  ```
  Confirm: no suspicious exception handlers. Every except block either raises, returns an explicit REJECT, or logs the error.

### Order Path Integrity

- [ ] **All new order paths go through unified dedup gate**
  If your change adds any code that can submit an order:
  ```bash
  grep -rn "submit_order\|place_order\|send_order" src/ --include="*.py"
  ```
  Confirm: every order submission call is preceded by dedup gate check and risk check.

### Async Task Safety

- [ ] **All new async tasks have done_callbacks**
  If your change creates any new async tasks:
  ```bash
  grep -rn "create_task\|ensure_future\|asyncio.Task" src/ --include="*.py"
  ```
  Confirm: every task creation has a corresponding `add_done_callback` call.

### Config Validation

- [ ] **All new config values are in the single config file with validation**
  If your change introduces new configuration:
  ```bash
  grep -rn "config\." src/ --include="*.py" | grep -v "test_\|__pycache__"
  ```
  Confirm: no hardcoded values that should be in config. All config values have startup validation.

### Data Freshness

- [ ] **All new price/data usage has freshness checks**
  If your change consumes market data:
  ```bash
  grep -rn "price\|close\|bid\|ask\|quote" src/ --include="*.py" | grep -v "test_\|__pycache__"
  ```
  Confirm: every data access point checks for staleness before using the data.

### Trailing Stop Monotonicity

- [ ] **Trailing stop monotonicity test passing** (if trailing stop code was modified)
  ```bash
  pytest tests/ -v -k "trailing_stop" -k "monoton"
  ```
  Confirm: trailing stop never decreases.

### Backtest Validation

- [ ] **Backtest results match expectations** (if strategy logic was changed)
  ```bash
  python -m backtest.run --strategy=<strategy_name> --report
  ```
  Confirm: Sharpe > 1.0, max drawdown within tolerance, profit factor > 1.2.

---

## Verification Output Format

When reporting verification results, use this format:

```
## Verification Report

### Test Suite
- Command: `pytest tests/ -v --tb=short`
- Result: X passed, Y failed, Z errors
- [PASS/FAIL]

### Fail-Closed Check
- Command: `grep -rn "except:" src/ --include="*.py"`
- Suspicious handlers found: [none / list them]
- [PASS/FAIL]

### Order Path Check
- New order submission points: [none / list them]
- All go through dedup + risk: [yes/no]
- [PASS/FAIL]

### Async Task Check
- New async tasks: [none / list them]
- All have done_callbacks: [yes/no]
- [PASS/FAIL]

### Config Check
- New config values: [none / list them]
- All have validation: [yes/no]
- [PASS/FAIL]

### Data Freshness Check
- New data access points: [none / list them]
- All have freshness checks: [yes/no]
- [PASS/FAIL]

### Overall: [ALL PASS / FAILURES FOUND]
```

---

## What Counts as "Verified"

| Status | Meaning | Can Claim Done? |
|--------|---------|-----------------|
| Tests ran, all green, output shown | Verified | YES |
| Tests ran, some failures | Not verified | NO -- fix failures first |
| Tests not run, "should work" | Not verified | NO -- run the tests |
| Tests ran on subset only | Partially verified | NO -- run full suite |
| Tests passed yesterday | Stale verification | NO -- run again now |
| Linter passes but no tests | Not verified | NO -- tests required |

## Before Committing

```bash
# 1. Run full test suite
pytest tests/ -v --tb=short

# 2. Check for lint/type errors (if applicable)
flake8 src/ --max-line-length=120
mypy src/ --ignore-missing-imports

# 3. Check for debug artifacts
grep -rn "breakpoint()\|pdb\|print(" src/ --include="*.py" | grep -v "test_\|__pycache__\|logging"

# 4. Check for hardcoded secrets
grep -rn "api_key\|secret\|password\|token" src/ --include="*.py" | grep -v "test_\|config\|__pycache__"

# 5. Review diff one more time
git diff --staged
```

## Before Creating a PR

Everything in "Before Committing" PLUS:

```bash
# 1. All commits are clean (no WIP, no fixup)
git log --oneline -10

# 2. Branch is up to date with base
git fetch origin && git log HEAD..origin/main --oneline

# 3. No merge conflicts
git merge --no-commit --no-ff origin/main && git merge --abort

# 4. Full verification checklist completed and documented
```

---

## Red Flags -- Stop

- **"I'm pretty sure it works"**: Run the tests. Be certain.
- **"Tests passed earlier"**: Run them again. Code has changed since then.
- **"Only my changes need testing"**: Run the full suite. Your changes may break other things.
- **"The linter passes, so it's fine"**: Linter checks syntax, not correctness. Run the tests.
- **Skipping the trading verification checklist**: Every checklist item exists because a bug was once deployed that it would have caught.
- **Committing without running `git diff --staged`**: Review what you are actually committing.
- **"It works on my machine"**: Run tests in the CI environment or equivalent. Environment differences cause bugs.

## Multi-Agent Review for Trading Code

For trading bot changes that touch order logic, risk controls, or signal processing, invoke the **trading-code-reviewer** skill before claiming completion. This runs a structured 5-agent review requiring dual agreement on all findings. Standard verification catches "does it work?" — the trading review catches "could it lose money?"
