# Getting Started with Cracktrader

This guide will help you install Cracktrader and run your first trading strategy.

## Installation

### Prerequisites

- **Python 3.9+** (recommended: 3.11 or 3.12)
- **pip** package manager
- **Git** (for development setup)

### Install Methods

#### Basic Installation
```bash
# Clone the repository
git clone https://github.com/your-org/cracktrader.git
cd cracktrader

# Install core features
pip install -e .
```

#### Web Features (Recommended)
```bash
# Install with web API and dashboard
pip install -e .'[web]'
```

#### Development Setup
```bash
# Install development dependencies
pip install -e .'[dev]'

# Verify installation
python -c "import cracktrader; print('✅ Cracktrader installed successfully')"
```

### Verify Installation

```bash
# Check core functionality
python -c "
from cracktrader import Cerebro, CCXTStore
print('✅ Core imports working')
"

# Check web features (if installed)
python -c "
from cracktrader.web.server.main import create_app
print('✅ Web features available')
"
```

## Configuration

### Basic Configuration

Create a configuration file to customize Cracktrader behavior:

```yaml
# config.yaml
trading:
  default_exchange: "binance"
  default_timeframe: "1h"
  sandbox_mode: true  # Always start with sandbox!

logging:
  level: "INFO"
  json_enabled: false
  console_enabled: true
  file_enabled: true

web:
  host: "0.0.0.0"
  port: 8000
  cors_enabled: true
```

### Exchange API Configuration

For live/paper trading, you'll need exchange API credentials:

```yaml
# config.yaml (add to existing file)
exchanges:
  binance:
    apiKey: "your_api_key_here"
    secret: "your_secret_here"
    sandbox: true  # Use testnet
    
  coinbase:
    apiKey: "your_coinbase_key"
    secret: "your_coinbase_secret"
    passphrase: "your_passphrase"
    sandbox: true
```

⚠️ **Security Note**: Never commit API keys to version control. Use environment variables in production:

```bash
export BINANCE_API_KEY="your_key"
export BINANCE_SECRET="your_secret"
```

## Your First Strategy

Let's build a simple moving average crossover strategy step by step.

### Step 1: Create Strategy File

Create `my_first_strategy.py`:

```python
import backtrader as bt
from cracktrader import Cerebro, CCXTStore, CCXTDataFeed
from cracktrader.utils import setup_logging, get_logger

# Set up logging
setup_logging(level='INFO')
logger = get_logger('strategy')

class MovingAverageCross(bt.Strategy):
    """
    Simple moving average crossover strategy.
    
    Buy when fast MA crosses above slow MA.
    Sell when fast MA crosses below slow MA.
    """
    
    params = (
        ('fast_period', 10),
        ('slow_period', 30),
        ('printlog', True),
    )
    
    def __init__(self):
        # Calculate moving averages
        self.fast_ma = bt.indicators.SimpleMovingAverage(
            self.data.close, period=self.params.fast_period
        )
        self.slow_ma = bt.indicators.SimpleMovingAverage(
            self.data.close, period=self.params.slow_period
        )
        
        # Create crossover signal
        self.crossover = bt.indicators.CrossOver(self.fast_ma, self.slow_ma)
        
        # Track orders
        self.order = None
        
    def notify_order(self, order):
        """Handle order status changes."""
        if order.status in [order.Completed]:
            if order.isbuy():
                logger.info(f'BUY EXECUTED: Price {order.executed.price:.2f}, '
                           f'Size {order.executed.size:.4f}')
            else:
                logger.info(f'SELL EXECUTED: Price {order.executed.price:.2f}, '
                           f'Size {order.executed.size:.4f}')
            self.order = None
            
        elif order.status in [order.Canceled, order.Margin, order.Rejected]:
            logger.warning(f'Order {order.status}')
            self.order = None
    
    def notify_trade(self, trade):
        """Handle completed trades."""
        if trade.isclosed:
            logger.info(f'TRADE PROFIT: ${trade.pnl:.2f}')
    
    def next(self):
        """Main strategy logic called for each bar."""
        if self.order:
            return  # Pending order exists
            
        if not self.position:  # Not in market
            if self.crossover > 0:  # Fast MA crosses above slow MA
                logger.info(f'BUY SIGNAL: Fast MA {self.fast_ma[0]:.2f} > Slow MA {self.slow_ma[0]:.2f}')
                self.order = self.buy()
                
        else:  # In market
            if self.crossover < 0:  # Fast MA crosses below slow MA
                logger.info(f'SELL SIGNAL: Fast MA {self.fast_ma[0]:.2f} < Slow MA {self.slow_ma[0]:.2f}')
                self.order = self.sell()

def run_backtest():
    """Run the strategy backtest."""
    
    # Create Cerebro engine
    cerebro = Cerebro()
    
    # Add strategy
    cerebro.addstrategy(MovingAverageCross)
    
    # Create data feed
    store = CCXTStore(exchange_id='binance')
    feed = CCXTDataFeed(
        store=store,
        symbol='BTC/USDT',
        timeframe='1h',
        start_date='2024-01-01T00:00:00Z',
        end_date='2024-03-01T23:59:59Z'
    )
    cerebro.adddata(feed)
    
    # Set initial cash
    cerebro.broker.setcash(10000.0)
    
    # Add commission (Binance: 0.1%)
    cerebro.broker.setcommission(commission=0.001)
    
    # Add analyzers for performance metrics
    cerebro.addanalyzer(bt.analyzers.SharpeRatio, _name='sharpe')
    cerebro.addanalyzer(bt.analyzers.DrawDown, _name='drawdown')
    cerebro.addanalyzer(bt.analyzers.TradeAnalyzer, _name='trades')
    
    # Print starting conditions
    logger.info(f'Starting Portfolio Value: ${cerebro.broker.getvalue():.2f}')
    
    # Run the strategy
    results = cerebro.run()
    
    # Print final results
    logger.info(f'Final Portfolio Value: ${cerebro.broker.getvalue():.2f}')
    
    # Extract performance metrics
    strategy = results[0]
    
    # Sharpe ratio
    sharpe = strategy.analyzers.sharpe.get_analysis()
    logger.info(f"Sharpe Ratio: {sharpe.get('sharperatio', 'N/A')}")
    
    # Drawdown
    drawdown = strategy.analyzers.drawdown.get_analysis()
    logger.info(f"Max Drawdown: {drawdown.get('max', {}).get('drawdown', 'N/A'):.2%}")
    
    # Trade analysis
    trades = strategy.analyzers.trades.get_analysis()
    total_trades = trades.get('total', {}).get('total', 0)
    won_trades = trades.get('won', {}).get('total', 0)
    win_rate = (won_trades / total_trades * 100) if total_trades > 0 else 0
    logger.info(f"Total Trades: {total_trades}, Win Rate: {win_rate:.1f}%")

if __name__ == '__main__':
    run_backtest()
```

### Step 2: Run Your Strategy

```bash
python my_first_strategy.py
```

You should see output like:
```
[12:34:56.789] [INFO    ] strategy     : Starting Portfolio Value: $10000.00
[12:34:57.123] [INFO    ] strategy     : BUY SIGNAL: Fast MA 42150.25 > Slow MA 42100.50
[12:34:57.124] [INFO    ] strategy     : BUY EXECUTED: Price 42155.30, Size 0.2371
...
[12:35:02.456] [INFO    ] strategy     : Final Portfolio Value: $10234.56
[12:35:02.457] [INFO    ] strategy     : Sharpe Ratio: 1.23
[12:35:02.458] [INFO    ] strategy     : Max Drawdown: 5.67%
[12:35:02.459] [INFO    ] strategy     : Total Trades: 8, Win Rate: 62.5%
```

## Next Steps: Paper Trading

Once your backtest looks good, test with live market data using paper trading:

```python
def run_paper_trading():
    """Run strategy with live data but virtual money."""
    
    cerebro = Cerebro()
    cerebro.addstrategy(MovingAverageCross)
    
    # Create store with sandbox mode
    store = CCXTStore(
        exchange_id='binance',
        config={
            'apiKey': 'your_api_key',
            'secret': 'your_secret',
            'sandbox': True  # Use testnet
        }
    )
    
    # Create live data feed
    feed = CCXTDataFeed(
        store=store,
        symbol='BTC/USDT',
        timeframe='1h',
        # No start/end dates = live data
    )
    cerebro.adddata(feed)
    
    # Use paper trading broker
    from cracktrader.broker import CCXTLiveBroker
    broker = CCXTLiveBroker(store=store, mode='paper')
    cerebro.broker = broker
    
    logger.info("Starting paper trading...")
    cerebro.run()

if __name__ == '__main__':
    # run_backtest()  # Comment out backtest
    run_paper_trading()  # Run paper trading instead
```

## CLI Tools

Cracktrader includes several command-line utilities:

### Web Server

Start the web interface:

```bash
# Basic web server
python scripts/run_web_server.py

# Custom configuration
python scripts/run_web_server.py --config config.yaml --port 8080

# Production mode
python scripts/run_web_server.py --workers 4 --log-level info
```

Access the dashboard at http://localhost:8000

### Testing Tools

```bash
# Run integration tests
python scripts/run_integration_tests.py

# Test specific components
python scripts/test_broker.py
python scripts/test_feed.py
python scripts/test_store.py

# Generate test coverage matrix
python scripts/generate_test_matrix.py
```

### Development Tools

```bash
# Run all tests
pytest

# Run tests with coverage
pytest --cov=cracktrader --cov-report=html

# Format code
black src/ tests/ examples/ scripts/

# Lint code
ruff src/ tests/ examples/ scripts/
```

## Web Interface Quickstart

### Start the Server

```bash
python scripts/run_web_server.py
```

### API Usage

Test the REST API:

```bash
# Check health
curl http://localhost:8000/api/v1/health

# Get status
curl http://localhost:8000/api/v1/status

# Start a backtest
curl -X POST http://localhost:8000/api/v1/run/backtest \
  -H "Content-Type: application/json" \
  -d '{
    "strategy_name": "TestStrategy",
    "symbols": ["BTC/USDT"],
    "timeframe": "1h",
    "initial_cash": 10000
  }'

# Get results
curl http://localhost:8000/api/v1/results
```

### Dashboard Features

- **Strategy Management** - Configure and deploy strategies
- **Live Monitoring** - Real-time performance metrics
- **Results Analysis** - Interactive charts and trade history
- **Health Dashboard** - System status and alerts

## Common Issues

### Import Errors

```bash
# ModuleNotFoundError: No module named 'cracktrader'
pip install -e .

# Missing web dependencies
pip install -e .'[web]'
```

### Exchange Connection Issues

```bash
# ccxt.NetworkError: binance
# Check your internet connection and API credentials

# ccxt.AuthenticationError
# Verify API key and secret are correct

# ccxt.PermissionDenied
# Enable trading permissions in exchange account
```

### Data Issues

```bash
# No data received
# Check symbol format: use 'BTC/USDT' not 'BTCUSDT'
# Verify date ranges are valid
# Ensure exchange supports the timeframe

# Timeframe not supported
# Check exchange.timeframes for supported intervals
```

### Performance Issues

```bash
# Slow backtests
# Reduce date range or use higher timeframes
# Enable only necessary analyzers

# Memory issues
# Process data in chunks for long backtests
# Use streaming feeds for live trading
```

## Configuration Reference

### Logging Configuration

```yaml
logging:
  level: "INFO"  # DEBUG, INFO, WARNING, ERROR, CRITICAL
  console_enabled: true
  file_enabled: true
  json_enabled: false  # Set true for structured logs
  log_dir: "logs"
  max_file_size: 10485760  # 10MB
  backup_count: 5
```

### Trading Configuration

```yaml
trading:
  default_exchange: "binance"
  default_timeframe: "1h" 
  sandbox_mode: true
  commission_rate: 0.001  # 0.1%
  slippage_rate: 0.0001   # 0.01%
```

### Web Configuration

```yaml
web:
  host: "0.0.0.0"
  port: 8000
  cors_enabled: true
  cors_origins: ["http://localhost:3000"]
  debug: false
  workers: 1
```

## What's Next?

- **[Strategy Development](strategy_guide.md)** - Learn advanced strategy patterns
- **[Data Feeds](feeds.md)** - Working with multiple data sources
- **[Web API](gui.md)** - Building applications with the REST API
- **[Testing](testing.md)** - Writing tests for your strategies

## Examples

Check out the `examples/` directory for complete implementations:

- `basic_backtest.py` - Simple backtest example
- `live_data_feed.py` - Real-time data consumption
- `logging_and_monitoring_demo.py` - Production monitoring setup
- `web_api_example.py` - Complete API workflow

---

**Having issues?** Check the [troubleshooting section](#common-issues) or open an issue on GitHub.