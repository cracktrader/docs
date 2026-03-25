# Quick Reference Guide

Essential commands and information for Cracktrader development.

## Setup Commands

```bash
# Initial setup
python -m venv .venv
.venv\Scripts\activate  # Windows
pip install -e ".[dev,test,web]" pre-commit
pre-commit install

# Verify setup
pytest -q tests/contracts/engine tests/contracts
pre-commit run --all-files
```

## Daily Development

```bash
# Before coding
git pull origin main

# Code quality checks
pre-commit run --all-files
pre-commit run --hook-stage pre-push --all-files
mypy src

# Run tests
pytest -q
pytest -q tests/contracts/engine tests/contracts

# Before committing
pytest -q tests/contracts/test_runtime_hook_contracts.py
```

## CI/CD Pipeline

- **Triggers**: Push to `main`/`develop`, version tags, and pull requests
- **Core jobs**: lint, type-check, test matrix, contracts, replay regression, web smoke
- **Rust jobs**: required parity gates plus an extended parity job
- **Artifacts**: build distributions and feed benchmark reports

## Key File Locations

- **CI/CD**: `.github/workflows/ci.yml`
- **Pre-commit**: `.pre-commit-config.yaml`
- **Config**: `pyproject.toml`
- **Tests**: `tests/unit/`, `tests/integration/`
- **Docs**: `docs/`

## Performance Benchmarks

- **Data Processing**: 55,000+ candles/second
- **Reordering**: 53,000+ candles/second  
- **Memory**: <1MB for 1000 candles
- **Coverage posture**: use boundary coverage docs, not a fixed percentage target

## Troubleshooting

```bash
# Fix common issues
ruff format src/ tests/
ruff check src/ tests/ --fix
pre-commit run --all-files

# Debug failing tests
pytest tests/path/to/test.py -v -s --tb=long

# Performance issues
pytest tests/unit/feed/test_sub_minute_timeframes.py -v -s
```

## Architecture Highlights

- **Feeds**: Support 1s, 10s, 30s, 1m+ timeframes with tick reordering
- **Brokers**: Paper and live trading with CCXT integration
- **Engine**: Native-first runtime entrypoints (`run_native*`)
- **Testing**: 19,200+ test lines vs 7,400 source lines
- **Quality**: Pre-commit, contract suites, replay regression, and Rust parity gates
