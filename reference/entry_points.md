# Entry Points & Public API Freeze

This document defines the user-facing entry points that Cracktrader guarantees going forward. Any library or script that targets these APIs can rely on their signatures and behaviour remaining stable, with only additive options being introduced in the future.

## Public Baseline

The following callables are the foundational, low-level surface area. They continue to work exactly as they do today and only receive backwards-compatible enhancements:

- `ct.Store(exchange: str, **kwargs)` – returns a cached/exchange-aware store instance.
- `ct.Feed(*, symbol: str | None = None, exchange: str | None = None, store=None, timeframe="1m", granularity=None, **kwargs)` – returns an exchange-specific data feed while sharing the appropriate store automatically.
- `ct.Broker(mode: str = "paper", exchange: str | None = None, store=None, **kwargs)` – returns the correct broker for the requested mode using the shared store.
- `ct.Cerebro(**kwargs)` – the Backtrader engine with Cracktrader enhancements (performance profiling, store propagation, vectorized mode, etc.).

**Freeze rule:** Parameter names and required positional arguments for the above functions are locked. Future changes may only add optional keyword arguments with safe defaults.

## High-Level Session Helper

To simplify day-to-day usage, the library will expose `ct.exchange(name, **options)` which returns an `ExchangeSession`. Sessions wrap the low-level factories so end users configure connections once and request feeds or brokers on demand.

### Session Construction

```python
session = ct.exchange(
    name="binance",            # Exchange routing key (case-insensitive)
    mode="paper",              # Default broker mode for this session
    instrument_type="spot",    # Optional: shared instrument type
    api_key="...",             # Optional credentials / config
    secret="...",
    enable_network=True,
    sandbox=False,
    store_options={...},        # Optional nested overrides
)
```

Implementations must:

1. Normalize the exchange name and dispatch through `EXCHANGE_REGISTRY` so custom integrations can supply their own session subclasses.
2. Create or reuse the shared store via `ct.Store`, honouring cached instances and `config_overrides` just like the factories do today.
3. Record default kwargs (mode, instrument type, etc.) for later feed/broker calls.

### Core Session API

`ExchangeSession` exposes the following stable surface:

- `session.store` (property): returns the underlying store instance for advanced manipulation.
- `session.feed(*, **feed_kwargs)` → `DataFeed`: builds a single feed using the stored defaults and returns it to the caller.
- `session.feeds(descriptors)` → `list[DataFeed]`: accepts an iterable of dictionaries/objects describing feeds (e.g., symbols, market ids) and returns a list with one feed per descriptor. Internally each call delegates to `session.feed`.
- `session.broker(*, mode: str | None = None, **broker_kwargs)` → `Broker`: creates a broker reusing the session store. `mode` defaults to the session’s mode but can be overridden ad-hoc.
- `session.close()` → `None`: releases resources when the underlying store spins background loops (primarily Polymarket).
- `session.summary()` / `session.dict()` (optional helpers): provide human-readable diagnostics; must not mutate state.

All methods return the constructed objects; nothing mutates a `Cerebro` automatically. Users remain in charge of wiring:

```python
cerebro = ct.Cerebro()
cerebro.addstrategy(MyStrategy)

# Multiple feeds example
for feed in session.feeds([
    {"symbol": "BTC/USDT", "timeframe": "5m"},
    {"symbol": "ETH/USDT", "granularity": "tick"},
]):
    cerebro.adddata(feed)

cerebro.addbroker(session.broker())
results = cerebro.run()
```

### Built-In Sessions

Two concrete subclasses must ship with the helper:

- `CCXTSession`: default for any exchange handled by the CCXT stack. It manages sandbox/live toggles, timeframe defaults, and passes through CCXT-specific kwargs.
- `PolymarketSession`: tailored for prediction markets. It prefers prediction-mode defaults (`granularity="tick"`, `instrument_type="prediction"`), exposes `discover_event_markets(...)` and `feeds_for_event(...)` helpers alongside `feed_binary(...)`, and still delegates to `.feed` internally. Metadata is hydrated lazily—sessions fetch only the requested events, persist them under `~/.cache/cracktrader/polymarket`, and spin up a background refresh so the complete catalogue arrives without blocking user flows.

Third-party integrations can register additional sessions via:

```python
register_exchange(
    "myexchange",
    store_cls=..., broker_cls=..., feed_factory=...,
    session_cls=MyCustomSession,
)
```

## Behaviour Guarantees

- **Store sharing:** Multiple feeds/brokers created through a session reuse the same cached store instance unless a custom store is supplied explicitly.
- **Registry routing:** All session methods must respect `EXCHANGE_REGISTRY` entries so out-of-tree exchanges behave like first-class citizens.
- **Polymarket parity:** Prediction markets stay aligned with CCXT semantics (instrument typing, commission routing, name formats) regardless of whether users call factories directly or go through sessions.

## Testing Requirements

To enforce the freeze:

1. Add contract tests under `tests/public_api/test_entry_points.py` that assert callable signatures using `inspect.signature` for `ct.Store`, `ct.Feed`, `ct.Broker`, `ct.Cerebro`, `ct.exchange`, and key `ExchangeSession` methods.
2. Introduce a smoke test that builds a session, creates multiple feeds and a broker, wires them into `ct.Cerebro`, and runs a no-op strategy to verify high-level behaviour.
3. Extend example smoke tests (in `cracktrader-extras`) to exercise the session workflow so documentation remains accurate.

## Documentation Responsibilities

- Update quick-start and cookbook pages to feature sessions as the primary onboarding path, while keeping a “manual wiring” section for advanced users.
- Record the freeze in the changelog and release notes so downstream consumers are aware of the guaranteed surface.
- Keep `AGENTS.md` aligned with this document so automated contributors follow the same contract.

## Change Control

Any future modification to these entry points must satisfy all of the following:

1. Maintain backwards compatibility with existing scripts (no breaking removals or required parameter changes).
2. Include updates to the contract tests described above.
3. Document the change in this reference file and the changelog.
4. If a breaking change is unavoidable, it must be called out explicitly and paired with a major/minor version bump per semantic versioning.

By adhering to this freeze, Cracktrader presents a stable, predictable interface to end users while leaving room for additive improvements and new exchange integrations.
