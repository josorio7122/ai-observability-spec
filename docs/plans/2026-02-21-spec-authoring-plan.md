# AI Observability Spec — Authoring Plan

> **Goal:** Produce a complete, language-agnostic spec for a lightweight AI observability platform, following the spec-driven development methodology exemplified by `whenwords`.

---

## What We're Building

A spec (not code) that fully describes a **lightweight AI observability platform**. The spec is the deliverable. Any engineer (or AI agent) should be able to read it and implement the platform in their language/stack of choice.

The spec covers: tracing, datasets, evals/experiments, scoring, annotation, and a minimal UI contract — the core loop of "instrument → evaluate → improve."

---

## File Structure

```
ai-observability-spec/
├── README.md                    # What this repo is; how to use the spec
├── SPEC.md                      # Master spec (overview + principles + data model)
├── specs/
│   ├── 01-tracing.md            # Tracing: spans, traces, OpenTelemetry alignment
│   ├── 02-datasets.md           # Datasets: structure, versioning, CRUD contract
│   ├── 03-experiments.md        # Experiments/evals: run lifecycle, comparisons
│   ├── 04-scoring.md            # Scorers: LLM-as-judge, rule-based, human
│   ├── 05-annotation.md         # Annotation: human review workflow on traces
│   └── 06-api.md                # HTTP API contract (endpoints, request/response shapes)
├── tests/
│   └── cases.yaml               # Language-agnostic test cases (input/output pairs)
├── schema/
│   ├── trace.json               # JSON Schema for a Trace object
│   ├── span.json                # JSON Schema for a Span object
│   ├── dataset.json             # JSON Schema for a Dataset + DatasetItem
│   ├── experiment.json          # JSON Schema for an Experiment + ExperimentRun
│   └── score.json               # JSON Schema for a Score object
├── INSTALL.md                   # How to implement the platform (prompt template)
└── docs/
    ├── research/
    │   └── findings.md          # Research notes (SDD methodology + AI observability)
    └── plans/
        └── 2026-02-21-spec-authoring-plan.md   # This file
```

---

## Spec Authoring Sequence

Work through the spec files in this order. Each section depends on the previous.

### Phase 1: Foundation

#### Step 1 — `SPEC.md` (Master Spec)
The root document. Defines overall scope, design principles, and the shared data model all sub-specs reference.

**Contents:**
- Overview (2-3 sentences: what this platform does)
- Design Principles (lightweight, OTel-native, pure interfaces, YAGNI)
- Glossary (Trace, Span, Dataset, Experiment, Score, Annotation — precise definitions)
- Core Data Model (entity relationships: how traces → datasets → experiments → scores connect)
- Deployment Contract (what a compliant implementation must provide: HTTP API, storage, SDK)
- Out of Scope (GPU monitoring, Kubernetes operator, online evals in v1, etc.)

#### Step 2 — `schema/` (JSON Schemas)
Define the canonical shape of every core object. All other specs reference these schemas.

**Objects to define:**
- `Span` — id, parent_id, trace_id, name, input, output, start_time, end_time, tokens, model, metadata
- `Trace` — id, project_id, spans[], root_span_id, created_at, metadata
- `DatasetItem` — id, dataset_id, input, expected_output, metadata
- `Dataset` — id, project_id, name, version, items[], created_at
- `ExperimentRun` — id, experiment_id, dataset_item_id, output, scores[], trace_id
- `Experiment` — id, project_id, name, dataset_id, runs[], created_at
- `Score` — id, run_id (or span_id), scorer_name, value, rationale, created_at

### Phase 2: Core Behaviors

#### Step 3 — `specs/01-tracing.md`
**Contents:**
- What a trace captures (full request path: input → retrieval → LLM call → tool call → output)
- Span hierarchy (parent-child, root span, nested spans)
- Required fields vs optional fields per span
- OpenTelemetry alignment (GenAI Semantic Conventions mapping table)
- SDK contract: `start_span()`, `end_span()`, `log_trace()` — behavior, not implementation
- Ingestion API: how spans are submitted to the platform
- Edge cases: spans with no parent, spans submitted out of order, missing fields

#### Step 4 — `specs/02-datasets.md`
**Contents:**
- What a dataset is (versioned collection of input/expected-output pairs)
- Creating datasets: from scratch, from annotated traces, from CSV/JSONL import
- Dataset versioning: immutable versions vs mutable head
- DatasetItem schema and constraints
- CRUD operations: create, read, update (add items), delete, fork
- Dataset reconciliation: the pattern of incremental updates (vs "golden dataset")

#### Step 5 — `specs/03-experiments.md`
**Contents:**
- What an experiment is (running an application against a dataset; capturing outputs + scores)
- Experiment lifecycle: created → running → completed → compared
- How a run maps to: one dataset item → one application call → one set of scores
- Comparison: how to diff two experiment results (score delta per item)
- Regression detection: defining pass/fail thresholds
- SDK contract: `run_experiment(dataset, fn, scorers)` — behavior contract

#### Step 6 — `specs/04-scoring.md`
**Contents:**
- What a score is: (run_id or span_id) + scorer_name + value + optional rationale
- Score types: numeric (0.0–1.0), categorical (pass/fail, label)
- Built-in scorer contracts (what a compliant platform must support):
  - `LLMJudge` — prompt template, model, extract score from response
  - `ExactMatch` — input vs expected, case sensitivity options
  - `Contains` — string containment check
  - `Regex` — pattern match
- Custom scorer interface: any function `(input, output, expected?) → Score`
- Scorer configuration: how scorers are declared and attached to experiments

#### Step 7 — `specs/05-annotation.md`
**Contents:**
- What annotation is: human review of a trace or span to produce a corrective signal
- Annotation objects: trace_id (or span_id), annotator, label, correction, notes, created_at
- Annotation workflow: queue → review → submit → save to dataset
- How annotations become dataset items (the conversion contract)
- Who can annotate: roles (developer, reviewer, domain expert)
- UI contract: minimum fields/interactions a compliant annotation UI must support

### Phase 3: API + Tests

#### Step 8 — `specs/06-api.md`
**Contents:**
- REST API contract for all platform operations
- Endpoint table: method, path, request body, response body, errors
- Authentication contract (bearer token; implementation-defined)
- Pagination contract (cursor-based)
- Versioning: API versioned at `/v1/`

Key endpoints:
```
POST   /v1/traces                    # Ingest trace
GET    /v1/traces/:id                # Fetch trace with spans
GET    /v1/traces                    # List traces (project-scoped, paginated)

POST   /v1/datasets                  # Create dataset
GET    /v1/datasets/:id              # Fetch dataset + items
POST   /v1/datasets/:id/items        # Add items
POST   /v1/datasets/:id/items/import # Bulk import from JSONL

POST   /v1/experiments               # Create experiment
POST   /v1/experiments/:id/run       # Submit a run result
GET    /v1/experiments/:id/results   # Fetch all run results with scores
GET    /v1/experiments/:id/compare/:other_id  # Diff two experiments

POST   /v1/scores                    # Submit a score for a run or span
GET    /v1/scores?run_id=...         # List scores

POST   /v1/annotations               # Submit annotation
GET    /v1/annotations?trace_id=...  # List annotations for a trace
```

#### Step 9 — `tests/cases.yaml`
Language-agnostic test cases for spec-verifiable behaviors:

- Trace ingestion: valid span, missing required fields, out-of-order spans
- Dataset operations: create, add items, version fork
- Scoring: exact match, contains, regex — input/output pairs
- Experiment comparison: score delta calculation
- Annotation: annotation → dataset item conversion

Format (mirrors `whenwords`):
```yaml
scoring:
  - name: "exact match - correct"
    function: exact_match
    input: { output: "Paris", expected: "Paris" }
    result: { value: 1.0 }

  - name: "exact match - incorrect"
    function: exact_match
    input: { output: "London", expected: "Paris" }
    result: { value: 0.0 }
```

### Phase 4: Packaging

#### Step 10 — `README.md`
- What this spec is and what it is not
- Quick orientation: read SPEC.md first, then the sub-specs in order
- How to implement: see INSTALL.md
- How to validate: run `tests/cases.yaml`
- License + contribution guide

#### Step 11 — `INSTALL.md`
A single prompt template (like `whenwords`) an engineer pastes into an AI to generate a compliant implementation:

```
Implement the AI observability platform described in SPEC.md.

1. Read SPEC.md for architecture and data model
2. Read specs/ in order (01 through 06) for detailed behavior
3. Read schema/ for canonical object shapes
4. Parse tests/cases.yaml and generate tests
5. Implement the HTTP API defined in specs/06-api.md
6. Run tests until all pass
7. Place implementation in [LOCATION]

All tests/cases.yaml test cases must pass.
```

---

## Quality Checklist (Per Spec File)

Before marking a spec file complete:

- [ ] Every behavior has a precise, unambiguous description
- [ ] Every edge case is documented (invalid input, missing fields, empty state)
- [ ] No implementation details leak in (no "use PostgreSQL", no "use Redis")
- [ ] All object references point to the correct schema file
- [ ] A human can read this section in under 10 minutes
- [ ] A test case exists in `cases.yaml` for every specified behavior

---

## Design Principles to Uphold Throughout

These must be preserved in every spec decision:

1. **Lightweight** — no required external services beyond a database and object storage
2. **OpenTelemetry-native** — tracing schema aligns with OTel GenAI Semantic Conventions
3. **Implementation-agnostic** — spec describes behavior, not technology choices
4. **Minimal surface area** — every feature must justify its inclusion
5. **Human-reviewable** — any section should be readable end-to-end in < 10 minutes
6. **Testable** — every behavioral claim maps to at least one test case in `cases.yaml`
7. **Composable** — each sub-spec (tracing, datasets, evals) is independently useful

---

## What's Out of Scope (v1)

Explicitly deferred to keep the spec tight:

- Online evals (scoring live production traffic in real-time)
- Prompt management / versioning
- Playground / interactive prompt testing
- GPU monitoring
- Kubernetes operator / auto-instrumentation
- Multi-tenancy / organizations
- SSO / SAML
- Billing / usage limits
- Webhooks / alerting
- Fine-tuning / RLHF pipelines
