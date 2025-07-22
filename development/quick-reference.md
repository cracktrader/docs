# Quick Reference Guide

Essential commands and information for Cracktrader development.

## Setup Commands

```bash
# Initial setup
python -m venv .venv
.venv\Scripts\activate  # Windows
pip install -e ".[dev,web]"
pre-commit install

# Verify setup
pytest tests/unit/ -x -q
ruff check src/ tests/
```

## Daily Development

```bash
# Before coding
git pull origin main
pre-commit run --all-files

# Code quality checks
ruff check src/ tests/ --fix
ruff format src/ tests/

# Run tests
pytest tests/unit/ -v
pytest tests/integration/ -v

# Before committing
pytest tests/unit/ -x -q
```

## CI/CD Pipeline

- **Triggers**: Push to `main`/`develop`, PRs to `main`
- **Matrix**: Python 3.11, 3.12 on Ubuntu
- **Duration**: ~2-5 minutes
- **Coverage**: Auto-updates README badge

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
- **Test Coverage**: >95%

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
- **Cerebro**: Full Backtrader compatibility including `preload=True`
- **Testing**: 19,200+ test lines vs 7,400 source lines
- **Quality**: Comprehensive linting, formatting, security scanning