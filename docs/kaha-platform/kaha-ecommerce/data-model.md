# restaurant-ecommerce — Data Model

> ℹ️ **Confluence page placement:** child of *restaurant-ecommerce → Overview*.
>
> **Document standard:** arc42 §8 + ER model. **Source of truth: `schema.dbml`** in the repo — this page explains it; the file *is* it.

---

## 1. ER Diagram

```mermaid
erDiagram
    CATEGORY ||--o{ MENU : "groups"
    CATEGORY }o--o| CATEGORY : "parent"
    MENU ||--o{ MENU_VARIANT : "has"
    MENU }o--o{ ADDON_GROUP : "via menu_addon_group"
    ADDON_GROUP ||--o{ ADDON : "has"
    CART ||--o{ CART_ITEM : "contains"
    CART_ITEM ||--o{ CART_ITEM_ADDON : "has"
    CART_ITEM }o--|| MENU : "of"
    ORDER ||--o{ ORDER_ITEM : "contains"
    ORDER_ITEM ||--o{ ORDER_ITEM_ADDON : "has"
    ORDER ||--o{ ORDER_STATUS_HISTORY : "audited by"
    ORDER_ITEM ||--o| REVIEW : "verified purchase"
    MENU ||--o{ REVIEW : "rated by"

    CATEGORY {
        uuid id PK
        varchar name
        varchar business_id "external -> Kaha Main"
        uuid parent_id FK "self-ref"
        boolean is_active
    }
    MENU {
        uuid id PK
        varchar name
        numeric base_price
        numeric discounted_price
        menu_service[] services "dine_in/takeaway/delivery"
        dietary_tag[] dietary_tags
        uuid category_id FK
        varchar business_id "external"
    }
    MENU_VARIANT {
        uuid id PK
        uuid menu_id FK
        varchar name "Small / 12 inch"
        numeric price
    }
    ADDON_GROUP {
        uuid id PK
        varchar business_id "external"
        addon_selection_type selection_type "single/multi"
        int min_select
        int max_select "null = unlimited"
    }
    ADDON {
        uuid id PK
        uuid addon_group_id FK
        numeric price
    }
    CART {
        uuid id PK
        varchar user_id "external"
        varchar business_id "external — UNIQUE(user,business)"
    }
    CART_ITEM {
        uuid id PK
        uuid cart_id FK "cascade"
        uuid menu_id FK
        int quantity
        numeric unit_price_snapshot
    }
    ORDER {
        uuid id PK
        varchar order_number UK "ORD-2026-000123"
        varchar user_id "external"
        varchar business_id "external"
        varchar customer_name "SNAPSHOT"
        jsonb delivery_address "SNAPSHOT from Kaha Main"
        numeric subtotal
        numeric tax_rate_snapshot
        numeric total_amount
        payment_status payment_status
        order_status current_status "denormalized hot-path"
    }
    ORDER_ITEM {
        uuid id PK
        uuid order_id FK
        varchar menu_name_snapshot "survives upstream delete"
        numeric unit_price_snapshot
        numeric line_total
    }
    ORDER_ITEM_ADDON {
        uuid id PK
        uuid order_item_id FK
        varchar addon_name_snapshot
        numeric unit_price_snapshot
    }
    ORDER_STATUS_HISTORY {
        uuid id PK
        uuid order_id FK
        order_status status
        varchar updated_by "external user id"
        timestamp created_at "APPEND-ONLY"
    }
    REVIEW {
        uuid id PK
        uuid menu_id FK
        uuid order_item_id FK "verified-purchase link"
        varchar rated_by "external"
        int rating "1..5"
    }
```

**In words (read this even if the diagram renders):**
The **catalog** side (CATEGORY → MENU → MENU_VARIANT, plus ADDON_GROUP/ADDON joined many-to-many via `menu_addon_group`) is mutable and lives only here. The **cart** is one-per-(user,business) and snapshots unit price at add time. The **order** side snapshots *everything*: customer identity, delivery address (copied from Kaha Main), menu/variant/addon names, tax rate, coupon. `ORDER_STATUS_HISTORY` is **append-only** (no `updated_at`, no `deleted_at`) — an immutable audit trail; `order.current_status` is a denormalized copy for fast dashboards. `REVIEW` links to `order_item` to enforce verified-purchase.

---

## 2. The Snapshot Pattern (the defining design)

| Snapshotted field | Source | Why |
|---|---|---|
| `customer_name/phone/email` | Kaha Main user | Order receipt must show who ordered, even if profile later changes/deletes |
| `delivery_address` (jsonb) | Kaha Main address | Delivery proof frozen at order time |
| `menu_name_snapshot` | local menu | Survives menu rename/delete |
| `unit_price_snapshot` | local menu/variant | Price the customer actually paid, immune to later price edits |
| `tax_rate_snapshot` | tax config | Historical tax correctness |
| `coupon_code_snapshot` | promo | Audit which discount applied |

> ⚠️ **Never resolve order display data by joining live catalog/user tables.** That would retroactively rewrite financial history. Always read the `*_snapshot` columns for anything order-historical.

---

## 3. Conventions

| Convention | Detail |
|---|---|
| **PK** | `uuid` |
| **Money** | `numeric(12,2)`; tax rate `numeric(5,4)`; currency `char(3)` default `NPR` |
| **External refs** | `user_id` / `business_id` are `varchar`, indexed, **no FK** (platform ADR-002) |
| **Soft delete** | `deleted_at` on catalog/order; **except** `order_status_history` (append-only) |
| **Hot-path indexes** | `(business_id, current_status, created_at)` dashboard; `(user_id, created_at)` history |

---

## 4. Where To Go Next

- Modules operating this → [architecture.md](architecture.md)
- The reasoning behind snapshots → [decisions.md](decisions.md)
