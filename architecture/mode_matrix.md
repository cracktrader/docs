# Mode Matrix

This page summarizes what should stay stable across runtime modes and where behavior is intentionally different.

## Core Rule

Strategies should target the same high-level contracts across backtest, paper, sandbox, and live.

Mode differences belong behind execution adapters, runtime safeguards, and environment-specific integrations.

## Runtime Modes

| Mode | Market input shape | Execution path | Determinism expectation | Typical use |
| --- | --- | --- | --- | --- |
| Backtest | Historical replay or stored feed input | Historical simulator adapter | Strong deterministic expectation | research, replay, regression tests |
| Paper | Live or replayed market input | Simulated execution adapter | Deterministic policy behavior, non-deterministic market arrival if live-fed | dry runs, strategy shakeout |
| Sandbox | Real venue sandbox/testnet input | Real broker adapter against sandbox venue | Environment-dependent; safer than live but not a simulator | adapter validation, pre-live checks |
| Live | Real venue input | Real broker adapter | No deterministic execution guarantee | production trading |

## What Should Not Change Across Modes

- strategy input and output contract shapes
- runtime identities such as `strategy_id`, `execution_id`, and `route_id`
- attribution on intents, reports, and diagnostics
- reference-data normalization rules
- risk policy semantics unless a mode-specific safety default is explicit

## What Can Change Across Modes

| Surface | Backtest / paper | Sandbox / live |
| --- | --- | --- |
| Fill formation | simulator realism model, latency injection, top-of-book assumptions | venue-reported fills and acknowledgements |
| Market timing | replay clock or local scheduling | venue/network timing |
| Broker capability | constrained to adapter simulation features | constrained by venue and adapter behavior |
| Safety defaults | stricter deterministic and test-oriented behavior | stricter operational safeguards and kill-switch expectations |

## Reading the Matrix Correctly

Do not model each mode as a separate architecture.

Instead:

- keep one runtime contract surface
- isolate mode-specific behavior behind execution adapters
- document the intentional divergences explicitly

For detailed linked divergences, see the [Mode Divergence Ledger](../mode_divergence_ledger.md).
