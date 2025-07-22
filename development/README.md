# Development Documentation

This directory contains comprehensive documentation for developing and contributing to Cracktrader.

## Getting Started

- 📚 **[Development Workflow](workflow.md)** - Complete guide to development process, CI/CD, and tools
- ⚡ **[Quick Reference](quick-reference.md)** - Essential commands and information
- 🧪 **[Testing Guide](../testing/known_gaps.md)** - Current testing status and gaps

## Key Areas

### Development Process
- Modern Python development workflow
- Automated quality assurance with pre-commit hooks
- Comprehensive CI/CD pipeline with GitHub Actions
- Security scanning and performance monitoring

### Code Quality
- **Ruff**: Fast linting and formatting (55,000+ ops/sec)
- **Black**: Code formatting backup
- **Bandit**: Security vulnerability scanning
- **Type checking**: MyPy integration (optional)

### Testing Strategy
- **Unit Tests**: Fast, isolated component testing
- **Integration Tests**: End-to-end system behavior
- **Performance Tests**: High-frequency data processing validation
- **Coverage**: >95% with automated reporting

### CI/CD Pipeline Features
- ✅ Matrix testing (Python 3.11, 3.12)
- ✅ Automated dependency caching
- ✅ Parallel test execution
- ✅ Security scanning
- ✅ Build verification
- ✅ Coverage badge updates
- ✅ Artifact collection

## Architecture Overview

Cracktrader is a high-performance cryptocurrency trading framework:

- **7,400 lines** of source code
- **19,200+ lines** of comprehensive tests
- **400+ exchanges** supported via CCXT
- **Sub-minute timeframes** (1s, 10s, 30s) with tick reordering
- **Full Backtrader compatibility** including analyzers and optimization

## Performance Benchmarks

- **Data Processing**: 55,000+ candles/second
- **Tick Reordering**: 53,000+ candles/second with chronological sorting
- **Memory Efficiency**: <1MB for 1000 high-frequency candles
- **Build Time**: ~2-5 minutes for full CI/CD pipeline

## Contributing

1. Follow the [Development Workflow](workflow.md)
2. Ensure all tests pass and code quality checks succeed
3. Add tests for new functionality
4. Update documentation as needed
5. Submit pull requests against `main` branch

The development workflow is designed to catch issues early and maintain production-ready code quality through automated testing and quality assurance.