Cracktrader Testing Guidelines

These are the structural and naming conventions for Cracktrader’s test suite.

Goal
- Idempotent (stable names and contracts)
- Behavioral (test what users rely on)
- Minimal (no duplication, no testing internals directly)
- Self‑documenting (intent clear from names and layout)

Core principles
- Structure follows interface, not implementation
- Organize by public behavior (broker, order, feed, store)
- Merge submodules into their owning interface
- Prefer user‑visible flows over internal state poking

Folder layout (indicative)
```
tests/
  unit/
    broker/
    order/
    feed/
    store/
    commission/
    config/
  integration/
  e2e/
```

Naming
- Files: test_<behavior>.py
- Classes: Test<InterfaceOrBehavior>
- Functions: test_<scenario>

Docstring metadata (recommended)
Each test function may include:
"""
Target: broker|order|feed|store|comm_info|factory|system
Type: unit|integration|e2e
"""

Execution profiles
- Default: unit + integration (mock/fake exchange only)
- Sandbox: add critical integration/e2e with `--sandbox`
- Live: opt‑in final checks with `--allow-real-orders`

Quality gates
- Deterministic by default (no network flakiness)
- Clear, actionable failure messages
- Shared fixtures and helpers to avoid duplication
