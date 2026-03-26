# Changelog

## v1.2.0 (2026-03-25)

### New Skill
- **spy-vix-regime-trading** — Complete SPY/VIX institutional trading framework: volatility regime detection (4 regimes), Rule of 16 (expected daily move), gap classification with fill probabilities, 5-minute ORB protocol, institutional order flow analysis (absorption, iceberg, big trades), VIX futures term structure, RSI-VIX predictive signals, pin risk detection, VIX/SPY hedged strategy, volatility-adjusted position sizing

### New Iron Law
- **EVERY DECISION ACCOUNTS FOR VOLATILITY REGIME** — Position sizing, stop placement, and strategy selection all depend on current VIX level

---

## v1.1.0 (2026-03-25)

### New Skills (4)
- **pre-trade-validation** — Symbol validation, market hours, liquidity/spread checks before orders
- **signal-source-integration** — Discord/Telegram webhook ingestion, NLP alert parsing, source authentication
- **trade-audit-and-replay** — Immutable audit trail, bot-vs-broker comparison, replay engine for determinism
- **eod-liquidation** — Scheduled EOD flatten, walk-down limit orders for illiquid options, overnight gap prevention

### Enhanced Skills (5)
- **options-trading-safety** — Added 0DTE gamma explosion risk (gamma→infinity as tau→0, dealer positioning, Gamma Flip), credit spread risk management (multiple-of-credit stops, defined vs undefined risk)
- **strategy-signal-validation** — Added confluence gate (3+ non-correlated indicator families), collinearity detection (correlation > 0.85 = same signal)
- **broker-api-integration** — Added PDT compliance (day trade counter, cash vs margin T+1 settlement tracking)
- **trading-bot-architecture** — Added CEP engine pattern, broker adapter layer, FIX protocol integration
- **backtesting-before-live** — Added strategy determinism requirements, replay engine, strategy checksum

### New Reference Docs (2)
- **docs/tax-optimization-reference.md** — Trader Tax Status, Section 475(f) mark-to-market, wash sale rule
- **docs/infrastructure-guide.md** — VPS vs home, co-location, CPU contention, recommended specs

### New Iron Laws (3)
- **VALIDATE BEFORE SUBMITTING** — Symbol, market hours, liquidity, spread checked proactively
- **AUTHENTICATE ALL EXTERNAL SIGNALS** — HMAC validation, rate limiting, typed parsing
- **FLATTEN BEFORE CLOSE** — Scheduled EOD exit, never rely on broker OnEndOfDay events

---

## v1.0.0 (2026-03-25)

### Launch

25 skills for building reliable day trading bots, encoding hard-won lessons from production trading bot failures.

**Trading Domain Skills (18):**
- trading-bot-architecture — Event-driven design with single execution gateway
- database-safety-for-trading — Transaction isolation, freshness guarantees, idempotent writes
- async-reliability — done_callbacks, no silent failures, fail-closed defaults
- broker-api-integration — Circuit breakers, idempotent orders, broker abstraction
- order-execution-integrity — Unified dedup gate, partial fill tracking, slippage monitoring
- position-reconciliation — Continuous reconciliation, ghost detection, startup checks
- risk-management-gates — Fail-closed risk checks, position sizing, stop-loss enforcement
- trailing-stop-mechanics — Monotonic stops, high water mark, ratchet correctness
- kill-switch-and-circuit-breakers — Multi-level halt, independent watchdog, safe flattening
- market-data-pipeline — Freshness checks, no sentinel magic numbers, failover
- strategy-signal-validation — Confirmation bars, anti-whipsaw, ATR-normalized signals
- indicator-math-verification — Reference comparison testing, edge cases
- trading-config-management — Single source, startup validation, no magic numbers
- trading-monitoring-and-alerts — Simplified invariants, structured logging, alert tiers
- trading-tdd — 10 mandatory trading test categories
- backtesting-before-live — Walk-forward validation, realistic slippage
- paper-to-live-progression — Graduated stage gates with rollback
- options-trading-safety — DTE management, expiration handling, assignment detection

**Engineering Skills (7):**
- using-trading-bot-skills — Bootstrap meta-skill
- writing-skills — TDD for skill creation
- brainstorming — Design before code
- writing-plans — Detailed implementation plans
- systematic-debugging — Root cause investigation
- verification-before-completion — Evidence before claims
- subagent-driven-development — Parallel execution with review
