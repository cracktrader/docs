# Cracktrader Refactor Decisions Log

This file is the append-only decision log for the architecture refactor program.
Use it to record non-trivial design decisions, intentional behavior changes, deferred cleanup, and escalation points.

## Entry Template

```md
## YYYY-MM-DD - [Phase/Slice] Short decision title
- **Status:** decided | provisional | needs human decision
- **Context:** what was being changed and why
- **Decision:** what was chosen
- **Alternatives considered:**
  - option 1
  - option 2
- **Why this choice:** tie to architecture spec, tests, and constraints
- **Impact radius:** files, tests, behavior/modes affected
- **Follow-ups:** deferred cleanup, TODOs, validation still needed
```

## 2026-02-22 - [Phase 0 / P0-S0] Start with baseline hardening artifacts
- **Status:** decided
- **Context:** Begin the refactor program with low-risk scaffolding before changing execution, state, or adapter behavior.
- **Decision:** Implement Phase 0 bootstrap artifacts first: decision log, mode divergence ledger, test/perf workflow doc, and contract test scaffolds.
- **Alternatives considered:**
  - Start directly with `ExecutionPolicy` extraction.
  - Add only docs and delay test scaffolds.
- **Why this choice:** Matches the architecture spec and execution playbook ordering, and creates guardrails that reduce risk for Phase 1 code moves.
- **Impact radius:** `docs/architecture/*`, `tests/contracts/adapters/*`, `tests/contracts/state/*`, `tests/contracts/test_execution_policy_contracts.py`, and links in plan docs.
- **Follow-ups:** Move to Phase 1 (`ExecutionPolicy` seam) with test-first slices.

## 2026-02-22 - [Phase 1 / P1-S1] Introduce execution policy seam via delegation
- **Status:** decided
- **Context:** `BaseBackBroker` and `CCXTSimulationBroker` each owned their pending-order simulation loop directly, with duplicated structure and no shared seam.
- **Decision:** Add `ExecutionPolicy` and `LegacySimulationExecutionPolicy`, attach a default policy to both brokers, and delegate `_process_pending_orders` through the policy to broker-local legacy implementations.
- **Alternatives considered:**
  - Full extraction of trigger/fill logic into policy in one slice.
  - Keep placeholders only and defer all wiring.
- **Why this choice:** Establishes the seam with minimal behavior risk and satisfies Phase 1 scaffolding goals before moving semantics in later slices.
- **Impact radius:** `src/cracktrader/broker/execution_policy.py`, `src/cracktrader/broker/base_back_broker.py`, `src/cracktrader/broker/ccxt_simulation_broker.py`, and `tests/contracts/test_execution_policy_contracts.py`.
- **Follow-ups:** Move trigger evaluation, expiry handling, and local-fill completion from broker classes into policy implementation(s) in the next Phase 1 slices.

## 2026-02-22 - [Phase 1 / P1-S2] Extract BaseBackBroker pending-order loop into shared policy
- **Status:** decided
- **Context:** `BaseBackBroker` still contained the simulation pending-order loop; only delegation seam existed.
- **Decision:** Introduce `SimulatedExecutionPolicy` with shared loop semantics and make it the default policy for `BaseBackBroker`.
- **Alternatives considered:**
  - Keep all loop logic broker-local until both brokers are migrated together.
  - Extract all trigger/fill/slippage methods in a single large patch.
- **Why this choice:** Removes a meaningful duplicate behavior chunk while keeping broker-specific trigger/fill/accounting logic untouched, minimizing behavior drift risk.
- **Impact radius:** `src/cracktrader/broker/execution_policy.py`, `src/cracktrader/broker/base_back_broker.py`, `tests/contracts/test_execution_policy_contracts.py`.
- **Follow-ups:** Migrate `CCXTSimulationBroker` to `SimulatedExecutionPolicy` with explicit hooks for expiry, cheat-on-close, and slippage behavior.

## 2026-02-22 - [Phase 1 / P1-S3] Migrate CCXTSimulationBroker to shared simulation policy
- **Status:** decided
- **Context:** `CCXTSimulationBroker` still used a legacy policy bridge to broker-local pending-order loop logic.
- **Decision:** Make `SimulatedExecutionPolicy` the default for `CCXTSimulationBroker` and preserve broker-specific expiry cancellation behavior through `_execution_policy_pre_process_order`.
- **Alternatives considered:**
  - Keep `LegacySimulationExecutionPolicy` for CCXT until a larger extraction pass.
  - Move expiry handling entirely into policy with broker-type branching.
- **Why this choice:** Keeps the shared loop centralized now while preserving CCXT-specific semantics via explicit broker hook points, avoiding policy-level type branches.
- **Impact radius:** `src/cracktrader/broker/ccxt_simulation_broker.py`, `src/cracktrader/broker/execution_policy.py`, `tests/contracts/test_execution_policy_contracts.py`.
- **Follow-ups:** Continue Phase 1 by extracting trigger/fill/price helpers into policy-facing contracts only where duplication can be removed without behavior drift.
