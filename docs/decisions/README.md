# Architecture Decision Records

Status meanings: **Accepted** means the direction is established; **Proposed** means the recommendation should be confirmed in the relevant implementation ticket; **Deferred** means intentionally postponed.

| ADR | Decision | Status |
|---|---|---|
| [0001](0001-go.md) | Go for backend services | Accepted |
| [0002](0002-service-boundaries.md) | Service boundaries and data ownership | Accepted |
| [0003](0003-communication-protocols.md) | REST, gRPC and Protobuf | Accepted |
| [0004](0004-kafka-events.md) | Kafka for asynchronous integration | Accepted |
| [0005](0005-checkout-saga.md) | Order-owned orchestrated checkout Saga | Proposed |
| [0006](0006-local-first-infrastructure.md) | Free, local-first infrastructure | Accepted |
| [0007](0007-identifiers-and-money.md) | UUIDs and integer monetary values | Proposed |

## Deferred

Resolve these only when the relevant ticket needs them:
- Authentication provider, token format, refresh/revocation and account lifecycle.
- Exact REST routes, RPC methods, error mapping and Protobuf fields.
- Event topic names, partitions, retention, replication and exact envelope schema.
- Migration tool, database schemas, indexes and pool sizes.
- Retry limits, timeout budgets, reservation TTL and DLQ retention.
- Redis keys, TTLs and invalidation policy.
- Kubernetes topology, ingress, secrets delivery and deployment target.
- Notification delivery channel/provider.

Do not block early service work on decisions that are not yet relevant.
