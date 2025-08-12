# Exchanges

CrackTrader supports 400+ cryptocurrency exchanges through CCXT.

## Supported Exchanges

Popular exchanges with full support:
- **Binance** - Spot, futures, margin
- **Coinbase Pro** - Spot trading
- **Kraken** - Spot, futures
- **Bybit** - Futures, spot
- **OKX** - Spot, futures, margin
- **Bitget** - Spot, futures
- **Gate.io** - Spot, futures

Full list: [CCXT Supported Exchanges](https://github.com/ccxt/ccxt#supported-cryptocurrency-exchange-markets)

## Exchange Configuration

### Binance
```python
config = {
    'exchange': 'binance',
    'sandbox': True,  # Use testnet
    'apiKey': 'your_api_key',
    'secret': 'your_secret'
}
```

### Coinbase Pro
```python
config = {
    'exchange': 'coinbase',
    'sandbox': True,
    'apiKey': 'your_api_key',
    'secret': 'your_secret',
    'passphrase': 'your_passphrase'
}
```

### Kraken
```python
config = {
    'exchange': 'kraken',
    'apiKey': 'your_api_key',
    'secret': 'your_secret'
}
```

## Exchange-Specific Features

### Binance
- **Asset types**: Spot, futures, margin
- **Timeframes**: 1s, 1m, 3m, 5m, 15m, 30m, 1h, 2h, 4h, 6h, 8h, 12h, 1d, 3d, 1w, 1M
- **Order types**: Market, limit, stop, OCO, bracket
- **WebSocket**: Real-time data feeds

### Coinbase Pro
- **Asset types**: Spot only
- **Timeframes**: 1m, 5m, 15m, 1h, 6h, 1d
- **Order types**: Market, limit, stop
- **WebSocket**: Real-time data feeds

## Adding New Exchanges

CrackTrader automatically supports any CCXT exchange:

```python
from cracktrader import CCXTStore

# Any CCXT exchange works
store = CCXTStore({'exchange': 'bybit', 'sandbox': True})
store = CCXTStore({'exchange': 'okx', 'sandbox': True})
store = CCXTStore({'exchange': 'gate', 'sandbox': True})
```

## Exchange Testing

### Sandbox/Testnet Support
```python
# Exchanges with sandbox support
sandbox_exchanges = [
    'binance',     # testnet.binance.vision
    'coinbase',    # public-sandbox.pro.coinbase.com
    'bybit',       # api-testnet.bybit.com
    'okx'          # www.okx.com (paper trading)
]

# Test connection
store = CCXTStore({'exchange': 'binance', 'sandbox': True})
```

### Connection Testing
```python
def test_exchange_connection(exchange_name, config):
    try:
        store = CCXTStore(config)
        # Test basic functionality
        markets = store.exchange.load_markets()
        balance = store.exchange.fetch_balance()
        print(f"{exchange_name}: Connected successfully")
        return True
    except Exception as e:
        print(f"{exchange_name}: Connection failed - {e}")
        return False
```

## Rate Limits

Each exchange has different rate limits:

```python
# Enable rate limiting (recommended)
config = {
    'exchange': 'binance',
    'enableRateLimit': True,
    'rateLimit': 1200  # ms between requests
}

store = CCXTStore(config)
```

## Market Data

### Available Data Types
- **OHLCV**: Candlestick data
- **Trades**: Individual trades
- **Order book**: Bid/ask levels
- **Ticker**: 24h statistics
- **Balance**: Account balances

### Timeframe Support
```python
# Check supported timeframes
exchange = ccxt.binance()
print(exchange.timeframes)
# Output: {'1m': '1m', '3m': '3m', '5m': '5m', ...}
```

## Commission Structures

Different exchanges have different fee structures:

```python
from cracktrader.comm_info import BinanceCommissionInfo, CoinbaseCommissionInfo

# Binance: 0.1% maker/taker
binance_fees = BinanceCommissionInfo()

# Coinbase: 0.5% taker, 0.0% maker
coinbase_fees = CoinbaseCommissionInfo()
```

## Multi-Exchange Trading

```python
# Trade on multiple exchanges simultaneously
binance_store = CCXTStore({'exchange': 'binance', 'sandbox': True})
coinbase_store = CCXTStore({'exchange': 'coinbase', 'sandbox': True})

binance_data = CCXTDataFeed(binance_store, symbol='BTC/USDT')
coinbase_data = CCXTDataFeed(coinbase_store, symbol='BTC-USD')

cerebro = bt.Cerebro()
cerebro.adddata(binance_data, name='binance_btc')
cerebro.adddata(coinbase_data, name='coinbase_btc')
```

## Exchange-Specific Considerations

### Symbol Formats
- **Binance**: BTC/USDT, ETH/USDT
- **Coinbase**: BTC-USD, ETH-USD
- **Kraken**: XXBTZUSD, XETHZUSD

### Minimum Order Sizes
- **Binance BTC/USDT**: 0.00001 BTC
- **Coinbase BTC-USD**: 0.001 BTC
- **Kraken BTC/USD**: 0.0001 BTC

### API Limits
- **Binance**: 1200 requests/minute
- **Coinbase**: 10 requests/second
- **Kraken**: 15 requests/minute

## Best Practices

1. **Use sandbox/testnet** for development
2. **Enable rate limiting** to avoid bans
3. **Check symbol formats** for each exchange
4. **Verify minimum order sizes** before trading
5. **Monitor API usage** to stay within limits
6. **Handle exchange-specific errors** appropriately

## Troubleshooting

### Common Issues
- **Invalid API keys**: Check config.json
- **Rate limit exceeded**: Enable rate limiting
- **Symbol not found**: Check exchange symbol format
- **Insufficient balance**: Check account funding
- **Network errors**: Check internet connection

### Debug Mode
```python
import logging
logging.getLogger('ccxt').setLevel(logging.DEBUG)

store = CCXTStore(config)
# Will show detailed API requests/responses
```
