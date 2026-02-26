# Rust CI Policy

This project uses a tiered Rust CI policy so parity coverage is predictable while CI remains stable.

## Required Rust Jobs

The following jobs in `.github/workflows/ci.yml` are required gates:

- `rust-parity-required`
- `rust-feed-parity-required`

`build` depends on both required Rust jobs, so PRs cannot merge cleanly unless these lanes pass.

### Required scope

`rust-parity-required` must:

- install/build the Rust extension in CI,
- run Python<->Rust engine parity suites in at least one required environment,
- fail on parity regressions.

Current required engine parity coverage includes:

- Rust bridge smoke tests,
- event normalization parity,
- sequencer parity,
- invariant parity,
- core parity fixtures,
- legal transition matrix parity,
- replay parity.

`rust-feed-parity-required` must:

- install/build the Rust extension in CI,
- run the feed parity gate (`scripts/run_feed_rust_parity_gate.py`) with benchmark checks,
- fail on feed parity or benchmark gate regressions.

## Optional Rust Job

`rust-parity-extended` is non-blocking (`continue-on-error: true`).

It runs a heavier parity gate (including optional integration and benchmark checks) to surface issues early without blocking merges on occasional environment noise.

## Local Skip/Fallback Policy

When the Rust extension is unavailable locally:

- Python<->Rust parity modules skip via runtime availability checks.
- Python-only suites remain runnable.
- Contributors can build the extension with:
  - `python scripts/install_rust_backend.py`

Local skips are acceptable for development, but PRs must pass required Rust CI lanes before merge.
