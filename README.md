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

7. **[`specs/05-annotation.md`](./specs/05-annotation.md)** — Annotation: human review workflow, immutability rules, conversion to dataset items, UI contract.

8. **[`specs/06-api.md`](./specs/06-api.md)** — Full HTTP API contract: every endpoint, request/response schemas, error codes.

9. **[`specs/07-ui.md`](./specs/07-ui.md)** — UI spec: developer view (dashboard, traces, experiments, datasets) and reviewer queue (annotation panel, rendering contract).

---

## How to Implement

This spec is designed to be used as context for an AI coding agent. Each subsystem is independently implementable.

**To implement a specific subsystem**, provide your agent with:
1. `CONSTITUTION.md` — the rules that apply everywhere
2. `DATA-MODEL.md` — the object shapes
3. The relevant spec file (e.g., `specs/01-tracing.md`) — the behavioral contract

**To implement the full platform**, provide your agent with all files in order.

The acceptance criteria in each spec file are your tests. A compliant implementation must satisfy every `GIVEN / WHEN / THEN` scenario in every spec file.

---

## How to Validate

Each spec file contains an **Acceptance Criteria** section with `GIVEN / WHEN / THEN` scenarios. These are implementation-language-agnostic test specifications. A compliant implementation must:

- Pass every acceptance criterion in every spec file
- Handle every edge case listed in the Edge Cases tables
- Conform to the error codes and response shapes in `specs/06-api.md`

---

## Repository Structure

```
ai-observability-spec/
├── README.md              ← You are here
├── CONSTITUTION.md        ← Cross-cutting rules (read first)
├── DATA-MODEL.md          ← Entity definitions (read second)
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
