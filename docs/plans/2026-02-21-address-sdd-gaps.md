# Address SDD Compliance Gaps

> **For Claude:** REQUIRED SUB-SKILL: Use executing-plans to implement this plan task-by-task.

**Goal:** Fix the three SDD compliance gaps identified in the spec review: missing glossary, missing cross-feature flow document, and UI spec prose that drifts into layout/implementation details.

**Architecture:** Three focused edits — one new file (GLOSSARY.md), one new spec file (specs/00-overview.md), and targeted rewrites of the prose sections in specs/07-ui.md. Acceptance criteria in 07-ui.md are left untouched (they are already behavioral). README and INSTALL.md updated to reference the new files.

**Tech Stack:** Markdown only. No code.

---

### Task 1: Add GLOSSARY.md

**The gap:** The spec uses domain terms (trace, span, experiment, run, scorer, annotation, dataset) without ever defining them in plain English in one place. New contributors and LLMs must infer meaning from field tables. The Thoughtworks SDD research requires ubiquitous language — terms defined clearly so every reader shares the same mental model.

**Files:**
- Create: `GLOSSARY.md`

**Step 1: Write GLOSSARY.md**

The glossary must:
- Define every core domain term in 2–4 plain sentences
- Include a "how they connect" paragraph that describes the relationship between terms in prose (not just a table)
- Avoid technical jargon and implementation details
- Be readable by a non-engineer (PM, domain expert)

Terms to define:
- **Project** — the top-level container; all data belongs to a project
- **Trace** — the complete record of one user request from start to finish
- **Span** — a single operation within a trace (one LLM call, one tool use, one retrieval)
- **Dataset** — a collection of test cases; each case has an input and optionally an expected output
- **Dataset Item** — one test case within a dataset
- **Experiment** — a record of running your application against a dataset to measure quality
- **Experiment Run** — the result of running your application on one dataset item
- **Score** — a quality measurement attached to a run or span; numeric (0–1) or a label
- **Scorer** — the function or method that produces a score (exact match, LLM judge, human, custom)
- **Annotation** — a human-authored correction or label on a trace, used to improve datasets
- **Review Queue** — a curated feed of traces waiting for human review and annotation

The "how they connect" paragraph should walk through the core loop in plain English:
you instrument your application to produce traces → you build a dataset from representative inputs → you run an experiment to see how your application performs on that dataset → scores quantify the results → when you find traces where the application failed, you annotate them with the correct response → those annotations become new dataset items → the next experiment catches the regression.

**Step 2: Verify**

Read the file end to end. Check:
- [ ] Every term above is defined
- [ ] No definition mentions a database, framework, or HTTP method
- [ ] The "how they connect" paragraph reads as a coherent story
- [ ] A non-engineer could read this without confusion

**Step 3: Commit**

```bash
git add GLOSSARY.md
git commit -m "docs: add GLOSSARY.md — ubiquitous language for all domain terms"
```

---

### Task 2: Add specs/00-overview.md

**The gap:** Each subsystem is specified in isolation. There is no document that describes the end-to-end flow connecting all subsystems — how a trace becomes an annotation becomes a dataset item becomes an experiment result. The `intent-driven.dev` research called this "systems thinking": detecting how features interact across the full lifecycle.

**Files:**
- Create: `specs/00-overview.md`

**Step 1: Write specs/00-overview.md**

This file must:
- Be the first file in the `specs/` reading order (hence `00-`)
- Describe the complete platform lifecycle in prose, not bullet points
- Show how every subsystem connects to every other
- Identify the two main workflows explicitly: the **developer eval workflow** and the **reviewer annotation workflow**
- Include a plain-text ASCII flow diagram showing the full lifecycle
- Reference the other spec files by name so a reader knows where to go for detail
- Be readable in under 5 minutes

**Structure:**

```
# Overview

> One-sentence description of what the platform does.

## The Problem This Platform Solves

2–3 paragraphs. Why AI observability is different from traditional observability.
What goes wrong without it. What "good" looks like.

## The Two Workflows

### Workflow 1: Developer Eval Loop
Step-by-step prose walkthrough:
1. Developer instruments their application → traces are ingested (specs/01-tracing.md)
2. Developer builds or imports a dataset of representative inputs (specs/02-datasets.md)
3. Developer runs their application against the dataset → experiment runs are submitted (specs/03-experiments.md)
4. Scores are attached to each run automatically or manually (specs/04-scoring.md)
5. Developer reviews the experiment summary and comparison to detect regressions
6. Developer sets a threshold and integrates with CI/CD

### Workflow 2: Reviewer Annotation Loop
Step-by-step prose walkthrough:
1. A trace is flagged for review (from the trace list or automatically via low score)
2. Reviewer opens the Review Queue — sees plain-text input and output, no JSON
3. Reviewer annotates: adds a label, provides a correction, leaves notes (specs/05-annotation.md)
4. Developer converts the annotation to a dataset item → it enters the eval loop
5. Next experiment run includes this item → regression is now caught automatically

## How the Subsystems Connect

ASCII diagram showing the full data flow:

Application
    │  instruments
    ▼
Traces / Spans ──── flagged for review ──► Review Queue
    │                                           │
    │                                      Annotation
    │                                           │
    ▼                                           ▼
Datasets ◄──────── annotation conversion ──── Dataset Items
    │
    ▼
Experiments ──► Runs ──► Scores ──► Summary / Comparison / Threshold
                              │
                              ▼
                    Low-scoring runs ──► flagged for review ──► Review Queue

## What Each Spec File Covers

Table: spec file → what it covers → which workflow it serves
```

**Step 2: Verify**

Read the file end to end. Check:
- [ ] Both workflows are described as complete narratives
- [ ] Every subsystem (tracing, datasets, experiments, scoring, annotation, UI) appears at least once
- [ ] The ASCII diagram is accurate — every arrow corresponds to a real relationship in DATA-MODEL.md
- [ ] The file references other spec files by name where appropriate
- [ ] No implementation details (no database, no framework, no HTTP verbs in the narrative prose)
- [ ] Readable in under 5 minutes

**Step 3: Commit**

```bash
git add specs/00-overview.md
git commit -m "spec: add 00-overview.md — end-to-end flow and cross-subsystem narrative"
```

---

### Task 3: Tighten specs/07-ui.md prose

**The gap:** The UI spec prose describes *how* the UI is laid out (left panel, right panel, full-page view, pre-populated fields) rather than *what the user must be able to do*. This is an SDD anti-pattern: specs that bleed into implementation details lose their value as behavioral contracts and unnecessarily constrain implementors.

**Rule for this task:** Acceptance criteria sections are NOT touched. They are correctly written as behavioral statements. Only the prose description sections above each set of criteria need rewriting.

**Files:**
- Modify: `specs/07-ui.md`

**Step 1: Identify all layout/implementation language in the prose sections**

Scan the file for phrases like:
- "Left panel / right panel" → layout decision, not behavior
- "Opens as a full-page view (or large side panel)" → layout decision
- "Two-column" → layout decision
- "A text field pre-populated from..." → implementation detail
- "Collapsed by default" → interaction default, acceptable only if it affects user capability
- Column names in tables that specify visual order → acceptable (content, not layout)

**Step 2: Rewrite the offending sections**

For each layout/implementation phrase, replace it with a behavioral statement:

| Before (layout/implementation) | After (behavioral) |
|---|---|
| "Layout: Two-column. Left: span tree. Right: span detail panel (updates on span selection)." | "A developer can view all spans in a trace as a navigable tree and inspect any span's full input, output, timing, and error details." |
| "Opens as a full-page view (or large side panel)" | "The annotation panel provides enough space to display the full input and output of a trace without truncation." |
| "A text field pre-populated from the user's session" | "The annotator field is populated with the current user's identifier by default and can be changed before submitting." |
| "Two sections — Review and Submit" | Remove — this is layout. The behavioral requirement is that the reviewer can see the trace content and submit an annotation in the same view. |

The test: after rewriting, every sentence in the prose sections should answer "what can the user do?" not "what does it look like?".

**Step 3: Verify**

Read the prose sections of 07-ui.md (not the acceptance criteria). Check:
- [ ] No directional layout language (left, right, top, bottom, two-column)
- [ ] No component type prescriptions (text field, dropdown, panel, modal) — unless they are in acceptance criteria
- [ ] No default-state prescriptions unless the default directly affects user capability
- [ ] Every sentence answers "what can the user do?" or "what must the user be able to see?"
- [ ] Acceptance criteria sections are unchanged

**Step 4: Commit**

```bash
git add specs/07-ui.md
git commit -m "spec: tighten 07-ui.md prose — remove layout details, keep behavioral statements"
```

---

### Task 4: Update README and reading order

**Files:**
- Modify: `README.md`

**Step 1: Add GLOSSARY.md to the reading order**

In the "How to Read This Spec" section, add GLOSSARY.md as the new first item (before CONSTITUTION.md). It is the entry point for anyone unfamiliar with the domain terms.

Updated order:
0. `GLOSSARY.md` — plain-language definitions of every domain term; read this first if any term is unfamiliar
1. `CONSTITUTION.md` — cross-cutting rules
2. `DATA-MODEL.md` — entity definitions
3. `specs/00-overview.md` — end-to-end flow and the two workflows
4–11. specs/01 through specs/07 (renumbered in the list, files unchanged)

**Step 2: Add specs/00-overview.md to the reading order**

Add it as item 3 (after DATA-MODEL.md, before specs/01-tracing.md) with a description: "The two core workflows (developer eval loop and reviewer annotation loop) and how every subsystem connects."

**Step 3: Update the repository structure diagram**

Add `GLOSSARY.md` and `specs/00-overview.md` to the file tree shown in the README.

**Step 4: Verify**

- [ ] GLOSSARY.md appears in the reading order before CONSTITUTION.md
- [ ] specs/00-overview.md appears in the reading order after DATA-MODEL.md
- [ ] Both files appear in the repository structure diagram
- [ ] No other sections need updating

**Step 5: Commit**

```bash
git add README.md
git commit -m "docs: update README reading order — add GLOSSARY.md and specs/00-overview.md"
```
