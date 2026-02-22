# AI Observability Spec

A behavioral specification for a lightweight AI observability platform. This repository contains the spec — not code.

---

## What This Is

A complete, implementation-agnostic specification for a platform that helps teams understand and improve the quality of their AI applications. It covers the core observability loop:

**Instrument → Evaluate → Improve**

1. **Instrument** your application to capture traces (every LLM call, tool use, retrieval step)
2. **Evaluate** outputs systematically by running your app against datasets and scoring the results
3. **Improve** by reviewing failures, annotating corrections, and feeding them back as new test cases

The spec defines what the platform must *do* — behaviors, contracts, and acceptance criteria — without prescribing how it's built.

---

## What This Is Not

- **Not a codebase** — there is no implementation here
- **Not a hosted service** — this spec describes something you build or generate
- **Not prescriptive about technology** — no language, database, or framework is required

---

## Scope (v1)

**In scope:**
- Tracing (spans, traces, OpenTelemetry alignment)
- Datasets (versioned input/expected-output collections)
- Experiments (running evals, scoring outputs, comparing results)
- Scoring (exact match, contains, regex, LLM-as-judge, human, custom)
- Annotation (human review workflow, trace → dataset conversion)
- HTTP API contract (all endpoints, request/response shapes, error codes)
- UI (developer view + cross-functional review queue)

**Out of scope for v1** (see `CONSTITUTION.md` for full list):
Online evals, prompt management, playground, GPU monitoring, multi-tenancy, SSO, billing, webhooks.

---

## How to Read This Spec

Read in this order — each document builds on the previous:

1. **[`CONSTITUTION.md`](./CONSTITUTION.md)** — Cross-cutting rules that apply everywhere: auth, error format, pagination, timestamps, IDs, API versioning, design principles.

2. **[`DATA-MODEL.md`](./DATA-MODEL.md)** — Every core entity (Span, Trace, Dataset, DatasetItem, Experiment, ExperimentRun, Score, Annotation) with full field definitions and relationships.

3. **[`specs/01-tracing.md`](./specs/01-tracing.md)** — Tracing behavior: span hierarchy, ingestion rules, OTel alignment, acceptance criteria.

4. **[`specs/02-datasets.md`](./specs/02-datasets.md)** — Dataset management: versioning, CRUD, bulk import, annotation conversion, reconciliation pattern.

5. **[`specs/03-experiments.md`](./specs/03-experiments.md)** — Experiments: lifecycle, run submission, aggregate summary, comparison, threshold evaluation.

6. **[`specs/04-scoring.md`](./specs/04-scoring.md)** — Scorers: built-in (exact_match, contains, regex, llm_judge), human, custom, attachment methods.

7. **[`specs/05-annotation.md`](./specs/05-annotation.md)** — Annotation: human review workflow, immutability rules, conversion to dataset items.

8. **[`specs/06-api.md`](./specs/06-api.md)** — Full HTTP API contract: every endpoint, request/response schemas, error codes.

9. **[`specs/07-ui.md`](./specs/07-ui.md)** — UI spec: developer view (dashboard, traces, experiments, datasets) and reviewer queue (annotation panel, rendering contract).

---

## Using This Spec with an LLM

This spec is designed to be fed directly to an AI coding agent. The structure is intentional: small reusable foundation files + independently consumable subsystem specs. This keeps context focused and reduces noise.

### The Golden Rule

**`CONSTITUTION.md` + `DATA-MODEL.md` are always in context.** They are small (~400 lines total) and every other file references them. Every prompt, every session, every agent invocation should include both.

**The sub-specs are modular.** Each one (`01-tracing.md` through `07-ui.md`) is independently consumable. Include only the one relevant to the task at hand.

### Context by Task

| Task | Files to include |
|---|---|
| Implement tracing ingestion | `CONSTITUTION.md` + `DATA-MODEL.md` + `specs/01-tracing.md` |
| Implement datasets | `CONSTITUTION.md` + `DATA-MODEL.md` + `specs/02-datasets.md` |
| Implement experiments + scoring | `CONSTITUTION.md` + `DATA-MODEL.md` + `specs/03-experiments.md` + `specs/04-scoring.md` |
| Implement annotation workflow | `CONSTITUTION.md` + `DATA-MODEL.md` + `specs/05-annotation.md` |
| Implement full backend API | `CONSTITUTION.md` + `DATA-MODEL.md` + `specs/06-api.md` |
| Implement the UI | `CONSTITUTION.md` + `DATA-MODEL.md` + `specs/06-api.md` + `specs/07-ui.md` |
| Validate an existing implementation | Paste only the **Acceptance Criteria** section from the relevant spec file |
| Add a new feature | Edit spec first → then `CONSTITUTION.md` + `DATA-MODEL.md` + the changed spec file |

### Three Usage Patterns

**Pattern 1 — One subsystem at a time (recommended)**

Work through the specs in dependency order (`01 → 02 → 03 → 04 → 05 → 06 → 07`), implementing and validating each before moving to the next. Each subsystem is small enough that the LLM has full, focused context with no noise from unrelated subsystems.

**Pattern 2 — API contract as the implementation target**

Use `specs/06-api.md` as the sole implementation target for the full backend. The API spec is a complete HTTP contract — every endpoint, every shape, every error code. The behavioral sub-specs (`01`–`05`) then become your validation layer: after implementation, feed each sub-spec to the agent and ask it to verify compliance.

**Pattern 3 — Spec-anchored iteration**

The spec is a living document. For every change to the platform:
1. Update the relevant spec file first
2. Feed the updated spec + existing code to the agent: *"The spec has changed. Update the implementation to match."*
3. Run the acceptance criteria as regression checks

Never change the implementation without changing the spec first. The spec is the source of truth; code follows.

### Validating Without Reimplementing

The `GIVEN / WHEN / THEN` acceptance criteria in each spec file can be used as standalone validation prompts — no need to provide the full spec:

```
Here is an existing implementation of [endpoint/feature].
Do these acceptance criteria all pass?
For each one that fails, explain why and suggest a fix.

[paste the Acceptance Criteria section from the relevant spec file]
```

This is the most efficient way to catch drift between spec and implementation over time.

---

## How to Implement

See **[`INSTALL.md`](./INSTALL.md)** for step-by-step instructions, technology recommendations, and ready-to-use prompts for every subsystem.

---

## How to Validate

Each spec file contains an **Acceptance Criteria** section with `GIVEN / WHEN / THEN` scenarios. A compliant implementation must:

- Pass every acceptance criterion in every spec file
- Handle every edge case listed in the Edge Cases tables
- Conform to the error codes and response shapes in `specs/06-api.md`

---

## Repository Structure

```
ai-observability-spec/
├── README.md              ← You are here
├── INSTALL.md             ← How to implement: prompts and step-by-step guide
├── CONSTITUTION.md        ← Cross-cutting rules (always include in context)
├── DATA-MODEL.md          ← Entity definitions (always include in context)
├── specs/
│   ├── 01-tracing.md
│   ├── 02-datasets.md
│   ├── 03-experiments.md
│   ├── 04-scoring.md
│   ├── 05-annotation.md
│   ├── 06-api.md
│   └── 07-ui.md
└── docs/
    ├── research/
    │   └── findings.md    ← Research notes: SDD methodology + AI observability landscape
    └── plans/
        └── 2026-02-21-spec-authoring-plan.md  ← How this spec was planned
```

---

## Design Philosophy

This spec follows the **spec-anchored** model of spec-driven development: the spec is a living document maintained alongside (but independent from) any implementation. When the platform evolves, the spec is updated first.

Key principles (from `CONSTITUTION.md`):
- **Lightweight** — no mandatory microservices or cloud dependencies
- **OpenTelemetry-aligned** — traces map to OTel GenAI Semantic Conventions
- **Implementation-agnostic** — behavior is specified, not technology
- **Composable** — each subsystem is independently useful
- **Human-first** — written to be read and understood by humans

---

## License

MIT
