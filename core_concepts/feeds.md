# Data Feeds

Cracktrader's data feeds system provides unified access to real-time and historical market data from 100+ cryptocurrency exchanges through CCXT integration. This guide covers data feed configuration, usage patterns, and optimization techniques.

## Overview

### Data Feed Architecture

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Strategies    │    │   Backtrader    │    │   Data Sources  │
│                 │◄──►│   Data API      │◄──►│                 │
└─────────────────┘    └─────────────────┘    │  ┌───────────┐  │
                                              │  │   CCXT    │  │
┌─────────────────┐    ┌─────────────────┐    │  │Exchange   │  │
│ CCXTDataFeed    │◄──►│   CCXTStore     │◄──►│  │  APIs     │  │
│                 │    │                 │    │  └───────────┘  │
└─────────────────┘    └─────────────────┘    │                 │
                                              │  ┌───────────┐  │
┌─────────────────┐    ┌─────────────────┐    │  │WebSocket  │  │
│StreamingFeed    │◄──►│ WebSocket Mgr   │◄──►│  │ Streams   │  │
│                 │    │                 │    │  └───────────┘  │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

### Key Components

- **`CCXTDataFeed`** - Main data feed for OHLCV candles
- **`CCXTStore`** - Exchange connectivity and data management
- **`StreamingFeed`** - Real-time WebSocket data streams
- **`OHLCVCandle`** - Standardized data format
- **Data Subsystems** - Specialized data processing modules

## Basic Usage

### Historical Data Feed

```python
from cracktrader import Cerebro, CCXTStore, CCXTDataFeed

# Create store connection
store = CCXTStore(exchange='binance')

# Create historical data feed
feed = CCXTDataFeed(
    store=store,
    symbol='BTC/USDT',
    ccxt_timeframe='1h',
    historical_limit=2000
)

# Add to strategy
cerebro = Cerebro()
cerebro.adddata(feed)
cerebro.addstrategy(MyStrategy)
cerebro.run()
```

### Live Data Feed

```python
# Live streaming data
feed = CCXTDataFeed(
    store=store,
    symbol='BTC/USDT',
    ccxt_timeframe='1m',
    live=True,
    historical_limit=1  # Seed with at least one candle
)

# Strategy receives real-time updates
class LiveStrategy(bt.Strategy):
    def next(self):
        # Called for each new candle
        current_time = self.data.datetime.datetime()
        current_price = self.data.close[0]
        print(f"{current_time}: BTC price is ${current_price:,.2f}")
```

### Paper Trading

```python
# Paper trading with sandbox data
store = CCXTStore(exchange='binance', sandbox=True, config={'apiKey': '...', 'secret': '...'})
feed = CCXTDataFeed(store=store, symbol='BTC/USDT', ccxt_timeframe='5m')
```

## Data Feed Configuration

### CCXTDataFeed Parameters

```python
feed = CCXTDataFeed(
    store=store,                    # CCXTStore instance
    symbol='BTC/USDT',             # Trading pair
    timeframe='1h',                # Candle timeframe
    start_date='2024-01-01T00:00:00Z',  # ISO format (optional)
    end_date='2024-03-01T23:59:59Z',    # ISO format (optional)

    # Optional parameters
    name='BTC_1h',                 # Data feed name
    compression=1,                 # Time compression (1 = no compression)
    tz='UTC',                      # Timezone
    historical=True,               # Fetch historical data
    live=False,                    # Enable live streaming

    # Backtrader data parameters
    dataname=None,                 # Data source name
    fromdate=None,                 # Alternative date format
    todate=None,                   # Alternative date format
    sessionstart=None,             # Session start time
    sessionend=None,               # Session end time
)
```

### Supported Timeframes

```python
# Common timeframes
TIMEFRAMES = {
    '1m': '1 minute',
    '3m': '3 minutes',
    '5m': '5 minutes',
    '15m': '15 minutes',
    '30m': '30 minutes',
    '1h': '1 hour',
    '2h': '2 hours',
    '4h': '4 hours',
    '6h': '6 hours',
    '8h': '8 hours',
    '12h': '12 hours',
    '1d': '1 day',
    '3d': '3 days',
    '1w': '1 week',
    '1M': '1 month'
}

# Check exchange support
store = CCXTStore(exchange_id='binance')
supported_timeframes = store.exchange.timeframes
print("Supported timeframes:", list(supported_timeframes.keys()))
```

### Symbol Formats

```python
# Correct symbol formats for different exchanges

# Binance
'BTC/USDT', 'ETH/USDT', 'ADA/USDT'

# Coinbase Pro
'BTC/USD', 'ETH/USD', 'LTC/USD'

# Kraken
'BTC/USD', 'ETH/USD', 'XRP/USD'

# Check available symbols
store = CCXTStore(exchange_id='binance')
markets = store.exchange.load_markets()
symbols = list(markets.keys())
print(f"Available symbols: {len(symbols)}")
print("Popular symbols:", [s for s in symbols if 'USDT' in s][:10])
```

## Multi-Asset Data Feeds

### Multiple Symbols

```python
def create_multi_asset_cerebro():
    """Create cerebro with multiple cryptocurrency feeds."""
    cerebro = Cerebro()
    store = CCXTStore(exchange_id='binance')

    symbols = ['BTC/USDT', 'ETH/USDT', 'ADA/USDT', 'DOT/USDT']

    for symbol in symbols:
        feed = CCXTDataFeed(
            store=store,
            symbol=symbol,
            timeframe='1h',
            start_date='2024-01-01T00:00:00Z',
            end_date='2024-03-01T23:59:59Z'
        )
        cerebro.adddata(feed, name=symbol.replace('/', '_'))

    return cerebro

# Use in strategy
class MultiAssetStrategy(bt.Strategy):
    def next(self):
        # Access different assets
        btc_price = self.datas[0].close[0]  # BTC/USDT
        eth_price = self.datas[1].close[0]  # ETH/USDT
        ada_price = self.datas[2].close[0]  # ADA/USDT

        # Calculate BTC dominance
        btc_market_cap = btc_price * 19_000_000  # Approximate supply
        eth_market_cap = eth_price * 120_000_000

        dominance = btc_market_cap / (btc_market_cap + eth_market_cap)

        if dominance > 0.7:  # High BTC dominance
            self.buy(data=self.datas[0])  # Buy BTC
```

### Multiple Timeframes

```python
def create_multi_timeframe_cerebro():
    """Create cerebro with multiple timeframes for the same symbol."""
    cerebro = Cerebro()
    store = CCXTStore(exchange_id='binance')

    # Primary timeframe (for trading)
    feed_1h = CCXTDataFeed(
        store=store,
        symbol='BTC/USDT',
        timeframe='1h',
        start_date='2024-01-01T00:00:00Z',
        end_date='2024-03-01T23:59:59Z'
    )
    cerebro.adddata(feed_1h, name='BTC_1h')

    # Higher timeframe (for trend)
    feed_4h = CCXTDataFeed(
        store=store,
        symbol='BTC/USDT',
        timeframe='4h',
        start_date='2024-01-01T00:00:00Z',
        end_date='2024-03-01T23:59:59Z'
    )
    cerebro.adddata(feed_4h, name='BTC_4h')

    # Even higher timeframe (for context)
    feed_1d = CCXTDataFeed(
        store=store,
        symbol='BTC/USDT',
        timeframe='1d',
        start_date='2024-01-01T00:00:00Z',
        end_date='2024-03-01T23:59:59Z'
    )
    cerebro.adddata(feed_1d, name='BTC_1d')

    return cerebro

class MultiTimeframeStrategy(bt.Strategy):
    def next(self):
        # 1h data (primary)
        current_price = self.datas[0].close[0]

        # 4h trend
        if len(self.datas[1]) > 0:
            trend_4h = self.datas[1].close[0] > self.datas[1].close[-5]
        else:
            trend_4h = True

        # 1d context
        if len(self.datas[2]) > 0:
            context_1d = self.datas[2].close[0] > self.datas[2].close[-10]
        else:
            context_1d = True

        # Trade only when all timeframes align
        if trend_4h and context_1d and not self.position:
            self.buy()
```

### Multiple Exchanges

```python
def create_multi_exchange_cerebro():
    """Compare prices across exchanges for arbitrage."""
    cerebro = Cerebro()

    # Binance data
    binance_store = CCXTStore(exchange_id='binance')
    binance_feed = CCXTDataFeed(
        store=binance_store,
        symbol='BTC/USDT',
        timeframe='1m'
    )
    cerebro.adddata(binance_feed, name='binance_btc')

    # Coinbase data
    coinbase_store = CCXTStore(exchange_id='coinbase')
    coinbase_feed = CCXTDataFeed(
        store=coinbase_store,
        symbol='BTC/USD',
        timeframe='1m'
    )
    cerebro.adddata(coinbase_feed, name='coinbase_btc')

    return cerebro

class ArbitrageStrategy(bt.Strategy):
    def next(self):
        binance_price = self.datas[0].close[0]  # BTC/USDT
        coinbase_price = self.datas[1].close[0]  # BTC/USD

        # Assume USDT ≈ USD for simplicity
        spread = (coinbase_price - binance_price) / binance_price

        if spread > 0.005:  # 0.5% arbitrage opportunity
            # Buy on Binance, sell on Coinbase
            self.buy(data=self.datas[0])
            self.sell(data=self.datas[1])
```

## Real-time Streaming

### WebSocket Data Streams

```python
from cracktrader.store.streaming_feed import StreamingFeed

class StreamingStrategy(bt.Strategy):
    """Strategy using real-time WebSocket data."""

    def __init__(self):
        # Access streaming feed
        self.streaming = self.data._streaming if hasattr(self.data, '_streaming') else None

        if self.streaming:
            # Subscribe to additional data streams
            self.streaming.subscribe_trades('BTC/USDT')
            self.streaming.subscribe_orderbook('BTC/USDT', limit=10)
            self.streaming.subscribe_ticker('BTC/USDT')

    def next(self):
        # Standard OHLCV data
        price = self.data.close[0]
        volume = self.data.volume[0]

        # Real-time trade data
        if self.streaming:
            latest_trades = self.streaming.get_recent_trades('BTC/USDT', limit=10)
            orderbook = self.streaming.get_orderbook('BTC/USDT')
            ticker = self.streaming.get_ticker('BTC/USDT')

            # Use real-time data for decision making
            if self.analyze_order_flow(latest_trades, orderbook):
                self.buy()

    def analyze_order_flow(self, trades, orderbook):
        """Analyze real-time order flow for trading signals."""
        if not trades or not orderbook:
            return False

        # Calculate buy/sell pressure
        recent_volume = sum(trade['amount'] for trade in trades[-10:])
        buy_volume = sum(trade['amount'] for trade in trades[-10:]
                        if trade['side'] == 'buy')

        buy_pressure = buy_volume / recent_volume if recent_volume > 0 else 0.5

        # Check orderbook imbalance
        bid_depth = sum(order[1] for order in orderbook['bids'][:5])
        ask_depth = sum(order[1] for order in orderbook['asks'][:5])

        order_imbalance = bid_depth / (bid_depth + ask_depth)

        # Signal when both metrics are bullish
        return buy_pressure > 0.6 and order_imbalance > 0.6

# Setup streaming data
store = CCXTStore(exchange_id='binance', streaming=True)
feed = CCXTDataFeed(store=store, symbol='BTC/USDT', timeframe='1m', live=True)

cerebro = Cerebro()
cerebro.adddata(feed)
cerebro.addstrategy(StreamingStrategy)
cerebro.run()
```

### Data Rate Limiting

```python
# Configure rate limits to avoid API restrictions
store = CCXTStore(
    exchange_id='binance',
    config={
        'rateLimit': 1200,  # Milliseconds between requests
        'enableRateLimit': True,
    }
)

# Batch data requests
feed = CCXTDataFeed(
    store=store,
    symbol='BTC/USDT',
    timeframe='1h',
    batch_size=1000,  # Fetch 1000 candles per request
)
```

## Data Quality and Validation

### Data Validation

```python
class ValidatedDataFeed(CCXTDataFeed):
    """Data feed with built-in validation."""

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.validation_enabled = True
        self.max_gap_minutes = 60  # Maximum acceptable gap

    def _load_data(self):
        """Override to add validation."""
        data = super()._load_data()

        if self.validation_enabled:
            data = self._validate_data(data)

        return data

    def _validate_data(self, df):
        """Validate OHLCV data quality."""
        import pandas as pd
        import numpy as np

        if df.empty:
            return df

        # Check for missing timestamps
        expected_freq = self._get_expected_frequency()
        df_reindexed = df.reindex(
            pd.date_range(
                start=df.index[0],
                end=df.index[-1],
                freq=expected_freq
            )
        )

        missing_count = df_reindexed.isnull().sum().sum()
        if missing_count > 0:
            logger.warning(f"Found {missing_count} missing data points")

            # Forward fill missing values
            df_reindexed = df_reindexed.fillna(method='ffill')

        # Validate OHLC relationships
        invalid_ohlc = (
            (df_reindexed['high'] < df_reindexed['low']) |
            (df_reindexed['high'] < df_reindexed['open']) |
            (df_reindexed['high'] < df_reindexed['close']) |
            (df_reindexed['low'] > df_reindexed['open']) |
            (df_reindexed['low'] > df_reindexed['close'])
        )

        if invalid_ohlc.any():
            logger.error(f"Found {invalid_ohlc.sum()} invalid OHLC relationships")
            # Fix invalid data
            df_reindexed.loc[invalid_ohlc, 'high'] = df_reindexed.loc[invalid_ohlc, ['open', 'close']].max(axis=1)
            df_reindexed.loc[invalid_ohlc, 'low'] = df_reindexed.loc[invalid_ohlc, ['open', 'close']].min(axis=1)

        # Check for extreme price movements (potential errors)
        price_changes = df_reindexed['close'].pct_change()
        extreme_moves = abs(price_changes) > 0.2  # 20% moves

        if extreme_moves.any():
            logger.warning(f"Found {extreme_moves.sum()} extreme price movements > 20%")

        return df_reindexed

    def _get_expected_frequency(self):
        """Get pandas frequency string for timeframe."""
        timeframe_map = {
            '1m': '1T', '5m': '5T', '15m': '15T', '30m': '30T',
            '1h': '1H', '4h': '4H', '1d': '1D', '1w': '1W'
        }
        return timeframe_map.get(self.timeframe, '1H')

# Usage
validated_feed = ValidatedDataFeed(
    store=store,
    symbol='BTC/USDT',
    timeframe='1h'
)
```

### Data Caching

```python
# Enable caching for better performance
store = CCXTStore(
    exchange_id='binance',
    cache_enabled=True,
    cache_ttl=3600,  # 1 hour cache
    cache_path='./data_cache'
)

# Persistent local storage
store = CCXTStore(
    exchange_id='binance',
    local_storage=True,
    storage_path='./historical_data',
    storage_format='parquet'  # Fast columnar format
)
```

## Custom Data Feeds

### Custom OHLCV Feed

```python
import pandas as pd
from cracktrader.feeds import CustomDataFeed

class CSVCryptoFeed(CustomDataFeed):
    """Load cryptocurrency data from CSV files."""

    params = (
        ('csv_file', None),
        ('datetime_col', 'timestamp'),
        ('symbol_col', 'symbol'),
        ('filter_symbol', None),
    )

    def _load_data(self):
        """Load data from CSV file."""
        if not self.params.csv_file:
            raise ValueError("csv_file parameter required")

        # Load CSV
        df = pd.read_csv(self.params.csv_file)

        # Filter by symbol if specified
        if self.params.filter_symbol:
            df = df[df[self.params.symbol_col] == self.params.filter_symbol]

        # Convert timestamp
        df[self.params.datetime_col] = pd.to_datetime(df[self.params.datetime_col])
        df.set_index(self.params.datetime_col, inplace=True)

        # Ensure required columns
        required_cols = ['open', 'high', 'low', 'close', 'volume']
        missing_cols = [col for col in required_cols if col not in df.columns]
        if missing_cols:
            raise ValueError(f"Missing required columns: {missing_cols}")

        return df[required_cols]

# Usage
csv_feed = CSVCryptoFeed(
    csv_file='data/btc_historical.csv',
    filter_symbol='BTC',
    timeframe='1h'
)
```

### Database Data Feed

```python
import sqlite3
import pandas as pd

class DatabaseFeed(CustomDataFeed):
    """Load data from SQLite database."""

    params = (
        ('db_path', 'crypto_data.db'),
        ('table_name', 'ohlcv'),
        ('symbol_filter', None),
        ('query', None),
    )

    def _load_data(self):
        """Load data from database."""
        conn = sqlite3.connect(self.params.db_path)

        if self.params.query:
            query = self.params.query
        else:
            query = f"""
            SELECT timestamp, open, high, low, close, volume
            FROM {self.params.table_name}
            WHERE symbol = '{self.params.symbol_filter}'
            ORDER BY timestamp
            """

        df = pd.read_sql_query(query, conn, parse_dates=['timestamp'])
        conn.close()

        df.set_index('timestamp', inplace=True)
        return df

# Usage
db_feed = DatabaseFeed(
    db_path='crypto_data.db',
    symbol_filter='BTC/USDT',
    timeframe='1h'
)
```

### API Data Feed

```python
import requests
import pandas as pd

class CustomAPIFeed(CustomDataFeed):
    """Load data from custom API."""

    params = (
        ('api_url', None),
        ('api_key', None),
        ('symbol', None),
        ('headers', {}),
    )

    def _load_data(self):
        """Load data from API."""
        headers = {
            'Authorization': f'Bearer {self.params.api_key}',
            **self.params.headers
        }

        params = {
            'symbol': self.params.symbol,
            'timeframe': self.timeframe,
            'start': self.start_date,
            'end': self.end_date
        }

        response = requests.get(
            self.params.api_url,
            headers=headers,
            params=params
        )
        response.raise_for_status()

        data = response.json()

        # Convert to DataFrame
        df = pd.DataFrame(data['ohlcv'])
        df['timestamp'] = pd.to_datetime(df['timestamp'])
        df.set_index('timestamp', inplace=True)

        return df[['open', 'high', 'low', 'close', 'volume']]

# Usage
api_feed = CustomAPIFeed(
    api_url='https://api.example.com/ohlcv',
    api_key='your_api_key',
    symbol='BTC/USDT',
    timeframe='1h'
)
```

## Performance Optimization

### Data Loading Optimization

```python
# 1. Use appropriate timeframes
# Avoid very low timeframes for long backtests
feed = CCXTDataFeed(store=store, symbol='BTC/USDT', timeframe='1h')  # Not '1m'

# 2. Limit date ranges
feed = CCXTDataFeed(
    store=store,
    symbol='BTC/USDT',
    timeframe='1h',
    start_date='2024-01-01T00:00:00Z',  # Last 3 months
    end_date='2024-03-01T23:59:59Z'     # Not years of data
)

# 3. Use data compression
feed = CCXTDataFeed(
    store=store,
    symbol='BTC/USDT',
    timeframe='1h',
    compression=4  # 4-hour compressed data
)

# 4. Enable caching
store = CCXTStore(
    exchange_id='binance',
    cache_enabled=True,
    cache_ttl=3600
)
```

### Memory Management

```python
class MemoryEfficientStrategy(bt.Strategy):
    """Strategy with memory-efficient data handling."""

    def __init__(self):
        # Limit indicator lookback
        self.sma = bt.indicators.SMA(period=20)
        self.sma.plotinfo.plot = False  # Disable plotting to save memory

        # Use minimal data history
        self.data.plotinfo.plot = False

    def next(self):
        # Don't store unnecessary data
        current_price = self.data.close[0]

        # Clear old data periodically
        if len(self.data) % 1000 == 0:
            self._cleanup_old_data()

    def _cleanup_old_data(self):
        """Clean up old data to manage memory."""
        # This is handled automatically by Backtrader
        # but you can implement custom cleanup if needed
        pass
```

### Concurrent Data Loading

```python
import asyncio
import concurrent.futures

async def load_multiple_feeds_async():
    """Load multiple data feeds concurrently."""

    def create_feed(symbol):
        store = CCXTStore(exchange_id='binance')
        return CCXTDataFeed(
            store=store,
            symbol=symbol,
            timeframe='1h',
            start_date='2024-01-01T00:00:00Z',
            end_date='2024-03-01T23:59:59Z'
        )

    symbols = ['BTC/USDT', 'ETH/USDT', 'ADA/USDT', 'DOT/USDT']

    # Load feeds concurrently
    with concurrent.futures.ThreadPoolExecutor(max_workers=4) as executor:
        loop = asyncio.get_event_loop()
        futures = [
            loop.run_in_executor(executor, create_feed, symbol)
            for symbol in symbols
        ]
        feeds = await asyncio.gather(*futures)

    return feeds

# Usage
feeds = asyncio.run(load_multiple_feeds_async())
for feed in feeds:
    cerebro.adddata(feed)
```

## Troubleshooting

### Common Data Issues

```python
# Issue: No data returned
feed = CCXTDataFeed(store=store, symbol='BTC/USDT', timeframe='1h')

# Solutions:
# 1. Check symbol format
store.exchange.load_markets()
valid_symbols = list(store.exchange.markets.keys())
print('BTC/USDT' in valid_symbols)

# 2. Check timeframe support
supported_tf = store.exchange.timeframes
print('1h' in supported_tf)

# 3. Verify date range
# Some exchanges have limited history
feed = CCXTDataFeed(
    store=store,
    symbol='BTC/USDT',
    timeframe='1h',
    start_date='2024-02-01T00:00:00Z',  # More recent
    end_date='2024-02-28T23:59:59Z'
)
```

### Rate Limiting

```python
# Issue: Rate limit exceeded
# Solution: Configure rate limiting
store = CCXTStore(
    exchange_id='binance',
    config={
        'rateLimit': 2000,      # 2 seconds between requests
        'enableRateLimit': True,
    }
)

# Or use caching
store = CCXTStore(
    exchange_id='binance',
    cache_enabled=True,
    cache_ttl=1800  # 30 minutes
)
```

### Data Quality Issues

```python
# Issue: Missing or invalid data
# Solution: Add validation and cleaning
class CleanedDataFeed(CCXTDataFeed):
    def _load_data(self):
        df = super()._load_data()

        # Remove outliers
        df = df[df['close'] > 0]
        df = df[df['volume'] >= 0]

        # Fill gaps
        df = df.resample('1H').ffill()

        return df

# Usage
feed = CleanedDataFeed(store=store, symbol='BTC/USDT', timeframe='1h')
```

---

**Next Steps:**
- [Broker Integration](brokers.md) - Order execution and management
- [Strategy Development](strategy_guide.md) - Using data feeds in strategies
- [Advanced Configuration](advanced.md) - Performance tuning and optimization
