# Polymarket + CCXT Parity Roadmap

## Context
- Keep factory/sessions API frozen while converging Polymarket behaviour toward the CCXT experience.
- Reduce bespoke code in Polymarket store/feed/broker by pushing neutral logic into shared bases.
- Align tests/docs so polymarket runs through the same generic coverage unless behaviour truly diverges.

## Guiding Principles
## Status Update (2025-10-20)
- **Done:** BaseStore loop helpers plus balance/order watcher hooks, order-event normalisation, shared mock-store hierarchy, deterministic OHLCV fixtures, simplex plotting guarded, Polymarket history shims (fetch prices/tokens) restored, broker unit suites now share a mode-aware data helper, and StrategyBase centralises order/trade logging for strategies/examples.
- **In Progress:** Parametrised broker/feed suites and shared fixture rollout; final verification of CCXT feed logging across selection suites.
- **Open Issues:** Intermittent hangs on long pytest runs; shared feed/broker extractions still pending broader refactor.

- Minimize churn for existing CCXT users; no breaking changes to factories, sessions, or public broker/feed signatures.
- Prefer additive hooks or mixins when extracting shared behaviour.
- Keep prediction-specific logic isolated and well named.
- Tests drive refactors: parity assertions should prove equivalence before code is moved.

## Architectural Convergence Plan

### 1. Store Layer
- **Goal:** Common lifecycle, loop, and event plumbing in `BaseStore`; thin CCXT/Polymarket overrides.
- Extract shared helpers:
  - Event-loop bootstrap/teardown (background thread, shutdown guards).
  - `_normalize_orderbook_symbol` default plus symbol resolution helper contracts.
  - Default `start_watching_{balance,orders,trades,ohlcv}` that manage pollers/queues.
  - Order-event helper (`_emit_order_event`) to encapsulate manager payload formatting.
- Define hook points Polymarket can override:
  - Symbol resolution / metadata lookups.
  - Watch coroutine factories.
  - Minimum size / tick sizing and fees.
- Remove duplicated balance seeding/order-manager wrappers from `polymarket_store.py` once helpers live in base.
- Ensure CCXT store adopts the same helpers (even if they no-op) to guarantee API symmetry.

### 2. Feed Layer
- **Goal:** Move timeframe/tick arbitration, queue helpers, and data naming into `CTFeedBase`.
- Shared features to lift:
  - `set_name`, `_enqueue`, `_set_candle`, `_load_queue`, `granularity` vs `timeframe` conflict resolution.
  - Context/profile plumbing already present; confirm CCXT feed uses base queue methods after extraction.
- Keep Polymarket-specific pieces (multi-outcome metrics, event summaries) in subclasses but document required overrides.
- Add fixtures for new hook behaviours (e.g., trade-to-bar aggregator) with tests to prevent regressions.

### 3. Broker Layer
- **Goal:** Provide exchange-neutral hooks so polymarket broker logic plugs into base order lifecycle.
- Add template methods in `BaseCCXTBroker`:
  - `_submit_exchange_order(order, *, amount, price, params)`
  - `_cancel_exchange_order(order)`
  - `_resolve_execution_price(order)` to consolidate offline simulation logic.
- Move notification/position updates (`_simulate_offline_fill`, `_on_order_update` glue) into base.
- Share prediction instrument treatment by centralising instrument constants and validation rules.
- Revisit `PositionTracker` dependency injection—allow broker to request tracker via registry rather than hard-coded dict.

### 4. Sessions & Factory
- Ensure session helpers automatically pick up new shared store/broker behaviour.
- Consider exposing `store_class`/`broker_class` hints through the registry to support future exchanges without modifying factory internals.
- Verify registry API remains stable; document any new optional keyword arguments (e.g., `order_event_class`).

## Testing Strategy Freeze (Proposal)

### General
- Default posture: every generic broker/feed/store test parametrizes across exchanges via fixtures.
- Only create exchange-specific test modules when behaviour is intentionally different.
- Introduce a `pytest.ini` marker to skip network-required polymarket tests unless `--live`.

### Unit Tests
- **Fixtures:**
  - Expand `tests/unit/conftest.py` factories to accept `exchange` parameter returning CCXT or Polymarket mock stores/brokers.
  - Ensure `create_polymarket_mock_store` mirrors new shared store hooks (balance seeds, order events).
- **Parametrisation:**
  - Broker suites (order submission, positions, validation) run for `exchange` in `{"ccxt", "polymarket"}` with instrument-type filters.
  - Feed suites use `granularity` table covering tick/timeframe for both exchanges.
  - Add contract tests proving `CTFeedBase` tick/ohlcv helpers behave identically in CCXT/Polymarket contexts.
- **New Coverage:**
  - Shared store helper tests (order event packaging, balance seeding) using lightweight stub subclass.
  - Session smoke test instantiating exchange session, building feeds/broker, wiring into Cerebro for both exchanges.

### Integration Tests
- Upgrade existing trading integration tests to accept `exchange_fixture` parameter.
- Keep Polymarket-only integration (`tests/integration/polymarket/test_polymarket_integration.py`) for multi-outcome behaviours; reduce overlapping assertions once shared suites cover basics.
- Validate background loop cleanup via BaseStore for both exchanges.

### E2E Tests
- Mirror CCXT E2E skeleton with Polymarket using the same harness; differentiate only where live APIs differ.
- Document expectations for running with `--live` (env vars, optional network pumps) and add smoke test to confirm offline simulations still pass.

### Testing Governance Questions
- Decide minimum exchange set for parametrized suites (both by default? toggle via env?).
- Agree on naming convention for parametrized fixtures (`exchange`, `store_kind`, etc.).
- Determine cadence for running live Polymarket tests (manual vs CI optional job).

## Documentation Plan
- Update developer docs (`docs/`) with:
  - Overview of shared architecture (store/feed/broker) and extension hooks.
  - Guidance on using new registry/session capabilities.
  - Testing handbook: how to add exchange-agnostic tests and when to create exchange-specific modules.
  - Migration notes for prediction markets once refactor lands, highlighting removed duplication.
- Refresh user-facing docs/examples to showcase `ct.exchange(...).feed()` parity between CCXT and Polymarket.
- Add release-notes entry summarizing convergence work and public API guarantees.

## Migration & Delivery Steps
1. **Assessment (done)** – capture current structure (this doc).
2. **Design Reviews** – confirm proposed hook signatures and base-class responsibilities.
3. **Incremental Refactors**
   - Move shared helpers into base classes with exhaustive tests.
   - Update Polymarket store/feed/broker to use new helpers.
   - Adjust CCXT implementations to rely on the same helpers.
4. **Test Suite Overhaul**
   - Introduce parametrized fixtures.
   - Port CCXT tests to the new structure and ensure Polymarket passes.
   - Remove redundant polymarket-only tests where overlap exists.
5. **Documentation Refresh**
   - Align guides/tutorials with new architecture.
   - Publish testing guidelines and exchange extension recipe.
6. **Stabilization**
   - Run full unit/integration/e2e suites for both exchanges.
   - Gather perf metrics to ensure no regressions.
   - Declare testing/doc strategy “frozen” and tag release.

## Open Questions & Decisions
- **BaseStore event loop ownership:** Default the loop/thread lifecycle to `BaseStore`, with an override hook for stores that need to supply their own loop. Centralizing the asyncio plumbing keeps store implementations simple while still allowing advanced integrations to inject a loop when required.
- **Order-manager hook exposure:** Treat the order manager as the broker/store bridge for order lifecycle events. Surface helpers such as `_emit_order_event(order_id, status, filled, remaining, average, info)` so exchanges hand over raw payloads without leaking internal data classes.
- **Mock store strategy:** Keep a shared mock-store base that covers queues, background-loop stubs, and the order manager. Exchange-specific subclasses extend it for metadata quirks or bespoke helpers, reducing duplication while preserving control.
- **Exchange coverage expectations:** Generic parametrized suites should exercise every registered exchange fixture (at minimum CCXT and Polymarket). Additional exchanges become eligible once credentials/fixtures are provided; tests should skip gracefully when prerequisites are absent.
- **Prediction and exotic instruments:** Include `prediction` and other special instrument types in parametrized cases whenever behaviour aligns. When an exchange lacks support, gate via fixture capability flags and rely on focused exchange-specific tests for unique flows.
- **Session extensibility:** No immediate need to expose third-party session registration beyond `register_exchange`. Document the current surface and defer custom session hooks until there is a concrete use case.
- **CI gating:** Run CCXT and Polymarket suites on every PR, accepting longer runtimes in exchange for parity confidence. Provide optional live/network jobs for extended coverage when credentials are available.
- **Compatibility layers:** No separate Polymarket compatibility layer is required. Once shared bases own historical/live handling, exchanges inherit the same behaviour; document expectations instead of adding adapters.

## Implementation Checklist

### Outstanding Tasks (Current Sprint)
- [x] Repair Polymarket store history shims so `fetch_prices_history` and `_fetch_event_tokens` operate without network clients.
- [x] Finalise CCXT malformed-candle logging integration and re-enable warnings without `caplog` gaps.
- [ ] Resume parametrisation work for broker/feed unit suites (shared fixtures, exchange filtering) ? broker suites updated; feed parametrisation still pending.
- [ ] Re-run full `pytest -q` once shims stabilise; capture runtime and confirm no hangs.
- [ ] Document new mock-store hierarchy and plotting behaviour in developer/testing guides.

### Phase 1 – Shared Infrastructure
- [x] Extract event-loop/bootstrap helpers into `BaseStore` with override hook (now shared between CCXT and Polymarket).
- [x] Provide `_emit_order_event` (or similar) in `BaseStore` and update stores to use it.
- [x] Lift balance/order watcher scaffolding into `BaseStore` (order-specific logic remains exchange-defined).
- [ ] Move feed queue/tick helpers into `CTFeedBase`.
- [ ] Introduce broker hook methods (`_submit_exchange_order`, `_cancel_exchange_order`, `_resolve_execution_price`).

### Phase 2 – Exchange Alignments
- [ ] Refactor `CCXTStore` to consume new helpers.
- [ ] Refactor `PolymarketStore` to consume new helpers and drop duplicates.
- [ ] Update `CCXTDataFeed` and Polymarket feeds to rely on `CTFeedBase` helpers.
- [ ] Adjust `PolymarketBroker` to use broker hooks; align CCXT brokers where needed.

### Phase 3 – Test Suite Convergence
- [x] Extend fixtures to accept `exchange` parameter and shared mock base.
- [ ] Parametrize broker tests across exchanges/instrument types, adding skips as needed.
- [ ] Parametrize feed tests across exchanges/granularities.
- [ ] Add contract tests for shared helpers.
- [ ] Ensure integration/e2e suites cover both exchanges or document gaps.

### Phase 4 – Documentation & Release
- [ ] Update developer docs describing new hooks and extension points.
- [ ] Document testing strategy, fixture usage, and CI expectations.
- [ ] Prepare release notes / migration guidance.
- [ ] Finalize “freeze” checklist once previous phases complete.


### Outstanding Questions During Implementation
- Should we add convenience emit helpers for orderbook/terminal events to the shared mock base or let exchanges extend it ad-hoc?

## Next Actions
- Circulate this plan for review; resolve open questions.
- Finalize hook signatures and fixture APIs.
- Draft testing/documentation “freeze” proposal based on agreed decisions.
