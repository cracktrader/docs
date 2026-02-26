# Analyzer Bundles

`cracktrader.analyzers.bundles` provides mode defaults and optional venue presets.

## Presets

- `backtest_default`: `equity_curve`, `trade_log`, `decision_log`
- `paper_default`: `equity_curve`, `trade_log`, `decision_log`
- `live_default`: `equity_curve`, `decision_log`

## Venue Presets

- `prediction_market`: `decision_log`, `trade_log`
- Auto-mapped when `venue_preset=True` and venue is `polymarket` or `kalshi`

## Config Shape

```python
config = {
    "bundle": "live_default",      # optional
    "venue_preset": True,          # bool or preset name, optional
    "include": ["trade_log"],      # optional additive list
    "exclude": ["decision_log"],   # optional subtractive list
}
```

Resolution precedence:

1. Mode default bundle (or explicit `bundle`)
2. Venue preset (optional)
3. `include`
4. `exclude`

## APIs

- `resolve_analyzer_bundle_names(mode, venue=None, config=None)`
- `build_observer_analyzers(mode, venue=None, config=None)`
- `analyzer_family_taxonomy()`
