---
name: trading-code-reviewer
description: Use when completing tasks, implementing major features, reviewing PRs, or before merging trading bot code — runs multi-agent review with order safety, strategy logic, robustness, and test coverage verification requiring dual-agent agreement before changes
---

# Trading Bot Code Review Supervisor

## Why This Skill Exists

Trading bot bugs don't just break features — they lose real money. A wrong side,
wrong quantity, wrong symbol, or missing stop can be catastrophic. Standard code
review misses trading-specific dangers because reviewers think about code quality,
not order safety.

This skill runs a structured multi-agent review that requires **at least two
independent reviewers to agree** before any change is accepted.

---

## When to Activate

- After implementing any trading feature or bugfix
- Before merging PRs that touch order logic, risk controls, or signal processing
- When reviewing existing code for production readiness
- Before paper-to-live progression
- On any code that constructs, modifies, or cancels orders

---

## Review Process

### Step 1: Spec & Intent Extraction

Before reviewing code, extract the spec:

1. **What is the bot supposed to do?** (strategy summary in 2-3 sentences)
2. **Explicit invariants** — extract from config, CLAUDE.md, and code comments:
   - Max risk per trade (e.g., "$2,000 budget scaled by confidence")
   - Position limits (e.g., "max 30 open positions")
   - Time restrictions (e.g., "only trade 9:30-15:50 ET")
   - Symbol restrictions (e.g., "only whitelisted tickers from chat signals")
   - Exit requirements (e.g., "always have stop loss, EOD flatten at 15:45")
   - Kill switch behavior (e.g., "blocks new entries, allows exits")
3. **Acceptance criteria** — what must be true for the code to be correct

Output a short SPEC document before proceeding to review.

### Step 2: Parallel Agent Review (5 Perspectives)

Spawn or simulate these 5 specialist reviews. Each reviews the same code + SPEC independently.

#### Agent 1: Order Safety

Focus ONLY on order construction and broker interaction:

- [ ] **Symbol/contract resolution** — is the right instrument being traded? Check `lastTradeDateOrContractMonth` (not `expiry`), correct strike, correct right (C/P)
- [ ] **Side correctness** — BUY/SELL matches the signal direction in ALL code paths
- [ ] **Quantity sizing** — respects max contracts, budget limits, confidence scaling; no path produces qty=0 or negative
- [ ] **Order type & parameters** — LMT/MKT/STP prices make sense; TIF is appropriate
- [ ] **Protective orders** — stop loss exists for every entry; cost basis recovery math is correct
- [ ] **Idempotency** — no duplicate orders on retries or reconnects
- [ ] **eTradeOnly/firmQuoteOnly** — set to False on all Order objects (IBKR warning 10268)
- [ ] **Contract multiplier** — 100x for equity options in all P&L calculations

Flag any path where the bot could send:
- A wrong-side order
- A wildly oversized order (budget check bypassed)
- An order for wrong ticker/expiry/strike
- An order without required stop loss

#### Agent 2: Strategy & Logic

Review signal generation and state management:

- [ ] **State machine correctness** — all transitions are valid; no stuck states
- [ ] **Signal-to-order mapping** — BULLISH signals produce BUY calls, BEARISH produce appropriate orders
- [ ] **Edge cases** — what happens with no data, stale signals, partial fills, simultaneous signals
- [ ] **Time/session filters** — no orders outside allowed hours; EOD flatten triggers reliably
- [ ] **Position tracking** — local state matches what was actually sent to broker
- [ ] **P&L math** — `(fill_price - entry) * qty * 100` with correct signs
- [ ] **Falsy zero** — `if not price` catches legitimate 0.0; use `if price is None` instead

#### Agent 3: Robustness & Error Handling

Focus on failure modes:

- [ ] **API error handling** — timeouts, disconnects, and retries have backoff (not infinite loops)
- [ ] **NaN/None/missing data** — all numeric paths handle missing values safely
- [ ] **Exception handling** — no bare `except: pass` in critical paths; exceptions logged with context
- [ ] **Connection resilience** — reconnect logic has dedup (no timer accumulation)
- [ ] **Graceful degradation** — bot degrades safely when Redis/DB/Gemini is down
- [ ] **Kill switch** — actually stops new entries when activated; tested path exists

#### Agent 4: Tests & Maintainability

Check test coverage and code quality:

- [ ] **Critical path tests** — order building, risk checks, and P&L calculations have tests
- [ ] **Integration tests** — DB operations tested against real PostgreSQL (not just mocks)
- [ ] **Edge case tests** — unfilled orders, partial fills, zero prices, connection drops
- [ ] **Config externalization** — risk limits in config, not hardcoded
- [ ] **No hardcoded secrets** — API keys from environment, not in source files

#### Agent 5: Trading-Specific Dangers

Domain-specific checks other agents miss:

- [ ] **Overnight risk** — positions closed before EOD (0DTE options especially)
- [ ] **Options expiration** — 0DTE handled correctly (PM settlement for SPX)
- [ ] **Timezone consistency** — all comparisons in ET; no naive datetime usage
- [ ] **Market data staleness** — stale prices detected and handled
- [ ] **Reconciliation** — mechanism to detect DB/broker position divergence
- [ ] **Budget enforcement** — daily budget can't be bypassed by concurrent signals

### Step 3: Agreement & Prioritization

**Dual-agreement rule**: An issue becomes HIGH PRIORITY only if **at least two agents independently flag the same problem area**.

Classify by severity:

| Severity | Definition | Action |
|----------|-----------|--------|
| **CRITICAL** | Can cause wrong or unsafe trades; money at risk | Must fix before any live trading |
| **IMPORTANT** | Can cause incorrect behavior, crashes, or data corruption | Should fix before next deploy |
| **NICE-TO-HAVE** | Refactors, style, minor clarity improvements | Fix when convenient |

### Step 4: Change Plan

Produce a structured plan for implementation:

For each issue:
```
ISSUE: [short title]
SEVERITY: CRITICAL / IMPORTANT / NICE-TO-HAVE
AGREED BY: [which agents flagged this]
CURRENT: [what the code does now]
DESIRED: [what it should do]
FILES: [exact file paths and functions to change]
TESTS: [tests to add or update]
```

Rules for the plan:
- Changes as small and local as possible
- Order-building changes always make things safer, not riskier
- Risk checks can only be tightened, never loosened
- Every order-logic change must have a corresponding test

### Step 5: Self-Check

Before finalizing:
- [ ] No change breaks SPEC invariants
- [ ] Order-related changes always make things safer
- [ ] No new bare `except: pass` introduced
- [ ] P&L calculations still use correct multiplier
- [ ] Kill switch still works after changes
- [ ] EOD flatten still works after changes

---

## Output Format

```
## SPEC & INVARIANTS
[2-3 sentence strategy summary]
[Bullet list of invariants]

## CRITICAL ISSUES (must fix before live)
[Issues with dual-agent agreement]

## IMPORTANT ISSUES (fix before next deploy)
[Issues with dual-agent agreement]

## NICE-TO-HAVE
[Lower priority items]

## CHANGE PLAN
[Structured list of changes for implementation]

## SUGGESTED TESTS
[Tests that should exist or be updated]
```

---

## What This Skill Does NOT Do

- Does NOT directly rewrite code — produces a review and change plan only
- Does NOT accept single-agent opinions as final — requires dual agreement
- Does NOT skip order safety checks for "simple" changes
- Does NOT approve changes that weaken risk controls

---

## Integration with Other Skills

Reference this skill from any skill that produces code changes in trading systems:

- **verification-before-completion**: Run this review before claiming work is done
- **order-execution-integrity**: This skill reviews the patterns that skill implements
- **risk-management-gates**: This skill verifies risk gates are correctly implemented
- **ibkr-api-precautions**: Reviewed contracts should pass precaution checks
- **ibkr-contract-validation**: Order safety agent validates contract fields
- **ibkr-pacing-limits**: Robustness agent checks for rate limit handling
