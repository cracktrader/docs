# Rust Core PR-1 Layout (Scaffold + Parity Harness)

Date: 2026-02-19  
Status: Draft for approval  
Branch: `feat/rust-core-pr1-scaffold`

Related guide:

- `docs/plans/rust_migration_guide.md`

## Progress Snapshot (2026-02-19)

- Implemented:
  - backend selection helpers (`python`/`rust`)
  - `rust_bridge.build_core(...)` and Rust availability checks
  - runtime wiring in `CerebroEngineRuntime` for backend selection
  - Rust workspace/crate scaffold (`rust/cracktrader_engine`) with PyO3 module shape
  - unit tests for backend selection/bridge behavior
  - contract parity harness scaffold and first fixture (auto-skips when Rust module is unavailable)
- Pending:
  - install/load compiled Rust extension in local dev + CI so parity contracts run unskipped
  - expand golden fixture matrix for broader parity coverage

## PR-1 Objective

Create the minimum Rust integration scaffold so Cracktrader can run with:

- `engine_backend="python"` (current default behavior)
- `engine_backend="rust"` (delegates deterministic core transitions to Rust)

without changing user-facing APIs (`ct.CracktraderEngine`, factories/sessions).

`ct.Cerebro` remains a supported compatibility surface in PR-1, but it is not
the primary migration target.

This PR should **not** port full strategy runtime, feeds, or broker adapters. It only introduces the Rust core boundary and parity harness.

## Why This Slice First

It targets the most stable seam already present in the repo:

- `src/cracktrader/engine/core.py`
- `src/cracktrader/engine/sequencer.py`
- `src/cracktrader/engine/events.py`
- `src/cracktrader/engine/intents.py`
- `src/cracktrader/engine/results.py`
- `src/cracktrader/engine/invariants.py`

These modules are already explicitly decoupled and exercised by engine unit/contracts tests.

## Non-Goals (PR-1)

- No rewrite of `engine_runtime.py` adapters.
- No migration of Backtrader classes to Rust.
- No direct behavior changes to `ct.Cerebro` compatibility mode.
- No packaging/release changes beyond local/dev build hooks required for CI.
- No performance optimization pass yet.

## Exact File Layout

### New Rust workspace files

1. `rust/Cargo.toml`
2. `rust/cracktrader_engine/Cargo.toml`
3. `rust/cracktrader_engine/src/lib.rs`
4. `rust/cracktrader_engine/src/domain.rs`
5. `rust/cracktrader_engine/src/events.rs`
6. `rust/cracktrader_engine/src/intents.rs`
7. `rust/cracktrader_engine/src/results.rs`
8. `rust/cracktrader_engine/src/core.rs`
9. `rust/cracktrader_engine/src/error.rs`
10. `rust/cracktrader_engine/src/python.rs` (PyO3 binding surface)

### New Python bridge files

1. `src/cracktrader/engine/rust_bridge.py`
2. `src/cracktrader/engine/backends.py`

### Existing files to change

1. `src/cracktrader/engine/runner.py`
2. `src/cracktrader/engine/__init__.py`
3. `src/cracktrader/engine/replay.py`
4. `src/cracktrader/engine_runtime.py`
5. `pyproject.toml` (dev/build config only; no API changes)

### New tests

1. `tests/unit/engine/test_rust_bridge_smoke.py`
2. `tests/unit/engine/test_runner_backend_selection.py`
3. `tests/contracts/engine/test_python_rust_core_parity.py`
4. `tests/contracts/engine/fixtures/` (golden event fixtures for parity)

## Proposed Public/Internal Interfaces

### Python: backend selection

- Add optional runtime kwarg:
  - `engine_backend: Literal["python", "rust"] = "python"`
- Thread this into runtime internals only.
- Keep all existing run signatures backward-compatible (additive only).

### Python: bridge contract (`rust_bridge.py`)

- `is_rust_available() -> bool`
- `build_core(initial_cash: float, backend: str) -> CoreProtocol`
- `CoreProtocol` methods:
  - `apply_event(event: EngineEvent) -> TransitionResult`
  - `process_intent(intent: EngineIntent) -> TransitionResult`
  - `get_cash() -> float`
  - `get_positions() -> dict[str, float]`
  - `snapshot() -> dict[str, Any]`
  - `reset() -> None`

### Rust: PyO3 module

- Expose class `RustDeterministicCore` mirroring Python `DeterministicCore`.
- All ingress/egress remains JSON/dict compatible for PR-1 simplicity.
- Error conversion: Rust domain errors -> Python `ValueError`/`RuntimeError`.

## Implementation Steps (Atomic)

1. Add Rust crate skeleton + compile gate.
2. Implement Rust domain enums and result structs equivalent to Python engine types.
3. Implement Rust `DeterministicCore` with:
   - order lifecycle transitions
   - fill handling
   - cash/position mutation
4. Add PyO3 bindings exposing `RustDeterministicCore`.
5. Add Python bridge selecting Python or Rust core backend.
6. Update `EngineRunner` to accept injected core protocol unchanged.
7. Add backend toggle path in `CerebroEngineRuntime` (`engine_backend` kwarg).
8. Add parity contract fixtures comparing:
   - transition trace
   - final cash
   - final positions
   - order terminal states
9. Add replay parity check path using existing `engine/replay.py`.

## Test and Gate Plan

Required green checks for PR-1:

1. `python -m pytest -q tests/unit/engine`
2. `python -m pytest -q tests/contracts/engine`
3. `python -m pytest -q tests/contracts/test_fees.py tests/contracts/test_order_lifecycle.py tests/contracts/test_order_scenarios.py`
4. `python examples/basics/hello_engine.py`
5. `python examples/basics/session_quickstart.py`
6. `python examples/basics/shared_store.py`

Parity assertions must include:

- Python core and Rust core generate equivalent `TransitionResult` sequences for shared fixtures.
- No invariant regression in strict mode for either backend on same fixture stream.

## Risk Controls

1. Backend fallback:
   - If Rust module is unavailable, `engine_backend="rust"` raises clear error.
   - `engine_backend="python"` remains unaffected.
2. Isolation:
   - Runtime adapters stay Python-owned in PR-1.
3. Determinism:
   - Golden fixtures include duplicate order reports and partial fill flows.

## Review Checklist

1. No breaking API changes to public entry points.
2. Rust backend is strictly opt-in.
3. Parity suite exists and fails loudly on divergence.
4. Existing example gates still pass.
5. Documentation updated:
   - this plan file
   - short note in `docs/plans/rust_engine_architecture.md` pointing to PR-1 scope.

## Explicit Approval Request

Before implementation, approve:

1. Branch name and PR title:
   - Branch: `feat/rust-core-pr1-scaffold`
   - PR: `engine: add rust deterministic-core scaffold with python parity harness`
2. Backend flag name:
   - `engine_backend` (`python`/`rust`)
3. PR-1 scope freeze:
   - no adapter rewrite, no strategy API rewrite, no packaging change beyond build hooks
