---
name: claude-md-for-trading
description: Use when setting up a new trading bot project with Claude Code, configuring CLAUDE.md for trading development, or establishing project-level rules and persistent memory for trading systems
---

# CLAUDE.md for Trading Projects

## The Stateless Problem

LLMs start every session with a blank slate. Without CLAUDE.md, you repeat the same
instructions every session: "use asyncio", "always fail-closed", "never call broker
directly", "risk checks return RiskDecision, not None". Session after session. You
forget once, the agent writes synchronous broker calls, and you spend 30 minutes
unwinding the damage.

CLAUDE.md is the project's constitution. Claude Code loads it automatically before
any prompt. Every rule you put there applies to every session, every task, every
sub-agent. Write it once, enforce it forever.

Without CLAUDE.md, session 2 the agent writes synchronous `requests.get()` to the
broker. Session 3 it writes `except: pass` around a risk check. Session 4 you go live
with a silent risk bypass. With CLAUDE.md, every session the agent reads "ALL broker
calls use asyncio. NO except:pass anywhere." No drift. No forgetting.

---

## Hierarchical CLAUDE.md Setup

CLAUDE.md works in three layers. Each layer overrides the one above it. The most
specific file wins when rules conflict.

### Layer 1: Global (`~/.claude/CLAUDE.md`)

Your personal coding style. Applies to every project you work on with Claude Code.

```
# Global CLAUDE.md
- Prefer dataclasses over raw dicts for structured data
- Use type hints on all function signatures
- Use pathlib instead of os.path
- Prefer f-strings over .format()
- Write docstrings for public functions
```

This file should be short (20-40 lines). It captures YOUR style, not project rules.

### Layer 2: Project Root (`CLAUDE.md`)

Architecture rules, build commands, Iron Laws reference. This is where trading-specific
rules live. Every contributor and every agent session reads this file.

```
# Trading Bot — CLAUDE.md
## Architecture Rules
- NEVER call broker API directly from strategy code
- ALL orders go through ExecutionGateway
- ALL risk checks return RiskDecision, never None
```

This is the primary file. Most of your effort goes here.

### Layer 3: Directory-Level (`strategies/CLAUDE.md`)

Strategy-specific or module-specific constraints. Overrides project-level rules
for files within that directory.

```
# strategies/CLAUDE.md
- All strategies inherit from BaseStrategy
- Signal methods return SignalResult, never raw floats
- No direct imports from broker/ — use execution_gateway
- Each strategy file has a matching test file in tests/strategies/
```

### Override Rules

When rules conflict, the most specific file wins:

- `strategies/CLAUDE.md` overrides `CLAUDE.md` (project root)
- `CLAUDE.md` (project root) overrides `~/.claude/CLAUDE.md` (global)
- Within a file, later rules override earlier rules

Example: Global says "prefer dicts for simple data." Project says "ALL data structures
use dataclasses." In this project, dataclasses win.

---

## Trading-Specific CLAUDE.md Template

Copy this template into your project root as `CLAUDE.md` and customize it. Keep it
under 200 lines. Every line should be a rule the agent needs to follow, not
documentation it can look up elsewhere.

```
# Trading Bot — CLAUDE.md

## Project
[1-2 lines: what this bot does, what market, what broker]
Example: Intraday SPY/QQQ momentum bot using Alpaca API. Paper and live modes.

## Build & Test
pip install -e ".[dev]"
pytest tests/ -x --timeout=30
ruff check src/ tests/
mypy src/ --strict

## Architecture Rules
- NEVER call broker API directly from strategy code
- ALL orders go through ExecutionGateway
- ALL risk checks return RiskDecision, never None
- NO except:pass anywhere
- Strategy code NEVER imports from broker/ directly
- Every public function has type hints
- Use asyncio for all I/O operations
- Dataclasses for all structured data (no raw dicts)

## Iron Laws (reference trading-bot-skills plugin)
- Risk gate before every order — no bypass
- Fail closed on errors — reject, do not guess
- Position reconciliation every N minutes
- Kill switch accessible and tested
[List the Iron Laws applicable to your project]

## Risk Parameters
- Max position: $X per symbol
- Max daily loss: Y%
- Max concurrent positions: N
- Max order size: Z shares/contracts
- Staleness threshold: 30 seconds

## Forbidden Patterns
- No magic number sentinels (-1, 999999)
- No mutable default arguments
- No global state for position tracking
- No bare except clauses
- No print() for logging (use logging module)
- No time.sleep() in async code
- No hardcoded API URLs

## Required Before Merge
- All trading-tdd test categories pass
- Characterization tests for modified code
- Risk gate returns RiskDecision on all paths
- No new ruff or mypy violations
- Manual review of any risk-path changes
```

---

## Auto-Memory Integration

Claude Code maintains two files that work together. Understanding the boundary between
them prevents rule rot and information loss.

### CLAUDE.md = Rules (What To Do)

Prescriptive. Imperative voice. Stable across sessions. You write and maintain this file.

```
# CLAUDE.md
- Use Alpaca API for all broker interactions
- ALL orders go through ExecutionGateway
- Max position size: $5000 per symbol
```

### MEMORY.md = Learnings (What Was Discovered)

Descriptive. Past tense. Accumulated over sessions. Claude Code updates this file
as it discovers things about your project.

```
# MEMORY.md
- Alpaca rate limits to 200 req/min; we use 150 to stay safe
- The VIX feed from CBOE has a 15-min delay; use broker's VIX instead
- pytest-asyncio requires mode="auto" in pyproject.toml to avoid warnings
- The order fill callback sometimes fires twice for the same fill_id; dedup required
```

### The Boundary

| Aspect       | CLAUDE.md                        | MEMORY.md                              |
|------------- |--------------------------------- |--------------------------------------- |
| Content      | Rules and constraints            | Discoveries and learnings              |
| Voice        | Imperative ("NEVER", "ALWAYS")   | Descriptive ("discovered", "learned")  |
| Author       | You (human)                      | Claude Code (agent)                    |
| Updates      | When architecture changes        | Every session as needed                |
| Staleness    | Review monthly                   | Self-correcting over sessions          |

### What Goes Where

**Put in CLAUDE.md (rules):**
- "Use Alpaca API" — this is a decision, not a discovery
- "Max position $5000" — this is a constraint
- "No except:pass" — this is a coding standard

**Put in MEMORY.md (learnings):**
- "Alpaca rate limit is 200/min" — this was discovered during development
- "fill callback fires twice" — this is a runtime behavior learned the hard way
- "pytest needs mode=auto" — this is a tooling quirk

**Do NOT put learnings in CLAUDE.md.** They make the file grow unbounded and stale.
Rate limits change. Bugs get fixed. CLAUDE.md should not track moving targets.

**Do NOT put rules in MEMORY.md.** They get buried under accumulated learnings and
lose their authority.

---

## Best Practices

### Keep It Short

Under 200 lines. The agent reads the entire file before every task. Ruthlessly cut
anything that is not a rule the agent needs RIGHT NOW.

### Use Imperative Voice

- GOOD: "ALWAYS validate position size before order submission"
- BAD: "It would be nice to validate position sizes"
- GOOD: "NEVER use except:pass"
- BAD: "Try to avoid bare except clauses when possible"

### Include Exact Commands

- GOOD: `pytest tests/ -x --timeout=30 -q`
- BAD: "run the tests"
- GOOD: `ruff check src/ tests/ --fix`
- BAD: "lint the code"

### Update When Architecture Changes

Update CLAUDE.md in the same PR that changes architecture. Stale rules cause the
agent to fight your new code.

### Review Monthly

Read every line. Ask: "Is this still true?" Delete what no longer applies.

---

## Common Mistakes

### Too Long (>500 lines)

The agent loses signal in noise. Architecture docs, API references, and historical
notes bury the actual rules. **Fix:** Move documentation to `docs/`. Keep ONLY rules
in CLAUDE.md.

### Too Vague

"Write good code" tells the agent nothing. **Fix:** Replace with specific, testable
rules. "ALL functions have type hints." "NO function exceeds 50 lines."

### Duplicating Documentation

CLAUDE.md is rules, not docs. Do not paste your README into CLAUDE.md. **Fix:**
Reference other files. "See docs/architecture.md for system overview."

### Never Updating

A CLAUDE.md that says "use requests" when you switched to httpx 4 months ago causes
wrong code every session. **Fix:** Update CLAUDE.md in the same PR that changes
architecture.

### Conflicting Rules

"Use dicts for simplicity" on line 12 and "Use dataclasses for all data" on line 45.
The agent picks one. You do not know which. **Fix:** One rule per concept. Search
for contradictions before adding new rules.

---

## Red Flags

Watch for these signs that your CLAUDE.md needs attention:

- **You repeat instructions every session:** The instruction belongs in CLAUDE.md, not
  in your prompt. If you say it twice, write it down once.
- **The agent contradicts your architecture:** Your CLAUDE.md is stale or missing rules.
  Check what the agent did and add the missing constraint.
- **CLAUDE.md exceeds 200 lines:** Time to audit. Move docs out. Delete stale rules.
  Merge overlapping rules.
- **Agent ignores a rule:** The rule may be buried too deep, contradicted by another
  rule, or too vague to act on. Simplify and move it higher in the file.

---

## Integration Points

This skill works alongside other trading-bot-skills:

- **`using-trading-bot-skills`** — Covers how to invoke and combine skills. Your
  CLAUDE.md should reference which skills your project uses so the agent knows which
  patterns to follow.
- **`trading-config-management`** — Your CLAUDE.md should state WHERE config lives
  (single source of truth) and the config schema. The config management skill
  enforces HOW config is structured and validated.
- **`trading-tdd`** — Your CLAUDE.md "Required Before Merge" section should reference
  the mandatory test categories from trading-tdd. This ensures every merge
  checklist includes the right test coverage.
