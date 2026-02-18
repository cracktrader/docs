# Backtrader Decoupling Gate Runbook

Date: 2026-02-18  
Owner: Cracktrader maintainers  
Purpose: Deterministic checklist and command set to satisfy Phase 0 gating before deeper decoupling work.

## Preconditions

- Run from repository root.
- Python environment has project dependencies installed.
- For networked examples, internet access is available.

## 1) Unit + Contract Baseline

Run:

```bash
python -m pytest -q tests/unit
python -m pytest -q tests/contracts
```

Pass criteria:

- `tests/unit` exits 0.
- `tests/contracts` exits 0.
- No unexpected runtime exceptions in output.

Validated snapshot (2026-02-18):

- `tests/unit`: `1663 passed, 88 skipped`
- `tests/contracts`: `38 passed`

## 2) Fee/Order-Lifecycle Flake Gate (3 Consecutive Passes)

Run:

```bash
python -m pytest -q tests/contracts/test_fees.py tests/contracts/test_order_lifecycle.py tests/contracts/test_order_scenarios.py
python -m pytest -q tests/contracts/test_fees.py tests/contracts/test_order_lifecycle.py tests/contracts/test_order_scenarios.py
python -m pytest -q tests/contracts/test_fees.py tests/contracts/test_order_lifecycle.py tests/contracts/test_order_scenarios.py
```

Pass criteria:

- All three runs exit 0.
- No intermittent failures across runs.

Validated snapshot (2026-02-18):

- Run 1: `12 passed`
- Run 2: `12 passed`
- Run 3: `12 passed`

## 3) Offline Example Gate

Run:

```bash
python examples/basics/hello_engine.py
python examples/basics/session_quickstart.py
python examples/basics/shared_store.py
python examples/prediction/kalshi_prediction_paper.py
```

Expected output signatures:

- `hello_engine.py`: `Engine run complete`
- `session_quickstart.py`: feed names for CCXT/Polymarket/Kalshi and broker cash lines
- `shared_store.py`: `feed1.store is feed2.store -> True` and portfolio summary lines
- `kalshi_prediction_paper.py`: `Final cash:`

Pass criteria:

- All four commands exit 0.
- Signatures above are present.

## 4) Networked Example Gate

Run:

```bash
python examples/backtesting/ccxt_backtest_baseline.py
python examples/live/ccxt_live_paper.py
python examples/backtesting/timeframes_and_ticks.py
python examples/prediction/polymarket_feed_demo.py
```

Expected output signatures:

- `ccxt_backtest_baseline.py`: final cash/portfolio summary lines
- `ccxt_live_paper.py`: `Streaming about` and `Max bars reached - stopping run.`
- `timeframes_and_ticks.py`: repeated `MIN` lines and final portfolio summary
- `polymarket_feed_demo.py`: `Finished run | bars=` line

Pass criteria:

- All commands exit 0.
- Each script emits its signature line(s).

## 5) Advanced/Compatibility Gate

Run:

```bash
python examples/multi_market/multi_exchange_two_feeds.py
python examples/orders/order_types.py
python examples/multi_market/mixed_ccxt_polymarket.py
python examples/prediction/polymarket_closed_outcome.py
```

Expected output signatures:

- `multi_exchange_two_feeds.py`: spread lines and composite/final portfolio values
- `order_types.py`: repeated `Submitted market buy`
- `mixed_ccxt_polymarket.py`: `Completed run | polymarket_bars=`
- `polymarket_closed_outcome.py`: `Tradeable flag reported by feed: False`

Pass criteria:

- All commands exit 0.
- All signature lines present.

## 6) One-Command Targets

Use Make targets:

```bash
make gate-unit
make gate-contracts
make gate-flake
make gate-offline
make gate-examples-network
make gate-examples-advanced
make gate-all
```

`gate-all` runs all required gates in sequence.
