# PBGUI Upstream Divergence

This is the reference for rebuilding PBGUI on top of `upstream/main`.
Do not reapply local patches just because they existed before. First verify
upstream behavior, then reapply only the smallest patch needed to preserve a
production workflow.

## Current Base

- Rebase branch: `rebase/pbgui-upstream-20260525`
- Upstream base: `upstream/main` at PBGUI `v1.79`
- Linked PB7 expectation: rebased PB7 with local metric, seed, Hyperliquid,
  side-filter, and limit-stat patches documented in PB7 divergence docs.

## Rebase Rules

1. Confirm the PB7 feature or UI workflow is still needed.
2. Check whether current upstream already implements it.
3. Keep UI exposure, config serialization, and runtime launching separate.
4. Prefer PB7 runtime metadata over hard-coded PBGUI copies where possible.
5. Update this file and `releases/unreleased.md` whenever a local divergence is
   reintroduced.

## Metric And Config Surface

PBGUI must expose PB7's metric/goal model and the local PB7 metrics used in
production. FastAPI optimize metadata should come from PB7 runtime modules via
`api/pb7_bridge.py`; legacy `Config.py` keeps a matching registry for older
callers.

Required local metric families:

- Actual exposure efficiency: `gain_per_actual_exposure`,
  `adg_per_actual_exposure`, `mdg_per_actual_exposure`, weighted variants, and
  long/short variants.
- Ulcer/UI metrics: `ulcer_index`, `adg_over_ui`, `gain_over_ui`.
- Realized side exposure metrics: `wallet_exposure_mean_long`,
  `wallet_exposure_median_long`, `wallet_exposure_max_long`,
  `wallet_exposure_mean_short`, `wallet_exposure_median_short`,
  `wallet_exposure_max_short`.
- Upstream ratio/trade metrics must remain visible: `paper_loss_ratio`,
  `paper_loss_mean_ratio`, `exposure_ratio`, `exposure_mean_ratio`, `win_rate`,
  and trade-loss metrics.

Semantic rule:

- `*_per_exposure_*` means configured wallet exposure limit.
- `*_per_actual_exposure*` means realized mean exposure.

The legacy EMA optimize bounds must allow `20000` for both `EMA_SPAN_0_MAX`
and `EMA_SPAN_1_MAX`, otherwise existing research configs can fail to load.

## Optimize Workflow

The FastAPI optimize UI must preserve these local workflow requirements:

- Pareto/result summary chips include `gain_over_ui`, `ulcer_index`,
  `wallet_exposure_mean_long`, and `wallet_exposure_mean_short`.
- Optimize limits support optional suite `scenario`; when a scenario is set,
  `stat` is omitted so PB7 evaluates that exact scenario value.
- Limit value inputs use text/decimal inputs and only normalize on commit/blur,
  preventing focus loss and broken decimal entry while typing.
- Optimize progress includes seed/starting-config evaluations in the displayed
  total so the progress bar does not hit 100% before seed evaluation is done.
- `/configs`, `/queue`, `/results`, result-config, and pareto-file reads may
  use short TTL caches, with cache invalidation on mutating actions, to keep
  the new FastAPI UI responsive on large result folders.
- JSON inspection modals keep a top-right copy button.

## Backtest Workflow

Backtest JSON inspection must also keep a copy button. `BT selected` and result
import flows should preserve the selected config's original `backtest.end_date`
and scenario data instead of silently rewriting to today's date or dropping
suite scenarios.

## Hyperliquid And Dashboard Behavior

Keep Hyperliquid-specific dashboard and runtime fallbacks when upstream does
not cover them:

- valid CCXT symbols such as `XYZ-XYZ100/USDC:USDC` must be preserved;
- empty Hyperliquid position/balance paths may need dex/default fallbacks;
- dashboard uPnL should prefer fresh DB prices over stale stored values.

## Compatibility Shims

During upstream rebases, verify old Streamlit import paths still boot. If an
upstream refactor removes symbols still imported by local pages, restore a
minimal compatibility layer rather than reverting unrelated upstream changes.

Known historical shims:

- legacy `from Config import Config` compatibility for older modules;
- FastAPI v7 navigation helpers in `pbgui_func.py` for Streamlit navigation.

## Validation Checklist

Run at least:

- `python -m py_compile Config.py api/optimize_v7.py api/backtest_v7.py api/pb7_bridge.py`
- Open FastAPI optimize and verify metric lists include `gain_over_ui`,
  `ulcer_index`, `wallet_exposure_mean_long`, `wallet_exposure_mean_short`, and
  `gain_per_actual_exposure`.
- Verify optimize limit rows can target a named suite scenario.
- Verify decimal limit editing does not lose focus while typing.
- Verify result/pareto JSON copy buttons appear.

## Intentional Non-Reapplications

- Do not restore old PB6 or old Streamlit-only flows when upstream FastAPI
  pages now own the workflow.
- Do not duplicate PB7 metric lists in FastAPI when PB7 runtime metadata can be
  queried directly.
- Do not keep local UI patches for disabled or unused PB7 features without a
  reproduced workflow need.
