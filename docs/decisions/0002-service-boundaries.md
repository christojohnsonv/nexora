# ADR-0002: Service boundaries and database ownership

- **Status:** Accepted
- **Date:** 2026-11-29
- **Supersedes:** The original seven-boundary proposal recorded on 2026-11-26

## Context

Nexora needs clear ownership boundaries for authentication, seller operations and commerce capabilities. Independent business capabilities must not become a distributed monolith through shared persistence or shared domain models. The current client scope is a React seller portal, not a public shopper storefront.

## Decision

Use one API Gateway and eight independently owned stateful services:

1. **Identity** — authentication identities and credential/identity lifecycle.
2. **Seller & Store** — seller business profiles, stores, configuration, status and seller-store ownership/membership.
3. **Product Catalog** — products, categories, variants and product attributes/media metadata.
4. **Customer** — buyer/customer profiles and delivery addresses.
5. **Order** — order lifecycle, purchase-time snapshots and checkout Saga coordination/state.
6. **Inventory** — authoritative stock, movements and reservations.
7. **Payment** — simulated payment attempts, outcomes, transactions and refunds; no real payment provider.
8. **Notification** — notification requests and delivery-processing state.

The API Gateway provides the external REST entry point and owns no business data. The React seller portal is a client, not a service. The initial seller model associates one seller account with one store.

Each stateful service owns one logically separate PostgreSQL database:

`identity_db`, `seller_db`, `product_db`, `customer_db`, `order_db`, `inventory_db`, `payment_db` and `notification_db`.

For free local development, these databases may share one PostgreSQL server/container, with separate least-privilege credentials and independent migrations. No service may access another service's database.

External clients use REST through the Gateway. Internal synchronous calls use gRPC/Protobuf; asynchronous domain events use Kafka. Service-to-service communication must use explicit contracts.

## Consequences

- Authentication identity and seller business profile are separate concepts linked by stable identifiers.
- Seller/store ownership and product ownership are explicit; product records reference seller/store IDs without cross-database foreign keys.
- Product Catalog is not authoritative for stock; Inventory is.
- Order owns its workflow and purchase snapshots; Inventory and Payment remain authoritative for their state.
- Cross-service access uses gRPC APIs or Kafka events; no cross-service joins or shared write transactions.
- Shared code is limited to technical utilities and generated contracts; business rules and persistence remain service-owned.
- A single local PostgreSQL container can host the eight logical databases without weakening the ownership model.
