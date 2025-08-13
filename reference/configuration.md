# Configuration Reference

Key parameters for programmatic configuration of Cracktrader components.

CCXTStore (exchange connectivity)

- `exchange` (str): CCXT exchange id, e.g., `"binance"`
- `sandbox` (bool): Use testnet/paper endpoints when available
- `config` (dict): Passed to CCXT (e.g., `apiKey`, `secret`, `enableRateLimit`)
- `cache_enabled` (bool): Enable historical data caching
- `cache_dir` (str): Cache directory path
- `exchange_instance` (object): Preconfigured CCXT instance (advanced/testing)

Example

```python
from cracktrader.store import CCXTStore

store = CCXTStore(
    exchange='binance',
    sandbox=True,
    config={'apiKey': '...', 'secret': '...', 'enableRateLimit': True},
    cache_enabled=True,
    cache_dir='./data'
)
```

CCXTDataFeed (market data)

- `symbol` (str): e.g., `"BTC/USDT"`
- `ccxt_timeframe` (str): e.g., `"1m"`, `"1h"`, `"1d"`
- `live` (bool): Stream live data via WebSocket
- `historical_limit` (int): Max historical candles to load
- `fromdate`/`todate` (datetime|None): Historical window
- `reorder_buffer_size` (int): Out‑of‑order tick buffer size
- `reorder_buffer_timeout` (float): Buffer flush timeout (seconds)

Example

```python
from datetime import datetime
from cracktrader.feeds import CCXTDataFeed

feed = CCXTDataFeed(
    store=store,
    symbol='BTC/USDT',
    ccxt_timeframe='1h',
    historical_limit=2000,
    fromdate=datetime(2024,1,1),
    todate=datetime(2024,3,1)
)
```

Brokers (order execution)

- Paper/backtest: `BrokerFactory.create(mode='paper', cash=..., commission=..., slip_perc=...)`
- Live: `BrokerFactory.create(mode='live', store=store)`

Example

```python
from cracktrader.broker import BrokerFactory

# Paper
paper = BrokerFactory.create(mode='paper', cash=10_000, commission=0.001)

# Live
live = BrokerFactory.create(mode='live', store=store)
```

Logging (Python logging)

- Configure standard library logging levels/handlers per your environment.

Secrets

- Prefer environment variables for API keys (e.g., `BINANCE_API_KEY`, `BINANCE_SECRET`).
- Do not commit secrets to source control.
