# Performance Overview

Current performance characteristics and optimization status.

## Performance Philosophy

- Measure everything with `bench.py` tool
- Cache historical data to avoid repeated API calls
- Profile bottlenecks before optimizing
- Document what's actually slow vs what's fast

## Current Performance Baseline

### System Performance (recent benchmarks)

| Component | Mock Mode | Sandbox Mode | Notes |
|-----------|-----------|--------------|-------|
| **Store Creation** | 4.6s | 0.6s | One-time startup cost |
| **Order Processing** | 17ms | N/A | Our code overhead (very good!) |
| **Data Processing** | 5ms | N/A | 1000 candles, raw processing |
| **Network Operations** | N/A | 200-1000ms | External exchange latency |
| **Total Benchmark** | 1.6s | 775ms | Full system test |

Note: Sandbox mode includes real network latency; use mock mode to measure pure code paths.

### Large Dataset Performance

| Dataset Size | Load Time | Memory Usage | With Caching |
|--------------|-----------|--------------|--------------|
| **1K candles** | 0.01s | 5MB | 0.001s |
| **100K candles** | 2-5s | 50MB | 0.1s (90% reduction) |
| **1M candles** | 20-50s | 500MB | 0.5s (98% reduction) |
| **5M candles** | 60-300s | 2GB | 2s (99% reduction) |

Note: Historical data caching provides 90-99% performance improvement for repeated backtesting.

## Performance Features

### 1. Historical Data Caching
Already implemented but not enabled by default.

```python
# Enable automatic caching for large datasets
store = CCXTStore(
    exchange='binance',
    cache_enabled=True,           # 90%+ speedup for backtesting
    cache_dir="./trading_data"    # Configurable location
)
```

**Features**:
- Monthly file segmentation for efficient storage
- Automatic deduplication and incremental updates
- Cross-exchange and multi-timeframe support
- Thread-safe operations for concurrent strategies

### **2. Memory-Efficient Data Processing**

```python
# Stream large datasets instead of loading all at once
feed = CCXTDataFeed(
    store=store,
    symbol="BTC/USDT",
    ccxt_timeframe="1m",
    historical_limit=1000000
)
```

### **3. Performance Monitoring**

Track timings in your strategy using standard tools:

```python
import time

start = time.perf_counter()
results = cerebro.run()
elapsed = (time.perf_counter() - start) * 1000
print(f"Backtest runtime: {elapsed:.1f} ms")
```

## Bottleneck Analysis

### What's Fast
- Order processing paths
- Data validation and transformation
- WebSocket streaming
- Cached data access

### What's Slow
1. Network calls (exchange dependent)
2. Initial data loading without caching
3. Complex indicators over large datasets
4. Cold start imports in some environments

### Optimization Targets
1. **Historical data pipeline** (90% solved with caching)
2. **Technical indicators** for large datasets (Rust candidate)
3. **Network layer** optimization (connection pooling)
4. **Memory usage** for multi-million candle backtests

## Performance Tools

### **Benchmarking Tool**

Run comprehensive performance tests:

```bash
# Fast mock tests (in-memory)
python performance/bench.py --verbose

# Realistic sandbox tests (with network)
python performance/bench.py --sandbox --verbose

# Deep profiling with memory analysis
python performance/bench.py --profile --verbose
```

Output includes component timings, memory usage, and bottleneck hints.

### Performance Reports

All benchmarks generate detailed reports:

```
performance/reports/
├── benchmark_mock_20250812.json      # Detailed metrics
├── memory_profile_mock_20250812.txt  # Memory analysis
├── latest_benchmark_mock.json        # Quick access
└── optimization_recommendations.json # Action items
```

## Optimization Roadmap

### **Phase 1: Enable Existing Optimizations** (Immediate)
- Enable caching by default for large datasets
- Document performance best practices
- Add configuration examples for different scales

### **Phase 2: Data Pipeline Enhancement** (Next)
- Parquet format support (5x smaller files, 10x faster loading)
- Streaming data processing for memory efficiency
- Vectorized technical indicators

### **Phase 3: Rust Integration** (Future)
- Technical indicator computation (100x speedup potential)
- Data I/O optimization
- Network layer improvements

## Performance Best Practices

### For Backtesting
```python
# ✅ Enable caching for repeated tests
store = CCXTStore(exchange_config, cache_enabled=True)

# ✅ Use appropriate historical limits
feed = CCXTDataFeed(historical_limit=50000)  # Not unlimited

# ✅ Profile your strategies
python performance/bench.py --profile
```

### For Live Trading
```python
# ✅ Monitor system health
cerebro.run(enable_health_monitoring=True)

# ✅ Use connection pooling
store = CCXTStore(exchange_config, connection_pool_size=10)

# ✅ Set appropriate log levels
import logging
logging.getLogger('cracktrader').setLevel(logging.WARNING)
```

### For Large Datasets
```python
# ✅ Stream data instead of loading all at once
feed = CCXTDataFeed(streaming=True)

# ✅ Use monthly data segmentation
cache_config = {
    "segmentation": "monthly",  # Optimal for large datasets
    "compression": True,
    "format": "parquet"  # Future: 5x smaller than pickle
}
```

## Try It Yourself

1. **Run the benchmark**: `python performance/bench.py --verbose`
2. **Enable caching**: Set `cache_enabled=True` in your store config
3. **Monitor a strategy**: Add `enable_health_monitoring=True` to cerebro.run()
4. **Profile bottlenecks**: Use `--profile` flag for detailed analysis

**Next**: [Benchmarking Guide](benchmarking.md) | [Large Datasets](large_datasets.md) | [Optimization Roadmap](optimization_roadmap.md)
