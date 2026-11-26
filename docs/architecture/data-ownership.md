# Data ownership and consistency

## Database ownership

| Service | Owned data |
|---|---|
| Product Catalog | Products, categories, variants |
| Customer | Profiles, delivery addresses |
| Order | Orders, items, purchase snapshots, Saga state |
| Inventory | Stock records, reservations |
| Payment | Simulated payment attempts, refunds |
| Notification | Notification requests and processing state |

The Gateway is not a system of record. Redis is a cache/coordination component, never the authoritative store for business records.

For free local development, logically isolated service databases may share one PostgreSQL server/container. Separate credentials and permissions must still enforce ownership. Physical database instances can be chosen later for operational reasons.

## Cross-service access

- Use service APIs or published events; never connect directly to another service's database.
- Avoid shared ORM models and shared persistence packages.
- Shared contracts describe communication, not shared ownership.
- Multi-service reads require API composition or an explicitly owned read model.

## Transactions and consistency

A database transaction is local to one service. It may atomically persist local business state and an outbox record. It cannot include a remote call. Cross-service workflows use Saga coordination and compensation.

Strong consistency applies inside a service transaction; eventual consistency is expected across services. Inventory and Payment are authoritative for their own state; Order maintains its own workflow view.

## Identifiers and money

Recommend opaque UUID identifiers for records crossing service boundaries. Represent money as integer minor units plus an ISO 4217 currency code; never use floating-point arithmetic for money. Currency conversion is outside initial scope.

Use version-controlled, forward-only migrations owned by each service. Migration tooling, schema, indexes and connection-pool sizing are selected in the relevant persistence tickets.
