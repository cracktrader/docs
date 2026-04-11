# Research Factory Workflow, Capability Taxonomy, and Experiment Families

Source repositories:

- [`cracktrader-lab`](https://github.com/cracktrader/cracktrader-lab)
- [`cracktrader-research-control`](https://github.com/cracktrader/cracktrader-research-control)

This page explains how Cracktrader research scales from single deterministic
runs to reusable experiment libraries with explicit capability coverage and
governance handoff.

Use it with:

- [Research Pipeline](research_pipeline.md)
- [Research Samplers](research_samplers.md)

## Mental Model

Treat research as a factory, not a sequence of one-off runs:

1. Define a strategy family and capability expectations.
2. Execute deterministic evaluations and falsification suites.
3. Aggregate outcomes at cohort/family level.
4. Seal evidence and hand off promotion decisions to governance surfaces.

## Capability Taxonomy

A capability is an operator/research expectation that a strategy family should
demonstrate under both baseline and stress conditions.

Recommended capability groups:

- Signal robustness: signal still behaves under noise/regime drift
- Execution resilience: route/latency/fee changes do not collapse intent quality
- Inventory stability: unwind/hedge paths remain controlled under stress
- Cost tolerance: edge survives realistic fee and slippage ranges
- Governance readiness: evidence quality supports repeatable gate evaluation

Capabilities should be declared at strategy-family level and referenced by
falsification suites and evaluation plans.

## Falsification Suite Catalog Model

The current `research_governance` model already has reusable suite contracts:

- `FalsificationSuite` (family, suite name, version, metadata)
- `FalsificationCase` (case id, perturbation set, expected failure modes)
- `execute_falsification_suite(...)` for deterministic suite execution
- suite registry (`register_falsification_suite`, `list_registered_suites`)

Catalog guidance:

- keep suites versioned and family-scoped
- tag each suite/case with target capability areas
- track expected failure modes explicitly
- publish suite metadata so coverage can be audited without reading code internals

## Experiment Families and Cohorts

An experiment family is a reusable batch template for a hypothesis, not a single
run. Cohorts are the concrete variants executed under that family.

Common cohort dimensions:

- baseline vs challenger strategy revisions
- market cohorts (venue groups, symbol groups, liquidity tiers)
- parameter cohorts (risk/execution/cost sweeps)
- regime cohorts (volatility/latency/funding conditions)

Target outcome: deterministic grouped comparisons that are easy to report and
re-run.

## Reference Workflow

### Step 1: Define family hypothesis and capability targets

For each strategy family, define:

- family id and revision lineage
- capability targets expected to pass
- failure modes expected to be falsified

### Step 2: Build deterministic pack inputs

Use `ctlab` strategy-pack artifacts (`strategy.json`, `evaluation.json`) as the
base declaration of strategy and evaluation intent.

Include:

- split plan and stress settings
- search lineage metadata
- candidate handoff metadata

### Step 3: Execute evaluation and falsification cohorts

Run deterministic evaluations and falsification suites for each cohort variant.
Persist run/report metadata through the run registry and report outputs.

### Step 4: Compare at family and cohort level

Use comparison tooling and run metadata to evaluate:

- baseline vs challenger deltas
- cohort consistency
- capability coverage completeness

### Step 5: Seal evidence for governance handoff

Package artifacts/runs into sealed evidence bundles and hand off to
`cracktrader-research-control` for stage-gate and rollout-state reasoning.

In the current lab flow, this starts from the candidate-pack boundary
(`ctlab candidate pack <slug>`) and continues through evidence sealing and gate
evaluation on the control-plane side.

This keeps policy evaluation deterministic while preserving control-plane
ownership of promotion truth.

## Current State vs Target Expansion

Current implemented building blocks:

- deterministic strategy-pack based run flow (`ctlab`)
- run registry and report pipeline
- deterministic falsification suite contracts and registry
- sealed evidence and promotion-state primitives in research-control

Target expansion tracked by active issues:

- richer falsification suite catalog and strategy capability declarations
- experiment-family templates and cohort orchestration primitives
- clearer discovery/reporting for capability and suite coverage

These extensions should remain deterministic and composable with existing run
registry and governance handoff contracts.
