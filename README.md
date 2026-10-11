# Event-Driven Backtesting Engine

A day-by-day portfolio simulator written from scratch in Python, built so that look-ahead bias is prevented by construction rather than by discipline.

The engine holds positions in shares rather than weights, marks the book to market daily, charges slippage and commission separately on every fill, and records each trade so the resulting P&L can be audited line by line. Five rebalancing rules run against a common interface and are compared under identical execution.

---

## Why write one rather than use a library

Most backtesting packages are fine. They also hide the two decisions that determine whether a backtest means anything: what information the strategy is allowed to see at each point in time, and what it costs to trade. Both are explicit here.

The design choice that follows from that is the main one in the project. A vectorised backtest builds a weight matrix, multiplies it by a return matrix and sums the rows. It runs in milliseconds and nothing in the arithmetic prevents a weight indexed at time *t* from having been computed with information from *t+1*. This engine iterates one trading day at a time and hands the strategy function only `prices.iloc[:i+1]`, so information that does not exist yet cannot be passed in. The cost is roughly three orders of magnitude in speed, which over six years of daily data is a few seconds.

## What it does

**Execution.** Slippage is applied to the fill price and is signed, so a buy fills above the close and a sell below it. Commission is applied to the notional and is unsigned. They are modelled separately because they scale differently with trade size. Sells execute before buys so proceeds fund purchases without an unintended intraday margin loan. Trades below a minimum notional are skipped rather than churning through sub-dollar corrections at every rebalance.

**Accounting.** Share counts plus a residual cash balance, not weights. A weight-based simulator assumes infinite divisibility and frictionless rebalancing; holding shares forces the leftovers into the simulation. `allow_fractional_shares` toggles between the two regimes so the effect can be measured rather than assumed away.

**Strategies.** Buy and hold, calendar rebalancing, threshold (no-trade band) rebalancing, a momentum overlay, and volatility targeting. All five implement one interface, so any difference in results comes from the rule and not from the plumbing.

**Analytics.** Sharpe, Sortino, CAGR, maximum drawdown with peak/trough/recovery decomposition, rolling risk estimates, monthly return heatmap, weight drift, and cumulative transaction costs.

**Sensitivity.** A sweep over commission from 0 to 100 bps for the quarterly rule. At that turnover it moves the Sharpe ratio from 1.06 to 1.03, so the result rests on the weights rather than on the cost assumption.

## Results, 2020 to 2025

| Rule | Sharpe | Max drawdown | Costs |
|---|---|---|---|
| Buy and hold | 0.98 | -47.3% | $100 |
| Quarterly calendar | 1.06 | -38.2% | $965 |
| Threshold 5% | 1.09 | -36.9% | $1,118 |
| Volatility target 20% | 0.92 | -34.0% | $1,539 |
| Momentum overlay | 0.81 | -50.1% | $10,192 |

Rebalancing frequency barely matters, rebalancing at all cuts drawdown by about nine points, and the momentum overlay concentrates into survivors and loses on every measure. The headline 38% CAGR is not a quotable figure: see the in-sample weights limitation below.

## Known limitations, stated rather than buried

**Cash earns nothing.** The engine holds idle balances at zero interest while the Sharpe ratio is computed against a 5% risk-free rate. Any strategy that deliberately sits in cash is penalised twice. The volatility-targeted rule is the one this distorts, and the comparison should be read with that in mind. Accruing the overnight rate on cash is the single highest-value fix outstanding.

**Missing prices are handled asymmetrically.** Mark-to-market skips NaN entries; the rebalance routine does not guard against them. The sample used here is complete, so it does not bite, but it would on a universe with a delisting.

**Free parameters were chosen, not fitted.** The 5% drift band, the 63-day momentum lookback and the 20% volatility target are all dimensions along which the result could have been improved after the fact. I have not tuned them, and a reader has no way to verify that, which is itself an argument for pre-registering parameter grids.

**The target weights are partly in-sample.** They come from the thesis document's tangency portfolio, computed in an external portfolio tool on 2019 to 2024 monthly returns, and the backtest runs 2020 to 2025. A tangency portfolio is by construction the best performer over its own estimation window, so the headline equity curve measures fit rather than strategy. The comparison between rebalancing rules is much less affected, since all five start from the same weights.

**Five strategies, best reported.** The expected maximum Sharpe ratio across *N* trials grows like sqrt(2 ln N) even with no true edge. With N = 5 the inflation is modest but not zero. Deflation is implemented in the companion project, not here.

## Running it

```bash
pip install -r requirements.txt
jupyter notebook backtesting_engine.ipynb
```

The price snapshot in `data_cache/` is committed, so the notebook reproduces the same numbers on every machine. Without the snapshot it falls back to a live `yfinance` download, and the figures will then depend on when you ran it.

## Structure

```
backtesting_engine.ipynb
│
├── DataLoader            Frozen snapshot first, live download as fallback
├── BacktestEngine        Day-by-day simulator
│   ├── run()             Mark to market, query strategy, rebalance, record
│   ├── _rebalance()      Order generation, sells before buys, cost application
│   └── get_trade_log()   Fill-level audit trail
├── Strategies            Five rules, one interface
├── PerformanceAnalytics  Return and risk-adjusted metrics
├── RiskAnalytics         Drawdown decomposition, rolling estimates
└── Visualizer            Five diagnostic charts
```

## Related

The portfolio under test comes from [ai-datacenter-portfolio](https://github.com/Ben-Graham-Wharton-Burry/ai-datacenter-portfolio), which constructs and validates the weights. This repository is about the simulation machinery.

## Author

Benjamin Kristensen, MSc Applied Economics and Finance, Copenhagen Business School.
