# Phase 02 — Metadata-Aware RAG and Policy Applicability

## 1. Goal

Fix a major Dense RAG failure class:

> A semantically similar chunk is not necessarily the applicable policy.

Introduce structured policy metadata, version/effective-date rules, metadata-aware retrieval, and heading-aware chunking while preserving the Phase 01 dense baseline.

---

## 2. Starting State

Phase 01 must already provide:

- FastAPI application
- PostgreSQL + pgvector
- policy ingestion
- recursive chunking
- dense retriever
- query/generation/citation pipeline
- golden dataset
- deterministic metric runner
- RAGAS runner
- Phase 01 baseline report

If any of these are absent, Phase 02 is not ready to start.

---

## 3. New Capabilities

Add:

- country metadata
- expense-type metadata
- employee-grade metadata
- policy status
- effective_from
- effective_to
- explicit policy version applicability
- deterministic query metadata extraction
- metadata filtering during dense retrieval
- heading-aware chunking strategy
- baseline comparison against Phase 01

Do not add lexical retrieval yet.

---

## 4. Dependency Delta

Prefer **zero new runtime dependency** if existing parser/Pydantic/text-splitter code can implement the phase.

Do not add an LLM just for metadata extraction.

Metadata extraction baseline must be deterministic using:

- known country names/codes
- known expense categories
- grade patterns such as L4/L5/L6
- policy-code patterns
- date parsing

If a tiny parsing dependency is truly required, document exactly why.

---

## 5. Schema Migration

Extend policy version metadata with:

- country
- expense_types
- employee_grades
- effective_from
- effective_to
- status

Keep existing IDs stable.

Do not rebuild the logical identity of documents unnecessarily.

Add indexes useful for metadata filtering.

Migration must support:

```bash
uv run alembic upgrade head
```

from a Phase 01 database.

Also verify a clean database can migrate directly to head.

---

## 6. Corpus Update

Update committed policy sources so metadata contains enough variation to create real applicability failures.

Required traps:

- same topic, different country
- same topic, different employee grade
- old and current version of same policy
- future policy version
- expired policy version

Example:

```text
Singapore Hotel Policy 2025
Singapore Hotel Policy 2026
Vietnam Hotel Policy 2026
```

The Phase 01 evaluation cases keep the same IDs.

Add new metadata-specific cases rather than replacing old cases.

---

## 7. Heading-Aware Chunking

Add second chunking strategy:

```text
recursive
heading-aware
```

Heading-aware chunks preserve:

- document title
- section hierarchy
- section identifier
- policy version metadata

Example:

```text
4. Travel Expenses
  4.1 Hotel
  4.2 Meals
```

A chunk from 4.1 must retain `4.1 Hotel` provenance.

Config:

- `CHUNKING_STRATEGY=recursive|heading`

Default for Phase 02:

```text
heading
```

Changing chunking strategy requires re-indexing.

---

## 8. Query Metadata Analysis

Introduce typed query analysis.

Example output:

```json
{
  "normalized_query": "hotel limit L6 Singapore",
  "country": "SG",
  "expense_type": "HOTEL",
  "employee_grade": "L6",
  "policy_code": null,
  "query_date": null
}
```

Rules must be deterministic and unit-tested.

No chain-of-thought.

No LLM router.

---

## 9. Policy Applicability Rules

When no historical date is supplied:

- status must be ACTIVE
- effective_from <= today
- effective_to is null or effective_to >= today

When a historical date is supplied:

- effective_from <= query_date
- effective_to is null or effective_to >= query_date

When metadata is absent in the query:

- do not invent a filter
- retrieval may search broader scope

When metadata is present:

- apply it before or within vector retrieval

The API must not rely on the LLM to choose which policy version is valid.

---

## 10. Dense Retrieval Changes

Preserve the Phase 01 dense retriever interface.

Add optional filters:

- country
- expense_type
- employee_grade
- policy_code
- applicability date/status

Candidate output remains backward-compatible.

Add debug fields showing applied filters.

---

## 11. Ingestion

Required command remains:

```bash
uv run python scripts/ingest.py --reset
```

It must now persist:

- heading provenance
- applicability metadata
- current chunking strategy

Re-index is mandatory because chunk boundaries and metadata changed.

The ingestion report must record:

- chunking strategy
- corpus checksum/version
- embedding model

---

## 12. Evaluation

Preserve all Phase 01 metrics.

Add failure-category reporting:

- wrong country
- wrong grade
- wrong version
- expired policy
- future policy
- wrong expense type

Compare:

```text
Phase 01: Dense
vs
Phase 02: Metadata-filtered Dense
```

Mandatory RAGAS comparison:

- Context Precision
- Context Recall
- Faithfulness
- Response Relevancy

Also compare heading-aware vs recursive chunking on the same dataset.

Do not change multiple variables in one experiment.

Recommended runs:

```text
A. recursive + dense
B. heading + dense
C. heading + metadata-filtered dense
```

---

## 13. Setup Option A — No Docker

Start from a Phase 01-compatible local PostgreSQL + pgvector database.

```bash
git checkout <phase-02-branch>
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

No new external service is introduced.

---

## 14. Setup Option B — Docker

```bash
git checkout <phase-02-branch>
docker compose up -d db

uv sync
cp .env.example .env

uv run alembic upgrade head
uv run python scripts/ingest.py --reset
uv run uvicorn app.main:app --reload
```

Then run smoke/full evaluation.

No additional container is allowed for this phase.

---

## 15. Tests

Unit:

- country extraction
- expense-type extraction
- employee-grade extraction
- date extraction
- policy applicability
- heading-aware chunk boundaries
- metadata filter construction

Integration:

- metadata-filtered pgvector search
- current vs expired version selection
- historical date selection
- re-ingestion with heading-aware chunks

API:

- debug endpoint exposes applied filters
- current-policy query
- historical-policy query
- no-filter semantic query remains supported

---

## 16. Acceptance Gate

Phase 02 completes when:

1. Phase 01 tests still pass unless intentionally updated by schema contract
2. migration works from Phase 01 and from empty DB
3. ingestion reset is reproducible
4. old evaluation case IDs remain intact
5. new metadata/version cases exist
6. Phase 01 and Phase 02 reports can be compared
7. mandatory RAGAS metrics exist
8. wrong-policy retrieval failures demonstrably decrease or are explicitly explained
9. no lexical-search dependency exists

---

## 17. Handoff to Phase 03

Phase 03 receives:

- structured query metadata
- applicable policy filtering
- heading-aware chunks
- filtered dense retriever
- expanded golden dataset
- stable candidate contract

Phase 03 adds a second retrieval family—lexical search—without changing dense behavior.
