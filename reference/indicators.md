# Indicators Reference

Complete reference for all available indicators in Cracktrader. Access indicators through `cracktrader.indicators` or `ct.indicators`.

## Quick Start

```python
import cracktrader as ct

# In your strategy
class MyStrategy(ct.bt.Strategy):
    def __init__(self):
        # Use indicators directly from cracktrader
        self.sma = ct.indicators.SMA(self.data.close, period=20)
        self.rsi = ct.indicators.RSI(self.data.close, period=14)
        self.bb = ct.indicators.BollingerBands(self.data.close, period=20)
```

## Moving Averages

### Simple Moving Average (SMA)
```python
SMA(data, period=30)
```
**Parameters:**
- `data`: Price series (usually close prices)
- `period`: Number of periods for average calculation

**Usage:**
```python
self.sma_short = ct.indicators.SMA(self.data.close, period=10)
self.sma_long = ct.indicators.SMA(self.data.close, period=30)

# Golden cross signal
if self.sma_short[0] > self.sma_long[0]:
    # Bullish signal
```

### Exponential Moving Average (EMA)
```python
EMA(data, period=30, alpha=None)
```
**Parameters:**
- `data`: Price series
- `period`: Number of periods
- `alpha`: Smoothing factor (optional, calculated from period if not provided)

**Usage:**
```python
self.ema_fast = ct.indicators.EMA(self.data.close, period=12)
self.ema_slow = ct.indicators.EMA(self.data.close, period=26)
```

### Weighted Moving Average (WMA)
```python
WMA(data, period=30)
```
**Usage:**
```python
self.wma = ct.indicators.WMA(self.data.close, period=20)
```

### Triple Exponential Moving Average (TEMA)
```python
TEMA(data, period=30)
```
**Usage:**
```python
self.tema = ct.indicators.TEMA(self.data.close, period=21)
```

## Momentum Oscillators

### Relative Strength Index (RSI)
```python
RSI(data, period=14, upperband=70, lowerband=30, safediv=False)
```
**Parameters:**
- `period`: Look-back period (default: 14)
- `upperband`: Overbought level (default: 70)
- `lowerband`: Oversold level (default: 30)

**Usage:**
```python
self.rsi = ct.indicators.RSI(self.data.close, period=14)

# Trading signals
if self.rsi[0] < 30:
    # Oversold - potential buy signal
elif self.rsi[0] > 70:
    # Overbought - potential sell signal
```

### Stochastic Oscillator
```python
Stochastic(data, period=14, period_dfast=3, period_dslow=3,
           upperband=80, lowerband=20, safediv=False)
```
**Lines:**
- `percK`: %K line (fast stochastic)
- `percD`: %D line (slow stochastic)

**Usage:**
```python
self.stoch = ct.indicators.Stochastic(self.data, period=14)

# %K crosses above %D
if self.stoch.percK[0] > self.stoch.percD[0]:
    # Bullish signal
```

### MACD (Moving Average Convergence Divergence)
```python
MACD(data, period_me1=12, period_me2=26, period_signal=9)
```
**Lines:**
- `macd`: MACD line
- `signal`: Signal line
- `histo`: Histogram (MACD - Signal)

**Usage:**
```python
self.macd = ct.indicators.MACD(self.data.close)

# MACD crosses above signal line
if self.macd.macd[0] > self.macd.signal[0]:
    # Bullish crossover
```

### Commodity Channel Index (CCI)
```python
CCI(data, period=20, factor=0.015)
```
**Usage:**
```python
self.cci = ct.indicators.CCI(self.data, period=20)

# Extreme readings
if self.cci[0] > 100:
    # Overbought
elif self.cci[0] < -100:
    # Oversold
```

### Williams %R
```python
WilliamsR(data, period=14, upperband=-20, lowerband=-80)
```
**Usage:**
```python
self.williams = ct.indicators.WilliamsR(self.data, period=14)
```

## Volatility Indicators

### Bollinger Bands
```python
BollingerBands(data, period=20, devfactor=2.0, movav=SMA)
```
**Lines:**
- `top`: Upper band
- `mid`: Middle band (moving average)
- `bot`: Lower band

**Usage:**
```python
self.bb = ct.indicators.BollingerBands(self.data.close, period=20, devfactor=2.0)

# Price touches lower band
if self.data.close[0] <= self.bb.bot[0]:
    # Potential buy signal (oversold)

# Price touches upper band
if self.data.close[0] >= self.bb.top[0]:
    # Potential sell signal (overbought)
```

### Average True Range (ATR)
```python
ATR(data, period=14, movav=SMA)
```
**Usage:**
```python
self.atr = ct.indicators.ATR(self.data, period=14)

# Use ATR for stop loss
stop_distance = self.atr[0] * 2  # 2 ATR stop
```

### Standard Deviation
```python
StdDev(data, period=20, movav=SMA)
```
**Usage:**
```python
self.std = ct.indicators.StdDev(self.data.close, period=20)

# Volatility-based position sizing
if self.std[0] > average_volatility:
    position_size *= 0.5  # Reduce size in high volatility
```

### Donchian Channel
```python
DonchianChannel(data, period=20)
```
**Lines:**
- `dcm`: Middle line
- `dch`: Upper channel
- `dcl`: Lower channel

**Usage:**
```python
self.donchian = ct.indicators.DonchianChannel(self.data, period=20)

# Breakout strategy
if self.data.close[0] > self.donchian.dch[-1]:
    # Upside breakout
```

### Keltner Channel
```python
KeltnerChannel(data, period=20, devfactor=2.0)
```
**Lines:**
- `top`: Upper channel
- `mid`: Middle line (EMA)
- `bot`: Lower channel

## Trend Indicators

### Average Directional Index (ADX)
```python
ADX(data, period=14)
```
**Lines:**
- `adx`: ADX line (trend strength)
- `plusDI`: +DI line
- `minusDI`: -DI line

**Usage:**
```python
self.adx = ct.indicators.ADX(self.data, period=14)

# Strong trend
if self.adx.adx[0] > 25:
    # Trend is strong, use trend-following strategies
    if self.adx.plusDI[0] > self.adx.minusDI[0]:
        # Uptrend
```

### Parabolic SAR
```python
ParabolicSAR(data, af=0.02, afmax=0.20)
```
**Usage:**
```python
self.psar = ct.indicators.ParabolicSAR(self.data)

# Trend reversal
if self.data.close[0] > self.psar[0]:
    # Price above SAR - uptrend
```

### Aroon Oscillator
```python
AroonOscillator(data, period=14)
```
**Lines:**
- `aroon`: Aroon oscillator
- `aroonup`: Aroon up
- `aroondown`: Aroon down

**Usage:**
```python
self.aroon = ct.indicators.AroonOscillator(self.data, period=14)
```

## Volume Indicators

### On Balance Volume (OBV)
```python
OnBalanceVolume(data)
```
**Usage:**
```python
self.obv = ct.indicators.OnBalanceVolume(self.data)

# Divergence analysis
if price_making_new_high and obv_not_making_new_high:
    # Bearish divergence
```

### Volume Weighted Average Price (VWAP)
```python
VWAP(data, period=30)
```
**Usage:**
```python
self.vwap = ct.indicators.VWAP(self.data, period=30)

# Price relative to VWAP
if self.data.close[0] > self.vwap[0]:
    # Above VWAP - bullish
```

### Accumulation/Distribution Line
```python
AccumulationDistribution(data)
```
**Usage:**
```python
self.ad = ct.indicators.AccumulationDistribution(self.data)
```

### Money Flow Index (MFI)
```python
MoneyFlowIndex(data, period=14)
```
**Usage:**
```python
self.mfi = ct.indicators.MoneyFlowIndex(self.data, period=14)

# Overbought/oversold with volume
if self.mfi[0] > 80:
    # Overbought with volume
elif self.mfi[0] < 20:
    # Oversold with volume
```

## Statistical Indicators

### Linear Regression
```python
LinearRegression(data, period=14)
```
**Usage:**
```python
self.lr = ct.indicators.LinearRegression(self.data.close, period=14)
```

### Correlation
```python
Correlation(data0, data1, period=30)
```
**Usage:**
```python
self.corr = ct.indicators.Correlation(
    self.datas[0].close,  # BTC
    self.datas[1].close,  # ETH
    period=30
)

# High correlation
if abs(self.corr[0]) > 0.8:
    # Assets moving together
```

### Covariance
```python
Covariance(data0, data1, period=30)
```

### Beta
```python
Beta(data0, data1, period=30)
```
**Usage:**
```python
# Bitcoin as market benchmark
self.beta = ct.indicators.Beta(
    self.data.close,     # Alt coin
    self.btc_data.close, # Bitcoin
    period=30
)
```

## Custom Crypto Indicators

### Funding Rate (Custom)
```python
class FundingRate(ct.bt.Indicator):
    """8-hour funding rate for perpetual futures"""
    lines = ('funding',)
    params = (('period', 8),)  # 8 hours
    
    def __init__(self):
        # Implementation depends on exchange data
        pass
```

### Exchange Premium
```python
class ExchangePremium(ct.bt.Indicator):
    """Price premium between exchanges"""
    lines = ('premium',)
    
    def __init__(self, data1, data2):
        self.lines.premium = (data1.close - data2.close) / data2.close * 100
```

### Liquidation Levels
```python
class LiquidationLevels(ct.bt.Indicator):
    """Calculate liquidation prices for leveraged positions"""
    lines = ('long_liq', 'short_liq')
    params = (('leverage', 3.0), ('maintenance_margin', 0.05))
    
    def __init__(self):
        # Long liquidation price
        self.lines.long_liq = self.data.close * (1 - (1 / self.params.leverage - self.params.maintenance_margin))
        
        # Short liquidation price  
        self.lines.short_liq = self.data.close * (1 + (1 / self.params.leverage - self.params.maintenance_margin))
```

## Composite Indicators

### Ichimoku Kinko Hyo
```python
Ichimoku(data, tenkan=9, kijun=26, senkou=52, senkou_lead=26, chikou=26)
```
**Lines:**
- `tenkan_sen`: Conversion line
- `kijun_sen`: Base line
- `senkou_span_a`: Leading span A
- `senkou_span_b`: Leading span B
- `chikou_span`: Lagging span

**Usage:**
```python
self.ichimoku = ct.indicators.Ichimoku(self.data)

# Bullish signal
if (self.data.close[0] > self.ichimoku.senkou_span_a[0] and
    self.data.close[0] > self.ichimoku.senkou_span_b[0]):
    # Above cloud - bullish
```

## Multi-Timeframe Indicators

```python
# Higher timeframe data
class MultiTimeFrameStrategy(ct.bt.Strategy):
    def __init__(self):
        # Daily SMA on 1-hour chart
        self.daily_sma = ct.indicators.SMA(
            self.data.close,
            period=24  # 24 hours = 1 day on hourly chart
        )
        
        # Weekly RSI on daily chart
        self.weekly_rsi = ct.indicators.RSI(
            self.data.close,
            period=7  # 7 days = 1 week on daily chart
        )
```

## Indicator Combinations

### Confluence Strategy
```python
class ConfluenceIndicators(ct.bt.Strategy):
    def __init__(self):
        # Multiple indicators for confluence
        self.sma_20 = ct.indicators.SMA(self.data.close, period=20)
        self.rsi = ct.indicators.RSI(self.data.close, period=14)
        self.bb = ct.indicators.BollingerBands(self.data.close, period=20)
        self.macd = ct.indicators.MACD(self.data.close)
        
    def check_bullish_confluence(self):
        """Check for bullish signal confluence"""
        signals = 0
        
        # Price above SMA
        if self.data.close[0] > self.sma_20[0]:
            signals += 1
            
        # RSI oversold but recovering
        if 30 < self.rsi[0] < 50:
            signals += 1
            
        # Bollinger Band bounce
        if self.data.close[0] > self.bb.bot[0]:
            signals += 1
            
        # MACD bullish
        if self.macd.macd[0] > self.macd.signal[0]:
            signals += 1
            
        return signals >= 3  # Require 3+ bullish signals
```

## Indicator Utilities

### List Available Indicators
```python
# Get all available indicators
all_indicators = ct.indicators.get_all_indicators()
print(f"Available indicators: {len(all_indicators)}")

# Get indicators by category
trend_indicators = ct.indicators.get_trend_indicators()
momentum_indicators = ct.indicators.get_momentum_indicators()
volatility_indicators = ct.indicators.get_volatility_indicators()
```

### Indicator Information
```python
# Get indicator documentation
info = ct.indicators.get_indicator_info('RSI')
print(f"RSI parameters: {info['params']}")
print(f"RSI description: {info['description']}")
```

## Performance Tips

1. **Minimize Indicator Creation**: Create indicators in `__init__()`, not in `next()`
2. **Use Efficient Indicators**: Some indicators are more computationally expensive
3. **Cache Results**: Store frequently used indicator values
4. **Vectorized Operations**: Use NumPy operations when possible

```python
# Good: Create once in __init__
def __init__(self):
    self.sma = ct.indicators.SMA(self.data.close, period=20)
    
def next(self):
    sma_value = self.sma[0]  # Fast access
    
# Bad: Create in next() - very slow
def next(self):
    sma = ct.indicators.SMA(self.data.close, period=20)  # Don't do this!
```

## See Also

- [Custom Indicators Tutorial](../tutorials/custom_indicators.md) - Creating your own indicators
- [Core Classes](core_classes.md) - Core API reference
- [Multi-Asset Strategies](../tutorials/multi_asset.md) - Using indicators across assets