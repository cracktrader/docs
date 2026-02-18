# Rust Engine Architecture and Python Interop Design

## Document Metadata
- Status: Draft proposal
- Audience: Core engine maintainers, backend integrators, testing/CI owners
- Scope: Define a Rust-first execution core while preserving Python strategy authoring
- Non-goal: Immediate implementation details for every backend adapter

---

## 1. Executive Summary

Cracktrader should evolve toward a Rust execution core that provides deterministic event ordering, order lifecycle correctness, and high-throughput I/O handling, while preserving Python as the strategy/indicator authoring surface.

The recommended shape is:
- Async at the I/O boundary (exchange sockets/REST, persistence, external control APIs)
- Deterministic synchronous state transitions in the core event processor
- A strict, typed boundary between Python strategy code and Rust engine state

This allows:
- Better reliability under live multi-feed load
- Stronger correctness constraints (ordering, idempotency, state-machine validity)
- Long-term performance gains without forcing users to rewrite strategy logic in Rust

---

## 2. Goals and Constraints

### 2.1 Goals
1. Preserve Python strategy UX and public API semantics where feasible.
2. Guarantee deterministic behavior via explicit event ordering rules.
3. Support both backtest and live execution from one core model.
4. Make parity measurable: same inputs should produce equivalent decisions/fills.
5. Reduce timing/ordering bug surface by centralizing state transitions.

### 2.2 Constraints
1. Existing Backtrader-shaped strategy code remains widely used.
2. Multiple backends (CCXT, Polymarket, Kalshi) have capability differences.
3. Tests must remain hermetic for unit scope and gated for integration/e2e.
4. Migration must be incremental, not a hard cutover.

---

## 3. Proposed Runtime Model

## 3.1 High-level Components

1. `Edge Layer (Async)`
- Market data adapters (WS, polling)
- Order transport adapters
- Persistence/log sinks
- External control plane (web API/webhooks)

2. `Core Layer (Deterministic)`
- Global sequencer + event dispatcher
- Order state machine
- Portfolio/risk/accounting state
- Simulation/matching model for backtests
- Strategy scheduling and intent application

3. `Python Strategy Layer`
- Reads immutable snapshots/event payloads
- Emits intents only (`PlaceOrder`, `CancelOrder`, etc.)
- No direct mutation of engine state

## 3.2 Sync vs Async Boundary

### Async responsibilities
- Fetch data from external systems concurrently
- Retry/backoff/timeouts
- Socket lifecycle/reconnect/heartbeat
- Queueing to core ingress

### Sync responsibilities (single logical timeline)
- Assign sequence IDs
- Resolve event ordering and tie-breakers
- Apply state machine transitions
- Trigger strategy callbacks
- Commit resulting state atomically

### Principle
Async handles waiting. Sync handles truth.

---

## 4. Core Event Contract

## 4.1 Event Envelope

Every inbound event is normalized into:

- `seq_id: u64` (assigned at ingress into core)
- `event_time: i64` (source timestamp)
- `ingest_time: i64` (engine timestamp)
- `source: enum` (`market_data`, `broker_report`, `timer`, `control`)
- `venue: string`
- `instrument: InstrumentId`
- `event_type: enum`
- `payload: typed struct`
- `idempotency_key: string | null`

## 4.2 Ordering Rules

1. Primary order: `seq_id` (strictly monotonic)
2. Secondary deterministic tie-breakers at enqueue time:
- source priority (configurable)
- venue
- instrument
- local producer index
3. Replays must preserve original sequence and produce identical transition path.

## 4.3 Intent Contract (Python -> Core)

Strategy may emit only typed intents:
- `PlaceOrder`
- `CancelOrder`
- `ReplaceOrder`
- `SetRiskFlag`
- `NoOp`

Core validates intents before transition:
- schema validity
- capability checks
- risk guard checks
- order/state-machine legal transition checks

---

## 5. Order Lifecycle and Determinism Constraints

## 5.1 Required Invariants

1. No fill before order acceptance.
2. Terminal orders cannot transition to non-terminal states.
3. Filled quantity is monotonic non-decreasing.
4. Remaining quantity is monotonic non-increasing.
5. Position/cash conservation checks after each committed event.
6. Duplicate reports with same idempotency key are no-ops.
7. Same replay log => same final state + same intent stream.

## 5.2 State Machine (minimum)

`Created -> Submitted -> Accepted -> PartiallyFilled -> Filled`
`Accepted/PartiallyFilled -> CancelPending -> Cancelled`
`Submitted/Accepted -> Rejected`

Invalid transitions fail closed and produce explicit diagnostics.

---

## 6. Python Interop Boundary (Rust <-> Python)

## 6.1 Interop Model

Use `PyO3` for bindings and keep a narrow crossing surface:

1. Core passes a compact `StrategyContext` and `EventView`.
2. Python strategy returns a list of intents.
3. Rust validates/applies intents synchronously.

## 6.2 GIL Strategy

1. Minimize per-event crossing overhead.
2. Batch callbacks when practical (e.g., multiple bar events in backtest chunk mode).
3. Keep high-frequency state transitions entirely in Rust.

## 6.3 Data Shapes

Prefer immutable snapshots and stable IDs over deep object graphs.

Recommended strategy callback shape:
- `on_event(ctx, event) -> list[Intent]`

Optional convenience compatibility wrapper can map this back to:
- `next()`-style semantics for legacy strategies.

---

## 7. Modes: Backtest vs Live

## 7.1 Backtest Mode

- Data source may be file/memory stream.
- Core runs deterministic timeline.
- Async edge can be bypassed or minimally used.
- Matching/fill simulation is deterministic and replayable.

## 7.2 Live Mode

- Async edge fully active.
- External reports normalized and sequenced.
- Core applies exact same state machine/invariants.

## 7.3 Why this unification matters

Single transition model reduces divergence between backtest and live behavior and makes parity verification realistic.

---

## 8. How this Differs from Current Python Shape

## 8.1 Current (simplified)

- Significant logic still tied to Backtrader control flow semantics.
- Async pathways exist but include adapters/shims.
- Strategy execution and broker stepping can be intertwined with framework internals.

## 8.2 Target

- Core event processor becomes the canonical source of truth.
- Framework adapters become compatibility layers, not state owners.
- Strategy API remains Pythonic, but state ownership and ordering move to Rust.

---

## 9. Changes Needed in Current Python Engine to Align Early

These changes should be done now in Python so the Rust migration is low-risk.

## 9.1 Introduce explicit event envelope internally

- Normalize feed/broker/timer/control events into one internal schema.
- Assign deterministic sequence IDs in one place.

## 9.2 Isolate a "deterministic core" module

- Move order state machine and portfolio transitions into framework-independent module.
- Make Backtrader/Cerebro integration call into this core rather than own transitions.

## 9.3 Restrict strategy outputs to intents

- Even in Python-only mode, convert direct `buy/sell/cancel` usage into intent recording + application path.
- Keep helper sugar, but route through canonical intent processor.

## 9.4 Add invariant checks as first-class runtime assertions

- Enabled in tests by default.
- Configurable strictness in production.

## 9.5 Add replay log

- Every event + decision + transition is serializable.
- Replayer can run determinism checks in CI.

## 9.6 Capability matrix formalization

- Explicit backend capability declarations used by both core behavior and tests.
- Prevent backend-specific conditionals from leaking into shared logic.

---

## 10. Testing Strategy for Rust Parity

## 10.1 Test Pyramid

1. Unit (Rust core)
- State machine transitions
- Invariant checks
- Sequencer ordering
- Idempotency behavior

2. Contract (cross-language)
- Same event fixture through Python core and Rust core should yield same transition trace.

3. Integration
- Adapter-level tests for feed/order transport behavior with mocked backends.

4. E2E
- Live/sandbox gated runs with strict cleanup guarantees.

## 10.2 Golden Trace Parity

Create canonical fixtures:
- Market data streams
- Order acknowledgements/fills/cancels
- Edge cases (out-of-order reports, duplicates, partial fills)

For each fixture:
1. Run current Python core -> collect transition trace + final state
2. Run Rust core -> collect same artifacts
3. Assert parity (allowing documented numeric tolerances where needed)

## 10.3 Differential/Fuzz Testing

- Generate randomized but valid event streams.
- Compare Rust and Python transition outcomes.
- Add shrinking/minimization to keep failing cases debuggable.

## 10.4 Backtest Equivalence Tests

- Strategy signal parity tests (indicator outputs, action timing)
- Fill parity tests under deterministic simulator
- Portfolio curve parity within tolerance bounds

## 10.5 Performance Benchmarks

Benchmark profiles:
1. CPU-heavy backtest (minimal I/O)
2. Multi-feed live simulation (high I/O)
3. High order-update burst scenarios

Track:
- events/sec
- end-to-end latency p50/p95/p99
- callback boundary overhead
- memory growth / queue depth under load

---

## 11. Migration Plan

## Phase 0: Alignment in Python (pre-Rust)

1. Add event envelope + sequencer.
2. Extract deterministic core transitions.
3. Add runtime invariants + replay logger.
4. Add parity fixture framework in tests/contracts.

Exit criteria:
- Current engine passes all existing tests.
- New determinism and replay tests are green.

## Phase 1: Rust core prototype

1. Implement sequencer + state machine + portfolio kernel in Rust.
2. Expose minimal PyO3 API.
3. Run parity fixtures against both engines in CI.

Exit criteria:
- Core parity on contract fixtures >= agreed threshold (target 100% on discrete transitions).

## Phase 2: Hybrid runtime

1. Python strategies execute against Rust-owned state.
2. Keep existing Python adapters, route through Rust core.
3. Enable feature flag: `engine="python"|"rust"`.

Exit criteria:
- Unit/contract suites green for both flags.
- Benchmark deltas understood and documented.

## Phase 3: Rust-first default

1. Default new runs to Rust engine.
2. Keep Python engine as fallback for deprecation window.
3. Publish migration notes and troubleshooting docs.

Exit criteria:
- Stable release quality and support runbook in place.

---

## 12. Failure Domains and Operational Controls

## 12.1 Backpressure

Define queue policies explicitly:
- block producer
- bounded drop policy
- fail-fast

Policy must be mode-specific and visible in diagnostics.

## 12.2 Crash Recovery

- Persist replayable event logs and checkpoints.
- Recover core state by replay to checkpoint + suffix.

## 12.3 Observability

Emit structured telemetry:
- event lag
- queue depth
- transition failure counts
- invalid transition attempts
- callback latency

---

## 13. Suggested Repository Changes (Documentation + Interfaces)

1. Add shared interface docs:
- `docs/reference/engine_events.md`
- `docs/reference/engine_intents.md`

2. Add contracts section:
- `tests/contracts/engine/` for parity fixtures.

3. Add benchmark suite:
- `tests/performance/engine_parity_benchmarks.py`

4. Add feature flag config path:
- `engine_backend = python | rust`

---

## 14. Risks and Mitigations

1. Python callback overhead dominates
- Mitigation: batch callbacks and keep transitions in Rust.

2. Behavior drift during migration
- Mitigation: golden trace parity in CI from Phase 0 onward.

3. Adapter complexity remains high
- Mitigation: strict edge/core ownership and typed event normalization.

4. Determinism regressions under concurrency
- Mitigation: single sequencer authority and deterministic tie-breaking.

---

## 15. Decision Log (Proposed Defaults)

1. Rust core is event-driven and deterministic.
2. Async only at I/O boundary; core transition path is logically synchronous.
3. Python strategy API preserved through compatibility layer.
4. Replay log and invariants are mandatory for core correctness.
5. Migration proceeds behind feature flags with parity gates.

---

## 16. Open Questions

1. Should strategy callbacks remain Backtrader-first (`next`) or move to event-native (`on_event`) with adapters?
2. What is the acceptable numeric tolerance for floating-point portfolio parity?
3. Which invariants are hard-fail in production vs warn-only?
4. How long should Python-engine fallback be maintained after Rust default launch?
5. Do we support mixed-mode deployments (Rust core + selected Python fallback adapters) in GA, or only during migration?

---

## 17. Immediate Next Actions

1. Approve the event envelope schema and intent schema.
2. Implement Python-side sequencer + replay logger first.
3. Add contract fixtures for ordering/idempotency/partial-fill edge cases.
4. Define CI parity gates for Rust prototype admission.
5. Draft user-facing migration guidance once Phase 1 parity is demonstrated.
