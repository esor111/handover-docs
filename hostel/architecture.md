# Hostel System — Architecture (Building Blocks)

> ℹ️ **Confluence page placement:** child of *Hostel System → Overview*.
>
> **Document standard:** arc42 §5 + C4 Level 2/3 + key runtime flow.

---

## 1. Container View (C4 — Level 2)

```mermaid
flowchart TB
    MgmtFE["🖥️ frontend-hostel-world<br/>React/Vite — staff ops"]
    PubSite["🌐 individual-hostel-web<br/>public booking"]

    subgraph boundary["hostel-server boundary"]
        API["⚙️ NestJS API<br/>(16 modules)"]
        DB[("🗄️ PostgreSQL")]
    end

    MgmtFE --> API
    PubSite --> API
    API -->|external/kaha| Main["⚙️ kaha-main-api-v3"]
    API -->|external/notifications| Notif["🔔 kaha-notification"]
    API --> DB

    style API fill:#5e35b1,color:#fff,stroke:#311b92,stroke-width:2px
```

External integrations are isolated in an `src/external/` module (`kaha`, `notifications`) — a clean seam: all outbound platform calls live in one place.

---

## 2. Component View (C4 — Level 3): Modules

```mermaid
flowchart LR
    subgraph inventory["🛏️ Inventory"]
        hostel["hostel"]
        rooms["rooms"]
    end
    subgraph occupancy["🧑‍🎓 Occupancy"]
        bookings["bookings"]
        students["students"]
        attendance["attendance"]
        activity["activity"]
    end
    subgraph money["💰 Finance"]
        financial["financial"]
    end
    subgraph ops["🛠️ Operations"]
        complaints["complaints"]
        operations["operations"]
        dashboard["dashboard"]
        reports["reports"]
        search["search"]
        publicsite["public-site"]
    end
    subgraph platform["🔌 Platform"]
        auth["auth"]
        notifications["notifications"]
        health["health"]
    end

    bookings --> rooms
    bookings --> students
    students --> attendance
    students --> financial
    notifications -.-> ext["external/kaha + notification"]
```

| Module | Responsibility |
|---|---|
| `hostel` | Tenant root — hostel profile, subdomain, story, legal info |
| `rooms` | Rooms + their beds; gender, status, layout, category |
| `bookings` | Booking request → bed assignment → becomes a student |
| `students` | Resident lifecycle; links to bed, booking-guest, contact person |
| `attendance` | Daily attendance (`firstCheckIn`/`lastCheckOut`) |
| `financial` | Invoices, ledger, payments, charges, discounts, fee types |
| `complaints` | Complaint logging + resolution |
| `operations` | Day-to-day ops (housekeeping, transfers) |
| `dashboard` / `reports` | Occupancy, revenue, attendance analytics |
| `search` | Bed/room availability search |
| `public-site` | Public booking endpoints |
| `auth` | HMS-style local auth |
| `notifications` | Wraps `external/notifications` → kaha-notification |
| `health` | Health/readiness probes |

---

## 3. Key Runtime Flow: Public Booking → Student

```mermaid
sequenceDiagram
    participant A as Applicant (public-site)
    participant B as bookings
    participant K as external/kaha
    participant Main as kaha-main-api-v3
    participant Bed as rooms/bed
    participant S as students
    participant N as notifications

    A->>B: Submit booking (contact phone, bed, check-in, duration)
    B->>K: GET /users/find-or-create/{phone}
    K->>Main: resolve / create contact user
    Main-->>K: KahaUser (contactUserId)
    B->>B: create booking (status PENDING) + booking-guest(s)
    Note over B: staff reviews → APPROVE
    B->>Bed: assign bed (status → occupied, occupiedSince)
    B->>S: create student (bedId, bookingGuestId, contactPersonId)
    B->>N: notify (kaha-notification)
```

**In words:** a booking carries contact details and target bed(s). The contact person is resolved in kaha-main by phone (`find-or-create`) so the platform has one identity for them. On staff approval, the bed flips to occupied and a `student` record is created linking bed + booking-guest + contact person. Notifications go out via the external seam.

---

## 4. Where To Go Next

- The bed/student/ledger tables → [data-model.md](data-model.md)
- Why bed-level / external seam → [decisions.md](decisions.md)
