# Research Findings: Spec-Driven Development & AI Observability

> This document captures research findings to preserve context for spec authoring.

---

## Part 1: Spec-Driven Development (SDD)

### What It Is

Spec-Driven Development is a development paradigm where **a well-crafted specification is written before code**, and that spec becomes the primary source of truth for humans, AI coding agents, tests, and documentation alike.

The core idea: instead of describing an idea in vague prompts and hoping AI gets it right ("vibe-coding"), you define *what* the system should do — behaviors, edge cases, constraints, interface contracts — and then generate code from that. This produces more reliable, intentional implementations and prevents AI from optimizing for speed over correctness.

It is directly inspired by, but distinct from, TDD and BDD — both of which shaped its vocabulary (Given/When/Then, acceptance criteria, executable specs). What's new is the AI context: specs now serve as high-quality, structured prompts for autonomous coding agents.

### There Is No Single Definition — Three Levels Exist

Thoughtworks researcher Birgitta Böckeler identified three distinct levels of SDD in practice (from Kiro, GitHub Spec-Kit, Tessl):

| Level | What it means |
|---|---|
| **Spec-first** | A spec is written before coding begins, used to guide a specific task, then discarded |
| **Spec-anchored** | The spec is kept after the task, used for evolution and maintenance over time |
| **Spec-as-source** | The spec is the *only* human-edited artifact; code is fully AI-generated and regenerable |

Most current tools are spec-first. Not all define a maintenance strategy. The debate in the industry is whether code or the spec is the ultimate source of truth — and this is genuinely unresolved.

### What Actually Makes a Good Spec

From Thoughtworks, GitHub, Scalable Path, Zencoder, and intent-driven.dev:

- **Behavior-oriented, not implementation-oriented** — defines input/output, preconditions/postconditions, invariants, state transitions. Does NOT specify tech stack or implementation patterns.
- **Domain-oriented ubiquitous language** — uses business terms, not tech terms
- **Given/When/Then structure** for scenarios — still valid and recommended from BDD
- **Completeness + conciseness** — covers the critical path without enumerating every possible edge case upfront
- **Semi-structured** — mix of natural language and structured inputs/outputs. Machine-readable specs reduce hallucinations and improve AI reasoning significantly.
- **Separation of concerns** — business requirement specs vs technical specs (though the boundary is often blurry in practice)

### Spec for a Platform vs Spec for a Library — A Critical Distinction

This matters a lot for us. `whenwords` is a *library* spec (pure functions, no state, no external dependencies, single interface). We are building a *platform* spec. These are fundamentally different:

| Dimension | Library spec | Platform spec |
|---|---|---|
| Scope | Single interface, pure functions | Multi-component, stateful, distributed |
| State | None — pure inputs/outputs | Database, sessions, storage, queues |
| Interface | Function signatures + types | HTTP API contracts, request/response schemas, auth |
| Concurrency | Not a concern | Race conditions, eventual consistency, ordering |
| Evolution | Semver of a function surface | Backward compatibility, API versioning, migrations |
| Cross-cutting | N/A | Auth, pagination, error formats, rate limits |
| Architecture | Not specified | Must describe components and how they interact |

A platform spec (per Zencoder and Martin Fowler articles) has five concerns a library spec doesn't:
1. **Data schemas and invariants** — structure, constraints, validation rules
2. **Interface contracts** — API capabilities, inputs/outputs, behavioral guarantees
3. **Security boundaries** — identity, trust zones, policy enforcement  
4. **Versioning semantics** — evolution, deprecation, migration
5. **Compatibility rules** — backward and forward compatibility

### The GitHub Spec-Kit Workflow (Most Applicable to Us)

GitHub's Spec-Kit provides the most relevant SDD workflow for building a real product with AI agents:

1. **Specify** — high-level description of *what* and *why*: user journeys, success criteria, outcomes. Not technical yet. Produces `spec.md`.
2. **Plan** — technical: stack, architecture, constraints, integrations. Produces `plan.md`.
3. **Tasks** — breaks spec + plan into small, independently implementable chunks with checkpoints. Produces `tasks.md`.
4. **Implement** — agent tackles tasks one by one; human reviews focused changes, not 1000-line dumps.

Key principle: **"Your role isn't just to steer. It's to verify."** At each phase, human reflects and refines before moving forward.

The memory bank (called "constitution" in Spec-Kit, "steering" in Kiro) contains cross-cutting context that applies to every session: architecture principles, security rules, coding standards. This is separate from per-feature specs.

### Kiro's Spec Structure (Most Prescriptive)

Kiro uses exactly three documents per feature:
- `requirements.md` — user stories in "As a… I want… So that…" format with Given/When/Then acceptance criteria
- `design.md` — component architecture, data flow, data models, error handling, testing strategy
- `tasks.md` — concrete tasks tracing back to requirement numbers, each small enough to validate independently

### SDD Anti-Patterns

- **Specification Theater** — writing specs no one reads or validates against
- **Premature Comprehensiveness** — specifying everything upfront instead of iterating
- **AI-Generated Spec Bloat** — accepting verbose AI-generated specs without human curation; if you're skimming it thinking "AI probably got it right," the feature is too large
- **Spec-Implementation Drift** — specs that diverge from code become liabilities, not assets
- **Vague boundaries** — specs that bleed into implementation details lose their value as behavioral contracts

### Key Takeaway for Our Use Case

We are writing a **spec-anchored platform spec** — the spec is meant to be kept, evolved, and used to drive implementations over time. It is not a `whenwords`-style "spec as product" where the spec *is* the library. Our spec describes a platform that will be implemented, and the spec will be maintained as the system evolves.

This means:
- We need a **constitution / memory bank** — cross-cutting principles that apply everywhere (auth, pagination, error formats)
- We need **component-level specs** — each major subsystem gets its own behavioral specification
- We need an **API contract** — HTTP endpoints, schemas, request/response format
- We need **acceptance criteria** per feature — testable Given/When/Then scenarios
- We do NOT need `tests.yaml`-style language-agnostic test pairs (those are for library functions, not stateful APIs)

---

## Part 2: AI Observability Platforms

### Why AI Observability Is Different

Traditional observability asks: **"Is it up?"** — metrics, latency, error rates, uptime.

AI observability asks: **"Is it good?"** An LLM app can return HTTP 200 while delivering a wrong, harmful, or hallucinated answer. Standard dashboards show green while the product is broken.

Key differences from traditional observability:
- AI systems are **probabilistic** — same input, different outputs across runs
- Spans are **massive** — average AI span ~50KB vs ~900 bytes in traditional tracing; complex agent runs can produce 10GB+ traces
- Quality depends on **prompts, model version, retrieval, tools, context length, training data** — none of which show up in latency metrics
- **Non-technical stakeholders** (PMs, domain experts, users) need to participate in quality review — this is rare in traditional observability

### The Three Pillars (Braintrust's Framework)

Braintrust identified that the traditional three pillars (metrics, logs, traces) don't map onto AI well. Their three AI-specific pillars:

#### Pillar 1: Traces
Reconstruct the **full decision path** for a request — every model call, tool invocation, retrieval step, and control flow operation.

- Each operation = a **span** (id, parent_id, input, output, latency, token usage, model params, error)
- Spans form a parent-child tree = one **trace** per user request
- Primary use: not performance profiling (as in traditional tracing), but **understanding what actually happened** — which docs were retrieved, what prompt was built, which step caused a bad response
- Requires purpose-built storage — traditional APM tools break at 10GB+ traces
- **OpenTelemetry** with GenAI Semantic Conventions is the emerging standard for instrumentation

#### Pillar 2: Evals
Systematically measure output quality — both before deployment and in production.

**Offline evals** (pre-production):
- Run the application against curated datasets of inputs + expected outputs
- Automated scorers check quality criteria (factual accuracy, relevance, safety, etc.)
- Act as regression tests: prompt/model/retrieval changes run through evals before shipping
- Can block deploys that fail quality thresholds

**Online evals** (production):
- Score live traffic as it happens, without adding user-visible latency
- Surface failure patterns that test datasets don't cover (real users are unpredictable)
- Feed failure cases back into offline datasets → the **eval feedback loop**

**Scoring methods:**
- **LLM-as-a-judge** — one model evaluates another's output against a rubric
- **Rule-based** — exact match, contains, regex, format validation
- **Human review** — expert judgment for edge cases; calibrates automated scores

The eval lifecycle: online evals discover failure patterns → add failing cases to offline datasets → fix and verify in dev → ship → confirm improvement in production.

#### Pillar 3: Annotation
Create **corrective signals** from humans to align AI behavior with actual intent.

- Humans (PMs, domain experts, users) review traces, flag problems, correct outputs
- Annotations are mutable metadata attached to traces/spans
- Annotations flow into **datasets** for offline eval
- Enables "dataset reconciliation" — continuously updating datasets to reflect real-world behavior, rather than a one-time "golden dataset"
- Requires a UI that non-technical users can operate without reading JSON

### Core Feature Set

| Feature | Description | Priority |
|---|---|---|
| **Tracing** | Capture full request execution: model calls, tool invocations, retrieval, with inputs/outputs/latency/tokens per span | Must have |
| **Datasets** | Versioned collections of input/expected-output pairs; foundation for evals | Must have |
| **Experiments / Evals** | Run app against a dataset, score outputs, compare runs across changes | Must have |
| **Scoring** | LLM-as-judge, rule-based scorers; numeric or categorical outputs | Must have |
| **Trace viewer** | UI to inspect individual traces and spans | Must have |
| **Annotation** | Human review workflow on traces; flag + correct + save to datasets | Should have |
| **Monitoring / Dashboards** | Aggregate quality scores, latency, cost, error rates over time | Should have |
| **Prompt Management** | Version-controlled prompt storage + testing | Could have |
| **Playground** | Interactive prompt + model testing | Could have |
| **Online Evals** | Auto-score live production traffic | Could have |

### The Open-Source Landscape

| Tool | Model | Key Differentiator |
|---|---|---|
| **Braintrust** | Hosted + self-hostable | Full-featured; Brainstore DB purpose-built for AI trace scale; annotation UI; hybrid deploy |
| **Arize Phoenix** | Open source | Best-known OSS alternative; strong tracing + eval; self-hosting with no friction |
| **OpenLIT** | Open source | OpenTelemetry-native from the start; 50+ LLM provider integrations; Kubernetes operator |
| **Langfuse** | Open source + hosted | Strong prompt management + tracing; popular for self-hosters in Europe |
| **Evidently** | Open source | 100+ pre-made metrics; ML monitoring heritage; strong CI/CD integration |

### What "Lightweight" Actually Means

From the landscape research, lightweight platforms share these traits:
- **Single deployable binary or minimal compose file** — no mandatory microservices architecture
- **OpenTelemetry-native instrumentation** — instrument once, no vendor lock-in
- **No proprietary abstractions** — traces, spans, scores, datasets map to well-understood concepts
- **Bring your own scorer model** — don't force a specific LLM for judging
- **Local-first dev experience** — works without cloud credentials or accounts
- **Small SDK footprint** — a few functions cover 90% of use cases

---

## Part 3: Implications for Our Spec

### On Methodology

We are building a **spec-anchored platform spec** following the GitHub Spec-Kit / Kiro model:
- The spec is a **living document** that evolves with the platform
- We need a **constitution** (cross-cutting principles applied everywhere)
- Each subsystem gets its own **behavioral spec** (not just a schema dump)
- We need **acceptance criteria** per feature in testable form

We are **not** building a `whenwords`-style spec. `whenwords` works because it specifies pure functions with deterministic inputs/outputs in a stateless library. A platform has state, HTTP APIs, storage, auth, and multi-step workflows — these require different spec artifacts.

### On Scope

The minimum viable spec covers:
1. **Tracing** — the foundation; nothing else works without it
2. **Datasets** — required by evals to have something to run against
3. **Experiments/Evals** — the core value proposition
4. **Scoring** — the mechanism that makes evals meaningful
5. **Annotation + trace viewer** — closes the human feedback loop

Explicitly out of scope for v1:
- Online evals (live production scoring — complex, needs traffic routing)
- Prompt management (separable, not core to observability)
- Playground (separable)
- GPU monitoring, Kubernetes operator
- Multi-tenancy, SSO, billing
