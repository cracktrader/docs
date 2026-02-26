# Mode Divergence Ledger

This ledger tracks intentional behavior differences across runtime modes (`backtest`, `paper`, `sandbox`, `live`).

Use this as the source of truth when:
- changing mode-specific behavior,
- introducing compatibility exceptions,
- adding/adjusting mode contract tests.

## Entry Schema

Each divergence entry should include:

- `id`: stable identifier (`MDL-###`)
- `area`: subsystem (engine, broker, feed, risk, etc.)
- `modes_affected`: explicit list
- `behavior`: short statement of the divergence
- `justification`: why divergence is intentional
- `owner`: maintainer/team
- `status`: `active` | `deprecated` | `removed`
- `introduced_in`: commit/PR reference
- `test_reference`: contract/integration test file(s) covering this behavior
- `notes`: optional migration or deprecation notes

## Active Entries

| id | area | modes_affected | behavior | justification | owner | status | introduced_in | test_reference | notes |
|---|---|---|---|---|---|---|---|---|---|
| MDL-001 | engine normalization | backtest vs paper/sandbox/live | In `backtest`, missing timestamps in `normalize_order_event`, `normalize_fill_event`, and `normalize_timer_event` raise an error instead of falling back to wall-clock. | Preserve deterministic replay and prevent non-reproducible event timelines in backtests. | engine team | active | #36 | `tests/unit/engine/test_events.py`, `tests/contracts/engine/test_python_rust_event_normalization_parity.py` | Paper/sandbox/live may still use wall-clock fallback when explicit timestamps are unavailable. |
| MDL-002 | broker submission path | backtest/paper vs sandbox/live | `backtest` and `paper` route submission through the simulated `CCXTBackBroker`, while `sandbox` and `live` use async real-time broker paths. | Keep offline determinism and test speed for research modes while preserving venue-coupled semantics for real-time modes. | engine team | active | #35 | `tests/contracts/test_mode_contracts.py`, `tests/unit/factory/test_broker_factory.py` | This divergence is intentional and relied on by mode contract suites. |

## Update Policy

When behavior changes:

1. Add or update a ledger entry in the same PR.
2. Add/adjust tests and record them in `test_reference`.
3. If divergence is removed, mark `status=removed` and keep the row for audit history.
