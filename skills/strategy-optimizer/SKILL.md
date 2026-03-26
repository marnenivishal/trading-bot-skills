---
name: strategy-optimizer
description: Use when stress-testing strategies with Monte Carlo simulations, detecting overfitting, validating strategy robustness through trade shuffling, or optimizing parameters without curve-fitting
---

# Strategy Optimizer

Iron Law: **"IF YOUR STRATEGY CANNOT SURVIVE 10,000 RANDOM TRADE ORDERINGS, IT IS OVERFIT."**

Every profitable backtest is guilty until proven innocent. This skill provides the rigorous statistical toolkit to separate genuine edge from curve-fitted illusions. Before a single dollar of real capital is deployed, a strategy must pass Monte Carlo trade shuffling, noise injection, randomized exit testing, and parameter discipline checks.

---

## 1. The Overfitting Trap

### Case Study: The Phantom Sharpe

A quantitative trader developed a mean-reversion strategy on ES futures. The backtest results were stunning:

| Metric              | Backtest Value |
|----------------------|---------------|
| Sharpe Ratio         | 3.5           |
| Max Drawdown         | 4.2%          |
| Win Rate             | 68%           |
| Profit Factor        | 2.8           |
| Annual Return        | 42%           |

The trader allocated $500,000 in live capital. Within the first month, the strategy lost 8% -- roughly $40,000. By month two, the drawdown hit 14%, exceeding the backtest maximum by over 3x. The strategy was shut down.

**Root Cause Analysis:**

The strategy's profitability depended on a specific sequence of trades that happened to occur during the test period. Three large winning trades occurred back-to-back in a low-volatility window, inflating the Sharpe ratio. When trade shuffling was applied post-mortem, the results were damning:

- **40% of random orderings produced negative returns**
- The median Sharpe across 10,000 shuffles was 0.9 (vs. 3.5 in the backtest)
- The 95th percentile max drawdown was 22% (vs. 4.2% historical)

The strategy had no real edge. It had a lucky sequence.

### Lessons Learned

1. A single backtest equity curve is a single sample from a distribution
2. The realized trade order is just one of N! possible orderings
3. Strategy evaluation must account for the full distribution of possible outcomes
4. High Sharpe ratios from short histories are almost always noise

---

## 2. Monte Carlo Trade Shuffling

The core technique: take the actual trades from a backtest, randomize their order thousands of times, and observe the distribution of outcomes.

### Why Order Matters

A strategy that produces trades [+5%, -2%, +8%, -1%, +6%] has the same final P&L regardless of order if you ignore compounding. But with compounding, position sizing, and drawdown limits, order changes everything. A string of losses early can trigger risk limits that prevent capturing later gains.

### Implementation

```python
from dataclasses import dataclass, field
import numpy as np
from typing import List, Optional, Tuple
import warnings


@dataclass
class Trade:
    """Represents a single completed trade."""
    entry_time: str
    exit_time: str
    symbol: str
    direction: str  # 'long' or 'short'
    pnl_pct: float
    pnl_abs: float
    duration_hours: float
    entry_price: float
    exit_price: float


@dataclass
class EquityCurveStats:
    """Statistics computed from a single equity curve permutation."""
    final_pnl: float
    max_drawdown: float
    max_drawdown_pct: float
    sharpe_ratio: float
    calmar_ratio: float
    max_consecutive_losses: int
    peak_equity: float
    trough_equity: float
    recovery_time_periods: int


@dataclass
class MonteCarloResult:
    """Aggregated results from all Monte Carlo permutations."""
    n_simulations: int
    pnl_distribution: np.ndarray
    drawdown_distribution: np.ndarray
    sharpe_distribution: np.ndarray
    median_pnl: float
    median_drawdown: float
    median_sharpe: float
    pnl_5th_percentile: float
    pnl_95th_percentile: float
    drawdown_5th_percentile: float
    drawdown_95th_percentile: float
    sharpe_5th_percentile: float
    sharpe_95th_percentile: float
    prob_negative_pnl: float
    prob_exceed_2x_drawdown: float
    original_sharpe: float
    original_drawdown: float
    original_pnl: float


class MonteCarloSimulator:
    """
    Performs Monte Carlo trade shuffling to assess strategy robustness.

    Takes the realized trade list from a backtest and randomizes the
    ordering to build a distribution of possible equity curves.
    """

    def __init__(
        self,
        trades: List[Trade],
        initial_capital: float = 100_000.0,
        n_simulations: int = 10_000,
        seed: Optional[int] = None,
    ):
        self.trades = trades
        self.initial_capital = initial_capital
        self.n_simulations = n_simulations
        self.rng = np.random.default_rng(seed)

        if len(trades) < 20:
            warnings.warn(
                f"Only {len(trades)} trades provided. Monte Carlo results "
                f"with fewer than 30 trades are unreliable."
            )

    def shuffle_trades(self) -> List[Trade]:
        """Return a randomly shuffled copy of the trade list."""
        indices = self.rng.permutation(len(self.trades))
        return [self.trades[i] for i in indices]

    def _compute_equity_curve(self, trades: List[Trade]) -> EquityCurveStats:
        """Compute equity curve statistics for a given trade ordering."""
        equity = self.initial_capital
        peak = equity
        max_dd = 0.0
        max_dd_pct = 0.0
        returns = []
        consecutive_losses = 0
        max_consecutive_losses = 0
        trough = equity
        peak_equity = equity
        recovery_periods = 0
        in_drawdown = False

        for trade in trades:
            equity *= (1 + trade.pnl_pct / 100.0)
            returns.append(trade.pnl_pct / 100.0)

            if equity > peak:
                peak = equity
                in_drawdown = False
            else:
                in_drawdown = True
                recovery_periods += 1

            dd = peak - equity
            dd_pct = dd / peak if peak > 0 else 0.0
            if dd > max_dd:
                max_dd = dd
                max_dd_pct = dd_pct
                trough = equity

            if equity > peak_equity:
                peak_equity = equity

            if trade.pnl_pct < 0:
                consecutive_losses += 1
                max_consecutive_losses = max(
                    max_consecutive_losses, consecutive_losses
                )
            else:
                consecutive_losses = 0

        returns_arr = np.array(returns)
        sharpe = 0.0
        if len(returns_arr) > 1 and returns_arr.std() > 0:
            sharpe = (returns_arr.mean() / returns_arr.std()) * np.sqrt(252)

        final_pnl = equity - self.initial_capital
        calmar = 0.0
        if max_dd_pct > 0:
            annual_return = (equity / self.initial_capital - 1)
            calmar = annual_return / max_dd_pct

        return EquityCurveStats(
            final_pnl=final_pnl,
            max_drawdown=max_dd,
            max_drawdown_pct=max_dd_pct,
            sharpe_ratio=sharpe,
            calmar_ratio=calmar,
            max_consecutive_losses=max_consecutive_losses,
            peak_equity=peak_equity,
            trough_equity=trough,
            recovery_time_periods=recovery_periods,
        )

    def run_simulation(self) -> MonteCarloResult:
        """
        Run the full Monte Carlo simulation.

        Shuffles trades n_simulations times, computes equity curve
        statistics for each permutation, and aggregates results.
        """
        # First compute original (unshuffled) statistics
        original_stats = self._compute_equity_curve(self.trades)

        pnl_results = []
        dd_results = []
        sharpe_results = []

        for _ in range(self.n_simulations):
            shuffled = self.shuffle_trades()
            stats = self._compute_equity_curve(shuffled)
            pnl_results.append(stats.final_pnl)
            dd_results.append(stats.max_drawdown_pct)
            sharpe_results.append(stats.sharpe_ratio)

        pnl_arr = np.array(pnl_results)
        dd_arr = np.array(dd_results)
        sharpe_arr = np.array(sharpe_results)

        return MonteCarloResult(
            n_simulations=self.n_simulations,
            pnl_distribution=pnl_arr,
            drawdown_distribution=dd_arr,
            sharpe_distribution=sharpe_arr,
            median_pnl=float(np.median(pnl_arr)),
            median_drawdown=float(np.median(dd_arr)),
            median_sharpe=float(np.median(sharpe_arr)),
            pnl_5th_percentile=float(np.percentile(pnl_arr, 5)),
            pnl_95th_percentile=float(np.percentile(pnl_arr, 95)),
            drawdown_5th_percentile=float(np.percentile(dd_arr, 5)),
            drawdown_95th_percentile=float(np.percentile(dd_arr, 95)),
            sharpe_5th_percentile=float(np.percentile(sharpe_arr, 5)),
            sharpe_95th_percentile=float(np.percentile(sharpe_arr, 95)),
            prob_negative_pnl=float(np.mean(pnl_arr < 0)),
            prob_exceed_2x_drawdown=float(
                np.mean(dd_arr > 2 * original_stats.max_drawdown_pct)
            ),
            original_sharpe=original_stats.sharpe_ratio,
            original_drawdown=original_stats.max_drawdown_pct,
            original_pnl=original_stats.final_pnl,
        )

    def compute_statistics(self, result: MonteCarloResult) -> dict:
        """Compute summary statistics and risk assessment from simulation."""
        return {
            "robustness_score": 1.0 - result.prob_negative_pnl,
            "sharpe_degradation": (
                (result.original_sharpe - result.median_sharpe)
                / result.original_sharpe
                if result.original_sharpe > 0
                else float("inf")
            ),
            "drawdown_expansion": (
                result.drawdown_95th_percentile / result.original_drawdown
                if result.original_drawdown > 0
                else float("inf")
            ),
            "sequence_dependency": (
                "HIGH"
                if result.prob_negative_pnl > 0.3
                else "MODERATE"
                if result.prob_negative_pnl > 0.1
                else "LOW"
            ),
            "confidence_interval_pnl": (
                result.pnl_5th_percentile,
                result.pnl_95th_percentile,
            ),
        }
```

---

## 3. Interpreting Monte Carlo Results

Raw simulation output is useless without decision rules. The `OverfitDetector` translates Monte Carlo distributions into actionable verdicts.

### Decision Rules

| Condition | Verdict | Severity |
|-----------|---------|----------|
| 95th percentile drawdown > 2x historical drawdown | OVERFIT | Critical |
| 5th percentile final P&L is negative | FRAGILE | High |
| Median Sharpe > 50% below backtest Sharpe | SEQUENCE-DEPENDENT | High |
| Probability of negative P&L > 30% | UNRELIABLE | Critical |
| Probability of negative P&L > 10% | SUSPECT | Moderate |
| All checks pass | ROBUST | None |

### Implementation

```python
from dataclasses import dataclass
from enum import Enum
from typing import List, Optional


class OverfitSeverity(Enum):
    NONE = "none"
    MODERATE = "moderate"
    HIGH = "high"
    CRITICAL = "critical"


class OverfitVerdict(Enum):
    ROBUST = "robust"
    SUSPECT = "suspect"
    FRAGILE = "fragile"
    SEQUENCE_DEPENDENT = "sequence_dependent"
    OVERFIT = "overfit"
    UNRELIABLE = "unreliable"


@dataclass
class OverfitFlag:
    """A single overfit indicator with explanation."""
    rule_name: str
    triggered: bool
    severity: OverfitSeverity
    observed_value: float
    threshold_value: float
    explanation: str


@dataclass
class OverfitAssessment:
    """Complete overfit assessment for a strategy."""
    strategy_name: str
    overall_verdict: OverfitVerdict
    overall_severity: OverfitSeverity
    flags: List[OverfitFlag]
    recommendation: str
    confidence: float  # 0.0 to 1.0
    n_trades_analyzed: int
    n_simulations_run: int
    summary: str

    def is_safe_to_deploy(self) -> bool:
        """Return True only if no high or critical flags are triggered."""
        return all(
            not f.triggered
            for f in self.flags
            if f.severity in (OverfitSeverity.HIGH, OverfitSeverity.CRITICAL)
        )

    def get_triggered_flags(self) -> List[OverfitFlag]:
        """Return only the flags that were triggered."""
        return [f for f in self.flags if f.triggered]


class OverfitDetector:
    """
    Analyzes Monte Carlo simulation results to detect overfitting.

    Applies a battery of statistical tests and returns a comprehensive
    OverfitAssessment with actionable recommendations.
    """

    # Configurable thresholds
    DRAWDOWN_EXPANSION_THRESHOLD = 2.0
    SHARPE_DEGRADATION_THRESHOLD = 0.50
    NEGATIVE_PNL_CRITICAL_THRESHOLD = 0.30
    NEGATIVE_PNL_MODERATE_THRESHOLD = 0.10
    MIN_TRADES_PER_PARAMETER = 10

    def __init__(
        self,
        strategy_name: str,
        mc_result: MonteCarloResult,
        n_parameters: int = 0,
    ):
        self.strategy_name = strategy_name
        self.result = mc_result
        self.n_parameters = n_parameters

    def _check_drawdown_expansion(self) -> OverfitFlag:
        """Check if tail drawdowns are excessively larger than historical."""
        ratio = (
            self.result.drawdown_95th_percentile
            / self.result.original_drawdown
            if self.result.original_drawdown > 0
            else float("inf")
        )
        triggered = ratio > self.DRAWDOWN_EXPANSION_THRESHOLD
        return OverfitFlag(
            rule_name="drawdown_expansion",
            triggered=triggered,
            severity=OverfitSeverity.CRITICAL,
            observed_value=ratio,
            threshold_value=self.DRAWDOWN_EXPANSION_THRESHOLD,
            explanation=(
                f"95th percentile drawdown is {ratio:.1f}x the historical "
                f"drawdown. Threshold: {self.DRAWDOWN_EXPANSION_THRESHOLD}x. "
                f"The strategy's drawdown profile is highly sensitive to "
                f"trade ordering."
            ),
        )

    def _check_pnl_fragility(self) -> OverfitFlag:
        """Check if the 5th percentile P&L is negative."""
        pnl_5th = self.result.pnl_5th_percentile
        triggered = pnl_5th < 0
        return OverfitFlag(
            rule_name="pnl_fragility",
            triggered=triggered,
            severity=OverfitSeverity.HIGH,
            observed_value=pnl_5th,
            threshold_value=0.0,
            explanation=(
                f"5th percentile P&L is ${pnl_5th:,.2f}. A negative value "
                f"means that in 5% of possible trade orderings, the strategy "
                f"loses money. The edge is not robust."
            ),
        )

    def _check_sharpe_degradation(self) -> OverfitFlag:
        """Check if median Sharpe is substantially below backtest Sharpe."""
        degradation = (
            (self.result.original_sharpe - self.result.median_sharpe)
            / self.result.original_sharpe
            if self.result.original_sharpe > 0
            else 0.0
        )
        triggered = degradation > self.SHARPE_DEGRADATION_THRESHOLD
        return OverfitFlag(
            rule_name="sharpe_degradation",
            triggered=triggered,
            severity=OverfitSeverity.HIGH,
            observed_value=degradation,
            threshold_value=self.SHARPE_DEGRADATION_THRESHOLD,
            explanation=(
                f"Median Sharpe ({self.result.median_sharpe:.2f}) is "
                f"{degradation:.0%} below backtest Sharpe "
                f"({self.result.original_sharpe:.2f}). The strategy's "
                f"risk-adjusted return depends on trade sequence."
            ),
        )

    def _check_negative_pnl_probability(self) -> OverfitFlag:
        """Check probability of negative P&L across permutations."""
        prob = self.result.prob_negative_pnl
        if prob > self.NEGATIVE_PNL_CRITICAL_THRESHOLD:
            severity = OverfitSeverity.CRITICAL
        elif prob > self.NEGATIVE_PNL_MODERATE_THRESHOLD:
            severity = OverfitSeverity.MODERATE
        else:
            severity = OverfitSeverity.NONE
        triggered = prob > self.NEGATIVE_PNL_MODERATE_THRESHOLD
        return OverfitFlag(
            rule_name="negative_pnl_probability",
            triggered=triggered,
            severity=severity,
            observed_value=prob,
            threshold_value=self.NEGATIVE_PNL_MODERATE_THRESHOLD,
            explanation=(
                f"{prob:.1%} of trade orderings produce negative P&L. "
                f"Critical threshold: {self.NEGATIVE_PNL_CRITICAL_THRESHOLD:.0%}. "
                f"Moderate threshold: {self.NEGATIVE_PNL_MODERATE_THRESHOLD:.0%}."
            ),
        )

    def assess(self) -> OverfitAssessment:
        """Run all overfit detection checks and produce assessment."""
        flags = [
            self._check_drawdown_expansion(),
            self._check_pnl_fragility(),
            self._check_sharpe_degradation(),
            self._check_negative_pnl_probability(),
        ]

        triggered = [f for f in flags if f.triggered]

        # Determine overall verdict (worst case wins)
        if any(
            f.rule_name == "drawdown_expansion" and f.triggered for f in flags
        ):
            verdict = OverfitVerdict.OVERFIT
            severity = OverfitSeverity.CRITICAL
        elif any(
            f.rule_name == "negative_pnl_probability"
            and f.severity == OverfitSeverity.CRITICAL
            for f in flags
        ):
            verdict = OverfitVerdict.UNRELIABLE
            severity = OverfitSeverity.CRITICAL
        elif any(
            f.rule_name == "sharpe_degradation" and f.triggered for f in flags
        ):
            verdict = OverfitVerdict.SEQUENCE_DEPENDENT
            severity = OverfitSeverity.HIGH
        elif any(
            f.rule_name == "pnl_fragility" and f.triggered for f in flags
        ):
            verdict = OverfitVerdict.FRAGILE
            severity = OverfitSeverity.HIGH
        elif triggered:
            verdict = OverfitVerdict.SUSPECT
            severity = OverfitSeverity.MODERATE
        else:
            verdict = OverfitVerdict.ROBUST
            severity = OverfitSeverity.NONE

        # Generate recommendation
        recommendations = {
            OverfitVerdict.ROBUST: "Strategy passes Monte Carlo validation. Proceed to paper trading.",
            OverfitVerdict.SUSPECT: "Strategy shows minor concerns. Increase sample size and retest.",
            OverfitVerdict.FRAGILE: "Strategy edge is fragile. Do NOT deploy live capital.",
            OverfitVerdict.SEQUENCE_DEPENDENT: "Strategy depends on trade ordering. Redesign entry/exit logic.",
            OverfitVerdict.OVERFIT: "Strategy is overfit. Discard and rebuild from scratch.",
            OverfitVerdict.UNRELIABLE: "Strategy is unreliable. Fundamental edge is absent.",
        }

        confidence = min(1.0, self.result.n_simulations / 10_000)

        return OverfitAssessment(
            strategy_name=self.strategy_name,
            overall_verdict=verdict,
            overall_severity=severity,
            flags=flags,
            recommendation=recommendations[verdict],
            confidence=confidence,
            n_trades_analyzed=len(self.result.pnl_distribution),
            n_simulations_run=self.result.n_simulations,
            summary=(
                f"Strategy '{self.strategy_name}': {verdict.value.upper()}. "
                f"{len(triggered)}/{len(flags)} flags triggered. "
                f"Median Sharpe: {self.result.median_sharpe:.2f} "
                f"(backtest: {self.result.original_sharpe:.2f}). "
                f"P(negative): {self.result.prob_negative_pnl:.1%}."
            ),
        )
```

---

## 4. Randomized Exit Testing

This technique isolates whether the edge comes from entries or exits. If a strategy remains profitable when exits are randomized, the entry signal is capturing a genuine market inefficiency.

### The Logic

- Keep all original entry signals exactly as-is
- Replace exit logic with random exits (hold for random duration between 1 and max_holding_period bars)
- Run N iterations with different random exit timings
- Compare profitability distributions

If profitability disappears with random exits, the strategy's edge lives in exit timing. Exit-dependent edges are more fragile because they require precise market microstructure knowledge that degrades over time.

### Implementation

```python
from dataclasses import dataclass
from typing import List, Callable, Optional, Tuple
import numpy as np


@dataclass
class RandomExitResult:
    """Result of a single randomized exit simulation."""
    iteration: int
    final_pnl: float
    sharpe_ratio: float
    win_rate: float
    n_trades: int
    avg_holding_period: float


@dataclass
class RandomExitAssessment:
    """Aggregated assessment of randomized exit testing."""
    original_pnl: float
    original_sharpe: float
    median_random_pnl: float
    median_random_sharpe: float
    pct_profitable_iterations: float
    edge_source: str  # 'entry', 'exit', or 'both'
    entry_alpha: float  # fraction of edge from entries
    explanation: str


class RandomizedExitTester:
    """
    Tests whether a strategy's edge comes from entries or exits.

    Keeps original entry signals, replaces exits with random holding
    periods, and measures how much profitability degrades.
    """

    def __init__(
        self,
        price_data: np.ndarray,
        entry_signals: List[int],  # bar indices where entries occur
        entry_directions: List[str],  # 'long' or 'short' for each entry
        original_pnl: float,
        original_sharpe: float,
        max_holding_period: int = 20,
        n_iterations: int = 1_000,
        seed: Optional[int] = None,
    ):
        self.price_data = price_data
        self.entry_signals = entry_signals
        self.entry_directions = entry_directions
        self.original_pnl = original_pnl
        self.original_sharpe = original_sharpe
        self.max_holding_period = max_holding_period
        self.n_iterations = n_iterations
        self.rng = np.random.default_rng(seed)

    def _simulate_random_exits(self) -> RandomExitResult:
        """Run one iteration with random exit timing."""
        trades_pnl = []
        holding_periods = []

        for i, (entry_bar, direction) in enumerate(
            zip(self.entry_signals, self.entry_directions)
        ):
            hold_time = self.rng.integers(1, self.max_holding_period + 1)
            exit_bar = min(entry_bar + hold_time, len(self.price_data) - 1)

            entry_price = self.price_data[entry_bar]
            exit_price = self.price_data[exit_bar]

            if direction == "long":
                pnl_pct = (exit_price - entry_price) / entry_price
            else:
                pnl_pct = (entry_price - exit_price) / entry_price

            trades_pnl.append(pnl_pct)
            holding_periods.append(exit_bar - entry_bar)

        if not trades_pnl:
            return RandomExitResult(
                iteration=0,
                final_pnl=0.0,
                sharpe_ratio=0.0,
                win_rate=0.0,
                n_trades=0,
                avg_holding_period=0.0,
            )

        pnl_arr = np.array(trades_pnl)
        final_pnl = float(np.sum(pnl_arr))
        sharpe = 0.0
        if pnl_arr.std() > 0:
            sharpe = float(
                (pnl_arr.mean() / pnl_arr.std()) * np.sqrt(252)
            )

        return RandomExitResult(
            iteration=0,
            final_pnl=final_pnl,
            sharpe_ratio=sharpe,
            win_rate=float(np.mean(pnl_arr > 0)),
            n_trades=len(trades_pnl),
            avg_holding_period=float(np.mean(holding_periods)),
        )

    def run(self) -> RandomExitAssessment:
        """Run full randomized exit test and produce assessment."""
        results = []
        for i in range(self.n_iterations):
            result = self._simulate_random_exits()
            result.iteration = i
            results.append(result)

        pnls = np.array([r.final_pnl for r in results])
        sharpes = np.array([r.sharpe_ratio for r in results])
        median_pnl = float(np.median(pnls))
        median_sharpe = float(np.median(sharpes))
        pct_profitable = float(np.mean(pnls > 0))

        # Compute entry alpha: what fraction of edge survives random exits
        if self.original_pnl != 0:
            entry_alpha = max(0.0, min(1.0, median_pnl / self.original_pnl))
        else:
            entry_alpha = 0.0

        if entry_alpha > 0.6:
            edge_source = "entry"
            explanation = (
                f"Entry signals retain {entry_alpha:.0%} of original edge "
                f"with random exits. The entry logic captures genuine alpha."
            )
        elif entry_alpha > 0.2:
            edge_source = "both"
            explanation = (
                f"Entry signals retain {entry_alpha:.0%} of original edge. "
                f"Both entry and exit logic contribute to profitability."
            )
        else:
            edge_source = "exit"
            explanation = (
                f"Entry signals retain only {entry_alpha:.0%} of original "
                f"edge. Profitability depends heavily on exit timing, which "
                f"is more fragile and prone to regime changes."
            )

        return RandomExitAssessment(
            original_pnl=self.original_pnl,
            original_sharpe=self.original_sharpe,
            median_random_pnl=median_pnl,
            median_random_sharpe=median_sharpe,
            pct_profitable_iterations=pct_profitable,
            edge_source=edge_source,
            entry_alpha=entry_alpha,
            explanation=explanation,
        )
```

---

## 5. Noise Injection

Robust strategies should tolerate small perturbations in price data. If adding 0.1% Gaussian noise to prices destroys profitability, the strategy is fitting to exact price levels rather than capturing structural patterns.

### Noise Levels and Interpretation

| Noise Level (Std Dev) | Interpretation |
|------------------------|---------------|
| 0.05% | Bid-ask spread noise. Strategy MUST survive this. |
| 0.10% | Normal market microstructure variation. |
| 0.20% | Moderate perturbation. Robust strategies survive. |
| 0.50% | Aggressive noise. Only truly robust strategies pass. |
| 1.00% | Extreme stress test. Few strategies should pass. |

### Implementation

```python
from dataclasses import dataclass
from typing import List, Callable, Optional
import numpy as np


@dataclass
class NoiseTestResult:
    """Result of a single noise injection test."""
    noise_level: float
    iteration: int
    final_pnl: float
    sharpe_ratio: float
    max_drawdown: float
    n_trades: int
    pnl_change_pct: float  # vs original


@dataclass
class NoiseRobustnessProfile:
    """Complete noise robustness profile across noise levels."""
    noise_levels_tested: List[float]
    survival_rates: dict  # noise_level -> fraction of profitable iterations
    mean_pnl_by_noise: dict  # noise_level -> mean P&L
    sharpe_by_noise: dict  # noise_level -> mean Sharpe
    breakdown_noise_level: Optional[float]  # lowest noise that kills strategy
    robustness_grade: str  # A through F


class NoiseInjector:
    """
    Injects calibrated Gaussian noise into price series and measures
    strategy degradation.

    Noise is applied multiplicatively: price_noisy = price * (1 + N(0, sigma))
    This preserves the general price level while perturbing tick-level data.
    """

    DEFAULT_NOISE_LEVELS = [0.0005, 0.001, 0.002, 0.005, 0.01]

    def __init__(
        self,
        price_data: np.ndarray,
        strategy_fn: Callable[[np.ndarray], float],
        sharpe_fn: Optional[Callable[[np.ndarray], float]] = None,
        drawdown_fn: Optional[Callable[[np.ndarray], float]] = None,
        noise_levels: Optional[List[float]] = None,
        n_iterations: int = 500,
        seed: Optional[int] = None,
    ):
        """
        Args:
            price_data: Original OHLCV price array.
            strategy_fn: Function that takes price data, returns final P&L.
            sharpe_fn: Function that takes price data, returns Sharpe ratio.
            drawdown_fn: Function that takes price data, returns max drawdown.
            noise_levels: Standard deviations for noise injection.
            n_iterations: Number of noisy runs per noise level.
            seed: Random seed for reproducibility.
        """
        self.price_data = price_data
        self.strategy_fn = strategy_fn
        self.sharpe_fn = sharpe_fn
        self.drawdown_fn = drawdown_fn
        self.noise_levels = noise_levels or self.DEFAULT_NOISE_LEVELS
        self.n_iterations = n_iterations
        self.rng = np.random.default_rng(seed)
        self.original_pnl = strategy_fn(price_data)

    def inject_noise(self, noise_level: float) -> np.ndarray:
        """Apply multiplicative Gaussian noise to price data."""
        noise = self.rng.normal(0, noise_level, size=self.price_data.shape)
        return self.price_data * (1 + noise)

    def test_noise_level(self, noise_level: float) -> List[NoiseTestResult]:
        """Run multiple iterations at a single noise level."""
        results = []
        for i in range(self.n_iterations):
            noisy_prices = self.inject_noise(noise_level)
            pnl = self.strategy_fn(noisy_prices)
            sharpe = self.sharpe_fn(noisy_prices) if self.sharpe_fn else 0.0
            dd = self.drawdown_fn(noisy_prices) if self.drawdown_fn else 0.0

            pnl_change = (
                (pnl - self.original_pnl) / abs(self.original_pnl) * 100
                if self.original_pnl != 0
                else 0.0
            )

            results.append(NoiseTestResult(
                noise_level=noise_level,
                iteration=i,
                final_pnl=pnl,
                sharpe_ratio=sharpe,
                max_drawdown=dd,
                n_trades=0,
                pnl_change_pct=pnl_change,
            ))
        return results

    def run_full_profile(self) -> NoiseRobustnessProfile:
        """Run noise injection across all configured noise levels."""
        survival_rates = {}
        mean_pnl = {}
        mean_sharpe = {}
        breakdown_level = None

        for level in self.noise_levels:
            results = self.test_noise_level(level)
            pnls = [r.final_pnl for r in results]
            sharpes = [r.sharpe_ratio for r in results]

            survival_rate = sum(1 for p in pnls if p > 0) / len(pnls)
            survival_rates[level] = survival_rate
            mean_pnl[level] = float(np.mean(pnls))
            mean_sharpe[level] = float(np.mean(sharpes))

            if survival_rate < 0.5 and breakdown_level is None:
                breakdown_level = level

        # Assign robustness grade
        if breakdown_level is None:
            grade = "A"
        elif breakdown_level >= 0.005:
            grade = "B"
        elif breakdown_level >= 0.002:
            grade = "C"
        elif breakdown_level >= 0.001:
            grade = "D"
        else:
            grade = "F"

        return NoiseRobustnessProfile(
            noise_levels_tested=self.noise_levels,
            survival_rates=survival_rates,
            mean_pnl_by_noise=mean_pnl,
            sharpe_by_noise=mean_sharpe,
            breakdown_noise_level=breakdown_level,
            robustness_grade=grade,
        )
```

---

## 6. Parameter Count Discipline

The most insidious form of overfitting is invisible: too many parameters relative to the number of trades. Every parameter adds a degree of freedom that can be tuned to fit noise.

### The Rule

**Minimum 10 trades per parameter.** No exceptions.

### Parameter Count vs. Minimum Trade Count

| Parameters | Minimum Trades | Notes |
|-----------|---------------|-------|
| 1 | 10 | Simple threshold strategy |
| 2 | 20 | Entry + exit threshold |
| 3 | 30 | Entry + exit + stop loss |
| 4 | 40 | Adding a filter |
| 5 | 50 | Two filters |
| 6 | 60 | Starting to get complex |
| 8 | 80 | Most strategies fall here |
| 10 | 100 | Complex multi-factor |
| 15 | 150 | Danger zone |
| 20 | 200 | Almost certainly overfit |

### What Counts as a Parameter?

Anything the developer chose that could have been different:
- Moving average lookback period
- RSI threshold
- Stop loss distance
- Take profit target
- Position sizing multiplier
- Filter on/off toggles
- Volatility lookback window
- Regime detection thresholds

### Information Criteria for Trading

Adapted from Akaike Information Criterion (AIC) and Bayesian Information Criterion (BIC):

```python
from dataclasses import dataclass
from typing import List
import numpy as np
import math


@dataclass
class ParameterDisciplineReport:
    """Assessment of parameter count vs. trade count."""
    n_parameters: int
    n_trades: int
    trades_per_parameter: float
    min_required_trades: int
    is_sufficient: bool
    aic_analog: float
    bic_analog: float
    overfitting_risk: str  # 'low', 'moderate', 'high', 'extreme'
    recommendation: str


class ParameterDisciplineChecker:
    """
    Enforces parameter count discipline and computes information
    criteria analogs for trading strategies.
    """

    TRADES_PER_PARAM = 10  # Iron rule

    def __init__(
        self,
        n_parameters: int,
        trade_returns: List[float],
        parameter_names: Optional[List[str]] = None,
    ):
        self.n_parameters = n_parameters
        self.trade_returns = np.array(trade_returns)
        self.n_trades = len(trade_returns)
        self.parameter_names = parameter_names or []

    def compute_aic_analog(self) -> float:
        """
        AIC analog for trading: penalizes model complexity.

        AIC = 2k - 2ln(L)
        Trading analog: 2 * n_params - n_trades * ln(sharpe_ratio^2)
        Lower is better.
        """
        if self.n_trades < 2:
            return float("inf")

        returns = self.trade_returns
        sharpe = returns.mean() / returns.std() if returns.std() > 0 else 0.0

        if sharpe <= 0:
            return float("inf")

        log_likelihood = self.n_trades * math.log(sharpe ** 2)
        return 2 * self.n_parameters - log_likelihood

    def compute_bic_analog(self) -> float:
        """
        BIC analog for trading: stronger complexity penalty.

        BIC = k * ln(n) - 2ln(L)
        Penalizes parameters more heavily than AIC for large samples.
        Lower is better.
        """
        if self.n_trades < 2:
            return float("inf")

        returns = self.trade_returns
        sharpe = returns.mean() / returns.std() if returns.std() > 0 else 0.0

        if sharpe <= 0:
            return float("inf")

        log_likelihood = self.n_trades * math.log(sharpe ** 2)
        return self.n_parameters * math.log(self.n_trades) - log_likelihood

    def check(self) -> ParameterDisciplineReport:
        """Run full parameter discipline check."""
        trades_per_param = (
            self.n_trades / self.n_parameters
            if self.n_parameters > 0
            else float("inf")
        )
        min_required = self.n_parameters * self.TRADES_PER_PARAM
        is_sufficient = self.n_trades >= min_required

        if trades_per_param >= 20:
            risk = "low"
            rec = "Parameter count is well-supported by trade count."
        elif trades_per_param >= 10:
            risk = "moderate"
            rec = "Meets minimum threshold. Consider reducing parameters."
        elif trades_per_param >= 5:
            risk = "high"
            rec = (
                f"Only {trades_per_param:.1f} trades per parameter. "
                f"Need {min_required} trades minimum, have {self.n_trades}. "
                f"Reduce parameters or increase sample."
            )
        else:
            risk = "extreme"
            rec = (
                f"CRITICAL: {trades_per_param:.1f} trades per parameter. "
                f"Strategy is almost certainly overfit. Reduce from "
                f"{self.n_parameters} to at most "
                f"{max(1, self.n_trades // self.TRADES_PER_PARAM)} parameters."
            )

        return ParameterDisciplineReport(
            n_parameters=self.n_parameters,
            n_trades=self.n_trades,
            trades_per_parameter=trades_per_param,
            min_required_trades=min_required,
            is_sufficient=is_sufficient,
            aic_analog=self.compute_aic_analog(),
            bic_analog=self.compute_bic_analog(),
            overfitting_risk=risk,
            recommendation=rec,
        )
```

---

## 7. Walk-Forward Optimization vs. Curve Fitting

Walk-forward optimization is the only legitimate form of parameter tuning for trading strategies. Full-sample optimization is curve fitting by definition.

### How Walk-Forward Works

1. Divide data into overlapping windows
2. In each window: optimize on in-sample portion, validate on out-of-sample portion
3. Roll forward and repeat
4. The out-of-sample results (stitched together) form the true performance estimate

### How Curve Fitting Masquerades as Optimization

| Walk-Forward Optimization | Curve Fitting |
|--------------------------|---------------|
| Optimizes in-sample, validates out-of-sample | Optimizes on full dataset |
| Parameters may change between windows | Single set of "optimal" parameters |
| Out-of-sample equity curve is the result | In-sample equity curve is the result |
| Reveals parameter instability | Hides parameter instability |
| Honest about future uncertainty | Creates illusion of certainty |

### The 30% Rule

**If optimal parameters change by more than 30% between adjacent walk-forward windows, the optimization is actually curve fitting.** The parameters are chasing noise, not capturing stable market structure.

### Implementation

```python
from dataclasses import dataclass
from typing import List, Dict, Callable, Optional, Tuple
import numpy as np


@dataclass
class WalkForwardWindow:
    """A single walk-forward optimization window."""
    window_id: int
    in_sample_start: int
    in_sample_end: int
    out_of_sample_start: int
    out_of_sample_end: int
    optimal_params: Dict[str, float]
    in_sample_sharpe: float
    out_of_sample_sharpe: float
    out_of_sample_pnl: float
    efficiency_ratio: float  # OOS Sharpe / IS Sharpe


@dataclass
class ParameterStabilityReport:
    """Assessment of parameter stability across walk-forward windows."""
    parameter_name: str
    values_by_window: List[float]
    mean_value: float
    std_value: float
    max_change_pct: float  # largest change between adjacent windows
    is_stable: bool  # True if max_change_pct < 30%


@dataclass
class WalkForwardResult:
    """Complete walk-forward optimization result."""
    windows: List[WalkForwardWindow]
    stitched_oos_pnl: float
    stitched_oos_sharpe: float
    mean_efficiency_ratio: float
    parameter_stability: List[ParameterStabilityReport]
    is_curve_fit: bool
    curve_fit_reasons: List[str]


class WalkForwardOptimizer:
    """
    Performs walk-forward optimization with parameter stability analysis.

    Detects when "optimization" is actually curve fitting by monitoring
    parameter drift across windows.
    """

    PARAM_CHANGE_THRESHOLD = 0.30  # 30% rule
    MIN_EFFICIENCY_RATIO = 0.50  # OOS should be at least 50% of IS

    def __init__(
        self,
        price_data: np.ndarray,
        optimize_fn: Callable[
            [np.ndarray], Tuple[Dict[str, float], float]
        ],
        evaluate_fn: Callable[
            [np.ndarray, Dict[str, float]], Tuple[float, float]
        ],
        n_windows: int = 5,
        in_sample_pct: float = 0.70,
        step_size: Optional[int] = None,
    ):
        """
        Args:
            price_data: Full price series.
            optimize_fn: Takes price data, returns (optimal_params, sharpe).
            evaluate_fn: Takes price data + params, returns (pnl, sharpe).
            n_windows: Number of walk-forward windows.
            in_sample_pct: Fraction of each window used for optimization.
            step_size: How far to advance each window. Defaults to auto.
        """
        self.price_data = price_data
        self.optimize_fn = optimize_fn
        self.evaluate_fn = evaluate_fn
        self.n_windows = n_windows
        self.in_sample_pct = in_sample_pct
        self.total_length = len(price_data)
        self.step_size = step_size or (
            self.total_length // (n_windows + 1)
        )

    def run(self) -> WalkForwardResult:
        """Execute walk-forward optimization across all windows."""
        window_size = int(self.total_length * 0.6)
        is_size = int(window_size * self.in_sample_pct)
        oos_size = window_size - is_size

        windows = []

        for i in range(self.n_windows):
            start = i * self.step_size
            is_end = start + is_size
            oos_end = is_end + oos_size

            if oos_end > self.total_length:
                break

            is_data = self.price_data[start:is_end]
            oos_data = self.price_data[is_end:oos_end]

            optimal_params, is_sharpe = self.optimize_fn(is_data)
            oos_pnl, oos_sharpe = self.evaluate_fn(oos_data, optimal_params)

            efficiency = oos_sharpe / is_sharpe if is_sharpe > 0 else 0.0

            windows.append(WalkForwardWindow(
                window_id=i,
                in_sample_start=start,
                in_sample_end=is_end,
                out_of_sample_start=is_end,
                out_of_sample_end=oos_end,
                optimal_params=optimal_params,
                in_sample_sharpe=is_sharpe,
                out_of_sample_sharpe=oos_sharpe,
                out_of_sample_pnl=oos_pnl,
                efficiency_ratio=efficiency,
            ))

        # Analyze parameter stability
        if windows:
            param_names = list(windows[0].optimal_params.keys())
        else:
            param_names = []

        stability_reports = []
        for param in param_names:
            values = [w.optimal_params[param] for w in windows]
            max_change = 0.0
            for j in range(1, len(values)):
                if values[j - 1] != 0:
                    change = abs(values[j] - values[j - 1]) / abs(
                        values[j - 1]
                    )
                    max_change = max(max_change, change)

            stability_reports.append(ParameterStabilityReport(
                parameter_name=param,
                values_by_window=values,
                mean_value=float(np.mean(values)) if values else 0.0,
                std_value=float(np.std(values)) if values else 0.0,
                max_change_pct=max_change,
                is_stable=max_change < self.PARAM_CHANGE_THRESHOLD,
            ))

        # Determine if curve fitting is occurring
        curve_fit_reasons = []
        unstable_params = [s for s in stability_reports if not s.is_stable]
        if unstable_params:
            curve_fit_reasons.append(
                f"{len(unstable_params)} parameters unstable across windows: "
                + ", ".join(s.parameter_name for s in unstable_params)
            )

        efficiencies = [w.efficiency_ratio for w in windows]
        mean_efficiency = float(np.mean(efficiencies)) if efficiencies else 0.0
        if mean_efficiency < self.MIN_EFFICIENCY_RATIO:
            curve_fit_reasons.append(
                f"Mean efficiency ratio {mean_efficiency:.2f} below "
                f"threshold {self.MIN_EFFICIENCY_RATIO}"
            )

        oos_sharpes = [w.out_of_sample_sharpe for w in windows]
        if oos_sharpes and np.std(oos_sharpes) > np.mean(oos_sharpes):
            curve_fit_reasons.append(
                "Out-of-sample Sharpe variance exceeds mean, indicating "
                "parameter instability"
            )

        stitched_pnl = sum(w.out_of_sample_pnl for w in windows)
        oos_returns = [w.out_of_sample_pnl for w in windows]
        stitched_sharpe = 0.0
        if len(oos_returns) > 1:
            arr = np.array(oos_returns)
            if arr.std() > 0:
                stitched_sharpe = float(
                    (arr.mean() / arr.std()) * np.sqrt(len(arr))
                )

        return WalkForwardResult(
            windows=windows,
            stitched_oos_pnl=stitched_pnl,
            stitched_oos_sharpe=stitched_sharpe,
            mean_efficiency_ratio=mean_efficiency,
            parameter_stability=stability_reports,
            is_curve_fit=len(curve_fit_reasons) > 0,
            curve_fit_reasons=curve_fit_reasons,
        )
```

---

## Red Flags

| Red Flag | Why It's Dangerous |
|----------|-------------------|
| Sharpe > 3.0 in backtests | Almost never persists live. Usually sequence-dependent or overfit to specific market regime. |
| Strategy only tested on one market regime | Bull-market strategies fail in bears. Range strategies fail in trends. Single-regime testing hides fragility. |
| More parameters than trades / 10 | Each parameter is a degree of freedom. Too many parameters fit noise, not signal. |
| Backtest uses exact fills at mid price | Real execution has slippage, partial fills, and latency. Strategies that need perfect fills have no real edge. |
| "Optimization" on full dataset | Full-sample optimization is curve fitting by definition. Only walk-forward results count. |
| Strategy fails with 0.1% noise | If bid-ask spread noise destroys profitability, the strategy is fitting to exact price levels. |
| Drawdown distribution has fat right tail | Monte Carlo reveals that rare but plausible trade orderings cause catastrophic drawdowns. |
| Parameters change >30% between WF windows | The parameters are chasing noise, not capturing stable market structure. |
| Strategy profitable only with specific exit logic | Exit-dependent edges degrade faster than entry-dependent edges as market microstructure evolves. |
| Win rate > 80% with low payoff ratio | High win rates with small average wins and large average losses are fragile to tail events. |
| No out-of-sample validation period | If 100% of data was used for development, there is zero evidence of forward performance. |
| Equity curve too smooth | Real strategies have drawdowns. Suspiciously smooth equity curves indicate lookahead bias or survivorship bias. |

---

## Integration Points

### backtest-expert
Feed backtest results into `MonteCarloSimulator` before accepting any backtest as valid. The backtest-expert skill produces trade lists; this skill stress-tests them. Every backtest should end with a Monte Carlo assessment -- no exceptions.

### backtesting-before-live
This skill sits between backtesting and live deployment. The `backtesting-before-live` skill defines the pipeline; `strategy-optimizer` provides the statistical gates that must be passed. A strategy that passes backtesting but fails Monte Carlo is NOT ready for live trading.

### strategy-signal-validation
The `RandomizedExitTester` directly complements signal validation. If `strategy-signal-validation` confirms entry signal quality but `RandomizedExitTester` shows exit-dependent edge, the strategy needs exit logic redesign before deployment.

---

## Usage Checklist

Before declaring any strategy "ready for live trading," verify:

1. Monte Carlo trade shuffling: 10,000+ permutations, <10% produce negative P&L
2. Overfit assessment: no CRITICAL or HIGH severity flags triggered
3. Randomized exit test: entry alpha > 0.4 (entries capture real edge)
4. Noise injection: strategy survives at least 0.2% noise (grade C or better)
5. Parameter discipline: minimum 10 trades per parameter, BIC favors the model
6. Walk-forward optimization: parameters stable within 30% across windows
7. Walk-forward efficiency ratio > 0.5 (out-of-sample at least half of in-sample)

If any single check fails, the strategy is not ready. Go back and simplify.
