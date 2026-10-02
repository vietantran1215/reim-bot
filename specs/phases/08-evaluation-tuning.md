# Phase 08 — Full Evaluation, Tuning, and Regression Baseline

## 1. Goal

Turn the accumulated Advanced RAG pipeline into a reproducible benchmarked system.

This phase adds no major retrieval architecture.

Its job is to answer:

> Which pipeline variant works best for which query category, at what quality/latency/context-cost trade-off?

The final project is complete only when those claims are backed by reproducible evaluation evidence.

---

## 2. Starting State

Phase 07 must provide:

- repository-owned policy corpus
- reproducible ingestion
- Dense retrieval
- metadata filtering
- Lexical retrieval
- Hybrid + RRF
- reranking
- Adaptive routing
- context compression
- citation validation
- golden dataset
- deterministic metrics
- mandatory RAGAS metrics
- smoke/full evaluation profiles

If Phase 08 has to invent a missing feature contract, there was a gap earlier.

---

## 3. New Capability

Add evaluation orchestration and tuning discipline, not a new retrieval engine.

Required variants:

```text
A. Dense
B. Metadata + Dense
C. Lexical
D. Hybrid + RRF
E. Hybrid + RRF + Rerank
F. Adaptive + Hybrid/Dense/Lexical + Rerank
G. Adaptive + Rerank + Compression
```

Each variant must be selectable through evaluation configuration without editing application source code.

---

## 4. Dependency Delta

Prefer:

```text
0 new runtime dependencies
```

Evaluation dependencies should already exist.

A plotting/dataframe package is optional only if it materially improves report generation.

The required outputs are JSON + Markdown, so plotting is not mandatory.

Do not add an observability stack just to display benchmark results.

---

## 5. Golden Dataset Finalization

Final target:

```text
60–100 cases
```

Keep stable IDs from all previous phases.

Required categories:

- semantic paraphrase
- exact identifier
- exact phrase
- country-specific
- employee-grade-specific
- expense-type-specific
- current version
- historical version
- future/expired version trap
- numeric limit
- mixed semantic + exact
- hard negative
- long/noisy context
- ambiguous question
- abstention/no-answer

Each case should define as applicable:

- expected route
- expected document IDs
- expected sections/chunks
- reference answer or reference facts
- should_abstain
- relevance grades for nDCG when used

---

## 6. Evaluation Runner

Required commands:

```bash
uv run python evals/run.py --profile smoke
uv run python evals/run.py --profile full
```

Add optional variant selection:

```bash
uv run python evals/run.py --profile full --variant dense
uv run python evals/run.py --profile full --variant hybrid-rerank
```

And all variants:

```bash
uv run python evals/run.py --profile full --all-variants
```

One command should be able to reproduce the final comparison.

---

## 7. Reproducibility Metadata

Every evaluation report must capture:

- Git commit SHA when available
- timestamp
- dataset version/checksum
- corpus checksum
- embedding model
- embedding dimension
- generation model/provider
- RAGAS evaluator model/provider
- chunking strategy
- chunk_size
- chunk_overlap
- Dense Top-K
- Lexical Top-K
- RRF K
- Fusion Top-K
- rerank model
- rerank Top-K
- route configuration
- compression enabled
- max context tokens

A report without its configuration is not reproducible.

---

## 8. Deterministic Metrics

Required:

- Hit Rate@K
- Recall@K
- Precision@K
- MRR
- nDCG@K when relevance grades exist
- route accuracy
- abstention accuracy
- citation validity
- expected fact coverage

Report:

- global average
- per-category breakdown

Do not hide category failures behind a global average.

---

## 9. Mandatory RAGAS

Every answer-producing variant must report:

- Faithfulness
- Response Relevancy / Answer Relevancy
- Context Precision
- Context Recall

RAGAS evaluator configuration must be recorded independently from application generation configuration.

If the evaluator model changes, the report belongs to a different evaluator cohort and must not be silently compared as if identical.

---

## 10. Operational Metrics

Record per request where available:

- Dense retrieval latency
- Lexical retrieval latency
- fusion latency
- reranking latency
- compression latency
- generation latency
- end-to-end latency
- retrieved candidate count
- final context count
- original context tokens
- compressed context tokens
- generation input/output tokens when provider exposes them

Summaries should include:

- mean
- median
- p95 where sample size is meaningful

---

## 11. Comparison Report

Generate:

```text
evals/reports/<run-id>.json
evals/reports/<run-id>.md
```

Markdown summary should contain at least:

```text
Variant
Recall@K
MRR
nDCG@K
RAGAS Faithfulness
RAGAS Context Precision
RAGAS Context Recall
RAGAS Response Relevancy
P95 latency
Context tokens
```

Also include per-category failures.

---

## 12. Tuning Protocol

Tune one parameter family at a time.

Example order:

1. chunking strategy
2. chunk size/overlap
3. Dense Top-K
4. Lexical Top-K
5. RRF K
6. Fusion Top-K
7. Rerank Top-K
8. route rules
9. context budget/compression threshold

Do not tune everything simultaneously.

Every experiment records:

- hypothesis
- changed variable
- unchanged controls
- metrics before
- metrics after
- conclusion

---

## 13. Regression Baseline

After tuning, select a named reference configuration.

Example:

```text
advanced-rag-v1
```

Commit its config.

Define regression tolerances only after observing repeated evaluation variance.

Do not invent arbitrary thresholds before measurement.

Future changes compare against this reference configuration.

---

## 14. Smoke Profile

Smoke set must remain small and fixed.

Minimum categories:

- semantic
- exact identifier
- metadata-sensitive
- wrong-version trap
- mixed query
- abstention
- hard negative
- noisy-context/compression

Smoke purpose:

- catch obvious regressions quickly

It is not sufficient for final architecture decisions.

---

## 15. Full Profile

Full profile uses all golden cases.

It is required for:

- phase completion
- final report
- choosing reference configuration
- claims that one pipeline variant is better for a category

If local evaluator runtime is slow, full evaluation may be manually invoked.

It still must be reproducible with one documented command.

---

## 16. Setup Option A — No Docker

Use the same local PostgreSQL + pgvector and Ollama/API evaluator setup.

```bash
git checkout <phase-08-branch>
uv sync
cp .env.example .env

uv run alembic upgrade head
uv run python scripts/ingest.py --reset

uv run python evals/run.py --profile smoke --all-variants
uv run python evals/run.py --profile full --all-variants
```

Starting the FastAPI server should not be required if the evaluator directly imports application services.

If evaluation is intentionally black-box over HTTP, document the server startup command explicitly and use it consistently.

Choose one approach and keep it stable.

---

## 17. Setup Option B — Docker

```bash
git checkout <phase-08-branch>
docker compose up -d db

uv sync
cp .env.example .env
uv run alembic upgrade head
uv run python scripts/ingest.py --reset

uv run python evals/run.py --profile smoke --all-variants
uv run python evals/run.py --profile full --all-variants
```

No new infrastructure container is required.

---

## 18. Tests

Unit:

- metric calculations
- report aggregation
- configuration serialization
- experiment variant configuration
- regression comparison

Integration:

- each variant can execute against same corpus
- report metadata is complete
- evaluator adapter works
- stable case IDs preserved

Evaluation self-checks:

- no duplicate case IDs
- required labels present
- expected source IDs exist in corpus
- RAGAS input mapping valid
- no missing configuration fields in report

---

## 19. Final Acceptance Gate

Project completes only when:

1. Docker setup works
2. no-Docker setup works
3. empty DB can migrate to head
4. full corpus can re-ingest from repository files
5. smoke evaluation runs with one command
6. full evaluation runs with one command
7. all required variants run from configuration
8. deterministic metrics exist globally and per category
9. all mandatory RAGAS metrics exist
10. operational metrics exist
11. machine-readable report exists
12. human-readable report exists
13. reference configuration is committed
14. known failure categories are documented
15. no mandatory dependency on Postman/cloud console/manual DB edits exists

---

## 20. Final Learning Outcome

The learner must be able to explain, with actual benchmark evidence:

- where Dense retrieval fails
- why metadata filtering matters
- why lexical retrieval exists
- what Hybrid + RRF changes
- why reranking is a second stage
- when Adaptive routing is useful
- when compression helps or hurts
- why retrieval metrics and RAGAS answer different questions
- how quality, latency, and context size trade off

The project is finished when those explanations are supported by reproducible runs rather than architecture slogans.
