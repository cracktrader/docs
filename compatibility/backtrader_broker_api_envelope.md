# Backtrader Broker API Compatibility Envelope

Issue: #48  
Scope: Broker-state refactor guardrails while state authority moves toward `DeterministicCore`.

## Strict Compatibility Requirements

- `getcash()` remains submission-stable:
  - placing an order does not mutate available cash before a fill event is applied.
- Fill-driven accounting remains deterministic:
  - after fill events, `getcash()` reflects realized cash movement for the filled quantity.
  - `getvalue(datas=[data])` reflects position value for the provided data feed.
- Order lifecycle callback timing remains strategy-safe:
  - when strategy-style `notify_order` handlers observe `Accepted`, `Partial`, and `Completed`,
    broker getters (`getcash()`, `getvalue(...)`) already expose post-transition values.
- Notification ordering remains monotonic for the common fill path:
  - `Accepted` -> `Partial` -> `Completed` for partial-then-complete fills.

## Allowed Behavior (Compatibility Envelope)

- `getvalue()` total-portfolio semantics may remain cash-dominant during this phase
  (for example equal to cash in backtest valuation paths), provided:
  - return type remains numeric,
  - value is not below `getcash()`,
  - data-scoped valuation via `getvalue(datas=[...])` stays stable.
- Notification queue internals may continue using mutable order objects, as long as
  immediate callback dispatch sees consistent lifecycle ordering and post-transition broker state.

## Guard Tests

- `tests/contracts/test_backtrader_broker_compatibility.py`
  - `test_backtrader_getcash_getvalue_surface_stable_backtest_contract`
  - `test_backtrader_order_notification_timing_contract_for_strategy_callbacks`
