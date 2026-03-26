---
name: confidence-thresholds
description: Use when implementing multi-model signal validation, ensemble scoring for trade decisions, or when adding confidence-based gates to prevent low-conviction trades
---

# Confidence Thresholds

Iron Law: **"NO TRADE EXECUTES WITHOUT MEETING THE CONFIDENCE THRESHOLD FOR ITS RISK TIER."**

Every signal must carry a quantified confidence score. Every risk tier defines a minimum threshold. If the score falls below the threshold, the trade does not execute. There is no override. There is no manual bypass. The gate is fail-closed.

## 1. The Case for Confidence Gates

Win rate alone is insufficient for profitable trading. A strategy can have a 65% win rate and still lose money. Here is why.

**Case Study: The 65% Win Rate Trap**

Consider a momentum strategy that fires 200 signals per month with a 65% win rate. Sounds profitable. But when you segment by confidence:

- High-confidence signals (top 30%): 85% win rate, average gain +2.1R
- Medium-confidence signals (middle 40%): 68% win rate, average gain +0.8R
- Low-confidence signals (bottom 30%): 42% win rate, average loss -1.4R

The low-confidence bucket has *negative expectancy*. It drags the entire strategy underwater. The blended 65% win rate masks the fact that 30% of trades are destroyers. Removing the bottom bucket raises the strategy's net expectancy from -0.05R to +0.72R per trade.

**Why Ensemble Scoring Prevents Acting on Noise**

A single model can hallucinate conviction. Momentum says "strong buy" while volatility regime says "risk-off." Without ensemble scoring, the loudest signal wins. Ensemble scoring forces agreement across multiple independent lenses. When models agree, confidence is real. When they disagree, the system flags the conflict rather than gambling on one voice.

Confidence gates turn a mediocre strategy into a profitable one — not by finding better entries, but by refusing bad ones.

## 2. Risk Tier Thresholds

Not all trades carry the same risk. A 100-share SPY position is not the same as a 0DTE concentrated options play. The confidence required to act must scale with the risk.

### Tier Definitions

| Tier | Examples | Minimum Confidence | Rationale |
|------|----------|-------------------|-----------|
| Standard | Liquid equities, small positions, hedged trades | 60-70% | Normal operations, losses are bounded |
| Elevated | Large position size, low-liquidity names, earnings plays | 75% | Slippage risk, exit difficulty, gap risk |
| High-Stakes | 0DTE options, concentrated single-name, illiquid options | 85% | Binary outcome, total loss possible, no recovery window |

### Implementation

```python
from dataclasses import dataclass, field
from enum import Enum
from typing import Optional


class RiskTier(Enum):
    """Risk classification for trade candidates."""
    STANDARD = "standard"
    ELEVATED = "elevated"
    HIGH_STAKES = "high_stakes"


class RiskDecision(Enum):
    """Fail-closed decision outcomes."""
    APPROVED = "approved"
    REJECTED_BELOW_THRESHOLD = "rejected_below_threshold"
    REJECTED_MISSING_SCORE = "rejected_missing_score"
    REJECTED_STALE_SIGNAL = "rejected_stale_signal"
    REJECTED_VETO = "rejected_veto"
    REJECTED_CONFLICTED = "rejected_conflicted"


@dataclass(frozen=True)
class ThresholdConfig:
    """Confidence thresholds per risk tier. All values are percentages (0-100)."""
    standard_min: float = 65.0
    elevated_min: float = 75.0
    high_stakes_min: float = 85.0
    veto_floor: float = 30.0
    conflict_spread: float = 20.0

    def get_threshold(self, tier: RiskTier) -> float:
        thresholds = {
            RiskTier.STANDARD: self.standard_min,
            RiskTier.ELEVATED: self.elevated_min,
            RiskTier.HIGH_STAKES: self.high_stakes_min,
        }
        return thresholds[tier]


@dataclass
class ConfidenceDecision:
    """Result of a confidence gate evaluation. Fail-closed by default."""
    decision: RiskDecision
    effective_confidence: float
    tier: RiskTier
    threshold_required: float
    reason: str
    model_scores: dict = field(default_factory=dict)
    vetoed_by: Optional[str] = None

    @property
    def is_approved(self) -> bool:
        return self.decision == RiskDecision.APPROVED
```

## 3. Multi-Model Ensemble Scoring

No single model owns the truth. Confidence comes from agreement across independent signal sources, each weighted by its historical reliability.

### Weighted Ensemble Approach

Each model contributes a score between 0 and 100. Each model has a reliability weight derived from its historical accuracy. The ensemble score is the weighted average:

```
ensemble_score = sum(score_i * weight_i) / sum(weight_i)
```

Models with proven track records get higher weight. New or experimental models start with low weight until they earn credibility through backtested performance.

```python
from dataclasses import dataclass, field
from typing import Dict, List, Optional
import time


@dataclass
class ModelScore:
    """A confidence score from a single model."""
    model_name: str
    score: float  # 0-100
    timestamp: float = field(default_factory=time.time)
    metadata: dict = field(default_factory=dict)


@dataclass
class EnsembleResult:
    """Aggregated result from all models."""
    weighted_score: float
    individual_scores: Dict[str, float]
    spread: float  # max - min across models
    is_conflicted: bool
    contributing_models: int


class EnsembleScorer:
    """
    Combines confidence scores from N models using reliability-weighted averaging.

    Fail-closed: if no scores are provided, ensemble score is 0.
    """

    def __init__(self, model_weights: Dict[str, float]):
        """
        Args:
            model_weights: mapping of model_name -> reliability weight (0-1).
                           Higher weight = more influence on final score.
        """
        self._weights = model_weights

    def score(self, model_scores: List[ModelScore]) -> EnsembleResult:
        if not model_scores:
            return EnsembleResult(
                weighted_score=0.0,
                individual_scores={},
                spread=0.0,
                is_conflicted=False,
                contributing_models=0,
            )

        scores_map: Dict[str, float] = {}
        weighted_sum = 0.0
        total_weight = 0.0

        for ms in model_scores:
            weight = self._weights.get(ms.model_name, 0.1)  # unknown models get minimal weight
            weighted_sum += ms.score * weight
            total_weight += weight
            scores_map[ms.model_name] = ms.score

        # Fail-closed: avoid division by zero
        ensemble = weighted_sum / total_weight if total_weight > 0 else 0.0

        all_values = list(scores_map.values())
        spread = max(all_values) - min(all_values)
        is_conflicted = spread > 20.0  # configurable via ThresholdConfig

        return EnsembleResult(
            weighted_score=round(ensemble, 2),
            individual_scores=scores_map,
            spread=round(spread, 2),
            is_conflicted=is_conflicted,
            contributing_models=len(model_scores),
        )
```

## 4. Veto System

Ensemble averaging can hide fundamental disagreement. If one model screams "danger" while others are mildly bullish, the average might still look acceptable. The veto system prevents this.

### Veto Rules

1. **Absolute veto**: Any single model scoring below 30% vetoes the trade. Period. A model that sees major risk should not be overruled by consensus.
2. **Conflict downgrade**: If models disagree by more than 20% spread, the signal is "conflicted." Conflicted signals receive a 15% penalty to effective confidence, making them harder to pass tier thresholds.

```python
@dataclass
class VetoResult:
    """Outcome of veto gate evaluation."""
    vetoed: bool
    vetoed_by: Optional[str]
    is_conflicted: bool
    effective_confidence: float
    raw_confidence: float
    conflict_penalty_applied: float


class VetoGate:
    """
    Evaluates ensemble results for veto conditions.

    Fail-closed: any ambiguity results in rejection.
    """

    CONFLICT_PENALTY = 15.0  # percentage points deducted for conflicted signals

    def __init__(self, config: ThresholdConfig):
        self._config = config

    def evaluate(
        self, ensemble: EnsembleResult, tier: RiskTier
    ) -> ConfidenceDecision:
        threshold = self._config.get_threshold(tier)

        # Check for absolute veto: any model below the veto floor
        for model_name, score in ensemble.individual_scores.items():
            if score < self._config.veto_floor:
                return ConfidenceDecision(
                    decision=RiskDecision.REJECTED_VETO,
                    effective_confidence=ensemble.weighted_score,
                    tier=tier,
                    threshold_required=threshold,
                    reason=f"Model '{model_name}' scored {score}%, below veto floor {self._config.veto_floor}%",
                    model_scores=ensemble.individual_scores,
                    vetoed_by=model_name,
                )

        # Apply conflict penalty if models disagree
        penalty = 0.0
        effective = ensemble.weighted_score
        if ensemble.is_conflicted:
            penalty = self.CONFLICT_PENALTY
            effective = ensemble.weighted_score - penalty

        # Check threshold after any penalty
        if effective < threshold:
            decision = (
                RiskDecision.REJECTED_CONFLICTED
                if ensemble.is_conflicted
                else RiskDecision.REJECTED_BELOW_THRESHOLD
            )
            return ConfidenceDecision(
                decision=decision,
                effective_confidence=round(effective, 2),
                tier=tier,
                threshold_required=threshold,
                reason=f"Effective confidence {effective:.1f}% below {tier.value} threshold {threshold}%"
                + (f" (conflict penalty: -{penalty}%)" if penalty else ""),
                model_scores=ensemble.individual_scores,
            )

        return ConfidenceDecision(
            decision=RiskDecision.APPROVED,
            effective_confidence=round(effective, 2),
            tier=tier,
            threshold_required=threshold,
            reason=f"Passed {tier.value} threshold: {effective:.1f}% >= {threshold}%",
            model_scores=ensemble.individual_scores,
        )
```

## 5. Confidence Decay

Signals are perishable. A high-confidence momentum signal from 30 minutes ago is not the same signal now. Market conditions change. Confidence must decay over time.

### Half-Life Model

Confidence decays by 50% every N minutes, where N is configurable per signal type:

- Momentum/scalping signals: 5-minute half-life
- Swing/mean-reversion signals: 30-minute half-life
- Macro/regime signals: 240-minute half-life

```python
import math
from dataclasses import dataclass
from typing import Dict


@dataclass(frozen=True)
class DecayConfig:
    """Half-life configuration per signal type, in minutes."""
    default_half_life_minutes: float = 15.0
    signal_half_lives: Dict[str, float] = None

    def __post_init__(self):
        if self.signal_half_lives is None:
            object.__setattr__(self, "signal_half_lives", {
                "momentum": 5.0,
                "scalping": 5.0,
                "mean_reversion": 30.0,
                "swing": 30.0,
                "macro": 240.0,
                "regime": 240.0,
            })

    def get_half_life(self, signal_type: str) -> float:
        return self.signal_half_lives.get(signal_type, self.default_half_life_minutes)


class ConfidenceDecay:
    """
    Applies time-based decay to confidence scores using exponential half-life model.

    Formula: decayed = original * (0.5 ^ (elapsed_minutes / half_life))

    Fail-closed: if timestamp is missing or in the future, score decays to 0.
    """

    def __init__(self, config: DecayConfig = None):
        self._config = config or DecayConfig()

    def apply(
        self,
        original_score: float,
        signal_age_minutes: float,
        signal_type: str = "default",
    ) -> float:
        # Fail-closed: invalid age means no confidence
        if signal_age_minutes < 0:
            return 0.0

        half_life = self._config.get_half_life(signal_type)
        decay_factor = math.pow(0.5, signal_age_minutes / half_life)
        decayed = original_score * decay_factor

        return round(decayed, 2)

    def is_stale(
        self,
        signal_age_minutes: float,
        signal_type: str = "default",
        staleness_threshold: float = 5.0,
    ) -> bool:
        """A signal is stale if its decayed value would be below staleness_threshold
        even if it started at 100% confidence."""
        decayed_max = self.apply(100.0, signal_age_minutes, signal_type)
        return decayed_max < staleness_threshold
```

## 6. Noise Filtering

Signals below minimum threshold are not "blocked." They are *discarded entirely*. They do not appear in logs as "rejected candidates." They do not trigger alerts. They do not exist.

### Why Total Discard Matters

Alert fatigue kills trading systems. If the system logs every sub-threshold signal as a "rejected trade," operators start seeing hundreds of non-events per day. They stop reading the logs. When a real rejection happens — one that indicates a systemic problem — it gets buried.

### Noise Filter Rules

- Signals below 40% raw confidence: silently discarded, no log entry
- Signals between 40% and tier minimum: logged as "filtered" (debug level only)
- Signals at or above tier minimum: proceed to veto gate

```python
class NoiseFilter:
    """
    First-pass filter that discards sub-threshold signals before they enter
    the evaluation pipeline. Prevents alert fatigue.
    """

    DISCARD_FLOOR = 40.0  # below this, signal is not even logged

    def should_evaluate(self, raw_score: float) -> bool:
        """Returns True only if the signal warrants full evaluation."""
        return raw_score >= self.DISCARD_FLOOR

    def classify(self, raw_score: float, tier_threshold: float) -> str:
        if raw_score < self.DISCARD_FLOOR:
            return "discard"  # silent drop
        elif raw_score < tier_threshold:
            return "filtered"  # debug-level log only
        else:
            return "evaluate"  # proceed to ensemble + veto
```

## 7. Integration with Risk Gates

Confidence is one gate in a multi-gate pipeline. It has a specific position in the chain.

### Pipeline Order

```
Signal Generation
    |
    v
Signal Validation (is the signal technically valid?)
    |
    v
Confidence Gate (does the signal meet conviction thresholds?)  <-- THIS SKILL
    |
    v
Risk Gate (does the trade fit within portfolio risk limits?)
    |
    v
Position Sizing (how large should the position be?)
    |
    v
Order Execution
```

The confidence gate runs AFTER signal validation but BEFORE position sizing and risk management. This ordering matters:

1. **After validation** because there is no point scoring confidence on a malformed signal.
2. **Before risk gates** because risk gates are expensive (they check portfolio state, margin, correlation). Filtering low-confidence signals early saves computation and reduces noise in risk logs.
3. **Before position sizing** because confidence can influence size. A 90% confidence signal might warrant full allocation. A 70% signal (just above threshold) might warrant half size.

### Full Pipeline Integration

```python
@dataclass
class TradeCandidate:
    """A validated signal ready for confidence evaluation."""
    symbol: str
    direction: str  # "long" or "short"
    signal_type: str
    raw_confidence: float
    signal_age_minutes: float
    risk_tier: RiskTier
    model_scores: list  # List[ModelScore]


class ConfidenceGate:
    """
    Complete confidence gate integrating noise filtering, decay,
    ensemble scoring, and veto evaluation.

    Fail-closed at every step.
    """

    def __init__(
        self,
        threshold_config: ThresholdConfig,
        ensemble_scorer: EnsembleScorer,
        decay: ConfidenceDecay,
        noise_filter: NoiseFilter,
    ):
        self._config = threshold_config
        self._ensemble = ensemble_scorer
        self._decay = decay
        self._noise = noise_filter
        self._veto = VetoGate(threshold_config)

    def evaluate(self, candidate: TradeCandidate) -> ConfidenceDecision:
        tier = candidate.risk_tier
        threshold = self._config.get_threshold(tier)

        # Step 1: Noise filter on raw score
        if not self._noise.should_evaluate(candidate.raw_confidence):
            return ConfidenceDecision(
                decision=RiskDecision.REJECTED_BELOW_THRESHOLD,
                effective_confidence=candidate.raw_confidence,
                tier=tier,
                threshold_required=threshold,
                reason="Discarded by noise filter (below discard floor)",
            )

        # Step 2: Check staleness
        if self._decay.is_stale(candidate.signal_age_minutes, candidate.signal_type):
            return ConfidenceDecision(
                decision=RiskDecision.REJECTED_STALE_SIGNAL,
                effective_confidence=0.0,
                tier=tier,
                threshold_required=threshold,
                reason=f"Signal is stale ({candidate.signal_age_minutes:.1f}min old)",
            )

        # Step 3: Apply decay to individual model scores
        decayed_scores = []
        for ms in candidate.model_scores:
            decayed_value = self._decay.apply(
                ms.score, candidate.signal_age_minutes, candidate.signal_type
            )
            decayed_scores.append(
                ModelScore(model_name=ms.model_name, score=decayed_value)
            )

        # Step 4: Ensemble scoring
        ensemble = self._ensemble.score(decayed_scores)

        # Step 5: Fail-closed if no models contributed
        if ensemble.contributing_models == 0:
            return ConfidenceDecision(
                decision=RiskDecision.REJECTED_MISSING_SCORE,
                effective_confidence=0.0,
                tier=tier,
                threshold_required=threshold,
                reason="No model scores available",
            )

        # Step 6: Veto gate (includes threshold check)
        return self._veto.evaluate(ensemble, tier)
```

## Red Flags

| Red Flag | Symptom | Response |
|----------|---------|----------|
| Threshold overrides in production | Someone adds a bypass or lowers thresholds via config change | Thresholds must be immutable during market hours. Config changes require restart. |
| Ensemble with one model | Only one signal source contributing scores | System should require minimum 2 contributing models for ELEVATED and HIGH_STAKES tiers. |
| Decay disabled | Signal age not being tracked or decay multiplier set to 1.0 | Every signal MUST carry a timestamp. Decay with multiplier 1.0 is equivalent to no decay. |
| Veto floor set to 0 | Effectively disables the veto system | Veto floor below 15% should trigger a configuration warning at startup. |
| Confidence score hardcoded | Model always returns the same score regardless of conditions | Monitor score distribution. Standard deviation near zero over 100+ signals indicates a broken model. |
| Alert fatigue in logs | Hundreds of "rejected" entries per hour | Check noise filter thresholds. Discard floor may be too low. |
| Stale signals executing | Trades based on signals older than 2x half-life | Audit signal timestamps against execution timestamps. Any gap > 2x half-life is a defect. |
| All trades approved | Approval rate above 90% over sustained period | Thresholds may be too low, or models may be over-fitting confidence to recent data. |

## Integration Points

- **strategy-signal-validation**: Confidence gate consumes validated signals. Invalid signals never reach confidence scoring.
- **risk-management-gates**: Confidence decisions feed into the risk gate as an input. A trade that passes confidence still must pass risk limits.
- **pre-trade-validation**: The confidence gate is one component of the broader pre-trade validation chain. It owns the "conviction" dimension.
- **market-sentiment-analyst**: Sentiment scores from market analysis agents can be one of the N models feeding into the ensemble scorer.
