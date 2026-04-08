# PBGUI Upstream Divergence

## Purpose

This is the single reference for PBGUI behavior that must be preserved on top
of `upstream/main`.

Use it when rebuilding PBGUI from a fresh upstream checkout. The goal is not to
list every historical edit, but to preserve the production behavior that still
matters.

## Rebuild Strategy

Start from `upstream/main`, then restore the following divergence packages:

1. Metric and config-surface parity with rebased PB7
2. Optimize and backtest workflow UX
3. Reverse-proxy and standalone dashboard behavior
4. Hyperliquid dashboard ingestion fallbacks
5. Pareto Explorer UX
6. Small runtime and accessibility fixes

## Package 1: Metric And Config-Surface Parity

### Files

- [Config.py](/app/pbgui/Config.py)
- [OptimizeV7.py](/app/pbgui/OptimizeV7.py)
- [BacktestV7.py](/app/pbgui/BacktestV7.py)
- [pbgui_help.py](/app/pbgui/pbgui_help.py)

### Required behavior

1. The PBGUI metric registry must expose the rebased PB7 metric set used in
   production.

   Local actual-exposure family:
   - `wallet_exposure_mean_long`
   - `wallet_exposure_median_long`
   - `wallet_exposure_max_long`
   - `wallet_exposure_mean_short`
   - `wallet_exposure_median_short`
   - `wallet_exposure_max_short`
   - `gain_per_actual_exposure`
   - `adg_per_actual_exposure`
   - `adg_w_per_actual_exposure`
   - `mdg_per_actual_exposure`
   - `mdg_w_per_actual_exposure`
   - `gain_per_actual_exposure_long`
   - `gain_per_actual_exposure_short`
   - `adg_per_actual_exposure_long`
   - `adg_per_actual_exposure_short`
   - `adg_w_per_actual_exposure_long`
   - `adg_w_per_actual_exposure_short`
   - `mdg_per_actual_exposure_long`
   - `mdg_per_actual_exposure_short`
   - `mdg_w_per_actual_exposure_long`
   - `mdg_w_per_actual_exposure_short`

   Local ulcer/UI family:
   - `ulcer_index`
   - `adg_over_ui`
   - `gain_over_ui`

   Upstream ratio/trade family:
   - `paper_loss_ratio`
   - `paper_loss_mean_ratio`
   - `exposure_ratio`
   - `exposure_mean_ratio`
   - `win_rate`
   - `trade_loss_max`
   - `trade_loss_mean`
   - `trade_loss_median`

   Existing shared exposure metric still used in UI:
   - `total_wallet_exposure_mean`

2. Preserve the semantic split between similarly named metric families:
   - `*_per_exposure_*` means configured `total_wallet_exposure_limit`
   - `*_per_actual_exposure*` means realized mean exposure

3. PBGUI must expose the full current PB7 config surface that matters for
   optimize/backtest workflows.

   Optimize-only workflow fields:
   - `starting_config_path`
   - `round_to_n_significant_digits`

   Shared live/backtest control fields:
   - `live.hedge_mode`
   - `backtest.liquidation_threshold`
   - `backtest.market_order_slippage_pct`
   - `live.market_order_near_touch_threshold`
   - `live.margin_mode_preference`
   - `live.hsl_signal_mode`
   - `live.hsl_position_during_cooldown_policy`

4. Optimize bounds UI must expose the newer PB7 bound families:
   - `forager_score_weights_*`
   - `forager_volatility_ema_span`
   - `forager_volume_drop_pct`
   - `forager_volume_ema_span`
   - `hsl_cooldown_minutes_after_red`
   - `hsl_ema_span_minutes`
   - `hsl_red_threshold`

5. Bot editing must expose side-level HSL runtime fields so PB7 defaults are
   not silently injected without PBGUI visibility:
   - `bot.{long,short}.hsl_enabled`
   - `bot.{long,short}.hsl_no_restart_drawdown_threshold`
   - `bot.{long,short}.hsl_orange_tier_mode`
   - `bot.{long,short}.hsl_panic_close_order_type`
   - `bot.{long,short}.hsl_tier_ratios.{yellow,orange}`

## Package 2: Optimize And Backtest Workflow UX

### Files

- [Config.py](/app/pbgui/Config.py)
- [OptimizeV7.py](/app/pbgui/OptimizeV7.py)
- [BacktestV7.py](/app/pbgui/BacktestV7.py)
- [pbgui_help.py](/app/pbgui/pbgui_help.py)

### Required behavior

1. Optimize results must show the local performance columns used in production:
   - `gain_per_actual_exposure`
   - `ulcer_index`
   - `adg_over_ui`
   - `gain_over_ui`
   - `total_wallet_exposure_mean`

2. Backtest result views must show:
   - `ulcer_index`
   - `adg_over_ui`
   - `gain_over_ui`

3. Optimize result labels must stay shortened:
   - `drawdown_worst` -> `dd_worst`
   - `omega_ratio` -> `omega`
   - `sharpe_ratio` -> `sharpe`
   - `adg_over_ui` -> `ADG/UI`
   - `gain_over_ui` -> `Gain/UI`
   - `gain_per_actual_exposure` -> `Gain/ActExp`

4. Optimize results must support one extra custom column path:
   - metric names such as `gain_per_actual_exposure_usd`
   - config paths such as `bot.long.total_wallet_exposure_limit`
   - live config paths such as `live.hedge_mode`

5. Optimize limits UI must support optional suite `scenario`.
   When `scenario` is set, `stat` must be disabled.

6. Old optimize payloads that do not contain a metric must not render a
   misleading `0`.

7. Optimize-result payloads that stored suite data at top-level under
   `backtest` must be normalized back into `backtest.suite` when reloaded into
   the editor.

## Package 3: Reverse-Proxy And Standalone Dashboard Behavior

### Files

- [pbgui_func.py](/app/pbgui/pbgui_func.py)
- [api/dashboard.py](/app/pbgui/api/dashboard.py)
- [frontend/dashboard_main.html](/app/pbgui/frontend/dashboard_main.html)
- [navi/info_dashboards.py](/app/pbgui/navi/info_dashboards.py)
- [components/log_viewer/index.html](/app/pbgui/components/log_viewer/index.html)
- [components/vps_monitor/index.html](/app/pbgui/components/vps_monitor/index.html)
- [components/nav_bridge/index.html](/app/pbgui/components/nav_bridge/index.html)

### Required behavior

1. Browser-facing API/app/WS bases must resolve correctly for both:
   - direct `:8501` access
   - same-origin reverse-proxy access

2. In reverse-proxy mode, browser-facing paths must stay relative:
   - `/api`
   - `/app`
   - `/ws`

3. Standalone dashboard refresh must reload the visible iframe, not only the
   dashboard list.

4. Standalone dashboard view mode must have a 3-minute fallback auto-refresh.

5. FastAPI dashboard list loading must not rely on Streamlit session objects.

6. Log viewer, VPS monitor, and nav bridge must accept relative websocket
   bases.

## Package 4: Hyperliquid Dashboard Ingestion Fallbacks

### Files

- [Database.py](/app/pbgui/Database.py)
- [Exchange.py](/app/pbgui/Exchange.py)
- [PBData.py](/app/pbgui/PBData.py)
- [api/dashboard.py](/app/pbgui/api/dashboard.py)

### Required behavior

1. `Exchange.fetch_positions()` retries Hyperliquid positions with
   `params={"dex": "xyz"}` when the default scope is empty.

2. `Exchange.fetch_balance()` for Hyperliquid prefers spot `total["USDC"]`,
   then falls back to dex/default only if needed.

3. Dashboard symbol recovery must preserve valid CCXT symbols such as
   `XYZ-XYZ100/USDC:USDC`.

4. Recent execution symbols must be used as fallback for orders, prices, and
   empty-position reconstruction.

5. Dashboard-side uPnl should use the newest DB price when available instead of
   trusting stale stored `position.upnl`.

## Package 5: Pareto Explorer UX

### Files

- [ParetoExplorer.py](/app/pbgui/ParetoExplorer.py)
- [ParetoVisualizations.py](/app/pbgui/ParetoVisualizations.py)

### Required behavior

1. The actual-exposure metric family must remain the preferred efficiency
   preset:
   - `adg_per_actual_exposure_long`
   - `adg_w_per_actual_exposure_long`

2. Metric help text must include:
   - `ulcer_index_usd`
   - `adg_over_ui_usd`
   - `gain_over_ui_usd`
   - `total_wallet_exposure_mean`

3. Top-metric cards must include:
   - `Top ADG/UI`
   - `Best UI`
   - `Top Gain/UI`
   - `Low Avg TWE`

4. Keep support for upstream structured scoring items.

## Package 6: Small Runtime And Accessibility Fixes

### Files

- [OptimizeV7.py](/app/pbgui/OptimizeV7.py)
- [navi/v7_backtest.py](/app/pbgui/navi/v7_backtest.py)
- [navi/v7_optimize.py](/app/pbgui/navi/v7_optimize.py)
- [navi/v7_pareto_explorer.py](/app/pbgui/navi/v7_pareto_explorer.py)

### Required behavior

1. Optimize queue stop on Linux must prefer graceful unwind:
   - `SIGINT`
   - then `SIGTERM`
   - kill only as last resort

2. Streamlit segmented controls must use non-empty labels with
   `label_visibility="collapsed"`.

## Intentional Non-Reapplications

1. Old in-Streamlit dashboard editor behavior when the current
   FastAPI/standalone flow is cleaner.
2. Older PB6 navigation or legacy service wiring from old local
   `pbgui_func.py`.
3. Local-only wording churn in help text that does not change behavior.

## Validation Checklist

1. `py_compile` passes for:
   - `Config.py`
   - `OptimizeV7.py`
   - `BacktestV7.py`
   - `pbgui_help.py`
   - `pbgui_func.py`
   - `api/dashboard.py`
   - `navi/info_dashboards.py`
   - `ParetoExplorer.py`
   - `Database.py`
   - `Exchange.py`
   - `PBData.py`

2. `streamlit run pbgui.py` starts cleanly.

3. Optimize queue command generation includes:
   - rebased PB7 `optimize.py`
   - optional `-t <starting_config_path>`

4. Metric registry contains at least:
   - `gain_per_actual_exposure`
   - `ulcer_index`
   - `wallet_exposure_mean_short`
   - `adg_over_ui`
   - `paper_loss_ratio`

5. Suite payload normalization restores `backtest.suite`.

6. Dashboard main page emits proxy-safe relative `/api` and `/ws` in proxy
   mode.

7. Hyperliquid dashboard views still resolve valid CCXT symbols.

## Status

This document is the current upstream-based reference for PBGUI divergence.
