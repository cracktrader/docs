# Basic Strategy Example

Complete runnable example of a simple moving average strategy.

## The Strategy

```python
#!/usr/bin/env python3
"""
Simple Moving Average Crossover Strategy

Buy when fast MA crosses above slow MA
Sell when fast MA crosses below slow MA
"""

import backtrader as bt
from cracktrader import CCXTStore, CCXTDataFeed

class MovingAverageCross(bt.Strategy):
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

        # For plotting
        self.fast_ma.plotinfo.plot = True
        self.slow_ma.plotinfo.plot = True

    def next(self):
        # Buy signal
        if not self.position and self.crossover > 0:
            size = self.broker.get_cash() * 0.95 / self.data.close[0]
            self.buy(size=size)
            print(f"BUY at {self.data.close[0]:.2f}")

        # Sell signal
        elif self.position and self.crossover < 0:
            self.sell(size=self.position.size)
            print(f"SELL at {self.data.close[0]:.2f}")

    def notify_order(self, order):
        if order.status in [order.Submitted, order.Accepted]:
            return

        if order.status == order.Completed:
            if order.isbuy():
                print(f"BUY EXECUTED: ${order.executed.price:.2f}")
            else:
                print(f"SELL EXECUTED: ${order.executed.price:.2f}")

    def notify_trade(self, trade):
        if trade.isclosed:
            print(f"TRADE PnL: ${trade.pnl:.2f}")

def main():
    # Setup
    store = CCXTStore(
        {'exchange': 'binance', 'sandbox': True},
        cache_enabled=True,
        cache_dir="./data"
    )

    data = CCXTDataFeed(
        store=store,
        symbol='BTC/USDT',
        timeframe='1h',
        historical_limit=500,
        live=False
    )

    # Create cerebro
    cerebro = bt.Cerebro()
    cerebro.adddata(data)
    cerebro.addstrategy(MovingAverageCross)
    cerebro.broker.setcash(10000.0)

    # Add analyzers
    cerebro.addanalyzer(bt.analyzers.SharpeRatio, _name='sharpe')
    cerebro.addanalyzer(bt.analyzers.DrawDown, _name='drawdown')

    # Run
    print(f"Starting Portfolio Value: ${cerebro.broker.getvalue():.2f}")
    results = cerebro.run()
    final_value = cerebro.broker.getvalue()
    print(f"Final Portfolio Value: ${final_value:.2f}")

    # Results
    strategy = results[0]
    sharpe = strategy.analyzers.sharpe.get_analysis().get('sharperatio', 'N/A')
    drawdown = strategy.analyzers.drawdown.get_analysis()['max']['drawdown']

    print(f"Sharpe Ratio: {sharpe}")
    print(f"Max Drawdown: {drawdown:.2f}%")
    print(f"Total Return: {((final_value - 10000) / 10000) * 100:.2f}%")

    # Optional: Plot results
    # cerebro.plot()

if __name__ == '__main__':
    main()
```

## Run the Example

Save as `basic_strategy.py` and run:

```bash
python basic_strategy.py
```

Expected output:
```
Starting Portfolio Value: $10000.00
BUY at 45000.00
BUY EXECUTED: $45000.00
SELL at 44500.00
SELL EXECUTED: $44500.00
TRADE PnL: $-22.50
...
Final Portfolio Value: $10150.00
Sharpe Ratio: 0.85
Max Drawdown: 5.2%
Total Return: 1.5%
```

## Key Features

### Caching
The `cache_enabled=True` setting caches historical data:
- **First run**: Downloads and caches data (slower)
- **Subsequent runs**: Uses cached data (much faster)

### Risk Management
```python
# Position sizing based on available cash
size = self.broker.get_cash() * 0.95 / self.data.close[0]
```

### Signal Generation
```python
# Uses Backtrader's CrossOver indicator
self.crossover = bt.indicators.CrossOver(self.fast_ma, self.slow_ma)

# In next():
if self.crossover > 0:  # Fast MA crossed above slow MA
    self.buy()
```

## Customization

### Different Parameters
```python
cerebro.addstrategy(MovingAverageCross, fast_period=5, slow_period=20)
```

### Different Symbol/Timeframe
```python
data = CCXTDataFeed(
    store=store,
    symbol='ETH/USDT',    # Different symbol
    timeframe='4h',       # Different timeframe
    historical_limit=200
)
```

### Add Stop Loss
```python
def next(self):
    if not self.position and self.crossover > 0:
        # Entry with stop loss
        entry_price = self.data.close[0]
        stop_price = entry_price * 0.95  # 5% stop loss

        self.buy_bracket(
            size=size,
            price=entry_price,
            stopprice=stop_price
        )
```

## Next Steps

- [Multi-Asset Strategy](multi_asset.md) - Trade multiple symbols
- [Live Trading](live_trading.md) - Deploy to production
- [Web Dashboard](web_dashboard.md) - Monitor with web interface
- [Performance Guide](../performance/overview.md) - Optimize for speed
