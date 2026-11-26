# ADR-0002: Service boundaries and database ownership

- **Status:** Accepted
- **Date:** 2026-11-26

## Context
Independent business capabilities should not become a distributed monolith through shared persistence or shared domain models.

## Decision
Use seven boundaries: API Gateway, Product Catalog, Customer, Order, Inventory, Payment and Notification. Each stateful business service owns its data and database lifecycle. No service may access another service's database.

For free local development, logically isolated service databases may share a PostgreSQL server/container, with separate credentials and permissions.

## Consequences
- Cross-service access uses explicit gRPC APIs or Kafka events.
- No cross-service joins or shared write transactions.
- Shared code is limited to technical utilities and generated contracts; business rules remain service-owned.
- Order owns purchase snapshots and workflow coordination; Inventory and Payment remain authoritative for their state.
