# Feature Index

All 41 skills in the trading-bot-skills plugin, organized by category.

---

## Risk Management

| Skill | Description |
|---|---|
| [risk-management-gates](../skills/risk-management-gates/SKILL.md) | Position sizing, stop-losses, max drawdown limits, and pre-trade risk checks |
| [kill-switch-and-circuit-breakers](../skills/kill-switch-and-circuit-breakers/SKILL.md) | Emergency stop mechanisms, daily loss limits, error-rate circuit breakers |
| [0dte-risk-management](../skills/0dte-risk-management/SKILL.md) | Zero-days-to-expiration options, gamma risk, auto-exit rules near expiration |
| [trailing-stop-mechanics](../skills/trailing-stop-mechanics/SKILL.md) | Trailing stops, ratchet stops, dynamic stop-loss adjustment |
| [confidence-thresholds](../skills/confidence-thresholds/SKILL.md) | Multi-model signal validation, ensemble scoring, confidence-based trade gates |
| [eod-liquidation](../skills/eod-liquidation/SKILL.md) | End-of-day position flattening and overnight risk avoidance |

## Order Execution

| Skill | Description |
|---|---|
| [order-execution-integrity](../skills/order-execution-integrity/SKILL.md) | Fill handling, partial fills, dedup, ghost position prevention |
| [pre-trade-validation](../skills/pre-trade-validation/SKILL.md) | Pre-trade checks for symbols, market hours, liquidity |
| [broker-api-integration](../skills/broker-api-integration/SKILL.md) | Broker API patterns: timeouts, retries, circuit breakers, idempotency |
| [mcp-broker-integration](../skills/mcp-broker-integration/SKILL.md) | Claude + brokerage APIs via MCP with human-in-the-loop approval |

## Market Data

| Skill | Description |
|---|---|
| [market-data-pipeline](../skills/market-data-pipeline/SKILL.md) | Data feeds, price freshness checks, sentinel value handling |
| [signal-source-integration](../skills/signal-source-integration/SKILL.md) | Ingesting signals from Discord, Telegram, webhooks, alert services |
| [gap-calculation](../skills/gap-calculation/SKILL.md) | Market gap classification and gap-based trading strategies |
| [indicator-math-verification](../skills/indicator-math-verification/SKILL.md) | EMA, ATR, RSI, VWAP, Bollinger Bands, position sizing math |

## Strategy & Signals

| Skill | Description |
|---|---|
| [strategy-signal-validation](../skills/strategy-signal-validation/SKILL.md) | Signal quality: EMA crossovers, false entries, signal flipping |
| [spy-vix-regime-trading](../skills/spy-vix-regime-trading/SKILL.md) | SPY/VIX regime detection, gap trading, opening range breakout, Rule of 16 |
| [strategy-optimizer](../skills/strategy-optimizer/SKILL.md) | Monte Carlo stress testing, overfitting detection, parameter optimization |
| [market-sentiment-analyst](../skills/market-sentiment-analyst/SKILL.md) | News/social media NLP sentiment, institutional flow detection |
| [options-trading-safety](../skills/options-trading-safety/SKILL.md) | Options expiration, Greeks, DTE management |

## Testing & Validation

| Skill | Description |
|---|---|
| [test-driven-development](../skills/test-driven-development/SKILL.md) | TDD workflow: tests before implementation |
| [trading-tdd](../skills/trading-tdd/SKILL.md) | Trading-specific TDD: partial fills, race conditions, fail-closed injection |
| [backtesting-before-live](../skills/backtesting-before-live/SKILL.md) | Mandatory backtesting before paper or live deployment |
| [backtest-expert](../skills/backtest-expert/SKILL.md) | Professional backtesting frameworks, walk-forward validation, execution cost modeling |
| [characterization-testing](../skills/characterization-testing/SKILL.md) | Locking legacy behavior with tests before refactoring |
| [paper-to-live-progression](../skills/paper-to-live-progression/SKILL.md) | Readiness criteria for moving from paper to live trading |
| [verification-before-completion](../skills/verification-before-completion/SKILL.md) | Run verification commands before claiming work is done |

## Architecture & Infrastructure

| Skill | Description |
|---|---|
| [trading-bot-architecture](../skills/trading-bot-architecture/SKILL.md) | Bot scaffolding, restructuring monoliths, separation of concerns |
| [distributed-trading-patterns](../skills/distributed-trading-patterns/SKILL.md) | Kafka/NATS event streaming, microservices, multi-bot coordination |
| [database-safety-for-trading](../skills/database-safety-for-trading/SKILL.md) | Transaction safety, stale data prevention, position persistence |
| [trading-config-management](../skills/trading-config-management/SKILL.md) | Configuration across environments, preventing config drift |
| [async-reliability](../skills/async-reliability/SKILL.md) | Asyncio tasks, silent failures, concurrent code in trading systems |

## Monitoring & Audit

| Skill | Description |
|---|---|
| [trading-monitoring-and-alerts](../skills/trading-monitoring-and-alerts/SKILL.md) | Dashboards, alerting, health checks, audit trails |
| [trade-audit-and-replay](../skills/trade-audit-and-replay/SKILL.md) | Trade logging, replay validation, bot-vs-broker reconciliation |
| [position-reconciliation](../skills/position-reconciliation/SKILL.md) | Position tracking, local-vs-broker mismatch detection, EOD reconciliation |

## Workflow & Process

| Skill | Description |
|---|---|
| [brainstorming](../skills/brainstorming/SKILL.md) | Explore requirements and design before implementation |
| [writing-plans](../skills/writing-plans/SKILL.md) | Multi-step trading bot task planning from specs |
| [writing-skills](../skills/writing-skills/SKILL.md) | Creating and modifying skills for this plugin |
| [systematic-debugging](../skills/systematic-debugging/SKILL.md) | Root-cause analysis before proposing fixes |
| [subagent-driven-development](../skills/subagent-driven-development/SKILL.md) | Parallelizing independent implementation tasks with subagents |
| [using-trading-bot-skills](../skills/using-trading-bot-skills/SKILL.md) | How to discover and invoke trading bot skills |
| [claude-md-for-trading](../skills/claude-md-for-trading/SKILL.md) | Setting up CLAUDE.md for trading projects |
