# OHLCVSubsystem Decision Checkpoint (Phase 5 P5-S1)

Date: 2026-02-22

## Decision

`OHLCVSubsystem` is retained as a legacy/test utility and is not wired into the active runtime store path.

## Evidence

- `BaseStore` constructs `MarketDataSubsystem` directly in `src/cracktrader/store/base_store.py`.
- `MarketDataSubsystem` uses `OHLCVCandleRouter` for OHLCV behavior in `src/cracktrader/store/subsystems/market_data_subsystem.py`.
- No runtime references to `OHLCVSubsystem` were found in `src/cracktrader` outside its own module.
- `OHLCVSubsystem` is still covered by unit tests in `tests/unit/store/test_ohlcv_subsystem.py`.

## Outcome

- Keep `OHLCVSubsystem` in the repository for now.
- Treat it as non-runtime by default.
- Preserve existing tests as documentation of legacy behavior.

## Follow-up

- During Phase 5 consolidation, either:
  - merge any valuable behavior into `OHLCVCandleRouter`, or
  - deprecate/remove `OHLCVSubsystem` with explicit test replacements.
