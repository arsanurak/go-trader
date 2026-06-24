# Learning Record — Lesson 04: Overfitting และ Walk-Forward Test

**Date:** 2026-06-24
**Lesson:** `lessons/0004-overfitting.html`
**Status:** taught

## Concepts Covered

- Overfitting definition: strategy memorizes historical data rather than learning a repeatable edge
- Walk-forward test structure: train fold (optimizer sees) vs test fold (out-of-sample)
- Reading fold results from `walkforward_folds.json`
- BTC/USDT 5-fold results for hl-squeeze_momentum: 4/5 folds positive, fold 5 failed (Sharpe -2.64, -10.1%, N=2)
- Why fold 5 matters most for live decisions — it represents the most recent regime
- Why N=2 in fold 5 limits conclusions — can't declare edge broken, but is a warning signal

## Real Data Used

`backtest/candidates/squeeze_983/walkforward_folds.json` — BTC/USDT 1h folds:

| Fold | Period | Sharpe | Max DD | Return | Trades |
|------|--------|--------|--------|--------|--------|
| 1 | Jun–Aug 2023 | 0.89 | -11.6% | +3.6% | 11 |
| 2 | Jan–Mar 2024 | 2.43 | -12.6% | +17.3% | 11 |
| 3 | Aug–Oct 2024 | 1.13 | -18.2% | +6.9% | 10 |
| 4 | Mar–May 2025 | 3.36 | -3.2% | +10.7% | 17 |
| 5 | Oct 2025–Jan 2026 | **-2.64** | -13.1% | **-10.1%** | **2** |

## Key Insight for the Student

Fold 5 is the most recent period and it failed — Sharpe -2.64. The live strategy is running in the regime that comes after this fold. This does not prove the edge is gone (N=2 is too small to conclude), but it means the current market regime may not favor this strategy. Enabling regime detection in the config could help the strategy skip entries during unfavorable periods.

## Next Lesson Options

1. Lesson 05: Regime Detection — how ADX classifies market state; how to enable it in go-trader config; why fold 5 likely shows a regime the strategy wasn't built for
2. Lesson 05: Position Sizing & ATR — how to calculate how much to risk per trade; why $48 is actually a decent starting point for learning
3. Lesson 05: Reading Live Logs — how to interpret go-trader dashboard output while the bot is running

## Notes

- Walkforward script freezes open params at Squeeze Momentum defaults; only optimizes close stack
- Metric used for fold optimization: `dd_adjusted_return` (return penalized by drawdown)
- ETH/USDT and SOL/USDT fold data also available in walkforward_folds.json — not yet analyzed
