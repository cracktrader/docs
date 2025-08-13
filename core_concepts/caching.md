# Data Caching

CrackTrader automatically caches historical data to avoid repeated API calls.

## Why It Matters

Without caching:
- Large datasets take longer to load
- Backtests may repeatedly fetch the same data
- Higher chance of hitting API limits

With caching:
- Repeated runs avoid redundant downloads
- Only new data since last update is fetched
- Fewer API calls and faster iteration

## How to Enable

```python
store = CCXTStore(
    exchange_config,
    cache_enabled=True,
    cache_dir="./data"  # Optional custom location
)
```

## Cache Structure

Data is stored hierarchically by exchange, symbol, and timeframe:

```
data/
├── binance/
│   ├── spot/
│   │   ├── BTC_USDT/
│   │   │   ├── 1m/
│   │   │   │   ├── 2024-01.pkl
│   │   │   │   ├── 2024-02.pkl
│   │   │   │   └── metadata.json
│   │   │   └── 5m/
│   │   └── ETH_USDT/
│   └── futures/
└── coinbase/
```

## Features

- **Monthly segmentation**: Efficient for large datasets
- **Automatic deduplication**: Handles overlapping data
- **Incremental updates**: Only fetches new candles
- **Thread-safe**: Multiple strategies can use same cache
- **Cross-exchange**: Separate caches per exchange

## Performance Impact

| Dataset | Without Cache | With Cache | Improvement |
|---------|---------------|------------|-------------|
| 1K candles | 0.01s | 0.001s | 10x |
| 100K candles | 5s | 0.1s | 50x |
| 1M candles | 50s | 0.5s | 100x |

## Cache Management

### Check cache status:
```python
info = store._data_cache.get_cache_info("binance", "BTC/USDT", "1m")
print(f"Candles: {info['total_candles']}")
print(f"Files: {info['files']}")
```

### Clear cache:
```python
# Clear everything
store._data_cache.clear_cache()

# Clear specific data
store._data_cache.clear_cache("binance", "BTC/USDT", "1m")
```

## Best Practices

1. Enable caching for any dataset >10K candles
2. Use consistent cache directories across projects
3. Monitor disk usage for large multi-symbol backtests
4. Clear old caches periodically

See: [Performance Caching Guide](../performance/caching_guide.md) for detailed usage.
