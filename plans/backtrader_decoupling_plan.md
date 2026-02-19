# Backtrader Decoupling Plan

Date: 2026-02-18  
Status: Draft implementation plan  
Scope: Remove Backtrader as a core runtime dependency while preserving current Cracktrader behavior, fee correctness, and user-facing workflows.

## Progress Snapshot (2026-02-19)

- Phase 0 (Stabilize Baseline and Gates): Done
- Phase 1 (Engine-Native Domain Types): Done
- Phase 2 (Fee/Commission Decoupling): In Progress
- Phase 3 (Broker Core Decoupling): In Progress
- Phase 4 (Feed Core Decoupling): In Progress
- Phase 5 (Strategy Runtime Decoupling): Not Started
- Phase 6 (Indicators/Analyzers Decoupling): Not Started
- Phase 7 (Cerebro Compatibility Isolation): In Progress
- Phase 8 (Native-First Tests): Not Started
- Phase 9 (Packaging and Deprecation): Not Started

Latest local validation snapshot (2026-02-19):
- `tests/unit`: `1863 passed, 88 skipped`
- `tests/contracts`: `38 passed`
- fee contracts: `tests/contracts/test_fees.py` -> `3 passed`

Latest feed-core slice (2026-02-19):
- Extracted engine-native candle payload normalization helper (`FeedCandleCore`).
- Added native feed-port primitives (`FeedPort`, `FeedCursor`, `InMemoryFeedPort`, `InMemoryFeedCursor`) as groundwork for BT `DataBase` replacement.
- Added `BacktraderFeedCursor` and rewired sync adapter ingest batching to read BT bars through feed-port cursor boundary.
- Added `NativeFeedAdapter` to ingest directly from native `FeedPort` batches without Backtrader feed classes.
- Added `NativeEngineRuntime` as a native-first entrypoint that runs `EngineRunner` from `FeedPort` via `NativeFeedAdapter`.
- Added `CracktraderEngine.run_native(...)` and a basic native feed-port example to establish a concrete user-facing non-`bt.DataBase` run path.

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
- Make Backtrader optional.

Work:
- Move Backtrader from core dependencies to optional extra.
- Add clear runtime errors when BT compatibility APIs are invoked without extra installed.
- Publish migration notes with examples for native APIs.

Deliverables:
- Packaging update.
- Migration guide and release notes.

Exit criteria:
- `pip install cracktrader` works without Backtrader.
- `pip install cracktrader[backtrader]` enables compatibility layer.

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
