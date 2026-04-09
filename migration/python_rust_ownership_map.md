# Python vs Rust Engine Ownership Map

This page is the quick-reference view of current engine ownership during the staged Rust migration.
It is intended to answer: "what is stable today, what is transitional, and what milestones are still ahead?"

## Scope and Reading Order

Use this page as a summary, then drill into:
- [Runtime Map](../architecture/runtime_map.md)
- [Rust Migration Guide](../plans/rust_migration_guide.md)
- [Rust Engine Architecture and Python Interop Design](../plans/rust_engine_architecture.md)
- [Schema-First Python-Rust Boundary RFC](../plans/schema_first_python_rust_boundary_rfc.md)

## Current Ownership Split

| Surface | Current ownership | Notes |
| --- | --- | --- |
| Strategy authoring and callback UX | Python-owned | Strategies remain Python-first; no required Rust rewrite path. |
| Session/factory APIs (`ct.exchange`, `ct.Store`, `ct.Feed`, `ct.Broker`) | Python-owned | Public user-facing API compatibility is preserved through migration. |
| Backtrader compatibility adapters | Python-owned (compatibility layer) | Kept as adapters while deterministic core ownership shifts. |
| Exchange/feed/broker integration glue | Python-owned | Transport and venue adaptation remain Python-focused today. |
| Runtime backend selection and bridge loading | Python-owned with Rust option | Runtime supports backend selection (`python` / `rust`) and optional Rust module loading. |
| Deterministic transition core (target state) | Transitional | Python deterministic core remains the baseline reference while Rust parity is expanded. |
| Cross-language contracts and parity fixtures | Shared Python/Rust surface | Contract tests and schema-first fixtures are the migration safety boundary. |

## What Rust Covers Today (Documented Baseline)

Based on the current migration guide baseline:
- runtime backend selection path (`python` / `rust`) exists
- Rust bridge and optional module loading exist
- Rust crate scaffold exists under `rust/cracktrader_engine`
- initial Python-vs-Rust parity contract harness exists
- local install helper exists for Rust backend setup

These items provide migration scaffolding, not complete Rust-first runtime ownership.

## What Rust Does Not Cover Yet

At the current documented stage, contributors should assume:
- Python remains the default and operationally complete ownership path
- Rust is not yet the universally preferred backend for all runtime flows
- adapter/integration parity still needs staged expansion and hardening
- migration policy still requires parity gates and benchmark evidence before default changes

## Migration Milestones (Contributor View)

The staged plan in the migration guide can be summarized as:

1. **Phase A: Core parity expansion**  
   Expand fixtures and prove transition/final-state equivalence across key edge cases.
2. **Phase B: Core completeness**  
   Close known parity deltas in legality, idempotency, and replay outcomes.
3. **Phase C: Runtime hardening**  
   Increase real runtime-path coverage with `engine_backend="rust"` while keeping Python strategy UX.
4. **Phase D: Rust-preferred core**  
   Make Rust the recommended backend only after sustained parity and benchmark evidence.

## Contribution Guidance

When planning changes, use this rule:
- if the change affects public strategy/session UX, treat Python surface compatibility as authoritative
- if the change affects deterministic transition semantics, update parity fixtures/contracts as part of the same work
- if the change expands Rust ownership, document milestone impact and benchmark/parity evidence in PR notes

For detailed acceptance criteria, use:
- [Rust Migration Guide](../plans/rust_migration_guide.md)
- [Schema-First Python-Rust Boundary RFC](../plans/schema_first_python_rust_boundary_rfc.md)
