# On-chain Token Registry (M1)

The on-chain token registry is an allowlist manifest for deterministic symbol/address resolution.

## Canonical identity
- Token identity is `(chain_id, token_address)`.
- Symbols are convenience aliases and must resolve unambiguously per chain.

## Manifest rules
- `chain_id`: integer EVM chain id.
- `symbol`: uppercase symbol.
- `address`: 20-byte hex address (`0x...`, 42 chars).
- `decimals`: integer base-unit precision.
- optional: `is_wrapped`, `native_symbol`.

## Update process (PR-governed)
1. Open a PR that edits the registry manifest.
2. Include rationale and source for new token metadata.
3. Add/update unit tests for resolution + decimals behavior.
4. Confirm unknown/ambiguous symbols still fail explicitly.
5. Link affected issue(s) and request reviewer approval.

No out-of-band registry mutation is allowed.
