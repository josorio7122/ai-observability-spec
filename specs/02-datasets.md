# Datasets

> Versioned collections of input/expected-output pairs that serve as the evaluation substrate for experiments. Datasets are built from curated examples, bulk imports, and annotated production traces.

Cross-cutting rules (auth, errors, pagination, timestamps) are defined in `CONSTITUTION.md`. Object schemas are defined in `DATA-MODEL.md`.

---

## Overview

A dataset is the bridge between real-world observations and systematic evaluation. Without datasets, there is nothing to run experiments against. The core workflow is:

1. **Collect** — gather representative inputs (manually, from imports, or from annotated traces)
2. **Curate** — optionally add expected outputs to create ground-truth pairs
3. **Evaluate** — run experiments against the dataset to measure application quality
4. **Reconcile** — continuously add new items as the system encounters new failure patterns

Datasets are versioned. Every time items are added or removed, the version increments. This makes it possible to track which version of a dataset was used in any given experiment.

---

## User Stories

- As a developer, I can create a dataset and add input/expected-output pairs so I have a baseline to evaluate my application against.
- As a developer, I can import items in bulk from a JSONL file so I can seed a dataset from existing logs or curated data.
- As a developer, I can convert an annotated trace into a dataset item so production failures become regression test cases.
- As a developer, I can list and inspect my datasets so I can find the right one to use for an experiment.
- As a developer, I can delete a dataset I no longer need.

---

## Dataset Versioning

- Every dataset starts at version `1`.
- The version increments by `1` whenever items are added or removed.
- A single bulk import that adds N items increments the version by `1` (not by N).
- The version is read-only — it cannot be set by the client.
- Experiments record which `dataset_id` they were run against. Since datasets are mutable (items can be added), implementors should store a snapshot of the dataset's item list at the time of the experiment run, or record the version number for reproducibility. This is implementation-defined.

---

## CRUD Operations

### Create a Dataset

- `name` is required and must be unique within a project.
- `description` is optional.
- Creating a dataset with a `name` that already exists in the project returns `409 CONFLICT` with error code `CONFLICT`.
- A newly created dataset has `version: 1` and `item_count: 0`.

### Add a Single Item

- `input` is required.
- `expected_output` is optional.
- `metadata` is optional.
- Adding an item increments `version` by 1 and `item_count` by 1.

### Bulk Import from JSONL

The import endpoint accepts a JSONL (newline-delimited JSON) body where each line is a DatasetItem object (`input` required, `expected_output` optional, `metadata` optional).

**Partial success behavior:**
- The import is processed line by line.
- Valid lines are imported.
- Invalid lines (malformed JSON, missing `input`) are skipped and reported in the response.
- The dataset is never left in a partial state mid-line — each line is either fully committed or fully skipped.
- The response includes `imported_count`, `skipped_count`, and a `skipped` array describing which lines failed and why.
- If all lines are invalid, no items are added and the version is not incremented.
- If at least one line is valid, the version increments by 1 regardless of how many items were added.

**JSONL format example:**
```jsonl
{"input": "What is the capital of France?", "expected_output": "Paris"}
{"input": "Summarize this document: ...", "metadata": {"source": "support-ticket-4821"}}
{"input": {"messages": [{"role": "user", "content": "Hello"}]}}
```

### Delete a Dataset

- Deletes the dataset and all its items permanently.
- Returns `404 NOT_FOUND` if the dataset does not exist.
- If the dataset is referenced by one or more experiments, deletion is still permitted. The experiments retain their runs and results, but the source dataset items are gone. This is a deliberate trade-off: experiments are records of past evaluations and should not be invalidated by dataset deletion.

### List Datasets

- Requires `project_id` as a query parameter.
- Returns datasets ordered by `created_at` descending (most recent first).
- Supports cursor-based pagination (see `CONSTITUTION.md`).

---

## Creating a DatasetItem from an Annotation

When a trace is annotated with a correction, that annotation can be converted into a DatasetItem. This is the mechanism by which production failures become regression test cases.

**Conversion rules:**

| DatasetItem field | Source |
|---|---|
| `dataset_id` | Specified by the caller in the conversion request |
| `input` | The `input` of the trace's root span |
| `expected_output` | The `correction` field of the annotation |
| `metadata.source_trace_id` | The `trace_id` of the annotation |
| `metadata.source_annotation_id` | The `id` of the annotation |

- If the annotation has no `correction`, the DatasetItem is created with `expected_output: null`.
- The conversion creates a new DatasetItem; it does not modify the annotation or the trace.
- The conversion endpoint is defined in `specs/05-annotation.md`.

---

## Dataset Reconciliation

The recommended pattern for maintaining datasets is **continuous reconciliation**, not one-time "golden dataset" curation:

- **Add frequently** — as your application encounters new failure patterns, convert those traces to dataset items.
- **Don't batch-curate upfront** — a small, well-chosen dataset updated regularly outperforms a large static one.
- **Use metadata to track provenance** — tag items with their source (manual, import, annotation conversion) so you can audit dataset composition.
- **Do not delete items to "fix" a dataset** — if an item is no longer representative, add better items; deletion loses the historical record that an experiment was run against it.

This is a recommendation, not an enforced constraint. The spec does not prevent deletion.

---

## Acceptance Criteria

```
GIVEN a project with no datasets
WHEN a dataset named "qa-baseline" is created
THEN a dataset is returned with version=1, item_count=0, and the provided name

GIVEN a project that already has a dataset named "qa-baseline"
WHEN another dataset named "qa-baseline" is created in the same project
THEN the API returns 409 with error code CONFLICT

GIVEN a dataset with 10 items
WHEN 5 new items are added one by one
THEN after each addition, item_count increments by 1
AND version increments by 1 for each addition
AND the final state is item_count=15, version=16

GIVEN a JSONL import with 3 valid lines and 1 malformed line (invalid JSON)
WHEN the bulk import is requested
THEN 3 items are added to the dataset
AND the response contains imported_count=3, skipped_count=1
AND the skipped array identifies the malformed line number and reason
AND the dataset version increments by 1

GIVEN a JSONL import where every line is malformed
WHEN the bulk import is requested
THEN no items are added
AND the dataset version is not incremented
AND the response contains imported_count=0, skipped_count=N

GIVEN a JSONL import where one line is missing the required "input" field
WHEN the bulk import is requested
THEN that line is skipped
AND other valid lines are imported
AND the skipped array reports the missing field

GIVEN a dataset referenced by an experiment
WHEN the dataset is deleted
THEN the dataset and its items are removed
AND the experiment and its runs remain intact
AND a subsequent GET for the dataset_id returns 404

GIVEN an annotation with a correction on trace T1 whose root span has input "What is 2+2?"
WHEN the annotation is converted to a dataset item in dataset D1
THEN a DatasetItem is created with input="What is 2+2?" and expected_output equal to the annotation's correction
AND metadata contains source_trace_id=T1 and source_annotation_id equal to the annotation's id

GIVEN a dataset that does not exist
WHEN a GET is performed for that dataset_id
THEN the API returns 404 with error code NOT_FOUND
```

---

## Edge Cases

| Scenario | Expected Behavior |
|---|---|
| Dataset with zero items | Valid. Experiments run against an empty dataset complete immediately with zero runs. |
| DatasetItem `input` is `null` | Rejected: `INVALID_REQUEST` — `input` must not be `null` |
| DatasetItem `input` is an empty string `""` | Valid — an empty string is a meaningful input |
| JSONL line that is valid JSON but not an object (e.g., `"hello"`) | Skipped and reported in `skipped` array |
| Converting an annotation with no `correction` to a dataset item | Allowed — `expected_output` will be `null` |
| Adding items to a dataset while an experiment is running against it | Allowed — the in-flight experiment is not affected; behavior depends on whether the implementation snapshots the dataset at experiment creation |
| Dataset `name` with leading/trailing whitespace | Implementation should trim; the trimmed name is used for uniqueness checks |
