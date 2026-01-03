# Quickstart

Get CrackTrader running in minutes.

## Installation

```bash
# Install from GitHub
pip install git+https://github.com/cracktrader/cracktrader.git

# With web interface
pip install "git+https://github.com/cracktrader/cracktrader.git[web]"

# Development setup (if you've cloned the repo)
pip install -e ".[dev]"
```

!!! tip "Repository URL"
    The repository URL may change in the future. Check the latest installation instructions in the README.

## Your First Strategy (backtest)

This example creates a simple moving average crossover strategy that trades Bitcoin.

```python
import backtrader as bt
import cracktrader as ct

class SimpleMA(ct.Strategy):
    """
    Simple Moving Average strategy that:
    - Buys when price crosses above the 20-period moving average
    - Sells when price crosses below the moving average
    """
    def __init__(self):
        self.sma = bt.indicators.SMA(period=20)

    def next(self):
        current_price = self.data.close[0]
        sma_value = self.sma[0]

        if current_price > sma_value and not self.position:
            self.buy(size=0.1)

        elif current_price < sma_value and self.position:
            self.close()

# Create a session for Binance in backtest mode
session = ct.exchange('binance', mode='backtest')

# Get a data feed from the session
feed = session.feed(
    symbol='BTC/USDT',
    timeframe='1h'
)

# Create backtesting engine
cerebro = ct.Cerebro()
cerebro.adddata(feed)
cerebro.addstrategy(SimpleMA)
cerebro.broker.setcash(10000)

# Run backtest
print(f"Starting Portfolio Value: {cerebro.broker.getvalue():.2f}")
results = cerebro.run()
print(f"Final Portfolio Value: {cerebro.broker.getvalue():.2f}")
```

## Trading Modes

CrackTrader supports multiple trading modes. To switch, just change the `mode` in the `ct.exchange()` call.

### Paper Trading Mode
```python
# Paper trading uses a simulated broker with a live data feed
session = ct.exchange('binance', mode='paper')
feed = session.feed(symbol='BTC/USDT', timeframe='1m', live=True)

cerebro = ct.Cerebro()
cerebro.adddata(feed)
cerebro.addstrategy(SimpleMA)
cerebro.broker.setcash(10000)
cerebro.run(runonce=False) # Stream live data
```

### Live Trading Mode
```python
# Live trading uses your real exchange account
session = ct.exchange('binance', mode='live', apiKey='YOUR_KEY', secret='YOUR_SECRET')
feed = session.feed(symbol='BTC/USDT', timeframe='1m', live=True)

cerebro = ct.Cerebro()
cerebro.adddata(feed)
cerebro.addstrategy(SimpleMA)
cerebro.run(runonce=False)
```

!!! warning "Live Trading"
    Live trading uses real money. Always test thoroughly in paper mode first!

## Prediction Markets (Polymarket, Kalshi)

Prediction markets use the same `ct.exchange(...).feed()/broker()` workflow.

```python
import cracktrader as ct

# Polymarket
pm = ct.exchange("polymarket", mode="paper", enable_network=True)
pm_feed = pm.feed(symbol="PM:demo-event:yes", granularity="1m", live=False)
pm_broker = pm.broker()  # paper broker by default

# Kalshi (scaffold)
kalshi = ct.exchange("kalshi", mode="paper", enable_network=False)
kalshi_feed = kalshi.feed(symbol="K:TEST", live=False)
kalshi_broker = kalshi.broker()
```

## Next Steps

- [First Strategy Tutorial](first_strategy.md) - Step-by-step guide
- [Configuration Guide](configuration.md) - All config options
- [Examples](../examples/) - More strategy examples
- [Performance](../performance/overview.md) - Optimize for large datasets
