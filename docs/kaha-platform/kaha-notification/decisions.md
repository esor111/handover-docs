# kaha-notification — Architecture Decision Records (ADRs)

> ℹ️ **Confluence page placement:** child of *kaha-notification → Overview*.
>
> **Document standard:** ADR / MADR format. Service-specific decisions; platform-wide ADRs live in [../kaha-main-api/decisions.md](../kaha-main-api/decisions.md).

---

## ADR-N01 — Push delivery is queued (BullMQ), not synchronous

**Status:** Accepted

**Context.** Firebase multicast can be slow or briefly unavailable. The backbone calls this service in the request path of user actions (login, booking).

**Decision.** Persist + enqueue the push, return `200 accepted` immediately, dispatch to Firebase from a BullMQ worker.

**Alternatives considered.**
- Synchronous FCM send in the HTTP handler → rejected: ties user action latency to Firebase latency.
- External queue (SQS/Kafka) → rejected: Redis is already required; BullMQ is sufficient and lower-ops.

**Consequences.**
- ✅ Decoupled latency; built-in retry/backoff via BullMQ.
- ⚠️ "Accepted" ≠ "delivered". Delivery truth is `NOTIFICATION_RECEIVER.status`, not the HTTP response.
- ⚠️ No Redis = no push. Redis is a hard dependency, not a cache nicety.

---

## ADR-N02 — One notification, many receiver rows

**Status:** Accepted

**Context.** A single broadcast can target thousands of users; the platform needs per-recipient delivery accountability.

**Decision.** Model `NOTIFICATION` (the message) separately from `NOTIFICATION_RECEIVER` (one row per recipient, with `status`).

**Alternatives considered.**
- Single row with a recipients array + aggregate status → rejected: can't answer "did *this* user get it?"; array mutation contention.

**Consequences.**
- ✅ Per-recipient delivery state and partial-failure reporting.
- ⚠️ Row volume grows with audience size — receiver table is the one to watch for growth/archival.

---

## ADR-N03 — Device registry decoupled from delivery history

**Status:** Accepted

**Context.** FCM tokens are volatile (reinstall, logout, OS refresh). Delivery history must be permanent.

**Decision.** `NOTIFICATION_DEVICE` is an independent table. Logout deletes the device row; notification/receiver history is untouched.

**Consequences.**
- ✅ Token churn never corrupts historical delivery data.
- ⚠️ A push enqueued just before logout may target a token that no longer exists → handled as a normal FCM failure on the receiver row.

---

## ADR-N04 — This service trusts, never validates, external IDs

**Status:** Accepted (consequence of platform ADR-002)

**Context.** `userId` / `businessId` originate in the backbone.

**Decision.** Store them as opaque indexed strings. Do not call back to validate them.

**Consequences.**
- ✅ Zero coupling back to the backbone; this service stays a pure sink.
- ⚠️ Garbage-in is possible — the *caller* (backbone) is responsible for ID correctness.

---

## How to add a new ADR

Use **Context → Decision → Alternatives → Consequences**, next number (`ADR-N0X`), status `Proposed` → `Accepted`. Never rewrite an accepted decision — supersede it.
