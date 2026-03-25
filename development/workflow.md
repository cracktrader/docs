# Development Workflow Guide

This guide documents the development workflow that the `cracktrader` repository actually runs today.

## Overview

Cracktrader relies on pre-commit, targeted local checks, and a CI pipeline that emphasizes contract coverage, replay safety, and Rust parity gates.

### Key Technologies

- Python 3.11+
- Ruff for linting and import-order checks
- Black for formatting compatibility
- isort in pre-commit
- pytest for unit, integration, contract, regression, and performance tests
- mypy for static type checks
- GitHub Actions for CI/CD

## Development Environment Setup

### 1. Initial Setup

```bash
git clone <repository-url>
cd cracktrader

python -m venv .venv
source .venv/bin/activate  # Linux/Mac
# or
.venv\Scripts\activate  # Windows

pip install -e ".[dev,test,web]" pre-commit
```

### 2. Install Pre-commit Hooks

```bash
pre-commit install
pre-commit run --all-files
```

### 3. Verify Installation

```bash
pytest -q
pytest -q tests/contracts/engine tests/contracts
pre-commit run --all-files
pre-commit run --hook-stage pre-push --all-files
mypy src
```

## Code Quality Tools

### Ruff

Ruff is the primary linting layer in the repository.

```toml
[tool.ruff]
line-length = 100

[tool.ruff.lint]
extend-select = ["I"]
ignore = ["E501"]
```

```bash
ruff check src/ tests/
ruff check src/ tests/ --fix
ruff format src/ tests/
```

### Black

Black remains part of pre-commit and CI compatibility checks.

```bash
black src/ tests/
black --check src/ tests/
```

### isort

isort still runs in pre-commit to normalize import blocks.

```bash
isort src/ tests/
```

## Testing Strategy

### Test Categories

1. Unit tests for isolated component behavior.
2. Integration tests for cross-component wiring that stays offline.
3. Contract tests for runtime, engine, and schema boundaries.
4. Replay and regression tests for deterministic behavior over time.
5. Performance tests for throughput and latency-sensitive paths.

### Common Commands

```bash
pytest -q
pytest -q tests/unit/
pytest -q tests/integration/
pytest -q tests/contracts/engine tests/contracts
pytest -q tests/regression/test_replay_regression.py
pytest tests/unit/feed/test_sub_minute_timeframes.py -v -s
```

### Coverage Guidance

Do not treat a single coverage percentage as the primary quality target. Prefer the boundary-based guidance in [`testing/test_coverage.md`](../testing/test_coverage.md) and ensure important runtime, replay, and contract seams are explicitly protected.

## CI/CD Pipeline

### Workflow File

- `.github/workflows/ci.yml`

### Trigger Conditions

```yaml
on:
  push:
    branches: [main, develop]
    tags: ['v*']
  pull_request:
```

### Current Pipeline Stages

1. `lint`
   Runs `pre-commit run --all-files` and the configured pre-push hook set.
2. `type-check`
   Runs `mypy src`.
3. `test`
   Runs the core pytest suite on Python 3.11 and 3.12.
4. `contracts`
   Validates the fixture manifest and runs `tests/contracts/engine` plus `tests/contracts`.
5. `rust-parity-required`
   Runs the required Rust parity suites.
6. `rust-parity-extended`
   Runs a broader parity gate as a non-blocking follow-up.
7. `rust-feed-parity-required`
   Runs the feed Rust parity gate and uploads benchmark artifacts.
8. `replay-regression`
   Runs the replay regression contract.
9. `test-web`
   Runs webhook smoke tests.
10. `build`
    Produces distribution artifacts once required checks pass.
11. `release`
    Publishes tagged builds when release credentials are configured.

### Artifact Collection

- Build artifacts (`wheel`, `sdist`)
- Feed benchmark artifacts and summaries

## Pre-commit Hooks

### Current Hook Set

1. Basic checks
   `end-of-file-fixer`, `check-yaml`
2. Code quality
   `ruff`, `black`, `isort`
3. Pre-push gate
   repo-level `ruff` and `black --check` runs across `src` and `tests`

### Useful Commands

```bash
pre-commit run --all-files
pre-commit run --hook-stage pre-push --all-files
pre-commit autoupdate
```

## Release Process

1. Update the version in `pyproject.toml`.
2. Update changelog or release notes.
3. Create a git tag such as `v1.x.x`.
4. Push the tag with `git push origin v1.x.x`.

## Development Best Practices

1. Keep commits atomic and scoped to one coherent change.
2. Prefer boundary coverage over percentage chasing.
3. Add explicit tests for runtime, replay, and contract-sensitive changes.
4. Avoid hardcoded secrets and be careful with network-facing code.
5. Run the minimum relevant local checks before opening a PR.

## Troubleshooting

### Pre-commit Failures

```bash
ruff check src/ tests/ --fix
ruff format src/ tests/
isort src/ tests/
pre-commit run --all-files
```

### CI Failures

1. Check the failing GitHub Actions job first.
2. Re-run the matching local command.
3. Verify extras are installed for the job you are reproducing.

## Summary

The current workflow prioritizes truthful local validation, strong boundary contracts, and CI gates that reflect real runtime risks rather than outdated percentage targets or tools that are no longer wired into the repository.
