---
name: writing-plans
description: Use when you have a spec or requirements for a multi-step trading bot task, before touching code
---

# Writing Plans

## Purpose

This skill converts approved design documents into detailed, step-by-step implementation plans. Every plan consists of bite-sized tasks (2-5 minutes each) with TDD steps baked into every task. No code is touched until the plan is written and reviewed.

## When to Use

- You have an approved design document (from the `brainstorming` skill).
- You have clear requirements for a multi-step task.
- You are about to start coding and need a structured approach.
- The work involves more than a single, obvious change.

## Plan Structure

### Plan Header

Every plan starts with a header that establishes context:

```markdown
# Implementation Plan: [Feature Name]

**Goal**: [One sentence: what this plan achieves when complete.]
**Architecture**: [Which components are affected. Reference design doc.]
**Tech Stack**: [Languages, frameworks, libraries involved.]
**Design Doc**: [Path to approved design document.]
**Estimated Tasks**: [Number of tasks.]
**Estimated Time**: [Total estimated time.]
```

### Task Format

Every task follows this exact format:

```markdown
## Task N: [Short descriptive title]

**Estimated time**: 2-5 min
**File(s)**: [Exact file paths that will be created or modified]

### Steps

1. **Write test**: [Exact test to write, referencing trading-tdd category if applicable]
   ```python
   # Exact test code or clear description of what the test asserts
   ```
2. **Verify test fails**: Run `pytest tests/path/test_file.py::TestClass::test_name -v` and confirm RED.
3. **Implement**: [Exact code to write or modify]
   ```python
   # Exact implementation code or clear description
   ```
4. **Verify test passes**: Run `pytest tests/path/test_file.py::TestClass::test_name -v` and confirm GREEN.
5. **Run full suite**: Run `pytest tests/ -v --tb=short` and confirm no regressions.
6. **Commit**: `git commit -m "feat: [description of what this task adds]"`

### Acceptance Criteria
- [ ] [Specific, verifiable criterion 1]
- [ ] [Specific, verifiable criterion 2]
```

### Task Size Rule

Every task MUST be 2-5 minutes of work. If a task is larger:
- Break it into smaller tasks.
- Each sub-task must independently pass its tests.
- Each sub-task must be independently committable.

If a task is smaller than 2 minutes, consider combining it with an adjacent task.

### TDD Steps in Every Task

Every task that modifies code MUST include these TDD steps:

1. **Write test** -- the test for this specific task.
2. **Verify RED** -- confirm the test fails before implementation.
3. **Implement** -- write the minimum code to pass the test.
4. **Verify GREEN** -- confirm the test passes.
5. **Run full suite** -- confirm no regressions in existing tests.
6. **Commit** -- atomic commit for this task.

Tasks that do not modify code (e.g., config file changes, documentation) may skip TDD steps but must still have verification steps.

## Plan Writing Process

### Step 1: Analyze the Design Document

Read the approved design document thoroughly. Identify:
- All components that need to be created or modified.
- All dependencies between components.
- The order in which components must be built (dependency graph).
- Which trading-tdd test categories apply to each component.

### Step 2: Decompose into Tasks

Break the work into 2-5 minute tasks. Rules:
- Tasks are ordered by dependency (build foundations first).
- Each task is independently testable and committable.
- Each task includes the test that validates it.
- Infrastructure/setup tasks come before feature tasks.
- Integration tests come after all unit-tested components are built.

### Step 3: Write the Plan

Write each task with the exact format above. Include:
- Exact file paths (not vague references).
- Exact test code or clear test descriptions.
- Exact implementation code or clear descriptions.
- Exact commands to run for verification.
- Trading-tdd category references where applicable.

### Step 4: Plan Review

Use the plan document reviewer subagent (`plan-document-reviewer-prompt.md` in this skill folder) to review the plan. The reviewer checks for:
- TDD steps in every code-modifying task.
- Complete, exact code (no pseudocode or "implement this somehow").
- Exact verification commands.
- Failure mode coverage (trading-tdd categories addressed).
- Correct task ordering (dependencies respected).
- Task size (2-5 minutes each).

If the reviewer finds issues, revise the plan and re-review.

### Step 5: User Review (Optional)

For complex or high-risk changes, present the plan to the user for review before execution.

### Step 6: Execution Handoff

The plan can be executed in two modes:

**Subagent-Driven (Recommended for independent tasks):**
- Use the `subagent-driven-development` skill.
- Each task dispatched to an implementer subagent.
- Results reviewed by quality reviewer subagent.
- Best for tasks that can be parallelized.

**Inline Execution (For sequential/dependent tasks):**
- Execute tasks sequentially in the current session.
- Follow each task's steps exactly as written.
- Do not skip TDD steps.
- Commit after each task.

## Trading Planning Considerations

Every plan for trading bot features MUST address these concerns. Include specific tasks for each applicable item.

### 1. Failure Mode Analysis Tasks

For every new component, include a task that:
- Writes fail-closed tests (trading-tdd Category 5).
- Verifies that exceptions return REJECT, not None.
- Tests every except block.

### 2. Fail-Closed Verification Tasks

Include explicit tasks to:
- Inject exceptions in every risk/validation code path.
- Verify REJECT behavior.
- Add tests for None return handling.

### 3. Dedup Check Tasks

If the feature involves signal processing or order submission:
- Include a task to verify all signals pass through the unified dedup gate.
- Write dedup tests (trading-tdd Category 10).
- Verify no new order path bypasses dedup.

### 4. Reconciliation Impact Tasks

If the feature affects position tracking:
- Include a task to verify reconciliation still works correctly.
- Write reconciliation tests (trading-tdd Category 6).
- Test for false positives and false negatives.

### 5. Backtest Requirement Tasks

If the feature changes strategy logic:
- Include a task to run backtests with the new logic.
- Verify backtest results meet minimum thresholds.
- Verify strategy code is identical between backtest and live.

### 6. Kill Switch Behavior Tasks

If the feature introduces new order paths or trading logic:
- Include a task to verify kill switch blocks the new path.
- Write kill switch tests (trading-tdd Category 9).

## Example Plan Skeleton

```markdown
# Implementation Plan: Add RSI-Based Signal Filter

**Goal**: Filter out buy signals when RSI > 70 (overbought) to reduce false entries.
**Architecture**: Modifies signal_processor.py, adds rsi_filter.py.
**Tech Stack**: Python, pytest, pandas.
**Design Doc**: docs/designs/rsi-filter.md
**Estimated Tasks**: 8
**Estimated Time**: 30 minutes

## Task 1: Add RSI calculation function with reference validation
**Estimated time**: 5 min
**File(s)**: src/indicators/rsi.py, tests/indicators/test_rsi.py
### Steps
1. Write test: RSI(14) on known price series matches pandas_ta reference (Category 8)
2. Verify RED
3. Implement RSI calculation
4. Verify GREEN
5. Full suite
6. Commit

## Task 2: Add insufficient-data handling for RSI
**Estimated time**: 3 min
**File(s)**: src/indicators/rsi.py, tests/indicators/test_rsi.py
### Steps
1. Write test: RSI with < 14 bars returns None (Category 3)
2. Verify RED
3. Add guard clause
4. Verify GREEN
5. Full suite
6. Commit

## Task 3: Add RSI filter with fail-closed behavior
**Estimated time**: 5 min
**File(s)**: src/filters/rsi_filter.py, tests/filters/test_rsi_filter.py
### Steps
1. Write test: Exception in RSI calculation returns REJECT (Category 5)
2. Verify RED
3. Implement filter with try/except -> REJECT
4. Verify GREEN
5. Full suite
6. Commit

## Task 4: Integrate RSI filter into signal processor
...

## Task 5: Add RSI filter config values with validation
...

## Task 6: Add stale price rejection in RSI filter
...

## Task 7: Verify dedup still works with RSI filter in pipeline
...

## Task 8: Integration test: full signal -> filter -> order path
...
```

## Red Flags -- Stop and Fix the Plan

- **Tasks without TDD steps**: Every code task needs write-test, verify-fail, implement, verify-pass.
- **Vague tasks**: "Implement the feature" is not a task. Specify exactly what code to write.
- **Tasks > 5 minutes**: Break them down further.
- **No trading-tdd categories referenced**: Which of the 10 categories apply? Call them out.
- **Missing fail-closed tasks**: If the feature has decision points, there must be fail-closed tests.
- **No verification commands**: Every task must specify the exact pytest command to run.
- **Missing commit step**: Every task ends with a commit. Atomic, testable progress.
- **Dependencies out of order**: Foundation tasks must come before tasks that depend on them.
