# PBGUI Local Divergence Note

## Summary

PBGUI keeps two local divergences from upstream:

1. Hyperliquid dashboard/data ingestion fixes
2. PB7 metric UI exposure fixes

These are UI/data-ingestion changes only. They do not change PB7 trading logic.

## Root cause

### Hyperliquid dashboard/data

Current deployment uses Hyperliquid in a way upstream PBGUI does not handle cleanly by default:

- live balance/positions are not reliably available from default scope alone
- dashboard total balance is expected to reflect portfolio-level USDC total, not dex-only account value
- HIP-3 symbols must preserve CCXT forms like `XYZ-XYZ100/USDC:USDC`

### PB7 metric UI

PB7 emits newer analysis metrics that upstream PBGUI does not fully expose across all supported result payload shapes.

## Current local divergence from upstream

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
2. Optimize UI exposes:
   - `omega_ratio`
   - `peak_recovery_hours`
   - `ulcer_index`
   - `adg_over_ui`
   - `gain_over_ui`
   - `total_wallet_exposure_mean`
3. Old-format optimize results do not render missing metrics as misleading `0`; missing values are left blank when the payload does not contain them.
4. EMA span slider max is raised to `20000`.

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

### PB7 metric display

Files:
- [BacktestV7.py](/app/pbgui/BacktestV7.py)
- [OptimizeV7.py](/app/pbgui/OptimizeV7.py)

Behavior:
- Prefer canonical `_usd` keys where available
- Support legacy/current payload shapes
- For old optimize payloads, try explicit legacy/current keys first and leave missing values blank instead of forcing `0`

## Why the divergence is kept

Without these local changes, PBGUI can show:

- `0` or stale Hyperliquid balance
- missing Hyperliquid position/order/price data
- `BadSymbol` errors for HIP-3 symbols
- PB7 metrics present in payloads but absent or misleadingly zero in the UI

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
- Old optimize results with missing metrics show blanks rather than false zeros

## Operational guidance

When updating upstream:

1. Re-check [Exchange.py](/app/pbgui/Exchange.py), [Database.py](/app/pbgui/Database.py), and [PBData.py](/app/pbgui/PBData.py) first.
2. Re-check [BacktestV7.py](/app/pbgui/BacktestV7.py) and [OptimizeV7.py](/app/pbgui/OptimizeV7.py) for metric wiring regressions.
3. If `Information -> Dashboards` fails again, verify PBApiServer on port `8000` is reachable from the browser path being used.
4. If optimize metrics revert to `0`, inspect the actual pareto payload shape before changing UI keys.

## Long-term proper fix

The proper long-term fix is to upstream these behaviors or replace them with cleaner shared contracts:

- Hyperliquid ingestion should have an upstream-safe contract for spot/dex/default scope semantics.
- PBGUI should not need deployment-specific symbol-preservation logic for HIP-3.
- PB7/PBGUI metric payload handling should converge so old-format optimize results do not require special missing-metric handling.
