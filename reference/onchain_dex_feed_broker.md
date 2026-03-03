# Onchain DEX Feed/Broker Parity (Uniswap + Pancake v2)

This release adds DEX-native feed/broker parity aligned with the CCXT/Polymarket architecture:

- Default `ct.Feed(exchange="uniswap"|"pancakeswap", ...)` now returns an OHLCV stream feed.
- Quote snapshot compatibility remains available via `kind="quote"`.
- DEX brokers are mode-routed with universal-broker-compatible classes:
  - `OnchainSimulationBroker` for `paper`/`backtest`
  - `OnchainLiveBroker` for `sandbox`/`live`
- `submit_swap(...)` remains supported on all onchain broker classes.

## Reliability Model

`OnchainStore` now supports reorg-aware, callback-driven ingestion:

- `on_new_head(number, block_hash, parent_hash)` drives block-based processing.
- Backfill uses block ranges (`last_processed_block + 1 .. head`).
- Reorgs invalidate orphan block state and replay the canonical range.
- Log dedupe is deterministic (`block_hash + tx_hash + log_index + event`).

## Pool State and Callbacks

New DEX-specific store APIs:

- `register_pool_state_callback(symbol, callback)`
- `get_latest_pool_state(symbol)`
- `register_raw_swap_callback(symbol, callback)` (alias: `register_swap_callback`)

OHLCV candles are aggregated from `Swap` events only.
Liquidity/reserve events (`Sync`, `Mint`, `Burn`) update pool state, not candle volume.

## Example

See:

- `examples/data_access/onchain_dex_ohlcv_and_pool_state.py`

It demonstrates:

- deterministic `newHeads`/`eth_getLogs`-style ingestion with a scripted provider
- OHLCV feed consumption
- pool-state callback consumption
