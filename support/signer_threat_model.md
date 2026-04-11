# Signer Threat Model and Operational Handling

This guide defines the current signer security baseline for Cracktrader on-chain execution.
It is written for operators and contributors who touch signer-related code or runtime setup.

## Scope

Applies to the current M1 signer backend:
- local encrypted key material
- explicit passphrase input for key loading
- signer usage behind the `Signer` interface

Related references:
- [On-chain Signer Security Checklist](../plans/onchain_signer_security_checklist.md)
- [On-chain Venue Architecture ADR](../plans/onchain_venue_architecture_adr.md)

## Threat Model (Current Guarantees)

The project currently aims to reduce common operational leakage risks around signer material.

### What the project is expected to protect against

- Accidental secret leakage through normal logs, diagnostics, and error handling.
- Plaintext key storage in repository files and normal checked-in config.
- Drift away from the `Signer` interface that would bypass shared safety rules.

### What the project does not guarantee

- Protection if the host, container, or CI worker is fully compromised.
- Protection if operators intentionally print or persist passphrases/private material outside sanctioned paths.
- Protection against external wallet/key-management systems that are configured insecurely.
- Full key-management policy for every organization deployment model.

## Operator Handling Expectations

### Passphrases

- Provide passphrases through secure runtime secret channels, not source-controlled files.
- Do not pass secrets through channels that are commonly persisted in history or telemetry.
- Treat passphrases as short-lived process input, not reusable config payload.

### Encrypted key material

- Keep encrypted key artifacts outside source control and shared public storage.
- Restrict filesystem and runtime permissions to the smallest operational scope possible.
- Rotate encrypted key artifacts and passphrases after suspected exposure.

### Error and log handling

- Assume logs can be broadly visible to operators and incident tooling.
- Preserve sanitized errors; do not reintroduce raw secret values in wrappers or adapters.
- During incidents, capture signer failure class and context, not key bytes, plaintext payloads, or passphrases.

## Contributor Review Expectations

Any signer-related PR should satisfy all of the following:

- Checklist compliance in [On-chain Signer Security Checklist](../plans/onchain_signer_security_checklist.md).
- Negative-path tests for decryption/authentication/signing failure handling.
- Confirmation that no new logs or error surfaces can emit secret material.
- Explicit PR notes for any crypto/KDF parameter changes.
- Interface compatibility preserved for future hardware/remote signer backends.

## Operational Escalation Baseline

When signer compromise is suspected:

1. Stop or restrict affected live signing workflows.
2. Rotate passphrases and encrypted key material.
3. Review logs/telemetry for accidental leakage indicators.
4. Revalidate sanitized error behavior before restoring normal operation.

This guide is a security baseline, not a substitute for organization-specific incident policy.
