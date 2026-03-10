# Known Gaps

This page documents real limitations and intentionally incomplete areas in the current testing and reliability story.

It should be read as a boundary document, not as a list of excuses.

## Current Gaps

### Deep Live-Market Realism

The simulator boundary now has a first realism slice, but it is not a full exchange emulator.

Do not assume the suite currently guarantees:

- queue-position realism
- venue-specific microstructure parity
- perfect partial-fill depth simulation
- live-like latency under all paths

### Environment-Backed Coverage

Sandbox and live coverage are intentionally gated.

That means default local and CI evidence is strongest for hermetic runtime guarantees, not for every external venue interaction.

### Docs Build Strictness

The docs repo still has pre-existing navigation and broken-link debt outside the newest runtime pages.

Normal MkDocs builds work, but strict mode still reports legacy warnings that need separate cleanup.

### Legacy Documentation Drift

Some older pages still exist for compatibility and migration context.

Even after the runtime-map reset, not every legacy page has been rewritten yet. Use the `architecture/` pages as the preferred truth when pages disagree.

## What This Means In Practice

- prefer deterministic contract evidence over environment-backed confidence when possible
- add regression tests when a gap turns into a real bug
- document explicit non-guarantees instead of implying the runtime is more mature than it is

## Next Cleanup Targets

- stale advanced docs with broken relative links
- larger docs information-architecture cleanup around migration and reference pages
- deeper simulator-realism documentation once the runtime surface expands
