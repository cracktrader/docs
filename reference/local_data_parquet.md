# Local Data Parquet Pipeline

Cracktrader local historical storage uses parquet files with zstd compression.

## Scope

- Categories:
  - `trades`
  - `ohlcv` (native and derived timeframes)
- Partitioning:
  - monthly partitions under stream paths (`YYYY/YYYY-MM.parquet`)
- Compression:
  - parquet column compression: `zstd`

## Schema Strategy

- `schema_version` is persisted in:
  - stream `manifest.json`
  - stream `_meta.json`
- Current version: `1`
- Evolution policy:
  - additive schema changes are preferred
  - breaking changes require explicit migration code and version bump

## Legacy Compatibility

Legacy pickle OHLCV cache can be migrated into the parquet store:

```bash
cracktrader data migrate-cache --source ./data --exchange binance --instrument spot
```

Dry-run mode:

```bash
cracktrader data migrate-cache --source ./data --dry-run
```

The migration expects legacy layout:

`{source}/{exchange}/{instrument}/{symbol_sanitized}/{timeframe}/YYYY-MM.pkl`

## Benchmark Snapshot

Local sample (2026-02-26, 100k 1m candles, CSV baseline):

- CSV read: `61.55 ms`
- Parquet/zstd read: `35.94 ms`
- Read speedup: `1.71x`
- CSV size: `4,200,038 bytes`
- Parquet size: `548,370 bytes`
- Size reduction: `7.66x`
