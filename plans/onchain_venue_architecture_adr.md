# On-chain Venue Architecture ADR (M1)

Status: Proposed
Date: 2026-02-27
Related issues: #72 #73 #74 #75 #76 #87
Parent epic: #71

## Context
Cracktrader needs a shared on-chain foundation for EVM venues (Ethereum, BSC, Polygon) that preserves existing runtime contracts while allowing venue-specific connectors (Uniswap/Pancake v2 first, v3-ready seams later).

## Decision
Adopt a composable architecture with explicit boundaries:
- `EvmCore`: chain config, rpc, nonce/gas policy, tx lifecycle tracking/finality.
- `TokenRegistry`: strict symbol-first allowlist and canonical token identity `(chain_id, token_address)`.
- `Signer`: injectable signer interface, with local encrypted-key signer backend in v1.
- `Broadcaster`: private-first tx broadcasting with deterministic public fallback policy.
- `DexConnector`: venue-specific quote/tx/receipt mapping layer (implemented in M2).
- `ApprovalManager`: allowance lifecycle orchestration (implemented in M2).

This ADR locks tx lifecycle semantics and tx->broker status mapping used by M1+ issues.

## Why v2 pools first
- v2 constant-product pools give deterministic quote/fill semantics suitable for first contract coverage.
- v3 complexity (ticks/concentrated liquidity/range math) is deferred while preserving interface seams.

## Canonical Tx Lifecycle

States:
- `created`
- `signed`
- `submitted_private`
- `submitted_public`
- `pending`
- `mined`
- `finalized`
- `replaced`
- `dropped`
- `reverted`
- `reorged`
- `failed`

Allowed transitions (high-level):
- `created -> signed`
- `signed -> submitted_private | submitted_public | failed`
- `submitted_private -> pending | submitted_public | failed`
- `submitted_public -> pending | failed`
- `pending -> mined | replaced | dropped | reverted | reorged | failed`
- `mined -> finalized | reorged | reverted`
- `reorged -> pending | dropped | reverted`
- `replaced -> pending` (new hash)
- terminal: `finalized`, `dropped`, `reverted`, `failed`

## Tx Lifecycle -> Broker Status Mapping

| Tx lifecycle state | Broker status (`OrderStatus`) | Notes |
|---|---|---|
| `created` | `submitted` | local intent accepted for signing |
| `signed` | `submitted` | signed but not broadcast |
| `submitted_private` | `accepted` | private relay path in flight |
| `submitted_public` | `accepted` | public mempool path in flight |
| `pending` | `accepted` | tx known by network but not mined |
| `mined` | `partial` | mined but finality threshold not met |
| `finalized` | `completed` | terminal success |
| `replaced` | `accepted` | old hash replaced by successor |
| `reorged` | `accepted` | previously mined tx lost block inclusion |
| `dropped` | `canceled` | terminal non-execution |
| `reverted` | `rejected` | terminal execution failure |
| `failed` | `rejected` | terminal local/transport/policy failure |

## Retry / replacement / reorg behavior
- Retry policy is explicit and policy-driven in broadcaster.
- Replacement is explicit and links old/new tx hashes.
- Reorg transitions are explicit and test-covered; finality rules are chain-specific.

## Required telemetry fields
Minimum fields on lifecycle/broadcast events:
- `chain_id`
- `tx_hash`
- `state`
- `nonce`
- `attempt`
- `private_attempted`
- `private_succeeded`
- `fallback_used`
- `latency_ms`
- `error_code` (if any)
- `error_message` (sanitized)
- `replaced_by` / `replaces` (when applicable)
- `confirmations`
- `block_number`

## Security and safety
- No secret material in logs/errors.
- Live/sandbox behavior remains explicit; no silent simulation fallback.
- Deterministic behavior required for test/backtest scaffolding.

## Consequences
- M1 can implement reusable foundations before connector integration.
- M2/M3 work uses stable tx lifecycle/status semantics.
- Adds explicit seams for future hardware/remote signers and v3 connectors.
