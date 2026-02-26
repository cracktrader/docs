# Checkpoint Schema (v1)

`cracktrader.engine.checkpoint` defines the canonical runtime checkpoint payload used for recovery.

## Versioning

- Current version: `schema_version = 1`
- Backward compatibility: explicit migrations only (currently `v0 -> v1`)
- Forward compatibility: newer unknown versions are rejected until a migration is added

## Required Fields

- `schema_version: int`
- `created_at: float` (Unix timestamp seconds)
- `run_id: str`
- `mode: str`
- `config_hash: str`
- `data_hash: str`
- `engine_state: object`
- `broker_state: object`
- `feed_cursors: object`
- `risk_state: object`
- `strategy_state: object`
- `cursor: object`

## Optional Fields

- `seed: int | null`
- `metadata: object` (defaults to `{}`)

## Core APIs

- `CheckpointEnvelope.from_dict(...)` / `.from_json(...)`
- `CheckpointEnvelope.to_dict()` / `.to_json()`
- `validate_checkpoint_payload(...)`
- `migrate_checkpoint_payload(...)`
- `build_checkpoint_from_runtime(...)`

## Notes

- Validation is strict and rejects corrupt or shape-incompatible payloads.
- Migration is conservative by design to keep recovery deterministic.
