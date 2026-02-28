# Using Cracktrader as an Arbitrage Substrate

This page describes how to use Cracktrader as a reusable connectivity layer for an external arbitrage runner.

## Why this pattern

Keep Cracktrader focused on:
- exchange sessions/stores/brokers
- normalized venue capabilities
- deterministic paper/backtest behavior

Keep strategy-specific arbitrage orchestration in a separate repo/service.

## Stable snapshot surface

`CCXTStore` exposes synchronous wrappers intended for scanner loops:

- `fetch_ticker(symbol) -> dict`
- `fetch_order_book(symbol, limit=20) -> dict`
- `fetch_trades(symbol, limit=200) -> list[dict]`

Use these wrappers instead of `store.exchange.*` to avoid coupling to raw exchange clients.

## Capability checks

You can preflight snapshot support through session capability composition:

```python
import cracktrader as ct

session = ct.exchange("binance", mode="paper")
view = session.capability_view()

assert view["composed"]["supports_ticker_snapshot"]
assert view["composed"]["supports_orderbook_snapshot"]
assert view["composed"]["supports_public_trades_snapshot"]
```

## Multi-session polling example

```python
import time
import cracktrader as ct

EXCHANGES = ["binance", "okx", "kucoin"]
SYMBOL = "BTC/USDT"

sessions = [ct.exchange(name, mode="paper") for name in EXCHANGES]
try:
    while True:
        rows = []
        now_ms = int(time.time() * 1000)
        for session in sessions:
            store = session.store
            ticker = store.fetch_ticker(SYMBOL)
            book = store.fetch_order_book(SYMBOL, limit=20)
            rows.append(
                {
                    "exchange": session.exchange,
                    "symbol": SYMBOL,
                    "ts_ms": now_ms,
                    "bid": ticker.get("bid"),
                    "ask": ticker.get("ask"),
                    "top_bid_size": (book.get("bids") or [[None, None]])[0][1],
                    "top_ask_size": (book.get("asks") or [[None, None]])[0][1],
                }
            )
        print(rows)
        time.sleep(2.0)
finally:
    for session in sessions:
        session.close()
```

## Notes

- For production scanners, add stale-data checks and per-venue timeouts.
- Use `session.close()` to release store resources cleanly.
- Keep execution orchestration and strategy logic outside Cracktrader core.
