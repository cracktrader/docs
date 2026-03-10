# Runtime Guarantees

This page documents the runtime guarantees that the current test suite is meant to protect.

It is intentionally narrower than "everything the system might do."

## Guarantees The Suite Tries To Hold

### Deterministic Native Runtime Behavior

When a scenario uses hermetic inputs and deterministic adapters, the runtime should preserve:

- stable step ordering
- deterministic strategy input shape
- deterministic attribution on intents, reports, and diagnostics
- deterministic replay and contract behavior in backtest-style paths

### Shared Snapshot Semantics

The runtime should materialize one coherent shared snapshot per step and fan it out without letting strategies mutate shared state.

The contract layer is meant to protect:

- shared-step fanout behavior
- filtered strategy views
- snapshot provenance and identity
- stable strategy attribution

### Route and Execution Attribution

Execution should stay attributable across the stack.

The suite protects the expectation that runtime objects can carry:

- `strategy_id`
- `execution_id`
- `route_id`
- broker or adapter identity
- snapshot or step provenance

### Explicit Risk and Inventory Boundaries

Risk and inventory are runtime services, not hidden strategy side effects.

The suite is meant to protect:

- explicit pre-submission risk decisions
- central inventory and exposure updates from execution reports
- route and execution-context visibility for those services

### Mode-Consistent Execution Contracts

Backtest, paper, sandbox, and live may diverge in environment behavior, but they should not require different strategy-facing execution contracts.

The suite protects:

- stable execution-intent/report shapes
- route-aware execution adapter selection
- deterministic simulation behavior where it is promised

## Non-Guarantees

These are not currently promised as universal truths:

- live-market determinism
- perfect exchange-emulator realism
- identical timing across sandbox and live venues
- universal end-to-end coverage for every integration in default CI

## Where The Evidence Lives

| Guarantee area | Primary suite layer |
| --- | --- |
| deterministic runtime semantics | `tests/contracts/engine/`, `tests/contracts/` |
| shared snapshot fanout and orchestration | `tests/contracts/test_orchestration_contracts.py`, `tests/public_api/` |
| route and execution attribution | `tests/contracts/`, `tests/public_api/`, targeted `tests/unit/engine/` |
| inventory and risk boundaries | `tests/contracts/`, `tests/unit/engine/`, `tests/unit/session/` |
| stable top-level engine entrypoints | `tests/public_api/` |
| narrow adapter and helper behavior | `tests/unit/` |

## Reading Failures Correctly

When a test fails, first decide whether it is proving:

- a local implementation bug
- a broken runtime contract
- a broken public API expectation
- an environment-specific integration problem

That distinction should drive where the fix belongs and which docs need updating with it.
