# Testing Infrastructure Guide

Cracktrader provides comprehensive testing infrastructure with 89 test files covering unit, integration, and end-to-end testing. This guide covers testing strategies, fixtures, mocking policies, and best practices for testing trading strategies.

## Testing Overview

### Test Architecture

```
tests/
├── unit/                    # Isolated component tests (71 files)
│   ├── broker/             # Order management, cash handling
│   ├── feed/               # Data feed functionality
│   ├── store/              # Exchange connectivity
│   ├── cerebro/            # Strategy execution engine
│   ├── commission/         # Fee calculations
│   └── web/                # API and web interface
├── integration/            # Cross-component tests (12 files)
│   ├── data_pipeline/      # End-to-end data flow
│   ├── multi_exchange/     # Cross-exchange integration
│   └── web/                # Web API workflows
├── e2e/                    # End-to-end scenarios (6 files)
│   ├── binance_specific/   # Exchange-specific tests
│   ├── live_data/          # Real market data tests
│   └── strategy_execution/ # Complete strategy workflows
├── fixtures/               # Reusable test components
│   ├── fake_exchange.py    # Mock exchange implementation
│   ├── broker_factory.py   # Test broker creation
│   └── common.py           # Shared test utilities
└── conftest.py             # Global test configuration
```

### Testing Philosophy

**Behavior-Driven Testing:**
- Tests focus on user behavior, not implementation details
- Clear test names describing business scenarios
- Comprehensive edge case coverage

**Isolation and Mocking:**
- Unit tests are completely isolated using mocks
- Integration tests use minimal real dependencies
- End-to-end tests can optionally use live exchanges

**Production Readiness:**
- 2:1 test-to-source ratio (89 test files, 44 source files)
- Comprehensive coverage of trading scenarios
- Performance and stress testing included

## Getting Started with Testing

### Running Tests

```bash
# Run all tests
pytest

# Run specific test categories
pytest tests/unit/                    # Unit tests only
pytest tests/integration/             # Integration tests
pytest tests/e2e/                     # End-to-end tests

# Run tests with coverage
pytest --cov=cracktrader --cov-report=html

# Run tests with detailed output
pytest -v -s

# Run specific test file
pytest tests/unit/broker/test_broker_order_lifecycle.py

# Run specific test method
pytest tests/unit/broker/test_broker_order_lifecycle.py::TestOrderLifecycle::test_market_order_execution
```

### Test Configuration

```python
# conftest.py - Global test configuration
import pytest
from tests.fixtures.fake_exchange import FakeExchange
from tests.fixtures.broker_factory import create_test_broker

@pytest.fixture(scope="session")
def fake_exchange():
    """Shared fake exchange for all tests."""
    return FakeExchange()

@pytest.fixture
def test_broker():
    """Create isolated test broker."""
    return create_test_broker()

@pytest.fixture(autouse=True)
def setup_test_logging():
    """Configure logging for tests."""
    import logging
    logging.getLogger('cracktrader').setLevel(logging.WARNING)
```

## Unit Testing

### Testing Strategies

```python
import unittest
from unittest.mock import Mock, patch, MagicMock
import backtrader as bt
from cracktrader import Cerebro
from tests.fixtures.fake_exchange import FakeExchange

class TestMyStrategy(unittest.TestCase):
    """Unit tests for strategy logic."""

    def setUp(self):
        """Set up test environment."""
        self.cerebro = Cerebro()
        self.strategy_class = MyStrategy

        # Create mock data
        self.mock_data = Mock()
        self.mock_data.close = Mock()
        self.mock_data.volume = Mock()

        # Create mock broker
        self.mock_broker = Mock()
        self.mock_broker.getvalue.return_value = 10000.0
        self.mock_broker.getcash.return_value = 5000.0

    def test_strategy_initialization(self):
        """Test strategy initializes correctly."""
        strategy = self.strategy_class()
        strategy.data = self.mock_data
        strategy.broker = self.mock_broker

        # Test parameter validation
        self.assertIsNotNone(strategy.params.period)
        self.assertGreater(strategy.params.period, 0)

    def test_buy_signal_generation(self):
        """Test buy signal logic."""
        strategy = self.strategy_class()
        strategy.data = self.mock_data

        # Mock indicator values for buy signal
        strategy.sma = Mock()
        strategy.sma.__getitem__ = Mock(return_value=100.0)
        self.mock_data.close.__getitem__ = Mock(return_value=105.0)

        # Test buy signal
        should_buy = strategy._should_buy()
        self.assertTrue(should_buy)

    def test_position_sizing(self):
        """Test position size calculation."""
        strategy = self.strategy_class()
        strategy.broker = self.mock_broker
        strategy.data = self.mock_data

        self.mock_data.close.__getitem__ = Mock(return_value=50000.0)

        position_size = strategy._calculate_position_size()

        # Verify position size is reasonable
        self.assertGreater(position_size, 0)
        self.assertLess(position_size, 1.0)  # Less than 100% of portfolio

    def test_risk_management(self):
        """Test risk management rules."""
        strategy = self.strategy_class()
        strategy.broker = self.mock_broker
        strategy.data = self.mock_data

        # Test stop loss
        strategy.buyprice = 50000.0
        self.mock_data.close.__getitem__ = Mock(return_value=47500.0)  # 5% loss

        should_stop = strategy._should_stop_loss()
        self.assertTrue(should_stop)

        # Test take profit
        self.mock_data.close.__getitem__ = Mock(return_value=57500.0)  # 15% gain

        should_take_profit = strategy._should_take_profit()
        self.assertTrue(should_take_profit)

    @patch('cracktrader.utils.log_trade')
    def test_trade_logging(self, mock_log_trade):
        """Test that trades are logged correctly."""
        strategy = self.strategy_class()

        # Simulate trade completion
        mock_trade = Mock()
        mock_trade.isclosed = True
        mock_trade.pnl = 500.0
        mock_trade.size = 0.1

        strategy.notify_trade(mock_trade)

        # Verify logging was called
        mock_log_trade.assert_called_once()
```

### Testing Data Feeds

```python
class TestCCXTDataFeed(unittest.TestCase):
    """Test CCXT data feed functionality."""

    def setUp(self):
        """Set up test environment."""
        self.fake_exchange = FakeExchange()
        self.store = Mock()
        self.store.exchange = self.fake_exchange

    def test_historical_data_loading(self):
        """Test loading historical OHLCV data."""
        from cracktrader.feeds import CCXTDataFeed

        feed = CCXTDataFeed(
            store=self.store,
            symbol='BTC/USDT',
            timeframe='1h',
            start_date='2024-01-01T00:00:00Z',
            end_date='2024-01-02T00:00:00Z'
        )

        # Mock data return
        self.fake_exchange.set_ohlcv_data([
            [1704067200000, 45000, 45500, 44500, 45200, 1000],  # Sample data
            [1704070800000, 45200, 45800, 45100, 45600, 1200],
        ])

        data = feed._load_data()

        # Verify data structure
        self.assertIsNotNone(data)
        self.assertEqual(len(data), 2)
        self.assertIn('open', data.columns)
        self.assertIn('high', data.columns)
        self.assertIn('low', data.columns)
        self.assertIn('close', data.columns)
        self.assertIn('volume', data.columns)

    def test_data_validation(self):
        """Test data validation and cleaning."""
        from cracktrader.feeds import CCXTDataFeed

        feed = CCXTDataFeed(store=self.store, symbol='BTC/USDT', timeframe='1h')

        # Test with invalid data
        invalid_data = [
            [1704067200000, 45000, 44500, 45500, 45200, 1000],  # high < low
            [1704070800000, 45200, 45800, 45100, 45600, -100],  # negative volume
        ]

        self.fake_exchange.set_ohlcv_data(invalid_data)

        # Should handle invalid data gracefully
        data = feed._load_data()
        self.assertIsNotNone(data)

    def test_timeframe_mapping(self):
        """Test timeframe conversion between CCXT and Backtrader."""
        from cracktrader.feeds.ccxt import CCXT_TO_BT_TIMEFRAME

        # Test common timeframes
        self.assertEqual(CCXT_TO_BT_TIMEFRAME['1m'], (bt.TimeFrame.Minutes, 1))
        self.assertEqual(CCXT_TO_BT_TIMEFRAME['1h'], (bt.TimeFrame.Minutes, 60))
        self.assertEqual(CCXT_TO_BT_TIMEFRAME['1d'], (bt.TimeFrame.Days, 1))
```

### Testing Brokers

```python
class TestCCXTBroker(unittest.TestCase):
    """Test broker order management."""

    def setUp(self):
        """Set up test broker."""
        from tests.fixtures.broker_factory import create_test_broker
        self.broker = create_test_broker(mode='test', initial_cash=10000.0)

    def test_market_order_execution(self):
        """Test market order execution."""
        # Create market buy order
        order = self.broker.buy(size=0.1)

        # Verify order created
        self.assertIsNotNone(order)
        self.assertEqual(order.exectype, bt.Order.Market)
        self.assertEqual(order.size, 0.1)

        # Simulate order execution
        self.broker._execute_order(order, price=50000.0)

        # Verify execution
        self.assertEqual(order.status, bt.Order.Completed)
        self.assertEqual(order.executed.price, 50000.0)
        self.assertEqual(order.executed.size, 0.1)

    def test_limit_order_execution(self):
        """Test limit order execution."""
        # Create limit buy order
        order = self.broker.buy(size=0.1, exectype=bt.Order.Limit, price=49000.0)

        # Test order doesn't execute above limit price
        self.broker._try_execute_order(order, current_price=50000.0)
        self.assertEqual(order.status, bt.Order.Accepted)

        # Test order executes at limit price
        self.broker._try_execute_order(order, current_price=48000.0)
        self.assertEqual(order.status, bt.Order.Completed)

    def test_stop_order_execution(self):
        """Test stop order execution."""
        # Create stop loss order
        order = self.broker.sell(size=0.1, exectype=bt.Order.Stop, price=48000.0)

        # Test order doesn't execute above stop price
        self.broker._try_execute_order(order, current_price=49000.0)
        self.assertEqual(order.status, bt.Order.Accepted)

        # Test order executes below stop price
        self.broker._try_execute_order(order, current_price=47000.0)
        self.assertEqual(order.status, bt.Order.Completed)

    def test_commission_calculation(self):
        """Test commission calculation."""
        # Set commission rate
        self.broker.setcommission(commission=0.001)  # 0.1%

        # Execute order
        order = self.broker.buy(size=0.1)
        self.broker._execute_order(order, price=50000.0)

        # Verify commission
        expected_commission = 0.1 * 50000.0 * 0.001  # size * price * rate
        self.assertAlmostEqual(order.executed.comm, expected_commission, places=2)

    def test_cash_and_value_tracking(self):
        """Test cash and portfolio value tracking."""
        initial_cash = self.broker.getcash()
        initial_value = self.broker.getvalue()

        # Execute buy order
        order = self.broker.buy(size=0.1)
        self.broker._execute_order(order, price=50000.0)

        # Verify cash reduction
        expected_cash = initial_cash - (0.1 * 50000.0) - order.executed.comm
        self.assertAlmostEqual(self.broker.getcash(), expected_cash, places=2)

        # Verify value remains approximately the same (minus commission)
        current_value = self.broker.getvalue()
        self.assertAlmostEqual(current_value, initial_value - order.executed.comm, places=2)
```

## Integration Testing

### Data Pipeline Integration

```python
class TestDataPipelineIntegration(unittest.TestCase):
    """Test complete data pipeline integration."""

    def test_store_to_feed_integration(self):
        """Test data flow from store to feed to strategy."""
        from cracktrader import Cerebro, CCXTStore, CCXTDataFeed
        from tests.fixtures.fake_exchange import FakeExchange

        # Create fake exchange with test data
        fake_exchange = FakeExchange()
        fake_exchange.set_ohlcv_data([
            [1704067200000, 45000, 45500, 44500, 45200, 1000],
            [1704070800000, 45200, 45800, 45100, 45600, 1200],
            [1704074400000, 45600, 46100, 45300, 45900, 1500],
        ])

        # Create components
        store = CCXTStore(exchange='fake', exchange_instance=fake_exchange)
        feed = CCXTDataFeed(store=store, symbol='BTC/USDT', ccxt_timeframe='1h')

        # Test strategy that captures data
        class DataCaptureStrategy(bt.Strategy):
            def __init__(self):
                self.data_points = []

            def next(self):
                self.data_points.append({
                    'datetime': self.data.datetime.datetime(),
                    'open': self.data.open[0],
                    'high': self.data.high[0],
                    'low': self.data.low[0],
                    'close': self.data.close[0],
                    'volume': self.data.volume[0]
                })

        # Run integration
        cerebro = Cerebro()
        cerebro.adddata(feed)
        cerebro.addstrategy(DataCaptureStrategy)

        results = cerebro.run()
        strategy = results[0]

        # Verify data flow
        self.assertEqual(len(strategy.data_points), 3)
        self.assertEqual(strategy.data_points[0]['open'], 45000)
        self.assertEqual(strategy.data_points[-1]['close'], 45900)

    def test_multi_feed_integration(self):
        """Test multiple data feeds working together."""
        from cracktrader import Cerebro, CCXTStore, CCXTDataFeed
        from tests.fixtures.fake_exchange import FakeExchange

        # Create fake exchange
        fake_exchange = FakeExchange()

        # Set up different symbols
        btc_data = [[1704067200000, 45000, 45500, 44500, 45200, 1000]]
        eth_data = [[1704067200000, 2500, 2550, 2450, 2520, 500]]

        fake_exchange.set_symbol_data('BTC/USDT', btc_data)
        fake_exchange.set_symbol_data('ETH/USDT', eth_data)

        # Create components
        store = CCXTStore(exchange='fake', exchange_instance=fake_exchange)

        btc_feed = CCXTDataFeed(store=store, symbol='BTC/USDT', ccxt_timeframe='1h')
        eth_feed = CCXTDataFeed(store=store, symbol='ETH/USDT', ccxt_timeframe='1h')

        class MultiAssetStrategy(bt.Strategy):
            def next(self):
                btc_price = self.datas[0].close[0]
                eth_price = self.datas[1].close[0]

                # Simple ratio strategy
                ratio = btc_price / eth_price
                if ratio > 18:  # BTC expensive relative to ETH
                    self.sell(data=self.datas[0])  # Sell BTC
                    self.buy(data=self.datas[1])   # Buy ETH

        # Run integration
        cerebro = Cerebro()
        cerebro.adddata(btc_feed, name='BTC')
        cerebro.adddata(eth_feed, name='ETH')
        cerebro.addstrategy(MultiAssetStrategy)

        results = cerebro.run()
        self.assertIsNotNone(results)
```

### Web API Integration

```python
class TestWebAPIIntegration(unittest.TestCase):
    """Test web API integration workflows."""

    def setUp(self):
        """Set up test client."""
        from cracktrader.web.server.main import create_app
        from fastapi.testclient import TestClient

        app = create_app()
        self.client = TestClient(app)

    def test_complete_backtest_workflow(self):
        """Test complete backtest via API."""
        # 1. Check API health
        response = self.client.get("/api/v1/health")
        self.assertEqual(response.status_code, 200)

        # 2. List available strategies
        response = self.client.get("/api/v1/strategies")
        self.assertEqual(response.status_code, 200)
        strategies = response.json()
        self.assertIn("TestStrategy", strategies["strategies"])

        # 3. Start backtest
        backtest_request = {
            "strategy_name": "TestStrategy",
            "symbols": ["BTC/USDT"],
            "timeframe": "1h",
            "initial_cash": 10000.0
        }

        response = self.client.post("/api/v1/run/backtest", json=backtest_request)
        self.assertEqual(response.status_code, 200)

        result = response.json()
        run_id = result["run_id"]
        self.assertIsNotNone(run_id)

        # 4. Check status
        response = self.client.get("/api/v1/status")
        self.assertEqual(response.status_code, 200)

        status = response.json()
        self.assertEqual(status["run_id"], run_id)
        self.assertIn(status["status"], ["starting", "running", "completed"])

        # 5. Get results (may need to wait for completion)
        import time
        max_wait = 30  # seconds
        start_time = time.time()

        while time.time() - start_time < max_wait:
            response = self.client.get("/api/v1/status")
            status = response.json()

            if status["status"] == "completed":
                break

            time.sleep(1)

        # Get final results
        response = self.client.get("/api/v1/results")
        self.assertEqual(response.status_code, 200)

        results = response.json()
        self.assertEqual(results["run_id"], run_id)
        self.assertEqual(results["status"], "completed")
```

## End-to-End Testing

### Live Exchange Testing

```python
class TestLiveExchangeIntegration(unittest.TestCase):
    """Test with real exchange connections (optional)."""

    def setUp(self):
        """Set up live exchange connection."""
        import os

        # Skip if no API credentials
        self.api_key = os.getenv('BINANCE_TESTNET_API_KEY')
        self.secret = os.getenv('BINANCE_TESTNET_SECRET')

        if not self.api_key or not self.secret:
            self.skipTest("Live exchange credentials not available")

    def test_live_data_feed(self):
        """Test live data feed with real exchange."""
        from cracktrader import CCXTStore, CCXTDataFeed

        store = CCXTStore(
            exchange_id='binance',
            config={
                'apiKey': self.api_key,
                'secret': self.secret,
                'sandbox': True  # Use testnet
            }
        )

        # Test market data access
        markets = store.exchange.load_markets()
        self.assertIn('BTC/USDT', markets)

        # Test historical data
        feed = CCXTDataFeed(
            store=store,
            symbol='BTC/USDT',
            timeframe='1h',
            start_date='2024-01-01T00:00:00Z',
            end_date='2024-01-02T00:00:00Z'
        )

        data = feed._load_data()
        self.assertIsNotNone(data)
        self.assertGreater(len(data), 0)

    def test_live_order_execution(self):
        """Test order execution on testnet."""
        from cracktrader import Cerebro, CCXTStore, CCXTDataFeed
        from cracktrader.broker import CCXTLiveBroker

        store = CCXTStore(
            exchange_id='binance',
            config={
                'apiKey': self.api_key,
                'secret': self.secret,
                'sandbox': True
            }
        )

        # Simple test strategy
        class TestOrderStrategy(bt.Strategy):
            def __init__(self):
                self.order_placed = False

            def next(self):
                if not self.order_placed and len(self.data) > 5:
                    # Place small test order
                    self.buy(size=0.001)  # Small BTC amount
                    self.order_placed = True

        # Create live trading setup
        cerebro = Cerebro()

        feed = CCXTDataFeed(store=store, symbol='BTC/USDT', timeframe='1m')
        cerebro.adddata(feed)

        broker = CCXTLiveBroker(store=store, mode='paper')
        cerebro.broker = broker

        cerebro.addstrategy(TestOrderStrategy)

        # Run for limited time
        import signal
        import time

        def timeout_handler(signum, frame):
            raise TimeoutError("Test timeout")

        signal.signal(signal.SIGALRM, timeout_handler)
        signal.alarm(30)  # 30 second timeout

        try:
            results = cerebro.run()
            self.assertIsNotNone(results)
        except TimeoutError:
            pass  # Expected timeout
        finally:
            signal.alarm(0)
```

### Performance Testing

```python
class TestPerformance(unittest.TestCase):
    """Test performance characteristics."""

    def test_backtest_performance(self):
        """Test backtest execution time."""
        import time
        from cracktrader import Cerebro, CCXTStore, CCXTDataFeed
        from tests.fixtures.fake_exchange import FakeExchange

        # Create large dataset
        fake_exchange = FakeExchange()
        large_dataset = []

        # Generate 1000 candles
        base_time = 1704067200000  # 2024-01-01
        for i in range(1000):
            timestamp = base_time + (i * 3600000)  # 1 hour intervals
            ohlcv = [timestamp, 45000, 45500, 44500, 45200, 1000]
            large_dataset.append(ohlcv)

        fake_exchange.set_ohlcv_data(large_dataset)

        # Set up backtest
        store = CCXTStore(exchange_id='fake', exchange=fake_exchange)
        feed = CCXTDataFeed(store=store, symbol='BTC/USDT', timeframe='1h')

        cerebro = Cerebro()
        cerebro.adddata(feed)
        cerebro.addstrategy(bt.strategies.BuyAndHold)

        # Measure execution time
        start_time = time.time()
        results = cerebro.run()
        execution_time = time.time() - start_time

        # Performance assertions
        self.assertLess(execution_time, 10.0)  # Should complete in under 10 seconds
        self.assertIsNotNone(results)

    def test_memory_usage(self):
        """Test memory usage during backtests."""
        import psutil
        import os

        process = psutil.Process(os.getpid())
        initial_memory = process.memory_info().rss / 1024 / 1024  # MB

        # Run memory-intensive backtest
        from cracktrader import Cerebro, CCXTStore, CCXTDataFeed
        from tests.fixtures.fake_exchange import FakeExchange

        # Large dataset with multiple feeds
        fake_exchange = FakeExchange()
        dataset = []
        for i in range(5000):  # 5000 candles
            timestamp = 1704067200000 + (i * 3600000)
            ohlcv = [timestamp, 45000, 45500, 44500, 45200, 1000]
            dataset.append(ohlcv)

        fake_exchange.set_ohlcv_data(dataset)

        cerebro = Cerebro()
        store = CCXTStore(exchange_id='fake', exchange=fake_exchange)

        # Add multiple feeds
        for symbol in ['BTC/USDT', 'ETH/USDT', 'ADA/USDT']:
            feed = CCXTDataFeed(store=store, symbol=symbol, timeframe='1h')
            cerebro.adddata(feed)

        cerebro.addstrategy(bt.strategies.BuyAndHold)

        # Run backtest
        results = cerebro.run()

        # Check memory usage
        final_memory = process.memory_info().rss / 1024 / 1024  # MB
        memory_increase = final_memory - initial_memory

        # Memory should not increase excessively (less than 500MB for this test)
        self.assertLess(memory_increase, 500)
```

## Test Fixtures and Utilities

### Fake Exchange Implementation

```python
# tests/fixtures/fake_exchange.py
class FakeExchange:
    """Mock exchange for testing."""

    def __init__(self):
        self.id = 'fake'
        self.name = 'Fake Exchange'
        self.markets = {
            'BTC/USDT': {'symbol': 'BTC/USDT', 'base': 'BTC', 'quote': 'USDT'},
            'ETH/USDT': {'symbol': 'ETH/USDT', 'base': 'ETH', 'quote': 'USDT'},
        }
        self.timeframes = {'1m': '1m', '5m': '5m', '1h': '1h', '1d': '1d'}
        self._ohlcv_data = {}
        self._symbol_data = {}

        # Default realistic data
        self._set_default_data()

    def _set_default_data(self):
        """Set default realistic OHLCV data."""
        base_time = 1704067200000  # 2024-01-01

        for i, symbol in enumerate(['BTC/USDT', 'ETH/USDT']):
            data = []
            base_price = 45000 if 'BTC' in symbol else 2500

            for j in range(100):  # 100 candles
                timestamp = base_time + (j * 3600000)  # 1 hour intervals

                # Generate realistic OHLCV
                open_price = base_price + (j * 10) + (i * 1000)
                high_price = open_price + 200
                low_price = open_price - 150
                close_price = open_price + 50
                volume = 1000 + (j * 10)

                data.append([timestamp, open_price, high_price, low_price, close_price, volume])

            self._symbol_data[symbol] = data

    def set_ohlcv_data(self, data):
        """Set OHLCV data for default symbol."""
        self._ohlcv_data['BTC/USDT'] = data

    def set_symbol_data(self, symbol, data):
        """Set OHLCV data for specific symbol."""
        self._symbol_data[symbol] = data

    def load_markets(self):
        """Return available markets."""
        return self.markets

    def fetch_ohlcv(self, symbol, timeframe, since=None, limit=None):
        """Fetch OHLCV data."""
        # Check specific symbol data first
        if symbol in self._symbol_data:
            data = self._symbol_data[symbol]
        elif symbol in self._ohlcv_data:
            data = self._ohlcv_data[symbol]
        else:
            # Return default data
            data = self._symbol_data.get('BTC/USDT', [])

        # Apply time filtering
        if since:
            data = [candle for candle in data if candle[0] >= since]

        # Apply limit
        if limit:
            data = data[:limit]

        return data

    def create_order(self, symbol, type, side, amount, price=None):
        """Create mock order."""
        import uuid

        order = {
            'id': str(uuid.uuid4()),
            'symbol': symbol,
            'type': type,
            'side': side,
            'amount': amount,
            'price': price,
            'status': 'open',
            'filled': 0,
            'remaining': amount,
            'timestamp': 1704067200000,
        }

        # Simulate immediate execution for market orders
        if type == 'market':
            order['status'] = 'closed'
            order['filled'] = amount
            order['remaining'] = 0

            # Use realistic prices
            if 'BTC' in symbol:
                order['price'] = 45000
            elif 'ETH' in symbol:
                order['price'] = 2500

        return order

    def fetch_balance(self):
        """Return mock balance."""
        return {
            'USDT': {'free': 10000, 'used': 0, 'total': 10000},
            'BTC': {'free': 0, 'used': 0, 'total': 0},
            'ETH': {'free': 0, 'used': 0, 'total': 0},
        }
```

### Test Data Generators

```python
# tests/fixtures/data_generators.py
import pandas as pd
import numpy as np
from datetime import datetime, timedelta

def generate_realistic_ohlcv(
    symbol='BTC/USDT',
    start_date='2024-01-01',
    end_date='2024-01-31',
    timeframe='1h',
    initial_price=45000,
    volatility=0.02
):
    """Generate realistic OHLCV data for testing."""

    # Create date range
    freq_map = {'1m': '1T', '5m': '5T', '1h': '1H', '1d': '1D'}
    freq = freq_map.get(timeframe, '1H')

    dates = pd.date_range(start=start_date, end=end_date, freq=freq)

    # Generate price series using random walk
    returns = np.random.normal(0, volatility, len(dates))
    prices = [initial_price]

    for ret in returns[1:]:
        new_price = prices[-1] * (1 + ret)
        prices.append(max(new_price, 1))  # Prevent negative prices

    # Generate OHLCV data
    ohlcv_data = []

    for i, (date, price) in enumerate(zip(dates, prices)):
        timestamp = int(date.timestamp() * 1000)

        # Generate OHLC from close price
        close = price
        open_price = prices[i-1] if i > 0 else price

        # Add some intrabar movement
        high = max(open_price, close) * (1 + np.random.uniform(0, 0.01))
        low = min(open_price, close) * (1 - np.random.uniform(0, 0.01))

        # Generate volume (correlated with volatility)
        base_volume = 1000
        volume_multiplier = 1 + abs(returns[i]) * 10
        volume = base_volume * volume_multiplier

        ohlcv_data.append([timestamp, open_price, high, low, close, volume])

    return ohlcv_data

def generate_market_scenarios():
    """Generate different market scenarios for testing."""
    scenarios = {}

    # Bull market
    scenarios['bull_market'] = generate_realistic_ohlcv(
        initial_price=45000,
        volatility=0.015,
        start_date='2024-01-01',
        end_date='2024-03-01'
    )

    # Bear market
    scenarios['bear_market'] = generate_realistic_ohlcv(
        initial_price=45000,
        volatility=0.025,
        start_date='2024-01-01',
        end_date='2024-03-01'
    )

    # Sideways market
    scenarios['sideways_market'] = generate_realistic_ohlcv(
        initial_price=45000,
        volatility=0.01,
        start_date='2024-01-01',
        end_date='2024-03-01'
    )

    # High volatility
    scenarios['high_volatility'] = generate_realistic_ohlcv(
        initial_price=45000,
        volatility=0.05,
        start_date='2024-01-01',
        end_date='2024-03-01'
    )

    return scenarios
```

## Best Practices

### Test Organization

```python
# Organize tests by behavior, not implementation
class TestOrderManagement:
    """Test order management behavior."""

    def test_market_order_immediate_execution(self):
        """Market orders should execute immediately at current price."""
        pass

    def test_limit_order_price_improvement(self):
        """Limit orders should execute when price improves."""
        pass

    def test_stop_order_activation(self):
        """Stop orders should activate when stop price is hit."""
        pass

class TestRiskManagement:
    """Test risk management behavior."""

    def test_position_size_limits(self):
        """Position sizes should respect maximum limits."""
        pass

    def test_stop_loss_execution(self):
        """Stop losses should execute when triggered."""
        pass

class TestDataIntegrity:
    """Test data quality and integrity."""

    def test_no_missing_timestamps(self):
        """Data feeds should not have missing timestamps."""
        pass

    def test_valid_ohlc_relationships(self):
        """OHLC data should maintain valid relationships."""
        pass
```

### Mocking Strategy

```python
# Mock external dependencies, not internal logic
class TestStrategy:
    @patch('cracktrader.store.CCXTStore')  # Mock external exchange
    def test_strategy_with_mocked_exchange(self, mock_store):
        # Set up mock behavior
        mock_store.return_value.fetch_ohlcv.return_value = [
            [1704067200000, 45000, 45500, 44500, 45200, 1000]
        ]

        # Test strategy behavior
        strategy = MyStrategy()
        # ... test logic

    def test_strategy_decision_logic(self):
        # Test strategy logic directly without mocking
        strategy = MyStrategy()
        strategy.sma = Mock(return_value=45000)
        strategy.data.close = Mock(return_value=45500)

        result = strategy._should_buy()
        self.assertTrue(result)
```

### Performance Testing

```python
def test_backtest_performance():
    """Test backtest performance characteristics."""
    import time
    import memory_profiler

    @memory_profiler.profile
    def run_backtest():
        cerebro = Cerebro()
        # ... setup backtest
        return cerebro.run()

    # Measure time
    start_time = time.time()
    results = run_backtest()
    execution_time = time.time() - start_time

    # Assert performance requirements
    assert execution_time < 30.0  # Should complete in under 30 seconds
    assert results is not None
```

## Continuous Integration

### GitHub Actions Configuration

```yaml
# .github/workflows/test.yml
name: Test Suite

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: [3.9, 3.10, 3.11, 3.12]

    steps:
    - uses: actions/checkout@v3

    - name: Set up Python ${{ matrix.python-version }}
      uses: actions/setup-python@v4
      with:
        python-version: ${{ matrix.python-version }}

    - name: Install dependencies
      run: |
        python -m pip install --upgrade pip
        pip install -e .'[dev]'

    - name: Run unit tests
      run: |
        pytest tests/unit/ -v --cov=cracktrader --cov-report=xml

    - name: Run integration tests
      run: |
        pytest tests/integration/ -v

    - name: Upload coverage
      uses: codecov/codecov-action@v3
      with:
        file: ./coverage.xml

    - name: Run end-to-end tests (optional)
      if: ${{ github.event_name == 'push' && github.ref == 'refs/heads/main' }}
      env:
        BINANCE_TESTNET_API_KEY: ${{ secrets.BINANCE_TESTNET_API_KEY }}
        BINANCE_TESTNET_SECRET: ${{ secrets.BINANCE_TESTNET_SECRET }}
      run: |
        pytest tests/e2e/ -v --tb=short
```

### Coverage Requirements

```python
# pyproject.toml
[tool.coverage.run]
source = ["src/cracktrader"]
omit = [
    "*/tests/*",
    "*/test_*",
    "*/conftest.py",
]

[tool.coverage.report]
exclude_lines = [
    "pragma: no cover",
    "def __repr__",
    "raise AssertionError",
    "raise NotImplementedError",
]

[tool.pytest.ini_options]
testpaths = ["tests"]
python_files = ["test_*.py"]
python_classes = ["Test*"]
python_functions = ["test_*"]
addopts = [
    "--strict-markers",
    "--strict-config",
    "--cov-fail-under=80",
]
```

---

**Next Steps:**
- [Strategy Development](strategy_guide.md) - Writing testable strategies
- [Broker Integration](brokers.md) - Testing order execution
- [Advanced Configuration](advanced.md) - Performance testing and optimization
