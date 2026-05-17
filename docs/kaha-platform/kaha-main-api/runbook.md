# kaha-main-api-v3 — Runbook (Setup & Operations)

> ℹ️ **Confluence page placement:** child of *kaha-main-api-v3 → Overview*.
>
> **Document standard:** arc42 §7 (Deployment View) + operational runbook.

---

## 1. Prerequisites

| Tool | Version | Why |
|---|---|---|
| Node.js | 18+ | NestJS runtime |
| Yarn | 1.x | Package manager used by this repo |
| Docker + Compose | latest | Postgres+PostGIS, optional full-stack |
| PostgreSQL | via `postgis/postgis` image only | Spatial columns need PostGIS |
| Redis | 6+ | Cache layer |

---

## 2. Local Setup

```bash
git clone https://github.com/kaha-app/kaha-main-api-v3.git
cd kaha-main-api-v3
yarn install
cp .env.example .env        # then fill in — see §3
```

Start the database (PostGIS on port **5434**, intentionally not 5432):

```bash
docker-compose up postgres -d
```

Run migrations (must build first — migrations run against compiled JS):

```bash
yarn build
yarn typeorm migration:run -d dist/database/data-source.js
```

Start the API:

```bash
yarn start:dev          # watch mode
# or
yarn build && yarn start:prod
```

API → `http://localhost:3006`

> ℹ️ **Full platform locally:** also start `kaha-notification` and `kaha-business-subscription` (each its own repo/compose) and point this service's `NOTIFICATION_URL` / `SUBSCRIPTION_URL` at them. The service runs fine without them — those calls just fail-soft (notification) or 500 on gated actions (subscription).

---

## 3. Environment Variables

> ⚠️ **`JWT_SECRET_TOKEN` must be byte-identical across `kaha-main-api-v3`, `kaha-notification`, `kaha-business-subscription`, `restaurant-ecommerce`.** A mismatch makes every cross-service call return 401. (See [decisions.md](decisions.md) ADR-002.)

| Variable | Purpose | Notes |
|---|---|---|
| `APP_PORT` | API port | Default 3006 |
| `DB_HOST` / `DB_PORT` / `DB_NAME` / `DB_USER_NAME` / `DB_PASSWORD` | PostgreSQL | Port 5434 in local compose |
| `JWT_SECRET_TOKEN` | JWT signing/validation | **Shared platform-wide** |
| `NOTIFICATION_URL` | → kaha-notification | Fire-and-forget target |
| `SUBSCRIPTION_URL` | → kaha-business-subscription | Sync, token-forwarded |
| `CHAT_URL` | → chat service | Proxy+enrich target |
| `EV_API_URL` | → EV charging API | |
| `HOSTEL_API_URL` | → hostel API | Defaults to `https://dev.kaha.com.np` |
| `CRS_BASE_URL` | → Hotel CRS | + `CRS_SERVICE_ACCOUNT_*` |
| `CRS_SERVICE_ACCOUNT_TOKEN` | CRS machine auth | Or phone/password fallback vars |
| `CRS_TOKEN_REFRESH_THRESHOLD_MS` | Auto-refresh window | Default 300000 (5 min) |
| `CRS_REQUEST_TIMEOUT_MS` | CRS call timeout | Default 30000 |
| `AWS_REGION` / `AWS_BUCKET_NAME` / `AWS_COMPRESSED_BUCKET_NAME` / `AWS_ACCESS_KEY` / `AWS_SECRET_KEY` | S3 uploads | `compressedBucketName` defaults to `compressedv2` |
| `REDIS_HOST` / `REDIS_PORT` | Redis cache | |
| `LOKI_URL` | Log shipping | |
| `KAHA_API_LINK` | Self-reference base URL | Used for self-referential links |

---

## 4. Operations Cheat-Sheet

| Symptom | Likely cause | Action |
|---|---|---|
| Every cross-service call 401s | `JWT_SECRET_TOKEN` mismatch | Align secret across all services, redeploy together |
| Migration fails: `uuid_generate_v4 does not exist` | Missing extension | `CREATE EXTENSION IF NOT EXISTS "uuid-ossp";` |
| Migration fails: spatial type unknown | Plain postgres image | Use `postgis/postgis` image |
| `port 5434 already in use` | Stale postgres container | Stop it or change compose port |
| Hotel endpoints hang ~30s then fail | CRS down | Expected — 30s timeout (ADR-004); check CRS, not this service |
| "Missing push notification" | Fire-and-forget swallowed it | Debug at **kaha-notification + BullMQ**, not here (ADR-005) |
| Redis connection refused | Redis not running | `docker run -d -p 6379:6379 redis` |
| Feature-gated action 500s | Subscription service down/unreachable | Check `SUBSCRIPTION_URL` + that service |

---

## 5. Deployment Notes

- **CI/CD:** GitHub Actions → Docker image build → deploy (workflow in `.github/`).
- **Dev environment:** `https://dev.kaha.com.np`.
- **DB migrations are not auto-run on deploy by default** — run `migration:run` against compiled `dist` as a deploy step.
- **Scaling priority:** this is the SPOF (ADR-001). Horizontal scaling + DB read replicas here before any satellite.

---

## 6. Where To Go Next

- Why these operational constraints exist → [decisions.md](decisions.md)
- Platform-wide failure behavior → [`../service-architecture.md`](../service-architecture.md) §5
