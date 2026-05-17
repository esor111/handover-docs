# kaha-business-subscription — Architecture (Building Blocks)

> ℹ️ **Confluence page placement:** child of *kaha-business-subscription → Overview*.
>
> **Document standard:** arc42 §5 + C4 Level 2/3 + key runtime flow.

---

## 1. Container View (C4 — Level 2)

```mermaid
flowchart TB
    subgraph boundary["kaha-business-subscription boundary"]
        API["⚙️ NestJS API<br/>(7 modules)"]
        DB[("🗄️ PostgreSQL")]
    end
    Main["⚙️ kaha-main-api-v3"] -->|HTTP + forwarded JWT| API
    Admin["🖥️ kaha-admin-v2"] -->|HTTP| API
    API --> DB
    style API fill:#8e24aa,color:#fff,stroke:#4a148c,stroke-width:2px
```

A single NestJS service over one PostgreSQL. Two caller roles: the **backbone** (runtime gating) and the **admin panel** (catalogue management).

---

## 2. Component View (C4 — Level 3): Modules

```mermaid
flowchart LR
    subgraph catalogue["Catalogue (admin-managed)"]
        tier["tier"]
        feature["feature"]
    end
    subgraph runtime["Runtime (backbone-driven)"]
        subscription["subscription"]
        custom["subscription-feature-customization"]
    end
    subgraph platform["Platform"]
        auth["auth"]
        logger["logger"]
        svccomm["service-communication"]
    end

    tier --> feature
    subscription --> tier
    custom --> subscription
```

| Module | Responsibility |
|---|---|
| `tier` | Plan definitions — name, price, `categoryId`. A tier bundles features with per-feature limits |
| `feature` | Feature catalogue — `name`, `type`, `isCountable` (metered vs boolean capability) |
| `subscription` | Business ↔ tier assignment with `status`, `startDate`/`endDate`, `allowCustomization`; tracks usage |
| `subscription-feature-customization` | Per-business override of a feature's enabled-state / limit on top of its tier |
| `auth` | Validates the forwarded JWT (shared secret) |
| `logger` | Structured logging |
| `service-communication` | Outbound HTTP (reserved) |

---

## 3. Key Runtime Flow: Use a Gated Feature

```mermaid
sequenceDiagram
    participant C as Client
    participant M as kaha-main-api-v3
    participant S as subscription module
    participant Cu as customization module

    C->>M: Action using a gated feature (Bearer userToken)
    M->>S: POST /subscription/use-feature (forward userToken)
    S->>S: find business subscription + tier
    S->>S: resolve tier-feature limit for this feature
    Cu-->>S: per-business override? (custom limit / disabled)
    S->>S: compare usageCount vs effective limit
    alt within limit
        S->>S: increment subscription-usage
        S-->>M: 200 OK
        M-->>C: action succeeds
    else exceeded / disabled
        S-->>M: error (quota exceeded)
        M-->>C: 500 / blocked
    end
```

**In words:** the *effective* limit = tier's `tier-feature.usagesLimit`, **unless** a `subscription-feature-customization` row overrides it for that business. If the feature is countable and within the effective limit, `subscription-usage.usageCount` is incremented; otherwise the action is blocked. This is why a feature change can be done two ways: edit the tier (affects all businesses on it) or add a customization (affects one business).

---

## 4. Free-Tier-by-Category (claim integration)

When a business is approved in the backbone, it asks this service for the **free tier matching the business's category** (`POST /tiers/search-by-category`), then creates a subscription. Tiers carry `categoryId` precisely so the right free plan is auto-assigned per category.

---

## 5. Where To Go Next

- The tables behind this → [data-model.md](data-model.md)
- Why tier/feature/customization are separate → [decisions.md](decisions.md)
