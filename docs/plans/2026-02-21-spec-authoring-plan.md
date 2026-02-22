# AI Observability Spec — Authoring Plan

> **Goal:** Produce a complete, spec-anchored behavioral specification for a lightweight AI observability platform. The spec is the living source of truth that drives implementations and evolves with the system.

---

## Methodology: Spec-Anchored Platform Spec

We are following the **spec-anchored** model (Kiro / GitHub Spec-Kit pattern), not the `whenwords` "spec-as-product" model. Key distinctions:

- The spec describes a **stateful platform with HTTP APIs**, not a pure-function library
- The spec is **kept and maintained** as the system evolves — it is not a one-time generation prompt
- Each subsystem gets a **behavioral spec** (user stories + acceptance criteria + data contracts)
- A **constitution** defines cross-cutting concerns that apply everywhere
- **No language-agnostic `tests.yaml`** — that pattern works for deterministic library functions, not stateful API behavior. Acceptance criteria in Given/When/Then format serve this role instead.

---

## File Structure

```
ai-observability-spec/
├── README.md                        # What this spec is; how to use it; how to implement
├── CONSTITUTION.md                  # Cross-cutting principles: auth, errors, pagination, versioning
├── DATA-MODEL.md                    # Canonical entity definitions and relationships
├── specs/
│   ├── 01-tracing.md                # Tracing: spans, traces, OTel alignment, ingestion
│   ├── 02-datasets.md               # Datasets: structure, versioning, CRUD, reconciliation
│   ├── 03-experiments.md            # Experiments: lifecycle, runs, comparisons, thresholds
│   ├── 04-scoring.md                # Scorers: types, configuration, custom scorer interface
│   ├── 05-annotation.md             # Annotation: human review workflow, trace → dataset
│   └── 06-api.md                    # HTTP API contract: all endpoints, schemas, errors
├── docs/
│   ├── research/
│   │   └── findings.md              # Research notes (preserved context)
│   └── plans/
│       └── 2026-02-21-spec-authoring-plan.md  # This file
```

**Why this structure:**
- `CONSTITUTION.md` is the "memory bank" — auth, error format, pagination, API versioning apply to every spec section; defining them once prevents drift
- `DATA-MODEL.md` is the shared entity graph all sub-specs reference; written before any behavioral spec
- `specs/` follows a dependency order: tracing is the primitive, datasets depend on traces (annotation), experiments depend on datasets, scoring depends on experiments, annotation depends on tracing + datasets, API assembles everything
- No `schema/` folder with JSON Schema files — schemas live inline in `DATA-MODEL.md` and `specs/06-api.md` as tables and examples, not as separate machine-readable files (keep it human-first)

---

## Spec Authoring Sequence

### Phase 1: Foundation (write these first — everything else references them)

#### Step 1 — `CONSTITUTION.md`
Cross-cutting rules applied universally. Every spec section assumes these without repeating them.

**Contents:**
- **Design principles** — lightweight, OTel-native, implementation-agnostic, YAGNI, composable subsystems
- **Authentication contract** — bearer token required on all endpoints; implementation-defined token issuance
- **Error format** — standard error envelope: `{ "error": { "code": string, "message": string, "details"?: object } }` with HTTP status semantics
- **Pagination contract** — cursor-based; `{ "items": [...], "next_cursor": string | null, "limit": number }`
- **API versioning** — all endpoints under `/v1/`; breaking changes require new version prefix
- **Timestamps** — all timestamps are ISO 8601 UTC strings
- **IDs** — all IDs are opaque strings (UUIDs recommended but not required)
- **Project scoping** — all resources belong to a project; project_id required on all list operations
- **Out of scope for v1** — online evals, prompt management, playground, GPU monitoring, multi-tenancy, SSO

#### Step 2 — `DATA-MODEL.md`
Defines every core entity, its fields, constraints, and how entities relate to each other. Written as prose + field tables (not JSON Schema). All sub-specs reference this document.

**Entities to define:**

**Span**
| Field | Type | Required | Description |
|---|---|---|---|
| id | string | yes | Unique span identifier |
| trace_id | string | yes | Parent trace |
| parent_span_id | string | no | Parent span (null = root) |
| name | string | yes | Human-readable operation name |
| input | any | no | Input to this operation |
| output | any | no | Output from this operation |
| start_time | timestamp | yes | When this span started |
| end_time | timestamp | no | When this span ended (null = in-progress) |
| duration_ms | number | no | Computed from start/end |
| tokens_input | number | no | Input token count (LLM spans) |
| tokens_output | number | no | Output token count (LLM spans) |
| model | string | no | Model identifier (LLM spans) |
| metadata | object | no | Arbitrary key-value pairs |
| error | object | no | Error details if span failed |

**Trace** — id, project_id, root_span_id, spans[], created_at, metadata

**Dataset** — id, project_id, name, description, version (integer, auto-incremented), created_at

**DatasetItem** — id, dataset_id, input (any), expected_output (any, optional), metadata, created_at

**Experiment** — id, project_id, name, dataset_id, created_at, metadata

**ExperimentRun** — id, experiment_id, dataset_item_id, output (any), trace_id (optional), scores[], created_at

**Score** — id, target_id, target_type (run | span), scorer_name, value (number 0–1 or string label), rationale (optional string), created_at

**Annotation** — id, trace_id, span_id (optional), annotator, label (optional), correction (optional any), notes (optional string), created_at

**Relationships:**
- A Trace contains many Spans (tree structure via parent_span_id)
- A Dataset contains many DatasetItems
- An Experiment references one Dataset
- An Experiment contains many ExperimentRuns
- An ExperimentRun references one DatasetItem
- An ExperimentRun has many Scores
- A Span can have many Scores (online scoring)
- An Annotation belongs to a Trace and optionally a Span

---

### Phase 2: Core Behavioral Specs

#### Step 3 — `specs/01-tracing.md`

**User stories:**
- As a developer, I can instrument my application to capture every LLM call, tool invocation, and retrieval step so I can understand what happened when a response was wrong
- As a developer, I can submit spans out-of-order and the platform assembles them into the correct tree
- As a developer, I can view a complete trace with its full span tree in a single API call

**Acceptance criteria (Given/When/Then examples):**
```
GIVEN a span submitted with a valid trace_id and no parent_span_id
WHEN the trace is fetched
THEN that span appears as the root span

GIVEN spans submitted out of order (child before parent)
WHEN all spans for a trace are submitted
THEN the trace assembles correctly with the right parent-child relationships

GIVEN a span with missing required field (name)
WHEN it is submitted via the ingestion API
THEN the API returns 400 with error code INVALID_SPAN
```

**Contents:**
- What a trace captures (full request execution path)
- Span hierarchy rules (one root per trace, parent-child via parent_span_id)
- Required vs optional fields (reference DATA-MODEL.md)
- OpenTelemetry alignment table (GenAI Semantic Conventions → our field names)
- Ingestion behavior: duplicate span_id handling, out-of-order submission, partial traces
- Trace retention: traces are immutable once all spans are submitted
- Edge cases: zero-span traces, circular parent references, missing root span

#### Step 4 — `specs/02-datasets.md`

**User stories:**
- As a developer, I can create a dataset and add input/expected-output pairs to use as eval test cases
- As a developer, I can import items from JSONL to build datasets from existing data
- As a developer, I can save an annotated trace to a dataset so production failures become test cases
- As a developer, I can fork a dataset to create a new version while preserving history

**Acceptance criteria examples:**
```
GIVEN a dataset with 10 items
WHEN 5 new items are added
THEN the dataset contains 15 items and version is incremented

GIVEN a JSONL file where one line is malformed
WHEN bulk import is requested
THEN valid lines are imported, the malformed line is reported in the response, and the dataset is not left in a partial state
```

**Contents:**
- Dataset creation and versioning model
- Item schema and constraints (input required; expected_output optional)
- CRUD operations with behavioral rules
- Bulk import from JSONL: format, error handling, partial success behavior
- Creating a dataset item from an annotation (the conversion contract)
- Dataset reconciliation pattern: incremental updates vs golden dataset
- Empty dataset behavior (valid — experiments on empty datasets return zero runs)

#### Step 5 — `specs/03-experiments.md`

**User stories:**
- As a developer, I can run my application against a dataset and capture the outputs for scoring
- As a developer, I can compare two experiment runs to see which prompt/model change improved quality
- As a developer, I can set a passing threshold and fail a CI check if quality drops below it

**Acceptance criteria examples:**
```
GIVEN an experiment with 50 dataset items
WHEN the experiment is run with a scorer
THEN 50 ExperimentRuns are created, each with at least one Score

GIVEN two experiments on the same dataset
WHEN a comparison is requested
THEN the response includes per-item score delta and aggregate improvement/regression metrics

GIVEN a score threshold of 0.8 and an experiment average score of 0.75
WHEN threshold evaluation is requested
THEN the result is FAIL with the gap reported
```

**Contents:**
- Experiment lifecycle: created → running → completed
- Run submission: one run per dataset item, idempotency rules
- Score attachment: how scores are linked to runs
- Comparison: how two experiment results are diffed (per-item and aggregate)
- Threshold evaluation: pass/fail determination contract
- Behavior when dataset has no items (returns completed with zero runs, no error)

#### Step 6 — `specs/04-scoring.md`

**User stories:**
- As a developer, I can attach a score to any experiment run or span to quantify output quality
- As a developer, I can use a built-in LLM-as-judge scorer with my own prompt template and model
- As a developer, I can use simple rule-based scorers for deterministic quality checks
- As a developer, I can implement a custom scorer and attach it to an experiment

**Acceptance criteria examples:**
```
GIVEN an ExactMatch scorer with case_sensitive: false
WHEN output "Paris" is scored against expected "paris"
THEN score value is 1.0

GIVEN an LLMJudge scorer with a grounding rubric
WHEN a hallucinated response is scored
THEN score value is between 0.0 and 1.0 and rationale is non-empty

GIVEN a Score submitted with value 1.5
WHEN the score is submitted
THEN the API returns 400 with error code INVALID_SCORE_VALUE
```

**Built-in scorers to specify:**
- `ExactMatch` — exact string comparison; options: case_sensitive (default true), strip_whitespace (default true)
- `Contains` — checks if output contains expected string; options: case_sensitive
- `Regex` — checks if output matches a regex pattern
- `LLMJudge` — sends output to a model with a rubric prompt; extracts numeric score from response

**Custom scorer interface:**
- Receives: `{ input, output, expected_output? }`
- Returns: `{ value: number (0-1) | string, rationale?: string }`
- Implementation-defined how custom scorers are registered and invoked

#### Step 7 — `specs/05-annotation.md`

**User stories:**
- As a domain expert, I can review a trace and flag it as problematic without reading JSON
- As a domain expert, I can provide a corrected output for a bad response
- As a developer, I can convert approved annotations into dataset items with one API call

**Acceptance criteria examples:**
```
GIVEN a trace that has been annotated with a correction
WHEN the annotation is converted to a dataset item
THEN a DatasetItem is created with the trace's root input and the annotation's correction as expected_output

GIVEN a trace with two annotations from different annotators
WHEN annotations are listed for that trace
THEN both annotations are returned with their respective annotator identifiers
```

**Contents:**
- Annotation object schema (reference DATA-MODEL.md)
- Who can annotate: no role enforcement in v1 (annotator is a free string)
- Annotation → DatasetItem conversion contract (which fields map where)
- Multiple annotations per trace: allowed; no deduplication; consumer decides which to use
- Annotation immutability: annotations are immutable once submitted; new annotation supersedes
- Minimum UI contract: what a compliant annotation UI must allow (view input/output, add label, add correction, add notes, submit)

---

### Phase 3: API Contract

#### Step 8 — `specs/06-api.md`

The full HTTP API surface. Every endpoint, request body, response body, and error code. References DATA-MODEL.md for object shapes.

**Endpoint table:**

```
# Traces
POST   /v1/traces/ingest              Ingest one or more spans
GET    /v1/traces/:id                 Fetch trace with full span tree
GET    /v1/traces                     List traces (project_id required, paginated)
DELETE /v1/traces/:id                 Delete trace and all spans

# Datasets
POST   /v1/datasets                   Create dataset
GET    /v1/datasets/:id               Fetch dataset metadata
GET    /v1/datasets/:id/items         List items (paginated)
POST   /v1/datasets/:id/items         Add one item
POST   /v1/datasets/:id/items/import  Bulk import from JSONL
DELETE /v1/datasets/:id               Delete dataset and all items
GET    /v1/datasets                   List datasets (project_id required)

# Experiments
POST   /v1/experiments                Create experiment
GET    /v1/experiments/:id            Fetch experiment metadata
POST   /v1/experiments/:id/runs       Submit one or more experiment runs
GET    /v1/experiments/:id/runs       List runs with scores (paginated)
GET    /v1/experiments/:id/summary    Aggregate score summary + pass/fail
GET    /v1/experiments/:id/compare/:other_id  Diff two experiments
GET    /v1/experiments                List experiments (project_id required)

# Scores
POST   /v1/scores                     Submit a score (for run or span)
GET    /v1/scores?target_id=&target_type=  List scores for a target

# Annotations
POST   /v1/annotations                Submit annotation
GET    /v1/annotations?trace_id=      List annotations for a trace
POST   /v1/annotations/:id/to-dataset-item  Convert annotation to dataset item
```

**Contents per endpoint:**
- Method + path
- Request body schema (field, type, required, description)
- Response body schema
- Error codes specific to that endpoint
- Behavioral notes (idempotency, side effects, ordering guarantees)

**Global error codes** (from CONSTITUTION.md):
- `UNAUTHORIZED` — missing or invalid token
- `NOT_FOUND` — resource does not exist
- `INVALID_REQUEST` — malformed request body
- `CONFLICT` — duplicate resource where uniqueness required
- `PROJECT_REQUIRED` — project_id missing on list operations

---

### Phase 4: README

#### Step 9 — `README.md`

**Contents:**
- What this spec is (a behavioral spec for a lightweight AI observability platform)
- What it is NOT (a codebase, a product, a hosted service)
- How to read it: start with CONSTITUTION.md → DATA-MODEL.md → specs/ in order
- How to implement: paste `CONSTITUTION.md` + `DATA-MODEL.md` + the relevant spec file into your AI agent; it has everything needed to produce a compliant implementation of any subsystem
- How to validate: acceptance criteria in each spec file are the test cases; any compliant implementation must satisfy all of them
- What's in scope / out of scope for v1
- License

---

## Quality Checklist (Per Spec File)

Before marking a spec section complete:

- [ ] Every behavior described from the user's perspective first (user story format)
- [ ] Every behavior has at least one Given/When/Then acceptance criterion
- [ ] Every edge case is explicitly documented (invalid input, empty state, missing optional fields)
- [ ] No implementation details (no "use PostgreSQL", no specific framework, no "use Redis")
- [ ] All object references point to DATA-MODEL.md
- [ ] A human can read the section end-to-end in under 10 minutes
- [ ] Cross-cutting concerns (auth, errors, pagination) are not re-specified — CONSTITUTION.md is referenced

---

## Design Principles (from CONSTITUTION.md)

These must be upheld in every spec decision:

1. **Lightweight** — no required external services beyond storage; single deployable unit is the target
2. **OpenTelemetry-aligned** — tracing schema maps to OTel GenAI Semantic Conventions
3. **Implementation-agnostic** — spec describes behavior, not technology choices
4. **Composable** — each subsystem (tracing, datasets, evals) is independently useful
5. **Human-first** — specs are read and validated by humans; machine-readability is secondary
6. **Minimal surface area** — every feature justifies its inclusion; YAGNI ruthlessly

---

## What's Out of Scope (v1)

Explicitly deferred to keep the spec tight and focused:

- Online evals (scoring live production traffic in real-time)
- Prompt management and versioning
- Playground / interactive testing UI
- GPU and infrastructure monitoring
- Kubernetes operator / auto-instrumentation injection
- Multi-tenancy / organizations / teams
- SSO / SAML / OAuth provider integration
- Billing, usage limits, rate limiting
- Webhooks and alerting
- Fine-tuning, RLHF, or training data pipelines
