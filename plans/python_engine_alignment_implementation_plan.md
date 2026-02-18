# Python Engine Alignment Implementation Plan

## Document Metadata
- Status: Proposed implementation plan
- Depends on:
  - `docs/plans/rust_engine_architecture.md`
  - `docs/plans/python_engine_execution_spec.md`
- Objective: Refactor current Python engine into a Rust-ready shape without breaking user strategy UX

---

## 1. Scope and Deliverables

This plan defines:
1. Exactly what stays stable.
2. Exactly what changes in existing classes/modules.
3. Which new classes/modules are required.
4. Execution order for implementation.
5. Required new tests and required test updates.

Deliverables at the end:
1. Python engine with explicit staged core (`INGEST -> SEQUENCE -> APPLY -> STRATEGY -> BROKER -> OBSERVE -> COMMIT`).
2. Shared deterministic transition core used by sync, vectorized, and async-edge paths.
3. Event envelope, intent model, invariant checker, replay log.
4. Parity test harness that compares paths and catches ordering/timing regressions.

---

## 2. What Stays the Same

## 2.1 User-facing behavior (target)

Keep stable:
1. Public high-level entry points:
- `ct.Cerebro(...)`
- `cerebro.adddata(...)`
- `cerebro.addstrategy(...)`
- `cerebro.run(...)`
2. Existing marker/flag test conventions and safety gates.
3. Existing strategy authoring model (`next`, `next_vectorized`) during migration.
4. Current backend factory/session abstractions.

## 2.2 Operational expectations

Keep stable:
1. Unit tests hermeticity requirements.
2. Integration/e2e flag gating.
3. Existing exchange capability-based testing approach.

---

## 3. What Changes

## 3.1 Core ownership changes

Move from framework-driven implicit transitions to explicit engine-owned transitions.

Concretely:
1. Order lifecycle transitions no longer scattered across broker callbacks and strategy timing paths.
2. Sequencing and ordering become first-class, centralized.
3. Strategy actions become intents applied by core.

## 3.2 Current class behavior changes

1. `src/cracktrader/cerebro.py`
- Keep as compatibility facade.
- Internally delegate to new staged engine coordinator.
- Remove direct ownership of ordering logic over time.

2. `src/cracktrader/async_cerebro.py`
- Keep as async ingress facade.
- Replace its custom loop logic with shared stage pipeline + async ingest adapter.
- No independent transition semantics.

3. Broker integration classes
- Broker calls must route through intent/result normalization.
- Notifications become event envelopes.

4. Vectorized execution path in `cerebro.py`
- Keep API semantics.
- Internally run through same transition core and commit phases.

---

## 4. New Modules and Classes (Required)

Create a new package: `src/cracktrader/engine/`

## 4.1 Core models

1. `engine/events.py`
- `EngineEvent`
- enums: `EventSource`, `EventType`
- payload variants and normalization helpers

2. `engine/intents.py`
- `EngineIntent`
- intent types: `PlaceOrderIntent`, `CancelOrderIntent`, `ReplaceOrderIntent`, `NoOpIntent`

3. `engine/results.py`
- `TransitionResult`
- `ExecutionStepResult`

## 4.2 Execution pipeline

4. `engine/sequencer.py`
- `EventSequencer`
- assigns `seq_id`, deterministic tie-break

5. `engine/core.py`
- `DeterministicCore`
- `apply_event(event) -> TransitionResult`
- order state machine + portfolio updates

6. `engine/runner.py`
- `EngineRunner`
- orchestrates stage pipeline per spec

7. `engine/invariants.py`
- `InvariantChecker`
- state transition and accounting assertions

8. `engine/replay.py`
- `ReplayRecord`
- `ReplayWriter`
- `ReplayRunner`

## 4.3 Adapters

9. `engine/adapters/backtrader_sync.py`
- bridges Backtrader sync data/broker lifecycle into `EngineRunner`

10. `engine/adapters/backtrader_async.py`
- async ingest adapter, same downstream stages

11. `engine/adapters/vectorized.py`
- vectorized action capture/replay integration as a stage-compatible adapter

## 4.4 Optional compatibility shim

12. `engine/compat/strategy_api.py`
- wrappers for legacy `next`/`next_vectorized` to intent output interface

---

## 5. Ordered Implementation Plan

## Phase 0: Baseline Lock and Instrumentation

Goal: Freeze behavior and establish measurable baseline.

Tasks:
1. Add baseline snapshot tests for current sync/vectorized/async-edge outputs on representative fixtures.
2. Add perf baselines (events/sec, step latency, memory) for existing engine paths.
3. Add transition trace hooks in current code (non-invasive logging only).

Files likely touched:
- `tests/unit/cerebro/*`
- `tests/performance/*`
- `src/cracktrader/cerebro.py` (trace hooks)

Exit criteria:
- Baseline artifacts committed.
- No behavior changes yet.

## Phase 1: Event and Intent Domain Introduction

Goal: Introduce typed event/intent contracts without changing execution flow.

Tasks:
1. Implement `engine/events.py`, `engine/intents.py`, `engine/results.py`.
2. Add normalizers for current broker notifications and feed updates.
3. Add adapter utility functions used by existing paths.

Exit criteria:
- Existing paths can emit `EngineEvent` and collect `EngineIntent` in compatibility mode.

## Phase 2: Sequencer and Invariants (Passive Mode)

Goal: Add deterministic sequencing and invariant checks in observer/passive mode.

Tasks:
1. Implement `EventSequencer`.
2. Thread sequencer through current sync/vectorized/async paths.
3. Implement `InvariantChecker` with warn-only mode first.

Exit criteria:
- Every processed step has `seq_id`.
- Invariant diagnostics available, but no hard-fail outside test mode.

## Phase 3: Deterministic Core Extraction

Goal: Centralize transition logic into `DeterministicCore`.

Tasks:
1. Implement order state machine in `engine/core.py`.
2. Route order transitions through core API from existing broker pathway.
3. Move position/cash/accounting mutations to core-owned methods.

Exit criteria:
- Core owns transition legality checks and transition outputs.
- Existing tests remain green.

## Phase 4: EngineRunner and Stage Pipeline

Goal: Introduce explicit stage execution pipeline and use it in sync mode first.

Tasks:
1. Implement `EngineRunner` with stages:
- `INGEST`, `SEQUENCE`, `APPLY`, `STRATEGY`, `BROKER`, `OBSERVE`, `COMMIT`
2. Add `backtrader_sync` adapter to drive this pipeline.
3. Refactor `CracktraderCerebro.run` to delegate to adapter + `EngineRunner`.

Exit criteria:
- Sync path uses staged engine internally.
- Public sync API unchanged.

## Phase 5: Vectorized Path Convergence

Goal: Move vectorized flow onto same staged core semantics.

Tasks:
1. Implement `engine/adapters/vectorized.py`.
2. Ensure vectorized strategy actions become intents and replay through same commit flow.
3. Ensure indicator compute mode dispatch remains intact.

Exit criteria:
- Vectorized and non-vectorized parity maintained via contract tests.

## Phase 6: Async-edge Path Convergence

Goal: Make async ingress use same downstream staged pipeline.

Tasks:
1. Implement `backtrader_async` adapter.
2. Refactor `AsyncCerebro.run_async` to use `EngineRunner`.
3. Retain only async-specific ingest concerns in async adapter.

Exit criteria:
- Async path transition semantics match sync path for same event fixtures.

## Phase 7: Replay System and Determinism Gates

Goal: Make replay and determinism first-class.

Tasks:
1. Implement replay writer and runner.
2. Persist step-level `ReplayRecord` during tests.
3. Add deterministic replay checks in CI for selected suites.

Exit criteria:
- Replay of recorded run reproduces transition trace and final state.

## Phase 8: Strictness and Cleanup

Goal: Make invariants hard-fail in test/CI and remove obsolete duplicate paths.

Tasks:
1. Enable strict invariant checker by default in unit/contract runs.
2. Remove legacy transition code branches no longer used.
3. Document engine contracts in reference docs.

Exit criteria:
- One canonical transition path remains.
- Docs updated.

---

## 6. File-Level Change Map

## 6.1 New files

1. `src/cracktrader/engine/__init__.py`
2. `src/cracktrader/engine/events.py`
3. `src/cracktrader/engine/intents.py`
4. `src/cracktrader/engine/results.py`
5. `src/cracktrader/engine/sequencer.py`
6. `src/cracktrader/engine/core.py`
7. `src/cracktrader/engine/runner.py`
8. `src/cracktrader/engine/invariants.py`
9. `src/cracktrader/engine/replay.py`
10. `src/cracktrader/engine/adapters/backtrader_sync.py`
11. `src/cracktrader/engine/adapters/backtrader_async.py`
12. `src/cracktrader/engine/adapters/vectorized.py`
13. `src/cracktrader/engine/compat/strategy_api.py`

## 6.2 Existing files to refactor

1. `src/cracktrader/cerebro.py`
- delegate to runner/adapters

2. `src/cracktrader/async_cerebro.py`
- delegate to runner/adapters

3. Broker/fill handling modules in `src/cracktrader/broker/`
- route transitions via core API

4. Relevant feed adapters
- emit normalized events rather than direct state mutation

---

## 7. Testing Plan: New and Changed Tests

## 7.1 New test suites

1. `tests/unit/engine/test_events.py`
- envelope schema correctness
- payload normalization

2. `tests/unit/engine/test_sequencer.py`
- monotonic sequence assignment
- deterministic tie-break ordering

3. `tests/unit/engine/test_invariants.py`
- lifecycle/accounting/idempotency assertions

4. `tests/unit/engine/test_core_state_machine.py`
- legal transitions
- illegal transition failures

5. `tests/unit/engine/test_replay.py`
- replay determinism
- trace equality

6. `tests/contracts/engine/test_sync_async_parity.py`
- same fixture => same transition trace

7. `tests/contracts/engine/test_sync_vectorized_parity.py`
- same fixture => equivalent intents/outcomes

8. `tests/contracts/engine/fixtures/*`
- deterministic event streams (partial fills, duplicates, out-of-order reports)

## 7.2 Existing tests to update

1. `tests/unit/test_cerebro_vectorized.py`
- assert parity via staged core traces in addition to output

2. `tests/unit/cerebro/test_async_cerebro.py`
- assert async path uses same stage ordering and transition outputs

3. `tests/unit/cerebro/test_vectorized_indicator_modes.py`
- keep current assertions; add stage-level ordering checks where appropriate

4. `tests/unit/feed/test_feed_data_flow.py`
- validate event normalization and sequencing

5. `tests/unit/indicators/test_indicator_vectorized_parity.py`
- unchanged intent; keep as guard for indicator mode correctness

## 7.3 CI updates

1. Add default fast checks:
- `pytest -q tests/unit/engine`
- `pytest -q tests/contracts/engine`

2. Add replay parity job:
- run fixed fixtures and diff transition traces

3. Keep existing unit/public_api jobs unchanged initially

---

## 8. Acceptance Criteria by Phase

## Phase 0
- Baselines collected and reproducible.

## Phase 1
- Event/intent types in place and used at boundaries.

## Phase 2
- Sequencer active with deterministic ordering and logs.

## Phase 3
- Core state machine owns transitions.

## Phase 4
- Sync path runs through `EngineRunner`.

## Phase 5
- Vectorized path converged and parity tests green.

## Phase 6
- Async-edge path converged and parity tests green.

## Phase 7
- Replay reproducibility proven in CI.

## Phase 8
- Invariants strict in tests, legacy transition branches removed.

---

## 9. Risks and Mitigation

1. Regression risk from core extraction
- Mitigation: phase gating + parity fixtures + replay diff.

2. Hidden Backtrader lifecycle dependencies
- Mitigation: adapter encapsulation and stage-order contract tests.

3. Async path divergence
- Mitigation: single `EngineRunner` with async-only ingest variation.

4. Test suite bloat/runtime
- Mitigation: keep engine unit/contract fixtures compact and deterministic.

---

## 10. Immediate Next Step (Execution Start)

Start Phase 0 with two first tickets:
1. Add trace capture utility + fixture run harness.
2. Add baseline parity snapshots for:
- sync non-vectorized
- vectorized
- async-edge

Once those are green, proceed to Phase 1 without changing user-facing APIs.
