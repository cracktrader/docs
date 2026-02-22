# Cracktrader Refactor Execution Playbook (Codex-Facing)
**Status:** Working refactor execution playbook  
**Audience:** Codex (primary), human reviewer (secondary)  
**Companion to:** `Cracktrader Target Architecture & Refactor Specification` (`cracktrader_architecture_refactor.md`)  
**Purpose:** Translate the architecture specification into concrete, staged, PR-sized implementation work with file targets, test targets, acceptance criteria, and execution discipline.

**Phase 0 tracking artifacts:**
- `docs/architecture/refactor_decisions.md`
- `docs/architecture/mode_divergence_ledger.md`
- `docs/architecture/refactor_test_and_perf_workflow.md`

---

## 0. How to use this document

This is an **execution playbook**, not just architecture prose.

Codex should:
1. Work **phase-by-phase**.
2. Split each phase into **PR-sized slices**.
3. Use a **branch per slice** (recommended).
4. **Commit as you go** (small, meaningful commits).
5. Update tests/examples/docs in the same slice when behavior or names change.
6. Record any mode divergence changes in the divergence ledger.
7. Create and maintain `docs/architecture/refactor_decisions.md` and log non-trivial architectural/refactor decisions there.
8. Operate **autonomously by default**; only pause for human input when blocked on a real decision (see **1.5 Autonomy & Escalation Policy**).

This document is intentionally concrete about likely files and test paths. If the repo has moved, Codex should:
- verify actual paths,
- map the intent to current files,
- preserve the architecture intent.

---

## 1. Global execution protocol (must follow)

### 1.1 PR slice workflow
For every PR-sized slice:

1. **Read the target files and nearby tests**
2. **Identify exact behavior to preserve/change**
3. **Implement the seam/extraction**
4. **Update affected tests**
5. **Run targeted tests**
6. **Run broader suite if slice impacts shared code**
7. **Run perf smoke (if hot path touched)**
8. **Update divergence ledger/docs if behavior changed**
9. **Update `refactor_decisions.md` for non-trivial choices made during the slice**
10. **Commit in small steps**
11. **Summarize intent + scope + behavior changes**

### 1.2 Commit discipline
Recommended commit sequence within a slice:
- `commit 1`: scaffolding / interfaces / no behavior change
- `commit 2`: wire first path to new seam
- `commit 3`: tests updated/added
- `commit 4`: duplicate logic removal / cleanup
- `commit 5`: docs/ledger notes (if needed)

### 1.3 Test running strategy
Run tests in this order:
1. **Most-local unit/contract tests**
2. **Affected integration tests**
3. **Parity tests** (if touching core/deterministic/event normalization)
4. **Broader suite** only when necessary / at phase checkpoints

### 1.4 Performance check strategy
If touching hot paths (`engine`, `feeds`, `streaming`, broker update loops):
- capture before/after perf using existing tools
- record result (even if rough)
- do not claim perf wins without measurement

### 1.5 Autonomy & Escalation Policy (when Codex should continue vs ask for help)

#### Default posture
Codex should continue working **autonomously through the refactor**, slice by slice, commit by commit, without asking for help on routine implementation details.

Codex should only pause and ask for human input when it is **blocked on a real decision** (not just a hard bug).

#### Codex should continue without asking for help when
These are normal and should be handled autonomously:
- test failures that can be debugged from code/tests
- import/path/type errors
- refactor fallout requiring updates to tests/examples/docs
- naming choices that clearly follow the architecture spec principles
- local design choices within a slice that do not change boundary ownership
- iterative fixes needed to make a slice pass targeted tests
- temporary compatibility shims needed during migration

When ambiguous but non-blocking, Codex should:
1. choose the option that best matches the architecture spec,
2. document the choice in `docs/architecture/refactor_decisions.md`,
3. continue.

#### Codex should stop and ask for help only when one or more of these happen

##### A. Architectural ambiguity that affects boundary ownership
Examples:
- two plausible designs both fit the spec but imply different long-term boundaries
- uncertainty about which subsystem should own a behavior (not just where code lives)
- a change would conflict with the target architecture principles

##### B. Product/behavior ambiguity (semantics decision)
Examples:
- unclear intended behavior for live vs paper vs backtest semantics
- reconciliation mismatch policy choice (warn vs fail vs repair) is required and not specified
- strategy determinism requirements conflict with current behavior and tests do not clarify intent

##### C. Spec conflict or contradiction
Examples:
- execution playbook and architecture spec imply different outcomes
- tests encode behavior that directly conflicts with the stated target architecture
- a prior phase decision blocks a later required phase and no clean path exists

##### D. High-risk destructive change
Examples:
- deleting or replacing a subsystem with unclear runtime usage (e.g., uncertain production dependency)
- changing public APIs in a way that cascades broadly without clear migration intent
- touching live-trading safety-critical behavior where intent is unclear

##### E. Hard block after reasonable attempts
Codex should ask for help if it has made a reasonable effort and is still blocked, e.g.:
- multiple implementation attempts fail due to hidden constraints
- tests are non-diagnostic / conflicting and no clear next move exists
- a required dependency/tooling/runtime assumption is missing

(Use judgment; the point is “genuinely stuck on a decision,” not “first sign of friction.”)

#### When Codex asks for help, it must provide a decision packet (not a vague question)
Codex should pause with a concise summary containing:

1. **What it was trying to do**
2. **Where it is blocked** (file/path + behavior)
3. **Why this is a decision, not just debugging**
4. **Options considered** (2–3 max)
5. **Recommended option** and why (based on the architecture spec)
6. **Impact radius** (files/tests/behavior)
7. **What remains unblocked** (what Codex can continue doing meanwhile, if anything)

Codex should also:
- add a pending entry to `docs/architecture/refactor_decisions.md` marked as **Needs human decision**, with the same summary.

#### Do not ask for help for these reasons
Codex should **not** pause just to ask:
- “Is this naming okay?” (if it follows the naming principles)
- “Should I update these tests too?” (yes, if affected)
- “Should I make a small compatibility shim?” (yes, if it reduces risk)
- “Should I commit now?” (yes, commit in small meaningful slices)
- “I found adjacent cleanup opportunities” (record and continue, don’t expand scope)

#### Escalation hierarchy (decision order)
If unsure, Codex should resolve ambiguity in this order:
1. explicit architecture spec requirement
2. explicit execution playbook requirement
3. existing tests/contracts/parity behavior
4. target architecture principles (clarity, determinism, boundary ownership)
5. smallest reversible change that preserves forward progress

#### Progress-first rule
Unless blocked by a true decision, Codex should keep moving and:
- finish the current slice,
- update tests,
- commit,
- note follow-ups,
- continue to the next slice.

#### Decision logging rule (mandatory)
Codex must maintain `docs/architecture/refactor_decisions.md` as an append-only working record of non-trivial decisions made during implementation.

If the file does not exist, Codex should create it early (Phase 0).

Codex should add an entry when:
- choosing between two or more plausible implementations
- making an intentional behavior change
- introducing a temporary compatibility shim
- deciding to defer cleanup
- resolving ambiguity without asking for human input
- escalating and requesting a human decision

Codex does **not** need to log trivial syntax, imports, or obvious mechanical changes.

---

## 2. Tracking checklist template (use per PR slice)

Codex should maintain a checklist like this in PR notes / local task notes.

### PR Slice Template
- **Slice ID:** `P1-S2` (example)
- **Branch:** `refactor/p1-s2-ccxt-sim-execution-policy`
- **Intent:** (one paragraph)
- **Scope (files):**
  - `...`
- **Behavior changes:** none / intentional (describe)
- **Tests added/updated:**
  - `...`
- **Tests run:**
  - `...`
- **Perf check:** not needed / done (summary)
- **Divergence ledger update:** yes/no
- **Decision log update (`refactor_decisions.md`):** yes/no
- **Open follow-ups:** `...`

### Refactor decision log entry template (`docs/architecture/refactor_decisions.md`)
Codex should use a consistent format such as:

```md
## YYYY-MM-DD — [Phase/Slice] Short decision title
- **Status:** decided / provisional / needs human decision
- **Context:** what was being changed and why
- **Decision:** what was chosen
- **Alternatives considered:** (1–3 bullets)
- **Why this choice:** tie to architecture spec / tests / constraints
- **Impact radius:** files, tests, behavior/modes affected
- **Follow-ups:** deferred cleanup, TODOs, validation needed
```

---

## 3. Phase map (execution order)

Recommended order:
1. Phase 0 — Baseline hardening
2. Phase 1 — ExecutionPolicy extraction (BaseBackBroker + CCXTSimulationBroker)
3. Phase 2 — ExecutionPolicy rollout (remaining simulation brokers + live cleanup)
4. Phase 3 — AccountState read-through + reconciliation scaffolding
5. Phase 4 — ExchangeAdapter contract + CCXT adapter
6. Phase 5 — Market data boundary consolidation (+ OHLCVSubsystem decision checkpoint)
7. Phase 6 — Strategy IO determinism contracts + adapters
8. Phase 7 — Adapter expansion (Polymarket/Kalshi)
9. Phase 8 — Rust boundary expansion (sequencer/invariants/etc.)

Phases may overlap slightly only if one phase creates scaffolding another needs. Do **not** run multiple major rewrites in parallel.

---

## 4. Phase 0 — Baseline hardening (safety setup)

### Goal
Create the minimum safety/control scaffolding before moving architecture boundaries.

### PR Slice P0-S0 — Create refactor decisions log
**Intent:** Create a durable file where Codex records non-trivial implementation decisions and escalations.

#### Likely files
- New doc:
  - `docs/architecture/refactor_decisions.md`

#### Tasks
- Create `docs/architecture/refactor_decisions.md`
- Add a short header explaining purpose and usage
- Add the decision entry template (or equivalent)
- Seed with one initial entry noting that the refactor is starting under the architecture spec + execution playbook
- Link to it from:
  - this execution playbook
  - the architecture/refactor spec (if practical in same slice)

#### Tests
- None required unless doc tooling enforces checks

#### Acceptance
- `docs/architecture/refactor_decisions.md` exists
- Codex can append entries throughout the refactor
- The file is referenced from the refactor docs

---

### PR Slice P0-S1 — Seed divergence ledger
**Intent:** Create a durable place to track intentional/accidental mode divergences.

#### Likely files
- New doc (suggested):
  - `docs/architecture/mode_divergence_ledger.md`
  - or `docs/architecture/mode_divergence_ledger.yaml` (if structured)
- Optional helper references:
  - `docs/architecture/...` (where main spec lives)

#### Tasks
- Create divergence ledger with schema/columns:
  - id
  - area
  - modes_affected
  - description
  - intentional (yes/no/unknown)
  - reason
  - test_coverage
  - risk
  - status
- Seed with known likely entries (mark `unknown` if not validated):
  - simulation fill timing vs exchange updates
  - balance update timing differences
  - cancel race semantics
  - partial fill sequencing differences

#### Tests
- None required unless doc tooling enforces checks

#### Acceptance
- Ledger exists and is referenced from the architecture spec / execution docs
- Entries can be added incrementally

---

### PR Slice P0-S2 — Baseline test + perf workflow notes
**Intent:** Document how Codex should run targeted tests and baseline perf checks before/after hot-path changes.

#### Likely files
- New doc (suggested):
  - `docs/architecture/refactor_test_and_perf_workflow.md`
- Existing tools referenced:
  - `scripts/benchmark_engine_backends.py`
  - `performance/automation/run_benchmarks.py`
  - `performance/bench.py`
  - `src/cracktrader/engine/perf.py`
  - `src/cracktrader/engine/benchmark_compare.py`

#### Tasks
- Document commands/flows (verify current commands)
- Document when to run targeted vs full suites
- Document perf environment variables if still current:
  - `CRACKTRADER_ENGINE_PERF_ENABLED`
  - `CRACKTRADER_ENGINE_PERF_JSONL`
- Add “hot path touched => perf smoke check” rule

#### Tests
- None required

#### Acceptance
- Reproducible instructions exist
- No code behavior changes

---

### PR Slice P0-S3 — Contract test scaffolds (execution/adapter/state)
**Intent:** Create test locations and basic fixtures so boundary work lands into stable homes.

#### Likely files (new)
- `tests/contracts/adapters/test_exchange_adapter_contract.py`
- `tests/contracts/state/test_order_state_contracts.py`
- `tests/contracts/state/test_account_state_contracts.py`
- `tests/contracts/test_execution_policy_contracts.py`

#### Tasks
- Add minimal skeletons/placeholders/xfail/skip if needed
- Add TODO markers linking to later phases
- Reuse existing contract fixture patterns where possible

#### Tests
- Run newly added test files (may be skipped/xfail)
- Run related existing contract collection if imports/shared fixtures touched

#### Acceptance
- Scaffolds exist without destabilizing suite
- Future slices have clear test destinations

---

## 5. Phase 1 — ExecutionPolicy extraction (first vertical slice)

### Goal
Extract shared simulation execution semantics into a pluggable policy and apply it first to:
- `BaseBackBroker`
- `CCXTSimulationBroker`

This is the first major architecture seam and should be done surgically.

---

### PR Slice P1-S1 — Introduce ExecutionPolicy scaffolding (no behavior change)
**Intent:** Add policy interface/base implementation shell and wire in a non-invasive way.

#### Likely files (new)
- `src/cracktrader/broker/execution_policy.py`
- optional split:
  - `src/cracktrader/broker/execution_policies/simulated.py`
  - `src/cracktrader/broker/execution_policies/exchange.py`

#### Likely files (existing)
- `src/cracktrader/broker/base_back_broker.py`
- `src/cracktrader/broker/ccxt_simulation_broker.py`

#### Tasks
- Define `ExecutionPolicy` concept/interface (final name may vary)
- Add hooks/methods for simulation semantics (trigger eval, expiry checks, fill completion)
- Inject policy into `BaseBackBroker` path, but keep current behavior path intact initially
- No duplicate logic removal yet

#### Tests to update/add
- Add/seed:
  - `tests/contracts/test_execution_policy_contracts.py`
- Run:
  - `tests/contracts/test_order_lifecycle.py`
  - relevant broker contract tests touching simulation behavior
  - targeted backtest integration tests

#### Acceptance
- Code compiles/tests run
- No behavior changes (or none intended)
- Policy seam exists and is ready for delegation

---

### PR Slice P1-S2 — Move BaseBackBroker simulation behavior into SimulatedExecutionPolicy
**Intent:** Extract the actual simulation semantics from `BaseBackBroker` into policy implementation.

#### Likely files
- `src/cracktrader/broker/base_back_broker.py`
- `src/cracktrader/broker/execution_policy.py`
- `src/cracktrader/broker/execution_policies/simulated.py` (if split)

#### Extraction targets (examples; verify exact names)
- trigger evaluation logic
- local fill completion logic
- expiry checks
- partial fill simulation semantics (if present)
- slippage behavior (if currently embedded)

#### Tasks
- Move logic into policy
- Have `BaseBackBroker` delegate to policy
- Preserve externally observable behavior (unless intentional changes are documented)
- Add tests around policy behavior independent of broker class where feasible

#### Tests to run
- `tests/contracts/test_order_lifecycle.py`
- `tests/contracts/test_broker_contracts.py`
- `tests/contracts/test_order_scenarios.py`
- `tests/integration/trading/test_backtest_mode_integration.py`

#### Perf check
- If backtest hot path affected materially, run perf smoke benchmark

#### Acceptance
- `BaseBackBroker` delegates simulation semantics to policy
- Duplicated logic reduced in `BaseBackBroker`
- Relevant tests green/updated
- Behavior changes documented (if any)

---

### PR Slice P1-S3 — Convert CCXTSimulationBroker to delegate to SimulatedExecutionPolicy
**Intent:** Validate the extracted policy pattern on the largest simulation broker path.

#### Likely files
- `src/cracktrader/broker/ccxt_simulation_broker.py`
- `src/cracktrader/broker/base_back_broker.py`
- policy files introduced in P1-S1/P1-S2

#### Tasks
- Remove broker-local duplicated simulation execution logic
- Delegate to shared policy
- Keep exchange-specific capability differences explicit (config/capability flags), not hidden forks

#### Tests to run
- `tests/contracts/test_broker_contracts.py`
- `tests/contracts/test_order_scenarios.py`
- `tests/contracts/test_balances.py` (if fill/accounting affected)
- `tests/integration/trading/test_backtest_mode_integration.py`
- `tests/integration/core/test_broker_store_integration.py` (if broker/store interactions changed)

#### Acceptance
- `CCXTSimulationBroker` uses shared policy
- Duplicate simulation logic materially reduced
- Contracts/integration tests pass (or are updated intentionally)
- Divergence ledger updated if any mode semantics changed

---

### PR Slice P1-S4 — Add/strengthen ExecutionPolicy contract tests
**Intent:** Lock in the new seam so rollout to other brokers is safer.

#### Likely files
- `tests/contracts/test_execution_policy_contracts.py`
- optional fixture helpers under `tests/contracts/` or `tests/helpers/`

#### Tasks
- Add cross-implementation assertions for simulation policy behavior:
  - trigger semantics
  - expiry behavior
  - partial fill semantics (if supported)
  - idempotent handling where relevant
- Structure tests so future policies (exchange/offline/test policies) can reuse same contract suite

#### Tests to run
- New policy contract tests
- Existing order lifecycle contracts (smoke)
- One representative integration flow

#### Acceptance
- New seam has contract coverage
- Future rollout risk reduced

---

## 6. Phase 2 — Roll out execution seam + live semantics cleanup

### Goal
Apply ExecutionPolicy across remaining simulation brokers and make live fallback simulation semantics explicit.

---

### PR Slice P2-S1 — PolymarketSimulationBroker delegates to shared policy
#### Likely files
- `src/cracktrader/broker/polymarket_simulation_broker.py`
- execution policy files
- optional capabilities/config modules

#### Tasks
- Delegate duplicated simulation behavior to shared policy
- Preserve market-specific semantics via explicit configuration/capability handling
- Remove copied logic where safely covered

#### Tests to run
- `tests/contracts/test_broker_contracts.py`
- `tests/contracts/test_order_scenarios.py`
- `tests/contracts/test_reliability.py`
- `tests/integration/polymarket/*` (targeted)
- backtest/simulation integration tests as applicable

#### Acceptance
- Shared policy used
- No silent behavior drift
- Tests updated/green

---

### PR Slice P2-S2 — KalshiSimulationBroker delegates to shared policy
#### Likely files
- `src/cracktrader/broker/kalshi_simulation_broker.py`
- execution policy files

#### Tasks
- Same as P2-S1
- Remove any explicitly copied fill-loop behavior once covered

#### Tests to run
- Kalshi-related contract/integration tests
- shared order lifecycle contracts

#### Acceptance
- Shared policy used
- Duplicated logic reduced
- Tests green

---

### PR Slice P2-S3 — Make live broker local simulation fallback explicit
**Intent:** Prevent hidden mode drift by making any live-mode local simulation behavior explicit and configurable.

#### Likely files
- `src/cracktrader/broker/polymarket_live_broker.py`
- `src/cracktrader/broker/kalshi_live_broker.py`
- `src/cracktrader/broker/ccxt_live_broker.py` (if affected)
- broker config/capability modules
- possibly factory/session config paths

#### Tasks
- Audit live brokers for local simulation/fallback behavior
- Replace hidden fallback with explicit mode/config path (or policy selection)
- Record intentional divergences in ledger
- Add/strengthen mode parity tests where practical

#### Tests to run
- `tests/integration/trading/test_live_mode_integration.py`
- `tests/integration/core/test_broker_store_integration.py`
- `tests/contracts/test_reliability.py`
- New/updated parity matrix tests (if introduced here)

#### Acceptance
- Live fallback semantics are explicit, not hidden
- Divergences documented
- Tests updated/green

---

### PR Slice P2-S4 — Mode parity matrix test (mock transport baseline)
**Intent:** Add an early-warning test for mode drift after execution seam rollout.

#### Likely files (new)
- `tests/integration/core/test_mode_parity_matrix.py`

#### Tasks
- Build a small parity matrix for critical scenarios (start minimal)
- Compare backtest/paper/live(mocked transport) for selected deterministic flows
- Focus on observable behavior and known intentional divergences

#### Tests to run
- New parity matrix test
- Existing backend integration/parity tests if shared harness touched:
  - `tests/integration/core/test_engine_backend_integration.py`

#### Acceptance
- Matrix test exists and catches major mode drift
- Intentional divergences are documented and exempted explicitly where needed

---

## 7. Phase 3 — AccountState read-through and reconciliation scaffolding

### Goal
Create a canonical account-state boundary **without** immediate high-risk state ownership changes.

---

### PR Slice P3-S1 — Introduce AccountState read-through wrapper (no behavior change)
#### Likely files (new)
- `src/cracktrader/engine/account_state.py` (or `src/cracktrader/state/account_state.py`)
- optional:
  - `src/cracktrader/engine/ledger_events.py` (if event abstractions introduced early)

#### Likely files (existing)
- `src/cracktrader/engine/core.py`
- `src/cracktrader/broker/universal_broker_base.py`
- `src/cracktrader/engine/results.py`
- `src/cracktrader/engine/broker_core.py` (if relevant)

#### Tasks
- Implement read-through snapshot abstraction over current core/broker state
- No ownership transfer yet
- No behavior changes
- Add initial snapshot schema/documentation

#### Tests to run
- `tests/contracts/test_balances.py`
- `tests/contracts/test_fees.py`
- `tests/contracts/test_order_scenarios.py`
- any direct tests for broker/core snapshots

#### Acceptance
- `AccountState` exists and can produce snapshots from current state
- Existing behavior preserved
- Tests green

---

### PR Slice P3-S2 — Add snapshot compare and mismatch diagnostics (warn only)
#### Likely files
- `src/cracktrader/engine/account_state.py`
- `src/cracktrader/broker/universal_broker_base.py`
- `src/cracktrader/engine/invariants.py` (if diagnostic hooks belong here)
- logging/config files as needed

#### Tasks
- Compare broker/core/account snapshots where both are available
- Emit structured mismatch diagnostics (severity/category)
- Default to warn/diagnostic, not hard fail in production paths
- Add test hooks to inspect mismatches

#### Tests to run
- accounting and order scenario contract tests
- compliance/risk integration tests if account snapshots are used:
  - `tests/integration/trading/test_compliance_risk.py` (or equivalent path)

#### Acceptance
- Mismatch compare works
- Diagnostics are categorized
- No behavior breakage unless intentional and documented

---

### PR Slice P3-S3 — Add test invariants for AccountState parity (hard fail in tests)
**Intent:** Start enforcing parity in tests before making AccountState canonical.

#### Likely files
- `tests/contracts/state/test_account_state_contracts.py`
- existing accounting/balance contract tests
- possibly test fixtures in `tests/helpers/`

#### Tasks
- Add assertions that `AccountState` snapshots match legacy snapshots in targeted scenarios
- Start with deterministic scenarios (avoid flaky live timing paths)
- Mark known mismatches explicitly (skip/xfail with linked divergence ledger entry) if needed

#### Tests to run
- new state contract tests
- `tests/contracts/test_balances.py`
- `tests/contracts/test_fees.py`

#### Acceptance
- Test-level parity checks exist
- Known mismatches are tracked, not ignored

---

## 8. Phase 4 — ExchangeAdapter contract + CCXT adapter (first adapter path)

### Goal
Introduce an adapter contract and implement a real CCXT adapter as a thin normalization/transport boundary.

---

### PR Slice P4-S1 — Define ExchangeAdapter contract and canonical event DTOs
#### Likely files (new)
- `src/cracktrader/exchanges/adapters/base.py`
- `src/cracktrader/exchanges/events.py` (canonical exchange event DTOs)
- optional capability types:
  - `src/cracktrader/exchanges/capabilities.py`

#### Tasks
- Define adapter responsibilities:
  - submit/cancel/query/stream
  - normalized order/fill/trade/balance events
  - capability exposure
- Keep signatures practical (avoid over-abstracting)
- Update contract tests scaffold to target actual interface

#### Tests to run
- `tests/contracts/adapters/test_exchange_adapter_contract.py` (now real)
- `tests/contracts/test_exchange_architecture_contracts.py` (if relevant)

#### Acceptance
- Adapter contract exists and is testable
- Canonical event DTOs are defined (or equivalent normalized event types)

---

### PR Slice P4-S2 — Implement CCXT adapter (thin wrapper)
#### Likely files (new)
- `src/cracktrader/exchanges/adapters/ccxt.py`

#### Likely files (existing)
- `src/cracktrader/store/ccxt_store.py`
- `src/cracktrader/broker/ccxt_live_broker.py`
- factory/session wiring if adapter selection introduced here

#### Tasks
- Implement CCXT adapter by wrapping current transport/store behavior
- Normalize outputs into canonical events
- Avoid rewriting all CCXT paths in one go
- Start with one vertical path (e.g., order submit/cancel + updates) before full breadth

#### Tests to run
- adapter contract tests
- `tests/integration/core/test_broker_store_integration.py`
- `tests/contracts/test_reliability.py`
- selected CCXT integration tests

#### Acceptance
- CCXT adapter passes contract tests
- At least one runtime path uses adapter-normalized outputs
- Behavior changes documented if normalization changes observable ordering/status semantics

---

### PR Slice P4-S3 — Route one broker/store path through CCXT adapter
**Intent:** Prove the adapter is not dead abstraction.

#### Likely files
- `src/cracktrader/broker/ccxt_live_broker.py`
- `src/cracktrader/store/ccxt_store.py`
- `src/cracktrader/factory.py` (if adapter selection/wiring done here)

#### Tasks
- Choose a narrow path (submit/cancel/update flow or stream update path)
- Route through adapter
- Preserve public behavior where possible or update tests/docs if changed

#### Tests to run
- adapter contract tests
- broker/store integration tests
- reliability contracts

#### Acceptance
- Adapter is used in real path, not just defined
- Tests cover the path
- Branching in broker/store begins to reduce

---

## 9. Phase 5 — Market data boundary consolidation + OHLCVSubsystem decision

### Goal
Resolve split ownership of market data normalization and OHLCV aggregation, with an explicit decision on `OHLCVSubsystem`.

---

### PR Slice P5-S1 — Investigate and document OHLCVSubsystem runtime role (decision checkpoint)
**Intent:** No silent deletion or wiring. Decide based on evidence.

#### Likely files
- `src/cracktrader/store/subsystems/ohlcv_subsystem.py`
- `src/cracktrader/store/subsystems/market_data_subsystem.py`
- `src/cracktrader/store/subsystems/ohlcv_candle_router.py`
- `src/cracktrader/feeds/native_ccxt.py`
- tests:
  - `tests/unit/store/test_ohlcv_subsystem.py`
  - `tests/unit/store/test_store_market_data.py`
  - `tests/unit/store/test_streaming_feed.py`

#### Tasks
- Confirm runtime usage (or absence) in `src/`
- Compare behavior against active OHLCV paths
- Record decision:
  - wire in / merge / retain test-only / deprecate-remove
- Update docs + comments accordingly
- Log the decision in `docs/architecture/refactor_decisions.md` (even if provisional)

#### Tests to run
- store/ohlcv unit tests
- market data integration tests (targeted)

#### Acceptance
- Decision recorded with reasoning
- No accidental behavior change unless the slice includes a decided migration

---

### PR Slice P5-S2 — Define/introduce MarketDataFeed contract (practical, not overbuilt)
#### Likely files (new)
- `src/cracktrader/store/market_data_feed.py` (or equivalent)

#### Likely files (existing)
- `src/cracktrader/store/subsystems/market_data_subsystem.py`
- `src/cracktrader/feeds/native_ccxt.py`
- `src/cracktrader/polymarket/store/ws.py` (later)
- store streaming components

#### Tasks
- Define contract for normalized subscriptions/snapshots
- Clarify sequencing/timestamp guarantees
- Add contract tests for monotonicity and malformed-data handling (minimal start)

#### Tests to run
- new market data contract tests (suggested path):
  - `tests/contracts/test_market_data_contracts.py`
- `tests/unit/store/test_store_market_data.py`
- `tests/unit/store/test_streaming_feed.py`

#### Acceptance
- MarketDataFeed contract exists
- Tests define basic guarantees

---

### PR Slice P5-S3 — Route NativeCCXT feed path through normalized market data contract
**Intent:** Remove duplicate aggregation/normalization ownership from feed driver where possible.

#### Likely files
- `src/cracktrader/feeds/native_ccxt.py`
- `src/cracktrader/store/subsystems/market_data_subsystem.py`
- `src/cracktrader/store/subsystems/ohlcv_candle_router.py`
- possible `market_data_feed.py`

#### Tasks
- Move feed driver toward consuming normalized market data instead of owning duplicate aggregation logic
- Reduce overlap incrementally
- Preserve event ordering semantics (or document/test intended changes)

#### Tests to run
- market data unit tests
- feed/store integration tests:
  - `tests/integration/core/test_feed_store_integration.py`
  - `tests/integration/streaming/test_data_quality_integration.py`
- parity/integration tests if event shape/order affects deterministic path

#### Acceptance
- Duplication reduced materially
- Event sequencing/data quality tests pass
- Any behavior/ordering changes are explicit and tested

---

## 10. Phase 6 — Strategy IO determinism contracts + adapters

### Goal
Define explicit deterministic strategy IO and adapt current strategy surfaces into it.

---

### PR Slice P6-S1 — Introduce StrategyInput / StrategyOutput contracts (no broad rewrite)
#### Likely files (new)
- `src/cracktrader/strategy/contracts.py`

#### Likely files (existing)
- `src/cracktrader/strategy/native.py`
- `src/cracktrader/engine/native_strategy.py`
- `src/cracktrader/engine/intents.py`
- `src/cracktrader/engine/runner.py`

#### Tasks
- Define deterministic strategy IO dataclasses/types
- Include deterministic keys (`run_id`, `step_id`, `event_seq` or equivalent)
- Keep old interfaces operating via adapters later

#### Tests to run
- `tests/unit/strategy/test_native_strategy_base.py`
- `tests/unit/strategy/test_strategy_defaults.py`

#### Acceptance
- Strategy IO contract exists and is documented
- No broad strategy breakage yet

---

### PR Slice P6-S2 — Add strategy adapters for current runtime path(s)
#### Likely files (new)
- `src/cracktrader/strategy/adapters/native.py`
- `src/cracktrader/strategy/adapters/compat.py` (if needed)

#### Likely files (existing)
- `src/cracktrader/engine/native_strategy.py`
- `src/cracktrader/strategy/native.py`
- `src/cracktrader/engine/runner.py`

#### Tasks
- Adapt existing callbacks/strategy actions into `StrategyInput` -> `StrategyOutput`
- Preserve behavior where possible
- Avoid strategy feature redesign in this slice

#### Tests to run
- strategy unit tests
- selected backtest integration tests
- deterministic engine integration tests if inputs/outputs changed in runner

#### Acceptance
- Current runtime path uses strategy adapter
- Behavior preserved or documented
- Tests updated/green

---

### PR Slice P6-S3 — Add strategy determinism contract test + remove one private-internals test dependency
#### Likely files
- `tests/contracts/test_strategy_determinism_contract.py` (new)
- one or more integration tests currently poking private internals
  - e.g. `tests/integration/trading/test_live_mode_integration.py` (verify exact locations)
- test helpers/harness files

#### Tasks
- Add deterministic replay/assertions for same inputs
- Convert at least one private-internals test to public harness helper path
- Record further candidates for cleanup

#### Tests to run
- new determinism contract
- affected integration test(s)

#### Acceptance
- Strategy determinism is test-enforced
- Private test internals dependence reduced in at least one touched test

---

## 11. Phase 7 — Adapter expansion (Polymarket / Kalshi)

### Goal
Finish moving exchange-specific normalization and transport branching behind adapters.

---

### PR Slice P7-S1 — Polymarket adapter implementation
#### Likely files (new)
- `src/cracktrader/exchanges/adapters/polymarket.py`

#### Likely files (existing)
- `src/cracktrader/store/polymarket_store.py`
- `src/cracktrader/polymarket/store/ws.py`
- `src/cracktrader/broker/polymarket_live_broker.py`
- factory wiring files

#### Tasks
- Wrap current Polymarket REST/WS transport and normalize outputs
- Reuse adapter contract tests
- Route one real path through adapter before broad cleanup
- Reduce scattered normalization logic gradually

#### Tests to run
- adapter contract tests
- Polymarket integration smoke tests
- reliability contracts
- broker/store integration tests touching Polymarket paths

#### Acceptance
- Polymarket adapter exists and passes contracts
- Normalization is more centralized
- No silent behavior drift

---

### PR Slice P7-S2 — Kalshi adapter implementation
#### Likely files (new)
- `src/cracktrader/exchanges/adapters/kalshi.py`

#### Likely files (existing)
- `src/cracktrader/broker/kalshi_live_broker.py`
- relevant Kalshi store/transport files (verify current paths)
- factory wiring files

#### Tasks
- Same pattern as Polymarket
- Normalize transport and update events
- Add capability exposure if needed for execution/strategy paths

#### Tests to run
- adapter contract tests
- Kalshi integration/reliability tests

#### Acceptance
- Kalshi adapter exists and passes contracts
- Exchange-specific branching reduced in broker/store layers

---

### PR Slice P7-S3 — Capability matrix cleanup + adapter test expansion
**Intent:** Consolidate capability exposure and strengthen onboarding path for future exchanges.

#### Likely files
- adapter base/contracts
- capability-related modules
- tests/contracts/adapters/*
- `tests/contracts/test_exchange_architecture_contracts.py`

#### Tasks
- Standardize capability fields used by execution/strategy/broker paths
- Expand adapter contract coverage (idempotency, reconnect replay behavior, normalization consistency)
- Document adapter onboarding checklist in dev docs (optional but useful)

#### Tests to run
- all adapter contract tests
- affected integration/reliability tests

#### Acceptance
- Capability usage more consistent
- Adapter contract suite is robust enough for future onboarding

---

## 12. Phase 8 — Rust boundary expansion (seam-first, parity-protected)

### Goal
Move additional deterministic/pure logic into Rust only after Python-side seams are stable.

### Important constraint
Do **not** expand Rust just because it’s tempting. Use the seam maturity rule:
- boundary stable,
- behavior test-covered,
- parity strategy clear.

---

### PR Slice P8-S1 — Sequencer parity extraction (Python + Rust)
#### Likely files
- `src/cracktrader/engine/sequencer.py`
- `rust/cracktrader_engine/src/*` (sequencer implementation)
- `src/cracktrader/engine/rust_bridge.py`
- backend selection/wiring files

#### Tasks
- Add Rust sequencer implementation
- Keep Python fallback
- Add parity tests / extend existing parity harness

#### Tests to run
- new/updated sequencer contract tests
- engine parity tests:
  - `tests/contracts/engine/test_python_rust_core_parity.py`
  - `tests/contracts/engine/test_python_rust_replay_parity.py`
  - `tests/contracts/engine/test_python_rust_differential_fuzz.py` (if applicable)
- integration backend parity:
  - `tests/integration/core/test_engine_backend_integration.py`

#### Acceptance
- Rust sequencer parity proven
- Python fallback preserved

---

### PR Slice P8-S2 — Invariants parity extraction
#### Likely files
- `src/cracktrader/engine/invariants.py`
- Rust core files
- rust bridge/backend wiring

#### Tasks
- Implement Rust invariant checks (or targeted subset first)
- Preserve invariant semantics
- Add parity tests / integration assertions

#### Tests to run
- parity + integration backend tests
- contract/state tests if invariant outputs are asserted there

#### Acceptance
- Invariants can run via Rust backend with parity coverage

---

### PR Slice P8-S3 — Event normalization pure logic extraction (optional / selective)
#### Likely files
- `src/cracktrader/engine/events.py`
- selected pure normalization helpers in feed paths
- Rust core files / bridge

#### Tasks
- Extract only pure, high-volume normalization helpers
- Do not pull exchange transport into Rust
- Maintain canonical event shapes/signatures

#### Tests to run
- event normalization unit tests
- parity tests where event shapes affect deterministic core
- streaming/data quality integration tests if needed

#### Acceptance
- High-ROI pure logic moved
- Parity preserved
- No transport coupling introduced into Rust

---

## 13. Cross-cutting low-risk performance slices (can be inserted into relevant phases)

These are small wins. Only do them when touching the area anyway, or as isolated tiny PRs.

### Perf Slice A — Replace list front-pops with deque in feed loading path
#### Likely file
- `src/cracktrader/feeds/native_ccxt.py`

#### Task
- Replace `pop(0)` style patterns with `collections.deque.popleft()` where hot-path queues are used

#### Tests
- feed/store integration + any affected unit tests
- perf smoke

---

### Perf Slice B — Reduce hot-path warning logging on normal broker update paths
#### Likely file
- `src/cracktrader/broker/universal_broker_base.py`

#### Task
- Audit `_on_order_update` (or equivalent hot path)
- downgrade noisy warnings to debug where warning is emitted on expected behavior
- preserve truly anomalous warnings/errors

#### Tests
- broker/store integration tests
- reliability contracts
- perf smoke (if possible)

---

### Perf Slice C — Avoid repeated parsing in simulation loops
#### Likely files
- simulation broker files
- execution policy files after extraction

#### Task
- cache/normalize parsed order/execution types at creation time or once-per-order

#### Tests
- order lifecycle/contracts
- simulation integration tests
- perf smoke

---

### Perf Slice D — Reduce lock churn in streaming dispatcher loops
#### Likely file
- `src/cracktrader/store/streaming_feed.py`

#### Task
- consolidate repeated queue state checks into local snapshots where safe
- preserve thread-safety semantics

#### Tests
- streaming unit tests
- data quality integration tests
- reliability tests if relevant

---

### Perf Slice E — Replace repeated trade-list sorting with running OHLCV aggregates
#### Likely file
- `src/cracktrader/store/subsystems/ohlcv_candle_router.py`

#### Task
- use running aggregate state for finalized windows where possible
- preserve ordering semantics for in-window updates/finalization

#### Tests
- OHLCV subsystem/router tests
- market data quality integration tests
- perf smoke

---

## 14. Test architecture improvement suggestions (optional, later)

You asked for A+B now, but here are good “C” style improvements to keep on the radar.

### 14.1 Integration fixture decomposition
Current large integration fixture modules can be split into:
- **environment/session lifecycle**
- **exchange sandbox/test resources**
- **scenario builders**
- **assertion helpers**

This reduces hidden coupling and makes mode parity tests easier to write.

### 14.2 Public harness helpers for integration tests
Create helper APIs so tests stop poking internals (`feed._enqueue`, `store._connection_lost`, etc.) directly.

Suggested directions:
- `tests/helpers/harness/market_events.py`
- `tests/helpers/harness/order_updates.py`
- `tests/helpers/harness/mode_parity.py`

### 14.3 Dedicated parity test package
Consider a dedicated `tests/parity/` package for:
- mode parity
- backend parity
- replay parity scenarios

This does not need to replace existing locations immediately.

### 14.4 Property-based test rollout plan (later)
When ready:
1. add PBT dependency (e.g., Hypothesis)
2. start with order transition legality
3. add accounting conservation
4. add sequencer monotonicity

Keep first PBTs small and deterministic to avoid CI pain.

---

## 15. Slice completion criteria (hard gate before moving to next slice)

A slice is complete when all are true:

- [ ] Target seam/change is implemented
- [ ] Tests for touched behavior are updated/added
- [ ] Targeted tests pass
- [ ] Broader tests run if shared code changed materially
- [ ] Perf smoke run if hot path touched
- [ ] Divergence ledger updated if mode behavior affected
- [ ] Refactor decision log updated for non-trivial choices made in the slice
- [ ] Duplicate code removed or explicitly left with TODO + reason
- [ ] Commit history is clean and incremental

Do not move on with a half-wired abstraction and “we’ll fix tests later.”

---

## 16. Phase checkpoint criteria (before advancing phases)

### After Phase 1 checkpoint
- `ExecutionPolicy` seam is real and used by `BaseBackBroker` + `CCXTSimulationBroker`
- contract tests exist for policy seam
- no major regressions in simulation paths

### After Phase 2 checkpoint
- all simulation brokers use shared execution semantics (or documented variants)
- live fallback simulation behavior is explicit
- mode parity matrix test exists

### After Phase 3 checkpoint
- `AccountState` read-through exists
- mismatch diagnostics exist
- test-level snapshot parity checks exist for core scenarios

### After Phase 4 checkpoint
- adapter contract exists and is enforced
- CCXT adapter is real and used in at least one path

### After Phase 5 checkpoint
- `OHLCVSubsystem` decision is documented
- market data ownership is clearer and duplication reduced

### After Phase 6 checkpoint
- strategy IO determinism contract exists
- adapters cover current path
- determinism test exists

### After Phase 7 checkpoint
- Polymarket/Kalshi adapters implemented (or blockers documented)
- adapter contract suite covers active adapters

### After Phase 8 checkpoint
- at least one additional Rust deterministic component migrated with parity
- Python fallback preserved

---

## 17. Final note to Codex (operational philosophy)

Do not optimize for “most elegant end-state patch.”

Optimize for:
1. extracting one seam cleanly,
2. proving it with tests,
3. reducing duplication/ambiguity,
4. moving to the next seam.

This repo already has strong tests and parity discipline. Use that as leverage, not friction.

When in doubt: **small slice, explicit behavior, tested result, commit** — and keep receipts in `docs/architecture/refactor_decisions.md`.
