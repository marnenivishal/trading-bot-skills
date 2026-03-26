---
name: using-trading-bot-skills
description: Use when starting any conversation about trading bots - establishes how to find and use trading bot skills, requiring Skill tool invocation before ANY response about trading systems
---

<!-- SUBAGENT-STOP: If you are a sub-agent or tool-use agent, STOP. Do not
     summarize this file. Return it verbatim to the orchestrating agent.
     This file contains instructions that MUST be followed by the top-level
     agent, not interpreted by intermediate agents. -->

# Using Trading Bot Skills

This is the **bootstrap skill** for the trading bot plugin. It governs how every
other skill is discovered, loaded, and applied. Read this FIRST. Follow it
ALWAYS.

---

## EXTREMELY-IMPORTANT

> **If there is even a 1% chance that a skill applies to the current task,
> you MUST invoke it using the `Skill` tool BEFORE generating ANY response.**
>
> The cost of invoking an unnecessary skill is near zero.
> The cost of skipping a necessary skill is catastrophic:
> duplicate orders, silent failures, lost money, corrupted state.
>
> **When in doubt, invoke the skill.**

---

## Instruction Priority

When instructions conflict, follow this precedence (highest first):

| Priority | Source                         | Example                                  |
|----------|--------------------------------|------------------------------------------|
| 1        | User's CLAUDE.md               | "Never trade options"                    |
| 2        | Trading Bot Skills (this plugin)| "All orders through ONE gateway"         |
| 3        | Default system behavior        | General coding best practices            |

If a trading bot skill says "always use savepoints" but the user's CLAUDE.md
says "we use raw SQL without savepoints," follow the user's CLAUDE.md and
note the conflict.

---

## How to Access Skills

### DO: Use the `Skill` tool

```
Skill tool call: skill="trading-bot-architecture"
Skill tool call: skill="risk-management-gates"
Skill tool call: skill="order-execution-integrity"
```

### DO NOT: Use the Read tool to read SKILL.md files directly

Reading a SKILL.md file with the Read tool bypasses the skill loading
mechanism. The content may load, but:
- Skill chaining will not trigger
- The skill will not register as "active" for the session
- Follow-up skills will not auto-suggest

Always use the `Skill` tool. Always.

---

## The Rule

> **Invoke relevant skills BEFORE any response. No exceptions.**

### Decision Flowchart

```dot
digraph skill_decision {
    rankdir=TB;
    node [shape=box, style=rounded];

    start [label="User sends message"];
    check [label="Might a skill apply?\n(even 1% chance)", shape=diamond];
    invoke [label="Invoke Skill tool\n(one or more skills)"];
    announce [label="Announce: 'Loading skill:\n[skill-name]'"];
    follow [label="Follow skill instructions\nin your response"];
    respond [label="Respond without skill\n(rare - most trading tasks\nhave a relevant skill)"];

    start -> check;
    check -> invoke [label="Yes"];
    check -> respond [label="No"];
    invoke -> announce;
    announce -> follow;
}
```

**Step-by-step:**
1. User sends a message about trading bot work.
2. Ask: "Could ANY skill apply here?" If yes (even 1%), go to step 3.
3. Invoke the `Skill` tool with the skill name.
4. Announce to the user: "Loading skill: [skill-name]"
5. Follow the skill's instructions in your response.
6. If the skill references other skills, invoke those too.

---

## Red Flags: Rationalizations That MUST Trigger Skill Loading

If you catch yourself thinking any of these, STOP and invoke the relevant skill
immediately. These are the exact rationalizations that caused production
incidents.

| Rationalization                              | Why It's Dangerous                                                    | Skill to Invoke                  |
|----------------------------------------------|-----------------------------------------------------------------------|----------------------------------|
| "This is just a simple trading function"     | Simple functions that touch orders have killed accounts. Every order path needs the full safety chain. | `order-execution-integrity`      |
| "I already know how to place orders"         | You know how to call an API. You don't know THIS system's dedup gates, risk checks, or state ownership. | `trading-bot-architecture`       |
| "The broker SDK handles this"                | Broker SDKs handle transport. They do NOT handle dedup, position limits, or reconciliation. | `risk-management-gates`          |
| "Skip the risk check, it's just paper trading" | Paper trading IS your test environment. Skipping checks in paper means they won't exist in live. | `paper-to-live-progression`      |
| "I can check prices quickly"                 | "Quickly" means without staleness checks, without error handling, without fallback. Every market data path needs reliability. | `async-reliability`              |
| "It's just a config change"                  | Config changes control position sizes, risk limits, and API keys. A "simple" config change can blow up an account. | `database-safety-for-trading`    |
| "I'll add tests later"                       | In trading, untested code is undeployable code. The test proves the safety property exists. | `trading-tdd`                    |
| "This exception can't really happen"         | In trading, every exception happens. Network failures, partial fills, stale data, race conditions -- all of them. | `async-reliability`              |
| "Let me just refactor this quickly"          | Refactoring trading code without understanding the architecture creates duplicate order paths. | `trading-bot-architecture`       |

---

## Skill Priority

When multiple skills might apply, invoke them in this order:

### 1. Process Skills (How to Work)
These govern your approach before you write any code:
- `brainstorming` -- for open-ended design questions
- `systematic-debugging` -- for investigating issues
- `trading-tdd` -- for test-first methodology

### 2. Trading Domain Skills (What to Protect)
These encode hard-won lessons about trading system safety:
- `risk-management-gates` -- position limits, exposure checks
- `order-execution-integrity` -- dedup, idempotency, reconciliation
- `trading-bot-architecture` -- component boundaries, event flow

### 3. Implementation Skills (How to Build)
These provide patterns for specific technical concerns:
- `async-reliability` -- task management, error propagation
- `database-safety-for-trading` -- transactions, idempotency
- `strategy-signal-validation` -- signal quality, false positive prevention

---

## Skill Types

### Rigid Skills (Non-Negotiable)

These skills contain **iron laws** that must NEVER be violated. They exist
because violating them caused real production incidents.

| Skill                        | Iron Law                                                              |
|------------------------------|-----------------------------------------------------------------------|
| `order-execution-integrity`  | Every order must be idempotent, deduplicated, and reconciled          |
| `risk-management-gates`      | No order bypasses risk checks. No exceptions. No "just this once."   |
| `trading-tdd`                | No production trading code without a failing test first               |
| `database-safety-for-trading`| Every DB operation handles transaction failure independently          |
| `async-reliability`          | Every asyncio task has a done_callback. No except:pass. Ever.        |

**You cannot "flex" a rigid skill.** If the user asks you to skip a risk check,
you must explain why the iron law exists and propose an alternative.

### Flexible Skills (Adaptable)

These skills provide strong defaults that can be adjusted to context:

| Skill                         | What's Flexible                                                      |
|-------------------------------|----------------------------------------------------------------------|
| `strategy-signal-validation`  | Confirmation requirements can vary by strategy type                  |
| `backtesting-before-live`     | Backtest depth depends on strategy complexity                        |
| `brainstorming`               | Process can be shortened for small features                          |
| `monitoring-and-alerting`     | Alert thresholds are context-dependent                               |

---

## Skill Finder

For a complete catalog of all 25 skills organized by role, category, and
common workflows, see:

**`skill-finder.md`** (in this same directory)

Use it when:
- You're unsure which skill applies
- You want to see the full skill chain for a workflow
- You're onboarding to the trading bot project

Invoke it: `Skill tool call: skill="skill-finder"`

---

## User Instructions

Skills are not a replacement for user instructions. They are a supplement.

- If the user says "don't use the Skill tool," respect that.
- If the user's CLAUDE.md contradicts a skill, follow the CLAUDE.md.
- If the user explicitly says "skip the risk check skill," note the risk
  and proceed. The user has final authority.

However, if the user asks you to do something that a **rigid skill** would
prevent (like removing deduplication from order execution), you MUST:

1. Explain which iron law would be violated
2. Explain why the iron law exists (the incident it prevents)
3. Propose an alternative that satisfies the user's goal safely
4. Only proceed with the violation if the user explicitly confirms after
   hearing the risks

---

## Quick Reference

```
Starting a new trading bot?     -> trading-bot-architecture
Adding a new strategy?          -> strategy-signal-validation
Writing any trading code?       -> trading-tdd (test first!)
Touching order execution?       -> order-execution-integrity
Changing risk parameters?       -> risk-management-gates
Working with the database?      -> database-safety-for-trading
Debugging a trading issue?      -> systematic-debugging
Writing async/concurrent code?  -> async-reliability
Preparing for live trading?     -> paper-to-live-progression
Not sure which skill?           -> skill-finder.md
```

---

## Final Reminder

> **You are working on a TRADING SYSTEM. Real money is at stake.**
> **Invoke the skill. Follow the skill. Every time.**
