# Database design

## 1. Design scope

This document defines the logical relational design for Nexora's eight stateful services. It is the baseline for service-owned PostgreSQL schemas and migrations; it is not a claim that these tables have already been implemented.

The design follows these decisions:

- One logical PostgreSQL database per stateful service.
- A local development PostgreSQL server/container may host all eight databases.
- Each service owns its schema, migrations, database credentials and data-access code.
- No cross-service foreign keys, joins, shared ORM models or direct database access.
- UUID primary keys are used for records that may be referenced across service boundaries.
- Money is stored as integer minor units with an ISO 4217 currency code. Floating-point types are prohibited for monetary values.
- Timestamps are stored as `timestamptz` in UTC.
- Mutable records carry `created_at` and `updated_at`; append-only history/event records carry `created_at`.
- State transitions are validated by the owning service, not by the Gateway.

## 2. Database inventory

| Service | Database | Primary responsibility |
|---|---|---|
| Identity | `identity_db` | Authentication identities, credentials, sessions and refresh-token lifecycle |
| Seller & Store | `seller_db` | Seller business profiles, store profile/configuration and seller membership |
| Product Catalog | `product_db` | Categories, products, variants, attributes and media metadata |
| Customer | `customer_db` | Buyer profiles and saved addresses |
| Order | `order_db` | Orders, item snapshots, status history, checkout Saga and outbox |
| Inventory | `inventory_db` | Stock balances, movements and reservations |
| Payment | `payment_db` | Simulated payment attempts, idempotency and refunds |
| Notification | `notification_db` | Notification intent, templates, delivery attempts and consumer deduplication |

## 3. Identity database — `identity_db`

Identity is the authority for authentication identity and credential lifecycle. Seller business details do not belong here.

### `users`
- `id UUID PK`
- `email CITEXT NOT NULL UNIQUE`
- `phone VARCHAR(32) NULL UNIQUE`
- `password_hash TEXT NULL` — nullable for identities that authenticate through an external identity provider.
- `status VARCHAR(20) NOT NULL` — `pending`, `active`, `suspended`, `disabled`.
- `email_verified_at TIMESTAMPTZ NULL`
- `phone_verified_at TIMESTAMPTZ NULL`
- `last_login_at TIMESTAMPTZ NULL`
- `created_at TIMESTAMPTZ NOT NULL`, `updated_at TIMESTAMPTZ NOT NULL`

Constraints/indexes: unique normalized email; unique non-null phone; index on `status`.

### `identity_providers`
- `id UUID PK`, `user_id UUID FK -> users.id`
- `provider VARCHAR(40) NOT NULL`, `provider_subject VARCHAR(255) NOT NULL`
- `created_at TIMESTAMPTZ NOT NULL`
- Unique (`provider`, `provider_subject`); index on `user_id`.
- Stores external identity subject identifiers, never provider access tokens.

### `refresh_tokens`
- `id UUID PK`, `user_id UUID FK -> users.id`
- `token_hash TEXT NOT NULL UNIQUE`
- `family_id UUID NOT NULL`, `expires_at TIMESTAMPTZ NOT NULL`
- `revoked_at TIMESTAMPTZ NULL`, `replaced_by_token_id UUID NULL FK -> refresh_tokens.id`
- `created_at TIMESTAMPTZ NOT NULL`, `created_ip INET NULL`, `user_agent TEXT NULL`
- Indexes on (`user_id`, `expires_at`) and `family_id`.
- Store only token hashes; never persist raw refresh tokens.

### `roles`, `permissions`, `user_roles`, `role_permissions`
- `roles(id UUID PK, code VARCHAR(60) UNIQUE NOT NULL, name VARCHAR(100) NOT NULL)`
- `permissions(id UUID PK, code VARCHAR(100) UNIQUE NOT NULL, description TEXT NULL)`
- `user_roles(user_id UUID FK -> users.id, role_id UUID FK -> roles.id, assigned_at TIMESTAMPTZ NOT NULL, PK(user_id, role_id))`
- `role_permissions(role_id UUID FK -> roles.id, permission_id UUID FK -> permissions.id, PK(role_id, permission_id))`
- Platform-level roles only. Seller/store membership roles belong to Seller & Store.

## 4. Seller & Store database — `seller_db`

The initial product assumption is one seller account associated with one store. Keep the schema explicit enough to extend later, but do not introduce multi-store organization complexity now.

### `seller_profiles`
- `id UUID PK`
- `identity_user_id UUID NOT NULL UNIQUE` — opaque Identity service user ID; no cross-database FK.
- `display_name VARCHAR(160) NOT NULL`
- `legal_name VARCHAR(200) NULL`
- `business_email CITEXT NULL`, `business_phone VARCHAR(32) NULL`
- `status VARCHAR(20) NOT NULL` — `pending`, `active`, `suspended`, `rejected`.
- `created_at TIMESTAMPTZ NOT NULL`, `updated_at TIMESTAMPTZ NOT NULL`
- Index on `status`.

### `stores`
- `id UUID PK`, `seller_id UUID NOT NULL UNIQUE FK -> seller_profiles.id`
- `name VARCHAR(160) NOT NULL`, `slug CITEXT NOT NULL UNIQUE`
- `description TEXT NULL`, `logo_media_id UUID NULL` — opaque media reference; media storage is not a separate service in the initial scope.
- `status VARCHAR(20) NOT NULL` — `draft`, `active`, `suspended`, `closed`.
- `default_currency CHAR(3) NOT NULL`
- `created_at TIMESTAMPTZ NOT NULL`, `updated_at TIMESTAMPTZ NOT NULL`
- One-to-one seller/store relationship enforced by unique `seller_id`.

### `store_settings`
- `store_id UUID PK FK -> stores.id`
- `timezone VARCHAR(64) NOT NULL DEFAULT 'UTC'`
- `locale VARCHAR(16) NOT NULL DEFAULT 'en'`
- `allow_order_placement BOOLEAN NOT NULL DEFAULT true`
- `created_at TIMESTAMPTZ NOT NULL`, `updated_at TIMESTAMPTZ NOT NULL`

### `seller_memberships`
- `id UUID PK`, `seller_id UUID NOT NULL FK -> seller_profiles.id`
- `identity_user_id UUID NOT NULL` — opaque Identity service user ID.
- `role VARCHAR(30) NOT NULL` — e.g. `owner`, `manager`, `catalog_editor`.
- `status VARCHAR(20) NOT NULL` — `invited`, `active`, `revoked`.
- `created_at TIMESTAMPTZ NOT NULL`, `updated_at TIMESTAMPTZ NOT NULL`
- Unique (`seller_id`, `identity_user_id`); index on `identity_user_id`.
- Initial flow may create only the owner membership. The table supports later staff access without changing Identity roles.

### `seller_verification_records`
- `id UUID PK`, `seller_id UUID NOT NULL FK -> seller_profiles.id`
- `verification_type VARCHAR(40) NOT NULL`, `status VARCHAR(20) NOT NULL`
- `reviewed_by UUID NULL` — identity user ID, opaque.
- `reviewed_at TIMESTAMPTZ NULL`, `notes TEXT NULL`
- `created_at TIMESTAMPTZ NOT NULL`
- Index on (`seller_id`, `created_at DESC`).

## 5. Product Catalog database — `product_db`

Catalog owns product definitions and merchandising data, not available stock or reservation state.

### `categories`
- `id UUID PK`, `parent_id UUID NULL FK -> categories.id`
- `name VARCHAR(120) NOT NULL`, `slug CITEXT NOT NULL UNIQUE`
- `description TEXT NULL`, `status VARCHAR(20) NOT NULL`
- `created_at TIMESTAMPTZ NOT NULL`, `updated_at TIMESTAMPTZ NOT NULL`
- Index on `parent_id`; prevent a category from being its own parent and validate deeper cycles in the service.

### `products`
- `id UUID PK`
- `store_id UUID NOT NULL`, `seller_id UUID NOT NULL` — opaque Seller & Store IDs.
- `category_id UUID NULL FK -> categories.id`
- `name VARCHAR(200) NOT NULL`, `slug CITEXT NOT NULL`
- `description TEXT NULL`, `status VARCHAR(20) NOT NULL` — `draft`, `active`, `archived`.
- `created_at TIMESTAMPTZ NOT NULL`, `updated_at TIMESTAMPTZ NOT NULL`, `published_at TIMESTAMPTZ NULL`
- Unique (`store_id`, `slug`); indexes on (`store_id`, `status`) and `category_id`.

### `product_variants`
- `id UUID PK`, `product_id UUID NOT NULL FK -> products.id`
- `sku VARCHAR(100) NOT NULL`, `name VARCHAR(160) NULL`
- `price_minor BIGINT NOT NULL CHECK (price_minor >= 0)`
- `currency CHAR(3) NOT NULL`
- `attributes JSONB NOT NULL DEFAULT '{}'`
- `is_default BOOLEAN NOT NULL DEFAULT false`, `status VARCHAR(20) NOT NULL`
- `created_at TIMESTAMPTZ NOT NULL`, `updated_at TIMESTAMPTZ NOT NULL`
- Unique (`product_id`, `sku`); index on `sku`.
- A product has one or more variants; the service enforces at most one default variant.

### `product_media`
- `id UUID PK`, `product_id UUID NOT NULL FK -> products.id`
- `variant_id UUID NULL FK -> product_variants.id`
- `storage_key TEXT NOT NULL`, `media_type VARCHAR(40) NOT NULL`
- `alt_text VARCHAR(255) NULL`, `position SMALLINT NOT NULL DEFAULT 0`
- `created_at TIMESTAMPTZ NOT NULL`
- Index on (`product_id`, `position`).

### `product_attribute_definitions`
- `id UUID PK`, `name VARCHAR(100) NOT NULL`, `code VARCHAR(80) NOT NULL UNIQUE`
- `data_type VARCHAR(20) NOT NULL`, `is_variant_axis BOOLEAN NOT NULL DEFAULT false`
- `created_at TIMESTAMPTZ NOT NULL`

Category assignment and rich attribute requirements can evolve; do not add a generic EAV subsystem until actual product requirements need it.

## 6. Customer database — `customer_db`

### `customers`
- `id UUID PK`
- `identity_user_id UUID NULL UNIQUE` — opaque Identity ID when customer authentication is enabled.
- `email CITEXT NULL`, `phone VARCHAR(32) NULL`
- `first_name VARCHAR(100) NOT NULL`, `last_name VARCHAR(100) NULL`
- `status VARCHAR(20) NOT NULL DEFAULT 'active'`
- `created_at TIMESTAMPTZ NOT NULL`, `updated_at TIMESTAMPTZ NOT NULL`
- Indexes on non-null email and phone where uniqueness is not required by the product's guest-customer policy.

### `customer_addresses`
- `id UUID PK`, `customer_id UUID NOT NULL FK -> customers.id`
- `label VARCHAR(40) NULL`, `recipient_name VARCHAR(160) NOT NULL`
- `phone VARCHAR(32) NOT NULL`, `address_line_1 VARCHAR(200) NOT NULL`
- `address_line_2 VARCHAR(200) NULL`, `locality VARCHAR(120) NOT NULL`
- `city VARCHAR(120) NOT NULL`, `region VARCHAR(120) NULL`
- `postal_code VARCHAR(24) NOT NULL`, `country_code CHAR(2) NOT NULL`
- `is_default BOOLEAN NOT NULL DEFAULT false`
- `created_at TIMESTAMPTZ NOT NULL`, `updated_at TIMESTAMPTZ NOT NULL`
- Index on `customer_id`; partial unique index on `customer_id` where `is_default = true`.

## 7. Order database — `order_db`

Order owns the order lifecycle and Saga. Item and address snapshots preserve what the customer purchased even if catalog/customer records later change.

### `orders`
- `id UUID PK`, `order_number VARCHAR(32) NOT NULL UNIQUE`
- `customer_id UUID NULL`, `store_id UUID NOT NULL`, `seller_id UUID NOT NULL` — opaque IDs owned by Customer and Seller & Store.
- `status VARCHAR(24) NOT NULL` — `pending`, `confirmed`, `processing`, `completed`, `cancelled`, `failed`.
- `currency CHAR(3) NOT NULL`
- `subtotal_minor BIGINT NOT NULL CHECK (subtotal_minor >= 0)`
- `discount_minor BIGINT NOT NULL DEFAULT 0 CHECK (discount_minor >= 0)`
- `shipping_minor BIGINT NOT NULL DEFAULT 0 CHECK (shipping_minor >= 0)`
- `tax_minor BIGINT NOT NULL DEFAULT 0 CHECK (tax_minor >= 0)`
- `total_minor BIGINT NOT NULL CHECK (total_minor >= 0)`
- `idempotency_key VARCHAR(128) NULL`
- `created_at TIMESTAMPTZ NOT NULL`, `updated_at TIMESTAMPTZ NOT NULL`, `placed_at TIMESTAMPTZ NULL`, `cancelled_at TIMESTAMPTZ NULL`
- Unique (`customer_id`, `idempotency_key`) where idempotency key is not null; indexes on (`store_id`, `created_at DESC`), (`customer_id`, `created_at DESC`) and (`status`, `created_at`).

### `order_items`
- `id UUID PK`, `order_id UUID NOT NULL FK -> orders.id`
- `product_id UUID NOT NULL`, `variant_id UUID NOT NULL` — opaque Catalog IDs.
- `product_name VARCHAR(200) NOT NULL`, `variant_name VARCHAR(160) NULL`, `sku VARCHAR(100) NOT NULL`
- `unit_price_minor BIGINT NOT NULL CHECK (unit_price_minor >= 0)`
- `quantity INTEGER NOT NULL CHECK (quantity > 0)`
- `line_total_minor BIGINT NOT NULL CHECK (line_total_minor >= 0)`
- `attributes_snapshot JSONB NOT NULL DEFAULT '{}'`
- `created_at TIMESTAMPTZ NOT NULL`
- Index on `order_id`; index on `product_id` for operational lookup (not a foreign key).

### `order_addresses`
- `id UUID PK`, `order_id UUID NOT NULL FK -> orders.id`
- `address_type VARCHAR(20) NOT NULL` — `shipping`, `billing`
- Snapshot columns: `recipient_name`, `phone`, `address_line_1`, `address_line_2`, `locality`, `city`, `region`, `postal_code`, `country_code`.
- `created_at TIMESTAMPTZ NOT NULL`
- Unique (`order_id`, `address_type`).

### `order_status_history`
- `id UUID PK`, `order_id UUID NOT NULL FK -> orders.id`
- `from_status VARCHAR(24) NULL`, `to_status VARCHAR(24) NOT NULL`
- `reason_code VARCHAR(60) NULL`, `notes TEXT NULL`
- `actor_type VARCHAR(20) NOT NULL`, `actor_id UUID NULL`
- `created_at TIMESTAMPTZ NOT NULL`
- Index on (`order_id`, `created_at`).

### `checkout_sagas`
- `id UUID PK`, `order_id UUID NOT NULL UNIQUE FK -> orders.id`
- `state VARCHAR(30) NOT NULL`, `current_step VARCHAR(40) NOT NULL`
- `failure_code VARCHAR(80) NULL`, `failure_detail TEXT NULL`
- `started_at TIMESTAMPTZ NOT NULL`, `completed_at TIMESTAMPTZ NULL`, `updated_at TIMESTAMPTZ NOT NULL`
- Index on (`state`, `updated_at`) for recovery scans.

### `saga_steps`
- `id UUID PK`, `saga_id UUID NOT NULL FK -> checkout_sagas.id`
- `step_name VARCHAR(40) NOT NULL`, `status VARCHAR(20) NOT NULL`
- `command_id UUID NOT NULL UNIQUE`, `attempt_count INTEGER NOT NULL DEFAULT 0`
- `last_error TEXT NULL`, `started_at TIMESTAMPTZ NULL`, `completed_at TIMESTAMPTZ NULL`
- Unique (`saga_id`, `step_name`); index on (`status`, `started_at`).

### `outbox_events`
- `id UUID PK`, `aggregate_type VARCHAR(80) NOT NULL`, `aggregate_id UUID NOT NULL`
- `event_type VARCHAR(160) NOT NULL`, `event_version INTEGER NOT NULL`
- `payload JSONB NOT NULL`, `occurred_at TIMESTAMPTZ NOT NULL`
- `published_at TIMESTAMPTZ NULL`, `publish_attempts INTEGER NOT NULL DEFAULT 0`, `last_error TEXT NULL`
- Index on (`published_at`, `occurred_at`) for unpublished-event polling.
- Business changes and outbox inserts occur in the same local transaction.

## 8. Inventory database — `inventory_db`

Inventory is authoritative for stock. A reservation is a time-bounded claim against stock; it is not a decrement that can be silently repeated.

### `stock_items`
- `id UUID PK`, `store_id UUID NOT NULL`, `product_id UUID NOT NULL`, `variant_id UUID NOT NULL` — opaque Seller/Catalog IDs.
- `sku VARCHAR(100) NOT NULL`
- `on_hand INTEGER NOT NULL CHECK (on_hand >= 0)`
- `reserved INTEGER NOT NULL DEFAULT 0 CHECK (reserved >= 0 AND reserved <= on_hand)`
- `version BIGINT NOT NULL DEFAULT 1`
- `created_at TIMESTAMPTZ NOT NULL`, `updated_at TIMESTAMPTZ NOT NULL`
- Unique (`store_id`, `variant_id`); index on `sku`.
- Available quantity is `on_hand - reserved`; do not persist a separately mutable available count.

### `stock_movements`
- `id UUID PK`, `stock_item_id UUID NOT NULL FK -> stock_items.id`
- `movement_type VARCHAR(24) NOT NULL` — `receipt`, `adjustment`, `reservation`, `release`, `fulfilment`.
- `quantity_delta INTEGER NOT NULL CHECK (quantity_delta <> 0)`
- `reference_type VARCHAR(40) NULL`, `reference_id UUID NULL`
- `reason TEXT NULL`, `created_at TIMESTAMPTZ NOT NULL`
- Index on (`stock_item_id`, `created_at DESC`) and (`reference_type`, `reference_id`).

### `inventory_reservations`
- `id UUID PK`, `order_id UUID NOT NULL` — opaque Order ID.
- `status VARCHAR(20) NOT NULL` — `pending`, `reserved`, `released`, `committed`, `expired`.
- `expires_at TIMESTAMPTZ NOT NULL`, `idempotency_key VARCHAR(128) NOT NULL UNIQUE`
- `created_at TIMESTAMPTZ NOT NULL`, `updated_at TIMESTAMPTZ NOT NULL`
- Index on (`status`, `expires_at`) for expiry workers; index on `order_id`.

### `reservation_items`
- `id UUID PK`, `reservation_id UUID NOT NULL FK -> inventory_reservations.id`
- `stock_item_id UUID NOT NULL FK -> stock_items.id`
- `quantity INTEGER NOT NULL CHECK (quantity > 0)`
- Unique (`reservation_id`, `stock_item_id`); index on `stock_item_id`.

### `inventory_outbox_events`
- Same transactional outbox pattern as Order: event identity, aggregate, event type/version, JSONB payload, occurrence/publication timestamps, attempts and error.
- Index on (`published_at`, `occurred_at`).

## 9. Payment database — `payment_db`

Payment is a simulated provider boundary. It must not store card numbers, CVVs or other payment credentials.

### `payment_attempts`
- `id UUID PK`, `order_id UUID NOT NULL` — opaque Order ID.
- `attempt_number INTEGER NOT NULL`, `status VARCHAR(24) NOT NULL` — `initiated`, `succeeded`, `failed`, `cancelled`, `refunded`.
- `amount_minor BIGINT NOT NULL CHECK (amount_minor >= 0)`, `currency CHAR(3) NOT NULL`
- `idempotency_key VARCHAR(128) NOT NULL UNIQUE`
- `simulation_result VARCHAR(24) NOT NULL` — deterministic configured outcome, not card data.
- `failure_code VARCHAR(80) NULL`, `failure_detail TEXT NULL`
- `created_at TIMESTAMPTZ NOT NULL`, `updated_at TIMESTAMPTZ NOT NULL`, `completed_at TIMESTAMPTZ NULL`
- Unique (`order_id`, `attempt_number`); index on (`order_id`, `created_at DESC`).

### `refunds`
- `id UUID PK`, `payment_attempt_id UUID NOT NULL FK -> payment_attempts.id`
- `amount_minor BIGINT NOT NULL CHECK (amount_minor > 0)`, `currency CHAR(3) NOT NULL`
- `status VARCHAR(20) NOT NULL` — `requested`, `succeeded`, `failed`.
- `idempotency_key VARCHAR(128) NOT NULL UNIQUE`, `reason TEXT NULL`
- `created_at TIMESTAMPTZ NOT NULL`, `updated_at TIMESTAMPTZ NOT NULL`, `completed_at TIMESTAMPTZ NULL`
- Index on `payment_attempt_id`.

### `payment_outbox_events`
- Transactional outbox with the same columns and indexing pattern as Order and Inventory.

## 10. Notification database — `notification_db`

### `notification_templates`
- `id UUID PK`, `code VARCHAR(100) NOT NULL`, `channel VARCHAR(20) NOT NULL`
- `locale VARCHAR(16) NOT NULL`, `subject_template TEXT NULL`, `body_template TEXT NOT NULL`
- `version INTEGER NOT NULL`, `active BOOLEAN NOT NULL DEFAULT true`
- `created_at TIMESTAMPTZ NOT NULL`, `updated_at TIMESTAMPTZ NOT NULL`
- Unique (`code`, `channel`, `locale`, `version`).

### `notifications`
- `id UUID PK`, `event_id UUID NULL` — source event ID for deduplication.
- `recipient_id UUID NULL`, `recipient_address TEXT NOT NULL`
- `channel VARCHAR(20) NOT NULL` — `email`, `sms`, `in_app`.
- `template_code VARCHAR(100) NOT NULL`, `template_version INTEGER NULL`
- `payload JSONB NOT NULL`, `status VARCHAR(20) NOT NULL` — `queued`, `processing`, `sent`, `failed`, `cancelled`.
- `scheduled_at TIMESTAMPTZ NOT NULL`, `sent_at TIMESTAMPTZ NULL`
- `created_at TIMESTAMPTZ NOT NULL`, `updated_at TIMESTAMPTZ NOT NULL`
- Unique (`event_id`, `channel`, `recipient_address`) where event_id is not null; index on (`status`, `scheduled_at`).

### `notification_attempts`
- `id UUID PK`, `notification_id UUID NOT NULL FK -> notifications.id`
- `attempt_number INTEGER NOT NULL`, `status VARCHAR(20) NOT NULL`
- `provider_reference VARCHAR(160) NULL`, `error_code VARCHAR(80) NULL`, `error_detail TEXT NULL`
- `attempted_at TIMESTAMPTZ NOT NULL`, `next_retry_at TIMESTAMPTZ NULL`
- Unique (`notification_id`, `attempt_number`); index on `next_retry_at`.

### `consumed_events`
- `consumer_name VARCHAR(120) NOT NULL`, `event_id UUID NOT NULL`
- `event_type VARCHAR(160) NOT NULL`, `consumed_at TIMESTAMPTZ NOT NULL`
- Primary key (`consumer_name`, `event_id`) for consumer idempotency.

## 11. Cross-service relationship map

Cross-service identifiers below are deliberately **not** PostgreSQL foreign keys.

| Owning database | Reference | Referenced owner | Purpose |
|---|---|---|---|
| `seller_db` | `identity_user_id` | Identity | Link seller/member to login identity |
| `product_db` | `store_id`, `seller_id` | Seller & Store | Product ownership and seller portal scoping |
| `customer_db` | `identity_user_id` | Identity | Optional authenticated customer profile |
| `order_db` | `customer_id` | Customer | Buyer reference |
| `order_db` | `store_id`, `seller_id` | Seller & Store | Seller/store order ownership |
| `order_db` | `product_id`, `variant_id` | Product Catalog | Source references alongside immutable item snapshots |
| `inventory_db` | `store_id`, `product_id`, `variant_id` | Seller & Store / Catalog | Stock ownership and variant reference |
| `inventory_db` | `order_id` | Order | Reservation reference |
| `payment_db` | `order_id` | Order | Payment reference |
| `notification_db` | `event_id`, `recipient_id` | Event producer / Identity or Customer | Event deduplication and recipient reference |

## 12. Entity relationship diagrams

The diagrams show relationships **inside each database only**. Cross-service references are labelled as external IDs and intentionally have no FK edge.

### Identity
~~~mermaid
erDiagram
  USERS ||--o{ IDENTITY_PROVIDERS : links
  USERS ||--o{ REFRESH_TOKENS : owns
  REFRESH_TOKENS o|--o| REFRESH_TOKENS : replaces
  USERS ||--o{ USER_ROLES : assigned
  ROLES ||--o{ USER_ROLES : grants
  ROLES ||--o{ ROLE_PERMISSIONS : contains
  PERMISSIONS ||--o{ ROLE_PERMISSIONS : included
~~~

### Seller & Store
~~~mermaid
erDiagram
  SELLER_PROFILES ||--|| STORES : owns
  STORES ||--|| STORE_SETTINGS : configures
  SELLER_PROFILES ||--o{ SELLER_MEMBERSHIPS : has
  SELLER_PROFILES ||--o{ SELLER_VERIFICATION_RECORDS : undergoes
~~~

### Product Catalog
~~~mermaid
erDiagram
  CATEGORIES o|--o{ CATEGORIES : parent
  CATEGORIES o|--o{ PRODUCTS : classifies
  PRODUCTS ||--|{ PRODUCT_VARIANTS : offers
  PRODUCTS ||--o{ PRODUCT_MEDIA : displays
  PRODUCT_VARIANTS o|--o{ PRODUCT_MEDIA : depicts
~~~

### Customer
~~~mermaid
erDiagram
  CUSTOMERS ||--o{ CUSTOMER_ADDRESSES : saves
~~~

### Order
~~~mermaid
erDiagram
  ORDERS ||--|{ ORDER_ITEMS : contains
  ORDERS ||--o{ ORDER_ADDRESSES : snapshots
  ORDERS ||--o{ ORDER_STATUS_HISTORY : records
  ORDERS ||--|| CHECKOUT_SAGAS : coordinates
  CHECKOUT_SAGAS ||--|{ SAGA_STEPS : executes
~~~

### Inventory
~~~mermaid
erDiagram
  STOCK_ITEMS ||--o{ STOCK_MOVEMENTS : records
  STOCK_ITEMS ||--o{ RESERVATION_ITEMS : reserved
  INVENTORY_RESERVATIONS ||--|{ RESERVATION_ITEMS : contains
~~~

### Payment
~~~mermaid
erDiagram
  PAYMENT_ATTEMPTS ||--o{ REFUNDS : refunded_by
~~~

### Notification
~~~mermaid
erDiagram
  NOTIFICATIONS ||--o{ NOTIFICATION_ATTEMPTS : attempts
~~~

## 13. Transaction, concurrency and retention rules

- Order creation, order status change and its outbox event are committed atomically in `order_db`.
- Inventory reservation creation and stock reserved-quantity changes are atomic in `inventory_db`. Use row-level locking or conditional versioned updates to prevent overselling; enforce `reserved <= on_hand`.
- Reservation commands must be idempotent. A retry with the same idempotency key returns the original result and must not reserve stock twice.
- Payment attempt creation is idempotent. Simulated outcomes are deterministic for a given attempt and do not represent real financial processing.
- Outbox publishers may publish more than once. Consumers must deduplicate by event ID and make their side effects idempotent.
- Saga steps and compensation outcomes must be persisted so interrupted workflows can resume after process restarts.
- Notification retry scheduling is persisted; do not rely on in-memory timers as the source of truth.
- Define retention/archival policies before production deployment for refresh tokens, outbox rows, consumed-event records, notification payloads and audit/history data. Do not delete operational records solely to keep tables small.

## 14. Implementation notes and decisions intentionally deferred

These are not blockers for the logical baseline; settle them in the relevant implementation tickets:

- Exact PostgreSQL major version and migration tool.
- Whether product attributes remain JSONB or need normalized attribute-value tables after concrete filtering requirements are known.
- Whether guest checkout is supported and how guest customer identity is represented.
- Tax, shipping-rate and discount calculation policies.
- Exact order state machine and Saga timeout/compensation policy.
- Whether notification delivery uses external providers or local development adapters.
- PII encryption, data-retention periods and deletion/anonymization workflows.
- Whether media metadata later becomes a dedicated Media service.
