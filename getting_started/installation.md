# Installation & Setup

## Requirements

- **Python 3.11+** (Python 3.12 recommended)
- **Git** for version control
- **Make** (optional, for development workflow)

## Quick Installation

### Basic Installation

```bash
# Install from PyPI (when published)
pip install cracktrader

# Or install from GitHub
pip install git+https://github.com/your-username/cracktrader.git
```

### Development Installation

```bash
# Clone the repository
git clone https://github.com/your-username/cracktrader.git
cd cracktrader

# Development setup (recommended)
make setup

# Or manual setup
pip install -e ".[dev,web,docs]"
pre-commit install
```

## Installation Options

Cracktrader provides several installation profiles for different use cases:

### Core Trading (`pip install cracktrader`)
- CCXT exchange integration
- Backtrader compatibility
- Basic data feeds and brokers

### Web Interface (`pip install "cracktrader[web]"`)
- FastAPI REST API server
- WebSocket real-time data streams
- React dashboard frontend

### Documentation (`pip install "cracktrader[docs]"`)
- MkDocs-Material documentation system
- API reference generation
- Example notebooks

### Development (`pip install "cracktrader[dev]"`)
- Testing framework (pytest, coverage)
- Code quality tools (ruff, black, mypy)
- Pre-commit hooks

### Performance Testing (`pip install "cracktrader[performance]"`)
- Benchmark suite (pytest-benchmark)
- Memory profiling
- Airspeed Velocity integration

## Verification

Test your installation:

```bash
# Test basic functionality
python -c "import cracktrader; print('✅ Cracktrader installed successfully')"

# Run example strategy
python examples/basic_strategy.py

# Check development tools (if installed)
make test
```

## Configuration

### API Keys Setup

For live trading, configure exchange API keys:

```bash
# Create configuration directory
mkdir -p ~/.config/cracktrader

# Create config file
cat > ~/.config/cracktrader/config.yaml << EOF
exchanges:
  binance:
    sandbox:
      apiKey: "your_sandbox_api_key"
      secret: "your_sandbox_secret"
      sandbox: true
    live:
      apiKey: "your_live_api_key"  
      secret: "your_live_secret"
      sandbox: false
EOF

# Secure the config file
chmod 600 ~/.config/cracktrader/config.yaml
```

### Environment Variables

Alternatively, use environment variables:

```bash
export CRACKTRADER_BINANCE_API_KEY="your_api_key"
export CRACKTRADER_BINANCE_SECRET="your_secret"
export CRACKTRADER_SANDBOX=true  # Use sandbox by default
```

## Docker Installation

Run Cracktrader in a container:

```bash
# Build image
docker build -t cracktrader .

# Run with volume for strategies
docker run -v $(pwd)/strategies:/app/strategies cracktrader

# Run with web interface
docker run -p 8000:8000 cracktrader --enable-web
```

## Troubleshooting

### Common Issues

**Import errors:**
```bash
# Ensure you're in the right Python environment
python -c "import sys; print(sys.executable)"

# Reinstall in development mode
pip install -e .
```

**Permission errors:**
```bash
# On macOS/Linux, you might need:
sudo pip install cracktrader

# Better: use virtual environment
python -m venv venv
source venv/bin/activate  # or `venv\Scripts\activate` on Windows
pip install cracktrader
```

**Exchange connection issues:**
```bash
# Test exchange connectivity
python -c "
import ccxt
exchange = ccxt.binance({'sandbox': True})
print(exchange.fetch_ticker('BTC/USDT'))
"
```

### Getting Help

- **Documentation**: [Full documentation](https://your-domain.com/cracktrader-docs)
- **Examples**: Check the `examples/` directory
- **Issues**: [GitHub Issues](https://github.com/your-username/cracktrader/issues)
- **API Reference**: Run `make docs-serve` for local API docs

## Next Steps

- [**Quick Start Guide**](quickstart.md) - Build your first strategy
- [**Configuration**](configuration.md) - Advanced setup options  
- [**Core Concepts**](../core_concepts/architecture.md) - Understanding the framework