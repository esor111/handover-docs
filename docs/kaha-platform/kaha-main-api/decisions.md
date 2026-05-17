# kaha-main-api-v3 — Architecture Decision Records (ADRs)

> ℹ️ **Confluence page placement:** child of *kaha-main-api-v3 → Overview*.
>
> **Document standard:** [ADR / MADR format](https://adr.github.io/) — the industry-standard way to record *why* a decision was made, the alternatives, and the consequences. Read these to make future decisions consistent with the existing system.

Each ADR: **Context → Decision → Alternatives → Consequences**. Status is `Accepted` unless noted.

---

## ADR-001 — Modular monolith backbone, not full microservices

**Status:** Accepted

**Context.** The platform needs identity + business directory + orchestration. A full microservice split (separate user-service, business-service, etc.) was possible.

**Decision.** Keep identity and business directory together as one modular-monolith deployable (`kaha-main-api-v3`, 50+ modules). Split out only *peripheral* concerns (notification, subscription, chat, ecommerce) as satellites.

**Alternatives considered.**
- Full microservices → rejected: cross-service joins for every "user + their business" call; distributed-transaction complexity; team size doesn't justify it.
- Single monolith including notification/billing → rejected: those have different scaling and failure profiles.

**Consequences.**
- ✅ One source of truth for the most-joined data; simple local dev.
- ⚠️ This service is a **platform-wide single point of failure** (see [overview.md](overview.md) §3). HA investment must prioritize it.
- ⚠️ The monolith will keep growing — module-group discipline ([architecture.md](architecture.md) §2) is the mitigation.

---

## ADR-002 — Database-per-service; cross-service references are string IDs

**Status:** Accepted (platform-wide rule)

**Context.** Satellites need to reference users/businesses owned here.

**Decision.** Each service owns its own PostgreSQL. No shared DB. Cross-service references are stored as **plain indexed string columns**, never foreign keys. Existence is validated over HTTP at write time (see `restaurant-ecommerce`).

**Alternatives considered.**
- Shared database → rejected: couples deploys and schema evolution; defeats service independence.
- Sync replication of user/business data into each service → rejected: stale data + complexity.

**Consequences.**
- ✅ Services deploy and evolve schemas independently.
- ⚠️ No referential integrity across services — orphaned references are possible; validate at boundaries.
- ⚠️ No cross-service `JOIN` — data composition happens in application code via HTTP (the proxy+enrich pattern).

---

## ADR-003 — Chat is proxied and enriched, not duplicated

**Status:** Accepted

**Context.** Chat channels need to display user names, avatars, and business info. The chat service only stores IDs.

**Decision.** `kaha-main-api-v3` fetches raw channels from the chat service, then hydrates them from its own `UserRepository` / `BusinessRepository` before returning to the client.

**Alternatives considered.**
- Chat service stores its own copy of user/business display data → rejected: violates single source of truth; sync problems.
- Client makes a second call to enrich → rejected: pushes N+1 to the client; leaks the architecture.

**Consequences.**
- ✅ One source of truth preserved; chat stays thin.
- ⚠️ N+1 lookups on channel listing — mitigated with `Promise.all` batching.
- ⚠️ Chat is **unusable without the backbone** (no display data on its own).

---

## ADR-004 — Hotel CRS accessed via a single service account with token auto-refresh

**Status:** Accepted

**Context.** Hotel reservations live in a 3rd-party Central Reservation System that has no concept of Kaha users.

**Decision.** Authenticate to the CRS as one machine identity (service account). Token auto-refreshes at a 5-minute-to-expiry threshold (`CRS_TOKEN_REFRESH_THRESHOLD_MS=300000`) with a phone/password re-login fallback. Every CRS request has a 30s timeout (`CRS_REQUEST_TIMEOUT_MS`).

**Alternatives considered.**
- Per-user CRS accounts → impossible: CRS doesn't know Kaha users.
- No timeout / no auto-refresh → rejected: a hung CRS would cascade into platform-wide hangs.

**Consequences.**
- ✅ Uniform hotel access; refresh is transparent to callers.
- ⚠️ CRS outage = **all** hotel reservations fail platform-wide. The 30s timeout is the deliberate blast-radius limiter.

---

## ADR-005 — Notification calls are fire-and-forget

**Status:** Accepted

**Context.** Core user actions (login, booking, logout) trigger push/email/SMS via the notification service.

**Decision.** All notification calls are wrapped in `try/catch` that logs but **never rethrows**. Logout's `deleteNotificationDevice` additionally has a 5s timeout and explicitly suppresses errors.

**Alternatives considered.**
- Synchronous, failure-propagating notifications → rejected: FCM/email downtime would break login and bookings.
- Local outbox/queue in the backbone → rejected: that responsibility belongs to the notification service's own BullMQ.

**Consequences.**
- ✅ Core flows are resilient to notification outages.
- ⚠️ Notification loss is **silent**. Debugging "missing push" must start at the notification service + its BullMQ queue, **not** the backbone logs.

---

## ADR-006 — `kahaId` is the external identifier, not the UUID primary key

**Status:** Accepted

**Context.** Entities need a public identifier for cross-service references and client use.

**Decision.** Keep the internal `uuid` PK private. Expose `kahaId` (unique, phone-derived) as the shareable identifier for users/businesses.

**Consequences.**
- ✅ Internal keys can change without breaking external consumers; fits Nepal phone-first identity.
- ⚠️ Two identifiers per entity — code must be clear which is in play at each boundary.

---

## How to add a new ADR

When you make a significant architectural decision: copy the **Context → Decision → Alternatives → Consequences** structure, give it the next number, set status `Proposed` until agreed, then `Accepted`. Never edit an accepted ADR's decision — supersede it with a new ADR that references the old one (`Superseded by ADR-0XX`).
