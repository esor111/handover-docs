# Hotel System (HMS) — Architecture Decision Records (ADRs)

> ℹ️ **Confluence page placement:** child of *Hotel System → Overview*.
>
> **Document standard:** ADR / MADR. HMS is a self-contained product — its decisions are independent of the platform ADRs, with one explicit divergence (ADR-H05).

---

## ADR-H01 — Shared-schema multi-tenancy keyed by `propertyId`

**Status:** Accepted

**Context.** One HMS deployment serves many hotels.

**Decision.** Single database, single schema; every core table carries `propertyId`; all queries are property-scoped.

**Alternatives considered.**
- Database-per-tenant → rejected: ops/migration cost scales with hotel count.
- Schema-per-tenant → rejected: same migration-fan-out problem.

**Consequences.**
- ✅ Cheap to operate, one migration path.
- ⚠️ **Cross-tenant data leakage is the #1 risk.** Every repository method must filter by `propertyId`. A missing filter = a security incident, not a bug.

---

## ADR-H02 — Rate snapshot on booking

**Status:** Accepted

**Context.** Rate plans and daily rates change; a confirmed booking's price must not.

**Decision.** Snapshot `ratePlanName`, `baseRate`, and per-day `dailyBreakdown[]` onto the booking at creation.

**Consequences.**
- ✅ Confirmed bookings are price-immutable; disputes resolvable.
- ⚠️ Never recompute an existing booking's price from live rates — read the snapshot.

---

## ADR-H03 — Idempotency on booking & payment submission

**Status:** Accepted

**Context.** Network retries / double-clicks can double-book a room or double-charge a card.

**Decision.** Dedicated `idempotency` module + `idempotency-key` entity guard booking and payment writes.

**Consequences.**
- ✅ Safe retries; no phantom double-bookings.
- ⚠️ Clients must send/replay the idempotency key correctly — document it in the API contract.

---

## ADR-H04 — `kaha-sync` is inbound; Kaha is a service account

**Status:** Accepted

**Context.** A Kaha business may need a corresponding HMS property.

**Decision.** `kaha-main-api-v3` calls HMS's `kaha-sync` to push a user + property (transactionally, stamped `source='kaha'`). Kaha authenticates as a service account holding roles `['external-service','kaha-integration']`.

**Alternatives considered.**
- HMS pulls from Kaha → rejected: HMS shouldn't know Kaha's internal model; push keeps HMS the system of record.
- Shared DB → rejected: violates separation; HMS has its own `hms_db`.

**Consequences.**
- ✅ HMS stays authoritative for hotel data; clear trust boundary via the service-account role.
- ⚠️ Sync is one-directional — changes made *in Kaha* after provisioning don't auto-propagate unless re-synced.

---

## ADR-H05 — HMS uses its own JWT secret (divergence from platform)

**Status:** Accepted — **explicit divergence**

**Context.** The Kaha platform standard is one shared `JWT_SECRET_TOKEN` across services.

**Decision.** HMS uses its **own** `JWT_SECRET` + separate `JWT_REFRESH_SECRET`, independent of the platform secret. Kaha↔HMS trust is via the service-account mechanism (ADR-H04), not a shared signing key.

**Consequences.**
- ✅ HMS is a self-contained product, sellable/deployable without the platform; blast radius of a secret rotation is HMS-only.
- ⚠️ A platform JWT is **not** valid in HMS and vice-versa. Engineers moving between codebases must not assume the shared-secret model here. This is the most common cross-codebase mistake — flagged in [overview.md](overview.md) and [runbook.md](runbook.md).

---

## ADR-H06 — Guest lifetime stats are denormalized

**Status:** Accepted

**Context.** Loyalty tiers and reports need per-guest totals frequently.

**Decision.** Maintain `totalStays`, `totalNights`, `totalSpend` on `guest`, updated on booking completion.

**Consequences.**
- ✅ O(1) loyalty/report reads.
- ⚠️ Must be updated transactionally with booking completion or it drifts; treat a reconciliation job as future work.

---

## How to add a new ADR

**Context → Decision → Alternatives → Consequences**, next number (`ADR-H0X`), `Proposed` → `Accepted`. Supersede, never rewrite.
