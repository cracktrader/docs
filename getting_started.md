# Getting Started

This page orients you to the fastest path based on your goal. For details, follow the linked guides.

Quick Links

- Quickstart: getting started in minutes
- Installation: full setup options
- First Strategy: step‑by‑step tutorial
- Configuration: exchange keys and settings

Prerequisites

- Python 3.11+ recommended (3.9+ supported)
- pip and Git for development

Quickstart

See Getting Started → Quickstart for a minimal, copy‑paste run.

First Strategy (outline)

```python
import backtrader as bt
from datetime import datetime
from cracktrader import Cerebro, CCXTStore, CCXTDataFeed

class MovingAverageCross(bt.Strategy):
    params = (('fast_period', 10), ('slow_period', 30))
    def __init__(self):
        fast = bt.indicators.SMA(self.data.close, period=self.p.fast_period)
        slow = bt.indicators.SMA(self.data.close, period=self.p.slow_period)
        self.crossover = bt.indicators.CrossOver(fast, slow)
    def next(self):
        if not self.position and self.crossover > 0:
            self.buy()
        elif self.position and self.crossover < 0:
            self.sell()

cerebro = Cerebro()
cerebro.addstrategy(MovingAverageCross)

store = CCXTStore(exchange='binance')
feed = CCXTDataFeed(
    store=store,
    symbol='BTC/USDT',
    ccxt_timeframe='1h',
    fromdate=datetime(2024, 1, 1),
    todate=datetime(2024, 3, 1),
)
cerebro.adddata(feed)
cerebro.broker.setcash(10000)
cerebro.broker.setcommission(commission=0.001)
cerebro.run()
```

Paper and Live

- Paper: keep Backtrader broker, use live data feed
- Live: attach `BrokerFactory.create(mode='live', store=store)` and provide API keys

Troubleshooting

- Import errors: run `pip install -e .`
- No data: check symbol format and timeframe support
- Exchange auth: verify API key/secret and permissions

```yaml
logging:
  level: "INFO"  # DEBUG, INFO, WARNING, ERROR, CRITICAL
  console_enabled: true
  file_enabled: true
  json_enabled: false  # Set true for structured logs
  log_dir: "logs"
  max_file_size: 10485760  # 10MB
  backup_count: 5
```

### Trading Configuration

```yaml
trading:
  default_exchange: "binance"
  default_timeframe: "1h" 
  sandbox_mode: true
  commission_rate: 0.001  # 0.1%
  slippage_rate: 0.0001   # 0.01%
```

### Web Configuration

```yaml
web:
  host: "0.0.0.0"
  port: 8000
  cors_enabled: true
  cors_origins: ["http://localhost:3000"]
  debug: false
  workers: 1
```

## What's Next?

- **[Strategy Development](strategy_guide.md)** - Learn advanced strategy patterns
- **[Data Feeds](feeds.md)** - Working with multiple data sources
- **[Web API](gui.md)** - Building applications with the REST API
- **[Testing](testing.md)** - Writing tests for your strategies

## Examples

Check out the `examples/` directory for complete implementations:

- `basic_backtest.py` - Simple backtest example
- `live_data_feed.py` - Real-time data consumption
- `logging_and_monitoring_demo.py` - Production monitoring setup
- `web_api_example.py` - Complete API workflow

---

**Having issues?** Check the [troubleshooting section](#common-issues) or open an issue on GitHub.
