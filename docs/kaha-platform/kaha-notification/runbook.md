# kaha-notification — Runbook (Setup & Operations)

> ℹ️ **Confluence page placement:** child of *kaha-notification → Overview*.
>
> **Document standard:** arc42 §7 + operational runbook.

---

## 1. Prerequisites

| Tool | Version | Why |
|---|---|---|
| Node.js | 18+ | NestJS 10 runtime |
| Yarn | 1.x | Package manager |
| PostgreSQL | 13+ | Persistence |
| Redis | 6+ | **Hard dependency** — BullMQ queue |
| Firebase project | — | Service-account credentials for FCM |

---

## 2. Local Setup

```bash
cd D:/shared-code/code/notifications-projects/kaha-notification
yarn install
cp .env.example .env        # fill in — see §3
docker-compose up postgres -d
yarn migration:run          # builds + runs against compiled JS (dynamic dev/prod)
yarn start:dev
```

> ℹ️ **Migration script is environment-aware.** `migration:run` uses `typeorm-dynamic`: in `production` it runs against `dist/`, in dev against `src/`. Don't hand-roll the typeorm path — use the yarn scripts.

---

## 3. Environment Variables

> ⚠️ **`JWT_SECRET_TOKEN` must equal the backbone's.** This service validates the JWT the backbone forwards. Mismatch = every call 401s.

| Variable | Purpose |
|---|---|
| `APP_PORT` | Service port (use a different port than backbone if co-located locally) |
| `DB_HOST` / `DB_PORT` / `DB_NAME` / `DB_USER_NAME` / `DB_PASSWORD` | PostgreSQL |
| `JWT_SECRET_TOKEN` | Shared platform-wide secret |
| `REDIS_HOST` / `REDIS_PORT` | BullMQ backing store — **required** |
| `FIREBASE_PROJECT_ID` / `FIREBASE_CLIENT_EMAIL` / `FIREBASE_PRIVATE_KEY` | `firebase-admin` service account |
| `SMTP_HOST` / `SMTP_USER` / `SMTP_PASS` | Nodemailer email |
| `SMS_*` | SMS provider credentials |

> ℹ️ **`FIREBASE_PRIVATE_KEY` newline gotcha:** the key contains `\n`. In `.env` it's usually stored with literal `\n` and the code replaces them — preserve quoting exactly as `.env.example` shows, or FCM auth fails cryptically.

---

## 4. Operations Cheat-Sheet

| Symptom | Likely cause | Action |
|---|---|---|
| Push accepted but never arrives | Worker not running / Redis down | Check Redis, check BullMQ jobs, check worker logs |
| All calls 401 | JWT secret mismatch | Align `JWT_SECRET_TOKEN` with backbone |
| FCM auth error | Bad `FIREBASE_PRIVATE_KEY` formatting | Re-check `\n` escaping vs `.env.example` |
| `notification` rows exist, no `receiver` rows | Token resolution returned none | Device not registered / wrong `userId` from caller |
| Email silently not sent | SMTP creds / fire-and-forget swallow | Check this service's logs (caller won't show it) |
| Queue backing up | Worker throughput < enqueue rate / FCM throttling | Scale workers, inspect BullMQ failed jobs |

---

## 5. Deployment Notes

- Same deployable runs API + BullMQ worker. Scaling workers = scaling the deployable (or running a worker-only mode if separated later).
- Redis must be HA-considered: it is a hard dependency for push.
- Migrations: run `migration:run` (env-aware) as a deploy step.

---

## 6. Where To Go Next

- The queue flow being operated → [architecture.md](architecture.md) §3
- Why queued / why fire-and-forget → [decisions.md](decisions.md)
