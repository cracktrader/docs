# Backtrader Decoupling Plan

Date: 2026-02-18  
Status: Active implementation  
Scope: Remove Backtrader as a core runtime dependency with hard cuts (no Cerebro compatibility preservation).

## Progress Snapshot (2026-02-19)

- Phase 0 (Stabilize Baseline and Gates): Done
- Phase 1 (Engine-Native Domain Types): Done
- Phase 2 (Fee/Commission Decoupling): In Progress
- Phase 3 (Broker Core Decoupling): In Progress
- Phase 4 (Feed Core Decoupling): Done
- Phase 5 (Strategy Runtime Decoupling): Not Started
- Phase 6 (Indicators/Analyzers Decoupling): Not Started
- Phase 7 (Cerebro Compatibility Isolation): Done (hard removal)
- Phase 8 (Native-First Tests): Not Started
- Phase 9 (Packaging and Deprecation): Not Started

Latest hard-cut updates (2026-02-20):
- Top-level `cracktrader` API no longer exports `Cerebro` / `AsyncCerebro`.
- `CracktraderEngine` is now native-only for execution entrypoints:
  - `run()` and `run_async()` hard-fail with guidance to use native runtime methods.
  - `run_native(...)` and `run_native_ohlcv_store(...)` remain as the supported paths.
- `experiments/runner.py` no longer depends on `ct.Cerebro`:
  - parameter-space expansion is implemented locally,
  - runtime uses `bt.Cerebro` directly where compatibility runtime is still needed.
- Shared test helpers were updated to remove top-level `AsyncCerebro` imports.
- Compatibility runtime modules removed:
  - deleted `src/cracktrader/cerebro.py`
  - deleted `src/cracktrader/async_cerebro.py`
  - `src/cracktrader/engine_runtime.py` now contains only native `CracktraderEngine` logic.
- Removed compatibility-only test surface tied to `CracktraderCerebro` / `AsyncCerebro`:
  - deleted `tests/unit/cerebro/*`
  - deleted `tests/unit/engine/test_cerebro_engine_runtime.py`
  - deleted `tests/contracts/engine/test_sync_async_parity.py`
  - deleted `tests/integration/compatibility/test_cerebro_compatibility.py`
  - deleted async verification scripts under `tests/verification/verify_async_*.py` and `verify_native_async.py`.
- Feed-core decoupling follow-up:
  - added `src/cracktrader/feeds/contracts.py` protocol layer (`SyncFeedLike`, `FeedRef`) to reduce direct `bt.DataBase`/`AbstractDataBase` type coupling at runtime boundaries.
  - migrated `src/cracktrader/feeds/adapter.py` and `src/cracktrader/strategy/pegged_order_manager.py` to use feed protocols instead of BT concrete feed typing.
  - cleaned stale runtime docstrings/comments referencing removed `AsyncCerebro`/`CracktraderCerebro`.
  - extracted queue/sanitization behavior into native `src/cracktrader/feeds/core.py` (`FeedQueueCore`) and delegated enqueue paths from:
    - `src/cracktrader/feeds/base.py`
    - `src/cracktrader/feeds/ccxt.py`
  - extracted candle parsing/state into native `FeedBarStateCore` (also in `src/cracktrader/feeds/core.py`) and delegated `_set_candle(...)` parsing paths from:
    - `src/cracktrader/feeds/base.py`
    - `src/cracktrader/feeds/ccxt.py`
    - `src/cracktrader/feeds/async_base.py`
  - extracted Backtrader line assignment into native `FeedLineWriterCore` (in `src/cracktrader/feeds/core.py`) and delegated line write paths from:
    - `src/cracktrader/feeds/base.py`
    - `src/cracktrader/feeds/ccxt.py`
    - `src/cracktrader/feeds/async_base.py`
  - extracted load/lifecycle state into native `FeedLoadStateCore` (in `src/cracktrader/feeds/core.py`) and delegated `CCXTDataFeed._load()` historical drain + replay-cache control flow.
  - introduced composed `FeedRuntimeCore` (in `src/cracktrader/feeds/core.py`) that unifies queue, bar parsing, line writing, and load-state cores; delegated runtime flow from:
    - `src/cracktrader/feeds/base.py`
    - `src/cracktrader/feeds/ccxt.py`
  - moved additional `_load()` decision helpers into `FeedRuntimeCore` (historical step status, queue pop helper, and to-date cutoff check) and simplified `CCXTDataFeed._load()` control flow around those helpers.
  - moved reorder/idle helper logic into `FeedRuntimeCore` (`should_flush_reorder_buffer`, `sort_reorder_buffer`, `build_idle_candle`) and updated `CCXTDataFeed` to delegate these decisions.
  - moved live-branch control helpers into `FeedRuntimeCore` (`should_start_live_watch_after_history`, `queue_empty_should_wait`, `live_poll_sleep_seconds`) and updated `CCXTDataFeed._load()` to delegate these branch decisions.
  - added `FeedRuntimeCore.load_step(...)` orchestrator for historical/replay/queue/todate paths and delegated `CCXTDataFeed._load()` main flow to this core method.
  - added `FeedRuntimeCore.queue_empty_step(...)` orchestrator for queue-empty fallback (idle injection, reorder flush/retry, live poll sleep) and delegated this branch from `CCXTDataFeed._load()`.
  - migrated `CTAsyncFeedBase` enqueue/candle mapping to `FeedRuntimeCore` (`sanitize`, `parse_bar`, `apply_bar`) so async feed base shares the same native core boundary as sync feeds.
  - introduced `FeedDriver` protocol in `src/cracktrader/feeds/contracts.py` and migrated engine feed-port typing (`BacktraderFeedCursor` / `BacktraderIngestFeedPort`) to this interface to further reduce BT-shaped type assumptions at ingestion boundaries.
  - added non-BT `NativeFeedDriver` (`src/cracktrader/feeds/native_driver.py`) and validated ingestion through `BacktraderIngestFeedPort` using protocol-based `set_index` hooks (no `bt.DataBase` inheritance required).
  - introduced generic feed-driver ingest aliases (`FeedDriverCursor`, `FeedDriverIngestPort`) in `src/cracktrader/engine/feed_port.py` so native runtime entrypoints can avoid BT-specific naming and assumptions.
  - removed BT-named ingest types from engine feed-port surface (`BacktraderFeedCursor` / `BacktraderIngestFeedPort`); engine ingestion now uses `FeedDriverCursor` / `FeedDriverIngestPort` only.
  - added `CracktraderEngine.run_native_feed_driver(...)` to run native engine flow directly from one or more protocol-compatible feed drivers.
  - added `NativeCCXTFeedDriver` (`src/cracktrader/feeds/native_ccxt.py`) and `CracktraderEngine.run_native_ccxt_feed(...)` as native-first CCXT feed runtime path without `bt.DataBase` inheritance.
  - hard-switched default CCXT feed surface to native driver:
    - `ct.Feed(exchange=<ccxt>, ...)` now returns `NativeCCXTFeedDriver`
    - `ct.CCXTDataFeed` now aliases `NativeCCXTFeedDriver`
    - `cracktrader.exchanges.ccxt.feed.CCXTDataFeed` now aliases `NativeCCXTFeedDriver`
    - `src/cracktrader/feeds/ccxt.py` is now a native-first shim with no Backtrader import.
  - migrated `src/cracktrader/feeds/kalshi.py` to a native feed-driver scaffold (`KalshiDataFeed` now extends `NativeFeedDriver`, no `CTFeedBase` / `bt.DataBase` dependency).
  - migrated `src/cracktrader/feeds/custom.py` to native feed implementation (no `CTFeedBase` / Backtrader inheritance).
  - replaced `src/cracktrader/feeds/async_base.py` and `src/cracktrader/feeds/async_ccxt.py` with native async feed implementations (no Backtrader imports).
  - replaced `src/cracktrader/feeds/base.py` with native queue/line base (`CTFeedBase` no longer inherits `bt.DataBase`).
  - removed Backtrader imports from Polymarket feed stack:
    - `src/cracktrader/polymarket/feed/base.py`
    - `src/cracktrader/polymarket/feed/outcomes.py`
  - final feed-area scan now returns no `backtrader` imports under:
    - `src/cracktrader/feeds/*`
    - `src/cracktrader/polymarket/feed/*`
  - broker/runtime decoupling follow-up:
    - introduced native position model `src/cracktrader/engine/position.py` (`Position` with `update`/`clone` semantics).
    - migrated `src/cracktrader/broker/position_tracker.py` to use native `Position` instead of `bt.Position`.
    - migrated `src/cracktrader/broker/universal_broker_base.py` position fallback/update paths to use native `Position` factories (`BrokerPositionCore` boundaries no longer require `bt.Position`).
    - exported native `Position` from `src/cracktrader/engine/__init__.py`.
    - introduced native broker scaffold `src/cracktrader/broker/native_base.py` to replace required `bt.BrokerBase` lifecycle/params/comminfo behavior.
    - removed `bt.BrokerBase` inheritance from:
      - `src/cracktrader/broker/universal_broker_base.py`
      - `src/cracktrader/broker/composite_broker.py`
    - broker-area `bt.BrokerBase` inheritance usage is now eliminated in `src/cracktrader/broker/*` (remaining BT coupling is order classes/constants and strategy/runtime compatibility edges).
    - introduced native order scaffold `src/cracktrader/broker/base_order.py` (`BaseOrder`) with:
      - order lifecycle methods (`submit`, `accept`, `reject`, `cancel`, `completed`, `alive`)
      - execution ledger (`executed` + `exbits`) compatible with existing broker/order tests.
    - migrated `src/cracktrader/broker/ccxt_order.py` from `bt.Order` inheritance to native `BaseOrder` inheritance.
    - removed direct Backtrader imports from `src/cracktrader/broker/ccxt_order.py` and preserved compatibility status/type constants via engine-domain mappers.
    - removed direct `bt.Order` constant branching/imports from broker runtime modules:
      - `src/cracktrader/broker/async_broker.py`
      - `src/cracktrader/broker/ccxt_live_broker.py`
      - `src/cracktrader/broker/ccxt_simulation_broker.py`
      - `src/cracktrader/broker/base_back_broker.py`
      - `src/cracktrader/broker/kalshi_simulation_broker.py`
      - `src/cracktrader/broker/polymarket_simulation_broker.py`
    - added enforcement test to prevent regressions:
      - `tests/unit/broker/test_broker_no_bt_order_constants.py`
    - removed remaining `bt.Order.Market` default-order fallback usage from broker order factories:
      - `src/cracktrader/broker/universal_broker_base.py`
      - `src/cracktrader/broker/ccxt_broker_base.py`

Latest local validation snapshot (2026-02-20):
- `tests/unit`: `1888 passed, 88 skipped`
- `tests/contracts`: `38 passed`
- fee contracts: `tests/contracts/test_fees.py` -> `3 passed`

Latest feed-core slice (2026-02-19):
- Extracted engine-native candle payload normalization helper (`FeedCandleCore`).
- Added native feed-port primitives (`FeedPort`, `FeedCursor`, `InMemoryFeedPort`, `InMemoryFeedCursor`) as groundwork for BT `DataBase` replacement.
- Added `BacktraderFeedCursor` and rewired sync adapter ingest batching to read BT bars through feed-port cursor boundary.
- Added `NativeFeedAdapter` to ingest directly from native `FeedPort` batches without Backtrader feed classes.
- Added `NativeEngineRuntime` as a native-first entrypoint that runs `EngineRunner` from `FeedPort` via `NativeFeedAdapter`.
- Added `CracktraderEngine.run_native(...)` and a basic native feed-port example to establish a concrete user-facing non-`bt.DataBase` run path.
- Added native strategy protocol support for `run_native(...)` (`strategy` object with `on_start`/`on_bar`/`on_stop`) while keeping callback mode.
- Added `StoreQueueFeedCursor` for native consumption of store/stream queue data (including stop sentinel handling and monotonic timestamp filtering).
- Added `PollingFeedPort` to keep non-exhausted cursors active across empty polls and support queue-backed feed paths in native runtime.
- Added `StreamingSubsystemFeedCursor` for direct consumption of `StreamingFeedSubsystem` (`get_data`/`is_active`) in native runtime loops.
- Added `CracktraderEngine.run_native_ohlcv_store(...)` to run native engine flow directly from store OHLCV stream subsystem via polling feed port.
- Added idle-poll controls on native feed ingest (`idle_sleep_s`, `max_idle_polls`) so polling ports can tolerate short empty windows before terminating.
- Added non-running-loop fallback in `run_native_ohlcv_store(...)` to consume direct store OHLCV queues (`_ohlcv_queue`) when stream workers are unavailable.
- Added session-backed example `examples/basics/native_ohlcv_store_runtime.py` to exercise native runtime from a real session/store object.
- Added `BacktraderIngestFeedPort` and routed `BacktraderSyncAdapter.ingest_generator` through `NativeFeedAdapter`, reducing direct feed-loop orchestration inside BT compatibility adapter.
- Updated `CerebroEngineRuntime` non-vectorized sync path to delegate through `NativeEngineRuntime` with `BacktraderIngestFeedPort` instead of constructing/running `EngineRunner` directly.
- Added native broker object protocol support for `run_native(...)` (`broker` object with `on_start`/`on_step`/`on_stop`) while preserving callback mode.
- Added native observer object protocol support for `run_native(...)` (`observer` object with `on_start`/`on_step`/`on_stop`) while preserving callback observer mode.

## Status Audit (2026-02-20)

### Branch and Validation Status

- Active decoupling branch: `feature/feed-core-decoupling`.
- Branch divergence vs `origin/main`: `0 behind`, `116 ahead`.
- Worktree state: clean.
- Latest local gates:
  - `tests/unit`: `1867 passed, 88 skipped`
  - `tests/contracts/test_fees.py`: `3 passed`

### What Is Completed So Far

- Feed-core native foundation exists:
  - Native candle normalization core.
  - Native feed-port and cursor interfaces.
  - Backtrader feed cursor boundary used by sync adapter ingest.
  - Native feed adapter and native engine runtime entrypoint.
  - Public `CracktraderEngine.run_native(...)` path.
  - Native strategy object protocol (`on_start`/`on_bar`/`on_stop`).
- Broker-core has been substantially extracted into engine-native modules with compatibility wrappers retained.
- Fee and broker regression gates are stable on current branch.

### Remaining Program-Level Work

- Phase 2 (fees) and Phase 3 (broker): complete remaining exit-criteria hardening and native-first parity closure.
- Phase 4 (feeds): finish migrating production feed behavior (queueing/reordering/live-vs-historical boundaries/timeframe plumbing) into native feed core with BT kept as an adapter.
- Feed decoupling status:
  - complete for feed modules and feed runtime boundaries.
  - remaining Backtrader coupling is outside feed core (broker/runtime compatibility layers).
- Phase 5 (strategy runtime): continue lifting runtime callbacks/notifications to native contracts with BT strategy bridge as compatibility path.
- Phase 6 (indicators/analyzers): introduce native interfaces and remove BT types from core runtime paths.
- Phase 7 (Cerebro isolation): reduce `ct.Cerebro` to compatibility facade over native orchestrator paths.
- Phase 8 (native-first tests): migrate contracts/fixtures to assert native domain behavior first; retain a smaller BT compatibility suite.
- Phase 9 (packaging/deprecation): remove Backtrader from core install path and publish migration notes for native APIs.

### Current Priority Queue

1. Complete next Phase 4 slice with a native store/queue-backed feed cursor path.
2. Push that path through one real runtime flow (not only in-memory demo feeds).
3. Then continue Phase 7 by increasing delegation from Cerebro compatibility runtime to native runtime/adapters.

## Prerequisites (Must Be Green Before Starting)

- [x] Examples baseline is validated and reproducible (validated 2026-02-18):
  - [x] Offline examples pass (minimum gate):
    - `examples/basics/hello_engine.py`
    - `examples/basics/session_quickstart.py`
    - `examples/basics/shared_store.py`
    - `examples/prediction/kalshi_prediction_paper.py`
  - [x] Previously unstable examples in the gate matrix are now green locally (network and advanced gates passed on 2026-02-18).
  - [x] Networked examples have a runbook and expected outcomes (see `docs/plans/backtrader_gate_runbook.md`).
- [x] Test baseline is stable and documented (validated 2026-02-18):
  - [x] `tests/unit` green on main (`1663 passed, 88 skipped`).
  - [x] Relevant `tests/contracts` green on main (`38 passed`).
  - [x] No flaky tests in fee/order-lifecycle areas for at least 3 repeated local runs (`12 passed` x 3).
- [x] Behavior parity scope is frozen:
  - [x] We define which Backtrader semantics must be preserved 1:1 (order status transitions, fill accounting, commissions, timeframe handling, strategy hooks).
  - [x] We define what is explicitly out of scope (full Backtrader feature completeness not currently used by Cracktrader).
- [x] Fee correctness baseline is locked:
  - [x] Existing fee tests for spot/prediction are green.
  - [x] Contract-level fee assertions include partial fills and mixed maker/taker paths.
- [x] Migration policy is agreed:
  - [x] Strangler pattern only (new engine-native core + BT compatibility adapters).
  - [x] No breaking public API changes without compatibility shims during migration window.

## Why This Plan Exists

Cracktrader currently has broad Backtrader coupling across runtime, broker/feed/strategy abstractions, tests, and examples. We need to decouple in a way that:

1. Preserves execution correctness and accounting invariants.
2. Preserves existing user workflows during migration.
3. Produces an architecture suitable for a future Rust core and API-driven GUI.

## Current Coupling Inventory (Snapshot)

### Counts

- `src`: 36 files reference Backtrader.
- `tests`: 91 files reference Backtrader.
- `examples`: 8 files directly reference Backtrader.

### Source hotspots by area

- `broker`: 15 files
- `feeds`: 6 files
- `indicators`: 4 files
- `strategy`: 2 files
- `comm_info`: 2 files
- `polymarket/feed`: 2 files
- runtime wrappers: `cerebro.py`, `engine_runtime.py`

### Hard inheritance anchors (highest coupling)

- `src/cracktrader/cerebro.py` -> `bt.Cerebro`
- `src/cracktrader/broker/universal_broker_base.py` -> `bt.BrokerBase`
- `src/cracktrader/feeds/base.py` -> `bt.DataBase`
- `src/cracktrader/strategy/base.py` -> `bt.Strategy`
- `src/cracktrader/comm_info/base.py` -> `bt.CommInfoBase`
- `src/cracktrader/indicators/base.py` -> `bt.Indicator`

### Semantic coupling hotspots

- BT order constants are heavily used (`Market`, `Limit`, `Stop`, `StopLimit`, `Submitted`, `Accepted`, `Partial`, `Completed`, etc.).
- BT timeframe constants are widely used (`Seconds`, `Minutes`, `Days`, `Ticks`).
- BT date conversions are embedded (`date2num`, `num2date`).

## Decoupling Principles

1. Engine-native first: domain semantics (orders, fills, bars, fees) must live in Cracktrader types.
2. Adapters at the boundary: Backtrader remains a compatibility layer, not the core.
3. Test-first migration: each step adds/updates parity tests before moving implementation.
4. Keep user APIs stable during migration (`ct.exchange`, `ct.Feed`, `ct.Broker`, `ct.Cerebro` compatibility).
5. Preserve accounting invariants at every step (cash, position, commission, order lifecycle).

## Explicit Parity Scope

This plan preserves parity for features Cracktrader currently relies on:

- Order lifecycle: created -> submitted -> accepted -> partial -> completed/canceled/rejected/expired.
- Order types: market, limit, stop, stop-limit, bracket/OCO behavior currently covered by tests.
- Fee/commission handling: maker/taker, global commission override, prediction-fee paths, partial fills.
- Cash/position accounting and invariant checks.
- Feed semantics: timeframe mapping, bar ordering, live/historical transitions.
- Strategy execution hooks (`start`, `next`, order/trade notifications).
- Existing session/factory routing behavior.

Out of scope for this migration:

- Full Backtrader feature parity beyond Cracktrader usage.
- Recreating all BT plotting internals as part of core decoupling.

## Recommended Order of Decoupling

The order below minimizes regression risk in fees and execution semantics.

### Phase 0: Stabilize Baseline and Gates

Objective:
- Make the current system measurable before architectural changes.

Work:
- Fix broken examples or explicitly quarantine them.
- Add an examples run matrix (offline/networked/live) with expected output signatures.
- Add a "migration gate" test target for order/fee invariants.

Deliverables:
- Example runbook doc.
- Passing baseline check command set.

Exit criteria:
- Baseline tests and agreed examples are green and reproducible.

### Phase 1: Introduce Engine-Native Domain Types

Objective:
- Stop using BT constants as core domain primitives.

Work:
- Add Cracktrader-native enums/types:
  - `OrderType`, `OrderStatus`, `TimeframeUnit`
  - `Bar`, `FillReport`, `OrderReport`
- Add mapping adapters:
  - `bt -> ct` and `ct -> bt` for statuses/types/timeframes.

Deliverables:
- New domain model module(s).
- Compatibility mappers with full unit coverage.

Exit criteria:
- Core engine, broker, and tests can assert against native enums without BT constants.

### Phase 2: Decouple Fee/Commission Engine from BT CommInfo

Objective:
- Make fee logic fully engine-native first (highest correctness risk area).

Work:
- Introduce a Cracktrader fee policy interface independent of `bt.CommInfoBase`.
- Port maker/taker and prediction fee logic into native fee policies.
- Keep BT `CommInfo` wrappers as adapters that delegate to native policies.
- Add strict parity tests:
  - full fill + partial fill + cancel after partial
  - buy/sell symmetry
  - maker/taker override behavior
  - prediction fee cache/load behavior

Deliverables:
- Native fee policy module.
- Adapter layer for existing BT pathways.
- Expanded fee contracts.

Exit criteria:
- Fee and cash deltas are identical before/after migration for parity fixtures.

### Phase 3: Decouple Broker Core from BT BrokerBase

Objective:
- Move order state machine and accounting into engine-native broker core.

Work:
- Extract broker core into BT-independent service:
  - order book/state registry
  - transitions
  - cash/position updates
  - notification events
- Implement BT compatibility broker wrapper(s) around native core.
- Migrate exchange-specific brokers to target native core APIs.

Deliverables:
- Native broker core package.
- BT wrapper preserving old external behavior.

Exit criteria:
- Existing broker lifecycle tests pass via wrapper, and new native broker tests pass directly.

### Phase 4: Decouple Feed Core from BT DataBase

Objective:
- Move feed ingestion/buffering/timeframe normalization into native feed core.

Work:
- Introduce native feed interfaces/events.
- Port queueing, candle sanitation, timeframe conversion, live/historical boundary logic.
- Keep BT DataBase adapters for compatibility.

Deliverables:
- Native feed core + BT adapter feeds.

Exit criteria:
- Feed data-flow tests pass for both native and BT adapter paths.

### Phase 5: Decouple Strategy Runtime Interface

Objective:
- Allow strategy execution without inheriting from `bt.Strategy`.

Work:
- Define engine-native strategy protocol (hooks + intent emission).
- Keep BT strategy adapter for existing user strategies.
- Migrate runtime to prefer native strategy protocol.

Deliverables:
- Native strategy base/protocol.
- BT strategy bridge.

Exit criteria:
- Engine executes both native and BT-adapted strategies with parity for order callbacks and lifecycle hooks.

### Phase 6: Decouple Indicators/Analyzers from BT Runtime Contracts

Objective:
- Prevent BT indicator/analyzer types from being required by core runtime.

Work:
- Introduce indicator/analyzer interfaces in Cracktrader runtime.
- Migrate built-in critical indicators/analyzers used by tests/examples.
- Keep BT indicator wrappers for compatibility.

Deliverables:
- Native indicator/analyzer interfaces and initial implementations.

Exit criteria:
- Engine runtime no longer imports BT indicator types in core execution path.

### Phase 7: Isolate Cerebro Compatibility Layer

Objective:
- Make `ct.Cerebro` a compatibility facade, not the core orchestrator.

Work:
- Create native orchestrator entrypoint as first-class runtime API.
- Rewire `CracktraderCerebro` to delegate via compatibility adapters only.
- Mark BT-bound APIs as compatibility-layer in docs.

Deliverables:
- Native orchestration entrypoint docs and stable API.
- `ct.Cerebro` compatibility implementation.

Exit criteria:
- Core can run without importing Backtrader internals except when compatibility path is chosen.

### Phase 8: Migrate Tests to Native-First Contracts

Objective:
- Stop validating core behavior through BT-only semantics.

Work:
- Move contract tests to native domain assertions.
- Keep a smaller BT compatibility suite (smoke + critical lifecycle + fee parity).
- Update fixtures to avoid hard BT dependency where not needed.

Deliverables:
- Native-first contracts.
- Reduced BT compatibility test matrix.

Exit criteria:
- Majority of unit/contracts run without Backtrader.

### Phase 9: Packaging and Deprecation

Objective:
- Remove Backtrader from the core runtime install.

Work:
- Remove Backtrader from core dependencies once remaining BT-bound modules are deleted/replaced.
- Hard-fail any legacy BT compatibility API entrypoints with migration guidance until they are removed.
- Publish migration notes with examples for native APIs.

Deliverables:
- Packaging update.
- Migration guide and release notes.

Exit criteria:
- `pip install cracktrader` works and runs native runtime flows without Backtrader installed.
- No supported runtime path requires Backtrader.

## Validation Strategy Per Phase

Each phase must include:

1. New/updated unit tests for the migrated component.
2. Contract tests for order lifecycle and fee invariants.
3. Example smoke tests (offline minimum set).
4. No regression in:
   - order transitions
   - commission totals
   - cash/value accounting
   - feed ordering/time monotonicity

## Example Validation Matrix (Use As Gate)

### Offline gate (run every phase)

- `python examples/basics/hello_engine.py`
- `python examples/basics/session_quickstart.py`
- `python examples/basics/shared_store.py`
- `python examples/prediction/kalshi_prediction_paper.py`

### Networked gate (run before merge windows)

- `python examples/backtesting/ccxt_backtest_baseline.py`
- `python examples/live/ccxt_live_paper.py`
- `python examples/backtesting/timeframes_and_ticks.py`
- `python examples/prediction/polymarket_feed_demo.py`

### Advanced/compatibility gate (milestone-based)

- `python examples/multi_market/multi_exchange_two_feeds.py`
- `python examples/orders/order_types.py`
- `python examples/multi_market/mixed_ccxt_polymarket.py`
- `python examples/prediction/polymarket_closed_outcome.py`

## Risk Register

1. Fee regressions during adapter transition.
   - Mitigation: phase 2 first, strict delta assertions.
2. Hidden BT assumptions in broker/order objects.
   - Mitigation: dual-path tests (native + BT wrapper) until cutover.
3. Test brittleness due to fixture BT dependencies.
   - Mitigation: create native fixture layer before broad migration.
4. Example drift (docs claim runnable but scripts break).
   - Mitigation: treat examples as gated artifacts with CI smoke lane.

## Work Packaging and Commit Strategy

- Small vertical slices only:
  - tests-first
  - one coherent change per commit
  - green checks before each commit
- Suggested commit scopes:
  - `engine-domain`
  - `fees`
  - `broker-core`
  - `feed-core`
  - `strategy-runtime`
  - `bt-compat`
  - `tests-contracts`
  - `examples`

## Definition of Done (Program Level)

Done means all are true:

- Core runtime can execute without Backtrader installed.
- Backtrader compatibility layer remains functional behind optional extra.
- Fee and accounting parity contracts pass.
- Required examples pass in their respective gates.
- Migration docs clearly show native-first APIs and compatibility paths.
