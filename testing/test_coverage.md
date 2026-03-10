# Test Coverage Responsibilities

This page explains what each part of the suite is responsible for.

It deliberately replaces the old giant test inventory, which became hard to trust as the runtime architecture changed.

## Coverage Is About Boundaries, Not File Counts

The strongest coverage signal in Cracktrader is whether a behavior is protected at the correct layer:

- unit for narrow logic
- contracts for shared guarantees
- public API for stable top-level behavior
- integration for real subsystem wiring

## Current Coverage Responsibilities

| Suite area | Primary responsibility |
| --- | --- |
| `tests/unit/engine/` | local runtime internals, adapters, registries, and helper logic |
| `tests/unit/session/` | session-owned registrations and construction helpers |
| `tests/unit/broker/` | broker mechanics and venue-specific order/accounting behavior |
| `tests/contracts/engine/` | deterministic runtime and backend parity guarantees |
| `tests/contracts/` | orchestration, execution, market-data, and reliability guarantees |
| `tests/public_api/` | entrypoints, exports, and high-level native runtime usage |
| `tests/integration/` | larger subsystem flows with explicit scope |
| `tests/e2e/` | opt-in environment-backed scenarios |

## Coverage Questions To Ask

When evaluating whether coverage is good enough, ask:

1. is the behavior protected at the highest stable layer users rely on
2. is there a smaller unit test needed for narrow local logic
3. are we relying on networked or environment-specific evidence when a deterministic contract test would be stronger

## Good Coverage Signals

- orchestration behavior covered in `tests/contracts/`
- entrypoint behavior covered in `tests/public_api/`
- route, risk, and inventory edge cases covered with targeted unit tests
- deterministic simulator behavior covered in contract tests
- regression tests added for real bugs

## Weak Coverage Signals

- only asserting mocks were called for user-visible runtime behavior
- large unit tests duplicating contract semantics end to end
- relying on live or sandbox tests for logic that could be proven hermetically
- treating a file-count inventory as proof that the runtime boundary is protected

## Related Pages

- [Testing Guidelines](../development/testing_guidelines.md)
- [Runtime Guarantees](runtime_guarantees.md)
- [Known Gaps](known_gaps.md)
