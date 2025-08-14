# Multi-Asset Strategies

Trade multiple cryptocurrencies simultaneously to create diversified portfolios, correlation strategies, and advanced trading systems.

## Overview

Multi-asset trading allows you to:

- **Diversify risk** across different cryptocurrencies
- **Capture correlation opportunities** between assets
- **Build momentum strategies** across market sectors
- **Create market-neutral positions** using pairs trading
- **Optimize portfolio allocation** dynamically

## Basic Multi-Asset Setup

```python
import cracktrader as ct
from cracktrader.indicators import SMA, RSI, Correlation

# Create store
store = ct.CCXTStore(
    exchange='binance',
    config={
        'apiKey': 'your_api_key',
        'secret': 'your_secret',
        'sandbox': True,
    }
)

# Create cerebro
cerebro = ct.Cerebro()

# Add multiple data feeds
symbols = ['BTC/USDT', 'ETH/USDT', 'ADA/USDT', 'LINK/USDT', 'DOT/USDT']

for symbol in symbols:
    data = ct.CCXTDataFeed(
        store=store,
        symbol=symbol,
        timeframe='1h'
    )
    cerebro.adddata(data, name=symbol.replace('/', '_'))

# Add broker
broker = ct.CCXTLiveBroker(store=store)
cerebro.setbroker(broker)
```

## Portfolio Momentum Strategy

```python
class PortfolioMomentumStrategy(ct.bt.Strategy):
    params = (
        ('lookback_period', 20),
        ('top_n_assets', 3),
        ('rebalance_frequency', 24),  # Hours
        ('position_size', 0.8),
    )
    
    def __init__(self):
        self.assets = {}
        self.momentum_scores = {}
        
        # Track each asset
        for i, data in enumerate(self.datas):
            symbol = data._name
            
            self.assets[symbol] = {
                'data': data,
                'sma': SMA(data.close, period=self.params.lookback_period),
                'rsi': RSI(data.close, period=14),
                'momentum': data.close / data.close(-self.params.lookback_period)
            }
            
        self.rebalance_timer = 0
        
    def next(self):
        self.rebalance_timer += 1
        
        if self.rebalance_timer >= self.params.rebalance_frequency:
            self.rebalance_portfolio()
            self.rebalance_timer = 0
            
    def calculate_momentum_scores(self):
        """Calculate momentum scores for all assets"""
        scores = {}
        
        for symbol, indicators in self.assets.items():
            # Combine multiple momentum factors
            price_momentum = indicators['momentum'][0] - 1  # Price change
            rsi_momentum = 50 - indicators['rsi'][0]  # RSI divergence from neutral
            trend_momentum = 1 if indicators['data'].close[0] > indicators['sma'][0] else -1
            
            # Weighted momentum score
            scores[symbol] = (
                price_momentum * 0.5 +
                rsi_momentum * 0.003 +  # Scale RSI to similar magnitude
                trend_momentum * 0.1
            )
            
        return scores
        
    def rebalance_portfolio(self):
        """Rebalance portfolio based on momentum scores"""
        scores = self.calculate_momentum_scores()
        
        # Sort assets by momentum score
        sorted_assets = sorted(scores.items(), key=lambda x: x[1], reverse=True)
        top_assets = sorted_assets[:self.params.top_n_assets]
        
        # Close positions in assets not in top N
        current_positions = {data._name: self.getposition(data) 
                           for data in self.datas if self.getposition(data)}
        
        for symbol, position in current_positions.items():
            if symbol not in [asset[0] for asset in top_assets]:
                self.close(data=self.get_data_by_name(symbol))
                
        # Open positions in top assets
        portfolio_value = self.broker.get_value()
        position_value = portfolio_value * self.params.position_size / len(top_assets)
        
        for symbol, score in top_assets:
            data = self.get_data_by_name(symbol)
            current_position = self.getposition(data)
            current_value = current_position.size * data.close[0]
            
            # Adjust position if significant difference
            if abs(current_value - position_value) > position_value * 0.1:
                target_size = position_value / data.close[0]
                size_diff = target_size - current_position.size
                
                if size_diff > 0:
                    self.buy(data=data, size=abs(size_diff))
                else:
                    self.sell(data=data, size=abs(size_diff))
                    
        self.log(f"Rebalanced portfolio. Top assets: {[a[0] for a in top_assets]}")
        
    def get_data_by_name(self, name):
        """Helper to get data feed by name"""
        for data in self.datas:
            if data._name == name:
                return data
        return None

# Add strategy
cerebro.addstrategy(PortfolioMomentumStrategy)
```

## Pairs Trading Strategy

```python
class PairsTradingStrategy(ct.bt.Strategy):
    params = (
        ('pair1', 'BTC_USDT'),  # Data feed names
        ('pair2', 'ETH_USDT'),
        ('lookback', 30),
        ('entry_threshold', 2.0),  # Standard deviations
        ('exit_threshold', 0.5),
        ('position_size', 0.1),
    )
    
    def __init__(self):
        # Get the two assets for pairs trading
        self.data1 = self.get_data_by_name(self.params.pair1)
        self.data2 = self.get_data_by_name(self.params.pair2)
        
        # Calculate price ratio
        self.ratio = self.data1.close / self.data2.close
        
        # Calculate rolling statistics
        self.ratio_sma = SMA(self.ratio, period=self.params.lookback)
        self.ratio_std = ct.indicators.StdDev(self.ratio, period=self.params.lookback)
        
        # Z-score for mean reversion
        self.zscore = (self.ratio - self.ratio_sma) / self.ratio_std
        
        # Track positions
        self.long_pair1 = False
        self.long_pair2 = False
        
    def next(self):
        zscore = self.zscore[0]
        
        # Entry signals
        if not self.long_pair1 and not self.long_pair2:
            if zscore > self.params.entry_threshold:
                # Ratio too high: short pair1, long pair2
                self.sell(data=self.data1, size=self.params.position_size)
                self.buy(data=self.data2, size=self.params.position_size)
                self.long_pair1 = False
                self.long_pair2 = True
                self.log(f"Entered pairs trade: Short {self.params.pair1}, Long {self.params.pair2}")
                
            elif zscore < -self.params.entry_threshold:
                # Ratio too low: long pair1, short pair2  
                self.buy(data=self.data1, size=self.params.position_size)
                self.sell(data=self.data2, size=self.params.position_size)
                self.long_pair1 = True
                self.long_pair2 = False
                self.log(f"Entered pairs trade: Long {self.params.pair1}, Short {self.params.pair2}")
                
        # Exit signals
        elif abs(zscore) < self.params.exit_threshold:
            # Close all positions
            self.close(data=self.data1)
            self.close(data=self.data2)
            self.long_pair1 = False
            self.long_pair2 = False
            self.log("Exited pairs trade")
            
    def get_data_by_name(self, name):
        """Helper to get data feed by name"""
        for data in self.datas:
            if data._name == name:
                return data
        return None
```

## Sector Rotation Strategy

```python
class SectorRotationStrategy(ct.bt.Strategy):
    params = (
        ('sectors', {
            'defi': ['UNI/USDT', 'AAVE/USDT', 'COMP/USDT'],
            'layer1': ['ETH/USDT', 'ADA/USDT', 'SOL/USDT', 'DOT/USDT'],
            'payments': ['XRP/USDT', 'LTC/USDT', 'BCH/USDT'],
            'store_of_value': ['BTC/USDT'],
        }),
        ('lookback_period', 30),
        ('rebalance_frequency', 48),  # Hours
    )
    
    def __init__(self):
        self.sector_data = {}
        self.sector_performance = {}
        
        # Organize data by sector
        for sector, symbols in self.params.sectors.items():
            self.sector_data[sector] = []
            for symbol in symbols:
                data = self.get_data_by_name(symbol.replace('/', '_'))
                if data:
                    self.sector_data[sector].append(data)
                    
        self.rebalance_timer = 0
        
    def calculate_sector_performance(self):
        """Calculate performance for each sector"""
        performance = {}
        
        for sector, data_list in self.sector_data.items():
            sector_returns = []
            
            for data in data_list:
                # Calculate return over lookback period
                current_price = data.close[0]
                past_price = data.close[-self.params.lookback_period]
                
                if past_price and past_price > 0:
                    returns = (current_price - past_price) / past_price
                    sector_returns.append(returns)
                    
            # Average sector performance
            if sector_returns:
                performance[sector] = sum(sector_returns) / len(sector_returns)
            else:
                performance[sector] = 0
                
        return performance
        
    def next(self):
        self.rebalance_timer += 1
        
        if self.rebalance_timer >= self.params.rebalance_frequency:
            self.rotate_sectors()
            self.rebalance_timer = 0
            
    def rotate_sectors(self):
        """Rotate capital to best performing sector"""
        performance = self.calculate_sector_performance()
        
        # Find best performing sector
        best_sector = max(performance.items(), key=lambda x: x[1])
        sector_name, sector_perf = best_sector
        
        self.log(f"Best sector: {sector_name} ({sector_perf:.2%})")
        
        # Close positions in other sectors
        for sector, data_list in self.sector_data.items():
            if sector != sector_name:
                for data in data_list:
                    if self.getposition(data):
                        self.close(data=data)
                        
        # Open positions in best sector
        best_sector_data = self.sector_data[sector_name]
        position_size = 0.8 / len(best_sector_data)  # Equal weight within sector
        
        for data in best_sector_data:
            if not self.getposition(data):
                self.buy(data=data, size=position_size)
                
    def get_data_by_name(self, name):
        """Helper to get data feed by name"""
        for data in self.datas:
            if data._name == name:
                return data
        return None
```

## Risk-Adjusted Portfolio

```python
class RiskAdjustedStrategy(ct.bt.Strategy):
    params = (
        ('lookback_period', 60),
        ('target_volatility', 0.15),  # 15% annual volatility
        ('max_position_size', 0.3),
        ('min_position_size', 0.05),
    )
    
    def __init__(self):
        self.asset_volatilities = {}
        self.asset_returns = {}
        
        for data in self.datas:
            symbol = data._name
            
            # Calculate returns
            self.asset_returns[symbol] = (data.close / data.close(-1) - 1)
            
            # Calculate rolling volatility
            self.asset_volatilities[symbol] = ct.indicators.StdDev(
                self.asset_returns[symbol], 
                period=self.params.lookback_period
            )
            
    def calculate_position_sizes(self):
        """Calculate risk-adjusted position sizes"""
        positions = {}
        total_weight = 0
        
        for data in self.datas:
            symbol = data._name
            volatility = self.asset_volatilities[symbol][0]
            
            if volatility > 0:
                # Inverse volatility weighting
                weight = (1 / volatility)
                
                # Apply constraints
                weight = max(self.params.min_position_size, 
                           min(self.params.max_position_size, weight))
                
                positions[symbol] = weight
                total_weight += weight
                
        # Normalize to sum to target exposure
        target_exposure = 0.9  # 90% invested
        for symbol in positions:
            positions[symbol] = (positions[symbol] / total_weight) * target_exposure
            
        return positions
        
    def next(self):
        if len(self) % 24 == 0:  # Rebalance daily
            position_sizes = self.calculate_position_sizes()
            
            for data in self.datas:
                symbol = data._name
                target_size = position_sizes.get(symbol, 0)
                current_position = self.getposition(data)
                
                # Calculate size difference
                portfolio_value = self.broker.get_value()
                target_value = portfolio_value * target_size
                current_value = current_position.size * data.close[0]
                
                value_diff = target_value - current_value
                
                # Rebalance if difference is significant
                if abs(value_diff) > portfolio_value * 0.02:  # 2% threshold
                    size_diff = value_diff / data.close[0]
                    
                    if size_diff > 0:
                        self.buy(data=data, size=size_diff)
                    else:
                        self.sell(data=data, size=abs(size_diff))
```

## Performance Monitoring

```python
class MultiAssetAnalyzer(ct.bt.Analyzer):
    def __init__(self):
        self.asset_returns = {}
        self.correlations = {}
        
    def next(self):
        # Track individual asset performance
        for data in self.strategy.datas:
            symbol = data._name
            if symbol not in self.asset_returns:
                self.asset_returns[symbol] = []
                
            if len(data.close) > 1:
                returns = (data.close[0] - data.close[-1]) / data.close[-1]
                self.asset_returns[symbol].append(returns)
                
    def stop(self):
        # Calculate correlation matrix
        import numpy as np
        
        symbols = list(self.asset_returns.keys())
        correlations = np.corrcoef([self.asset_returns[s] for s in symbols])
        
        self.correlations = {
            symbols[i]: {symbols[j]: correlations[i][j] 
                        for j in range(len(symbols))}
            for i in range(len(symbols))
        }
        
    def get_analysis(self):
        return {
            'individual_returns': self.asset_returns,
            'correlations': self.correlations,
        }

# Add analyzer to cerebro
cerebro.addanalyzer(MultiAssetAnalyzer, _name='multi_asset')
```

## Best Practices

1. **Diversification**: Don't concentrate too much in correlated assets
2. **Risk Management**: Use position sizing based on volatility
3. **Rebalancing**: Regular rebalancing maintains target allocations
4. **Correlation Monitoring**: Watch for increasing correlations during market stress
5. **Sector Awareness**: Understand which assets belong to which sectors

## See Also

- [Multi-Exchange Trading](multi_exchange.md) - Trade across multiple exchanges
- [Risk Management](risk_management.md) - Advanced risk controls
- [Custom Indicators](custom_indicators.md) - Building portfolio indicators