# Cracktrader End-State Architecture — Knowns, Constraints, and Open Questions

**Status:** Draft (architecture anchor)  
**Purpose:** Capture what is already directionally agreed, what is plausibly locked-in, and what remains undecided pending codebase inspection.  
**Scope:** Backtest / paper / live trading engine with CCXT + Polymarket integration, Python + Rust codebase, long-term Rust expansion.

---

## 1) North Star (locked-in intent)

Cracktrader is a **unified trading engine** for backtesting, paper trading, and live trading, with the long-term goal that these modes share the same core semantics and differ mainly at I/O boundaries (data source and execution transport).

The desired end-state is:
- **event-driven**
- **deterministic at the core**
- **strongly invariant-driven** (orders, fills, balances, positions)
- **safe by default** (risk boundaries and runtime checks)
- **performance-oriented**, with Rust covering most core logic over time

This is **not** intended to be “just a toy backtester.” The target shape is a production-grade engine whose backtest/paper/live behaviors are comparable because they share a canonical model of events, state transitions, and accounting.

---

## 2) What appears locked-in (high confidence architectural decisions)

These are the architectural decisions/constraints that seem stable enough to treat as baseline assumptions unless the code strongly disproves them.

### 2.1 Unified multi-mode engine
The system supports:
- Backtest
- Paper
- Live

These are treated as **modes of one engine**, not separate systems.

### 2.2 Deterministic/event-driven core as the target
A central design goal is a **deterministic state-transition core** driven by a canonical event stream (or equivalent deterministic input sequencing).

Implication:
- Reproducibility and replay are first-class concerns.
- Mode differences should be explicit and documented.

### 2.3 Strong domain invariants are a core requirement
The architecture should enforce invariants around:
- order lifecycle/state transitions
- fill bounds and partial fills
- balance/position accounting
- idempotency / duplicate event handling
- quantization/precision correctness
- terminal state immutability

Whether enforced via Rust types, runtime asserts, tests, or all three is an implementation detail. The requirement itself is locked.

### 2.4 Rust expansion is a strategic direction
Current state: small Rust core with Python/Rust parity in parts.  
Target direction: **grow Rust coverage across core engine logic** for correctness/performance.

This is a strategic direction, not necessarily a fully finalized boundary yet.

### 2.5 Python remains important for research ergonomics (at least in some role)
Python is expected to remain useful for:
- strategy authoring
- experimentation/research workflows
- orchestration and tooling

Exact long-term strategy boundary (Python vs Rust vs hybrid) is still open.

### 2.6 Venue-specific behavior must be normalized behind canonical engine semantics
CCXT and Polymarket have venue quirks. The architecture must provide a normalization/adaptation layer so the core sees canonical events/intents/reports rather than raw venue-specific weirdness.

### 2.7 Reconciliation is mandatory in live/paper-like environments
The architecture must include a reconciliation model for:
- orders
- positions
- balances
- fills (where applicable)

The exact cadence, authority rules, and recovery mechanics remain open.

### 2.8 Explicit mode contracts / divergence tracking are needed
Known problem class: hidden semantic drift between backtest, paper, sandbox, and live.

The architecture should include:
- explicit mode contracts
- a divergence ledger (or equivalent)
- tests asserting mode invariants and allowed divergences

---

## 3) Candidate end-state shape (provisionally accepted, pending code validation)

This section captures the *likely* shape of the end-state, but some details are still provisional.

### 3.1 Architectural philosophy
**“Research in Python, execution/accounting/state transitions in Rust”** is the leading candidate philosophy.

This likely means:
- Rust is authoritative for state transitions and accounting
- Python strategies consume state and emit intents
- Venue adapters translate between engine semantics and external APIs

### 3.2 Core engine should be authoritative over internal ledger
The engine should maintain an authoritative internal ledger (for real-time decisions), while external venues remain authoritative for eventual reconciliation of actual fills/order status.

Working assumption:
- **internal ledger = authority for immediate decisions**
- **venue/exchange = authority for external truth on reconciliation/historical fills**

### 3.3 Canonical order/event lifecycle should exist
Regardless of venue, the engine should operate with a canonical lifecycle for:
- order intent
- submit/ack/reject
- partial fills
- terminal states (filled/canceled/rejected/expired/etc.)

Exact state names may vary, but the concept is locked.

### 3.4 Mutation should be deterministic and tightly controlled
Working assumption:
- state mutation path should be serial/deterministic (even if ingestion/external IO is concurrent)
- concurrency should be pushed to edges (network adapters, feed ingestion, polling)

Whether this becomes a single-threaded core, partitioned actors, or another model is open.

---

## 4) Known hard requirements (non-negotiable properties)

These are not implementation choices; they are properties the final system should satisfy.

### 4.1 Accounting correctness
- Position and balance updates must be derivable from fill/order events.
- Precision/rounding/quantization rules must be explicit and venue-aware.
- Invalid states should fail loudly.

### 4.2 Replayability / auditability
The system should support enough event/state capture to:
- debug post-hoc
- reconstruct causality (intent → order action → broker report → fill → accounting)
- investigate divergences and regressions

Exact persistence strategy is still open.

### 4.3 Safety boundaries
The architecture must support hard risk controls so a buggy strategy cannot trivially violate system-level constraints.

Examples:
- max notional / size / open orders
- symbol allowlists / blocklists
- trading halt / kill switch
- sanity checks on stale data / clock skew

### 4.4 Explicitly modeled mode differences
Differences between backtest/paper/live must be:
- documented
- testable
- intentional

No “it just behaves a bit differently in live because reasons.”

---

## 5) Open questions (needs code inspection and design decisions)

These are the key unresolved questions. They should be answered by inspecting the codebase, existing tests, and current abstractions.

---

### Q1. What is the actual authoritative state boundary today?
**Question:** Where is authoritative state currently held for orders/positions/balances in each mode (Python core, Rust core, adapters, broker classes, stores, mixed)?  
**Why it matters:** Determines whether the end-state should evolve toward a single authoritative core vs distributed state ownership.  
**What to inspect in code:** engine core modules, broker/store classes, accounting/position trackers, Rust bindings/core usage paths, mode-specific codepaths.  
**Decision needed:** Define the final authority model and what components are forbidden from owning duplicated business state.

---

### Q2. Is the current system already close to an event-sourced model, or is it transition-based without durable event semantics?
**Question:** Is there a canonical event stream today (explicit or implicit), and is it persisted/replayable?  
**Why it matters:** Determinism claims depend on event sequencing, persistence, and replay semantics.  
**What to inspect in code:** event classes, queues, callbacks, broker notifications, logs, replay/backtest harnesses, state snapshots.  
**Decision needed:** Choose the end-state persistence/replay model:
- event-sourced (WAL + replay)
- snapshot + append-only journal
- snapshot-only with limited audit replay
- hybrid

---

### Q3. What should the concurrency model be in the final design?
**Question:** Should the deterministic core be single-threaded, symbol-partitioned, actor-based, or something else?  
**Why it matters:** Concurrency model determines correctness complexity, performance ceilings, and replay determinism.  
**What to inspect in code:** async tasks, callback paths, locking, queues, thread usage, shared mutable state, adapter IO concurrency.  
**Decision needed:** Pick the final mutation/concurrency model and define which boundaries may be concurrent.

---

### Q4. Where should risk checks live, and are risk decisions part of the deterministic event stream?
**Question:** Are risk checks currently pre-engine filters, inside execution paths, inside brokers, or in strategy code?  
**Why it matters:** If risk decisions are not represented in the event stream, replay fidelity and auditability break.  
**What to inspect in code:** pre-trade checks, order validation, broker submit paths, any “risk” modules, strategy wrappers.  
**Decision needed:** Specify final risk architecture:
- pre-core
- in-core
- layered (static + dynamic)
- and whether risk accept/reject is an explicit event

---

### Q5. What are the actual mode divergences today (especially fill timing, sequencing, accounting visibility)?
**Question:** What semantic divergences exist between backtest/paper/live/sandbox right now, and which are intentional?  
**Why it matters:** This directly affects the end-state mode contract spec.  
**What to inspect in code/tests:** mode divergence docs/ledgers, broker implementations, backtest fill simulator, live adapters, contract/integration tests.  
**Decision needed:** Define:
- shared invariants across all modes
- allowed divergences
- forbidden divergences

---

### Q6. What is the desired backtest execution realism model?
**Question:** Should backtest execution be:
- simple bar-based fill model,
- tick/orderbook simulation,
- pluggable execution model(s),
- venue-specific simulation policies?  
**Why it matters:** This controls comparability between backtest and live and affects architecture boundaries.  
**What to inspect in code:** fill simulation modules, backtest broker logic, slippage/latency modeling, test expectations.  
**Decision needed:** Specify final simulation contract and where it plugs into the engine.

---

### Q7. What is the final Rust↔Python boundary (and what survives as Python)?
**Question:** Which logic should permanently live in Rust vs Python?  
**Why it matters:** Avoids endless parity drift and architecture churn.  
**What to inspect in code:** current Rust modules, PyO3/FFI bindings, duplicated logic, tests, performance hotspots, strategy APIs.  
**Decision needed:** Finalize one of:
- Rust authority + Python strategy interface
- Rust libs + Python orchestrator
- shared spec + dual implementations (if justified)
- another model with clear anti-drift mechanisms

---

### Q8. How is precision/quantization handled today across CCXT and Polymarket, and where should it live?
**Question:** Is rounding/step-size enforcement centralized or scattered?  
**Why it matters:** Precision bugs are silent account killers.  
**What to inspect in code:** order normalization, adapter submit code, fee handling, quantity/price formatting, tests for venue precision.  
**Decision needed:** Define the canonical precision model and whether quantization happens:
- at domain boundary
- in adapter boundary
- both (with assertions)

---

### Q9. What should the canonical venue abstraction be (especially for Polymarket)?
**Question:** Can CCXT-style adapters and Polymarket coexist under one clean abstraction without leaky assumptions?  
**Why it matters:** Polymarket is not just “another CCXT exchange”; lifecycle and settlement semantics differ.  
**What to inspect in code:** Polymarket integration modules, order/fill/settlement handling, market metadata models, adapter interfaces.  
**Decision needed:** Define the end-state venue abstraction:
- common base + venue extensions
- capability-based interfaces
- separate execution/accounting adapters per venue family

---

### Q10. What are the crash recovery semantics?
**Question:** On process restart, what state is restored, reconstructed, reconciled, or discarded?  
**Why it matters:** This is a core production property and currently underspecified.  
**What to inspect in code:** startup/bootstrap code, persisted state, order rehydration, reconnect/reconcile logic, runbooks/docs.  
**Decision needed:** Define end-state crash recovery contract:
- what persists
- what is replayed
- what is reconciled
- what halts the engine pending operator intervention

---

### Q11. What observability data is currently available, and what is missing for postmortem-grade debugging?
**Question:** Can the current system reconstruct a causal timeline for a trade lifecycle?  
**Why it matters:** Without this, debugging live issues becomes folklore.  
**What to inspect in code:** logging, structured log fields, tracing, metrics, event IDs/correlation IDs, snapshot/debug tooling.  
**Decision needed:** Define end-state observability contract and required schemas.

---

### Q12. Are the proposed performance budgets realistic for this architecture?
**Question:** What are the actual hotspots and boundary costs (Python callbacks, serialization, logging, adapter IO)?  
**Why it matters:** Unrealistic budgets create bad architectural decisions.  
**What to inspect in code/tests:** benchmarks, profiling hooks, backtest throughput, Python↔Rust crossing frequency, allocations, serialization format.  
**Decision needed:** Set defensible performance budgets for:
- internal loop latency
- throughput
- backtest replay speed
- p95/p99 targets
- with hot-path/cold-path distinction

---

## 6) Tentative invariant set (to preserve and formalize)

These are good candidate invariants to treat as architecture-level constraints, subject to code validation and exact naming.

1. **Monotonic sequencing** of processed events within the core mutation stream.
2. **Idempotent event handling** for duplicate reports/messages.
3. **No illegal order-state jumps** (canonical state machine transitions only).
4. **Terminal order states are immutable**.
5. **Fill quantity sum cannot exceed order quantity** (unless a venue-specific exceptional condition is explicitly modeled).
6. **Position updates are derived from fills**, not ad hoc.
7. **Balance updates are consistent with fills + fees + transfers**.
8. **No negative available balance** unless margin/borrow is explicitly enabled and modeled.
9. **Quantization/precision rules are explicit and enforced before venue submission**.
10. **Every accepted engine order maps to a unique external order reference or explicit rejection outcome**.
11. **Duplicate external events produce no duplicate accounting effects**.
12. **Timestamps and sequence semantics are explicit** (including handling for skew/out-of-order data).
13. **Risk rejections are auditable** (ideally evented).
14. **Mode-specific divergence is documented and tested**, not accidental.
15. **Reconciliation never silently mutates state without an audit trail**.

---

## 7) Documentation/operational guardrails (known useful direction)

Even before final architecture decisions, the repo likely benefits from a stable documentation spine.

### 7.1 Suggested docs structure
- `docs/architecture/end-state-knowns-and-open-questions.md` (this file)
- `docs/architecture/invariants.md` (canonical invariant list and rationale)
- `docs/architecture/mode-contracts.md` (backtest/paper/live shared semantics + allowed divergence)
- `docs/architecture/venue-capabilities.md` (CCXT + Polymarket capability matrix)
- `docs/adr/` (architecture decisions)
- `agents.md` (AI/human operational rules for repo interaction)

### 7.2 `agents.md` should eventually include
- how to run core tests / contract tests / integration tests
- rules for touching accounting/execution code
- rules for updating invariants and mode contracts
- how to add a venue adapter
- how to add/extend Rust bindings
- how to run replay/reconciliation tooling
- required evidence for architecture changes (tests, benchmarks, docs)

---

## 8) What this document is / is not

This document **is**:
- a stable architecture anchor
- a knowns-vs-unknowns ledger
- a decision intake for codebase-driven architecture review

This document is **not**:
- a refactor plan
- a migration roadmap
- a promise that every provisional statement is already true in the codebase

---

## 9) Decision record template (for resolving open questions)

When resolving any question above, use this template:

- **Question ID:** Qx
- **Decision:** (chosen answer)
- **Confidence:** High / Medium / Low
- **Evidence from code/tests:** (file paths, symbols, tests)
- **Why this choice fits the end-state:** (architectural rationale)
- **What remains unresolved:** (if anything)