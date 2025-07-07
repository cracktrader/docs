# Fixture Strategy

## Goals
- Minimize duplication
- Provide flexible test setup for live/back brokers

## Structure
- `make_broker(mode="live"|"back")`
- `make_data(symbol="BTC/USDT", type="spot")`
- `make_order(...)`
- Parametrize brokers and feeds in shared test logic

## Lifecycle

| Fixture | Scope | Purpose |
|---------|--------|---------|
| `mock_store` | function | Simulates exchange store methods |
| `setup_broker` | function | Live or back broker factory |
| `simulate_order_fill` | function | Simulate exchange fill |

