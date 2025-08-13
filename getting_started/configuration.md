# Configuration

How to configure CrackTrader for different exchanges and environments.

## Config File Structure

Create `config.json` in your project root:

```json
{
  "binance": {
    "sandbox": {
      "apiKey": "your_testnet_api_key",
      "secret": "your_testnet_secret"
    },
    "live": {
      "apiKey": "your_live_api_key",
      "secret": "your_live_secret"
    }
  },
  "coinbase": {
    "sandbox": {
      "apiKey": "coinbase_sandbox_key",
      "secret": "coinbase_sandbox_secret",
      "passphrase": "coinbase_passphrase"
    }
  }
}
```

## Exchange-Specific Settings

### Binance
```json
{
  "binance": {
    "sandbox": {
      "apiKey": "...",
      "secret": "...",
      "enableRateLimit": true,
      "rateLimit": 1200,
      "sandbox": true
    }
  }
}
```

### Coinbase
```json
{
  "coinbase": {
    "sandbox": {
      "apiKey": "...",
      "secret": "...",
      "passphrase": "...",
      "sandbox": true,
      "enableRateLimit": true
    }
  }
}
```

## Environment Variables

Alternative to config file:

```bash
# Binance
export BINANCE_API_KEY="your_api_key"
export BINANCE_SECRET="your_secret"

# Coinbase
export COINBASE_API_KEY="your_api_key"
export COINBASE_SECRET="your_secret"
export COINBASE_PASSPHRASE="your_passphrase"
```

## Store Configuration

```python
from cracktrader import CCXTStore

# Basic configuration
store = CCXTStore(exchange='binance', sandbox=True)

# With caching enabled
store = CCXTStore(
    exchange='binance',
    sandbox=True,
    cache_enabled=True,
    cache_dir="./trading_data"
)

# With CCXT-specific settings
store = CCXTStore(
    exchange='binance',
    sandbox=True,
    config={'enableRateLimit': True, 'rateLimit': 1200},
    cache_enabled=True,
    cache_dir="./data"
)
```

## Cache Configuration

```python
# Default cache location
cache_dir = "./data"

# User-specific cache
cache_dir = os.path.expanduser("~/trading_data")

# Project-specific cache
cache_dir = "./cache/backtest_data"
```

## Logging Configuration

```python
import logging

# Set log levels
logging.getLogger('cracktrader').setLevel(logging.INFO)
logging.getLogger('cracktrader.store').setLevel(logging.DEBUG)
logging.getLogger('cracktrader.broker').setLevel(logging.WARNING)
```

## Common Settings

- Backtesting: sandbox can be False; enable caching for repeat runs.
- Live trading: sandbox=False; pass API keys via `config={...}`; disable caching.
- Development: sandbox=True; enable caching for speed; keep logging at INFO.

## Security Best Practices

1. **Never commit API keys** to version control
2. **Use environment variables** for sensitive data
3. **Use sandbox mode** for development
4. **Rotate API keys** regularly
5. **Limit API key permissions** (trading, reading only)
