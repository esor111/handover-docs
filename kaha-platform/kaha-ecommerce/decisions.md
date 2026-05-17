# restaurant-ecommerce — Architecture Decision Records (ADRs)

> ℹ️ **Confluence page placement:** child of *restaurant-ecommerce → Overview*.
>
> **Document standard:** ADR / MADR. Platform-wide ADRs: [../kaha-main-api/decisions.md](../kaha-main-api/decisions.md).

---

## ADR-E01 — Heavy snapshot pattern on orders

**Status:** Accepted

**Context.** Menus, prices, customer profiles, and addresses live upstream (local catalog or Kaha Main) and *change*. An order is a financial + legal record that must not change after the fact.

**Decision.** Copy every order-relevant value into `*_snapshot` columns on `order` / `order_item` / `order_item_addon` at order creation.

**Alternatives considered.**
- Join live tables for order display → rejected: a price edit or profile delete would silently rewrite history.
- Event-sourcing the order → rejected: heavy machinery for a need that snapshots solve directly.

**Consequences.**
- ✅ Financial history is immutable and correct regardless of upstream edits/deletes.
- ⚠️ Storage duplication; **discipline required** — never "fix" an old order by reading live data.

---

## ADR-E02 — No foreign keys to users/businesses; validate-on-use

**Status:** Accepted (consequence of platform ADR-002)

**Context.** Users/businesses are owned by `kaha-main-api-v3` in a separate database.

**Decision.** Store `user_id` / `business_id` as indexed `varchar`. Validate existence via HTTP at checkout (and role checks), not on every cart action.

**Alternatives considered.**
- Replicate user/business tables here → rejected: stale data, sync burden.
- Validate on every cart mutation → rejected: needless backbone load + latency on a hot path.

**Consequences.**
- ✅ Decoupled; cart stays fast.
- ⚠️ A brief window where a referenced entity could vanish between validation and use — acceptable; snapshots absorb it.

---

## ADR-E03 — `order_status_history` is append-only

**Status:** Accepted

**Context.** Order status transitions are an audit/compliance trail.

**Decision.** History rows have only `created_at` — no `updated_at`, no `deleted_at`. `order.current_status` is a denormalized cache of the latest.

**Alternatives considered.**
- Mutable status with no history → rejected: no audit, disputes unresolvable.
- History only, derive current via MAX(created_at) → rejected: dashboard queries too slow.

**Consequences.**
- ✅ Tamper-evident timeline + fast current-status reads.
- ⚠️ Two sources for "status" — `current_status` (fast) vs history (truth). They must be written together transactionally.

---

## ADR-E04 — One cart per (user, business)

**Status:** Accepted

**Context.** A customer may have items from multiple restaurants conceptually, but an order is per-restaurant.

**Decision.** `cart` has a `UNIQUE(user_id, business_id)`. Carts are scoped per restaurant.

**Consequences.**
- ✅ Checkout maps cleanly to one order/one restaurant; no cross-restaurant order ambiguity.
- ⚠️ "Multi-restaurant basket" UX is intentionally not supported at the data layer.

---

## ADR-E05 — `schema.dbml` is the design source of truth

**Status:** Accepted

**Context.** The schema is intricate (snapshots, enums, indexes) and read by multiple people.

**Decision.** Maintain `schema.dbml` (dbdiagram.io format) as the canonical design artifact; code follows it.

**Consequences.**
- ✅ One reviewable, visual contract for the data model.
- ⚠️ It must be updated *with* every migration or it rots into a lie — treat drift as a bug.

---

## How to add a new ADR

**Context → Decision → Alternatives → Consequences**, next number (`ADR-E0X`), `Proposed` → `Accepted`. Supersede, never rewrite.
