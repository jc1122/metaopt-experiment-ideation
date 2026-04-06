---
name: metaopt-experiment-ideation
description: "Use when the ml-metaoptimization orchestrator needs fresh experiment proposals. Generates non-overlapping, concrete ML experiment ideas based on campaign goals, baselines, and prior learnings. Keywords: ideation, proposal generation, experiment ideas, hypothesis generation, metaoptimization worker."
---

# metaopt-experiment-ideation

## Overview

Background worker skill for the `ml-metaoptimization` orchestrator.
Runs in the **ideation** lane (`background` slot class) to generate and refine non-overlapping experiment proposals during the proposal-accumulation phase of the metaoptimization campaign.

This skill is a leaf worker — it does not manage state, dispatch subagents, or interact with the queue backend.
It receives campaign context from the orchestrator via a subagent prompt, produces proposal candidates, and returns them.
The orchestrator is responsible for persisting proposals into the appropriate pool.

Model class: `general_worker` (prefer a capable general model like GPT-5.4, fallback to any stronger available model).

## Input Contract

The orchestrator supplies all inputs as structured context in the subagent prompt. This skill never reads files directly from the campaign repo.

Required inputs:

| Field | Type | Description |
|-------|------|-------------|
| `goal` | string | The campaign's top-level improvement goal |
| `metric` | string | The objective metric being optimized |
| `objective_direction` | string | `minimize` or `maximize` |
| `aggregation` | string | How per-dataset scores combine into the aggregate metric |
| `aggregate_baseline` | number | Current aggregate baseline score |
| `per_dataset_baselines` | map | Baseline score for each dataset |
| `key_learnings` | list | Learnings extracted from prior iterations |
| `completed_experiments` | list | Summary of all previously run experiments |
| `current_proposal_pool` | list | Proposals already in the current cycle's pool |
| `next_proposal_pool_context` | list | Proposals carried over for the next cycle |
| `proposal_policy` | object | Policy governing proposal generation (see below) |

### Proposal Policy Fields

| Field | Type | Description |
|-------|------|-------------|
| `current_target` | integer | Target number of proposals for the current pool |
| `current_floor` | integer | Minimum proposals before selection can proceed |
| `next_cap` | integer | Maximum proposals allowed in the next pool |
| `distinctness_rule` | string | Rule describing what counts as a distinct proposal |

## Output Contract

Return one or more proposal candidates. Each proposal must include:

| Field | Type | Description |
|-------|------|-------------|
| `title` | string | Concise name for the experiment (≤ 12 words) |
| `rationale` | string | Short explanation of the hypothesis and why it may improve the metric |
| `expected_impact` | object | `{ direction: "improve" \| "neutral", magnitude: "small" \| "medium" \| "large" }` |
| `target_area` | string | Which pipeline area this targets (see allowed values below) |

### Allowed `target_area` Values

- `feature_engineering`
- `model_architecture`
- `training_procedure`
- `hyperparameter_tuning`
- `data_augmentation`
- `data_preprocessing`
- `loss_function`
- `regularization`
- `ensemble_strategy`
- `inference_optimization`
- `evaluation_methodology`

If the proposal pool is already at `next_cap`, return no proposals and instead return a saturation signal:

```json
{ "saturated": true, "reason": "next_cap reached" }
```

## Behavioral Rules

1. **No duplication.** Never duplicate or trivially rephrase an existing proposal in `current_proposal_pool`, `next_proposal_pool_context`, or `completed_experiments`. A proposal is a duplicate if it targets the same aspect with the same mechanism, even if worded differently.

2. **Respect distinctness_rule.** The `distinctness_rule` from `proposal_policy` is the authoritative definition of what makes two proposals distinct. Follow it exactly.

3. **Concrete proposals only.** Every proposal must be specific enough that an experiment designer can translate it into a concrete experiment specification without ambiguity. Avoid vague suggestions like "try a better model" or "improve feature engineering."

4. **No code changes.** This skill proposes ideas only. It never generates code, patches, or implementation artifacts.

5. **Saturation awareness.** If `next_proposal_pool_context` has reached `next_cap`, generate no new proposals. Return the saturation signal instead.

6. **Leverage learnings.** Use `key_learnings` and `completed_experiments` to avoid repeating failed approaches and to build on successful directions.

7. **Respect objective direction.** Proposals must target improvement in the direction specified by `objective_direction`. A proposal for a metric that moves in the wrong direction is invalid.

8. **Scope to campaign goal.** All proposals must be relevant to the stated `goal` and `metric`. Off-topic proposals waste selection cycles.

9. **Impact honesty.** `expected_impact.magnitude` should reflect realistic expectations, not optimistic guesses. Use `small` for incremental improvements, `medium` for meaningful gains, and `large` only for fundamental approach changes.

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Rephrasing a completed experiment as a new proposal | Check `completed_experiments` for semantic overlap before proposing |
| Ignoring `key_learnings` that mark an approach as exhausted | Read all learnings and avoid proposals in directions already proven unfruitful |
| Proposing vague ideas like "tune hyperparameters" | Specify which hyperparameters, what range, and why |
| Generating proposals when pool is at `next_cap` | Check pool size against `next_cap` first; return saturation signal if at capacity |
| Proposing changes that contradict the objective direction | Verify each proposal improves the metric in the correct direction |
| Producing a single proposal when more are needed | Generate multiple diverse proposals when the pool is far from `current_target` |
| Duplicating proposals already in the current pool | Diff against all entries in `current_proposal_pool` and `next_proposal_pool_context` |
| Including implementation code in the proposal | Proposals describe what to try and why, never how to implement it in code |

## References

This skill is part of the `ml-metaoptimization` orchestrator ecosystem:

- **`ml-metaoptimization/references/worker-lanes.md`** — Authoritative lane contract for the ideation slot, including input/output expectations and slot class rules.
- **`ml-metaoptimization/references/contracts.md`** — State file schema, proposal pool semantics, and slot field definitions.
- **`ml-metaoptimization/SKILL.md`** — Orchestrator skill contract defining dispatch invariants and worker policy.
