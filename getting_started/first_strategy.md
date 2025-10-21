# First Strategy Tutorial

Step-by-step guide to building your first CrackTrader strategy.

## What We'll Build

A simple moving average crossover strategy that:

- Buys when a fast moving average crosses above a slow one.
- Sells when the fast moving average crosses below the slow one.
- Runs a historical backtest on Binance BTC/USDT data.

## Step 1: Setup

Create a new file `my_strategy.py` and import `backtrader` and `cracktrader`:

```python
import backtrader as bt
import cracktrader as ct
```

## Step 2: Define the Strategy

This is a standard `backtrader.Strategy` class. `cracktrader.Strategy` is a helpful alias with some extra logging helpers, but the core logic is the same.

```python
class MovingAverageCross(ct.Strategy):
    # Strategy parameters
    params = (
        ('fast_period', 10),
        ('slow_period', 30),
    )

    def __init__(self):
        # Calculate moving averages
        self.fast_ma = bt.indicators.SMA(self.data.close, period=self.p.fast_period)
        self.slow_ma = bt.indicators.SMA(self.data.close, period=self.p.slow_period)

        # Crossover signal
        self.crossover = bt.indicators.CrossOver(self.fast_ma, self.slow_ma)

    def next(self):
        # Buy signal: fast MA crosses above slow MA
        if not self.position and self.crossover > 0:
            self.buy(size=0.1)
            print(f"BUY signal at {self.data.close[0]:.2f}")

        # Sell signal: fast MA crosses below slow MA
        elif self.position and self.crossover < 0:
            self.close()
            print(f"SELL signal at {self.data.close[0]:.2f}")

    def notify_trade(self, trade):
        if trade.isclosed:
            print(f"TRADE CLOSED: PnL: {trade.pnl:.2f}")
```

## Step 3: Setup Data and Broker

This is where CrackTrader's session helper simplifies things. We create a session for the exchange, then ask it for feeds and a broker.

```python
def run_strategy():
    # Create a session for Binance in backtest mode
    session = ct.exchange('binance', mode='backtest')

    # Setup data feed (backtesting mode)
    feed = session.feed(
        symbol='BTC/USDT',
        timeframe='1h',
        historical_limit=1000,
        live=False  # Important for backtesting
    )

    # Create cerebro engine
    cerebro = ct.Cerebro()

    # Add strategy and data
    cerebro.addstrategy(MovingAverageCross)
    cerebro.adddata(feed)

    # Set initial capital
    cerebro.broker.setcash(10000.0)

    # Add analyzers
    cerebro.addanalyzer(bt.analyzers.SharpeRatio, _name='sharpe')
    cerebro.addanalyzer(bt.analyzers.DrawDown, _name='drawdown')
    cerebro.addanalyzer(bt.analyzers.TradeAnalyzer, _name='trades')

    # Run backtest
    print(f"Starting portfolio value: {cerebro.broker.getvalue():.2f}")
    results = cerebro.run()
    print(f"Final portfolio value: {cerebro.broker.getvalue():.2f}")

    # Print results
    strategy = results[0]
    sharpe = strategy.analyzers.sharpe.get_analysis().get('sharperatio', 'N/A')
    drawdown = strategy.analyzers.drawdown.get_analysis()['max']['drawdown']
    trades = strategy.analyzers.trades.get_analysis()
    
    print(f"Sharpe Ratio: {sharpe}")
    print(f"Max Drawdown: {drawdown:.2f}%")
    print(f"Total trades: {trades.get('total', {}).get('total', 0)}")

if __name__ == '__main__':
    run_strategy()
```

!!! info "What `ct.exchange` Does"
    The `ct.exchange()` helper is the recommended way to start. It creates a shared connection (`Store`) that is automatically passed to the feeds and brokers it generates. This ensures they all coordinate without you having to wire them up manually.

## Step 4: Run the Strategy

```bash
python my_strategy.py
```

Expected output:
```
Starting portfolio value: 10000.00
BUY signal at 45000.00
TRADE CLOSED: PnL: -22.50
...
Final portfolio value: 10150.00
Sharpe Ratio: 0.85
Max Drawdown: 5.20%
Total trades: 12
```

## Step 5: Live Trading

To switch to live paper trading, just change the `mode` in the session and set `live=True` for the feed.

```python
# Change these settings for live paper trading
session = ct.exchange('binance', mode='paper')

data = session.feed(
    symbol='BTC/USDT',
    timeframe='1h',
    live=True
)

# For real money, change mode to 'live' and provide API keys
# session = ct.exchange('binance', mode='live', apiKey='...', secret='...')
```

## Next Steps

- [Configuration Guide](configuration.md) - Set up multiple exchanges
- [Examples](../examples/basic_strategy.md) - More strategy examples
- [Performance](../performance/overview.md) - Optimize for large datasets
