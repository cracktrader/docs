# Documentation Publishing

How to publish CrackTrader documentation to the public repository.

## Overview

CrackTrader uses a **git subtree** approach to publish documentation from the private main repository to a separate public documentation repository at `https://github.com/LachlanBridges/cracktrader-docs`.

## Setup (One-time)

### 1. Add Public Repository Remote
```bash
git remote add docs-public https://github.com/LachlanBridges/cracktrader-docs.git
```

### 2. Create Public Repository on GitHub
- Repository name: `cracktrader-docs`
- Visibility: **Public**
- Don't initialize with README (we'll push content)

### 3. Enable GitHub Pages
- Go to repository Settings → Pages
- Set Source to "GitHub Actions"
- The included workflow will handle deployment

## Publishing Documentation

### Method 1: Use Publish Script (Recommended)
```bash
# From project root
./scripts/publish_docs.sh
```

The script handles:
- ✅ Directory validation
- ✅ Remote existence check
- ✅ Uncommitted changes warning
- ✅ Git subtree push
- ✅ Success confirmation

### Method 2: Manual Git Subtree
```bash
# From project root
git subtree push --prefix=docs docs-public main
```

## Workflow

### Daily Development
1. **Edit Documentation**: Make changes in `docs/` directory
2. **Test Locally**: `mkdocs serve` to preview changes
3. **Commit Changes**: `git add docs/ && git commit -m "Update documentation"`
4. **Publish**: `./scripts/publish_docs.sh`

### What Gets Published
The entire `docs/` directory becomes the root of the public repository:
```
docs/index.md              → index.md
docs/getting_started/       → getting_started/
docs/performance/           → performance/
docs/mkdocs.yml             → mkdocs.yml
docs/.github/workflows/     → .github/workflows/
```

## Automatic Deployment

Once published, GitHub Actions automatically:
1. **Installs** MkDocs and dependencies
2. **Builds** static documentation
3. **Deploys** to GitHub Pages
4. **Makes live** at `https://lachlanbridges.github.io/cracktrader-docs/`

## Troubleshooting

### Permission Denied
```bash
git remote set-url docs-public https://github.com/LachlanBridges/cracktrader-docs.git
```

### Subtree Push Fails
```bash
# Force push (use carefully)
git subtree push --prefix=docs docs-public main --squash
```

### GitHub Pages Not Working
1. Check repository Settings → Pages → Source is "GitHub Actions"
2. Check Actions tab for build failures
3. Verify `docs/.github/workflows/deploy.yml` exists

### Local Build Issues
```bash
# Install docs dependencies
pip install -r requirements-docs.txt

# Test build
mkdocs build --clean
```

## Repository Structure

### Main Repository (Private)
- `/docs/` - Documentation source
- `/scripts/publish_docs.sh` - Publishing script
- `/mkdocs.yml` - MkDocs config (also copied to docs/)
- Source code, tests, etc.

### Public Repository
- Root contains only documentation
- No source code exposure
- Clean, professional presentation
- Automatic deployment via GitHub Actions

## Best Practices

1. **Test Before Publishing**: Always run `mkdocs serve` locally first
2. **Meaningful Commits**: Use descriptive commit messages for docs
3. **Atomic Changes**: Separate documentation commits from code changes
4. **Review Changes**: Check the public site after publishing
5. **Regular Updates**: Keep documentation in sync with code changes

## Integration with Development Workflow

### In CLAUDE.md
Reference this workflow in the main development documentation.

### In CI/CD
Consider adding documentation publishing to release workflows.

### For Contributors
Document this process for team members who update documentation.
