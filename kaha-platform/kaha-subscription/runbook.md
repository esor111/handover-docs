# kaha-business-subscription — Runbook (Setup & Operations)

> ℹ️ **Confluence page placement:** child of *kaha-business-subscription → Overview*.
>
> **Document standard:** arc42 §7 + operational runbook.

---

## 1. Prerequisites

| Tool | Version | Why |
|---|---|---|
| Node.js | 18+ | NestJS 10 |
| Yarn | 1.x | Package manager |
| PostgreSQL | 13+ | Persistence |

No Redis/Firebase — this service is pure relational logic.

---

## 2. Local Setup

```bash
cd D:/shared-code/code/kaha-business-subscription/kaha-business-subscription
yarn install
cp .env.example .env        # see §3
# start a postgres instance
yarn migration:run          # env-aware (typeorm-dynamic): dist in prod, src in dev
yarn start:dev
```

> ℹ️ **Same env-aware migration pattern as kaha-notification.** Use the yarn `migration:*` scripts; don't invoke typeorm directly or you'll target the wrong data-source for the environment.

> ⚠️ **Operational invariant:** every business category must have a defined **free tier** (ADR-S04). Seed tiers (with `categoryId`) before testing the claim flow, or business approval in the backbone will fail at subscription creation.

---

## 3. Environment Variables

> ⚠️ **`JWT_SECRET_TOKEN` must equal the backbone's** — this service validates the user JWT the backbone forwards (`@nestjs/jwt`).

| Variable | Purpose |
|---|---|
| `APP_PORT` | Service port (distinct from backbone/notification if co-located) |
| `DB_HOST` / `DB_PORT` / `DB_NAME` / `DB_USER_NAME` / `DB_PASSWORD` | PostgreSQL |
| `JWT_SECRET_TOKEN` | Shared platform-wide secret |

Minimal env — the small surface is intentional (ADR: pure relational service).

---

## 4. Operations Cheat-Sheet

| Symptom | Likely cause | Action |
|---|---|---|
| Gated business action 500s | This service down/unreachable | Check `SUBSCRIPTION_URL` from backbone + this service health |
| Business approved but no subscription | No free tier for that `categoryId` | Seed the category's free tier (ADR-S04 invariant) |
| Override ignored | `subscription.allowCustomization = false` | Flip the flag, or fix the data — overrides are gated by it (ADR-S02) |
| Quota wrong | Read tier-feature alone, ignored customization | Compute *effective* limit: customization else tier-feature |
| All calls 401 | JWT secret mismatch | Align `JWT_SECRET_TOKEN` with backbone |
| Tier change not reflected | Looking at `subscription-request`, not `subscription` | Request ≠ active plan (ADR-S05) |

---

## 5. Deployment Notes

- Stateless except its DB; horizontally scalable.
- In the critical path of gated features (ADR-S03) — size availability accordingly, but it is *not* the platform SPOF (only gated actions degrade).
- Migrations: `yarn migration:run` (env-aware) as a deploy step.
- Seeding tiers/features per category is a **release prerequisite**, not optional data.

---

## 6. Where To Go Next

- Effective-limit logic being operated → [architecture.md](architecture.md) §3
- Why the invariants exist → [decisions.md](decisions.md)
