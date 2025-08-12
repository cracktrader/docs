# Quick Start

Get CrackTrader running in 5 minutes.

## Installation

```bash
# Basic installation
pip install -e .

# With web interface
pip install -e ".[web]"

# Development setup
pip install -e ".[dev]"
```

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

## Your First Strategy

```python
import backtrader as bt
from cracktrader import CCXTStore, CCXTDataFeed

class SimpleMA(bt.Strategy):
    def __init__(self):
        self.sma = bt.indicators.SMA(period=20)

    def next(self):
        if self.data.close[0] > self.sma[0] and not self.position:
            self.buy()
        elif self.data.close[0] < self.sma[0] and self.position:
            self.sell()

# Setup
store = CCXTStore({'exchange': 'binance', 'sandbox': True})
data = CCXTDataFeed(store=store, symbol='BTC/USDT', timeframe='1h')

# Run
cerebro = bt.Cerebro()
cerebro.adddata(data)
cerebro.addstrategy(SimpleMA)
results = cerebro.run()
```

## Test It Works

```bash
# Run basic example
python examples/basic_strategy.py

# Run performance test
python performance/bench.py

# Check connections
python scripts/test_sandbox_connection.py
```

## Enable Caching

For faster backtesting:

```python
store = CCXTStore(
    {'exchange': 'binance', 'sandbox': True},
    cache_enabled=True,
    cache_dir="./data"
)
```

## Next Steps

- [First Strategy Tutorial](first_strategy.md) - Step-by-step guide
- [Configuration Guide](configuration.md) - All config options
- [Examples](../examples/) - More strategy examples
- [Performance](../performance/overview.md) - Optimize for large datasets

## Common Issues

**Connection errors**: Check your API keys in `config.json`
**Slow backtests**: Enable caching with `cache_enabled=True`
**Test failures**: Run `python -m pytest tests/unit/` to check setup
