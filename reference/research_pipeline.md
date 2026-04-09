# Research Pipeline: Lab Runs, SQLite Registry, and Governance Ingress

Primary repositories:
- [`cracktrader-lab`](https://github.com/cracktrader/cracktrader-lab): deterministic research authoring and run execution
- [`cracktrader-research-control`](https://github.com/cracktrader/cracktrader-research-control): governance/control-plane ingestion, evidence sealing, and promotion state

This page reflects the current lab-first flow where runs are produced in `cracktrader-lab`, then candidate bundles are ingested into `cracktrader-research-control`.

## Lab Workflow Entry Points

`ctlab` is the primary contributor-facing entrypoint:

- `ctlab init strategy <slug>`
- `ctlab validate <slug>`
- `ctlab run <slug>`
- `ctlab family plan <slug>`
- `ctlab family run <slug>`
- `ctlab compare <slug>`
- `ctlab falsify <slug>`
- `ctlab candidate pack <slug>`

These commands execute and inspect deterministic research runs, then package governance handoff bundles.

## Run Artifact Layout

By default, run artifacts are written under:

- `.cracktrader/runs/<run_id>/`

Each run directory contains:

- `request.json`: validated evaluation request payload
- `report.json`: strategy report with split metrics, stress summaries, and failure capsule
- `metadata.json`: compact run metadata for reproducibility and comparison
- `splits/<split_id>/trades.csv`
- `splits/<split_id>/equity_curve.csv`
- additional stress/falsification artifacts when present

## Run Registry Storage (SQLite-Backed)

`RunRegistry` is now SQLite-backed via:

- `.cracktrader/runs/registry.sqlite3`

Core tables:

- `request_runs`: request hash -> run id mapping
- `run_metadata`: metadata payload rows keyed by run id

`RunRegistry.list_metadata()` reads from SQLite-backed metadata rows and is the canonical way to enumerate runs.

Legacy JSON indexes (`index.json`, `metadata_index.json`) are treated as migration input only. If present, they are imported into SQLite and are no longer the source of truth.

## Candidate Bundle Handoff

`ctlab candidate pack <slug>` writes a governance handoff bundle to:

- `.cracktrader/candidates/<candidate_revision_id>/candidate_bundle.json`

The bundle includes:

- candidate identity and lineage metadata
- run linkage (`run_id`, hashes, summary metrics)
- artifact references and content hashes
- seal request payload for governance ingestion

## Research-Control Ingress and Promotion State

The control-plane ingress path is:

- `POST /candidate-bundles/ingest`

Ingestion materializes governance facts and updates derived promotion state. Common read surfaces include:

- `GET /promotion-state/{subject_type}/{subject_id}`
- `GET /promotion-plans/{subject_type}/{subject_id}/{stage}`
- `GET /pending-experiments/{subject_type}/{subject_id}[/{stage}]`
- `GET /deployable-registry/{family_id}`

This keeps deterministic research generation (`cracktrader-lab`) separated from authoritative governance state (`cracktrader-research-control`).

## Comparison and Validation

Comparison is driven through `ctlab compare` and registry-backed metadata/report reads.

For sampler and leakage-safety details, see [Research Samplers](research_samplers.md).
