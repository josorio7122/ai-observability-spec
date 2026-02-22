# Overview

> A lightweight platform for understanding and improving the quality of AI applications — by capturing what they do, measuring how well they do it, and feeding human judgment back into the process.

---

## The Problem This Platform Solves

A traditional web application either works or it doesn't. When it fails, a stack trace or an error log points to the cause. AI applications are different. A language model can process a request successfully — no errors, no timeouts, correct HTTP status — and still produce a response that is wrong, harmful, incomplete, or confidently hallucinated. The system looks healthy from the outside while quietly failing the user.

This means the standard tools for understanding software behavior — uptime monitors, error rates, latency dashboards — are necessary but not sufficient for AI applications. They tell you the system ran. They don't tell you whether the system was any good. Answering that question requires a different kind of infrastructure: one that captures what the AI actually did, measures whether it was correct, and makes it possible for the people who know what "correct" means to weigh in.

The compounding problem is that AI quality is not static. Change a prompt, update a model, add a retrieval step — and behavior changes in ways that are hard to predict and easy to miss. Without a systematic way to measure quality before and after every change, teams find out about regressions from user complaints rather than from their own monitoring.

This platform addresses both problems. It gives developers visibility into what their AI application did on every request, a structured way to measure output quality against known test cases, and a feedback loop that turns production failures into regression tests. The goal is to shift quality from something you discover to something you track.

---

## The Two Workflows

### Workflow 1: The Developer Eval Loop

This is the primary workflow for developers who want to measure and improve application quality systematically.

**Step 1 — Instrument.** The developer adds tracing to their application. Every request now produces a trace: a record of every operation the application performed — each language model call, each retrieval step, each tool invocation — with its inputs, outputs, timing, and token usage. Traces flow into the platform automatically. See `specs/01-tracing.md`.

**Step 2 — Build a dataset.** The developer collects representative inputs: the kinds of questions, tasks, or prompts the application should handle well. For each input, they optionally add an expected output — the correct or ideal response. These inputs and expected outputs become dataset items. Datasets can be built manually, imported in bulk from existing logs, or grown over time as the application encounters new cases. See `specs/02-datasets.md`.

**Step 3 — Run an experiment.** The developer runs their application on every item in the dataset and submits the outputs to the platform. Each output is recorded as an experiment run. This can happen in a local script, a CI pipeline, or any automated process — the platform is not involved in running the application, only in recording what it produced. See `specs/03-experiments.md`.

**Step 4 — Score.** Scorers evaluate each run and produce a quality signal. A rule-based scorer might check whether the output contains the expected answer. An LLM-as-judge scorer might rate the response for factual grounding or relevance. A human reviewer might assign a label after reading the output. All scorers produce the same artifact: a score — a number between 0 and 1, or a categorical label. See `specs/04-scoring.md`.

**Step 5 — Review results.** The experiment summary shows aggregate quality: mean score per scorer, the distribution of scores, which items passed and which failed. The comparison view shows the delta between this experiment and a previous one — which items improved, which regressed. A threshold check tells a CI pipeline whether quality is acceptable or whether the build should fail.

**Step 6 — Iterate.** The developer makes a change — to a prompt, a model, a retrieval strategy — and runs the experiment again. The comparison tells them immediately whether the change helped or hurt. Regressions are caught before deployment.

---

### Workflow 2: The Reviewer Annotation Loop

This is the workflow for non-technical reviewers — product managers, domain experts, QA engineers — who can identify bad responses but don't need to understand the platform's internals.

**Step 1 — Flag.** A trace is added to the review queue. This can happen because a developer noticed a low-scoring run and flagged the associated trace, because a user reported a bad response, or because an automated rule surfaced it. The flagging is a deliberate act — the review queue is not a fire hose of all production traffic.

**Step 2 — Review.** The reviewer opens the review queue. Each item shows the input the user gave and the output the application produced, in plain readable text — no JSON, no span trees, no technical metadata unless the reviewer asks for it. The reviewer's job is simple: was this response good?

**Step 3 — Annotate.** For responses that were wrong, the reviewer adds an annotation: a label ("hallucination", "off-topic", "incomplete"), a correction (the response that should have been given), and optional notes. The annotation is attached to the trace. See `specs/05-annotation.md`.

**Step 4 — Convert.** A developer takes the annotation's correction and converts it into a dataset item. The input from the original trace becomes the dataset input; the correction becomes the expected output. This takes one action.

**Step 5 — Close the loop.** The next time an experiment runs against that dataset, the previously-failing case is now included. If the application still gets it wrong, the experiment score reflects it. The regression is now caught automatically, without anyone having to remember to check for it.

---

## How the Subsystems Connect

```
Your Application
      │
      │  emits spans during execution
      ▼
┌─────────────┐
│   Traces    │ ◄── one trace per user request
│   + Spans   │     one span per operation
└──────┬──────┘
       │
       │  developer flags trace for review
       ▼
┌─────────────┐        ┌─────────────┐
│   Review    │        │  Annotation │ ◄── reviewer adds label,
│    Queue    │───────►│             │     correction, notes
└─────────────┘        └──────┬──────┘
                              │
                              │  developer converts annotation
                              ▼
┌─────────────────────────────────────┐
│              Datasets               │
│  ┌──────────────────────────────┐   │
│  │  Dataset Item                │   │
│  │  input: original trace input │   │
│  │  expected: correction        │   │
│  └──────────────────────────────┘   │
└──────────────────┬──────────────────┘
                   │
                   │  experiment runs application on every item
                   ▼
┌─────────────────────────────────────┐
│            Experiments              │
│                                     │
│  Run 1: input → output → score      │
│  Run 2: input → output → score      │
│  Run N: input → output → score      │
│                                     │
│  Summary: mean score, distribution  │
│  Comparison: delta vs. previous     │
│  Threshold: pass / fail for CI      │
└──────────────────┬──────────────────┘
                   │
                   │  low-scoring runs surfaced
                   ▼
             Review Queue  (loop continues)
```

Every arrow in this diagram corresponds to a real operation in the platform. Traces feed the review queue. Annotations feed datasets. Datasets feed experiments. Experiments surface new candidates for review. The loop is continuous.

---

## What Each Spec File Covers

| File | What it covers | Workflows |
|---|---|---|
| `CONSTITUTION.md` | Cross-cutting rules: auth, error format, pagination, timestamps, IDs | Both |
| `DATA-MODEL.md` | Every entity's fields, constraints, and relationships | Both |
| `specs/00-overview.md` | This file — end-to-end flow, the two workflows, subsystem connections | Both |
| `specs/01-tracing.md` | How spans are ingested, assembled into traces, and queried | Developer eval loop |
| `specs/02-datasets.md` | Creating datasets, adding items, bulk import, annotation conversion | Both |
| `specs/03-experiments.md` | Experiment lifecycle, run submission, summary, comparison, thresholds | Developer eval loop |
| `specs/04-scoring.md` | Built-in scorers, LLM-as-judge, human scores, custom scorers | Developer eval loop |
| `specs/05-annotation.md` | Human review workflow, annotation rules, converting to dataset items | Reviewer annotation loop |
| `specs/06-api.md` | Full HTTP API contract: all endpoints, request/response shapes, error codes | Both |
| `specs/07-ui.md` | UI behavioral spec: developer view and reviewer queue | Both |

---

## Key Design Decisions

A few decisions shape the whole system and are worth understanding upfront:

**The platform records results; it does not run your application.** When you run an experiment, you run your own application, collect its outputs, and submit them to the platform. The platform is a ledger, not an executor. This keeps it lightweight and avoids coupling it to any specific runtime, language, or framework.

**Traces are immutable; annotations are additive.** Once a span is submitted, it cannot be changed. Annotations sit alongside traces as a separate layer — they do not modify the underlying data. This means the historical record of what your application actually did is always preserved, separate from what humans said about it.

**Datasets grow continuously; they are not curated once.** The recommended pattern is to keep adding dataset items as your application encounters new failure modes. A small, up-to-date dataset catches more regressions than a large, stale one.

**Every subsystem is independently useful.** You can use tracing without datasets. You can use datasets without annotation. You can run experiments without a UI. The platform is designed so that each piece delivers value on its own and the full loop is available when you are ready for it.
