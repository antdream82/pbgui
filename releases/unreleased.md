# Unreleased

- Rebase PBGUI onto upstream v1.79 and restore the local PB7 metric surface for actual-exposure, ulcer/UI, and side wallet-exposure metrics.
- Add Optimize v7 pareto summary chips for `gain_over_ui`, `ulcer_index`, `wallet_exposure_mean_long`, and `wallet_exposure_mean_short`.
- Restore suite-scenario-specific Optimize v7 limits, decimal-safe limit editing, seed-inclusive optimize progress totals, and JSON copy actions.
- Add short-TTL caching to Optimize v7 config, queue, result, and pareto reads so large result folders do not make the FastAPI UI sluggish.
- Increase legacy optimize EMA span bounds to 20000 for both EMA span sliders.
- Fix dashboard position `Next DCA` and `Next TP` values for short positions by classifying open buy/sell orders with the correct side-aware logic and nearest-price selection in both snapshot and live API paths.
- Fix the dashboard orders chart entry-line profit/loss color for short positions so it no longer uses long-only price-vs-entry comparisons.
- Add regression tests covering long/short dashboard order classification so nearest DCA/TP price selection stays correct for both snapshot and live API helpers.
- Make the dashboard Orders widget hedge-aware without a DB migration by sending the selected position side through `/ws/candles` and `/dashboard/orders_data`, showing `Orders: unknown` for hedged DB snapshots, and replacing that placeholder once live exchange order metadata can identify the correct leg.
