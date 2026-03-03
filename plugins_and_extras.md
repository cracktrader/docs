# Plugins & Extras

Cracktrader keeps the core library focused on exchange connectivity, stores,
feeds, and brokers. Optional indicators, analyzers, strategies, and runnable
examples now live in the unified companion package named
[`cracktrader_extras`](https://github.com/cracktrader/cracktrader-extras).

## Installing extras

Extras are published as a separate wheel. Install it alongside the core package
when you need the additional components:

```bash
pip install cracktrader_extras
```

The extras project registers entry points so Cracktrader can discover optional
features at runtime. Use `cracktrader.load(group)` to fetch all plugins for a
specific group:

```python
import cracktrader

indicators = cracktrader.load("cracktrader.plugins.indicators")
print(indicators.keys())
```

If no extras are installed `load()` returns an empty dictionary, allowing you to
write fallback logic easily.

## Available entry point groups

The core library reserves three plugin namespaces:

- `cracktrader.plugins.indicators`
- `cracktrader.plugins.analyzers`
- `cracktrader.plugins.strategies`

Custom projects can contribute to these groups to register new functionality
without modifying core code. Each entry point should resolve to an object (for
example an indicator class) that Cracktrader can instantiate or expose.

## Working inside the extras repo

`cracktrader_extras` packages community indicators and example strategies. It also
doubles as an integration test suite because the runnable examples are executed
as smoke tests in CI. Refer to that repository for contribution guidelines and
additional documentation.
