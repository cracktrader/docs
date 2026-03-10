# Runtime Terms

Use these terms consistently across architecture docs, code review, and issue planning.

## Stable IDs

| Term | Meaning |
| --- | --- |
| `strategy_id` | Stable identity for one registered strategy instance |
| `execution_id` | Stable identity for one execution context |
| `route_id` | Stable identity for one named execution route |
| `broker_id` | Identity for the underlying broker or adapter binding behind a route |
| `snapshot_id` | Identity for one immutable shared runtime snapshot |

## Shared Runtime Terms

| Term | Meaning |
| --- | --- |
| Session | High-level owner for shared runtime construction and registrations |
| Reference data | Canonical instrument and venue metadata used by execution, simulation, and risk |
| State coordinator | Shared owner of observed state, projections, and snapshot materialization |
| Shared snapshot | The immutable per-step state view produced once for the active graph |
| Filtered view | The strategy-scoped slice of the shared snapshot handed to one strategy |
| Strategy registration | Binding of a strategy to a `strategy_id`, subscriptions, and optional execution/risk references |
| Execution context | Named execution surface that groups one or more routes |
| Route | Stable execution target bound to a broker/account adapter |

## Execution and Post-Trade Terms

| Term | Meaning |
| --- | --- |
| Execution intent | Attributed request to submit, cancel, or modify execution behavior against a route |
| Order report | Normalized order lifecycle update emitted by the runtime or adapter |
| Fill report | Normalized fill lifecycle update with attribution and execution metadata |
| Inventory service | Central application point for execution reports into positions and exposure |
| Exposure view | Read model over inventory grouped by route, execution context, or symbol |
| Risk decision | Structured allow/reject outcome applied before execution submission |
| Post-trade sink | Hook that consumes normalized post-trade records after runtime processing |
| Control plane | Lightweight runtime surface for route status, operational gating, and kill-switch semantics |

## Transitional Terms

These terms still appear, but should not be treated as the whole architecture:

| Term | How to read it now |
| --- | --- |
| Cerebro | Important compatibility/runtime entrypoint, not the whole system map |
| Feed | Market input surface within a broader shared-state runtime |
| Broker | One execution adapter or route binding within a broader execution model |
| Store | Exchange I/O and transport owner, not the owner of runtime orchestration |
