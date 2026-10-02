# Reim Bot — Phase Execution Plan

## Purpose

This directory defines the implementation checkpoints for Reim Bot.

Each phase is designed to become a separate **cumulative branch** later.

The critical rule is:

> Phase N starts from the exact completed state of Phase N-1. No phase may assume code, schema, data, dependency, configuration, or evaluation artifacts that were not introduced previously.

The project is learning-first, free/local-first, and setup-light.

---

## Branch Strategy

Recommended future branch chain:

```text
main
  |
  v
phase/01-basic-rag
  |
  v
phase/02-metadata-aware-rag
  |
  v
phase/03-lexical-rag
  |
  v
phase/04-hybrid-rag
  |
  v
phase/05-reranking
  |
  v
phase/06-adaptive-rag
  |
  v
phase/07-context-compression
  |
  v
phase/08-evaluation-tuning
```

Do **not** create every phase branch independently from `main`.

Each phase branch must be created from the previous completed phase branch.

This guarantees that the code history itself represents the learning progression.

---

## Phase Order

| Phase | Main Learning Goal | New Capability |
|---|---|---|
| 01 | Understand complete baseline RAG | Dense retrieval + generation + citations + RAGAS baseline |
| 02 | Prevent semantically-similar but inapplicable retrieval | Metadata filtering + policy versioning + heading-aware chunking |
| 03 | Solve exact-term retrieval failures | PostgreSQL Full-Text Search |
| 04 | Combine semantic and lexical evidence | Hybrid retrieval + RRF + deduplication |
| 05 | Improve candidate ordering | Local cross-encoder reranking |
| 06 | Choose retrieval strategy per query | Deterministic Adaptive Retrieval router |
| 07 | Reduce context noise/token load | Extractive context compression |
| 08 | Compare and tune the complete system | Full RAGAS + retrieval benchmark + regression gates |

This order is intentional.

No phase uses a technique before the prerequisites required to understand and evaluate it exist.

---

## Global Reproducibility Rules

Every phase must satisfy all of these before it is complete.

### 1. Exact dependency lock

The branch must contain:

```text
pyproject.toml
uv.lock
.python-version
.env.example
```

Use `uv`.

The implementation must not depend on packages installed outside the locked environment, except:

- PostgreSQL + pgvector for the no-Docker database option
- Docker Desktop / Docker Engine for the Docker option
- Ollama for the default free local generation/evaluator profile

### 2. Database migration

Every database change must have an Alembic migration.

A learner must be able to start from an empty database and run:

```bash
uv run alembic upgrade head
```

without manual SQL except the one-time `CREATE EXTENSION vector` bootstrap when the selected database image/provider does not create it automatically.

### 3. Reproducible corpus

All mandatory learning documents live in:

```text
knowledge/expense-policies/
```

No mandatory evaluation case may depend on a private file or external URL.

### 4. Reproducible ingestion

Each phase must provide:

```bash
uv run python scripts/ingest.py --reset
```

The reset option must rebuild the phase's index from repository data.

### 5. Reproducible evaluation

Each phase must provide:

```bash
uv run python evals/run.py --profile smoke
uv run python evals/run.py --profile full
```

RAGAS is mandatory from Phase 1.

### 6. No hidden manual setup

Mandatory validation must not require:

- Postman
- manually editing database rows
- manually inserting embeddings
- clicking a cloud console
- copying data between services
- undocumented shell commands

### 7. Cumulative data contract

When a phase introduces new metadata/schema fields, it must also update:

- migration
- ingestion
- corpus metadata
- evaluation dataset
- tests
- debug output where relevant

No orphaned schema.

---

## Two Supported Setup Paths

Every phase must support both paths.

# Option A — No Docker

Use:

- Python 3.13
- uv
- local PostgreSQL with pgvector
- local Ollama for the default free generation/evaluator profile

Expected flow:

```bash
cd reim-bot
uv sync
cp .env.example .env

createdb reim_bot
psql reim_bot -c "CREATE EXTENSION IF NOT EXISTS vector;"

uv run alembic upgrade head
uv run python scripts/ingest.py --reset
uv run uvicorn app.main:app --reload
```

Database credentials may differ by operating system, so `DATABASE_URL` remains configurable.

If the learner already has a compatible PostgreSQL + pgvector database, no local database installation step is needed.

# Option B — Docker

Only the database is mandatory in Docker.

Expected flow:

```bash
docker compose up -d db

uv sync
cp .env.example .env

uv run alembic upgrade head
uv run python scripts/ingest.py --reset
uv run uvicorn app.main:app --reload
```

The Compose file should expose one PostgreSQL + pgvector container.

Do not add extra containers unless a later phase strictly requires them.

Ollama may remain host-installed in both setup paths to avoid GPU/container complexity.

An OpenAI-compatible API endpoint may replace Ollama through environment configuration, but the default learning path remains local/free.

---

## Default Model Profile

The implementation should keep model names configurable, but the docs must provide one reproducible default.

### Embedding

```text
BAAI/bge-small-en-v1.5
```

Local via sentence-transformers.

### Reranker

Introduced only in Phase 5:

```text
cross-encoder/ms-marco-MiniLM-L-6-v2
```

### Generation / RAGAS evaluator

Default free profile:

```text
Ollama
```

The exact chat model must be pinned in the implementation branch documentation and `.env.example`.

The application and RAGAS evaluator must talk through an OpenAI-compatible adapter so a remote provider can be substituted without changing retrieval code.

---

## Mandatory Evaluation Layers

Each phase reports two independent metric families.

### Deterministic retrieval metrics

- Hit Rate@K
- Recall@K
- Precision@K
- MRR
- nDCG@K when graded relevance is available

### RAGAS metrics

- Faithfulness
- Response Relevancy / Answer Relevancy
- Context Precision
- Context Recall

Later phases add phase-specific metrics such as:

- route accuracy
- compression ratio
- reranker latency

RAGAS does not replace retrieval metrics.

Retrieval metrics do not replace RAGAS.

---

## Evaluation Profiles

### Smoke

Small fixed subset.

Purpose:

- fast learner feedback
- development regression
- branch validation

### Full

Complete golden dataset.

Purpose:

- phase completion
- baseline comparison
- final report

A phase is not complete based only on smoke evaluation.

---

## Phase Completion Contract

Every phase must end with:

1. clean setup instructions for both setup options
2. successful migration from an empty database
3. reproducible ingestion
4. API scenario tests
5. unit/integration tests relevant to the phase
6. smoke evaluation
7. full evaluation
8. comparison against the previous phase
9. machine-readable report
10. human-readable summary
11. documented known failure cases

Negative evaluation results are valid.

A phase does not need to beat the previous phase on every metric.

It must explain the trade-off.

---

## Dependency Policy

Dependencies are cumulative but minimal.

A later phase may add a package only if its feature is introduced in that phase.

Examples:

- do not install reranker dependencies before Phase 5
- do not add a router LLM dependency in Phase 6 because the baseline router is deterministic
- do not add a second search engine in Phase 3 because PostgreSQL FTS already exists in the selected database
- do not add an observability stack just to display evaluation metrics

The learning objective is Advanced RAG, not infrastructure administration.

---

## Files

- `01-basic-rag.md`
- `02-metadata-aware-rag.md`
- `03-lexical-rag.md`
- `04-hybrid-rag.md`
- `05-reranking.md`
- `06-adaptive-rag.md`
- `07-context-compression.md`
- `08-evaluation-tuning.md`
