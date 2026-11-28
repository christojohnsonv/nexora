# System overview

## Purpose and constraints

Nexora is a production-oriented, medium-scale distributed commerce platform. The client application in the current scope is a **React seller portal** used by sellers to manage their stores and products; this is not a public shopper storefront. The platform also models customer data and order workflows for commerce operations.

The architecture uses independently owned services and data stores while remaining practical to develop and demonstrate with free, locally runnable infrastructure. Avoid enterprise complexity without a measured need.

## Target architecture

~~~mermaid
flowchart TB
  SellerPortal[React Seller Portal] -->|REST / HTTPS| Gateway[API Gateway - Gin]

  Gateway -->|gRPC| Identity[Identity Service]
  Gateway -->|gRPC| Seller[Seller & Store Service]
  Gateway -->|gRPC| Catalog[Product Catalog]
  Gateway -->|gRPC| Customer[Customer]
  Gateway -->|gRPC| Order[Order]
  Gateway -->|gRPC| Inventory[Inventory]
  Gateway -->|gRPC| Payment[Payment]

  Order -->|gRPC commands / queries| Catalog
  Order -->|gRPC commands / queries| Customer
  Order -->|gRPC commands| Inventory
  Order -->|gRPC commands| Payment

  Seller -.-> Kafka[(Kafka)]
  Catalog -.-> Kafka
  Order -.-> Kafka
  Inventory -.-> Kafka
  Payment -.-> Kafka
  Kafka -.-> Order
  Kafka -.-> Notification[Notification]

  Identity --> IdentityDB[(Identity PostgreSQL)]
  Seller --> SellerDB[(Seller PostgreSQL)]
  Catalog --> CatalogDB[(Catalog PostgreSQL)]
  Customer --> CustomerDB[(Customer PostgreSQL)]
  Order --> OrderDB[(Order PostgreSQL)]
  Inventory --> InventoryDB[(Inventory PostgreSQL)]
  Payment --> PaymentDB[(Payment PostgreSQL)]
  Notification --> NotificationDB[(Notification PostgreSQL)]

  Gateway -.-> Redis[(Redis - when introduced)]
~~~

The diagram describes the target architecture, not a claim that every component is implemented. Introduce components only through their planned tickets. Kafka arrows are illustrative; each event producer/consumer relationship must be defined in the event catalog.

## Service responsibilities

| Component | Responsibility | Persistence |
|---|---|---|
| API Gateway | Single external entry point; REST routing, edge validation, request correlation, rate limiting and enforcement of authentication/authorization policies | None |
| Identity | Authentication identities, credential/identity lifecycle, token or session-related state | `identity_db` |
| Seller & Store | Seller profiles, business details, store creation/configuration/status, seller-to-store ownership and membership | `seller_db` |
| Product Catalog | Products, categories, variants, attributes and product media metadata; not authoritative stock | `product_db` |
| Customer | Buyer/customer profiles and delivery addresses used by commerce workflows | `customer_db` |
| Order | Order lifecycle, order items, purchase-time product/price snapshots and checkout Saga orchestration/state | `order_db` |
| Inventory | Authoritative stock, stock movements and concurrency-safe reservations | `inventory_db` |
| Payment | Simulated payment attempts, outcomes, transaction records and refunds; no real payment provider | `payment_db` |
| Notification | Asynchronous notification requests, delivery attempts and processing state from business events | `notification_db` |

The React seller portal is a client, not a microservice. The API Gateway is an edge component, not a business system of record.

### Ownership boundaries

- Identity owns authentication identity and credential-related data. Seller & Store owns seller business profiles, stores and their membership/ownership relationships. Authentication identity is linked to a seller profile through an identifier; the databases are not joined.
- Seller & Store owns store configuration and lifecycle. Product Catalog owns product definitions and category/variant data. A product stores the owning seller/store identifiers as references, not cross-database foreign keys.
- Product Catalog does not own stock availability. Inventory is authoritative for stock and reservations.
- Order owns the order record, purchase-time snapshots and Saga state. It does not own inventory or payment state.
- Customer owns buyer profile and delivery-address data. Order may retain the necessary immutable delivery and purchase snapshots for the order lifecycle.
- Notification owns notification processing records, not the source business state that triggered a notification.

For the initial model, one seller account is associated with one store. Do not assume multi-store membership or organization hierarchies unless a later decision explicitly introduces them.

## Communication

- The React seller portal accesses the platform through versioned REST APIs over HTTPS at the Gateway.
- Gin is the HTTP framework for HTTP services where appropriate.
- Internal synchronous service-to-service calls use gRPC and versioned Protocol Buffers.
- Kafka carries asynchronous domain events and integration notifications.
- The Gateway delegates business operations to owning services; it must not implement domain workflows or query service databases directly.
- Do not force Gin into gRPC servers, Kafka consumers, workers or domain packages.

## Data and infrastructure guardrails

- Each stateful service owns a logically separate PostgreSQL database, its schema, migrations, credentials and data-access layer.
- For free local development, these databases may share one PostgreSQL server/container. Logical databases and separate least-privilege credentials preserve ownership boundaries; separate containers are not required.
- No service reads or writes another service's database. Cross-service references are identifiers, not cross-database foreign keys.
- Redis is a cache/coordination component when introduced, never the authoritative store for business records.
- Use bounded resource pools, explicit timeouts, indexed queries, observable queues and controlled retries.
- Prefer local PostgreSQL, Kafka, Redis and observability components. Kubernetes is a later target, not a prerequisite for early development.
- Do not introduce sharding, multi-region active-active, service meshes or custom operators without evidence.

## Non-goals

No public shopper storefront in the current client scope, no real payment gateway, no cross-service database access and no paid managed-infrastructure requirement.
