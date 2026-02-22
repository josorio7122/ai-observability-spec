# HTTP API Contract

> The complete surface of the platform's HTTP API. Every endpoint, its request and response shapes, error codes, and behavioral notes. All object field definitions are in `DATA-MODEL.md`. All cross-cutting conventions (auth, errors, pagination, timestamps) are in `CONSTITUTION.md`.

---

## Conventions

- All endpoints are prefixed with `/v1/`.
- All request and response bodies are JSON (`Content-Type: application/json`).
- All list endpoints support `cursor` and `limit` query parameters (see `CONSTITUTION.md`).
- All endpoints require `Authorization: Bearer <token>`.
- Error responses follow the envelope defined in `CONSTITUTION.md`.

---

## Traces

### `POST /v1/traces/ingest`

Ingest one or more spans. Creates a trace implicitly if the `trace_id` has not been seen before.

**Request body:**

```json
{
  "spans": [ <Span>, ... ]
}
```

`spans` must contain at least one span. Maximum batch size is implementation-defined (recommended: 1000 spans per request).

**Response `201 Created`:**

```json
{
  "accepted": 3,
  "trace_ids": ["trace-abc", "trace-def"]
}
```

`trace_ids` lists all distinct trace IDs seen in the batch.

**Error codes:**

| Code | Status | When |
|---|---|---|
| `INVALID_SPAN` | 400 | Any span is missing required fields or has invalid structure |
| `DUPLICATE_SPAN` | 409 | A span ID already exists within the same trace |
| `INVALID_SPAN_PARENT` | 400 | `parent_span_id` references a span in a different trace |
| `CIRCULAR_SPAN_REFERENCE` | 400 | Span creates a circular parent reference |

**Notes:**
- The entire batch is rejected if any span is invalid (atomic behavior).
- `id` is assigned by the client, not the server.

---

### `GET /v1/traces/:id`

Fetch a trace with its full span tree.

**Response `200 OK`:**

```json
{
  "id": "trace-abc",
  "project_id": "proj-1",
  "root_span_id": "span-root",
  "spans": [ <Span>, ... ],
  "created_at": "2026-02-21T22:00:00.000Z",
  "metadata": {}
}
```

Spans are ordered by `start_time` ascending.

**Error codes:** `NOT_FOUND`

---

### `GET /v1/traces`

List traces for a project.

**Query parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `project_id` | string | yes | Filter to this project |
| `cursor` | string | no | Pagination cursor |
| `limit` | integer | no | Default 50, max 200 |
| `after` | timestamp | no | Only traces created after this time |
| `before` | timestamp | no | Only traces created before this time |

**Response `200 OK`:** Paginated list of Trace objects (without `spans` array — use `GET /v1/traces/:id` to fetch spans).

```json
{
  "items": [
    {
      "id": "trace-abc",
      "project_id": "proj-1",
      "root_span_id": "span-root",
      "span_count": 4,
      "created_at": "2026-02-21T22:00:00.000Z",
      "metadata": {}
    }
  ],
  "next_cursor": null,
  "limit": 50
}
```

**Error codes:** `PROJECT_REQUIRED`

---

### `DELETE /v1/traces/:id`

Permanently delete a trace and all its spans.

**Response `200 OK`:**

```json
{ "deleted": true, "id": "trace-abc" }
```

**Error codes:** `NOT_FOUND`

---

## Datasets

### `POST /v1/datasets`

Create a new dataset.

**Request body:**

```json
{
  "project_id": "proj-1",
  "name": "customer-support-baseline",
  "description": "Optional description"
}
```

**Response `201 Created`:**

```json
{
  "id": "ds-1",
  "project_id": "proj-1",
  "name": "customer-support-baseline",
  "description": "Optional description",
  "version": 1,
  "item_count": 0,
  "created_at": "2026-02-21T22:00:00.000Z",
  "updated_at": "2026-02-21T22:00:00.000Z"
}
```

**Error codes:** `INVALID_REQUEST` (missing `name` or `project_id`), `CONFLICT` (duplicate name in project)

---

### `GET /v1/datasets/:id`

Fetch dataset metadata (not items).

**Response `200 OK`:** Dataset object (without items).

**Error codes:** `NOT_FOUND`

---

### `GET /v1/datasets`

List datasets for a project.

**Query parameters:** `project_id` (required), `cursor`, `limit`

**Response `200 OK`:** Paginated list of Dataset objects.

**Error codes:** `PROJECT_REQUIRED`

---

### `GET /v1/datasets/:id/items`

List items in a dataset.

**Query parameters:** `cursor`, `limit`

**Response `200 OK`:**

```json
{
  "items": [ <DatasetItem>, ... ],
  "next_cursor": "cursor-xyz",
  "limit": 50
}
```

**Error codes:** `NOT_FOUND`

---

### `POST /v1/datasets/:id/items`

Add a single item to a dataset.

**Request body:**

```json
{
  "input": "What is the capital of France?",
  "expected_output": "Paris",
  "metadata": { "source": "manual" }
}
```

`input` is required. `expected_output` and `metadata` are optional.

**Response `201 Created`:** DatasetItem object.

**Error codes:** `NOT_FOUND` (dataset), `INVALID_REQUEST` (missing `input`)

---

### `POST /v1/datasets/:id/items/import`

Bulk import items from a JSONL body.

**Request:**
- `Content-Type: application/x-ndjson` (or `application/jsonl`)
- Body: newline-delimited JSON, one DatasetItem per line

**Response `200 OK`:**

```json
{
  "imported_count": 8,
  "skipped_count": 2,
  "skipped": [
    { "line": 3, "reason": "Missing required field: input" },
    { "line": 7, "reason": "Invalid JSON" }
  ]
}
```

**Error codes:** `NOT_FOUND` (dataset)

**Notes:**
- Returns `200` (not `201`) because it is a mixed-result operation.
- If all lines are invalid, `imported_count` is 0 and no version increment occurs.

---

### `DELETE /v1/datasets/:id`

Delete a dataset and all its items.

**Response `200 OK`:**

```json
{ "deleted": true, "id": "ds-1" }
```

**Error codes:** `NOT_FOUND`

**Notes:** Experiments that reference this dataset are not affected (see `specs/02-datasets.md`).

---

## Experiments

### `POST /v1/experiments`

Create a new experiment.

**Request body:**

```json
{
  "project_id": "proj-1",
  "name": "gpt-4o-baseline",
  "dataset_id": "ds-1",
  "metadata": { "model": "gpt-4o", "prompt_version": "v3" }
}
```

**Response `201 Created`:** Experiment object with `status: "created"`.

**Error codes:** `INVALID_REQUEST`, `NOT_FOUND` (dataset)

---

### `GET /v1/experiments/:id`

Fetch experiment metadata.

**Response `200 OK`:** Experiment object.

**Error codes:** `NOT_FOUND`

---

### `GET /v1/experiments`

List experiments for a project.

**Query parameters:** `project_id` (required), `cursor`, `limit`

**Response `200 OK`:** Paginated list of Experiment objects.

**Error codes:** `PROJECT_REQUIRED`

---

### `POST /v1/experiments/:id/runs`

Submit one or more experiment runs.

**Request body:**

```json
{
  "runs": [
    {
      "dataset_item_id": "item-1",
      "output": "Paris",
      "trace_id": "trace-abc",
      "scores": [
        {
          "scorer_name": "exact_match",
          "config": { "case_sensitive": false },
          "value": 1.0
        }
      ]
    }
  ]
}
```

`scores` is optional. If provided inline, each score must include `scorer_name` and `value`. `config` is optional scorer configuration (for built-in scorers).

**Response `201 Created`:**

```json
{
  "accepted": 1,
  "run_ids": ["run-xyz"]
}
```

**Error codes:**

| Code | Status | When |
|---|---|---|
| `DUPLICATE_RUN` | 409 | A run for the same `dataset_item_id` already exists in this experiment |
| `INVALID_DATASET_ITEM` | 422 | `dataset_item_id` does not belong to the experiment's dataset |
| `EXPERIMENT_COMPLETED` | 422 | Experiment status is `completed` |
| `INVALID_SCORE_VALUE` | 400 | Inline score value is out of range |

---

### `GET /v1/experiments/:id/runs`

List runs with their scores.

**Query parameters:** `cursor`, `limit`, `scorer_name` (optional filter)

**Response `200 OK`:**

```json
{
  "items": [
    {
      "id": "run-xyz",
      "experiment_id": "exp-1",
      "dataset_item_id": "item-1",
      "output": "Paris",
      "trace_id": "trace-abc",
      "scores": [ <Score>, ... ],
      "created_at": "2026-02-21T22:00:00.000Z"
    }
  ],
  "next_cursor": null,
  "limit": 50
}
```

**Error codes:** `NOT_FOUND`

---

### `GET /v1/experiments/:id/summary`

Aggregate quality summary for an experiment.

**Response `200 OK`:**

```json
{
  "experiment_id": "exp-1",
  "status": "completed",
  "run_count": 50,
  "dataset_item_count": 50,
  "scores_by_scorer": {
    "exact_match": {
      "scorer_name": "exact_match",
      "scored_run_count": 50,
      "mean": 0.82,
      "min": 0.0,
      "max": 1.0,
      "distribution": null
    }
  },
  "threshold_result": null
}
```

**Error codes:** `NOT_FOUND`

---

### `POST /v1/experiments/:id/complete`

Explicitly mark an experiment as completed.

**Request body:** empty (`{}`)

**Response `200 OK`:** Updated Experiment object with `status: "completed"`.

**Error codes:** `NOT_FOUND`, `EXPERIMENT_COMPLETED` (already completed)

---

### `GET /v1/experiments/:id/compare/:other_id`

Compare two experiments on the same dataset.

**Response `200 OK`:**

```json
{
  "base_experiment_id": "exp-1",
  "compare_experiment_id": "exp-2",
  "scorer_comparisons": [
    {
      "scorer_name": "exact_match",
      "base_mean": 0.72,
      "compare_mean": 0.85,
      "delta": 0.13,
      "improved_count": 9,
      "regressed_count": 2,
      "unchanged_count": 39,
      "only_in_base": 0,
      "only_in_compare": 0
    }
  ],
  "per_item_results": [
    {
      "dataset_item_id": "item-1",
      "scorer_name": "exact_match",
      "base_score": 0.0,
      "compare_score": 1.0,
      "delta": 1.0
    }
  ]
}
```

**Error codes:** `NOT_FOUND`, `INCOMPATIBLE_EXPERIMENTS` (different datasets)

---

### `POST /v1/experiments/:id/threshold`

Evaluate an experiment against a quality threshold.

**Request body:**

```json
{
  "scorer_name": "exact_match",
  "metric": "mean",
  "threshold": 0.80,
  "comparison": "gte"
}
```

`comparison` defaults to `"gte"`. Allowed values: `"gte"`, `"gt"`, `"lte"`, `"lt"`.

**Response `200 OK`:**

```json
{
  "passed": false,
  "actual_value": 0.75,
  "threshold": 0.80,
  "scorer_name": "exact_match",
  "metric": "mean",
  "gap": -0.05
}
```

**Error codes:** `NOT_FOUND`, `UNSUPPORTED_THRESHOLD_TYPE` (categorical scorer)

---

## Scores

### `POST /v1/scores`

Submit a score for an experiment run or a span.

**Request body:**

```json
{
  "target_id": "run-xyz",
  "target_type": "run",
  "scorer_name": "human",
  "value": 0.9,
  "rationale": "Response was accurate but slightly verbose."
}
```

`rationale` is optional. `value` may be a number (`0.0`–`1.0`) or a non-empty string (categorical label).

**Response `201 Created`:** Score object.

**Error codes:**

| Code | Status | When |
|---|---|---|
| `INVALID_SCORE_VALUE` | 400 | Numeric value outside `[0.0, 1.0]` or empty string |
| `NOT_FOUND` | 404 | `target_id` does not exist |
| `INVALID_REQUEST` | 400 | `target_type` is not `"run"` or `"span"` |
| `INVALID_SCORER_CONFIG` | 400 | Invalid scorer configuration (e.g., invalid regex pattern) |

---

### `GET /v1/scores`

List scores for a target.

**Query parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `target_id` | string | yes | ID of the run or span |
| `target_type` | string | yes | `"run"` or `"span"` |
| `scorer_name` | string | no | Filter to a specific scorer |
| `cursor` | string | no | |
| `limit` | integer | no | |

**Response `200 OK`:** Paginated list of Score objects.

**Error codes:** `INVALID_REQUEST` (missing `target_id` or `target_type`)

---

## Annotations

### `POST /v1/annotations`

Submit a new annotation on a trace or span.

**Request body:**

```json
{
  "trace_id": "trace-abc",
  "span_id": "span-B",
  "annotator": "alice@example.com",
  "label": "hallucination",
  "correction": "The correct answer is Paris, not Lyon.",
  "notes": "Model confused capital with second-largest city."
}
```

`span_id` is optional. At least one of `label`, `correction`, `notes` must be present.

**Response `201 Created`:** Annotation object.

**Error codes:**

| Code | Status | When |
|---|---|---|
| `NOT_FOUND` | 404 | `trace_id` does not exist |
| `INVALID_ANNOTATION_SCOPE` | 422 | `span_id` does not belong to the specified `trace_id` |
| `EMPTY_ANNOTATION` | 400 | None of `label`, `correction`, `notes` provided |
| `INVALID_REQUEST` | 400 | `annotator` is empty, or `label` is an empty string |

---

### `GET /v1/annotations`

List annotations for a trace.

**Query parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `trace_id` | string | yes | Trace to fetch annotations for |
| `span_id` | string | no | Filter to a specific span |
| `cursor` | string | no | |
| `limit` | integer | no | |

**Response `200 OK`:** Paginated list of Annotation objects.

**Error codes:** `INVALID_REQUEST` (missing `trace_id`)

---

### `POST /v1/annotations/:id/to-dataset-item`

Convert an annotation into a DatasetItem.

**Request body:**

```json
{
  "dataset_id": "ds-1"
}
```

**Response `201 Created`:** DatasetItem object.

**Error codes:**

| Code | Status | When |
|---|---|---|
| `NOT_FOUND` | 404 | Annotation or dataset does not exist |
| `NO_ROOT_SPAN` | 422 | The annotation's trace has no root span |

---

## Error Code Reference

Complete list of all error codes used across the API:

| Code | Status | Defined In |
|---|---|---|
| `UNAUTHORIZED` | 401 | CONSTITUTION.md |
| `NOT_FOUND` | 404 | CONSTITUTION.md |
| `INVALID_REQUEST` | 400 | CONSTITUTION.md |
| `CONFLICT` | 409 | CONSTITUTION.md |
| `PROJECT_REQUIRED` | 400 | CONSTITUTION.md |
| `INVALID_SCORE_VALUE` | 400 | specs/04-scoring.md |
| `INVALID_SPAN` | 400 | specs/01-tracing.md |
| `DUPLICATE_SPAN` | 409 | specs/01-tracing.md |
| `INVALID_SPAN_PARENT` | 400 | specs/01-tracing.md |
| `CIRCULAR_SPAN_REFERENCE` | 400 | specs/01-tracing.md |
| `DUPLICATE_RUN` | 409 | specs/03-experiments.md |
| `INVALID_DATASET_ITEM` | 422 | specs/03-experiments.md |
| `EXPERIMENT_COMPLETED` | 422 | specs/03-experiments.md |
| `INCOMPATIBLE_EXPERIMENTS` | 422 | specs/03-experiments.md |
| `UNSUPPORTED_THRESHOLD_TYPE` | 422 | specs/03-experiments.md |
| `INVALID_SCORER_CONFIG` | 400 | specs/04-scoring.md |
| `EMPTY_ANNOTATION` | 400 | specs/05-annotation.md |
| `INVALID_ANNOTATION_SCOPE` | 422 | specs/05-annotation.md |
| `NO_ROOT_SPAN` | 422 | specs/05-annotation.md |
