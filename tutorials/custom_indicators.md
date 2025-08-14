# Custom Indicators

Cracktrader provides transparent access to all Backtrader indicators through the `cracktrader.indicators` module, and allows you to create custom indicators for specialized trading strategies.

## Using Standard Indicators

All standard technical indicators are available through the `cracktrader.indicators` module:

```python
from cracktrader.indicators import SMA, EMA, RSI, MACD, BollingerBands

class MyStrategy(bt.Strategy):
    def __init__(self):
        # Moving averages
        self.sma20 = SMA(self.data.close, period=20)
        self.ema12 = EMA(self.data.close, period=12)

        # Oscillators
        self.rsi = RSI(self.data.close, period=14)
        self.macd = MACD(self.data.close)

        # Volatility
        self.bbands = BollingerBands(self.data.close, period=20)
```

## Available Indicators

List all available indicators:

```python
from cracktrader.indicators import list_indicators

# Get all available indicators
indicators = list_indicators()
print(f"Available indicators: {len(indicators)}")
for indicator in indicators[:10]:  # Show first 10
    print(f"  - {indicator}")
```

Get information about a specific indicator:

```python
from cracktrader.indicators import get_indicator_info

# Get details about RSI
info = get_indicator_info('RSI')
print(f"Name: {info['name']}")
print(f"Parameters: {info['params']}")
print(f"Lines: {info['lines']}")
```

## Creating Custom Indicators

### Simple Custom Indicator

```python
import backtrader as bt
from cracktrader.indicators import SMA

class CustomRSI(bt.Indicator):
    """
    Custom RSI implementation with additional features.
    """
    lines = ('rsi', 'oversold', 'overbought')
    params = (
        ('period', 14),
        ('oversold', 30),
        ('overbought', 70),
    )

    def __init__(self):
        # Calculate price changes
        self.up = bt.If(self.data > self.data(-1),
                       self.data - self.data(-1), 0)
        self.down = bt.If(self.data < self.data(-1),
                         self.data(-1) - self.data, 0)

        # Calculate averages
        self.avg_up = SMA(self.up, period=self.params.period)
        self.avg_down = SMA(self.down, period=self.params.period)

        # Calculate RSI
        rs = self.avg_up / self.avg_down
        self.lines.rsi = 100 - (100 / (1 + rs))

        # Define oversold/overbought levels
        self.lines.oversold = self.params.oversold
        self.lines.overbought = self.params.overbought

# Usage
class Strategy(bt.Strategy):
    def __init__(self):
        self.custom_rsi = CustomRSI(self.data.close)

    def next(self):
        if self.custom_rsi.rsi < self.custom_rsi.oversold:
            print("RSI Oversold signal")
        elif self.custom_rsi.rsi > self.custom_rsi.overbought:
            print("RSI Overbought signal")
```

### Advanced Multi-Timeframe Indicator

```python
class MultiTimeframeMA(bt.Indicator):
    """
    Moving average that uses data from multiple timeframes.
    """
    lines = ('ma_short', 'ma_long', 'signal')
    params = (
        ('period_short', 20),
        ('period_long', 50),
        ('timeframe_mult', 4),  # 4x higher timeframe
    )

    def __init__(self):
        # Get higher timeframe data
        self.data_htf = self.data.resample(
            timeframe=self.data._timeframe,
            compression=self.params.timeframe_mult
        )

        # Calculate moving averages
        self.lines.ma_short = SMA(self.data.close, period=self.params.period_short)
        self.lines.ma_long = SMA(self.data_htf.close, period=self.params.period_long)

        # Generate signal when short-term MA crosses above long-term MA
        self.lines.signal = bt.If(
            self.lines.ma_short > self.lines.ma_long, 1,
            bt.If(self.lines.ma_short < self.lines.ma_long, -1, 0)
        )
```

### Crypto-Specific Indicators

```python
class CryptoVolatilityIndex(bt.Indicator):
    """
    Custom volatility index for cryptocurrency markets.
    Accounts for 24/7 trading and high volatility.
    """
    lines = ('cvi', 'volatility_signal')
    params = (
        ('period', 24),  # 24 hours for crypto
        ('threshold_low', 20),
        ('threshold_high', 80),
    )

    def __init__(self):
        # Calculate hourly returns
        returns = (self.data.close / self.data.close(-1) - 1) * 100

        # Calculate rolling volatility
        volatility = bt.indicators.StdDev(returns, period=self.params.period)

        # Normalize to 0-100 scale
        vol_min = bt.indicators.Lowest(volatility, period=self.params.period * 7)
        vol_max = bt.indicators.Highest(volatility, period=self.params.period * 7)

        self.lines.cvi = ((volatility - vol_min) / (vol_max - vol_min)) * 100

        # Generate signals
        self.lines.volatility_signal = bt.If(
            self.lines.cvi < self.params.threshold_low, 1,  # Low vol - trend continuation
            bt.If(self.lines.cvi > self.params.threshold_high, -1, 0)  # High vol - reversal
        )

class FundingRateIndicator(bt.Indicator):
    """
    Indicator for tracking funding rates in perpetual futures.
    """
    lines = ('funding_rate', 'funding_signal')
    params = (
        ('extreme_positive', 0.1),  # 0.1% funding rate
        ('extreme_negative', -0.1),
    )

    def __init__(self, funding_data):
        """
        Args:
            funding_data: Additional data feed with funding rates
        """
        self.lines.funding_rate = funding_data.close

        # Generate signals based on extreme funding rates
        self.lines.funding_signal = bt.If(
            self.lines.funding_rate > self.params.extreme_positive, -1,  # Short bias
            bt.If(self.lines.funding_rate < self.params.extreme_negative, 1, 0)  # Long bias
        )
```

## Indicator Composition

Combine multiple indicators for complex signals:

```python
class CompositeSignal(bt.Indicator):
    """
    Composite signal from multiple indicators.
    """
    lines = ('signal', 'strength')
    params = (
        ('rsi_period', 14),
        ('ma_short', 20),
        ('ma_long', 50),
    )

    def __init__(self):
        # Individual indicators
        self.rsi = RSI(self.data.close, period=self.params.rsi_period)
        self.ma_short = SMA(self.data.close, period=self.params.ma_short)
        self.ma_long = SMA(self.data.close, period=self.params.ma_long)
        self.macd = MACD(self.data.close)

        # Combine signals
        rsi_bull = self.rsi < 30  # Oversold
        ma_bull = self.ma_short > self.ma_long  # Trend up
        macd_bull = self.macd.macd > self.macd.signal  # Momentum up

        # Count bullish signals
        bull_count = rsi_bull + ma_bull + macd_bull

        # Generate composite signal
        self.lines.signal = bt.If(bull_count >= 2, 1,
                                 bt.If(bull_count <= 1, -1, 0))
        self.lines.strength = bull_count / 3.0  # Signal strength 0-1
```

## Performance Optimization

### Efficient Indicator Calculation

```python
class FastSMA(bt.Indicator):
    """
    Optimized SMA using numpy for better performance.
    """
    lines = ('sma',)
    params = (('period', 20),)

    def __init__(self):
        self.addminperiod(self.params.period)

    def next(self):
        # Use numpy for faster calculation on larger windows
        import numpy as np
        data_slice = np.array([self.data.get(ago=i)
                              for i in range(self.params.period)])
        self.lines.sma[0] = np.mean(data_slice)
```

### Memory Efficient Indicators

```python
class MemoryEfficientIndicator(bt.Indicator):
    """
    Indicator that doesn't store full history.
    """
    lines = ('value',)
    params = (('period', 20),)

    def __init__(self):
        self.data_buffer = []
        self.sum = 0.0

    def next(self):
        # Add new value
        new_value = self.data[0]
        self.data_buffer.append(new_value)
        self.sum += new_value

        # Remove old values to maintain window size
        if len(self.data_buffer) > self.params.period:
            old_value = self.data_buffer.pop(0)
            self.sum -= old_value

        # Calculate average
        self.lines.value[0] = self.sum / len(self.data_buffer)
```

## Best Practices

1. **Use Existing Indicators**: Always check if what you need already exists
2. **Optimize for Performance**: Use vectorized operations when possible
3. **Test Thoroughly**: Validate your indicators with known data
4. **Document Parameters**: Clearly explain what each parameter does
5. **Handle Edge Cases**: Consider division by zero, missing data, etc.

## See Also

- [Strategy Tutorials](multi_exchange.md) - Using indicators in strategies
- [Reference: Indicators](../reference/indicators.md) - Complete indicator reference
- [Performance](../performance/overview.md) - Optimizing indicator performance
