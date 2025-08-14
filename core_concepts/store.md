# Store Component

The Store is the central hub of Cracktrader, managing connections to cryptocurrency exchanges and coordinating data flow between different components.

## Overview

The Store acts as:
- **Exchange Gateway**: Manages connections to multiple exchanges
- **Data Coordinator**: Routes market data to feeds and strategies
- **Order Router**: Handles order placement and execution
- **Cache Manager**: Coordinates data caching and retrieval

## Architecture

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Strategy  │    │   Broker    │    │  Data Feed  │
└──────┬──────┘    └──────┬──────┘    └──────┬──────┘
       │                  │                  │
       └──────────────────┼──────────────────┘
                          │
                   ┌──────▼──────┐
                   │    Store    │
                   └──────┬──────┘
                          │
       ┌──────────────────┼──────────────────┐
       │                  │                  │
┌──────▼──────┐    ┌──────▼──────┐    ┌──────▼──────┐
│  Exchange A │    │  Exchange B │    │  Exchange C │
└─────────────┘    └─────────────┘    └─────────────┘
```

## Key Features

### Multi-Exchange Support
- Connect to multiple exchanges simultaneously
- Unified API across different exchange protocols
- Automatic failover and load balancing

### Real-Time Data Streaming
- WebSocket connections for live market data
- Efficient data distribution to multiple consumers
- Built-in data quality monitoring

### Order Management
- Unified order placement across exchanges
- Order status tracking and updates
- Support for complex order types

### Data Persistence
- Intelligent caching of historical data
- Configurable storage backends
- Automatic data synchronization

## Configuration

```python
from cracktrader import CCXTStore

# Basic store configuration
store = CCXTStore(
    exchange='binance',
    sandbox=True,
    config={
        'apiKey': 'your_api_key',
        'secret': 'your_secret',
        'enableRateLimit': True,
        'sandbox': True
    }
)
```

## Advanced Usage

### Multiple Exchanges
```python
# Configure multiple exchange connections
stores = {
    'binance': CCXTStore('binance', config=binance_config),
    'coinbase': CCXTStore('coinbase', config=coinbase_config),
    'kraken': CCXTStore('kraken', config=kraken_config)
}
```

### Custom Data Routing
```python
# Route specific symbols to preferred exchanges
store.set_symbol_routing({
    'BTC/USDT': 'binance',
    'ETH/USD': 'coinbase',
    'ADA/EUR': 'kraken'
})
```

## Best Practices

1. **Connection Management**
   - Use connection pooling for high-frequency trading
   - Implement proper reconnection logic
   - Monitor connection health

2. **Error Handling**
   - Handle exchange-specific errors gracefully
   - Implement circuit breakers for unstable connections
   - Log all exchange interactions

3. **Performance Optimization**
   - Use data caching to reduce API calls
   - Batch operations when possible
   - Monitor rate limits

## See Also

- [Broker Component](brokers.md) - How brokers interact with the store
- [Data Feeds](feeds.md) - How feeds consume store data
- [Configuration](../reference/configuration.md) - Store configuration reference
