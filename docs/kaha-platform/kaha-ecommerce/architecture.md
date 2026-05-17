# restaurant-ecommerce — Architecture (Building Blocks)

> ℹ️ **Confluence page placement:** child of *restaurant-ecommerce → Overview*.
>
> **Document standard:** arc42 §5 + C4 Level 2/3 + key runtime flow.

---

## 1. Container View (C4 — Level 2)

```mermaid
flowchart TB
    subgraph boundary["restaurant-ecommerce boundary"]
        API["⚙️ NestJS API<br/>(8 modules)"]
        DB[("🗄️ PostgreSQL")]
    end
    Customer["👤 Customer"] -->|HTTPS + JWT| API
    API -->|HTTP validate-on-use| Main["⚙️ kaha-main-api-v3"]
    API --> DB
    style API fill:#fb8c00,color:#fff,stroke:#e65100,stroke-width:2px
```

---

## 2. Component View (C4 — Level 3): Modules

```mermaid
flowchart LR
    subgraph catalog["Catalog"]
        category["category"]
        menu["menu"]
        addons["addons"]
    end
    subgraph ordering["Ordering"]
        cart["cart"]
        order["order"]
    end
    subgraph feedback["Feedback"]
        rating["menu-rating (review)"]
    end
    subgraph platform["Platform"]
        auth["auth"]
        svccomm["service-communication"]
    end

    menu --> category
    menu --> addons
    cart --> menu
    order --> cart
    rating --> order
    order --> svccomm
    cart --> svccomm
    svccomm -.->|validate| Main["kaha-main-api-v3"]
```

| Module | Responsibility |
|---|---|
| `category` | Menu categories per business (self-referential tree) |
| `menu` | Menu items + variants; pricing, dietary tags, services (dine-in/takeaway/delivery) |
| `addons` | Add-on groups + options (single/multi select, min/max) |
| `cart` | One cart per (user, business); items + selected addons with price snapshots |
| `order` | Order lifecycle, money math, status history, snapshots |
| `menu-rating` | Reviews — verified-purchase linked to `order_item` |
| `auth` | Validates the platform JWT |
| `service-communication` | `GET /users/:id`, `/businesses/:id`, `/business-users/:b/:u` on the backbone |

---

## 3. Key Runtime Flow: Cart → Order

```mermaid
sequenceDiagram
    participant C as Customer
    participant Cart as cart module
    participant SC as service-communication
    participant Main as kaha-main-api-v3
    participant Order as order module
    participant DB as PostgreSQL

    C->>Cart: Add items (+ addons)
    Cart->>DB: persist with unit_price_snapshot
    C->>Order: Checkout
    Order->>SC: GET /users/:id, /businesses/:id
    SC->>Main: validate user + business exist
    Main-->>SC: user/business data
    Order->>Order: snapshot customer, address, menu, tax, coupon
    Order->>DB: write order + order_item + order_item_addon
    Order->>DB: append order_status_history (PENDING)
    Order-->>C: order_number (ORD-2026-000123)
```

**In words:** prices are snapshotted as early as cart time (`unit_price_snapshot`) and re-snapshotted comprehensively at order time (customer, address, menu name, tax rate, coupon). Validation against the backbone happens at checkout — not on every cart mutation — to keep the cart fast.

---

## 4. Order Status Lifecycle

```mermaid
stateDiagram-v2
    [*] --> PENDING: order placed
    PENDING --> CONFIRMED: restaurant accepts
    CONFIRMED --> PREPARING
    PREPARING --> READY
    READY --> OUT_FOR_DELIVERY: delivery
    READY --> DELIVERED: dine-in / takeaway
    OUT_FOR_DELIVERY --> DELIVERED
    PENDING --> CANCELLED
    CONFIRMED --> CANCELLED
    DELIVERED --> [*]
```

`order.current_status` is denormalized for hot-path dashboard queries; the full transition log is **append-only** in `order_status_history` (see [data-model.md](data-model.md)).

---

## 5. Where To Go Next

- The snapshot data model → [data-model.md](data-model.md)
- Why validate-on-use / snapshots → [decisions.md](decisions.md)
