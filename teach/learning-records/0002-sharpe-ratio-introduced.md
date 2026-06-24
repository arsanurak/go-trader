---
name: sharpe-ratio-introduced
description: Sharpe ratio taught — formula, scale, annualization, and go-trader connection
metadata:
  type: project
---

Lesson 02 taught: Sharpe = μ/σ (per trade), annualized by × √N_trades_per_year.

Key concepts covered:
- Why return alone is insufficient (same return, different drawdown risk)
- Sharpe formula with worked example (μ=0.20%, σ=0.50% vs σ=2.0% → Sharpe 0.40 vs 0.10)
- Quality scale: <0.5 weak, 0.5–1.0 acceptable, 1.0–1.5 good, 1.5+ excellent (or overfit)
- Annualization: S_annual = S_trade × √252 for daily strategies
- go-trader connection: sharpe.go (running live Sharpe), circuit_breaker_alert.go (fast Sharpe degradation detector)

**Why:** Sharpe is the primary single metric in backtest output and the foundation for reading any backtest result.

**How to apply:** Next lesson can build on Sharpe to explain how backtest output is read holistically (Sharpe + drawdown + N trades together). Also sets up overfitting discussion.

Interactive element included: live Sharpe Lab with μ/σ sliders, equity curve simulation, and gradient spectrum needle — helps build intuition for the ratio before seeing it in real backtest numbers.
