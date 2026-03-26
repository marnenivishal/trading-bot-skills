# Testing Trading Bot Skills

## Overview

This document covers how to test the trading-bot-skills plugin itself — verifying that skills load correctly, trigger on the right prompts, and produce the expected behavior.

## Quick Test (Local)

Run Claude Code with the plugin loaded locally:

```bash
claude --plugin-dir /path/to/trading-bot-skills --print "Build me a trading bot"
```

**Expected:** Claude should invoke `brainstorming` and `trading-bot-architecture` skills before any implementation.

## Test Categories

### 1. Skill Loading Tests (~2 min)

Verify all 25 skills have valid frontmatter:

```bash
# Count SKILL.md files
find skills -name "SKILL.md" | wc -l
# Expected: 25

# Verify all have name: field
grep -l "^name:" skills/*/SKILL.md | wc -l
# Expected: 25

# Verify all have description starting with "Use when"
grep -l 'description:.*Use when' skills/*/SKILL.md | wc -l
# Expected: 25 (or close — some use "You MUST use this")
```

### 2. Skill Triggering Tests (~10 min each)

Run Claude Code in headless mode and verify correct skills trigger:

| Prompt | Expected Skills |
|--------|----------------|
| "Build a new trading bot" | brainstorming, trading-bot-architecture |
| "Add an EMA crossover strategy" | brainstorming, strategy-signal-validation, backtesting-before-live |
| "Fix this ghost position bug" | systematic-debugging, position-reconciliation |
| "Place an order for AAPL" | order-execution-integrity, risk-management-gates |
| "Go live with this strategy" | paper-to-live-progression |
| "Add trailing stop to positions" | trailing-stop-mechanics, trading-tdd |
| "Set up market data feed" | market-data-pipeline |
| "Configure the bot settings" | trading-config-management |
| "Add options trading" | options-trading-safety, risk-management-gates |

### 3. Integration Tests (~30 min)

Full workflow tests that verify skill chains:

**New Bot Workflow:**
1. "Build a trading bot" -> brainstorming triggers
2. Design approved -> writing-plans triggers
3. Plan approved -> subagent-driven-development triggers
4. Implementation done -> verification-before-completion triggers

**Bug Fix Workflow:**
1. "Fix this duplicate order bug" -> systematic-debugging triggers
2. Root cause found -> trading-tdd triggers (write failing test)
3. Fix implemented -> verification-before-completion triggers

## Running Tests

### Headless Mode

```bash
claude --plugin-dir /path/to/trading-bot-skills \
       --print "Build me a simple EMA crossover trading bot" \
       > session-output.jsonl
```

### Analyzing Output

```bash
# Check which skills were invoked
grep -o 'trading-bot-skills:[a-z-]*' session-output.jsonl | sort -u

# Check token usage
python tests/analyze-token-usage.py session-output.jsonl
```

## Writing New Tests

When adding a new skill, create a test scenario:

1. **Define the trigger prompt** — what should cause this skill to activate
2. **Define expected behavior** — what the skill should make Claude do
3. **Run headless** — verify the skill triggers
4. **Check output** — verify Claude follows the skill's process

## Common Issues

### Skill Not Triggering
- Check the `description` field — it must contain keywords matching the user's prompt
- Run `grep -i "keyword" skills/*/SKILL.md` to verify
- Description should describe triggering conditions, NOT the skill's workflow

### Skill Triggering Incorrectly
- Description too broad — narrow the triggering conditions
- Overlapping descriptions — make each skill's trigger unique

### Session Hook Not Running
- Verify `hooks/hooks.json` is valid JSON
- Verify `hooks/session-start` is executable (`chmod +x`)
- Check bash is available on the system
