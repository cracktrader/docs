# Trading Different Instruments

Cracktrader supports various cryptocurrency trading instruments including spot, margin, futures, and options across different exchanges.

## Overview

Different instruments offer unique opportunities:

- **Spot Trading**: Direct ownership of cryptocurrencies
- **Margin Trading**: Leverage your positions up to exchange limits
- **Futures**: Trade contracts with expiration dates and leverage
- **Perpetual Swaps**: Futures without expiration dates
- **Options**: Right to buy/sell at specific prices (limited exchanges)

## Instrument Configuration

Configure different instruments through the store:

```python
import cracktrader as ct

# Spot trading (default)
spot_store = ct.CCXTStore(
    exchange='binance',
    config={
        'apiKey': 'your_api_key',
        'secret': 'your_secret',
        'options': {
            'defaultType': 'spot',
        }
    }
)

# Futures trading
futures_store = ct.CCXTStore(
    exchange='binance',
    config={
        'apiKey': 'your_api_key',
        'secret': 'your_secret',
        'options': {
            'defaultType': 'future',
        }
    }
)

# Margin trading
margin_store = ct.CCXTStore(
    exchange='kraken',
    config={
        'apiKey': 'your_api_key',
        'secret': 'your_secret',
        'options': {
            'leverage': 2,  # 2x leverage
        }
    }
)
```

## Spot Trading

Basic spot trading with direct asset ownership:

```python
class SpotTradingStrategy(ct.bt.Strategy):
    params = (
        ('position_size', 0.1),
        ('stop_loss', 0.05),  # 5%
    )
    
    def __init__(self):
        self.sma_short = ct.indicators.SMA(self.data.close, period=10)
        self.sma_long = ct.indicators.SMA(self.data.close, period=30)
        
    def next(self):
        if not self.position:
            # Golden cross entry
            if self.sma_short[0] > self.sma_long[0] and self.sma_short[-1] <= self.sma_long[-1]:
                self.buy(size=self.params.position_size)
                
        else:
            # Stop loss exit
            if self.data.close[0] < self.position.price * (1 - self.params.stop_loss):
                self.close()

# Create spot data feed
spot_data = ct.CCXTDataFeed(
    store=spot_store,
    symbol='BTC/USDT',
    timeframe='1h'
)

cerebro = ct.Cerebro()
cerebro.adddata(spot_data)
cerebro.addstrategy(SpotTradingStrategy)
cerebro.setbroker(ct.CCXTLiveBroker(store=spot_store))
```

## Futures Trading

Trade futures contracts with leverage and expiration dates:

```python
class FuturesStrategy(ct.bt.Strategy):
    params = (
        ('leverage', 3),
        ('position_size', 0.2),
        ('risk_per_trade', 0.02),  # 2% risk per trade
    )
    
    def __init__(self):
        self.rsi = ct.indicators.RSI(self.data.close, period=14)
        self.atr = ct.indicators.ATR(self.data, period=20)
        
    def next(self):
        if not self.position:
            # RSI oversold entry
            if self.rsi[0] < 30:
                # Calculate position size based on ATR risk
                atr_value = self.atr[0]
                stop_distance = atr_value * 2  # 2 ATR stop
                
                portfolio_value = self.broker.get_value()
                risk_amount = portfolio_value * self.params.risk_per_trade
                
                # Position size = Risk Amount / Stop Distance
                position_size = min(
                    risk_amount / stop_distance,
                    self.params.position_size
                )
                
                self.buy(size=position_size)
                self.stop_price = self.data.close[0] - stop_distance
                
        else:
            # ATR-based stop loss
            if self.data.close[0] <= self.stop_price:
                self.close()

# Futures data feed
futures_data = ct.CCXTDataFeed(
    store=futures_store,
    symbol='BTC/USDT',  # Perpetual futures
    timeframe='15m'
)

cerebro.adddata(futures_data)
cerebro.addstrategy(FuturesStrategy)
cerebro.setbroker(ct.CCXTLiveBroker(store=futures_store))
```

## Margin Trading

Use leverage to amplify positions:

```python
class MarginTradingStrategy(ct.bt.Strategy):
    params = (
        ('leverage', 2.0),
        ('max_drawdown', 0.15),  # 15% max drawdown
        ('position_size', 0.3),
    )
    
    def __init__(self):
        self.bb = ct.indicators.BollingerBands(self.data.close, period=20)
        self.drawdown = 0
        self.peak_value = self.broker.get_value()
        
    def next(self):
        # Update drawdown tracking
        current_value = self.broker.get_value()
        if current_value > self.peak_value:
            self.peak_value = current_value
            
        self.drawdown = (self.peak_value - current_value) / self.peak_value
        
        # Risk management: reduce leverage if drawdown too high
        if self.drawdown > self.params.max_drawdown:
            if self.position:
                self.close()
            return
            
        if not self.position:
            # Bollinger band bounce entry
            if self.data.close[0] <= self.bb.lines.bot[0]:
                self.buy(size=self.params.position_size)
                
            elif self.data.close[0] >= self.bb.lines.top[0]:
                self.sell(size=self.params.position_size)
                
        else:
            # Exit at middle band
            if abs(self.data.close[0] - self.bb.lines.mid[0]) < self.bb.lines.mid[0] * 0.01:
                self.close()

# Margin trading setup
margin_data = ct.CCXTDataFeed(
    store=margin_store,
    symbol='BTC/USD',
    timeframe='1h'
)

cerebro.adddata(margin_data)
cerebro.addstrategy(MarginTradingStrategy)
cerebro.setbroker(ct.CCXTLiveBroker(store=margin_store))
```

## Multi-Instrument Strategy

Trade across different instruments simultaneously:

```python
class MultiInstrumentStrategy(ct.bt.Strategy):
    params = (
        ('spot_allocation', 0.4),
        ('futures_allocation', 0.4),
        ('cash_allocation', 0.2),
    )
    
    def __init__(self):
        # Assume multiple data feeds: spot, futures
        self.spot_data = self.datas[0]
        self.futures_data = self.datas[1]
        
        # Indicators for each instrument
        self.spot_rsi = ct.indicators.RSI(self.spot_data.close, period=14)
        self.futures_rsi = ct.indicators.RSI(self.futures_data.close, period=14)
        
        # Correlation indicator
        self.correlation = ct.indicators.Correlation(
            self.spot_data.close,
            self.futures_data.close,
            period=30
        )
        
    def next(self):
        portfolio_value = self.broker.get_value()
        
        # Spot trading logic
        spot_position = self.getposition(self.spot_data)
        target_spot_value = portfolio_value * self.params.spot_allocation
        
        if self.spot_rsi[0] < 30 and not spot_position:
            spot_size = target_spot_value / self.spot_data.close[0]
            self.buy(data=self.spot_data, size=spot_size)
            
        elif self.spot_rsi[0] > 70 and spot_position:
            self.close(data=self.spot_data)
            
        # Futures trading logic
        futures_position = self.getposition(self.futures_data)
        target_futures_value = portfolio_value * self.params.futures_allocation
        
        # Use futures for hedging when correlation is high
        if self.correlation[0] > 0.8 and spot_position and not futures_position:
            # Hedge spot position with short futures
            hedge_size = spot_position.size * 0.5
            self.sell(data=self.futures_data, size=hedge_size)
            
        elif self.correlation[0] < 0.3 and futures_position:
            # Remove hedge when correlation breaks down
            self.close(data=self.futures_data)

# Setup multiple instruments
cerebro = ct.Cerebro()

# Add spot data
spot_data = ct.CCXTDataFeed(store=spot_store, symbol='BTC/USDT', timeframe='1h')
cerebro.adddata(spot_data, name='spot')

# Add futures data  
futures_data = ct.CCXTDataFeed(store=futures_store, symbol='BTC/USDT', timeframe='1h')
cerebro.adddata(futures_data, name='futures')

cerebro.addstrategy(MultiInstrumentStrategy)
```

## Exchange-Specific Features

### Binance Futures
```python
binance_futures_config = {
    'options': {
        'defaultType': 'future',
        'hedgeMode': False,  # One-way or hedge mode
        'positionSide': 'BOTH',  # BOTH, LONG, SHORT
    }
}
```

### FTX Futures (if available)
```python
ftx_futures_config = {
    'options': {
        'subAccount': 'futures_trading',  # Use subaccount
    }
}
```

### Kraken Margin
```python
kraken_margin_config = {
    'options': {
        'leverage': 2,
        'margin': True,
    }
}
```

## Risk Management by Instrument

Different instruments require different risk approaches:

```python
class InstrumentRiskManager:
    def __init__(self):
        self.max_exposure = {
            'spot': 0.6,      # 60% in spot
            'futures': 0.3,   # 30% in futures  
            'margin': 0.2,    # 20% in margin
        }
        
        self.max_leverage = {
            'spot': 1.0,
            'futures': 5.0,
            'margin': 3.0,
        }
        
    def check_position_limits(self, instrument_type, position_size, price):
        """Check if position is within limits for instrument type"""
        max_exposure = self.max_exposure.get(instrument_type, 0.1)
        portfolio_value = self.get_portfolio_value()
        
        position_value = position_size * price
        exposure_ratio = position_value / portfolio_value
        
        return exposure_ratio <= max_exposure
        
    def get_max_position_size(self, instrument_type, price):
        """Get maximum allowed position size"""
        max_exposure = self.max_exposure.get(instrument_type, 0.1)
        portfolio_value = self.get_portfolio_value()
        
        max_value = portfolio_value * max_exposure
        return max_value / price
```

## Best Practices

### 1. Understand Instrument Specifics
- **Expiration dates** for futures
- **Funding rates** for perpetual swaps
- **Margin requirements** for leveraged positions
- **Settlement procedures** for different contracts

### 2. Risk Management
- Use appropriate position sizing for each instrument
- Monitor margin levels closely
- Set stop losses based on instrument volatility
- Diversify across instruments to reduce risk

### 3. Cost Considerations
- **Futures**: Funding costs/rebates
- **Margin**: Interest on borrowed funds
- **Spot**: No carrying costs
- **Options**: Time decay

### 4. Liquidity Awareness
- Check order book depth before trading
- Use appropriate order types for each instrument
- Monitor bid-ask spreads

## See Also

- [Using Leverage](leverage.md) - Advanced leverage techniques
- [Risk Management](risk_management.md) - Comprehensive risk controls
- [Multi-Exchange Trading](multi_exchange.md) - Trading across exchanges