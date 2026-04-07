# PB7 Upstream Rebase Plan

## Purpose

This file is the working execution plan for rebuilding local PB7 behavior on top of `upstream/master`.

Use this together with:

- [PBGUI_LOCAL_DIVERGENCE_NOTE.md](/app/pbgui/data/PBGUI_LOCAL_DIVERGENCE_NOTE.md)

This file is not the divergence reference itself. It is the step-by-step work plan to follow while executing the migration.

## Current decision

`pb7` has diverged far enough from upstream that selective backports are no longer the preferred strategy.

Target strategy:

1. Start from `upstream/master`
2. Rebuild the full required local PB7 behavior on top of it
3. Keep upstream semantics where they are better defined
4. Reapply local-only behavior that remains required in production

## Target end state

The final rebased PB7 should satisfy all of the following:

1. Uses `upstream/master` as the base
2. Preserves required local Hyperliquid live behavior
3. Adopts upstream scoring semantics based on `metric + goal`
4. Adopts upstream new optimizer-facing metrics
5. Preserves local actual exposure metrics
6. Preserves local scenario-aware optimize limits
7. Preserves zero-fill backtest resample safety
8. Passes the local regression subset required for PB7 and PBGUI integration

## Non-goals

Do not try to preserve the old local PB7 file layout if upstream refactored it.

Do not force old local implementations into upstream files when upstream now has a cleaner contract for the same behavior.

Do not touch PBGUI until the PB7 behavior is stable enough to expose.

## Source of truth for required local behavior

Read these sections in the divergence note before each phase:

1. [PB7 actual exposure metrics and scenario-aware limits](/app/pbgui/data/PBGUI_LOCAL_DIVERGENCE_NOTE.md#L50)
2. [Upstream full-pull reapply guide](/app/pbgui/data/PBGUI_LOCAL_DIVERGENCE_NOTE.md#L428)
3. [Reapply package 1: PB7 actual exposure metrics and scenario-aware limits](/app/pbgui/data/PBGUI_LOCAL_DIVERGENCE_NOTE.md#L496)
4. [Rebase Preparation Design](/app/pbgui/data/PBGUI_LOCAL_DIVERGENCE_NOTE.md#L778)

## Worktree strategy

Recommended approach:

1. Keep current local `master` unchanged as the rollback reference
2. Create a new branch from `upstream/master`
3. Reapply behavior in small commits
4. Verify after every phase

Suggested branch name:

- `rebase/pb7-upstream-20260407`

Suggested bootstrap commands:

```bash
git -C /app/pb7 fetch upstream
git -C /app/pb7 checkout -b rebase/pb7-upstream-20260407 upstream/master
```

## Required behavior inventory

Nothing in the current local PB7 patch set should be considered optional for production.

The local PB7 behavior falls into four execution packages.

### Package 1: `pb7-runtime-hyperliquid`

Purpose:

- restore production-safe Hyperliquid live behavior

Primary files:

- [hyperliquid.py](/app/pb7/src/exchanges/hyperliquid.py)
- [passivbot.py](/app/pb7/src/passivbot.py)
- [utils.py](/app/pb7/src/utils.py)

Required behavior:

1. HIP-3 symbol handling must not leak internal symbols into CCXT-facing paths
2. Reverse mapping and pseudo-symbol fallback must behave like the current local fork
3. DEX-scoped balance/position/equity aggregation must preserve current live semantics
4. Cancel-gone suppression and tombstone persistence must prevent stale order reintroduction loops
5. Sizing and orchestration must preserve the snapped-balance local behavior
6. Margin mode and metadata handling must preserve the current local Hyperliquid behavior
7. Spot namespace collision mitigation must remain intact

Verification:

- [test_hyperliquid_balance_cache.py](/app/pb7/tests/test_hyperliquid_balance_cache.py)
- [test_order_orchestration.py](/app/pb7/tests/test_order_orchestration.py)
- [test_passivbot_balance_split.py](/app/pb7/tests/test_passivbot_balance_split.py)
- [test_stock_perps.py](/app/pb7/tests/test_stock_perps.py)
- [test_utils_maps.py](/app/pb7/tests/test_utils_maps.py)

### Package 2: `pb7-upstream-scoring-and-metrics`

Purpose:

- adopt upstream scoring semantics and new upstream metrics before adding local optimize extensions

Primary files:

- [backtest.py](/app/pb7/src/backtest.py)
- [analysis.rs](/app/pb7/passivbot-rust/src/analysis.rs)
- [types.rs](/app/pb7/passivbot-rust/src/types.rs)
- [optimize.py](/app/pb7/src/optimize.py)
- [pareto_store.py](/app/pb7/src/pareto_store.py)
- [iterative_backtester.py](/app/pb7/src/tools/iterative_backtester.py)
- upstream config layer equivalents such as:
  - `src/config/scoring.py`
  - `src/config/metrics.py`

Required behavior:

1. Optimizer scoring is driven by explicit `metric + goal`
2. Engine-space objective handling follows upstream semantics
3. Upstream new metric families are available:
   - `*_pnl`
   - `paper_loss_*`
   - `exposure_*`
   - `win_rate*`
   - `trade_loss_*`
4. Canonical metric naming follows upstream rules

Verification:

- upstream config/scoring tests as available
- local optimize smoke path after scoring config normalization

### Package 3: `pb7-local-optimize-extensions`

Purpose:

- restore local optimize semantics that upstream still does not provide

Primary files:

- [backtest.py](/app/pb7/src/backtest.py)
- [config_utils.py](/app/pb7/src/config_utils.py)
- [limit_utils.py](/app/pb7/src/limit_utils.py)
- [optimize.py](/app/pb7/src/optimize.py)
- [pareto_store.py](/app/pb7/src/pareto_store.py)
- [iterative_backtester.py](/app/pb7/src/tools/iterative_backtester.py)

Required behavior:

1. Local actual exposure metrics are present:
   - `wallet_exposure_mean_long`
   - `wallet_exposure_median_long`
   - `wallet_exposure_max_long`
   - `wallet_exposure_mean_short`
   - `wallet_exposure_median_short`
   - `wallet_exposure_max_short`
2. Local actual-exposure-normalized metrics are present:
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
3. Short exposure is derived from `abs(twe_short)`
4. `optimize.limits[]` supports optional `scenario`
5. `scenario + stat` is rejected
6. Scenario-specific values resolve correctly both at optimize runtime and in pareto reevaluation
7. Zero-fill backtest runs do not fail during balance/equity resampling

Verification:

- [test_backtest_analysis.py](/app/pb7/tests/test_backtest_analysis.py)
- [test_optimizer_limits_integration.py](/app/pb7/tests/test_optimizer_limits_integration.py)
- [test_pareto_limits.py](/app/pb7/tests/test_pareto_limits.py)

### Package 4: `pb7-test-realignment`

Purpose:

- align local regression tests with upstream APIs without losing local intent

Primary files:

- [test_backtest_analysis.py](/app/pb7/tests/test_backtest_analysis.py)
- [test_optimizer_limits_integration.py](/app/pb7/tests/test_optimizer_limits_integration.py)
- [test_pareto_limits.py](/app/pb7/tests/test_pareto_limits.py)
- [test_hyperliquid_balance_cache.py](/app/pb7/tests/test_hyperliquid_balance_cache.py)
- [test_order_orchestration.py](/app/pb7/tests/test_order_orchestration.py)
- [test_passivbot_balance_split.py](/app/pb7/tests/test_passivbot_balance_split.py)
- [test_stock_perps.py](/app/pb7/tests/test_stock_perps.py)
- [test_utils_maps.py](/app/pb7/tests/test_utils_maps.py)

Required behavior:

1. Tests reflect upstream API shape where needed
2. Tests still encode the required local semantics
3. The minimal PB7 regression set stays runnable in `/venv_pb7`

## Execution order

Apply in this order:

1. Create rebased branch from `upstream/master`
2. Validate upstream baseline
3. Restore `pb7-runtime-hyperliquid`
4. Verify Hyperliquid package
5. Adopt `pb7-upstream-scoring-and-metrics`
6. Verify scoring and upstream metric package
7. Restore `pb7-local-optimize-extensions`
8. Verify optimize and backtest package
9. Align tests
10. Run the final PB7 regression subset
11. Only then start PBGUI restoration work

## Baseline check before any local reapply

Before reapplying local behavior:

1. Confirm branch starts from `upstream/master`
2. Run a minimal import and compile check
3. Run the upstream test subset that is relevant and available
4. Note any upstream failures before adding local changes

Suggested commands:

```bash
cd /app/pb7
PYTHONPATH=/app/pb7/src /venv_pb7/bin/python -m py_compile src/backtest.py src/optimize.py
```

## Verification gates

### Gate 1: Hyperliquid runtime

Must be true before moving on:

1. Hyperliquid symbol mapping behavior matches the current local fork
2. Cancel-gone suppression works
3. Balance/equity behavior matches local production expectations

### Gate 2: Upstream scoring and metrics

Must be true before moving on:

1. `metric + goal` scoring config works
2. Canonical metric naming is stable
3. New upstream metrics are present in outputs and available to optimization

### Gate 3: Local optimize extensions

Must be true before moving on:

1. Actual exposure metrics are present
2. `wallet_exposure_mean_short` works with negative raw `twe_short`
3. `scenario` limits resolve correctly
4. `scenario + stat` is rejected
5. Zero-fill backtest resample path is safe

### Gate 4: Final regression

Run at minimum:

```bash
cd /app/pb7
PYTHONPATH=/app/pb7/src /venv_pb7/bin/python -m pytest \
  tests/test_backtest_analysis.py \
  tests/test_optimizer_limits_integration.py \
  tests/test_pareto_limits.py \
  tests/test_hyperliquid_balance_cache.py \
  tests/test_order_orchestration.py \
  tests/test_passivbot_balance_split.py \
  tests/test_stock_perps.py \
  tests/test_utils_maps.py -q
```

## High-conflict files

Expect careful manual integration here:

- [config_utils.py](/app/pb7/src/config_utils.py)
- [optimize.py](/app/pb7/src/optimize.py)
- [pareto_store.py](/app/pb7/src/pareto_store.py)
- [backtest.py](/app/pb7/src/backtest.py)
- [passivbot.py](/app/pb7/src/passivbot.py)
- [hyperliquid.py](/app/pb7/src/exchanges/hyperliquid.py)

Rule:

Do not copy old local files wholesale over upstream versions.

Instead:

1. keep upstream structure
2. port local behavior intentionally
3. confirm with tests immediately after each package

## Commit plan

Recommended commit breakdown:

1. `pb7: restore hyperliquid live behavior on upstream base`
2. `pb7: adopt upstream scoring goals and new optimizer metrics`
3. `pb7: restore local actual exposure metrics and scenario limits`
4. `pb7: realign regression tests with rebased APIs`

## Stop conditions

Pause and reassess if any of these happen:

1. Upstream already implements a local behavior under a different contract
2. A local behavior conflicts directly with upstream metric semantics
3. Hyperliquid runtime behavior cannot be preserved without rethinking the package split
4. Test breakage expands beyond the expected PB7 subset

## Completion criteria

This plan is complete only when:

1. PB7 upstream base is in place
2. All required local PB7 behaviors are restored
3. Regression tests pass
4. PBGUI can begin its own reapply work against the rebased PB7
