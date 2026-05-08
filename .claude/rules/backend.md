# Backend Rules — LAR-E

## ⚠️ Active Directory

Always edit `lare-backend-documents/` — deployed to Railway. `lare-backend-desktop/` is stale, never edit it.

---

## FastAPI Conventions

- All handlers `async def`. Auth: `Depends(get_current_client)`. DB: `Depends(get_db)`.
- `FastAPI(redirect_slashes=False)` — every URL must be exact.

## Trailing Slash Convention (Bug 4 — do not regress)

```
POST /api/auth/login       NO slash
POST /api/auth/signup      NO slash
GET  /api/auth/me          NO slash
POST /api/leads/upload     NO slash
GET  /api/leads/           YES slash  ← exception
GET  /api/campaigns        NO slash
POST /api/campaigns/       NO slash
```

When adding an endpoint: decide slash, add to this list, audit all frontend calls.

---

## SQLAlchemy 2.0

Use `select()` + `session.execute()` (not legacy `session.query()`). Relationships: explicit `selectinload()` or `joinedload()` — lazy loading disabled in async. Every query must scope by `client_id`.

---

## NaN Defense Pattern (Bug 2 — two-layer fix, both layers required)

Apply **after** any sort/reset operations on the DataFrame:

```python
# Layer 1 — astype(object) required; without it pandas re-flips None to NaN in float64 cols
valid_df = valid_df.astype(object).where(pd.notna(valid_df), None)

# Layer 2 — wrap every field in Lead() constructor
def _v(value):
    return None if (isinstance(value, float) and math.isnan(value)) else value

Lead(name=_v(row["name"]), phone=_v(row["phone"]), ...)
```

Never skip Layer 1 even if Layer 2 is present. `score_and_tier()` calls `.sort_values()` + `.reset_index()` which re-flips dtypes to float64.

---

## CORS

`allow_origins=["*"]` + credentials = TypeError. Origins in Railway env var `CORS_ORIGINS`. Current list: `https://lar-e-frontend-desig-f3lg.bolt.host` · `https://*.bolt.host` · `http://localhost:3000` · `http://localhost:5173`. Add new origins to Railway env, then restart.

---

## arq Workers

Live in `app/workers/`. Separate processes — changes need Railway redeploy. Always check arq job function signature matches enqueuing call. Reminder window: 45–75 min.

---

## Alembic

Run `alembic upgrade head` locally before deploying. Railway runs it on deploy automatically. Never skip migration generation for model changes.

---

## Watch Items

**BE-04 — Dashboard route conflict:** `/api/dashboard/summary` and `/api/dashboard/{campaign_id}` share a prefix. `summary` works because it's declared first. Do not reorder these routes.

**Mock mode:** Backend defaults to mock for BSP + LLM when credentials absent. `MOCK_MODE=true` logs a warning on startup — intentional, not a bug.

---

## Architectural Decisions (do not revisit without cause)

| Decision | Rationale |
|----------|-----------|
| No RAG | Full KB fits in Claude's context — RAG is over-engineering for MVP |
| No multi-agent | Single well-prompted agent with intent classification is sufficient |
| arq + Redis | Lightweight, Python-native, fits async FastAPI |
| Booking link (not Calendar API) | Simpler, works with any scheduling tool |
