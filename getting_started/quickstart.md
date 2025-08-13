# Quickstart

Get CrackTrader running in minutes.

## Installation

```bash
# Install from GitHub
pip install git+https://github.com/LachlanBridges/cracktrader.git

# With web interface
pip install "git+https://github.com/LachlanBridges/cracktrader.git[web]"

# Development setup (if you've cloned the repo)
pip install -e ".[dev]"
```

!!! tip "Repository URL"
    The repository URL may change in the future. Check the latest installation instructions in the README.

## Configuration

Create `config.json`:

```json
{
  "binance": {
    "sandbox": {
      "apiKey": "your_testnet_key",
      "secret": "your_testnet_secret"
    },
    "live": {
      "apiKey": "your_live_key",
      "secret": "your_live_secret"
    }
  }
}
```

### Adding Other Exchanges

CrackTrader supports 400+ exchanges via CCXT. To add another exchange:

```json
{
  "binance": { /* ... */ },
  "kucoin": {
    "sandbox": {
      "apiKey": "your_kucoin_testnet_key",
      "secret": "your_kucoin_testnet_secret"
    }
  },
  "okx": {
    "live": {
      "apiKey": "your_okx_live_key",
      "secret": "your_okx_live_secret",
      "password": "your_okx_passphrase"
    }
  }
}
```

!!! info "Configuration Guide"
    See the [Configuration Guide](configuration.md) for all available options, exchange-specific settings, and advanced configuration.

## Your First Strategy (backtest)

This example creates a simple moving average crossover strategy that trades Bitcoin. Here's what each part does:

```python
import backtrader as bt
from cracktrader import CCXTStore
from cracktrader.feeds import CCXTDataFeed

class SimpleMA(bt.Strategy):
    """
    Simple Moving Average strategy that:
    - Buys when price crosses above the 20-period moving average
    - Sells when price crosses below the moving average
    """
    def __init__(self):
        # SMA is a built-in Backtrader indicator
        # Available indicators: RSI, MACD, Bollinger Bands, etc.
        self.sma = bt.indicators.SMA(period=20)

    def next(self):
        # Called for each new price bar
        current_price = self.data.close[0]  # Current closing price
        sma_value = self.sma[0]            # Current SMA value

        # Buy signal: price above SMA and no existing position
        if current_price > sma_value and not self.position:
            self.buy()

        # Sell signal: price below SMA and holding position
        elif current_price < sma_value and self.position:
            self.sell()

# Setup data connection
store = CCXTStore(exchange='binance', sandbox=True)
data = CCXTDataFeed(
    store=store,
    symbol='BTC/USDT',      # Trading pair
    ccxt_timeframe='1h'     # 1-hour candles
)

# Create backtesting engine
cerebro = bt.Cerebro()
cerebro.adddata(data)          # Add price data
cerebro.addstrategy(SimpleMA)  # Add trading strategy
cerebro.broker.setcash(10000)  # Set starting capital

# Run backtest
print(f"Starting Portfolio Value: {cerebro.broker.getvalue():.2f}")
results = cerebro.run()
print(f"Final Portfolio Value: {cerebro.broker.getvalue():.2f}")
```

### Understanding Backtrader Indicators

Backtrader comes with 100+ built-in indicators. The `SMA` (Simple Moving Average) used above is just one example:

- **bt.indicators.SMA()** - Simple Moving Average
- **bt.indicators.RSI()** - Relative Strength Index
- **bt.indicators.MACD()** - Moving Average Convergence Divergence
- **bt.indicators.BollingerBands()** - Bollinger Bands

All indicators work the same way - they analyze price data and provide signals for your strategy logic.

## Test It Works

```bash
# Run basic example
python examples/basic_strategy.py

# Run performance test
python performance/bench.py

# Check connections
python scripts/test_sandbox_connection.py
```

## Data Caching

CrackTrader **automatically caches** historical data to speed up backtests. This is enabled by default:

```python
# Caching is enabled by default - no configuration needed!
store = CCXTStore(exchange='binance', sandbox=True)

# Customize cache settings (optional)
store = CCXTStore(
    exchange='binance',
    sandbox=True,
    cache_enabled=True,        # Default: True
    cache_dir="./my_cache"     # Default: "./data"
)

# Disable caching (not recommended)
store = CCXTStore(
    exchange='binance',
    sandbox=True,
    cache_enabled=False
)
```

!!! tip "Cache Benefits"
    Cached backtests run 10x faster on subsequent runs. The first run fetches data from the exchange, later runs use cached data.

## Next Steps

- [First Strategy Tutorial](first_strategy.md) - Step-by-step guide
- [Configuration Guide](configuration.md) - All config options
- [Examples](../examples/) - More strategy examples
- [Performance](../performance/overview.md) - Optimize for large datasets

## Common Issues

- Connection errors: Check your API keys in `config.json`
- Slow backtests: Enable caching with `cache_enabled=True`
- Test failures: Run `python -m pytest tests/unit/` to check setup

## Trading Modes

CrackTrader supports three trading modes using the same strategy code. Even backtests need a broker - the system automatically selects the right one:

### Backtest Mode (default)
```python
# Backtest uses CCXTBackBroker automatically - no broker setup needed
cerebro = bt.Cerebro()
cerebro.adddata(data)
cerebro.addstrategy(SimpleMA)
results = cerebro.run()  # Uses historical data only
```

### Paper Trading Mode
```python
from cracktrader.broker import BrokerFactory

store = CCXTStore(exchange='binance', sandbox=True)
broker = BrokerFactory.create(mode='paper', store=store)

cerebro = bt.Cerebro()
cerebro.setbroker(broker)  # Use paper trading broker
cerebro.adddata(data)      # Live data feed
cerebro.addstrategy(SimpleMA)
results = cerebro.run()    # Trades with virtual money
```

### Live Trading Mode
```python
from cracktrader.broker import BrokerFactory

store = CCXTStore(exchange='binance', sandbox=False)  # Use live exchange
broker = BrokerFactory.create(mode='live', store=store)

cerebro = bt.Cerebro()
cerebro.setbroker(broker)  # Use live trading broker
cerebro.adddata(data)      # Live data feed
cerebro.addstrategy(SimpleMA)
results = cerebro.run()    # Real trades with real money!
```

!!! warning "Live Trading"
    Live trading uses real money. Always test thoroughly in sandbox/paper mode first!
