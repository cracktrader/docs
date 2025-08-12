# CrackTrader Documentation

Cryptocurrency trading framework that connects CCXT (400+ exchanges) with Backtrader (backtesting engine).

## What it does

- **[CCXT](https://github.com/ccxt/ccxt)** - Connect to 400+ cryptocurrency exchanges
- **[Backtrader](https://github.com/mementum/backtrader)** - Backtesting and strategy development
- **Web interface** - FastAPI + React dashboard for monitoring

## Features

- **Trading modes**: Backtesting, paper trading, live trading
- **Data**: Historical data caching, real-time WebSocket feeds
- **Exchanges**: 400+ exchanges via CCXT
- **Testing**: 89 test files, 2:1 test-to-source ratio
- **Monitoring**: Health checks, structured logging
- **Web API**: FastAPI + React dashboard

## Quick Start

### 1. Installation

```bash
# Basic installation
pip install git+https://github.com/LachlanBridges/cracktrader.git

# With web API and dashboard
pip install "cracktrader[web] @ git+https://github.com/LachlanBridges/cracktrader.git"

# Development setup
git clone https://github.com/LachlanBridges/cracktrader.git
cd cracktrader
pip install -e ".[dev]"
```

### 2. Your First Strategy

```python
import backtrader as bt
from cracktrader import Cerebro, CCXTStore, CCXTDataFeed

class MovingAverageCross(bt.Strategy):
    params = (('fast_period', 10), ('slow_period', 30))

    def __init__(self):
        self.fast_ma = bt.indicators.SMA(period=self.p.fast_period)
        self.slow_ma = bt.indicators.SMA(period=self.p.slow_period)
        self.crossover = bt.indicators.CrossOver(self.fast_ma, self.slow_ma)

    def next(self):
        if not self.position and self.crossover > 0:
            # OCO order with stop-loss and take-profit
            size = self.broker.get_cash() * 0.95 / self.data.close[0]
            entry_price = self.data.close[0]

            self.buy_bracket(
                size=size,
                price=entry_price,
                stopprice=entry_price * 0.98,   # 2% stop loss
                limitprice=entry_price * 1.04   # 4% take profit
            )

# Setup with caching
store = CCXTStore(exchange='binance', cache_enabled=True)
data = CCXTDataFeed(store=store, symbol='BTC/USDT', ccxt_timeframe='1h')

cerebro = Cerebro()
cerebro.adddata(data)
cerebro.addstrategy(MovingAverageCross)
cerebro.run()
```

**Run the complete example:**
```bash
python examples/moving_average_cross.py
```

### 3. Web API (Optional)

Enable the REST API internally within cerebro - no standalone server needed:

```python
# Enable web API in cerebro
cerebro = Cerebro()
cerebro.adddata(data)
cerebro.addstrategy(MovingAverageCross)

# Run with integrated web API
results = cerebro.run(web_api=True, web_host="0.0.0.0", web_port=8000)
```

**Key Features:**

- Integrated with cerebro execution - no separate process
- Multi-client support - multiple frontends can connect simultaneously
- Real-time strategy monitoring - WebSocket streams for live data
- RESTful endpoints - full strategy and market data access

Access API documentation: `http://localhost:8000/docs`

## Web API Integration

### Frontend Setup

The React frontend is completely standalone and connects to any running API:

```bash
# Frontend connects to any API endpoint
cd src/cracktrader/web/frontend
npm install
npm start  # Connects to http://localhost:8000 by default
```

### API Endpoints

The integrated web API provides comprehensive access to your trading system:

1. **Strategy Management**
   - `GET /api/v1/strategies` - List all running strategies
   - `POST /api/v1/strategies/{id}/start` - Start a strategy
   - `POST /api/v1/strategies/{id}/stop` - Stop a strategy

2. **Portfolio Data**
   - `GET /api/v1/portfolio` - Current portfolio value and positions
   - `GET /api/v1/portfolio/history` - Historical performance data
   - `GET /api/v1/positions` - Current open positions

3. **Market Data**
   - `GET /api/v1/data/{symbol}` - Latest market data for symbol
   - `GET /api/v1/candles/{symbol}` - Historical candlestick data
   - `WS /ws/data` - Real-time market data stream

4. **System Health**
   - `GET /api/v1/health` - System health status
   - `GET /api/v1/status` - Trading system status
   - `GET /api/v1/metrics` - Performance metrics

## Architecture Overview

    ┌─────────────────┐         ┌─────────────────┐
    │   Strategies    │         │   Web Dashboard │
    │   (Backtrader)  │         │     (React)     │
    └─────────┬───────┘         └─────────┬───────┘
              │                           │
              │ (loaded into)        (HTTP requests)
              │                           │
              ▼                   ┌───────▼───────┐
    ┌─────────────────┐           │   REST API    │
    │     Cerebro     │◄──────────│   (FastAPI)   │
    │   (Execution)   │(accesses) └───────────────┘
    └─────────┬───────┘
              │ (connects to)
              │
              ├─────────────────────┐
              │                     │
              ▼                     ▼
    ┌─────────────────┐   ┌─────────────────┐
    │   Data Feeds    │   │     Brokers     │
    │ (CCXTDataFeed)  │   │ (Live/Back/Ppr) │
    └─────────┬───────┘   └─────────┬───────┘
              │                     │
              │ (reads from)        │ (trades via)
              │                     │
              └─────────┬───────────┘
                        │
                        ▼
          ┌─────────────────────────────────┐
          │           CCXT Store            │
          │      (Exchange Interface)       │
          └─────────────┬───────────────────┘
                        │ (connects to)
                        ▼
    ┌─────────────────────────────────────────┐
    │          Exchange Ecosystem             │
    │    Binance  │  Coinbase  │  Kraken      │
    │   Bybit │ OKX │ Bitget │ Gate.io │ ... │
    └─────────────────────────────────────────┘

    Data Flow:
    • Strategies ──→ Cerebro ──→ Data Feeds ──→ Store ──→ Exchanges
    • Orders: Cerebro ──→ Brokers ──→ Store ──→ Exchanges
    • API: Web Dashboard ──→ REST API ──→ Cerebro (monitoring/control)

## What Makes Cracktrader Different?

Built on Backtrader with full compatibility for existing strategies and indicators, plus:

### CCXT Integration

- Trade on 400+ CCXT-supported exchanges
- Real-time WebSocket streaming
- Unified symbols across exchanges
- Multiple asset types (spot, futures, margin)

### Performance & Scale

- Sub-minute intervals (1s, 10s, 30s candle support)
- Async architecture with non-blocking I/O
- Automatic historical data caching
- Vectorized backtesting for optimization

### Integrated Web API

- REST API runs internally within cerebro execution
- Multiple frontends can connect to same API instance
- Real-time strategy performance monitoring
- Standalone React frontend connects via HTTP

### Production Features

- System health checks and alerts
- Structured JSON logging
- Automatic reconnection and retry logic
- Full type hints and validation

## Documentation Structure

### Getting Started

- [Installation & Setup](getting_started.md) - Complete installation guide
- [First Strategy](getting_started.md#your-first-strategy) - Build and run your first trading strategy
- [CLI Guide](getting_started.md#cli-tools) - Command-line tools and utilities

### Core Concepts

- [Strategy Development](strategy_guide.md) - Advanced strategy patterns and best practices
- [Data Feeds](feeds.md) - Working with real-time and historical data
- [Broker Integration](brokers.md) - Order types, execution modes, and linking
- [Backtrader Compatibility](backtrader_compat.md) - Using existing Backtrader features

### Advanced Topics

- [Testing Infrastructure](testing.md) - Unit, integration, and end-to-end testing
- [Advanced Configuration](advanced.md) - Performance tuning and custom setups
- [Extending Cracktrader](extending.md) - Adding new exchanges, feeds, and brokers
- [Web API & Dashboard](gui.md) - REST API usage and frontend development

### Reference

- [API Documentation](WEB_API.md) - Complete REST API reference
- [Configuration Reference](advanced.md#configuration) - All configuration options
- [CLI Reference](getting_started.md#cli-reference) - Command-line tool documentation

## Example Strategies

All examples are runnable files in the `examples/` directory:

### Moving Average Crossover

```bash
python examples/moving_average_cross.py
```

- OCO bracket orders with automatic stop-loss/take-profit
- Configurable MA periods and risk parameters
- Built-in caching for faster backtests

### Mean Reversion (RSI)

```bash
python examples/mean_reversion.py
```

- RSI-based oversold/overbought signals
- Sub-minute timeframe support (15m candles)
- Risk management with OCO orders

### Live Data Streaming

```bash
python examples/live_data_feed.py
```

- Real-time WebSocket data feeds
- Multiple exchange support
- Health monitoring integration

### Web API Integration

```bash
python examples/web_api_strategy.py
```

- Integrated API server within cerebro execution
- Real-time strategy monitoring via HTTP endpoints
- Standalone React frontend connects to API
- Multiple dashboard support

## Community & Support

- **Issues & Features** - [GitHub Issues](https://github.com/your-repo/cracktrader/issues)
- **Documentation** - This comprehensive guide
- **Examples** - See `examples/` directory for complete use cases
- **API Reference** - Interactive docs at `/docs` when running web server

## License

MIT License - see [LICENSE](../LICENSE) for details.

---

**Ready to start trading?** → [Installation Guide](getting_started.md)
