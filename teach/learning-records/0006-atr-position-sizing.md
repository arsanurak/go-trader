# Learning Record — Lesson 06: ATR และ Position Sizing

**Date:** 2026-07-01
**Lesson:** `lessons/0006-atr-position-sizing.html`
**Status:** taught

## Concepts Covered

- True Range formula: `max(H−L, |H−Close_prev|, |L−Close_prev|)` — handles gaps
- ATR = Wilder's smoothed average of TR over N bars (default 14 in go-trader)
- Stop loss placement:
  - Long: `stop = entry − ATR × stop_loss_atr_mult`
  - Short: `stop = entry + ATR × stop_loss_atr_mult`
- Position sizing from dollar risk:
  - `dollar_risk = capital × risk_fraction`
  - `position_size = dollar_risk ÷ stop_distance`
- Key invariant: dollar risk stays constant; position size shrinks when market is volatile

## go-trader Connections

- `stop_loss_atr_mult` — config field in `scheduler/config.go`
- `stampEntryATRIfOpened()` — records ATR at entry into `Position.EntryATR`; rejects ATR > 50% AvgCost
- `EffectiveStopLossPct(sc)` — converts mult to % using `EntryATR / AvgCost`
- `scheduler/hyperliquid_protection.go` — places reduce-only stop on Hyperliquid using the computed %
- `shared_tools/atr.py:standard_atr()` — rounds to int when ATR ≥ 100

## Key Insights

1. ATR-based stop adapts to market "breathing room" — in calm markets the stop is narrower, in volatile markets wider; a fixed % stop would be miscalibrated in both directions
2. When ATR doubles, position size halves — this is the mechanism that keeps dollar risk constant regardless of volatility
3. The worked example (ATR=1740, mult=1.5, entry=90000): stop at $87,390, position=0.0383 BTC from $100 risk on $10,000 capital

## Connection to Prior Lessons

- Regime detection (Lesson 05) is indirectly ATR-aware: trending regimes tend to have higher ADX AND higher ATR → smaller position sizes in the same conditions where signal quality is better
- Overfitting warning (Lesson 04): optimizing `stop_loss_atr_mult` on historical data is a classic overfitting vector — N trades required to validate any change

## Next Lesson Options

1. Lesson 07: Circuit Breaker — what fires it, what it prevents, how to read it when it triggers in production
2. Lesson 07: Tiered TP (take-profit tiers) — how `tiered_tp_atr` exits at multiple levels; connects directly to ATR sizing
3. Lesson 07: Reading the Discord status summary — what each field means while the bot is running live
