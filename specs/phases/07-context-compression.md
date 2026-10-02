# Phase 07 — Query-Aware Context Compression

## 1. Goal

Reduce final context size and irrelevant text after retrieval/reranking without losing evidence needed for a correct answer.

This phase starts **after** retrieval quality has already been improved.

Do not use compression to hide a bad retriever.

---

## 2. Starting State

Phase 06 provides:

- Adaptive route selection
- Dense/Lexical/Hybrid retrievers
- RRF
- local reranker
- final selected candidates
- citation provenance
- RAGAS evaluation pipeline
- route labels and reports

Phase 07 must leave retrieval ordering unchanged when comparing compression.

---

## 3. New Capability

Add default free extractive compression.

Pipeline:

```text
retrieved/reranked chunks
          |
          v
sentence segmentation
          |
          v
query-aware sentence scoring
          |
          v
provenance-preserving selection
          |
          v
context token budget
          |
          v
compressed evidence
```

No extra LLM call in the baseline.

---

## 4. Dependency Delta

Prefer zero new external service.

A small sentence/token utility dependency may be added only if existing packages cannot do the job cleanly.

Do not add:

- summarization API
- second LLM request per chunk
- vector database
- agent framework

Compression must be inspectable.

---

## 5. Compressor Contract

Input:

- original user query
- ordered final candidates
- max context budget

Output for each compressed evidence item:

- chunk_id
- original section metadata
- selected text spans/sentences
- original rank
- original source provenance
- compressed token count

Citation identity remains the original chunk/source identity.

Compression must never fabricate source metadata.

---

## 6. Baseline Compression Algorithm

Recommended baseline:

1. split final chunk text into sentences
2. score sentence relevance to query
3. retain top relevant sentences while preserving original order
4. preserve neighboring sentence when necessary for meaning
5. stop at context budget
6. deduplicate repeated sentences/spans

The exact relevance scoring method must be deterministic/free for the baseline.

Possible implementations:

- lexical overlap
- embedding similarity using already loaded embedding model
- weighted combination

Do not add a new model solely for compression.

---

## 7. Context Budget

Add:

- `MAX_CONTEXT_TOKENS`

The Context Builder must enforce the budget after compression.

Record:

- original context token count
- compressed context token count
- compression ratio

Do not truncate blindly through the middle of citation evidence if a sentence-aware choice can avoid it.

---

## 8. Debug Visibility

`POST /rag/retrieve` should show:

- final candidates before compression
- selected spans after compression
- token counts before/after
- preserved citation IDs

Do not expose hidden reasoning.

This debug data is algorithm output, not chain-of-thought.

---

## 9. API Behavior

Normal `POST /rag/query` uses compressed context by default in Phase 07.

For evaluation/debug allow:

```json
{
  "compression": false
}
```

versus:

```json
{
  "compression": true
}
```

This switch exists so the same branch can compare the technique.

---

## 10. Golden Dataset Expansion

Add cases where:

- relevant sentence is buried in a long chunk
- chunk contains multiple rules but only one applies
- neighboring sentence is needed for condition/exception
- duplicate evidence appears across chunks
- over-compression would remove an important exception

These cases prevent the benchmark from rewarding aggressive token deletion.

---

## 11. Evaluation

Freeze retrieval/reranking configuration.

Compare:

```text
A. Adaptive + Rerank + raw context
B. Adaptive + Rerank + compressed context
```

Operational:

- context tokens
- compression ratio
- compression latency
- end-to-end latency

Deterministic:

- expected fact coverage
- citation validity
- abstention correctness

Mandatory RAGAS:

- Context Precision
- Context Recall
- Faithfulness
- Response Relevancy

Expected lesson:

Compression should reduce context noise/tokens while maintaining evidence coverage.

If Context Precision rises but Context Recall collapses, compression is too aggressive.

---

## 12. Setup Option A — No Docker

No new infrastructure.

```bash
git checkout <phase-07-branch>
uv sync
cp .env.example .env

uv run alembic upgrade head
uv run python scripts/ingest.py --reset
uv run uvicorn app.main:app --reload
```

No re-index is required solely because the compressor changed, but reproducible reset ingestion remains mandatory.

Run both eval profiles.

---

## 13. Setup Option B — Docker

```bash
docker compose up -d db

uv sync
cp .env.example .env
uv run alembic upgrade head
uv run python scripts/ingest.py --reset
uv run uvicorn app.main:app --reload
```

No additional service.

---

## 14. Tests

Unit:

- sentence selection
- context budget
- stable original-order reconstruction
- duplicate sentence removal
- citation provenance preservation
- empty/short chunk behavior
- compression disabled behavior

Integration:

- real retrieved candidates compressed
- token budget respected
- source IDs preserved to generation

API:

- compression on/off
- debug before/after context
- citation validation after compression

Evaluation:

- compression ratio
- RAGAS comparison
- expected fact coverage

---

## 15. Acceptance Gate

Phase 07 completes when:

1. retrieval/reranking baseline is unchanged for comparison
2. compression introduces no new infrastructure
3. citation provenance is preserved
4. context budget is enforced
5. compression ratio is reported
6. expected-fact coverage is reported
7. all mandatory RAGAS metrics exist
8. long/noisy-context benchmark cases exist
9. quality degradation, if any, is explicitly documented

---

## 16. Handoff to Phase 08

Phase 08 receives the full feature set:

```text
metadata-aware retrieval
+
Dense
+
Lexical
+
RRF Hybrid
+
Reranking
+
Adaptive routing
+
Context compression
+
RAGAS
```

Phase 08 does not add another major retrieval technique.

It makes the complete system reproducibly comparable and tuneable.
