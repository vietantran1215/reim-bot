# Phase 03 — Lexical Retrieval with PostgreSQL Full-Text Search

## 1. Goal

Solve query classes that semantic embeddings handle poorly:

- exact policy codes
- distinctive terminology
- exact phrases
- grade identifiers
- structured tokens

Add lexical retrieval using the same PostgreSQL database.

No new search service is allowed.

---

## 2. Starting State

Phase 02 must already provide:

- metadata-aware dense retrieval
- query analysis
- policy applicability filtering
- heading-aware chunks
- stable candidate contract
- expanded evaluation dataset
- RAGAS + deterministic evaluation pipeline

Phase 03 must not bypass these components.

---

## 3. New Capability

Add:

```text
PostgreSQL Full-Text Search
```

The system now has two retrievers:

```text
DenseRetriever
LexicalRetriever
```

They remain independent in this phase.

Do not fuse them yet.

---

## 4. Dependency Delta

Expected new Python dependency count:

```text
0
```

PostgreSQL already provides full-text search.

Do not add:

- OpenSearch
- Elasticsearch
- Meilisearch
- Whoosh
- external BM25 service

If the implementation uses native PostgreSQL ranking such as `ts_rank`, call it PostgreSQL FTS/lexical ranking.

Do not label it BM25 unless BM25 is actually implemented.

---

## 5. Schema Migration

Add a persisted or generated search vector strategy for chunk content.

The exact PostgreSQL implementation may use:

- generated `tsvector`
- maintained `tsvector` column
- expression index

It must be migration-managed.

Add the required GIN index.

The migration must work:

- from Phase 02 database
- from empty database to head

---

## 6. Lexical Query Construction

Build lexical search input from normalized query text.

Preserve exact high-value tokens such as:

- TRV-0042
- L6
- SGD
- USD
- policy-specific terms

Do not over-normalize identifiers into useless tokens.

If PostgreSQL tokenization splits policy codes, implement an explicit exact-match boost/filter path using ordinary indexed columns where appropriate.

This is still lexical retrieval, not hybrid fusion.

---

## 7. Candidate Contract

Lexical candidate must use the same common retrieval candidate model introduced earlier.

Populate:

- chunk_id
- content
- metadata
- lexical_score
- lexical_rank
- retrieval_source = LEXICAL

Dense-specific fields may remain null.

This shared contract is a prerequisite for Phase 04 fusion.

---

## 8. Metadata and Applicability

Lexical retrieval must apply the same Phase 02 applicability rules.

A lexical exact match in an expired policy must not automatically win.

Required flow:

```text
query analysis
   |
   v
applicability filters
   |
   v
PostgreSQL lexical search
```

Do not duplicate policy applicability logic inside the lexical repository.

Use the same shared filter builder/domain logic.

---

## 9. Debug API

Extend:

```http
POST /rag/retrieve
```

Allow selecting strategy explicitly for learning/debug purposes:

```json
{
  "question": "What does TRV-0042 say?",
  "strategy": "LEXICAL"
}
```

Debug response includes lexical:

- raw score
- rank
- applied filters
- chunk metadata

Normal `/rag/query` may continue using dense by default until Phase 06 adaptive routing.

---

## 10. Golden Dataset Expansion

Add exact-term categories:

- policy code
- employee grade
- exact phrase
- uncommon reimbursement term
- currency/amount wording

Keep all previous case IDs.

Label expected relevant document/chunk/section IDs.

This phase needs cases where dense retrieval is intentionally weak but lexical retrieval can succeed.

---

## 11. Evaluation

Compare independently:

```text
Metadata-filtered Dense
vs
Metadata-filtered Lexical
```

Do not fuse yet.

Deterministic metrics:

- Hit Rate@K
- Recall@K
- Precision@K
- MRR
- nDCG@K

Mandatory RAGAS:

- Faithfulness
- Response Relevancy
- Context Precision
- Context Recall

Report metrics by query category.

A single global average can hide the reason lexical retrieval exists.

Required breakdown:

```text
semantic
exact_identifier
metadata
version
no_answer
```

---

## 12. Setup Option A — No Docker

No new infrastructure.

```bash
git checkout <phase-03-branch>
uv sync
cp .env.example .env

uv run alembic upgrade head
uv run python scripts/ingest.py --reset
uv run uvicorn app.main:app --reload
```

Run:

```bash
uv run python evals/run.py --profile smoke
uv run python evals/run.py --profile full
```

The local PostgreSQL installation from earlier phases is sufficient.

---

## 13. Setup Option B — Docker

```bash
git checkout <phase-03-branch>
docker compose up -d db

uv sync
cp .env.example .env

uv run alembic upgrade head
uv run python scripts/ingest.py --reset
uv run uvicorn app.main:app --reload
```

Still exactly one mandatory infrastructure container.

---

## 14. Tests

Unit:

- lexical query normalization
- exact identifier preservation
- shared filter construction
- lexical candidate mapping

PostgreSQL integration:

- FTS index created
- lexical search returns expected chunks
- metadata filter works with lexical search
- expired policy does not leak into current-policy results
- ranking is deterministic enough for fixed corpus/tests

API:

- explicit lexical debug retrieval
- exact policy-code case
- exact phrase case
- invalid strategy validation

---

## 15. Acceptance Gate

Complete Phase 03 only when:

1. no new infrastructure service was added
2. migration works incrementally and from empty DB
3. ingestion rebuilds FTS data/indexes
4. dense behavior from Phase 02 remains intact
5. lexical results use the shared candidate contract
6. exact-term evaluation cases show why lexical retrieval is useful
7. deterministic + RAGAS reports exist
8. Phase 04 can consume dense and lexical candidate lists without schema changes

---

## 16. Handoff to Phase 04

Phase 04 receives two independent, already evaluated retrievers:

```text
DenseRetriever
LexicalRetriever
```

Both:

- use the same query analysis
- apply the same policy applicability rules
- emit the same candidate contract

Phase 04 only needs to combine their ranked lists.
