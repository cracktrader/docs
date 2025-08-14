# Tested Exchanges

Cracktrader supports hundreds of cryptocurrency exchanges through integration with exchange APIs. Here's the current testing status for major exchanges.

## Thoroughly Tested Exchanges

These exchanges have been extensively tested with Cracktrader and are recommended for production use:

### **Binance** ⭐⭐⭐⭐⭐
- **Spot Trading**: ✅ Full support
- **Futures Trading**: ✅ Full support
- **Margin Trading**: ✅ Full support
- **WebSocket Streams**: ✅ All streams tested
- **Order Types**: ✅ All supported types
- **Special Features**: OCO orders, bracket orders, advanced time-in-force

### **Coinbase Pro** ⭐⭐⭐⭐
- **Spot Trading**: ✅ Full support
- **WebSocket Streams**: ✅ All streams tested
- **Order Types**: ✅ Market, limit, stop orders
- **Special Features**: Advanced order routing

### **Kraken** ⭐⭐⭐⭐
- **Spot Trading**: ✅ Full support
- **Futures Trading**: ✅ Full support
- **Margin Trading**: ✅ Full support
- **WebSocket Streams**: ✅ All streams tested
- **Special Features**: Advanced order options

## Well Tested Exchanges

These exchanges have good test coverage and are suitable for most trading strategies:

### **FTX** ⭐⭐⭐⭐
- **Spot Trading**: ✅ Full support
- **Futures Trading**: ✅ Full support
- **WebSocket Streams**: ✅ Tested
- **Order Types**: ✅ Most types supported

### **Huobi** ⭐⭐⭐
- **Spot Trading**: ✅ Full support
- **Futures Trading**: ⚠️ Limited testing
- **WebSocket Streams**: ✅ Tested

### **OKX** ⭐⭐⭐
- **Spot Trading**: ✅ Full support
- **WebSocket Streams**: ✅ Tested
- **Order Types**: ✅ Basic types

### **Bybit** ⭐⭐⭐
- **Spot Trading**: ✅ Full support
- **Derivatives Trading**: ⚠️ Limited testing
- **WebSocket Streams**: ✅ Tested

## Basic Testing

These exchanges have basic functionality tested but may require additional validation:

### **KuCoin** ⭐⭐
- **Spot Trading**: ✅ Basic support
- **WebSocket Streams**: ⚠️ Limited testing

### **Gate.io** ⭐⭐
- **Spot Trading**: ✅ Basic support
- **Order Types**: ⚠️ Basic types only

### **Bitfinex** ⭐⭐
- **Spot Trading**: ✅ Basic support
- **Margin Trading**: ⚠️ Limited testing

## Additional Supported Exchanges

Cracktrader supports **400+ exchanges** through standard APIs. While not all have been extensively tested, most standard functionality should work. Supported exchanges include:

- Bitstamp, Bittrex, Poloniex, HitBTC
- Gemini, CEX.IO, BitMEX, Deribit
- And many more regional and specialized exchanges

## Testing Methodology

Our testing approach includes:

1. **Functional Testing**
   - Order placement and execution
   - Data feed accuracy
   - WebSocket connectivity
   - Error handling

2. **Performance Testing**
   - Latency measurements
   - Throughput testing
   - Stress testing under load

3. **Integration Testing**
   - Multi-exchange scenarios
   - Complex order workflows
   - Real-time data synchronization

4. **Production Validation**
   - Live trading verification
   - Extended monitoring periods
   - Community feedback integration

## Recommendations

### For Production Trading
Use **thoroughly tested** exchanges (5-star rating) for live trading with real funds.

### For Development & Testing
**Well tested** exchanges (3-4 stars) are suitable for development and paper trading.

### For Specific Use Cases
Contact our support team for guidance on exchange selection for specialized requirements.

## Contributing Test Results

Help improve our exchange coverage by:
- Reporting issues with specific exchanges
- Sharing successful configurations
- Contributing test cases for untested features

## See Also

- [Known Limitations](known_gaps.md) - Current testing gaps
- [Configuration](../reference/configuration.md) - Exchange configuration examples
- [Support](../support/contact.md) - Report exchange issues
