# CrackTrader

Professional crypto trading framework for traders and quants

Connect to 100+ CCXT exchanges. Backtest with precision. Deploy with confidence.

---

## Overview

CrackTrader lets you build, test, and deploy algorithmic strategies across CCXT-supported exchanges with native Backtrader compatibility. It’s designed for practitioners who care about execution quality, data integrity, and reproducibility.

## Key Benefits

- Fast development: Unified exchange layer via CCXT
- Solid data: WebSocket streaming and historical caching
- Reliable: Comprehensive tests and clear failure modes
- Web-ready: REST API and dashboard integrations
- Performance: Async pipeline with sub-minute timeframes

---

## Quick Start

### 1. Install

```bash
pip install git+https://github.com/LachlanBridges/cracktrader.git
```

### 2. Your First Strategy (backtest)

```python
import backtrader as bt
from cracktrader import CCXTStore, CCXTDataFeed

class SimpleMovingAverage(bt.Strategy):
    def __init__(self):
        self.sma = bt.indicators.SMA(period=20)

    def next(self):
        if not self.position and self.data.close[0] > self.sma[0]:
            self.buy()
        elif self.position and self.data.close[0] < self.sma[0]:
            self.sell()

# Connect to an exchange (shared CCXT store)
store = CCXTStore(exchange='binance', cache_enabled=True)
data = CCXTDataFeed(store=store, symbol='BTC/USDT', ccxt_timeframe='1h')

cerebro = bt.Cerebro()
cerebro.adddata(data)
cerebro.addstrategy(SimpleMovingAverage)
cerebro.run()
```

### 3. Go Live (when ready)

```python
# Switch to live trading
store = CCXTStore(
    exchange='binance',
    sandbox=False,
    config={'apiKey': '...', 'secret': '...'}
)
```

Use Backtrader’s broker for backtesting. Switch to `CCXTLiveBroker` for live execution. The store uses a registry so your broker and feeds share the same connection automatically.

---

## Explore the Docs

- Getting Started: [Quickstart](getting_started/quickstart.md), [Installation](getting_started/installation.md), [Configuration](getting_started/configuration.md), [First Strategy](getting_started/first_strategy.md)
- How‑to: [Strategy Cookbook](strategy_guide.md), [Web API](WEB_API.md), [Backtrader Compatibility](integrations/backtrader_compat.md)
- Concepts: [Architecture](core_concepts/architecture.md), [Strategies](core_concepts/strategies.md), [Feeds](core_concepts/feeds.md), [Brokers](core_concepts/brokers.md), [Exchanges](core_concepts/exchanges.md), [Caching](core_concepts/caching.md)
- Reference: [Web API Reference](reference/web_api.md), [Configuration Reference](reference/configuration.md)
- Development: [Performance](performance/overview.md), [Testing Methodology](testing/known_gaps.md), [Contributing](development/README.md)

---

## For Traders & Quants

- CCXT integration: Reliable connectivity, unified symbols, thorough order support
- Backtrader native: Run existing strategies with minimal changes
- Web API: FastAPI server plus dashboard for monitoring and control
- Performance: Async streaming, caching, and sub‑minute intervals
- Quality: Unit, integration, and end‑to‑end test coverage

---

## Architecture Overview

```mermaid
graph TB
    A[Trading Strategies] --> B[Cerebro Engine]
    B --> C[Data Feeds]
    B --> D[Brokers]
    C --> E[CCXT Store]
    D --> E
    E --> F[Exchanges]

    G[Web Dashboard] --> H[REST API]
    H --> B

    subgraph "100+ Exchanges"
        F1[Binance]
        F2[Coinbase]
        F3[Kraken]
        F4[Bybit]
    end

    F --> F1
    F --> F2
    F --> F3
    F --> F4
```

Data flows from exchanges through CCXT to your strategies. The web API provides real-time monitoring and control. See [Performance](performance/overview.md) for current characteristics and benchmarks.

---

## Ready to Start?

### New to Algorithmic Trading?
→ [Quickstart Tutorial](getting_started/quickstart.md) - Learn the basics with guided examples

### Experienced with Backtrader?
→ [Migration Guide](integrations/backtrader_compat.md) - Adapt existing strategies in minutes

### Building for Production?
→ [Architecture Guide](core_concepts/architecture.md) - Understand the system design

### Need API Integration?
→ [Web API Reference](reference/web_api.md) - Complete REST and WebSocket documentation

---
