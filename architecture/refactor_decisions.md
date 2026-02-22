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

## 2026-02-22 - [Phase 5 / P5-S3] Route NativeCCXTFeedDriver through MarketDataFeed contract when available
- **Status:** decided
- **Context:** `NativeCCXTFeedDriver` still directly used legacy store OHLCV registration/start/stop methods, bypassing the new `market_data_feed` contract.
- **Decision:** Prefer `store.market_data_feed` (`register_ohlcv_callback`, `start_ohlcv`, `stop_ohlcv`) when available, with fallback to legacy store methods for compatibility.
- **Alternatives considered:**
  - Keep feed driver on legacy methods until complete market-data refactor.
  - Break compatibility and require `market_data_feed` unconditionally.
- **Why this choice:** Moves one active feed path onto the new contract without breaking existing stores.
- **Impact radius:** `src/cracktrader/feeds/native_ccxt.py`, `tests/unit/feeds/test_native_ccxt_market_data_feed.py`.
- **Follow-ups:** Apply the same routing pattern to other feed drivers that consume OHLCV streams.

## 2026-02-22 - [Bugfix] Drop out-of-order ticks in NativeCCXTFeedDriver queue path
- **Status:** decided
- **Context:** Integration test `test_out_of_order_tick_handling` showed out-of-order ticks were still queued and loaded (`3` instead of expected `2`).
- **Decision:** Enable monotonic timestamp enforcement in `NativeCCXTFeedDriver` queue runtime (`enforce_monotonic_ts=True`) and add unit regression coverage.
- **Alternatives considered:**
  - Add bespoke ordering logic in `drain_queue`.
  - Enable reordering by default instead of dropping late ticks.
- **Why this choice:** Uses existing queue-core semantics with the least code and aligns with current expected contract behavior.
- **Impact radius:** `src/cracktrader/feeds/native_ccxt.py`, `tests/unit/feeds/test_native_ccxt_market_data_feed.py`, integration behavior in `tests/integration/core/test_feed_store_integration.py`.
- **Follow-ups:** If future requirements need late-tick acceptance, gate it behind explicit config (e.g., reorder buffer policy).

## 2026-02-22 - [Phase 5] Harden OHLCV callback handling for malformed payloads
- **Status:** decided
- **Context:** Market data contract tests showed malformed OHLCV rows could raise in `OHLCVCandleRouter` and break callback flow.
- **Decision:** Catch parse errors in OHLCV router callback path and skip malformed candles, preserving stream continuity.
- **Alternatives considered:**
  - Let exceptions propagate and fail fast.
  - Add validation earlier in streaming feed layer only.
- **Why this choice:** Keeps live stream processing resilient and matches contract-level expectation that malformed candles are ignored.
- **Impact radius:** `src/cracktrader/store/subsystems/ohlcv_candle_router.py`, `tests/contracts/test_market_data_contracts.py`.
- **Follow-ups:** Add metrics/counters for dropped malformed candles if observability is needed.

## 2026-02-22 - [Phase 6 / P6-S1] Introduce deterministic StrategyInput and StrategyOutput contracts
- **Status:** decided
- **Context:** Strategy runtime paths used callback signatures directly (`events, core -> intents`) without explicit deterministic IO envelopes or stable keys.
- **Decision:** Add `StrategyInput` and `StrategyOutput` dataclasses in `src/cracktrader/strategy/contracts.py` with required deterministic keys (`run_id`, `step_id`) and event-sequence identity (`event_seq`), plus immutable metadata/diagnostics normalization.
- **Alternatives considered:**
  - Delay contracts until adapter wiring (`P6-S2`) and define fields implicitly.
  - Add contracts with mutable dict/list payloads and defer immutability concerns.
- **Why this choice:** Matches Phase 6 slice ordering (contracts first, adapters later) and enforces deterministic envelope shape early without broad runtime rewrites.
- **Impact radius:** `src/cracktrader/strategy/contracts.py`, `src/cracktrader/strategy/__init__.py`, `tests/unit/strategy/test_strategy_contracts.py`.
- **Follow-ups:** Wire native runtime callback paths through adapter(s) that translate existing strategy callbacks to `StrategyInput -> StrategyOutput` in `P6-S2`.

## 2026-02-22 - [Phase 6 / P6-S2] Route native strategy runtime through StrategyInput/StrategyOutput adapter
- **Status:** decided
- **Context:** `NativeStrategyCallbackAdapter` still invoked `on_bar(events, core)` directly and returned raw intents without passing through the new deterministic IO envelope.
- **Decision:** Add `NativeStrategyContractAdapter` in `src/cracktrader/strategy/adapters/native.py`, and update `engine/native_strategy.py` to build `StrategyInput` per step and convert adapter `StrategyOutput` back to runner callback intents.
- **Alternatives considered:**
  - Keep direct callback invocation until a broader runner signature change.
  - Replace runtime callback signatures immediately with `StrategyInput -> StrategyOutput` (broad rewrite).
- **Why this choice:** Moves an active runtime path onto the contract seam now while preserving existing strategy object APIs and avoiding a high-blast-radius runner rewrite.
- **Impact radius:** `src/cracktrader/strategy/adapters/native.py`, `src/cracktrader/strategy/adapters/__init__.py`, `src/cracktrader/engine/native_strategy.py`, `tests/unit/strategy/test_native_strategy_adapter.py`, `tests/unit/engine/test_native_runtime.py`.
- **Follow-ups:** Expand contract adapters for additional runtime paths and callback forms in later slices.

## 2026-02-22 - [Phase 6 / P6-S3] Add strategy determinism contract and reduce one private feed test dependency
- **Status:** decided
- **Context:** Phase 6 required explicit determinism contract coverage and at least one step away from private-internals dependence in touched tests.
- **Decision:** Add `tests/contracts/test_strategy_determinism_contract.py` to assert identical strategy output for identical `StrategyInput` on the native strategy adapter path. Add public `NativeCCXTFeedDriver.ingest_ohlcv(...)` and migrate selected tests to that public API (while keeping reconnection edge-case internals path unchanged to preserve behavior).
- **Alternatives considered:**
  - Defer determinism contract to engine parity phase.
  - Keep all feed tests on `_enqueue` only.
- **Why this choice:** Enforces Phase 6 determinism intent now with low overhead and reduces one private test dependency without broad feed runtime redesign.
- **Impact radius:** `tests/contracts/test_strategy_determinism_contract.py`, `src/cracktrader/feeds/native_ccxt.py`, `tests/unit/feeds/test_native_ccxt_market_data_feed.py`, `tests/integration/trading/test_live_mode_integration.py`.
- **Follow-ups:** Continue migrating additional test paths off private feed/store internals as public harness methods become available.

## 2026-02-22 - [Phase 7 / P7-S1] Add Polymarket adapter and route live submit/cancel through adapter seam
- **Status:** decided
- **Context:** Adapter contract rollout covered CCXT paths, while Polymarket live broker still called store transport directly without adapter normalization seam.
- **Decision:** Implement `PolymarketExchangeAdapter`, add adapter contract tests, export adapter in exchange adapter package, and route `PolymarketLiveBroker` submit/cancel dispatch and submission normalization through `exchange_adapter`.
- **Alternatives considered:**
  - Delay Polymarket adapter until Kalshi adapter lands.
  - Add adapter type branches inside shared broker base before concrete adapter implementation.
- **Why this choice:** Delivers one real Polymarket path through adapter seam now with low migration risk and immediately reuses existing adapter contract pattern.
- **Impact radius:** `src/cracktrader/exchanges/adapters/polymarket.py`, `src/cracktrader/exchanges/adapters/__init__.py`, `src/cracktrader/broker/polymarket_live_broker.py`, `tests/contracts/adapters/test_exchange_adapter_contract.py`, `tests/unit/broker/test_prediction_live_broker_submission.py`.
- **Follow-ups:** Extend Polymarket adapter consumption to order-stream/balance normalization and apply same pattern to Kalshi (`P7-S2`).

## 2026-02-22 - [Phase 7 / P7-S2] Add Kalshi adapter and route live submit/cancel through adapter seam
- **Status:** decided
- **Context:** Kalshi live broker still used direct store transport paths and lacked a dedicated exchange adapter implementation aligned with the adapter contract suite.
- **Decision:** Implement `KalshiExchangeAdapter`, export it, extend adapter contract tests, and route `KalshiLiveBroker` submit/cancel and submission normalization through `exchange_adapter`.
- **Alternatives considered:**
  - Keep Kalshi on store-direct paths while only Polymarket uses adapter seams.
  - Create a shared prediction adapter first and delay concrete Kalshi implementation.
- **Why this choice:** Keeps Polymarket/Kalshi adapter rollout symmetric and enforces one contract-tested seam per exchange before broader capability cleanup.
- **Impact radius:** `src/cracktrader/exchanges/adapters/kalshi.py`, `src/cracktrader/exchanges/adapters/__init__.py`, `src/cracktrader/broker/kalshi_live_broker.py`, `tests/contracts/adapters/test_exchange_adapter_contract.py`, `tests/unit/broker/test_prediction_live_broker_submission.py`.
- **Follow-ups:** Expand both prediction adapters to balance/order-stream normalization coverage and consolidate capability matrix in `P7-S3`.

## 2026-02-22 - [Phase 8 / P8-S1] Add sequencer backend bridge and Rust sequencer surface
- **Status:** decided
- **Context:** Phase 8 required sequencer boundary expansion, but backend bridging previously covered only deterministic core (`build_core`), with no Rust sequencer selection path.
- **Decision:** Add `build_sequencer` and `_RustSequencerAdapter` in `engine/rust_bridge.py`, wire `EngineRunner` with optional `sequencer_backend` selection while preserving explicit `sequencer=` override, add `RustEventSequencer` in Rust extension bindings, and add Python/Rust sequencer parity contract tests (auto-skip when Rust extension unavailable).
- **Alternatives considered:**
  - Keep runner on Python sequencer only and defer all backend wiring.
  - Add bridge helper only without Rust extension class surface.
- **Why this choice:** Establishes the seam-first backend wiring now, preserves Python fallback behavior, and creates parity enforcement hooks for environments where Rust extension is enabled.
- **Impact radius:** `src/cracktrader/engine/rust_bridge.py`, `src/cracktrader/engine/runner.py`, `src/cracktrader/engine/__init__.py`, `rust/cracktrader_engine/src/python.rs`, `rust/cracktrader_engine/src/lib.rs`, `tests/unit/engine/test_rust_bridge_smoke.py`, `tests/unit/engine/test_runner.py`, `tests/contracts/engine/test_python_rust_sequencer_parity.py`.
- **Follow-ups:** Run full Rust parity suites in an environment with compiled `cracktrader_rust` extension and continue Phase 8 with invariant extraction (`P8-S2`).

## 2026-02-22 - [Phase 8 / P8-S2] Add invariant backend bridge and Rust invariant checker surface
- **Status:** decided
- **Context:** Invariant enforcement remained Python-only (`InvariantChecker`) while Phase 8 required backend parity expansion beyond the core/sequencer path.
- **Decision:** Add `build_invariant_checker` and `_RustInvariantCheckerAdapter` in `engine/rust_bridge.py`, wire `EngineRunner` with optional `invariant_backend` selection while preserving explicit `invariant_checker=` override, add `RustInvariantChecker` in Rust extension bindings, and add Python/Rust invariant parity contract coverage (auto-skip without Rust extension).
- **Alternatives considered:**
  - Keep invariant checks Python-only until all Rust parity infra is available.
  - Add bridge helper only without Rust extension class surface.
- **Why this choice:** Keeps seam-first backend wiring consistent with P8-S1 and creates a concrete invariant boundary for parity testing while preserving current Python fallback and strict-mode behavior.
- **Impact radius:** `src/cracktrader/engine/rust_bridge.py`, `src/cracktrader/engine/runner.py`, `src/cracktrader/engine/__init__.py`, `rust/cracktrader_engine/src/python.rs`, `rust/cracktrader_engine/src/lib.rs`, `tests/unit/engine/test_rust_bridge_smoke.py`, `tests/unit/engine/test_runner.py`, `tests/contracts/engine/test_python_rust_invariants_parity.py`.
- **Follow-ups:** In Rust-enabled CI, run full invariant parity matrix and tune/align any rule-order or context-shape differences before promoting Rust invariant backend outside contracts.

## 2026-02-22 - [Phase 8 / P8-S3] Add event normalization backend bridge and Rust normalizer surface
- **Status:** decided
- **Context:** Event normalization helpers (`normalize_bar_event`, `normalize_order_event`, etc.) were Python-only and Phase 8 called for selective extraction of pure normalization logic.
- **Decision:** Add `build_event_normalizer` in `engine/rust_bridge.py` with Python and Rust adapter implementations, expose `RustEventNormalizer` in Rust bindings, and add Python/Rust normalization parity contracts for order/bar normalization paths (auto-skip without Rust extension).
- **Alternatives considered:**
  - Keep normalization helpers Python-only and defer extraction.
  - Move normalization logic directly into runtime call sites without bridge seam.
- **Why this choice:** Provides a minimal pure-logic extraction seam with no transport coupling and preserves canonical `EngineEvent` shape while enabling parity validation.
- **Impact radius:** `src/cracktrader/engine/rust_bridge.py`, `src/cracktrader/engine/__init__.py`, `rust/cracktrader_engine/src/python.rs`, `rust/cracktrader_engine/src/lib.rs`, `tests/unit/engine/test_rust_bridge_smoke.py`, `tests/contracts/engine/test_python_rust_event_normalization_parity.py`.
- **Follow-ups:** Expand parity coverage to fill/timer normalization and run full Rust-enabled parity suite in CI.

## 2026-02-22 - [Phase 7 / P7-S3] Standardize adapter capability matrix and expand contract coverage
- **Status:** decided
- **Context:** Exchange adapters exposed similar but ad-hoc capability dicts, and normalization contracts did not enforce cross-adapter capability consistency or idempotent normalization behavior.
- **Decision:** Introduce shared capability matrix constants/helpers in `adapters/base.py`, route CCXT/Polymarket/Kalshi adapters through `default_adapter_capabilities()`, normalize order status consistently using `normalize_order_status_text(...)`, and add contract tests for capability key parity plus normalization idempotency.
- **Alternatives considered:**
  - Keep per-adapter capability dicts and only document preferred keys.
  - Delay capability matrix standardization until additional exchanges are onboarded.
- **Why this choice:** Delivers the Phase 7 S3 capability-cleanup objective with minimal runtime risk and stronger contract enforcement for future adapter onboarding.
- **Impact radius:** `src/cracktrader/exchanges/adapters/base.py`, `src/cracktrader/exchanges/adapters/ccxt.py`, `src/cracktrader/exchanges/adapters/polymarket.py`, `src/cracktrader/exchanges/adapters/kalshi.py`, `tests/contracts/adapters/test_exchange_adapter_contract.py`, `docs/architecture/adapter_onboarding_checklist.md`.
- **Follow-ups:** Expand adapter contracts for reconnect replay behavior when explicit replay semantics are implemented in live transport layers.

## 2026-02-22 - [Phase 8 / P8-S3 follow-up] Expand event normalization parity contracts to fill/timer paths
- **Status:** decided
- **Context:** Initial P8-S3 parity coverage validated order/bar normalization only; fill/timer normalization remained unasserted in Python/Rust parity contracts.
- **Decision:** Extend `test_python_rust_event_normalization_parity.py` with fill and timer parity assertions using deterministic timestamps.
- **Alternatives considered:**
  - Leave fill/timer parity untested until Rust-enabled CI run.
  - Add parity through integration tests only.
- **Why this choice:** Keeps parity guarantees close to the normalization seam and increases confidence before Rust-enabled execution environments run the full contract matrix.
- **Impact radius:** `tests/contracts/engine/test_python_rust_event_normalization_parity.py`.
- **Follow-ups:** Run parity contracts in Rust-enabled environment and address any runtime binding deltas if surfaced.

## 2026-02-22 - [Perf Infra] Repair benchmark runner paths and native engine benchmark harness
- **Status:** decided
- **Context:** Performance smoke entrypoints were failing from two sources: deprecated benchmark harness calls (`CracktraderEngine.adddata/addstrategy`) and automation script references to missing runner files plus stripped subprocess environment on Windows.
- **Decision:** Rework engine backend benchmark harness to run through native runtime (`NativeEngineRuntime` + `EngineRunner(build_core(...))`) and update automation fallback logic to use pytest benchmark mode when sync runner scripts are absent, while preserving process environment variables in subprocess calls.
- **Alternatives considered:**
  - Restore removed legacy runtime API just for benchmarks.
  - Keep automation script fixed to one platform-specific runner path.
- **Why this choice:** Keeps benchmark tooling aligned with current native-only runtime architecture and restores cross-platform execution without reintroducing legacy API surface.
- **Impact radius:** `src/cracktrader/engine/benchmark.py`, `performance/automation/run_benchmarks.py`, `tests/unit/engine/test_benchmark_runtime_api.py`, `tests/unit/engine/test_run_benchmarks_automation.py`.
- **Follow-ups:** Execute full Rust parity/perf gate in Rust-enabled environment and tune baseline thresholds as needed.

## 2026-02-22 - [Parity Gate] Fix Rust invariant binding type mismatch and run full gate
- **Status:** decided
- **Context:** Rust backend install failed during parity-gate setup due to a PyO3 type mismatch in `RustInvariantChecker` transition parsing (`PyObject` vs `Bound<PyAny>`), blocking Rust-enabled contract and benchmark validation.
- **Decision:** Bind transition `PyObject` values to `Bound<PyAny>` before calling transition helper readers, then rerun `scripts/install_rust_backend.py` and full `scripts/run_rust_parity_gate.py`.
- **Alternatives considered:**
  - Disable invariant transition checks in Rust checker temporarily.
  - Skip Rust gate and defer fix to a later pass.
- **Why this choice:** Smallest targeted fix that restores build correctness and unlocks full parity/perf verification immediately.
- **Impact radius:** `rust/cracktrader_engine/src/python.rs`; verification commands `scripts/install_rust_backend.py` and `scripts/run_rust_parity_gate.py`.
- **Follow-ups:** Optional cleanup for remaining Rust compile warnings (`RustEngineError` unused, intent fields currently unused).
