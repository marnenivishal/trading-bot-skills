# System Audit Report Template

Use this template for the final audit report produced in Step 4 (Synthesis).

---

## 1. System Understanding

- [Bullet 1: What the system trades and how]
- [Bullet 2: Signal source and classification pipeline]
- [Bullet 3: Risk management approach]
- [Bullet 4: Execution mechanism (broker, order types)]
- [Bullet 5: Exit strategy (stops, targets, time-based)]
- [Bullet 6: State management (DB, Redis, in-memory)]
- [Bullet 7: External dependencies (APIs, AI services)]
- [Bullet 8: UI/Dashboard overview]
- [Bullet 9-10: Any other key aspects]

**Core Invariants:**
- [Invariant 1: e.g., "No trade without passing all risk gates"]
- [Invariant 2: e.g., "Global risk limits cannot be bypassed"]
- [Invariant 3: e.g., "All positions closed by EOD"]
- [Invariant 4+: additional invariants]

---

## 2. Per-Agent Findings Summary

### Architecture & Data Agent
- Findings: [N] (Critical: X, Warning: Y, Info: Z)
- Key concerns: [1-2 sentence summary of main themes]

### Trading Logic & Risk Agent
- Findings: [N] (Critical: X, Warning: Y, Info: Z)
- Key concerns: [1-2 sentence summary]

### Execution & Reliability Agent
- Findings: [N] (Critical: X, Warning: Y, Info: Z)
- Key concerns: [1-2 sentence summary]

### LLM Interaction Agent *(if dispatched)*
- Findings: [N] (Critical: X, Warning: Y, Info: Z)
- Key concerns: [1-2 sentence summary]

### Observability Agent
- Findings: [N] (Critical: X, Warning: Y, Info: Z)
- Key concerns: [1-2 sentence summary]

### UI/UX Agent *(if dispatched)*
- Findings: [N] (Critical: X, Warning: Y, Info: Z)
- Key concerns: [1-2 sentence summary]

---

## 3. Validated Issues Table

Issues confirmed by 2+ specialist agents.

| ID | Title | Severity | Supporting Agents | Impact |
|----|-------|----------|-------------------|--------|
| VAL-001 | [short title] | CRITICAL | RISK, EXEC | [one-line impact] |
| VAL-002 | [short title] | WARNING | ARCH, OBS | [one-line impact] |
| VAL-003 | [short title] | WARNING | RISK, ARCH | [one-line impact] |

---

## 4. Detailed Validated Issues

### VAL-001: [Title]

**Severity:** CRITICAL
**Supporting Agents:** RISK-3, EXEC-5
**Context:** [Where this appears in the system — modules, files, data flow]
**Failure Mode:** [What goes wrong, under what conditions]
**Example:**
```
[Code snippet, scenario, or input → wrong behavior demonstration]
```
**Fix Direction:** [Concrete change — what to add/modify/remove]
**Relevant Skill:** `trading-bot-skills:[skill-name]`

---

### VAL-002: [Title]

**Severity:** WARNING
**Supporting Agents:** ARCH-2, OBS-4
**Context:** [Where this appears]
**Failure Mode:** [What goes wrong]
**Example:**
```
[Code snippet or scenario]
```
**Fix Direction:** [What to change]
**Relevant Skill:** `trading-bot-skills:[skill-name]`

---

*(Repeat for each validated issue)*

---

## 5. Disputed Issues

Issues where agents disagreed. Listed for awareness.

| ID | Title | Raised By | Disputed By | Notes |
|----|-------|-----------|-------------|-------|
| DIS-001 | [title] | RISK-2 | ARCH (cleared) | [brief explanation of disagreement] |

---

## 6. Single-Agent Issues

Issues raised by only one agent. Unvalidated — treat as hypotheses worth
investigating but not confirmed bugs.

| ID | Title | Raised By | Severity | Notes |
|----|-------|-----------|----------|-------|
| SA-001 | [title] | EXEC-7 | WARNING | [why it might matter] |
| SA-002 | [title] | OBS-3 | INFO | [why it might matter] |

---

## 7. Recommended Next Steps

### Quick Wins (< 1 hour each, high impact)

1. **[Action]** — Fixes VAL-003. [One sentence on what to do.]
2. **[Action]** — Fixes VAL-005. [One sentence on what to do.]

### Larger Refactors

1. **[Action]** — Fixes VAL-001, VAL-002. Estimated effort: [S/M/L].
   [One sentence on approach.]
2. **[Action]** — Fixes VAL-004. Estimated effort: [S/M/L].
   [One sentence on approach.]

### Suggested Skill Invocations for Fixes

| Fix | Invoke Skill |
|-----|-------------|
| Risk gate gaps | `trading-bot-skills:risk-management-gates` |
| Async reliability | `trading-bot-skills:async-reliability` |
| Order execution | `trading-bot-skills:order-execution-integrity` |
| Position tracking | `trading-bot-skills:position-reconciliation` |
| Dashboard fixes | `trading-bot-skills:streamlit-dashboard-patterns` |
