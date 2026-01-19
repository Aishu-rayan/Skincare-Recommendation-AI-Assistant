# Implementation Notes (Engineering Deep Dive)

This document expands the recruiter-facing `README.md` with a more technical view of the intended v1 implementation.

## 1) Core idea

Turn **unstructured review text** into **structured, queryable “effect signals”**, then use those signals to power:

1) **transparent scoring** (TF–IDF over effects, segment match), and
2) **semantic retrieval** (one embedding per product),

while keeping outputs phrased as **“users report…”** with review evidence.

## 2) Database: source tables + derived tables

### 2.1 Source tables (existing)

- `product_info`
- `select_customer_reviews`

### 2.2 Derived tables (v1)

Defined in `docs/db_schema.md`:

- `review_effects`: per-review structured extraction output
- `extraction_errors`: parse/validation failures for retry
- `effect_vocabulary`: canonical effect labels (benefit / negative / side_effect)
- `product_effect_profile`: per-product TF–IDF weights over effects
- `product_segment_stats`: per-product counts for oily/dry/sensitive review coverage
- `product_vectors`: per-product effect summary + embedding (BLOB)

## 3) Offline pipeline (one-time build)

### Step A — Dry-run gate (cost/time)

Run extraction on a representative sample (1k–5k reviews) to estimate:

- average prompt/output tokens
- reviews/min throughput
- total runtime and cost for (a) full pass and (b) sampling policy

This determines whether extraction runs **remote (low-cost model)** vs **local (Ollama)**.

### Step B — Review extraction (`select_customer_reviews` → `review_effects`)

For each review, extract a strict JSON payload like:

```json
{
  "positive_effects": ["hydration", "reduced_redness"],
  "negative_effects": ["caused_breakouts"],
  "side_effects": ["stinging"],
  "skin_type_signals": ["oily", "sensitive"],
  "conditions_helped": ["acne", "redness"],
  "satisfaction_score": 0.8
}
```

Normalization rules (v1):

- lower-case, canonical labels only (via `effect_vocabulary`)
- skin types normalized to `{dry, normal, combination, oily, sensitive}`
- idempotent writes by `(review_rowid, extraction_version)`

### Step C — TF–IDF effect profiles (`review_effects` → `product_effect_profile`)

Compute TF–IDF weights per `(product_id, effect)`:

- `TF_{p,e} = log(c_pe + 1)` where `c_pe` is the count of reviews mentioning effect `e` for product `p`
- `IDF_e = log(N / (df_e + 1))`

Store top effects per product; these are used for:

- transparent matching (boost/penalty)
- building `effect_summary` strings

### Step D — Per-product “users report…” summary + embeddings

Deterministically generate an `effect_summary` per product (e.g., top 5 benefits + top 3 negatives + best-for segments), then embed it and store in `product_vectors`.

Vector search is done in SQLite using `sqlite-vector`.

## 4) Online runtime (FastAPI)

### 4.1 Session profile

Profile is session-only (no auth):

- `POST /profile` creates/replaces
- `PATCH /profile` updates partially

See examples in `docs/api_contracts.md`.

### 4.2 Recommendation flow (`POST /recommend`)

1. **Metadata filter** by product type/category (e.g., sunscreen / cleanser).
2. **Vector retrieval**: embed the user’s query string and KNN over `product_vectors`.
3. **Re-rank** candidates using:
   - benefit matches from `product_effect_profile`
   - penalties for conflicting negatives/side effects
   - segment multiplier using `product_segment_stats` (e.g., oily/dry/sensitive match)
4. Return:
   - ranked results
   - `users_report_summary`
   - top matched effects

### 4.3 Routine generation (`POST /routine`)

Given a candidate pool per routine slot (AM/PM), call a configurable LLM to assemble a safe routine:

- Morning: cleanser → treatment/serum → moisturizer → sunscreen
- Evening: cleanser → treatment/serum → moisturizer

The routine response is structured JSON plus a short disclaimer.

### 4.4 Evidence (“why this product?”)

- `GET /explain/product/{product_id}` returns top effects
- `POST /explain/evidence` returns review snippets for an effect

Optional performance upgrade: SQLite FTS5 over review text (see `docs/db_schema.md`).

## 5) Planned code layout

The proposed module layout is documented in `docs/folder_structure.md`.

## 6) Design references

- System design: `skincare_ai_design_plan.md`
- Requirements: `docs/technical_requirements.md`
- DB schema: `docs/db_schema.md`
- API contracts: `docs/api_contracts.md`
