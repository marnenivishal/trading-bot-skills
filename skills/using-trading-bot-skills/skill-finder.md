---
name: skill-finder
description: Use when unsure which trading bot skill to invoke, onboarding to the project, or needing to discover available skills and their relationships
---

# Trading Bot Skill Finder

Complete catalog and discovery guide for all trading bot skills.

---

## Role-Based Onboarding

### Trading Bot Developer

You build and maintain the core bot infrastructure: execution, connectivity,
state management, deployment.

**Start here:**
1. `trading-bot-architecture` -- understand the component boundaries
2. `order-execution-integrity` -- learn the order safety chain
3. `async-reliability` -- master task lifecycle management
4. `database-safety-for-trading` -- learn transaction safety patterns
5. `trading-tdd` -- adopt test-first for all trading code

**Your daily drivers:**
- `systematic-debugging` -- when things break
- `risk-management-gates` -- when touching risk parameters
- `monitoring-and-alerting` -- when adding observability
- `paper-to-live-progression` -- when promoting to production

### Quant / Strategy Developer

You design and implement trading strategies: signals, indicators, position
sizing, entry/exit logic.

**Start here:**
1. `strategy-signal-validation` -- signal quality framework
2. `backtesting-before-live` -- validation before deployment
3. `trading-bot-architecture` -- understand where strategies plug in
4. `risk-management-gates` -- know what constraints apply

**Your daily drivers:**
- `brainstorming` -- when designing new strategies
- `trading-tdd` -- when implementing strategy logic
- `market-data-integrity` -- when consuming price feeds
- `position-sizing-rules` -- when tuning allocation

### DevOps / Infrastructure

You manage deployment, monitoring, secrets, and operational reliability.

**Start here:**
1. `deployment-and-rollback` -- safe deployment patterns
2. `monitoring-and-alerting` -- observability setup
3. `secrets-and-api-key-management` -- credential safety
4. `paper-to-live-progression` -- environment promotion
5. `async-reliability` -- understand runtime failure modes

**Your daily drivers:**
- `systematic-debugging` -- when investigating incidents
- `database-safety-for-trading` -- when managing migrations
- `trading-bot-architecture` -- when scaling components
- `disaster-recovery` -- when planning for the worst

---

## Decision Tree

Use this when you have a specific task and need to find the right skill(s).

### "I'm building a new trading bot"
```
brainstorming
  -> trading-bot-architecture
    -> order-execution-integrity
    -> risk-management-gates
    -> database-safety-for-trading
    -> async-reliability
    -> trading-tdd (throughout)
```

### "I'm adding a new strategy"
```
strategy-signal-validation
  -> backtesting-before-live
    -> market-data-integrity
    -> position-sizing-rules
    -> risk-management-gates
    -> trading-tdd (throughout)
```

### "I'm debugging an execution issue"
```
systematic-debugging
  -> order-execution-integrity
    -> async-reliability
    -> database-safety-for-trading
    -> monitoring-and-alerting
```

### "I'm going live with a bot"
```
paper-to-live-progression
  -> risk-management-gates
    -> monitoring-and-alerting
    -> deployment-and-rollback
    -> disaster-recovery
    -> secrets-and-api-key-management
```

### "I'm fixing a data issue"
```
systematic-debugging
  -> database-safety-for-trading
    -> market-data-integrity
    -> async-reliability
```

### "I'm refactoring trading code"
```
characterization-testing (lock behavior first!)
  -> trading-bot-architecture
    -> order-execution-integrity
      -> trading-tdd (test existing behavior first)
      -> async-reliability
```

### "I'm building a backtesting framework"
```
backtest-expert
  -> strategy-optimizer (Monte Carlo validation)
    -> backtesting-before-live (go-live requirements)
    -> gap-calculation (if trading gaps)
```

### "I'm integrating a broker via MCP"
```
mcp-broker-integration
  -> broker-api-integration
    -> order-execution-integrity
    -> risk-management-gates
```

### "I'm trading 0DTE options"
```
0dte-risk-management
  -> options-trading-safety
    -> risk-management-gates
    -> eod-liquidation
```

### "I'm adding sentiment analysis"
```
market-sentiment-analyst
  -> confidence-thresholds
    -> strategy-signal-validation
    -> signal-source-integration
```

### "I'm scaling to multiple services"
```
distributed-trading-patterns
  -> trading-bot-architecture
    -> async-reliability
    -> kill-switch-and-circuit-breakers
```

### "I'm getting NaN / no data from IBKR"
```
ibkr-session-watchdog              -- triage: subscriptions, sessions, data lines, API
  -> ibkr-market-data-subscriptions -- if Layer 1 (subscription) is the problem
  -> ibkr-bot-architect             -- if Layer 2 (session/permissions) needs redesign
  -> ibkr-gateway-docker            -- if Layer 4 (Docker/connection) is the problem
  -> ibkr-api-edge-cases            -- if all 4 layers pass and it's actually a code issue
```

### "I'm building an IBKR trading bot"
```
ibkr-bot-architect                 -- architecture, permissions, prerequisites
  -> ibkr-market-data-subscriptions -- verify data entitlements
  -> ibkr-gateway-docker            -- Docker deployment setup
  -> ibkr-api-edge-cases            -- API patterns (order IDs, brackets, pacing)
  -> ibkr-risk-officer              -- pre-trade risk validation
  -> ibkr-session-watchdog          -- runtime data/session diagnostics
```

---

## Complete Skill Map

All 46 skills organized by category.

### Core Architecture (3 skills)

| Skill                        | Description                                                     | Type   |
|------------------------------|-----------------------------------------------------------------|--------|
| `trading-bot-architecture`   | Event-driven design, component boundaries, single entry point   | Rigid  |
| `order-execution-integrity`  | Dedup, idempotency, reconciliation for every order              | Rigid  |
| `risk-management-gates`      | Position limits, exposure checks, kill switches                 | Rigid  |

### Strategy & Signals (7 skills)

| Skill                        | Description                                                     | Type     |
|------------------------------|-----------------------------------------------------------------|----------|
| `strategy-signal-validation` | Signal quality, confirmation requirements, false positive gates | Flexible |
| `backtesting-before-live`    | Historical validation, walk-forward testing, regime detection   | Flexible |
| `market-data-integrity`      | Staleness detection, gap handling, feed failover                | Rigid    |
| `position-sizing-rules`      | Kelly criterion, volatility scaling, correlation adjustments    | Flexible |
| `backtest-expert`            | Professional backtesting methodology, hypothesis-driven testing | Rigid    |
| `strategy-optimizer`         | Monte Carlo stress testing, overfitting detection, trade shuffling | Rigid |
| `market-sentiment-analyst`   | NLP news classification, social feed processing, institutional flow | Flexible |

### Data & Persistence (3 skills)

| Skill                        | Description                                                     | Type   |
|------------------------------|-----------------------------------------------------------------|--------|
| `database-safety-for-trading`| Transaction isolation, idempotent writes, freshness guarantees  | Rigid  |
| `state-reconciliation`       | Broker-local state sync, conflict resolution, drift detection   | Rigid  |
| `audit-trail-design`         | Immutable logs, regulatory compliance, replay capability        | Rigid  |

### Async & Reliability (3 skills)

| Skill                        | Description                                                     | Type   |
|------------------------------|-----------------------------------------------------------------|--------|
| `async-reliability`          | Task lifecycle, error propagation, heartbeat patterns           | Rigid  |
| `websocket-lifecycle`        | Connection management, reconnection, message ordering           | Rigid  |
| `rate-limiting-and-throttle` | API rate limits, backoff strategies, queue management            | Flexible|

### Risk & Safety (6 skills)

| Skill                        | Description                                                     | Type   |
|------------------------------|-----------------------------------------------------------------|--------|
| `circuit-breaker-patterns`   | Automatic halt on anomalies, cascading failure prevention       | Rigid  |
| `paper-to-live-progression`  | Environment promotion gates, parity validation                  | Rigid  |
| `disaster-recovery`          | Position flattening, emergency procedures, data recovery        | Rigid  |
| `0dte-risk-management`       | 0DTE theta acceleration, liquidity degradation, auto-exit rules | Rigid  |
| `confidence-thresholds`      | Multi-model ensemble scoring, veto system, risk-tiered gates    | Rigid  |
| `gap-calculation`            | Adjusted close prices, gap classification, volume validation    | Rigid  |

### Operations & Deployment (3 skills)

| Skill                        | Description                                                     | Type     |
|------------------------------|-----------------------------------------------------------------|----------|
| `deployment-and-rollback`    | Zero-downtime deploys, rollback procedures, canary releases     | Rigid    |
| `monitoring-and-alerting`    | Metric collection, alert rules, dashboard design                | Flexible |
| `secrets-and-api-key-management` | Key rotation, vault patterns, environment isolation         | Rigid    |

### Process & Methodology (3 skills)

| Skill                        | Description                                                     | Type     |
|------------------------------|-----------------------------------------------------------------|----------|
| `trading-tdd`                | Test-first for trading code, safety property verification       | Rigid    |
| `systematic-debugging`       | Structured investigation, hypothesis-driven debugging           | Flexible |
| `brainstorming`              | Divergent design, option generation, decision matrices          | Flexible |

### Integration & External (4 skills)

| Skill                        | Description                                                     | Type     |
|------------------------------|-----------------------------------------------------------------|----------|
| `broker-api-integration`     | SDK wrapping, error mapping, partial fill handling              | Rigid    |
| `notification-systems`       | Alert delivery, escalation chains, dedup of notifications       | Flexible |
| `logging-for-trading`        | Structured logging, PII redaction, performance impact           | Flexible |
| `mcp-broker-integration`     | MCP server integration, deterministic execution loop, human-in-the-loop | Rigid |

### IBKR (Interactive Brokers) (6 skills)

| Skill                            | Description                                                     | Type     |
|----------------------------------|-----------------------------------------------------------------|----------|
| `ibkr-bot-architect`             | Bot design framework: permissions, sessions, API choices        | Flexible |
| `ibkr-market-data-subscriptions` | Subscription coverage, TWS-only vs API-enabled bundles          | Rigid    |
| `ibkr-gateway-docker`            | Docker deployment, port mapping, IBC config, ib_insync          | Rigid    |
| `ibkr-api-edge-cases`            | Order IDs, brackets, reconnection, historical data pacing       | Rigid    |
| `ibkr-risk-officer`              | Pre-trade gatekeeper: order size, permissions, options levels   | Rigid    |
| `ibkr-session-watchdog`          | Runtime triage: NaN quotes, session conflicts, data-line limits | Rigid    |

### Scaling & Distribution (1 skill)

| Skill                          | Description                                                     | Type     |
|--------------------------------|-----------------------------------------------------------------|----------|
| `distributed-trading-patterns` | Kafka/NATS event streaming, microservices, distributed kill-switch | Flexible |

### Testing & Quality (2 skills)

| Skill                        | Description                                                     | Type     |
|------------------------------|-----------------------------------------------------------------|----------|
| `characterization-testing`   | Lock legacy behavior before refactoring, Michael Feathers technique | Rigid |
| `claude-md-for-trading`      | CLAUDE.md constitution patterns, hierarchical config, auto-memory | Flexible |

---

## Skill Chains

Pre-built sequences for common workflows. Invoke skills in order.

### Chain: New Trading Bot (Full Lifecycle)

```
1. brainstorming              -- explore design space
2. trading-bot-architecture   -- establish component boundaries
3. database-safety-for-trading -- design persistence layer
4. async-reliability          -- set up task management
5. order-execution-integrity  -- build order safety chain
6. risk-management-gates      -- implement risk checks
7. monitoring-and-alerting    -- add observability
8. trading-tdd                -- (active throughout steps 2-7)
9. paper-to-live-progression  -- validate in paper
10. deployment-and-rollback   -- go live safely
```

### Chain: New Strategy

```
1. brainstorming              -- explore strategy ideas
2. strategy-signal-validation -- define signal quality requirements
3. backtesting-before-live    -- historical validation
4. position-sizing-rules      -- determine allocation
5. risk-management-gates      -- set strategy-specific limits
6. market-data-integrity      -- validate data feeds
7. trading-tdd                -- (active throughout steps 2-6)
8. paper-to-live-progression  -- paper trade the strategy
```

### Chain: Bug Fix

```
1. systematic-debugging       -- structured investigation
2. [domain skill]             -- skill for the affected component
3. trading-tdd                -- write failing test reproducing bug
4. [fix implementation]       -- guided by domain skill
5. monitoring-and-alerting    -- add detection for this class of bug
```

### Chain: Going Live

```
1. paper-to-live-progression  -- promotion checklist
2. risk-management-gates      -- verify all limits
3. state-reconciliation       -- verify broker-local parity
4. monitoring-and-alerting    -- verify dashboards and alerts
5. disaster-recovery          -- verify emergency procedures
6. secrets-and-api-key-management -- verify credential setup
7. deployment-and-rollback    -- execute deployment
```

### Chain: Incident Response

```
1. disaster-recovery          -- immediate safety (flatten if needed)
2. systematic-debugging       -- investigate root cause
3. [domain skill]             -- skill for the affected component
4. trading-tdd                -- write test proving the fix
5. monitoring-and-alerting    -- add alerting for this failure mode
6. audit-trail-design         -- verify incident is fully logged
```

---

## How to Use This File

1. **Find your role** in Role-Based Onboarding above.
2. **Follow the "Start here" sequence** to build foundational knowledge.
3. **Use the Decision Tree** when you have a specific task.
4. **Follow a Skill Chain** for multi-step workflows.
5. **Check the Skill Map** when you need to find a specific skill.

When invoking skills, always use the `Skill` tool:

```
Skill tool call: skill="trading-bot-architecture"
Skill tool call: skill="order-execution-integrity"
```

Never read skill files directly with the Read tool -- always use the Skill tool
to ensure proper skill loading and chaining.

---

## Cross-Reference: Which Skills Prevent Which Incidents

| Incident Type                    | Primary Skill                  | Supporting Skills                          |
|----------------------------------|--------------------------------|--------------------------------------------|
| Duplicate orders                 | `order-execution-integrity`    | `trading-bot-architecture`, `database-safety-for-trading` |
| Position limit breach            | `risk-management-gates`        | `state-reconciliation`, `circuit-breaker-patterns` |
| Silent task failure              | `async-reliability`            | `monitoring-and-alerting`, `logging-for-trading` |
| Stale market data                | `market-data-integrity`        | `websocket-lifecycle`, `circuit-breaker-patterns` |
| Transaction corruption           | `database-safety-for-trading`  | `state-reconciliation`, `audit-trail-design` |
| Strategy on bad signals          | `strategy-signal-validation`   | `backtesting-before-live`, `market-data-integrity` |
| Failed deployment                | `deployment-and-rollback`      | `monitoring-and-alerting`, `disaster-recovery` |
| Credential leak                  | `secrets-and-api-key-management` | `logging-for-trading`, `audit-trail-design` |
| Cascading failures               | `circuit-breaker-patterns`     | `async-reliability`, `disaster-recovery` |
| Paper-live behavior mismatch     | `paper-to-live-progression`    | `state-reconciliation`, `monitoring-and-alerting` |
| 0DTE assignment/pin risk         | `0dte-risk-management`         | `options-trading-safety`, `eod-liquidation` |
| Overfit strategy in production   | `strategy-optimizer`           | `backtest-expert`, `backtesting-before-live` |
| Low-confidence trade losses      | `confidence-thresholds`        | `strategy-signal-validation`, `risk-management-gates` |
| Phantom gaps from corporate actions | `gap-calculation`           | `market-data-pipeline`, `indicator-math-verification` |
| MCP tool hallucination           | `mcp-broker-integration`       | `broker-api-integration`, `order-execution-integrity` |
| Refactoring breaks trading logic | `characterization-testing`     | `trading-tdd`, `systematic-debugging` |
| Sentiment manipulation losses    | `market-sentiment-analyst`     | `confidence-thresholds`, `signal-source-integration` |
| NaN quotes / missing IBKR data   | `ibkr-session-watchdog`        | `ibkr-market-data-subscriptions`, `ibkr-bot-architect` |
| IBKR session conflicts            | `ibkr-session-watchdog`        | `ibkr-bot-architect`, `ibkr-gateway-docker` |
| IBKR API connection failures      | `ibkr-api-edge-cases`          | `ibkr-gateway-docker`, `ibkr-session-watchdog` |
