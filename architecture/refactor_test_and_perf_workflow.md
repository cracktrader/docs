# Refactor Test and Performance Workflow

This workflow defines the minimum checks to run while executing architecture refactor slices.

## Test Workflow

1. Run targeted tests for the files and behavior touched in the slice.
2. Run adjacent contract tests when touching shared broker/store/state boundaries.
3. Run broader suites only when shared behavior was changed materially.

### Recommended Commands

```powershell
# Targeted contract tests for broker/order behavior
pytest -q tests/contracts/test_order_lifecycle.py
pytest -q tests/contracts/test_order_scenarios.py
pytest -q tests/contracts/test_broker_contracts.py

# Account and reliability checks when state/execution semantics change
pytest -q tests/contracts/test_balances.py
pytest -q tests/contracts/test_reliability.py

# Contract scaffold sanity
pytest -q tests/contracts/test_execution_policy_contracts.py
pytest -q tests/contracts/adapters/test_exchange_adapter_contract.py
pytest -q tests/contracts/state/test_order_state_contracts.py
pytest -q tests/contracts/state/test_account_state_contracts.py
```

## Performance Workflow

Run perf smoke checks whenever a hot path is touched (`engine`, `feeds`, `streaming`, broker update loops).

### Baseline / Compare Commands

```powershell
# Engine backend comparison report
python scripts/benchmark_engine_backends.py

# Aggregate benchmark automation runner
python performance/automation/run_benchmarks.py
```

### Optional Instrumentation

```powershell
$env:CRACKTRADER_ENGINE_PERF_ENABLED = "1"
$env:CRACKTRADER_ENGINE_PERF_JSONL = "performance/reports/engine_perf.jsonl"
```

If the env vars are enabled, capture before/after output paths in slice notes.

## Slice Rule

- If a hot path changed, include before/after perf evidence or explicitly document why perf was not run.
- Do not claim performance gains without measurements.
