# Hostel System — Architecture Decision Records (ADRs)

> ℹ️ **Confluence page placement:** child of *Hostel System → Overview*.
>
> **Document standard:** ADR / MADR.

---

## ADR-HO01 — Bed is the atomic bookable unit

**Status:** Accepted

**Context.** Hostels are dormitory-style: many beds per room, each independently occupied/priced.

**Decision.** Model `BED` as a first-class entity (`monthlyRate`, `status`, `occupantId`). Rooms are containers; all occupancy/pricing/transfer logic targets beds.

**Alternatives considered.**
- Book at room level with a capacity counter → rejected: can't track *which* bed, gender-mixing rules, or per-bed pricing.

**Consequences.**
- ✅ Accurate dormitory occupancy, per-bed rates, bed transfers.
- ⚠️ Availability logic is bed-granular — never reason about "rooms free"; reason about beds.

---

## ADR-HO02 — `booking-guest` bridges application and residency

**Status:** Accepted

**Context.** A booking is submitted before any resident record exists; a guest may or may not become a student.

**Decision.** `BOOKING_GUEST` holds the bed target + applicant details and links forward to `STUDENT` once approved.

**Alternatives considered.**
- Create the student immediately on booking → rejected: pollutes resident data with unapproved/rejected applicants.

**Consequences.**
- ✅ Clean separation: applicants vs residents; rejections leave no student rows.
- ⚠️ Two-step lifecycle — code must handle the `booking-guest` → `student` transition explicitly.

---

## ADR-HO03 — Ledger is separate from invoice and payment

**Status:** Accepted

**Context.** Monthly billing needs an auditable per-student financial truth across many invoices and payments.

**Decision.** `LEDGER` is a running per-student account (debit/credit rows with `referenceId` to the source doc), independent of `INVOICE` and `PAYMENT`.

**Alternatives considered.**
- Derive balance by summing invoices − payments on read → rejected: slow, fragile, no audit trail of adjustments/discounts.

**Consequences.**
- ✅ O(1) balance, full auditable history including adjustments.
- ⚠️ Ledger entries must be written transactionally with their source (invoice/payment/discount) or the account drifts.

---

## ADR-HO04 — Contact identity delegated to kaha-main (find-or-create by phone)

**Status:** Accepted

**Context.** The booking contact person should be a platform identity, not a hostel-local duplicate.

**Decision.** Resolve the contact via `GET /users/find-or-create/{phone}` on kaha-main; store only `contactUserId` / `contactPersonId`.

**Consequences.**
- ✅ One platform identity per person; no duplicate user stores.
- ⚠️ Booking depends on kaha-main availability for new contacts; phone is the join key (phone quality matters).

---

## ADR-HO05 — External integrations isolated in `src/external/`

**Status:** Accepted

**Context.** The hostel calls two platform services (kaha-main, kaha-notification).

**Decision.** All outbound platform calls live in `src/external/` (`kaha`, `notifications`) behind service wrappers — not scattered across feature modules.

**Consequences.**
- ✅ One seam to audit/mock integrations; feature modules stay platform-agnostic.
- ⚠️ New platform calls must go through this module, not be inlined — enforce in review.

---

## ADR-HO06 — Multi-tenant by `hostelId` (shared schema)

**Status:** Accepted (same class as Hotel ADR-H01)

**Context.** One deployment serves many hostels.

**Decision.** Single schema; `hostelId` on every core table; all queries hostel-scoped.

**Consequences.**
- ✅ Cheap multi-tenant ops.
- ⚠️ Cross-tenant leakage is a security risk — every query filters by `hostelId`.

---

## How to add a new ADR

**Context → Decision → Alternatives → Consequences**, next number (`ADR-HO0X`), `Proposed` → `Accepted`. Supersede, never rewrite.
