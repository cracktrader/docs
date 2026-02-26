# Recovery Runner

The recovery path is implemented in `cracktrader.engine.recovery.RecoveryRunner`.

## CLI

```bash
cracktrader engine recover --checkpoint /path/checkpoint.json --replay /path/replay.jsonl
```

Optional flags:

- `--start-step N` override checkpoint cursor and replay from `seq_id > N`
- `--strict` fail the command when parity mismatches are found
- `--backend {python,rust}` choose recovery core backend

## Behavior

1. Load and validate checkpoint envelope.
2. Restore core snapshot state.
3. Replay suffix steps from replay log using checkpoint cursor (`step_seq`).
4. Compare recovered state against replay invariants:
   - cash
   - positions
   - data cursor diagnostics (`expected_data_clock`, `expected_data_index`) when present
5. Emit diagnostics (`replayed_steps`, `start_step`, mismatch list, deterministic flag).
