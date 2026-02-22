# INSTALL.md

How to turn this spec into a running implementation using an AI coding agent.

---

## Before You Start

Read these two files first. They are short and every prompt below assumes you have internalized them:

- [`CONSTITUTION.md`](./CONSTITUTION.md) — auth, error format, pagination, timestamps, API versioning
- [`DATA-MODEL.md`](./DATA-MODEL.md) — all entity definitions and relationships

Every prompt in this file includes both. Do not remove them.

---

## Choosing Your Approach

### Option A — Subsystem by subsystem (recommended)

Build and validate one subsystem at a time, in dependency order. Each subsystem is small enough for a single focused agent session. You review the output before moving on.

**Order:** Tracing → Datasets → Experiments → Scoring → Annotation → API → UI

### Option B — Full backend from the API contract

Use `specs/06-api.md` as the single implementation target. Best when you want to scaffold the entire HTTP API in one pass and fill in business logic afterward.

### Option C — Full platform in one shot

Provide all spec files at once. Works well with larger context windows (Claude, GPT-4o) for greenfield projects where you want maximum speed. Less predictable output — validate each subsystem's acceptance criteria afterward.

---

## Technology Choices

The spec is implementation-agnostic. Common choices that work well:

| Layer | Options |
|---|---|
| **Backend language** | Python (FastAPI), TypeScript (Node/Hono/Fastify), Go, Ruby (Rails/Sinatra) |
| **Database** | PostgreSQL (recommended), SQLite (for local/lightweight), MySQL |
| **Storage (for large trace payloads)** | S3-compatible object store, local filesystem for dev |
| **UI framework** | React, Vue, Svelte — anything that can consume a JSON API |
| **Auth (token issuance)** | Any — the spec only requires bearer token validation, not how tokens are issued |

If you have no preference, **Python + FastAPI + PostgreSQL** is a well-trodden path with strong LLM training data coverage. **TypeScript + Hono + PostgreSQL** is a good choice if you want end-to-end type safety between backend and frontend.

---

## Step-by-Step Prompts

Copy each prompt, paste into your agent of choice (Claude, Cursor, Copilot, etc.), and review the output before proceeding to the next step.

---

### Step 1 — Project Scaffold

Before implementing any spec, get a clean project structure in place.

```
Set up a new [LANGUAGE/FRAMEWORK] project for a REST API server.

Requirements:
- A working HTTP server that starts and responds to requests
- A database connection layer (I will be using [DATABASE])
- A project structure that is easy to extend with new route handlers
- A README with instructions for running locally

Do not implement any routes yet. Just the scaffold.
```

---

### Step 2 — Tracing

```
I am building a lightweight AI observability platform. Implement the tracing subsystem.

Read the following spec files and implement a compliant solution:

--- CONSTITUTION.md ---
[paste contents of CONSTITUTION.md]

--- DATA-MODEL.md ---
[paste contents of DATA-MODEL.md]

--- specs/01-tracing.md ---
[paste contents of specs/01-tracing.md]

Implementation requirements:
- Implement all endpoints described in specs/01-tracing.md
- Use [LANGUAGE/FRAMEWORK] and [DATABASE]
- All acceptance criteria in the Acceptance Criteria section must pass
- Handle all edge cases listed in the Edge Cases table
- Write tests for each acceptance criterion

Do not implement any other subsystem yet.
```

**Validate before continuing:**
```
Here is my implementation of the tracing subsystem.
Do all of these acceptance criteria pass?
For each failure, explain why and show the fix.

[paste the Acceptance Criteria section from specs/01-tracing.md]

[paste your implementation]
```

---

### Step 3 — Datasets

```
I am extending my AI observability platform. Add the datasets subsystem.

The tracing subsystem is already implemented. Do not modify it.

Read the following spec files and implement a compliant solution:

--- CONSTITUTION.md ---
[paste contents of CONSTITUTION.md]

--- DATA-MODEL.md ---
[paste contents of DATA-MODEL.md]

--- specs/02-datasets.md ---
[paste contents of specs/02-datasets.md]

Implementation requirements:
- Implement all behavior described in specs/02-datasets.md
- All acceptance criteria must pass
- Handle all edge cases
- Write tests for each acceptance criterion
```

---

### Step 4 — Experiments and Scoring

These two subsystems are tightly coupled — implement them together.

```
I am extending my AI observability platform. Add the experiments and scoring subsystems.

Tracing and datasets are already implemented. Do not modify them.

Read the following spec files and implement a compliant solution:

--- CONSTITUTION.md ---
[paste contents of CONSTITUTION.md]

--- DATA-MODEL.md ---
[paste contents of DATA-MODEL.md]

--- specs/03-experiments.md ---
[paste contents of specs/03-experiments.md]

--- specs/04-scoring.md ---
[paste contents of specs/04-scoring.md]

Implementation requirements:
- Implement all behavior described in both spec files
- Scoring may be submitted inline with runs or separately — support both
- All acceptance criteria in both files must pass
- Write tests for each acceptance criterion
```

---

### Step 5 — Annotation

```
I am extending my AI observability platform. Add the annotation subsystem.

Tracing, datasets, experiments, and scoring are already implemented. Do not modify them.

--- CONSTITUTION.md ---
[paste contents of CONSTITUTION.md]

--- DATA-MODEL.md ---
[paste contents of DATA-MODEL.md]

--- specs/05-annotation.md ---
[paste contents of specs/05-annotation.md]

Implementation requirements:
- Implement all behavior described in specs/05-annotation.md
- Pay special attention to the annotation → dataset item conversion mapping table
- All acceptance criteria must pass
- Write tests for each acceptance criterion
```

---

### Step 6 — Complete the API

Once all subsystems are in place, use the API spec to fill any gaps and verify the full surface is correct.

```
I have implemented the following subsystems of my AI observability platform:
tracing, datasets, experiments, scoring, annotation.

Using the full API contract below, do two things:
1. Identify any endpoints in the spec that are not yet implemented and implement them.
2. Verify that all existing endpoints match the request/response shapes in the spec exactly.
   If any shapes differ, update the implementation to match the spec.

--- CONSTITUTION.md ---
[paste contents of CONSTITUTION.md]

--- DATA-MODEL.md ---
[paste contents of DATA-MODEL.md]

--- specs/06-api.md ---
[paste contents of specs/06-api.md]
```

---

### Step 7 — UI

The UI can be built independently of the backend — it only needs the API contract and the UI spec.

```
I am building the frontend for a lightweight AI observability platform.
The backend API is already running at [API_BASE_URL].

Read the following spec files and implement a compliant UI:

--- CONSTITUTION.md ---
[paste contents of CONSTITUTION.md]

--- DATA-MODEL.md ---
[paste contents of DATA-MODEL.md]

--- specs/06-api.md ---
[paste contents of specs/06-api.md]

--- specs/07-ui.md ---
[paste contents of specs/07-ui.md]

Implementation requirements:
- Use [UI_FRAMEWORK]
- Implement both the Developer View and the Review Queue as described in specs/07-ui.md
- Apply the rendering contract from the "Rendering Contract" section for all input/output display
- All acceptance criteria in specs/07-ui.md must be satisfied
- No charts in v1 — tables and numbers only
- The project selector must be present; all data must be project-scoped
```

---

## Option B — Full Backend from the API Contract

If you prefer to scaffold the entire HTTP API in one pass:

```
I am building a lightweight AI observability platform.
Implement the complete HTTP API defined in the spec below.

--- CONSTITUTION.md ---
[paste contents of CONSTITUTION.md]

--- DATA-MODEL.md ---
[paste contents of DATA-MODEL.md]

--- specs/06-api.md ---
[paste contents of specs/06-api.md]

Implementation requirements:
- Use [LANGUAGE/FRAMEWORK] and [DATABASE]
- Implement every endpoint in the spec
- Request/response shapes must match exactly
- All error codes must be returned as defined
- Use the conventions in CONSTITUTION.md for auth, pagination, timestamps, and error format
- Generate a database schema that supports all entities in DATA-MODEL.md

Do not implement business logic beyond what is described in the API contract.
I will add behavioral constraints from the sub-specs in subsequent steps.
```

After this, run the acceptance criteria from each sub-spec (`01`–`05`) as validation prompts to catch any missing business logic.

---

## Option C — Full Platform in One Shot

For experienced users who want maximum speed on a greenfield project:

```
I am building a lightweight AI observability platform from scratch.
Implement the complete platform — backend API and frontend UI — based on the spec files below.

--- CONSTITUTION.md ---
[paste contents of CONSTITUTION.md]

--- DATA-MODEL.md ---
[paste contents of DATA-MODEL.md]

--- specs/01-tracing.md ---
[paste]

--- specs/02-datasets.md ---
[paste]

--- specs/03-experiments.md ---
[paste]

--- specs/04-scoring.md ---
[paste]

--- specs/05-annotation.md ---
[paste]

--- specs/06-api.md ---
[paste]

--- specs/07-ui.md ---
[paste]

Implementation requirements:
- Backend: [LANGUAGE/FRAMEWORK], [DATABASE]
- Frontend: [UI_FRAMEWORK]
- All acceptance criteria in all spec files must pass
- All edge cases must be handled
- Generate a database schema, all API route handlers, and the full UI
- Include a docker-compose.yml so the full stack can be started with one command
```

---

## Ongoing: Spec-Anchored Iteration

Once you have a running implementation, use this workflow for every change:

**1. Update the spec first.**
Edit the relevant spec file before writing any code. If you're adding an endpoint, add it to `specs/06-api.md`. If you're changing behavior, update the acceptance criteria.

**2. Feed the diff to your agent.**
```
The spec for [subsystem] has changed. Here is the updated spec file.
Update the implementation to match. Do not change anything not covered by the spec change.

--- CONSTITUTION.md ---
[paste]

--- DATA-MODEL.md ---
[paste]

--- specs/[changed-file].md ---
[paste updated file]

--- Current implementation ---
[paste relevant implementation files]
```

**3. Validate with acceptance criteria.**
```
Does this implementation satisfy all of these acceptance criteria?
List any that fail and explain why.

[paste the Acceptance Criteria section from the changed spec file]

[paste implementation]
```

**4. Commit spec and implementation together.**
The spec and the code that implements it should always be in sync. Commit them in the same commit so the history stays coherent.

---

## Troubleshooting

**The agent keeps adding features not in the spec.**
Add this to your prompt: *"Implement only what is described in the spec. Do not add endpoints, fields, or behaviors that are not explicitly specified."*

**The agent ignores the error codes.**
Add this: *"Every error response must use the exact error code strings defined in CONSTITUTION.md and specs/06-api.md. Do not invent new error codes."*

**The output is too large to review.**
Switch to Option A (subsystem by subsystem). The full-platform prompt works best when you can review 500–800 lines at a time, not 3,000+.

**The agent changes things from a previous subsystem.**
Add this: *"[Subsystem X] is already implemented and tested. Do not modify any files related to it. Only add new files and routes for the subsystem described in this spec."*

**The implementation drifts from the spec over time.**
Run a validation pass: feed the current implementation + the relevant spec's acceptance criteria to your agent and ask it to identify any drift. This is cheaper than a full rewrite.
