# DSL Research Factory Capability Matrix (Phase A)

This document inventories Cracktrader capabilities against the DSL evaluator checklist
in `dsl.txt` Section 9. It is the Phase A deliverable that highlights what is already
supported, where it lives, and what is missing.

Decisions captured for implementation:
- Dataset resolution uses the existing `LocalDataStore` in `src/cracktrader/data/local_store.py`.
- Run registry / artifact root is configurable (default path TBD by the evaluator config).
- `evaluate()` will live under a new `cracktrader.research` module (no CLI surface yet).

Legend:
- Status: Yes / Partial / No
- Effort: Small / Medium / Large (for closing the gap)

## 9.1 Backtesting Capabilities

| Capability | Status | Evidence | Notes / Gaps | Effort |
| --- | --- | --- | --- | --- |
| Run backtest on OHLCV for specified symbols/timeframe | Yes | `src/cracktrader/cerebro.py`, `src/cracktrader/feeds/ccxt.py`, `src/cracktrader/feeds/polymarket.py` | Backtrader-based engine supports OHLCV feeds and timeframe selection. | Small |
| Compute/execute timing separation (close vs next open) | Partial | `src/cracktrader/broker/ccxt_simulation_broker.py`, `src/cracktrader/broker/polymarket_simulation_broker.py`, `tests/unit/broker/test_backtest_market_fill_timing.py` | Market orders can fill next-bar open when cheat-on-close is disabled; compute timing is strategy-driven (no global enforcement). | Medium |
| Deterministic results guarantee | Partial | `src/cracktrader/cerebro.py` | Deterministic for fixed data, but no explicit determinism contract, seed logging, or hash tracking. | Medium |

## 9.2 Execution Modeling

| Capability | Status | Evidence | Notes / Gaps | Effort |
| --- | --- | --- | --- | --- |
| Market orders | Yes | `src/cracktrader/broker/ccxt_simulation_broker.py`, `src/cracktrader/broker/polymarket_simulation_broker.py` | Supported in simulation and live flows. | Small |
| Limit orders | Yes | `src/cracktrader/broker/ccxt_simulation_broker.py`, `src/cracktrader/broker/polymarket_simulation_broker.py` | Supported; Polymarket enforces limit-only in sim/live. | Small |
| Limit-with-timeout | No | N/A | No timeout handling or expiry-by-bar support in brokers. | Medium |
| Partial fills modeling | Partial | `src/cracktrader/broker/ccxt_order.py` | Live order updates handle partial fills; backtest/paper sim fills full size only. | Medium |
| Slippage model hooks | No | N/A | No slippage model in broker or comm info. | Medium |
| Spread modeling | No | N/A | No spread model; fills use bar prices. | Medium |

## 9.3 Cost Modeling

| Capability | Status | Evidence | Notes / Gaps | Effort |
| --- | --- | --- | --- | --- |
| Maker/taker fees | Yes | `src/cracktrader/comm_info/base.py` | `TradingCommInfo` and `PredictionCommInfo` handle maker/taker. | Small |
| Slippage in bps | No | N/A | Not implemented in broker or comm info. | Medium |
| Slippage as function of vol/liquidity | No | N/A | No hooks for dynamic slippage. | Medium |

## 9.4 Risk and Constraints

| Capability | Status | Evidence | Notes / Gaps | Effort |
| --- | --- | --- | --- | --- |
| Position size constraints | No | `src/cracktrader/broker/universal_broker_base.py` | Placeholders exist (`max_position_size`, `max_order_size`) but not enforced. | Medium |
| Leverage constraints | Partial | `src/cracktrader/comm_info/base.py`, `src/cracktrader/broker/universal_broker_base.py` | Commission info supports leverage math; no enforcement logic or risk checks. | Medium |
| Daily loss / max drawdown kill-switch | No | `src/cracktrader/broker/universal_broker_base.py` | Placeholders exist (`daily_loss_limit`, `kill_switch_active`) but no implementation. | Medium |
| Max order count / turnover sanity checks | No | N/A | No broker- or strategy-level enforcement. | Medium |

## 9.5 Splits + Walk-Forward Evaluation

| Capability | Status | Evidence | Notes / Gaps | Effort |
| --- | --- | --- | --- | --- |
| IS/OOS split execution | No | N/A | Requires orchestrator to slice data and run repeated evals. | Medium |
| Final exam split access control | No | N/A | Needs evaluator controls layer. | Medium |
| Regime breakdown reporting | No | N/A | No regime segmentation or reporting pipeline. | Medium |

## 9.6 Stress Tests (Robustness)

| Capability | Status | Evidence | Notes / Gaps | Effort |
| --- | --- | --- | --- | --- |
| Cost sensitivity reruns | No | N/A | Requires evaluator orchestration and cost model hooks. | Medium |
| Execution delay reruns | No | N/A | Needs orchestrated reruns with delay offsets. | Medium |
| Parameter neighborhood reruns | No | N/A | Requires parameter perturbation engine. | Medium |

## 9.7 Paper Trading Integration

| Capability | Status | Evidence | Notes / Gaps | Effort |
| --- | --- | --- | --- | --- |
| Promote strategy config to paper trading | Partial | `src/cracktrader/broker/ccxt_back_broker.py`, `src/cracktrader/broker/polymarket_back_broker.py` | Paper/backtest brokers exist; no promotion workflow or evaluator hooks. | Medium |
| Operational logging + metrics capture | Partial | `src/cracktrader/analyzers/research.py`, `src/cracktrader/strategy/base.py` | Trade logs and equity curves exist; no unified StrategyReport format. | Medium |

## 9.8 Reporting Outputs

| Capability | Status | Evidence | Notes / Gaps | Effort |
| --- | --- | --- | --- | --- |
| Trades list with timestamps, fills, fees, slippage | Partial | `src/cracktrader/analyzers/research.py` | Trade log includes open/close, pnl, pnl_commission; no per-fill fees or slippage attribution. | Medium |
| Equity curve | Yes | `src/cracktrader/analyzers/research.py` | EquityCurveAnalyzer records cash/value per bar. | Small |
| Drawdown series | Partial | Backtrader analyzers | Available via Backtrader, but not surfaced in a standard report. | Small |
| Per-split metrics | No | N/A | Requires evaluation orchestrator. | Medium |
| Regime segmented metrics | No | N/A | Requires regime segmentation. | Medium |
| Failure classification hooks | No | N/A | No FailureCapsule or classifier implementation. | Medium |

