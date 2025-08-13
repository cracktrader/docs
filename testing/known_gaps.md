## Testing Methodology

This page outlines how Cracktrader is tested and what each tier validates.

Test Tiers

- Unit: Isolated modules with fakes/mocks to validate contracts
- Integration: Cross‑component behavior using a fake exchange
- End‑to‑End: Optional sandbox/live runs for final validation

Environments

- Mock/fake exchange by default — deterministic, fast, and free
- Sandbox for selected critical paths — realistic but network‑dependent
- Live opt‑in only — used sparingly before releases

Performance

- Benchmark core paths on mock data to avoid network noise
- Track latency targets separately for mock vs sandbox where relevant

Quality Assurance

- CI runs unit and integration suites on every change
- Fixtures provide stable, reusable setups
- Clear failure messages and reproducible scenarios
