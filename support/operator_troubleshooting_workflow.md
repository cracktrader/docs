# Operator Troubleshooting and SLO Investigation Workflow

This guide defines the standard operator loop for investigating unhealthy
runtimes, rollout regressions, and stale evidence in the current Cracktrader
platform.

Use it with:

- [Web API Reference](../reference/web_api.md)
- [Runtime Map](../architecture/runtime_map.md)
- [On-chain Ops Runbook](onchain_ops_runbook.md)

## What This Workflow Answers

During an incident, answer these questions in order:

1. What is unhealthy right now?
2. What changed recently?
3. Is this a rollout regression, stale input/evidence problem, or runtime fault?
4. What evidence should be inspected next?

The sections below map each question to concrete UI and API surfaces.

## Primary Operator Surfaces

### UI surfaces (current)

- Fleet overview: first-pass health, kill-switch state, and attention queue
- Runtime detail:
  - `Overview`: lifecycle and top-level runtime posture
  - `Routes`: venue/route health and execution-path degradation
  - `Events`: event chronology for change detection
  - `Portfolio` and `Risk`: impact and policy posture
  - `Reports`: latest completed run/report context

### Backend/read-model surfaces (current)

From `cracktrader` Web API:

- `GET /api/v1/health`
- `GET /api/v1/status`
- `GET /api/v1/status/detailed`
- `GET /api/v1/results`
- `WS /api/v1/ws/status`

From `cracktrader-research-control`:

- `GET /promotion-state/{subject_type}/{subject_id}`
- `GET /rollout-state/{deployable_version_id}`

### Planned incident/readiness surfaces

The operator epic (`cracktrader-org#35`) is tracking higher-level incident
summaries for:

- rollout regression timelines
- stale evidence windows
- readiness/SLO summaries per runtime and deployable

Until those land, use the correlation workflow below.

## SLO-First Triage Loop

Treat every incident as an SLO investigation instead of a generic debug session.

### Step 1: Confirm blast radius and urgency

Check:

- fleet attention queue in UI
- `GET /api/v1/health` for service-level checks
- `GET /api/v1/status` for active runtime posture

Record:

- affected runtime IDs
- first-observed timestamp
- current severity (`degraded`, `kill_switch`, or hard failure)

### Step 2: Build a change timeline

Use:

- runtime `Events` tab in UI
- websocket status stream (`WS /api/v1/ws/status`)
- runtime `status/detailed` payload snapshots

Goal:

- identify the first event that materially changed runtime behavior
- separate persistent failure from transient spikes

### Step 3: Classify failure mode

Use this decision split:

- Rollout regression:
  runtime degraded after a deployable/stage transition or rollout event
- Stale evidence/input:
  runtime appears live but evidence/read-model freshness is behind expectations
- Runtime fault:
  runtime health drops independent of rollout changes (route faults, venue issues,
  policy trips, infra faults)

For rollout-linked incidents, inspect control-plane state directly:

- `GET /rollout-state/{deployable_version_id}`
- `GET /promotion-state/{subject_type}/{subject_id}`

### Step 4: Inspect next evidence layer

Pick evidence based on classification:

- Rollout regression:
  rollout-state transitions, promotion verdict context, recent runtime events
- Stale evidence/input:
  latest evidence timestamps, missing bundle updates, stream lag indicators
- Runtime fault:
  route-level health, risk posture, execution/report anomalies

If on-chain routes are involved, run the on-chain checklist from
[On-chain Ops Runbook](onchain_ops_runbook.md).

### Step 5: Decide and record operator action

Use runtime controls only after classification:

- `pause` when behavior is uncertain and containment is needed
- `resume` when checks pass and evidence is current
- `kill` when active risk or policy breach is confirmed

Every incident note should include:

- affected runtime/deployable identifiers
- classification (`rollout_regression`, `stale_evidence`, `runtime_fault`)
- primary triggering event
- evidence inspected
- operator action and timestamp

## Investigation Matrix

| Question | UI first stop | Backend confirmation | Typical next action |
| --- | --- | --- | --- |
| What is unhealthy right now? | Fleet overview | `/api/v1/health`, `/api/v1/status` | Scope affected runtimes and severity |
| What changed recently? | Runtime `Events` tab | `/api/v1/status/detailed`, websocket status stream | Build incident timeline |
| Is this rollout-linked? | Runtime `Overview` and `Events` | `/rollout-state/...`, `/promotion-state/...` | Confirm stage transition vs runtime-only fault |
| Is evidence stale? | Runtime `Reports` and event cadence | promotion/rollout state recency, latest results | Hold rollout or pause runtime pending fresh evidence |
| What should I do now? | Runtime action controls | confirm classification + risk state | pause/resume/kill with incident record |

## Operational Notes

- Keep incident narratives SLO-oriented. Avoid ad hoc logs-first investigations.
- Prefer immutable event/state reads before intervention.
- When uncertain between stale-evidence and runtime-fault classes, contain first
  (`pause`) and continue evidence collection.
