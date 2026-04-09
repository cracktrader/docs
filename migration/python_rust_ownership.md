# Python vs Rust Engine Ownership (Snapshot: April 9, 2026)

This page is the contributor-facing snapshot for engine ownership during the Python-to-Rust migration.

Use this when you need one fast answer to:

- what is Python-owned today
- what is already Rust-backed
- which milestones are active vs still planned

For detailed design or policy, follow the linked plan/RFC pages.

## Current Ownership Matrix

| Surface | Owner Today | Rust Status | Notes |
| --- | --- | --- | --- |
| Strategy authoring and callback UX (`ct.Strategy`, `next()`/compat flows) | Python | Not migrated | Strategy authoring remains Python-first by design. |
| Session/factory/runtime entrypoints (`ct.exchange`, `ct.Store`, `ct.Feed`, `ct.Broker`) | Python | Not migrated | Public user ergonomics stay Python-owned during migration. |
| Deterministic transition core (`build_core`, runtime `engine_backend`) | Python default | Rust optional | Rust backend is opt-in; default backend token remains `python`. |
| Event sequencer (`build_sequencer`) | Python default | Rust optional | Rust sequencer exists behind bridge selection. |
| Invariant checker (`build_invariant_checker`) | Python default | Rust optional | Rust checker exists behind bridge selection. |
| Event normalization (`build_event_normalizer`) | Python default | Rust optional | Rust normalizer exists behind bridge selection. |
| Replay and recovery orchestration | Python | Hybrid execution path | Python orchestration can execute against selected backend via core bridge. |
| Exchange/feed/broker adapters and integration glue | Python | Not migrated (engine core scope only) | Rust engine migration does not yet move adapter ownership. |
| Runtime daemon/control-plane/read-model services | Python | Not migrated | Outside current deterministic-core migration scope. |
| Schema contracts for Python-Rust boundary | Shared contract surface | Active | Versioned JSON Schema package is in-repo and used by contract tests. |

## What Rust Covers Today

Current Rust coverage is real but scoped:

1. Optional `cracktrader_rust` extension integration exists.
2. Backend selection exists (`python`/`rust`) with Python as default.
3. Rust-backed components are available behind bridge adapters for:
   - deterministic core
   - sequencer
   - invariant checker
   - event normalizer
4. Rust parity is enforced by required CI lanes (`rust-parity-required`, `rust-feed-parity-required`).
5. Schema-first boundary contracts are accepted and implemented for engine payloads.

## What Rust Does Not Cover Yet

As of this snapshot, these remain intentionally Python-owned:

1. Strategy callback authoring/runtime UX.
2. Adapter-heavy edges (exchange/store/feed/broker orchestration).
3. Runtime daemon, runtime pub/sub/control-plane, and read-model service layer.
4. Default runtime backend selection (still `python` unless explicitly changed).

Feed acceleration has a separate Rust track documented in [Rust Feed Acceleration](../performance/rust_feed_acceleration.md). That does not change the main engine ownership split described here.

## Milestone Map

| Milestone | Status | What It Means |
| --- | --- | --- |
| PR-1 scaffold and backend selection | Implemented | Backend tokens, Rust bridge, crate scaffold, and initial parity harness are in place. |
| Schema-first boundary | Implemented | JSON schemas and validators define canonical Python-Rust contract payloads. |
| Parity expansion and CI discipline | Active | Required Rust parity lanes and fixture expansion continue to harden equivalence. |
| Runtime hardening with Python strategy layer | Planned/ongoing | More integration coverage and performance tuning while strategy UX stays Python-first. |
| Rust-preferred default backend | Planned | Rust becomes recommended/default only after sustained parity and performance evidence. |

## Contributor Guidance

1. If you change deterministic engine semantics, treat it as a cross-backend change:
   - update schema/fixtures when payload contracts change
   - keep Python and Rust parity suites green
2. If you change strategy/session/adapter/runtime-daemon surfaces, assume Python ownership unless a migration issue explicitly says otherwise.
3. Do not treat optional Rust backend availability as proof of Rust ownership for the entire runtime stack.

## Primary References

- [Rust Migration Guide](../plans/rust_migration_guide.md)
- [Rust Engine Architecture and Python Interop Design](../plans/rust_engine_architecture.md)
- [Rust Core PR-1 Scaffold Plan](../plans/rust_pr1_scaffold_plan.md)
- [RFC: Schema-First Python-Rust Engine Boundary](../plans/schema_first_python_rust_boundary_rfc.md)
- [Rust CI Policy](../testing/rust_ci_policy.md)
