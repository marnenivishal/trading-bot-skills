---
name: backtest-expert
description: Use when building professional-grade backtesting frameworks, designing hypothesis-driven strategy tests, implementing walk-forward validation, or modeling realistic execution costs
---

# Backtest Expert

## Iron Law

**EVERY BACKTEST STARTS WITH A WRITTEN HYPOTHESIS. "DOES THIS PATTERN MAKE MONEY?" IS NOT A HYPOTHESIS.**

A hypothesis is specific, falsifiable, and measurable. Before you write a single line of backtest code, you must be able to state exactly what you expect to observe, the metric you will measure, the threshold for success, and what the null hypothesis is. If you cannot write this down, you are data mining, not testing.

## Scope

This skill covers **methodology** -- how to build backtests correctly. The existing `backtesting-before-live` skill covers **requirements** -- what gates must pass before going live. Do not duplicate checklists or go/no-go criteria here. If you need acceptance criteria for advancing to paper trading, see `backtesting-before-live`.

---

## 1. Hypothesis-Driven Testing

Every backtest answers a specific, falsifiable question. The question must be precise enough that someone else could independently evaluate whether it passed or failed.

### Good vs Bad Hypotheses

| Quality | Example | Why |
|---------|---------|-----|
| **Good** | "SPY gaps >1% with 2x average volume fill within 2 hours in bull regimes with >60% probability" | Specific instrument, specific condition, measurable threshold, defined regime |
| **Good** | "Mean reversion entries at z-score < -2.0 on SPY produce >0.3% average return within 20 bars during low-VIX regimes" | Falsifiable, bounded timeframe, regime-aware |
| **Good** | "Adding a 10-period RSI filter below 30 to momentum entries improves Sharpe by >0.2 without reducing trade count by more than 40%" | Comparative, multi-metric, guards against trivial reduction |
| **Bad** | "Does gap trading work?" | Not falsifiable -- what instrument? What size gap? What timeframe? What is "work"? |
| **Bad** | "Is momentum profitable?" | Too vague to test. Momentum of what, measured how, over what period? |
| **Bad** | "This strategy makes money" | Not a question. Not falsifiable. No threshold. |

### Hypothesis Dataclass

```python
from dataclasses import dataclass
from typing import Optional


@dataclass
class BacktestHypothesis:
    """
    Every backtest must begin with a written hypothesis.

    This is not optional. If you cannot fill in every field,
    you are not ready to write the backtest.
    """
    statement: str          # The specific, falsifiable claim
    metric: str             # What you measure (Sharpe, win rate, avg return, etc.)
    threshold: float        # The minimum value for the hypothesis to hold
    null_hypothesis: str    # What you expect if the strategy has NO edge
    instrument: str         # What you are testing on
    regime: Optional[str] = None   # Market regime constraint, if any
    timeframe: Optional[str] = None  # Bar size / holding period

    def evaluate(self, observed_value: float) -> dict:
        """Evaluate whether the hypothesis holds."""
        passed = observed_value >= self.threshold
        return {
            "hypothesis": self.statement,
            "metric": self.metric,
            "threshold": self.threshold,
            "observed": observed_value,
            "passed": passed,
            "conclusion": (
                f"Hypothesis SUPPORTED: {self.metric}={observed_value:.4f} >= {self.threshold}"
                if passed
                else f"Hypothesis REJECTED: {self.metric}={observed_value:.4f} < {self.threshold}"
            ),
        }


# Usage
hypothesis = BacktestHypothesis(
    statement="SPY gaps >1% with 2x volume fill within 2 hours in bull regimes with >60% probability",
    metric="fill_probability",
    threshold=0.60,
    null_hypothesis="Gap fills are random; fill probability equals the base rate of ~45%",
    instrument="SPY",
    regime="bull (SPY above 200-day SMA)",
    timeframe="2-hour window after open",
)

result = hypothesis.evaluate(observed_value=0.67)
# hypothesis SUPPORTED: fill_probability=0.6700 >= 0.6
```

### Hypothesis Log

Every hypothesis tested must be logged permanently, whether it passes or fails. Failed hypotheses are valuable -- they prevent you from retesting the same dead end six months later.

```python
import json
from datetime import datetime


def log_hypothesis_result(hypothesis: BacktestHypothesis, result: dict, filepath: str):
    """Append hypothesis result to a persistent log file."""
    entry = {
        "timestamp": datetime.utcnow().isoformat(),
        "hypothesis": hypothesis.statement,
        "metric": hypothesis.metric,
        "threshold": hypothesis.threshold,
        "null_hypothesis": hypothesis.null_hypothesis,
        "observed": result["observed"],
        "passed": result["passed"],
        "conclusion": result["conclusion"],
    }
    with open(filepath, "a") as f:
        f.write(json.dumps(entry) + "\n")
```

---

## 2. Parameter Robustness Testing

If your strategy is profitable with a lookback of 20 but loses money with a lookback of 18 or 22, your strategy is curve-fit. A robust strategy produces stable results across a reasonable range of parameter values.

### The Sensitivity Test

For every tunable parameter, vary it +/- 20% from the optimal value and measure outcome stability. If profit disappears or Sharpe drops below 1.0 with small changes, the "optimal" parameters are an artifact of overfitting to noise.

### Parameter Stability Map

A stability map is a heatmap showing strategy performance across a grid of parameter combinations. Robust strategies show a broad plateau of profitability. Curve-fit strategies show a narrow spike.

```python
from dataclasses import dataclass, field
from typing import Callable
import itertools


@dataclass
class ParameterRange:
    """A parameter with its test range."""
    name: str
    optimal: float
    variation_pct: float = 0.20  # +/- 20%
    steps: int = 5               # Number of test points per side

    @property
    def test_values(self) -> list[float]:
        """Generate test values centered on optimal."""
        low = self.optimal * (1 - self.variation_pct)
        high = self.optimal * (1 + self.variation_pct)
        step_size = (high - low) / (self.steps * 2)
        return [low + step_size * i for i in range(self.steps * 2 + 1)]


@dataclass
class SensitivityResult:
    """Result of a single parameter combination test."""
    params: dict
    sharpe: float
    total_return: float
    max_drawdown: float
    trade_count: int


class ParameterSensitivityAnalyzer:
    """
    Vary each parameter +/- 20% and measure outcome stability.

    If profit disappears with small parameter changes, the strategy
    is curve-fit to historical noise, not capturing a real edge.
    """

    def __init__(self, backtest_fn: Callable, parameters: list[ParameterRange]):
        """
        Args:
            backtest_fn: Function that takes a param dict and returns metrics dict
                         with keys: sharpe, total_return, max_drawdown, trade_count
            parameters: List of parameters to vary
        """
        self.backtest_fn = backtest_fn
        self.parameters = parameters
        self.results: list[SensitivityResult] = []

    def run(self) -> list[SensitivityResult]:
        """Run backtest across all parameter combinations."""
        param_grids = {p.name: p.test_values for p in self.parameters}
        keys = list(param_grids.keys())
        value_lists = [param_grids[k] for k in keys]

        self.results = []
        for combo in itertools.product(*value_lists):
            params = dict(zip(keys, combo))
            metrics = self.backtest_fn(params)
            self.results.append(SensitivityResult(
                params=params,
                sharpe=metrics["sharpe"],
                total_return=metrics["total_return"],
                max_drawdown=metrics["max_drawdown"],
                trade_count=metrics["trade_count"],
            ))

        return self.results

    def stability_score(self, metric: str = "sharpe") -> float:
        """
        Compute stability score: what fraction of parameter combinations
        produce acceptable results (Sharpe > 1.0, positive return).

        Score > 0.5: reasonably robust
        Score < 0.3: likely curve-fit
        """
        if not self.results:
            raise ValueError("Run analysis first")

        acceptable = sum(
            1 for r in self.results
            if r.sharpe > 1.0 and r.total_return > 0
        )
        return acceptable / len(self.results)

    def report(self) -> dict:
        """Generate stability report."""
        if not self.results:
            raise ValueError("Run analysis first")

        sharpes = [r.sharpe for r in self.results]
        returns = [r.total_return for r in self.results]

        return {
            "total_combinations": len(self.results),
            "stability_score": self.stability_score(),
            "sharpe_mean": sum(sharpes) / len(sharpes),
            "sharpe_std": (sum((s - sum(sharpes)/len(sharpes))**2 for s in sharpes) / len(sharpes)) ** 0.5,
            "sharpe_min": min(sharpes),
            "sharpe_max": max(sharpes),
            "return_mean": sum(returns) / len(returns),
            "pct_profitable": sum(1 for r in returns if r > 0) / len(returns),
            "verdict": (
                "ROBUST" if self.stability_score() > 0.5
                else "FRAGILE -- likely curve-fit" if self.stability_score() < 0.3
                else "MARGINAL -- investigate further"
            ),
        }
```

### Interpreting Results

- **Stability score > 0.5**: Strategy is reasonably robust. Most parameter neighborhoods are profitable.
- **Stability score 0.3 - 0.5**: Marginal. The strategy may have a real edge, but it is fragile. Investigate which parameters drive sensitivity.
- **Stability score < 0.3**: Likely curve-fit. The "optimal" parameters are fitting noise. Simplify the strategy or find a different edge.

---

## 3. Walk-Forward Methodology

The existing `backtesting-before-live` skill overviews walk-forward validation and provides the go/no-go checklist. This section goes deeper into the methodology itself.

### Rolling vs Anchored Windows

**Rolling windows** slide both the start and end of the training period forward. Each fold sees the same amount of training data, but older data is dropped.

```
Rolling Walk-Forward (12-month data, 4-month train, 1-month test):

Fold 1: |==TRAIN==|=TEST=|
Fold 2:    |==TRAIN==|=TEST=|
Fold 3:       |==TRAIN==|=TEST=|
Fold 4:          |==TRAIN==|=TEST=|
```

**Anchored windows** keep the start fixed and grow the training period. Each fold sees more training data.

```
Anchored Walk-Forward (12-month data, 1-month test):

Fold 1: |==TRAIN==|=TEST=|
Fold 2: |====TRAIN====|=TEST=|
Fold 3: |======TRAIN======|=TEST=|
Fold 4: |========TRAIN========|=TEST=|
```

**When to use which:**
- **Rolling**: When you believe older data is less relevant (regime changes, structural market shifts). Most common choice.
- **Anchored**: When you believe more data always helps (stable statistical relationships). Useful for fundamental factors.

### Minimum Train/Test Ratio

The training window must be at least **4x the test window**. Shorter training periods mean the optimizer does not have enough data to find stable parameters. Longer training periods are fine but slow.

| Train Window | Minimum Test Window | Ratio |
|-------------|--------------------:|:-----:|
| 12 months | 3 months | 4:1 |
| 8 months | 2 months | 4:1 |
| 4 months | 1 month | 4:1 |
| 20 days | 5 days | 4:1 |

### Parameter Stability Across Folds

This is the most under-used diagnostic in walk-forward analysis. After running all folds, examine how much the optimal parameters change between adjacent folds.

**Rule: If optimal parameters change by more than 30% between folds, the strategy is unstable.** The "optimal" parameters are chasing noise in each window rather than capturing a persistent signal.

```python
def parameter_stability_check(fold_params: list[dict]) -> dict:
    """
    Check if optimal parameters are stable across walk-forward folds.

    Args:
        fold_params: List of dicts, one per fold, containing optimal params.

    Returns:
        Stability report with per-parameter drift analysis.
    """
    if len(fold_params) < 2:
        return {"error": "Need at least 2 folds"}

    param_names = list(fold_params[0].keys())
    stability = {}

    for name in param_names:
        values = [fp[name] for fp in fold_params]
        drifts = []
        for i in range(1, len(values)):
            if values[i-1] != 0:
                pct_change = abs(values[i] - values[i-1]) / abs(values[i-1])
                drifts.append(pct_change)

        avg_drift = sum(drifts) / len(drifts) if drifts else 0
        max_drift = max(drifts) if drifts else 0

        stability[name] = {
            "values_by_fold": values,
            "avg_drift_pct": avg_drift * 100,
            "max_drift_pct": max_drift * 100,
            "stable": avg_drift < 0.30,  # Less than 30% average change
        }

    all_stable = all(s["stable"] for s in stability.values())
    return {
        "parameters": stability,
        "all_stable": all_stable,
        "verdict": "STABLE" if all_stable else "UNSTABLE -- parameters chasing noise",
    }
```

### Walk-Forward Engine

```python
from dataclasses import dataclass
from typing import Callable, Optional
import pandas as pd


@dataclass
class WalkForwardConfig:
    """Configuration for walk-forward validation."""
    train_period: int          # Number of bars in training window
    test_period: int           # Number of bars in test window
    step_size: Optional[int] = None  # How far to slide (default: test_period)
    anchored: bool = False     # True = anchored, False = rolling
    min_train_ratio: float = 4.0  # Minimum train/test ratio

    def __post_init__(self):
        if self.step_size is None:
            self.step_size = self.test_period
        ratio = self.train_period / self.test_period
        if ratio < self.min_train_ratio:
            raise ValueError(
                f"Train/test ratio {ratio:.1f} is below minimum {self.min_train_ratio}. "
                f"Increase train_period or decrease test_period."
            )


@dataclass
class WalkForwardFold:
    """Results from a single walk-forward fold."""
    fold_number: int
    train_start: int
    train_end: int
    test_start: int
    test_end: int
    optimal_params: dict
    in_sample_metrics: dict
    out_of_sample_metrics: dict


class WalkForwardEngine:
    """
    Walk-forward validation engine with configurable window sizes.

    Splits data into rolling or anchored train/test windows,
    optimizes on training data, and evaluates on test data.
    """

    def __init__(
        self,
        data: pd.DataFrame,
        config: WalkForwardConfig,
        optimize_fn: Callable,
        evaluate_fn: Callable,
    ):
        """
        Args:
            data: Full dataset (must have enough rows for at least one fold)
            config: Walk-forward configuration
            optimize_fn: Function(train_data) -> optimal_params dict
            evaluate_fn: Function(test_data, params) -> metrics dict
        """
        self.data = data
        self.config = config
        self.optimize_fn = optimize_fn
        self.evaluate_fn = evaluate_fn
        self.folds: list[WalkForwardFold] = []

    def generate_folds(self) -> list[tuple[int, int, int, int]]:
        """Generate (train_start, train_end, test_start, test_end) tuples."""
        folds = []
        n = len(self.data)
        fold_start = 0

        while True:
            if self.config.anchored:
                train_start = 0
            else:
                train_start = fold_start

            train_end = fold_start + self.config.train_period
            test_start = train_end
            test_end = test_start + self.config.test_period

            if test_end > n:
                break

            folds.append((train_start, train_end, test_start, test_end))
            fold_start += self.config.step_size

        if not folds:
            raise ValueError(
                f"Not enough data ({n} bars) for even one fold. "
                f"Need at least {self.config.train_period + self.config.test_period} bars."
            )

        return folds

    def run(self) -> list[WalkForwardFold]:
        """Execute walk-forward validation across all folds."""
        fold_indices = self.generate_folds()
        self.folds = []

        for i, (tr_start, tr_end, te_start, te_end) in enumerate(fold_indices):
            train_data = self.data.iloc[tr_start:tr_end]
            test_data = self.data.iloc[te_start:te_end]

            # Step 1: Optimize on training data ONLY
            optimal_params = self.optimize_fn(train_data)

            # Step 2: Evaluate on training data (for comparison)
            in_sample = self.evaluate_fn(train_data, optimal_params)

            # Step 3: Freeze params and evaluate on test data
            out_of_sample = self.evaluate_fn(test_data, optimal_params)

            fold = WalkForwardFold(
                fold_number=i + 1,
                train_start=tr_start,
                train_end=tr_end,
                test_start=te_start,
                test_end=te_end,
                optimal_params=optimal_params,
                in_sample_metrics=in_sample,
                out_of_sample_metrics=out_of_sample,
            )
            self.folds.append(fold)

        return self.folds

    def aggregate_results(self) -> dict:
        """Aggregate out-of-sample results across all folds."""
        if not self.folds:
            raise ValueError("Run walk-forward first")

        oos_sharpes = [f.out_of_sample_metrics.get("sharpe", 0) for f in self.folds]
        is_sharpes = [f.in_sample_metrics.get("sharpe", 0) for f in self.folds]
        all_params = [f.optimal_params for f in self.folds]

        avg_oos_sharpe = sum(oos_sharpes) / len(oos_sharpes)
        avg_is_sharpe = sum(is_sharpes) / len(is_sharpes)

        # Decay ratio: how much performance drops out-of-sample
        decay = avg_oos_sharpe / avg_is_sharpe if avg_is_sharpe != 0 else 0

        return {
            "num_folds": len(self.folds),
            "avg_in_sample_sharpe": avg_is_sharpe,
            "avg_out_of_sample_sharpe": avg_oos_sharpe,
            "performance_decay": decay,
            "parameter_stability": parameter_stability_check(all_params),
            "verdict": (
                "PASS" if decay > 0.5 and avg_oos_sharpe > 1.0
                else "FAIL -- likely overfit" if decay < 0.3
                else "MARGINAL -- review parameter stability"
            ),
        }
```

---

## 4. Slippage Models

Slippage is the difference between the price you expected and the price you actually got. Every backtest must model slippage, but the level of realism matters. Three tiers, from naive to realistic.

### Level 1: Fixed Tick (Naive, Minimum Viable)

Add a fixed number of ticks of adverse slippage to every trade. Simple, fast, and better than nothing. Use this as the absolute floor -- never backtest with zero slippage.

### Level 2: Percentage-Based with Volume Impact (Better)

Slippage scales with order size relative to average volume. Large orders in thin markets pay more slippage. This catches strategies that look great on paper but trade illiquid instruments.

### Level 3: Fill Probability Curves (Realistic)

Model the probability of getting filled at all, based on order size vs average volume. Large orders may only partially fill or not fill at all. This is the standard for production-grade backtests.

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass
from typing import Optional
import math


@dataclass
class Fill:
    """Represents a simulated order fill."""
    requested_shares: int
    filled_shares: int
    requested_price: float
    fill_price: float
    slippage_per_share: float
    total_slippage: float
    fill_rate: float  # filled_shares / requested_shares


class SlippageModel(ABC):
    """Base class for slippage models."""

    @abstractmethod
    def simulate_fill(
        self,
        price: float,
        shares: int,
        side: str,
        avg_daily_volume: Optional[int] = None,
        tick_size: float = 0.01,
    ) -> Fill:
        """
        Simulate order fill with slippage.

        Args:
            price: Expected fill price
            shares: Number of shares to trade
            side: "buy" or "sell"
            avg_daily_volume: Average daily volume for the instrument
            tick_size: Minimum price increment
        """
        ...


class FixedTickSlippage(SlippageModel):
    """
    Level 1: Fixed tick slippage.

    Adds a fixed number of ticks adverse to every trade.
    Naive but better than zero. Use as minimum viable model.
    """

    def __init__(self, ticks: int = 1):
        self.ticks = ticks

    def simulate_fill(
        self,
        price: float,
        shares: int,
        side: str,
        avg_daily_volume: Optional[int] = None,
        tick_size: float = 0.01,
    ) -> Fill:
        slip = self.ticks * tick_size
        if side == "buy":
            fill_price = price + slip  # Buy higher
        else:
            fill_price = price - slip  # Sell lower

        return Fill(
            requested_shares=shares,
            filled_shares=shares,  # Always full fill in Level 1
            requested_price=price,
            fill_price=fill_price,
            slippage_per_share=slip,
            total_slippage=slip * shares,
            fill_rate=1.0,
        )


class VolumeImpactSlippage(SlippageModel):
    """
    Level 2: Percentage-based with volume impact.

    Slippage scales with order size relative to average daily volume.
    Captures the reality that large orders in thin markets pay more.
    """

    def __init__(self, base_bps: float = 5.0, impact_exponent: float = 0.5):
        """
        Args:
            base_bps: Base slippage in basis points (5 bps = 0.05%)
            impact_exponent: Market impact scaling (square root model)
        """
        self.base_bps = base_bps
        self.impact_exponent = impact_exponent

    def simulate_fill(
        self,
        price: float,
        shares: int,
        side: str,
        avg_daily_volume: Optional[int] = None,
        tick_size: float = 0.01,
    ) -> Fill:
        if avg_daily_volume is None or avg_daily_volume == 0:
            # Fall back to base slippage only
            volume_ratio = 0.0
        else:
            volume_ratio = shares / avg_daily_volume

        # Square-root market impact model
        impact_bps = self.base_bps + (volume_ratio ** self.impact_exponent) * 100
        slip = price * (impact_bps / 10_000)

        if side == "buy":
            fill_price = price + slip
        else:
            fill_price = price - slip

        return Fill(
            requested_shares=shares,
            filled_shares=shares,
            requested_price=price,
            fill_price=fill_price,
            slippage_per_share=slip,
            total_slippage=slip * shares,
            fill_rate=1.0,
        )


class FillProbabilitySlippage(SlippageModel):
    """
    Level 3: Fill probability curves based on order size vs average volume.

    Models partial fills and fill probability. Large orders relative
    to volume may not fully fill, which is the reality of live trading.
    """

    def __init__(
        self,
        base_bps: float = 5.0,
        impact_exponent: float = 0.5,
        max_participation_rate: float = 0.10,
    ):
        """
        Args:
            base_bps: Base slippage in basis points
            impact_exponent: Market impact scaling
            max_participation_rate: Max fraction of daily volume you can realistically trade
        """
        self.base_bps = base_bps
        self.impact_exponent = impact_exponent
        self.max_participation_rate = max_participation_rate

    def simulate_fill(
        self,
        price: float,
        shares: int,
        side: str,
        avg_daily_volume: Optional[int] = None,
        tick_size: float = 0.01,
    ) -> Fill:
        if avg_daily_volume is None or avg_daily_volume == 0:
            avg_daily_volume = shares * 100  # Assume liquid if unknown

        volume_ratio = shares / avg_daily_volume

        # Fill probability decreases as order size increases relative to volume
        # Sigmoid curve: near 100% for small orders, drops off for large ones
        fill_probability = 1.0 / (1.0 + math.exp(10 * (volume_ratio - self.max_participation_rate)))
        filled_shares = int(shares * min(fill_probability, 1.0))
        filled_shares = max(filled_shares, 0)

        # Slippage increases with volume participation
        impact_bps = self.base_bps + (volume_ratio ** self.impact_exponent) * 100
        slip = price * (impact_bps / 10_000)

        if side == "buy":
            fill_price = price + slip
        else:
            fill_price = price - slip

        fill_rate = filled_shares / shares if shares > 0 else 0

        return Fill(
            requested_shares=shares,
            filled_shares=filled_shares,
            requested_price=price,
            fill_price=fill_price,
            slippage_per_share=slip,
            total_slippage=slip * filled_shares,
            fill_rate=fill_rate,
        )
```

### Which Level to Use

| Context | Minimum Level |
|---------|:-------------:|
| Quick feasibility check | Level 1 |
| Strategy development and optimization | Level 2 |
| Final validation before paper trading | Level 3 |
| High-frequency or large-size strategies | Level 3 |

---

## 5. Commission Models

Commissions eat edge. A strategy that trades 50 times per day needs far more gross edge than one that trades 3 times per week. Model commissions explicitly and calculate how many basis points of edge they consume.

### Commission Structures

```python
from abc import ABC, abstractmethod


class CommissionModel(ABC):
    """Base class for commission calculation."""

    @abstractmethod
    def calculate(self, shares: int, price: float) -> float:
        """Return total commission for this trade."""
        ...


class PerShareCommission(CommissionModel):
    """Per-share pricing (e.g., Interactive Brokers tiered)."""

    def __init__(self, rate_per_share: float = 0.005, minimum: float = 1.0):
        self.rate_per_share = rate_per_share
        self.minimum = minimum

    def calculate(self, shares: int, price: float) -> float:
        return max(shares * self.rate_per_share, self.minimum)


class PerTradeCommission(CommissionModel):
    """Flat per-trade pricing."""

    def __init__(self, rate: float = 4.95):
        self.rate = rate

    def calculate(self, shares: int, price: float) -> float:
        return self.rate


class TieredCommission(CommissionModel):
    """Tiered pricing based on monthly volume."""

    def __init__(self, tiers: list[tuple[int, float]]):
        """
        Args:
            tiers: List of (volume_threshold, rate_per_share) pairs,
                   sorted ascending. Last tier applies to all volume above.
        """
        self.tiers = sorted(tiers, key=lambda t: t[0])

    def calculate(self, shares: int, price: float) -> float:
        for threshold, rate in reversed(self.tiers):
            if shares >= threshold:
                return shares * rate
        return shares * self.tiers[0][1]
```

### Break-Even Analysis

How many basis points of gross edge do commissions consume? If your strategy produces 10 bps of gross edge per trade and commissions cost 8 bps, your net edge is 2 bps -- one bad fill away from unprofitable.

```python
def commission_impact_bps(
    avg_trade_value: float,
    commission_per_trade: float,
    trades_per_day: int,
) -> dict:
    """
    Calculate how many basis points of edge commissions consume.

    Returns impact per trade and per day.
    """
    bps_per_trade = (commission_per_trade / avg_trade_value) * 10_000
    bps_per_day = bps_per_trade * trades_per_day

    return {
        "commission_bps_per_trade": bps_per_trade,
        "commission_bps_per_day": bps_per_day,
        "annual_commission_drag_bps": bps_per_day * 252,
        "warning": (
            "HIGH DRAG: commissions consume >5 bps/trade"
            if bps_per_trade > 5 else None
        ),
    }
```

---

## 6. Bias Detection

Biases are silent killers. Your backtest looks amazing, but the results are an illusion caused by methodological errors. Every backtest must be checked for these common violations.

### Look-Ahead Bias

Using information that would not have been available at the time of the trading decision. The most common and most dangerous bias.

**Examples:**
- Using today's close to make today's trading decision (you do not know the close until after the close).
- Computing indicators on the entire dataset including future data.
- Using adjusted prices that were adjusted after the fact (splits, dividends).

### Survivorship Bias

Only testing on stocks that exist today. Your universe excludes companies that went bankrupt, were delisted, or were acquired. This inflates returns because you are only looking at "winners."

### Selection Bias

Cherry-picking test periods that happen to work. Testing on 2020-2021 and declaring your momentum strategy works -- ignoring that 2022 destroyed momentum.

### Bias Detector

```python
from dataclasses import dataclass
import pandas as pd


@dataclass
class BiasWarning:
    """A detected potential bias in the backtest."""
    bias_type: str
    severity: str     # "critical", "warning", "info"
    description: str
    evidence: str


class BiasDetector:
    """
    Check for common backtesting biases.

    Run this on every backtest BEFORE trusting results.
    """

    def __init__(self, data: pd.DataFrame, signals: pd.DataFrame):
        """
        Args:
            data: Market data used in backtest (must have 'timestamp' or datetime index)
            signals: Generated signals with timestamps
        """
        self.data = data
        self.signals = signals
        self.warnings: list[BiasWarning] = []

    def check_lookahead(self) -> list[BiasWarning]:
        """
        Check for look-ahead bias: signals that use future data.

        Detection: If a signal at time T references data from time > T,
        look-ahead bias exists.
        """
        warnings = []

        # Check if signals reference future bar data
        if hasattr(self.signals, "data_timestamp") and hasattr(self.signals, "signal_timestamp"):
            future_refs = self.signals[
                self.signals["data_timestamp"] > self.signals["signal_timestamp"]
            ]
            if len(future_refs) > 0:
                warnings.append(BiasWarning(
                    bias_type="look-ahead",
                    severity="critical",
                    description="Signals reference data from the future",
                    evidence=f"{len(future_refs)} signals use future data points",
                ))

        self.warnings.extend(warnings)
        return warnings

    def check_survivorship(self, full_universe: list[str], test_universe: list[str]) -> list[BiasWarning]:
        """
        Check for survivorship bias: test universe missing delisted symbols.

        Args:
            full_universe: All symbols that existed during the test period
                           (including delisted/acquired)
            test_universe: Symbols actually tested
        """
        warnings = []
        missing = set(full_universe) - set(test_universe)

        if missing:
            pct_missing = len(missing) / len(full_universe) * 100
            severity = "critical" if pct_missing > 10 else "warning"
            warnings.append(BiasWarning(
                bias_type="survivorship",
                severity=severity,
                description=f"{len(missing)} symbols ({pct_missing:.1f}%) excluded from test universe",
                evidence=f"Missing symbols include: {list(missing)[:10]}...",
            ))

        self.warnings.extend(warnings)
        return warnings

    def check_selection(self, test_start: str, test_end: str) -> list[BiasWarning]:
        """
        Check for selection bias: suspiciously narrow or favorable test periods.
        """
        warnings = []
        start = pd.Timestamp(test_start)
        end = pd.Timestamp(test_end)
        duration_days = (end - start).days

        if duration_days < 180:
            warnings.append(BiasWarning(
                bias_type="selection",
                severity="warning",
                description=f"Test period is only {duration_days} days. Minimum 180 recommended.",
                evidence=f"Period: {test_start} to {test_end}",
            ))

        self.warnings.extend(warnings)
        return warnings

    def run_all_checks(
        self,
        full_universe: list[str] = None,
        test_universe: list[str] = None,
        test_start: str = None,
        test_end: str = None,
    ) -> list[BiasWarning]:
        """Run all bias checks and return combined warnings."""
        self.warnings = []
        self.check_lookahead()
        if full_universe and test_universe:
            self.check_survivorship(full_universe, test_universe)
        if test_start and test_end:
            self.check_selection(test_start, test_end)
        return self.warnings

    def has_critical(self) -> bool:
        """Return True if any critical bias was detected."""
        return any(w.severity == "critical" for w in self.warnings)
```

---

## 7. Out-of-Sample Reserve

Walk-forward uses rolling train/test splits -- this is part of development. The out-of-sample (OOS) reserve is different: it is the **final 20% of your data that you NEVER touch during development**. It is the final exam.

### Rules

1. **Reserve 20% of data before you start.** Mark it as OOS. Do not look at it, do not peek at charts from that period, do not check "just one thing."
2. **All development uses the remaining 80%.** Walk-forward, parameter optimization, hypothesis testing -- all happens on the development portion.
3. **Run OOS exactly once.** When you are done with all development and walk-forward validation, run the frozen strategy on OOS data. This is a one-shot test.
4. **Once you look at OOS, the test is contaminated.** If you change anything after seeing OOS results and re-run, you are now fitting to OOS data. The test is invalid. You need new OOS data.

### Implementation

```python
def split_oos_reserve(
    data: pd.DataFrame,
    reserve_pct: float = 0.20,
) -> tuple[pd.DataFrame, pd.DataFrame]:
    """
    Split data into development and OOS reserve.

    The OOS reserve is sacred. Touch it ONCE, at the very end.
    """
    split_idx = int(len(data) * (1 - reserve_pct))
    development = data.iloc[:split_idx].copy()
    oos_reserve = data.iloc[split_idx:].copy()

    print(f"Development: {len(development)} bars ({split_idx} / {len(data)})")
    print(f"OOS Reserve: {len(oos_reserve)} bars (FINAL EXAM -- DO NOT PEEK)")

    return development, oos_reserve


def evaluate_oos(
    oos_data: pd.DataFrame,
    strategy_params: dict,
    evaluate_fn,
    development_metrics: dict,
) -> dict:
    """
    Run the final OOS evaluation. This is a ONE-SHOT test.

    Compare OOS metrics to development metrics. If OOS performance
    is less than 50% of development performance, the strategy is
    likely overfit.
    """
    oos_metrics = evaluate_fn(oos_data, strategy_params)

    dev_sharpe = development_metrics.get("sharpe", 0)
    oos_sharpe = oos_metrics.get("sharpe", 0)
    decay = oos_sharpe / dev_sharpe if dev_sharpe != 0 else 0

    return {
        "oos_metrics": oos_metrics,
        "development_metrics": development_metrics,
        "performance_decay": decay,
        "verdict": (
            "PASS -- strategy generalizes"
            if decay > 0.50 and oos_sharpe > 0.5
            else "FAIL -- strategy does not generalize (likely overfit)"
        ),
        "WARNING": "This test is now CONSUMED. Do NOT re-run after making changes.",
    }
```

---

## Red Flags -- Stop and Investigate

| Red Flag | What It Means | Action |
|----------|---------------|--------|
| No written hypothesis before coding | Data mining, not testing | Write hypothesis first or stop |
| Backtest Sharpe > 3.0 | Almost certainly overfit | Run walk-forward; check parameter sensitivity |
| Profit vanishes with +/- 10% parameter change | Curve-fit to noise | Simplify strategy; reduce parameters |
| Optimal params change >30% between walk-forward folds | Strategy chasing regime noise | Rethink signal; may not be tradeable |
| Zero slippage in backtest | Results are fantasy | Add at minimum Level 1 slippage |
| Commission impact > 5 bps per trade | Edge may be consumed by costs | Reduce trade frequency or find thicker edge |
| No OOS reserve | All results are in-sample | Reserve 20% before development |
| OOS peeked at and re-run | OOS test is contaminated | Need fresh data for valid OOS |
| Survivorship bias in universe | Inflated returns from dropped losers | Use point-in-time universe data |
| Perfect equity curve with no drawdowns | Look-ahead bias or bug | Audit signal generation timestamps |
| Only tested on one regime | Strategy may fail in other conditions | Extend data to cover 3+ regimes |
| Strategy has 10+ tunable parameters | Overfitting risk is extreme | Simplify to 3-5 core parameters |

---

## Integration Points

- **trading-bot-skills:backtesting-before-live** -- Defines the go/no-go gates for advancing to paper trading. This skill (backtest-expert) provides the methodology; backtesting-before-live provides the checklists and acceptance criteria.
- **trading-bot-skills:strategy-optimizer** -- Parameter optimization feeds into walk-forward validation. The optimizer finds parameters; this skill validates they are robust.
- **docs/quantitative-metrics-reference.md** -- Definitions of Sharpe, Sortino, max drawdown, profit factor, and other metrics referenced throughout this skill.
- **trading-bot-skills:indicator-math-verification** -- Indicators used in strategies must be mathematically verified before they are used in backtests. Garbage indicators produce garbage backtest results.
