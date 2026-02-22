# Cracktrader Target Architecture & Refactor Specification (Codex-Facing)

**Status:** Working architecture & refactor specification  
**Audience:** Codex (primary), human reviewer (secondary)  
**Companion to:** `Cracktrader Refactor Execution Playbook` (`cracktrader_refactor_execution_playbook.md`)  
**Purpose:** Define the target architecture, design principles, boundary ownership, and phased refactor outcomes for cracktrader so implementation decisions are consistent, deterministic, and testable across backtest/paper/sandbox/live modes.

**Phase 0 tracking artifacts:**
- `docs/architecture/refactor_decisions.md`
- `docs/architecture/mode_divergence_ledger.md`
- `docs/architecture/refactor_test_and_perf_workflow.md`

---

## 1. Why this document exists

This document defines:

1. **What this software should become** (target architecture and design principles).
2. **What changes should be made** (refactor phases and task slices).
3. **How to perform the work safely** (tests, performance checks, migration discipline).

This is **not** a “preserve legacy APIs at all costs” document. Breaking changes are allowed and expected **if they improve structure** and **tests/examples are updated accordingly**.

This is also **not** a big-bang rewrite plan. Changes should be incremental, committed in small slices, with behavior protected by tests and parity checks.

---

## 2. High-level goals (what “better” means)

The refactor should improve the system along these axes:

- **Clarity**
  - Stronger boundaries between concerns (market data, execution, state, strategy, exchange transport).
  - Better naming (names should reflect responsibilities, not historical accidents).
  - Fewer overlapping or partially-duplicated subsystems.

- **Determinism / parity**
  - Backtest, paper, sandbox, and live should be easier to reason about and compare.
  - Python vs Rust deterministic core parity should remain a first-class safety net.

- **Reliability**
  - Fewer state divergence bugs.
  - Clearer order/account lifecycle invariants.
  - Explicit handling of reconciliation and mismatch cases.

- **Extensibility**
  - Adding an exchange should not require editing multiple unrelated layers.
  - Strategy APIs should be stable and deterministic.
  - Execution semantics should be pluggable (simulation vs exchange transport).

- **Performance**
  - Eliminate obvious hot-path inefficiencies in Python.
  - Improve instrumentation and make perf regressions visible.
  - Move high-ROI deterministic logic toward Rust over time.

---

## 3. Non-goals (for the current refactor program)

To prevent scope explosion, the following are **not** goals of the initial phases:

- No big-bang rewrite of all broker/store/factory code.
- No immediate full Rust rewrite of all runtime paths.
- No exchange onboarding initiative during boundary extraction (unless explicitly in a later phase).
- No speculative redesign of strategy features beyond boundary stabilization.
- No broad “cosmetic-only” repo churn without architecture payoff.
- No forced backward compatibility if it significantly slows structural cleanup.

---

## 4. Codex rules of engagement (important)

These are execution rules for Codex while applying this spec.

### 4.1 General approach
- Perform changes **phase-by-phase**, **PR-slice-by-PR-slice**, **commit-by-commit**.
- Prefer **wrappers/adapters/extractions** before moving behavior.
- Do not do a giant rewrite “because the target architecture is clear.”
- If a boundary is unclear, implement the **minimum seam** that enables tests and future movement.

### 4.2 Breaking changes policy
- Breaking changes are allowed.
- If a public API / example / test breaks, **update the tests and examples in the same slice**.
- Do not preserve legacy signatures purely for sentimentality if they block architecture cleanup.
- If a rename is high-value but risky, use temporary compatibility aliases/comments during migration.

### 4.3 Naming policy
- Rename for correctness **now**, especially where names are misleading and blocking architecture clarity.
- Do not perform repo-wide rename churn unless tied to an active refactor slice.
- When renaming:
  - update tests/examples/docs,
  - add brief migration notes,
  - optionally leave “formerly X” comments in transitional code.

### 4.4 Test discipline
- Every non-trivial refactor slice must include:
  - tests updated/added,
  - a statement of what behavior was preserved/changed.
- Prefer contract tests and parity tests over brittle internals tests.
- Avoid introducing new tests that depend on private internals if public harnesses can be used.

### 4.5 Performance discipline
- Do not claim performance wins without measurement.
- For hot-path changes, capture before/after with existing benchmark/perf instrumentation.
- If perf regresses, document why (and whether acceptable).

### 4.6 Scope discipline
- Touch only files relevant to the current slice unless necessary.
- Avoid mixing architecture extraction and unrelated feature work.
- If a tempting cleanup appears, record it in a TODO/notes section, do not silently expand scope.

---

## 5. Current architecture (summary and refactor focus)

The current system is a multi-exchange trading platform with shared factory composition (`Store` / `Feed` / `Broker`) plus a deterministic engine path (Python today, Rust bridge present and expanding). It supports backtest/paper/sandbox/live and has extensive tests (unit / contracts / integration / e2e / performance / parity).

### Primary structural problems to fix
1. **Execution semantics duplicated across simulation/live broker classes**
2. **Market data aggregation/streaming logic split across overlapping layers**
3. **State duplicated across deterministic core and broker/account internals**
4. **Exchange normalization logic scattered across stores/brokers/ws handlers**
5. **Strategy boundary partially duplicated / insufficiently deterministic across modes**

These become the core refactor themes.

---

## 6. Target architecture principles (more important than exact names)

This section defines what the system should *feel like structurally*.

### 6.1 Separate “what happened” from “what to do”
- Market/exchange updates and normalized events should be explicit objects/events.
- Strategy decisions should be explicit outputs (commands/intents), not hidden side effects.
- Order/account state should be derived from events through well-defined transitions.

### 6.2 One responsibility per boundary
A boundary should own one kind of logic:
- **Market data ingestion/normalization**
- **Execution transport and execution semantics**
- **Order state transitions**
- **Account/balance/position state**
- **Strategy decisioning**
- **Exchange-specific adaptation**

Avoid boundaries that partially own behavior and partially leak it upward.

### 6.3 Determinism first (especially in core and strategy seams)
- Same input batch + same account snapshot + same config = same strategy output.
- Event sequencing rules should be explicit.
- Replay parity should be testable and a normal workflow, not a special effort.

### 6.4 Normalize at the edge
Exchange-specific weirdness should be handled close to exchange adapters, not spread across broker/store/core layers.

### 6.5 Prefer explicit reconciliation over implicit “single source of truth” claims
A single source of truth only works if mismatch policy is defined:
- which source is authoritative by mode,
- how mismatches are detected,
- what action is taken (warn / repair / fail / halt).

### 6.6 Extract behavior seams before platform seams
Do not introduce six new abstractions at once. First extract the seam that removes duplicated behavior and enables tests.

---

## 7. Naming and terminology plan (renames are allowed now)

Exact naming can be adjusted, but names should follow these principles:

### 7.1 Naming principles
- Use **Adapter** for exchange-specific transport + normalization.
- Use **Policy** for pluggable execution semantics (e.g., simulated vs exchange).
- Use **State** for canonical derived state (order/account snapshots).
- Use **Feed** for normalized market-data streams (not mixed execution logic).
- Use **Executor** for order submit/cancel/update transport behavior.
- Use **Contract** for test-enforced interface behavior across implementations.

### 7.2 Recommended naming directions (initial)
These are recommended, not absolute, but prefer them unless there is a better fit.

- `ExecutionPolicy` (new): owns execution semantics (simulation / exchange behavior semantics)
- `SimulatedExecutionPolicy` (new)
- `ExchangeExecutionPolicy` (new)
- `AccountState` (new): canonical balances/positions/pnl views and reconciliation hooks
- `ExchangeAdapter` (new): exchange transport + normalization
- `MarketDataFeed` (new/protocol): normalized subscriptions/snapshots
- `StrategyInput` / `StrategyOutput` (new): deterministic strategy IO
- `OrderState` / `OrderStore` / `OrderStateMachine` (new or extracted): canonical order transitions

### 7.3 Rename execution strategy
- Rename **as part of functional refactors**, not in an isolated “rename-only mega-PR.”
- Temporary aliases/comments are acceptable:
  - e.g. `# Formerly CCXTSimulationBroker-specific fill loop`
- Update tests/examples/docs in the same slice.

---

## 8. Target architecture shape (conceptual)

This is the target shape. Codex may choose exact implementation details if it preserves these boundaries.

### 8.1 Market data boundary
**Responsibility:** provide normalized market streams and snapshots.

Should own:
- subscriptions for trades/orderbook/ohlcv
- normalization into canonical market event shapes
- timestamp and sequence monotonicity guarantees (or explicit rules)
- aggregation policy wiring (where applicable)

Should not own:
- order execution
- account mutation logic
- strategy side effects

### 8.2 Execution boundary
Split into two concepts:

#### A) Order executor / transport
**Responsibility:** submit/cancel orders, stream exchange updates, normalize acknowledgements and updates.

#### B) Execution policy (semantics)
**Responsibility:** execution behavior semantics (simulation fill/expiry/slippage/partials vs exchange-driven behavior)

This avoids each broker subclass re-implementing lifecycle and fill behavior.

### 8.3 Order state boundary
**Responsibility:** canonical order transition legality and queryable order state.

Should own:
- legal transitions
- transition history
- idempotency handling for duplicate updates
- status derivation

Should not own:
- network calls
- exchange websocket handling
- account pnl math (except event fields that are part of order/fill records)

### 8.4 Account state boundary
**Responsibility:** canonical balances/positions/pnl and reconciliation behavior.

Should own:
- derived balances/positions
- realized/unrealized pnl logic
- margin usage (if applicable)
- reconciliation comparisons and mismatch policy
- mode-dependent authority rules (backtest/paper/live)

Should not own:
- strategy generation
- order transport calls
- exchange-specific parsing

### 8.5 Exchange adapter boundary
**Responsibility:** isolate exchange-specific APIs and normalize to canonical events/commands.

Should own:
- create/cancel/query/stream transport details
- normalization of order/fill/trade/balance payloads
- exchange capability exposure

Should not own:
- strategy logic
- global account state decisions
- deterministic engine transitions

### 8.6 Strategy boundary
**Responsibility:** deterministic mapping from inputs to decisions.

Should own:
- transformation of market/account inputs into intents/commands
- optional diagnostics/telemetry outputs

Should not own:
- order execution side effects
- internal store/broker private method calls
- hidden mutable state that breaks replay determinism (unless explicit and serialized)

---

## 9. Canonical contracts (draft, principle-led)

These are architectural contract shapes. Codex can refine signatures, but must preserve responsibilities and determinism.

> Note: These are *guiding contracts*, not strict final signatures for all phases.

### 9.1 Market data feed (concept)
- subscribe to normalized market streams by explicit stream type (trade/orderbook/ohlcv), symbol, timeframe (if applicable)
- provide snapshot access where appropriate
- expose sequencing/timestamp guarantees in docs/tests

### 9.2 Exchange adapter (concept)
- submit/cancel/query
- stream updates
- normalize external payloads into canonical events
- expose capabilities (supported order types, flags, etc.)

### 9.3 Execution policy (concept)
- evaluate whether/how orders progress (simulation or exchange-backed flow semantics)
- handle expiry, slippage, partials, and fill completion semantics (for simulated policy)
- avoid duplicating identical fill-loop logic across brokers

### 9.4 Order state machine/store (concept)
- apply canonical order events
- enforce legal transitions
- provide current state + transition history
- idempotency on duplicate events

### 9.5 Account state (concept)
- apply relevant events (fills, fees, funding, balance snapshots, corrections)
- derive canonical snapshot
- reconcile against exchange snapshots when relevant
- expose mismatch status and policy actions

### 9.6 Strategy IO (concept)
- `StrategyInput`: market data + account snapshot + metadata + deterministic keys
- `StrategyOutput`: intents/commands + optional diagnostics
- deterministic output under same input

---

## 10. Reconciliation and “single source of truth” policy (must be explicit)

This is critical. “Single source of truth” is not enough without mismatch policy.

### 10.1 Authority by mode (target behavior)
Codex should implement or prepare for explicit mode-based authority rules.

- **Backtest**
  - Authority is local deterministic state (simulated execution + deterministic core)
- **Paper / sandbox**
  - Local state is primary for immediate transitions; exchange snapshots/updates may reconcile depending on transport fidelity
- **Live**
  - Exchange-reported updates/snapshots are authoritative for external truth, but local derived state remains required for deterministic behavior and diagnostics

### 10.2 Mismatch handling policy (staged)
Implement in stages:

- **Stage A (read-through wrapper only):**
  - no behavior change
  - compare views only when possible

- **Stage B (mirror writes + compare):**
  - maintain current writes
  - compute `AccountState` view in parallel
  - record mismatches (warn/diagnostic)

- **Stage C (test hard-fail on mismatches):**
  - enforce parity in tests/invariants
  - production behavior may still degrade to warning based on mode/config

- **Stage D (canonical AccountState):**
  - route mutations through canonical state service
  - legacy mirrored state removed

### 10.3 Mismatch severity classes (recommended)
Introduce categories for reconciliation output:
- `INFO` (transient/inconsequential)
- `WARN` (recoverable mismatch)
- `ERROR` (state divergence requiring repair)
- `FATAL` (unsafe to continue in mode/context)

---

## 11. Intentional divergence ledger (backtest / paper / sandbox / live)

Codex should create and maintain a machine-readable or markdown ledger of divergences.

### 11.1 Purpose
Track differences between modes so the team can distinguish:
- intended divergences,
- accidental divergences,
- unverified divergences.

### 11.2 Minimum schema (markdown table or JSON/YAML)
Each divergence entry should include:
- `id`
- `area` (execution/accounting/latency/order-types/etc.)
- `modes_affected`
- `description`
- `intentional` (yes/no/unknown)
- `reason`
- `test_coverage` (path or none)
- `risk`
- `status` (open/verified/retired)

### 11.3 Examples of likely entries
- Simulated fill timing vs exchange acknowledgements
- Partial fill sequencing differences
- Balance update timing/order in live vs simulation
- Order cancellation race semantics

---

## 12. Phased refactor plan (implementation program)

This is the executable program. Codex should work phase-by-phase, slicing each phase into PR-sized tasks and committing as it goes.

---

### Phase 0 — Baseline hardening (pre-refactor safety setup)
**Goal:** Increase confidence and observability before moving boundaries.

#### Outcomes
- Confirm baseline test collection / major suite health
- Identify hot-path benchmarks and baseline perf measurement workflow
- Add/prepare divergence ledger
- Document initial architecture assumptions and unknowns

#### Tasks (PR-sized)
1. Add/seed divergence ledger (initial entries can be marked unknown)
2. Add/refine adapter contract test scaffold (empty or minimal if needed)
3. Add/refine state/account mismatch diagnostic hooks (non-invasive)
4. Capture benchmark/perf baseline instructions and output locations

#### Acceptance criteria
- Baseline suites run as documented
- Divergence ledger exists
- No behavior changes introduced
- Perf baseline capture path documented and reproducible

---

### Phase 1 — Extract execution semantics seam (highest ROI)
**Goal:** Remove duplicated simulation execution behavior by introducing `ExecutionPolicy` and applying it to the first two brokers.

#### Scope (initial)
- `BaseBackBroker`
- `CCXTSimulationBroker`

#### Why first
This removes duplication with manageable blast radius and creates a repeatable pattern before touching all simulation/live brokers.

#### Target outcome
- Shared simulation execution behavior is implemented in one place (policy)
- Broker classes delegate execution semantics instead of owning duplicate fill loops
- Public behavior is preserved or explicitly documented when changed
- Contract/integration tests remain green (or updated to reflect intended changes)

#### Implementation requirements
- Introduce `ExecutionPolicy` boundary (name can be refined if a better one follows naming principles)
- Extract duplicated behavior (e.g. trigger eval, local fill completion, expiry checks) into policy
- Keep broker class responsibilities focused on routing/notifications/state integration
- Do not simultaneously redesign all brokers

#### Acceptance criteria
- `BaseBackBroker` uses extracted policy
- `CCXTSimulationBroker` delegates to same policy
- Duplicate execution logic reduced materially
- Relevant contract + integration tests updated and passing
- Behavioral differences (if any) are documented in PR notes and/or divergence ledger

#### Perf requirements
- No major regression in backtest/simulation hot paths without justification
- Benchmark/perf smoke run executed if hot path changed materially

---

### Phase 2 — Expand execution seam across simulation brokers + live semantics cleanup
**Goal:** Apply the pattern consistently and reduce mode drift.

#### Scope
- Remaining simulation brokers (e.g., Polymarket/Kalshi simulation brokers)
- Live broker local simulation fallback behavior (make explicit / constrained)

#### Target outcome
- Simulation execution semantics are centralized
- Live brokers do not silently contain simulation semantics except where explicitly configured/tested
- Mode divergence becomes more explicit and testable

#### Acceptance criteria
- All simulation brokers use shared execution semantics or documented capability-driven variants
- Live fallback behavior is explicit and configurable (not hidden accidental simulation)
- New/updated mode parity tests cover critical flows on mocked transport
- Divergence ledger updated

---

### Phase 3 — Account state read-through wrapper and reconciliation scaffolding
**Goal:** Start resolving duplicated core/broker account state without destabilizing behavior.

#### Scope
- Introduce `AccountState` read-through wrapper
- No immediate canonical writes
- Add compare/mismatch diagnostics

#### Target outcome
- Account state is representable through one coherent service/view
- Existing state remains intact during transition
- Mismatch detection starts generating useful signals

#### Implementation stages
1. Read-through wrapper (no behavior change)
2. Optional mirror-write mode (behind config/test path)
3. Snapshot compare + mismatch reporting
4. Invariant assertions in tests

#### Acceptance criteria
- `AccountState` wrapper exists and is used in at least one path
- Snapshot compare works in tests for targeted scenarios
- Mismatch outputs are categorized (severity / type)
- No unintentional behavior changes in core accounting tests

---

### Phase 4 — Exchange adapter contract + CCXT adapter implementation
**Goal:** Normalize exchange-specific behavior at the edge and reduce scattered branching.

#### Scope (first pass)
- Adapter contract(s)
- CCXT adapter implementation (thin wrapper over existing transport/store behavior)
- Adapter contract tests

#### Target outcome
- A first real adapter exists and is test-enforced
- Broker/store code begins depending on normalized adapter outputs instead of raw exchange payloads
- A path exists for Polymarket/Kalshi migration later

#### Acceptance criteria
- `ExchangeAdapter` contract defined and tested
- CCXT adapter passes adapter contract tests
- At least one live/simulation path consumes adapter-normalized outputs
- No broad rewrite of all exchanges in same phase

---

### Phase 5 — Market data boundary consolidation (with OHLCVSubsystem decision checkpoint)
**Goal:** Resolve split ownership of market-data aggregation/streaming logic.

#### Problem being solved
Market data behavior is currently split across store subsystems, feed drivers, and an apparently disconnected/test-only OHLCV subsystem.

#### Required decision checkpoint: `OHLCVSubsystem`
Codex must not silently delete or wire this in without evaluation.

##### Decision criteria
Evaluate and document:
- Is it used in runtime paths?
- Does it duplicate active production logic?
- Does it encode behavior/tests worth preserving?
- Is it a better host for existing OHLCV logic than current paths?

##### Allowed outcomes
- **Retain and wire into runtime**
- **Retain as internal/test utility**
- **Deprecate and remove (with test replacements)**
- **Merge into another subsystem**

#### Target outcome
- One clear owner (or contract) for normalized market data / OHLCV aggregation behavior
- Feed drivers consume normalized streams rather than re-implementing aggregation logic
- Timestamp/sequence monotonicity is contract-tested

#### Acceptance criteria
- `OHLCVSubsystem` decision recorded (doc + code comments/PR notes)
- Market data contract tests added/updated
- Duplication in aggregation paths reduced materially
- No silent behavioral drift in data quality tests

---

### Phase 6 — Strategy IO determinism contract and adapters
**Goal:** Unify strategy boundary semantics without overhauling all strategy implementations at once.

#### Scope
- Introduce `StrategyInput` / `StrategyOutput` dataclasses/contracts
- Add adapters around existing native/backward-compat strategy paths
- Add determinism tests

#### Target outcome
- Strategy boundary is explicit, deterministic, replay-friendly
- Existing strategy implementations can be adapted rather than rewritten immediately
- Fewer tests depend on private feed/store internals

#### Acceptance criteria
- Determinism contract test exists and passes for at least one representative strategy path
- Strategy adapters cover current runtime path(s)
- At least one private-internals integration test is converted to public harness usage

---

### Phase 7 — Adapter expansion (Polymarket, Kalshi, others)
**Goal:** Finish moving exchange-specific normalization/transport behind adapters.

#### Scope
- Polymarket adapter
- Kalshi adapter
- Additional adapters as needed
- Capability matrix alignment

#### Target outcome
- Exchange-specific branching reduced in brokers/stores/factory
- Adapter contract suite is the primary exchange integration safety net

#### Acceptance criteria
- Polymarket and Kalshi adapters implemented (or documented blockers)
- Adapter contract tests cover all active adapters
- Exchange-specific broker branching materially reduced
- Capability exposure standardized enough for tests/automation

---

### Phase 8 — Rust boundary expansion (sequencer / invariants / normalization core)
**Goal:** Move additional deterministic, high-ROI logic into Rust after Python seams are stable.

#### Sequence (recommended)
1. Sequencer
2. Invariants
3. Event normalization helpers / selected high-volume pure logic
4. (Later) simulation execution core after `ExecutionPolicy` seam is mature

#### Requirements
- Maintain Python fallback
- Extend parity tests for newly migrated logic
- Keep adapter boundary stable so transport remains Python for now

#### Acceptance criteria
- New Rust-backed logic has parity contract coverage
- Python fallback path remains available
- Integration parity suite remains green (or changes are intentional and documented)

---

## 13. Test architecture strategy (1700+ tests)

The test suite is a strategic asset. Refactoring should strengthen, not bypass, it.

### 13.1 What to preserve
- Strict network hermeticity and test control plane
- Python/Rust parity contracts
- Integration parity paths
- Public API / examples tests (even if APIs change, the discipline should remain)

### 13.2 What to improve (priority)
#### Priority A (do during refactors)
- Add contract tests for new boundaries:
  - execution policy
  - exchange adapters
  - account state / order state
  - strategy determinism
- Replace private internals test pokes when touching related tests
- Add mode parity matrix tests for critical flows

#### Priority B (selective cleanup)
- Reduce print-heavy / noisy fixtures in hot paths
- Convert brittle integration tests to public harness helpers
- Consolidate repeated test setup patterns when already touching those files

### 13.3 Suggested larger test refactor ideas (optional / later)
These are suggestions, not initial requirements:
- Split integration fixture “control plane” vs “scenario builders”
- Create reusable mode parity harness utilities
- Create adapter contract test fixture package under `tests/contracts/adapters/`
- Create state contract test package under `tests/contracts/state/`
- Add deterministic replay golden scenarios package

### 13.4 Property-based tests (defer implementation, define now)
Property-based testing is recommended but not required in the first refactor waves.

#### Define properties now (implement later)
1. **Order transition legality**
   - No illegal state transitions under arbitrary valid/invalid event sequences
2. **Cash/position conservation**
   - Fill permutations preserve accounting invariants (with fees/partials)
3. **Event sequencing monotonicity**
   - Sequencer yields monotonic ordering even under equal-timestamp shuffle rules

#### Guidance
- Add spec placeholders/tests marked xfail/skip if needed
- Introduce Hypothesis (or chosen PBT lib) later when core seams are stable

---

## 14. Performance and benchmarking policy

Performance matters, but target numbers are not yet fixed. Use a baseline-and-regression policy.

### 14.1 Baseline-first policy
Before major hot-path refactors:
- capture baseline benchmark/perf output using existing tools/instrumentation
- record commands and outputs (or paths)

### 14.2 Acceptable performance posture (until formal targets exist)
- No significant regression in key hot paths without written justification
- Small regressions may be acceptable if they materially improve correctness/structure and are temporary
- Hot-path inefficiency fixes should be prioritized when low-risk and obvious

### 14.3 Immediate low-risk optimizations (can be folded into relevant phases)
Examples (if validated by current code state):
- replace O(n) list front pops with deque popleft in feed loading paths
- reduce hot-path warning logs to debug where warnings are emitted on normal paths
- reduce repeated parsing/type checks inside simulation loops
- reduce lock churn in queue/dispatcher loops
- use running aggregates instead of repeated sorting where possible

### 14.4 Performance acceptance criteria for hot-path slices
For any slice touching engine/feed/execution hot paths:
- before/after perf measurement captured
- regression status stated
- if regressed, rationale + follow-up plan recorded

---

## 15. Branching / PR / commit workflow (Codex execution instructions)

Codex should operate in small, reviewable slices.

### 15.1 Branch strategy
Use **one branch per PR-sized task/slice** (recommended), or at most per phase if explicitly requested.

### 15.2 Commit strategy
Commit as work progresses. Do not accumulate giant uncommitted refactors.

Recommended commit cadence:
- extraction scaffolding
- behavior delegation switch
- tests updated/added
- cleanup/removal of dead duplicate code
- docs/ledger updates

### 15.3 PR slice template (internal checklist)
Each PR-sized slice should include:
- **Intent** (what architectural move this slice makes)
- **Scope** (files touched)
- **Behavior changes** (if any)
- **Tests run / added**
- **Perf check** (if hot path touched)
- **Divergence ledger updates** (if mode behavior affected)
- **Open follow-ups** (if intentionally deferred)

---

## 16. “Do not change yet” list (unless a phase explicitly says so)

This list prevents accidental scope creep.

- Do not rewrite the entire factory/session composition system in one pass.
- Do not expand Rust boundary before Python seams for that area are stable.
- Do not attempt a full strategy framework rewrite during execution/accounting seam extraction.
- Do not remove parity tests because they are inconvenient during refactors.
- Do not delete ambiguous subsystems (e.g., `OHLCVSubsystem`) without a documented decision checkpoint.
- Do not mix exchange onboarding feature work into architecture extraction phases.

---

## 17. File/path guidance (conceptual, not mandatory exact names)

Codex may choose exact file names, but should prefer coherent placement and stable vertical boundaries.

### 17.1 Suggested new/refined areas
- `src/cracktrader/execution/` or `src/cracktrader/broker/execution_*`
  - execution policy interfaces and implementations
- `src/cracktrader/exchanges/adapters/`
  - adapter base and exchange implementations
- `src/cracktrader/state/` or `src/cracktrader/engine/state/`
  - account/order state and reconciliation logic
- `src/cracktrader/strategy/contracts.py` (+ adapters)
  - deterministic strategy IO boundary
- `tests/contracts/adapters/`
- `tests/contracts/state/`
- `tests/parity/` (optional; if useful beyond existing locations)

### 17.2 Placement principle
Prefer placing code near the layer that owns the responsibility today, unless that layer is precisely what is being decomposed.

---

## 18. Acceptance criteria (global definition of “done enough” for this refactor program)

The refactor program is considered successful when the following are true (not necessarily all at once, but as cumulative outcomes):

### 18.1 Architecture outcomes
- Execution semantics are no longer duplicated across multiple simulation brokers
- Exchange-specific normalization is substantially moved to adapters/contracts
- Account state has an explicit reconciliation-aware canonical boundary
- Strategy IO determinism is explicit and testable
- Market data ownership is clearer and overlapping aggregation paths are reduced

### 18.2 Test outcomes
- Contract tests exist for major new boundaries
- Mode parity coverage improves
- Python/Rust parity discipline is preserved and extended where relevant
- Brittle private-internals test dependence is reduced in touched areas

### 18.3 Performance outcomes
- No major untracked regressions in key hot paths
- Benchmark/perf measurements are part of hot-path refactor workflow
- At least a few obvious inefficiencies are removed

### 18.4 Documentation/process outcomes
- Divergence ledger exists and is maintained
- Refactor slices are documented with intent/scope/behavior/test notes
- Unknowns and decision checkpoints are recorded, not silently resolved

---

## 19. Open questions / decision checkpoints (must be recorded during implementation)

Codex should explicitly maintain/update this section (or a separate tracking doc) as it proceeds.

### 19.1 `OHLCVSubsystem` runtime role
- Runtime owner?
- Test-only?
- Merge/remove decision and why?

### 19.2 Final naming of major boundaries
- Are `ExecutionPolicy`, `ExchangeAdapter`, `AccountState`, etc. the best names?
- If changed, does the new naming follow the principles in this doc better?

### 19.3 Live-mode reconciliation strictness
- Which mismatches should hard-fail vs warn by default?
- What is configurable by mode/env?

### 19.4 Strategy compatibility surface
- How much legacy strategy API should be adapted vs retired?
- What migration path is acceptable?

---

## 20. Immediate next steps (recommended order of attack)

This is the recommended start sequence for Codex.

1. **Phase 0 baseline hardening**
   - seed divergence ledger
   - confirm test/perf baseline workflow
   - add/refine contract test scaffolds

2. **Phase 1 execution seam extraction**
   - introduce `ExecutionPolicy`
   - wire `BaseBackBroker`
   - wire `CCXTSimulationBroker`
   - update tests and measure behavior/perf

3. **Phase 3 account read-through wrapper (early)**
   - introduce `AccountState` view and mismatch compare hooks
   - no canonical writes yet

4. **Phase 4 adapter contract + CCXT adapter**
   - define contract
   - implement CCXT adapter
   - start consuming normalized outputs in one path

Then proceed to broader adapter expansion / market data consolidation / strategy IO contracts.

---

## 21. Appendix: implementation posture summary (for Codex)

When in doubt, follow this order of priorities:

1. **Correctness and determinism**
2. **Test safety and parity coverage**
3. **Clarity of boundaries**
4. **Extensibility**
5. **Performance**
6. **Cosmetic cleanup**

And remember the meta-rule:

> Extract the smallest seam that removes duplication or clarifies ownership, prove it with tests, then continue.
