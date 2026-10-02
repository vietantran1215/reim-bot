# Phase 06 — Adaptive Retrieval Routing

## 1. Goal

Stop running the same retrieval strategy for every question.

Introduce a deterministic query router that chooses among already implemented retrieval capabilities.

Routes:

```text
DENSE
LEXICAL
HYBRID
NO_RETRIEVAL
```

Reranking remains a post-retrieval stage and can be enabled for HYBRID or other configured routes.

This is Adaptive RAG, not Agentic RAG.

---

## 2. Starting State

Phase 05 provides:

- Dense retriever
- Lexical retriever
- Hybrid + RRF
- local reranker
- common candidate contract
- structured query analysis
- metadata/applicability filters
- RAGAS + deterministic evaluation

If any route points to a pipeline that cannot already run independently, there is a gap. Fix the prior phase first.

---

## 3. Dependency Delta

Expected new dependency count:

```text
0
```

The baseline router must be deterministic Python logic.

Do not add:

- LangGraph
- agent frameworks
- planner
- LLM router dependency
- new model call

The router should be cheap, explainable, and easy to evaluate.

---

## 4. Router Input

Use the existing typed query analysis.

Example:

```json
{
  "normalized_query": "TRV-0042 hotel limit Singapore",
  "country": "SG",
  "expense_type": "HOTEL",
  "employee_grade": null,
  "policy_code": "TRV-0042",
  "has_exact_identifier": true,
  "has_quoted_phrase": false
}
```

Do not parse the raw query independently inside the router.

---

## 5. Router Output

Typed contract:

```json
{
  "strategy": "HYBRID",
  "reason_code": "MIXED_SEMANTIC_AND_EXACT"
}
```

Allowed reason codes should be enums.

Suggested initial reasons:

- EXACT_POLICY_IDENTIFIER
- EXACT_PHRASE
- SEMANTIC_POLICY_QUESTION
- MIXED_SEMANTIC_AND_EXACT
- NO_KNOWLEDGE_REQUIRED

No hidden chain-of-thought.

---

## 6. Deterministic Routing Rules

Example priority:

### NO_RETRIEVAL

Only for explicit non-knowledge interactions supported by the API.

Keep this narrow.

### LEXICAL

Prefer when the query is primarily:

- exact policy code lookup
- exact quoted phrase
- identifier-centric lookup

### DENSE

Prefer broad semantic policy questions with no strong exact token.

### HYBRID

Prefer mixed questions containing:

- exact identifier + semantic question
- structured constraint + semantic question
- ambiguous blend of exact and conceptual wording

Rules must be simple enough to inspect in one module.

---

## 7. Pipeline Composition

Router chooses first-stage retrieval.

Then existing post-processing applies.

Example:

```text
query
  |
analysis
  |
router
  |
  +--> DENSE ----+
  |              |
  +--> LEXICAL --+--> optional rerank --> context
  |              |
  +--> HYBRID ---+
  |
  +--> NO_RETRIEVAL --> supported direct response
```

Metadata applicability filtering remains shared and mandatory where retrieval occurs.

---

## 8. Configuration

Add only router-related configuration that is actually needed.

Examples:

- `DEFAULT_RETRIEVAL_STRATEGY`
- `RERANK_ENABLED`

Do not create dozens of tuning flags for rule thresholds unless tests/evaluation justify them.

---

## 9. API Changes

Normal `POST /rag/query` now uses router-selected strategy.

Response includes:

```json
{
  "strategy": "HYBRID",
  "route_reason": "MIXED_SEMANTIC_AND_EXACT"
}
```

Debug endpoint may accept an override:

```json
{
  "question": "...",
  "strategy_override": "DENSE"
}
```

This exists only for comparison/testing.

Normal user flow should use automatic routing.

---

## 10. Golden Dataset Expansion

Every case must now include:

```text
expected_route
```

Add route-specific examples.

Required categories:

- semantic-only
- identifier-only
- quoted exact phrase
- mixed semantic + exact
- metadata-sensitive semantic
- no-retrieval supported case

Keep old IDs.

Backfill expected routes for previous cases rather than recreating them.

---

## 11. Evaluation

Add deterministic:

- route accuracy
- per-route confusion matrix
- unnecessary retrieval rate
- route-specific latency

Continue:

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

Compare:

```text
A. Always HYBRID + Rerank
B. Adaptive routing + configured rerank policy
```

The Adaptive system does not need higher quality on every case.

It may win by preserving quality while reducing:

- retrieval work
- reranker work
- latency

Report quality and cost/latency together.

---

## 12. Setup Option A — No Docker

No new service or model.

```bash
git checkout <phase-06-branch>
uv sync
cp .env.example .env

uv run alembic upgrade head
uv run python scripts/ingest.py --reset
uv run uvicorn app.main:app --reload
```

No re-index is required if corpus/index configuration did not change, but `--reset` must still work reproducibly.

Run smoke/full evaluation.

---

## 13. Setup Option B — Docker

Same one-container DB setup:

```bash
docker compose up -d db
uv sync
cp .env.example .env
uv run alembic upgrade head
uv run python scripts/ingest.py --reset
uv run uvicorn app.main:app --reload
```

No new container.

---

## 14. Tests

Unit:

- each routing rule
- priority when multiple conditions match
- reason codes
- unsupported/empty query handling
- override behavior

Integration:

- routed Dense path
- routed Lexical path
- routed Hybrid path
- metadata filters survive routing
- reranker called only when configured

API:

- response reports route
- debug override
- invalid override
- no-retrieval supported path

Evaluation:

- route accuracy computed from golden labels

---

## 15. Acceptance Gate

Phase 06 completes when:

1. router uses existing query analysis
2. no new model/dependency is required for routing
3. every route points to a working pre-existing pipeline
4. golden cases contain expected routes
5. route accuracy is reported
6. quality metrics remain reported
7. RAGAS remains mandatory
8. latency/work reduction is measured against always-Hybrid baseline
9. no agent loop/tool calling exists

---

## 16. Handoff to Phase 07

Phase 07 receives:

- final selected retrieval strategy
- optional reranked final candidates
- stable provenance
- measured final context before generation

Phase 07 does not change retrieval strategy.

It reduces noise inside the already selected evidence.
