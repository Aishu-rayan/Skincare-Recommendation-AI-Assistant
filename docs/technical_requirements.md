# Technical Requirements (v1)

![Architecture overview](./diagrams/architecture_overview.png)

## 1. Purpose
Build a local (offline MVP) skincare recommendation web app that:
- Derives product effects from Sephora review text (not marketing claims).
- Recommends products by category and generates AM/PM routines.
- Explains recommendations with review-grounded evidence, phrased as **“users report…”**.

## 2. Scope (v1)
### 2.1 In scope
- Product categories: **cleanser, moisturizer, sunscreen, toner, treatment**.
- Offline, one-time pipeline to enrich the existing SQLite DB (`sephora_select_reviews.db`).
- Runtime app: **FastAPI backend** + **React/Vite frontend**.
- User profiling via Q&A (no face photo).
- Similarity retrieval using SQLite + `sqlite-vector` (via `sqliteai-vector`).
- LLM-based routine generation via remote OpenAI model (configurable).

### 2.2 Out of scope
- Face photo analysis.
- Auth, accounts, persistent profiles.
- Ingredient conflict checking.
- Continuous ingestion of new reviews.

## 3. Users and key flows
### 3.1 Primary user stories
1. As a user, I answer a short questionnaire about skin type, concerns, and goals.
2. I ask for a product type (e.g., “Recommend a sunscreen for oily skin with redness”).
3. I receive a ranked product list with concise “users report…” reasoning.
4. I request a routine; I receive AM/PM steps with products + short guidance.
5. I ask “why” for a product; I see evidence snippets from reviews.

### 3.2 Data flow summary
**Offline build** → enrich DB with extracted effects + TF–IDF + embeddings.
**Online runtime** → user query → vector retrieval → TF–IDF/segment rerank → routine LLM → response.

## 4. Functional requirements
### 4.1 Offline pipeline
#### FR-PIPE-1: Token/cost/time dry-run gate
- Run extraction on a representative sample (e.g., 1k–5k reviews) and record:
  - avg input tokens, avg output tokens
  - throughput (reviews/min)
  - estimated total cost for full-pass and for a sampling policy
- Decision output: choose extraction model strategy (remote low-cost vs local Ollama) and sampling policy.

#### FR-PIPE-2: Review extraction
- Input: `select_customer_reviews.review_text` + metadata (`rating`, `is_recommended`, `skin_type`, categories).
- Output: `review_effects` rows with:
  - `positive_effects`, `negative_effects`, `side_effects`
  - `skin_type_signals`, `conditions_helped`
  - optional `satisfaction_score`
- Must be idempotent by (`extraction_version`, `review_rowid`).

#### FR-PIPE-3: Vocabulary (simplified)
- Start from a seed list of canonical labels.
- One LLM-assisted normalization pass maps raw phrases → canonical labels.
- Vocabulary frozen for v1.

#### FR-PIPE-4: TF–IDF profiles
- Compute global per-product TF–IDF in `product_effect_profile`.
- TF: `log(c_pe + 1)` (optionally weighted by `satisfaction_score`).
- IDF: `log(N/(df+1))`.

#### FR-PIPE-5: Effect summary generation
- Deterministically generate `effect_summary` per product from top TF–IDF items.
- Must use “users report…” phrasing in summaries.

#### FR-PIPE-6: Embeddings + sqlite-vector storage
- Compute one embedding per product from `effect_summary`.
- Store in `product_vectors.embedding` as `BLOB`, along with `embedding_model` and `embedding_dim`.
- Enable vector search via `sqlite-vector` initialization.

### 4.2 Runtime backend (FastAPI)
#### FR-API-1: Profile capture and update
- Maintain `user_information` in memory for the session.

#### FR-API-2: Recommend products
- Input: user profile + requested product type.
- Steps:
  1) metadata filter by category
  2) vector KNN search over `product_vectors.embedding`
  3) rerank using TF–IDF match + segment match multiplier
- Output: ranked products with:
  - top matched benefits/negatives
  - short “users report…” explanation

#### FR-API-3: Routine generation
- Input: user profile + shortlisted candidates per slot.
- Output: JSON routine with:
  - AM routine steps and PM routine steps
  - chosen product per slot
  - brief usage notes

#### FR-API-4: Evidence explanations
- Given a product + effect, return a small set of review snippets.
- Must support fast lookup via indexes; optional FTS5 for text snippet retrieval.

### 4.3 Runtime frontend (React/Vite)
- Questionnaire UI to collect:
  - skin type (dry/normal/oily), sensitive yes/no
  - goals + conditions
  - age (optional)
  - additional concerns (free text)
- Recommendation UI:
  - select product type
  - show ranked results + expand for “why”
- Routine UI:
  - generate and display AM/PM routine

## 5. Non-functional requirements
### 5.1 Offline MVP constraints
- Works on a single laptop; 1–2 concurrent users.
- Reproducible: every derived artifact tagged with version fields.

### 5.2 Safety/claims
- Responses must be phrased as observational: **“users report…”**.
- Avoid diagnosis/treatment language.

### 5.3 Configurability
- Routine LLM model must be configurable.
- Extraction model strategy must be configurable and decided after dry-run.
- Embedding model + dimension must be configurable.

## 6. Acceptance criteria (v1)
- Offline pipeline can enrich the DB and produce `product_vectors` with embeddings.
- Product recommendation returns consistent results for common scenarios (manual spot-check).
- Routine endpoint returns valid JSON with AM/PM routines.
- “Why this product” returns evidence snippets sourced from review text.
- App runs locally with separate frontend and backend dev servers.

## 7. Known constraints / caveats
- `sqlite-vector` is licensed under ELv2; treat this as an MVP/research constraint.
