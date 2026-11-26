# Checkout workflow

## Ownership

Order owns the order lifecycle and coordinates checkout. Inventory owns stock and reservation state. Payment owns payment-attempt state. No transaction spans service databases.

## Target flow

1. Client submits checkout through the API Gateway with an idempotency key.
2. Order validates the request and obtains required customer and catalogue information through service contracts.
3. Order persists a pending order and purchase-time snapshots (product description, selected variant, unit price, currency and quantity).
4. Order requests a stock reservation from Inventory. Inventory is authoritative for available stock.
5. When stock is reserved, Order requests a simulated payment using a stable payment reference.
6. On confirmed payment success, Order completes the order and the reservation is committed/consumed according to the Inventory contract.
7. On terminal payment failure, Order initiates release of the reservation and records a failed/cancelled outcome.
8. Durable domain events are published for downstream consumers. Notification processing does not block checkout.

## Saga recommendation

Use an Order-owned orchestrated Saga. Checkout is a bounded workflow with a clear business owner; explicit orchestration makes progress, state transitions and recovery inspectable. Persist Saga progress so a process restart cannot lose the workflow.

| Forward action | Compensation / recovery |
|---|---|
| Create pending order | Record a failed/cancelled terminal state |
| Reserve stock | Release the reservation; release must be repeat-safe |
| Initiate simulated payment | Reconcile by stable reference if outcome is unknown; never blindly repeat a successful charge |
| Complete order | Publish completion through durable event publication |

Compensation is a new business action, not a distributed rollback. Compensation can fail and needs retry and reconciliation.

## Invariants

- Repeating checkout with the same idempotency key does not create multiple orders.
- Repeating reserve/release commands does not duplicate or release unrelated stock.
- A stable payment reference cannot produce multiple successful simulated charges.
- Order is not marked paid until Payment confirms success.
- Failed checkout does not leave stock reserved indefinitely.
- Historical order values do not change when catalogue data changes.
- Notification availability does not determine checkout success.

## Failure scenarios to test

Duplicate requests; insufficient stock; inventory timeout before/after reservation; payment decline; payment timeout with unknown outcome; process restart between steps; duplicate/out-of-order events; failed compensation; Kafka outage during publication; unavailable notification consumer.

Exact RPC/event boundaries, reservation TTL, state enums, retry limits and event schemas are finalized in the relevant implementation tickets.
