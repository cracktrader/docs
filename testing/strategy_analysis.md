# Testing Strategy

How we validate Cracktrader reliably and efficiently across environments.

Test Tiers

- Unit: Isolated modules with fakes/mocks to validate contracts
- Integration: Cross‑component behavior using a fake exchange
- End‑to‑End: Optional sandbox/live runs for final validation

Environments

- Mock/fake exchange by default — deterministic, fast, and free
- Sandbox for selected critical paths — realistic but network‑dependent
- Live opt‑in only — used sparingly before releases

Execution Profiles

- Local development: unit and integration (mock only)
- Pre‑release: add critical sandbox tests with `--sandbox`
- Release validation: opt‑in live tests with `--allow-real-orders`

Performance Testing

- Benchmark core paths on mock data to avoid network noise
- Track latency targets separately for mock vs sandbox where relevant

Quality Gates

- Deterministic by default (no network flakiness)
- Clear, actionable failure messages
- Shared fixtures and helpers to avoid duplication

Quick Commands

```bash
# Unit + integration
make test

# Integration (mock)
make test-integration

# Sandbox (selected tests)
pytest tests/integration/ --sandbox
pytest tests/e2e/ --sandbox
```
