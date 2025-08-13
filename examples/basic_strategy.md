# Basic Strategy Example

A complete, runnable moving-average crossover strategy. This page includes the source directly from the examples directory so it stays in sync.

## Source

--8<-- "examples/moving_average_cross.py"

## Run It

```bash
python examples/moving_average_cross.py
```

## Notes

- Caching: set `cache_enabled=True` on the store for repeated backtests
- Timeframes: pass CCXT-style strings via `ccxt_timeframe` (e.g., `"1m"`, `"1h"`)
- Brokers: use the backtesting broker for research; switch to a live broker via the BrokerFactory for production

### Add Stop Loss
```python
def next(self):
    if not self.position and self.crossover > 0:
        # Entry with stop loss
        entry_price = self.data.close[0]
        stop_price = entry_price * 0.95  # 5% stop loss

        self.buy_bracket(
            size=size,
            price=entry_price,
            stopprice=stop_price
        )
```

## Next Steps

- [Multi-Asset Strategy](multi_asset.md) - Trade multiple symbols
- [Live Trading](live_trading.md) - Deploy to production
- [Web Dashboard](web_dashboard.md) - Monitor with web interface
- [Performance Guide](../performance/overview.md) - Optimize for speed
