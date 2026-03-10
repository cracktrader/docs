# Architecture Index

Read this page first if you need to ingest Cracktrader quickly as an engineer, reviewer, or agent.

## Fast Path

Use these pages in order:

1. [Runtime Map](runtime_map.md)
2. [Mode Matrix](mode_matrix.md)
3. [Runtime Terms](runtime_terms.md)
4. [Entry Points and API Freeze](../reference/entry_points.md)

If you are checking whether an older page is still authoritative, assume the pages above are the current truth unless they explicitly point you elsewhere.

## What Cracktrader Is Now

Cracktrader is no longer best understood as "a Cerebro app with feeds and brokers around it."

The current runtime direction is:

- a session-owned reference-data and state layer
- one shared snapshot per step
- one or more registered strategies consuming filtered views of that snapshot
- execution contexts and stable routes selecting broker adapters
- centralized inventory and risk checks before execution
- unified live, paper, and backtest execution boundaries
- post-trade and control-plane hooks around the runtime edge

Backtrader compatibility still matters, but it is now a compatibility surface, not the primary architecture map.

## Ownership Map

| Runtime surface | Owns | Main user-facing entrypoints |
| --- | --- | --- |
| Session | Shared construction, registrations, default runtime services | `exchange(...)`, session helpers, session `add_*` registration methods |
| Reference data | Canonical symbols, venue mappings, execution validation metadata | reference data registry helpers |
| State coordinator | Observed state, projections, shared snapshot materialization | session observation/project/indicator registration |
| Strategy orchestration | Filtered snapshot fanout, strategy attribution, shared-step dispatch | strategy registrations, native runtime strategy contracts |
| Execution contexts and routes | Stable route identity, broker bindings, route selection | broker registrations, execution-context registrations |
| Inventory and exposure | Route/execution-scoped positions and fills | inventory service, exposure view |
| Risk engine | Pre-submission policy evaluation and explicit decisions | risk profiles, route/execution bindings |
| Execution adapters | Mode-specific execution behind one contract | live, paper, and backtest execution adapters |
| Post-trade and control plane | Runtime sinks, kill-switches, operational hooks | post-trade sink, runtime control plane |

## Interface Index

These are the main contracts to anchor on when reading the codebase or docs.

| Contract family | What it tells you |
| --- | --- |
| Runtime identities | Stable IDs for strategies, execution contexts, routes, brokers, and snapshots |
| Strategy input/output | What strategies receive each step and how intents/diagnostics are attributed |
| Execution intents and reports | How orders, fills, and route-targeted execution move through the runtime |
| Reference data and route metadata | How instruments, venue symbols, and route capabilities are normalized |
| Inventory and exposure | What is centrally tracked after execution reports are applied |
| Risk decisions | Why an intent was allowed, modified, or rejected |
| Mode matrix | Which guarantees stay constant across backtest, paper, sandbox, and live |

## Stable Runtime Vocabulary

| Term | Meaning |
| --- | --- |
| `strategy_id` | Stable identity for one registered strategy |
| `execution_id` | Named execution context that groups one or more routes |
| `route_id` | Stable execution route chosen at intent time |
| `broker_id` | Identity of the underlying broker/adapter binding |
| `snapshot_id` | Identity of one immutable runtime snapshot/step |
| Shared snapshot | The session-owned state view materialized once per step |
| Filtered view | The strategy-scoped subset of the shared snapshot handed to one strategy |

## Current Truth vs Transitional Pages

Use this rule when the docs disagree:

- `architecture/` pages describe the preferred current runtime map.
- `reference/` pages describe stable interfaces and behavior details.
- `testing/` and `development/` pages describe suite and contribution rules.
- `compatibility/` and selected `core_concepts/` pages may explain older mental models or migration context.

See [Legacy Architecture Context](../core_concepts/architecture.md) when you need to understand older Cerebro/store/feed-centric framing without treating it as the canonical system map.
