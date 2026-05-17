# Glossary

> ℹ️ **Confluence page placement:** child of *Kaha Platform — Engineering Handover*. Read this before the service docs — it's the shared vocabulary.

Terms are grouped: **Platform & Architecture**, **Business Domain**, **Hospitality**, **Nepal-specific**. Each entry says what it is and links to where it matters.

---

## Platform & Architecture

| Term | Meaning |
|---|---|
| **Backbone** | `kaha-main-api-v3` — the central service owning identity + business data. Everything depends on it. |
| **Satellite service** | A microservice around the backbone (notification, subscription, chat, ecommerce). Owns its own DB. |
| **Database-per-service** | Platform rule: no two services share a database. Cross-service refs are string IDs, never FKs. See [service-architecture](kaha-platform/service-architecture.md). |
| **Shared JWT secret** | All *platform* services validate the same `JWT_SECRET_TOKEN` → stateless auth. **Exception: Hotel HMS uses its own secret** (ADR-H05). |
| **Validate-on-use** | A satellite stores an external `userId`/`businessId` without an FK and checks it exists via HTTP only when needed (e.g. ecommerce at checkout). |
| **Snapshot** | Copying upstream data (price, name, address, tax) onto a record at creation so later upstream edits don't rewrite history. Used in ecommerce orders, hotel bookings. |
| **Fire-and-forget** | The backbone calls notification without awaiting success; failures are logged, never thrown. Lost notifications are *silent* — debug at the notification service. |
| **Token forwarding** | The backbone passes the end-user's JWT to the subscription service so it authorizes as that user. |
| **Proxy + enrich** | The backbone fetches thin data from a satellite (chat) then hydrates it with user/business detail from its own DB. |
| **Service account** | A machine identity (not a person) used for system-to-system auth. Backbone↔CRS and Kaha→Hotel-sync use this. |
| **kaha-sync** | The **inbound** integration where the backbone pushes a user + property into Hotel HMS, stamping records `source='kaha'`. One-directional. |
| **SPOF** | Single Point of Failure. The backbone is one by design — see [service-architecture §5](kaha-platform/service-architecture.md). |
| **ADR** | Architecture Decision Record — the Context→Decision→Alternatives→Consequences format used in every `decisions.md`. |
| **C4 model** | Architecture diagram standard: L1 Context → L2 Container → L3 Component. Used across the architecture pages. |
| **Modular monolith** | The backbone: 50+ modules in one deployable (not internal microservices). See [kaha-main-api ADR-001](kaha-platform/kaha-main-api/decisions.md). |

---

## Business Domain

| Term | Meaning |
|---|---|
| **Business** | The core domain object in the backbone — a listed entity (restaurant, hotel, shop). Self-referential (parent/branch, located-in). |
| **kahaId** | The public, phone-derived unique identifier for a user/business. The internal UUID PK stays private. See [kaha-main-api ADR-006](kaha-platform/kaha-main-api/decisions.md). |
| **Claim / Entity Claim** | The workflow where a user proves ownership of a business (uploads registration + PAN/VAT certs) → admin approves → triggers subscription. |
| **Business User** | The join of *user + role + business* — "User X is a Manager at Business Y". The access-control pivot. |
| **Tier** | A subscription plan (Free, Premium…) priced per business category. |
| **Feature** | A gated capability. `isCountable` = metered (has a usage limit) vs a boolean capability. |
| **Tier-Feature** | The catalogue join: "this tier includes this feature with this `usagesLimit`". |
| **Effective limit** | The limit actually enforced = per-business `customUsageLimit` if a customization row exists, **else** the tier-feature default. The heart of subscription. |
| **Feature usage** | A counter (`subscription-usage`) incremented each time a business uses a countable feature; compared to the effective limit. |
| **Free-tier-by-category** | On claim approval the backbone asks subscription for the free tier matching the business's category. Every category must have one (operational invariant). |
| **Topic / Topic subscription** | Notification pub/sub: users subscribe to a topic; a push to the topic reaches all subscribers. |
| **Notification device** | A registered FCM token tied to a user/platform; created on login, deleted on logout. |

---

## Hospitality (Hotel & Hostel)

| Term | Meaning |
|---|---|
| **HMS** | Hotel Management System — the self-contained hotel product (`hms-api` + 3 frontends). |
| **Property** | The tenant root in HMS — one hotel. *Every* HMS table is scoped by `propertyId`. |
| **Property Member** | The join of *user + role + property* — multi-tenant RBAC in HMS (analogous to Business User). |
| **Rate plan** | HMS pricing rules: cancellation policy, inclusions (breakfast/wifi), min/max nights. |
| **Daily rate** | Per-date rate override on top of a room-type's base price (seasonal/event pricing). |
| **Rate snapshot** | A booking copies `ratePlanName` + `baseRate` + per-day `dailyBreakdown[]` at creation so later rate edits don't change confirmed bookings. |
| **Idempotency key** | A token making a booking/payment submission safe to retry without double-booking/double-charging (HMS ADR-H03). |
| **Bed** | The atomic bookable unit in **Hostel** (not the room). Has its own `monthlyRate`, `status`, occupant. |
| **Hostel vs Hotel** | Hostel = bed-level, student residents, **monthly** billing. Hotel = room-level, short-stay guests, **per-night** billing. Different domains — don't assume patterns transfer. |
| **Booking-guest** | Hostel bridge entity: an applicant on a booking who *becomes* a student once approved. |
| **Student** | A hostel long-stay resident (linked to a bed, a booking-guest, and a kaha-main contact person). |
| **Ledger** | Hostel per-student running financial account (debit/credit rows) — the audit truth, separate from invoices/payments. |
| **Attendance** | Daily per-student presence record in Hostel (`firstCheckIn`/`lastCheckOut`). |
| **CRS** | Central Reservation System — a **3rd-party** hotel reservation system the backbone proxies via a service account (kaha-main ADR-004). |

---

## Nepal-specific

| Term | Meaning |
|---|---|
| **Khalti** | A Nepal payment gateway. Hotel HMS integrates it (test vs prod base URLs; webhook needs a public URL). |
| **Tolet / To-Let** | "To Let" — room/flat-for-rent listings (the Nepal rental market). A backbone module (`tolet`, `general-tolets`). |
| **PAN / VAT certificate** | Nepal business tax registration documents — uploaded during the business claim flow as ownership proof. |
| **NPR** | Nepalese Rupee — default currency (e.g. ecommerce `currency char(3) default 'NPR'`). |
| **kaha.com.np** | Platform domain. `dev.kaha.com.np` is the dev environment / default hostel API URL. |
| **B.S. (Bikram Sambat)** | The Nepali calendar (e.g. "Established 2074 B.S."). Appears in association/business data. |

---

## How to extend this glossary

Add a term when a reviewer or new joiner had to ask "what does X mean?". Keep entries one line where possible; link to the ADR or doc where the term is load-bearing rather than re-explaining it here.
