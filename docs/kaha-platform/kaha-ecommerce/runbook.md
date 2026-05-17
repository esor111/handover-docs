# restaurant-ecommerce — Runbook (Setup & Operations)

> ℹ️ **Confluence page placement:** child of *restaurant-ecommerce → Overview*.
>
> **Document standard:** arc42 §7 + operational runbook.

---

## 1. Prerequisites

| Tool | Version | Why |
|---|---|---|
| Node.js | 18+ | NestJS runtime |
| npm | 9+ | Package manager (this repo uses npm) |
| PostgreSQL | 13+ | Persistence |
| `kaha-main-api-v3` | running | Required for user/business validation at checkout |

---

## 2. Local Setup

```bash
cd C:/Users/ishwor/Music/own-organize/arju/restaurant-ecommerce
npm install
cp .env.example .env        # see §3
# start a postgres instance (docker-compose.yml provided)
docker-compose up postgres -d
npm run start:dev
```

> ℹ️ **Visualize the schema:** open `schema.dbml` from the repo root and paste it into [dbdiagram.io](https://dbdiagram.io). This is the fastest way to onboard to the data model — it *is* the design (ADR-E05).

> ℹ️ **Backbone dependency:** browsing the catalog works standalone, but **checkout will fail** without `kaha-main-api-v3` reachable at `KAH_API_V3_BASE_URL` — it validates user/business there (ADR-E02).

---

## 3. Environment Variables

> ⚠️ **`JWT_SECRET_TOKEN` must equal the backbone's** (shared platform secret).

| Variable | Purpose |
|---|---|
| `APP_PORT` | Service port |
| `DB_HOST` / `DB_PORT` / `DB_NAME` / `DB_USER_NAME` / `DB_PASSWORD` | PostgreSQL |
| `JWT_SECRET_TOKEN` | Shared platform secret |
| `KAH_API_V3_BASE_URL` | Backbone base URL for user/business validation |

---

## 4. Operations Cheat-Sheet

| Symptom | Likely cause | Action |
|---|---|---|
| Checkout 500s, browsing fine | Backbone unreachable | Check `KAH_API_V3_BASE_URL` + backbone health (ADR-E02) |
| Old order shows new price/name | Code joined live catalog instead of snapshots | Bug — read `*_snapshot` columns (ADR-E01) |
| `current_status` ≠ latest history | Non-transactional status write | Ensure status update + history append in one transaction (ADR-E03) |
| Duplicate cart error | `UNIQUE(user_id, business_id)` violated | Expected — one cart per restaurant (ADR-E04); reuse existing cart |
| Schema diagram wrong | `schema.dbml` drifted from migrations | Update `schema.dbml` to match — drift is a bug (ADR-E05) |
| All calls 401 | JWT secret mismatch | Align `JWT_SECRET_TOKEN` with backbone |

---

## 5. Deployment Notes

- Stateless except its DB; horizontally scalable.
- Not in any other service's critical path — failure degrades *only* ordering.
- Keep `schema.dbml` updated in the same PR as any migration (release checklist item).

---

## 6. Where To Go Next

- The snapshot model you're operating → [data-model.md](data-model.md)
- Why these invariants exist → [decisions.md](decisions.md)
