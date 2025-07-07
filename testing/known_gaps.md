## Needs Design Decision

- Should `getvalue()` return `0.0` or raise if balances are unavailable?
- Should `_get_latest_price()` silently return 0.0 or raise when OHLCV is missing?
- How should broker behave on unknown instrument types? Log + skip? Raise?
- Should fee fetch failures kill startup, or fall back silently?

## Known Test Gaps

### Missing

### Weak Coverage

### TODO

