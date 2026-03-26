---
name: backtesting-before-live
description: Use when developing or modifying any trading strategy, before running it on paper or live, or when all analysis is post-trade rather than pre-trade
---

# Backtesting Before Live

## Purpose

This skill enforces pre-trade validation of every trading strategy. No strategy touches paper or live without demonstrating profitability in historical data under realistic assumptions. This prevents the most expensive discovery: finding out your strategy loses money with real capital.

## The Iron Law

**NO STRATEGY RUNS ON PAPER OR LIVE WITHOUT PASSING BACKTESTS FIRST.**

This is a hard gate. If a strategy cannot demonstrate profitability in historical data with realistic cost assumptions, it does not advance. Period.

## Why This Exists

Without backtesting:
- You are gambling, not trading.
- You have no baseline to compare live performance against.
- You cannot detect when a strategy has broken or the market regime has changed.
- You discover bugs with real money instead of historical data.
- You cannot distinguish between "strategy is bad" and "strategy hit normal drawdown."

## Backtesting Requirements

Every backtest MUST meet ALL of the following criteria before the strategy advances to paper trading.

### 1. Minimum Data Duration: 6+ Months

- 6 months is the absolute minimum. 12+ months is strongly preferred.
- Data must span multiple market conditions (see regime requirements below).
- Less than 6 months means you are likely curve-fitting to a single regime.

### 2. Include Volatility Events

The data period MUST include at least one significant volatility event:
- VIX spike above 25.
- Market drawdown of 5%+ from peak.
- Gap event (overnight move > 2%).
- If your 6-month window contains none of these, extend the window until it does.

### 3. Multiple Market Regimes

The backtest MUST cover at least three of these five regimes:
- **Trending up**: sustained uptrend over 4+ weeks.
- **Trending down**: sustained downtrend over 4+ weeks.
- **Ranging/choppy**: sideways market for 4+ weeks.
- **High volatility**: VIX > 25 period.
- **Low volatility**: VIX < 15 period.

### 4. Realistic Cost Assumptions

The backtest MUST include:
- **Commissions**: Use your actual broker commission rates. Even if zero, model explicitly.
- **Slippage**: Model at minimum 1 tick adverse slippage per trade. For illiquid instruments, use 2-5 ticks.
- **Spread**: Model the bid-ask spread. Do not assume mid-price fills.
- **Market impact**: For positions > 1% of average daily volume, model additional price impact.

### 5. Required Performance Metrics

Every backtest report MUST include all of these:

| Metric | Minimum Threshold | Notes |
|--------|-------------------|-------|
| Total return | Positive | After all costs |
| Sharpe ratio | > 1.0 | Annualized, risk-free rate adjusted |
| Max drawdown | < 20% | Or within your documented risk tolerance |
| Win rate | Reported | No minimum, but must be understood |
| Profit factor | > 1.2 | Gross profit / gross loss |
| Number of trades | > 30 | Minimum for statistical significance |
| Average trade duration | Reported | Sanity check for strategy type |
| Worst single trade | Reported | Must be financially survivable |
| Average winner vs average loser | Reported | Understand your edge profile |

### 6. Drawdown Analysis

Beyond max drawdown percentage, you must also know:
- Maximum drawdown duration (calendar days from peak to recovery).
- Drawdown frequency (how often drawdowns > 5% occur).
- Longest losing streak (consecutive losing trades).
- Verify your capital can survive the worst drawdown with margin of safety.
- Verify YOU can psychologically survive the worst drawdown.

## Walk-Forward Validation

Backtesting on the full dataset and then declaring victory is overfitting. You MUST use walk-forward validation.

### Train/Test Split

```
Total data: 12 months

|---- Training (8 months) ----|---- Testing (4 months) ----|

Step 1: Optimize on months 1-8, test on months 9-12
Step 2: Optimize on months 3-10, test on months 11-14
Step 3: Continue rolling forward...
```

**Rule**: You NEVER look at test data during optimization. The test period is sacred.

### Overfit Detection

Your strategy is likely overfit if ANY of these are true:
- Test performance is less than 50% of training performance.
- Strategy has more than 5 tunable parameters.
- Performance is sensitive to small parameter changes (10% parameter change causes > 50% performance change).
- Strategy only works in one market regime.
- Number of trades in test period is < 20.
- Sharpe ratio > 3.0 in training (almost always too good to be true).

### Walk-Forward Process

1. **Split data** into rolling training and testing windows.
2. **Optimize** strategy parameters on training data only.
3. **Freeze parameters** -- no changes after this point.
4. **Run** frozen strategy on test data (out-of-sample).
5. **Record** all out-of-sample metrics.
6. **Roll forward** the window and repeat steps 1-5.
7. **Aggregate** all out-of-sample results for the true performance estimate.

The aggregated out-of-sample performance is your realistic expectation. Not the in-sample number.

## Forward Testing (Paper Trading Validation)

After backtests pass all criteria, the strategy MUST run on paper for a minimum of 2 weeks.

### Forward Test Requirements

- **Duration**: Minimum 2 weeks, preferably 4 weeks.
- **Comparison**: Track all backtest metrics in real-time. Compare to backtest expectations.
- **Tolerance**: Live metrics must be within 20% of backtest metrics (Sharpe, win rate, avg trade).
- **Data logging**: Log every signal, order, fill, and position change for post-analysis.
- **Same code**: Use the identical strategy code path as backtest (see framework pattern below).

### Paper Trading Acceptance Criteria

- [ ] Paper traded for minimum 2 weeks.
- [ ] Sharpe ratio within 20% of backtest Sharpe.
- [ ] Win rate within 20% of backtest win rate.
- [ ] Max drawdown not exceeding backtest max drawdown by more than 50%.
- [ ] No technical issues (crashes, data gaps, missed signals).
- [ ] Fill rates and slippage within expected range.
- [ ] All monitoring and alerting systems verified working.

## Backtest Framework Pattern

### The Critical Rule: Identical Code

The strategy code that runs in backtesting MUST BE IDENTICAL to the code that runs live. Same function. Same logic. Same parameters. Two execution modes, one strategy implementation.

```python
# CORRECT: One strategy, two execution contexts

class MeanReversionStrategy:
    """This EXACT class runs in both backtest and live."""

    def __init__(self, config: StrategyConfig):
        self.lookback = config.lookback_period
        self.entry_zscore = config.entry_zscore
        self.exit_zscore = config.exit_zscore

    def evaluate(self, market_data: MarketData) -> Optional[Signal]:
        """Called identically by both backtest engine and live engine."""
        if len(market_data.closes) < self.lookback:
            return None  # Insufficient data

        mean = np.mean(market_data.closes[-self.lookback:])
        std = np.std(market_data.closes[-self.lookback:])
        if std == 0:
            return None

        zscore = (market_data.closes[-1] - mean) / std

        if zscore < -self.entry_zscore and not self.has_position:
            return Signal(action="buy", symbol=market_data.symbol)
        elif zscore > self.exit_zscore and self.has_position:
            return Signal(action="sell", symbol=market_data.symbol)
        return None


# Backtest execution
strategy = MeanReversionStrategy(config)
backtest = BacktestEngine(strategy=strategy, data=historical_data)
results = backtest.run()

# Live execution -- SAME strategy object
strategy = MeanReversionStrategy(config)
live = LiveEngine(strategy=strategy, broker=broker_connection)
live.run()
```

```python
# WRONG: Separate implementations that WILL diverge

class BacktestMeanReversion:   # BAD
    def evaluate(self, data): ...

class LiveMeanReversion:       # BAD -- will drift from backtest version
    def evaluate(self, data): ...
```

### Framework Architecture

```
                    +-------------------+
                    |    Strategy       |
                    |    evaluate()     |
                    +--------+----------+
                             |
                    +--------v----------+
                    |  Execution Engine |
                    |  (interface)      |
                    +--------+----------+
                             |
              +--------------+--------------+
              |                             |
    +---------v----------+      +-----------v----------+
    | BacktestExecutor   |      |   LiveExecutor       |
    | - Historical data  |      |   - Real-time data   |
    | - Simulated fills  |      |   - Broker API       |
    | - No latency       |      |   - Real latency     |
    | - Cost modeling    |      |   - Real costs       |
    +--------------------+      +----------------------+
```

## Strategy Determinism and Replay Validation

### Determinism Requirement

The same market data input MUST produce the same signals and orders every time. If it does not, your strategy has a bug. Non-determinism makes backtesting meaningless because you cannot reproduce results, and it makes debugging live issues impossible because you cannot replay what happened.

### Sources of Non-Determinism to Eliminate

1. **Random number generation without fixed seed**: Any use of `random.random()` or similar must use a fixed, logged seed so results are reproducible.
2. **System clock dependency**: Use bar timestamps from market data, not `datetime.now()`. The system clock changes between backtest and live, and between runs.
3. **Floating-point precision**: Use `Decimal` for all price calculations, not `float`. Floating-point arithmetic is not associative and results can vary across platforms.
4. **Dictionary ordering**: Use sorted iteration over dictionaries. While Python 3.7+ preserves insertion order, the insertion order itself may vary between runs if data arrives in different orders.
5. **Async timing**: Execution order of concurrent tasks is non-deterministic. Strategy logic must not depend on which async task completes first.

### Strategy Checksum

Hash strategy parameters and code version together. Log the checksum with every trade. If the checksum changes between backtest and live, strategy code has drifted and results are not comparable.

```python
import hashlib, inspect

def strategy_checksum(strategy_class, config: dict) -> str:
    code = inspect.getsource(strategy_class)
    config_str = json.dumps(config, sort_keys=True)
    return hashlib.sha256(f"{code}{config_str}".encode()).hexdigest()[:16]
```

### Replay Engine

Record the complete market data feed and all signals during live trading. Later, replay the recorded data through the strategy. The outputs must match exactly. If they do not, you have a non-determinism bug that must be found and fixed before trusting any backtest results.

### Determinism Test

Run the strategy on the same dataset twice in isolated environments. Assert that outputs are identical:

```
assert output_run1 == output_run2
```

If this assertion fails, bisect your strategy logic to find which component introduces non-determinism.

### Determinism Red Flags

- **Strategy uses `random.random()` without a fixed seed**: Results change every run. Fix by setting and logging the seed.
- **`datetime.now()` in signal logic**: Signals depend on wall-clock time instead of market data timestamps. Replace with bar timestamp.
- **Float arithmetic for prices**: `0.1 + 0.2 != 0.3` in floating point. Use `Decimal` for all price math.
- **Different behavior on replay vs live**: Non-determinism bug exists. Do not go live until replay matches.

---

## Checklist: Before Moving to Paper

- [ ] Backtest uses 6+ months of data.
- [ ] Data includes at least one volatility event (VIX > 25).
- [ ] Data covers 3+ market regimes.
- [ ] Commissions and slippage modeled realistically.
- [ ] Sharpe ratio > 1.0 (annualized, after costs).
- [ ] Max drawdown within acceptable risk tolerance.
- [ ] Profit factor > 1.2.
- [ ] At least 30 trades for statistical significance.
- [ ] Walk-forward validation completed (not just in-sample optimization).
- [ ] Out-of-sample performance within 50% of in-sample.
- [ ] Strategy code is IDENTICAL between backtest and live execution paths.
- [ ] Backtest report saved and versioned for later comparison.
- [ ] Drawdown analysis completed and survivability confirmed.

## Red Flags -- Stop and Investigate

- **No backtest at all**: Strategy goes straight to paper or live. NEVER acceptable.
- **Only 1 month of data**: Insufficient for any meaningful conclusion. Extend to 6+ months.
- **Ignoring slippage and commissions**: Backtest shows profit, but real trading shows loss. Model all costs.
- **Different code for backtest vs live**: Results WILL diverge. Use the identical code pattern.
- **Backtest Sharpe > 3.0**: Almost certainly overfit. Verify with walk-forward validation.
- **No out-of-sample testing**: In-sample results are meaningless without validation.
- **Strategy has 10+ tunable parameters**: Likely overfit to historical noise. Simplify.
- **Perfect equity curve**: Real strategies have drawdowns. Suspiciously smooth curves indicate look-ahead bias or survivorship bias.
- **No drawdown analysis**: You MUST know the worst case before risking capital.
- **"The market has changed"**: If your strategy only works in one regime, it is not robust. Fix the strategy, do not blame the market.
