# ADR-0005: Order-owned checkout Saga

- **Status:** Proposed
- **Date:** 2026-11-26

## Context
Checkout spans Order, Inventory and Payment, each with its own database. A single ACID transaction cannot cover these services.

## Decision
Recommend an orchestrated Saga owned by Order. Persist workflow progress. Use Inventory and Payment APIs for commands requiring an immediate result, and durable domain events for downstream consumers and asynchronous integration.

Compensations must be repeat-safe. A payment timeout with unknown outcome must be reconciled using a stable reference rather than blindly issuing another payment attempt.

## Alternatives
- **Choreography:** distributes workflow knowledge among event handlers and makes a multi-step checkout harder to inspect and recover.
- **Distributed transaction/2PC:** couples independent service databases and is not a practical fit here.

## Consequences
Order owns workflow state, not inventory/payment records. Saga state must survive restarts. Compensation failures need retries and operational visibility. Confirm exact command/event boundaries, reservation expiry and terminal states in the relevant service tickets.
