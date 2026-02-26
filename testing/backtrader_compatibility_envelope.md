# Backtrader Compatibility Envelope

This document defines the current compatibility contract for Backtrader-facing broker behavior while state-authority refactors continue.

## Strict Compatibility (must not regress)

- `broker.getcash()` is stable and deterministic at submission time.
- `broker.getvalue(datas=[data])` reflects data-scoped position value after fills.
- Order callback timing (`notify_order`) observes already-updated broker balances/values.
- Reactive strategy callbacks that place follow-up orders from `notify_order` are supported.
- Lifecycle callback status progression remains stable for the tested path:
  - `Accepted -> Partial -> Completed`
  - `Accepted -> Canceled` (non-fill path preserves balances)

Primary contract tests:

- `tests/contracts/test_backtrader_broker_compatibility.py`
- `tests/contracts/engine/test_broker_core_unity_contract.py`

## Allowed Compatibility Envelope (current phase)

- `broker.getvalue()` total portfolio valuation may remain cash-dominant in this phase.
- Even in this relaxed envelope, total value must:
  - remain a float,
  - never fall below `broker.getcash()` in tested scenarios.

This envelope is intentionally temporary and should be tightened as core-authority migration completes.

## Why this exists

Backtrader-facing strategy code often depends on callback timing and portfolio access patterns. Explicitly documenting strict guarantees vs temporary allowances prevents accidental regressions and clarifies expected behavior during migration.
