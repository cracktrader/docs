# Testing Guidelines

This page describes the current Cracktrader test suite as it exists now.

Use it together with [Runtime Guarantees](../testing/runtime_guarantees.md) when reviewing changes to shared runtime behavior.

## Suite Map

The current suite is organized around runtime boundaries, not only around old unit/integration/e2e buckets.

| Suite area | What belongs there |
| --- | --- |
| `tests/unit/` | narrow component logic, adapters, local state machines, helpers, and exchange-specific details |
| `tests/contracts/` | shared runtime guarantees that should hold across modes, routes, and implementations |
| `tests/public_api/` | stable top-level entrypoints, exports, and wrapper behavior |
| `tests/integration/` | controlled subsystem wiring when unit or contract tests cannot prove the behavior |
| `tests/e2e/` | explicitly gated end-to-end scenarios |
| `tests/performance/` | benchmark and throughput checks |
| `tests/research/` | research/runtime alignment checks |
| `tests/regression/` | bug reproductions that need to stay fixed |

## Preferred Testing Order

For user-visible or shared runtime behavior:

1. start with a contract or public API test
2. add a unit test only for the narrow local logic the higher-level test does not isolate well
3. use integration coverage only when real subsystem wiring is the behavior under test

## What Goes Where

### `tests/unit/`

Use for:

- route-registry validation
- helper logic
- local adapter behavior
- narrow state-coordinator internals
- venue-specific or broker-specific mechanics

### `tests/contracts/`

Use for:

- orchestration semantics
- shared snapshot guarantees
- mode-consistent runtime behavior
- strategy, route, and execution attribution
- explicit runtime policy expectations

### `tests/public_api/`

Use for:

- `CracktraderEngine` entrypoints
- exported engine contracts
- high-level wrapper forwarding
- black-box native runtime setup users call directly

### `tests/integration/`

Use only when:

- the wiring between subsystems is itself the behavior under test
- a contract cannot be proven at unit or public API level

## Shared Test Infrastructure

Prefer existing helpers before creating new setup islands.

Important current helpers include:

- `tests/helpers/orchestration.py`
- `tests/contracts/harness/`
- shared fixtures in `tests/fixtures/`

## Recommended Local Evidence

For runtime, orchestration, or contract changes:

```bash
python -m pytest -q tests/contracts/engine tests/contracts
python -m pytest -q tests/public_api
python -m pytest -q tests/unit/<targeted area>
python -m pre_commit run --all-files
```

For docs-only changes in this repo, the relevant check is:

```bash
python -m mkdocs build -f mkdocs-subtree.yml
```

## Review Heuristics

If a change affects:

- shared-state orchestration
- multi-strategy fanout
- execution contexts or routes
- inventory or risk
- mode-specific execution adapters

then unit coverage alone is usually insufficient.

The contract and public API layers should say the behavior is stable before the change is considered well protected.
