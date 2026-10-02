# Phase 01 — Basic Dense RAG Baseline

## 1. Goal

Build the smallest complete RAG system that can be reproduced from an empty database and evaluated with both deterministic retrieval metrics and RAGAS.

This phase establishes the baseline that every later phase must beat, match, or explain.

Do not implement metadata filtering, lexical retrieval, hybrid retrieval, reranking, adaptive routing, or compression yet.

---

## 2. Starting State

Repository contains only specifications.

No application code, schema, corpus index, or evaluation output is assumed.

This phase must create every prerequisite needed by Phase 02.

---

## 3. End State

At completion, the branch contains:

- one FastAPI application
- PostgreSQL + pgvector persistence
- policy ingestion from repository files
- recursive chunking
- local embeddings
- dense vector retrieval
- generation with citations
- abstention behavior
- retrieval debug endpoint
- golden evaluation dataset
- deterministic retrieval metrics
- mandatory RAGAS baseline
- smoke/full evaluation commands
- Docker and no-Docker setup instructions

---

## 4. Dependencies Introduced

Only add packages required by this phase.

Expected categories:

- FastAPI + ASGI server
- Pydantic settings
- SQLAlchemy async + async PostgreSQL driver
- Alembic
- pgvector Python integration
- sentence-transformers
- text splitter package
- HTTP client
- RAGAS
- evaluator/model adapter
- pytest
- Ruff
- mypy

Use `uv` and commit `uv.lock`.

Do not add:

- reranker packages
- OpenSearch client
- Redis
- LangGraph
- agent packages
- task queue
- frontend packages

---

## 5. Data Corpus

Create repository-owned synthetic expense policies under:

```text
knowledge/expense-policies/
```

Phase 01 needs enough data to demonstrate semantic retrieval.

Minimum recommended corpus:

- Global Expense Policy
- Singapore Travel Policy
- Vietnam Travel Policy
- Hotel Policy
- Meal Policy
- Transport Policy
- Receipt Policy

Metadata may exist in source front matter, but Phase 01 retrieval must not filter by metadata yet.

The corpus must be committed so every learner evaluates the same data.

---

## 6. Database Schema

Create Alembic migration for:

### policy_documents

Minimum fields:

- id
- document_id
- policy_code
- title
- policy_type

### policy_versions

Minimum fields:

- id
- policy_document_id
- version
- source_path
- checksum
- created_at

### policy_chunks

Minimum fields:

- id
- policy_version_id
- section
- chunk_index
- content
- embedding
- token_count
- metadata JSONB

Enable pgvector.

The embedding vector dimension must come from the configured embedding model and be consistently reflected in the migration/index configuration.

### ingestion_runs

Minimum fields:

- id
- started_at
- completed_at
- document_count
- chunk_count
- embedding_model
- chunking_strategy
- status
- error_message

---

## 7. Ingestion Pipeline

Implement:

```text
repository policy files
        |
        v
parse text/front matter
        |
        v
recursive chunking
        |
        v
local embedding
        |
        v
store chunk + vector + provenance
```

Required properties:

- deterministic file ordering
- deterministic chunk ordering
- source checksum
- no duplicate chunks when same version/checksum is re-ingested
- `--reset` mode recreates the phase index from repository data

Required command:

```bash
uv run python scripts/ingest.py --reset
```

---

## 8. Dense Retrieval

Implement cosine or equivalent configured pgvector similarity.

Input:

```text
question
```

Output internal candidate contract:

```text
chunk_id
content
document_id
version
section
raw_score
rank
metadata
```

Config:

- `DENSE_TOP_K`

No lexical or metadata filtering yet.

This intentionally allows some bad retrieval so later phases have real failures to improve.

---

## 9. Generation

Implement a small generation interface.

Default free profile:

- OpenAI-compatible local endpoint backed by Ollama

Alternative:

- configured OpenAI-compatible remote endpoint

Generation input:

- user question
- retrieved chunks
- stable citation IDs

Generation requirements:

- answer only from supplied context
- abstain if context is insufficient
- cite only supplied evidence

Do not add conversation memory.

---

## 10. Citation Contract

Normal query response must include structured citations derived from stored metadata.

Example:

```json
{
  "answer": "...",
  "strategy": "DENSE",
  "citations": [
    {
      "chunk_id": "...",
      "document_id": "travel-policy-sg",
      "version": "2026.1",
      "section": "4.1 Hotel"
    }
  ]
}
```

Validate that every emitted citation refers to a retrieved chunk.

---

## 11. API Surface

Required endpoints:

```http
POST /policies/ingest
GET  /policies
POST /rag/query
POST /rag/retrieve
```

`/rag/retrieve` is development/debug oriented and returns:

- dense candidates
- ranks
- scores
- final selected context

It must not expose hidden chain-of-thought.

---

## 12. Golden Dataset

Create:

```text
evals/dataset.json
```

Phase 01 minimum:

```text
20–30 cases
```

Include at least:

- semantic paraphrase
- direct semantic lookup
- cross-policy confusion
- no-answer case
- country-sensitive question even though filtering is not implemented yet
- exact-code question even though lexical retrieval is not implemented yet

The difficult cases are intentional baseline failures.

Each case must contain stable IDs.

---

## 13. Mandatory Evaluation

Required command:

```bash
uv run python evals/run.py --profile smoke
uv run python evals/run.py --profile full
```

### Deterministic metrics

Record:

- Hit Rate@K
- Recall@K
- Precision@K
- MRR
- nDCG@K where relevance labels exist
- retrieval latency

### Mandatory RAGAS metrics

Record:

- Faithfulness
- Response Relevancy / Answer Relevancy
- Context Precision
- Context Recall

Store reports under:

```text
evals/reports/
```

At minimum produce:

- JSON machine-readable report
- Markdown summary

This report is the official baseline for every later phase.

---

## 14. Setup Option A — No Docker

Prerequisites:

- Python 3.13
- uv
- PostgreSQL with pgvector available
- Ollama for default free generation/evaluator profile

Setup:

```bash
git checkout <phase-01-branch>
uv sync
cp .env.example .env

createdb reim_bot
psql reim_bot -c "CREATE EXTENSION IF NOT EXISTS vector;"

uv run alembic upgrade head
uv run python scripts/ingest.py --reset
uv run uvicorn app.main:app --reload
```

In another terminal:

```bash
uv run python evals/run.py --profile smoke
```

If local database credentials differ, edit only `DATABASE_URL`.

---

## 15. Setup Option B — Docker

Prerequisites:

- Python 3.13
- uv
- Docker
- Ollama for the default free generation/evaluator profile

Setup:

```bash
git checkout <phase-01-branch>
docker compose up -d db

uv sync
cp .env.example .env

uv run alembic upgrade head
uv run python scripts/ingest.py --reset
uv run uvicorn app.main:app --reload
```

Then:

```bash
uv run python evals/run.py --profile smoke
```

Compose must contain only the PostgreSQL + pgvector database unless another component is strictly required.

---

## 16. Tests

Unit tests:

- checksum behavior
- chunk ordering
- recursive chunking
- citation validation
- context construction
- abstention decision helper

PostgreSQL integration tests:

- vector insert
- vector retrieval
- ingestion idempotency
- reset ingestion
- migration from empty DB

API tests:

- ingest
- list policies
- query
- retrieve debug
- validation errors

---

## 17. Acceptance Gate

Phase 01 is complete only when:

1. both setup paths work from an empty environment
2. migration succeeds from an empty database
3. ingestion rebuilds the index deterministically
4. all API tests pass
5. smoke evaluation passes operationally
6. full evaluation report exists
7. deterministic metrics are recorded
8. all four mandatory RAGAS metrics are recorded
9. known baseline failures are documented
10. no Phase 02+ dependency exists

Do not tune the dataset to make the baseline look good.

---

## 18. Handoff to Phase 02

Phase 02 starts from this exact state.

It may rely on:

- committed corpus
- stable document/version/chunk IDs
- dense retriever contract
- evaluation dataset and runner
- RAGAS adapter
- baseline report format

Phase 02 must not replace these silently.

It extends them with metadata applicability and improved chunk structure.
