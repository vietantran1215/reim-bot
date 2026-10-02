# Reim Bot

Learning-first FastAPI backend for Advanced RAG over enterprise expense reimbursement policies.

The implementation specification is in:

- `specs/advanced-rag-backend.md`

Design priorities:

1. Free or local-first technology choices.
2. Minimal setup and infrastructure.
3. One FastAPI backend, not microservices.
4. Retrieval engineering is explicit and measurable.
5. Each advanced RAG technique is introduced only after a baseline exists to compare against.


## Phase specifications

The cumulative, reproducible implementation sequence is defined in:

- `specs/phases/00-phase-plan.md`
- `specs/phases/01-basic-rag.md`
- `specs/phases/02-metadata-aware-rag.md`
- `specs/phases/03-lexical-rag.md`
- `specs/phases/04-hybrid-rag.md`
- `specs/phases/05-reranking.md`
- `specs/phases/06-adaptive-rag.md`
- `specs/phases/07-context-compression.md`
- `specs/phases/08-evaluation-tuning.md`

Each later phase is designed to branch from the completed previous phase rather than independently from `main`.
