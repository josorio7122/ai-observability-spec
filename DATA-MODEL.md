# Data Model

> Canonical definitions of every core entity in the platform. All sub-specs in `specs/` reference this document for object shapes. Cross-cutting rules (IDs, timestamps, error format) are defined in `CONSTITUTION.md`.

---

## Entity Overview

```
Project
 ├── Trace
 │    └── Span (tree, via parent_span_id)
 │         └── Score (online scoring)
 ├── Dataset
 │    └── DatasetItem
 ├── Experiment
 │    └── ExperimentRun
 │         └── Score
 └── Annotation → DatasetItem (conversion)
```

Every entity belongs to a project. Traces and Datasets are the two primary input sources. Experiments link Datasets to application outputs. Scores quantify quality. Annotations capture human corrections and feed back into Datasets.

---

## Span

A single operation within a trace — a model call, a tool invocation, a retrieval step, or any other instrumented unit of work.

| Field | Type | Required | Description |
|---|---|---|---|
| `id` | string | yes | Unique identifier for this span. Assigned by the client before submission (so client can correlate parent/child spans). |
| `trace_id` | string | yes | ID of the trace this span belongs to. |
| `parent_span_id` | string | no | ID of the parent span. `null` or absent means this is the root span of the trace. |
| `name` | string | yes | Human-readable name for this operation (e.g., `"llm_call"`, `"vector_search"`, `"tool:weather_api"`). |
| `input` | any | no | The input to this operation. May be a string, object, array, or `null`. |
| `output` | any | no | The output of this operation. May be a string, object, array, or `null`. |
| `start_time` | timestamp | yes | When this span began (ISO 8601 UTC). |
| `end_time` | timestamp | no | When this span ended. `null` or absent if the span is still in progress. |
| `duration_ms` | number | no | Duration in milliseconds. If omitted, implementations should compute it from `start_time` and `end_time`. |
| `tokens_input` | integer | no | Number of input tokens consumed. Applies to LLM spans. |
| `tokens_output` | integer | no | Number of output tokens generated. Applies to LLM spans. |
| `model` | string | no | Model identifier (e.g., `"gpt-4o"`, `"claude-3-5-sonnet-20241022"`). Applies to LLM spans. |
| `metadata` | object | no | Arbitrary key-value pairs for implementor or user-defined context. Values must be strings, numbers, booleans, or `null`. |
| `error` | object | no | Present if this span errored. See [Span Error](#span-error) below. |

### Span Error

When a span fails, the `error` field contains:

| Field | Type | Required | Description |
|---|---|---|---|
| `message` | string | yes | Human-readable error message. |
| `type` | string | no | Error type or class name (e.g., `"RateLimitError"`, `"TimeoutError"`). |
| `stack` | string | no | Stack trace, if available. |

### Span Constraints

- A trace may have **at most one root span** (a span with no `parent_span_id`).
- A span's `parent_span_id` must reference another span with the same `trace_id`.
- Circular references (A is parent of B, B is parent of A) are invalid.
- `start_time` is required; `end_time` may be omitted for in-progress spans.
- `id` is assigned by the **client** (not the server) to allow out-of-order ingestion with correct parent-child linking.

---

## Trace

A complete record of a single user request's execution, composed of one or more spans.

| Field | Type | Required | Description |
|---|---|---|---|
| `id` | string | yes | Unique identifier. Assigned by the server on first span ingestion for this trace_id. |
| `project_id` | string | yes | Project this trace belongs to. |
| `root_span_id` | string | no | ID of the root span. `null` if no root span has been ingested yet. |
| `spans` | Span[] | yes | All spans belonging to this trace, ordered by `start_time`. |
| `created_at` | timestamp | yes | When the first span for this trace was received. |
| `metadata` | object | no | Arbitrary key-value pairs (same constraints as Span metadata). |

### Trace Constraints

- A trace is created implicitly when the first span with a new `trace_id` is ingested.
- Traces are **immutable** in the sense that submitted spans cannot be modified after submission. New spans may be added to an in-progress trace.
- There is no explicit "close" operation for a trace; it is considered complete when no more spans are submitted to it.

---

## Dataset

A versioned collection of input/expected-output pairs used to evaluate application behavior.

| Field | Type | Required | Description |
|---|---|---|---|
| `id` | string | yes | Unique identifier. Assigned by server. |
| `project_id` | string | yes | Project this dataset belongs to. |
| `name` | string | yes | Human-readable name (e.g., `"customer-support-v1"`). Must be unique within a project. |
| `description` | string | no | Longer description of the dataset's purpose. |
| `version` | integer | yes | Monotonically increasing version number. Starts at `1`. Incremented whenever items are added or removed. |
| `item_count` | integer | yes | Current number of items in the dataset. |
| `created_at` | timestamp | yes | When this dataset was created. |
| `updated_at` | timestamp | yes | When items were last modified. |

### Dataset Constraints

- `name` must be unique within a project. Attempting to create a duplicate returns `409 CONFLICT`.
- Deleting a dataset also deletes all its items.
- Deleting a dataset that is referenced by an experiment is allowed; the experiment retains its runs and results but can no longer access the source dataset items.

---

## DatasetItem

A single evaluation case within a dataset.

| Field | Type | Required | Description |
|---|---|---|---|
| `id` | string | yes | Unique identifier. Assigned by server. |
| `dataset_id` | string | yes | Dataset this item belongs to. |
| `input` | any | yes | The input to feed to the application under evaluation. May be any JSON value. |
| `expected_output` | any | no | The expected or reference output. Used by scorers that compare against a ground truth. |
| `metadata` | object | no | Arbitrary key-value pairs. |
| `created_at` | timestamp | yes | When this item was created. |

### DatasetItem Constraints

- `input` is required and may not be `null`.
- `expected_output` is optional; some scorers (e.g., LLMJudge) do not require it.
- Items are immutable once created. To update a dataset, add new items and optionally delete old ones.

---

## Experiment

A record of intent to evaluate an application against a dataset. An experiment has many runs — one per dataset item.

| Field | Type | Required | Description |
|---|---|---|---|
| `id` | string | yes | Unique identifier. Assigned by server. |
| `project_id` | string | yes | Project this experiment belongs to. |
| `name` | string | yes | Human-readable name (e.g., `"gpt-4o-baseline"`, `"prompt-v3-test"`). |
| `dataset_id` | string | yes | The dataset this experiment evaluates against. |
| `status` | string | yes | One of: `created`, `running`, `completed`. |
| `metadata` | object | no | Arbitrary key-value pairs (e.g., model version, prompt hash). |
| `created_at` | timestamp | yes | When this experiment was created. |
| `completed_at` | timestamp | no | When the last run was submitted. |

### Experiment Status Transitions

```
created → running   (first run is submitted)
running → completed (explicitly marked complete, or implementation-defined)
```

Implementations may choose to automatically transition to `completed` when runs have been submitted for all dataset items.

---

## ExperimentRun

The result of running the application on a single DatasetItem.

| Field | Type | Required | Description |
|---|---|---|---|
| `id` | string | yes | Unique identifier. Assigned by server. |
| `experiment_id` | string | yes | Experiment this run belongs to. |
| `dataset_item_id` | string | yes | The DatasetItem used as input for this run. |
| `output` | any | yes | The application's output for this input. |
| `trace_id` | string | no | ID of the trace generated during this run (for linking to full execution context). |
| `scores` | Score[] | yes | Scores attached to this run. May be empty. |
| `created_at` | timestamp | yes | When this run was submitted. |

### ExperimentRun Constraints

- Each `dataset_item_id` within an experiment should have at most one run. Submitting a second run for the same item within the same experiment is a `409 CONFLICT`.
- `output` may not be `null` (use an empty string or empty object if the application produced no output).

---

## Score

A quality measurement attached to either an ExperimentRun or a Span.

| Field | Type | Required | Description |
|---|---|---|---|
| `id` | string | yes | Unique identifier. Assigned by server. |
| `target_id` | string | yes | ID of the resource being scored. |
| `target_type` | string | yes | One of: `run`, `span`. |
| `scorer_name` | string | yes | Name of the scorer that produced this score (e.g., `"exact_match"`, `"llm_judge_grounding"`, `"human"`). |
| `value` | number \| string | yes | The score. Numeric scores must be in the range `[0.0, 1.0]`. String scores are categorical labels (e.g., `"pass"`, `"fail"`, `"relevant"`). |
| `rationale` | string | no | Human or LLM-generated explanation for the score. |
| `created_at` | timestamp | yes | When this score was created. |

### Score Constraints

- Numeric `value` must be in `[0.0, 1.0]`. Values outside this range are rejected with `INVALID_SCORE_VALUE`.
- String `value` (categorical labels) may be any non-empty string. No enumeration is enforced by the spec.
- Multiple scores from different scorers may be attached to the same target.
- Scores are immutable once submitted.

---

## Annotation

A human-authored correction or label attached to a trace (and optionally a specific span within it).

| Field | Type | Required | Description |
|---|---|---|---|
| `id` | string | yes | Unique identifier. Assigned by server. |
| `trace_id` | string | yes | The trace being annotated. |
| `span_id` | string | no | A specific span within the trace being annotated. If absent, the annotation applies to the trace as a whole. |
| `annotator` | string | yes | Identifier for the person or system that submitted this annotation (free string; not validated). |
| `label` | string | no | A categorical classification for the trace/span (e.g., `"correct"`, `"hallucination"`, `"off-topic"`). |
| `correction` | any | no | A corrected version of the output. Used when the application's output was wrong. |
| `notes` | string | no | Free-text notes from the annotator. |
| `created_at` | timestamp | yes | When this annotation was submitted. |

### Annotation Constraints

- At least one of `label`, `correction`, or `notes` must be present (an annotation with none of these is meaningless).
- Annotations are **immutable** once submitted. If a correction is needed, submit a new annotation.
- Multiple annotations may exist for the same `trace_id` (from different annotators, or different passes).
- Annotations do not modify the underlying trace or span.

---

## Entity Relationship Summary

| Relationship | Cardinality |
|---|---|
| Project → Trace | one-to-many |
| Project → Dataset | one-to-many |
| Project → Experiment | one-to-many |
| Trace → Span | one-to-many (tree) |
| Span → Score | one-to-many |
| Dataset → DatasetItem | one-to-many |
| Experiment → ExperimentRun | one-to-many |
| ExperimentRun → DatasetItem | many-to-one |
| ExperimentRun → Score | one-to-many |
| Trace → Annotation | one-to-many |
| Annotation → DatasetItem | one-to-one (via conversion) |
