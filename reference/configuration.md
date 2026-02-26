# Configuration Reference

Key parameters for programmatic configuration of Cracktrader components.

CCXTStore (exchange connectivity)

- `exchange` (str): CCXT exchange id, e.g., `"binance"`
- `sandbox` (bool): Use testnet/paper endpoints when available
- `config` (dict): Passed to CCXT (e.g., `apiKey`, `secret`, `enableRateLimit`)
- `cache_enabled` (bool): Enable historical data caching (default: `True`)
- `cache_dir` (str): Cache directory path
- `exchange_instance` (object): Preconfigured CCXT instance (advanced/testing)

Example

```python
from cracktrader.store import CCXTStore

store = CCXTStore(
    exchange='binance',
    sandbox=True,
    config={'apiKey': '...', 'secret': '...', 'enableRateLimit': True},
    cache_dir='./data'
)
```

Cache invalidation and refresh controls

- Config (`read_policy`):
  - `cache_enabled` (bool, default `true`)
  - `cache_dir` (str, default `"data"`)
  - `refresh_cache` (bool, default `false`) for store-level default refresh behavior
- API:
  - `store.invalidate_cache(exchange=..., symbol=..., timeframe=..., instrument_type=...)`
  - `store.fetch_historical_ohlcv(..., refresh_cache=True)` to force refresh a stream
- CLI:
  - `cracktrader data invalidate --exchange binance --instrument spot --symbol BTC/USDT --timeframe 1m`
  - `cracktrader data backfill --exchange binance --symbol BTC/USDT --timeframe 1m --refresh`

Determinism and stale-data semantics

- Default behavior (`refresh_cache=False`): cached candles are reused when they satisfy the request.
- `refresh_cache=True`: matching cache scope is invalidated before remote fetch, then replaced with fresh data.
- `use_cache=False`: bypass read-path cache usage for that call.

Reference benchmark delta (2026-02-26, local dev sample)

- Dataset: 2,000 1m candles, simulated remote latency.
- First fetch (cold path): `73.82 ms`
- Second fetch (cache hit): `16.54 ms`
- Observed speedup: `4.46x`

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
- Backtest execution realism (v1 slippage): `execution_realism={"slippage": {"model": "fixed_bps", "bps": 10.0}}`
  - Buy fills: `price * (1 + bps/10000)`
  - Sell fills: `price * (1 - bps/10000)`
  - Limit orders remain price-protected (slippage cannot violate the limit bound)
  - Default remains `0` bps when not configured
- Backtest deterministic latency model (v1):
  - `execution_realism={"latency": {"model": "bar_delay", "bars": 1}}`
  - Semantics: defer order fill/trigger evaluation by N engine bars after submission
  - Default remains `0` bars when not configured

Example

```python
from cracktrader.broker import BrokerFactory

# Paper
paper = BrokerFactory.create(mode='paper', cash=10_000, commission=0.001)

# Backtest/paper with deterministic fixed-bps slippage
paper_slippage = BrokerFactory.create(
    mode='paper',
    cash=10_000,
    commission=0.001,
    execution_realism={"slippage": {"model": "fixed_bps", "bps": 10.0}},
)

# Backtest/paper with deterministic 1-bar execution delay
paper_latency = BrokerFactory.create(
    mode='paper',
    cash=10_000,
    execution_realism={"latency": {"model": "bar_delay", "bars": 1}},
)

# Live
live = BrokerFactory.create(mode='live', store=store)
```

Logging (Python logging)

- Configure standard library logging levels/handlers per your environment.

Secrets

- Prefer environment variables for API keys (e.g., `BINANCE_API_KEY`, `BINANCE_SECRET`).
- Do not commit secrets to source control.
