# PBGUI PB7 Metrics Local Divergence From Upstream

## Summary

PBGUI currently has a small local divergence from upstream to expose newer PB7 analysis metrics in the UI.

This divergence is UI/plumbing only. It does not create metrics itself. PB7 Rust remains the source of truth.

## Current local divergence from upstream

PBGUI currently differs from upstream in two ways:

1. Backtest UI exposes ulcer-related metrics with `_usd`-first and legacy fallback handling.
2. Optimize UI reads and displays newer PB7 metrics across multiple legacy/current payload shapes.

Current local UI changes cover:

- `ulcer_index`
- `adg_over_ui`
- `gain_over_ui`
- optimize-result table updates that also include:
  - `omega_ratio`
  - `peak_recovery_hours`
  - `total_wallet_exposure_mean`

## Why this divergence exists

PB7 now emits newer analysis metrics in backtest and optimize payloads, but upstream PBGUI did not expose all of them in the result views used here.

Local PBGUI was updated so:

- backtest result rows can display the new ulcer-related metrics
- optimize result rows can read the new metrics across multiple payload shapes

Without this divergence:

- PB7 payloads could contain the metrics
- but PBGUI tables would not show them

## Exact files changed

- [BacktestV7.py](/app/pbgui/BacktestV7.py)
- [OptimizeV7.py](/app/pbgui/OptimizeV7.py)

## Current local divergence from upstream

### 1. Backtest result parsing exposes ulcer-related metrics

File:

- [BacktestV7.py](/app/pbgui/BacktestV7.py)

Class / area:

- `BacktestV7Result`

Current local behavior:

- reads `ulcer_index_usd` with fallback to `ulcer_index`
- reads `adg_over_ui_usd` with fallback to `adg_over_ui`
- reads `gain_over_ui_usd` with fallback to `gain_over_ui`

This allows PBGUI to support both:

- current PB7 `_usd` payloads
- older/legacy unsuffixed payloads

### 2. Backtest result table shows ulcer-related columns

File:

- [BacktestV7.py](/app/pbgui/BacktestV7.py)

Current local behavior:

- adds table columns:
  - `Ulcer Index`
  - `ADG/UI`
  - `Gain/UI`
- adds these fields to the sort dropdown

### 3. Optimize result rows expose newer PB7 metrics

File:

- [OptimizeV7.py](/app/pbgui/OptimizeV7.py)

Current local behavior:

- in suite payloads, non-suite payloads, `analyses_combined`, and single-analysis legacy paths, PBGUI now reads:
  - `omega_ratio`
  - `peak_recovery_hours`
  - `ulcer_index`
  - `adg_over_ui`
  - `gain_over_ui`
  - `total_wallet_exposure_mean`

Canonical metric keys used where available:

- `omega_ratio_usd`
- `peak_recovery_hours_equity_usd`
- `ulcer_index_usd`
- `adg_over_ui_usd`
- `gain_over_ui_usd`

### 4. Optimize result column set differs from upstream

File:

- [OptimizeV7.py](/app/pbgui/OptimizeV7.py)

Current local row set no longer matches older upstream columns exactly.

Local rows replaced:

- `mdg`
- `loss_profit_ratio`
- `position_held_hours_max`

with:

- `omega_ratio`
- `peak_recovery_hours`
- `ulcer_index`
- `adg_over_ui`
- `gain_over_ui`
- `total_wallet_exposure_mean`

The local optimize row set was adjusted to better match the metrics currently used for result review and candidate selection in this deployment.

## Exact local patch behavior

### Backtest UI

If a backtest result payload contains:

- `ulcer_index_usd`
- `adg_over_ui_usd`
- `gain_over_ui_usd`

PBGUI shows them directly.

If only legacy unsuffixed names exist, PBGUI falls back to those.

### Optimize UI

PBGUI reads newer metrics across all supported payload shapes, not only the latest suite format.

This matters because `OptimizeV7.py` still supports:

- suite payloads
- non-suite `metrics.stats`
- old `analyses_combined`
- old single-analysis payloads

## What was intentionally not changed

Not changed:

- PB7 metric calculation
- Rust analysis logic
- PB7 optimizer objective math
- PBGUI does not synthesize ulcer metrics on its own

If PB7 payloads stop emitting these metrics, PBGUI should not invent replacements.

## Validation / expected runtime behavior

Expected backtest behavior:

- backtest result table can display:
  - `Ulcer Index`
  - `ADG/UI`
  - `Gain/UI`

Expected optimize behavior:

- optimize result table rows contain the new metrics across supported payload formats
- if the PB7 payload includes `_usd` metrics, PBGUI should prefer those

## Operational guidance

If a metric is missing in PBGUI:

1. first verify the metric exists in PB7 output JSON
2. then verify whether the payload shape is:
   - suite
   - non-suite stats
   - `analyses_combined`
   - legacy single-analysis
3. only after that adjust PBGUI reader logic

Useful checks:

```bash
rg -n "ulcer_index|adg_over_ui|gain_over_ui|omega_ratio|peak_recovery_hours" /app/pbgui/BacktestV7.py /app/pbgui/OptimizeV7.py
```

```bash
git -C /app/pbgui diff upstream/main -- BacktestV7.py OptimizeV7.py
```

## Long-term proper fix

The long-term proper state is:

- upstream PBGUI exposes the PB7 metrics needed by this deployment
- PBGUI reads canonical PB7 metric keys consistently
- legacy payload-shape support can eventually be simplified only after old artifacts are no longer needed
