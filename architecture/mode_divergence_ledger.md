# Mode Divergence Ledger

Track intentional and accidental behavior differences across `backtest`, `paper`, `sandbox`, and `live`.
Update this table whenever a slice changes mode semantics or clarifies existing drift.

## Schema

| id | area | modes_affected | description | intentional | reason | test_coverage | risk | status |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| DIV-001 | execution | backtest, paper, live | Simulated fill timing differs from exchange ack/update timing in live paths. | unknown | Pending explicit `ExecutionPolicy` alignment and mode policy documentation. | tests/contracts/test_order_lifecycle.py | medium | open |
| DIV-002 | execution | backtest, paper, sandbox, live | Partial-fill sequencing can differ by backend and transport event order. | unknown | Exchange transports emit updates with backend-specific ordering characteristics. | tests/contracts/test_order_scenarios.py | high | open |
| DIV-003 | accounting | paper, sandbox, live | Balance/position visibility timing differs between locally-derived state and exchange snapshots. | unknown | Reconciliation boundary is not yet centralized under `AccountState`. | tests/contracts/test_balances.py | high | open |
| DIV-004 | execution | paper, sandbox, live | Cancel/fill race outcomes vary by transport latency and update ordering. | unknown | Cancellation and fill updates can cross in-flight depending on backend behavior. | tests/contracts/test_reliability.py | high | open |
