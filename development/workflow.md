# Development Workflow Guide

This guide explains the comprehensive development workflow for Cracktrader, including CI/CD automation, code quality tools, and best practices.

## Table of Contents

- [Overview](#overview)
- [Development Environment Setup](#development-environment-setup)
- [Code Quality Tools](#code-quality-tools)
- [Testing Strategy](#testing-strategy)
- [CI/CD Pipeline](#cicd-pipeline)
- [Pre-commit Hooks](#pre-commit-hooks)
- [Release Process](#release-process)
- [Performance Monitoring](#performance-monitoring)

## Overview

Cracktrader uses a modern development workflow with automated quality checks, comprehensive testing, and continuous integration. The system is designed to catch issues early and maintain high code quality.

### Key Technologies

- **Python 3.11+**: Primary language
- **Ruff**: Fast Python linter and formatter
- **Black**: Code formatting
- **pytest**: Testing framework
- **Bandit**: Security scanning
- **GitHub Actions**: CI/CD automation
- **Pre-commit**: Git hooks for quality checks

## Development Environment Setup

### 1. Initial Setup

```bash
# Clone repository
git clone <repository-url>
cd cracktrader

# Create virtual environment
python -m venv .venv
source .venv/bin/activate  # Linux/Mac
# or
.venv\Scripts\activate  # Windows

# Install dependencies
pip install -e ".[dev,web]"
```

### 2. Install Pre-commit Hooks

```bash
# Install pre-commit hooks
pre-commit install

# Test hooks
pre-commit run --all-files
```

### 3. Verify Installation

```bash
# Run tests
pytest tests/unit/ -v

# Check code quality
ruff check src/ tests/
ruff format --check src/ tests/

# Run security scan
bandit -r src/ -f json -o bandit-results.json
```

## Code Quality Tools

### Ruff (Primary Linter and Formatter)

Ruff is our primary code quality tool, replacing flake8, isort, and other tools with a single fast implementation.

**Configuration**: `pyproject.toml`

```toml
[tool.ruff]
line-length = 100
target-version = "py311"

[tool.ruff.lint]
select = ["E", "F", "W", "C90", "I", "N", "UP", "YTT", "S", "B", "A", "COM", "DTZ", "EM", "G", "INP", "PIE", "T20", "PT", "Q", "RSE", "RET", "SIM", "TID", "ARG", "ERA", "PD", "PGH", "PL", "TRY", "NPY", "PERF", "RUF"]
ignore = ["S101", "PLR0913", "PLR0912", "PLR0915"]
```

**Commands**:
```bash
# Check code
ruff check src/ tests/

# Auto-fix issues
ruff check src/ tests/ --fix

# Format code
ruff format src/ tests/

# Check formatting
ruff format --check src/ tests/
```

### Black (Backup Formatter)

While Ruff handles most formatting, Black is included for compatibility.

```bash
# Format with Black
black src/ tests/

# Check formatting
black --check src/ tests/
```

### Bandit (Security Scanner)

Bandit scans for common security issues in Python code.

```bash
# Run security scan
bandit -r src/ -f json -o bandit-results.json

# Run with configuration
bandit -c pyproject.toml -r src/
```

## Testing Strategy

### Test Structure

```
tests/
├── unit/           # Unit tests for individual components
├── integration/    # Integration tests for component interaction
└── helpers/        # Test utilities and fixtures
```

### Test Categories

1. **Unit Tests**: Fast, isolated tests for individual functions/classes
2. **Integration Tests**: Test component interactions and system behavior
3. **Performance Tests**: Validate performance characteristics
4. **Security Tests**: Embedded in regular tests, plus Bandit scans

### Running Tests

```bash
# Run all tests
pytest

# Run specific test categories
pytest tests/unit/ -v
pytest tests/integration/ -v

# Run with coverage
pytest tests/unit/ --cov=src/cracktrader --cov-report=html

# Run performance tests
pytest tests/unit/feed/test_sub_minute_timeframes.py -v -s

# Run specific test
pytest tests/integration/test_cerebro_compatibility.py::TestCerebroCompatibility::test_cerebro_run_with_preload_true -v
```

### Test Configuration

**Configuration**: `pyproject.toml`

```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
python_files = ["test_*.py"]
python_classes = ["Test*"]
python_functions = ["test_*"]
addopts = "-v --tb=short"
timeout = 30
```

## CI/CD Pipeline

### GitHub Actions Workflow

The CI/CD pipeline runs automatically on push and pull requests to `main` and `develop` branches.

**File**: `.github/workflows/ci.yml`

### Pipeline Stages

#### 1. **Test Matrix**
- Python 3.11 and 3.12
- Ubuntu latest
- Parallel execution for speed

#### 2. **Dependency Caching**
- Caches pip dependencies
- Speeds up subsequent runs
- Cache key based on requirements files

#### 3. **Code Quality Checks**
```yaml
- name: Lint with ruff
  run: |
    ruff check src/ tests/ --output-format=github
    ruff format --check src/ tests/
```

#### 4. **Type Checking** (Optional)
```yaml
- name: Type check with mypy
  continue-on-error: true
  run: |
    pip install mypy || echo "mypy not available, skipping"
    mypy src/ || echo "Type checking completed with issues"
```

#### 5. **Unit Tests with Coverage**
```yaml
- name: Run unit tests with coverage
  run: |
    pytest tests/unit/ \
      --cov=src/cracktrader \
      --cov-report=xml \
      --cov-report=html \
      --cov-report=term-missing \
      --junitxml=test-results-unit.xml \
      -v
```

#### 6. **Integration Tests**
```yaml
- name: Run integration tests
  run: |
    pytest tests/integration/ \
      --junitxml=test-results-integration.xml \
      -v
```

#### 7. **Security Scanning**
- Separate job for security analysis
- Uses Bandit for Python security issues
- Uploads results as artifacts

#### 8. **Build Verification**
- Builds Python package
- Verifies package integrity with twine
- Uploads artifacts

#### 9. **Documentation Build**
- Builds documentation if MkDocs is configured
- Uploads documentation artifacts

#### 10. **Coverage Badge Update**
- Updates README coverage badge automatically
- Runs only on main branch
- Uses green/yellow/red color coding

### Workflow Triggers

```yaml
on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]
```

### Artifact Collection

The pipeline automatically collects:
- Test results (JUnit XML)
- Coverage reports (HTML, XML)
- Security scan results (JSON)
- Build artifacts (wheel, sdist)
- Documentation

## Pre-commit Hooks

Pre-commit hooks run automatically before each commit to ensure code quality.

**Configuration**: `.pre-commit-config.yaml`

### Hooks Enabled

1. **Basic Checks**:
   - trailing-whitespace
   - end-of-file-fixer
   - check-yaml
   - check-added-large-files
   - check-merge-conflict
   - debug-statements

2. **Code Formatting**:
   - Black (code formatting)
   - Ruff (linting and formatting)

3. **Security**:
   - Bandit (security scanning)

4. **Testing**:
   - pytest-check (runs unit tests)

### Hook Execution

```bash
# Manual execution
pre-commit run --all-files

# Skip hooks (emergency only)
git commit -m "message" --no-verify

# Update hooks
pre-commit autoupdate
```

## Release Process

### Version Management

1. **Update Version**: Increment version in `src/cracktrader/_version.py`
2. **Update Changelog**: Document changes
3. **Create Git Tag**: `git tag v1.x.x`
4. **Push Tag**: `git push origin v1.x.x`

### Automated Release (Future Enhancement)

The pipeline can be extended to automatically:
- Build and test release candidates
- Publish to PyPI
- Create GitHub releases
- Generate release notes

## Performance Monitoring

### Continuous Performance Testing

- Performance tests run in CI/CD
- Detect performance regressions
- Monitor memory usage and processing speed

### Key Metrics Tracked

1. **Data Processing Performance**:
   - 55,000+ candles/second throughput
   - <1MB memory usage for 1000 candles
   - <1s processing time for high-frequency data

2. **Reordering Performance**:
   - 53,000+ candles/second with reordering
   - Maintains chronological order
   - Configurable buffer sizes

3. **Memory Efficiency**:
   - Queue-based architecture prevents memory leaks
   - Automatic garbage collection
   - Bounded buffer sizes

### Performance Test Examples

```bash
# Run performance tests
pytest tests/unit/feed/test_sub_minute_timeframes.py::TestSubMinuteTimeframes::test_sub_minute_data_processing_performance -v -s

# Run with profiling
pytest tests/unit/feed/test_tick_reordering.py -v -s --profile
```

## Development Best Practices

### Code Quality Guidelines

1. **Follow PEP 8**: Enforced by ruff and black
2. **Write Docstrings**: Document all public functions/classes
3. **Type Hints**: Use type hints where appropriate
4. **Test Coverage**: Maintain >80% test coverage
5. **Security First**: No hardcoded secrets, use bandit scanning

### Git Workflow

1. **Branch Naming**: `feature/description`, `fix/description`, `docs/description`
2. **Commit Messages**: Clear, descriptive messages
3. **Pull Requests**: Required for main branch
4. **Code Review**: All changes reviewed before merge

### Testing Guidelines

1. **Test-Driven Development**: Write tests first when possible
2. **Comprehensive Coverage**: Unit + integration + performance tests
3. **Mock External Dependencies**: Use mocks for exchanges, networks
4. **Performance Regression**: Include performance tests for critical paths

## Troubleshooting

### Common Issues

**1. Pre-commit Hook Failures**
```bash
# Fix formatting issues
ruff format src/ tests/
ruff check src/ tests/ --fix

# Re-run hooks
pre-commit run --all-files
```

**2. Test Failures**
```bash
# Run specific failing test
pytest tests/path/to/test.py::TestClass::test_method -v -s

# Debug with pdb
pytest --pdb tests/path/to/test.py::TestClass::test_method
```

**3. CI/CD Pipeline Issues**
- Check GitHub Actions logs
- Verify dependency versions
- Ensure all tests pass locally first

**4. Performance Issues**
- Run performance tests locally
- Check memory usage patterns
- Review queue sizes and buffer configurations

### Getting Help

1. **Documentation**: Check existing docs in `docs/`
2. **Tests**: Look at test examples for usage patterns
3. **Code Comments**: Comprehensive docstrings throughout codebase
4. **Issues**: Create GitHub issues for bugs or feature requests

## Summary

The Cracktrader development workflow provides:

- ✅ **Automated Quality Assurance**: Pre-commit hooks + CI/CD pipeline
- ✅ **Comprehensive Testing**: Unit, integration, and performance tests  
- ✅ **Security First**: Automated security scanning with Bandit
- ✅ **Performance Monitoring**: Continuous performance regression detection
- ✅ **Developer Experience**: Fast feedback loops and helpful error messages
- ✅ **Production Ready**: 19,200+ lines of tests vs 7,400 lines of source code

This workflow ensures high code quality, catches issues early, and maintains the production-ready status of the Cracktrader cryptocurrency trading framework.