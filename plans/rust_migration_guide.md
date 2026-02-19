# Rust Migration Guide (Python Strategies, Rust Core)

Date: 2026-02-19  
Status: Working migration guide  
Audience: Cracktrader maintainers and advanced contributors

## 1) Scope and Intent

This guide defines how to migrate Cracktrader toward a Rust-owned deterministic core **without** rewriting strategy authoring in Rust.

Non-negotiable constraint:

- Strategies remain Python-first (`ct.Strategy`, existing `next()`/compat flows).
- Rust owns transition-heavy internals where correctness/performance benefit most.

This document complements:

- `docs/plans/rust_engine_architecture.md` (target architecture)
- `docs/plans/python_engine_execution_spec.md` (runtime stage contract)
- `docs/plans/rust_pr1_scaffold_plan.md` (current scaffold status)

## 2) What Stays Python vs Moves to Rust

### Stays Python

- Strategy definition and callback UX.
- Session/factory APIs (`ct.exchange`, `ct.Store`, `ct.Feed`, `ct.Broker`).
- Adapter edges for Backtrader compatibility and exchange integration glue.
- User-facing examples and ecosystem extensions.

### Moves to Rust (incrementally)

- Deterministic transition core (order lifecycle, fill accounting, cash/positions).
- Sequencing/ordering primitives where determinism is critical.
- Potentially hot-path validation and invariant checks.
- Replay-validated state evolution kernel.

### Rationale

This split preserves developer ergonomics while moving the most bug-prone/high-throughput semantics into a typed runtime.

## 3) Current State (As of 2026-02-19)

Implemented in PR-1 scaffolding:

- `engine_backend` selection path (`python`/`rust`) in runtime.
- Rust bridge and optional module loading (`cracktrader_rust`).
- Rust crate scaffold under `rust/cracktrader_engine`.
- Initial parity contract harness in `tests/contracts/engine/test_python_rust_core_parity.py`.
- Local installer helper: `scripts/install_rust_backend.py`.

## 4) Migration Principles

1. Strangler pattern only: new Rust core sits behind stable Python interfaces.
2. No strategy rewrite requirement for users.
3. Test-first with parity gates before behavior cutovers.
4. Backward-compatible public API changes only.
5. Rust adoption via opt-in flags first, default changes only after sustained parity.

## 5) Phased Migration Plan

## Phase A: Core Parity Expansion (Immediate)

Goal: prove semantic equivalence of Python and Rust cores across edge cases.

Work:

- Expand contract fixtures for:
  - partial fills
  - duplicate order reports
  - cancel-after-partial
  - out-of-order report handling
- Assert parity on:
  - transition traces
  - final cash/value
  - final positions
  - terminal order states

Exit:

- Expanded fixtures pass for both backends locally.

## Phase B: Core Completeness

Goal: close remaining behavior gaps in Rust core vs Python deterministic core.

Work:

- Align transition legality semantics exactly.
- Align idempotency handling and warning/diagnostic behavior.
- Ensure same replay outcome on canonical fixtures.

Exit:

- No known parity deltas in core contracts.

## Phase C: Runtime Hardening (Still Python strategy layer)

Goal: run real engine paths with Rust backend for backtest/sim while preserving UX.

Work:

- Increase adapter-level coverage with `engine_backend="rust"`.
- Ensure async/sync/vectorized flows remain behaviorally consistent.
- Address performance regressions at Python<->Rust boundary.

Exit:

- Engine contracts and selected integration tests green on both backends.

## Phase D: Rust-Preferred Core (Not strategy migration)

Goal: make Rust core the recommended backend when parity + perf are stable.

Work:

- Keep Python fallback.
- Keep strategy layer Pythonic.
- Publish migration notes and troubleshooting runbook.

Exit:

- Rust backend demonstrated stable, measurable gains on agreed benchmark profiles.

## 6) Parity Gate Policy

Minimum parity suite:

- `tests/unit/engine`
- `tests/contracts/engine`
- Deterministic fixture replay checks

Promotion criteria before broad rollout:

1. Core parity: 100% on contract fixtures (no transition mismatch).
2. Determinism: replay equivalence holds under repeated runs.
3. Operational: no regressions in offline example gate.

## 7) Performance Benchmarking Plan (Python vs Rust)

The migration must be evidence-driven.

Use `scripts/benchmark_engine_backends.py` for apples-to-apples backend comparisons over identical synthetic data and strategy behavior.

### What to measure

- Mean/median/p95 run duration per backend.
- Speedup ratio (`python_mean_ms / rust_mean_ms`).
- Cash/value parity during benchmark runs (sanity invariant).

### Benchmark command

```powershell
.\.venv\Scripts\python.exe scripts/benchmark_engine_backends.py --iterations 10 --warmup 3 --bars 128 --json-out performance/reports/engine_backend_compare.json
```

### Baseline compare command

```powershell
.\.venv\Scripts\python.exe scripts/benchmark_engine_backends.py --iterations 10 --warmup 3 --bars 128 --json-out performance/reports/engine_backend_compare.json --baseline-json performance/baselines/engine_backend_compare_baseline.json --max-slowdown-ratio 1.10
```

Baseline snapshots are stored in `performance/baselines/`.

### Baseline update command

```powershell
.\.venv\Scripts\python.exe scripts/benchmark_engine_backends.py --iterations 10 --warmup 3 --bars 128 --json-out performance/reports/engine_backend_compare.json --baseline-json performance/baselines/engine_backend_compare_baseline.json --update-baseline --force-baseline-overwrite
```

If Rust extension is unavailable, default run executes Python-only. To include Rust:

```powershell
.\.venv\Scripts\python.exe scripts/install_rust_backend.py
```

### Benchmark acceptance guidance

- Report both absolute latency and speedup ratio.
- Require parity (cash/value alignment) before treating speedup as valid.
- Use at least 5 measured iterations and warmup >= 1.
- Fail comparison if Rust mean latency regresses beyond `--max-slowdown-ratio` against baseline.

## 8) Developer Workflow

1. Implement feature/change behind parity tests first.
2. Run backend benchmark script and capture JSON report.
3. Compare with previous baseline; note deltas in PR summary.
4. If parity breaks, fix parity first before optimizing throughput.

### One-shot parity gate

Run the full Rust parity gate with one command:

```powershell
.\.venv\Scripts\python.exe scripts/run_rust_parity_gate.py
```

Useful options:
- `--skip-install` (when the Rust wheel is already installed)
- `--skip-benchmark` (for quick local test-only validation)

## 9) Risk Register

1. Python<->Rust boundary overhead hides gains.
- Mitigation: batch crossings; keep transition-heavy code Rust-owned.

2. Behavior drift between cores.
- Mitigation: golden fixtures + replay parity as merge gates.

3. Adapter complexity masks core correctness.
- Mitigation: keep adapter logic thin; test core semantics directly.

4. Tooling friction (wheel/build/install).
- Mitigation: `scripts/install_rust_backend.py` as standard dev entrypoint.

5. Logging drift obscures parity debugging.
- Mitigation: keep shared log event categories and level intent parity (`DEBUG`/`INFO`/`WARNING`/`ERROR`) across Python and Rust, while allowing implementation-specific message text.

## 10) FAQ

### Do strategy authors need to rewrite in Rust?

No. Strategy authoring stays in Python.

### Are we removing Backtrader compatibility now?

No. Compatibility remains during migration; core ownership shifts behind adapters.

### Should every module move to Rust?

No. Only modules where correctness/performance justify boundary costs.

## 11) Recommended Next Rust Tasks

1. Expand parity fixture matrix (partial/duplicate/cancel race/out-of-order).
2. Add differential/fuzz parity generator for event streams.
3. Add benchmark baselines to `performance/reports` for backend comparisons.
4. Tune Rust core diagnostics to match Python core semantics where required.
