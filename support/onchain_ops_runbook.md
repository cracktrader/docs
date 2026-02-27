# On-chain Operations Runbook

This runbook covers common on-chain execution failures and recovery steps for Uniswap/PancakeSwap venues.

## Scope

Applies to:
- `ct.exchange("uniswap")`
- `ct.exchange("pancakeswap")`
- paper/backtest/live paths with on-chain broker diagnostics

## Required Diagnostics

For each failed or delayed order, capture:
- `tx_state`
- `tx_hash`
- `chain_id`
- `attempt_id`
- `replacement_attempt_count`
- `fallback_used`

These are available in on-chain broker result metadata diagnostics.

## Failure: Stuck Pending Tx

Symptoms:
- status remains `accepted`
- no mined/finalized transition

Actions:
1. Confirm nonce and mempool visibility for `tx_hash`.
2. Check current base/priority fee against submitted gas settings.
3. Re-submit as replacement (same nonce, higher fee cap).
4. Record replacement attempt in diagnostics.

## Failure: Replacement / Retry Path

Symptoms:
- original tx not finalizing
- replacement tx submitted

Actions:
1. Link original and replacement hashes in incident notes.
2. Verify `replacement_attempt_count` incremented.
3. Confirm fee attribution includes replacement gas component.
4. If replacement also stalls, repeat with bounded retries only.

## Failure: Private Relay Fallback

Symptoms:
- private submission path fails
- public fallback used

Actions:
1. Confirm fallback policy is expected for environment.
2. Verify `fallback_used=True` in diagnostics.
3. Audit relay/network errors for cause (rate limit, auth, transport).
4. Continue tracking tx under public hash.

## Failure: Reorg

Symptoms:
- tx was mined then state regresses (reorged/pending/reverted path)

Actions:
1. Delay final accounting until finality threshold is met.
2. Re-check confirmations and chain head stability.
3. If tx drops after reorg, classify as canceled/rejected per lifecycle mapping.
4. Reconcile downstream PnL and position records.

## Failure: Reverted Tx

Symptoms:
- terminal `reverted` or `failed`

Actions:
1. Inspect revert reason / execution logs where available.
2. Verify pool path and token approvals.
3. Confirm slippage constraints were not too strict (`minAmountOut`).
4. Re-run with corrected parameters if policy allows.

## Approval/Gas Recovery

When approval is linked to the same logical order chain:
1. Include approval gas in fee attribution.
2. Confirm fee breakdown surfaces approval component separately.
3. If approval failed, rotate to explicit approve+retry flow.

## Escalation Checklist

- Incident includes tx diagnostics fields above.
- Lifecycle timeline captured (submitted/pending/mined/finalized or failure path).
- Fee attribution snapshot recorded.
- Root cause tagged: gas, nonce, relay, path/liquidity, chain stability, config.

