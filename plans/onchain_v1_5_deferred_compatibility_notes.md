# On-chain v1.5+ Deferred Compatibility Notes

Status: Proposed  
Date: 2026-02-27  
Related: #86

## Purpose

Document compatibility constraints for deferred on-chain enhancements so v1 interfaces remain stable while v1.5+ capabilities are added.

## Locked v1 interfaces to preserve

- Factory/session entry points:
  - `ct.exchange("uniswap" | "pancakeswap")`
  - `session.feed(...)`
  - `session.broker(...)`
- On-chain broker result contract:
  - tx lifecycle -> `OrderStatus` mapping
  - `fee_breakdown` structure
  - diagnostics payload fields (`tx_state`, `tx_hash`, `chain_id`, `attempt_id`, replacement/fallback flags)
- Connector protocol seam:
  - `quote`, `build_swap_tx`, `parse_receipt`, `metadata`, `validate_path`
- Backtest seams:
  - `OnchainBacktestDataset`
  - `OnchainBacktestFillSimulator`
  - `OnchainBacktestBrokerCallback`

## Deferred track requirements

### 1) Contract executor backend (on-chain enforcement)

Must be additive and not replace current broker callback/factory usage.

Compatibility requirements:
- Keep current `submit_swap` API valid for existing callers.
- Enforcer policy failures must map to existing lifecycle/status semantics (`failed`/`rejected` pathways).
- New policy metadata should be additive under diagnostics or dedicated metadata keys.

### 2) Permit2 approval flow

Compatibility requirements:
- Keep current approval manager behavior as default fallback.
- Permit2 path should reuse same fee attribution model and include Permit2 gas in breakdown.
- No change to public session/factory shape required for migration.

### 3) Mempool watcher / MEV-aware execution

Compatibility requirements:
- Watcher signals must not bypass existing lifecycle state machine.
- Replacement/fallback diagnostics remain canonical event source for reliability tests.
- Any anti-MEV policy knobs should be additive config options.

### 4) v3 connector support (Uniswap/Pancake concentrated liquidity)

Compatibility requirements:
- Keep connector protocol shape stable; v3 extends behavior behind same protocol.
- v2 routes and tests remain unchanged and green.
- Backtest simulation path must support deterministic replay parity for v3 math with separate fixtures.

## Migration and rollout guidance

- Additive rollout only: no hard-breaking changes to v1 module/class names.
- Feature flags/config gates for new execution policies.
- Extend existing contract/integration matrix first, then enable defaults.
- Maintain explicit docs and runbook deltas for each new capability.

