# Getting Started

This quick tour introduces the Cracktrader core concepts and walks through a
minimal strategy. For deeper guidance, check the dedicated tutorials under
`getting_started/`.

## Installation

```bash
pip install cracktrader
```

Extras such as indicators and runnable examples now live in
[`cracktrader-extras`](https://github.com/LachlanBridges/cracktrader-extras).
Install it when you need the additional toolbox.

## Core ideas

Cracktrader exposes three factory functions:

- `Store(exchange=...)` – shared connection state for an exchange
- `Feed(symbol=..., exchange=..., store=...)` – Backtrader-compatible market data feed
- `Broker(mode=..., exchange=..., store=...)` – order routing built on top of the store

All three components mirror the CCXT integration and work identically for
Polymarket.

## Minimal example

```python
import backtrader as bt
from cracktrader import Broker, Cerebro, Feed, Store

class Momentum(bt.Strategy):
    params = dict(period=10)

    def __init__(self):
        close = self.datas[0].close
        self.avg = bt.indicators.SimpleMovingAverage(close, period=self.p.period)

    def next(self):
        price = self.data.close[0]
        if price > self.avg[0] and not self.position:
            self.buy()
        elif price < self.avg[0] and self.position:
            self.sell()

cerebro = Cerebro()
store = Store(exchange="polymarket")
feed = Feed(symbol="POLYMARKET:MARKET_ID", exchange="polymarket", store=store, live=False)
broker = Broker(mode="paper", exchange="polymarket", store=store)

cerebro.adddata(feed)
cerebro.setbroker(broker)
cerebro.addstrategy(Momentum)
results = cerebro.run()
```

The store is cached internally, so subsequent calls to `Feed` or `Broker` with
the same exchange reuse the existing connection automatically.

## Next steps

- Explore the `getting_started/quickstart.md` guide for a ready-to-run script.
- Review the [Exchange Concepts](core_concepts/exchanges.md) page to understand
  how store caching and routing work.
- Install `cracktrader-extras` for community indicators and examples.
