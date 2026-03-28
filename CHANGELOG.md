# Changelog

## v3.0.0 (2026-03-28)

### New Skills (12) — Emabot Deep Lessons Extraction

Comprehensive analysis of 611 commits from the emabot production trading bot, extracting hard-won lessons into reusable skills.

**Data Handling & Defensive Programming:**
- **falsy-zero-and-sentinel-values** — Python's `or 0` trap: VIX=0.0 treated as missing (13+ commits). Explicit None checks, fail-closed sentinels (99.0 for VIX, -99999.0 for equity).
- **timestamp-and-timezone-in-trading** — UTC storage, ET trading logic, ZoneInfo over pytz (25+ commits). DST transition bugs, `.dt` accessor crashes, Streamlit cache datetime deserialization.

**Signal Ingestion Pipeline:**
- **chrome-extension-signal-bridge** — Browser signal capture with IndexedDB offline queue (3 extension versions). MV3 service worker lifecycle, content-hash dedup, heartbeat, fallback chain.
- **chat-signal-parsing-and-dedup** — 3-layer dedup (browser→server→DB), signal tier routing (10+ commits). Username normalization, 5-min window hashing, unified cooldown across all engines.

**Architecture & Infrastructure:**
- **database-transaction-patterns** — InFailedSqlTransaction prevention (22+ commits). SAVEPOINT isolation, separate connections for risky sub-queries, UPDATE WHERE status guards, pool recovery.
- **docker-and-scraper-reliability** — Playwright Docker setup, Cloudflare circuit breaker. Scraper hung 30+ minutes undetected. Session persistence, memory limits, fallback chain.
- **multi-engine-coordination** — SPY entered 11x from 4 independent engines (24+ commits). Unified position table, cross-engine dedup, global cooldown, nanosecond client_order_id.

**Monitoring & Audit:**
- **pnl-calculation-and-reconciliation** — Broker equity as truth (28+ commits). Options x100 multiplier missing, 4-table UNION with dedup, partial fill cost basis, ghost position worst-case P&L.
- **audit-trail-and-forensic-analysis** — 66 audit rules grew organically causing alert fatigue. 5-zone model, severity tiers with CRITICAL veto, deterministic rules, broker-vs-DB forensic cross-checks.

**AI & Learning:**
- **llm-integration-for-trading-bots** — 129 Gemini JSON parsing failures (20+ commits). Safe extraction, provider fallback chain, model version pinning, cost tracking with budget circuit breaker.
- **self-tuning-and-learning-systems** — Feedback loop poison: AI recommended "be more aggressive" after losses (17+ commits). 3-tier knowledge hierarchy, locked parameters, consensus validation, auto-rollback.

**Dashboard & UI:**
- **streamlit-dashboard-patterns** — 15+ UI bugs from state management (9+ commits). 3-tier state model, live data without cache, widget clearing, datetime deserialization after cache.

### Updated Docs
- **docs/emabot-postmortem.md** — Expanded from 15 to 27 production failures with full coverage matrix
- **docs/feature-index.md** — Updated from 41 to 53 skills with new categories

### New Iron Laws (9)
- **NEVER USE `value or default` ON NUMERIC TRADING DATA** — Zero is valid
- **STORE UTC, TRADE IN ET, DISPLAY IN USER TZ** — No naive datetimes
- **DEDUP AT EVERY LAYER** — Browser, webhook, AND database
- **A FAILED QUERY POISONS THE ENTIRE TRANSACTION** — Use SAVEPOINT
- **BROKER EQUITY IS TRUTH** — DB P&L is approximation
- **ONE POSITION TABLE, ONE DEDUP GATE** — All engines check same state
- **LLM RESPONSE IS UNTRUSTED INPUT** — Parse defensively
- **FEEDBACK LOOPS NEED POISON DETECTION** — Lock core parameters
- **AUDIT RULES MUST BE DETERMINISTIC** — Max 10 per zone

---

## v2.0.0 (2026-03-25)

### New Skills (9)
- **0dte-risk-management** — 0DTE-specific mechanics: theta acceleration curves, liquidity degradation near expiration, auto-exit proximity rules ($0.50 to short strike), SPY physical vs SPX cash settlement, expected-value-per-minute framework, gamma hedging frequency
- **confidence-thresholds** — Multi-model ensemble scoring with tiered confidence gates (60-70% standard, 85% high-stakes), veto system (any model <30% vetoes), confidence decay for stale signals, noise filtering
- **gap-calculation** — Adjusted close prices for gap calculations (preventing phantom gaps from splits/dividends), volume multiplier validation (1.5x 20-period avg), institutional vs noise gap classification, gap fill probabilities
- **backtest-expert** — Professional backtesting methodology: hypothesis-driven testing, parameter robustness analysis, walk-forward implementation (rolling vs anchored), slippage models (fixed/percentage/volume-impact), look-ahead and survivorship bias detection
- **strategy-optimizer** — Monte Carlo stress testing: trade shuffling (10,000+ permutations), randomized exit testing, noise injection, overfitting detection (95th percentile drawdown comparison), parameter count discipline (10:1 trades-per-parameter)
- **market-sentiment-analyst** — NLP news classification (bullish/bearish/neutral), social feed processing with bot filtering, Fed transcript hawkish/dovish analysis, institutional flow detection (dark pool, options flow), sentiment decay half-life, anti-manipulation detection
- **mcp-broker-integration** — Model Context Protocol integration: 5-step deterministic execution loop (Discovery→Context→Preflight→Confirmation→Execution), Public.com and Alpaca MCP server patterns, human-in-the-loop approval tiers, MCP error handling
- **distributed-trading-patterns** — Scaling beyond single-process: Kafka/NATS event streaming, actor model concurrency, Redis/Hazelcast distributed state, microservices decomposition, distributed kill-switch, OpenTelemetry tracing
- **characterization-testing** — Michael Feathers' technique for legacy trading code: lock behavior before refactoring, 5-step characterization process, trading-specific adaptations for order flow and risk gates, incremental refactoring loop
- **claude-md-for-trading** — CLAUDE.md constitution patterns for trading projects, hierarchical config (global→project→directory), trading-specific template, auto-memory integration, best practices for <200 line project files

### Enhanced Skills (5)
- **risk-management-gates** — Added real-time exposure auditing (ExposureSnapshot, ExposureAwareRiskGate), ensemble confidence gate integration, sector concentration and correlation checks
- **options-trading-safety** — Added Integration section with cross-references to 0dte-risk-management, mcp-broker-integration (multileg orders), spy-vix-regime-trading, eod-liquidation
- **backtesting-before-live** — Added Integration section with cross-references to backtest-expert (methodology), strategy-optimizer (Monte Carlo), quantitative-metrics-reference
- **trading-bot-architecture** — Added "When to Scale" section with trigger points, cross-reference to distributed-trading-patterns, mcp-broker-integration
- **using-trading-bot-skills** — Updated skill count to 40, added 9 new skills to cheat sheet, rigid skills table, rationalization guards, and skill-finder decision trees

### New Reference Docs (3)
- **docs/quantitative-metrics-reference.md** — Canonical source for 7 essential metrics (Sharpe, MaxDD, Win Rate, Profit Factor, Sortino, Recovery Factor, Beta) with correct annualization formulas, common calculation errors, and confidence intervals
- **docs/mcp-configuration-guide.md** — Practical MCP server setup: claude_desktop_config.json patterns for Public.com and Alpaca, environment variable security, testing workflow, troubleshooting
- **docs/eda-reference-architecture.md** — Event-driven and microservices architecture reference: technology comparison (Kafka vs NATS vs Redis vs ZeroMQ), WebSocket vs polling, service decomposition, deployment patterns

### New Iron Laws (8)
- **NO 0DTE POSITION WITHOUT AUTOMATED EXIT RULES** — Gamma is infinite at expiration
- **NO TRADE WITHOUT MEETING CONFIDENCE THRESHOLD** — Tiered by risk level (60-85%)
- **EVERY GAP USES ADJUSTED CLOSE PRICES** — Unadjusted prices produce phantom gaps
- **EVERY BACKTEST STARTS WITH A WRITTEN HYPOTHESIS** — "Does this make money?" is not a hypothesis
- **10,000 RANDOM ORDERINGS OR IT'S OVERFIT** — Monte Carlo trade shuffling is mandatory
- **SENTIMENT IS NEVER A SOLE DECISION DRIVER** — Must pass through validation gates like any signal
- **MCP FOLLOWS THE 5-STEP DETERMINISTIC LOOP** — Discovery, Context, Preflight, Confirmation, Execution
- **LOCK BEHAVIOR BEFORE REFACTORING** — Characterization tests capture truth before any changes

### Version Fix
- Fixed version inconsistency: package.json, plugin.json, and marketplace.json were stuck at 1.0.0 while CHANGELOG was at 1.2.0. All now at 2.0.0.

---

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
