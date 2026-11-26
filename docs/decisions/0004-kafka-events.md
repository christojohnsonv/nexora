# ADR-0004: Kafka for asynchronous integration

- **Status:** Accepted
- **Date:** 2026-11-26

## Context
Downstream consumers should not need to be available during every originating request. Business events should support independent processing.

## Decision
Use Apache Kafka for asynchronous domain events and consumer groups. Use gRPC when an immediate command result is required. Adopt at-least-once processing and idempotent consumers. Use a transactional outbox when local state and event publication intent must be atomic.

## Consequences
- Eventual consistency is expected.
- Events need stable IDs, type/version metadata and correlation information.
- Select partition keys around aggregates whose ordering matters.
- Topic names, partitions, retention, retry topics and DLQ policy are finalized before Kafka integration.
- Introduce Kafka only in its implementation phase.
