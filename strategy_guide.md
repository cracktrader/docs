# Strategy Development Guide

Advanced patterns and practical techniques for building robust strategies with CrackTrader. This guide focuses on clarity and real-world usage.

## Strategy Template

### Basic Strategy Structure

```python
import backtrader as bt
from cracktrader.utils import get_logger

class MyStrategy(bt.Strategy):
    """
    Template for a robust trading strategy.
    """

    # Strategy parameters - always use params tuple
    params = (
        ('period', 20),
        ('threshold', 0.02),
        ('max_position_size', 0.1),  # 10% of portfolio
        ('stop_loss', 0.05),         # 5% stop loss
        ('take_profit', 0.15),       # 15% take profit
        ('printlog', True),
    )

    def __init__(self):
        """Initialize strategy indicators and state."""
        self.logger = get_logger(f'strategy.{self.__class__.__name__}')

        # Initialize indicators
        self.sma = bt.indicators.SimpleMovingAverage(period=self.params.period)
        self.rsi = bt.indicators.RSI(period=14)

        # Track orders and positions
        self.order = None
        self.buyprice = None
        self.buycomm = None

        # Risk management
        self.position_size = 0

        self.logger.info(f"Strategy initialized with params: {dict(self.params._gettuple())}")

    def notify_order(self, order):
        """Handle order execution events."""
        if order.status in [order.Completed]:
            if order.isbuy():
                self.logger.info(f'BUY EXECUTED: {order.executed.price:.2f}')
                self.buyprice = order.executed.price
                self.buycomm = order.executed.comm
            else:
                self.logger.info(f'SELL EXECUTED: {order.executed.price:.2f}')

            self.bar_executed = len(self)

        elif order.status in [order.Canceled, order.Margin, order.Rejected]:
            self.logger.warning(f'Order {order.status}: {order.getstatusname()}')

        # Clear order reference
        self.order = None

    def notify_trade(self, trade):
        """Handle completed trades."""
        if not trade.isclosed:
            return

        self.logger.info(f'TRADE RESULT: PnL ${trade.pnl:.2f}, Net ${trade.pnlcomm:.2f}')

        # Reset tracking variables
        self.buyprice = None
        self.buycomm = None

    def next(self):
        """Main strategy logic - called for each bar."""
        # Check for pending orders
        if self.order:
            return

        # Get current values
        current_price = self.data.close[0]
        portfolio_value = self.broker.getvalue()

        # Risk management checks
        if not self._risk_checks():
            return

        # Strategy logic
        if not self.position:  # Not in market
            if self._should_buy():
                size = self._calculate_position_size()
                self.order = self.buy(size=size)
                self.logger.info(f'BUY ORDER: Size {size:.4f} at {current_price:.2f}')

        else:  # In market
            if self._should_sell():
                self.order = self.sell()
                self.logger.info(f'SELL ORDER at {current_price:.2f}')
            elif self._should_stop_loss():
                self.order = self.sell()
                self.logger.warning(f'STOP LOSS at {current_price:.2f}')
            elif self._should_take_profit():
                self.order = self.sell()
                self.logger.info(f'TAKE PROFIT at {current_price:.2f}')

    def _should_buy(self):
        """Determine if we should enter a long position."""
        # Example: Buy when price above SMA and RSI oversold
        return (self.data.close[0] > self.sma[0] and
                self.rsi[0] < 30)

    def _should_sell(self):
        """Determine if we should exit position normally."""
        # Example: Sell when price below SMA and RSI overbought
        return (self.data.close[0] < self.sma[0] and
                self.rsi[0] > 70)

    def _should_stop_loss(self):
        """Check stop loss condition."""
        if not self.position or not self.buyprice:
            return False

        current_price = self.data.close[0]
        loss_pct = (self.buyprice - current_price) / self.buyprice
        return loss_pct >= self.params.stop_loss

    def _should_take_profit(self):
        """Check take profit condition."""
        if not self.position or not self.buyprice:
            return False

        current_price = self.data.close[0]
        profit_pct = (current_price - self.buyprice) / self.buyprice
        return profit_pct >= self.params.take_profit

    def _calculate_position_size(self):
        """Calculate position size based on risk management."""
        portfolio_value = self.broker.getvalue()
        max_value = portfolio_value * self.params.max_position_size
        current_price = self.data.close[0]
        return max_value / current_price

    def _risk_checks(self):
        """Perform risk management checks."""
        # Example checks
        portfolio_value = self.broker.getvalue()

        # Don't trade if portfolio below threshold
        if portfolio_value < 1000:
            return False

        # Don't trade during low volume periods
        if len(self.data.volume) > 0 and self.data.volume[0] < 1000:
            return False

        return True

    def stop(self):
        """Called when strategy stops."""
        self.logger.info(f'Strategy stopped. Final value: ${self.broker.getvalue():.2f}')
```

## Trading Modes

### Backtest Mode

```python
def run_backtest(strategy_class, **params):
    """Run strategy in backtest mode."""
    from cracktrader import Cerebro, CCXTStore, CCXTDataFeed

    cerebro = Cerebro()

    # Add strategy with parameters
    cerebro.addstrategy(strategy_class, **params)

    # Create historical data feed
    store = CCXTStore(exchange='binance')
    feed = CCXTDataFeed(
        store=store,
        symbol='BTC/USDT',
        ccxt_timeframe='1h'
    )
    cerebro.adddata(feed)

    # Configure broker for backtesting
    cerebro.broker.setcash(10000.0)
    cerebro.broker.setcommission(commission=0.001)

    # Add analyzers
    cerebro.addanalyzer(bt.analyzers.SharpeRatio, _name='sharpe')
    cerebro.addanalyzer(bt.analyzers.Returns, _name='returns')
    cerebro.addanalyzer(bt.analyzers.TradeAnalyzer, _name='trades')
    cerebro.addanalyzer(bt.analyzers.DrawDown, _name='drawdown')

    return cerebro.run()

# Usage
results = run_backtest(MyStrategy, period=20, threshold=0.02)
```

### Paper Trading Mode

```python
def run_paper_trading(strategy_class, **params):
    """Run strategy in paper trading mode."""
    from cracktrader import Cerebro, CCXTStore, CCXTDataFeed
    from cracktrader.broker import BrokerFactory

    cerebro = Cerebro()
    cerebro.addstrategy(strategy_class, **params)

    # Create store with sandbox/testnet
    store = CCXTStore(exchange='binance', sandbox=True, config={'apiKey': '...', 'secret': '...'})

    # Create live data feed
    feed = CCXTDataFeed(store=store, symbol='BTC/USDT', ccxt_timeframe='1h')
    cerebro.adddata(feed)

    # Use paper trading broker
    broker = BrokerFactory.create(mode='paper', cash=10000)
    cerebro.setbroker(broker)

    # Run indefinitely (Ctrl+C to stop)
    cerebro.run()

```

## Example Strategies

### Moving Average Crossover

--8<-- "examples/moving_average_cross.py"

### Mean Reversion

--8<-- "examples/mean_reversion.py"

### Backtesting Wrapper

--8<-- "examples/basic_backtest.py"

### Live Data Feed Utility

--8<-- "examples/live_data_feed.py"

### Usage

```python
run_paper_trading(MyStrategy, period=20, threshold=0.02)
```

### Live Trading Mode

```python
def run_live_trading(strategy_class, **params):
    """Run strategy in live trading mode."""
    from cracktrader import Cerebro, CCXTStore, CCXTDataFeed
    from cracktrader.broker import CCXTLiveBroker
    import logging
    import os

    # Set up production logging
    logging.basicConfig(level=logging.INFO)

    cerebro = Cerebro()
    cerebro.addstrategy(strategy_class, **params)

    # Create store with real credentials
    store = CCXTStore(
        exchange='binance',
        config={
            'apiKey': os.getenv('BINANCE_API_KEY'),
            'secret': os.getenv('BINANCE_SECRET'),
            'sandbox': False  # REAL MONEY!
        }
    )

    feed = CCXTDataFeed(store=store, symbol='BTC/USDT', ccxt_timeframe='1h')
    cerebro.adddata(feed)

    # Use live trading broker
    broker = CCXTLiveBroker(store=store, mode='live')
    cerebro.broker = broker

    # Run with monitoring
    try:
        cerebro.run()
    except KeyboardInterrupt:
        print("Strategy stopped by user")
    except Exception as e:
        print(f"Strategy failed: {e}")
        # Add alerting logic here

# Usage (with extreme caution!)
# run_live_trading(MyStrategy, period=20, threshold=0.02)
```

## Advanced Strategy Patterns

### Multi-Timeframe Strategy

```python
class MultiTimeframeStrategy(bt.Strategy):
    """Strategy using multiple timeframes for confirmation."""

    params = (
        ('fast_period', 10),
        ('slow_period', 50),
    )

    def __init__(self):
        # Primary timeframe (1h)
        self.data0 = self.datas[0]
        self.fast_ma_1h = bt.indicators.SMA(self.data0, period=self.params.fast_period)
        self.slow_ma_1h = bt.indicators.SMA(self.data0, period=self.params.slow_period)

        # Higher timeframe (4h) - resample
        self.data1 = self.datas[1] if len(self.datas) > 1 else None
        if self.data1:
            self.trend_4h = bt.indicators.SMA(self.data1, period=20)

    def next(self):
        # Get trend from higher timeframe
        if self.data1:
            higher_tf_trend = self.data1.close[0] > self.trend_4h[0]
        else:
            higher_tf_trend = True  # Default to bullish if no higher TF

        # Only trade in direction of higher timeframe trend
        if not self.position:
            if (self.fast_ma_1h[0] > self.slow_ma_1h[0] and
                higher_tf_trend):
                self.buy()
        else:
            if self.fast_ma_1h[0] < self.slow_ma_1h[0]:
                self.sell()

# Add multiple timeframes
def create_multi_tf_cerebro():
    cerebro = Cerebro()

    store = CCXTStore(exchange='binance')

    # 1h data (primary)
    feed_1h = CCXTDataFeed(store=store, symbol='BTC/USDT', ccxt_timeframe='1h')
    cerebro.adddata(feed_1h, name='1h')

    # 4h data (higher timeframe)
    feed_4h = CCXTDataFeed(store=store, symbol='BTC/USDT', ccxt_timeframe='4h')
    cerebro.adddata(feed_4h, name='4h')

    cerebro.addstrategy(MultiTimeframeStrategy)
    return cerebro
```

### Portfolio Strategy (Multiple Assets)

```python
class PortfolioStrategy(bt.Strategy):
    """Strategy trading multiple assets with correlation analysis."""

    params = (
        ('rebalance_freq', 24),  # Rebalance every 24 bars
        ('correlation_window', 50),
    )

    def __init__(self):
        self.rebalance_counter = 0

        # Track each asset
        self.assets = {}
        for i, data in enumerate(self.datas):
            symbol = data._name if hasattr(data, '_name') else f'asset_{i}'
            self.assets[symbol] = {
                'data': data,
                'sma': bt.indicators.SMA(data, period=20),
                'rsi': bt.indicators.RSI(data, period=14),
                'weight': 1.0 / len(self.datas)  # Equal weight initially
            }

    def next(self):
        self.rebalance_counter += 1

        if self.rebalance_counter >= self.params.rebalance_freq:
            self._rebalance_portfolio()
            self.rebalance_counter = 0

    def _rebalance_portfolio(self):
        """Rebalance portfolio based on strategy signals."""
        total_value = self.broker.getvalue()

        for symbol, asset_info in self.assets.items():
            data = asset_info['data']
            current_price = data.close[0]

            # Calculate target position value
            target_value = total_value * asset_info['weight']
            target_size = target_value / current_price

            # Get current position
            current_size = self.getposition(data).size

            # Calculate difference and trade
            size_diff = target_size - current_size

            if abs(size_diff) > 0.001:  # Minimum trade size
                self.order_target_percent(data=data, target=asset_info['weight'])

# Create portfolio cerebro
def create_portfolio_cerebro():
    cerebro = Cerebro()

    store = CCXTStore(exchange='binance')

    # Add multiple assets
    symbols = ['BTC/USDT', 'ETH/USDT', 'ADA/USDT', 'DOT/USDT']
    for symbol in symbols:
        feed = CCXTDataFeed(store=store, symbol=symbol, ccxt_timeframe='1h')
        cerebro.adddata(feed, name=symbol.replace('/', '_'))

    cerebro.addstrategy(PortfolioStrategy)
    return cerebro
```

### Machine Learning Strategy

```python
import numpy as np
import pandas as pd
from sklearn.ensemble import RandomForestClassifier
from sklearn.preprocessing import StandardScaler

class MLStrategy(bt.Strategy):
    """Strategy using machine learning for predictions."""

    params = (
        ('lookback_period', 50),
        ('retrain_frequency', 100),
        ('prediction_threshold', 0.6),
    )

    def __init__(self):
        self.ml_model = RandomForestClassifier(n_estimators=100, random_state=42)
        self.scaler = StandardScaler()
        self.bar_count = 0
        self.is_trained = False

        # Technical indicators for features
        self.sma_10 = bt.indicators.SMA(period=10)
        self.sma_20 = bt.indicators.SMA(period=20)
        self.rsi = bt.indicators.RSI(period=14)
        self.macd = bt.indicators.MACD()
        self.bb = bt.indicators.BollingerBands(period=20)

    def next(self):
        self.bar_count += 1

        # Need enough data for features
        if len(self.data) < self.params.lookback_period:
            return

        # Retrain model periodically
        if (self.bar_count % self.params.retrain_frequency == 0 or
            not self.is_trained):
            self._train_model()

        if not self.is_trained:
            return

        # Make prediction
        features = self._extract_features()
        prediction_proba = self.ml_model.predict_proba([features])[0]

        # Trading logic based on prediction
        if not self.position:
            if prediction_proba[1] > self.params.prediction_threshold:  # Buy signal
                self.buy()
        else:
            if prediction_proba[0] > self.params.prediction_threshold:  # Sell signal
                self.sell()

    def _extract_features(self):
        """Extract features for ML model."""
        # Price-based features
        close_prices = np.array([self.data.close[-i] for i in range(10, 0, -1)])
        returns = np.diff(close_prices) / close_prices[:-1]

        # Technical indicator features
        features = [
            self.sma_10[0] / self.data.close[0] - 1,  # SMA ratio
            self.sma_20[0] / self.data.close[0] - 1,
            self.rsi[0] / 100,  # Normalized RSI
            self.macd.macd[0],
            (self.data.close[0] - self.bb.bot[0]) / (self.bb.top[0] - self.bb.bot[0]),  # BB position
        ]

        # Add return features
        features.extend(returns[-5:])  # Last 5 returns

        # Add volume features if available
        if len(self.data.volume) > 0:
            volume_sma = np.mean([self.data.volume[-i] for i in range(5, 0, -1)])
            features.append(self.data.volume[0] / volume_sma - 1)

        return features

    def _train_model(self):
        """Train the ML model on historical data."""
        if len(self.data) < self.params.lookback_period + 20:
            return

        # Prepare training data
        X, y = [], []

        for i in range(self.params.lookback_period, len(self.data) - 1):
            # Extract features at bar i
            features = self._extract_features_at_bar(i)

            # Label: 1 if next bar goes up, 0 if down
            next_return = (self.data.close[-len(self.data)+i+1] -
                          self.data.close[-len(self.data)+i]) / self.data.close[-len(self.data)+i]
            label = 1 if next_return > 0 else 0

            X.append(features)
            y.append(label)

        if len(X) < 20:  # Need minimum samples
            return

        # Scale features and train
        X_scaled = self.scaler.fit_transform(X)
        self.ml_model.fit(X_scaled, y)
        self.is_trained = True

    def _extract_features_at_bar(self, bar_idx):
        """Extract features at a specific historical bar."""
        # This is a simplified version - you'd implement the full feature extraction
        # for any historical bar index
        return [0.0] * 10  # Placeholder
```

## Risk Management

### Position Sizing

```python
class VolatilityPositionSizing(bt.Strategy):
    """Strategy with volatility-based position sizing."""

    params = (
        ('risk_per_trade', 0.02),  # Risk 2% per trade
        ('volatility_period', 20),
    )

    def __init__(self):
        self.atr = bt.indicators.ATR(period=self.params.volatility_period)

    def calculate_position_size(self):
        """Calculate position size based on volatility."""
        portfolio_value = self.broker.getvalue()
        risk_amount = portfolio_value * self.params.risk_per_trade

        # Use ATR as proxy for volatility
        volatility = self.atr[0]

        if volatility > 0:
            # Position size = Risk Amount / (Stop Distance)
            # Assuming stop at 2 ATRs below entry
            stop_distance = 2 * volatility
            position_value = risk_amount / (stop_distance / self.data.close[0])
            position_size = position_value / self.data.close[0]

            # Cap at maximum position size
            max_position_value = portfolio_value * 0.1  # 10% max
            max_position_size = max_position_value / self.data.close[0]

            return min(position_size, max_position_size)

        return 0

    def next(self):
        if not self.position and self.should_buy():
            size = self.calculate_position_size()
            if size > 0:
                self.buy(size=size)
```

### Stop Loss and Take Profit

```python
class AdvancedRiskManagement(bt.Strategy):
    """Strategy with advanced risk management features."""

    params = (
        ('stop_loss_atr_mult', 2.0),
        ('take_profit_atr_mult', 3.0),
        ('trailing_stop_atr_mult', 1.5),
    )

    def __init__(self):
        self.atr = bt.indicators.ATR(period=20)
        self.entry_price = None
        self.stop_price = None
        self.target_price = None
        self.highest_price = None

    def notify_order(self, order):
        if order.status == order.Completed:
            if order.isbuy():
                self.entry_price = order.executed.price
                self.stop_price = self.entry_price - (self.params.stop_loss_atr_mult * self.atr[0])
                self.target_price = self.entry_price + (self.params.take_profit_atr_mult * self.atr[0])
                self.highest_price = self.entry_price
            else:
                self.entry_price = None
                self.stop_price = None
                self.target_price = None
                self.highest_price = None

    def next(self):
        if self.position:
            current_price = self.data.close[0]

            # Update highest price for trailing stop
            if current_price > self.highest_price:
                self.highest_price = current_price
                # Update trailing stop
                trailing_stop = self.highest_price - (self.params.trailing_stop_atr_mult * self.atr[0])
                self.stop_price = max(self.stop_price, trailing_stop)

            # Check exit conditions
            if current_price <= self.stop_price:
                self.sell()  # Stop loss
            elif current_price >= self.target_price:
                self.sell()  # Take profit
```

## Performance Analysis

### Custom Analyzers

```python
class DetailedTradeAnalyzer(bt.Analyzer):
    """Custom analyzer for detailed trade analysis."""

    def __init__(self):
        self.trades = []
        self.current_trade = None

    def notify_order(self, order):
        if order.status == order.Completed:
            if order.isbuy():
                self.current_trade = {
                    'entry_time': self.strategy.datetime.datetime(),
                    'entry_price': order.executed.price,
                    'size': order.executed.size,
                    'entry_value': order.executed.value
                }
            elif order.issell() and self.current_trade:
                self.current_trade.update({
                    'exit_time': self.strategy.datetime.datetime(),
                    'exit_price': order.executed.price,
                    'exit_value': order.executed.value,
                    'commission': order.executed.comm
                })

                # Calculate metrics
                pnl = self.current_trade['exit_value'] - self.current_trade['entry_value']
                pnl_pct = pnl / self.current_trade['entry_value']
                duration = self.current_trade['exit_time'] - self.current_trade['entry_time']

                self.current_trade.update({
                    'pnl': pnl,
                    'pnl_pct': pnl_pct,
                    'duration': duration.total_seconds() / 3600  # hours
                })

                self.trades.append(self.current_trade)
                self.current_trade = None

    def get_analysis(self):
        if not self.trades:
            return {}

        # Calculate detailed statistics
        pnls = [trade['pnl'] for trade in self.trades]
        pnl_pcts = [trade['pnl_pct'] for trade in self.trades]
        durations = [trade['duration'] for trade in self.trades]

        winners = [pnl for pnl in pnls if pnl > 0]
        losers = [pnl for pnl in pnls if pnl < 0]

        return {
            'total_trades': len(self.trades),
            'winners': len(winners),
            'losers': len(losers),
            'win_rate': len(winners) / len(self.trades),
            'avg_win': np.mean(winners) if winners else 0,
            'avg_loss': np.mean(losers) if losers else 0,
            'profit_factor': abs(sum(winners) / sum(losers)) if losers else float('inf'),
            'avg_duration': np.mean(durations),
            'max_duration': max(durations),
            'min_duration': min(durations),
            'total_pnl': sum(pnls),
            'avg_pnl_pct': np.mean(pnl_pcts),
            'max_win': max(pnls),
            'max_loss': min(pnls),
            'trades': self.trades
        }

# Usage
cerebro.addanalyzer(DetailedTradeAnalyzer, _name='detailed_trades')
results = cerebro.run()
detailed_analysis = results[0].analyzers.detailed_trades.get_analysis()
```

## Strategy Optimization

### Parameter Optimization

```python
def optimize_strategy():
    """Optimize strategy parameters using grid search."""

    cerebro = Cerebro()

    # Add data
    store = CCXTStore(exchange_id='binance')
    feed = CCXTDataFeed(
        store=store,
        symbol='BTC/USDT',
        timeframe='1h',
        start_date='2024-01-01T00:00:00Z',
        end_date='2024-03-01T23:59:59Z'
    )
    cerebro.adddata(feed)

    # Optimize parameters
    cerebro.optstrategy(
        MyStrategy,
        period=range(10, 51, 5),      # Test periods 10, 15, 20, ..., 50
        threshold=[0.01, 0.02, 0.03], # Test thresholds 1%, 2%, 3%
    )

    cerebro.addanalyzer(bt.analyzers.SharpeRatio, _name='sharpe')
    cerebro.addanalyzer(bt.analyzers.Returns, _name='returns')

    results = cerebro.run()

    # Find best parameters
    best_result = None
    best_sharpe = -float('inf')

    for result in results:
        strategy = result[0]
        sharpe = strategy.analyzers.sharpe.get_analysis().get('sharperatio', 0)

        if sharpe > best_sharpe:
            best_sharpe = sharpe
            best_result = strategy

    print(f"Best parameters: {dict(best_result.params._gettuple())}")
    print(f"Best Sharpe ratio: {best_sharpe}")

    return best_result

# Run optimization
best_strategy = optimize_strategy()
```

## Best Practices

### 1. **Logging and Monitoring**

```python
from cracktrader.utils import setup_logging, get_logger, log_trade, log_performance

class ProductionStrategy(bt.Strategy):
    def __init__(self):
        # Set up proper logging
        setup_logging(level='INFO', json_enabled=True)
        self.logger = get_logger('strategy.production')

        # Log strategy initialization
        self.logger.info(f"Strategy started with params: {dict(self.params._gettuple())}")

    def notify_trade(self, trade):
        if trade.isclosed:
            # Use structured logging for trades
            log_trade(
                symbol=self.data._name,
                side='SELL',  # Closing trade
                size=abs(trade.size),
                price=trade.price,
                pnl=trade.pnl,
                trade_id=str(trade.ref)
            )

            # Log performance metrics
            log_performance(
                metric='trade_pnl',
                value=trade.pnl,
                strategy=self.__class__.__name__
            )
```

### 2. **Error Handling**

```python
class RobustStrategy(bt.Strategy):
    def next(self):
        try:
            # Main strategy logic
            if self.should_trade():
                self.execute_trade()

        except Exception as e:
            self.logger.error(f"Strategy error: {e}", exc_info=True)
            # Don't let strategy crash - continue to next bar

    def should_trade(self):
        # Defensive programming
        if len(self.data) < 10:
            return False

        if not hasattr(self, 'sma') or self.sma is None:
            return False

        if self.data.close[0] <= 0:
            self.logger.warning("Invalid price data")
            return False

        return True
```

### 3. **Configuration Management**

```python
import yaml
from dataclasses import dataclass

@dataclass
class StrategyConfig:
    """Type-safe strategy configuration."""
    fast_period: int = 10
    slow_period: int = 30
    stop_loss: float = 0.05
    take_profit: float = 0.15
    max_position: float = 0.1

def load_strategy_config(config_file: str) -> StrategyConfig:
    """Load strategy configuration from file."""
    with open(config_file, 'r') as f:
        config_dict = yaml.safe_load(f)

    return StrategyConfig(**config_dict.get('strategy', {}))

class ConfigurableStrategy(bt.Strategy):
    def __init__(self):
        # Load configuration
        self.config = load_strategy_config('strategy_config.yaml')

        # Use configuration
        self.sma_fast = bt.indicators.SMA(period=self.config.fast_period)
        self.sma_slow = bt.indicators.SMA(period=self.config.slow_period)
```

## Testing Strategies

### Unit Testing

```python
import unittest
from unittest.mock import Mock, patch

class TestMyStrategy(unittest.TestCase):
    def setUp(self):
        self.strategy = MyStrategy()
        self.strategy.data = Mock()
        self.strategy.broker = Mock()

    def test_should_buy_signal(self):
        # Mock indicator values
        self.strategy.sma = Mock()
        self.strategy.sma.__getitem__ = Mock(return_value=100)
        self.strategy.data.close = Mock()
        self.strategy.data.close.__getitem__ = Mock(return_value=105)

        # Test buy signal
        result = self.strategy._should_buy()
        self.assertTrue(result)

    def test_position_sizing(self):
        self.strategy.broker.getvalue.return_value = 10000
        self.strategy.data.close = Mock()
        self.strategy.data.close.__getitem__ = Mock(return_value=100)

        size = self.strategy._calculate_position_size()
        self.assertGreater(size, 0)
        self.assertLess(size, 1.0)  # Max 10% position
```

### Integration Testing

```python
def test_strategy_integration():
    """Test strategy with realistic data."""
    from tests.fixtures.fake_exchange import FakeExchange

    cerebro = Cerebro()
    cerebro.addstrategy(MyStrategy, period=20)

    # Use fake exchange for testing
    fake_exchange = FakeExchange()
    store = CCXTStore(exchange_id='fake', exchange=fake_exchange)
    feed = CCXTDataFeed(store=store, symbol='BTC/USDT', timeframe='1h')
    cerebro.adddata(feed)

    results = cerebro.run()

    # Verify strategy ran without errors
    assert len(results) > 0
    assert results[0].analyzers is not None
```

---

**Next Steps:**
- [Data Feeds Guide](core_concepts/feeds.md) - Working with market data
- [Broker Integration](core_concepts/brokers.md) - Order management and execution
- [Testing Strategies](advanced/testing.md) - Comprehensive testing approaches
