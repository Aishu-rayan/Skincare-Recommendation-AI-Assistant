## Skincare AI Assistant Design Using Sephora Reviews (Inspired by Glowe)

This document describes the end-to-end design for a skincare AI assistant built on the `sephora_select_reviews.db` SQLite database (≈239k reviews, ≈8.5k products), inspired by the Glowe app.

---

## 1. Data Understanding & Target Entities

### 1.1 Existing tables

- **`product_info`** (~8,494 rows)
  - Key columns: `product_id`, `product_name`, `brand_id`, `brand_name`, `ingredients`, `price_usd`, `rating`, `reviews`, `highlights`, `primary_category`, `secondary_category`, `tertiary_category`, etc.
- **`select_customer_reviews`** (~238,929 rows)
  - Key columns: `primary_category`, `secondary_category`, `tertiary_category`, `product_id`, `product_name`, `brand_name`, `review_title`, `review_text`, `rating`, `is_recommended`, `skin_type`, `skin_tone`, `eye_color`, `ingredients`, `price_usd`, `highlights`.

We join reviews to products via `product_id` and focus primarily on skincare: rows where `primary_category = 'Skincare'` and `secondary_category` in `{Moisturizers, Sunscreens, Cleansers, Treatments, Toners}`.

### 1.2 Target conceptual entities

Logical entities used across the system:

- **Product** – metadata per SKU from `product_info`.
- **Raw Review** – original user review row from `select_customer_reviews`.
- **Structured Review Effects** – structured fields extracted from `review_text` by an LLM.
- **Per-product Effect Profile** – TF–IDF style weights of effects per product.
- **Effect Vocabulary** – canonical list of effect labels (e.g. `hydration`, `reduced_redness`).
- **Embeddings** – vectors stored in SQLite (via `sqlite-vector`) for similarity search.
- **User Profile** – structured dictionary like `user_information` (goals, conditions, skin type score, sensitivities, age, additional concerns).

---

## 2. Offline Review-Extraction Pipeline

Goal: convert free-text reviews into structured signals that describe what each product actually does, for which skin profiles, and with what experience.

### 2.1 New table: `review_effects`

Create a table in SQLite to store per-review extracted information:

```sql
CREATE TABLE review_effects (
    review_effect_id   INTEGER PRIMARY KEY,
    review_rowid       INTEGER NOT NULL,  -- rowid / PK from select_customer_reviews
    product_id         TEXT NOT NULL,

    -- core extracted info (JSON arrays)
    positive_effects   TEXT,              -- JSON list[str]
    negative_effects   TEXT,              -- JSON list[str]
    side_effects       TEXT,              -- JSON list[str]

    -- reviewer context signals (JSON arrays)
    skin_type_signals  TEXT,              -- JSON list[str] ("dry", "normal", "combination", "oily", "sensitive")
    conditions_helped  TEXT,              -- JSON list[str]

    -- optional weighting
    satisfaction_score REAL,              -- 0-1

    -- meta
    extraction_version TEXT NOT NULL,
    extracted_at       TEXT NOT NULL
);

CREATE INDEX idx_review_effects_product_id ON review_effects(product_id);
CREATE INDEX idx_review_effects_review_rowid ON review_effects(review_rowid);
```

**Fields to extract per review (v1):**

- `positive_effects`: benefits experienced (e.g. `hydration`, `reduced_redness`).
- `negative_effects`: undesirable outcomes (e.g. `caused_breakouts`, `increased_redness`).
- `side_effects`: tolerability issues (e.g. `stinging`, `burning`).
- `skin_type_signals`: normalized to `{dry, normal, combination, oily, sensitive}`.
- `conditions_helped`: normalized to a fixed condition vocabulary.
- `satisfaction_score`: optional numeric signal derived from rating / recommendation + text sentiment.

### 2.2 LLM extraction process

For each `select_customer_reviews` row:

1. Fetch full `review_text`, with `rating`, `is_recommended`, `skin_type`, and categories as context.
2. Call an LLM with a strict JSON schema and a fixed label set for conditions and skin types.
3. Validate the LLM response (types, required keys) and normalize strings (lowercase, canonical labels) before writing to `review_effects`.
4. Log any parsing failures in a small `extraction_errors` table for later re-processing.

The pipeline runs offline in batches (e.g. 1,000 reviews per batch), is restartable/idempotent using `extraction_version` and `review_rowid` ranges, and is intended as a one-time MVP build step (no continuous ingestion).

### 2.3 Cost/time gating for full vs sampled extraction

Before running extraction across all reviews:

1. Run a small pilot on a representative sample (e.g. 1,000–5,000 reviews).
2. Record average input tokens, average output tokens, and wall-clock throughput.
3. Estimate total cost and runtime for:
   - full-pass processing, and
   - stratified sampling per product (e.g. N reviews/product split across ratings and skin types).
4. Choose model (local vs remote) and review volume based on the budget cap.

---

## 3. Effect Vocabulary & TF–IDF Profiling

After `review_effects` is populated, build a canonical effect vocabulary and a per-product effect profile.

### 3.1 New table: `effect_vocabulary`

```sql
CREATE TABLE effect_vocabulary (
    effect_id     INTEGER PRIMARY KEY,
    effect_name   TEXT UNIQUE NOT NULL,  -- e.g. "hydration", "reduced_redness"
    effect_type   TEXT NOT NULL,         -- {"benefit", "negative", "side_effect"}
    description   TEXT,
    is_condition  INTEGER NOT NULL DEFAULT 0
);
```

### 3.2 Simpler vocabulary creation (v1)

1. Start with a compact seed list of canonical effects (e.g. 50–150 total across benefit/negative/side-effect).
2. Collect raw phrases from a sample of extracted outputs.
3. Use one LLM-assisted normalization pass to map raw phrases → closest canonical label (and optionally propose a small number of new labels).
4. Manually approve new labels once, then freeze the vocabulary for v1.

This avoids embedding clustering while still producing stable, searchable labels.

### 3.3 New table: `product_effect_profile` (TF–IDF)

For each `(product_id, effect_name)` pair, compute TF–IDF style weights.

```sql
CREATE TABLE product_effect_profile (
    product_id    TEXT NOT NULL,
    effect_name   TEXT NOT NULL,
    tf            REAL NOT NULL,
    idf           REAL NOT NULL,
    tfidf_weight  REAL NOT NULL,
    effect_type   TEXT NOT NULL,
    PRIMARY KEY (product_id, effect_name)
);
```

#### 3.3.1 TF simplification

Use a single TF form for v1:

```text
TF_{p,e} = log(c_pe + 1)
```

where `c_pe` is the number of reviews for product `p` that mention effect `e`. If `satisfaction_score` is present, treat each mention as a fractional count (e.g. add `satisfaction_score` instead of `1`).

#### 3.3.2 IDF

```text
IDF_e = log(N / (df_e + 1))
```

Adding `+1` in the denominator avoids edge cases when `df_e` is very small.

### 3.4 Simple segment weighting (v1)

Compute TF–IDF once globally, then apply a simple re-ranking multiplier at query time based on segment match:

- If user is oily/dry/sensitive, boost products where a larger share of extracted reviews contain the matching `skin_type_signals`.

This can be stored as per-product segment counts and used without maintaining separate segment-specific TF–IDF tables.

---

## 4. Single-Embedding Strategy (v1)

Use one embedding per product derived from aggregated review effects.

### 4.1 Product effect summary text

For each product, generate a compact “effect summary” string from `product_effect_profile` (and supporting counts), for example:

> "Moisturizer. Users report: hydration, reduced redness, soothed skin. Common negatives: heavy feel, caused breakouts. Best for: dry/sensitive."

This text is embedded using a sentence embedding model and stored per product.

### 4.2 Storage and search in SQLite via `sqlite-vector`

Store embeddings as `BLOB` vectors inside SQLite and query using the `sqlite-vector` extension (https://github.com/sqliteai/sqlite-vector).

Implementation approach:

- Table `product_vectors(product_id TEXT PRIMARY KEY, effect_summary TEXT, embedding BLOB, embedding_dim INTEGER, embedding_model TEXT, created_at TEXT)`
- Initialize vector search for that table/column via `vector_init(...)`
- Query nearest neighbors via the extension’s scan function and join back to product metadata.

---

## 5. Retrieval & Recommendation Logic

Given a `user_information` dict and a requested product type (e.g. "moisturizer"), embed a short query string and retrieve similar products.

### 5.1 Build a user query string

Generate a short, consistent text query from user goals/conditions/skin type, for example:

> "moisturizer for oily skin; concerns: acne, redness; wants: hydrate; avoid irritation"

Embed this query using the same embedding model used for product effect summaries.

### 5.2 Candidate filtering

Before vector search, filter products by metadata:

- `primary_category = 'Skincare'`.
- `secondary_category` / `tertiary_category` matching requested type.
- Optional constraints: price range, specific brands.

### 5.3 Vector similarity search

Run K-NN search over `product_vectors.embedding` and return top-N products.

### 5.4 Re-ranking with TF–IDF + segment match

After retrieving candidates, re-rank using a transparent scoring layer:

- Add bonuses when top TF–IDF benefits match the user’s goals/conditions.
- Subtract penalties when top TF–IDF negatives/side effects conflict with the user’s concerns.
- Apply a simple segment match multiplier (oily/dry/sensitive) using per-product segment counts.

Top-K results from this step are fed into the routine builder.

---

## 6. Routine Generation (Morning / Evening)

The routine builder uses an LLM to assemble structured routines (morning and evening) from shortlisted products.

### 6.1 Category-to-slot mapping

Define canonical routine slots:

- Morning: `cleanser`, `treatment/serum`, `moisturizer`, `sunscreen`.
- Evening: `cleanser`, `treatment/serum`, `moisturizer`.
- Optional: `toner` when relevant.

### 6.2 Inputs to the routine builder LLM

For each routine generation request provide:

- The full `user_information` structure.
- For each slot, a small set (e.g. 5) of top-scoring candidate products, including:
  - `product_id`, `product_name`, `brand_name`, `price_usd`.
  - Top positive and negative effects from `product_effect_profile` (e.g. 5 highest `tfidf_weight` per type).
  - A few evidence snippets (Section 7.4).

The prompt instructs the LLM to:

1. Choose at most one product per slot for morning and evening.
2. Respect user conditions and constraints.
3. Output both:
   - Structured JSON describing the routine steps.
   - Natural language explanations and usage instructions.

---

## 7. Agentic Chat Interface

The chatbot front-end orchestrates user profiling, product search, routine generation, and explanation.

### 7.1 Core flows

- **Onboarding Q&A:** ask a short sequence of questions to populate `user_information` (goals, conditions, skin type score, sensitivities, age, budget, product-type request).
- **Product-type recommendation:** user asks, e.g., "Recommend a moisturizer for my oily, acne-prone skin" → update profile, run retrieval for moisturizers, show ranked list.
- **Routine creation:** once enough signals are collected, call the routine builder LLM to generate morning and evening routines.
- **Evidence-based explanations:** when user asks "Why this product for redness?", fetch representative reviews from `review_effects` where `reduced_redness` appears and show short snippets.
- **Response stance:** keep recommendations direct, but phrase claims as "users report…" and avoid diagnosis/treatment language.

### 7.2 Internal tools for the agent

Expose a small set of tools/functions that the agent can call:

1. `update_user_profile(fields)` – modify `user_information` from chat answers.
2. `search_products_by_effect(user_profile, product_type)` – implements Sections 5.1–5.4 and returns ranked products.
3. `build_routine(user_profile)` – orchestrates calling the routine builder LLM.
4. `get_effect_evidence(product_id, effect_name)` – retrieves a few `review_text` snippets where the effect is present.

The agent chain decides which tool to call based on user input and conversation context.

### 7.3 Persistence

- No login/auth in v1.
- `user_information` is maintained in-memory for the current session only.

### 7.4 Evidence retrieval (reasonably simple indexing)

To support “why this product” explanations:

- Index `review_effects(product_id)` (already included above).
- Optionally add an SQLite FTS5 table over `select_customer_reviews.review_text` (keyed by `rowid`) to retrieve short snippets quickly.

---

## 8. Implementation Notes and Open Decisions

### 8.1 Models and infrastructure

- **LLM for extraction and routines:** start with small sampling runs to measure token usage/cost and throughput, then choose model (local vs remote) and review volume to stay within the budget cap.
- **Embedding model:** use one consistent sentence embedding model for:
  - Product effect summary text
  - User query strings
- **Vector DB:** SQLite + `sqlite-vector` via the `sqliteai-vector` Python package, storing vectors as `BLOB`s inside SQLite and performing vector search through the extension.
- **Deployment:** offline-only MVP (local execution; no continuous/online review processing).

### 8.2 Evaluation strategy

- Manually inspect a sample of products and their top effects against real reviews to validate extraction.
- Construct simple offline test cases (e.g. oily skin with redness) and check if top results from effect-based retrieval make sense to a domain expert.
- Compare single-embedding vs dual-embedding performance on these cases to confirm the benefit of the effect embedding.
