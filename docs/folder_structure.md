# Folder Structure

## Current layout

```text
Skincare-Recommendation-AI-Assistant/
  data/
    sephora_select_reviews.db            # source+derived DB (backup handled manually)

  docs/
    skincare_ai_design_plan.md           # main design document
    technical_requirements.md
    tech_stack.md
    folder_structure.md
    api_contracts.md
    db_schema.md
    database_findings.md
    database_cleanup_report.md
    README_IMPLEMENTATION.md
    prompts/                             # AI prompt instructions used during development
      data_analysis_instruction.md
      data_cleanup_instruction.md
    diagrams/
      architecture_overview.png
      data_model.png
      docs_architecture_overview.png

  planning/                              # early brainstorming, course context, project proposal material
    Initial_Plan.txt
    initial_prompt1.txt
    assignment_goals.md
    course_description.txt
```

## Proposed layout (when code is added)

```text
Skincare-Recommendation-AI-Assistant/
  data/
    sephora_select_reviews.db

  docs/
    ...                                  # (same as current)

  planning/
    ...                                  # (same as current)

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
      types/                             # optional shared JSON schemas

  scripts/
    offline_build/                       # one-shot runners (extract/tfidf/embed)

  tests/
    backend/
    frontend/
```
