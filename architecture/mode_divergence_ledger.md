# Mode Divergence Ledger

Track intentional and accidental behavior differences across `backtest`, `paper`, `sandbox`, and `live`.
Update this table whenever a slice changes mode semantics or clarifies existing drift.

## Schema

| id | area | modes_affected | description | intentional | reason | test_coverage | risk | status |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| DIV-001 | execution | backtest, paper, live | Simulated fill timing differs from exchange ack/update timing in live paths. | unknown | Pending explicit `ExecutionPolicy` alignment and mode policy documentation. | tests/contracts/test_order_lifecycle.py | medium | open |
| DIV-002 | execution | backtest, paper, sandbox, live | Partial-fill sequencing can differ by backend and transport event order. | unknown | Exchange transports emit updates with backend-specific ordering characteristics. | tests/contracts/test_order_scenarios.py | high | open |
| DIV-003 | accounting | paper, sandbox, live | Balance/position visibility timing still differs between locally-derived state and exchange snapshots, but local mutation authority now routes through `AccountState` by default. | yes | Exchange snapshots can still arrive asynchronously, but canonical local writes are centralized in `AccountState` and compared explicitly. | tests/contracts/state/test_account_state_contracts.py | medium | verified |
| DIV-004 | execution | paper, sandbox, live | Cancel/fill race outcomes vary by transport latency and update ordering. | unknown | Cancellation and fill updates can cross in-flight depending on backend behavior. | tests/contracts/test_reliability.py | high | open |
| DIV-005 | execution | sandbox, live | Local simulation fallback in live brokers is now mode-gated (`sandbox` default enabled, `live` default disabled unless explicit opt-in). | yes | Prevent accidental simulated fills in `live` mode while preserving sandbox testing paths. | tests/unit/broker/test_prediction_live_broker_submission.py | medium | verified |
| DIV-006 | accounting | paper, sandbox, live | AccountState diagnostics include read-through and strict mismatch policy checks while canonical local cash/fill mutations now default to AccountState write authority. | yes | Staged migration kept compatibility flags, but write-authoritative local balance mutation is now centralized via `AccountState` in `UniversalBrokerBase`. | tests/contracts/state/test_account_state_contracts.py | medium | verified |
