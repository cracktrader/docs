# Installation & Setup

## Requirements

- **Python 3.11+** (Python 3.12 recommended)

## Quick Installation

### Basic Installation

```bash
# Install from GitHub
pip install git+https://github.com/LachlanBridges/cracktrader.git
```

### Development Installation

For developers who want to contribute or modify the code:

```bash
# Clone the repository
git clone https://github.com/LachlanBridges/cracktrader.git
cd cracktrader

# Development setup (recommended)
make setup

# Or manual setup
pip install -e ".[dev,web,docs]"
pre-commit install
```

**Development Requirements:**
- **Git** for version control
- **Make** (optional, for development workflow)

## Installation Options

Cracktrader supports optional extras that can be installed alongside the core package:

```bash
# Core installation (trading only)
pip install git+https://github.com/LachlanBridges/cracktrader.git

# With web interface
pip install "git+https://github.com/LachlanBridges/cracktrader.git[web]"

# With multiple extras
pip install "git+https://github.com/LachlanBridges/cracktrader.git[web,docs]"
```

**Available Extras:**

- **`web`** - FastAPI REST API server, WebSocket streams, React dashboard
- **`docs`** - MkDocs documentation system, API reference generation
- **`dev`** - Testing framework, code quality tools, pre-commit hooks
- **`performance`** - Benchmark suite, memory profiling tools
- **`test`** - Testing dependencies only

## Verification

Test your installation:

```bash
# Test basic functionality
python -c "import cracktrader; print('Cracktrader installed successfully')"

# Test exchange connectivity
python -c "import ccxt; print('CCXT working:', ccxt.exchanges[:5])"
```

**For Development Installation:**
```bash
# Run example strategy (requires cloned repository)
python examples/basic_strategy.py

# Run tests
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

<!-- ## Docker Installation

⚠️ **Docker support is experimental and not fully tested**

Run Cracktrader in a container:

```bash
# Build image
docker build -t cracktrader .

# Run with volume for strategies
docker run -v $(pwd)/strategies:/app/strategies cracktrader

# Run with web interface
docker run -p 8000:8000 cracktrader --enable-web
```
-->

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
python -m venv .venv
source .venv/bin/activate  # or `.venv\Scripts\activate` on Windows
pip install git+https://github.com/LachlanBridges/cracktrader.git
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

- **Documentation**: [https://lachlanbridges.github.io/cracktrader-docs/](https://lachlanbridges.github.io/cracktrader-docs/)
- **Examples**: Browse examples in the [GitHub repository](https://github.com/LachlanBridges/cracktrader/tree/main/examples)
- **Issues**: Report bugs or request features at [GitHub Issues](https://github.com/LachlanBridges/cracktrader/issues)
- **API Reference**: Available in the online documentation or run `make docs-serve` locally for development

## Next Steps

- [**Quickstart Guide**](quickstart.md) - Build your first strategy
- [**Configuration**](configuration.md) - Advanced setup options
- [**Core Concepts**](../core_concepts/architecture.md) - Understanding the framework
