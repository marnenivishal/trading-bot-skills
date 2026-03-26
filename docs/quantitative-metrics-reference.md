# Quantitative Metrics Reference for Trading Strategy Evaluation

> **This is the CANONICAL source for all strategy metrics.**
> Individual skills should cross-reference this document, not duplicate formulas.
> When a skill needs a metric definition, link here rather than redefining.

---

## Table of Contents

1. [Seven Essential Metrics](#seven-essential-metrics)
2. [Annualization Table](#annualization-table)
3. [Common Calculation Errors](#common-calculation-errors)
4. [Metric Confidence Intervals](#metric-confidence-intervals)

---

## Seven Essential Metrics

### 1. Sharpe Ratio

**Target:** > 2.0

**Formula:**

```
SR = mean(R) / std(R) * sqrt(N)
```

Where `R` is the series of periodic returns and `N` is the number of periods per year.

| Frequency | N |
|-----------|-----|
| Daily     | 252 |
| Hourly    | 1,638 (252 x 6.5) |
| Minutely  | 98,280 (252 x 6.5 x 60) |

**Interpretation:**
- SR < 0.5 — Poor. Strategy barely compensates for risk.
- SR 0.5 - 1.0 — Acceptable for long-horizon strategies.
- SR 1.0 - 2.0 — Good. Competitive with institutional benchmarks.
- SR > 2.0 — Excellent. Verify it is not the result of overfitting.
- SR > 3.0 — Suspicious. Almost certainly overfit or computed incorrectly.

**Common Mistakes:**
- Using `N=365` or `N=12` for daily equity strategies. Calendar days are wrong;
  use trading days (252). Using the wrong N inflates Sharpe artificially.
- Comparing Sharpe values computed over different timeframes without annualizing.
- Including weekends/holidays in the return series, which introduces zero-return
  observations and deflates volatility.
- Computing Sharpe on gross returns instead of excess returns (returns minus
  the risk-free rate). For most short-term strategies the risk-free rate is
  negligible, but for long-horizon strategies it matters.

---

### 2. Maximum Drawdown (MDD)

**Target:** < 15%

**Formula:**

```
MDD = (peak - trough) / peak
```

Where `peak` is the running maximum of the equity curve and `trough` is the
lowest point reached after that peak before a new peak is established.

**Interpretation:**
- MDD < 10% — Conservative, suitable for most retail accounts.
- MDD 10-20% — Moderate. Acceptable if Sharpe and recovery are strong.
- MDD 20-30% — Aggressive. Requires high returns to justify.
- MDD > 30% — Dangerous. May trigger margin calls or emotional exits.

**Duration matters.** A 10% drawdown lasting 60 days is psychologically and
financially worse than a 10% drawdown lasting 5 days. Always track:
- **Drawdown depth** (the percentage loss)
- **Drawdown duration** (calendar days from peak to trough)
- **Recovery time** (calendar days from trough back to a new peak)

**Common Mistakes:**
- Computing drawdown from starting equity instead of the running peak.
- Reporting only the deepest drawdown; the second- and third-largest drawdowns
  are also important for understanding tail risk.
- Ignoring intraday drawdowns when using end-of-day data.

---

### 3. Win Rate

**Target:** 50-70%

**Formula:**

```
Win Rate = number_of_winning_trades / total_trades * 100
```

**Interpretation:**
- Win Rate alone is **misleading**.
- A 90% win rate with average win = $10 and average loss = $200 is a net loser:
  `(0.9 * 10) - (0.1 * 200) = 9 - 20 = -$11 per trade`.
- A 35% win rate with average win = $500 and average loss = $50 is highly
  profitable: `(0.35 * 500) - (0.65 * 50) = 175 - 32.5 = $142.50 per trade`.

**Always pair Win Rate with Profit Factor or the average win/loss ratio.**

**Common Mistakes:**
- Reporting win rate without context on average win vs. average loss size.
- Counting scratch trades (breakeven) as wins, which inflates the number.
- Computing win rate on too few trades (need 100+ for reliability).

---

### 4. Profit Factor

**Target:** > 1.75

**Formula:**

```
PF = gross_profit / gross_loss
```

Where `gross_profit` is the sum of all winning trade P&L and `gross_loss` is
the absolute value of the sum of all losing trade P&L.

**Interpretation:**
- PF = 1.0 — Breakeven (before costs).
- PF < 1.5 — Marginal. Commissions, slippage, and spread may consume the edge.
- PF 1.5 - 2.0 — Decent. Viable if transaction costs are low.
- PF 2.0 - 3.0 — Strong. Robust edge.
- PF > 3.0 — Suspicious. Likely overfit to historical data, or the sample size
  is too small.

**Common Mistakes:**
- Computing profit factor on gross P&L without subtracting commissions and
  slippage. Net profit factor is the meaningful number.
- Small sample sizes producing artificially high PF values.
- Not distinguishing between realized and unrealized P&L.

---

### 5. Sortino Ratio

**Target:** > 2.0

**Formula:**

```
Sortino = mean(R) / downside_deviation * sqrt(N)
```

Where `downside_deviation` is the standard deviation of negative returns only
(returns below the target return, typically 0).

**Interpretation:**
- Similar scale to Sharpe, but does not penalize upside volatility.
- A strategy that occasionally produces large gains but consistently avoids
  large losses will have a higher Sortino than Sharpe.
- Better metric for strategies with asymmetric return distributions (e.g.,
  trend-following, options selling).

**When to prefer Sortino over Sharpe:**
- Strategies with positively skewed returns (big winners, small losers).
- Options strategies where upside and downside have fundamentally different
  distributions.
- Any strategy where penalizing upside volatility would be misleading.

**Common Mistakes:**
- Using total standard deviation instead of downside-only deviation.
- Setting the target return to the mean return instead of zero or the risk-free
  rate.

---

### 6. Recovery Factor

**Target:** > 4.0

**Formula:**

```
RF = net_profit / max_drawdown
```

**Interpretation:**
- RF < 2.0 — Drawdowns consume most of the profit. Strategy is fragile.
- RF 2.0 - 4.0 — Moderate resilience. Acceptable for strategies with low MDD.
- RF 4.0 - 6.0 — Strong. Strategy generates meaningful profit relative to its
  worst drawdown.
- RF > 6.0 — Excellent. Strategy recovers quickly and repeatedly.

**Common Mistakes:**
- Using gross profit instead of net profit (after commissions and slippage).
- Computing recovery factor over a short backtest where the maximum drawdown
  has not been fully explored.
- Not accounting for the time dimension — a high RF over 10 years is less
  impressive than the same RF over 2 years.

---

### 7. Beta

**Target:** < 1.0 (for non-directional strategies)

**Formula:**

```
Beta = cov(R_strategy, R_market) / var(R_market)
```

Where `R_strategy` is the strategy's return series and `R_market` is the
benchmark market return series (typically SPY for US equities).

**Interpretation:**
- Beta = 0 — Market-neutral. Strategy returns are uncorrelated with the market.
- Beta = 1.0 — Strategy moves in lockstep with the market (no added value
  beyond market exposure).
- Beta > 1.0 — Strategy amplifies market moves. Gains more in bull markets,
  loses more in bear markets.
- Beta < 0 — Inverse correlation with market. Useful for hedging.

**Good market-neutral strategies have Beta near 0,** meaning returns come from
alpha (skill) rather than beta (market exposure).

**Common Mistakes:**
- Using the wrong benchmark. An options strategy should benchmark against the
  relevant underlying, not necessarily SPY.
- Computing beta over too short a period. Beta shifts over market regimes.
- Ignoring that beta is not constant — a strategy may be low-beta in calm
  markets and high-beta during crises.

---

## Annualization Table

Use this table to annualize any periodic metric. Multiply periodic Sharpe by
`sqrt(N)` to get annualized Sharpe.

| Frequency  | N (periods/year) | sqrt(N) | Notes                                |
|------------|-------------------|---------|--------------------------------------|
| Daily      | 252               | 15.87   | US equity trading days               |
| Hourly     | 1,638             | 40.47   | 252 days x 6.5 hours/day             |
| Minutely   | 98,280            | 313.50  | 252 days x 6.5 hours x 60 min        |
| Weekly     | 52                | 7.21    | Calendar weeks                       |
| Monthly    | 12                | 3.46    | Calendar months                      |
| Quarterly  | 4                 | 2.00    | Calendar quarters                    |

**Important notes:**
- Crypto markets trade 24/7/365, so use N=365 for daily, N=8,760 for hourly.
- Forex markets trade ~5.5 days/week, ~22 hours/day. Adjust N accordingly.
- Futures markets have extended hours. Confirm exchange-specific schedules.

---

## Common Calculation Errors

| # | Error | Why It Is Wrong | Correct Approach |
|---|-------|-----------------|------------------|
| 1 | Using calendar days (365) instead of trading days (252) | Inflates annualization factor by ~20% | Use 252 for US equities |
| 2 | Not annualizing at all | A daily Sharpe of 0.1 looks poor but annualizes to 1.59 | Always annualize before comparing |
| 3 | Including non-trading periods in volatility | Zero-return weekends deflate std, inflating Sharpe | Filter to trading periods only |
| 4 | Computing drawdown from starting equity | Misses drawdowns that occur above starting equity | Use running peak (high-water mark) |
| 5 | Ignoring commissions/slippage in profit calculations | Overstates real performance by 20-50% for active strategies | Deduct realistic transaction costs |
| 6 | Survivorship bias in win rate | Only counting strategies that "survived" to present | Include all strategies, including abandoned ones |
| 7 | Look-ahead bias in metric computation | Using future data in indicator calculations | Strictly use point-in-time data |
| 8 | Comparing metrics across different time periods | Bull-market Sharpe != bear-market Sharpe | Compare over identical or similar market regimes |
| 9 | Using arithmetic mean for multi-period returns | Overestimates compounded growth | Use geometric mean for compounded returns |
| 10 | Risk-free rate mismatch | Using annual rate with daily returns | Convert risk-free rate to match return frequency |

---

## Metric Confidence Intervals

Statistical significance of performance metrics depends on sample size. Small
samples produce unreliable estimates.

### Minimum Sample Sizes for 95% Confidence

| Metric | Minimum Sample | Notes |
|--------|----------------|-------|
| Sharpe Ratio | 2+ years of daily data (~504 observations) | Standard error of Sharpe ~ `sqrt((1 + 0.5*SR^2) / N)` |
| Win Rate | 100+ trades | Binomial confidence interval narrows slowly |
| Profit Factor | 100+ trades | Highly sensitive to outliers with fewer trades |
| Maximum Drawdown | 3+ years | Short periods underestimate true MDD |
| Beta | 1+ year of daily data | Regime-dependent; longer is better |
| Sortino Ratio | 2+ years of daily data | Same considerations as Sharpe |
| Recovery Factor | Full market cycle (5-7 years ideal) | Must include both bull and bear periods |

### Sharpe Ratio Confidence Interval

The standard error of the Sharpe Ratio is approximately:

```
SE(SR) = sqrt((1 + 0.5 * SR^2) / N)
```

For a strategy with SR = 2.0 and N = 252 (one year of daily data):

```
SE = sqrt((1 + 0.5 * 4) / 252) = sqrt(3 / 252) = sqrt(0.0119) = 0.109
```

95% confidence interval: `2.0 +/- 1.96 * 0.109 = [1.79, 2.21]`

With only 6 months of data (N = 126):

```
SE = sqrt(3 / 126) = 0.154
```

95% confidence interval: `2.0 +/- 1.96 * 0.154 = [1.70, 2.30]`

The interval widens substantially, making it harder to distinguish a 2.0 Sharpe
strategy from a 1.7 strategy.

### Monte Carlo Confidence Bands

For metrics where analytical confidence intervals are impractical (e.g., MDD,
Recovery Factor), use Monte Carlo simulation:

1. **Bootstrap resampling:** Randomly resample daily returns with replacement
   to create 1,000+ synthetic equity curves.
2. **Compute the metric** on each synthetic curve.
3. **Take the 2.5th and 97.5th percentiles** as the 95% confidence band.

This approach captures the full distribution of outcomes and is more robust
than parametric assumptions.

### Practical Guidance

- **Do not trust a Sharpe above 2.0 with less than 1 year of data.** The
  confidence interval is too wide to distinguish genuine skill from luck.
- **Do not trust a win rate with fewer than 100 trades.** A 60% win rate over
  30 trades has a 95% CI of roughly [41%, 77%].
- **Always report the sample size alongside the metric.** A Sharpe of 3.0 over
  3 months is not a finding; it is noise.
- **Out-of-sample validation is more valuable than in-sample confidence
  intervals.** A strategy with SR = 1.5 in-sample and SR = 1.3 out-of-sample
  is more trustworthy than one with SR = 3.0 in-sample only.

---

## Cross-References

- Backtesting workflow: see `skills/backtest-workflow.md`
- Risk management integration: see `skills/risk-management.md`
- Options-specific metrics (Greeks, IV analysis): see `skills/options-strategy.md`

---

*This document is the single source of truth for quantitative metrics.
If a formula here conflicts with one in a skill file, this document is correct.*
