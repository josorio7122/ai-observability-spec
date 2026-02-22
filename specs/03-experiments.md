# Experiments

> The mechanism for systematically measuring application quality by running it against a dataset and capturing scored outputs. Experiments produce the evidence needed to make confident decisions about prompt changes, model upgrades, and retrieval improvements.

Cross-cutting rules (auth, errors, pagination, timestamps) are defined in `CONSTITUTION.md`. Object schemas are defined in `DATA-MODEL.md`.

---

## Overview

An experiment answers the question: "How well does my application perform on this set of inputs?" It connects three things:

1. A **dataset** — the inputs (and optional expected outputs) to evaluate against
2. An **application** — run externally by the developer; the platform records its outputs
3. **Scorers** — functions that measure output quality (defined in `specs/04-scoring.md`)

The platform does not run the application. It provides a place to record the results. The developer's code (or CI pipeline) calls the platform API to submit runs and scores, then queries the platform to analyze results.

---

## User Stories

- As a developer, I can create an experiment tied to a dataset so I have a container for recording evaluation results.
- As a developer, I can submit my application's output for each dataset item so the platform records what it produced.
- As a developer, I can attach scores to each run so quality is quantified, not just observed.
- As a developer, I can get an aggregate summary of an experiment so I can see overall quality at a glance.
- As a developer, I can compare two experiments so I can see whether a change improved or degraded quality.
- As a developer, I can set a passing threshold so my CI pipeline fails automatically when quality drops.

---

## Experiment Lifecycle

```
created ──► running ──► completed
```

- **`created`** — The experiment exists but no runs have been submitted yet.
- **`running`** — At least one run has been submitted.
- **`completed`** — The experiment is closed. No more runs may be submitted.

### Transitioning to `completed`

Implementations may choose one or both of:
1. **Explicit close** — A client calls the complete endpoint (`POST /v1/experiments/:id/complete`) to mark it done.
2. **Automatic close** — The platform transitions to `completed` when runs have been submitted for every item in the referenced dataset.

Both mechanisms must be supported. The spec does not require a specific default.

Once `completed`, the experiment is immutable — runs cannot be added or modified.

---

## Submitting Runs

A run records the application's output for a single dataset item.

- Each run references one `dataset_item_id`.
- Each `dataset_item_id` may have **at most one run per experiment**. Submitting a second run for the same item in the same experiment returns `409 CONFLICT` with error code `DUPLICATE_RUN`.
- Runs may be submitted individually or in batches (up to implementation-defined limits).
- `output` is required and must not be `null`.
- `trace_id` is optional but recommended — linking a run to its trace enables drill-down into the full execution context.

### Submitting Scores with Runs

Scores may be included inline when submitting a run (in a `scores` array on the run object). This is equivalent to submitting the run and then calling `POST /v1/scores` for each score separately. The inline format is a convenience to reduce round-trips.

See `specs/04-scoring.md` for scorer definitions and scoring behavior.

---

## Aggregate Summary

The experiment summary endpoint (`GET /v1/experiments/:id/summary`) returns:

| Field | Type | Description |
|---|---|---|
| `experiment_id` | string | |
| `status` | string | Current experiment status |
| `run_count` | integer | Total runs submitted |
| `dataset_item_count` | integer | Total items in the referenced dataset at time of query |
| `scores_by_scorer` | object | Per-scorer aggregate stats (see below) |
| `threshold_result` | object \| null | Pass/fail result if a threshold was evaluated (see below) |

**`scores_by_scorer`** is a map from `scorer_name` to:

| Field | Type | Description |
|---|---|---|
| `scorer_name` | string | |
| `scored_run_count` | integer | Number of runs that have a score from this scorer |
| `mean` | number \| null | Mean numeric score across all scored runs. `null` if scorer produces categorical labels. |
| `min` | number \| null | Minimum score. `null` for categorical scorers. |
| `max` | number \| null | Maximum score. `null` for categorical scorers. |
| `distribution` | object \| null | For categorical scorers: map of label → count. `null` for numeric scorers. |

---

## Experiment Comparison

Two experiments on the **same dataset** can be compared to quantify the impact of a change.

The comparison endpoint (`GET /v1/experiments/:id/compare/:other_id`) returns:

| Field | Type | Description |
|---|---|---|
| `base_experiment_id` | string | The experiment in the `:id` position (the baseline) |
| `compare_experiment_id` | string | The experiment in the `:other_id` position (the candidate) |
| `scorer_comparisons` | object[] | Per-scorer comparison results |
| `per_item_results` | object[] | Per-dataset-item score deltas |

**Each `scorer_comparison` contains:**

| Field | Type | Description |
|---|---|---|
| `scorer_name` | string | |
| `base_mean` | number \| null | Mean score in the base experiment |
| `compare_mean` | number \| null | Mean score in the candidate experiment |
| `delta` | number \| null | `compare_mean - base_mean`. Positive = improvement. `null` if either is `null`. |
| `improved_count` | integer | Items where candidate score > base score |
| `regressed_count` | integer | Items where candidate score < base score |
| `unchanged_count` | integer | Items where scores are equal |
| `only_in_base` | integer | Items scored in base but not candidate |
| `only_in_compare` | integer | Items scored in candidate but not base |

**Each `per_item_result` contains:**

| Field | Type | Description |
|---|---|---|
| `dataset_item_id` | string | |
| `scorer_name` | string | |
| `base_score` | number \| string \| null | Score in base experiment (null if not scored) |
| `compare_score` | number \| string \| null | Score in candidate experiment (null if not scored) |
| `delta` | number \| null | `compare_score - base_score`. `null` for categorical or missing scores. |

**Constraints:**
- Both experiments must reference the same `dataset_id`. Comparing experiments on different datasets returns `422` with error code `INCOMPATIBLE_EXPERIMENTS`.
- Comparison is always read-only — it does not modify either experiment.

---

## Threshold Evaluation

A threshold defines the minimum acceptable quality for an experiment to be considered passing. This enables CI/CD integration: run the eval, check the threshold, fail the build if quality drops.

The threshold evaluation endpoint (`POST /v1/experiments/:id/threshold`) accepts:

| Field | Type | Required | Description |
|---|---|---|---|
| `scorer_name` | string | yes | Which scorer's scores to evaluate |
| `metric` | string | yes | One of: `mean`, `min`, `max` |
| `threshold` | number | yes | Minimum required value (0.0–1.0) |
| `comparison` | string | no | One of: `gte` (default), `gt`, `lte`, `lt` |

The response:

| Field | Type | Description |
|---|---|---|
| `passed` | boolean | Whether the experiment meets the threshold |
| `actual_value` | number \| null | The computed metric value |
| `threshold` | number | The threshold value that was checked |
| `scorer_name` | string | |
| `metric` | string | |
| `gap` | number \| null | `actual_value - threshold`. Negative means failing. |

**Constraints:**
- If no runs have been scored by the specified scorer, `actual_value` is `null` and `passed` is `false`.
- Threshold evaluation is stateless — it does not mark the experiment or change its status.
- Thresholds on categorical scorers are not supported (`422 UNSUPPORTED_THRESHOLD_TYPE`).

---

## Acceptance Criteria

```
GIVEN a dataset with 3 items
WHEN an experiment is created referencing that dataset
THEN the experiment has status=created and is returned with the correct dataset_id

GIVEN an experiment in status=created
WHEN a run is submitted for one of its dataset items
THEN the experiment transitions to status=running

GIVEN an experiment with 3 dataset items where runs have been submitted for 2 items
WHEN the experiment is explicitly completed via the complete endpoint
THEN the experiment status becomes=completed

GIVEN a completed experiment
WHEN a new run is submitted
THEN the API returns 422 with error code EXPERIMENT_COMPLETED

GIVEN an experiment where a run for dataset_item_id="item-1" has already been submitted
WHEN another run for dataset_item_id="item-1" is submitted to the same experiment
THEN the API returns 409 with error code DUPLICATE_RUN

GIVEN an experiment with 3 runs scored by "exact_match" with values 1.0, 0.0, 1.0
WHEN the summary is requested
THEN scores_by_scorer["exact_match"].mean equals 0.667 (rounded to 3 decimal places)
AND scores_by_scorer["exact_match"].min equals 0.0
AND scores_by_scorer["exact_match"].max equals 1.0

GIVEN two experiments A (mean exact_match=0.6) and B (mean exact_match=0.8) on the same dataset
WHEN A is compared to B
THEN scorer_comparisons[0].delta equals 0.2
AND scorer_comparisons[0].improved_count reflects items where B scored higher

GIVEN two experiments on different datasets
WHEN a comparison is requested
THEN the API returns 422 with error code INCOMPATIBLE_EXPERIMENTS

GIVEN an experiment with mean exact_match score of 0.75 and threshold of 0.80
WHEN threshold evaluation is requested with metric=mean, scorer=exact_match, threshold=0.80
THEN passed=false
AND gap=-0.05

GIVEN an experiment with mean exact_match score of 0.85 and threshold of 0.80
WHEN threshold evaluation is requested with metric=mean, scorer=exact_match, threshold=0.80
THEN passed=true
AND gap=0.05

GIVEN a dataset with 0 items
WHEN an experiment is created and immediately completed
THEN the summary shows run_count=0 and scores_by_scorer is empty
AND the experiment completes without error
```

---

## Edge Cases

| Scenario | Expected Behavior |
|---|---|
| Experiment referencing a deleted dataset | Experiment and runs remain valid; `dataset_item_count` in summary reflects 0 since items are gone |
| Run submitted with a `dataset_item_id` not in the experiment's dataset | Rejected: `422 INVALID_DATASET_ITEM` |
| Run submitted with no scores | Valid — scores may be attached later via `POST /v1/scores` |
| Comparing an experiment to itself | Allowed — all deltas will be 0, improved/regressed counts will be 0 |
| Threshold evaluation when no runs have been scored | `actual_value=null`, `passed=false` |
| Experiment with runs from multiple scorer types | Summary includes a `scores_by_scorer` entry for each distinct `scorer_name` |
| Summary requested on a `created` experiment with no runs | Returns `run_count=0`, empty `scores_by_scorer`, `threshold_result=null` |
