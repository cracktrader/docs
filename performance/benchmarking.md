# Benchmarking Guide

How to measure and profile CrackTrader performance.

## Quick Start

Run the unified benchmark tool:

```bash
# Basic mock test (fast)
python performance/bench.py

# Sandbox test with real network
python performance/bench.py --sandbox

# Full profiling with memory analysis
python performance/bench.py --profile --verbose
```

## What gets measured

### Mock Mode (Fast)
- Store creation: ~4.6s (test fixture import overhead)
- Order processing: ~17ms (our code overhead)
- Data processing: ~5ms (1000 candles)
- Backtrader baseline: ~110ms
- Total: ~1.6s

### Sandbox Mode (Realistic)
- Network calls: 200-1000ms per request
- Real exchange latency included
- Uses minimal testnet funds
- Average: ~775ms
- Performance score: 56/100

## Profiling Output

### Timing Breakdown
```
[STATS] Performance Summary:
   Total benchmark time: 1629.7ms
   Number of operations tested: 11

[SLOW] Top 5 Bottlenecks:
   1. store_creation: 1316.43ms avg (test fixture imports)
   2. pandas_operations: 260.05ms avg (benchmark simulation)
   3. pure_backtrader: 44.92ms avg (external library)
   4. cracktrader_overhead: 5.00ms avg (our code)
   5. backtrader_data_ingestion: 1.85ms avg
```

### Memory Analysis
```
[PROFILE] Memory profile saved to: performance/reports/profiles/memory_profile_mock_20250812.txt

Top memory usage:
- store_creation: 90.2MB (import overhead)
- pandas_operations: 23.9MB (data processing)
```

## Generated Reports

All benchmarks save detailed reports:

```
performance/reports/
├── benchmark_mock_20250812_210132.json     # Full metrics
├── memory_profile_mock_20250812.txt        # Memory breakdown
├── latest_benchmark_mock.json              # Latest results
└── optimization_recommendations.json       # Action items
```

## Understanding the Results

### What’s typically fast
- Order processing paths
- Data validation
- Cached data loading

### What's Actually Slow
- **Network calls**: 200-1000ms (can't optimize - external exchange)
- **Pandas operations**: 700ms for complex calculations on large datasets
- **System startup**: 4.6s (test infrastructure only, not production)

### False bottlenecks
- Store creation slowness from test fixture imports
- Standalone pandas benchmark artifacts (not core code)

## Optimization Recommendations

The tool automatically identifies optimization candidates:

```
[RUST] Top Rust/Go Optimization Candidates:
   • Memory: store_creation using 90.2MB memory
     Solution: Consider memory optimization or object pooling
   • Data Processing: pandas_operations taking 707ms
     Solution: Consider vectorized operations or Rust implementation
```

## Custom Profiling

### Profile Your Own Code
```python
from performance.bench import PerformanceBenchmark

benchmark = PerformanceBenchmark(profile=True)
with benchmark.time_section("my_strategy"):
    # Your strategy code here
    results = cerebro.run()

benchmark._analyze_results()
```

### Memory Profiling
```python
import tracemalloc

tracemalloc.start()
# Your code here
snapshot = tracemalloc.take_snapshot()
top_stats = snapshot.statistics('lineno')

for stat in top_stats[:10]:
    print(stat)
```

## Interpreting Benchmarks

- Use mock mode for code baselines; sandbox includes network latency
- Compare medians across multiple runs; watch variance
- Investigate outliers before optimizing

## Next Steps

1. **Enable caching**: `cache_enabled=True` for 90% improvement
2. **Profile strategies**: Use `--profile` flag for detailed analysis
3. **Check memory**: Monitor for memory leaks in long-running backtests
4. **Network optimization**: Consider connection pooling for live trading

See: [Large Datasets](large_datasets.md) | [Optimization Roadmap](optimization_roadmap.md)
