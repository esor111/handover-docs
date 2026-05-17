# Hotel System (HMS) — Architecture (Building Blocks)

> ℹ️ **Confluence page placement:** child of *Hotel System → Overview*.
>
> **Document standard:** arc42 §5 + C4 Level 2/3 + key runtime flows.

---

## 1. Container View (C4 — Level 2)

```mermaid
flowchart TB
    GuestFE["🌐 hms-frontend<br/>React/Vite — guest booking"]
    AdminFE["🖥️ hms-admin<br/>React/Vite — staff ops"]
    Site["🏨 property-site<br/>Next.js — marketing/careers"]

    subgraph boundary["hms-api boundary"]
        API["⚙️ NestJS API<br/>(33 modules)"]
        DB[("🗄️ hms_db<br/>PostgreSQL")]
    end

    GuestFE --> API
    AdminFE --> API
    Site --> API
    Main["⚙️ kaha-main-api-v3"] -->|kaha-sync| API
    API --> DB
    API --> Khalti["💰 Khalti"]
    API --> SMTP["📧 SMTP"]

    style API fill:#00897b,color:#fff,stroke:#004d40,stroke-width:2px
```

| Container | Role |
|---|---|
| `hms-frontend` | Guest-facing: search, book, pay |
| `hms-admin` | Staff-facing: front desk, housekeeping, rates, reports |
| `property-site` | Public marketing + careers (Next.js, partly static) |
| `hms-api` | The system of record — all 33 modules, one DB |

---

## 2. Component View (C4 — Level 3): Module Groups

```mermaid
flowchart LR
    subgraph inventory["🏨 Inventory & Rates"]
        property["property"]
        rooms["rooms"]
        roomtypes["room-types"]
        rateplans["rate-plans"]
        dailyrates["daily-rates"]
        inv["inventory"]
        amenities["amenities"]
    end
    subgraph bookingg["📅 Booking"]
        booking["booking"]
        bookingroom["booking-room"]
        bookingcharges["booking-charges"]
        roomstatus["room-status-history"]
        search["search"]
    end
    subgraph guestf["🧳 Guest & Finance"]
        guests["guests"]
        payments["payments"]
        invoices["invoices"]
        promocodes["promo-codes"]
        feetemplates["fee-templates"]
        taxrules["tax-rules"]
        reviews["reviews"]
        favourites["favourites"]
    end
    subgraph platform["🔌 Platform & Tenant"]
        auth["auth"]
        users["users"]
        roles["roles"]
        kahasync["kaha-sync"]
        apikeys["api-keys"]
        idempotency["idempotency"]
        auditlog["audit-log"]
        email["email"]
        events["events"]
        careers["careers"]
        publicm["public"]
        platformm["platform"]
        reporting["reporting"]
    end

    booking --> rooms
    booking --> rateplans
    booking --> dailyrates
    payments --> booking
    invoices --> booking
    kahasync -.->|inbound from Kaha| users
```

> ℹ️ **Navigation tip:** a "double booking" issue → **Booking** group (`booking`, `booking-room`, `search`, `idempotency`). A "wrong price" issue → **Inventory & Rates** (`rate-plans`, `daily-rates`, `room-types`). A "tenant data leak" → check property-scoping in the relevant module + `auth`.

| Module | Responsibility |
|---|---|
| `property` | The tenant root — hotel profile, location, settings, members, owners |
| `rooms` / `room-types` | Physical rooms + their categories (occupancy, base price) |
| `rate-plans` | Pricing rules: cancellation policy, inclusions (breakfast/wifi), min/max nights |
| `daily-rates` | Per-date rate overrides (seasonal/event pricing) |
| `inventory` | Room-type availability per date |
| `booking` / `booking-room` / `booking-charges` | Reservation lifecycle + room assignment + extra charges |
| `room-status-history` | Audit of room status & housekeeping transitions |
| `search` | Availability search (dates, occupancy, property) |
| `guests` | Guest profiles + lifetime stats (`totalStays`, `totalNights`, `totalSpend`) |
| `payments` / `invoices` / `refunds` | Khalti payments, invoicing, refunds |
| `promo-codes` / `fee-templates` / `tax-rules` | Discounts, reusable fees, VAT |
| `auth` / `users` / `roles` | HMS-local auth + RBAC (separate from platform JWT) |
| `kaha-sync` | **Inbound** provisioning from kaha-main (service account) |
| `idempotency` | Dedupe payment/booking submissions |
| `api-keys` | 3rd-party integration keys |
| `audit-log` | Action trail per property |
| `careers` / `events` / `public` | Marketing/careers content, public endpoints |

---

## 3. Key Runtime Flow A: Booking with Rate Snapshot

```mermaid
sequenceDiagram
    participant G as Guest (hms-frontend)
    participant S as search
    participant B as booking
    participant RP as rate-plans / daily-rates
    participant I as inventory
    participant DB as hms_db

    G->>S: Search (property, dates, occupancy)
    S->>I: check availability per date
    S-->>G: available room-types + prices
    G->>B: Create booking
    B->>RP: resolve rate plan + per-day rates
    B->>B: snapshot ratePlanName, baseRate, dailyBreakdown[]
    B->>I: decrement inventory (idempotency-guarded)
    B->>DB: persist booking + booking-room + status history
    B-->>G: booking confirmed
```

**In words:** the booking **snapshots** the rate plan name, base rate, and a per-day `dailyBreakdown[]` at creation. Later rate edits never change an existing booking's price (same discipline as ecommerce orders). Inventory decrement is idempotency-guarded to prevent double-booking on retries.

## 4. Key Runtime Flow B: Khalti Payment

```mermaid
sequenceDiagram
    participant G as Guest
    participant API as hms-api/payments
    participant K as Khalti
    G->>API: POST /payments/initiate
    API->>K: initiate (amount, return URL)
    K-->>API: pidx
    API-->>G: redirect to Khalti checkout
    G->>K: complete payment
    K->>API: webhook (needs PUBLIC url)
    API->>K: POST /payment/verify (pidx)
    K-->>API: status
    API->>API: idempotency check, update payment + booking
```

> ⚠️ **Webhook needs a public URL.** Locally use `ngrok` and set `BACKEND_PUBLIC_URL` — otherwise verification never completes (see [runbook.md](runbook.md)).

---

## 5. Where To Go Next

- The multi-tenant tables → [data-model.md](data-model.md)
- Why snapshot / idempotency / own-JWT → [decisions.md](decisions.md)
