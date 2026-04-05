# PBGUI / PB7 Local Divergence Note

## Summary

Current local divergence spans both PBGUI and PB7:

1. PB7 actual exposure metrics and scenario-aware optimize limits
2. PBGUI metric/UI wiring for newer PB7 metrics
3. Hyperliquid dashboard/data ingestion fixes
4. Reverse-proxy-friendly dashboard/log viewer routing fixes
5. PB7 optimize-result suite/scenario compatibility fixes

These changes are not limited to PBGUI UI behavior. They include PB7 analysis and optimize-backend behavior, but they do not change PB7 live trading logic or Rust strategy math.

## Root cause

### Hyperliquid dashboard/data

Current deployment uses Hyperliquid in a way upstream PBGUI does not handle cleanly by default:

- live balance/positions are not reliably available from default scope alone
- dashboard total balance is expected to reflect portfolio-level USDC total, not dex-only account value
- HIP-3 symbols must preserve CCXT forms like `XYZ-XYZ100/USDC:USDC`

### PB7 metrics and optimize backend

Local deployment uses PB7 metrics and optimize controls that upstream does not fully expose or evaluate by default:

- realized long/short exposure metrics are needed, not only configured exposure-limit metrics
- optimize limits need optional suite `scenario` targeting
- PBGUI must expose these PB7 metrics consistently across supported payload shapes

### Reverse-proxy dashboard/log routing

Current deployment uses PBGUI behind a reverse proxy. Upstream paths still assume that some dashboard/log viewer flows can safely build browser URLs against `host:8000`. In proxy mode that can resolve to internal/local addresses instead of the browser-visible origin.

### Optimize-result suite/scenario loading

`Optimize Result -> BT selected` loads the selected `pareto/*.json` directly into the backtest editor. Some PB7 result payloads store suite metadata at the top level of `backtest`:

- `backtest.scenarios`
- `backtest.aggregate`
- `backtest.include_base_scenario`
- `backtest.base_label`

PBGUI's editor primarily reads suite data from `backtest.suite.scenarios`, so those result payloads appeared to have empty scenarios even when the optimize run was a suite run.

## Current local divergence from upstream

### PB7 actual exposure metrics and scenario-aware limits

Files:
- [backtest.py](/app/pb7/src/backtest.py)
- [config_utils.py](/app/pb7/src/config_utils.py)
- [limit_utils.py](/app/pb7/src/limit_utils.py)
- [optimize.py](/app/pb7/src/optimize.py)
- [pareto_store.py](/app/pb7/src/pareto_store.py)
- [iterative_backtester.py](/app/pb7/src/tools/iterative_backtester.py)

Current kept behavior:

1. PB7 analysis emits realized side exposure metrics:
   - `wallet_exposure_mean_long`
   - `wallet_exposure_median_long`
   - `wallet_exposure_max_long`
   - `wallet_exposure_mean_short`
   - `wallet_exposure_median_short`
   - `wallet_exposure_max_short`
2. PB7 analysis emits actual-exposure-normalized metrics:
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
3. Short realized exposure is derived from `abs(twe_short)` because raw short TWE is signed negative.
4. PB7 metric allow-lists and objective weight maps include the new actual exposure metrics.
5. Optimize `limits` support an optional `scenario` field for suite runs.
6. When `scenario` is set on a limit entry, `stat` is intentionally disallowed.
7. Scenario-specific limit resolution works both during optimize runtime and during pareto-store reevaluation.

### Hyperliquid ingestion

Files:
- [Exchange.py](/app/pbgui/Exchange.py)
- [Database.py](/app/pbgui/Database.py)
- [PBData.py](/app/pbgui/PBData.py)

Current kept behavior:

1. `Exchange.fetch_positions()` retries Hyperliquid positions with `params={"dex": "xyz"}` when the default scope is empty.
2. `Exchange.fetch_balance()` for Hyperliquid uses `fetch_balance(type="spot") -> total["USDC"]` as dashboard total balance, with dex/default fallback only if needed.
3. PBData does not treat Hyperliquid balance/positions as websocket-fed state; those updates are intentionally handled through REST/shared polling.
4. Hyperliquid price/order symbol mapping preserves recent execution CCXT symbols such as `XYZ-XYZ100/USDC:USDC` instead of reconstructing invalid forms like `XYZXYZ100/USDC:USDC`.
5. Dashboard API display paths recalculate Hyperliquid position uPnl from latest DB price where available, instead of trusting a potentially stale stored `position.upnl` value.

### PB7 metric UI exposure

Files:
- [BacktestV7.py](/app/pbgui/BacktestV7.py)
- [OptimizeV7.py](/app/pbgui/OptimizeV7.py)
- [Config.py](/app/pbgui/Config.py)
- [ParetoExplorer.py](/app/pbgui/ParetoExplorer.py)
- [ParetoVisualizations.py](/app/pbgui/ParetoVisualizations.py)

Current kept behavior:

1. Backtest UI exposes:
   - `ulcer_index`
   - `adg_over_ui`
   - `gain_over_ui`
   - `live.hedge_mode` toggle
2. Optimize UI exposes:
   - `omega_ratio`
   - `peak_recovery_hours`
   - `ulcer_index`
   - `adg_over_ui`
   - `gain_over_ui`
   - `total_wallet_exposure_mean`
   - `gain_per_actual_exposure`
   - `live.hedge_mode` toggle
3. Old-format optimize results do not render missing metrics as misleading `0`; missing values are left blank when the payload does not contain them.
4. EMA span slider max is raised to `20000`.
5. Optimize result table uses shorter display labels for some high-frequency metrics:
   - `drawdown_worst` -> `dd_worst`
   - `omega_ratio` -> `omega`
   - `sharpe_ratio` -> `sharpe`
   - `adg_over_ui` -> `ADG/UI`
   - `gain_over_ui` -> `Gain/UI`
6. Optimize result table supports one custom column path input:
   - metric names such as `gain_per_actual_exposure_usd`
   - config paths such as `bot.long.total_wallet_exposure_limit`
   - live config paths such as `live.hedge_mode`
   - the entered path is also used as the column header
7. Top-level segmented navigation controls use non-empty accessibility labels with collapsed visibility:
   - Backtest -> `Backtest view`
   - Optimize -> `Optimize view`
   - Pareto Explorer -> `Pareto view`
   - this preserves the existing UI while avoiding Streamlit empty-label warnings
8. Optimize queue stop prefers graceful interruption on Linux:
   - sends `SIGINT` to the optimize process group first
   - allows PB7 `optimize.py` to run its `finally` cleanup and unlink SharedMemory segments under `/dev/shm`
   - escalates to stronger termination only if the process does not exit in time
9. Optimize UI supports an optional `starting_config_path`:
   - when `starting_config` is enabled and the path is empty, the current optimize config remains the seed source
   - when the path is set, PBGUI passes that file or directory to PB7 `optimize.py -t`
   - this makes it possible to seed an optimize run from an external config folder without editing the run config itself
10. Optimize UI exposes `sig_digits` for `optimize.round_to_n_significant_digits`:
   - this controls the global precision used for continuous bounds without explicit steps
   - explicit `[low, high, step]` bounds still take precedence over the global setting

### Reverse-proxy-friendly dashboard/log viewer routing

Files:
- [Dashboard.py](/app/pbgui/Dashboard.py)
- [api/dashboard.py](/app/pbgui/api/dashboard.py)
- [frontend/dashboard_main.html](/app/pbgui/frontend/dashboard_main.html)
- [navi/info_dashboards.py](/app/pbgui/navi/info_dashboards.py)
- [navi/system_vps_monitor.py](/app/pbgui/navi/system_vps_monitor.py)
- [pbgui_func.py](/app/pbgui/pbgui_func.py)
- [components/log_viewer/index.html](/app/pbgui/components/log_viewer/index.html)
- [components/vps_monitor/index.html](/app/pbgui/components/vps_monitor/index.html)
- [components/nav_bridge/index.html](/app/pbgui/components/nav_bridge/index.html)

Current kept behavior:

1. Browser-facing FastAPI URLs are resolved via same-origin `/api`, `/app`, `/ws` paths in reverse-proxy mode.
2. Direct `:8501` access still uses explicit `host:8000` API/WS targets for compatibility.
3. Dashboard main page no longer derives browser API/WS bases from server-side `request.url` in proxy mode.
4. Dashboard list loading does not depend on Streamlit session-state objects inside FastAPI routes.
5. Standalone dashboard main page auto-loads the first available dashboard when `current` is empty.
6. Log viewer, VPS monitor, and dashboard nav bridge accept relative `/ws` bases and resolve them against the browser-visible parent origin.
7. Standalone dashboard refresh button reloads the currently visible dashboard, not only the sidebar list.
8. Standalone dashboard view mode includes a 3-minute fallback auto-refresh in case websocket-triggered updates do not arrive.

### Optimize-result suite/scenario compatibility

Files:
- [Config.py](/app/pbgui/Config.py)
- [OptimizeV7.py](/app/pbgui/OptimizeV7.py)
- [BacktestV7.py](/app/pbgui/BacktestV7.py)

Current kept behavior:

1. When a backtest/result payload contains top-level suite fields under `backtest`, PBGUI normalizes them into `backtest.suite` during load.
2. `BT selected` from optimize results can reopen suite runs with scenario labels intact instead of showing an empty scenario table.

## Exact local patch behavior

### Hyperliquid dashboard total balance

File:
- [Exchange.py](/app/pbgui/Exchange.py)

Function:
- `Exchange.fetch_balance()`

Behavior for `hyperliquid`:

1. Try `fetch_balance(params={"type": "spot"})`
2. Return `total["USDC"]`
3. If that fails, fall back to dex/default scope balance fetches

Effect:
- Dashboard total balance reflects portfolio-level USDC total instead of dex-only account value.

### Hyperliquid positions

File:
- [Exchange.py](/app/pbgui/Exchange.py)

Function:
- `Exchange.fetch_positions()`

Behavior:
- Try default positions first
- If empty, retry with `params={"dex": "xyz"}`

Effect:
- PBGUI can read live HIP-3 positions held in dex scope.

### Hyperliquid polling mode

File:
- [PBData.py](/app/pbgui/PBData.py)

Behavior:
- Hyperliquid balance/position watcher tasks are not started
- `hyper` balance/positions are handled by REST/shared combined poll instead

Effect:
- PBData does not wait on nonexistent/unsuitable Hyperliquid WS balance/position streams.

### Hyperliquid symbol preservation

Files:
- [Database.py](/app/pbgui/Database.py)
- [PBData.py](/app/pbgui/PBData.py)

Behavior:
- Recent execution CCXT symbols are used as the source-of-truth fallback symbol set
- Hyperliquid symbol matching maps compact internal symbols back to valid CCXT symbols

Effect:
- Prevents `BadSymbol` errors from invalid reconstructed symbols.

### Dashboard display-side uPnl recalculation

File:
- [api/dashboard.py](/app/pbgui/api/dashboard.py)

Endpoints:
- `/api/dashboard/balance`
- `/api/dashboard/positions_data`

Behavior:
- When a latest DB price exists for a position symbol, dashboard API responses recalculate display `upnl` from:
  - `size`
  - `entry`
  - `side`
  - current DB `price`
- If no current price is available, the stored DB `position.upnl` is used as fallback.

Effect:
- Dashboard `Balance` and `Positions` views are less sensitive to stale `position.upnl` snapshots and better reflect current price-derived PnL.

### PB7 metric display

Files:
- [BacktestV7.py](/app/pbgui/BacktestV7.py)
- [OptimizeV7.py](/app/pbgui/OptimizeV7.py)

Behavior:
- Prefer canonical `_usd` keys where available
- Support legacy/current payload shapes
- For old optimize payloads, try explicit legacy/current keys first and leave missing values blank instead of forcing `0`

### Reverse-proxy-friendly dashboard/log routing

Files:
- [pbgui_func.py](/app/pbgui/pbgui_func.py)
- [navi/info_dashboards.py](/app/pbgui/navi/info_dashboards.py)
- [api/dashboard.py](/app/pbgui/api/dashboard.py)
- [frontend/dashboard_main.html](/app/pbgui/frontend/dashboard_main.html)
- [components/log_viewer/index.html](/app/pbgui/components/log_viewer/index.html)
- [components/vps_monitor/index.html](/app/pbgui/components/vps_monitor/index.html)
- [components/nav_bridge/index.html](/app/pbgui/components/nav_bridge/index.html)

Behavior:
- `_resolve_browser_fastapi_urls()` returns same-origin `/api`, `/app`, `/ws` in proxy mode and explicit `host:8000` only for direct `:8501` access.
- `info_dashboards.dashboard()` redirects using browser-side location logic instead of trusting forwarded headers.
- `/api/dashboard/main_page` emits `API_BASE` and `WS_BASE` appropriate for direct vs proxy mode.
- `/api/dashboard/dashboards` returns dashboard names directly from `data/dashboards/*.json` without relying on Streamlit session state.
- `dashboard_main.html` auto-loads the first dashboard when available.
- Log viewer / VPS monitor / nav bridge resolve relative websocket bases against the browser-visible origin.

### Optimize-result suite/scenario normalization

File:
- [Config.py](/app/pbgui/Config.py)

Function:
- `Backtest.backtest` setter

Behavior:
- Before normal backtest parsing, if the payload has top-level suite fields such as `scenarios`, `aggregate`, `include_base_scenario`, or `base_label`, PBGUI builds/updates `backtest.suite` from those values.
- If `suite_enabled` exists, it is also mapped into `suite.enabled` when needed.

Effect:
- Optimize pareto result files that use `backtest.scenarios` load into the backtest editor with scenarios intact.
- `Optimize Result -> BT selected` no longer drops suite scenario information in the editor.

## Why the divergence is kept

Without these local changes, PBGUI can show:

- `0` or stale Hyperliquid balance
- missing Hyperliquid position/order/price data
- `BadSymbol` errors for HIP-3 symbols
- stale display uPnl even when latest price is already in the DB
- PB7 metrics present in payloads but absent or misleadingly zero in the UI
- proxy-mode dashboard and log pages trying to connect to internal `127.0.0.1:8000` or `localhost:8000` addresses
- suite optimize results reopening in backtest editor with empty scenarios despite the result payload still containing scenario data

## What was intentionally not changed

Not changed:

- PB7 live trading logic
- Rust strategy/risk math
- general dashboard design beyond what was needed for Hyperliquid data correctness

Temporary access/debug handoff notes are not part of the durable state and are intentionally excluded from this summary.

## Validation / expected runtime behavior

Expected current runtime behavior:

- Hyperliquid balance in `pbgui.db` is near portfolio-level USDC total (`~25500` in the recent observed state)
- Hyperliquid position rows exist in `pbgui.db`
- Hyperliquid orders rows exist when live open orders exist
- Hyperliquid price mapping uses valid CCXT symbols
- Dashboard `uPnl` reflects latest DB price when available, not only the last stored position snapshot
- Old optimize results with missing metrics show blanks rather than false zeros
- Reverse-proxy access works when `/api/*`, `/ws/*`, `/app/*`, and `/health` are routed to port `8000`, with the remaining app traffic routed to Streamlit on `8501`
- Loading a suite pareto result through `BT selected` shows populated scenario labels instead of an empty scenario list

## Operational guidance

When updating upstream:

1. Re-check [Exchange.py](/app/pbgui/Exchange.py), [Database.py](/app/pbgui/Database.py), and [PBData.py](/app/pbgui/PBData.py) first.
2. Re-check [BacktestV7.py](/app/pbgui/BacktestV7.py) and [OptimizeV7.py](/app/pbgui/OptimizeV7.py) for metric wiring regressions.
3. Re-check [pbgui_func.py](/app/pbgui/pbgui_func.py), [api/dashboard.py](/app/pbgui/api/dashboard.py), and [navi/info_dashboards.py](/app/pbgui/navi/info_dashboards.py) if proxy-mode dashboard/log routing regresses.
4. If `Information -> Dashboards` or log viewer fails again behind a proxy, verify `/api/*`, `/ws/*`, `/app/*`, and `/health` reach PBApiServer on port `8000`.
5. If optimize metrics revert to `0`, inspect the actual pareto payload shape before changing UI keys.
6. If suite results reopen with empty scenarios again, inspect whether PB7 changed the optimize-result schema away from `backtest.suite` or `backtest.scenarios`.

## Long-term proper fix

The proper long-term fix is to upstream these behaviors or replace them with cleaner shared contracts:

- Hyperliquid ingestion should have an upstream-safe contract for spot/dex/default scope semantics.
- PBGUI should not need deployment-specific symbol-preservation logic for HIP-3.
- PB7/PBGUI metric payload handling should converge so old-format optimize results do not require special missing-metric handling.
- Dashboard/log viewer routing should be origin/path-based upstream by default, without direct `host:8000` assumptions in proxy mode.
- PB7 optimize-result schema and PBGUI config loader should converge on one stable suite format so result reopening does not need compatibility normalization.

## 2026-03-26: Actual side exposure metrics for optimize limits

### Context

Existing `*_per_exposure_{long,short}` metrics divide by configured `bot.{side}.total_wallet_exposure_limit`, not by realized exposure observed during the backtest.

### Local change

- Added realized side exposure metrics derived from the backtest `twe_long` / `twe_short` state series:
  - `wallet_exposure_mean_long`
  - `wallet_exposure_median_long`
  - `wallet_exposure_max_long`
  - `wallet_exposure_mean_short`
  - `wallet_exposure_median_short`
  - `wallet_exposure_max_short`
- Added realized-exposure-normalized return metrics:
  - `adg_per_actual_exposure`, `adg_w_per_actual_exposure`
  - `mdg_per_actual_exposure`, `mdg_w_per_actual_exposure`
  - `gain_per_actual_exposure`
  - `adg_per_actual_exposure_long`, `adg_per_actual_exposure_short`
  - `adg_w_per_actual_exposure_long`, `adg_w_per_actual_exposure_short`
  - `mdg_per_actual_exposure_long`, `mdg_per_actual_exposure_short`
  - `mdg_w_per_actual_exposure_long`, `mdg_w_per_actual_exposure_short`
  - `gain_per_actual_exposure_long`, `gain_per_actual_exposure_short`
- Registered these metrics in PB7/PBGui metric registries so they can be used in optimize limits and appear in result payloads.
- Short exposure uses `abs(twe_short)` because PB7 stores short TWE as signed negative in the raw state series.

### Impact

- Optimize/backtest analysis payloads now expose realized long/short exposure directly.
- Optimize limits can target realized side exposure metrics instead of only configured TWEL-derived metrics.
- Existing `*_per_exposure_*` metrics remain unchanged for backward compatibility.
- Optimize limits now also accept an optional `scenario` field for suite runs.
- When `scenario` is set, the limit resolves against that scenario label only and `stat` is intentionally disallowed.

### Metric meaning

- `*_per_exposure_{long,short}`:
  - Denominator is configured `bot.{side}.total_wallet_exposure_limit`
  - Use when the intent is "return relative to allowed max capital budget"
- `*_per_actual_exposure_{long,short}`:
  - Denominator is realized `wallet_exposure_mean_{side}`
  - Use when the intent is "return relative to actual average deployed capital"
- `*_per_actual_exposure`:
  - Denominator is realized `total_wallet_exposure_mean`
  - Use when the intent is "return relative to actual average deployed capital across both sides"
- `wallet_exposure_mean/max/median_{long,short}`:
  - Realized side exposure statistics from the backtest state series
  - Use directly in optimize limits when capping or targeting actual long/short capital usage

## Upstream full-pull reapply guide

This section is intended for the case where upstream PBGUI/PB7 is pulled in full later and the local behavior in this document must be restored with minimal rediscovery work.

### Recommended strategy

Do not try to reapply everything at once without checking upstream behavior first.

Recommended order:

1. Pull upstream into `pb7` and `pbgui`
2. Run the app once without local reapply patches
3. Diff the updated upstream files against the behaviors documented here
4. Reapply only the divergences still needed
5. Verify runtime behavior before moving to the next divergence family

Reason:

- some local patches may already be present upstream later
- some local patches may need to move because upstream refactored file boundaries
- the highest-risk conflicts are in config/optimize/metric wiring, so reapplying in smaller slices is safer

### Reapply order

Reapply in this order unless there is a strong reason not to:

1. PB7 metric and optimize backend changes
2. PBGui metric registry and UI wiring
3. Optimize-result suite/scenario compatibility
4. Hyperliquid data/dashboard behavior
5. Reverse-proxy routing behavior
6. Help text and divergence documentation cleanup

This order matters because:

- PBGui metric UI depends on PB7 metric names existing
- optimize-result compatibility is isolated and can be checked independently
- Hyperliquid and reverse-proxy changes are deployment-specific and should be applied last

### Files to inspect first after upstream pull

PB7:

- [backtest.py](/app/pb7/src/backtest.py)
- [config_utils.py](/app/pb7/src/config_utils.py)
- [limit_utils.py](/app/pb7/src/limit_utils.py)
- [optimize.py](/app/pb7/src/optimize.py)
- [pareto_store.py](/app/pb7/src/pareto_store.py)
- [iterative_backtester.py](/app/pb7/src/tools/iterative_backtester.py)

PBGui:

- [Config.py](/app/pbgui/Config.py)
- [OptimizeV7.py](/app/pbgui/OptimizeV7.py)
- [ParetoExplorer.py](/app/pbgui/ParetoExplorer.py)
- [BacktestV7.py](/app/pbgui/BacktestV7.py)
- [pbgui_help.py](/app/pbgui/pbgui_help.py)
- [Exchange.py](/app/pbgui/Exchange.py)
- [Database.py](/app/pbgui/Database.py)
- [PBData.py](/app/pbgui/PBData.py)
- [api/dashboard.py](/app/pbgui/api/dashboard.py)
- [pbgui_func.py](/app/pbgui/pbgui_func.py)
- [navi/info_dashboards.py](/app/pbgui/navi/info_dashboards.py)
- [frontend/dashboard_main.html](/app/pbgui/frontend/dashboard_main.html)
- [components/log_viewer/index.html](/app/pbgui/components/log_viewer/index.html)
- [components/vps_monitor/index.html](/app/pbgui/components/vps_monitor/index.html)
- [components/nav_bridge/index.html](/app/pbgui/components/nav_bridge/index.html)

### Reapply package 1: PB7 actual exposure metrics and scenario-aware limits

Goal:

- restore actual long/short exposure metrics
- restore actual-exposure-normalized return metrics
- restore suite `limits[].scenario`

Files:

- [backtest.py](/app/pb7/src/backtest.py)
- [config_utils.py](/app/pb7/src/config_utils.py)
- [limit_utils.py](/app/pb7/src/limit_utils.py)
- [optimize.py](/app/pb7/src/optimize.py)
- [pareto_store.py](/app/pb7/src/pareto_store.py)
- [iterative_backtester.py](/app/pb7/src/tools/iterative_backtester.py)

Required behavior:

1. Backtest analysis must emit:
   - `wallet_exposure_mean_long`
   - `wallet_exposure_median_long`
   - `wallet_exposure_max_long`
   - `wallet_exposure_mean_short`
   - `wallet_exposure_median_short`
   - `wallet_exposure_max_short`
2. Backtest analysis must emit:
   - `gain_per_actual_exposure`
   - `adg_per_actual_exposure`
   - `adg_w_per_actual_exposure`
   - `mdg_per_actual_exposure`
   - `mdg_w_per_actual_exposure`
3. Backtest analysis must emit:
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
4. Short realized exposure must be computed from `abs(twe_short)`, not from positive clamp logic
5. New metrics must be registered in PB7 metric allow-lists and scoring weights
6. `optimize.limits[]` must support optional `scenario`
7. If `scenario` is set, `stat` must be rejected
8. Scenario-specific limits must resolve against the selected suite scenario label in both optimize runtime and pareto-store reevaluation

Implementation notes:

- `backtest.py` is the source of truth for metric computation
- `config_utils.py` is the highest-conflict file because upstream also changes config cleaning and limit preservation there
- if upstream later adds its own `optimize.limits` preservation helpers, keep those and reapply only:
  - `scenario` field preservation
  - `scenario` plus `stat` validation
- `pareto_store.py` must be checked separately because optimize-time and post-hoc pareto-limit reevaluation use different code paths

Minimum verification:

1. Run a synthetic or local backtest payload through analysis expansion and confirm all new metric keys are present
2. Confirm `wallet_exposure_mean_short` is nonzero for a test case where raw `twe_short` is negative but active
3. Confirm a limit such as:
   - `{"metric":"drawdown_worst_usd","scenario":"Stress_COVID","penalize_if":"greater_than","value":0.25}`
   is accepted
4. Confirm:
   - `{"metric":"drawdown_worst_usd","scenario":"Stress_COVID","stat":"max","penalize_if":"greater_than","value":0.25}`
   raises an error
5. Confirm pareto reevaluation resolves scenario-specific values instead of aggregate stats only

### Reapply package 2: PBGui metric registry, score/limit exposure, and presets

Goal:

- make the PB7 metrics visible and selectable in PBGui
- keep metric descriptions accurate
- keep exposure-related explorer presets aligned with actual exposure metrics

Files:

- [Config.py](/app/pbgui/Config.py)
- [OptimizeV7.py](/app/pbgui/OptimizeV7.py)
- [ParetoExplorer.py](/app/pbgui/ParetoExplorer.py)
- [BacktestV7.py](/app/pbgui/BacktestV7.py)
- [pbgui_help.py](/app/pbgui/pbgui_help.py)

Required behavior:

1. Metric registry exposes all new actual exposure metrics
2. Aggregate metric registry includes:
   - `wallet_exposure_mean_long`
   - `wallet_exposure_median_long`
   - `wallet_exposure_max_long`
   - `wallet_exposure_mean_short`
   - `wallet_exposure_median_short`
   - `wallet_exposure_max_short`
3. Metric descriptions clearly distinguish:
   - configured exposure limit based metrics
   - realized actual exposure based metrics
4. Optimize `limits` editor includes a `Scenario` selector
5. When `Scenario` is selected, `Stat` is disabled
6. Pareto explorer efficiency presets prefer actual exposure metrics where intended
7. Backtest/optimize UI should leave truly missing old-format values blank instead of forcing misleading `0`
8. Optimize result table should expose `gain_per_actual_exposure` when present and keep the shortened display labels `dd_worst`, `omega`, `sharpe`, `ADG/UI`, and `Gain/UI`
9. Optimize result table should support one custom column path input for either metrics or config paths, using the entered path as the column header
10. Backtest and optimize editors should expose a `hedge_mode` toggle wired to `live.hedge_mode`

Implementation notes:

- [Config.py](/app/pbgui/Config.py) is the primary source of PBGui metric definitions
- [OptimizeV7.py](/app/pbgui/OptimizeV7.py) uses registry-driven metric options, so missing metrics there usually mean the registry was not restored correctly
- [ParetoExplorer.py](/app/pbgui/ParetoExplorer.py) must be checked if presets silently revert to configured-limit metrics

Minimum verification:

1. Open PBGui and confirm new metrics appear in score/limit selectors
2. Confirm `Scenario` dropdown appears in suite optimize limit editing
3. Confirm selecting a scenario disables `Stat`
4. Confirm Pareto explorer can use actual exposure preset metrics without blank axes or missing-key errors

### Reapply package 3: Optimize-result suite/scenario compatibility

Goal:

- preserve the ability to reopen suite optimize results through `BT selected`

Files:

- [Config.py](/app/pbgui/Config.py)
- [OptimizeV7.py](/app/pbgui/OptimizeV7.py)
- [BacktestV7.py](/app/pbgui/BacktestV7.py)

Required behavior:

1. If a result payload stores suite data as top-level `backtest.scenarios`, `backtest.aggregate`, `backtest.include_base_scenario`, `backtest.base_label`, or `backtest.suite_enabled`, load-time normalization must rebuild `backtest.suite`
2. `Optimize Result -> BT selected` must reopen the result with populated scenario rows

Implementation notes:

- the critical logic lives in the [Config.py](/app/pbgui/Config.py) backtest loader
- this patch is independent from the actual exposure work and can be restored separately

Minimum verification:

1. Load a known suite pareto result JSON that uses top-level `backtest.scenarios`
2. Trigger `BT selected`
3. Confirm scenario labels are present in the backtest editor

### Reapply package 4: Hyperliquid data/dashboard behavior

Goal:

- restore the current deployment's Hyperliquid-specific data correctness behavior

Files:

- [Exchange.py](/app/pbgui/Exchange.py)
- [Database.py](/app/pbgui/Database.py)
- [PBData.py](/app/pbgui/PBData.py)
- [api/dashboard.py](/app/pbgui/api/dashboard.py)

Required behavior:

1. Hyperliquid total balance prefers `fetch_balance(type="spot") -> total["USDC"]`
2. Hyperliquid positions retry with `params={"dex":"xyz"}` when default scope is empty
3. Hyperliquid balance/position updates are treated as REST/shared polling behavior, not websocket-driven watcher behavior
4. Hyperliquid symbol mapping preserves valid CCXT HIP-3 symbols
5. Dashboard API display-side `uPnl` is recalculated from latest DB price when available

Implementation notes:

- these changes are deployment-specific and may or may not belong upstream later
- if upstream changes Hyperliquid adapters substantially, verify behavior rather than matching old code literally

Minimum verification:

1. Hyperliquid balance in DB/API is near expected portfolio-level USDC total
2. Position rows exist when live positions exist
3. Order rows exist when live orders exist
4. No `BadSymbol` errors occur for HIP-3 symbols
5. Dashboard `uPnl` reflects current DB price rather than only stored snapshot values

### Reapply package 5: Reverse-proxy-friendly dashboard and log routing

Goal:

- preserve same-origin proxy-safe behavior for dashboard and log-related pages

Files:

- [pbgui_func.py](/app/pbgui/pbgui_func.py)
- [api/dashboard.py](/app/pbgui/api/dashboard.py)
- [navi/info_dashboards.py](/app/pbgui/navi/info_dashboards.py)
- [frontend/dashboard_main.html](/app/pbgui/frontend/dashboard_main.html)
- [components/log_viewer/index.html](/app/pbgui/components/log_viewer/index.html)
- [components/vps_monitor/index.html](/app/pbgui/components/vps_monitor/index.html)
- [components/nav_bridge/index.html](/app/pbgui/components/nav_bridge/index.html)

Required behavior:

1. Reverse-proxy mode uses browser-facing same-origin `/api`, `/app`, and `/ws`
2. Direct `:8501` access still supports explicit `host:8000` API/WS targets
3. Dashboard main page does not derive proxy browser URLs from internal server request assumptions
4. Dashboard list loading does not depend on Streamlit session state inside FastAPI routes
5. Dashboard main page auto-loads the first available dashboard when no explicit dashboard is selected
6. Log viewer, VPS monitor, and nav bridge accept relative websocket bases
7. Dashboard refresh should reload the active dashboard view, not only refresh the sidebar dashboard list
8. Dashboard view mode should have a fallback auto-refresh timer of 3 minutes for stale websocket/update cases

Minimum verification:

1. Through proxy, dashboard main page loads without trying to hit `127.0.0.1:8000` or `localhost:8000`
2. Through proxy, log viewer websocket connects successfully
3. Through proxy, dashboard list is populated
4. Through direct `:8501` access, old direct mode still works

### Reapply package 6: Documentation and help text

Goal:

- keep the user-facing meaning of the local divergence understandable
- avoid future rediscovery work

Files:

- [pbgui_help.py](/app/pbgui/pbgui_help.py)
- [PBGUI_LOCAL_DIVERGENCE_NOTE.md](/app/pbgui/data/PBGUI_LOCAL_DIVERGENCE_NOTE.md)

Required behavior:

1. Help text explains configured exposure versus actual exposure metrics
2. Help text explains scenario-specific limits and the `stat` restriction
3. This divergence note remains current after any reapply pass

### Conflict hotspots to expect

These files are the most likely merge-conflict or semantic-conflict points after a future full upstream pull:

- [config_utils.py](/app/pb7/src/config_utils.py)
- [optimize.py](/app/pb7/src/optimize.py)
- [Config.py](/app/pbgui/Config.py)
- [OptimizeV7.py](/app/pbgui/OptimizeV7.py)
- [api/dashboard.py](/app/pbgui/api/dashboard.py)

Why:

- `config_utils.py` changes frequently upstream for config normalization and template cleanup
- `optimize.py` changes whenever scoring/fitness/limits behavior moves
- `Config.py` is the PBGui metric registry and config parsing hub
- `OptimizeV7.py` is the PBGui limit UI and often changes with form/state refactors
- `api/dashboard.py` is a natural conflict point for deployment-specific routing/data behavior

### Suggested reapply workflow

Use this workflow for the least painful future reapply:

1. Create a temporary branch after the full upstream pull
2. Restore PB7 backend package first
3. Run targeted PB7 verification before touching PBGui
4. Restore PBGui metric/UI package
5. Verify optimize/backtest screens
6. Restore suite-result compatibility patch
7. Verify `BT selected`
8. Restore Hyperliquid behavior
9. Verify dashboard data correctness
10. Restore reverse-proxy behavior
11. Verify proxy and direct-access modes separately
12. Update this note with any path or function moves discovered during reapply

### Suggested commit split

If reapplying after a full upstream pull, keep the work split into separate commits:

1. `pb7: restore actual exposure metrics and scenario-aware limits`
2. `pbgui: restore actual exposure metric registry and scenario limit ui`
3. `pbgui: restore suite optimize-result scenario normalization`
4. `pbgui: restore hyperliquid dashboard/data behavior`
5. `pbgui: restore reverse-proxy dashboard/log routing`
6. `docs: update local divergence note`

This makes the next future reapply much easier because each divergence family stays isolated.
