# Docs Source of Truth and Migration Map

This page explains how to read the docs tree during the architecture reset.

The point is not to pretend every older page disappeared. The point is to make it obvious which pages describe current truth, which pages describe transition, and which pages are retained as appendices or legacy context.

## Reading Order

If you want the current runtime story, read in this order:

1. `architecture/`
2. `testing/` and `development/testing_guidelines.md`
3. `reference/`
4. `migration/` and `compatibility/` only when you need transitional or older context

## Source-of-Truth Categories

| Category | What it means | Typical locations |
| --- | --- | --- |
| Current architecture truth | Preferred current runtime map and ownership boundaries | `architecture/` |
| Runtime reliability truth | What the suite and runtime guarantees actually protect | `testing/`, `development/testing_guidelines.md` |
| Stable user-facing reference | Entrypoints, configuration, and API surfaces users should rely on | `reference/` |
| Migration guidance | Pages that explain how old and new models coexist during staged refactors | `migration/`, selected `compatibility/` pages |
| Legacy context | Older mental models kept for historical links and compatibility reading | selected `core_concepts/`, `compatibility/` |
| Appendices and planning material | deeper notes, RFCs, or internal plans that are informative but not primary truth | `plans/`, `internal/`, specialized reference pages |

## What To Trust When Pages Disagree

Use this precedence:

1. `architecture/`
2. `testing/` and current runtime guarantee pages
3. `reference/`
4. migration and compatibility pages
5. older concept pages and planning docs

If a page lower on that list contradicts one higher on the list, treat the higher page as canonical unless it explicitly says otherwise.

## Why This Split Exists

Cracktrader is in the middle of a staged architecture reset:

- session-owned shared state
- multi-strategy orchestration
- execution contexts and routes
- central inventory and risk
- unified execution adapters
- post-trade and control-plane hooks

Older docs were written for a more Backtrader-centric mental model. Those pages are still useful, but they should not silently override the new runtime map.

## Migration Buckets

### Current Runtime Pages

Use for:

- architecture ingestion
- current system ownership questions
- runtime contracts and guarantees

### Migration Pages

Use for:

- understanding renamed or reframed concepts
- knowing why an older page still exists
- staged rollout context while the tree is being reorganized

### Legacy and Compatibility Pages

Use for:

- Backtrader compatibility
- older feeds/brokers/store explanations
- implementation detail that has not yet been rewritten into the new architecture map

### Appendices

Use for:

- RFCs
- planning notes
- specialized operational or research references

These pages can be valuable, but they are not the first place to look for current architecture truth.

## Recommended Cross-Links

- [Architecture Index](../architecture/agent_index.md)
- [Runtime Map](../architecture/runtime_map.md)
- [Runtime Guarantees](../testing/runtime_guarantees.md)
- [Legacy Architecture Context](../core_concepts/architecture.md)
