# Hostel System — Runbook (Setup & Operations)

> ℹ️ **Confluence page placement:** child of *Hostel System → Overview*.
>
> **Document standard:** arc42 §7 + operational runbook.

---

## 1. Prerequisites

| Tool | Version | Why |
|---|---|---|
| Node.js | 18+ | NestJS + React |
| npm | 9+ | Package manager |
| PostgreSQL | 14+ | Persistence |
| Docker | any | Test database |
| `kaha-main-api-v3` | reachable | `find-or-create` contact users |

---

## 2. Backend — `hostel-server`

```bash
cd D:/shared-code/code/oh-hostel/hostel-server/hostel-world-class-backend
npm install
cp .env.example .env        # see §4
# start postgres (14+)
npm run migration:run        # check package.json for exact script name
npm run start:dev
```

## 3. Management Frontend — `frontend-hostel-world`

```bash
cd D:/shared-code/code/frontend-world-hostel/new-hostel-management-system
npm install
echo "VITE_API_URL=http://localhost:3002" > .env.local   # match backend port
npm run dev
```

> ℹ️ See `FEATURE_TIER_SYSTEM.md` in the frontend repo — premium features are tier-gated; behavior differs by the hostel's plan.

---

## 4. Environment Variables (backend)

| Variable | Purpose |
|---|---|
| `APP_PORT` / `PORT` | Service port |
| `DB_*` | PostgreSQL connection |
| `JWT_SECRET*` | Hostel-local auth |
| `KAHA_MAIN_API_URL` | → kaha-main (`/users/find-or-create/:phone`, `/businesses/:id`) |
| `KAHA_NOTIFICATION_API_URL` | → kaha-notification |

> ℹ️ External URLs are read centrally (`getExternalApiUrls()` in `src/external`). Configure them there/via env — don't hardcode in feature modules (ADR-HO05).

---

## 5. Operations Cheat-Sheet

| Symptom | Likely cause | Action |
|---|---|---|
| Cross-hostel data appears | Missing `hostelId` filter | **Security incident** — audit query (ADR-HO06) |
| Booking fails for new contact | kaha-main unreachable | Check `KAHA_MAIN_API_URL`; `find-or-create` needs it (ADR-HO04) |
| Student balance wrong | Ledger not written with source doc | Reconcile ledger; ensure transactional writes (ADR-HO03) |
| "Room full" but beds free | Reasoned at room level | Use bed-level availability (ADR-HO01) |
| Approved applicant has no student record | `booking-guest → student` step skipped | Trigger the transition on approval (ADR-HO02) |
| Notifications not sent | external/notifications misconfigured | Check `KAHA_NOTIFICATION_API_URL` + that service |

---

## 6. Deployment Notes

- Deployables: `hostel-server` (+ Postgres), `frontend-hostel-world`, `individual-hostel-web`.
- Multi-tenant: capacity shared across all hostels (ADR-HO06).
- Depends on kaha-main for new-contact bookings — factor that into availability/SLOs.
- Migrations: run the repo's migration script as a deploy step (verify name in `package.json`).

---

## 7. Where To Go Next

- The flows being operated → [architecture.md](architecture.md)
- Why these invariants exist → [decisions.md](decisions.md)
