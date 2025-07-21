# Backtrader Compatibility Guide

Cracktrader is built on top of Backtrader, providing full compatibility with existing Backtrader strategies while adding cryptocurrency trading capabilities. This guide covers supported features, differences, and migration patterns.

## Overview

### What's Supported ✅

**Core Strategy Features:**
- All `bt.Strategy` methods (`next()`, `__init__()`, `notify_*()`)
- Parameter system (`params` tuple)
- Position management (`buy()`, `sell()`, `close()`)
- Order management (market, limit, stop orders)
- Data access patterns (`self.data.close[0]`, `self.data.high[-1]`)

**Indicators (100+ Built-in):**
- All standard Backtrader indicators
- Technical Analysis indicators
- Custom indicator development
- Indicator chaining and combinations

**Analyzers:**
- Performance analyzers (Sharpe, Returns, Drawdown)
- Trade analyzers (win rate, profit factor)
- Custom analyzer development
- Portfolio analysis

**Data Handling:**
- Multiple timeframes
- Data resampling and replaying
- Historical and live data
- Custom data feeds

### What's Enhanced 🚀

**Exchange Integration:**
- Real-time data from 400+ exchanges via CCXT
- Live order execution on real exchanges
- Unified paper/live trading modes
- Advanced order types (OCO, brackets)

**Modern Infrastructure:**
- Async/await support for real-time operations
- Comprehensive logging and monitoring
- Web API for remote control
- Health monitoring and alerting

### What's Different ⚠️

**Data Feeds:**
- CCXT-based feeds instead of file-based
- Real-time streaming capabilities
- Cryptocurrency-specific features (funding rates, etc.)

**Broker Behavior:**
- Live execution involves actual exchange APIs
- Commission models reflect real exchange fees
- Slippage models based on orderbook depth

## Migration Guide

### Basic Strategy Migration

**Before (Pure Backtrader):**
```python
import backtrader as bt

class MyStrategy(bt.Strategy):
    def __init__(self):
        self.sma = bt.indicators.SMA(period=20)
    
    def next(self):
        if self.data.close[0] > self.sma[0]:
            self.buy()

# Setup
cerebro = bt.Cerebro()
cerebro.addstrategy(MyStrategy)

# Add CSV data
data = bt.feeds.YahooFinanceData(dataname='AAPL')
cerebro.adddata(data)

cerebro.run()
```

**After (Cracktrader):**
```python
import backtrader as bt
from cracktrader import Cerebro, CCXTStore, CCXTDataFeed

class MyStrategy(bt.Strategy):  # No changes needed!
    def __init__(self):
        self.sma = bt.indicators.SMA(period=20)
    
    def next(self):
        if self.data.close[0] > self.sma[0]:
            self.buy()

# Enhanced setup
cerebro = Cerebro()  # Use Cracktrader's Cerebro
cerebro.addstrategy(MyStrategy)

# Add crypto data
store = CCXTStore(exchange_id='binance')
feed = CCXTDataFeed(
    store=store,
    symbol='BTC/USDT',
    timeframe='1h',
    start_date='2024-01-01',
    end_date='2024-03-01'
)
cerebro.adddata(feed)

cerebro.run()
```

### Data Feed Migration

**CSV/Yahoo Finance → CCXT:**

```python
# Old way
data = bt.feeds.YahooFinanceData(
    dataname='AAPL',
    fromdate=datetime(2023, 1, 1),
    todate=datetime(2023, 12, 31)
)

# New way
store = CCXTStore(exchange_id='binance')
data = CCXTDataFeed(
    store=store,
    symbol='BTC/USDT',
    timeframe='1d',
    start_date='2023-01-01T00:00:00Z',
    end_date='2023-12-31T23:59:59Z'
)
```

**Custom Data Feeds:**

```python
# Backtrader custom feed
class MyCSVData(bt.feeds.GenericCSVData):
    params = (
        ('datetime', 0),
        ('open', 1),
        ('high', 2),
        ('low', 3),
        ('close', 4),
        ('volume', 5),
    )

# Cracktrader equivalent - wrap with CCXTDataFeed
from cracktrader.feeds import CustomDataFeed

class MyCryptoFeed(CustomDataFeed):
    def _load_data(self):
        # Load data from your source
        return df  # Return pandas DataFrame with OHLCV columns
```

### Broker Migration

**Backtrader Broker:**
```python
cerebro.broker.setcash(10000)
cerebro.broker.setcommission(commission=0.001)
```

**Cracktrader Broker (Backtest):**
```python
# Same API - no changes needed
cerebro.broker.setcash(10000)
cerebro.broker.setcommission(commission=0.001)

# Or use enhanced broker
from cracktrader.broker import CTBroker
broker = CTBroker.create(mode="back", cash=10000)
cerebro.broker = broker
```

**Cracktrader Broker (Live):**
```python
from cracktrader.broker import CCXTLiveBroker

# Paper trading
broker = CCXTLiveBroker(store=store, mode='paper')
cerebro.broker = broker

# Live trading
broker = CCXTLiveBroker(store=store, mode='live')
cerebro.broker = broker
```

## Feature Compatibility Matrix

### Core Strategy Methods

| Method | Backtrader | Cracktrader | Notes |
|--------|------------|-------------|-------|
| `__init__()` | ✅ | ✅ | Fully compatible |
| `next()` | ✅ | ✅ | Fully compatible |
| `prenext()` | ✅ | ✅ | Fully compatible |
| `nextstart()` | ✅ | ✅ | Fully compatible |
| `start()` | ✅ | ✅ | Fully compatible |
| `stop()` | ✅ | ✅ | Enhanced with logging |
| `notify_order()` | ✅ | ✅ | Enhanced order states |
| `notify_trade()` | ✅ | ✅ | Enhanced trade info |
| `notify_cashvalue()` | ✅ | ✅ | Real-time updates |

### Order Types

| Order Type | Backtrader | Cracktrader | Notes |
|------------|------------|-------------|-------|
| Market | ✅ | ✅ | Immediate execution |
| Limit | ✅ | ✅ | Exchange limit orders |
| Stop | ✅ | ✅ | Stop-loss orders |
| StopLimit | ✅ | ✅ | Stop-limit combinations |
| Bracket | ✅ | ✅ | Enhanced with OCO |
| OCO | ❌ | ✅ | Cracktrader exclusive |
| Trailing Stop | ❌ | ✅ | Cracktrader exclusive |

### Indicators

**All Backtrader indicators are supported:**

```python
# Standard indicators work unchanged
self.sma = bt.indicators.SimpleMovingAverage(period=20)
self.rsi = bt.indicators.RSI(period=14)
self.macd = bt.indicators.MACD()
self.bb = bt.indicators.BollingerBands()

# Plotting works the same way
self.sma.plotinfo.plot = True
self.sma.plotinfo.plotname = 'SMA(20)'
```

**Enhanced Cryptocurrency Indicators:**

```python
from cracktrader.indicators import FundingRate, PerpetualPremium

# Crypto-specific indicators
self.funding = FundingRate(period=8)  # 8-hour funding
self.premium = PerpetualPremium(spot_data=self.data0, perp_data=self.data1)
```

### Analyzers

**Standard Analyzers:**

| Analyzer | Backtrader | Cracktrader | Enhancement |
|----------|------------|-------------|-------------|
| Returns | ✅ | ✅ | Real-time calculation |
| SharpeRatio | ✅ | ✅ | Multiple benchmarks |
| DrawDown | ✅ | ✅ | Live monitoring |
| TradeAnalyzer | ✅ | ✅ | Enhanced metrics |
| PositionsValue | ✅ | ✅ | Multi-asset support |

**Enhanced Analyzers:**

```python
from cracktrader.analyzers import CryptoAnalyzer, RiskAnalyzer

# Crypto-specific analysis
cerebro.addanalyzer(CryptoAnalyzer, _name='crypto')
cerebro.addanalyzer(RiskAnalyzer, _name='risk')
```

## Advanced Features

### Multi-Exchange Support

```python
# Connect to multiple exchanges
binance_store = CCXTStore(exchange_id='binance')
coinbase_store = CCXTStore(exchange_id='coinbase')

# Compare prices across exchanges
binance_feed = CCXTDataFeed(binance_store, 'BTC/USDT')
coinbase_feed = CCXTDataFeed(coinbase_store, 'BTC/USD')

cerebro.adddata(binance_feed, name='binance')
cerebro.adddata(coinbase_feed, name='coinbase')

class ArbitrageStrategy(bt.Strategy):
    def next(self):
        binance_price = self.datas[0].close[0]
        coinbase_price = self.datas[1].close[0]
        
        spread = (coinbase_price - binance_price) / binance_price
        if spread > 0.01:  # 1% arbitrage opportunity
            self.buy(data=self.datas[0])  # Buy on Binance
            self.sell(data=self.datas[1])  # Sell on Coinbase
```

### Real-time Data Streaming

```python
class LiveStrategy(bt.Strategy):
    def next(self):
        # This runs in real-time with live market data
        current_time = self.data.datetime.datetime()
        current_price = self.data.close[0]
        
        # Real-time decision making
        if self.should_trade():
            self.buy()

# Setup for live trading
store = CCXTStore(exchange_id='binance', sandbox=True)
feed = CCXTDataFeed(store=store, symbol='BTC/USDT', timeframe='1m')
# No start/end dates = live streaming

cerebro = Cerebro()
cerebro.adddata(feed)
cerebro.addstrategy(LiveStrategy)
cerebro.run()  # Runs until stopped
```

### Enhanced Order Management

```python
class AdvancedOrderStrategy(bt.Strategy):
    def next(self):
        if not self.position:
            # Place bracket order (entry + stop + target)
            main_order = self.buy()
            
            # Automatic stop loss and take profit
            stop_price = self.data.close[0] * 0.95  # 5% stop
            target_price = self.data.close[0] * 1.10  # 10% target
            
            self.sell(exectype=bt.Order.Stop, price=stop_price, parent=main_order)
            self.sell(exectype=bt.Order.Limit, price=target_price, parent=main_order)

# OCO (One-Cancels-Other) orders
class OCOStrategy(bt.Strategy):
    def next(self):
        if not self.position:
            # Entry order
            entry_order = self.buy_bracket(
                price=self.data.close[0] * 1.01,  # Entry above current
                stopprice=self.data.close[0] * 0.95,  # Stop loss
                limitprice=self.data.close[0] * 1.15   # Take profit
            )
```

## Performance Considerations

### Backtrader vs Cracktrader Performance

**Backtrader (File-based):**
- Fast for historical backtests
- Limited by disk I/O for large datasets
- No real-time capabilities

**Cracktrader (Network-based):**
- Slightly slower for pure backtests (network overhead)
- Excellent for real-time operations
- Scalable with caching and optimization

### Optimization Tips

```python
# 1. Use appropriate timeframes
# High-frequency: Use lower timeframes sparingly
feed = CCXTDataFeed(store=store, symbol='BTC/USDT', timeframe='5m')  # Not '1s'

# 2. Limit historical data range
feed = CCXTDataFeed(
    store=store,
    symbol='BTC/USDT',
    timeframe='1h',
    start_date='2024-01-01',  # Don't go back years unless needed
    end_date='2024-03-01'
)

# 3. Use data caching
store = CCXTStore(exchange_id='binance', cache_enabled=True)

# 4. Optimize indicator calculations
class OptimizedStrategy(bt.Strategy):
    def __init__(self):
        # Calculate once, use many times
        self.sma = bt.indicators.SMA(period=20)
        self.signal = self.data.close > self.sma  # Boolean series
    
    def next(self):
        if self.signal[0]:  # Direct boolean check
            self.buy()
```

## Testing Compatibility

### Running Backtrader Tests

```python
# Most Backtrader test patterns work unchanged
import unittest
from cracktrader import Cerebro  # Drop-in replacement

class TestMyStrategy(unittest.TestCase):
    def setUp(self):
        self.cerebro = Cerebro()
        self.cerebro.addstrategy(MyStrategy)
        
        # Use test data
        from tests.fixtures.fake_exchange import FakeExchange
        store = CCXTStore(exchange_id='fake', exchange=FakeExchange())
        feed = CCXTDataFeed(store=store, symbol='BTC/USDT')
        self.cerebro.adddata(feed)
    
    def test_strategy_runs(self):
        results = self.cerebro.run()
        self.assertIsNotNone(results)
```

### Mocking for Tests

```python
from unittest.mock import Mock, patch

class TestCryptoStrategy(unittest.TestCase):
    @patch('cracktrader.store.CCXTStore')
    def test_with_mocked_exchange(self, mock_store):
        # Mock exchange behavior
        mock_store.return_value.fetch_ohlcv.return_value = [
            [1640995200000, 50000, 51000, 49000, 50500, 100],  # OHLCV data
        ]
        
        cerebro = Cerebro()
        cerebro.addstrategy(MyStrategy)
        # Test continues...
```

## Migration Checklist

### Pre-Migration Assessment

- [ ] **Inventory existing strategies** - List all Backtrader strategies
- [ ] **Identify data sources** - CSV files, Yahoo Finance, custom feeds
- [ ] **Review custom indicators** - Any custom-built indicators
- [ ] **Check broker configurations** - Commission, slippage settings
- [ ] **List analyzers used** - Performance analysis requirements

### Migration Steps

1. **Install Cracktrader**
   ```bash
   pip install -e .'[web]'
   ```

2. **Update imports**
   ```python
   # Add to existing imports
   from cracktrader import Cerebro, CCXTStore, CCXTDataFeed
   ```

3. **Replace Cerebro**
   ```python
   # cerebro = bt.Cerebro()
   cerebro = Cerebro()
   ```

4. **Replace data feeds**
   ```python
   # Replace file-based feeds with CCXT feeds
   store = CCXTStore(exchange_id='binance')
   feed = CCXTDataFeed(store=store, symbol='BTC/USDT', timeframe='1h')
   cerebro.adddata(feed)
   ```

5. **Test in sandbox mode**
   ```python
   store = CCXTStore(exchange_id='binance', sandbox=True)
   ```

6. **Validate results**
   - Compare backtest results with original
   - Verify indicator calculations
   - Check performance metrics

7. **Enable live trading** (when ready)
   ```python
   store = CCXTStore(exchange_id='binance', sandbox=False)
   ```

### Post-Migration Validation

```python
def validate_migration():
    """Validate that migrated strategy produces expected results."""
    
    # Run original Backtrader strategy
    bt_cerebro = bt.Cerebro()
    bt_cerebro.addstrategy(OriginalStrategy)
    # Add equivalent data...
    bt_results = bt_cerebro.run()
    
    # Run migrated Cracktrader strategy
    ct_cerebro = Cerebro()
    ct_cerebro.addstrategy(MigratedStrategy)
    # Add CCXT data...
    ct_results = ct_cerebro.run()
    
    # Compare results
    bt_final_value = bt_results[0].broker.getvalue()
    ct_final_value = ct_results[0].broker.getvalue()
    
    difference = abs(bt_final_value - ct_final_value) / bt_final_value
    assert difference < 0.01, f"Results differ by {difference:.2%}"
    
    print("✅ Migration validated successfully")
```

## Troubleshooting Common Issues

### Import Errors

```python
# Issue: ModuleNotFoundError
# Solution: Check installation
pip list | grep backtrader
pip install -e .'[web]'

# Issue: Conflicting Backtrader versions
# Solution: Use virtual environment
python -m venv cracktrader_env
source cracktrader_env/bin/activate  # Linux/Mac
# cracktrader_env\Scripts\activate  # Windows
pip install -e .'[web]'
```

### Data Format Issues

```python
# Issue: Date format differences
# Backtrader expects datetime objects
# CCXT returns timestamps

# Solution: Automatic conversion in CCXTDataFeed
feed = CCXTDataFeed(
    store=store,
    symbol='BTC/USDT',
    timeframe='1h',
    start_date='2024-01-01T00:00:00Z',  # ISO format
    end_date='2024-03-01T23:59:59Z'
)
```

### Performance Differences

```python
# Issue: Slower backtests compared to file-based
# Solution: Use caching and optimize data fetching

store = CCXTStore(
    exchange_id='binance',
    cache_enabled=True,
    cache_ttl=3600  # 1 hour cache
)

# Or use local data storage
store = CCXTStore(
    exchange_id='binance',
    local_storage=True,
    storage_path='./data_cache'
)
```

---

**Migration Support:**
- **Issues**: Report migration problems on GitHub
- **Examples**: See `examples/migration/` for complete migration examples
- **Community**: Join discussions for migration tips and best practices

**Next Steps:**
- [Data Feeds Guide](feeds.md) - CCXT data integration
- [Strategy Development](strategy_guide.md) - Advanced strategy patterns
- [Testing Guide](testing.md) - Testing migrated strategies