# Configuration Reference

This page provides a comprehensive reference for all configuration options in CrackTrader.

## Exchange Configuration

### Basic Settings

```json
{
  "exchange": {
    "name": "binance",
    "sandbox": true,
    "apiKey": "your_api_key",
    "secret": "your_secret",
    "password": "your_passphrase"
  }
}
```

### Rate Limiting

```json
{
  "rateLimit": {
    "requests_per_second": 10,
    "burst_size": 20
  }
}
```

## Data Feed Configuration

### WebSocket Settings

```json
{
  "websocket": {
    "enabled": true,
    "reconnect_delay": 5,
    "max_reconnects": 10
  }
}
```

### OHLCV Settings

```json
{
  "ohlcv": {
    "timeframes": ["1m", "5m", "15m", "1h", "4h", "1d"],
    "limit": 1000,
    "cache_size": 10000
  }
}
```

## Broker Configuration

### Paper Trading

```json
{
  "paper_trading": {
    "initial_cash": 10000,
    "commission": 0.001,
    "slippage": 0.0005
  }
}
```

### Live Trading

```json
{
  "live_trading": {
    "max_position_size": 0.1,
    "stop_loss": true,
    "take_profit": true
  }
}
```

## Logging Configuration

```json
{
  "logging": {
    "level": "INFO",
    "format": "json",
    "file": "cracktrader.log",
    "max_size": "100MB",
    "backup_count": 5
  }
}
```

## Performance Configuration

```json
{
  "performance": {
    "cache_enabled": true,
    "cache_size": 10000,
    "parallel_processing": true,
    "max_workers": 4
  }
}
```

## Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `CRACKTRADER_CONFIG` | Path to config file | `config.json` |
| `CRACKTRADER_LOG_LEVEL` | Log level | `INFO` |
| `CRACKTRADER_CACHE_DIR` | Cache directory | `.cache` |
| `CRACKTRADER_DATA_DIR` | Data directory | `data` |

## Configuration File Locations

1. `./config.json` (current directory)
2. `~/.cracktrader/config.json` (user home)
3. `/etc/cracktrader/config.json` (system-wide)

Configuration files are loaded in order, with later files overriding earlier ones.
