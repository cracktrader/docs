# Brokers

CrackTrader provides three broker types for different trading scenarios.

## Broker Types

### CCXTBackBroker
For backtesting with historical data.

```python
from cracktrader.broker import CCXTBackBroker

broker = CCXTBackBroker(store=store, startingcash=10000.0)
```

**Features**:
- Uses historical data only
- Simulated order execution
- No real money involved
- Fast backtesting

### CCXTLiveBroker
For live trading with real money.

```python
from cracktrader.broker import CCXTLiveBroker

broker = CCXTLiveBroker(store=store)
```

**Features**:
- Real exchange connectivity
- Actual order placement
- Real money execution
- Live position tracking

### CCXTPaperBroker
For paper trading with live data.

```python
from cracktrader.broker import CCXTPaperBroker

broker = CCXTPaperBroker(store=store, startingcash=10000.0)
```

**Features**:
- Live market data
- Simulated execution at real prices
- No real money involved
- Good for strategy validation

## Order Types

All brokers support standard Backtrader order types:

### Market Orders
```python
self.buy()  # Market buy
self.sell()  # Market sell
```

### Limit Orders
```python
self.buy(exectype=bt.Order.Limit, price=45000.0)
self.sell(exectype=bt.Order.Limit, price=46000.0)
```

### Stop Orders
```python
self.buy(exectype=bt.Order.Stop, price=45000.0)
self.sell(exectype=bt.Order.Stop, price=44000.0)
```

### OCO (One-Cancels-Other) Orders
```python
# Bracket order with stop loss and take profit
self.buy_bracket(
    size=0.001,
    price=45000.0,           # Entry price
    stopprice=44000.0,       # Stop loss
    limitprice=46000.0       # Take profit
)
```

## Commission Info

Each exchange has specific commission structures:

```python
from cracktrader.comm_info import BinanceCommissionInfo

# Binance spot trading
comm_info = BinanceCommissionInfo()

# Custom commission rates
comm_info = BinanceCommissionInfo(
    commission=0.001,  # 0.1%
    margin=None,       # Spot trading
    mult=1.0
)
```

## Position Tracking

Brokers automatically track positions:

```python
def next(self):
    # Check current position
    if self.position.size > 0:
        print(f"Long position: {self.position.size}")
    elif self.position.size < 0:
        print(f"Short position: {self.position.size}")
    else:
        print("No position")

    # Position value
    print(f"Position value: {self.position.size * self.data.close[0]}")
```

## Cash Management

```python
def next(self):
    # Available cash
    cash = self.broker.get_cash()

    # Total portfolio value
    value = self.broker.get_value()

    # Calculate position size based on risk
    risk_amount = value * 0.02  # 2% risk
    stop_distance = self.data.close[0] * 0.02  # 2% stop
    position_size = risk_amount / stop_distance
```

## Error Handling

Brokers handle common trading errors:

```python
def notify_order(self, order):
    if order.status == order.Rejected:
        print(f"Order rejected: {order.info}")
    elif order.status == order.Margin:
        print("Insufficient margin")
    elif order.status == order.Cancelled:
        print("Order cancelled")
```

## Broker Configuration

### Backtest Broker
```python
broker = CCXTBackBroker(
    store=store,
    startingcash=10000.0,
    commission=0.001  # 0.1% commission
)
```

### Live Broker
```python
broker = CCXTLiveBroker(
    store=store,
    # Cash managed by exchange account
)
```

### Paper Broker
```python
broker = CCXTPaperBroker(
    store=store,
    startingcash=10000.0,
    slippage=0.0005  # 0.05% slippage simulation
)
```

## Best Practices

1. **Start with backtesting**: Test strategies with CCXTBackBroker
2. **Validate with paper trading**: Use CCXTPaperBroker with live data
3. **Go live gradually**: Start with small positions on CCXTLiveBroker
4. **Monitor positions**: Always check broker.get_value() and positions
5. **Handle errors**: Implement notify_order() and notify_trade() methods

## Common Patterns

### Strategy Switching
```python
class MyStrategy(bt.Strategy):
    def __init__(self):
        # Detect broker type
        if isinstance(self.broker, CCXTLiveBroker):
            self.is_live = True
            self.position_size_factor = 0.1  # Smaller positions for live
        else:
            self.is_live = False
            self.position_size_factor = 1.0  # Full size for backtesting
```

### Risk Management
```python
def next(self):
    # Check available margin before trading
    if self.broker.get_cash() < self.data.close[0] * 0.001:  # Min position size
        return  # Skip this trade

    # Position sizing based on portfolio value
    portfolio_value = self.broker.get_value()
    max_position_value = portfolio_value * 0.1  # Max 10% per position
    max_size = max_position_value / self.data.close[0]
```
