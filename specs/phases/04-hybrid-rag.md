# Phase 04 — Hybrid Retrieval with Reciprocal Rank Fusion

## 1. Goal

Combine semantic and lexical evidence without incorrectly mixing incompatible raw score scales.

Introduce:

- parallel dense + lexical retrieval
- candidate deduplication
- Reciprocal Rank Fusion
- hybrid debug visibility
- category-level evaluation against both previous retrievers

Do not add reranking yet.

---

## 2. Starting State

Phase 03 provides:

- metadata-aware dense retriever
- metadata-aware lexical retriever
- shared candidate contract
- exact-term evaluation cases
- semantic evaluation cases
- stable query analysis
- common policy applicability logic

If the two retrievers do not emit the same candidate identity/provenance structure, fix Phase 03 before starting fusion.

---

## 3. Dependency Delta

Expected new dependency count:

```text
0
```

RRF is simple application code.

Do not add a fusion library.

The learning goal is to understand rank fusion directly.

---

## 4. Hybrid Retrieval Flow

Implement:

```text
               query
                 |
          query analysis
                 |
          shared filters
             /       \
            v         v
         dense      lexical
            \         /
             \       /
              dedupe
                 |
                 v
                RRF
                 |
                 v
           fused Top-K
```

Config:

- `DENSE_TOP_K`
- `LEXICAL_TOP_K`
- `FUSION_TOP_K`
- `RRF_K`

---

## 5. Deduplication

Before fusion, identify the same evidence by stable `chunk_id`.

If the same chunk appears in both lists:

- keep one candidate
- preserve both ranks/scores
- add both contributions to RRF

Optional content-hash deduplication may handle accidental duplicate chunks.

Do not dedupe different policy versions solely because text is similar.

Version provenance matters.

---

## 6. RRF

Use:

```text
score(d) = Σ 1 / (k + rank_i(d))
```

For each candidate store:

- dense_rank
- dense_score
- lexical_rank
- lexical_score
- rrf_score
- rrf_rank

Raw dense and lexical scores remain visible only for debugging.

Final fusion ordering comes from RRF.

Do not normalize and add unrelated score scales.

---

## 7. Candidate Contract Extension

Extend the common candidate model in a backward-compatible way.

Fields:

```text
chunk_id
content
metadata

dense_score?
dense_rank?

lexical_score?
lexical_rank?

rrf_score?
rrf_rank?
```

Phase 05 reranker will extend this same object rather than invent a second structure.

---

## 8. API Behavior

`POST /rag/retrieve` must support:

```text
DENSE
LEXICAL
HYBRID
```

Hybrid debug response should show:

- dense list
- lexical list
- deduplicated set
- fused ordering

`POST /rag/query` may use HYBRID as the new phase default.

Adaptive selection does not happen until Phase 06.

---

## 9. Evaluation

Required comparison:

```text
A. Metadata Dense
B. Metadata Lexical
C. Hybrid + RRF
```

Use the exact same golden dataset and corpus build.

Do not re-chunk or change embeddings while evaluating fusion.

Deterministic metrics:

- Hit Rate@K
- Recall@K
- Precision@K
- MRR
- nDCG@K

Mandatory RAGAS:

- Context Precision
- Context Recall
- Faithfulness
- Response Relevancy

Report by category:

- semantic
- exact identifier
- metadata
- version
- mixed semantic+exact
- abstention

Expected learning outcome:

Hybrid usually improves coverage across mixed workloads, but may increase noise. That trade-off becomes the reason for Phase 05 reranking.

---

## 10. Dataset Expansion

Add mixed-query cases such as:

```text
"Under TRV-0042, what is the hotel limit for L6 in Singapore?"
```

These require both:

- exact identifier handling
- semantic policy content matching
- metadata applicability

Keep previous cases unchanged.

---

## 11. Setup Option A — No Docker

No new infrastructure.

```bash
git checkout <phase-04-branch>
uv sync
cp .env.example .env

uv run alembic upgrade head
uv run python scripts/ingest.py --reset
uv run uvicorn app.main:app --reload
```

No schema change is strictly required unless fusion debug persistence is deliberately added; prefer not persisting transient fusion results.

Run smoke/full evaluation as before.

---

## 12. Setup Option B — Docker

```bash
git checkout <phase-04-branch>
docker compose up -d db

uv sync
cp .env.example .env

uv run alembic upgrade head
uv run python scripts/ingest.py --reset
uv run uvicorn app.main:app --reload
```

Still one database container only.

---

## 13. Tests

Unit:

- RRF formula
- stable rank ordering
- same chunk present in both lists
- candidate only in dense
- candidate only in lexical
- deduplication
- tie behavior

Integration:

- run dense + lexical against same PostgreSQL corpus
- fuse results
- verify stable chunk provenance

API:

- HYBRID debug strategy
- mixed semantic/exact query
- response includes fused ranks in debug mode

---

## 14. Acceptance Gate

Phase 04 completes when:

1. dense and lexical retrievers remain independently runnable
2. hybrid uses no new external service
3. RRF is unit-tested
4. fusion preserves source provenance
5. no incompatible raw score addition occurs
6. mixed-query dataset exists
7. comparison report covers Dense vs Lexical vs Hybrid
8. RAGAS metrics are present
9. observed hybrid noise is documented for Phase 05

---

## 15. Handoff to Phase 05

Phase 05 receives:

- fused candidates
- common ranking/provenance object
- `FUSION_TOP_K`
- evaluation evidence showing whether hybrid increased noise

Phase 05 adds a second-stage relevance model to reorder the fused candidate set.
