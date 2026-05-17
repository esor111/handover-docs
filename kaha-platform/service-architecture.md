# Kaha Platform — Service Architecture & Integration

> **Read this first.** Every other project doc assumes you understand this page.
> This describes *how the services connect* and *why they were designed this way*.

---

## 1. The Big Picture

Kaha is a **central backbone + satellite microservices** architecture. One service (`kaha-main-api-v3`) owns identity and business data. Every other service either calls the backbone or is called by it. No service shares a database with another.

```mermaid
flowchart TB
    subgraph clients["CLIENTS"]
        direction LR
        Mobile["📱 kaha-revamp<br/>Flutter app"]
        AdminV2["🖥️ kaha-admin-v2<br/>React"]
        Web["🌐 Web frontends"]
    end

    Backbone["⚙️ <b>kaha-main-api-v3</b><br/>NestJS · Port 3006<br/><i>Identity · Business · Orchestration</i>"]

    subgraph satellites["SATELLITE SERVICES (owned by Kaha)"]
        direction LR
        Notif["🔔 kaha-notification<br/>NestJS"]
        Sub["💳 kaha-business-subscription<br/>NestJS"]
        Chat["💬 chat service<br/>NestJS"]
        Ecom["🛒 restaurant-ecommerce<br/>NestJS"]
    end

    subgraph external["EXTERNAL / SEPARATE SYSTEMS"]
        direction LR
        CRS["🏨 Hotel CRS<br/>(3rd-party)"]
        EV["⚡ EV Charging API"]
        Hostel["🏠 Hostel API<br/>dev.kaha.com.np"]
        Firebase["🔥 Firebase FCM"]
    end

    clients --> Backbone
    Backbone -->|"push / email / sms"| Notif
    Backbone -->|"tiers / billing"| Sub
    Backbone -->|"chat channels"| Chat
    Backbone -->|"reservation proxy"| CRS
    Backbone -->|"charging points"| EV
    Backbone -->|"hostel lookups"| Hostel
    Ecom -->|"user / business lookups"| Backbone
    Notif --> Firebase

    style Backbone fill:#1e88e5,color:#fff,stroke:#0d47a1,stroke-width:3px
    style clients fill:#e3f2fd,stroke:#90caf9
    style satellites fill:#e8f5e9,stroke:#a5d6a7
    style external fill:#fff3e0,stroke:#ffcc80
```

**Reading the diagram:** arrows point in the direction of the HTTP call. `kaha-main-api-v3` is the hub — it calls 6 systems outward, and is called inward by `restaurant-ecommerce` (and the hotel/hostel backends for auth).

---

## 2. Every Integration, Explained

Each row is a real, code-verified HTTP integration. Env var names are exact.

| Caller | Callee | Env Var | What it does | Pattern |
|---|---|---|---|---|
| `kaha-main-api-v3` | `kaha-notification` | `NOTIFICATION_URL` | Push, email, SMS, topic subs, device tokens | Fire-and-forget |
| `kaha-main-api-v3` | `kaha-business-subscription` | `SUBSCRIPTION_URL` | Subscribe business, use-feature quota, tiers | Sync + token forward |
| `kaha-main-api-v3` | chat service | `CHAT_URL` | Fetch chat channels, then enrich with user/business data | Proxy + enrich |
| `kaha-main-api-v3` | Hotel CRS | `CRS_BASE_URL` | Hotel search/book/cancel reservation proxy | Service account |
| `kaha-main-api-v3` | EV API | `EV_API_URL` | Filter EV charging points | Sync passthrough |
| `kaha-main-api-v3` | Hostel API | `HOSTEL_API_URL` | Hostel-related business lookups | Sync passthrough |
| `restaurant-ecommerce` | `kaha-main-api-v3` | `KAH_API_V3_BASE_URL` | Get user, business, business-user role | Sync (validate-on-use) |
| `hms-api` (hotel) | `kaha-main-api-v3` | (kaha-sync module) | Service-account sync of property/booking | Service account |
| `hostel-server` | `kaha-main-api-v3` | `KAHA_MAIN_API_URL` | User auth validation | Stateless JWT |

---

## 3. The Four Integration Patterns (and why each was chosen)

The codebase uses **four distinct patterns**. Knowing which is which tells you how failures behave.

### Pattern A — Fire-and-forget (notifications)

```mermaid
sequenceDiagram
    participant M as kaha-main-api-v3
    participant N as kaha-notification
    M->>N: POST /push-notifications/structured
    Note over M: try/catch swallows errors<br/>main flow continues regardless
    N-->>M: (response ignored)
```

**Decision:** Notification calls are wrapped in `try/catch` that logs but never throws. On logout, `deleteNotificationDevice` even has a 5s timeout and explicitly does *not* rethrow.
**Why:** A notification failure must never break a user action. If FCM is down, login should still succeed. Reliability of the core flow > delivery guarantee of a push.
**Consequence for maintainers:** Lost notifications are *silent*. Don't debug "missing push" by looking at the main API logs — look at the notification service + BullMQ queue.

### Pattern B — Sync with token forwarding (subscription)

```mermaid
sequenceDiagram
    participant C as Client
    participant M as kaha-main-api-v3
    participant S as kaha-business-subscription
    C->>M: Action (Bearer userToken)
    M->>S: POST /subscription/use-feature<br/>Authorization: userToken
    S->>S: Validate same JWT secret
    S-->>M: quota result (throws if exceeded)
    M-->>C: success / 500 if subscription failed
```

**Decision:** The user's *own* `Authorization` token is forwarded to the subscription service. The subscription service validates it independently using the **same `JWT_SECRET_TOKEN`**.
**Why:** Subscription decisions are per-user/per-business and must be authorized as that user. Forwarding the token avoids a second auth handshake and keeps the subscription service stateless.
**Consequence:** All services **must** share the identical `JWT_SECRET_TOKEN`. Rotating it is a coordinated, all-services deploy — not a single-service change.

### Pattern C — Proxy + enrich (chat)

```mermaid
sequenceDiagram
    participant C as Client
    participant M as kaha-main-api-v3
    participant Ch as chat service
    participant DB as kaha-main DB
    C->>M: GET chat channels
    M->>Ch: GET /{path} (Bearer token)
    Ch-->>M: raw channels (user/business IDs only)
    M->>DB: Look up user + business details
    M-->>C: enriched channels (names, avatars, business info)
```

**Decision:** The chat service stores only IDs. `kaha-main-api-v3` fetches raw channels then hydrates them from its own `UserRepository` / `BusinessRepository`.
**Why:** Chat shouldn't duplicate user/business data (single source of truth stays in the backbone). The trade-off is N+1 lookups, mitigated with `Promise.all`.
**Consequence:** Chat is unusable without the backbone — it has no display data on its own.

### Pattern D — Service account (Hotel CRS)

```mermaid
sequenceDiagram
    participant M as kaha-main-api-v3
    participant CRS as Hotel CRS (3rd-party)
    Note over M: Long-lived service-account JWT<br/>auto-refresh at 5-min threshold
    M->>CRS: Reservation request + service token
    alt token near expiry
        M->>CRS: re-login (phone/password fallback)
        CRS-->>M: fresh token
    end
    CRS-->>M: reservation data
```

**Decision:** CRS is a *3rd-party* system. The backbone authenticates as a single service account (not per-user), with token auto-refresh (`CRS_TOKEN_REFRESH_THRESHOLD_MS=300000`) and a 30s request timeout.
**Why:** The CRS doesn't know about Kaha users. One machine identity proxies all hotel traffic.
**Consequence:** CRS outages affect all hotel reservations platform-wide. The 30s timeout is the blast-radius limiter.

---

## 4. Cross-Cutting Architectural Decisions

These decisions hold across **all** services. They're the load-bearing rules — changing one is a platform-wide event.

| Decision | Rationale | What breaks if ignored |
|---|---|---|
| **Database-per-service, no shared DB** | Independent deploy + schema evolution. Each service owns its data. | Cross-service `JOIN`s are impossible by design — use HTTP calls or ID references |
| **Cross-service refs are string IDs, not FKs** | DBs are physically separate; FKs can't span them | No referential integrity across services — validate via HTTP at write time (see ecommerce) |
| **Shared `JWT_SECRET_TOKEN`, stateless auth** | Any service validates a token alone — no central session store, no auth round-trip | Mismatched secret = every cross-service call 401s. Rotation = coordinated deploy |
| **Snapshot financial/historical data** | E.g. ecommerce orders copy menu/price at order time | Upstream edit/delete would silently corrupt order history |
| **PostGIS on backbone + notification** | Geofencing, proximity search, geo-hash device targeting | Plain `postgres` image fails migrations — must use `postgis/postgis` |
| **NestJS everywhere** | One framework, one mental model across 6+ backends | Onboarding cost stays low; patterns are copy-paste portable |

---

## 5. Failure Behavior Cheat-Sheet

When something breaks, this tells you the blast radius:

```mermaid
flowchart LR
    A["kaha-notification down"] --> A1["✅ Core flows OK<br/>❌ No push/email/SMS<br/>(silent — fire-and-forget)"]
    B["kaha-subscription down"] --> B1["❌ Feature-gated actions 500<br/>✅ Non-gated flows OK"]
    C["chat service down"] --> C1["❌ Chat unusable<br/>✅ Everything else OK"]
    D["Hotel CRS down"] --> D1["❌ All hotel reservations fail<br/>✅ Non-hotel OK"]
    E["kaha-main-api-v3 down"] --> E1["❌ PLATFORM DOWN<br/>(everything depends on it)"]

    style E1 fill:#ffcdd2,stroke:#c62828
    style A1 fill:#fff9c4,stroke:#f9a825
```

**The backbone is the single point of failure by design.** That's the trade-off of this architecture: simplicity and one source of truth, at the cost of `kaha-main-api-v3` being mission-critical. Any HA/scaling effort should prioritize it first.

---

## 6. Where To Go Next

- Backbone internals → [`kaha-main-api/overview.md`](kaha-main-api/overview.md)
- Notification service → [`kaha-notification/overview.md`](kaha-notification/overview.md)
- Subscription service → [`kaha-subscription/overview.md`](kaha-subscription/overview.md)
- E-commerce service → [`kaha-ecommerce/overview.md`](kaha-ecommerce/overview.md)
