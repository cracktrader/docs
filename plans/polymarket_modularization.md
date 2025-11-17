# Polymarket Modularization Plan

Intent: make Polymarket components easier to evolve and swap by untangling the current monoliths (`feeds/polymarket.py`, `store/polymarket_store.py`, `utils/polymarket_helpers.py`) into smaller, composable modules, and by unifying shared broker simulation logic.

## Target package layout

```
src/cracktrader/polymarket/
    __init__.py
    feed/
        __init__.py
        atomic.py          # single-outcome feed
        composite.py       # binary/bin aggregations, outcome snapshots
        history.py         # preload, prices->candles, timeframe/granularity helpers
    store/
        __init__.py
        core.py            # orchestration + background loop
        metadata.py        # metadata service + cache
        ws.py              # websocket pump + handlers
        rest.py            # REST clients/history fetch
    utils/
        __init__.py
        symbols.py         # symbol/outcome resolution helpers
        timeframes.py      # granularity conflict handling, tf math
```

Existing files map:
- `feeds/polymarket.py` -> feed/atomic.py + feed/composite.py + feed/history.py.
- `store/polymarket_store.py` -> store/core.py with small imports from store/ws.py, store/metadata.py, store/rest.py.
- `utils/polymarket_helpers.py` -> utils/symbols.py + utils/timeframes.py + feed/history.py (candles transform).
- `store/polymarket/ws_pump.py`, `ws_handlers.py`, `ohlcv.py` -> folded into store/ws.py + feed/history.py.

## Broker convergence

Goal: one shared simulation/back base that both CCXT and Polymarket wrappers delegate to. Extract the pending-order evaluation/fill loop from `broker/base_back_broker.py` and `broker/polymarket_simulation_broker.py` into a common mixin/base with small hooks (`supports_stop`, `supports_market`, `price_from_data`, `reject_reason`).

## Incremental steps

1) Carve helper modules without moving public imports by adding a new `polymarket` package that re-exports current symbols. Start by relocating pure helpers (timeframe/candle transforms, symbol resolution) from `utils/polymarket_helpers.py` into `polymarket/utils`, updating internal imports only.
2) Split `feeds/polymarket.py` into atomic vs composite vs history helpers; keep a shim that re-exports existing class names to avoid breaking imports.
3) Move websocket pump/handlers into `polymarket/store/ws.py`; keep thin import shims in the old locations.
4) Extract shared back/paper simulation base and update CCXT/Polymarket back brokers to subclass it; remove duplicated fill/eval code.
5) After shims are in place, update docs/import paths and optional deprecation warnings; remove shims in a follow-up release once downstreams are migrated.

## Testing impact

- Unit: adjust import paths in Polymarket-specific tests; keep behaviour identical. Add focused tests for new helper modules (timeframe conflict resolution, price->candle transforms).
- Integration/E2E: rerun existing Polymarket coverage; no behavioural changes expected.
- Lint: ensure new package is picked up by existing ruff settings; keep ASCII.

## Notes

- Preserve API freeze: public `ct.Feed/Store/Broker` and `ct.exchange` must remain unchanged; new modules are internal.
- Optional deps remain guarded; keep network defaults off in tests.
