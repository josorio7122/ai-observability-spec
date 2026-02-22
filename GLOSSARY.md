# Glossary

> Plain-language definitions of every domain term used in this spec. Read this before anything else if any term feels unfamiliar. No technical jargon, no implementation details — just what things mean.

---

## Terms

### Project

A project is the top-level container for all your data. Everything in the platform — traces, datasets, experiments, annotations — belongs to a project. If you are building a customer support chatbot, that chatbot is a project. If you also have a document summarization tool, that is a separate project. Projects keep data from different applications from mixing together.

### Trace

A trace is the complete record of one user request, from the moment it arrives to the moment a response is sent. If a user asks your AI assistant "What is the capital of France?", everything that happens to produce the answer — looking up documents, constructing a prompt, calling a language model, formatting the result — is captured in a single trace. A trace is your primary debugging tool: when something goes wrong, you look at the trace to find out exactly what happened.

### Span

A span is one step within a trace. A trace is made up of many spans, each representing a single operation: one call to a language model, one search through a document index, one call to an external tool, one formatting step. Spans are nested — a "handle user query" span might contain a "search documents" span and an "LLM call" span as children. Together, the spans in a trace form a tree that shows the full sequence of operations and how long each one took.

### Dataset

A dataset is a collection of test cases used to evaluate your application. Each test case has an input — something you would feed to your application — and optionally an expected output — the correct or ideal response. A dataset might contain fifty example customer support questions with their ideal answers, or a hundred document summarization tasks. Datasets are the foundation of systematic evaluation: without them, you can only judge quality by looking at individual traces.

### Dataset Item

A dataset item is a single test case within a dataset. It has an input (required) and an expected output (optional). The input is what gets fed to your application during an experiment. The expected output is what the application *should* produce, used by scorers that check whether the application got it right. Dataset items are immutable once created — if a test case needs to change, you add a new one.

### Experiment

An experiment is a record of running your application against a dataset to measure its quality. You create an experiment, run your application on every item in the dataset, submit the outputs, and attach scores. The experiment then gives you an aggregate picture: what percentage of responses were correct, where the failures clustered, and how this run compares to previous ones. Experiments are how you answer "did this change make things better or worse?"

### Experiment Run

An experiment run is the result of running your application on one dataset item. It captures the output your application produced for that specific input. If an experiment has fifty dataset items, it will have fifty runs when complete — one per item. Each run can have scores attached to it, and optionally a link to the trace that was generated when your application processed that input.

### Score

A score is a quality measurement on an experiment run or a span. It is either a number between 0 and 1 (where 1 means perfect and 0 means completely wrong) or a categorical label (like "correct", "hallucination", or "off-topic"). Scores make quality measurable and comparable. Instead of reading through fifty outputs and forming a vague impression, you can see that your application scored 0.84 on average, or that 12 out of 50 responses were labelled "hallucination".

### Scorer

A scorer is the method used to produce a score. Some scorers are rule-based and fully automatic: an exact-match scorer checks whether the output equals the expected output character-for-character; a regex scorer checks whether the output matches a pattern. Other scorers use a language model as a judge: you give it a rubric and it rates the output. Human scorers are scores that a person assigns manually after reviewing a response. Custom scorers are any scoring logic you implement yourself and submit via the API.

### Annotation

An annotation is a human-authored note on a trace. When a reviewer — a developer, a product manager, a domain expert — looks at a trace and finds that the application's response was wrong, incomplete, or harmful, they add an annotation. An annotation can include a label ("hallucination"), a correction (the response that should have been given instead), and free-text notes. Annotations are the mechanism by which human judgment feeds back into the system: a correction in an annotation can become the expected output for a new dataset item, which then gets tested in the next experiment.

### Review Queue

The review queue is a curated feed of traces that have been flagged for human review. Traces end up in the queue either because a developer explicitly flagged them (a low-scoring run, a trace with an error) or because they were surfaced as worth reviewing for another reason. The review queue is designed for non-technical reviewers: inputs and outputs are shown as readable text, not raw data structures. A reviewer works through the queue, annotating the traces they find, without needing to understand how the platform works internally.

---

## How They Connect

Here is the full loop in plain English.

You instrument your application so that every request it handles produces a **trace**. Each trace is made up of **spans** — one per operation — that together show exactly what your application did and how long each step took.

Meanwhile, you build a **dataset** of representative inputs: the kinds of questions or tasks your application should be able to handle well. For each input, you optionally add an expected output — the correct answer — as a **dataset item**.

You then run an **experiment**: your application processes every item in the dataset, and you submit the outputs to the platform. A **scorer** evaluates each output and produces a **score** — a number or label that says how well the application did on that input. An **experiment run** is created for each item, holding the output and its scores. Once all runs are submitted, you can view an aggregate summary, compare this experiment to a previous one, and set a quality threshold for your CI pipeline.

When you find traces where the application failed — whether through low experiment scores, production errors, or user complaints — you add them to the **review queue**. A reviewer works through the queue and adds **annotations**: labels, corrections, and notes. Those corrections can be converted into new **dataset items**, which enter the next experiment. The next time you run the eval, the previously-failing case is now tested automatically.

This is the core loop: **instrument → dataset → experiment → score → annotate → improve**.