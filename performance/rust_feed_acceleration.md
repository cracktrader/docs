# Rust Feed Acceleration (Roadmap Baseline)

This document tracks the feed-side Rust acceleration boundary and baseline benchmarking workflow.

## Current boundary

Feed acceleration currently targets two hot paths:

- Tick bucketing / candle aggregation from trades
- Reorder-buffer sorting for out-of-order ticks
- Feed payload parsing / normalization (trade + OHLCV)

Python and Rust share the same bridge contract in `cracktrader.feeds.rust_bridge`:

- `aggregate_trade_bucket(...) -> {emit, next_bucket_start_ms, next_bucket}`
- `sort_reorder_buffer(candles) -> {sorted, out_of_order_count}`
- `normalize_trade_payload(payload) -> {timestamp_ms, price, amount} | None`
- `normalize_ohlcv_payload(payload) -> [ts, open, high, low, close, volume] | None`

`NativeCCXTFeedDriver` accepts `feed_backend="python" | "rust"` and keeps Python fallback if Rust is unavailable.

## Fallback policy

- Default backend is `python`.
- If `feed_backend="rust"` is requested but extension is not installed, the driver logs a warning and falls back to Python.
- This keeps runtime behavior stable while enabling opt-in acceleration.

## Profiling / benchmark workflow

Run:

```bash
python scripts/benchmark_feed_accelerators.py --ticks 50000 --timeframe-ms 1000
```

Optional parser normalization workload control:

```bash
python scripts/benchmark_feed_accelerators.py --ticks 50000 --timeframe-ms 1000 --parser-samples 20000
```

Baseline comparisons are strict by default: `parser_samples_used` must match baseline/current.
Override only for exploratory checks:

```bash
python scripts/benchmark_feed_accelerators.py --allow-parser-samples-mismatch
```

Feed parity gate runner:

```bash
python scripts/run_feed_rust_parity_gate.py
```

By default, the gate requires the benchmark baseline file to exist when benchmark checks are enabled.
For exploratory local runs only, you can bypass this guard:

```bash
python scripts/run_feed_rust_parity_gate.py --allow-missing-baseline
```

The gate also requires benchmark output for both `python` and `rust` backends by default.
You can override required backends for ad-hoc local checks:

```bash
python scripts/run_feed_rust_parity_gate.py --require-backend python
```

Notes:

- Requires Rust toolchain (`rustup`/`rustc`) and `maturin` available.
- Gate includes benchmark baseline comparison against:
  - `performance/baselines/feed_accelerator_benchmark_baseline.json`
- CI required lane (`rust-feed-parity-required`) runs this gate with benchmark enabled.
- CI lane uploads feed benchmark artifacts and emits a benchmark table in job summary.
- Benchmark step requires both `python` and `rust` backend results in gate runs.
- Benchmark script validates summary shape (ticks/timeframe/backend metrics/hotpath fields) unless `--skip-summary-validate` is set.

Outputs:

- JSON report at `performance/feed_accelerator_benchmark.json`
- backend elapsed timing and optional speedup ratio (`speedup_python_over_rust`) when Rust is available

Published baseline artifact (current branch):

- `performance/reports/feed_accelerator_benchmark_latest.json`
- `performance/baselines/feed_accelerator_benchmark_baseline.json`
- Environment sample captured on February 26, 2026:
  - backend: `python`
  - ticks: `50000`
  - elapsed: `79.621 ms`
  - emitted candles: `4999`
  - parser samples: `5000`
  - hotpaths (ms):
    - aggregation: `61.341`
    - parser normalization: `7.124`
    - reorder sort: `0.399`

## Parity expectations

Rust and Python backends must match for:

- bucket open/high/low/close/volume updates
- bucket rollover emission boundaries
- reorder sort order for out-of-order timestamps

Parity is guarded by feed unit tests and bridge smoke tests.

When Rust extension is installed, cross-backend parity harness runs:

- `tests/unit/feed/test_rust_feed_parity.py`
