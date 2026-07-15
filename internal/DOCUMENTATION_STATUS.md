# Documentation status

This file tracks structural cleanup, not product claims. For current technical truth, follow the
precedence in `migration/docs_source_of_truth.md`.

## Current state

- The canonical MkDocs build is strict-clean on the `codex/docs-strict-build` branch/PR #24.
- `architecture/` owns the current runtime and service map.
- `testing/` plus `development/testing_guidelines.md` own reliability guarantees.
- `reference/` owns stable user-facing contracts.
- `migration/` and `compatibility/` explain staged transitions.
- `plans/` and `internal/` are non-canonical implementation history and handoff material.

## Cleanup backlog

1. `advanced/web_api.md` (~1,119 lines) and `reference/web_api.md` overlap. Keep conceptual/operator
   guidance in `advanced/`; keep endpoints, payloads, and authentication contracts in `reference/`.
   Replace duplicated sections with links and add a strict-build link check.
2. `strategy_guide.md` (~964 lines) should become an index plus focused pages under `tutorials/` and
   `reference/`. Move one topic per PR and retain redirects/links for existing anchors.
3. `core_concepts/feeds.md` (~852 lines) mixes concepts, adapter details, and recipes. Split current
   concepts from venue-specific reference and tutorials without changing runtime semantics.
4. The large architecture/refactor plans are historical inputs, not current truth. Add front matter
   or an opening status block naming the date, implementation state, and canonical replacement.
5. Consolidate the root `getting_started.md` overview with the `getting_started/` index while
   preserving the navigation entry and external links.
6. Audit quantitative performance/exchange-count claims against reproducible evidence before
   publishing them. Remove or qualify claims that lack a versioned source.

## Acceptance for content moves

- `python -m mkdocs build --strict --config-file mkdocs-subtree.yml` passes.
- No duplicate canonical explanation is introduced.
- Old links are preserved or replaced with explicit redirects.
- Code/config examples are validated against the pinned workspace, not copied from historical
  plans.
