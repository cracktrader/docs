# Research Samplers (Walk-Forward, Purged CV, Embargo)

Source repository: [`cracktrader-lab`](https://github.com/cracktrader/cracktrader-lab).

`research.dsl_evaluator.samplers` provides time-series-safe sampling helpers for research validation.

## Supported samplers

- `walk_forward_windows`: rolling train/test windows with strict train-before-test ordering.
- `purged_kfold_windows`: k-fold CV where train windows are purged around the test window.
- `embargo`: implemented via `PurgedCVConfig.embargo` to drop post-test bars from training.

## Leakage constraints

For each fold:

- Test window is contiguous and never included in training.
- `purge` removes bars immediately before test start from training.
- `embargo` removes bars immediately after test end from training.
- Train ranges are returned as disjoint `[start, end)` index intervals.

## API examples

```python
from research.dsl_evaluator.samplers import (
    PurgedCVConfig,
    WalkForwardConfig,
    purged_kfold_windows,
    walk_forward_windows,
)

wf = walk_forward_windows(1000, WalkForwardConfig(train_size=400, test_size=100, step_size=100))
cv = purged_kfold_windows(1000, PurgedCVConfig(n_splits=5, purge=10, embargo=20))
```

## Fold metric aggregation

Use `aggregate_fold_metrics` to average metrics across fold/window outputs:

```python
from research.dsl_evaluator.samplers import aggregate_fold_metrics

summary = aggregate_fold_metrics(
    [
        {"sampler": "walk_forward", "metrics": {"sharpe": 1.2, "return": 0.08}},
        {"sampler": "walk_forward", "metrics": {"sharpe": 1.0, "return": 0.06}},
    ],
    metric_keys=["sharpe", "return"],
)
```
