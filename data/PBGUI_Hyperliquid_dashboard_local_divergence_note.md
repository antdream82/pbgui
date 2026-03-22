# PBGUI Hyperliquid Dashboard Local Divergence From Upstream

## Summary

PBGUI currently keeps a small local divergence from upstream for Hyperliquid dashboard/data ingestion.

This divergence exists to make the `Information -> Dashboards` balance/position/price/order panels work correctly for the current Hyperliquid deployment, where:

- balance and positions live in the `dex='xyz'` scope rather than the default scope
- dashboard total balance is expected to reflect portfolio-level USDC cash (`spotClearinghouseState`), not builder-scoped dex account value
- Hyperliquid HIP-3 symbols must preserve CCXT symbols like `XYZ-XYZ100/USDC:USDC` instead of reconstructing `XYZXYZ100/USDC:USDC`

## Root Cause

Three issues were present in PBGUI's Hyperliquid ingestion path:

1. `Exchange.fetch_balance()` and `Exchange.fetch_positions()` only trusted the default Hyperliquid scope.
   In the current deployment, the live `XYZ-XYZ100` position and its account value are in `dex='xyz'`.

2. PBData treated Hyperliquid balance/positions as websocket-fed state, which caused `hyper` to be classified as `ws` in `fetch_summary.json` even though balance/position updates were effectively coming from REST/shared-poll paths.

3. PBGUI rebuilt Hyperliquid HIP-3 symbols from compact internal symbols (`XYZXYZ100USDC -> XYZXYZ100/USDC:USDC`), which is not a valid CCXT Hyperliquid symbol.
   The valid CCXT symbol is `XYZ-XYZ100/USDC:USDC`.

## Current Local Divergence From Upstream

Currently kept local changes:

1. `Exchange.fetch_positions()` retries Hyperliquid positions with `params={"dex": "xyz"}` when the default scope returns no positions.
2. `Exchange.fetch_balance()` for Hyperliquid uses `fetch_balance(type="spot") -> total["USDC"]` as the dashboard total balance source, with dex/default fallbacks only if the spot fetch fails.
3. PBData does not start Hyperliquid balance/position websocket watchers; Hyperliquid balance/position updates are intentionally treated as REST/shared-poll data.
4. PBGUI Hyperliquid price/order symbol mapping uses recent execution CCXT symbols to preserve valid HIP-3 symbols like `XYZ-XYZ100/USDC:USDC`.

## Exact Local Patch Behavior

### 1. Hyperliquid balance source

File:

- [Exchange.py](/app/pbgui/Exchange.py)

Function:

- `Exchange.fetch_balance()`

Current behavior for `self.id == "hyperliquid"`:

1. Try `self.instance.fetch_balance(params={"type": "spot"})`
2. Return `float(spot_balance["total"]["USDC"])`
3. If that fails, fall back to:
   - `fetch_balance(params={"type": market_type, "dex": "xyz"})`
   - then default `fetch_balance(params={"type": market_type})`

This makes dashboard `Total Balance` reflect portfolio-level USDC cash instead of dex-only account value.

### 2. Hyperliquid positions source

File:

- [Exchange.py](/app/pbgui/Exchange.py)

Function:

- `Exchange.fetch_positions()`

Current behavior for `self.id == "hyperliquid"`:

1. Try default `self.instance.fetch_positions()`
2. If empty, retry `self.instance.fetch_positions(params={"dex": "xyz"})`

This allows PBGUI to read the live `XYZ-XYZ100` position currently held in the dex scope.

### 3. Hyperliquid balance/position polling mode

File:

- [PBData.py](/app/pbgui/PBData.py)

Functions:

- `_ensure_balance_watcher()`
- `_ensure_position_watcher()`

Current behavior:

- for `user.exchange == "hyperliquid"`, PBData does not create balance/position websocket watcher tasks
- therefore `hyper` is classified as `rest` for balances/positions in `fetch_summary.json`
- shared combined poll is responsible for calling:
  - `db.update_balances(user)`
  - `db.update_positions(user)`

### 4. Hyperliquid HIP-3 price/order symbol preservation

Files:

- [Database.py](/app/pbgui/Database.py)
- [PBData.py](/app/pbgui/PBData.py)

Current behavior:

- recent execution CCXT symbols are used as the source-of-truth fallback symbol set
- for Hyperliquid, internal compact symbols from positions are matched back to recent CCXT symbols via:
  - `Database._match_recent_ccxt_symbols()`
  - `PBData._match_recent_ccxt_symbols()`

This prevents invalid reconstructions such as:

- `XYZXYZ100/USDC:USDC`

and preserves the valid CCXT symbol:

- `XYZ-XYZ100/USDC:USDC`

Affected paths:

- `Database.update_orders()`
- `Database.update_prices()`
- `PBData._build_price_mapping_for_exchange()`

## Why The Divergence Is Kept

Without these local changes, PBGUI can show one or more of the following wrong states for Hyperliquid:

- balance shown as `0.0`
- balance shown as dex-only account value (`~19000`) when the dashboard should show portfolio-level USDC total (`~25500`)
- positions missing from the dashboard
- prices missing from the dashboard
- orders missing from the dashboard
- repeated `BadSymbol` errors from invalid `XYZXYZ100/USDC:USDC` reconstruction

The current local changes are intentionally narrow and aimed at PBGUI ingestion only.

## What Was Intentionally Not Changed

Not changed:

- PB7 live trading logic
- PB7 Rust strategy/risk math
- PB7 Hyperliquid balance denominator logic
- Hyperliquid order-generation logic
- dashboard layout/config itself

This note covers PBGUI data ingestion and display semantics only.

## Validation / Expected Runtime Behavior

Expected current behavior:

- `fetch_summary.json` should show:
  - `hyper` in `balances.rest`
  - `hyper` in `positions.rest`
- Database should contain:
  - a `hyper` balance row near the spot USDC total (`~25500`)
  - a nonzero `hyper` position row for `XYZXYZ100USDC`
  - `hyper` prices
  - `hyper` orders

Expected log examples:

- `update_balances fetched balance for user=hyper: 255xx.xxxxx`
- `update_positions fetched 1 row(s) for user=hyper`
- no continuing `watch_tickers subscribe ERROR ... XYZXYZ100/USDC:USDC` after the latest restart

## Operational Guidance

If Hyperliquid dashboard data looks wrong again:

1. Check `/app/pbgui/data/logs/fetch_summary.json`
   - `hyper` should be `rest` for balances/positions
2. Check latest balance/position rows in `/app/pbgui/data/pbgui.db`
3. Check logs for:
   - `update_balances fetched balance for user=hyper`
   - `update_positions fetched ...`
   - `watch_tickers subscribe ERROR for exchange hyperliquid`
4. If `BadSymbol` reappears, verify that the symbol source still resolves to `XYZ-XYZ100/USDC:USDC`, not `XYZXYZ100/USDC:USDC`

Useful checks:

```bash
rg -n "update_balances fetched balance for user=hyper|update_positions fetched .*hyper|watch_tickers subscribe ERROR for exchange hyperliquid" /app/pbgui/data/logs/Database.log /app/pbgui/data/logs/PBData.log | tail -n 80
```

```bash
python - <<'PY'
import sqlite3
conn=sqlite3.connect('/app/pbgui/data/pbgui.db')
cur=conn.cursor()
print('balances', list(cur.execute("select * from balances where user='hyper' order by id desc limit 5")))
print('position', list(cur.execute("select * from position where user='hyper' order by id desc limit 5")))
print('prices', list(cur.execute("select * from prices where user='hyper' order by id desc limit 5")))
print('orders', list(cur.execute("select * from orders where user='hyper' order by id desc limit 10")))
conn.close()
PY
```

## Long-Term Proper Fix

The long-term proper solution would be to make Hyperliquid support in PBGUI explicit rather than deployment-specific:

1. Make Hyperliquid scope handling first-class in PBGUI, instead of using deployment-specific `dex='xyz'` fallback behavior.
2. Store and propagate canonical Hyperliquid CCXT symbols directly, rather than compacting and reconstructing them.
3. Upstream any PBGUI Hyperliquid fixes that are generally valid.

Until then, the current local divergence is kept for operational correctness in this deployment.
