# Strategies

Strategy development patterns and best practices for CrackTrader.

## Strategy Basics

CrackTrader uses standard Backtrader strategies:

```python
import backtrader as bt

class MyStrategy(bt.Strategy):
    params = (
        ('period', 20),
        ('risk_percent', 0.02),
    )

    def __init__(self):
        self.sma = bt.indicators.SMA(period=self.p.period)

    def next(self):
        if self.data.close[0] > self.sma[0] and not self.position:
            self.buy()
```

## Common Strategy Patterns

### Trend Following
```python
class TrendFollowing(bt.Strategy):
    def __init__(self):
        self.fast_ma = bt.indicators.SMA(period=10)
        self.slow_ma = bt.indicators.SMA(period=30)
        self.crossover = bt.indicators.CrossOver(self.fast_ma, self.slow_ma)

    def next(self):
        if self.crossover > 0:  # Golden cross
            self.buy()
        elif self.crossover < 0:  # Death cross
            self.sell()
```

### Mean Reversion
```python
class MeanReversion(bt.Strategy):
    def __init__(self):
        self.rsi = bt.indicators.RSI(period=14)
        self.sma = bt.indicators.SMA(period=20)

    def next(self):
        if self.rsi < 30 and self.data.close[0] < self.sma[0]:  # Oversold + below MA
            self.buy()
        elif self.rsi > 70:  # Overbought
            self.sell()
```

### Breakout Strategy
```python
class Breakout(bt.Strategy):
    params = (('lookback', 20),)

    def __init__(self):
        self.highest = bt.indicators.Highest(self.data.high, period=self.p.lookback)
        self.lowest = bt.indicators.Lowest(self.data.low, period=self.p.lookback)

    def next(self):
        if self.data.close[0] > self.highest[-1] and not self.position:
            self.buy()
        elif self.data.close[0] < self.lowest[-1] and self.position:
            self.sell()
```

## Risk Management

### Position Sizing
```python
class RiskManagedStrategy(bt.Strategy):
    params = (('risk_percent', 0.02),)  # 2% risk per trade

    def calculate_position_size(self, entry_price, stop_price):
        portfolio_value = self.broker.getvalue()
        risk_amount = portfolio_value * self.p.risk_percent
        risk_per_share = abs(entry_price - stop_price)
        return risk_amount / risk_per_share if risk_per_share > 0 else 0

    def next(self):
        if self.should_buy():
            entry_price = self.data.close[0]
            stop_price = entry_price * 0.98  # 2% stop loss
            size = self.calculate_position_size(entry_price, stop_price)

            self.buy_bracket(
                size=size,
                price=entry_price,
                stopprice=stop_price,
                limitprice=entry_price * 1.06  # 3:1 risk/reward
            )
```

### OCO Orders
```python
def next(self):
    if self.signal_to_buy():
        entry_price = self.data.close[0]

        # OCO order with stop loss and take profit
        self.buy_bracket(
            size=0.001,
            price=entry_price,
            stopprice=entry_price * 0.95,  # 5% stop loss
            limitprice=entry_price * 1.10   # 10% take profit
        )
```

## Multi-Timeframe Strategies

```python
class MultiTimeframe(bt.Strategy):
    def __init__(self):
        # Assuming data0 is 1h, data1 is 4h
        self.short_ma = bt.indicators.SMA(self.data0, period=20)  # 1h MA
        self.long_ma = bt.indicators.SMA(self.data1, period=20)   # 4h MA

    def next(self):
        # Trade on 1h timeframe, filter with 4h trend
        if (self.data0.close[0] > self.short_ma[0] and
            self.data1.close[0] > self.long_ma[0]):
            self.buy()
```

## Multi-Asset Strategies

```python
class PairTrading(bt.Strategy):
    def __init__(self):
        # Assuming data0 is BTC, data1 is ETH
        self.btc_sma = bt.indicators.SMA(self.data0, period=20)
        self.eth_sma = bt.indicators.SMA(self.data1, period=20)

        # Calculate ratio
        self.ratio = self.data0.close / self.data1.close
        self.ratio_sma = bt.indicators.SMA(self.ratio, period=20)

    def next(self):
        current_ratio = self.data0.close[0] / self.data1.close[0]

        if current_ratio > self.ratio_sma[0] * 1.02:  # BTC overvalued vs ETH
            self.sell(data=self.data0)  # Sell BTC
            self.buy(data=self.data1)   # Buy ETH
        elif current_ratio < self.ratio_sma[0] * 0.98:  # BTC undervalued vs ETH
            self.buy(data=self.data0)   # Buy BTC
            self.sell(data=self.data1)  # Sell ETH
```

## Strategy Analysis

### Built-in Analyzers
```python
cerebro.addanalyzer(bt.analyzers.SharpeRatio, _name='sharpe')
cerebro.addanalyzer(bt.analyzers.DrawDown, _name='drawdown')
cerebro.addanalyzer(bt.analyzers.TradeAnalyzer, _name='trades')
cerebro.addanalyzer(bt.analyzers.Returns, _name='returns')

results = cerebro.run()
strategy = results[0]

print(f"Sharpe Ratio: {strategy.analyzers.sharpe.get_analysis()['sharperatio']:.2f}")
print(f"Max Drawdown: {strategy.analyzers.drawdown.get_analysis()['max']['drawdown']:.2f}%")
```

### Custom Analyzers
```python
class WinRateAnalyzer(bt.Analyzer):
    def __init__(self):
        self.wins = 0
        self.losses = 0

    def notify_trade(self, trade):
        if trade.isclosed:
            if trade.pnl > 0:
                self.wins += 1
            else:
                self.losses += 1

    def get_analysis(self):
        total = self.wins + self.losses
        return {
            'win_rate': self.wins / total if total > 0 else 0,
            'total_trades': total
        }
```

## Strategy Optimization

### Parameter Optimization
```python
cerebro.optstrategy(
    MyStrategy,
    fast_period=range(5, 20, 5),      # Test 5, 10, 15
    slow_period=range(20, 50, 10),    # Test 20, 30, 40
    risk_percent=[0.01, 0.02, 0.03]   # Test different risk levels
)
```

### Walk-Forward Analysis
```python
# Split data into training and testing periods
def walk_forward_analysis():
    training_periods = 180  # 180 days training
    testing_periods = 30    # 30 days testing

    # Run optimization on training data
    # Test on out-of-sample data
    # Repeat for multiple periods
```

## Strategy Monitoring

### Live Strategy Monitoring
```python
class MonitoredStrategy(bt.Strategy):
    def __init__(self):
        self.trade_count = 0
        self.daily_pnl = 0

    def notify_trade(self, trade):
        if trade.isclosed:
            self.trade_count += 1
            self.daily_pnl += trade.pnl

            # Log trade details
            print(f"Trade #{self.trade_count}: PnL={trade.pnl:.2f}")

            # Alert on large losses
            if trade.pnl < -1000:
                self.send_alert(f"Large loss: {trade.pnl:.2f}")

    def send_alert(self, message):
        # Implement your alerting mechanism
        print(f"ALERT: {message}")
```

### Health Checks
```python
def next(self):
    # Check for unusual conditions
    if self.broker.get_value() < self.initial_value * 0.9:  # 10% drawdown
        print("WARNING: Portfolio down 10%")

    # Check for stale data
    if len(self.data) > 0:
        last_update = self.data.datetime.datetime(0)
        if (datetime.now() - last_update).seconds > 300:  # 5 minutes
            print("WARNING: Stale data detected")
```

## Best Practices

### Strategy Development
1. **Start simple**: Basic moving average before complex ML models
2. **Use proper risk management**: Always set stop losses
3. **Test thoroughly**: Backtest on different market conditions
4. **Monitor performance**: Track key metrics in live trading
5. **Keep it maintainable**: Clear, readable code

### Performance Optimization
1. **Enable caching**: For faster backtesting iterations
2. **Use efficient indicators**: Avoid recalculating the same values
3. **Limit lookback**: Don't use entire history if not needed
4. **Profile bottlenecks**: Use performance tools to identify slow code

### Production Deployment
1. **Paper trade first**: Validate with live data
2. **Start small**: Use minimal position sizes initially
3. **Monitor closely**: Check positions and PnL regularly
4. **Have exit plan**: Know when to stop the strategy
5. **Log everything**: Comprehensive logging for debugging
