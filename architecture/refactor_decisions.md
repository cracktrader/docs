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

## 2026-02-22 - [Phase 1 / P1-S4] Strengthen execution policy contracts and complete extraction
- **Status:** decided
- **Context:** Pending-order looping was shared, but trigger/limit/stop/fill-price semantics remained duplicated across `BaseBackBroker` and `CCXTSimulationBroker`.
- **Decision:** Move those semantics into `SimulatedExecutionPolicy` (`evaluate_order_trigger`, `limit_crossed`, `stop_triggered`, `determine_fill_price`) and keep broker-specific differences in explicit hooks (market timing and expiry pre-processing).
- **Alternatives considered:**
  - Leave trigger/price duplication in broker classes until Phase 2.
  - Move all broker-specific behavior into policy with broker-type branching.
- **Why this choice:** Completes Phase 1 extraction for the two target brokers while preserving behavior and avoiding brittle type-conditional logic inside policy code.
- **Impact radius:** `src/cracktrader/broker/execution_policy.py`, `src/cracktrader/broker/base_back_broker.py`, `src/cracktrader/broker/ccxt_simulation_broker.py`, `tests/contracts/test_execution_policy_contracts.py`.
- **Follow-ups:** Phase 2 rollout for remaining simulation brokers (`polymarket`, `kalshi`) using the same hook model.

## 2026-02-22 - [Phase 1] Perf smoke checkpoint and blockers
- **Status:** provisional
- **Context:** Phase 1 touched broker execution hot paths, so perf smoke was required.
- **Decision:** Attempted existing benchmark entrypoints and recorded failures due to pre-existing tooling issues.
- **Alternatives considered:**
  - Skip perf smoke entirely.
  - Repair benchmark tooling inside this slice.
- **Why this choice:** Kept Phase 1 scope focused on execution seam extraction while still documenting benchmark command outcomes.
- **Impact radius:** benchmark command paths only (`scripts/benchmark_engine_backends.py`, `performance/automation/run_benchmarks.py`), no production behavior changes.
- **Follow-ups:** Fix benchmark runner paths/engine API mismatch in a dedicated perf-infra slice, then re-run smoke baseline for Phase 1-delivered broker changes.

## 2026-02-22 - [Phase 2 / P2-S1] Migrate Polymarket and Kalshi simulation brokers to shared policy
- **Status:** decided
- **Context:** `PolymarketSimulationBroker` and `KalshiSimulationBroker` still had broker-local pending-order loops and trigger logic, causing mode drift risk and duplicated semantics.
- **Decision:** Set `SimulatedExecutionPolicy` as default for both brokers and delegate pending-order and trigger/limit checks to policy methods, while preserving broker-specific behavior through explicit hooks.
- **Alternatives considered:**
  - Leave prediction brokers on broker-local loops until a larger prediction-market rewrite.
  - Move all prediction-specific behavior into policy with broker-type branching.
- **Why this choice:** Aligns simulation broker semantics under one policy seam while keeping market-specific differences explicit (`_execution_policy_supports_order_type`, `_execution_policy_market_trigger`, and pre-process expiry hook).
- **Impact radius:** `src/cracktrader/broker/polymarket_simulation_broker.py`, `src/cracktrader/broker/kalshi_simulation_broker.py`, `src/cracktrader/broker/execution_policy.py`, `tests/contracts/test_execution_policy_contracts.py`.
- **Follow-ups:** Phase 2 live-mode fallback cleanup and explicit config controls for any simulated behavior in live brokers.

## 2026-02-22 - [Phase 2 / P2-S3] Make live local simulation fallback explicit
- **Status:** decided
- **Context:** `PolymarketLiveBroker` and `KalshiLiveBroker` could silently simulate local fills/cancels when exchange async transport was missing.
- **Decision:** Add `allow_live_simulation_fallback` control and default fallback behavior to disabled in `live` mode (enabled only when explicit, or by default in `sandbox` mode).
- **Alternatives considered:**
  - Preserve implicit fallback behavior in all modes.
  - Remove fallback entirely and hard-fail whenever transport is unavailable.
- **Why this choice:** Keeps a safe testing/sandbox path while preventing accidental simulated execution in live mode.
- **Impact radius:** `src/cracktrader/broker/polymarket_live_broker.py`, `src/cracktrader/broker/kalshi_live_broker.py`, `tests/unit/broker/test_prediction_live_broker_submission.py`.
- **Follow-ups:** Add mode-parity coverage for live-vs-sandbox fallback expectations in integration matrix tests.

## 2026-02-22 - [Phase 3 / P3-S1] Introduce AccountState read-through wrapper and mismatch diagnostics
- **Status:** decided
- **Context:** Account balances were tracked directly in broker internals without a canonical reconciliation-aware wrapper.
- **Decision:** Add `AccountState` with normalized read-through snapshots and mode-aware mismatch diagnostics, then wire it into `UniversalBrokerBase._on_balance_update` without changing balance mutation authority.
- **Alternatives considered:**
  - Delay account state work until after adapter rollout.
  - Make `AccountState` write-authoritative immediately.
- **Why this choice:** Matches Phase 3 Stage A/B goals: create a coherent read-through view and comparison signals first, without destabilizing existing accounting behavior.
- **Impact radius:** `src/cracktrader/state/account_state.py`, `src/cracktrader/state/__init__.py`, `src/cracktrader/broker/universal_broker_base.py`, `tests/contracts/state/test_account_state_contracts.py`.
- **Follow-ups:** Add mirror-write mode behind config and promote mismatch policies toward stricter enforcement in tests.

## 2026-02-22 - [Phase 3 / P3-S2] Add mirror-write mode and test hard-fail mismatch policy
- **Status:** decided
- **Context:** Phase 3 needed Stage B/C behavior: optional mirror writes and strict mismatch enforcement in tests.
- **Decision:** Add broker-level `account_state_mirror_write` and `account_state_mismatch_policy` controls, update local balance mutation paths (`setcash`, `_set_cash`, `_adjust_balances_on_fill`) to mirror through `AccountState` when enabled, and raise `AssertionError` on mismatches when policy is `raise`.
- **Alternatives considered:**
  - Keep compare-only behavior and delay mirror writes to a later phase.
  - Always enforce hard-fail mismatches in all modes.
- **Why this choice:** Delivers staged migration semantics without forcing production behavior changes, while enabling strict invariant enforcement in tests.
- **Impact radius:** `src/cracktrader/state/account_state.py`, `src/cracktrader/broker/universal_broker_base.py`, `tests/contracts/state/test_account_state_contracts.py`.
- **Follow-ups:** Evaluate promotion of mirror-write mode to default in selected modes once adapter/state parity coverage is broader.

## 2026-02-22 - [Phase 4 / P4-S1,P4-S2] Define adapter contract and add CCXT adapter path
- **Status:** decided
- **Context:** Exchange transport and normalization were broker-local, and adapter contract scaffolds were still placeholders.
- **Decision:** Introduce `ExchangeAdapter` protocol, canonical exchange event DTOs, and a thin `CCXTExchangeAdapter`; then route CCXT live submit/cancel transport and submission event normalization through the adapter.
- **Alternatives considered:**
  - Keep broker-local transport logic until all adapters are implemented.
  - Build a wide adapter abstraction with all exchanges in one pass.
- **Why this choice:** Delivers a real adapter seam with minimal blast radius and proves one runtime path consumes normalized adapter outputs.
- **Impact radius:** `src/cracktrader/exchanges/events.py`, `src/cracktrader/exchanges/adapters/base.py`, `src/cracktrader/exchanges/adapters/ccxt.py`, `src/cracktrader/exchanges/adapters/__init__.py`, `src/cracktrader/broker/ccxt_live_broker.py`, `tests/contracts/adapters/test_exchange_adapter_contract.py`, `tests/unit/broker/test_ccxt_live_broker_submission.py`.
- **Follow-ups:** Expand adapter usage beyond submission/cancel path and add adapter implementations for Polymarket/Kalshi.

## 2026-02-22 - [Phase 4 / P4-S3] Route CCXT live balance updates through adapter normalization
- **Status:** decided
- **Context:** Submit/cancel and order submission callbacks were adapter-backed, but balance updates still applied raw exchange payloads directly in `CCXTLiveBroker`.
- **Decision:** Normalize balance updates via `exchange_adapter.normalize_balance_event` before applying broker balance snapshots, and add contract assertion that adapter normalization is invoked when adapter exists.
- **Alternatives considered:**
  - Keep raw balance payload handling in broker until full adapter rollout.
  - Move all balance update handling into adapter callbacks in one step.
- **Why this choice:** Extends adapter consumption to another high-value live path with low risk and clear test coverage.
- **Impact radius:** `src/cracktrader/broker/ccxt_live_broker.py`, `tests/unit/broker/test_ccxt_live_broker_submission.py`, `tests/contracts/test_balances.py`.
- **Follow-ups:** Add normalized order-stream update path via adapter (not just submission callback) in subsequent slices.

## 2026-02-22 - [Phase 4 / P4-S3] Route CCXT live order-stream updates through adapter normalization
- **Status:** decided
- **Context:** CCXT live order submission callback used adapter normalization, but streaming order updates still entered broker lifecycle as raw payloads.
- **Decision:** Add `CCXTLiveBroker._on_order_update` override that normalizes via `exchange_adapter.normalize_order_event` and forwards normalized payload to `UniversalBrokerBase._on_order_update`.
- **Alternatives considered:**
  - Keep raw order-stream payload handling until all exchange adapters exist.
  - Rewrite full broker update pipeline around adapter event objects now.
- **Why this choice:** Completes adapter normalization on the active CCXT live order-update path with minimal code movement and direct test coverage.
- **Impact radius:** `src/cracktrader/broker/ccxt_live_broker.py`, `tests/unit/broker/test_ccxt_live_broker_submission.py`.
- **Follow-ups:** Reuse the same pattern when wiring Polymarket/Kalshi adapters in later phases.

## 2026-02-22 - [Phase 5 / P5-S1] OHLCVSubsystem runtime-role decision
- **Status:** decided
- **Context:** Market-data ownership consolidation required an explicit decision on whether `OHLCVSubsystem` is active runtime behavior or legacy overlap.
- **Decision:** Retain `OHLCVSubsystem` as legacy/test utility for now; active runtime path remains `BaseStore -> MarketDataSubsystem -> OHLCVCandleRouter`.
- **Alternatives considered:**
  - Wire `OHLCVSubsystem` into runtime immediately.
  - Remove `OHLCVSubsystem` now.
- **Why this choice:** Evidence shows runtime already uses `OHLCVCandleRouter`; immediate wiring/removal would create avoidable risk before explicit consolidation work.
- **Impact radius:** `docs/architecture/ohlcv_subsystem_decision.md`, `src/cracktrader/store/subsystems/ohlcv_subsystem.py`, `tests/unit/store/test_ohlcv_subsystem_role.py`.
- **Follow-ups:** During Phase 5 consolidation, merge useful behavior or deprecate/remove subsystem with replacement tests.

## 2026-02-22 - [Phase 5 / P5-S2] Introduce MarketDataFeed contract wrapper
- **Status:** decided
- **Context:** Market data APIs were available through `BaseStore` methods, but there was no explicit feed contract object for normalized OHLCV interactions.
- **Decision:** Add `MarketDataFeed` protocol and `StoreMarketDataFeed` wrapper, expose it as `BaseStore.market_data_feed`, and add contract tests asserting contract surface and behavior.
- **Alternatives considered:**
  - Keep only direct `BaseStore` methods.
  - Introduce a larger event/stream abstraction in one pass.
- **Why this choice:** Provides an explicit seam for Phase 5 consolidation with minimal runtime disruption and preserves existing store method behavior.
- **Impact radius:** `src/cracktrader/store/market_data_feed.py`, `src/cracktrader/store/base_store.py`, `tests/contracts/test_market_data_contracts.py`.
- **Follow-ups:** Extend contract coverage to sequencing/monotonicity guarantees and malformed-data handling.

## 2026-02-22 - [Phase 5 / P5-S2] Add OHLCV callback monotonicity contract coverage
- **Status:** decided
- **Context:** MarketDataFeed contract existed, but sequencing/ordering behavior was not yet asserted in contract tests.
- **Decision:** Add contract test asserting OHLCV callback delivery order is monotonic for ordered input ticks, using the active OHLCV router callback path.
- **Alternatives considered:**
  - Defer monotonicity assertions to integration-only tests.
  - Introduce a broader event-sequencer abstraction first.
- **Why this choice:** Adds immediate, low-cost coverage for one key market-data contract guarantee while staying aligned with current runtime architecture.
- **Impact radius:** `tests/contracts/test_market_data_contracts.py`.
- **Follow-ups:** Add malformed-data and out-of-order tick contract assertions with explicit expected handling policy.
