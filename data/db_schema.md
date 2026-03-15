# Database Schema (v1)

![Data model](./diagrams/data_model.png)

Target DB: update the existing `sephora_select_reviews.db` (backup managed manually).

## 1. Existing tables (read)
- `product_info`
- `select_customer_reviews`

## 2. New / derived tables

### 2.1 `review_effects`
Stores per-review extraction outputs (minimal v1 schema).

```sql
CREATE TABLE IF NOT EXISTS review_effects (
  review_effect_id   INTEGER PRIMARY KEY,
  review_rowid       INTEGER NOT NULL,
  product_id         TEXT NOT NULL,

  positive_effects   TEXT,
  negative_effects   TEXT,
  side_effects       TEXT,

  skin_type_signals  TEXT,
  conditions_helped  TEXT,

  satisfaction_score REAL,

  extraction_version TEXT NOT NULL,
  extracted_at       TEXT NOT NULL,

  UNIQUE(review_rowid, extraction_version)
);

CREATE INDEX IF NOT EXISTS idx_review_effects_product_id ON review_effects(product_id);
CREATE INDEX IF NOT EXISTS idx_review_effects_review_rowid ON review_effects(review_rowid);
```

### 2.2 `extraction_errors`
Captures parse failures to allow retries.

```sql
CREATE TABLE IF NOT EXISTS extraction_errors (
  error_id           INTEGER PRIMARY KEY,
  review_rowid       INTEGER NOT NULL,
  extraction_version TEXT NOT NULL,
  error_type         TEXT NOT NULL,
  error_message      TEXT,
  created_at         TEXT NOT NULL,
  UNIQUE(review_rowid, extraction_version)
);
```

### 2.3 `effect_vocabulary`

```sql
CREATE TABLE IF NOT EXISTS effect_vocabulary (
  effect_id     INTEGER PRIMARY KEY,
  effect_name   TEXT UNIQUE NOT NULL,
  effect_type   TEXT NOT NULL,          -- benefit | negative | side_effect
  description   TEXT,
  is_condition  INTEGER NOT NULL DEFAULT 0
);
```

### 2.4 `product_effect_profile` (TF–IDF)

```sql
CREATE TABLE IF NOT EXISTS product_effect_profile (
  product_id    TEXT NOT NULL,
  effect_name   TEXT NOT NULL,
  tf            REAL NOT NULL,
  idf           REAL NOT NULL,
  tfidf_weight  REAL NOT NULL,
  effect_type   TEXT NOT NULL,
  PRIMARY KEY (product_id, effect_name)
);

CREATE INDEX IF NOT EXISTS idx_product_effect_profile_product ON product_effect_profile(product_id);
CREATE INDEX IF NOT EXISTS idx_product_effect_profile_effect ON product_effect_profile(effect_name);
```

### 2.5 `product_segment_stats`
Simple per-product counts for segment match (used at query time).

```sql
CREATE TABLE IF NOT EXISTS product_segment_stats (
  product_id            TEXT PRIMARY KEY,
  n_reviews_extracted    INTEGER NOT NULL,
  n_oily                 INTEGER NOT NULL,
  n_dry                  INTEGER NOT NULL,
  n_sensitive            INTEGER NOT NULL,
  updated_at             TEXT NOT NULL
);
```

### 2.6 `product_vectors`
Holds one embedding per product for vector search.

```sql
CREATE TABLE IF NOT EXISTS product_vectors (
  product_id      TEXT PRIMARY KEY,
  effect_summary  TEXT NOT NULL,
  embedding       BLOB NOT NULL,
  embedding_dim   INTEGER NOT NULL,
  embedding_model TEXT NOT NULL,
  created_at      TEXT NOT NULL
);
```

## 3. Optional: FTS for evidence snippets
If fast snippet search over review text is needed:

```sql
-- Requires SQLite compiled with FTS5.
CREATE VIRTUAL TABLE IF NOT EXISTS reviews_fts USING fts5(
  review_rowid UNINDEXED,
  review_text,
  content='select_customer_reviews',
  content_rowid='rowid'
);
```

## 4. sqlite-vector usage notes
- Vectors are stored in ordinary tables as `BLOB`.
- The extension must be loaded on the SQLite connection.
- Initialize and query via sqlite-vector functions (e.g., `vector_init(...)` and scan/join patterns described in sqlite-vector docs).
