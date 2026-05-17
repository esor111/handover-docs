# Hotel System (HMS) — Data Model

> ℹ️ **Confluence page placement:** child of *Hotel System → Overview*.
>
> **Document standard:** arc42 §8 + ER model. Code-verified from `hms-api/src` entities (48 entities). **Scope:** core booking + tenant aggregate is diagrammed; careers/events content tables follow `BaseEntity` and are listed, not diagrammed (signal over noise).

---

## 1. Core ER Diagram

```mermaid
erDiagram
    PROPERTY ||--o{ ROOM_TYPE : "offers"
    PROPERTY ||--o{ ROOM : "has"
    PROPERTY ||--o{ RATE_PLAN : "defines"
    PROPERTY ||--o{ PROPERTY_MEMBER : "staffed by"
    PROPERTY ||--o{ BOOKING : "receives"
    ROOM_TYPE ||--o{ ROOM : "categorizes"
    ROOM_TYPE ||--o{ ROOM_TYPE_INVENTORY : "availability"
    ROOM_TYPE ||--o{ ROOM_TYPE_RATE : "priced by"
    BOOKING ||--o{ BOOKING_ROOM : "assigns"
    BOOKING ||--o{ BOOKING_CHARGE : "extra charges"
    BOOKING ||--o{ BOOKING_STATUS_HISTORY : "audited"
    BOOKING ||--o{ PAYMENT : "paid by"
    BOOKING ||--o| INVOICE : "billed"
    PAYMENT ||--o| REFUND : "may refund"
    BOOKING_ROOM }o--|| ROOM : "of"
    GUEST ||--o{ BOOKING : "books"
    USER ||--o{ PROPERTY_MEMBER : "member via"
    PROPERTY_MEMBER }o--|| ROLE : "granted"
    ROLE ||--o{ ROLE_PERMISSION : "has"

    PROPERTY {
        uuid id PK
        enum propertyType
        enum propertyStatus
        enum subscriptionStatus
        string city
        string district
        decimal latitude
        decimal longitude
        string source "kaha | native"
    }
    ROOM_TYPE {
        uuid id PK
        uuid propertyId FK
        string roomTypeCode
        int baseOccupancy
        int maxOccupancy
        decimal basePrice
    }
    ROOM {
        uuid id PK
        uuid propertyId FK
        uuid roomTypeId FK
        string roomNumber
        int floor
        enum status
        enum housekeepingStatus
    }
    RATE_PLAN {
        uuid id PK
        uuid propertyId FK
        string policyType
        int freeCancellationDays
        decimal taxRate
        bool breakfast
        int minNights
    }
    BOOKING {
        uuid id PK
        uuid propertyId FK
        enum bookingStatus
        enum bookingSource
        string ratePlanName "SNAPSHOT"
        decimal baseRate "SNAPSHOT"
        jsonb dailyBreakdown "SNAPSHOT per-day"
        decimal dailyRate
    }
    BOOKING_ROOM {
        uuid id PK
        uuid bookingId FK
        uuid roomId FK
    }
    GUEST {
        uuid id PK
        uuid propertyId FK
        string firstName
        string idNumber
        int totalStays
        int totalNights
        decimal totalSpend
    }
    PAYMENT {
        uuid id PK
        uuid propertyId FK
        uuid bookingId FK
        decimal amount
        enum paymentMethod
        enum paymentStatus
        enum paymentType
    }
    INVOICE {
        uuid id PK
        uuid propertyId FK
        uuid bookingId FK
        string invoiceNumber
        enum invoiceStatus
    }
    USER {
        uuid id PK
        string phone
        string email
        string passwordHash
        enum userType
        string source "kaha | native"
    }
    PROPERTY_MEMBER {
        uuid id PK
        uuid propertyId FK
        uuid userId FK
        uuid roleId FK
        string status "invited/active"
    }
    ROLE { uuid id PK }
    ROLE_PERMISSION { uuid id PK }
    REFUND { uuid id PK }
    ROOM_TYPE_INVENTORY { uuid id PK }
    ROOM_TYPE_RATE { uuid id PK }
    BOOKING_CHARGE { uuid id PK }
    BOOKING_STATUS_HISTORY { uuid id PK }
```

**In words (read this even if the diagram renders):**
**PROPERTY** is the tenant root — *every* core table carries `propertyId`. It offers **ROOM_TYPE**s (categories with occupancy/price), which classify physical **ROOM**s and drive **ROOM_TYPE_INVENTORY** (availability per date) and **ROOM_TYPE_RATE**.

A **BOOKING** belongs to a property and a **GUEST**, snapshots its rate (`ratePlanName`, `baseRate`, `dailyBreakdown[]`), assigns rooms via **BOOKING_ROOM**, accrues **BOOKING_CHARGE**s, logs transitions in **BOOKING_STATUS_HISTORY**, is paid via **PAYMENT** (→ optional **REFUND**) and billed via **INVOICE**.

Staff access is the **USER → PROPERTY_MEMBER → ROLE → ROLE_PERMISSION** chain — a user is a *member of a property* with a role (multi-tenant RBAC, same shape as kaha-main's `business-user`). `USER.source` / `PROPERTY.source` = `kaha` when provisioned via `kaha-sync`, else native.

---

## 2. Conventions

| Convention | Detail |
|---|---|
| **Tenant key** | `propertyId` on every core table — **the** isolation boundary |
| **PK** | `uuid` from `BaseEntity` |
| **Money** | `decimal(precision,scale)` — never float |
| **Rate snapshot** | Bookings copy rate-plan/day data at creation |
| **Origin tracking** | `source` column distinguishes Kaha-provisioned vs native records |
| **Audit** | `*-status-history` + `audit-log` are append-style trails |

---

## 3. Data Decisions

- **`propertyId` everywhere (shared-schema multi-tenancy)** — one DB, tenant-filtered, instead of DB-per-tenant. Cheaper ops; the trade-off is *every query must filter by property* (ADR-H01).
- **Rate snapshot on booking** — `dailyBreakdown[]` freezes nightly pricing so later rate edits don't rewrite confirmed bookings (same principle as ecommerce ADR-E01).
- **`source` discriminator** — lets Kaha-provisioned and natively-created properties/users coexist in one tenant model without separate tables.
- **Guest lifetime stats denormalized** (`totalStays/Nights/Spend`) — fast loyalty/reporting reads without aggregating all bookings.
- **PROPERTY_MEMBER join entity** — carries role + invitation state; a plain M:N table couldn't model "invited but not yet joined."

---

## 4. Where To Go Next

- Modules that own these tables → [architecture.md](architecture.md)
- Why multi-tenant / snapshot → [decisions.md](decisions.md)
