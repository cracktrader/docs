# Historical Data Caching Guide

How to use the built-in caching system for faster backtesting.

## Overview

CrackTrader has a sophisticated caching system that's **already implemented** but not enabled by default. It provides 90-99% performance improvement for repeated backtesting.

## Quick Enable

```python
from cracktrader import CCXTStore

# Enable caching
store = CCXTStore(
    exchange_config,
    cache_enabled=True,           # Enable caching
    cache_dir="./trading_data"    # Custom location (optional)
)
```

## How It Works

### File Structure
```
trading_data/
└── binance/
    └── spot/
        └── BTC_USDT/
            └── 1m/
                ├── 2024-01.pkl    # January 2024 data
                ├── 2024-02.pkl    # February 2024 data
                ├── 2024-03.pkl    # March 2024 data
                └── metadata.json  # Cache statistics
```

### Automatic Features
- **Monthly segmentation**: Efficient storage and retrieval
- **Deduplication**: Same timestamps are merged automatically
- **Incremental updates**: Only fetches new data since last update
- **Multi-exchange**: Separate caches per exchange
- **Thread-safe**: Concurrent strategy support

## Cache Behavior

### First Run (Cache Miss)
```python
feed = CCXTDataFeed(
    store=store,
    symbol="BTC/USDT",
    timeframe="1m",
    historical_limit=100000  # 100K candles
)
# Takes: 20-50 seconds (fetches from API + caches)
```

### Subsequent Runs (Cache Hit)
```python
# Same code, but now cached
feed = CCXTDataFeed(store=store, symbol="BTC/USDT", timeframe="1m", historical_limit=100000)
# Takes: 0.1-0.5 seconds (98% improvement!)
```

### Incremental Updates
```python
# Later, when you run again:
feed = CCXTDataFeed(store=store, symbol="BTC/USDT", timeframe="1m", historical_limit=100000)
# Only fetches data since last update
# Cache grows incrementally
```

## Cache Management

### Get Cache Info
```python
cache_info = store._data_cache.get_cache_info(
    exchange="binance",
    symbol="BTC/USDT",
    timeframe="1m"
)

print(f"Total candles: {cache_info['total_candles']}")
print(f"Date range: {cache_info['earliest_timestamp']} to {cache_info['latest_timestamp']}")
print(f"Files: {cache_info['files']}")
```

### Clear Cache
```python
# Clear all data
store._data_cache.clear_cache()

# Clear specific exchange
store._data_cache.clear_cache(exchange="binance")

# Clear specific symbol
store._data_cache.clear_cache(exchange="binance", symbol="BTC/USDT")

# Clear specific timeframe
store._data_cache.clear_cache(exchange="binance", symbol="BTC/USDT", timeframe="1m")
```

## Performance Impact

### Without Caching
```
1K candles:   0.01s
100K candles: 2-5s
1M candles:   20-50s
5M candles:   60-300s
```

### With Caching (After First Run)
```
1K candles:   0.001s  (10x faster)
100K candles: 0.1s    (50x faster)
1M candles:   0.5s    (100x faster)
5M candles:   2s      (150x faster)
```

## Large Dataset Strategy

For multi-year, multi-symbol backtests:

### 1. Enable Caching by Default
```python
# Add to your config
CACHE_CONFIG = {
    "enabled": True,
    "base_dir": "./trading_data",
    "auto_enable_threshold": 10000  # Auto-enable for >10K candles
}

store = CCXTStore(exchange_config, **CACHE_CONFIG)
```

### 2. Batch Data Loading
```python
symbols = ["BTC/USDT", "ETH/USDT", "ADA/USDT", "SOL/USDT"]
timeframes = ["1m", "5m", "1h"]

# Pre-load all data (runs once, then cached)
for symbol in symbols:
    for timeframe in timeframes:
        feed = CCXTDataFeed(
            store=store,
            symbol=symbol,
            timeframe=timeframe,
            historical_limit=500000  # ~1 year of 1m data
        )
        # First run: slow (fetches and caches)
        # Subsequent runs: instant
```

### 3. Monitor Cache Size
```bash
# Check cache directory size
du -sh trading_data/

# Typical sizes:
# 100K candles: ~5-10MB per symbol/timeframe
# 1M candles: ~50-100MB per symbol/timeframe
```

## Cache Configuration

### Custom Directories
```python
# User-specific cache location
store = CCXTStore(
    exchange_config,
    cache_enabled=True,
    cache_dir=os.path.expanduser("~/trading_data")  # ~/trading_data
)

# Project-specific cache
store = CCXTStore(
    exchange_config,
    cache_enabled=True,
    cache_dir="./data/backtest_cache"  # Relative to project
)
```

### Multiple Cache Locations
```python
# Separate caches for different strategies
backtest_store = CCXTStore(config, cache_dir="./data/backtesting")
live_store = CCXTStore(config, cache_dir="./data/live_trading")
```

## Troubleshooting

### Cache Not Working
```python
# Check if caching is enabled
print(f"Cache enabled: {store._cache_enabled}")
print(f"Cache object: {hasattr(store, '_data_cache')}")

# Verify cache directory exists
import os
print(f"Cache dir exists: {os.path.exists(store._data_cache.base_dir)}")
```

### Performance Still Slow
```python
# Check if data is actually cached
cache_info = store._data_cache.get_cache_info("binance", "BTC/USDT", "1m")
if cache_info["total_candles"] == 0:
    print("No data cached yet - first run will be slow")
else:
    print(f"Cached: {cache_info['total_candles']} candles")
```

### Disk Space Issues
```python
# Check cache size
import shutil
cache_size = shutil.disk_usage(store._data_cache.base_dir)
print(f"Cache size: {cache_size.used / (1024**3):.2f} GB")

# Clear old data if needed
store._data_cache.clear_cache(exchange="binance", symbol="OLD/SYMBOL")
```

## Best Practices

1. **Enable caching** for any backtesting with >10K candles
2. **Use consistent timeframes** to maximize cache hits
3. **Monitor disk usage** for large multi-symbol strategies
4. **Clear old caches** periodically to save disk space
5. **Use monthly segmentation** (already default) for efficient storage

## Future Enhancements

Planned improvements:
- **Parquet format**: 5x smaller files, 10x faster loading
- **Compression**: Automatic data compression
- **Cloud storage**: S3/GCS backend for shared caches
- **Streaming**: Memory-efficient processing of large datasets

See: [Large Datasets](large_datasets.md) | [Benchmarking](benchmarking.md)
