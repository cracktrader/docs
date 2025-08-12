# Performance Overview

Current performance characteristics and optimization status.

## Performance Philosophy

- Measure everything with `bench.py` tool
- Cache historical data to avoid repeated API calls
- Profile bottlenecks before optimizing
- Document what's actually slow vs what's fast

## Current Performance Baseline

### System Performance (Latest Benchmarks)

| Component | Mock Mode | Sandbox Mode | Notes |
|-----------|-----------|--------------|-------|
| **Store Creation** | 4.6s | 0.6s | One-time startup cost |
| **Order Processing** | 17ms | N/A | Our code overhead (very good!) |
| **Data Processing** | 5ms | N/A | 1000 candles, raw processing |
| **Network Operations** | N/A | 200-1000ms | External exchange latency |
| **Total Benchmark** | 1.6s | 775ms | Full system test |

Performance Score: 56/100 (sandbox mode with network latency included)

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
    exchange_config,
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
    timeframe="1m",
    historical_limit=1000000,    # 1M candles - handled efficiently
    streaming=True               # Memory-efficient streaming
)
```

### **3. Performance Monitoring**

Built-in performance tracking for production deployments:

```python
from cracktrader.utils import HealthMonitor

# Automatic performance tracking
monitor = HealthMonitor()
with monitor.track_operation("strategy_execution"):
    # Your strategy code
    results = cerebro.run()

# View performance metrics
print(monitor.get_performance_summary())
```

## 🔍 **Bottleneck Analysis**

### **What's Fast** ✅
- **Order processing**: 17ms overhead (excellent for Python)
- **Data validation**: Sub-millisecond for typical operations
- **WebSocket streaming**: Near real-time latency
- **Cached data access**: 0.1-0.5s for million-candle datasets

### **What's Slow** ⚠️
1. **Network calls**: 200-1000ms (external exchange limitation)
2. **Initial data loading**: Without caching, CSV reading can take 20-80s
3. **Complex indicators**: Pandas operations on large datasets
4. **System startup**: Test infrastructure import overhead (4.6s)

### **Optimization Targets** 🎯
1. **Historical data pipeline** (90% solved with caching)
2. **Technical indicators** for large datasets (Rust candidate)
3. **Network layer** optimization (connection pooling)
4. **Memory usage** for multi-million candle backtests

## 🚀 **Performance Tools**

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

**Output includes**:
- Component-by-component timing
- Memory usage per operation
- Bottleneck identification
- Optimization recommendations (including Rust candidates!)

### **Performance Reports**

All benchmarks generate detailed reports:

```
performance/reports/
├── benchmark_mock_20250812.json      # Detailed metrics
├── memory_profile_mock_20250812.txt  # Memory analysis
├── latest_benchmark_mock.json        # Quick access
└── optimization_recommendations.json # Action items
```

## 📈 **Optimization Roadmap**

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

## 💡 **Performance Best Practices**

### **For Backtesting**
```python
# ✅ Enable caching for repeated tests
store = CCXTStore(exchange_config, cache_enabled=True)

# ✅ Use appropriate historical limits
feed = CCXTDataFeed(historical_limit=50000)  # Not unlimited

# ✅ Profile your strategies
python performance/bench.py --profile
```

### **For Live Trading**
```python
# ✅ Monitor system health
cerebro.run(enable_health_monitoring=True)

# ✅ Use connection pooling
store = CCXTStore(exchange_config, connection_pool_size=10)

# ✅ Set appropriate log levels
import logging
logging.getLogger('cracktrader').setLevel(logging.WARNING)
```

### **For Large Datasets**
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

## 🎮 **Try It Yourself**

1. **Run the benchmark**: `python performance/bench.py --verbose`
2. **Enable caching**: Set `cache_enabled=True` in your store config
3. **Monitor a strategy**: Add `enable_health_monitoring=True` to cerebro.run()
4. **Profile bottlenecks**: Use `--profile` flag for detailed analysis

**Next**: [Benchmarking Guide](benchmarking.md) | [Large Datasets](large_datasets.md) | [Optimization Roadmap](optimization_roadmap.md)
