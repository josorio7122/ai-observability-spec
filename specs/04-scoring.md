# Scoring

> The mechanism that turns observations into measurements. A score attaches a quantified quality signal — numeric or categorical — to an experiment run or a span, making it possible to track, compare, and threshold application behavior systematically.

Cross-cutting rules (auth, errors, pagination, timestamps) are defined in `CONSTITUTION.md`. Object schemas are defined in `DATA-MODEL.md`.

---

## Overview

Scoring is what separates AI observability from AI logging. Logs tell you what happened; scores tell you how well it went.

A score attaches to either:
- An **ExperimentRun** — measuring the quality of the application's output on a specific dataset item
- A **Span** — measuring the quality of a specific operation within a trace (e.g., whether a retrieval step returned relevant documents)

Scores may be produced by:
- **Built-in scorers** — rule-based functions the platform evaluates automatically
- **LLM-as-judge** — a model evaluates another model's output against a rubric
- **Human review** — a person submits a score manually
- **Custom scorers** — developer-defined functions whose results are submitted via the API

The platform does not run scorers on behalf of the caller (except LLMJudge, which is implementation-defined — see below). For all other scorer types, the caller computes the score and submits it.

---

## User Stories

- As a developer, I can attach a numeric score to an experiment run to quantify how well my application performed on a specific input.
- As a developer, I can use built-in rule-based scorers by specifying scorer parameters in my run submission so I don't have to implement common checks myself.
- As a developer, I can submit a human-assigned score so expert judgment is captured alongside automated measurements.
- As a developer, I can attach scores to spans (not just runs) so I can measure quality at a sub-request level.
- As a developer, I can retrieve all scores for a run or span to inspect quality measurements.

---

## Score Values

Scores are either **numeric** or **categorical**:

| Type | Value | Meaning |
|---|---|---|
| Numeric | `0.0` to `1.0` (inclusive) | Higher is better. `1.0` = perfect; `0.0` = complete failure. |
| Categorical | Any non-empty string | A label (e.g., `"pass"`, `"fail"`, `"relevant"`, `"hallucination"`). No enumeration enforced. |

A single scorer always produces the same type (numeric or categorical) for all its scores. Mixing types within one `scorer_name` produces undefined aggregate behavior and should be avoided.

Numeric values outside `[0.0, 1.0]` are rejected with `400 INVALID_SCORE_VALUE`.

---

## Built-in Scorers

Built-in scorers are rule-based functions with deterministic behavior. They are specified by name and configuration when submitting a score or as part of an experiment run.

### `exact_match`

Compares the run's `output` to the dataset item's `expected_output` as strings.

**Configuration:**

| Parameter | Type | Default | Description |
|---|---|---|---|
| `case_sensitive` | boolean | `true` | Whether comparison is case-sensitive |
| `strip_whitespace` | boolean | `true` | Whether leading/trailing whitespace is trimmed before comparison |

**Scoring logic:**
- Both `output` and `expected_output` are coerced to strings before comparison.
- If `expected_output` is absent or `null`, the score cannot be computed: returns `null` (no score created).
- Result: `1.0` if strings match, `0.0` if they do not.

**Example:**
```
output: "  Paris  "
expected_output: "paris"
case_sensitive: false, strip_whitespace: true
→ value: 1.0
```

---

### `contains`

Checks whether the run's `output` contains the `expected_output` as a substring.

**Configuration:**

| Parameter | Type | Default | Description |
|---|---|---|---|
| `case_sensitive` | boolean | `true` | Whether the containment check is case-sensitive |

**Scoring logic:**
- Both values are coerced to strings.
- If `expected_output` is absent or `null`, score cannot be computed: returns `null`.
- Result: `1.0` if `output` contains `expected_output`, `0.0` otherwise.

---

### `regex`

Checks whether the run's `output` matches a regular expression.

**Configuration:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `pattern` | string | yes | A regular expression pattern |
| `flags` | string | no | Regex flags (e.g., `"i"` for case-insensitive). Implementation-defined flag syntax. |

**Scoring logic:**
- `output` is coerced to a string.
- Result: `1.0` if the pattern matches anywhere in `output`, `0.0` if it does not.
- If `pattern` is an invalid regex, the score submission is rejected: `400 INVALID_SCORER_CONFIG`.

---

### `llm_judge`

Uses a language model to evaluate the output against a rubric. This is the highest-leverage scorer for quality dimensions that cannot be captured by rules (factual grounding, relevance, tone, safety).

**Configuration:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `model` | string | yes | The model to use for judging (e.g., `"gpt-4o"`, `"claude-3-5-sonnet-20241022"`) |
| `prompt_template` | string | yes | A prompt that instructs the judge model. Must include `{{input}}`, `{{output}}`, and optionally `{{expected_output}}` as placeholders. |
| `score_extraction` | string | no | One of: `numeric` (default), `label`. How to extract the score from the judge's response. |
| `score_range` | object | no | `{ "min": number, "max": number }`. The numeric range the judge is asked to produce. Scores are normalized to `[0.0, 1.0]`. Default: `{ "min": 0, "max": 1 }`. |

**Scoring logic:**
1. The `prompt_template` is rendered by substituting `{{input}}`, `{{output}}`, and `{{expected_output}}` with the run's values.
2. The rendered prompt is sent to the specified model.
3. The model's response is parsed to extract the score:
   - For `score_extraction: "numeric"` — the first number found in the response is used. If no number is found, the score is `null` and an error is recorded.
   - For `score_extraction: "label"` — the full trimmed response (or the first word, implementation-defined) is used as a categorical label.
4. Numeric scores are normalized from the `score_range` to `[0.0, 1.0]`.
5. The model's response (before parsing) is stored as `rationale`.

**LLMJudge execution model:**

Whether the platform executes LLMJudge automatically or the caller is responsible for running it is **implementation-defined**. Two valid approaches:

- **Platform-executed**: The caller submits a run with an `llm_judge` scorer configuration; the platform calls the model and stores the result.
- **Caller-executed**: The caller runs the prompt themselves and submits the resulting score via `POST /v1/scores` with `scorer_name: "llm_judge"` (or any name).

Implementations must document which model they use and which approach they support.

**Example prompt template:**
```
You are evaluating the factual grounding of an AI assistant's response.

User input: {{input}}
Assistant output: {{output}}
Reference answer: {{expected_output}}

Score the response from 0 to 10:
- 10: Perfectly grounded, no hallucinations
- 5: Partially correct with minor errors
- 0: Completely hallucinated or wrong

Respond with only a number.
```

---

## Human Scores

Human scores are submitted via the same `POST /v1/scores` endpoint as automated scores, with `scorer_name` set to `"human"` (or any descriptive name like `"human_review"`, `"sme_eval"`).

There is no special human scorer configuration. The caller submits whatever value they assign. This is how annotation-based scoring integrates: an annotator reviews a trace, assigns a quality label or numeric score, and it is submitted as a score on the corresponding span.

---

## Custom Scorers

Custom scorers are developer-implemented functions. The platform has no knowledge of them — the developer runs the scorer externally and submits results via `POST /v1/scores`.

**Expected interface (informational, not enforced by the spec):**

```
Input:  { input: any, output: any, expected_output?: any }
Output: { value: number (0.0–1.0) | string, rationale?: string }
```

The `scorer_name` in the submitted score should identify the custom scorer uniquely (e.g., `"custom_toxicity_check"`, `"domain_accuracy_v2"`).

---

## Attaching Scores

Scores can be attached in three ways:

### 1. Inline with a run submission
Include a `scores` array when submitting an ExperimentRun to `POST /v1/experiments/:id/runs`. Each entry in the array becomes a Score with `target_type: "run"`.

### 2. Post-submission via the scores endpoint
Call `POST /v1/scores` with `target_id` and `target_type` after the run or span already exists.

### 3. Inline with a span (online scoring)
Include a `scores` array on a span during ingestion to `POST /v1/traces/ingest`. This enables online scoring of production spans.

All three approaches produce identical Score objects. The attachment method is a convenience distinction only.

---

## Acceptance Criteria

```
GIVEN a run with output "Paris" and dataset item expected_output "paris"
WHEN an exact_match score is submitted with case_sensitive=false, strip_whitespace=true
THEN the score value is 1.0

GIVEN a run with output "  Paris  " and dataset item expected_output "Paris"
WHEN an exact_match score is submitted with strip_whitespace=true (default)
THEN the score value is 1.0

GIVEN a run with output "The capital of France is Paris, a beautiful city"
WHEN a contains score is submitted with expected_output="Paris", case_sensitive=true
THEN the score value is 1.0

GIVEN a run with output "The capital of France is paris"
WHEN a contains score is submitted with expected_output="Paris", case_sensitive=true
THEN the score value is 0.0

GIVEN a run with output "Order ID: ABC-12345"
WHEN a regex score is submitted with pattern="[A-Z]+-\d+"
THEN the score value is 1.0

GIVEN a run with output "Order confirmed"
WHEN a regex score is submitted with pattern="[A-Z]+-\d+"
THEN the score value is 0.0

GIVEN a regex scorer configuration with pattern="[invalid"
WHEN the score is submitted
THEN the API returns 400 with error code INVALID_SCORER_CONFIG

GIVEN a score submission with value=1.5
WHEN the score is submitted
THEN the API returns 400 with error code INVALID_SCORE_VALUE

GIVEN a score submission with value=-0.1
WHEN the score is submitted
THEN the API returns 400 with error code INVALID_SCORE_VALUE

GIVEN a run that has been scored by "exact_match" with value 0.8
WHEN scores are listed for that run
THEN the score with scorer_name="exact_match" and value=0.8 is returned

GIVEN a span "span-A" in an existing trace
WHEN a score is submitted with target_id="span-A" and target_type="span"
THEN the score is attached to the span
AND is returned when scores are listed for target_id="span-A"

GIVEN a run with no expected_output on the dataset item
WHEN an exact_match score is submitted
THEN no score is created (result is null)
AND no error is returned

GIVEN an experiment run with output "France"
WHEN multiple scores from different scorer_names are submitted
THEN all scores are returned when listing scores for that run
AND each appears independently in the experiment summary's scores_by_scorer
```

---

## Edge Cases

| Scenario | Expected Behavior |
|---|---|
| Score submitted for a `target_id` that does not exist | `404 NOT_FOUND` |
| Score submitted with `target_type` that is not `"run"` or `"span"` | `400 INVALID_REQUEST` |
| Two scores with the same `scorer_name` on the same target | Both are stored; aggregates use all scores for that `scorer_name` |
| `llm_judge` response contains no parseable number (for `score_extraction: "numeric"`) | Score is not created; error is logged by implementation; no error returned to caller if platform-executed |
| Categorical score used in threshold evaluation | `422 UNSUPPORTED_THRESHOLD_TYPE` (see `specs/03-experiments.md`) |
| Empty string as categorical score value | `400 INVALID_REQUEST` — categorical values must be non-empty strings |
| `exact_match` scorer where `output` is a JSON object (not a string) | Both `output` and `expected_output` are JSON-serialized before comparison |
