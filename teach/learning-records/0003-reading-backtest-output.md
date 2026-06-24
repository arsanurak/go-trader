---
name: reading-backtest-output
description: Three-pillar backtest reading framework taught — Sharpe + Max Drawdown + N trades
metadata:
  type: project
---

Lesson 03 taught: Backtest output requires three metrics read together — Sharpe (edge quality), Max Drawdown (worst-case risk), and N trades (statistical validity).

Key concepts covered:
- Max Drawdown definition and formula; why psychological breaking point matters more than the number itself
- DD quality scale: <10% excellent, 10–20% acceptable, 20–35% caution, >35% reject
- N trades minimum: 30 to trust averages (Law of Large Numbers), 50+ to trust Sharpe
- hl-squeeze_momentum has N=3 and DD=20% — N is the binding constraint against scaling capital now
- Discord summary column mapping: PnL% → return, DD → max drawdown, #T → N trades, Book Sharpe → live Sharpe (on dashboard)
- Sharpe > 2.0 is a red flag for overfitting, not a green flag for edge

**Why:** Connects directly to the Trust Threshold concept from the domain model. User can now read the live Discord output and know exactly what each number means and whether it warrants scaling capital.

**How to apply:** Next lesson should cover overfitting — Lesson 03 quiz Q3 teases it and the footer nav already points to 0004-overfitting.html. The user now has the vocabulary to understand why in-sample Sharpe ≠ live Sharpe.

Interactive element: Backtest Judge — 5 strategy scorecards (including hl-squeeze_momentum itself) where user picks "Edge จริง / น่าสงสัย / N น้อยเกิน" with immediate feedback.
