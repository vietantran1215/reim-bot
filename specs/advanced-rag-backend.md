# Reim Bot — Advanced RAG Backend Specification

## 1. Purpose

Reim Bot is a **learning-first Advanced RAG backend** built with FastAPI for the Enterprise Expense Reimbursement domain.

The project is intentionally designed to teach how retrieval quality evolves from a basic vector-search baseline into a measurable Advanced RAG pipeline.

The primary learning question is:

> When a RAG answer is wrong, which retrieval decision caused it, what technique should improve it, and what metric proves the improvement?

This is not a generic chatbot project.

This is not an agent project.

This is not a microservices project.

The project should remain small enough that a learner can clone the repository, start one database, install Python dependencies, start the API, and experiment with retrieval strategies without spending most of the time configuring infrastructure.

---

## 2. Design Priorities

When two solutions are technically valid, prefer them in this order:

1. Free.
2. Local-first.
3. Fewer setup steps.
4. Fewer infrastructure components.
5. Easier to inspect and understand.
6. Production-relevant concept.
7. Replaceable later if scale requires it.

The project must not add technology because it is commonly associated with "enterprise architecture."

Every dependency must justify its learning value.

---

## 3. Business Problem

Employees need reliable answers from company expense reimbursement policies.

Example questions:

- "What is the maximum hotel reimbursement in Singapore?"
- "Can an L6 employee claim a USD 450 hotel?"
- "What does policy TRV-0042 say about airport transport?"
- "Is client entertainment reimbursable in Vietnam?"
- "Which policy applies after January 1, 2026?"
- "What changed between the 2025 and 2026 travel policies?"
- "Can I claim taxi expenses without a receipt?"
- "What is the meal allowance for Japan?"

The policy corpus contains retrieval challenges that expose weaknesses of vector-only RAG:

- exact policy identifiers
- country and region constraints
- employee grade constraints
- expense categories
- numeric limits
- policy versioning
- effective dates
- semantically similar terminology
- overlapping policies
- obsolete policy versions
- questions that should return no answer

---

## 4. Learning Outcomes

A learner completing this project should be able to explain and implement:

1. Basic semantic RAG.
2. Metadata-aware retrieval.
3. Lexical retrieval.
4. Hybrid retrieval.
5. Reciprocal Rank Fusion.
6. Reranking.
7. Adaptive retrieval strategy selection.
8. Query normalization and structured metadata extraction.
9. Context deduplication.
10. Context compression.
11. Policy version filtering.
12. Citation generation from trusted retrieval metadata.
13. Retrieval evaluation.
14. Generation evaluation.
15. Latency and token-cost trade-offs.
16. Failure analysis across ingestion, retrieval, context construction, and generation.

The learner must not merely implement a final pipeline.

Each technique must be compared against an earlier baseline.

---

## 5. Explicit Non-Goals

Do not implement the following unless a later learning exercise explicitly extends the project:

- authentication service
- microservices
- frontend application
- agent loop
- LangGraph
- MCP
- tool calling
- long-term memory
- multi-agent systems
- Redis
- Kafka
- RabbitMQ
- Kubernetes
- service mesh
- API gateway
- OpenSearch
- Elasticsearch
- dedicated vector database
- GraphRAG
- multimodal RAG
- OCR
- fine-tuning

These would distract from the retrieval-engineering objective.

---

## 6. System Architecture

The baseline system contains one FastAPI application and one PostgreSQL instance.

```text
Client
  |
  v
FastAPI
  |
  +-------------------+
  |                   |
  v                   v
Policy Ingestion    RAG Query
  |                   |
  v                   v
Chunking         Query Analysis
  |                   |
  v                   v
Embeddings       Strategy Router
  |                   |
  v           +-------+-------+
PostgreSQL      |               |
+ pgvector      v               v
             Dense          Lexical
             Search          Search
                \             /
                 \           /
                  v         v
                     RRF
                      |
                      v
                   Reranker
                      |
                      v
              Context Builder
                      |
                      v
                 Compressor
                      |
                      v
                    LLM
                      |
                      v
             Answer + Citations
```

Evaluation is a local CLI workflow against the same application components and database.

No evaluation microservice is required.

---

## 7. Infrastructure

### Required infrastructure

Exactly one infrastructure service is required for the default learning path:

```text
PostgreSQL + pgvector
```

Run it through Docker Compose.

The project must not require learners to install PostgreSQL directly on their machine.

### Default local topology

```text
Local machine
├── FastAPI process
├── local embedding model
├── local reranker model
├── optional local LLM runtime
│
└── Docker Compose
    └── PostgreSQL + pgvector
```

---

## 8. Technology Stack

### Backend

- Python 3.13
- FastAPI
- Pydantic v2
- SQLAlchemy 2.x
- Alembic
- HTTPX
- uv

### Database

- PostgreSQL
- pgvector
- PostgreSQL Full-Text Search

### RAG

Use LangChain selectively where it removes boilerplate, especially for:

- model adapters
- embeddings
- selected document loaders
- text splitters

Core retrieval behavior should remain explicit Python code.

Do not hide the learning flow behind a large prebuilt RAG chain.

### Embeddings

Default learning profile:

- local sentence-transformer embedding model
- small enough to run on CPU
- no API charge

Recommended implementation profile:

```text
BAAI/bge-small-en-v1.5
```

The embedding model name must remain configuration.

The vector dimension must never be hard-coded outside the embedding/index configuration boundary.

### Reranker

Default:

```text
cross-encoder/ms-marco-MiniLM-L-6-v2
```

Run locally.

The reranker implementation must be behind an interface so another reranker can be substituted later.

### Generation model

Generation must use a replaceable provider interface.

Two supported learning profiles:

#### Free/local profile

Use a local Ollama-compatible chat model.

This is the preferred path when zero API cost matters more than the extra local model installation.

#### API profile

Allow any configured OpenAI-compatible chat model.

This is optional and may incur cost.

Retrieval evaluation must not require a paid generation provider.

### Evaluation

Default free metrics should be implemented directly in Python.

Ragas may be added for generation-quality metrics when an evaluator model is available.

Do not make paid LLM-as-a-judge evaluation mandatory.

---

## 9. Repository Target Structure

```text
reim-bot/
├── app/
│   ├── main.py
│   │
│   ├── core/
│   │   ├── config.py
│   │   └── errors.py
│   │
│   ├── db/
│   │   └── session.py
│   │
│   ├── policies/
│   │   ├── models.py
│   │   ├── schemas.py
│   │   ├── repository.py
│   │   ├── parser.py
│   │   ├── chunker.py
│   │   └── ingestion.py
│   │
│   ├── rag/
│   │   ├── contracts.py
│   │   ├── query_analyzer.py
│   │   ├── router.py
│   │   ├── dense.py
│   │   ├── lexical.py
│   │   ├── fusion.py
│   │   ├── reranker.py
│   │   ├── compressor.py
│   │   ├── context_builder.py
│   │   ├── generator.py
│   │   └── service.py
│   │
│   └── api/
│       ├── policies.py
│       └── rag.py
│
├── knowledge/
│   └── expense-policies/
│
├── evals/
│   ├── dataset.json
│   ├── run.py
│   └── reports/
│
├── tests/
│
├── migrations/
├── compose.yml
├── pyproject.toml
└── README.md
```

Do not create empty future-phase modules before they are needed.

The structure should evolve with the phases.

---

## 10. Policy Corpus

The repository should contain a small synthetic or openly shareable policy corpus.

Recommended size:

```text
20–50 policy documents or policy sections
```

The corpus should deliberately include difficult retrieval cases.

### Required variation

Include:

- Singapore policy
- Vietnam policy
- Japan policy
- United States policy
- global expense policy
- travel policy
- hotel policy
- meal policy
- transport policy
- client entertainment policy
- receipt policy

### Required ambiguity

Include examples where:

- two countries use different reimbursement limits
- two employee grades have different limits
- old and new versions coexist
- exact policy IDs matter
- similar terms appear across unrelated sections
- obsolete policies contain plausible but incorrect answers

This corpus is part of the learning design.

A corpus where every answer is trivially retrieved does not teach Advanced RAG.

---

## 11. Policy Metadata

Every policy version must carry structured metadata.

Minimum metadata:

```json
{
  "document_id": "travel-policy-sg",
  "policy_code": "TRV-0042",
  "version": "2026.1",
  "title": "Singapore Travel Policy",
  "country": "SG",
  "policy_type": "TRAVEL",
  "expense_types": ["HOTEL", "MEAL", "TRANSPORT"],
  "employee_grades": ["L4", "L5", "L6"],
  "effective_from": "2026-01-01",
  "effective_to": null,
  "status": "ACTIVE"
}
```

Chunk-level metadata must also contain:

- chunk_id
- document_id
- version_id
- section
- chunk_index
- source checksum

---

## 12. Data Model

Keep the schema small.

### policy_documents

Represents the stable policy identity.

Suggested fields:

- id
- document_id
- policy_code
- title
- policy_type

### policy_versions

Represents one policy version.

Suggested fields:

- id
- policy_document_id
- version
- country
- employee_grades
- expense_types
- effective_from
- effective_to
- status
- source_path
- checksum
- created_at

### policy_chunks

Represents retrievable chunks.

Suggested fields:

- id
- policy_version_id
- section
- chunk_index
- content
- embedding
- search_vector
- token_count
- metadata JSONB

### ingestion_runs

Suggested fields:

- id
- started_at
- completed_at
- document_count
- chunk_count
- embedding_model
- chunking_strategy
- status
- error_message

Evaluation results may initially be stored as JSON files rather than database tables.

Do not add evaluation tables until persistence creates real learning value.

---

## 13. Ingestion Pipeline

The ingestion pipeline is:

```text
Policy files
   |
   v
Parse
   |
   v
Validate metadata
   |
   v
Determine policy version
   |
   v
Chunk
   |
   v
Embed
   |
   v
Store chunks + metadata + search vector
```

### Supported source formats

Default:

- Markdown
- plain text

PDF parsing is optional.

Do not make PDF parsing a prerequisite because it introduces document-parser noise unrelated to Advanced RAG.

### Idempotency

Ingestion should calculate a source checksum.

If the same document version and checksum already exist, re-ingestion should not create duplicate chunks.

### Re-indexing

Changing any of the following should require a new ingestion/index build:

- embedding model
- embedding dimension
- chunking strategy
- major metadata extraction logic

The ingestion run must record enough configuration to reproduce the index.

---

## 14. Chunking

### Baseline

Start with recursive text chunking.

Configurable values:

- chunk_size
- chunk_overlap

### Advanced experiment

Add heading-aware chunking.

Policy example:

```text
4. Travel Expenses
  4.1 Hotel
  4.2 Meals
  4.3 Ground Transport
```

The heading-aware strategy should preserve:

- heading hierarchy
- section identifier
- section metadata

### Required experiment

Compare at least:

```text
Recursive chunking
vs
Heading-aware chunking
```

using the same evaluation dataset.

Do not claim one strategy is better before measuring it.

---

## 15. Query Contract

Primary endpoint:

```http
POST /rag/query
```

Request:

```json
{
  "question": "What is the hotel limit for an L6 employee in Singapore?"
}
```

Response:

```json
{
  "answer": "The applicable hotel limit is ...",
  "strategy": "HYBRID",
  "citations": [
    {
      "document_id": "travel-policy-sg",
      "policy_code": "TRV-0042",
      "version": "2026.1",
      "section": "4.1 Hotel",
      "chunk_id": "..."
    }
  ]
}
```

The response must not expose chain-of-thought.

---

## 16. Query Analysis

Before retrieval, normalize the question into a typed query-analysis object.

Example:

```json
{
  "normalized_query": "hotel limit L6 Singapore",
  "country": "SG",
  "expense_type": "HOTEL",
  "employee_grade": "L6",
  "policy_code": null,
  "has_exact_identifier": false,
  "has_numeric_constraint": false
}
```

### Default implementation

Prefer deterministic extraction for known structured values:

- country codes/names
- employee grades
- policy identifiers
- expense categories
- dates
- currency values

This is free and debuggable.

An LLM-based structured query analyzer may be added later as an experiment, not as the baseline.

---

## 17. Metadata Filtering

Metadata filtering must occur before or during candidate retrieval.

Examples:

```text
country = SG
status = ACTIVE
effective_from <= query_date
employee_grades contains L6
expense_types contains HOTEL
```

The system must prevent an expired policy from outranking the applicable active policy merely because it is semantically similar.

### Policy applicability

When no explicit historical date is requested:

- prefer ACTIVE versions
- require effective_from <= current date
- exclude versions whose effective_to is already past

When a historical date is supplied:

- retrieve the version effective on that date

---

## 18. Dense Retrieval

Dense retrieval uses pgvector.

Purpose:

- semantic similarity
- paraphrases
- conceptually related language

Example:

```text
"airport taxi"
~>
"ground transportation between airport and hotel"
```

Config:

- dense_top_k

Return a typed candidate object containing:

- chunk_id
- content
- metadata
- retrieval_source = DENSE
- raw score
- rank

Do not immediately pass dense results to the LLM.

---

## 19. Lexical Retrieval

Use PostgreSQL Full-Text Search.

Purpose:

- policy identifiers
- exact terminology
- codes
- distinctive tokens
- numeric/structured wording

Examples:

```text
TRV-0042
L6
Singapore
per diem
```

PostgreSQL FTS is the default because it requires no additional search infrastructure.

Do not call PostgreSQL FTS "BM25" unless the implementation actually uses a BM25 ranking algorithm.

Return the same candidate contract used by dense retrieval.

---

## 20. Hybrid Retrieval

Hybrid retrieval executes both:

```text
Dense retrieval
+
Lexical retrieval
```

Candidate sets are merged using Reciprocal Rank Fusion.

### RRF

Use rank rather than incompatible raw score scales.

Conceptually:

```text
RRF(document) = sum(1 / (k + rank))
```

The exact constant must be configurable.

Store:

- dense rank
- lexical rank
- fused score
- fused rank

This makes retrieval debugging possible.

---

## 21. Reranking

Hybrid retrieval should retrieve broadly.

Example:

```text
Dense top 20
+
Lexical top 20
   |
   v
RRF candidates
   |
   v
Top 20–30
   |
   v
Cross-encoder reranker
   |
   v
Top 5
```

Config:

- candidate_top_k
- rerank_top_k

The reranker must receive:

- original user query
- candidate chunk content

It must return:

- rerank score
- rerank rank

---

## 22. Adaptive Retrieval

The retrieval router chooses among:

```text
NO_RETRIEVAL
DENSE
LEXICAL
HYBRID
```

### Example routing rules

#### LEXICAL

Prefer when query contains:

- explicit policy code
- highly specific identifier
- exact quoted phrase

#### DENSE

Prefer for broad semantic questions without strong exact terms.

#### HYBRID

Prefer when a query mixes semantic meaning and structured/exact constraints.

#### NO_RETRIEVAL

Only use for explicitly supported non-knowledge interactions.

For this project, most policy questions should retrieve.

### Router implementation

Start with deterministic rules.

Later compare against an optional structured LLM router.

The router output must include a reason code such as:

```text
EXACT_POLICY_IDENTIFIER
SEMANTIC_POLICY_QUESTION
MIXED_SEMANTIC_AND_EXACT
NO_KNOWLEDGE_REQUIRED
```

Do not store or return hidden chain-of-thought.

---

## 23. Context Deduplication

Hybrid retrieval may return overlapping chunks.

Before reranking or final context assembly, deduplicate by:

1. chunk ID
2. normalized content hash
3. optional near-duplicate similarity threshold

Do not waste context tokens on repeated evidence.

---

## 24. Context Compression

Compression is introduced only after reranking is measured.

Default free implementation should be extractive.

Possible approach:

1. split selected chunks into sentences
2. score sentence relevance against the query
3. preserve section/citation provenance
4. keep the most relevant sentences under a context budget

Optional later experiment:

- LLM-based contextual compression

Compare:

```text
No compression
vs
Extractive compression
```

Measure:

- context tokens
- context precision
- faithfulness
- answer completeness
- latency

---

## 25. Context Builder

The Context Builder receives only final selected evidence.

Responsibilities:

- enforce max context size
- preserve provenance
- order evidence consistently
- clearly separate documents
- attach stable citation IDs

Example:

```text
[SOURCE: C1]
Policy: TRV-0042
Version: 2026.1
Section: 4.1 Hotel
...
```

The Context Builder must not invent citation metadata.

---

## 26. Generation

The generator receives:

- user question
- final context
- citation identifiers

Generation rules:

1. answer from provided evidence
2. do not invent policy facts
3. state when evidence is insufficient
4. reference citation IDs used for claims
5. do not cite chunks that were not supplied

### Abstention

If retrieval confidence/evidence is insufficient, return an explicit insufficient-evidence response.

The system should prefer:

```text
"I could not find enough applicable policy evidence."
```

over guessing.

---

## 27. Citation Validation

Citations are not trusted merely because the model emitted them.

Post-generation validation must ensure:

- cited chunk IDs exist in supplied context
- document metadata comes from the database
- version and section match the chunk
- duplicate citations are normalized

Do not allow arbitrary model-generated source URLs.

---

## 28. API Surface

Keep the API deliberately small.

### Ingest policies

```http
POST /policies/ingest
```

For learning convenience, this may ingest repository policy files rather than implement multipart upload initially.

### List policies

```http
GET /policies
```

Optional filters:

- country
- policy_type
- status

### Query

```http
POST /rag/query
```

### Retrieval debug

Development-only:

```http
POST /rag/retrieve
```

Response should expose retrieval stages:

```json
{
  "analysis": {},
  "strategy": "HYBRID",
  "dense": [],
  "lexical": [],
  "fused": [],
  "reranked": [],
  "final_context": []
}
```

This endpoint is important for learning.

It lets learners inspect why a result was selected without exposing model chain-of-thought.

---

## 29. Error Contract

Use stable API errors.

Example:

```json
{
  "code": "POLICY_NOT_FOUND",
  "message": "Policy was not found"
}
```

Suggested error codes:

- VALIDATION_ERROR
- POLICY_NOT_FOUND
- INGESTION_FAILED
- RETRIEVAL_FAILED
- GENERATION_FAILED
- MODEL_UNAVAILABLE

Do not expose stack traces in normal API responses.

---

## 30. Configuration

Configuration should remain small.

Example environment variables:

```text
DATABASE_URL
EMBEDDING_MODEL
RERANKER_MODEL
GENERATION_PROVIDER
GENERATION_MODEL
OPENAI_API_KEY
OLLAMA_BASE_URL

CHUNK_SIZE
CHUNK_OVERLAP

DENSE_TOP_K
LEXICAL_TOP_K
FUSION_TOP_K
RERANK_TOP_K
MAX_CONTEXT_TOKENS
```

Only variables used by the current implementation phase should exist.

Do not pre-create dozens of future configuration flags.

---

# 31. Implementation Phases

The project evolves in eight phases.

Each phase must retain a measurable baseline from the previous phase.

---

## Phase 1 — Basic RAG Baseline

### Implement

- FastAPI application
- PostgreSQL + pgvector
- policy ingestion
- recursive chunking
- local embeddings
- vector storage
- dense top-k retrieval
- simple context construction
- generation
- citations
- abstention when no evidence is found

### API

- POST /policies/ingest
- GET /policies
- POST /rag/query
- POST /rag/retrieve

### Learning target

Understand the complete RAG path:

```text
documents
-> chunks
-> embeddings
-> retrieval
-> context
-> generation
```

### Baseline metrics

Record:

- Recall@K
- MRR
- retrieval latency
- end-to-end latency
- context size

### Acceptance criteria

A learner can explain where a wrong answer originated:

- ingestion
- chunking
- retrieval
- context
- generation

---

## Phase 2 — Metadata-Aware Retrieval

### Add

- country metadata
- expense type metadata
- employee-grade metadata
- version metadata
- effective-date filtering
- active/expired policy filtering

### Experiment

Compare:

```text
Dense-only
vs
Metadata-filtered dense
```

### Required failure cases

Dataset must include queries where vector-only retrieval selects:

- wrong country
- wrong grade
- expired policy
- wrong expense category

### Acceptance criteria

Metadata filtering must demonstrably reduce cross-policy retrieval errors.

---

## Phase 3 — Lexical Retrieval

### Add

PostgreSQL Full-Text Search.

### Target queries

Include:

- policy codes
- employee-grade identifiers
- exact terminology
- distinctive policy phrases

### Experiment

Compare:

```text
Dense
vs
Lexical
```

### Acceptance criteria

Learner can identify query classes where lexical retrieval outperforms dense retrieval.

---

## Phase 4 — Hybrid Retrieval + RRF

### Add

- dense candidates
- lexical candidates
- Reciprocal Rank Fusion
- deduplication

### Experiment

Compare:

```text
Dense
Lexical
Hybrid + RRF
```

### Acceptance criteria

Hybrid retrieval must show measurable improvement on the mixed evaluation set or the learner must explain why it does not.

No architecture decision is considered successful merely because it was implemented.

---

## Phase 5 — Reranking

### Add

Local cross-encoder reranking.

Recommended flow:

```text
retrieve 20–30
-> rerank
-> keep 5
```

### Experiment

Compare:

```text
Hybrid
vs
Hybrid + Rerank
```

### Measure

- Recall@K
- MRR
- nDCG@K
- context precision
- reranking latency

### Acceptance criteria

Learner can explain the distinction between:

- retrieval recall
- reranking precision

---

## Phase 6 — Adaptive Retrieval

### Add

Query analysis and strategy routing.

Routes:

- DENSE
- LEXICAL
- HYBRID
- NO_RETRIEVAL

### Baseline router

Deterministic rules.

### Optional experiment

Structured LLM router.

### Measure

- route accuracy
- retrieval quality
- unnecessary retrieval rate
- latency per route

### Acceptance criteria

The router must be evaluated against labeled expected routes.

Do not use subjective inspection only.

---

## Phase 7 — Context Compression

### Add

- context deduplication improvements
- extractive query-aware compression
- max-context budget

### Experiment

Compare:

```text
Reranked context
vs
Reranked + compressed context
```

### Measure

- context token count
- context precision
- answer completeness
- faithfulness
- latency

### Acceptance criteria

Compression should reduce context size without materially damaging answer quality.

---

## Phase 8 — Advanced RAG Evaluation and Tuning

### Goal

Compare complete system variants.

Required variants:

```text
A. Dense baseline
B. Metadata + Dense
C. Lexical
D. Hybrid + RRF
E. Hybrid + RRF + Rerank
F. Adaptive + Hybrid + Rerank
G. Adaptive + Hybrid + Rerank + Compression
```

Generate one machine-readable report and one human-readable summary.

The project is not complete until architectural choices are supported by evaluation evidence.

---

## 32. Evaluation Dataset

Create a version-controlled golden dataset.

Recommended initial size:

```text
60–100 questions
```

Each case should contain:

```json
{
  "id": "eval-001",
  "question": "What is the hotel limit for L6 in Singapore?",
  "category": "metadata",
  "expected_route": "HYBRID",
  "expected_document_ids": ["travel-policy-sg"],
  "expected_sections": ["4.1 Hotel"],
  "reference_facts": [
    "..."
  ],
  "should_abstain": false
}
```

### Required categories

- semantic paraphrase
- exact identifier
- exact phrase
- metadata constraint
- country-specific
- grade-specific
- policy version
- historical policy
- numeric limit
- mixed semantic + exact
- ambiguous question
- no-answer / abstention

Do not build an evaluation dataset consisting only of easy semantic questions.

---

## 33. Retrieval Metrics

Default free metrics:

### Hit Rate@K

Was any relevant chunk retrieved?

### Recall@K

How much expected evidence was retrieved?

### Precision@K

How much retrieved evidence was relevant?

### MRR

How early did the first relevant result appear?

### nDCG@K

How good was the ranked ordering when multiple relevance grades exist?

These metrics must be computed without requiring an LLM.

---

## 34. Generation Metrics

Default deterministic checks:

- citation validity
- citation coverage
- expected fact coverage
- abstention correctness

Optional evaluator-model metrics:

- faithfulness
- answer relevancy
- context precision
- context recall

Ragas may be used for these optional metrics.

If an evaluator model incurs cost, evaluation must support a smaller smoke subset.

---

## 35. Operational Metrics

Each evaluation variant should record:

- retrieval latency
- reranking latency
- generation latency
- end-to-end latency
- retrieved candidate count
- final context count
- context token count
- generation input tokens when available
- generation output tokens when available

The objective is not maximum quality at any cost.

Learners must observe quality/latency/cost trade-offs.

---

## 36. Evaluation Report

Produce a comparison table similar to:

```text
Variant                         Recall@5   MRR   nDCG@5   Context Tokens   P95
Dense                           ...
Metadata + Dense                ...
Lexical                         ...
Hybrid + RRF                    ...
Hybrid + RRF + Rerank           ...
Adaptive + Hybrid + Rerank      ...
+ Compression                   ...
```

The report should also identify failure categories.

Example:

```text
Dense-only failures:
- exact policy code
- historical version selection

Hybrid failures:
- ambiguous country
- overlapping version metadata
```

This turns evaluation into diagnosis rather than score collection.

---

## 37. Retrieval Debuggability

Every retrieved candidate should carry stage information.

Suggested internal contract:

```text
chunk_id
content
metadata

dense_score
dense_rank

lexical_score
lexical_rank

rrf_score
rrf_rank

rerank_score
rerank_rank
```

Fields may be null when the candidate did not participate in a stage.

This information should be available through the development retrieval endpoint and evaluation reports.

It should not be included in normal end-user answers.

---

## 38. Testing Strategy

### Unit tests

Cover:

- metadata parsing
- policy applicability
- query normalization
- router decisions
- RRF calculation
- deduplication
- context budgeting
- citation validation

### PostgreSQL integration tests

Cover:

- pgvector retrieval
- metadata filtering
- PostgreSQL FTS
- combined filters
- policy version queries
- ingestion idempotency

Do not replace PostgreSQL with SQLite for these tests.

### API tests

Cover:

- ingestion
- policy list
- query
- retrieval debug
- validation errors
- stable error contracts

### Evaluation regression

Maintain a small fast subset that can run during development.

Full evaluation may run manually because local models can be slow.

---

## 39. Learner Experience

The intended setup should be approximately:

```bash
docker compose up -d
uv sync
uv run alembic upgrade head
uv run python scripts/ingest.py
uv run uvicorn app.main:app --reload
```

Then learners should be able to run:

```bash
uv run python evals/run.py
```

No Postman configuration should be required for mandatory validation.

A future implementation should provide executable Python scenario scripts for:

- ingestion
- baseline query
- retrieval comparison
- full evaluation

---

## 40. Free-First Rules

The following choices are deliberate:

### PostgreSQL FTS instead of OpenSearch

Reason:

- no extra service
- no JVM
- one database
- enough to learn lexical/hybrid retrieval

### pgvector instead of dedicated vector database

Reason:

- one database
- free
- local
- metadata filtering and vectors together
- no vendor account

### Local embeddings

Reason:

- no embedding API bill
- reproducible evaluations

### Local reranker

Reason:

- no reranking API bill
- suitable for learning reranking mechanics

### Deterministic router first

Reason:

- no extra LLM call
- easy to inspect
- easy to label and evaluate

### Extractive compression first

Reason:

- no additional generation call
- isolates the context-compression concept

---

## 41. Replaceability Rules

Learning-first does not mean hard-coding every local choice.

The following should have explicit interfaces:

- embedding provider
- dense retriever
- lexical retriever
- reranker
- compressor
- generation model

This allows later experiments with:

- OpenSearch
- Pinecone
- managed reranking APIs
- different embedding models
- cloud LLM providers

without rewriting orchestration.

Do not introduce generic abstraction layers beyond these actual substitution points.

---

## 42. Anti-Patterns

The implementation must avoid:

### Vector-only dogma

Do not assume embeddings solve exact lookup.

### Retrieval after generation

Evidence must be selected before answer generation.

### Unfiltered policy versions

Do not retrieve obsolete policies as if they are current.

### Score mixing

Do not directly add cosine similarity and lexical scores from incompatible scales.

Use rank fusion.

### Huge Top-K context dumping

Retrieving more chunks is not automatically better.

### LLM-generated citations

Citation metadata comes from retrieved records.

### Framework-hidden retrieval

Learners should be able to inspect each stage.

### Evaluation by eyeballing

Every advanced technique must be measured.

### Premature agents

A retrieval router is not an agent.

Do not introduce agent loops to make the project sound more advanced.

---

## 43. Phase Completion Gate

A phase is complete only when all of the following are true:

1. the feature works
2. tests cover it
3. the evaluation dataset contains cases that exercise it
4. metrics are recorded
5. comparison against the previous baseline exists
6. a learner can inspect intermediate retrieval outputs
7. the new technique has a documented reason for existing

If a new technique does not improve the relevant metric, keep the result.

A negative result is a valid learning outcome.

Do not manipulate the benchmark to justify the architecture.

---

## 44. Final Definition of Done

Reim Bot is complete when the learner can run the system locally and demonstrate, with evidence, the progression:

```text
Dense RAG
   |
   v
Metadata-aware Dense
   |
   +--> Lexical Retrieval
   |        |
   +--------+
       |
       v
   Hybrid + RRF
       |
       v
    Reranking
       |
       v
Adaptive Retrieval
       |
       v
Context Compression
       |
       v
Measured Advanced RAG
```

The final output is not merely a chatbot that answers policy questions.

The final output is a retrieval system whose behavior can be inspected, compared, diagnosed, and justified with metrics.
