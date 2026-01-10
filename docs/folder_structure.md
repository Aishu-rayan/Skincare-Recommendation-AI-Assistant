# High-level Folder Structure (proposed)

This is a proposed repo layout for v1 (no code yet).

```text
Skincare-Recommendation-AI-Assistant/
  data/
    sephora_select_reviews.db            # source+derived DB (backup handled manually)

  docs/
    technical_requirements.md
    tech_stack.md
    folder_structure.md
    api_contracts.md
    db_schema.md
    diagrams/
      architecture_overview.png
      data_model.png

  src/
    backend/
      skincare_ai/
        api/                             # FastAPI routes
        core/                            # config/constants
        db/                              # sqlite connections, queries
        llm/                             # prompt templates + clients
        pipelines/                       # offline extraction/aggregation/embedding
        retrieval/                       # vector search + rerank
        evidence/                        # snippet retrieval
        schemas/                         # Pydantic models
      main.py                            # FastAPI entrypoint

    frontend/
      package.json
      vite.config.*
      src/
        pages/
        components/
        api/                             # typed client for FastAPI endpoints
        types/                           # shared UI types

    shared/
      types/                             # optional shared JSON schemas (kept in sync manually)

  scripts/
    offline_build/                       # one-shot runners (extract/tfidf/embed)

  tests/
    backend/
    frontend/
```
