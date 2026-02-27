# On-chain Signer Security Checklist (M1)

Use this checklist for any signer backend changes.

- [ ] No secret material logged (private key bytes, decrypted payloads, passphrases).
- [ ] Error messages are sanitized and do not leak key material.
- [ ] Decryption/authentication failure paths are covered by tests.
- [ ] Signing path is deterministic and tested for stable inputs.
- [ ] Key loading requires explicit passphrase input.
- [ ] Backward-compatible interface seam remains for hardware/remote signers.
- [ ] Any crypto/KDF parameter changes are documented in PR notes.

M1 backend: local encrypted-key signer.
Future backends: hardware and remote signers (additive, interface-compatible).
