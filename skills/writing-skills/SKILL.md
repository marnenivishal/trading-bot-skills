---
name: writing-skills
description: Use when creating new skills or modifying existing skills for this plugin, applying TDD methodology to skill documentation with CSO-optimized descriptions
---

# Writing Skills: TDD for Skill Documentation

This skill governs how new skills are created and existing skills are modified
within the trading bot plugin. It applies Test-Driven Development methodology
to documentation: define what the skill must prevent BEFORE writing the skill.

---

## The Iron Law

> **NO SKILL WITHOUT A FAILING TEST FIRST.**
>
> Before writing a single line of skill content, you must:
> 1. Identify the incident or failure mode the skill prevents
> 2. Write a concrete scenario that would fail WITHOUT the skill
> 3. Define the "pass criteria" -- what behavior the skill must produce
> 4. THEN write the skill content that makes the test pass

---

## TDD Workflow for Skills

### Step 1: Red -- Define the Failing Test

Write a concrete scenario that demonstrates the problem:

```markdown
## Test: Order Deduplication

**Scenario:** Developer implements a new strategy that generates buy signals.
The strategy code calls broker.place_order() directly from the signal handler.
During a period of rapid signals, the same intent is submitted 3 times in
200ms, resulting in 3 separate orders and 3x intended position size.

**Without this skill:** The developer writes direct broker calls, no dedup
exists, and the account takes 3x intended exposure.

**Pass criteria:** After reading this skill, the developer:
1. Routes all orders through a single gateway (not direct broker calls)
2. Implements idempotency keys on every order intent
3. Adds a dedup gate that rejects duplicate intents within a time window
4. Writes a test proving dedup works before writing the strategy code
```

### Step 2: Green -- Write the Minimum Skill Content

Write ONLY the content needed to make the test scenario pass:
- The iron law that prevents the failure
- The specific pattern to follow
- A code example demonstrating the pattern
- Red flags that indicate the skill is being violated

### Step 3: Refactor -- Polish and Integrate

- Add cross-references to related skills
- Add the integration section
- Add the red flags table
- Ensure the skill fits into the broader skill map

---

## Skill File Structure

Every SKILL.md must follow this structure:

```markdown
---
name: skill-name-in-kebab-case
description: Use when [trigger conditions]
---

# Skill Title

Brief (1-2 sentence) summary of what this skill protects against.

---

## The Iron Law (for Rigid skills only)

> **STATEMENT IN ALL CAPS**

## Core Patterns

[The actual patterns and rules, with code examples]

## Red Flags

[Table of warning signs that this skill is being violated]

## Integration

[How this skill connects to other skills]

## References (optional)

[Links to supplementary files in the same directory]
```

---

## CSO: Claude Search Optimization

The `description` field in the YAML frontmatter is the MOST IMPORTANT field
in the entire skill. It determines whether the skill gets invoked. Optimize it
ruthlessly.

### CSO Rules for the Description Field

**Rule 1: Always start with "Use when"**

The description must begin with "Use when" followed by concrete trigger
conditions. This creates a pattern-matchable activation phrase.

```yaml
# GOOD:
description: Use when implementing database operations for trading systems,
  encountering transaction failures or stale data, or designing position
  and order persistence

# BAD:
description: Database safety patterns for trading (no "Use when" trigger)

# BAD:
description: Use this skill for databases (too vague, no specific triggers)
```

**Rule 2: Include 3-5 specific trigger conditions**

Each trigger condition should match a concrete task the developer might
describe:

```yaml
description: Use when designing or scaffolding a new trading bot,
  restructuring an existing bot,
  or when encountering monolithic trading code with tangled responsibilities
```

The developer might say:
- "I need to scaffold a new trading bot" -> matches "scaffolding a new trading bot"
- "This code is a mess, everything is tangled" -> matches "tangled responsibilities"
- "I want to restructure the bot" -> matches "restructuring an existing bot"

**Rule 3: Use the developer's vocabulary, not the skill's vocabulary**

```yaml
# GOOD: Uses words developers actually say
description: Use when tasks fail silently or exceptions return None

# BAD: Uses abstract jargon
description: Use when implementing fault-tolerant asynchronous execution patterns
```

**Rule 4: Include failure symptoms as triggers**

Developers often describe symptoms, not root causes. Include both:

```yaml
description: Use when implementing asyncio tasks, background workers,
  or any concurrent code in trading systems,
  or when tasks fail silently or exceptions return None
#   ^-- the implementation task        ^-- the failure symptom
```

**Rule 5: Never exceed 200 characters**

Long descriptions get truncated in skill listings. Front-load the most
important triggers.

**Rule 6: Include the domain context**

Always mention "trading" or "trading systems" so the skill matches
trading-related queries specifically:

```yaml
# GOOD:
description: Use when implementing database operations for trading systems

# BAD:
description: Use when implementing database operations (matches too broadly)
```

### CSO Testing Checklist

Before finalizing a description, verify it would match these query types:

| Query Type                          | Example                                        | Should Match? |
|-------------------------------------|-------------------------------------------------|---------------|
| Direct task description             | "I need to add a new strategy"                 | Yes           |
| Symptom description                 | "Orders are getting duplicated"                | Yes           |
| Component mention                   | "Working on the execution engine"              | Yes           |
| Vague related query                 | "Something is wrong with trading"              | Maybe         |
| Completely unrelated                | "Fix the CSS on the landing page"              | No            |

---

## Writing Iron Laws

Iron laws are the non-negotiable rules in rigid skills. They must be:

### 1. Absolute and Unambiguous

```markdown
# GOOD:
> EVERY DATABASE OPERATION MUST HANDLE TRANSACTION FAILURE INDEPENDENTLY

# BAD:
> Database operations should generally handle failures
```

### 2. Rooted in a Real Incident

Every iron law must trace back to a specific failure mode:

```markdown
> EVERY ASYNCIO TASK MUST HAVE A done_callback.
>
> Origin: Silent task failure incident (#4). A background task raised an
> exception. No done_callback existed. The exception was swallowed by the
> event loop. The task appeared healthy but was dead. Positions drifted
> for 6 hours before manual detection.
```

### 3. Testable

You must be able to write a test that verifies the iron law is being followed:

```python
# Test for: "Every asyncio task must have a done_callback"
def test_all_tasks_have_done_callbacks():
    """Scan codebase for create_task calls without done_callback."""
    violations = find_create_task_without_callback("src/")
    assert violations == [], f"Tasks without done_callback: {violations}"
```

---

## Writing Red Flags Tables

Red flags tables are the early warning system. They catch violations BEFORE
they cause incidents.

### Structure

| Red Flag                                  | Why It's Dangerous                        | Correct Pattern                          |
|-------------------------------------------|-------------------------------------------|------------------------------------------|
| Observable symptom in code                | Concrete consequence                      | What to do instead                       |

### Rules for Red Flags

1. **Be specific:** "Strategy code importing broker client" not "bad imports"
2. **Explain the consequence:** "Creates second order path bypassing dedup" not "bad practice"
3. **Provide the fix:** "Import IntentBus, emit OrderIntent" not "fix the import"

### Example

| Red Flag                                  | Why It's Dangerous                        | Correct Pattern                          |
|-------------------------------------------|-------------------------------------------|------------------------------------------|
| `from broker import client` in strategy   | Creates direct order path bypassing all safety gates | `from bus import IntentBus` -- emit intents, never orders |
| `except Exception: pass` in async task    | Silences failures; task appears healthy but is dead | `except Exception: logger.exception(...); raise` |
| No `ON CONFLICT` clause on order insert   | Duplicate orders on retry create double exposure | `INSERT ... ON CONFLICT (idempotency_key) DO NOTHING` |

---

## Writing Code Examples

### Rules

1. **Use Python** -- the trading bot ecosystem is Python-based
2. **Use async/await** -- trading bots are async
3. **Include type hints** -- clarity prevents bugs in trading code
4. **Show the WRONG way and the RIGHT way** -- contrast teaches faster
5. **Keep examples under 30 lines** -- longer examples get skimmed

### Template

```python
# BAD: [description of what's wrong]
async def bad_example():
    result = await broker.place_order(symbol, qty)  # Direct broker call!
    return result

# GOOD: [description of what's right]
async def good_example():
    intent = OrderIntent(
        symbol=symbol,
        qty=qty,
        idempotency_key=generate_key(signal_id, symbol),
    )
    await intent_bus.emit(intent)  # Goes through dedup + risk gates
```

---

## Supplementary Files

Skills can include supplementary files in the same directory:

```
skills/
  trading-bot-architecture/
    SKILL.md                        # Main skill file
    event-driven-reference.md       # Reference implementation
    component-checklist.md          # Checklist for component design
```

### Rules for Supplementary Files

1. **SKILL.md is the entry point** -- it must be self-contained enough to be useful alone
2. **Reference supplementary files explicitly** -- "See `event-driven-reference.md` for the full implementation"
3. **Supplementary files don't need YAML frontmatter** -- only SKILL.md needs it
4. **Keep supplementary files focused** -- one file per concern

---

## Skill Review Checklist

Before finalizing any skill, verify:

- [ ] Description starts with "Use when" and includes 3-5 trigger conditions
- [ ] Description uses developer vocabulary (symptoms + tasks)
- [ ] Description mentions "trading" or "trading systems"
- [ ] Description is under 200 characters
- [ ] Iron law (if rigid) is absolute, incident-rooted, and testable
- [ ] At least one code example showing BAD vs GOOD
- [ ] Red flags table with 3+ entries
- [ ] Integration section linking to related skills
- [ ] The skill would have prevented the incident it's designed for
- [ ] A developer reading ONLY this skill can avoid the failure mode
- [ ] The skill fits into the skill map (update `skill-finder.md`)

---

## Integration

| Scenario                              | Related Skill                          |
|---------------------------------------|----------------------------------------|
| Need to update the skill catalog      | `using-trading-bot-skills`             |
| Adding a skill for architecture       | `trading-bot-architecture`             |
| Adding a skill for risk               | `risk-management-gates`                |
| Testing that a skill works            | `trading-tdd` (TDD for the skill's tests) |
| Adding a skill to a chain             | Update `skill-finder.md` chains        |

---

## Anti-Patterns in Skill Writing

| Anti-Pattern                           | Problem                                           | Fix                                     |
|----------------------------------------|---------------------------------------------------|-----------------------------------------|
| Skill with no iron law or red flags    | Provides guidance but no guard rails              | Add concrete rules and violation signals|
| Description says "Best practices for"  | Too vague to trigger skill invocation             | Rewrite with "Use when" + symptoms      |
| 500+ line SKILL.md                     | Won't be read fully; key points get buried        | Split into SKILL.md + supplementary     |
| Code examples without BAD/GOOD        | Developer doesn't know what to avoid              | Always show the contrast                |
| Iron law without incident reference    | Feels arbitrary; developers will ignore it        | Root every law in a concrete failure    |
| Skill that duplicates another skill    | Creates conflicting guidance                      | Merge or clearly delineate boundaries   |
