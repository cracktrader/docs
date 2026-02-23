# OHLCVSubsystem Decision Checkpoint (Phase 5 Consolidation)

Date: 2026-02-22

## Final Decision

`OHLCVSubsystem` is removed. Runtime OHLCV ownership is consolidated on:

- `BaseStore -> MarketDataSubsystem -> OHLCVCandleRouter`
- `StoreMarketDataFeed` contract surface for feed-driver interaction

## Evidence

- `BaseStore` constructs `MarketDataSubsystem` directly in `src/cracktrader/store/base_store.py`.
- `MarketDataSubsystem` owns OHLCV routing via `OHLCVCandleRouter` in `src/cracktrader/store/subsystems/market_data_subsystem.py`.
- No production call sites depended on `OHLCVSubsystem`.
- Legacy module and legacy module-specific tests were removed.

## Outcome

- Single runtime owner for OHLCV behavior is now explicit.
- Legacy/test-only subsystem ambiguity is eliminated.
- Role coverage remains in `tests/unit/store/test_ohlcv_subsystem_role.py`.

## Follow-up

- Keep market-data contract tests (`tests/contracts/test_market_data_contracts.py`) as the authority for OHLCV callback/ordering behavior.
