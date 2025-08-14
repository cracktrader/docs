# Core Classes Reference

Complete API reference for Cracktrader's core classes and components.

## Store Classes

### CCXTStore

The central connection hub to cryptocurrency exchanges.

```python
class CCXTStore:
    """Central store for exchange connectivity and data management."""
    
    def __init__(self, exchange: str, config: dict = None):
        """
        Initialize store with exchange connection.
        
        Args:
            exchange: Exchange name (e.g., 'binance', 'coinbase')
            config: Exchange configuration including API keys
        """
```

#### Methods

**Connection Management**
```python
def connect() -> bool:
    """Establish connection to exchange."""
    
def disconnect() -> None:
    """Close connection to exchange."""
    
def is_connected() -> bool:
    """Check if connected to exchange."""
```

**Market Data**
```python
def get_ohlcv(symbol: str, timeframe: str, since: int = None, limit: int = None) -> List[List]:
    """
    Fetch OHLCV data.
    
    Args:
        symbol: Trading pair (e.g., 'BTC/USDT')
        timeframe: Candle timeframe ('1m', '5m', '1h', '1d')
        since: Unix timestamp to start from
        limit: Maximum number of candles
        
    Returns:
        List of [timestamp, open, high, low, close, volume]
    """
    
def get_ticker(symbol: str) -> dict:
    """Get current ticker information."""
    
def get_order_book(symbol: str, limit: int = None) -> dict:
    """Get order book for symbol."""
```

**Trading Operations**
```python
def create_order(symbol: str, type: str, side: str, amount: float, 
                price: float = None, params: dict = None) -> dict:
    """
    Create trading order.
    
    Args:
        symbol: Trading pair
        type: Order type ('market', 'limit')
        side: Order side ('buy', 'sell')
        amount: Order quantity
        price: Order price (for limit orders)
        params: Additional exchange-specific parameters
        
    Returns:
        Order information dictionary
    """
    
def cancel_order(order_id: str, symbol: str) -> dict:
    """Cancel existing order."""
    
def fetch_order(order_id: str, symbol: str) -> dict:
    """Fetch order status and information."""
```

**Account Information**
```python
def fetch_balance() -> dict:
    """Get account balance information."""
    
def fetch_positions() -> List[dict]:
    """Get open positions (for margin/futures)."""
    
def fetch_orders(symbol: str = None, since: int = None, limit: int = None) -> List[dict]:
    """Get order history."""
```

### CCXTDataFeed

Provides OHLCV data to strategies from exchanges.

```python
class CCXTDataFeed(bt.DataBase):
    """Data feed for exchange OHLCV data."""
    
    params = (
        ('symbol', ''),           # Trading pair symbol
        ('timeframe', '1m'),      # Data timeframe
        ('compression', 1),       # Compression factor
        ('historical', True),     # Load historical data
        ('backfill', True),       # Backfill missing data
        ('stream', True),         # Enable live streaming
    )
```

#### Methods

```python
def start() -> None:
    """Start data feed."""
    
def stop() -> None:
    """Stop data feed."""
    
def islive() -> bool:
    """Check if feed is live."""
    
def haslivedata() -> bool:
    """Check if live data is available."""
```

## Broker Classes

### BaseCCXTBroker

Base class for all CCXT brokers.

```python
class BaseCCXTBroker:
    """Base broker with common functionality."""
    
    def __init__(self, store: CCXTStore):
        """Initialize broker with store connection."""
```

#### Methods

**Order Management**
```python
def buy(data: DataFeed, size: float, price: float = None, **kwargs) -> Order:
    """Create buy order."""
    
def sell(data: DataFeed, size: float, price: float = None, **kwargs) -> Order:
    """Create sell order."""
    
def cancel(order: Order) -> bool:
    """Cancel pending order."""
```

**Account Information**
```python
def get_cash() -> float:
    """Get available cash balance."""
    
def get_value() -> float:
    """Get total account value."""
    
def getposition(data: DataFeed) -> Position:
    """Get current position for data feed."""
```

### CCXTLiveBroker

Live trading broker for real exchange connectivity.

```python
class CCXTLiveBroker(BaseCCXTBroker):
    """Live broker for real trading."""
    
    params = (
        ('use_positions', True),    # Use exchange positions
        ('lever', 1.0),            # Default leverage
        ('check_submit', False),    # Validate orders before submission
    )
```

### CCXTBackBroker

Backtesting broker using historical data.

```python
class CCXTBackBroker(BaseCCXTBroker):
    """Backtesting broker with simulated execution."""
    
    params = (
        ('cash', 10000),           # Starting cash
        ('commission', 0.001),     # Commission rate
        ('margin', None),          # Margin requirements
        ('mult', 1.0),             # Contract multiplier
    )
```

### CCXTPaperBroker

Paper trading broker with live data but simulated execution.

```python
class CCXTPaperBroker(BaseCCXTBroker):
    """Paper trading broker with live prices."""
    
    params = (
        ('cash', 100000),          # Paper money amount
        ('commission', 0.001),     # Simulated commission
        ('track_orders', True),    # Track order lifecycle
    )
```

## Order Classes

### CCXTOrder

Enhanced order class with exchange-specific features.

```python
class CCXTOrder(bt.Order):
    """Extended order with CCXT exchange features."""
    
    # Order types
    Market = 1
    Limit = 2
    Stop = 3
    StopLimit = 4
    
    # Time in force
    GTC = 'GTC'  # Good Till Cancelled
    IOC = 'IOC'  # Immediate or Cancel
    FOK = 'FOK'  # Fill or Kill
```

#### Attributes

```python
@property
def exchange_id(self) -> str:
    """Exchange-specific order ID."""
    
@property
def status(self) -> str:
    """Current order status."""
    
@property
def filled(self) -> float:
    """Filled quantity."""
    
@property
def remaining(self) -> float:
    """Remaining quantity."""
    
@property
def fee(self) -> dict:
    """Order execution fees."""
```

## Position Classes

### Position

Enhanced position tracking.

```python
class Position:
    """Position information."""
    
    def __init__(self, data: DataFeed, size: float = 0, price: float = 0):
        """Initialize position."""
```

#### Properties

```python
@property
def size(self) -> float:
    """Position size (positive=long, negative=short)."""
    
@property
def price(self) -> float:
    """Average entry price."""
    
@property
def datetime(self) -> datetime:
    """Position entry datetime."""
    
@property
def upnl(self) -> float:
    """Unrealized P&L."""
    
@property
def pnl(self) -> float:
    """Realized P&L."""
```

## Commission Info Classes

### CCXTCommissionInfo

Exchange-specific commission calculation.

```python
class CCXTCommissionInfo(bt.CommInfoBase):
    """Commission info for CCXT exchanges."""
    
    params = (
        ('commission', 0.001),     # Base commission rate
        ('margin', None),          # Margin requirements
        ('mult', 1.0),             # Contract multiplier
        ('stocklike', True),       # Stock-like position sizing
    )
```

#### Methods

```python
def getcommission(self, size: float, price: float) -> float:
    """Calculate commission for order."""
    
def get_margin(self, price: float) -> float:
    """Get margin requirement."""
    
def getsize(self, price: float, cash: float) -> float:
    """Calculate position size for cash amount."""
```

## Indicator Classes

Cracktrader includes all Backtrader indicators plus crypto-specific ones.

### Standard Indicators

```python
# Moving Averages
SMA(data, period=30)           # Simple Moving Average
EMA(data, period=30)           # Exponential Moving Average
WMA(data, period=30)           # Weighted Moving Average

# Oscillators
RSI(data, period=14)           # Relative Strength Index
Stochastic(data, period=14)    # Stochastic Oscillator
MACD(data, period_me1=12, period_me2=26, period_signal=9)

# Volatility
BollingerBands(data, period=20, devfactor=2.0)
ATR(data, period=14)           # Average True Range
StdDev(data, period=20)        # Standard Deviation

# Volume
VolumeWeightedAveragePrice(data, period=20)
OnBalanceVolume(data)
AccumulationDistribution(data)
```

### Crypto-Specific Indicators

```python
# Funding Rate Indicator
FundingRate(data, period=8)    # 8-hour funding periods

# Exchange Premium
ExchangePremium(data1, data2)  # Premium between exchanges

# Liquidation Levels
LiquidationLevels(data, leverage=3.0)
```

## Analyzer Classes

### Standard Analyzers

```python
# Performance Analysis
Returns()                      # Return analysis
SharpeRatio()                 # Sharpe ratio calculation
DrawDown()                    # Drawdown analysis
TradeAnalyzer()               # Trade statistics

# Risk Analysis
VaR()                         # Value at Risk
SQN()                         # System Quality Number
```

### Crypto-Specific Analyzers

```python
class FundingCostAnalyzer(bt.Analyzer):
    """Analyze funding costs for perpetual futures."""
    
class LiquidationAnalyzer(bt.Analyzer):
    """Track liquidation risks and near-liquidation events."""
    
class ExchangeLatencyAnalyzer(bt.Analyzer):
    """Monitor exchange response times and slippage."""
```

## Strategy Base Classes

### CCXTStrategy

Enhanced strategy base class for crypto trading.

```python
class CCXTStrategy(bt.Strategy):
    """Enhanced strategy for crypto trading."""
    
    params = (
        ('printlog', True),        # Print log messages
        ('risk_per_trade', 0.02), # Risk per trade
        ('max_positions', 5),      # Max concurrent positions
    )
```

#### Methods

```python
def log(self, txt: str, dt: datetime = None) -> None:
    """Log message with timestamp."""
    
def notify_order(self, order: Order) -> None:
    """Handle order status updates."""
    
def notify_trade(self, trade: Trade) -> None:
    """Handle trade completion."""
    
def get_exchange_info(self, symbol: str) -> dict:
    """Get exchange-specific symbol information."""
```

## Utility Classes

### TimeFrameConverter

Convert between different timeframe formats.

```python
class TimeFrameConverter:
    """Convert between timeframe formats."""
    
    @staticmethod
    def to_ccxt(timeframe: str) -> str:
        """Convert to CCXT timeframe format."""
        
    @staticmethod
    def to_seconds(timeframe: str) -> int:
        """Convert timeframe to seconds."""
        
    @staticmethod
    def to_pandas_freq(timeframe: str) -> str:
        """Convert to pandas frequency string."""
```

### SymbolMapper

Handle symbol format differences between exchanges.

```python
class SymbolMapper:
    """Map symbols between different exchange formats."""
    
    def __init__(self, exchange: str):
        """Initialize with exchange-specific mappings."""
        
    def normalize(self, symbol: str) -> str:
        """Normalize symbol to standard format."""
        
    def to_exchange(self, symbol: str) -> str:
        """Convert to exchange-specific format."""
```

## Configuration Classes

### ExchangeConfig

Exchange-specific configuration management.

```python
class ExchangeConfig:
    """Manage exchange-specific configuration."""
    
    def __init__(self, exchange: str, config: dict = None):
        """Initialize configuration."""
        
    def get_trading_fees(self) -> dict:
        """Get trading fee structure."""
        
    def get_order_limits(self) -> dict:
        """Get order size and price limits."""
        
    def get_rate_limits(self) -> dict:
        """Get API rate limit information."""
```

## Error Classes

### CCXTError

Base exception class for Cracktrader errors.

```python
class CCXTError(Exception):
    """Base exception for CCXT-related errors."""
    
class ConnectionError(CCXTError):
    """Exchange connection errors."""
    
class OrderError(CCXTError):
    """Order placement/management errors."""
    
class InsufficientFunds(CCXTError):
    """Insufficient balance for operation."""
    
class InvalidOrder(CCXTError):
    """Invalid order parameters."""
    
class RateLimitExceeded(CCXTError):
    """API rate limit exceeded."""
```

## See Also

- [Configuration API](configuration.md) - Configuration options
- [Web API](web_api.md) - REST API reference  
- [Indicators](indicators.md) - Complete indicator reference