# PB7 Local Divergence Note

## Summary

PB7 keeps a small Hyperliquid-specific local divergence from upstream for live safety and state correctness.

These changes are exchange-ingestion and live-input plumbing only. They do not change Rust strategy math.

## Root cause

Two Hyperliquid-specific issues required local handling:

1. Raw Hyperliquid open orders can include spot-style symbols like `@107`, while PB7 live mode loads `swap + hip3` markets only. In that mode, CCXT namespace resolution can mis-normalize raw `@...` spot orders into futures state.
2. Hyperliquid HIP-3 live state is split across default scope and dex scope. For PB7 live behavior, default quote total is the stable denominator source, while dex-scoped state is needed for positions/open orders visibility.

## Current local divergence from upstream

1. In [src/exchanges/hyperliquid.py](/app/pb7/src/exchanges/hyperliquid.py), `fetch_open_orders()` skips raw spot-style `@...` orders before futures normalization/state injection.
2. In [src/exchanges/hyperliquid.py](/app/pb7/src/exchanges/hyperliquid.py), local cancel-gone order removal normalizes ids with `str(id)`.
3. In [src/exchanges/hyperliquid.py](/app/pb7/src/exchanges/hyperliquid.py), HIP-3 margin-mode detection uses market metadata (`onlyIsolated`, `isolatedOnly`, `marginMode == "noCross"`) instead of prefix-only forcing.
4. In [src/exchanges/hyperliquid.py](/app/pb7/src/exchanges/hyperliquid.py), HIP-3 positions/open orders are fetched via dex-scoped CCXT paths so builder-scoped state remains visible.
5. In [src/exchanges/hyperliquid.py](/app/pb7/src/exchanges/hyperliquid.py), live balance denominator uses default `total[quote]` and does not add dex `accountValue` on top.
6. In [src/passivbot.py](/app/pb7/src/passivbot.py), `match_check` diagnostics log only final `result=pair` / `result=unmatched` outcomes, not every intermediate candidate comparison.

## Exact local patch behavior

### Spot-style raw order skip

File:
- [src/exchanges/hyperliquid.py](/app/pb7/src/exchanges/hyperliquid.py)

Function:
- `HyperliquidBot.fetch_open_orders()`

Behavior:
- If `order["symbol"]` is a string starting with `@`, log `[diag][skip_spot_open_order]` and skip the order.
- Non-`@...` orders continue through the normal futures normalization path.

### Cancel-gone id normalization

File:
- [src/exchanges/hyperliquid.py](/app/pb7/src/exchanges/hyperliquid.py)

Function:
- `HyperliquidBot.execute_cancellation()`

Behavior:
- Local open-order removal compares ids using `str(id)` so int/string mismatches do not leave stale local cache entries.

### HIP-3 margin mode and state visibility

File:
- [src/exchanges/hyperliquid.py](/app/pb7/src/exchanges/hyperliquid.py)

Functions:
- `_requires_isolated_margin()`
- `_get_hl_hip3_state_dexes()`
- `_fetch_hip3_positions()`
- `fetch_open_orders()`

Behavior:
- Margin mode is inferred from market metadata, not symbol prefix alone.
- HIP-3 positions/open orders are read from dex-scoped fetches and normalized before entering bot state.

### Live denominator source

File:
- [src/exchanges/hyperliquid.py](/app/pb7/src/exchanges/hyperliquid.py)

Function:
- `_fetch_positions_and_balance()`

Behavior:
- Live denominator source is default `total[quote]`.
- Dex `accountValue` is not summed on top.

### `match_check` diagnostic refinement

File:
- [src/passivbot.py](/app/pb7/src/passivbot.py)

Function:
- `_apply_order_match_tolerance()`

Behavior:
- Diagnostics show final pair/unmatched results only.
- Intermediate old-order candidate comparisons are not logged.

## Why the divergence is kept

Without these local changes, PB7 can:

- ingest spot orders into futures state
- mis-handle Hyperliquid HIP-3 state visibility
- use a distorted live denominator
- produce noisy matching diagnostics that obscure the real remaining edge-order behavior

## What was intentionally not changed

Not changed:

- Rust entry/close/risk/unstuck logic
- strategy math
- order generation formulas
- tolerance behavior beyond diagnostic logging refinement

Temporary tombstone persistence/suppress experiments from the earlier `335406020602` investigation were removed and are not part of the current state.

## Validation / expected runtime behavior

Expected live behavior:

- Raw `@...` spot orders may still appear in raw exchange responses, but PB7 logs `[diag][skip_spot_open_order]` and they do not enter futures `open_orders`.
- Hyperliquid balance stays near the expected default quote total instead of jumping from dex `accountValue` aggregation.
- `match_check` logs appear as `result=pair` or `result=unmatched`, without candidate spam.

## Operational guidance

When updating upstream:

1. Re-check [src/exchanges/hyperliquid.py](/app/pb7/src/exchanges/hyperliquid.py) first.
2. Preserve the raw `@...` skip unless upstream has a proper `spotMeta`-aware resolver.
3. Preserve the default `total[quote]` denominator behavior unless upstream has a verified Hyperliquid denominator fix.
4. Keep the `match_check` diagnostic refinement unless upstream logging is already equivalent.

## Long-term proper fix

The proper long-term fix is upstream/structural:

- Hyperliquid `@index` resolution should use raw `spotMeta` as source-of-truth.
- CCXT/PB7 mixed-mode symbol resolution should distinguish spot `@...` namespace from futures ids cleanly.
- If upstream adopts a correct Hyperliquid portfolio/denominator contract, local denominator handling should be re-evaluated.
