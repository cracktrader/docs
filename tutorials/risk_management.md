# Risk Management

Comprehensive risk management is crucial for successful cryptocurrency trading. Cracktrader provides extensive tools and patterns for managing various types of trading risk.

## Overview

Risk management in crypto trading involves:

- **Position sizing**: Determining appropriate trade sizes
- **Stop losses**: Limiting downside on individual trades  
- **Portfolio diversification**: Spreading risk across assets and strategies
- **Leverage management**: Controlling borrowed capital usage
- **Drawdown control**: Limiting overall portfolio losses
- **Correlation monitoring**: Understanding asset relationships

## Core Risk Management Strategy

```python
import cracktrader as ct
from cracktrader.indicators import SMA, RSI, ATR, StdDev

class RiskManagedStrategy(ct.bt.Strategy):
    params = (
        # Risk parameters
        ('max_risk_per_trade', 0.02),    # 2% max risk per trade
        ('max_portfolio_risk', 0.06),    # 6% max total portfolio risk
        ('max_drawdown', 0.15),          # 15% max drawdown
        ('max_positions', 5),            # Max concurrent positions
        
        # Position sizing
        ('base_position_size', 0.1),     # 10% base allocation
        ('volatility_adjustment', True), # Adjust size for volatility
        
        # Stop loss settings
        ('atr_stop_multiplier', 2.0),    # 2x ATR for stops
        ('max_stop_loss', 0.08),         # 8% max stop loss
        ('trailing_stop', True),         # Use trailing stops
    )
    
    def __init__(self):
        # Technical indicators
        self.sma = SMA(self.data.close, period=20)
        self.rsi = RSI(self.data.close, period=14)
        self.atr = ATR(self.data, period=20)
        self.volatility = StdDev(self.data.close.pct_change(), period=20)
        
        # Risk tracking
        self.positions_count = 0
        self.portfolio_risk = 0
        self.peak_value = 0
        self.drawdown = 0
        self.stop_prices = {}
        
    def calculate_position_size(self, entry_price, stop_price):
        """Calculate position size based on risk parameters"""
        portfolio_value = self.broker.get_value()
        
        # Risk amount per trade
        risk_per_trade = min(
            portfolio_value * self.params.max_risk_per_trade,
            portfolio_value * self.params.max_portfolio_risk - self.portfolio_risk
        )
        
        # Stop distance
        stop_distance = abs(entry_price - stop_price)
        
        # Base position size
        if stop_distance > 0:
            base_size = risk_per_trade / stop_distance
        else:
            base_size = portfolio_value * self.params.base_position_size / entry_price
            
        # Volatility adjustment
        if self.params.volatility_adjustment:
            current_vol = self.volatility[0]
            avg_vol = 0.02  # Assume 2% average daily volatility
            vol_adjustment = avg_vol / max(current_vol, 0.01)  # Reduce size in high vol
            base_size *= vol_adjustment
            
        # Maximum position size constraint
        max_size = (portfolio_value * self.params.base_position_size) / entry_price
        
        return min(base_size, max_size)
        
    def calculate_stop_price(self, entry_price, position_type):
        """Calculate stop loss price using ATR"""
        atr_value = self.atr[0]
        atr_stop_distance = atr_value * self.params.atr_stop_multiplier
        
        if position_type == 'long':
            stop_price = entry_price - atr_stop_distance
            # Don't allow stop loss greater than max
            max_stop = entry_price * (1 - self.params.max_stop_loss)
            stop_price = max(stop_price, max_stop)
        else:  # short
            stop_price = entry_price + atr_stop_distance
            max_stop = entry_price * (1 + self.params.max_stop_loss)
            stop_price = min(stop_price, max_stop)
            
        return stop_price
        
    def update_trailing_stops(self):
        """Update trailing stops for open positions"""
        if not self.params.trailing_stop:
            return
            
        current_price = self.data.close[0]
        
        for position in self.broker.positions:
            if position.size == 0:
                continue
                
            position_id = id(position)
            
            if position.size > 0:  # Long position
                # Calculate new trailing stop
                atr_distance = self.atr[0] * self.params.atr_stop_multiplier
                new_stop = current_price - atr_distance
                
                # Only update if new stop is higher (for long positions)
                if (position_id not in self.stop_prices or 
                    new_stop > self.stop_prices[position_id]):
                    self.stop_prices[position_id] = new_stop
                    
            else:  # Short position
                atr_distance = self.atr[0] * self.params.atr_stop_multiplier
                new_stop = current_price + atr_distance
                
                # Only update if new stop is lower (for short positions)
                if (position_id not in self.stop_prices or 
                    new_stop < self.stop_prices[position_id]):
                    self.stop_prices[position_id] = new_stop
                    
    def check_drawdown(self):
        """Check portfolio drawdown and adjust risk if needed"""
        current_value = self.broker.get_value()
        
        if current_value > self.peak_value:
            self.peak_value = current_value
            
        self.drawdown = (self.peak_value - current_value) / self.peak_value
        
        # If drawdown exceeds maximum, close all positions
        if self.drawdown > self.params.max_drawdown:
            self.log(f"Max drawdown exceeded: {self.drawdown:.2%}")
            for position in self.broker.positions:
                if position.size != 0:
                    self.close(data=position.data)
            return True
            
        return False
        
    def next(self):
        # Risk management checks
        if self.check_drawdown():
            return
            
        self.update_trailing_stops()
        
        current_price = self.data.close[0]
        
        # Check stop losses
        for position in self.broker.positions:
            if position.size == 0:
                continue
                
            position_id = id(position)
            stop_price = self.stop_prices.get(position_id)
            
            if stop_price:
                if (position.size > 0 and current_price <= stop_price) or \
                   (position.size < 0 and current_price >= stop_price):
                    self.close(data=position.data)
                    self.log(f"Stop loss hit at {current_price:.2f}")
                    
        # Count current positions
        self.positions_count = sum(1 for p in self.broker.positions if p.size != 0)
        
        # Entry logic (only if within risk limits)
        if (self.positions_count < self.params.max_positions and 
            self.portfolio_risk < self.params.max_portfolio_risk):
            
            # Your entry conditions
            if self.rsi[0] < 30 and self.data.close[0] > self.sma[0]:
                # Calculate position size and stop
                stop_price = self.calculate_stop_price(current_price, 'long')
                position_size = self.calculate_position_size(current_price, stop_price)
                
                if position_size > 0:
                    order = self.buy(size=position_size)
                    if order:
                        self.stop_prices[id(order)] = stop_price
                        self.log(f"Long entry: {current_price:.2f}, Stop: {stop_price:.2f}")

# Setup with risk management
cerebro = ct.Cerebro()
cerebro.addstrategy(RiskManagedStrategy)
```

## Portfolio-Level Risk Management

```python
class PortfolioRiskManager:
    def __init__(self, max_portfolio_risk=0.1, max_correlation=0.7):
        self.max_portfolio_risk = max_portfolio_risk
        self.max_correlation = max_correlation
        self.positions = {}
        self.correlations = {}
        
    def calculate_portfolio_var(self, positions, timeframe_days=1, confidence=0.05):
        """Calculate Value at Risk for portfolio"""
        import numpy as np
        
        # Get returns for each position
        returns = {}
        weights = {}
        total_value = sum(pos['value'] for pos in positions.values())
        
        for symbol, pos in positions.items():
            returns[symbol] = pos['returns']  # Historical returns
            weights[symbol] = pos['value'] / total_value
            
        # Create returns matrix
        symbols = list(returns.keys())
        returns_matrix = np.array([returns[s] for s in symbols]).T
        weights_array = np.array([weights[s] for s in symbols])
        
        # Calculate portfolio returns
        portfolio_returns = np.dot(returns_matrix, weights_array)
        
        # Calculate VaR
        var_percentile = np.percentile(portfolio_returns, confidence * 100)
        return abs(var_percentile) * total_value * np.sqrt(timeframe_days)
        
    def check_correlation_risk(self, new_position_symbol, existing_positions):
        """Check if adding position would create correlation risk"""
        for symbol, position in existing_positions.items():
            correlation = self.get_correlation(new_position_symbol, symbol)
            if abs(correlation) > self.max_correlation:
                return False, f"High correlation with {symbol}: {correlation:.2f}"
        return True, "Correlation check passed"
        
    def get_correlation(self, symbol1, symbol2):
        """Get correlation between two assets"""
        # This would typically fetch from your data source
        # For now, return cached value or default
        key = tuple(sorted([symbol1, symbol2]))
        return self.correlations.get(key, 0.0)
        
    def calculate_optimal_position_size(self, symbol, entry_price, stop_price, 
                                      existing_positions, target_risk=0.02):
        """Calculate optimal position size considering portfolio risk"""
        # Individual position risk
        individual_risk = abs(entry_price - stop_price) / entry_price
        
        # Portfolio concentration risk
        total_exposure = sum(pos['value'] for pos in existing_positions.values())
        max_position_value = total_exposure * 0.2  # 20% max per position
        
        # Calculate size
        risk_based_size = target_risk / individual_risk
        concentration_based_size = max_position_value / entry_price
        
        return min(risk_based_size, concentration_based_size)

class MultiAssetRiskStrategy(ct.bt.Strategy):
    def __init__(self):
        self.risk_manager = PortfolioRiskManager()
        
        # Track positions across all assets
        self.asset_data = {}
        for i, data in enumerate(self.datas):
            symbol = data._name
            self.asset_data[symbol] = {
                'data': data,
                'rsi': RSI(data.close, period=14),
                'atr': ATR(data, period=20),
                'returns': data.close.pct_change(),
            }
            
    def next(self):
        # Update portfolio risk metrics
        current_positions = {}
        for symbol, asset_info in self.asset_data.items():
            position = self.getposition(asset_info['data'])
            if position.size != 0:
                current_positions[symbol] = {
                    'size': position.size,
                    'price': position.price,
                    'value': abs(position.size * asset_info['data'].close[0]),
                    'returns': [asset_info['returns'][i] for i in range(-30, 0)],  # Last 30 days
                }
                
        # Calculate portfolio VaR
        if current_positions:
            portfolio_var = self.risk_manager.calculate_portfolio_var(current_positions)
            portfolio_value = self.broker.get_value()
            
            if portfolio_var > portfolio_value * self.risk_manager.max_portfolio_risk:
                self.log(f"Portfolio VaR too high: {portfolio_var/portfolio_value:.2%}")
                return  # Skip trading this period
                
        # Individual asset trading logic
        for symbol, asset_info in self.asset_data.items():
            data = asset_info['data']
            position = self.getposition(data)
            
            if not position and len(current_positions) < 5:  # Max 5 positions
                # Check entry conditions
                if asset_info['rsi'][0] < 30:
                    # Check correlation risk
                    can_trade, msg = self.risk_manager.check_correlation_risk(
                        symbol, current_positions
                    )
                    
                    if can_trade:
                        # Calculate optimal position size
                        current_price = data.close[0]
                        stop_price = current_price - asset_info['atr'][0] * 2
                        
                        size = self.risk_manager.calculate_optimal_position_size(
                            symbol, current_price, stop_price, current_positions
                        )
                        
                        if size > 0:
                            self.buy(data=data, size=size)
                            self.log(f"Entered {symbol} with size {size:.4f}")
```

## Dynamic Risk Adjustment

```python
class AdaptiveRiskStrategy(ct.bt.Strategy):
    params = (
        ('base_risk', 0.02),
        ('volatility_lookback', 20),
        ('regime_lookback', 60),
    )
    
    def __init__(self):
        # Market regime indicators
        self.volatility = StdDev(self.data.close.pct_change(), period=self.params.volatility_lookback)
        self.trend_strength = ct.indicators.ADX(self.data, period=14)
        self.market_stress = self.calculate_market_stress()
        
        # Risk adjustment factors
        self.current_risk_multiplier = 1.0
        
    def calculate_market_stress(self):
        """Calculate market stress indicator"""
        # Combine volatility, drawdowns, and correlation breakdowns
        rolling_max = ct.indicators.Highest(self.data.close, period=self.params.regime_lookback)
        drawdown = (rolling_max - self.data.close) / rolling_max
        
        # Market stress increases with volatility and drawdowns
        return drawdown + self.volatility * 10  # Scale volatility
        
    def adjust_risk_multiplier(self):
        """Dynamically adjust risk based on market conditions"""
        current_vol = self.volatility[0]
        current_stress = self.market_stress[0]
        trend_strength = self.trend_strength[0]
        
        # Base adjustment for volatility
        if current_vol > 0.03:  # High volatility (>3% daily)
            vol_adjustment = 0.5
        elif current_vol < 0.01:  # Low volatility (<1% daily) 
            vol_adjustment = 1.5
        else:
            vol_adjustment = 1.0
            
        # Adjustment for market stress
        if current_stress > 0.2:  # High stress
            stress_adjustment = 0.3
        elif current_stress < 0.05:  # Low stress
            stress_adjustment = 1.2
        else:
            stress_adjustment = 1.0
            
        # Adjustment for trend strength
        if trend_strength > 25:  # Strong trend
            trend_adjustment = 1.3
        elif trend_strength < 15:  # Weak trend
            trend_adjustment = 0.7
        else:
            trend_adjustment = 1.0
            
        # Combine adjustments
        self.current_risk_multiplier = (vol_adjustment * 
                                       stress_adjustment * 
                                       trend_adjustment)
        
        # Apply bounds
        self.current_risk_multiplier = max(0.1, min(2.0, self.current_risk_multiplier))
        
    def get_adjusted_position_size(self, base_size):
        """Get position size adjusted for current risk level"""
        return base_size * self.current_risk_multiplier
        
    def next(self):
        self.adjust_risk_multiplier()
        
        # Use adjusted risk in position sizing
        if not self.position:
            # Entry logic here
            base_size = 0.1  # 10% base allocation
            adjusted_size = self.get_adjusted_position_size(base_size)
            
            # Your entry conditions
            if self.should_enter():  # Your entry logic
                self.buy(size=adjusted_size)
                self.log(f"Position size adjusted by {self.current_risk_multiplier:.2f}")
```

## Risk Monitoring and Alerts

```python
class RiskMonitor:
    def __init__(self, alert_thresholds=None):
        self.thresholds = alert_thresholds or {
            'drawdown': 0.10,
            'daily_loss': 0.05,
            'var_breach': 1.5,
            'correlation_spike': 0.8,
        }
        
        self.alerts_sent = set()
        
    def check_risk_alerts(self, strategy):
        """Check various risk metrics and send alerts"""
        alerts = []
        
        # Drawdown alert
        if hasattr(strategy, 'drawdown') and strategy.drawdown > self.thresholds['drawdown']:
            alert_key = f"drawdown_{strategy.drawdown:.2f}"
            if alert_key not in self.alerts_sent:
                alerts.append(f"ALERT: Drawdown {strategy.drawdown:.2%} exceeds threshold")
                self.alerts_sent.add(alert_key)
                
        # Daily loss alert
        daily_pnl = self.calculate_daily_pnl(strategy)
        if daily_pnl < -self.thresholds['daily_loss']:
            alert_key = f"daily_loss_{daily_pnl:.3f}"
            if alert_key not in self.alerts_sent:
                alerts.append(f"ALERT: Daily loss {daily_pnl:.2%} exceeds threshold")
                self.alerts_sent.add(alert_key)
                
        # Position concentration alert
        concentration = self.calculate_position_concentration(strategy)
        if concentration > 0.4:  # 40% in single position
            alerts.append(f"ALERT: High position concentration: {concentration:.2%}")
            
        return alerts
        
    def calculate_daily_pnl(self, strategy):
        """Calculate daily P&L percentage"""
        current_value = strategy.broker.get_value()
        start_value = getattr(strategy, 'start_of_day_value', current_value)
        
        return (current_value - start_value) / start_value
        
    def calculate_position_concentration(self, strategy):
        """Calculate largest position as percentage of portfolio"""
        total_value = strategy.broker.get_value()
        largest_position = 0
        
        for position in strategy.broker.positions:
            if position.size != 0:
                position_value = abs(position.size * position.price)
                largest_position = max(largest_position, position_value)
                
        return largest_position / total_value if total_value > 0 else 0

# Risk monitoring analyzer
class RiskAnalyzer(ct.bt.Analyzer):
    def __init__(self):
        self.risk_monitor = RiskMonitor()
        self.daily_stats = []
        
    def next(self):
        # Check for risk alerts
        alerts = self.risk_monitor.check_risk_alerts(self.strategy)
        
        for alert in alerts:
            print(f"{self.strategy.data.datetime.datetime()}: {alert}")
            
        # Log daily risk statistics
        stats = {
            'datetime': self.strategy.data.datetime.datetime(),
            'portfolio_value': self.strategy.broker.get_value(),
            'positions_count': len([p for p in self.strategy.broker.positions if p.size != 0]),
            'drawdown': getattr(self.strategy, 'drawdown', 0),
        }
        
        self.daily_stats.append(stats)
        
    def get_analysis(self):
        return {
            'daily_stats': self.daily_stats,
            'max_drawdown': max(stat['drawdown'] for stat in self.daily_stats),
            'avg_positions': sum(stat['positions_count'] for stat in self.daily_stats) / len(self.daily_stats),
        }

# Add to cerebro
cerebro.addanalyzer(RiskAnalyzer, _name='risk')
```

## Risk Management Best Practices

### 1. Never Risk More Than You Can Afford to Lose
```python
# Set absolute maximum risk limits
MAX_ACCOUNT_RISK = 0.20  # Never risk more than 20% of account
EMERGENCY_STOP_LOSS = 0.15  # Close all positions at 15% drawdown
```

### 2. Diversify Across Multiple Dimensions
- **Assets**: Different cryptocurrencies
- **Strategies**: Multiple trading approaches  
- **Timeframes**: Different time horizons
- **Exchanges**: Counterparty risk diversification

### 3. Use Position Sizing Rules
```python
def calculate_kelly_position_size(win_rate, avg_win, avg_loss):
    """Calculate Kelly Criterion position size"""
    if avg_loss == 0:
        return 0
        
    kelly_pct = win_rate - ((1 - win_rate) / (avg_win / avg_loss))
    return max(0, min(kelly_pct * 0.25, 0.1))  # Conservative Kelly
```

### 4. Monitor Correlations
```python
def check_portfolio_correlation(positions):
    """Ensure portfolio isn't over-concentrated in correlated assets"""
    correlations = calculate_correlation_matrix(positions)
    
    # Alert if average correlation > 0.7
    avg_correlation = correlations.mean()
    if avg_correlation > 0.7:
        return False, f"Portfolio over-correlated: {avg_correlation:.2f}"
    return True, "Correlation acceptable"
```

## See Also

- [Using Leverage](leverage.md) - Leverage-specific risk management
- [Multi-Asset Strategies](multi_asset.md) - Portfolio-level risk controls
- [Multi-Exchange Trading](multi_exchange.md) - Counterparty risk management