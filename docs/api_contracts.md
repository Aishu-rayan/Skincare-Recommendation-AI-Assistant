# API Contracts (v1)

Base URL (local): `http://localhost:<backend_port>`

## 1) Health
### `GET /health`
**200**
```json
{ "status": "ok" }
```

## 2) User profile (session-only)
### `POST /profile`
Create/replace the current session profile.

Request
```json
{
  "skin_type": "oily",
  "sensitive_skin": false,
  "age": 30,
  "goals": { "hydrate": true, "glow_up": false, "protect": true },
  "conditions": { "acne": true, "redness": true, "pigmentation": false },
  "additional_concerns": "I don't tolerate strong fragrances"
}
```

Response **200**
```json
{ "ok": true }
```

### `PATCH /profile`
Partial update.

## 3) Recommendations
### `POST /recommend`
Request
```json
{
  "product_type": "sunscreen",
  "top_k": 20,
  "filters": {
    "max_price_usd": 60,
    "brand_whitelist": ["Supergoop!"],
    "min_reviews": 50
  }
}
```

Response **200**
```json
{
  "product_type": "sunscreen",
  "results": [
    {
      "product_id": "...",
      "product_name": "...",
      "brand_name": "...",
      "price_usd": 42.0,
      "score": 0.812,
      "users_report_summary": "Users report hydration and reduced redness; some report pilling.",
      "top_benefits": [{"effect": "hydration", "weight": 0.12}],
      "top_negatives": [{"effect": "pills_under_makeup", "weight": 0.05}]
    }
  ]
}
```

## 4) Routine generation
### `POST /routine`
Generates AM/PM routine using remote OpenAI LLM (model configurable).

Request
```json
{
  "include_optional_toner": true,
  "candidate_pool_size": 50
}
```

Response **200**
```json
{
  "morning": [
    {"step": 1, "slot": "cleanser", "product_id": "...", "notes": "..."},
    {"step": 2, "slot": "treatment", "product_id": "...", "notes": "..."},
    {"step": 3, "slot": "moisturizer", "product_id": "...", "notes": "..."},
    {"step": 4, "slot": "sunscreen", "product_id": "...", "notes": "..."}
  ],
  "evening": [
    {"step": 1, "slot": "cleanser", "product_id": "...", "notes": "..."},
    {"step": 2, "slot": "treatment", "product_id": "...", "notes": "..."},
    {"step": 3, "slot": "moisturizer", "product_id": "...", "notes": "..."}
  ],
  "disclaimer": "These suggestions summarize what users report in reviews and are not medical advice."
}
```

## 5) Evidence / explanations
### `GET /explain/product/{product_id}`
Response **200**
```json
{
  "product_id": "...",
  "top_effects": {
    "benefits": ["hydration", "reduced_redness"],
    "negatives": ["caused_breakouts"],
    "side_effects": ["stinging"]
  }
}
```

### `POST /explain/evidence`
Request
```json
{ "product_id": "...", "effect": "reduced_redness", "limit": 3 }
```

Response **200**
```json
{
  "product_id": "...",
  "effect": "reduced_redness",
  "snippets": [
    {"review_rowid": 123, "text": "..."}
  ]
}
```
