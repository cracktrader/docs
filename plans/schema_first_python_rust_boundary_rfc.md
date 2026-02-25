# RFC: Schema-First Python-Rust Engine Boundary

Status: Accepted

Issue: [#46](https://github.com/cracktrader/cracktrader/issues/46)  
Parent epic: [#29](https://github.com/cracktrader/cracktrader/issues/29)

## Summary

Adopt a schema-first boundary for engine contracts shared between Python and Rust.  
The canonical source of truth is a versioned JSON Schema set for:

- `EngineEvent`
- `EngineIntent`
- `OrderTransition`
- `TransitionResult`
- `ExecutionStepResult`
- Core state snapshots used in parity/replay checks

Python and Rust implementations validate against the same schemas, and parity tests are generated from shared fixtures that conform to these schemas.

## Motivation

Current parity relies on hand-maintained assumptions across two implementations.  
A schema-first boundary reduces drift by:

- codifying required fields and allowed variants
- tightening compatibility expectations during refactors
- enabling automated differential tests from one fixture set

## Decision

We will use **JSON Schema (draft 2020-12)** as the boundary contract for v1.

Why JSON Schema first:

- human-readable and repo-native for design/review
- straightforward validation in Python and Rust
- low-friction incremental adoption without immediate codegen lock-in

Protobuf remains a future option if transport optimization becomes necessary, but it is out of scope for this RFC.

## Schema Candidates

The initial schema package will define:

1. `engine_event.schema.json`
2. `engine_intent.schema.json`
3. `order_transition.schema.json`
4. `transition_result.schema.json`
5. `execution_step_result.schema.json`
6. `core_snapshot.schema.json`

Rules:

- required `type`-style discriminators for event and intent variants
- strict enum domains for lifecycle statuses
- no silent additional properties by default (`additionalProperties: false`)
- explicit backward-compatible evolution rules via schema versioning

## Validation and Generation Strategy

Python:

- Validate normalized payloads in test and contract layers.
- Optional runtime validation toggled for debug/reliability modes.

Rust:

- Validate bridge payloads at boundary tests.
- Keep runtime hot paths free of heavy validation by default.

Parity generation:

- Maintain schema-valid fixture corpus in `tests/contracts/engine/fixtures/`.
- Build fixture runner used by both backends and compare canonicalized outputs.
- CI gate checks schema validation plus Python-vs-Rust output parity.

## Migration Plan

1. Introduce schemas and fixture corpus in a non-breaking path.
2. Add validators to contract tests and CI parity jobs.
3. Move existing parity tests to schema-driven fixtures.
4. Tighten runtime boundary checks behind explicit debug policy.
5. Freeze v1 schema and document versioning process.

## Non-Goals

- Rewriting internal in-memory data structures to match wire shape.
- Forcing protobuf/codegen in v1.
- Blocking ongoing engine work on full schema coverage in one PR.

## Follow-up Implementation Issues

- [#41](https://github.com/cracktrader/cracktrader/issues/41): VenueAdapter boundary
- [#42](https://github.com/cracktrader/cracktrader/issues/42): reconciliation contracts
- [#45](https://github.com/cracktrader/cracktrader/issues/45): timeline and trace correlation

Additional schema-specific implementation issues may be split from #46 as needed.
