# Phase 05 — Cross-Encoder Reranking

## 1. Goal

Improve precision after broad Hybrid retrieval by adding a local second-stage relevance model.

Phase 04 optimized recall across semantic and lexical query classes.

Phase 05 addresses the expected side effect:

> Better recall can introduce more irrelevant context.

The reranker reorders already retrieved candidates. It does not replace retrieval.

---

## 2. Starting State

Phase 04 must provide:

- metadata-aware Dense retrieval
- metadata-aware Lexical retrieval
- Hybrid retrieval
- RRF fusion
- stable candidate IDs/provenance
- `FUSION_TOP_K`
- mixed-query golden cases
- deterministic + RAGAS Phase 04 report

Phase 05 must not change the Dense, Lexical, or RRF implementations while measuring reranking.

---

## 3. New Capability

Add one local reranker.

Default:

```text
cross-encoder/ms-marco-MiniLM-L-6-v2
```

Required flow:

```text
Dense + Lexical
      |
      v
     RRF
      |
      v
fused candidates
      |
      v
 cross-encoder
      |
      v
 reranked Top-K
```

Recommended:

```text
FUSION_TOP_K = 20–30
RERANK_TOP_K = 5
```

Exact values are configuration and must be evaluated.

---

## 4. Dependency Delta

Add only the package(s) required to run the selected local cross-encoder.

Prefer reusing `sentence-transformers` if it is already installed for embeddings.

If it already exists, expected new package count:

```text
0
```

Do not add a managed reranking SDK in the baseline.

No new infrastructure service.

---

## 5. Reranker Contract

Introduce a small interface:

```text
rerank(query, candidates, top_k)
    -> reranked candidates
```

Input candidate already contains:

- chunk_id
- content
- metadata
- dense rank/score
- lexical rank/score
- RRF rank/score

Add:

- rerank_score
- rerank_rank

Never discard original retrieval scores in debug mode.

---

## 6. Batch Behavior

The reranker should score candidate pairs in batches where supported.

Do not issue one model initialization per request.

Model lifecycle:

```text
application startup
      |
      v
load reranker once
      |
      v
reuse per request
```

Keep implementation simple.

No model-serving microservice.

---

## 7. Failure Handling

If the reranker cannot load at application startup:

- fail startup clearly for the Phase 05 default profile

If reranking fails during a request:

- return a stable application error in normal mode
- do not silently pretend reranking happened

A later production system could support graceful fallback, but this learning phase should make failures visible.

---

## 8. Context Selection

Generation receives only the top reranked chunks.

Example:

```text
Hybrid fused 25
      |
      v
reranker
      |
      v
top 5
      |
      v
context builder
```

Do not increase final context size just because the candidate pool became larger.

---

## 9. API Changes

Normal:

```http
POST /rag/query
```

Phase default pipeline:

```text
HYBRID + RERANK
```

Debug:

```http
POST /rag/retrieve
```

Must expose:

- fused ordering
- rerank score
- rerank rank
- final selected context

Allow debug comparison with reranking disabled without changing deployment configuration.

Example:

```json
{
  "question": "...",
  "strategy": "HYBRID",
  "rerank": false
}
```

and:

```json
{
  "question": "...",
  "strategy": "HYBRID",
  "rerank": true
}
```

---

## 10. Evaluation

Freeze:

- corpus
- chunking strategy
- embedding model
- query analysis
- Dense settings
- Lexical settings
- RRF settings

Compare only:

```text
A. Hybrid + RRF
B. Hybrid + RRF + Rerank
```

Deterministic metrics:

- Recall@K
- Precision@K
- MRR
- nDCG@K

Mandatory RAGAS:

- Context Precision
- Context Recall
- Faithfulness
- Response Relevancy

Operational metrics:

- reranker latency
- end-to-end latency
- final context count

Expected lesson:

- first-stage retrieval protects recall
- reranker improves ordering/precision
- reranker adds latency

---

## 11. Evaluation Dataset

Keep all prior IDs.

Add hard-negative cases:

- semantically similar wrong country
- semantically similar wrong expense category
- same term in current and obsolete policy
- same policy code mentioned in cross-reference section
- long chunk with query keyword but wrong rule

These cases make ranking quality measurable.

---

## 12. Setup Option A — No Docker

No new infrastructure service.

The local model downloads on first use unless pre-cached.

```bash
git checkout <phase-05-branch>
uv sync
cp .env.example .env

uv run alembic upgrade head
uv run python scripts/ingest.py --reset
uv run uvicorn app.main:app --reload
```

Then:

```bash
uv run python evals/run.py --profile smoke
uv run python evals/run.py --profile full
```

Document approximate model download size in the implementation README once the exact model version is locked.

---

## 13. Setup Option B — Docker

Database remains the only mandatory container:

```bash
docker compose up -d db

uv sync
cp .env.example .env
uv run alembic upgrade head
uv run python scripts/ingest.py --reset
uv run uvicorn app.main:app --reload
```

Reranker stays in the application process.

Do not create a model-serving container.

---

## 14. Tests

Unit:

- reranker adapter preserves candidate identity
- top_k respected
- order follows rerank score
- empty candidate list
- fewer candidates than top_k
- original Dense/Lexical/RRF fields preserved

Integration:

- local model loads
- fixed query/candidate set produces deterministic-enough relative ordering
- complete Hybrid → Rerank pipeline works

API:

- debug with reranking on/off
- normal query uses reranked context
- error contract on model failure where testable

---

## 15. Acceptance Gate

Phase 05 is complete when:

1. Phase 04 Hybrid pipeline still runs without reranking in debug/eval mode
2. local reranker is loaded once and reused
3. no new infrastructure service exists
4. candidate provenance survives reranking
5. full report compares Hybrid vs Hybrid+Rerank
6. mandatory RAGAS metrics exist
7. reranking latency is measured
8. hard-negative cases exist
9. the reason for moving to Phase 06 is not hidden inside reranker logic

---

## 16. Handoff to Phase 06

Phase 06 receives three already working retrieval modes:

```text
DENSE
LEXICAL
HYBRID + optional RERANK
```

Phase 06 does not invent a new retriever.

It learns how to choose the appropriate existing strategy per query.
