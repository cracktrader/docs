# Rust CI Policy

This project uses a two-tier Rust CI policy so parity coverage is predictable while CI remains stable.

## Required Job

`rust-parity-required` in `.github/workflows/ci.yml` is a required gate.

It must:

- install/build the Rust extension in CI,
- run Python↔Rust parity suites in at least one required environment,
- fail the PR when parity regressions are detected.

Current required parity coverage includes:

- Rust bridge smoke tests,
- event normalization parity,
- sequencer parity,
- invariant parity,
- core parity fixtures,
- legal transition matrix parity,
- replay parity.

## Optional Job

`rust-parity-extended` is non-blocking (`continue-on-error: true`).

It runs a heavier parity gate script (including optional integration and benchmark checks) to surface issues early without blocking merges on occasional environment noise.

## Local Skip/Fallback Policy

When the Rust extension is unavailable locally:

- Python↔Rust parity modules use `is_rust_available()` gates and skip gracefully.
- Python-only tests remain executable.
- Contributors can build the extension via:
  - `python scripts/install_rust_backend.py`

PRs are still expected to pass the required Rust parity CI job before merge.
