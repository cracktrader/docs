# CrackTrader Documentation

Public documentation for CrackTrader - A cryptocurrency trading framework.

## View Documentation

**Live Documentation**: https://cracktrader.github.io/docs/

## Local Development

### Setup
```bash
# Clone repository
git clone https://github.com/cracktrader/cracktrader-docs.git
cd cracktrader-docs

# Install dependencies
pip install -r requirements.txt
```

### Serve Locally
```bash
# Serve with live reload
mkdocs serve

# Open http://localhost:8000
```

### Build Static Site
```bash
# Build for production
mkdocs build

# Output in site/ directory
```

## Deployment

Documentation is automatically deployed to GitHub Pages when changes are pushed to `main` branch.

### Manual Deployment
```bash
# Deploy to gh-pages branch
mkdocs gh-deploy
```

## Repository Structure

```
cracktrader-docs/
├── docs/                  # Documentation source
│   ├── index.md          # Homepage
│   ├── getting_started/  # Setup guides
│   ├── core_concepts/    # Architecture docs
│   ├── performance/      # Performance guides
│   ├── advanced/         # Advanced topics
│   ├── examples/         # Code examples
│   └── reference/        # API reference
├── mkdocs.yml            # MkDocs configuration
├── requirements.txt      # Python dependencies
└── .github/workflows/    # GitHub Pages deployment
```

## Sync with Main Repository

This documentation is maintained in the main CrackTrader repository and synced here using git subtree:

```bash
# From main repo
git subtree push --prefix=docs docs-public main
```

## Contributing

Documentation improvements should be made in the main CrackTrader repository and will be synced here.

## License

MIT License - see main repository for details.
