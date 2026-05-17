# Hotel System (HMS) — Runbook (Setup & Operations)

> ℹ️ **Confluence page placement:** child of *Hotel System → Overview*.
>
> **Document standard:** arc42 §7 + operational runbook. Covers all 4 repos.

---

## 1. Prerequisites

| Tool | Version | Why |
|---|---|---|
| Node.js | 18+ | NestJS + React |
| npm | 9+ | All 4 repos use npm |
| PostgreSQL | 15 | `hms_db` |
| ngrok (local) | any | Khalti webhook needs a public URL |
| Khalti test creds | — | Payment testing |

---

## 2. Backend — `hms-api`

```bash
cd C:/Users/ishwor/Music/own-organize/hotel/hms-api
npm install
cp .env.example .env        # see §6
docker run -d --name hms-postgres -e POSTGRES_DB=hms_db \
  -e POSTGRES_USER=postgres -e POSTGRES_PASSWORD=password \
  -p 5432:5432 postgres:15
npm run migration:run
node reset-and-seed.js       # optional: seed test data
npm run start:dev
```
API → `http://localhost:3000` · Swagger → `/api/v1/docs` (if enabled)

## 3. Guest Frontend — `hms-frontend`

```bash
cd ../hms-frontend && npm install
echo "VITE_API_URL=http://localhost:3000/api/v1" > .env.local
npm run dev          # → http://localhost:5173
```

## 4. Admin Panel — `hms-admin`

```bash
cd ../hms-admin && npm install
echo "VITE_API_URL=http://localhost:3000/api/v1" > .env.local
npm run dev          # → http://localhost:5174
```

## 5. Property Website — `property-site` (Next.js)

```bash
cd ../property-site && npm install
npm run dev          # → http://localhost:3001
```

---

## 6. Environment Variables (`hms-api`)

> ⚠️ **HMS JWT is independent of the platform.** `JWT_SECRET` here has **no relation** to the platform `JWT_SECRET_TOKEN` (ADR-H05). Don't copy the platform secret expecting cross-validation.

| Variable | Purpose |
|---|---|
| `NODE_ENV` / `PORT` | Env + port (3000) |
| `DB_HOST/PORT/USERNAME/PASSWORD/NAME` | `hms_db` |
| `JWT_SECRET` (≥32 chars) | HMS access token — **HMS-local** |
| `JWT_REFRESH_SECRET` (different) | HMS refresh token |
| `JWT_EXPIRES_IN` / `JWT_REFRESH_EXPIRES_IN` | `15m` / `7d` |
| `EMAIL_USER` / `EMAIL_PASSWORD` | SMTP (email verification) |
| `FRONTEND_URL` | Used in verification links |
| `BACKEND_PUBLIC_URL` | **Public** URL for Khalti webhook |
| `ALLOWED_ORIGINS` | CORS allowlist |
| `KHALTI_SECRET_KEY` | From Khalti dashboard |
| `KHALTI_BASE_URL` | `https://dev.khalti.com/api/v2` (test) / `https://khalti.com/api/v2` (prod) |

---

## 7. Khalti Test Credentials

Test number `9800000000`, MPIN `1111`, OTP `987654`. Test keys: [docs.khalti.com](https://docs.khalti.com).

---

## 8. Operations Cheat-Sheet

| Symptom | Likely cause | Action |
|---|---|---|
| Cross-tenant data appears | Missing `propertyId` filter | **Security incident** — audit the query (ADR-H01) |
| Payment never confirms locally | Khalti webhook can't reach you | `ngrok http 3000`; set `BACKEND_PUBLIC_URL` |
| Double booking on retry | Idempotency key not sent/replayed | Fix client to send stable key (ADR-H03) |
| Old booking shows new rate | Recomputed from live rates | Read booking snapshot, not rate-plan (ADR-H02) |
| Platform JWT rejected by HMS | Expected — separate secret | Use HMS auth; Kaha integrates via service account (ADR-H04/H05) |
| `hms_db does not exist` | DB not created | Use the docker command in §2 |
| Kaha changes not in HMS | Sync is one-way at provision time | Re-trigger `kaha-sync` (ADR-H04) |

---

## 9. Deployment Notes

- 4 deployables: `hms-api` (+ `hms_db`), `hms-frontend`, `hms-admin`, `property-site`.
- `hms-api` scales horizontally (stateless + DB); multi-tenant means capacity is shared across all properties.
- Migrations: `npm run migration:run` as a deploy step.
- Khalti prod cutover = swap `KHALTI_BASE_URL` + prod secret; verify webhook public URL.

---

## 10. Where To Go Next

- The flows being operated → [architecture.md](architecture.md)
- Why these invariants exist → [decisions.md](decisions.md)
