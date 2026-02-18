# Python Engine Execution Spec (Backtrader-Aligned, Rust-Ready)

## Document Metadata
- Status: Draft spec
- Scope: Define exact execution order for Cracktrader Python engine
- Inputs: Backtrader control-flow semantics + Cracktrader constraints
- Relationship: Companion to `docs/plans/rust_engine_architecture.md`

---

## 1. Purpose

This document specifies how the Cracktrader Python engine should execute in deterministic order, while remaining compatible with Backtrader-style strategies during migration.

It answers:
1. What order should engine phases run in?
2. What should happen in each phase?
3. Where are async vs sync boundaries?
4. Which invariants must hold at each boundary?

---

## 2. Source Behavior Baseline (Backtrader)

Backtrader execution behavior (observed from `Cerebro.run`, `runstrategies`, `_runnext`, `_runonce`):

1. Run configuration and mode resolution
- set run flags (`runonce`, `preload`, exactbars/live/replay overrides)
- initialize writers
- materialize strategy combinations

2. Runtime startup (`runstrategies`)
- start stores, broker, feeds
- reset/start/preload datas
- instantiate strategies
- add observers/analyzers/sizers
- call `strat._start()`

3. Main execution loop
- non-vectorized path: `_runnext`
  - poll feeds / advance bars
  - resolve timemaster dt
  - notify data/store
  - timer checks (cheat and normal)
  - broker notify
  - strategy `_next`
  - writer flush
- vectorized path: `_runonce`
  - indicator/strategy `_once` precompute
  - advance bars pseudo-event style
  - timer checks, broker notify, strategy post hooks

4. Shutdown
- `strat._stop()`
- broker stop
- data/feed/store stop
- writer stop

This baseline is the compatibility reference.

---

## 3. Cracktrader Python Engine Target Shape

## 3.1 Core Principle

Engine execution is split into:
1. Async edge ingestion (optional)
2. Deterministic sync transition core (required)

Even when async feeds are used, state transitions occur in one deterministic ordered loop.

## 3.2 Canonical Runtime Stages

Every run (sync or async ingress) must execute these stages in order:

1. `CONFIGURE`
2. `BOOTSTRAP`
3. `INGEST`
4. `SEQUENCE`
5. `APPLY`
6. `STRATEGY`
7. `BROKER`
8. `OBSERVE`
9. `COMMIT`
10. `SHUTDOWN`

No stage may be skipped except where explicitly gated by mode.

---

## 4. Detailed Stage Spec

## 4.1 CONFIGURE

Inputs:
- run kwargs
- engine flags (`vectorized`, `live`, `preload`, `runonce`, `exactbars`)
- optimization/search flags

Actions:
1. Resolve runtime mode
2. Normalize conflicting flags
3. Initialize telemetry/writers/reporters
4. Resolve strategy matrix

Outputs:
- immutable `RunConfig`

Invariants:
- mode and flags are final before bootstrap
- no side effects to market/broker state yet

## 4.2 BOOTSTRAP

Actions:
1. Start stores
2. Start broker
3. Start feeds
4. Reset and start data series
5. Preload when enabled
6. Instantiate strategies and attach analyzers/observers/sizers
7. Call strategy start lifecycle (`_start`/`start`)

Invariants:
- all runtime components are either fully started or run aborts
- strategy objects exist before first event

## 4.3 INGEST

Actions:
1. Collect raw edge events (bars, order reports, timers, control)
2. Normalize into internal event envelope
3. Attach source metadata and idempotency keys where present

Mode notes:
- Sync historical: ingest is mostly data-advance driven
- Async/live: ingest may await adapters, but never mutates core state directly

Invariant:
- ingest may enqueue events only; no transitions here

## 4.4 SEQUENCE

Actions:
1. Assign monotonic `seq_id`
2. Apply deterministic tie-break rules
3. Emit ordered event batch for transition core

Invariant:
- ordered batch is stable/replayable

## 4.5 APPLY

Actions:
1. Advance data clock/index to event cursor
2. Apply market-data state updates
3. Apply inbound broker/execution updates through order state machine
4. Validate legal state transitions

Invariants:
- no invalid order lifecycle transition
- idempotent handling for duplicate reports

## 4.6 STRATEGY

Actions:
1. Build immutable strategy view/context
2. Execute strategy callback (`_next`/`next` or vectorized hook policy)
3. Collect intents/actions (buy/sell/cancel etc.)

Compatibility requirement:
- preserve Backtrader minperiod/prenext behavior
- preserve strategy callback ordering relative to broker notify/timers

Invariant:
- strategy cannot mutate canonical engine state directly

## 4.7 BROKER

Actions:
1. Translate intents into broker requests
2. Execute broker step (`broker.next` in sync mode or async adapter process)
3. Capture resulting notifications/reports

Invariant:
- all broker side effects are represented as events/reports

## 4.8 OBSERVE

Actions:
1. Deliver notifications to strategy/analyzers/observers
2. Flush writer outputs for the current step
3. Collect telemetry and diagnostics

Invariant:
- observers see committed step-consistent state

## 4.9 COMMIT

Actions:
1. Finalize state snapshot for step
2. Append replay record (`event`, `intents`, `transitions`, `result`)
3. Run invariant checks (strict in test mode)

Invariant:
- step is atomic from replay perspective

## 4.10 SHUTDOWN

Actions:
1. Stop strategies (`_stop`/`stop`)
2. Stop broker
3. Stop data/feed/store
4. Stop writers
5. Emit final summary and diagnostics

Invariant:
- stop order is deterministic and idempotent

---

## 5. Required Ordering Rules

For each execution tick/event batch, required order is:

1. data/store notifications ingestion
2. timer (cheat-on-open path, if enabled)
3. broker notify/process
4. timer (normal path)
5. strategy execution
6. observer/analyzer/writer emission

This mirrors Backtrader semantics while clarifying separation into explicit stages.

---

## 6. Mode-Specific Execution

## 6.1 Sync Historical Mode

Primary loop driver:
- data advancement (Backtrader-style `_runnext` equivalent)

Flow:
- no awaits
- deterministic feed advancement and broker stepping per bar

## 6.2 Vectorized Mode

Primary loop driver:
- precompute indicators/strategy vector hooks

Required behavior:
1. indicator compute mode resolution (`batch`, `rolling`, `one_by_one`)
2. strategy vectorized hook execution
3. action capture and deterministic replay across bar indices
4. broker step replay in bar order

Invariant:
- vectorized path must preserve execution semantics relative to non-vectorized path.

## 6.3 Async Edge Mode

Primary loop driver:
- async adapters supply events

Required behavior:
1. async tasks ingest external events concurrently
2. sequencer creates deterministic ordered batches
3. transition/apply/strategy/broker remains single ordered core path

Invariant:
- async does not change transition semantics, only ingestion concurrency.

---

## 7. Python Engine API/Boundary Spec

## 7.1 Internal Engine Interfaces (Python)

1. `EngineEvent`
- normalized envelope for all inbound events

2. `EngineIntent`
- strategy output command type

3. `TransitionResult`
- explicit result of applying one event+intent batch

4. `ReplayRecord`
- persisted trace unit for parity/replay checks

## 7.2 Adapter Interfaces

1. Feed adapter
- `poll_next() -> Event | None` (sync)
- `__anext__() -> Event` (async)

2. Broker adapter
- `process(intents) -> reports`
- async variant permitted, but result always normalized to events

---

## 8. Required Engine Invariants

Must be checked per commit step:

1. Event sequence monotonicity
2. Order lifecycle validity
3. Position/cash conservation
4. No duplicate effect from same idempotency key
5. Data index/time monotonicity per stream
6. Deterministic replay equivalence in test mode

---

## 9. Gap Analysis: Current Cracktrader vs Target Spec

Current strengths:
1. Existing sync engine compatibility via `CracktraderCerebro`
2. Async ingestion now routes through the same deterministic runtime stages
3. Vectorized strategy/indicator path exists and now has stronger parity tests

Current gaps to close:
1. No first-class internal event envelope used across all pathways
2. Transition core still interwoven with framework callback mechanics
3. Continue reducing wrapper/framework-specific coupling around the canonical runtime
4. Replay log/invariant enforcement not yet first-class runtime contract

---

## 10. Implementation Plan for Python Engine Alignment

## Phase A: Deterministic Core Extraction

1. Create `engine/core.py` with transition state machine
2. Route order updates through explicit transition API
3. Add invariant checker module

## Phase B: Event Envelope + Sequencer

1. Create canonical event schema
2. Normalize feed/broker/timer/control into schema
3. Add sequencer with stable ordering policy

## Phase C: Adapter Convergence

1. Refactor `Cerebro` path to call core stages explicitly
2. Refactor `AsyncCerebro` to use same stages with async ingestion only
3. Keep Backtrader-facing strategy API as compatibility layer

## Phase D: Replay and Parity

1. Add replay record writer
2. Add deterministic replay runner
3. Add contract tests to compare modes (`sync`, `vectorized`, `async-edge`)

---

## 11. Test Spec (for this execution model)

## 11.1 Unit

1. Sequencer ordering tests
2. State machine transition legality tests
3. Idempotency tests
4. Invariant checker tests

## 11.2 Contract

1. Same fixture through sync and async-edge paths => identical transition trace
2. Same fixture through non-vectorized and vectorized => equivalent strategy intents and portfolio outcome

## 11.3 Integration

1. Multi-feed timestamp ordering tests
2. Broker notification ordering tests
3. Timer ordering tests (cheat-on-open and normal)

## 11.4 Performance

1. Throughput baseline for sync historical
2. Throughput/latency for async-edge multi-feed
3. Boundary cost tests for strategy callback frequency

---

## 12. Pseudocode Reference

```text
run(config):
  cfg = CONFIGURE(config)
  runtime = BOOTSTRAP(cfg)

  while runtime.active:
    raw_events = INGEST(runtime)
    ordered = SEQUENCE(raw_events)

    for event in ordered:
      APPLY(runtime, event)
      intents = STRATEGY(runtime, event)
      reports = BROKER(runtime, intents)
      OBSERVE(runtime, event, intents, reports)
      COMMIT(runtime, event, intents, reports)

  SHUTDOWN(runtime)
```

Async-edge only changes `INGEST`; all later stages are identical.

---

## 13. Acceptance Criteria

This spec is considered implemented when:

1. Sync, vectorized, and async-edge paths run through the same ordered transition core.
2. Contract tests show deterministic parity across modes for canonical fixtures.
3. Invariant failures are actionable and include step/event identifiers.
4. Replay log can reproduce state and strategy-intent trace deterministically.
5. Backtrader compatibility layer passes existing unit/public API suites.
