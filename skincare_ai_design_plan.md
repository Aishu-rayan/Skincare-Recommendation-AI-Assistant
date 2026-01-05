## Skincare AI Assistant Design Using Sephora Reviews (Inspired by Glowe)

This document describes the end-to-end design for a skincare AI assistant built on the `sephora_select_reviews.db` SQLite database (≈239k reviews, ≈8.5k products), inspired by the Glowe app.

---

## 1. Data Understanding & Target Entities

### 1.1 Existing tables

- **`product_info`** (~8,494 rows)
  - Key columns: `product_id`, `product_name`, `brand_id`, `brand_name`, `ingredients`, `price_usd`, `rating`, `reviews`, `highlights`, `primary_category`, `secondary_category`, `tertiary_category`, etc.
- **`select_customer_reviews`** (~238,929 rows)
  - Key columns: `primary_category`, `secondary_category`, `tertiary_category`, `product_id`, `product_name`, `brand_name`, `review_title`, `review_text`, `rating`, `is_recommended`, `skin_type`, `skin_tone`, `eye_color`, `ingredients`, `price_usd`, `highlights`.

We join reviews to products via `product_id` and focus primarily on skincare: rows where `primary_category = 'Skincare'` and `secondary_category` in `{Moisturizers, Sunscreens, Cleansers, Treatments, Toners, Masks, Eye Creams}`.

### 1.2 Target conceptual entities

Logical entities used across the system:

- **Product** – metadata per SKU from `product_info`.
- **Raw Review** – original user review row from `select_customer_reviews`.
- **Structured Review Effects** – structured fields extracted from `review_text` by an LLM.
- **Per-product Effect Profile** – TF–IDF style weights of effects per product.
- **Effect Vocabulary** – canonical list of effect labels (e.g. `hydration`, `reduced_redness`).
- **Embeddings** – vectors for products and effects stored in a vector database.
- **User Profile** – structured dictionary like `user_information` (goals, conditions, skin type score, sensitivities, age, additional concerns).

---

## 2. Offline Review-Extraction Pipeline

Goal: convert free-text reviews into structured signals that describe what each product actually does, for which skin profiles, and with what experience.

### 2.1 New table: `review_effects`

Create a table (initially in SQLite, or a downstream warehouse) to store per-review extracted information:

```sql
CREATE TABLE review_effects (
    review_effect_id     INTEGER PRIMARY KEY,
    review_rowid         INTEGER NOT NULL,  -- rowid / PK from select_customer_reviews
    product_id           TEXT NOT NULL,

    -- core extracted info
    positive_effects     TEXT NOT NULL,     -- JSON list[str]
    negative_effects     TEXT NOT NULL,     -- JSON list[str]
    side_effects         TEXT NOT NULL,     -- JSON list[str]
    usage_experience     TEXT NOT NULL,     -- JSON list[str] (e.g. "absorbs quickly", "pills under makeup")

    -- user & context signals
    skin_type_signals    TEXT NOT NULL,     -- JSON list[str] ("dry", "normal", "combination", "oily", "sensitive")
    conditions_helped    TEXT NOT NULL,     -- JSON list[str] ("acne", "redness", "pigmentation", "wrinkles", etc.)
    conditions_worsened  TEXT NOT NULL,     -- JSON list[str]
    age_group            TEXT NOT NULL,     -- e.g. "<20", "20-29", "30-39", "40+"
    time_to_see_results  TEXT NOT NULL,     -- JSON list[str], e.g. ["1 week", "1 month"]

    -- behavioral / satisfaction
    overall_sentiment    REAL NOT NULL,     -- [-1, 1]
    satisfaction_score   REAL NOT NULL,     -- 0-1
    repurchase_intent    REAL NOT NULL,     -- 0-1 probability
    recommend_to_friend  REAL NOT NULL,     -- 0-1

    -- meta
    extraction_version   TEXT NOT NULL,
    extracted_at         TEXT NOT NULL
);
```

**Fields to extract per review:**

- `positive_effects`: benefits experienced (e.g. `hydration`, `reduced_redness`, `soothed_skin`, `brightening`, `reduced_pores`).
- `negative_effects`: undesirable outcomes (e.g. `caused_breakouts`, `increased_redness`, `left_white_cast`).
- `side_effects`: tolerability issues (e.g. `eye_irritation`, `stinging`, `burning`).
- `usage_experience`: UX descriptors (e.g. `lightweight`, `heavy_greasy`, `strong_fragrance`, `no_white_cast`, `pills_under_makeup`).
- `skin_type_signals`: reviewer skin info mapped to a small fixed set `{dry, normal, combination, oily, sensitive}`.
- `conditions_helped` / `conditions_worsened`: from a fixed vocab matching `user_information.conditions` keys plus a few extras (`dark_circles`, `rosacea`, etc.).
- `age_group`: bucketed from any age clues in the text (or left generic if unknown).
- `time_to_see_results`: approximate time horizon for effects.
- `overall_sentiment`: continuous sentiment score.
- `satisfaction_score`, `repurchase_intent`, `recommend_to_friend`: behavior and satisfaction indicators.

### 2.2 LLM extraction process

For each `select_customer_reviews` row:

1. Fetch full `review_text`, with `rating`, `is_recommended`, `skin_type`, and categories as context.
2. Call an LLM (e.g. Claude, GPT-4.x) with a strict JSON schema and a fixed label set for conditions and skin types.
3. Validate the LLM response (types, required keys) and normalize strings (lowercase, canonical labels) before writing to `review_effects`.
4. Log any parsing failures in a small `extraction_errors` table for later re-processing.

The pipeline runs in batches (e.g. 1,000 reviews per batch) and can be restarted idempotently using `extraction_version` and `review_rowid` ranges.

### 2.3 Handling noisy or short reviews

- If `review_text` is short or vague, allow the model to output empty effect lists and a neutral sentiment.
- Down-weight these later by scaling counts with `satisfaction_score` and review length.

---

## 3. Effect Vocabulary & TF–IDF Profiling

After `review_effects` is populated, we build a canonical effect vocabulary and a per-product effect profile similar to Glowe.

### 3.1 New table: `effect_vocabulary`

```sql
CREATE TABLE effect_vocabulary (
    effect_id     INTEGER PRIMARY KEY,
    effect_name   TEXT UNIQUE NOT NULL,  -- e.g. "hydration", "reduced_redness"
    effect_type   TEXT NOT NULL,         -- {"benefit", "negative", "side_effect", "experience"}
    description   TEXT,
    is_condition  INTEGER NOT NULL DEFAULT 0  -- 1 if maps directly to a condition key
);
```

### 3.2 Building the vocabulary

1. Aggregate all raw phrases from `positive_effects`, `negative_effects`, `side_effects`, and `usage_experience`.
2. Normalize via heuristics: lowercase, trim, singularize, remove stopwords.
3. Optionally cluster phrases using embeddings and merge semantically similar ones into 200–300 canonical `effect_name` labels.
4. Manually review cluster labels once to ensure domain sense (e.g. `"hydrated skin"`, `"very moisturizing"` → `hydration`).
5. Store final labels and metadata in `effect_vocabulary` and update `review_effects` so all effect lists reference canonical names.

Examples of canonical effects:

- Benefits: `hydration`, `reduced_redness`, `oil_control`, `radiance`, `evened_pigmentation`, `reduced_fine_lines`, `calmed_sensitivity`.
- Negatives: `caused_breakouts`, `increased_redness`, `drying`, `caused_flaking`.
- Side effects: `eye_irritation`, `stinging`, `burning`, `headache_from_fragrance`.
- Experience: `lightweight`, `heavy_greasy`, `strong_fragrance`, `no_white_cast`, `pills_under_makeup`.

### 3.3 New table: `product_effect_profile`

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

#### 3.3.1 Term Frequency (TF)

For product `p` and effect `e`:

- Let `c_pe` be the sum over all reviews for product `p` of mentions of effect `e`, optionally weighted by `satisfaction_score` and review length.
- Define:

```text
TF_{p,e} = log(c_pe + 1)
```

#### 3.3.2 Inverse Document Frequency (IDF)

- Let `N` be total number of products.
- Let `df_e` be the number of distinct products with `c_pe > 0`.

```text
IDF_e = log(N / df_e)
```

#### 3.3.3 Normalized TF–IDF weight

For each product `p`:

```text
w_{p,e} = (TF_{p,e} * IDF_e) / sum_j (TF_{p,j} * IDF_j)
```

Store `w_{p,e}` as `tfidf_weight`. Weights per product sum to ~1 and represent how strongly the product is associated with each effect versus others.

### 3.4 Condition- and skin-type-specific profiles (optional)

To tailor recommendations more precisely, compute alternate profiles restricted to certain reviewer segments, e.g.:

- `product_effect_profile_oily` – only reviews where `skin_type_signals` includes `oily`.
- `product_effect_profile_sensitive` – only reviews with `sensitive`.

These use the same TF–IDF logic but on filtered subsets of `review_effects`.

---

## 4. Dual-Embedding Strategy

We use two embeddings per product, as in Glowe: a **product embedding** and an **effect embedding**.

### 4.1 Product embedding

**Purpose:** general product search, brand/category navigation, and description-level similarity.

**Input text per product:**

- `product_name`, `brand_name`.
- `primary_category`, `secondary_category`, `tertiary_category`.
- `highlights` (e.g. `Hydrating`, `Oil Free`, `Good for: Dark spots`).
- Short ingredient summary (truncated to manageable length).

Example assembled text:

> "Hydra Vizor Invisible Moisturizer Broad Spectrum SPF 30 Sunscreen with Niacinamide + Kalahari Melon by Fenty Skin. Category: Skincare > Moisturizers > Moisturizers. Highlights: Hydrating, Oil Free, Clean at Sephora, Black Owned. Key ingredients: Niacinamide, Hyaluronic Acid, Kalahari Melon Seed Oil."

This string is embedded using a sentence embedding model and stored as `product_embedding` for that `product_id`.

### 4.2 Effect embedding

**Purpose:** capture what the product actually does based on reviews, using TF–IDF weighted effects.

#### 4.2.1 Static effect vectors

For each canonical `effect_name` in `effect_vocabulary`, compute an embedding by encoding:

> "EFFECT_NAME – short explanation"

e.g. `"hydration – increases skin moisture and plumpness"`.

Store the resulting vector as `v_effect[effect_name]`.

#### 4.2.2 Per-product effect embedding

For product `p` with effect weights `w_{p,e}`:

```text
v_effect(p) = sum_e w_{p,e} * v_effect[e]
```

Optionally mix in a small amount of the product description embedding to encourage diversity:

```text
v_effect_final(p) = v_effect(p) + gamma * v_product_description(p)
```

where `gamma` is a small scalar (e.g. 0.1).

### 4.3 Storage in a vector DB

Use a vector database (e.g. Weaviate, Qdrant, pgvector) with either:

- One collection `products` with two named vectors: `product_embedding` and `effect_embedding`, or
- Two collections keyed by `product_id`: `product_text_vectors` and `product_effect_vectors`.

Each stored object also includes scalar metadata for filtering (categories, price, tags like `has_retinol`, `has_strong_acid`, etc.).

---

## 5. Retrieval & Recommendation Logic

Given a `user_information` dict and a requested product type (e.g. `"moisturizer"`), we build a user effect profile and search in effect space.

### 5.1 Mapping user profile to a target effect vector

The `user_information` structure:

```python
user_information = {
    "goals": {
        "firmer_skin": False,
        "hydrate": False,
        "glow_up": False,
        "repair_renew": False,
        "soothe_relax": False,
        "protect": False
    },
    "conditions": {
        "acne": False,
        "clogged_pores": False,
        "redness": True,
        "wrinkles": True,
        "pigmentation": True,
        "rough_texture": False
    },
    "skin_type": 67,
    "sensitive_skin": False,
    "age": 30,
    "additional_concerns": "My skin doesn't work well with retinol"
}
```

We derive user weights `u_e` over `effect_name`:

- From goals:
  - `hydrate=True` → increase weights for `hydration`, `strengthened_skin_barrier`.
  - `glow_up=True` → `radiance`, `evened_pigmentation`.
  - `repair_renew=True` → `reduced_fine_lines`, `skin_texture_smoothing`.
- From conditions:
  - `redness=True` → `reduced_redness`, avoid `increased_redness`.
  - `pigmentation=True` → `reduced_dark_spots`, `evened_pigmentation`.
  - `acne=True` → `reduced_acne`, penalize `caused_breakouts`.
- From `skin_type` and `sensitive_skin`:
  - If `skin_type > 66` (oily) → emphasize `oil_control`, `matte_finish`, penalize `heavy_greasy`.
  - If `skin_type < 33` (dry) → emphasize `hydration`, penalize `drying`.
  - If `sensitive_skin=True` → penalize `strong_fragrance`, `stinging`, `burning`, etc.
- From `additional_concerns`:
  - Small LLM mapping step to parse text and add constraints (e.g. `avoid_retinol`, extra penalties for effects related to `retinol_irritation`).

We then build a user vector in the same effect space:

```text
v_user = sum_e u_e * v_effect[e]
```

### 5.2 Candidate filtering

Before vector search, filter products by metadata:

- `primary_category = 'Skincare'`.
- `secondary_category` / `tertiary_category` matching requested type (e.g. `"Moisturizers"`, `"Sunscreens"`, `"Toners"`).
- Optional constraints: price range, specific brands, `sephora_exclusive`, etc.
- Ingredient/label constraints derived from `additional_concerns` (e.g. avoid `has_retinol = 1`, `has_strong_acid = 1`).

### 5.3 Effect-based similarity search

Run K-NN search in the **effect_embedding** space:

- Query vector: `v_user`.
- Candidate set: products passing filters.
- Metric: cosine similarity.

Return top-N (e.g. 50) products with similarity scores and associated `product_effect_profile` rows so we can explain why each product matched.

### 5.4 Re-ranking with constraints

Apply a lightweight scoring layer on top of similarity:

- Subtract penalties for high `tfidf_weight` on negative or side-effect labels that conflict with the user profile (e.g. `caused_breakouts` when `acne=True`).
- Add bonuses when positive effects directly address user goals/conditions.
- Optionally boost products with a large number of reviews from users with similar `skin_type_signals`.

The final score is something like:

```text
score = sim_effect
score -= 0.3 * weight("caused_breakouts") if user.conditions["acne"] else 0
score -= 0.2 * weight("strong_fragrance") if user.sensitive_skin else 0
score += 0.1 * weight("hydration") if user.goals["hydrate"] else 0
...
```

Top-K results from this step are fed into the routine builder.

---

## 6. Routine Generation (Morning / Evening)

The routine builder uses an LLM to assemble structured routines (morning and evening) from shortlisted products.

### 6.1 Category-to-slot mapping

Define canonical routine slots:

- Morning: `cleanser`, `treatment/serum`, `moisturizer`, `sunscreen`.
- Evening: `cleanser`, `treatment/serum`, `moisturizer`.
- Optional: `toner`, `mask`, `eye_cream` when relevant.

Map `secondary_category` / `tertiary_category` from `product_info` into these slots (e.g. `Moisturizers` → `moisturizer`, `Toners` → `toner`).

### 6.2 Inputs to the routine builder LLM

For each routine generation request provide:

- The full `user_information` structure.
- For each slot, a small set (e.g. 5) of top-scoring candidate products, including:
  - `product_id`, `product_name`, `brand_name`, `price_usd`.
  - `highlights`, truncated ingredients list, tags such as `has_retinol`, `has_strong_acid`, `has_fragrance`.
  - Top positive and negative effects from `product_effect_profile` (e.g. 5 highest `tfidf_weight` per type).

The prompt instructs the LLM to:

1. Choose at most one product per slot for morning and evening.
2. Respect user conditions and constraints (avoid retinol, avoid fragrance for sensitive skin, etc.).
3. Avoid conflicting combinations of actives within the same routine (e.g. no strong acid + retinol on the same night for sensitive users).
4. Output both:
   - Structured JSON describing the routine steps.
   - Natural language explanations and usage instructions.

### 6.3 Ingredient conflict checks

Implement a simple rule layer, independent of the LLM:

- Parse ingredient lists to set boolean flags: `has_retinol`, `has_strong_acid`, `has_vitamin_c`, `has_high_alcohol`, `has_fragrance`.
- Provide these flags to the LLM and optionally pre-filter candidate combinations that obviously violate rules (e.g. no retinol in morning, limit number of strong acid products in one routine).

---

## 7. Agentic Chat Interface

The chatbot front-end orchestrates user profiling, product search, routine generation, and explanation.

### 7.1 Core flows

- **Onboarding Q&A:** ask a short sequence of questions to populate `user_information` (goals, conditions, skin type score, sensitivities, age, budget, product-type request).
- **Product-type recommendation:** user asks, e.g., "Recommend a moisturizer for my oily, acne-prone skin" → update profile, run effect-based retrieval for moisturizers, show ranked list.
- **Routine creation:** once enough signals are collected, call the routine builder LLM to generate morning and evening routines.
- **Evidence-based explanations:** when user asks "Why this product for redness?", fetch representative reviews from `review_effects` where `reduced_redness` appears and show short snippets.

### 7.2 Internal tools for the agent

Expose a small set of tools/functions that the agent can call:

1. `update_user_profile(fields)` – modify `user_information` from chat answers.
2. `search_products_by_effect(user_profile, product_type)` – implements Sections 5.1–5.4 and returns ranked products.
3. `build_routine(user_profile)` – orchestrates calling the routine builder LLM.
4. `get_effect_evidence(product_id, effect_name)` – retrieves a few `review_text` snippets where the effect is present.

The agent chain decides which tool to call based on user input and conversation context.

### 7.3 Persistence

- If users log in, store `user_id` with `user_information`, previously chosen routines, and feedback (likes/dislikes of recommendations).
- Use this history to refine future recommendations (e.g. penalize products the user previously disliked).

---

## 8. Implementation Notes and Open Decisions

### 8.1 Models and infrastructure

- **LLM for extraction and routines:** use a strong model (e.g. Claude, GPT-4.x) for offline extraction and online routine building.
- **Embedding model:** use a consistent sentence embedding model for:
  - Effect labels
  - Product description text
  - Optional natural-language user queries
- **Vector DB:** Weaviate (for close alignment with Glowe) or an alternative like Qdrant/pgvector depending on deployment constraints.

### 8.2 Evaluation strategy

- Manually inspect a sample of products and their top effects against real reviews to validate extraction.
- Construct simple offline test cases (e.g. oily skin with redness) and check if top results from effect-based retrieval make sense to a domain expert.
- Compare single-embedding vs dual-embedding performance on these cases to confirm the benefit of the effect embedding.

### 8.3 Extension: image-based profiling

In later stages, plug an image model into the onboarding flow to infer `skin_type`, `acne_severity`, or other attributes from face photos. These signals simply feed into the same mapping from `user_information` to `v_user` used by the effect-based retrieval.
