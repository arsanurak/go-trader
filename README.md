# go-trader — Crypto Trading Bot

[![GitHub release](https://img.shields.io/github/v/release/richkuo/go-trader)](https://github.com/richkuo/go-trader/releases/latest)
[![Discord](https://img.shields.io/badge/Discord-Join-5865F2?logo=discord&logoColor=white)](https://discord.com/invite/44BykmWZsP)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

A Go + Python hybrid trading system. A single Go binary (~8MB idle RAM) orchestrates 50+ strategies across spot, options, perpetual futures, and CME futures by spawning short-lived Python scripts. Both paper and live execution are supported per strategy.

Supported platforms: Binance US, Deribit, IBKR/CME, Hyperliquid, TopStep, Robinhood (crypto + stock options), OKX (spot + perps + options), Luno. Per-platform Discord/Telegram channels post hourly summaries plus immediate trade alerts. When a new release ships, the bot DMs the configured owner — reply **yes** and it pulls, rebuilds, and restarts itself.

Join the Discord: [https://discord.gg/46d7Fa2dXz](https://discord.gg/46d7Fa2dXz)

---

## Getting Started

**Quick flow for a new server:** tell OpenClaw or Hermes `install https://github.com/richkuo/go-trader and init`.

### AI Agent Setup (Recommended)

Give your AI agent [SKILL.md](SKILL.md) (raw: `https://raw.githubusercontent.com/richkuo/go-trader/main/SKILL.md`) — it clones the repo, installs deps, walks through configuration, builds the binary, and starts the service. For non-Claude agents see [AGENTS.md](AGENTS.md). Using [OpenClaw](https://openclaw.ai) or [Hermes](https://hermes-agent.nousresearch.com/)? Just say "Set up go-trader".

### Interactive Setup (go-trader init)

After building the binary, run the config wizard:

```bash
./go-trader init
```

It walks asset/strategy/platform/capital/risk/Discord choices and writes `scheduler/config.json`. Defaults to a minimal BTC spot starter; risk prompts (warn threshold, portfolio kill-switch) appear only when live trading is selected.

For scripted deployments, use `--json`:

```bash
./go-trader init --json '{"assets":["BTC"],"enableSpot":true,"spotStrategies":["sma_crossover"],"spotCapital":1000,"spotDrawdown":10}' --output config.json
```

### Manual Setup

```bash
# 1. Clone
git clone https://github.com/richkuo/go-trader.git
cd go-trader

# 2. Install Python dependencies
curl -LsSf https://astral.sh/uv/install.sh | sh   # install uv if needed
uv sync                                             # creates .venv from lockfile

# 3. Build (requires Go 1.26.2)
VER=$(git describe --tags --always --dirty 2>/dev/null || echo dev)
cd scheduler && go build -ldflags "-X main.Version=$VER" -o ../go-trader . && cd ..

# 4. Generate config
./go-trader init                                    # interactive wizard (recommended)
# — or —
./go-trader init --json '{"assets":["BTC"],...}'   # non-interactive (scripted)
# — or —
cp scheduler/config.example.json scheduler/config.json
# then edit scheduler/config.json manually

# 5. Test one cycle
./go-trader --config scheduler/config.json --once

# 6. Run as service (installs, reloads, enables, and starts — survives reboot)
export DISCORD_BOT_TOKEN="your-token"
sudo bash scripts/install-service.sh

# 7. Verify
curl -s localhost:8099/status | python3 -m json.tool
```

`scripts/install-service.sh` copies the unit into `/etc/systemd/system/`, runs `daemon-reload`, enables the service for boot, starts it, and pre-creates the `logs/` directory with the right ownership.

### Running multiple instances (paper / live / testing)

For ad hoc variants deployed alongside the main instance, use the templated unit at `systemd/go-trader@.service`. Each instance lives under `/opt/go-trader-<name>/` and is addressed as `go-trader@<name>.service`. Pre-populate the instance directory before installing the template:

```bash
# 1. Create the instance directory and copy in the binary + config
sudo mkdir -p /opt/go-trader-paper-testing/scheduler
sudo cp go-trader /opt/go-trader-paper-testing/
sudo cp scheduler/config.json /opt/go-trader-paper-testing/scheduler/
sudo chown -R go-trader:go-trader /opt/go-trader-paper-testing

# 2. Install the templated unit for this instance
sudo bash scripts/install-service.sh systemd/go-trader@.service paper-testing
# → installs the template, enables + starts go-trader@paper-testing.service
```

Or copy `go-trader.service` to a named variant, edit paths, and install:

```bash
sudo bash scripts/install-service.sh go-trader-paper-testing.service
```

Set `NO_START=1` to enable without starting immediately.

---

## Architecture

```
Go scheduler (always running, ~8MB idle)
  ↓ each cycle, spawns short-lived Python check scripts
  ↓ receives JSON signals, executes paper/live trades, manages risk
  ↓ persists to scheduler/state.db, serves localhost:8099/status
  ↓ posts Discord/Telegram summaries and trade alerts

Python adapters: binanceus, deribit, ibkr, hyperliquid, topstep, robinhood, okx, luno
```

Python provides the quant libraries (pandas, numpy, scipy, CCXT); Go provides memory efficiency. 50+ strategies peak around ~220MB for ~30s, then back to ~8MB idle.

---

## Strategies

Strategies are auto-discovered from `shared_strategies/` at `go-trader init` time; the lists below show common picks rather than the full registry.

### Spot (1h, BTC/ETH/SOL)

`sma_crossover`, `ema_crossover`, `momentum`, `rsi`, `bollinger_bands`, `macd`, `mean_reversion`, `volume_weighted`, `triple_ema`, `tema_cross`, `rsi_macd_combo`, `pairs_spread`, `stoch_rsi`, `ichimoku_cloud`, `order_blocks`, `vwap_reversion`, `chart_pattern`, `liquidity_sweeps`, `parabolic_sar`, `range_scalper`, `sweep_squeeze_combo`, `adx_trend`, `donchian_breakout`.

### Options (4h, BTC/ETH)

Deribit and IBKR/CME run the same core set: `vol_mean_reversion`, `momentum_options`, `protective_puts`, `covered_calls`. New trades are scored against existing positions for strike distance, expiry spread, and Greek balance. Max 4 positions per strategy; min score 0.3 to execute.

### Perps (1h, any HL-listed asset)

Hyperliquid perps run the full spot suite plus dedicated bidirectional/short entries: `triple_ema_bidir`, `tema_cross_bd`, `session_breakout`, `donchian_breakout`, `chart_pattern`, `liquidity_sweeps`, `bear_pullback_st`, `vwap_rejection_st`, `delta_neutral_funding`.

Direction is per-strategy via `direction: "long" | "short" | "both"` (#658). `long` (default) opens longs only; `short` opens shorts only; `both` flips long↔short on reversals. Short-focused strategies (`bear_pullback_st`, `vwap_rejection_st`) require `"short"` or `"both"`. Legacy `allow_shorts` migrates automatically (`false`→`"long"`, `true`→`"both"`).

Live requires `HYPERLIQUID_SECRET_KEY`; paper needs no key. Multiple HL strategies (including `type: "manual"`) can share a coin on the same wallet — they share one on-chain position with per-strategy SQLite bookkeeping. Peers on the same coin must share `margin_mode` and exchange `leverage`, and at most one peer may run a trailing stop (cancel/replace would race). Reduce-only SL and N-tier TPs are sized per strategy. Sub-accounts are the only path to fully independent direction/leverage/margin.

### Futures (1h, ES/NQ/MES/MNQ/CL/GC)

TopStep CME futures: `momentum`, `mean_reversion`, `rsi`, `macd`, `breakout`, `session_breakout`, `tema_cross`, `tema_cross_bd` (and others auto-discovered). Live mode requires `TOPSTEP_API_KEY` / `TOPSTEP_API_SECRET` / `TOPSTEP_ACCOUNT_ID`; paper uses Yahoo Finance.

### Robinhood Crypto (1h)

Runs the spot strategy suite on Robinhood crypto (BTC, ETH, DOGE, etc.). Paper uses Yahoo Finance OHLCV; live requires `ROBINHOOD_USERNAME` / `ROBINHOOD_PASSWORD` / `ROBINHOOD_TOTP_SECRET`.

### OKX (spot + perps + options, BTC/ETH/SOL)

Spot and perpetual swap via CCXT, plus BTC/ETH options through `check_options.py --platform=okx`. Paper uses the public API; live requires `OKX_API_KEY` / `OKX_API_SECRET` / `OKX_PASSPHRASE`. Set `OKX_SANDBOX=1` for demo trading.

### Robinhood Stock Options (4h)

US equity options on SPY/QQQ/AAPL/MSFT/etc. using the options strategy set: `covered_calls`, `protective_puts`, `momentum_options`, `vol_mean_reversion`, `wheel`, `butterfly`. Paper uses Black-Scholes; live uses `robin_stocks` for real chains and greeks.

---

## Platforms

| Platform | Type | Assets | Features |
|----------|------|--------|----------|
| Binance US | Spot | BTC, ETH, SOL | CCXT, paper trading |
| Deribit | Options | BTC, ETH | Live quotes, real expiries/strikes |
| IBKR/CME | Options | BTC, ETH | CME Micro contracts, Black-Scholes pricing |
| Hyperliquid | Perps | BTC, ETH, SOL | Paper + live trading via SDK |
| TopStep | Futures | ES, NQ, MES, MNQ, CL, GC | Paper (yfinance) + live trading via TopStepX API |
| Robinhood | Crypto | BTC, ETH, SOL, DOGE, etc. | Paper (yfinance) + live trading via robin_stocks |
| Robinhood | Options | SPY, QQQ, AAPL, MSFT, etc. | Paper (Black-Scholes) + live chains via robin_stocks |
| OKX | Spot + Perps + Options | BTC, ETH, SOL | CCXT, paper + live, MiCA/EU licensed |
| Luno | Spot | BTC, ETH, etc. | South African crypto exchange |

---

## Configuration Reference

### `scheduler/config.json`

Use `./go-trader init` (interactive) or `./go-trader init --json '...'` (scripted) to generate this file. The full structure:

```json
{
  "config_version": 14,
  "interval_seconds": 3600,
  "db_file": "scheduler/state.db",
  "log_dir": "logs",
  "auto_update": "daily",
  "status_port": 8099,
  "risk_free_rate": 0.04,
  "default_stop_loss_atr_mult": 1.0,
  "portfolio_risk": {
    "max_drawdown_pct": 25,
    "max_notional_usd": 0,
    "warn_threshold_pct": 60
  },
  "regime": {
    "enabled": false,
    "period": 14,
    "adx_threshold": 20
  },
  "discord": {
    "enabled": true,
    "token": "",
    "owner_id": "",
    "channels": { "spot": "CHANNEL_ID", "options": "CHANNEL_ID", "hyperliquid": "CHANNEL_ID", "topstep": "CHANNEL_ID", "robinhood": "CHANNEL_ID", "okx": "CHANNEL_ID", "luno": "CHANNEL_ID" },
    "trade_alert_channels": { "hyperliquid": "TRADE_CHANNEL_ID" }
  },
  "platforms": {
    "hyperliquid": { "risk": { "max_drawdown_pct": 50 } }
  },
  "strategies": [ ... ]
}
```

`config_version` is bumped automatically by `go-trader init` and migrated on startup. Recent migrations: v9 added HL perps stop-loss / margin-mode fields; v11 added the `regime` block; v13 reshaped `open_strategy`/`close_strategies` into co-located `{name, params}` refs; v14 replaced `allow_shorts` with the `direction` enum.

### Portfolio Risk

| Field | Description | Default |
|-------|-------------|---------|
| `portfolio_risk.max_drawdown_pct` | Kill switch — halt all trading if portfolio drops this % from peak | 25 |
| `portfolio_risk.max_notional_usd` | Hard cap on total notional exposure (0 = disabled) | 0 |
| `portfolio_risk.warn_threshold_pct` | Emit a Discord/Telegram warning when drawdown reaches this % of `max_drawdown_pct` (repeats every cycle while in band) | 60 |
| `risk_free_rate` | Annualized risk-free rate used in Sharpe-ratio calculations (e.g. `0.04` for 4%); `null`/omitted → default rate | 0.04 |
| `status_port` | HTTP status server port; auto-falls-back up to 5 ports on collision. Override via `--status-port` CLI flag. | 8099 |
| `default_stop_loss_atr_mult` | Top-level fallback ATR multiplier used to arm fixed-ATR stops on HL perps strategies that omit all five `stop_loss_*` / `trailing_stop_*` fields (#605/#606). Set to `0` to opt every such strategy out fleet-wide. | 1.0 |

### Regime Detection

Optional ADX+DI 3-state market regime gate (`trending_up` / `trending_down` / `ranging`). Computed once per check from the same OHLCV the strategy uses, then forwarded to entry/exit evaluators. Per-strategy `allowed_regimes` blocks new entries when the current regime isn't in the list (closes always pass).

```json
{
  "regime": { "enabled": true, "period": 14, "adx_threshold": 20 },
  "strategies": [
    { "id": "hl-momentum-btc", "allowed_regimes": ["trending_up", "trending_down"], ... }
  ]
}
```

| Field | Description | Default |
|-------|-------------|---------|
| `regime.enabled` | Compute regime label each cycle and persist on the trade row | false |
| `regime.period` | ADX/DI lookback in bars | 14 |
| `regime.adx_threshold` | ADX value below which the regime is labeled `ranging` | 20 |
| `<strategy>.allowed_regimes` | Optional whitelist of regime labels under which new entries may open | (no gate) |

`regime.enabled` toggles require restart; `allowed_regimes` is SIGHUP-reloadable. Options strategies don't currently emit a regime label.

### Correlation Tracking

Monitor portfolio-level directional exposure across all strategies. Disabled by default — opt in by setting `correlation.enabled: true`.

```json
{
  "correlation": {
    "enabled": true,
    "max_concentration_pct": 60,
    "max_same_direction_pct": 75
  }
}
```

| Field | Description | Default |
|-------|-------------|---------|
| `correlation.enabled` | Enable correlation tracking and warnings | false |
| `correlation.max_concentration_pct` | Warn when one asset exceeds this % of portfolio gross exposure | 60 |
| `correlation.max_same_direction_pct` | Warn when more than this % of strategies on an asset share a direction | 75 |

When thresholds are exceeded, warnings are sent to all active Discord channels and DM'd to the owner (if configured). The correlation snapshot is also available via the `/status` endpoint.

### Auto-Update & DM Upgrades

`auto_update`: `"off"` (default), `"daily"`, or `"heartbeat"` (every cycle). When an update is found, all active Discord channels are notified; if `discord.owner_id` is set, the bot also DMs you `Would you like me to upgrade automatically? (yes/no)`. Reply **yes** → it runs `scripts/update.sh` (git pull → uv sync → go build) and restarts itself.

After an upgrade, any new config fields introduced since your `config_version` are collected via DM (10-minute reply window per field) and written back to `config.json` atomically.

Discord user ID: right-click your username → **Copy User ID** (Developer Mode: Settings → Advanced).

### Discord Settings

| Field | Description |
|-------|-------------|
| `discord.enabled` | Enable/disable Discord notifications |
| `discord.token` | Leave blank — use `DISCORD_BOT_TOKEN` env var |
| `discord.owner_id` | Your Discord user ID — enables DM upgrade prompts and post-upgrade config migration. Use `DISCORD_OWNER_ID` env var. |
| `discord.channels` | Map of channel IDs keyed by platform/type — `"spot"`, `"options"`, `"hyperliquid"`, `"topstep"`, `"robinhood"`, `"okx"`, etc. Options post per-check; others post hourly + on trades. |
| `discord.trade_alert_channels` | Optional override map (same key scheme) routing trade-fill alerts to a different channel from summaries (#573). A `stratType` key (e.g. `"perps"`) reroutes that type across all platforms; falls back to `channels` when unset. SIGHUP-reloadable. |
| `telegram.trade_alert_channels` | Same override for Telegram |
| `config_version` | Schema version (set automatically by `go-trader init`; migration runs on startup when behind current version) |

### Summary Frequency

Control how often each channel posts a summary via the top-level `summary_frequency` map. Keys match the `discord.channels` keys (e.g. `"spot"`, `"hyperliquid"`). Trades always force an immediate post regardless of the configured cadence.

```json
{
  "summary_frequency": {
    "spot": "hourly",
    "options": "every",
    "hyperliquid": "every",
    "topstep": "30m"
  }
}
```

| Value | Behavior |
|-------|----------|
| `"every"` / `"per_check"` / `"always"` | Post every scheduler cycle |
| `"hourly"` | Post once per hour (wall-clock) |
| `"daily"` | Post once per day (wall-clock) |
| `"30m"`, `"2h"`, etc. | Post when this much wall-clock time has elapsed since the last post (Go duration syntax) |
| `""` (omitted) | Legacy default — options/perps/futures post every channel run; spot posts hourly |

Cadence is wall-clock based and survives restarts: per-channel last-post timestamps are persisted in SQLite (`app_state.last_summary_post`), so variable scheduler wake-ups and SIGHUP reloads no longer reset the throttle window (#474).

### Strategy Entry

| Field | Description | Default |
|-------|-------------|---------|
| `id` | Unique identifier (e.g., `momentum-btc`, `hl-momentum-btc`) | Required |
| `type` | `"spot"`, `"options"`, `"perps"`, `"futures"`, or `"manual"` (HL perps tracking strategy for hand-placed positions, #571) | Required |
| `platform` | `"binanceus"`, `"deribit"`, `"ibkr"`, `"hyperliquid"`, `"topstep"`, `"robinhood"`, `"okx"`, or `"luno"` | Required |
| `script` | Python script path (relative) — auto-filled for `type: "manual"` | Required |
| `args` | Arguments passed to script | Required |
| `capital` | Starting capital in USD | 1000 |
| `max_drawdown_pct` | Circuit breaker threshold — peak-relative for spot/options/futures; margin-relative for perps (#292) | Spot: 5%, Options: 10%, Perps: 5% |
| `interval_seconds` | Check interval (0 = use global) | 0 |
| `htf_filter` | Enable higher-timeframe trend filter | false |
| `open_strategy` | Co-located ref `{"name": "...", "params": {...}}` overriding the entry strategy (v13+ shape). Bare strings are migrated automatically (#642). Falls back to `args[0]` if omitted. | null |
| `close_strategies` | Ordered list of co-located refs `[{"name": "...", "params": {...}}, ...]`; the one with the largest `close_fraction` per cycle wins (max-wins). Each ref carries its own params, so per-close knobs no longer leak into the open strategy (#642). | null |
| `leverage` | Perps only — exchange leverage used for margin drawdown and HL `update_leverage` (#497). If `sizing_leverage` is omitted, this also controls order sizing for backwards compatibility. | 1 |
| `sizing_leverage` | Perps only — position-sizing multiplier used for `cash * sizing_leverage` order budgets (#497). Set lower than exchange `leverage` to run high exchange leverage without oversized orders. | `leverage` |
| `margin_per_trade_usd` | HL perps only — fixed per-trade margin override; notional becomes `min(margin_per_trade_usd, cash) × leverage`, replacing the legacy 0.95 buffer (#520). | omitted |
| `stop_loss_pct` | HL perps only — reduce-only stop-loss trigger as a % of entry price. Omit to fall back to the next field in priority order. Explicit `0` opts out. | omitted |
| `stop_loss_margin_pct` | HL perps only — leverage-aware alternative; price % derives as `stop_loss_margin_pct / leverage` (#490/#497). Mutually exclusive with the other four stop-loss fields when positive. | omitted |
| `stop_loss_atr_mult` | HL perps only — fixed ATR-based stop placed at `avg_cost ± mult * entry_atr` once `entry_atr` is known (#563). Never updated after arming. Mutually exclusive with the other four stop fields when positive. | omitted |
| `trailing_stop_pct` | HL perps only — synthetic trailing stop distance (% from high-water mark). Cancel/replace debounced by `trailing_stop_min_move_pct` (#501/#502). Mutually exclusive with the other four stop fields. Capped at 50%. | omitted |
| `trailing_stop_atr_mult` | HL perps only — ATR-distance trailing stop frozen at open (`mult * entry_atr / avg_cost`, #507). Mutually exclusive with the other four stop fields. | omitted |
| `trailing_stop_min_move_pct` | HL trailing stop only — minimum trigger-price move (%) before issuing a cancel/replace. Reduces churn against HL's 1000-OID account cap. | 0.5 |
| `margin_mode` | HL perps only — `"isolated"` or `"cross"`; applied via `update_leverage` from flat (#486) | `isolated` |
| `direction` | Perps only — `"long"`, `"short"`, or `"both"` (#658). Replaces the legacy `allow_shorts` boolean, which is migrated automatically. Long-only strategies should keep `"long"`; bidirectional or short-focused strategies opt in. | `"long"` |
| `allowed_regimes` | Optional whitelist of regime labels (`trending_up` / `trending_down` / `ranging`) under which entries may open. Closes always run. Requires top-level `regime.enabled`. | (no gate) |
| `theta_harvest` | Early exit config for sold options | null |

When all five HL perps stop-loss / trailing-stop fields are omitted, the scheduler arms a fixed ATR stop at `default_stop_loss_atr_mult * entry_atr` (default `1.0`, top-level config, #605/#606). Set the top-level field to `0` to opt every such strategy out fleet-wide. Same-coin peers may carry independent fixed-distance stops, but at most one peer may run a trailing stop (cancel/replace would race).

### Custom Strategy Parameters

Override default strategy parameters per-strategy. Useful for tuning indicators to specific assets or timeframes:

```json
{
  "id": "ts-st-es",
  "type": "futures",
  "platform": "topstep",
  "script": "shared_scripts/check_topstep.py",
  "args": ["supertrend", "ES", "5m", "--mode=paper"],
  "capital": 5000,
  "max_drawdown_pct": 10,
  "interval_seconds": 300,
  "params": {"multiplier": 2.0, "atr_period": 10}
}
```

The `params` object is passed to `apply_strategy()` and merged with the strategy's built-in defaults. Any key in `params` overrides the corresponding default. For strategies that also receive runtime data (e.g. Hyperliquid/OKX funding rates), runtime values take priority over config params.

### Theta Harvesting (Options)

Closes sold options early based on profit target, stop loss, or approaching expiry:

```json
{
  "theta_harvest": {
    "enabled": true,
    "profit_target_pct": 60,
    "stop_loss_pct": 200,
    "min_dte_close": 3
  }
}
```

| Field | Description |
|-------|-------------|
| `profit_target_pct` | Close when this % of premium captured (e.g., 60 = take profit at 60%) |
| `stop_loss_pct` | Close if loss exceeds this % of premium (e.g., 200 = 2× premium) |
| `min_dte_close` | Force-close positions with fewer than N days to expiry |

---

## Manual Trading on Hyperliquid

For positions opened by hand on Hyperliquid (or via TradingView alerts) but still tracked for P&L, stops/TPs, and Discord summaries, declare a `type: "manual"` strategy and use the `manual-open` / `manual-close` CLI. The scheduler auto-fills `script`, `args`, `interval_seconds`, and a default `stop_loss_atr_mult: 1.0` + `tiered_tp_atr_live` close strategy.

```bash
# Open a long with reduce-only SL + TP placed inline
./go-trader manual-open --strategy hl-manual-btc --side long --notional 500 --atr 250

# Record a manual fill from outside the scheduler
./go-trader manual-open --strategy hl-manual-btc --side short --size 0.05 --record-only --fill-price 64500

# Close (full or partial)
./go-trader manual-close --strategy hl-manual-btc
./go-trader manual-close --strategy hl-manual-btc --fraction 0.5
```

Sizing is mutually exclusive: `--size` (coin units) / `--notional` (USD) / `--margin` (USD margin). When `--atr` is omitted, the scheduler arms a leverage-aware fallback ATR (`0.1 * fill_price / leverage`). SL + N-tier TPs are placed inline so the position is never naked between fill and the next cycle; if the queue insert fails after a successful fill, the scheduler auto-flattens and cancels the protective orders.

---

## Backfilling Hyperliquid Fees

Historical `exchange_fee = 0` rows can be rewritten from Hyperliquid `userFills`:

```bash
./go-trader backfill hl-fees --strategy hl-btc-momentum               # dry-run, single strategy
./go-trader backfill hl-fees --all                                     # dry-run, all HL strategies
./go-trader backfill hl-fees --strategy hl-btc-momentum --apply        # apply changes
./go-trader backfill hl-fees --all --apply --reset-cash                # also replay strategies.cash
```

`--apply` refuses to run while another `go-trader` process is alive on the same DB.

---

## Build & Deploy

The canonical update path is `scripts/update.sh` — atomic `git pull --ff-only` → `uv sync` → `go build` (version-stamped) → optional `systemctl restart`, shared by operators and the auto-update DM flow. A startup compatibility probe refuses to launch on a Go/Python version mismatch, so prefer the script over hand-rolled rebuilds.

```bash
sudo bash scripts/update.sh --restart   # update + restart service
bash scripts/update.sh                  # update without restart
```

| Change | Action |
|--------|--------|
| Go or Python source | `sudo bash scripts/update.sh --restart` |
| Config (hot-reloadable subset) | `systemctl kill -s HUP go-trader` — applies capital/drawdown/intervals/params/risk knobs/channels/`allowed_regimes` in place; rejects shape changes (strategy add/remove, type/platform, leverage/`direction` with open positions) |
| Config (roster, script/args/type/platform, `regime` block) | `systemctl restart go-trader` |
| Service file | `systemctl daemon-reload && systemctl restart go-trader` |

---

## Monitoring

```bash
systemctl status go-trader              # service health
curl -s localhost:8099/status            # live prices + P&L (default port 8099; override with --status-port)
curl -s localhost:8099/health            # simple health check
journalctl -u go-trader -n 50           # recent logs
```

Discord strategy summaries show columns `Init | Value | PnL | PnL% | DD | Wallet% | Tf | Int | #T | W/L` plus a `Book Sharpe (realized, annualized)` footer and the go-trader version + PID in the title. `okx-options` and `robinhood-options` channel keys route options summaries separately from spot/perps. `#T`/`W/L` come from the SQLite trades table; partial closes collapse into one round trip per position.

Open-position lines append `SL: $<trigger_px> (<signed_pct>%)` when a Hyperliquid stop-loss trigger is set (percent sign-flipped for shorts so it always reads as the loss if hit), `<N>x ($<margin> margin)` for leveraged perps, and tier marks (`✓`) for filled TP rungs. Spot and 1× perps stay clean.

---

## Risk Management

- **Portfolio kill switch** — halts all trading if portfolio drawdown exceeds threshold (default: 25%); also submits real close orders on Hyperliquid, OKX perps, Robinhood crypto, and TopStep live positions, clearing virtual state only after every platform confirms flat.
- **Notional cap** — optional hard limit on total notional exposure.
- **Correlation tracking** — per-asset directional exposure; warns when one asset exceeds concentration (default: 60%) or too many strategies share a direction (default: 75%). Opt-in via `correlation.enabled`.
- **Per-strategy circuit breakers** — pause on max-drawdown breach (24h cooldown). Spot/options/futures measure peak-relative; perps measure relative to deployed margin so leveraged wipes fire in time. HL perps, OKX perps, Robinhood crypto, and TopStep CBs auto-close on-chain (reduce-only); OKX spot and Robinhood options surface an `operator-required` warning every cycle until flattened by hand.
- **Per-trade Hyperliquid stop-loss** — every HL perps strategy gets an exchange-side reduce-only trigger. Pick one of five mutually-exclusive fields when positive: `stop_loss_pct`, `stop_loss_margin_pct` (price-% = `stop_loss_margin_pct / leverage`), `stop_loss_atr_mult` (fixed ATR distance armed at open), `trailing_stop_pct` (high-water trailing %), `trailing_stop_atr_mult` (ATR-distance trailing frozen at open). All five omitted → arms `default_stop_loss_atr_mult` (default `1.0`); set the top-level field to `0` to opt out fleet-wide, or any per-strategy field to explicit `0` to opt one strategy out. Trailing stops debounce via `trailing_stop_min_move_pct` to stay under HL's 1000-OID account cap.
- **On-chain N-tier TP/SL ladders** — `tiered_tp_atr` / `tiered_tp_atr_live` close evaluators place reduce-only TPs on-chain at configured ATR multiples (default `[{1×, 0.5}, {2×, 1.0}]`); final tier absorbs per-tier rounding dust. On full close, all SL+TP OIDs are cancelled in one shot. Same-coin peers each carry their own SL/TP sized to virtual quantity.
- **Market regime gate** — with `regime.enabled: true`, per-strategy `allowed_regimes` blocks new entries when the current ADX+DI label (`trending_up` / `trending_down` / `ranging`) isn't in the list; close legs always run.
- **HL margin mode** — defaults to `isolated` so one losing strategy can't drain unrelated positions. Override per-strategy with `margin_mode: "cross"`. Applied via `update_leverage` from flat only.
- **Consecutive loss tracking** — 5 losses in a row → 1h pause.
- **Spot**: max 95% capital per position. **Options**: max 4 positions per strategy, portfolio-aware scoring. **Theta harvesting**: configurable early exit on sold options.

---

## TradingView Export

Export recorded SQLite trades to a TradingView portfolio-import CSV (`Symbol,Side,Qty,Status,Fill Price,Commission,Closing Time`):

```bash
./go-trader export tradingview --strategy hl-btc-momentum --output tv-hl-btc.csv
./go-trader export tradingview --strategy hl-btc-momentum --strategy okx-eth-breakout --output tv-selected.csv
./go-trader export tradingview --all --output tv-all.csv
```

Built-in mappings cover known OKX and BinanceUS pairs. For platforms/symbols without a safe default, set per-symbol overrides in config:

```json
"tradingview_export": {
  "symbol_overrides": { "hl:BTC": "BYBIT:BTCUSDT" }
}
```

Circuit-breaker close trades are included; their direction is parsed from the trade's `Details` field ("Close long" → sell, "Close short" → buy).

---

## Trading Fees

| Market | Fee | Slippage |
|--------|-----|----------|
| Binance US Spot | 0.1% taker | ±0.05% |
| Deribit Options | 0.03% of premium | — |
| IBKR/CME Options | $0.25/contract | — |
| Hyperliquid Perps | 0.035% taker | ±0.05% |
| TopStep Futures | Per-contract (configurable) | ±0.05% |
| Robinhood Crypto | No commission (spread embedded) | ±0.05% |
| Robinhood Options | $0.03/contract (regulatory fee) | — |

Live `--execute` fills on Hyperliquid, OKX, Robinhood, and TopStep record the **exchange-reported** fee (and per-leg fees on multi-leg fills) plus the exchange order ID, so backfills and TradingView exports match the venue ledger instead of the calculated estimate (#453, #461).

---

## File Structure

```
go-trader/
├── scheduler/          # Go scheduler, config, state DB, HTTP status, risk, notifications
├── shared_scripts/     # Python entry points called by the scheduler
├── platforms/          # Exchange adapters
├── shared_tools/       # Shared Python utilities
├── shared_strategies/  # Strategy registry and strategy implementations
├── backtest/           # Backtesting and optimization tools
├── systemd/            # Template service units
├── scripts/            # Install/service helper scripts
├── SKILL.md            # AI agent setup guide
└── AGENTS.md           # Agent project context
```

---

## Dependencies

- **Python 3.12+** — managed by [uv](https://github.com/astral-sh/uv) (ccxt, pandas, numpy, scipy, hyperliquid-python-sdk)
- **Go 1.26.2** — [`github.com/bwmarrin/discordgo`](https://github.com/bwmarrin/discordgo) for WebSocket gateway (DM support)
- **systemd** — service management

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| No Discord messages | Check `DISCORD_BOT_TOKEN` env var, channel IDs, bot permissions |
| Service won't start | `journalctl -u go-trader -n 50` |
| Service didn't come back after reboot | Unit was installed but not enabled. Run `sudo bash scripts/install-service.sh` (or `systemctl enable <unit>`) — `systemctl start` alone does not survive reboot. |
| Strategy not trading | Check circuit breaker in `/status`, verify params |
| Reset positions | `rm scheduler/state.db && systemctl restart go-trader` |
| Hyperliquid live mode fails | Set `HYPERLIQUID_SECRET_KEY` env var; paper mode works without it |
| TopStep live mode fails | Set `TOPSTEP_API_KEY`, `TOPSTEP_API_SECRET`, `TOPSTEP_ACCOUNT_ID` env vars |
| Robinhood live mode fails | Set `ROBINHOOD_USERNAME`, `ROBINHOOD_PASSWORD`, `ROBINHOOD_TOTP_SECRET` env vars |
| OKX live mode fails | Set `OKX_API_KEY`, `OKX_API_SECRET`, `OKX_PASSPHRASE` env vars; use `OKX_SANDBOX=1` for demo |
| "state DB missing but live strategies configured" warning on startup | The update process likely wiped the repo directory instead of `git pull`ing in place. Restore `scheduler/state.db` from backup, or set `GO_TRADER_ALLOW_MISSING_STATE=1` for a genuine first-run deployment (#339). |

---

## Risk Disclaimer

This application is provided for informational and educational purposes only. It does not constitute financial advice, investment advice, or a recommendation to buy or sell any asset.

Trading involves substantial risk of loss. Past performance is not indicative of future results. You may lose some or all of your invested capital. Only trade with funds you can afford to lose.

This application is an automated tool and makes no guarantees regarding accuracy, profitability, or outcomes. Market conditions can change rapidly and the application may not react appropriately to all scenarios.

By using this application, you acknowledge that you are solely responsible for your own investment decisions and any resulting gains or losses. The creators, developers, and operators of this application accept no liability for any financial losses incurred through its use.

*This is not financial advice. Trade at your own risk.*
