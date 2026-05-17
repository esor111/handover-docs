# kaha-business-subscription — Architecture Decision Records (ADRs)

> ℹ️ **Confluence page placement:** child of *kaha-business-subscription → Overview*.
>
> **Document standard:** ADR / MADR. Platform-wide ADRs: [../kaha-main-api/decisions.md](../kaha-main-api/decisions.md).

---

## ADR-S01 — Normalized tier / feature / tier-feature catalogue

**Status:** Accepted

**Context.** Plans share features at different limits (Free: 3 images; Premium: unlimited images).

**Decision.** Three tables: `TIER`, `FEATURE`, and `TIER_FEATURE` (the join carrying `usagesLimit`).

**Alternatives considered.**
- Features as columns on tier → rejected: every new feature is a migration; sparse table.
- Features as a JSON blob on tier → rejected: no integrity, hard to query "who has feature X".

**Consequences.**
- ✅ Add a feature once, attach to any tier at any limit; queryable.
- ⚠️ Reading a tier's full capability set is a join, not a single row.

---

## ADR-S02 — Per-business customization is a separate override table

**Status:** Accepted

**Context.** Sales/ops need to give one business a non-standard limit without inventing a bespoke tier.

**Decision.** `SUBSCRIPTION_FEATURE_CUSTOMIZATION` overrides `TIER_FEATURE` per business. Effective limit = customization if present, else tier default.

**Alternatives considered.**
- Clone the tier and tweak it → rejected: tier explosion, unmanageable catalogue.
- Mutate the subscription's tier link → rejected: loses the "what plan are they nominally on" answer.

**Consequences.**
- ✅ One-off deals don't pollute the catalogue; tier stays the source of defaults.
- ⚠️ Limit resolution is two-step — always compute effective limit, never read tier-feature alone.
- ⚠️ Gated by `subscription.allowCustomization` — check that flag before honoring overrides.

---

## ADR-S03 — Synchronous, failure-propagating gating (not fire-and-forget)

**Status:** Accepted (contrast with notification ADR-N01)

**Context.** A quota check must be authoritative *before* the gated action proceeds.

**Decision.** The backbone calls `use-feature` synchronously and forwards the user's JWT; a failure/over-limit propagates as an error that blocks the action.

**Alternatives considered.**
- Async metering after the fact → rejected: allows quota overrun; billing integrity lost.

**Consequences.**
- ✅ Hard quota enforcement; correct billing.
- ⚠️ This service is in the critical path of gated actions — its availability == those features' availability.

---

## ADR-S04 — Tier carries `categoryId` for automatic free-tier assignment

**Status:** Accepted

**Context.** Different business categories deserve different default free plans.

**Decision.** Tier has `categoryId`; on business approval the backbone queries free tier *by category* and auto-subscribes.

**Consequences.**
- ✅ Zero-touch correct onboarding per category.
- ⚠️ Every category must have a defined free tier or claim-time subscription fails — operational invariant.

---

## ADR-S05 — Tier-change requests are modeled separately from active subscriptions

**Status:** Accepted

**Context.** A business asking to upgrade is not the same as it being upgraded.

**Decision.** `SUBSCRIPTION_REQUEST` (with `status`, `remarks`) is distinct from `SUBSCRIPTION`.

**Consequences.**
- ✅ Auditable request history; approval workflow independent of live state.
- ⚠️ Two places to look when answering "what plan is this business on / wants" — current vs requested.

---

## How to add a new ADR

**Context → Decision → Alternatives → Consequences**, next number (`ADR-S0X`), `Proposed` → `Accepted`. Supersede, never rewrite.
