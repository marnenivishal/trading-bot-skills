---
name: paper-to-live-progression
description: Use when preparing to move a trading strategy from paper trading to live, or when determining readiness for live deployment
---

# Paper-to-Live Progression

## Purpose

This skill defines the stage gates a trading strategy must pass through before risking real capital. Every gate has explicit, measurable criteria. No gate may be skipped. No amount of confidence or urgency justifies bypassing these gates.

## The Progression Pipeline

```
+------------+     +-----------+     +-------------+     +------------+
|  Backtest  | --> |   Paper   | --> |  Small Live | --> |  Full Live |
|  (6+ mo)   |     | (2+ wks)  |     |  (2+ wks)  |     |  (ongoing) |
+------------+     +-----------+     +-------------+     +------------+
      |                  |                  |                   |
   Gate 1             Gate 2            Gate 3              Continuous
   Metrics            Compare to        Scale-Up            Monitoring
   Pass?              Backtest?         Criteria?           & Health
```

Each gate is a one-way door. You only advance when ALL criteria are met. You go BACKWARD when any gate criteria are violated.

---

## Gate 1: Backtest to Paper

### Entry Criteria (ALL must be true)

- [ ] Backtest completed on 6+ months of historical data.
- [ ] Backtest includes volatility events and multiple market regimes.
- [ ] Sharpe ratio > 1.0 (annualized, after commissions and slippage).
- [ ] Max drawdown within acceptable risk tolerance (typically < 20%).
- [ ] Walk-forward validation completed; out-of-sample results within 50% of in-sample.
- [ ] Profit factor > 1.2.
- [ ] At least 30 trades for statistical significance.
- [ ] Strategy code is identical between backtest and live execution paths.
- [ ] Backtest report saved and versioned for future comparison.

### What Happens

- Strategy deployed to paper trading environment.
- All signals, orders, fills, and positions logged for analysis.
- Real-time metrics tracked and compared to backtest expectations.
- No real money at risk.

See `backtesting-before-live` skill for full backtest requirements.

---

## Gate 2: Paper to Small Live

### Duration Requirement

**Minimum 2 weeks of paper trading.** 4 weeks preferred. No shortcuts.

### Entry Criteria (ALL must be true)

**Performance:**
- [ ] Paper traded for minimum 2 continuous weeks.
- [ ] Paper Sharpe ratio within 20% of backtest Sharpe ratio.
- [ ] Paper win rate within 20% of backtest win rate.
- [ ] Paper max drawdown does not exceed backtest max drawdown by more than 50%.
- [ ] Paper average trade P&L within 30% of backtest average.

**System Health:**
- [ ] Zero kill switch activations during paper period.
- [ ] Zero position mismatches (reconciliation passing every cycle).
- [ ] Zero unhandled exceptions in logs.
- [ ] All async tasks healthy (done_callbacks firing, no silent failures).
- [ ] Data feeds stable throughout paper period (no extended outages).

**Infrastructure:**
- [ ] Kill switch tested and confirmed working (manually triggered during paper).
- [ ] Reconciliation tested and confirmed working.
- [ ] All monitoring dashboards operational.
- [ ] All alerting (Slack/email/SMS) tested and confirmed received.
- [ ] Logging captures every signal, order, fill, and position change.

### What "Within 20%" Means

Example: Backtest Sharpe = 1.5

| Paper Sharpe | Verdict | Action |
|---|---|---|
| >= 1.2 | PASS | Advance to Gate 3 |
| 1.0 to 1.2 | REVIEW | Investigate divergence, extend paper period |
| < 1.0 | FAIL | Back to backtest, diagnose root cause |

---

## Gate 3: Small Live to Full Live

### Small Live Sizing

**Start at 10% of intended full position size.** This limits maximum dollar loss while validating real-world execution quality.

### Scale-Up Schedule

| Period | Position Size | Condition to Advance |
|--------|--------------|---------------------|
| Weeks 1-2 | 10% of full size | All criteria below met |
| Week 3 | 25% of full size | All criteria still met |
| Week 4 | 50% of full size | All criteria still met |
| Week 5 | 75% of full size | All criteria still met |
| Week 6+ | 100% of full size | All criteria still met |

**Scale up by 25% per week IF AND ONLY IF all criteria continue to pass.** If any criterion fails at any size level, HALT and investigate before continuing.

### Criteria for Each Scale-Up Step (ALL must be true)

- [ ] No kill switch activations since last scale-up.
- [ ] No position mismatches with broker (reconciliation clean).
- [ ] Real slippage within 2x of backtest slippage assumption.
- [ ] Fill quality within expected range (no systematic adverse fills).
- [ ] P&L tracking matches broker statement.
- [ ] All async tasks healthy (no silent failures in task registry).
- [ ] All data feeds stable.
- [ ] All alerts and notifications working.
- [ ] Human operator comfortable with observed strategy behavior.

---

## Go/No-Go Checklist: Before First Real Dollar

This is the final checkpoint before ANY real money is at risk. Every box must be checked. A second person should review this checklist independently.

### Systems Verification

- [ ] Kill switch tested: triggers on max daily loss threshold.
- [ ] Kill switch tested: triggers on broker disconnect.
- [ ] Kill switch tested: manual activation works and halts all activity.
- [ ] Kill switch tested: blocks ALL new orders when active (even high-confidence signals).
- [ ] Reconciliation running: local positions match broker positions consistently.
- [ ] Reconciliation tested: deliberately introduced mismatch was detected and halted trading.
- [ ] Data feed monitoring: stale data detection verified (inject stale timestamp, confirm rejection).
- [ ] Alerting: Slack/email/SMS notifications tested and confirmed received by operator.
- [ ] Logging: every signal, order, fill, position change, and error captured.
- [ ] Error handling: exception injection tested, all paths are fail-closed (REJECT, not None).

### Strategy Verification

- [ ] Backtest report reviewed, saved, and versioned.
- [ ] Paper trading results reviewed and within 20% of backtest expectations.
- [ ] Maximum position size configured and validated (test order at max size on paper).
- [ ] Maximum daily loss configured and validated (test kill switch trigger).
- [ ] Order types correct for strategy (limit vs market vs stop).
- [ ] Slippage assumptions validated during paper trading.
- [ ] Dedup gate verified: duplicate signals produce single order.

### Operations Verification

- [ ] Trading account funded with intended starting capital.
- [ ] API key permissions verified: can trade, CANNOT withdraw funds.
- [ ] Initial position size set to 10% of full target.
- [ ] Monitoring dashboard accessible and showing live data.
- [ ] A human will be actively watching during the first live trading session.
- [ ] Rollback plan documented (see below) and accessible.
- [ ] Broker support contact information immediately accessible.
- [ ] First live session scheduled during regular market hours (not overnight, not holiday).

### Human Verification

- [ ] You understand the strategy logic and WHY it should be profitable.
- [ ] You understand the maximum possible loss (worst drawdown) and can survive it financially.
- [ ] You understand the maximum possible loss and can tolerate it emotionally without panic-closing.
- [ ] You have a written plan for what to do if things go wrong.
- [ ] A second person has reviewed the strategy, deployment plan, and this checklist.

---

## Rollback Protocol

### Rollback Triggers

Rollback is MANDATORY when ANY of these occur:

- Kill switch activation for any reason.
- Position mismatch detected by reconciliation.
- Performance diverging > 50% from backtest on any key metric.
- Unhandled exception in production logs.
- Data feed failure lasting longer than staleness threshold.
- Any behavior you do not understand or did not expect.
- Slippage consistently > 3x backtest assumption.

### Rollback Steps

```
1. HALT         -> Stop all new signal processing. Activate kill switch if needed.
2. ASSESS       -> Are positions safe to hold? Or must they be flattened?
3. FLATTEN      -> If unsafe, close all positions (manually if automated close is suspect).
4. INVESTIGATE  -> Review logs. Identify root cause. Do not guess.
5. CLASSIFY     -> Is this TRANSIENT or STRUCTURAL?
6. FIX          -> Apply fix and verify.
7. VALIDATE     -> Re-run through appropriate gate before resuming.
```

### Transient vs Structural Issues

**Transient** (broker hiccup, data blip, one-time network issue):
- Fix the immediate issue.
- Add monitoring/alerting to detect recurrence.
- Resume at current size with increased monitoring.
- Document the incident.

**Structural** (code bug, logic error, design flaw, missing test):
- Strategy goes ALL THE WAY BACK to paper trading.
- Fix the root cause.
- Add tests that would have caught this (reference trading-tdd categories).
- Re-pass Gate 2 before returning to live.
- Start live again at 10% size (Gate 3 from scratch).

Examples of structural issues:
- Code bug causing incorrect order sizing.
- Risk check that was fail-open instead of fail-closed.
- Missing dedup allowing duplicate orders.
- Strategy logic error that missed an edge case.
- Async task that failed silently (no done_callback).

---

## Ongoing Monitoring (Post Full-Live Deployment)

### Daily Checks

- [ ] P&L within 2 standard deviations of backtest expected daily P&L.
- [ ] No position mismatches (reconciliation clean).
- [ ] No kill switch activations.
- [ ] Slippage within normal range.
- [ ] All async tasks healthy.
- [ ] No unhandled exceptions in logs.

### Weekly Checks

- [ ] Rolling 5-day Sharpe within 30% of backtest Sharpe.
- [ ] Drawdown within expected bounds.
- [ ] Fill quality and execution latency stable.
- [ ] Review all alerts and warnings from the week.
- [ ] Check broker statements against internal P&L tracking.

### Monthly Checks

- [ ] Full performance review vs backtest expectations.
- [ ] Market regime assessment: has the regime changed?
- [ ] Strategy still appropriate for current market conditions?
- [ ] Parameter drift analysis: are current parameters still optimal?
- [ ] If parameter adjustment needed: go through backtest gate FIRST. No live-tuning.

---

## Red Flags -- Stop and Investigate

- **Skipping paper trading**: Backtest straight to live. NEVER acceptable.
- **Backtest to full position size**: No 10% start. Use the graduated scale.
- **No human watching first live session**: Someone MUST be present.
- **"It works in backtest, ship it"**: Paper trading exists for a reason. Use it.
- **Scaling up during a losing period**: Only scale when ALL criteria pass. Hope is not a strategy.
- **Ignoring small mismatches**: Small today, catastrophic tomorrow. Investigate immediately.
- **Resuming after rollback without root cause fix**: The problem WILL recur. Fix first.
- **Live-tuning parameters**: All parameter changes go through backtest first. No exceptions.
- **"Just this once" exceptions to any gate**: Gates exist because past failures demanded them. Respect them.

## Pre-Live Code Review Gate

Before progressing from paper to live, invoke the **trading-code-reviewer** skill for a full multi-agent review of all order construction, risk management, and error handling code. This is a mandatory gate — no code goes live without dual-agent agreement that order safety and risk controls are correct.
