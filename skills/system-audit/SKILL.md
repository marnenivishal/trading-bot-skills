---
name: system-audit
description: Use when auditing a trading system's codebase, running a multi-agent code review, requesting a comprehensive system health check, or when the developer says "audit"
---

# System Audit Orchestrator

Orchestrates a comprehensive, multi-agent audit of a trading system by
dispatching 6 specialist agents in parallel. Each agent reviews the codebase
from a different angle. Findings are only confirmed when 2+ agents independently
flag the same issue.

---

## The Iron Law

> **EVERY AUDIT FINDING MUST BE VALIDATED BY AT LEAST TWO INDEPENDENT
> SPECIALIST AGENTS BEFORE BEING REPORTED AS A CONFIRMED ISSUE.
> SINGLE-AGENT FINDINGS ARE FLAGGED AS UNVALIDATED HYPOTHESES.**
>
> Origin: Single-perspective code reviews produce false positives. An
> architecture concern that looks like a bug may be an intentional
> trade-off. A risk agent's objection may be mitigated by an execution
> safeguard the risk agent didn't see. Requiring cross-agent agreement
> filters noise and ensures findings reflect genuine systemic issues.

---

## When to Use

- Full system audit before deployment or after major refactors
- Periodic health check of a trading codebase
- Pre-live-trading review (complement to `paper-to-live-progression`)
- Developer says "audit", "review the system", "check everything"

## When NOT to Use

| Situation | Use Instead |
|-----------|-------------|
| Quick bug investigation | `systematic-debugging` |
| Adding runtime health-check rules | `audit-trail-and-forensic-analysis` |
| Trade log investigation / replay | `trade-audit-and-replay` |
| Single-file code review | `superpowers:requesting-code-review` |

---

## The 6 Specialist Agents

| # | Agent | Prefix | Focus | Conditional? |
|---|-------|--------|-------|-------------|
| 1 | Architecture & Data | ARCH | Data flow, contracts, state consistency, edge cases | Always |
| 2 | Trading Logic & Risk | RISK | Signal gating, risk controls, position sizing, exits | Always |
| 3 | Execution & Reliability | EXEC | Order API, latency, error handling, async safety | Always |
| 4 | LLM Interaction | LLM | Prompts, fallback, schema validation, cost controls | Skip if no LLM usage |
| 5 | Observability | OBS | Logging, metrics, traceability, alerting, replay | Always |
| 6 | UI/UX | UI | Dashboard usability, real-time accuracy, controls | Skip if no dashboard |

---

## Audit Process

### Step 1: Understand the System

Before dispatching any agents, the coordinator (you) must build context:

1. Read `CLAUDE.md`, project structure, config files, main entry points.
2. Produce a **System Understanding**: 5-10 bullet summary covering:
   - What the system trades and how (strategy type, instruments, timeframes).
   - Key components and data flow (signal source -> classification -> risk -> execution -> exit).
   - External dependencies (broker API, databases, caches, AI services).
   - UI/dashboard if present.
3. Identify **invariants** (things that must always be true):
   - "No trade without passing all risk gates."
   - "Global risk limits cannot be bypassed."
   - "All positions must be closed by EOD."
4. Determine which agents to dispatch:
   - **Always dispatch:** Architecture, Trading Logic, Execution, Observability.
   - **Conditionally dispatch:** LLM (if system uses LLM/AI classification), UI (if system has a dashboard).

### Step 2: Dispatch Agent Passes

Dispatch all applicable agents **in parallel** using the Agent tool. Each agent
receives its prompt from `agent-prompts.md`, the system understanding from
Step 1, and a scoped file list.

**File scoping per agent:**

| Agent | Scan Scope |
|-------|-----------|
| Architecture & Data | All modules, data models, config, DB schemas, state machines |
| Trading Logic & Risk | Signal handlers, risk gates, position sizing, exit logic, strategy code |
| Execution & Reliability | Broker API client, order execution, async tasks, error handling, reconnection |
| LLM Interaction | LLM client code, prompt templates, response parsing, fallback logic |
| Observability | Logging setup, metrics, audit trail, alert routing, dashboard data feeds |
| UI/UX | Dashboard code (Streamlit/React/etc.), API endpoints serving UI data |

**Agent dispatch template:**

```
Use the Agent tool for each specialist. Provide:

1. The agent's full prompt (copy from agent-prompts.md)
2. Replace {SYSTEM_UNDERSTANDING} with your Step 1 summary
3. Replace {FILE_LIST} with the scoped file paths for that agent
4. Set subagent_type to "general-purpose"
5. Dispatch all agents in a single message (parallel execution)
```

### Step 3: Cross-Agent Agreement

After all agents return their findings:

1. **Collect** all findings into a combined list.
2. **Match** findings across agents:
   - Same file + same category of concern = potential match.
   - Same root cause from different angles = match.
   - Be generous in matching — agents use different vocabulary for the same issue.
3. **Classify** each finding:
   - **VALIDATED**: 2+ agents flagged the same underlying issue. Use the highest severity any supporting agent assigned.
   - **DISPUTED**: One agent flagged it, another explicitly cleared the same area. Note both perspectives.
   - **SINGLE-AGENT**: Only one agent raised it. Flag as unvalidated hypothesis.
4. **Deduplicate**: Merge findings about the same root cause into one issue with a new `VAL-NNN` ID.

### Step 4: Synthesis

Produce the final report using the format in `output-template.md`:

1. **System Understanding** (from Step 1).
2. **Per-Agent Summary** (finding counts and key concerns per agent).
3. **Validated Issues Table** (ID, title, severity, supporting agents, impact).
4. **Detailed Validated Issues** (context, failure mode, code example, fix direction, relevant skill).
5. **Disputed/Single-Agent Issues** (optional but recommended).
6. **Recommended Next Steps**:
   - **Quick Wins** (< 1 hour, high impact) — prioritize these.
   - **Larger Refactors** (multi-day) — with effort estimate.
   - Reference the relevant `trading-bot-skills` for each fix direction.

---

## Targeted Audit Mode

For focused concerns, dispatch only 2-3 relevant agents instead of all 6:

| Concern | Agents to Dispatch |
|---------|-------------------|
| "Is my risk management solid?" | Trading Logic, Execution, Architecture |
| "Are there async bugs?" | Execution, Architecture, Observability |
| "Is the dashboard accurate?" | UI/UX, Observability, Architecture |
| "Is the LLM usage safe?" | LLM Interaction, Trading Logic, Execution |
| "Can I trace signals end-to-end?" | Observability, Architecture, Trading Logic |

The Iron Law still applies: 2+ agents must agree for a finding to be validated.

---

## Red Flags

| Red Flag | Why It's Dangerous | Correct Pattern |
|----------|-------------------|-----------------|
| Reporting single-agent findings as confirmed bugs | False positives erode trust and waste dev time | Require 2+ agent agreement; single-agent = hypothesis |
| Skipping agents to save time | Blind spots in uncovered domains miss critical issues | Run all applicable agents; use targeted mode for focus |
| No file scoping for agents | Agent overwhelmed by irrelevant code, shallow analysis | Scope files per agent domain (see table above) |
| Audit without system understanding first | Agents lack context, produce irrelevant or duplicate findings | Always complete Step 1 before dispatching |
| Findings without fix direction | Audit creates anxiety without actionable path forward | Every finding needs a concrete next step + relevant skill |
| Mixing runtime audit with dev-time audit | Conflates "is the code correct?" with "is the running system healthy?" | Use `audit-trail-and-forensic-analysis` for runtime checks |

---

## Integration

| Scenario | Related Skill |
|----------|--------------|
| Runtime audit rules (health checks during trading) | `audit-trail-and-forensic-analysis` |
| Trade-level logging and replay investigation | `trade-audit-and-replay` |
| Implementing fixes from audit findings | `subagent-driven-development` |
| Creating a plan from audit findings | `writing-plans` |
| Debugging a specific finding in depth | `systematic-debugging` |
| Risk control gaps found by audit | `risk-management-gates`, `kill-switch-and-circuit-breakers` |
| Async reliability issues found by audit | `async-reliability` |
| Execution integrity issues found | `order-execution-integrity` |
| Position tracking issues found | `position-reconciliation` |
| LLM integration issues found | `llm-integration-for-trading-bots` |

---

## References

- See `agent-prompts.md` for the full specialist agent prompts.
- See `output-template.md` for the complete output report format.
