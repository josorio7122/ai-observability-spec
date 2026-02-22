# Research Findings: Spec-Driven Development & AI Observability

> This document captures research findings to preserve context for spec authoring. It covers (1) spec-driven development methodology and (2) AI observability platform features.

---

## Part 1: Spec-Driven Development (SDD)

### What It Is

Spec-Driven Development is a methodology where a **precise, human-readable specification is the primary artifact** — written before any code, serving as the source of truth throughout development, testing, and documentation.

The core shift: instead of code being the authoritative description of behavior, the **spec is**. Code is an implementation of the spec, not the other way around.

### The `whenwords` Example (Reference Implementation)

`whenwords` (https://github.com/dbreunig/whenwords) is the canonical minimal example:
- A relative time formatting library with **zero code** in the repo
- Contains only: `SPEC.md`, `tests.yaml`, `INSTALL.md`
- `SPEC.md` — full behavioral specification (functions, edge cases, types, error handling)
- `tests.yaml` — language-agnostic input/output test pairs any implementation must pass
- `INSTALL.md` — a single prompt to paste into an AI to generate any implementation in any language
- Result: works in Ruby, Python, Rust, Elixir, Swift, PHP, Bash, and even Excel — all from one spec

**Key insight:** The spec + tests are the product. Implementations are ephemeral artifacts.

### The 3-Phase Workflow

```
1. DEFINE   → Write the spec (what, why, behavior, constraints)
2. IMPLEMENT → Build against the spec
3. VALIDATE  → Automated checks ensure alignment with spec
```

The living spec evolves with the system. Each change touches the spec first.

### Best Practices

| Practice | Why It Matters |
|---|---|
| **Human reviewability first** | If you're skimming the spec thinking "AI probably got it right," it's too large |
| **Start minimal** | Avoid long chains of "and" — concise specs prevent context window exhaustion |
| **Decompose meaningfully** | Use INVEST/MoSCoW — slices should deliver value independently |
| **Spec as source of truth** | Never let spec drift from implementation; continuous validation catches this |
| **Pure functions, explicit inputs** | `whenwords` never touches the system clock — reference time is always passed explicitly |
| **Language-agnostic tests** | `tests.yaml` format means tests outlive any implementation |
| **No premature packaging** | Generate minimal files (source + tests + usage.md); skip CI/CD, gemspecs, etc. |

### Anti-Patterns to Avoid

- **Specification Theater** — writing specs no one reads or validates
- **Premature Comprehensiveness** — specifying everything upfront instead of iterating
- **Tool Over-Reliance** — no tool compensates for poor decomposition
- **Spec-Implementation Drift** — specs that diverge from reality become liabilities
- **AI-Generated Spec Bloat** — accepting verbose AI output without human curation

### SDD + AI Agents

SDD is particularly powerful with AI coding agents because:
- Clear specs provide **design boundaries** that keep agents aligned
- Well-defined specs enable **parallel work streams** (frontend + backend + integrations simultaneously)
- Specs prevent "vibe coding" where agents optimize for speed over correctness
- Human role shifts from *writing code* to *ensuring specs capture genuine intent*

### Structure of a Good Spec (from `whenwords`)

```
1. Overview          — what the library/system does in 2-3 sentences
2. Design Principles — philosophy, non-negotiables (e.g., pure functions only)
3. Output Structure  — what files to generate; what NOT to generate
4. Type Conventions  — abstract types that work across languages
5. Error Handling    — by function, by language idiom
6. Core Behavior     — each function: signature, arguments, behavior table, edge cases
7. Testing           — test data format, how to use tests, input field mapping, examples
```

---

## Part 2: AI Observability Platforms

### Why AI Observability Is Different from Traditional Observability

Traditional observability asks: **"Is it up?"** (metrics, logs, latency, error rates)

AI observability asks: **"Is it good?"** An LLM app can return HTTP 200 with a completely wrong, harmful, or hallucinated answer. Standard dashboards show green while the product is broken.

Key differences:
- AI systems are **probabilistic** — same input, different outputs
- Spans are **huge** — average AI span is ~50KB vs ~900 bytes in traditional tracing
- **Quality depends on** prompts, model versions, retrieval, tools, context length, training data
- Non-technical stakeholders (PMs, domain experts) need to participate in quality review

### The Three Pillars of AI Observability (Braintrust's Framework)

#### Pillar 1: Traces
Reconstruct the full decision path for a request — model calls, tools, retrieval, control flow.

- Records each operation as a **span** (input, output, latency, token usage, model params, errors)
- Spans form a parent-child tree = one **trace** per request
- Used to debug: "which retrieved docs were bad? what prompt was sent? which step caused the failure?"
- Requires purpose-built infrastructure — traditional APM backends break on 10GB+ traces
- **OpenTelemetry** (with GenAI Semantic Conventions) is the emerging standard

#### Pillar 2: Evals
Systematically measure output quality — both in development and production.

**Offline evals (pre-production):**
- Run against curated datasets of representative inputs + expected outputs
- Act as regression tests for prompt/model/retrieval changes
- Block deploys that fail quality thresholds

**Online evals (production):**
- Score live traffic as it happens
- Surface failure patterns that don't appear in test datasets
- Feed real failure cases back into offline datasets → the eval feedback loop

**Scoring methods:**
- **LLM-as-a-judge** — one model scores another's output against criteria
- **Rule-based** — format validation, pattern matching, keyword detection
- **Human review** — expert judgment for edge cases and calibrating automated scores

#### Pillar 3: Annotation
Create corrective signals to inject expertise and ground results in user expectations.

- Trace viewer enables PMs / domain experts / users to flag issues and correct outputs
- Annotations are mutable metadata on traces
- Curated annotations flow into **datasets** for offline evaluation
- Replaces "golden dataset" with continuous "dataset reconciliation" — frequent incremental updates

### Core Feature Set (What a Platform Needs)

| Feature | Description |
|---|---|
| **Tracing** | Capture full request execution: model calls, tool invocations, retrieval steps, with inputs/outputs/latency/tokens per span |
| **Datasets** | Versioned collections of input/output pairs used for evals; built from annotated traces |
| **Experiments / Evals** | Run application against a dataset; score outputs; compare runs across prompt/model changes |
| **Scoring** | LLM-as-judge, rule-based, and human scorers; produce numeric or categorical scores |
| **Prompt Management** | Version-controlled prompt storage; test prompts against datasets before deploying |
| **Annotation** | Human review workflow on traces; flag, correct, save to datasets |
| **Monitoring / Dashboards** | Aggregate metrics over time: quality scores, latency, cost, error rates |
| **Playground** | Interactive testing of prompts + models before committing to code |
| **Online Evals** | Auto-score live production traffic; surface regression patterns |

### Lightweight / Open-Source Alternatives to Braintrust

| Tool | Approach | Key Differentiator |
|---|---|---|
| **Braintrust** | Hosted + self-hostable | Full-featured; purpose-built Brainstore DB; 80x faster than data warehouses |
| **Arize Phoenix** | Open source | Popular Braintrust OSS alternative; strong tracing + eval; friction-free self-hosting |
| **OpenLIT** | Open source | OpenTelemetry-native; integrates 50+ LLM providers; Kubernetes operator |
| **Evidently** | Open source | 100+ pre-made metrics; strong ML monitoring heritage; CI/CD integration |
| **Langfuse** | Open source + hosted | Strong prompt management + tracing; popular in EU/self-host crowd |

### What Makes a Platform "Lightweight"

Based on the landscape:
- **Single deployable unit** — avoid microservices sprawl
- **OpenTelemetry-native** — instrument once, route anywhere; no vendor lock-in
- **Minimal schema** — traces, spans, scores, datasets — no proprietary abstractions
- **Bring your own model for scoring** — don't force a specific judge LLM
- **No mandatory cloud dependency** — local dev works out of the box
- **SDK surface area is small** — `log_trace()`, `score()`, `run_eval()` covers 80% of use cases

---

## Part 3: Key Takeaways for Our Spec

### What SDD Tells Us About Writing This Spec

1. The spec should be **the product** — implementations in any language/framework should be derivable from it
2. Include **language-agnostic test cases** (like `tests.yaml`) so the spec is verifiable
3. **Pure interfaces** — specify behavior, not implementation details
4. **Minimal surface area** — YAGNI ruthlessly; no feature unless it has a clear use case
5. The spec itself should be **decomposed into independently specifiable sections** (tracing, evals, datasets, etc.)

### What AI Observability Research Tells Us About Scope

The minimal viable platform needs:
1. **Tracing** (with OpenTelemetry alignment) — the foundation everything else builds on
2. **Datasets** — without these, evals have nothing to run against
3. **Evals / Experiments** — the core value proposition; offline first
4. **Scoring** — at least one scorer type (LLM-as-judge is highest leverage)
5. **UI for trace viewing + annotation** — enables the human feedback loop

Nice-to-have (defer):
- Online evals (complex; requires production traffic routing)
- Prompt management (useful but separable)
- Playground (useful but separable)
- GPU monitoring, Kubernetes operator, etc.
