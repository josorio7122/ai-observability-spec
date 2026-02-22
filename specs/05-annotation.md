# Annotation

> The human feedback loop. Annotation lets domain experts, product managers, and developers review production traces, flag problems, provide corrections, and feed those corrections back into datasets — closing the loop between production observations and evaluation improvements.

Cross-cutting rules (auth, errors, pagination, timestamps) are defined in `CONSTITUTION.md`. Object schemas are defined in `DATA-MODEL.md`.

---

## Overview

Annotation is what separates a static eval pipeline from one that improves over time. Without annotation, datasets are populated only from manually curated examples. With annotation, every production trace that goes wrong becomes a potential regression test case.

The workflow:

1. **Observe** — A trace is ingested from production.
2. **Review** — A human (developer, PM, domain expert) views the trace in a UI.
3. **Annotate** — They flag what's wrong: add a label, provide a correction, leave notes.
4. **Convert** — The annotation is converted to a DatasetItem in a chosen dataset.
5. **Evaluate** — The next experiment run picks up the new item and tests whether the fix holds.

Annotation is not a quality gate — it does not modify traces or block execution. It is a signal layer on top of immutable traces.

---

## User Stories

- As a domain expert, I can review a production trace and flag it as problematic without needing to read raw JSON.
- As a domain expert, I can provide a corrected output for a response that was wrong so that correction can become a ground-truth example.
- As a developer, I can convert an annotation's correction into a dataset item with a single API call.
- As a developer, I can list all annotations for a trace to understand what reviewers found problematic.

---

## Annotation Rules

### Immutability

Annotations are **immutable once submitted**. If a reviewer changes their mind:
- They submit a new annotation on the same trace.
- The old annotation remains.
- The platform does not deduplicate. The consumer (developer or dataset tooling) decides which annotation to use when converting.

This is intentional: the history of what reviewers said at each point in time is itself valuable information.

### Minimum content

An annotation must contain at least one of: `label`, `correction`, or `notes`. An annotation with none of these has no signal and is rejected with `400 INVALID_REQUEST` (error code `EMPTY_ANNOTATION`).

### Scope

- An annotation may be scoped to a **trace** (by `trace_id` alone) — applying to the overall request.
- An annotation may be scoped to a **specific span** within a trace (by both `trace_id` and `span_id`) — useful when a specific step (e.g., a retrieval span) was the problem.
- `span_id` must belong to the specified `trace_id`. Mismatched references are rejected: `422 INVALID_ANNOTATION_SCOPE`.

### Annotator identity

`annotator` is a **free string** — an email address, a username, a system name, anything. The platform does not validate or authenticate annotators separately from the API token. Access control for annotation is implementation-defined.

### Multiple annotations

Multiple annotations may exist for the same trace (from different reviewers, or multiple passes by the same reviewer). There is no deduplication. All annotations are returned when listing by `trace_id`.

---

## Annotation → DatasetItem Conversion

The conversion endpoint (`POST /v1/annotations/:id/to-dataset-item`) creates a new DatasetItem from an annotation. This is the primary mechanism for turning production failures into eval test cases.

**Request body:**

| Field | Type | Required | Description |
|---|---|---|---|
| `dataset_id` | string | yes | The dataset to add the item to |

**Conversion mapping:**

| DatasetItem field | Source |
|---|---|
| `dataset_id` | From request body |
| `input` | `input` of the trace's root span |
| `expected_output` | `correction` from the annotation (may be `null` if no correction) |
| `metadata.source_trace_id` | `trace_id` from the annotation |
| `metadata.source_annotation_id` | `id` of the annotation |
| `metadata.annotator` | `annotator` from the annotation |

**Behavior:**
- If the referenced trace has no root span (partial trace), the conversion returns `422 NO_ROOT_SPAN`.
- If `correction` is absent, `expected_output` is `null` on the created item.
- The same annotation may be converted multiple times (into the same or different datasets). Each conversion creates a new DatasetItem. There is no deduplication check.
- Converting an annotation does not modify the annotation.

---

## Minimum UI Contract

A compliant annotation UI must provide the following capabilities for each trace being reviewed:

| Capability | Description |
|---|---|
| **View input** | Display the root span's `input` in a human-readable format (not raw JSON by default) |
| **View output** | Display the root span's `output` in a human-readable format |
| **View span tree** | Allow drilling into individual spans and their inputs/outputs |
| **Add label** | A text field or dropdown to assign a categorical label to the trace or a span |
| **Add correction** | A text or structured input field to provide a corrected output |
| **Add notes** | A free-text field for open-ended comments |
| **Submit** | Submit the annotation to `POST /v1/annotations` |
| **View existing annotations** | Show previously submitted annotations for the same trace |

The UI may offer additional features (queuing, bulk review, filtering). These are not constrained by this spec.

---

## Acceptance Criteria

```
GIVEN a valid trace T1 whose root span has input "What is the capital of France?"
WHEN an annotation is submitted with trace_id=T1, annotator="alice@example.com", correction="Paris"
THEN the annotation is created and returned with all submitted fields
AND created_at is set

GIVEN an annotation submitted with no label, no correction, and no notes
WHEN the annotation is submitted
THEN the API returns 400 with error code EMPTY_ANNOTATION

GIVEN a trace T1 with two annotations from different annotators
WHEN annotations are listed for trace_id=T1
THEN both annotations are returned with their respective annotator values

GIVEN an annotation A1 with correction="Paris" on trace T1
WHEN A1 is converted to a dataset item in dataset D1
THEN a DatasetItem is created with:
  - input equal to T1's root span input
  - expected_output="Paris"
  - metadata.source_trace_id=T1
  - metadata.source_annotation_id=A1

GIVEN an annotation A1 with no correction on trace T1
WHEN A1 is converted to a dataset item in dataset D1
THEN a DatasetItem is created with expected_output=null
AND no error is returned

GIVEN annotation A1 is converted to a DatasetItem in dataset D1
WHEN A1 is converted again to the same dataset D1
THEN a second DatasetItem is created (no deduplication)
AND the original DatasetItem is not modified

GIVEN a span span-B belonging to trace T1
WHEN an annotation is submitted with trace_id=T1 and span_id=span-B
THEN the annotation is created scoped to span-B

GIVEN span-Z that belongs to trace T2 (not T1)
WHEN an annotation is submitted with trace_id=T1 and span_id=span-Z
THEN the API returns 422 with error code INVALID_ANNOTATION_SCOPE

GIVEN an annotation A1 that was submitted
WHEN the annotation is fetched by its ID
THEN the annotation is returned unchanged (immutable)

GIVEN a trace with no root span (partial trace: spans were submitted but all have parent_span_id set)
WHEN an annotation on that trace is converted to a dataset item
THEN the API returns 422 with error code NO_ROOT_SPAN

GIVEN a trace T1 that does not exist
WHEN an annotation is submitted with trace_id=T1
THEN the API returns 404 with error code NOT_FOUND
```

---

## Edge Cases

| Scenario | Expected Behavior |
|---|---|
| Annotation on a trace that was deleted | `404 NOT_FOUND` at submission time |
| Converting an annotation whose trace was deleted after annotation creation | `404 NOT_FOUND` with a message indicating the trace no longer exists |
| Annotation with only `notes` (no label or correction) | Valid — notes alone are meaningful signal |
| Annotation `label` is an empty string | `400 INVALID_REQUEST` — if present, label must be non-empty |
| `annotator` is an empty string | `400 INVALID_REQUEST` — annotator must be non-empty |
| Listing annotations for a trace with no annotations | Returns empty `items` array with `next_cursor: null` |
| Converting the same annotation to a dataset 100 times | 100 DatasetItems are created; no error |
