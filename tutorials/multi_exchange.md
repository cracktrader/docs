# Multi-Exchange Trading

Cracktrader makes it easy to trade across multiple exchanges simultaneously, giving you access to different markets, better liquidity, and arbitrage opportunities.

## Overview

Multi-exchange trading allows you to:

- **Access more markets**: Different exchanges list different coins
- **Better liquidity**: Spread orders across exchanges for better fills
- **Arbitrage opportunities**: Take advantage of price differences
- **Risk distribution**: Don't put all eggs in one basket
- **Geographic advantages**: Access region-specific exchanges

## Basic Multi-Exchange Setup

```python
import cracktrader as ct

# Create session instances for different exchanges
binance_session = ct.exchange(
    'binance',
    mode='paper',
    apiKey='your_binance_key',
    secret='your_binance_secret'
)

coinbase_session = ct.exchange(
    'coinbasepro',
    mode='paper',
    apiKey='your_coinbase_key',
    secret='your_coinbase_secret',
    passphrase='your_passphrase'
)

# Create cerebro
cerebro = ct.Cerebro()

# Add data feeds from both exchanges
btc_binance = binance_session.feed(symbol='BTC/USDT', timeframe='1m')
btc_coinbase = coinbase_session.feed(symbol='BTC-USD', timeframe='1m')

cerebro.adddata(btc_binance, name='BTC_BINANCE')
cerebro.adddata(btc_coinbase, name='BTC_COINBASE')
```

## Multi-Exchange Strategy

```python
class ArbitrageStrategy(ct.bt.Strategy):
    params = (
        ('threshold', 0.005),  # 0.5% price difference
        ('position_size', 0.1),
    )
    
    def __init__(self):
        # Get data feeds
        self.binance_data = self.datas[0]  # BTC_BINANCE
        self.coinbase_data = self.datas[1]  # BTC_COINBASE
        
        # Price difference indicator
        self.price_diff = (self.coinbase_data.close - self.binance_data.close) / self.binance_data.close
        
    def next(self):
        price_diff = self.price_diff[0]
        
        # Arbitrage opportunity: Coinbase higher than Binance
        if price_diff > self.params.threshold:
            # Buy on Binance, Sell on Coinbase
            if not self.getposition(data=self.binance_data):
                self.buy(data=self.binance_data, size=self.params.position_size)
                
        # Arbitrage opportunity: Binance higher than Coinbase  
        elif price_diff < -self.params.threshold:
            # Buy on Coinbase, Sell on Binance
            if not self.getposition(data=self.coinbase_data):
                self.buy(data=self.coinbase_data, size=self.params.position_size)
                
        # Close positions when spread narrows
        elif abs(price_diff) < self.params.threshold / 2:
            self.close(data=self.binance_data)
            self.close(data=self.coinbase_data)

# Add strategy to cerebro
cerebro.addstrategy(ArbitrageStrategy)
cerebro.run()
```

## Exchange-Specific Configuration

Pass exchange-specific options directly to the `ct.exchange` helper:

### Binance Configuration
```python
binance_session = ct.exchange(
    'binance',
    mode='paper',
    instrument_type='future',
    store_kwargs={
        'config': {
            'options': {'adjustForTimeDifference': True},
            'rateLimit': 1200
        }
    }
)
```

### Coinbase Pro Configuration
```python
coinbase_session = ct.exchange(
    'coinbasepro',
    mode='paper',
    store_kwargs={
        'config': {
            'urls': {'api': 'https://api-public.sandbox.pro.coinbase.com'}
        }
    }
)
```

### Kraken Configuration
```python
kraken_session = ct.exchange(
    'kraken',
    mode='paper',
    instrument_type='margin',
    store_kwargs={'config': {'options': {'leverage': 2}}}
)
```

## Risk Management

Multi-exchange trading requires careful risk management:

```python
class MultiExchangeRiskManager:
    def __init__(self):
        self.max_exposure_per_exchange = 0.3  # 30% max per exchange
        self.total_exposure_limit = 0.8      # 80% total exposure
        self.exchange_positions = {}
        
    def check_position_size(self, exchange, size, price):
        """Check if position size is within limits"""
        current_value = self.exchange_positions.get(exchange, 0)
        new_value = current_value + (size * price)
        
        # Check per-exchange limit
        if new_value > self.max_exposure_per_exchange:
            return False
            
        # Check total exposure
        total_exposure = sum(self.exchange_positions.values()) + (size * price)
        if total_exposure > self.total_exposure_limit:
            return False
            
        return True
        
    def update_position(self, exchange, size, price):
        """Update position tracking"""
        if exchange not in self.exchange_positions:
            self.exchange_positions[exchange] = 0
        self.exchange_positions[exchange] += size * price
```

## Best Practices

### 1. Test in Sandbox First
Always test multi-exchange strategies in sandbox/testnet mode by setting `mode='paper'` or `mode='sandbox'`.

### 2. Handle Different Symbol Formats
Different exchanges use different symbol formats:

```python
symbol_mapping = {
    'binance': 'BTC/USDT',
    'coinbasepro': 'BTC-USD', 
    'kraken': 'BTC/USD',
}

def get_symbol_for_exchange(exchange, base, quote):
    """Get properly formatted symbol for exchange"""
    if exchange == 'binance':
        return f"{base}/{quote}"
    elif exchange == 'coinbasepro':
        return f"{base}-{quote}"
    elif exchange == 'kraken':
        return f"{base}/{quote}"
```

### 3. Monitor Exchange Status
Check exchange status before placing orders:

```python
def check_exchange_status(session):
    """Check if exchange is operational"""
    try:
        status = session.store.exchange.fetch_status()
        return status['status'] == 'ok'
    except Exception as e:
        ct.get_logger().error(f"Exchange status check failed: {e}")
        return False
```

### 4. Handle Rate Limits
Rate limits are handled automatically by the underlying store, but you can override them if needed.

## Common Patterns

### Cross-Exchange Price Monitoring
```python
class PriceMonitor(ct.bt.Strategy):
    def __init__(self):
        self.exchanges = {}
        for i, data in enumerate(self.datas):
            exchange_name = data._name.split('_')[1]  # Extract from name
            self.exchanges[exchange_name] = data
            
    def next(self):
        prices = {}
        for exchange, data in self.exchanges.items():
            prices[exchange] = data.close[0]
            
        # Log price differences
        max_price = max(prices.values())
        min_price = min(prices.values())
        spread = (max_price - min_price) / min_price
        
        if spread > 0.01:  # 1% spread
            self.log(f"Significant spread detected: {spread:.2%}")
```

### Portfolio Rebalancing Across Exchanges
```python
class RebalanceStrategy(ct.bt.Strategy):
    params = (
        ('rebalance_frequency', 24),  # Hours
        ('target_allocation', {'binance': 0.4, 'coinbase': 0.3, 'kraken': 0.3}),
    )
    
    def __init__(self):
        self.rebalance_timer = 0
        
    def next(self):
        self.rebalance_timer += 1
        
        if self.rebalance_timer >= self.params.rebalance_frequency:
            self.rebalance_portfolio()
            self.rebalance_timer = 0
            
    def rebalance_portfolio(self):
        total_value = self.broker.get_value()
        
        for exchange, target_pct in self.params.target_allocation.items():
            current_value = self.get_exchange_value(exchange)
            target_value = total_value * target_pct
            diff = target_value - current_value
            
            if abs(diff) > total_value * 0.05:  # 5% threshold
                # Rebalance if difference is significant
                self.adjust_position(exchange, diff)
```

## See Also

- [Multi-Asset Strategies](multi_asset.md) - Trading multiple cryptocurrencies
- [Risk Management](risk_management.md) - Advanced risk controls
- [Testing & Quality](../testing/tested_exchanges.md) - Supported exchanges
