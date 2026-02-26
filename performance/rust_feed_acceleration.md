# Rust Feed Acceleration (Roadmap Baseline)

This document tracks the feed-side Rust acceleration boundary and baseline benchmarking workflow.

## Current boundary

Feed acceleration currently targets two hot paths:

- Tick bucketing / candle aggregation from trades
- Reorder-buffer sorting for out-of-order ticks

Python and Rust share the same bridge contract in `cracktrader.feeds.rust_bridge`:

- `aggregate_trade_bucket(...) -> {emit, next_bucket_start_ms, next_bucket}`
- `sort_reorder_buffer(candles) -> {sorted, out_of_order_count}`

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

Outputs:

- JSON report at `performance/feed_accelerator_benchmark.json`
- backend elapsed timing and optional speedup ratio (`speedup_python_over_rust`) when Rust is available

## Parity expectations

Rust and Python backends must match for:

- bucket open/high/low/close/volume updates
- bucket rollover emission boundaries
- reorder sort order for out-of-order timestamps

Parity is guarded by feed unit tests and bridge smoke tests.
