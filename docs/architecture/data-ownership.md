# Data ownership and consistency

## Database-per-service model

Nexora uses **one logically independent PostgreSQL database per stateful service**. During local development, all databases may run on a single PostgreSQL server/container to reduce resource usage. This is still database-per-service: each service has its own database, schema/migrations, credentials and ownership boundary.

| Service | Database | Authoritative data |
|---|---|---|
| Identity | `identity_db` | Authentication identities and credential/identity lifecycle data |
| Seller & Store | `seller_db` | Seller business profiles, stores, store configuration/status, seller-store ownership and membership |
| Product Catalog | `product_db` | Products, categories, variants, attributes and product media metadata |
| Customer | `customer_db` | Buyer/customer profiles and delivery addresses |
| Order | `order_db` | Orders, order items, purchase-time snapshots, order lifecycle and Saga state |
| Inventory | `inventory_db` | Stock records, stock movements and reservations |
| Payment | `payment_db` | Simulated payment attempts, transaction records, outcomes and refunds |
| Notification | `notification_db` | Notification requests, delivery attempts and processing state |

The API Gateway owns no business database. Redis is a cache/coordination component, never the authoritative store for business records.

## Ownership rules

- Identity owns authentication identity and credentials. Seller & Store owns seller business data and store membership. Link the records using stable identifiers; do not join their databases.
- Seller & Store owns store configuration and lifecycle. Product Catalog owns product definitions. Product records may contain seller/store identifiers as references, but no cross-database foreign keys.
- Product Catalog does not own stock. Inventory is authoritative for stock and reservation state.
- Order owns the order lifecycle, purchase-time snapshots and Saga state. Inventory and Payment remain authoritative for their respective state.
- Customer owns buyer profiles and delivery addresses. Order stores immutable order-time snapshots needed to preserve the order record.
- Notification owns notification processing state, not the source business record.

For the initial model, one seller account is associated with one store. Multi-store seller membership is not part of the initial assumption.

## Cross-service access

- Use service APIs or published events; never connect directly to another service's database.
- Avoid shared ORM models and shared persistence packages.
- Shared contracts describe communication, not shared ownership.
- Cross-service identifiers are references only; never create cross-database foreign keys.
- Multi-service reads require API composition or an explicitly owned read model.

## Transactions and consistency

A database transaction is local to one service. It may atomically persist local business state and an outbox record. It cannot include a remote call. Cross-service workflows use Saga coordination and compensation.

Strong consistency applies inside a service transaction; eventual consistency is expected across services. Inventory and Payment are authoritative for their own state; Order maintains its own workflow view.

## Identifiers and money

Use opaque UUID identifiers for records crossing service boundaries. Represent money as integer minor units plus an ISO 4217 currency code; never use floating-point arithmetic for money. Currency conversion is outside initial scope.

Use version-controlled, forward-only migrations owned by each service. Migration tooling, schema, indexes and connection-pool sizing are selected in the relevant persistence tickets.
