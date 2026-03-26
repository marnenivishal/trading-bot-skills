# Trading Bot Skills

### The Claude Code Plugin That Thinks Like a Battle-Scarred Trading Systems Engineer

> Install one plugin. Claude instantly knows every way a trading bot can fail —
> and how to prevent each one. Zero configuration.

## Why Trading Bot Skills?

### Knows Every Failure Mode
Claude understands ghost positions, transaction poisoning, stop-loss slippage, sentinel value traps, and 11 more failure classes. Because they all happened in production.

### Enforces Fail-Closed
Before any code can handle an error, Claude verifies it returns explicit REJECT, never None. It's a hard gate, not a suggestion.

### TDD with Trading-Specific Tests
10 mandatory test categories: partial fills, race conditions, fail-closed injection, slippage detection, dedup verification, reconciliation, config validation, math correctness, kill switch, and stale data handling. Every test written BEFORE implementation code.

### Makes You Never Repeat Mistakes
40 skills trigger automatically. Claude backtests before live, writes fail-closed code, adds done_callbacks to async tasks, and verifies before claiming done.

### MCP Broker Integration
Connect Claude directly to brokerage APIs via Model Context Protocol. Deterministic 5-step execution loop with human-in-the-loop approval. No hallucinated prices — every data point from live MCP tools.

## Quick Start

```bash
# Install from marketplace (when published)
/plugin install trading-bot-skills

# Or load locally for development
claude --plugin-dir /path/to/trading-bot-skills
```

## Try It Locally

### Option 1: One Session (CLI flag)

```bash
claude --plugin-dir /path/to/trading-bot-skills
```

Skills available for this session only.

### Option 2: Persistent (Dev Marketplace)

Add to `~/.claude/settings.json`:

```json
{
  "enabledPlugins": ["/path/to/trading-bot-skills"]
}
```

Skills available every session.

### Option 3: Headless (CI / Testing)

```bash
claude --plugin-dir /path/to/trading-bot-skills \
       --print "Build me a trading bot"
```

## How It Works

```
You say something
       │
       ▼
┌─────────────────┐
│ Might any skill  │──── No ───► Normal response
│ apply? (even 1%) │
└────────┬────────┘
         │ Yes
         ▼
┌─────────────────┐
│ Invoke Skill     │
│ tool             │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Follow skill     │
│ exactly          │
└─────────────────┘
```

Skills trigger automatically based on what you're doing. No manual invocation needed.

## Cheat Sheet

| When you say... | Claude automatically uses... |
|---|---|
| "Build a new trading bot" | brainstorming → trading-bot-architecture → writing-plans |
| "Add an EMA crossover strategy" | brainstorming → strategy-signal-validation → backtesting-before-live → trading-tdd |
| "Fix this ghost position bug" | systematic-debugging → position-reconciliation → trading-tdd |
| "Go live with this strategy" | paper-to-live-progression (HARD GATE) |
| "Place an order" | order-execution-integrity → risk-management-gates |
| "Add trailing stops" | trailing-stop-mechanics → indicator-math-verification → trading-tdd |
| "Set up market data" | market-data-pipeline |
| "Configure settings" | trading-config-management |
| "Add options trading" | options-trading-safety → risk-management-gates |
| "Debug this execution issue" | systematic-debugging → order-execution-integrity |
| "Ingest Discord/Telegram alerts" | signal-source-integration → strategy-signal-validation |
| "Flatten positions at EOD" | eod-liquidation |
| "Trade 0DTE options" | 0dte-risk-management → options-trading-safety |
| "Check PDT compliance" | broker-api-integration (PDT compliance) |
| "Set up trade audit trail" | trade-audit-and-replay |
| "Trade SPY with VIX regime" | spy-vix-regime-trading (ORB, gaps, Rule of 16, flow) |
| "Build a backtesting framework" | backtest-expert → strategy-optimizer |
| "Add sentiment analysis" | market-sentiment-analyst → confidence-thresholds |
| "Integrate broker via MCP" | mcp-broker-integration → broker-api-integration |
| "Scale to microservices" | distributed-trading-patterns → trading-bot-architecture |
| "Refactor legacy trading code" | characterization-testing → trading-tdd |
| "Calculate market gaps" | gap-calculation → indicator-math-verification |
| "Set up CLAUDE.md for trading" | claude-md-for-trading |
| "Add confidence gates" | confidence-thresholds → risk-management-gates |

## All 40 Skills

### Trading Domain Skills (32)

| Skill | What It Does |
|-------|-------------|
| **trading-bot-architecture** | Event-driven design with single execution gateway. No direct broker calls from strategy code. |
| **database-safety-for-trading** | Transaction isolation, freshness guarantees, idempotent writes. No shared transactions. |
| **async-reliability** | done_callbacks on every task, no except:pass, fail-closed defaults. |
| **broker-api-integration** | Circuit breakers, idempotent order IDs, broker abstraction layer. |
| **order-execution-integrity** | Unified dedup gate, partial fill tracking, slippage monitoring. |
| **position-reconciliation** | Continuous broker reconciliation, ghost detection, startup checks. |
| **risk-management-gates** | Fail-closed risk checks. Every error returns REJECT, never None. |
| **trailing-stop-mechanics** | Monotonic stops with high water mark. Mathematically proven correct. |
| **kill-switch-and-circuit-breakers** | Multi-level halt independent of main loop. Safe flattening. |
| **market-data-pipeline** | Freshness checks on every price. No magic number sentinels. |
| **strategy-signal-validation** | Confirmation bars, anti-whipsaw, ATR-normalized signals. |
| **indicator-math-verification** | Reference comparison testing for every indicator function. |
| **trading-config-management** | Single source of truth, validated at startup, no magic numbers. |
| **trading-monitoring-and-alerts** | 5-10 core invariants instead of 58 audit rules. Structured logging. |
| **trading-tdd** | 10 mandatory trading test categories. Tests before code. |
| **backtesting-before-live** | Walk-forward validation. Identical code for backtest and live. |
| **paper-to-live-progression** | Graduated gates: Backtest → Paper → Small Live → Full Live. |
| **options-trading-safety** | DTE management, expiration handling, 0DTE gamma gates, credit spread stops. |
| **pre-trade-validation** | Symbol exists, market open, liquidity check, spread check before orders. |
| **signal-source-integration** | Discord/Telegram webhook ingestion, NLP alert parsing, source auth. |
| **trade-audit-and-replay** | Immutable audit trail, bot-vs-broker comparison, replay engine. |
| **eod-liquidation** | Scheduled EOD flatten, walk-down limit orders, overnight gap prevention. |
| **spy-vix-regime-trading** | VIX regime detection, Rule of 16, gap trading, ORB, institutional flow, pin risk. |
| **0dte-risk-management** | 0DTE theta acceleration, liquidity degradation, auto-exit proximity rules, SPY/SPX settlement. |
| **confidence-thresholds** | Multi-model ensemble scoring, tiered confidence gates, veto system, confidence decay. |
| **gap-calculation** | Adjusted close prices, gap classification (institutional vs noise), volume validation. |
| **backtest-expert** | Hypothesis-driven backtesting, parameter robustness, walk-forward methodology, slippage models. |
| **strategy-optimizer** | Monte Carlo trade shuffling, noise injection, overfitting detection, parameter discipline. |
| **market-sentiment-analyst** | NLP news classification, social feed processing, Fed transcript analysis, institutional flow. |
| **mcp-broker-integration** | MCP server integration, 5-step deterministic execution loop, human-in-the-loop approval. |
| **distributed-trading-patterns** | Kafka/NATS streaming, actor model, microservices decomposition, distributed kill-switch. |

### Engineering Skills (8)

| Skill | What It Does |
|-------|-------------|
| **using-trading-bot-skills** | Bootstrap: ensures all skills auto-activate when relevant. |
| **writing-skills** | TDD applied to creating new skills for this plugin. |
| **brainstorming** | Design before code. Questions before answers. |
| **writing-plans** | Detailed plans with TDD steps, exact code, exact commands. |
| **systematic-debugging** | Root cause first. 4 phases. No quick fixes. |
| **verification-before-completion** | Evidence before claims. Run the command, read the output. |
| **subagent-driven-development** | Fresh subagent per task with two-stage review. |
| **characterization-testing** | Lock legacy behavior with characterization tests before refactoring. |
| **claude-md-for-trading** | CLAUDE.md constitution patterns, hierarchical config, auto-memory. |

| Skill | What It Does |
|-------|-------------|
| **using-trading-bot-skills** | Bootstrap: ensures all skills auto-activate when relevant. |
| **writing-skills** | TDD applied to creating new skills for this plugin. |
| **brainstorming** | Design before code. Questions before answers. |
| **writing-plans** | Detailed plans with TDD steps, exact code, exact commands. |
| **systematic-debugging** | Root cause first. 4 phases. No quick fixes. |
| **verification-before-completion** | Evidence before claims. Run the command, read the output. |
| **subagent-driven-development** | Fresh subagent per task with two-stage review. |

## The 11 Iron Laws

These are hard gates enforced by skills — not suggestions:

1. **FAIL-CLOSED, NEVER FAIL-OPEN** — Every error handler returns explicit REJECT, never None
2. **UNIFIED DEDUP GATE** — ALL entry paths check ONE function before ANY order
3. **EVERY ASYNC TASK HAS done_callback** — No fire-and-forget
4. **EVERY DB OPERATION IS TRANSACTION-SAFE** — No shared transactions for independent ops
5. **EVERY PRICE HAS A FRESHNESS CHECK** — Stale data = halt, not trade
6. **BACKTEST BEFORE PAPER, PAPER BEFORE LIVE** — Graduated progression with gates
7. **SINGLE SOURCE OF TRUTH FOR CONFIG** — One file, validated at startup
8. **NO MAGIC NUMBER SENTINELS** — Use typed Optional, not 99.0
9. **TRAILING STOPS ARE MONOTONIC** — Can only tighten, mathematically proven
10. **BROKER IS SOURCE OF TRUTH** — Track filled_qty, reconcile continuously
11. **NO CODE WITHOUT A FAILING TEST FIRST** — TDD with 10 trading-specific categories
12. **NO 0DTE WITHOUT AUTOMATED EXIT RULES** — Gamma is infinite at expiration
13. **NO TRADE WITHOUT CONFIDENCE THRESHOLD** — Tiered gates: 60-70% standard, 85% high-stakes
14. **EVERY GAP USES ADJUSTED CLOSE** — Unadjusted prices produce phantom gaps
15. **EVERY BACKTEST STARTS WITH A HYPOTHESIS** — "Does this make money?" is not a hypothesis
16. **10,000 RANDOM ORDERINGS OR OVERFIT** — Monte Carlo trade shuffling is mandatory
17. **SENTIMENT IS NEVER SOLE DECISION DRIVER** — Must pass validation gates like any signal
18. **MCP FOLLOWS 5-STEP DETERMINISTIC LOOP** — Discovery, Context, Preflight, Confirmation, Execution
19. **LOCK BEHAVIOR BEFORE REFACTORING** — Characterization tests before any changes

## Emabot Failure Coverage

Every skill exists because of a real production failure:

| Failure | What Happened | Preventing Skill |
|---------|--------------|-----------------|
| Transaction Poisoning | 9 positions unmanaged after DB crash | database-safety-for-trading |
| Duplicate Entries | SPY entered 11 times from same signal | order-execution-integrity |
| Ghost Positions | Broker positions with no local record | position-reconciliation |
| Silent Failures | Tasks dying without detection | async-reliability |
| Stale Audit Data | Old data masking real problems | database-safety-for-trading |
| Stop-Loss Slippage | 40% slippage undetected | risk-management-gates |
| Config Drift | Settings in 3 places, silently failing | trading-config-management |
| Trailing Stop Bugs | Stops loosening when should tighten | trailing-stop-mechanics |
| Cloud Flip Exits | Enter then immediately exit at loss | strategy-signal-validation |
| VIX Sentinel | Fake VIX=99 tightening all positions | market-data-pipeline |
| No Backtesting | Untested strategies with real money | backtesting-before-live |
| Task Invisibility | Async tasks failing silently | async-reliability |
| Fail-Open | Exceptions bypassing all safety gates | risk-management-gates |
| Complexity Explosion | 58 audit rules creating noise | trading-monitoring-and-alerts |
| EMA Curl Noise | False signals from 5-bar lookback | strategy-signal-validation |

## Tutorial: Your First Trading Bot Feature

```
You: "I want to add RSI divergence detection to my trading bot"

Claude: Using brainstorming skill to explore this before implementation.
        Let me understand the current project first...

        Question 1: What's your entry signal — are you looking for
        bullish divergence (price lower low, RSI higher low) as a
        reversal signal, or using RSI as confirmation for existing
        EMA cloud signals?

You: "Bullish divergence as a standalone entry signal"

Claude: [Asks more questions about timeframe, RSI period,
         confirmation requirements...]

        Here's the design: [presents architecture with signal
        validation, confirmation bars, fail-closed gates]

You: "Looks good, build it"

Claude: Using writing-plans skill to create implementation plan...

        [Creates plan with TDD steps for each component:
         - RSI calculation with reference test (vs pandas-ta)
         - Divergence detection with edge case tests
         - Signal validation with confirmation bars
         - Integration through unified dedup gate
         - Backtest requirement before paper trading]

You: "Execute it"

Claude: Using subagent-driven-development...
        [Dispatches fresh subagent per task, reviews between each]
```

## Tips

### Do
- Let skills guide you — they encode real failure prevention
- Trust the TDD process — every test category exists for a reason
- Run backtests before paper, paper before live
- Keep config in one place

### Don't
- Skip brainstorming ("it's simple") — simple projects have the most assumptions
- Bypass risk gates ("it's just paper trading") — paper trading validates your safety systems
- Write code before tests — if the test passes immediately, it tests nothing
- Use magic numbers for unavailable data — use typed Optional

### Power User Shortcuts
- "What skills do I have?" → shows categorized skill map
- "Onboard me" → role-based skill introduction
- "Which skill for X?" → decision tree to right skills
