# Constitution

> Cross-cutting rules that apply to every part of this spec. All sub-specs in `specs/` assume these without repeating them.

---

## Design Principles

1. **Lightweight** — No required external services beyond a single storage backend. The platform must be deployable as a single unit without mandatory microservices, message queues, or cloud dependencies.
2. **OpenTelemetry-aligned** — Tracing primitives map to OTel GenAI Semantic Conventions. Instrumentation code written once should work with this platform and any OTel-compatible backend.
3. **Implementation-agnostic** — This spec describes *behavior*, not technology. It does not prescribe a programming language, database, framework, or deployment model.
4. **Composable subsystems** — Each subsystem (tracing, datasets, experiments, scoring, annotation) is independently useful. An implementation may choose to support a subset.
5. **Human-first** — Specs are written to be read and validated by humans. Machine-readability is a secondary concern.
6. **Minimal surface area** — Every feature must justify its inclusion. When in doubt, leave it out.

---

## Authentication

- All API endpoints require a bearer token in the `Authorization` header:
  ```
  Authorization: Bearer <token>
  ```
- The platform does not specify how tokens are issued or validated — this is implementation-defined.
- Requests missing a valid token receive `401 Unauthorized` with error code `UNAUTHORIZED`.
- Token scoping (per-project, per-user, read-only, etc.) is implementation-defined and out of scope for this spec.

---

## Error Format

All error responses use a consistent envelope regardless of status code:

```json
{
  "error": {
    "code": "SNAKE_CASE_ERROR_CODE",
    "message": "Human-readable description of what went wrong.",
    "details": {}
  }
}
```

- `code` — machine-readable, stable identifier for the error type (see global codes below)
- `message` — human-readable explanation; may vary across implementations
- `details` — optional object with additional context (e.g., which field failed validation)

### HTTP Status → Semantic Mapping

| Status | When to use |
|---|---|
| `200 OK` | Successful read or update |
| `201 Created` | Resource successfully created |
| `400 Bad Request` | Malformed request body or invalid field values |
| `401 Unauthorized` | Missing or invalid authentication token |
| `404 Not Found` | Resource does not exist |
| `409 Conflict` | Duplicate resource where uniqueness is required |
| `422 Unprocessable Entity` | Request is well-formed but semantically invalid |

### Global Error Codes

| Code | HTTP Status | Meaning |
|---|---|---|
| `UNAUTHORIZED` | 401 | Missing or invalid bearer token |
| `NOT_FOUND` | 404 | Requested resource does not exist |
| `INVALID_REQUEST` | 400 | Malformed or missing required fields in request body |
| `CONFLICT` | 409 | A resource with a conflicting unique identifier already exists |
| `PROJECT_REQUIRED` | 400 | `project_id` is missing on a list operation that requires it |
| `INVALID_SCORE_VALUE` | 400 | Score value is outside the allowed range or type |
| `INVALID_SPAN` | 400 | Span is missing required fields or has invalid structure |

Additional error codes specific to individual endpoints are defined in `specs/06-api.md`.

---

## Pagination

All list endpoints that may return multiple items use **cursor-based pagination**.

### Request parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `limit` | integer | 50 | Maximum number of items to return. Max value: 200. |
| `cursor` | string | — | Opaque cursor from a previous response's `next_cursor`. Omit for first page. |

### Response envelope

```json
{
  "items": [...],
  "next_cursor": "opaque-string-or-null",
  "limit": 50
}
```

- `next_cursor` is `null` when there are no more items.
- Cursors are opaque strings. Implementations may encode them however they choose (e.g., base64 of an offset, a keyset value).
- Cursor values are not stable across schema changes or data deletions.

---

## Timestamps

- All timestamps in request and response bodies are **ISO 8601 strings in UTC**.
- Format: `"2026-02-21T22:00:00.000Z"`
- Implementations must accept and produce timestamps in this format.
- Millisecond precision is recommended but not required.

---

## IDs

- All resource IDs are **opaque strings**.
- UUID v4 is the recommended format but implementations are not required to use it.
- IDs are case-sensitive.
- IDs must be unique within their resource type.
- IDs are assigned by the server on creation and returned in the response. Clients may not specify their own IDs unless explicitly stated in the endpoint spec.

---

## Project Scoping

- All resources belong to a **project**, identified by `project_id`.
- `project_id` is a required field on all resources.
- All list endpoints require `project_id` as a query parameter.
- Cross-project operations (e.g., comparing experiments from different projects) are not supported in v1.
- Project creation and management are implementation-defined and out of scope for this spec.

---

## Request and Response Format

- All request and response bodies are **JSON** (`Content-Type: application/json`).
- `null` values for optional fields may be omitted from responses; clients must treat missing optional fields the same as `null`.
- Unknown fields in request bodies must be ignored (forward compatibility).
- Unknown fields in response bodies must be ignored by clients (forward compatibility).

---

## API Versioning

- All endpoints are prefixed with `/v1/`.
- Breaking changes (removed fields, changed semantics, new required fields) require a new version prefix (e.g., `/v2/`).
- Additive changes (new optional fields, new endpoints) do not require a version bump.

---

## Out of Scope for v1

The following are explicitly excluded from this specification. They may be addressed in future versions:

- **Online evals** — scoring live production traffic in real-time
- **Prompt management** — versioned storage and testing of prompt templates
- **Playground** — interactive prompt and model experimentation UI
- **GPU and infrastructure monitoring** — hardware metrics
- **Kubernetes operator / auto-instrumentation** — automatic code injection
- **Multi-tenancy** — organizations, teams, member management
- **SSO / SAML / OAuth provider integration**
- **Billing, usage limits, rate limiting**
- **Webhooks and alerting**
- **Fine-tuning, RLHF, or training data pipelines**
