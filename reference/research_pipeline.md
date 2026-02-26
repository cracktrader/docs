# Research Pipeline: Run Metadata, Artifacts, and Comparison

The DSL evaluator writes each run under `.cracktrader/runs/<run_id>/` (or `EvaluationOptions.artifacts_root` when provided).

## Stored outputs

- `request.json`: validated evaluation request payload.
- `report.json`: `StrategyReport` with split metrics, stress tests, and failure capsule.
- `metadata.json`: compact run metadata (`RunMetadataSpec`) for indexing and reproducibility.
- `splits/<split_id>/trades.csv`
- `splits/<split_id>/equity_curve.csv`
- `stress/...` artifacts for cost/delay/parameter stress reruns.

## Metadata schema highlights

`RunMetadataSpec` includes:

- `run_id`
- `reproducibility_key` (deterministic hash of request/strategy/dataset hashes)
- `request_hash`, `strategy_hash`, `config_hash`
- `dataset_id`, `dataset_hash`
- `seed` (optional)
- `engine_versions`
- `artifacts` (path registry)
- `metrics` (summary metrics snapshot)

## Registry indexes

- `index.json`: request-hash to run-id cache mapping.
- `metadata_index.json`: queryable run metadata records keyed by `run_id`.

Use `RunRegistry.list_metadata()` to enumerate runs without loading each report.

## Comparison tooling

`research.dsl_evaluator.comparison` provides:

- `compare_reports(left, right, metric="net_return_after_costs")`
- `leaderboard(reports, metric="net_return_after_costs", role="out_of_sample")`
- `compare_registered_runs(registry, run_ids=None, metric=..., role=...)`

Example:

```python
from pathlib import Path

from research.dsl_evaluator.comparison import compare_registered_runs
from research.dsl_evaluator.registry import RunRegistry

registry = RunRegistry(Path(".cracktrader/runs"))
rows = compare_registered_runs(registry)
best = rows[0]
print(best["run_id"], best["metric"])
```

For time-series validation sampler details, see `docs/reference/research_samplers.md`.
