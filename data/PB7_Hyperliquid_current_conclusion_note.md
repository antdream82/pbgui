# PB7 Hyperliquid Current Conclusion Note

## 1. Summary

PB7 current Hyperliquid live deployment keeps a small local divergence from upstream for safety.

This note covers the currently relevant Hyperliquid live divergences from upstream.

This note also covers the kept local divergence for live balance denominator handling (`balance_raw` / snapped balance / equity input semantics) on Hyperliquid.

The active divergence is:

- skipping raw spot-style `@...` open orders during futures open-order ingestion
- normalizing cancel-gone local order removal by `str(id)`
- keeping HIP-3 margin-mode detection metadata-based
- using dex-scoped HIP-3 position/open-order visibility instead of relying only on default-scope state
- using Hyperliquid default quote total as the live balance denominator source instead of summing builder `accountValue`

These are exchange-state and live-input plumbing changes. They do not change Rust strategy math.

## 2. Root cause

Raw Hyperliquid open-order payloads can include spot-style internal symbols such as `@107`.

In current PB7 live mode, the bot loads CCXT `swap + hip3` markets only. In that mode, CCXT market metadata can expose a futures namespace entry that collides with the raw spot-style `@...` identifier space.

Because PB7 is running as a futures bot and does not manage spot order state, raw `@...` orders must not be injected into the futures open-order state.

Affected path:

- [hyperliquid.py](/app/pb7/src/exchanges/hyperliquid.py)
- `HyperliquidBot.fetch_open_orders()`

## 3. Current local divergence from upstream

Currently kept local changes:

1. In Hyperliquid open-order ingestion, raw spot-style `@...` orders are skipped before futures normalization/state injection.
2. Local open-order removal on cancel-gone normalizes ids via `str(id)`.
3. HIP-3 margin-mode detection uses market metadata (`onlyIsolated`, `isolatedOnly`, `marginMode == "noCross"`) instead of prefix-only forcing.
4. HIP-3 positions/open orders are fetched through dex-scoped CCXT paths so builder-scoped state is visible to the bot.
5. Live balance denominator uses Hyperliquid default `total[quote]` as the portfolio source and does not add dex-scoped `accountValue` on top.

Temporary tombstone persistence/suppress changes from the `335406020602` investigation were reverted and are not part of the current state.

## 4. Exact local patch behavior

### Spot-style raw order skip

File:

- [hyperliquid.py](/app/pb7/src/exchanges/hyperliquid.py)

Function:

- `HyperliquidBot.fetch_open_orders()`

Behavior:

- `raw_symbol = order["symbol"]`
- if `raw_symbol` is a string and `raw_symbol.startswith("@")`:
  - log `[diag][skip_spot_open_order]`
  - skip the order
- otherwise continue through the existing futures normalization path

Effect:

- raw `@...` spot orders do not enter:
  - local `open_orders`
  - futures reconciliation
  - cancel-gone / stale-order handling

### Cancel-gone local removal normalization

File:

- [hyperliquid.py](/app/pb7/src/exchanges/hyperliquid.py)

Function:

- `HyperliquidBot.execute_cancellation()`

Behavior:

- local order removal compares ids using `str(id)`

Effect:

- local cache cleanup does not fail due to int/string id mismatch

### HIP-3 margin-mode detection

File:

- [hyperliquid.py](/app/pb7/src/exchanges/hyperliquid.py)

Function:

- `HyperliquidBot._requires_isolated_margin()`

Behavior:

- uses market metadata flags
- does not force isolated mode from `xyz:` / `XYZ-` / `XYZ:` prefix alone

### HIP-3 state visibility

File:

- [hyperliquid.py](/app/pb7/src/exchanges/hyperliquid.py)

Functions:

- `HyperliquidBot._get_hl_hip3_state_dexes()`
- `HyperliquidBot._fetch_hip3_positions()`
- `HyperliquidBot.fetch_open_orders()`
- `HyperliquidBot._normalize_ccxt_position()`

Behavior:

- HIP-3 state is queried via dex-scoped CCXT fetches
- internal ids returned by those fetches are normalized through the exchange reverse map before entering bot state

Effect:

- builder-scoped positions/open orders remain visible to the futures bot
- the bot does not behave as if resting HIP-3 orders or positions are missing

### Live balance / balance_raw / snapped-equity denominator source

File:

- [hyperliquid.py](/app/pb7/src/exchanges/hyperliquid.py)

Function:

- `HyperliquidBot._fetch_positions_and_balance()`

Behavior:

- uses default-scope `info["total"][self.quote]` as the live balance/equity denominator source
- does not sum dex-scoped builder `accountValue` on top
- pulls positions from `fetch_positions()` plus dex-scoped HIP-3 position fetches instead of reconstructing from default-scope `assetPositions`

Effect:

- avoids double-counting builder collateral / reserved margin in live denominator
- reduces false WEL/TWEL pressure caused by adding dex `accountValue` to the default quote total

## 5. Why the divergence is kept

This divergence is kept because current PB7 live deployment is a futures bot and should not interpret raw spot `@...` orders as futures orders.

Without this guard:

- raw spot orders can be mis-normalized into the futures namespace
- local futures open-order state can be contaminated
- diagnostics and stale-order handling can become misleading

The divergences are intentionally narrow:

- they block known-invalid state ingestion
- they expose builder-scoped HIP-3 state that default-scope CCXT calls miss
- they avoid live denominator double-counting from default total plus builder `accountValue`
- they do not alter Rust strategy math
- they do not alter order behavior in backtest/Rust

## 6. What was intentionally not changed

Not changed:

- Rust strategy/order/risk logic
- general order generation logic
- non-`@...` futures order handling
- generic churn logic except where separately patched

This note does not describe reverted investigation-only diagnostics or tombstone persistence, because they are not part of the current state.

## 7. Validation / expected runtime behavior

Expected runtime behavior:

- raw `@...` orders may still appear in fresh exchange fetches
- they should be logged as `[diag][skip_spot_open_order]`
- they should not enter futures open-order state
- non-`@...` futures orders should continue through the normal path
- HIP-3 positions and resting HIP-3 futures orders should remain visible through dex-scoped state fetches
- live balance should track the default quote total and should not jump upward from builder `accountValue` being added on top

Expected log pattern:

- repeated fetch of raw `@...` order from exchange is possible
- PB7 should skip it before state ingestion

## 8. Operational guidance

If investigating Hyperliquid order-state issues:

1. First check whether the raw exchange order symbol starts with `@`
2. If yes, treat it as spot-style raw order input, not futures state
3. Confirm PB7 is logging `[diag][skip_spot_open_order]`
4. Do not treat such an order as a futures tombstone/stale-order issue unless a new code path proves otherwise
5. For HIP-3 exposure/risk anomalies, verify the bot is using default quote total as denominator and not reintroducing builder `accountValue` summation

Useful check:

```bash
rg -n "skip_spot_open_order|@107|335406020602" /app/pbgui/data/run_v7/hyper/passivbot.log | tail -n 200
```

Reconsider this patch if:

- PB7 begins intentionally managing Hyperliquid spot order state
- CCXT fixes `@index` resolution safely in the currently used market-loading mode
- PB7 adds a dedicated spotMeta-based resolver for raw `@...` symbols
- Hyperliquid/CCXT begins exposing a single reliable portfolio-equity source that safely includes builder state without double-counting

## 9. Long-term proper fix

The proper long-term fix is:

1. use raw Hyperliquid `spotMeta` as the source-of-truth for `@index` spot-pair resolution
2. adopt an upstream CCXT fix that safely resolves raw `@...` spot identifiers in mixed deployment modes
3. use a single exchange-truth portfolio denominator source for Hyperliquid live balance inputs instead of mixing default total and builder `accountValue`

If PB7 ever needs to manage spot order state intentionally, a dedicated spotMeta-based `@index` resolver should replace the current skip-based mitigation.
