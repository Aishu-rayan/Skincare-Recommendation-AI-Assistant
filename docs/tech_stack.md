# Tech Stack (v1)

## Backend
- **Language/runtime:** Python 3.14
- **Package manager/runner:** `uv`
- **API framework:** FastAPI
- **Server:** Uvicorn
- **Data validation:** Pydantic v2
- **Database:** SQLite (`sephora_select_reviews.db`), updated in-place
- **Vector search:** `sqlite-vector` extension via the **`sqliteai-vector`** Python package
- **Logging:** Python `logging` (structured JSON format recommended)

## Frontend
- **Framework:** React
- **Build tooling:** Vite
- **Language:** TypeScript
- **Dev:** frontend and backend run as separate servers (two commands / two terminals)

## LLM + Embeddings
### Routine generation (remote)
- **Provider:** OpenAI
- **Model:** configurable (e.g., `gpt-4o-mini` or `gpt-4o`)

### Review extraction (decision via dry-run)
Extraction is token-heavy, so the pipeline must start with a dry-run estimator and then choose:
- **Remote low-cost model** (configurable; e.g., GPT-3.5 / OpenAI o3 depending on cost/quality), or
- **Local model** via Ollama (download and run on macOS) if runtime is acceptable.

### Embeddings (default: local)
- **Default approach:** local embeddings with a sentence-transformer family model.
- **Default model choice:** `intfloat/e5-small-v2` (384-dim) as a sensible baseline.
- **Config:** store `embedding_model` + `embedding_dim` with vectors to allow swaps.

## SQLite + sqlite-vector operational notes
- Enable extension loading in SQLite connections.
- Store vectors in a `BLOB` column, plus `embedding_dim`.
- Initialize/search using sqlite-vector functions (documented in `docs/db_schema.md`).
