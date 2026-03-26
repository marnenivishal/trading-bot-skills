# Emabot Postmortem: 15 Production Failures

This document records every class of failure from the emabot trading bot project. Each failure is mapped to the skill(s) that prevent it. This is not history — it is a permanent reference for why each skill exists.

## The Bot

**CloudTrader Bot** — an automated intraday trading system that:
- Scraped signals from a private chatroom using Playwright
- Validated signals using Ripster-style EMA cloud analysis (3 exponential moving averages)
- Executed equity bracket orders and options trades via Alpaca broker API
- Managed positions with trailing stops, partial exits, and cost-basis recovery

**Tech stack:** Python 3.11+, asyncio, alpaca-py, pandas/numpy, PostgreSQL, Redis, Streamlit, FastAPI, Docker Compose

**Outcome:** Failed in production due to execution reliability bugs, not strategy failures.

---

## Failure #1: Database Transaction Poisoning

**What happened:** PostgreSQL transactions were "poisoned" by failed sub-queries. Dedup queries and timezone operations inside reconciliation transactions could fail, but `except: pass` swallowed errors. ALL subsequent queries in that transaction failed with `InFailedSqlTransaction`.

**Real impact:** Bot crashed at 9:19 AM with 9 broker positions having ZERO DB records. No stops, no management.

**Root cause:** Independent database operations nested in shared transactions without SAVEPOINT isolation.

**Preventing skill:** `trading-bot-skills:database-safety-for-trading`

---

## Failure #2: Duplicate Entry Attacks

**What happened:** The same ticker could be entered 8+ times because 4 independent entry paths each had their own cooldown dict:
- Cloud event path: `_options_entry_cooldown`
- Chat learning path: `_signal_cooldown`
- Chat signal path: DB `get_recent_signal`
- Flow webhook path: atomic DB dedup

**Real impact:** SPY entered 11 times in one session from the same signal.

**Root cause:** No unified dedup gate. Each entry path maintained its own independent state.

**Preventing skills:** `trading-bot-skills:order-execution-integrity`, `trading-bot-skills:trading-bot-architecture`

---

## Failure #3: Ghost Positions

**What happened:** Multiple sources:
1. Partial fills tracked by `requested_qty` instead of `filled_qty`
2. Cancel-race window: position placed but cancel failed, leaving position unrecorded
3. Options expiring to $0 force-closed without grace period for data confirmation
4. Reconciliation producing negative quantities

**Real impact:** Broker positions with no DB record — no management, no trailing stops, no kill switch.

**Preventing skills:** `trading-bot-skills:position-reconciliation`, `trading-bot-skills:options-trading-safety`

---

## Failure #4: Silent Failures

**What happened:** `except: pass` blocks, failed DB queries with no visibility, asyncio tasks failing without `done_callback`. The system appeared healthy while silently losing money.

**Real impact:** Management loops hung for 5+ minutes without detection. Scheduler jobs failed to run without notification.

**Preventing skills:** `trading-bot-skills:async-reliability`, `trading-bot-skills:trading-monitoring-and-alerts`

---

## Failure #5: Stale Audit Data

**What happened:** `fetch_recent_trades()` returned last 500 rows by ID with NO date filter. Same 14 zero-entry trades showed in audit every day, same 56 contract mismatches repeating.

**Real impact:** Audit rules couldn't detect real problems — signal overwhelmed by noise from stale data.

**Preventing skills:** `trading-bot-skills:database-safety-for-trading`, `trading-bot-skills:market-data-pipeline`

---

## Failure #6: Stop-Loss Slippage

**What happened:** Stop set at $2.00 but filled at $1.20 (40% slippage). The audit system only checked entry slippage, not exit slippage.

**Real impact:** Significant unexpected losses on exits that should have been controlled.

**Preventing skills:** `trading-bot-skills:risk-management-gates`, `trading-bot-skills:order-execution-integrity`

---

## Failure #7: Configuration Drift

**What happened:** Settings existed in 3+ places: code defaults, `.env` file, database `runtime_settings` table. Runtime updates silently failed. Operators thought settings changed but they didn't.

**Real impact:** Bot running with unintended parameters. Config clamps missing (stop_loss had no min/max bounds).

**Preventing skill:** `trading-bot-skills:trading-config-management`

---

## Failure #8: Trailing Stop Logic Bugs

**What happened:** Ratchet guard used `max(new, prev)` — sounds correct but the implementation computed new stops from current price without maintaining a high water mark. When price declined, the "new" stop declined too, and `max()` kept the old one — BUT the logic also had paths where the old stop could be overwritten.

**Real impact:** Trailing stops that should only tighten were occasionally loosening, allowing larger losses.

**Preventing skills:** `trading-bot-skills:trailing-stop-mechanics`, `trading-bot-skills:indicator-math-verification`

---

## Failure #9: Cloud Flip False Exits

**What happened:** 15-minute EMA cloud was too thin (ema5 approximately equal to ema12). Signal entered on "green" cloud, but within minutes the cloud flipped, triggering an immediate exit at a loss.

**Real impact:** Rapid entry-exit cycles destroying capital through commissions and slippage.

**Preventing skill:** `trading-bot-skills:strategy-signal-validation`

---

## Failure #10: VIX Sentinel Misuse

**What happened:** When VIX data was unavailable, the system used `99.0` as a sentinel value. But the VIX gate treated this as VIX=99 (extreme fear), applying aggressive position tightening to all positions.

**Real impact:** Positions tightened and stopped out based on fake VIX data.

**Preventing skills:** `trading-bot-skills:market-data-pipeline`, `trading-bot-skills:trading-config-management`

---

## Failure #11: No Forward Backtesting

**What happened:** All analysis was post-trade. The hypothetical tracker, intratrade tracker, and opportunity assessor all analyzed what already happened. No strategy was validated with historical data before going live.

**Real impact:** Untested strategies running with real money. No baseline expectations to compare against.

**Preventing skills:** `trading-bot-skills:backtesting-before-live`, `trading-bot-skills:paper-to-live-progression`

---

## Failure #12: Async/Task Invisibility

**What happened:** `asyncio.create_task()` calls without `.add_done_callback()`. When tasks raised exceptions, they were silently garbage-collected. No one knew they failed.

**Real impact:** Critical background tasks (reconciliation, heartbeat, management loop) dying without detection.

**Preventing skill:** `trading-bot-skills:async-reliability`

---

## Failure #13: Fail-Open Defaults

**What happened:** Exception handlers returned `None` or empty values. Callers treated `None` as "no problem found" or "check passed". Result: exceptions bypassed all safety gates.

**Real impact:** Risk checks, VIX gates, and signal validation all failed open — the exact opposite of safe behavior.

**Preventing skills:** `trading-bot-skills:risk-management-gates`, `trading-bot-skills:async-reliability`

---

## Failure #14: Complexity Explosion

**What happened:** 58 audit rules across 5 zones. Each bug fix added new audit rules. Each new rule added new edge cases. The audit system became so complex that it masked real problems with noise.

**Real impact:** Operators couldn't distinguish real issues from audit noise. False alarms led to alert fatigue.

**Preventing skill:** `trading-bot-skills:trading-monitoring-and-alerts` (simplified to 5-10 core invariants)

---

## Failure #15: EMA Curl Detection Unreliable

**What happened:** 5-bar lookback on 5-minute bars (25 minutes of data) was too sensitive to price noise. The curl detection fired on random fluctuations, generating false signals.

**Real impact:** False entry signals leading to losing trades based on noise, not trends.

**Preventing skills:** `trading-bot-skills:strategy-signal-validation`, `trading-bot-skills:indicator-math-verification`

---

## Coverage Matrix

| Failure | Primary Skill | Secondary Skill |
|---------|--------------|-----------------|
| #1 Transaction Poisoning | database-safety-for-trading | — |
| #2 Duplicate Entries | order-execution-integrity | trading-bot-architecture |
| #3 Ghost Positions | position-reconciliation | options-trading-safety |
| #4 Silent Failures | async-reliability | trading-monitoring-and-alerts |
| #5 Stale Audit Data | database-safety-for-trading | market-data-pipeline |
| #6 Stop-Loss Slippage | risk-management-gates | order-execution-integrity |
| #7 Config Drift | trading-config-management | — |
| #8 Trailing Stop Bugs | trailing-stop-mechanics | indicator-math-verification |
| #9 Cloud Flip Exits | strategy-signal-validation | — |
| #10 VIX Sentinel | market-data-pipeline | trading-config-management |
| #11 No Backtesting | backtesting-before-live | paper-to-live-progression |
| #12 Task Invisibility | async-reliability | — |
| #13 Fail-Open | risk-management-gates | async-reliability |
| #14 Complexity | trading-monitoring-and-alerts | — |
| #15 EMA Curl | strategy-signal-validation | indicator-math-verification |

Every failure is covered. Most have defense in depth (two skills).
