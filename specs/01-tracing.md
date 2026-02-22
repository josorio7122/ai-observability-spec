# Tracing

> Captures the full execution path of a user request — every LLM call, tool invocation, retrieval step, and intermediate operation — so developers can understand exactly what happened when a response was wrong.

Cross-cutting rules (auth, errors, pagination, timestamps) are defined in `CONSTITUTION.md`. Object schemas are defined in `DATA-MODEL.md`.

---

## Overview

Traditional observability answers "is it running?" Tracing in an AI observability platform answers "what actually happened?" When a user gets a bad response, tracing shows which documents were retrieved, what prompt was constructed, which model was called with what parameters, and where the execution degraded.

A **trace** is the complete record of one user request. It contains one or more **spans** — each span representing a single instrumented operation. Spans form a tree: one root span represents the top-level request, and child spans represent sub-operations.

---

## User Stories

- As a developer, I can instrument my application to capture every LLM call, tool invocation, and retrieval step so I can understand what happened when a response was wrong.
- As a developer, I can submit spans from multiple parts of my application (possibly out of order) and the platform assembles them into the correct tree.
- As a developer, I can view a complete trace with its full span tree in a single API call.
- As a developer, I can search and filter traces by project, time range, and metadata to find relevant examples.

---

## Span Hierarchy

Every span in a trace carries a `trace_id` and an optional `parent_span_id`.

- A span with no `parent_span_id` is the **root span** of the trace.
- A trace may have **at most one root span**.
- All other spans must reference a `parent_span_id` that belongs to the same trace.
- The resulting structure is a **tree** rooted at the root span.

```
Trace T1
 └── Span A (root)  name="handle_user_query"
      ├── Span B    name="vector_search"
      ├── Span C    name="llm_call"  model="gpt-4o"
      │    └── Span D  name="tool:weather_api"
      └── Span E    name="format_response"
```

---

## Ingestion Behavior

### Out-of-order submission

Spans may arrive in any order. The platform assembles the tree based on `trace_id` and `parent_span_id` regardless of submission order. A span referencing a `parent_span_id` that has not yet been received is held and linked once the parent arrives.

### Duplicate span IDs

If a span with an already-ingested `id` is submitted again for the same trace, the platform returns `409 CONFLICT` with error code `DUPLICATE_SPAN`. The original span is not modified.

### Partial traces

A trace with spans submitted but no root span is valid and queryable. It is a partial trace. Implementations must not reject or discard partial traces.

### Span ID assignment

Span `id` values are **assigned by the client** before submission, not by the server. This allows the client to link parent and child spans correctly before any network round-trips.

### Batch ingestion

The ingestion endpoint accepts a **batch of one or more spans** in a single request. All spans in the batch are processed atomically: either all are accepted or the entire batch is rejected. If any span in the batch is invalid, the error response identifies which span(s) failed.

---

## Required vs Optional Fields

See `DATA-MODEL.md` for full field definitions. For ingestion purposes:

**Required fields (missing any of these → `400 INVALID_SPAN`):**
- `id`
- `trace_id`
- `name`
- `start_time`

**Optional fields:**
- `parent_span_id` — absent means root span
- `input`, `output` — may be any JSON value or absent
- `end_time` — absent means in-progress
- `tokens_input`, `tokens_output`, `model` — for LLM spans
- `metadata` — arbitrary key-value context
- `error` — present if the span failed

---

## OpenTelemetry Alignment

This spec's span fields map to [OpenTelemetry GenAI Semantic Conventions](https://opentelemetry.io/docs/specs/semconv/gen-ai/). Implementations that accept OTel-formatted spans should translate fields according to this table:

| This spec | OTel GenAI Convention | Notes |
|---|---|---|
| `name` | span name | Direct mapping |
| `start_time` | `startTimeUnixNano` | Convert nanoseconds → ISO 8601 |
| `end_time` | `endTimeUnixNano` | Convert nanoseconds → ISO 8601 |
| `model` | `gen_ai.request.model` | |
| `tokens_input` | `gen_ai.usage.input_tokens` | |
| `tokens_output` | `gen_ai.usage.output_tokens` | |
| `input` | `gen_ai.prompt` (events) | May be structured differently in OTel |
| `output` | `gen_ai.completion` (events) | May be structured differently in OTel |
| `error.message` | `exception.message` | |
| `error.type` | `exception.type` | |
| `error.stack` | `exception.stacktrace` | |
| `metadata` | span attributes | Arbitrary attributes become metadata |

Implementations are encouraged to accept OTel-formatted spans natively and translate at ingestion time, but this is not required for spec compliance.

---

## Trace Immutability

- Submitted spans **cannot be modified or deleted individually** after ingestion.
- A trace (and all its spans) may be deleted as a whole via the delete endpoint — this is a hard delete with no recovery.
- Implementations may choose to support a retention policy that automatically deletes traces after a configurable period; this is implementation-defined.

---

## Acceptance Criteria

```
GIVEN a span submitted with a valid trace_id and no parent_span_id
WHEN the trace is fetched
THEN that span appears as the root span of the trace

GIVEN spans submitted in this order: child (B, parent=A), then parent (A, root)
WHEN the trace is fetched after both spans are submitted
THEN the trace tree shows A as root with B as its child

GIVEN a span missing the required field "name"
WHEN it is submitted via the ingestion API
THEN the API returns 400 with error code INVALID_SPAN
AND the response details identify "name" as the missing field

GIVEN a span with id "span-123" already ingested in trace "T1"
WHEN another span with id "span-123" is submitted for trace "T1"
THEN the API returns 409 with error code DUPLICATE_SPAN
AND the original span is unchanged

GIVEN a batch of 3 spans where the 2nd span is missing a required field
WHEN the batch is submitted
THEN the API returns 400 with error code INVALID_SPAN
AND no spans from the batch are persisted

GIVEN a trace that has spans but no root span (all have parent_span_id set)
WHEN the trace is fetched
THEN the trace is returned with a null root_span_id
AND all spans are included in the response

GIVEN a span with parent_span_id referencing a span in a different trace
WHEN the span is submitted
THEN the API returns 400 with error code INVALID_SPAN_PARENT

GIVEN a trace that exists
WHEN it is deleted
THEN the trace and all its spans are permanently removed
AND a subsequent GET for that trace_id returns 404
```

---

## Edge Cases

| Scenario | Expected Behavior |
|---|---|
| Trace with zero spans | Cannot exist — a trace is created by ingesting its first span |
| Span referencing a non-existent `parent_span_id` (from a different trace) | Rejected: `INVALID_SPAN_PARENT` |
| Circular parent reference (A → B → A) | Rejected on the span that creates the cycle: `CIRCULAR_SPAN_REFERENCE` |
| `end_time` before `start_time` | Rejected: `INVALID_SPAN` with details indicating the time ordering issue |
| `duration_ms` provided but inconsistent with start/end | Implementation may recompute or return `INVALID_SPAN`; must document which |
| Trace ingested across multiple batches | Valid; the trace is assembled from all received spans |
| Metadata value that is a nested object | Rejected: `INVALID_SPAN` — metadata values must be strings, numbers, booleans, or `null` |
