# Onboarding — Day 1 / Week 1

> ℹ️ **Confluence page placement:** child of *Kaha Platform — Engineering Handover*. This is the "how to start" guide — a concrete reading + setup path, not a reference.

The handover is large. This page tells you **what to read, in what order, and what to do** — by role. Don't read everything; follow your track.

---

## Day 1 — Everyone (≈ half a day)

Goal: understand the shape of the system before any code.

1. [`index.md`](index.md) — the system map + repo directory (15 min)
2. [`GLOSSARY.md`](GLOSSARY.md) — skim all, you'll re-read as needed (15 min)
3. [`kaha-platform/service-architecture.md`](kaha-platform/service-architecture.md) — **the most important page.** The 4 integration patterns + the failure cheat-sheet. Read it twice (45 min)

> ⚠️ **The one thing to internalize Day 1:** the backbone is a single point of failure, services don't share databases, and the platform shares one JWT secret — *except Hotel HMS, which doesn't*. Most cross-codebase mistakes come from forgetting this.

By end of Day 1 you should be able to draw the system map from memory and explain what happens when each service goes down.

---

## Week 1 — By Role

Each track: read in order, then do the setup, then the first task. Every service's `runbook.md` has the setup steps and an ops cheat-sheet.

### 🔧 Backbone / Platform Engineer

| Day | Read | Do |
|---|---|---|
| 2 | kaha-main-api: [overview](kaha-platform/kaha-main-api/overview.md) → [architecture](kaha-platform/kaha-main-api/architecture.md) | Run backbone locally ([runbook](kaha-platform/kaha-main-api/runbook.md)) |
| 3 | kaha-main-api: [data-model](kaha-platform/kaha-main-api/data-model.md) → [decisions](kaha-platform/kaha-main-api/decisions.md) (all 6 ADRs) | Trace the claim→subscription flow in code |
| 4 | [service-architecture](kaha-platform/service-architecture.md) again, now with code open | Run backbone + notification + subscription together |
| 5 | The satellite you'll touch first | **First task:** a read-only endpoint or a bugfix in one module |

### 🔔 Notifications Engineer

| Day | Read | Do |
|---|---|---|
| 2 | kaha-notification: [overview](kaha-platform/kaha-notification/overview.md) → [architecture](kaha-platform/kaha-notification/architecture.md) | Run it locally + Redis ([runbook](kaha-platform/kaha-notification/runbook.md)) |
| 3 | [data-model](kaha-platform/kaha-notification/data-model.md) → [decisions](kaha-platform/kaha-notification/decisions.md) | Send a test push end-to-end; watch the BullMQ job |
| 4 | kaha-main-api [service-communication] section in its [architecture](kaha-platform/kaha-main-api/architecture.md) | Trace a push from backbone call → receiver status |
| 5 | — | **First task:** add/adjust a notification type; verify the 4-step debug path |

### 💳 Billing / Subscription Engineer

| Day | Read | Do |
|---|---|---|
| 2 | kaha-subscription: [overview](kaha-platform/kaha-subscription/overview.md) → [architecture](kaha-platform/kaha-subscription/architecture.md) | Run it locally ([runbook](kaha-platform/kaha-subscription/runbook.md)); seed tiers per category |
| 3 | [data-model](kaha-platform/kaha-subscription/data-model.md) → [decisions](kaha-platform/kaha-subscription/decisions.md) | Trace effective-limit resolution (customization vs tier) |
| 4 | Backbone claim→subscription flow ([architecture §4](kaha-platform/kaha-main-api/architecture.md)) | Run backbone + subscription; do a claim end-to-end |
| 5 | — | **First task:** add a feature to a tier; verify gating |

### 🛒 E-commerce Engineer

| Day | Read | Do |
|---|---|---|
| 2 | restaurant-ecommerce: [overview](kaha-platform/kaha-ecommerce/overview.md) → [architecture](kaha-platform/kaha-ecommerce/architecture.md) | Open `schema.dbml` in dbdiagram.io; run locally |
| 3 | [data-model](kaha-platform/kaha-ecommerce/data-model.md) → [decisions](kaha-platform/kaha-ecommerce/decisions.md) | Trace cart→order; find every `*_snapshot` write |
| 4 | The snapshot ADRs (E01/E03) | Run backbone + ecommerce; place an order |
| 5 | — | **First task:** a catalog or cart change — **never** touch order snapshots |

### 🏨 Hotel Engineer

| Day | Read | Do |
|---|---|---|
| 2 | Hotel: [overview](hotel/overview.md) → [architecture](hotel/architecture.md) | Run `hms-api` + one frontend ([runbook](hotel/runbook.md)) |
| 3 | [data-model](hotel/data-model.md) → [decisions](hotel/decisions.md) (esp. **H01 multi-tenant**, **H05 own-JWT**) | Make a booking; follow the rate snapshot |
| 4 | Khalti flow ([architecture §4](hotel/architecture.md)) | Wire ngrok; complete a test Khalti payment |
| 5 | — | **First task:** a property-scoped feature — **every query filters by `propertyId`** |

### 🏠 Hostel Engineer

| Day | Read | Do |
|---|---|---|
| 2 | Hostel: [overview](hostel/overview.md) (esp. Hotel-vs-Hostel table) → [architecture](hostel/architecture.md) | Run `hostel-server` + frontend ([runbook](hostel/runbook.md)) |
| 3 | [data-model](hostel/data-model.md) → [decisions](hostel/decisions.md) (esp. **HO01 bed-as-unit**, **HO03 ledger**) | Trace public-booking → student creation |
| 4 | external/kaha integration | Run backbone + hostel; book with a new contact phone |
| 5 | — | **First task:** a bed/availability feature — reason in **beds, not rooms** |

---

## The Five Mistakes New Joiners Make

> ⚠️ Read this list now and again at end of Week 1.

1. **Assuming the shared JWT works in Hotel HMS.** It doesn't — HMS has its own secret (Hotel ADR-H05).
2. **Joining live tables for historical data.** Orders/bookings use `*_snapshot` columns — joining live catalog rewrites financial history.
3. **Forgetting the tenant filter.** Missing `propertyId`/`hostelId` in a query is a *security incident*, not a bug.
4. **Debugging missing pushes in the backbone.** Fire-and-forget swallows them — the evidence is in kaha-notification + BullMQ.
5. **Reasoning about hostel in rooms.** The atomic unit is the **bed**.

---

## Environment Setup Checklist (any backend)

- [ ] Node 18+, the repo's package manager (yarn or npm — check the repo)
- [ ] Docker (PostgreSQL; PostGIS image for backbone/notification)
- [ ] Redis (required for kaha-notification)
- [ ] `.env` from `.env.example` — **align `JWT_SECRET_TOKEN`** across platform services (not HMS)
- [ ] Run migrations via the repo's `migration:run` script (env-aware in some repos)
- [ ] For full flows, run the backbone + the satellite together

Per-service exact steps + troubleshooting: each service's `runbook.md`.

---

## When You're Stuck

1. The service's `runbook.md` **Operations Cheat-Sheet** — symptom → cause → action.
2. The service's `decisions.md` — most "why is it like this?" is an ADR.
3. [`service-architecture.md`](kaha-platform/service-architecture.md) §5 — failure behavior across services.
4. [`GLOSSARY.md`](GLOSSARY.md) — unfamiliar term.
