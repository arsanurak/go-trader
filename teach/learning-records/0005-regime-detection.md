# Learning Record — Lesson 05: Regime Detection และ ADX

**Date:** 2026-06-26
**Lesson:** `lessons/0005-regime-detection.html`
**Status:** taught

## Concepts Covered

- ADX measures trend *strength*, not direction — high ADX = strong trend regardless of up/down
- Default `adx_threshold = 20.0` in go-trader (`scheduler/config.go` line 77)
- 3-state ADX classification (from `shared_tools/regime.py` lines 271–282):
  - ADX < 20.0 → `ranging` (check stops here, DI values ignored)
  - ADX ≥ 20.0 AND DI+ > DI− → `trending_up`
  - ADX ≥ 20.0 AND DI− > DI+ → `trending_down`
- Enabling regime detection: `regime.enabled = true` at top-level config
- Per-strategy gate: `allowed_regimes: ["trending_up", "trending_down"]`
- Gate applies to entries only — open positions still close normally
- Warning if `allowed_regimes` set but `regime.enabled = false` (silent no-op)

## Connection to Fold 5

Fold 5 (Oct 2025–Jan 2026, Sharpe −2.64, N=2) likely represents a ranging market where
squeeze_momentum received false breakout signals. With `allowed_regimes: ["trending_up", "trending_down"]`
those 2 entries would likely have been skipped (ADX below threshold), turning a −10.1% period into
a flat period.

## Key Insight for the Student

"ADX first" rule is the single most important thing to internalize: even if DI+ >> DI−, if ADX < 20
the regime is `ranging`. The regime gate doesn't improve the strategy — it prevents the strategy
from operating in conditions it wasn't designed for.

## Tradeoff to Remember

Regime filtering reduces N trades. If too many regimes are filtered, the remaining N is too small
to trust Sharpe (lesson 03 threshold: N ≥ 50 to trust Sharpe). Must verify filtered backtest
still has sufficient N.

## Next Lesson Options

1. Lesson 06: Position Sizing & ATR — how to calculate how much capital to risk per trade; what
   `stop_loss_atr_mult` means in numbers; why ATR-based sizing is regime-aware by nature
2. Lesson 06: Reading go-trader Dashboard — how to interpret the Discord summary and `/status`
   output while the bot is running; what each column means
3. Lesson 06: Composite Regime (7 states) — how `trending_up_clean` vs `trending_up_choppy` differ
   and when to use the composite classifier over ADX

## Notes

- Quiz Q4 is an intentional trap: DI+ = 22.3 >> DI− = 8.9 but ADX = 15.6 < 20 → ranging
- The "gate entries only" rule is important: a strategy cannot be forced to exit by a regime change
- Regime detection requires `uv sync` and `check_regime.py` to be probed at startup
