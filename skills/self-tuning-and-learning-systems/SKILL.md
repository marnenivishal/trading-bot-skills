---
name: self-tuning-and-learning-systems
description: "Use when implementing AI-driven strategy adaptation, building feedback loops that learn from trade outcomes, designing knowledge hierarchies for trading insights, or any system that modifies its own trading parameters based on past performance"
---

# Self-Tuning and Learning Systems for Trading

A trading bot that learns from its own trades can amplify its worst tendencies. If the system lost money on conservative entries, a naive learner concludes "be more aggressive" — creating a feedback loop that accelerates losses. Self-tuning requires poison detection, locked core parameters, human review gates, and separation of technique learning from parameter tuning.

---

## The Iron Law

> **A SYSTEM THAT LEARNS FROM ITS OWN BAD TRADES WILL AMPLIFY ITS WORST TENDENCIES. FEEDBACK LOOPS MUST HAVE POISON DETECTION, LOCKED CORE PARAMETERS, AND HUMAN REVIEW GATES.**
>
> Origin: emabot's Gemini learning system analyzed past trades and recommended parameter changes. But it learned from trades that were themselves products of bad parameters — reinforcing errors. It also learned from simulated/paper trades mixed with real ones, contaminating the training data. Commit ed49df2 broke the feedback loop. 17+ commits across the learning pipeline.

---

## Core Patterns

### Pattern 1: Three-Tier Knowledge Hierarchy

Not all learnings are equal. Separate proven knowledge from experimental insights.

```python
from enum import Enum

class KnowledgeTier(Enum):
    GOLD = "gold"      # Human-verified, backtested, locked
    SILVER = "silver"  # Backtested positive, auto-apply with bounds
    BRONZE = "bronze"  # AI-suggested, requires human review

class TradingInsight:
    def __init__(self, content: str, tier: KnowledgeTier, source: str):
        self.content = content
        self.tier = tier
        self.source = source  # 'human', 'backtest', 'ai_gemini', 'ai_claude'
        self.confidence: float = 0.0
        self.trades_validated: int = 0

    def can_auto_apply(self) -> bool:
        """Only SILVER tier with sufficient validation can auto-apply."""
        return (
            self.tier == KnowledgeTier.SILVER
            and self.confidence >= 0.65
            and self.trades_validated >= 30
        )

# GOLD: "Never enter 0DTE after 3:30 PM ET" — human rule, locked
# SILVER: "SPY calls perform 15% better when VIX < 18" — backtested
# BRONZE: "Gemini suggests tighter stops on Mondays" — needs review
```

### Pattern 2: Locked Settings (Never Auto-Modified)

```python
# Settings that the learning system can NEVER modify
LOCKED_SETTINGS = frozenset({
    "kill_switch",
    "paper_mode",
    "max_daily_loss_pct",
    "max_position_size",
    "stop_loss_min_pct",       # Floor for stop-loss
    "max_open_positions",
    "circuit_breaker_threshold",
    "emergency_close_enabled",
})

async def apply_tuning_proposal(proposal: dict) -> tuple[bool, str]:
    """Apply a tuning proposal with safety guards."""
    setting_name = proposal["setting"]

    # Hard gate: locked settings cannot be modified
    if setting_name in LOCKED_SETTINGS:
        return False, f"BLOCKED: '{setting_name}' is locked — cannot be auto-tuned"

    # Bounds check: every tunable setting has min/max
    bounds = SETTING_BOUNDS.get(setting_name)
    if bounds:
        value = proposal["new_value"]
        if not bounds["min"] <= value <= bounds["max"]:
            return False, (
                f"BLOCKED: {setting_name}={value} outside bounds "
                f"[{bounds['min']}, {bounds['max']}]"
            )

    # Apply with audit trail
    old_value = await db.get_setting(setting_name)
    await db.set_setting(setting_name, proposal["new_value"])
    await db.log_tuning_change(setting_name, old_value, proposal["new_value"],
                                proposal["source"], proposal["confidence"])
    return True, "applied"

SETTING_BOUNDS = {
    "options_stop_loss_pct": {"min": 5.0, "max": 50.0},
    "trailing_stop_pct": {"min": 2.0, "max": 30.0},
    "entry_confidence_threshold": {"min": 0.3, "max": 0.95},
    "max_contracts_per_ticker": {"min": 1, "max": 10},
}
```

### Pattern 3: Feedback Loop Poison Detection

```python
async def detect_feedback_poison(proposals: list[dict], recent_trades: list) -> list[dict]:
    """Filter out proposals that would have made recent losing trades WORSE."""
    safe_proposals = []

    for proposal in proposals:
        would_worsen = False

        # Check: would this change have increased losses on recent losers?
        losing_trades = [t for t in recent_trades if t["pnl"] < 0]
        for trade in losing_trades[-10:]:  # Check last 10 losers
            simulated_pnl = simulate_trade_with_new_param(
                trade, proposal["setting"], proposal["new_value"]
            )
            if simulated_pnl < trade["pnl"]:
                would_worsen = True
                break

        if would_worsen:
            logger.warning(
                f"POISON DETECTED: {proposal['setting']}={proposal['new_value']} "
                f"would worsen recent losing trades — rejected"
            )
        else:
            safe_proposals.append(proposal)

    return safe_proposals
```

### Pattern 4: Training Data Contamination Guard

```python
# BAD: Learn from ALL closed trades — includes simulated, AI-suggested, paper
trades = await db.fetch("SELECT * FROM trades WHERE status = 'CLOSED'")
insights = await gemini.analyze(trades)

# GOOD: Only learn from verified real trades with clean source tags
VALID_LEARNING_SOURCES = {"chat_signal", "flow_webhook", "manual"}
EXCLUDED_SOURCES = {"gemini_ai", "rahe_shadow", "paper", "simulated", "backtest"}

async def fetch_learning_trades(days: int = 30) -> list:
    """Fetch ONLY real trades for learning. Exclude simulated/AI-generated."""
    return await db.fetch("""
        SELECT * FROM trades
        WHERE status = 'CLOSED'
          AND source = ANY($1)
          AND closed_at >= NOW() - INTERVAL '%s days'
          AND pnl IS NOT NULL
    """, list(VALID_LEARNING_SOURCES), days)

# Also deduplicate: use DISTINCT ON content_hash to prevent double-weight
```

### Pattern 5: Multi-Model Consensus for Proposals

```python
async def validate_proposal_with_consensus(
    proposal: dict,
    models: list["LLMProvider"],
    min_agreement: int = 2,
) -> bool:
    """Require multiple AI models to agree before applying a change."""
    votes = []
    prompt = format_validation_prompt(proposal)

    for model in models:
        try:
            response = await model.generate(prompt)
            vote = extract_json(response.text)
            if vote and "approve" in vote:
                votes.append(vote["approve"])
        except Exception:
            votes.append(False)  # Failure = no vote

    approvals = sum(1 for v in votes if v)
    logger.info(
        f"Consensus vote for {proposal['setting']}: "
        f"{approvals}/{len(votes)} approve (need {min_agreement})"
    )
    return approvals >= min_agreement
```

### Pattern 6: Technique-Based vs Stats-Based Learning

```python
# BAD: Stats-based only — "average stop-loss should be 12.5%"
# Problem: Single number hides context (market regime, volatility, DTE)

# GOOD: Technique-based — "WHEN VIX < 18 AND DTE > 2, wider stops work better"
class TradingTechnique:
    def __init__(self):
        self.name: str = ""
        self.conditions: dict = {}    # When to apply
        self.action: str = ""         # What to do
        self.evidence: list = []      # Trades that validate this
        self.win_rate: float = 0.0
        self.sample_size: int = 0

# Learn techniques, not just numbers
techniques = [
    TradingTechnique(
        name="wide_stops_low_vix",
        conditions={"vix_below": 18, "dte_above": 2},
        action="Set stop_loss_pct to 20% (wider than default 15%)",
        win_rate=0.72,
        sample_size=45,
    ),
    TradingTechnique(
        name="tight_stops_high_vix",
        conditions={"vix_above": 25, "dte_below": 1},
        action="Set stop_loss_pct to 8% (tighter than default 15%)",
        win_rate=0.68,
        sample_size=38,
    ),
]
```

### Pattern 7: Auto-Rollback on Performance Degradation

```python
class TuningRollbackGuard:
    """Track applied changes and auto-revert if performance degrades."""

    def __init__(self, lookback_trades: int = 20, degradation_threshold: float = -0.10):
        self.applied_changes: list[dict] = []
        self.lookback = lookback_trades
        self.threshold = degradation_threshold

    async def check_and_rollback(self, db):
        """If win rate dropped >10% since last change, revert it."""
        if not self.applied_changes:
            return

        last_change = self.applied_changes[-1]
        recent_trades = await db.fetch_last_n_trades(self.lookback)

        if not recent_trades:
            return

        current_win_rate = sum(1 for t in recent_trades if t["pnl"] > 0) / len(recent_trades)
        baseline_win_rate = last_change.get("baseline_win_rate", 0.5)

        if current_win_rate < baseline_win_rate + self.threshold:
            logger.warning(
                f"Performance degraded after tuning {last_change['setting']}: "
                f"win rate {baseline_win_rate:.1%} → {current_win_rate:.1%}. "
                f"ROLLING BACK to {last_change['old_value']}"
            )
            await db.set_setting(last_change["setting"], last_change["old_value"])
            self.applied_changes.pop()
```

---

## Red Flags

| Red Flag | Why It's Dangerous | Correct Pattern |
|----------|-------------------|-----------------|
| Learning from AI-generated trades | Feedback loop — learning from own bad suggestions | Filter to `VALID_LEARNING_SOURCES` only |
| No locked settings list | Learning system can modify stop-loss minimum, kill switch | `LOCKED_SETTINGS` frozenset, checked before every apply |
| Single model drives parameter changes | One hallucination = bad tuning applied | Multi-model consensus (2+ must agree) |
| No poison detection | System recommends changes that worsen recent losers | Simulate proposal against last 10 losing trades |
| Stats-based learning only ("avg stop = 12.5%") | Hides context — wrong for different regimes | Technique-based: conditions + action + evidence |
| No rollback mechanism | Bad tuning persists until manually detected | Auto-rollback if win rate drops >10% after change |
| Learning from < 30 trades | Insufficient sample — noise mistaken for signal | Minimum 30 validated trades before any auto-apply |

---

## Integration

| Scenario | Related Skill |
|----------|--------------|
| LLM calls for analysis | `trading-bot-skills:llm-integration-for-trading-bots` |
| P&L data for learning | `trading-bot-skills:pnl-calculation-and-reconciliation` |
| Config persistence for tuned settings | `trading-bot-skills:trading-config-management` |
| Confidence scoring for proposals | `trading-bot-skills:confidence-thresholds` |
| Backtest validation of proposals | `trading-bot-skills:backtesting-before-live` |
| Audit trail for tuning changes | `trading-bot-skills:audit-trail-and-forensic-analysis` |
