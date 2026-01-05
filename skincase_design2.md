# Skincare AI Assistant Design Using Sephora Reviews (Inspired by Glowe)

Below is a detailed design plan for turning your `sephora_select_reviews.db` (≈239k reviews, ≈8.5k products) into a Glowe‑style, agentic skincare assistant focused on moisturizers, sunscreens, toners, etc.

The plan is organized into:
1. Data understanding & target schema
2. Offline review‑extraction pipeline
3. Effect vocabulary & TF‑IDF profiling
4. Dual‑embedding design (product vs. effects)
5. Retrieval & recommendation flow
6. Routine generation & chat agent
7. Implementation notes (LLMs, evaluation, ops)
8. Clarifying questions for you

I’ll reference the architecture diagram we generated and adapt Glowe’s ideas to your exact tables (`product_info`, `select_customer_reviews`).

---

## 1. Data Understanding & Target Entities

### 1.1 Existing tables

From the SQLite inspection:
- `product_info` (8,494 rows)
  - Key columns: `product_id`, `product_name`, `brand_id`, `brand_name`, `ingredients`, `price_usd`, `rating`, `reviews` count, `highlights`, `primary_category`, `secondary_category`, `tertiary_category`, etc.
- `select_customer_reviews` (238,929 rows)
  - Key columns: `primary_category`, `secondary_category`, `tertiary_category`, `product_id`, `product_name`, `brand_name`, `review_title`, `review_text`, `rating`, `is_recommended`, `skin_type`, `skin_tone`, `eye_color`, `ingredients`, `price_usd`, `highlights`.

This is enough to:
- Join reviews → products via `product_id`.
- Focus on skincare categories: `primary_category='Skincare'` and `secondary_category` in `{Moisturizers, Sunscreens, Cleansers, Treatments, Toners, Masks, Eye Creams}`.

### 1.2 Target conceptual entities

Define the following logical entities (some backed by new DB tables):
- **Product** – from `product_info`, 1 row per SKU.
- **Raw Review** – from `select_customer_reviews`, 1 row per user review.
- **Structured Review Effects** – per review, fields extracted by an LLM.
- **Per‑product Effect Profile** – TF‑IDF weighting of effects for that product.
- **Effect Vocabulary** – canonical list of effect labels (e.g. `hydration`, `reduced_redness`).
- **Embeddings** – vectors for products and effects stored in a vector store.
- **User Profile** – `user_information` dict you showed (goals, conditions, skin type, sensitivities, age, etc.).

---

## 2. Offline Review‑Extraction Pipeline

Goal: convert free‑text reviews into rich, structured signals that describe what each product *actually does* for different skin profiles and in what context.

### 2.1 New table: `review_effects`

Create a table (in SQLite or, better, a downstream warehouse) with schema similar to:

```sql
CREATE TABLE review_effects (
    review_effect_id   INTEGER PRIMARY KEY,
    review_rowid       INTEGER NOT NULL,         -- rowid / PK from select_customer_reviews
    product_id         TEXT NOT NULL,

    -- core extracted info
    positive_effects   TEXT NOT NULL,            -- JSON list[str]
    negative_effects   TEXT NOT NULL,            -- JSON list[str]
    side_effects       TEXT NOT NULL,            -- JSON list[str]
    usage_experience   TEXT NOT NULL,            -- JSON list[str] ("absorbs quickly", "pills under makeup")

    -- user & context signals
    skin_type_signals  TEXT NOT NULL,            -- JSON list[str] ("oily", "dry", "sensitive")
    conditions_helped  TEXT NOT NULL,            -- JSON list[str] ("acne", "redness", "hyperpigmentation")
    conditions_worsened TEXT NOT NULL,           -- JSON list[str]
    age_group          TEXT NOT NULL,            -- e.g. "<20", "20-29", "30-39", "40+"
    time_to_see_results TEXT NOT NULL,           -- JSON list[str], e.g. ["1 week", "1 month"]

    -- behavioral / satisfaction
    overall_sentiment  REAL NOT NULL,            -- [-1, 1]
    satisfaction_score REAL NOT NULL,            -- 0-1
    repurchase_intent  REAL NOT NULL,            -- 0-1 probability
    recommend_to_friend REAL NOT NULL,           -- 0-1

    -- meta
    extraction_version TEXT NOT NULL,
    extracted_at       TEXT NOT NULL
);
```

**Why these fields?**
- Positive/negative/side effects: the core of “what this did to my skin”.
- Usage experience: texture, smell, white cast, pilling – critical for sunscreen/moisturizer UX.
- Skin/condition signals: allow you to match a user with similar reviewers.
- Behavioral scores: weight reviews (e.g., strongly positive + repurchase intent) higher in TF‑IDF.

### 2.2 LLM extraction prompt design

Use a robust model (e.g., Claude, GPT‑4.1, or similar) in a batch/offline pipeline.
For each review row, feed:
- `review_text` (full, not truncated, so for ETL read from source, not UI‑truncated view).
- `rating`, `is_recommended`, `skin_type` if present.

Ask the model to output a strict JSON with constrained vocabularies where possible.

Example extraction instruction (simplified):

```json
{
  "positive_effects": ["Hydration", "Reduced redness"],
  "negative_effects": ["Caused breakouts"],
  "side_effects": ["Stung around eyes"],
  "usage_experience": ["No white cast", "Lightweight", "Strong fragrance"],
  "skin_type_signals": ["oily", "sensitive"],
  "conditions_helped": ["acne", "redness"],
  "conditions_worsened": ["clogged pores"],
  "age_group": "30-39",
  "time_to_see_results": ["2 weeks"],
  "overall_sentiment": 0.7,
  "satisfaction_score": 0.8,
  "repurchase_intent": 0.9,
  "recommend_to_friend": 0.9
}
```

Implementation details:
- Use a **fixed label set** for conditions (`acne`, `redness`, `pigmentation`, `wrinkles`, `rough_texture`, `clogged_pores`, `dark_circles`, etc.) matching your `user_information.conditions` keys + a few extras.
- For `skin_type_signals`, map to `{dry, normal, combination, oily, sensitive}`.
- Normalize text (lowercase, lemmatize where appropriate) but keep human‑readable labels for debugging.
- Include `extraction_version` to allow re‑runs with improved prompts.

### 2.3 Batch processing strategy

- Work in chunks by `product_id` or by rowid ranges (e.g., 1,000 reviews per batch) to parallelize.
- Store raw LLM JSON output, then run a validation/normalization step (ensures keys exist, lists are lists, etc.).
- Keep a simple `extraction_errors` table to log rows where the LLM output was invalid and need re‑processing.

### 2.4 Handling noisy or short reviews

- If `review_text` is below some length threshold or contains no clear effect, allow the model to output empty arrays and a neutral sentiment.
- You can down‑weight these reviews later based on `satisfaction_score` and review length.

---

## 3. Effect Vocabulary & TF‑IDF Profiling

After `review_effects` is populated, you build a **canonical effect vocabulary** and a **per‑product effect profile** à la Glowe.

### 3.1 New table: `effect_vocabulary`

Store canonical effect names and their type:

```sql
CREATE TABLE effect_vocabulary (
    effect_id      INTEGER PRIMARY KEY,
    effect_name    TEXT UNIQUE NOT NULL,  -- e.g. "hydration", "reduced_redness"
    effect_type    TEXT NOT NULL,         -- {"benefit", "negative", "side_effect", "experience"}
    description    TEXT,
    is_condition   INTEGER NOT NULL DEFAULT 0  -- 1 if maps directly to condition key
);
```

### 3.2 Building the vocabulary

1. Collect all raw phrases from
   - `positive_effects` / `negative_effects` / `side_effects` / `usage_experience`.
2. Cluster + normalize them into ~200–300 canonical labels (similar to Glowe’s 250 effects):
   - Use simple heuristics first: lowercase, strip punctuation, singularize (`hydrating` → `hydration`).
   - Optionally, use an embedding model to cluster similar phrases and label each cluster manually once.
3. Map each raw phrase to a canonical `effect_name`.
   - During normalization step for each review, replace raw strings with canonical labels.

Examples:
- `"my skin feels so hydrated"`, `"very moisturizing"` → `hydration` (benefit).
- `"reduced redness"`, `"calmed my rosacea"` → `reduced_redness` (benefit, condition‑linked).
- `"caused breakouts"`, `"clogged my pores"` → `caused_breakouts` (negative, condition‑linked to acne/clogged_pores).
- `"burned around eyes"` → `eye_irritation` (side_effect).
- `"strong fragrance"` → `strong_fragrance` (experience/negative for sensitive users).

### 3.3 New table: `product_effect_profile`

For each `(product_id, effect_name)` pair, compute TF‑IDF style weights.

```sql
CREATE TABLE product_effect_profile (
    product_id   TEXT NOT NULL,
    effect_name  TEXT NOT NULL,
    tf           REAL NOT NULL,
    idf          REAL NOT NULL,
    tfidf_weight REAL NOT NULL,
    effect_type  TEXT NOT NULL,         -- denormalized from vocabulary
    PRIMARY KEY (product_id, effect_name)
);
```

#### 3.3.1 Term Frequency (TF)

For product *p* and effect *e*:

- Let `c_pe` = sum over all its reviews of:
  - 1 per mention, optionally multiplied by `satisfaction_score` or `overall_sentiment` to reward strong positive reviews.
- Then:

```math
TF_{p,e} = log(c_{p,e} + 1)
```

#### 3.3.2 Inverse Document Frequency (IDF)

- Let N = total number of products.
- Let `df_e` = number of distinct products where `c_pe > 0`.

```math
IDF_e = log(N / df_e)
```

This makes rare but meaningful effects (e.g. `reduced_redness`) more important than common ones (e.g. `hydration`).

#### 3.3.3 Normalized TF‑IDF weight

For each product *p*:

```math
w_{p,e} = (TF_{p,e} * IDF_e) / \sum_j (TF_{p,j} * IDF_j)
```

Store `w_{p,e}` as `tfidf_weight`, so all weights for a product roughly sum to 1.

### 3.4 Optional condition‑aware profiles

You can also compute variant profiles filtered by reviewer conditions or skin types, e.g.:
- `product_effect_profile_oily_only`
- `product_effect_profile_sensitive_only`

These re‑compute TF / DF using only reviews where `skin_type_signals` contains `oily`, etc., so you can answer “best sunscreens for oily, acne‑prone skin” more precisely.

---

## 4. Dual‑Embedding Design

Following Glowe, build **two separate embeddings** per product, each stored as its own named vector in a vector DB (Weaviate, Qdrant, pgvector, etc.).

### 4.1 Product embedding

**Purpose:** general product search and to provide semantic context (ingredients, marketing claims, categories).

**Input text for each product:**
- `product_name`, `brand_name`
- `primary/secondary/tertiary_category`
- `highlights` list (e.g. `Hydrating`, `Oil Free`, `Good for: Dark spots`)
- Short ingredients summary (truncate to avoid very long strings)

Example text:

> "Hydra Vizor Invisible Moisturizer Broad Spectrum SPF 30 Sunscreen with Niacinamide + Kalahari Melon by Fenty Skin. Category: Skincare > Moisturizers > Moisturizers. Highlights: Hydrating, Oil Free, Clean at Sephora, Black Owned. Key ingredients: Niacinamide, Hyaluronic Acid, Kalahari Melon Seed Oil."

Embed this using a sentence embedding model (e.g., `all-MiniLM-L12-v2`, OpenAI text‑embedding, etc.).

Store: `product_embedding` associated with `product_id`.

### 4.2 Effect embedding

**Purpose:** reflect what the product *actually does* on skin, using weighted review evidence.

#### 4.2.1 Static effect embeddings

For each canonical `effect_name`, generate a static vector:
- Simple approach: embed just the effect name, optionally with a short description.
  - e.g., `"Hydration (increases skin moisture, plumpness)"`.

Store them in memory or in a small `effect_vectors` table.

#### 4.2.2 Combine using TF‑IDF weights

For each product *p*:

```math
v_{effect}(p) = \sum_e w_{p,e} * v_e
```

where `v_e` is the embedding of effect *e*.

Optionally, add a small fraction of the product description vector for diversity (as Glowe does):

```math
v_{effect}^{final}(p) = v_{effect}(p) + γ * v_{description}(p)
```

with `γ` a small scalar (e.g. 0.1).

Store: `effect_embedding` associated with `product_id`.

### 4.3 Storage

Choose a vector DB that supports named vectors or store two collections:
- `products` collection: includes product metadata and both vectors: `{product_embedding, effect_embedding}`.
- Or two collections: `product_text_vectors` and `product_effect_vectors` keyed by `product_id`.

---

## 5. Retrieval & Recommendation Flow

Goal: for a given `user_information` + requested product type (e.g. "moisturizer"), return a ranked shortlist of products.

### 5.1 Mapping user profile → target effect vector

Given `user_information`:

```python
user_information = {
  "goals": {"hydrate": True, "glow_up": True, ...},
  "conditions": {"acne": False, "redness": True, ...},
  "skin_type": 67,  # oily
  "sensitive_skin": False,
  "age": 30,
  "additional_concerns": "My skin doesn't work well with retinol"
}
```

Construct a **text query** or an explicit **effect weight vector**:

1. From goals:
   - `hydrate` → high weight on `hydration` effect.
   - `glow_up` → `radiance`, `even_skin_tone`.
   - `repair_renew` → `skin_barrier_repair`, `reduced_fine_lines`.
2. From conditions:
   - `redness=True` → emphasize `reduced_redness`, avoid `increased_redness` negative effects.
   - `pigmentation=True` → `reduced_dark_spots`, `evened_pigmentation`.
3. From `skin_type` & `sensitive_skin`:
   - `>66` → oily: emphasize `oil_control`, `matte_finish`, avoid `heavy_greasy`.
   - `sensitive_skin=True` → avoid `strong_fragrance`, `stinging`, `burning`, `alcohol_drying`.
4. From `additional_concerns`:
   - Run a small LLM mapping: `"My skin doesn't work well with retinol"` → add constraint tag `avoid_retinol` and map to effects like `retinol_irritation` to exclude.

You can either:
- (A) Generate a **natural language query** for effect search like:

  > "Find moisturizers that provide hydration, glow, reduce redness and pigmentation for oily, non‑sensitive skin, avoid irritation and retinol."

  and embed this text with the same effect embedding model, or
- (B) Build an explicit vector: sum static effect vectors weighted by inferred user weights (similar to how you built product effect vectors).

Glowe effectively does (B). I’d suggest:

```math
v_{user} = \sum_e u_e * v_e
```

where `u_e` are user weights from goals/conditions.

### 5.2 Filtering by product type & rules

Before vector search, filter products by:
- `primary_category='Skincare'`.
- `secondary_category` / `tertiary_category` matching requested type (`"Moisturizers"`, `"Sunscreens"`, `"Toners"`).
- Optional filters: price range, brand exclusions, `sephora_exclusive`, `new` vs `classic`.
- Constraint filters from `additional_concerns` (e.g. ingredients to avoid like `retinol`, `fragrance`, `essential oils`).

### 5.3 Effect‑based similarity search

Run K‑NN search in the **effect_embedding** space:
- Query vector: `v_user`.
- Candidates: all products in the requested category, respecting filters.
- Metric: cosine similarity.

Return top‑N (e.g. 50) products plus their similarity scores and per‑effect contributions (`w_{p,e}`) for explainability.

### 5.4 Re‑ranking with constraints

Post‑process candidates:
- Penalize products with high TF‑IDF weights for negative/side‑effect labels that conflict with user (`"caused_breakouts"` for acne, `"strong_fragrance"` for sensitivity).
- Slightly boost products with many reviews for the user’s skin type / conditions.
- You can maintain a small rule‑based scorer:

```python
score = sim_effect
score -= 0.3 * weight("caused_breakouts") if user.conditions["acne"] else 0
score -= 0.2 * weight("strong_fragrance") if user.sensitive_skin else 0
score += 0.1 * weight("hydration") if user.goals["hydrate"] else 0
...
```

Final top‑K are passed to the LLM routine builder.

---

## 6. Routine Generation (Morning/Evening)

Like Glowe, routine generation is a separate LLM step that reasons over the shortlisted products.

### 6.1 Product category selection

For a basic flow:
- Morning baseline: `cleanser`, `treatment/serum` (optional), `moisturizer`, `sunscreen`.
- Evening baseline: `cleanser`, `treatment/serum`, `moisturizer`.
- Add optional slots (toner, mask, eye cream) when relevant to goals/conditions.

Map your `secondary_category` / `tertiary_category` strings to these canonical slots.

### 6.2 Routine Builder LLM input

Provide the LLM:
- User profile (`user_information`).
- For each category slot, top ~5 products from the effect search with metadata:
  - Name, brand, price, highlights, key ingredients.
  - Summary of top 5 positive/negative effects from `product_effect_profile`.
  - Flags for conflict ingredients (retinol, strong acids, etc.).

Ask the LLM to:
1. Select at most one product per slot for morning and evening.
2. Respect user constraints (avoid retinol etc.).
3. Avoid conflicting active combinations in the same routine.
4. Produce:
   - Structured JSON routine (morning and evening steps, ordered).
   - Natural language explanation: why each product is chosen, which effects and user goals it addresses, what to watch out for.

Example JSON:

```json
{
  "morning": [
    {"step": 1, "slot": "cleanser", "product_id": "P123"},
    {"step": 2, "slot": "moisturizer", "product_id": "P456"},
    {"step": 3, "slot": "sunscreen", "product_id": "P467249"}
  ],
  "evening": [
    {"step": 1, "slot": "cleanser", "product_id": "P123"},
    {"step": 2, "slot": "treatment", "product_id": "P440949"},
    {"step": 3, "slot": "moisturizer", "product_id": "P789"}
  ],
  "notes": "Avoid using exfoliating treatment on consecutive nights if irritation occurs..."
}
```

### 6.3 Ingredient conflict checks

You can either:
- Pre‑compute simple rules over ingredient lists (regex match on `retinol`, `AHA`, `BHA`, `benzoyl peroxide`, etc.), or
- Ask the LLM directly to flag conflicts when given full ingredient lists.

Start simple:
- Tags at product level: `has_strong_acid`, `has_retinol`, `has_high_alcohol`, `has_fragrance`.
- Provide tags to the LLM, and rules like “avoid more than one strong acid per routine; avoid retinol in morning.”

---

## 7. Agentic Chat Frontend

The chat interface wraps the retrieval & routine builder pipeline and lets users explore.

### 7.1 Core capabilities

- **Onboarding Q&A** → fill `user_information`.
- **Product‑type queries**: “Recommend a moisturizer for my oily, acne‑prone skin”.
- **Routine refinement**: “This moisturizer feels heavy, can you swap it?” → remove one product and re‑run retrieval for that slot.
- **Review‑grounded answers**: When user asks “Why this product for redness?”, fetch representative reviews where `reduced_redness` was extracted and quote them.

### 7.2 Tools/functions for the agent

Define internal tools the LLM agent can call:
1. `update_user_profile(fields)` – modify `user_information`.
2. `search_products_by_effect(user_profile, product_type)` – runs the effect embedding search.
3. `build_routine(user_profile)` – runs routine builder.
4. `show_evidence(product_id, effect_name)` – returns a few review snippets that mention that effect.

Use an agent framework (or a simple controller) to decide which tool to call based on chat turns.

### 7.3 Personalization persistence

- Store `user_id` with their `user_information` and any previously chosen routine.
- Allow the model to retrieve previous sessions to refine recommendations over time.

---

## 8. Implementation Notes

### 8.1 Models & infra

- **LLM for extraction & routine building:** High‑quality model (Claude, GPT‑4.x, or similar) run offline for extraction, online for routine generation.
- **Embedding model:** A robust sentence embedding model; use the same model for:
  - Effect names
  - Product description
  - Optional user query text
- **Vector DB:** Weaviate (to stay closest to Glowe), Qdrant, or pgvector.

### 8.2 Evaluation strategy

- Manual inspection: sample 100 products and see if effect labels align with real reviews.
- Offline ranking tests: create small test sets (“user with redness and oily skin”) and check top‑N products vs. intuition.
- A/B similar to Glowe: try single‑embedding vs your dual‑embedding; see if effect‑based vector gives better, more interpretable stacks.

### 8.3 Extending to image‑based skin typing

Later, plug your face‑image model upstream of `user_information` to infer `skin_type`, `acne_severity`, etc. This just becomes another signal in the mapping from user → `v_user`.

---

## 9. Clarifying Questions

To refine this design to your exact needs, I’d like to clarify:

1. **Vector DB choice:** Do you already have preferences or constraints (e.g., must be self‑hosted vs. SaaS like Weaviate Cloud)?
2. **LLM budget:** Are you comfortable running a full LLM pass over all ~240k reviews (potentially re‑running as prompts improve), or do we need a more budget‑constrained extraction strategy (sampling reviews per product, etc.)?
3. **Coverage of product categories:** For v1, do you want to support *all* skincare categories, or just `{cleanser, moisturizer, sunscreen, toner, treatment}`?
4. **User data storage:** Will this be a logged‑in experience with persistent user profiles, or more of an anonymous, one‑off session?
5. **Regulatory / medical stance:** How conservative should the system be when discussing conditions like acne and rosacea – purely cosmetic advice, or allowed to speak more strongly about outcomes?
6. **Tech stack preferences:** Do you already lean toward Python + FastAPI, or any other backend stack, so I can align further details (e.g., job queues, ETL tools) with that?

Once you answer these, I can adjust the design (e.g., number of effect labels, batching strategies, stack choices) before you start implementing.