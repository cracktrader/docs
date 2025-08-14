# Using Leverage

Learn how to safely and effectively use leverage in your cryptocurrency trading strategies with Cracktrader.

## Overview

Leverage allows you to control larger positions with smaller amounts of capital, amplifying both potential profits and losses. Cracktrader provides comprehensive leverage support across multiple exchanges and instruments.

**Key Benefits:**
- Amplified returns on successful trades
- Capital efficiency - trade larger positions with less capital
- Ability to hedge positions
- Access to short selling

**Key Risks:**
- Amplified losses 
- Margin calls and liquidation
- Interest costs on borrowed funds
- Increased volatility exposure

## Basic Leverage Setup

```python
import cracktrader as ct

# Configure leverage for different exchanges
binance_config = {
    'apiKey': 'your_api_key',
    'secret': 'your_secret',
    'options': {
        'defaultType': 'future',  # Enable futures for leverage
        'leverage': 3,            # 3x leverage
    }
}

kraken_config = {
    'apiKey': 'your_api_key', 
    'secret': 'your_secret',
    'options': {
        'leverage': 2,            # 2x leverage
        'margin': True,           # Enable margin trading
    }
}

# Create leveraged stores
binance_store = ct.CCXTStore(exchange='binance', config=binance_config)
kraken_store = ct.CCXTStore(exchange='kraken', config=kraken_config)
```

## Conservative Leverage Strategy

Start with lower leverage and strict risk management:

```python
class ConservativeLeverageStrategy(ct.bt.Strategy):
    params = (
        ('leverage', 2.0),              # 2x leverage
        ('risk_per_trade', 0.01),       # 1% risk per trade
        ('max_positions', 3),           # Max concurrent positions
        ('stop_loss_pct', 0.03),        # 3% stop loss
        ('take_profit_pct', 0.06),      # 6% take profit (2:1 R/R)
    )
    
    def __init__(self):
        # Technical indicators
        self.sma_fast = ct.indicators.SMA(self.data.close, period=10)
        self.sma_slow = ct.indicators.SMA(self.data.close, period=30)
        self.rsi = ct.indicators.RSI(self.data.close, period=14)
        self.atr = ct.indicators.ATR(self.data, period=20)
        
        # Risk management
        self.open_positions = 0
        self.entry_price = None
        
    def next(self):
        current_price = self.data.close[0]
        
        # Only trade if we haven't reached max positions
        if self.open_positions >= self.params.max_positions:
            return
            
        if not self.position:
            # Entry conditions
            trend_up = self.sma_fast[0] > self.sma_slow[0]
            rsi_oversold = self.rsi[0] < 35
            rsi_overbought = self.rsi[0] > 65
            
            if trend_up and rsi_oversold:
                # Calculate position size based on risk
                portfolio_value = self.broker.get_value()
                risk_amount = portfolio_value * self.params.risk_per_trade
                stop_distance = current_price * self.params.stop_loss_pct
                
                # Position size = Risk Amount / Stop Distance / Leverage
                base_size = risk_amount / stop_distance
                leveraged_size = base_size / self.params.leverage  # Adjust for leverage
                
                self.buy(size=leveraged_size)
                self.entry_price = current_price
                self.open_positions += 1
                
                self.log(f"Long entry at {current_price:.2f}, size: {leveraged_size:.4f}")
                
            elif not trend_up and rsi_overbought:
                # Short entry (if supported by exchange)
                portfolio_value = self.broker.get_value()
                risk_amount = portfolio_value * self.params.risk_per_trade
                stop_distance = current_price * self.params.stop_loss_pct
                
                base_size = risk_amount / stop_distance
                leveraged_size = base_size / self.params.leverage
                
                self.sell(size=leveraged_size)
                self.entry_price = current_price
                self.open_positions += 1
                
                self.log(f"Short entry at {current_price:.2f}, size: {leveraged_size:.4f}")
                
        else:
            # Exit conditions
            if self.position.size > 0:  # Long position
                stop_price = self.entry_price * (1 - self.params.stop_loss_pct)
                take_profit_price = self.entry_price * (1 + self.params.take_profit_pct)
                
                if current_price <= stop_price:
                    self.close()
                    self.log(f"Stop loss hit at {current_price:.2f}")
                    self.open_positions -= 1
                    
                elif current_price >= take_profit_price:
                    self.close()
                    self.log(f"Take profit hit at {current_price:.2f}")
                    self.open_positions -= 1
                    
            elif self.position.size < 0:  # Short position
                stop_price = self.entry_price * (1 + self.params.stop_loss_pct)
                take_profit_price = self.entry_price * (1 - self.params.take_profit_pct)
                
                if current_price >= stop_price:
                    self.close()
                    self.log(f"Stop loss hit at {current_price:.2f}")
                    self.open_positions -= 1
                    
                elif current_price <= take_profit_price:
                    self.close()
                    self.log(f"Take profit hit at {current_price:.2f}")
                    self.open_positions -= 1

# Setup strategy
cerebro = ct.Cerebro()
data = ct.CCXTDataFeed(store=binance_store, symbol='BTC/USDT', timeframe='15m')
cerebro.adddata(data)
cerebro.addstrategy(ConservativeLeverageStrategy)
cerebro.setbroker(ct.CCXTLiveBroker(store=binance_store))
```

## Advanced Leverage Management

Dynamic leverage adjustment based on market conditions:

```python
class AdaptiveLeverageStrategy(ct.bt.Strategy):
    params = (
        ('base_leverage', 2.0),
        ('max_leverage', 5.0),
        ('volatility_threshold', 0.02),  # 2% daily volatility
        ('risk_per_trade', 0.015),
    )
    
    def __init__(self):
        # Market condition indicators
        self.volatility = ct.indicators.StdDev(
            self.data.close.pct_change(), 
            period=20
        ) * (252 ** 0.5)  # Annualized volatility
        
        self.trend_strength = ct.indicators.ADX(self.data, period=14)
        self.market_regime = ct.indicators.SMA(self.data.close, period=50)
        
        # Risk metrics
        self.drawdown = 0
        self.peak_value = 0
        
    def calculate_dynamic_leverage(self):
        """Adjust leverage based on market conditions"""
        current_vol = self.volatility[0]
        trend_strength = self.trend_strength[0]
        
        # Base leverage adjustment
        if current_vol > self.params.volatility_threshold:
            # Reduce leverage in high volatility
            vol_adjustment = max(0.5, 1 - (current_vol - self.params.volatility_threshold))
        else:
            vol_adjustment = 1.0
            
        # Trend strength adjustment
        if trend_strength > 25:  # Strong trend
            trend_adjustment = 1.2
        elif trend_strength < 15:  # Weak trend
            trend_adjustment = 0.8
        else:
            trend_adjustment = 1.0
            
        # Drawdown adjustment
        if self.drawdown > 0.1:  # 10% drawdown
            dd_adjustment = 0.5
        elif self.drawdown > 0.05:  # 5% drawdown
            dd_adjustment = 0.8
        else:
            dd_adjustment = 1.0
            
        # Calculate final leverage
        dynamic_leverage = (self.params.base_leverage * 
                          vol_adjustment * 
                          trend_adjustment * 
                          dd_adjustment)
        
        return min(dynamic_leverage, self.params.max_leverage)
        
    def next(self):
        # Update drawdown tracking
        current_value = self.broker.get_value()
        if current_value > self.peak_value:
            self.peak_value = current_value
        self.drawdown = (self.peak_value - current_value) / self.peak_value
        
        # Calculate dynamic leverage
        leverage = self.calculate_dynamic_leverage()
        
        # Trade logic with dynamic leverage
        if not self.position:
            # Entry signal
            if (self.data.close[0] > self.market_regime[0] and 
                self.trend_strength[0] > 20):
                
                # Calculate position size with dynamic leverage
                portfolio_value = self.broker.get_value()
                risk_amount = portfolio_value * self.params.risk_per_trade
                
                # Use ATR for stop distance
                atr = ct.indicators.ATR(self.data, period=20)[0]
                stop_distance = atr * 1.5
                
                position_size = (risk_amount / stop_distance) / leverage
                
                self.buy(size=position_size)
                self.log(f"Entry with {leverage:.1f}x leverage")
```

## Margin Management

Monitor and manage margin requirements:

```python
class MarginMonitorStrategy(ct.bt.Strategy):
    params = (
        ('initial_margin', 0.2),        # 20% initial margin (5x leverage)
        ('maintenance_margin', 0.1),    # 10% maintenance margin
        ('margin_buffer', 0.05),        # 5% buffer above maintenance
        ('max_drawdown', 0.15),         # 15% max portfolio drawdown
    )
    
    def __init__(self):
        self.margin_level = 1.0  # Start at 100% margin level
        self.liquidation_price = None
        
    def calculate_margin_level(self):
        """Calculate current margin level"""
        if not self.position:
            return 1.0
            
        position_value = abs(self.position.size) * self.data.close[0]
        account_equity = self.broker.get_value()
        
        # Margin level = Account Equity / Used Margin
        used_margin = position_value * self.params.initial_margin
        return account_equity / used_margin if used_margin > 0 else 1.0
        
    def calculate_liquidation_price(self):
        """Calculate liquidation price"""
        if not self.position:
            return None
            
        position_size = self.position.size
        entry_price = self.position.price
        account_balance = self.broker.get_value()
        
        if position_size > 0:  # Long position
            # Liquidation when equity drops to maintenance margin
            liquidation = entry_price * (
                1 - (account_balance / (abs(position_size) * entry_price) - 
                     self.params.maintenance_margin)
            )
        else:  # Short position
            liquidation = entry_price * (
                1 + (account_balance / (abs(position_size) * entry_price) - 
                     self.params.maintenance_margin)
            )
            
        return liquidation
        
    def next(self):
        # Update margin calculations
        self.margin_level = self.calculate_margin_level()
        self.liquidation_price = self.calculate_liquidation_price()
        
        current_price = self.data.close[0]
        
        # Margin call warning
        if self.margin_level < (self.params.maintenance_margin + self.params.margin_buffer):
            self.log(f"WARNING: Margin level low: {self.margin_level:.2%}")
            
            # Reduce position size to improve margin
            if self.position:
                reduction_size = abs(self.position.size) * 0.3  # Reduce by 30%
                if self.position.size > 0:
                    self.sell(size=reduction_size)
                else:
                    self.buy(size=reduction_size)
                    
        # Emergency exit if approaching liquidation
        if (self.liquidation_price and 
            abs(current_price - self.liquidation_price) / current_price < 0.05):  # 5% from liquidation
            
            self.log(f"EMERGENCY: Close to liquidation at {self.liquidation_price:.2f}")
            self.close()
            
        # Regular trading logic
        if not self.position and self.margin_level > 0.5:  # Only trade with good margin
            # Your entry logic here
            pass

# Add margin monitoring analyzer
class MarginAnalyzer(ct.bt.Analyzer):
    def __init__(self):
        self.margin_history = []
        
    def next(self):
        margin_level = self.strategy.margin_level
        liquidation_price = self.strategy.liquidation_price
        
        self.margin_history.append({
            'datetime': self.strategy.data.datetime.datetime(),
            'margin_level': margin_level,
            'liquidation_price': liquidation_price,
            'current_price': self.strategy.data.close[0],
        })
        
    def get_analysis(self):
        return {
            'margin_history': self.margin_history,
            'min_margin_level': min(h['margin_level'] for h in self.margin_history),
            'margin_calls': len([h for h in self.margin_history if h['margin_level'] < 0.15]),
        }

cerebro.addanalyzer(MarginAnalyzer, _name='margin')
```

## Leverage Best Practices

### 1. Start Small
```python
# Progressive leverage scaling
leverage_progression = {
    'beginner': 1.5,
    'intermediate': 2.5,
    'advanced': 5.0,
}

# Use based on experience and account size
current_leverage = leverage_progression['beginner']
```

### 2. Risk Management Rules
```python
class LeverageRiskRules:
    def __init__(self):
        self.max_risk_per_trade = 0.02  # 2% max risk per trade
        self.max_leverage = 3.0         # 3x max leverage
        self.max_portfolio_leverage = 2.0  # Overall portfolio leverage
        
    def calculate_position_size(self, entry_price, stop_price, leverage):
        """Calculate safe position size"""
        portfolio_value = self.get_portfolio_value()
        risk_amount = portfolio_value * self.max_risk_per_trade
        
        stop_distance = abs(entry_price - stop_price)
        base_position_size = risk_amount / stop_distance
        
        # Adjust for leverage
        leveraged_size = base_position_size / leverage
        
        return leveraged_size
```

### 3. Monitor Key Metrics
```python
class LeverageMetrics:
    def __init__(self):
        self.metrics = {
            'effective_leverage': 0,
            'margin_level': 1.0,
            'unrealized_pnl': 0,
            'liquidation_distance': float('inf'),
        }
        
    def update_metrics(self, position, current_price, account_value):
        """Update leverage metrics"""
        if position:
            position_value = abs(position.size) * current_price
            self.metrics['effective_leverage'] = position_value / account_value
            
            # Calculate unrealized PnL
            if position.size > 0:  # Long
                pnl = (current_price - position.price) * position.size
            else:  # Short
                pnl = (position.price - current_price) * abs(position.size)
                
            self.metrics['unrealized_pnl'] = pnl
            self.metrics['margin_level'] = account_value / (position_value * 0.1)  # 10% margin
```

### 4. Emergency Procedures
```python
class EmergencyProtocol:
    def __init__(self, strategy):
        self.strategy = strategy
        self.emergency_triggers = {
            'margin_level': 0.15,      # 15% margin level
            'drawdown': 0.20,          # 20% drawdown
            'volatility': 0.08,        # 8% daily volatility
        }
        
    def check_emergency_conditions(self):
        """Check if emergency exit is needed"""
        margin_level = self.strategy.margin_level
        current_drawdown = self.strategy.drawdown
        current_vol = self.strategy.volatility[0]
        
        if (margin_level < self.emergency_triggers['margin_level'] or
            current_drawdown > self.emergency_triggers['drawdown'] or
            current_vol > self.emergency_triggers['volatility']):
            
            return True
        return False
        
    def execute_emergency_exit(self):
        """Emergency position closure"""
        self.strategy.log("EMERGENCY EXIT TRIGGERED")
        self.strategy.close()  # Close all positions
        # Optionally send alerts, notifications, etc.
```

## Common Leverage Mistakes to Avoid

1. **Over-leveraging**: Using too much leverage for your experience level
2. **Ignoring margin requirements**: Not monitoring margin levels
3. **No stop losses**: Leveraged positions without stops are dangerous
4. **Emotional trading**: Leverage amplifies emotional decisions
5. **Not understanding costs**: Funding fees and interest can accumulate

## Exchange-Specific Leverage Features

### Binance Futures
- Up to 125x leverage (highly risky)
- Cross margin and isolated margin modes
- Funding rates every 8 hours

### Kraken
- Up to 5x leverage on spot margin
- Daily interest charges
- Margin call at 80% maintenance level

### FTX (if available)
- Up to 101x leverage
- Subaccount isolation
- Advanced order types for risk management

## See Also

- [Risk Management](risk_management.md) - Comprehensive risk controls
- [Different Instruments](instruments.md) - Trading futures and margin
- [Multi-Asset Strategies](multi_asset.md) - Leveraged portfolio approaches