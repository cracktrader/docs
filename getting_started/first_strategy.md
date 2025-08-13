# First Strategy Tutorial

Step-by-step guide to building your first CrackTrader strategy.

## What We'll Build

A simple moving average crossover strategy that:
- Buys when fast MA crosses above slow MA
- Sells when fast MA crosses below slow MA
- Uses OCO orders for risk management

## Step 1: Setup

Create a new file `my_strategy.py`:

```python
import backtrader as bt
from cracktrader import CCXTStore, CCXTDataFeed
```

## Step 2: Define the Strategy

```python
class MovingAverageCross(bt.Strategy):
    # Strategy parameters
    params = (
        ('fast_period', 10),
        ('slow_period', 30),
        ('risk_percent', 0.02),  # 2% risk per trade
    )

    def __init__(self):
        # Calculate moving averages
        self.fast_ma = bt.indicators.SMA(self.data.close, period=self.p.fast_period)
        self.slow_ma = bt.indicators.SMA(self.data.close, period=self.p.slow_period)

        # Crossover signal
        self.crossover = bt.indicators.CrossOver(self.fast_ma, self.slow_ma)

        # Track orders
        self.order = None

    def next(self):
        # Skip if we have a pending order
        if self.order:
            return

        # Buy signal: fast MA crosses above slow MA
        if not self.position and self.crossover > 0:
            # Calculate position size (2% risk)
            current_price = self.data.close[0]
            stop_price = current_price * 0.98  # 2% stop loss
            risk_per_share = current_price - stop_price
            position_size = (self.broker.cash * self.p.risk_percent) / risk_per_share

            # Place OCO order with stop loss and take profit
            self.order = self.buy_bracket(
                size=position_size,
                price=current_price,
                stopprice=stop_price,         # Stop loss at -2%
                limitprice=current_price * 1.06  # Take profit at +6%
            )
            print(f"BUY signal: {current_price:.2f}, size: {position_size:.4f}")

        # Sell signal: fast MA crosses below slow MA
        elif self.position and self.crossover < 0:
            self.order = self.sell()
            print(f"SELL signal: {self.data.close[0]:.2f}")

    def notify_order(self, order):
        if order.status in [order.Submitted, order.Accepted]:
            return

        if order.status in [order.Completed]:
            if order.isbuy():
                print(f"BUY EXECUTED: {order.executed.price:.2f}")
            else:
                print(f"SELL EXECUTED: {order.executed.price:.2f}")

        self.order = None

    def notify_trade(self, trade):
        if trade.isclosed:
            print(f"TRADE CLOSED: PnL: {trade.pnl:.2f}")
```

## Step 3: Setup Data and Broker

```python
def run_strategy():
    # Configure exchange connection (sandbox recommended for testing)
    store = CCXTStore(
        exchange='binance',
        sandbox=True,
        cache_enabled=True,
        cache_dir='./data'
    )

    # Setup data feed (backtesting mode)
    data = CCXTDataFeed(
        store=store,
        symbol='BTC/USDT',
        ccxt_timeframe='1h',
        historical_limit=1000,
        live=False
    )

    # Create cerebro engine
    cerebro = bt.Cerebro()

    # Add strategy and data
    cerebro.addstrategy(MovingAverageCross)
    cerebro.adddata(data)

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
    print(f"Sharpe Ratio: {strategy.analyzers.sharpe.get_analysis().get('sharperatio', 'N/A')}")
    print(f"Max Drawdown: {strategy.analyzers.drawdown.get_analysis()['max']['drawdown']:.2f}%")

    trade_analysis = strategy.analyzers.trades.get_analysis()
    total = trade_analysis.get('total', {}).get('total', 0)
    wins = trade_analysis.get('won', {}).get('total', 0)
    win_rate = (wins / total * 100) if total else 0
    print(f"Total trades: {total}")
    print(f"Win rate: {win_rate:.1f}%")

if __name__ == '__main__':
    run_strategy()
```

### Broker choices

- Backtest or paper: use the built-in backtesting broker.
- Live: use the CCXT live broker to send real orders.

Backtest/paper (optional explicit broker):
```python
from cracktrader.broker import BrokerFactory

broker = BrokerFactory.create(mode='paper', cash=10000)
cerebro.setbroker(broker)
```

Live trading (testnet or live):
```python
from cracktrader.broker import BrokerFactory

store = CCXTStore(exchange='binance', sandbox=True, config={'apiKey': '...', 'secret': '...'})
broker = BrokerFactory.create(mode='live', store=store)
cerebro.setbroker(broker)
```

Note: CCXTStore uses a registry (singleton-like) keyed by exchange and settings. Creating a store with the same arguments returns the same instance, so your broker and data feed automatically share the connection.

## Step 4: Run the Strategy

```bash
python my_strategy.py
```

Expected output:
```
Starting portfolio value: 10000.00
BUY signal: 45000.00, size: 0.0045
BUY EXECUTED: 45000.00
SELL signal: 44500.00
SELL EXECUTED: 44500.00
TRADE CLOSED: PnL: -22.50
...
Final portfolio value: 10150.00
Sharpe Ratio: 0.85
Max Drawdown: 5.2%
Total trades: 12
Win rate: 58.3%
```

## Step 5: Optimization

### Test Different Parameters
```python
# Add optimization to run_strategy()
cerebro.optstrategy(
    MovingAverageCross,
    fast_period=range(5, 20, 5),    # Test 5, 10, 15
    slow_period=range(20, 50, 10)   # Test 20, 30, 40
)
```

### Add More Indicators
```python
def __init__(self):
    # ... existing code ...

    # Add RSI filter
    self.rsi = bt.indicators.RSI(self.data.close)

def next(self):
    # Only buy if RSI is not overbought
    if not self.position and self.crossover > 0 and self.rsi < 70:
        # ... buy logic ...
```

## Step 6: Live Trading

Once backtesting looks good:

```python
# Change these settings for live trading
store = CCXTStore(
    exchange='binance',
    sandbox=False,  # Live trading
    config={'apiKey': '...', 'secret': '...'},
    cache_enabled=False
)

data = CCXTDataFeed(
    store=store,
    symbol='BTC/USDT',
    ccxt_timeframe='1h',
    live=True
)
```

## Common Issues

**No trades**: Check for enough historical data for MA calculation
**Cache errors**: Ensure `./data` is writable
**API errors**: Verify API keys in your config
**Slow backtests**: Enable caching with `cache_enabled=True`

## Next Steps

- [Configuration Guide](configuration.md) - Set up multiple exchanges
- [Examples](../examples/basic_strategy.md) - More strategy examples
- [Performance](../performance/overview.md) - Optimize for large datasets

## Full Example Source

The complete moving average crossover example is available here:

--8<-- "examples/moving_average_cross.py"
