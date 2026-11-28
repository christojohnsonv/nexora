# Database design

This directory contains the logical relational model for Nexora's service-owned PostgreSQL databases.

## Documents

- [Service schemas and entity relationships](service-schemas.md) — tables, columns, keys, constraints, indexes, cross-service references, ER diagrams and transaction rules.

## Core rules

- Eight stateful services own eight logically separate PostgreSQL databases.
- A single local PostgreSQL server/container may host all eight databases.
- Cross-service identifiers are opaque references, never cross-database foreign keys.
- Each service owns its migrations and persistence implementation.
- The design is the target logical baseline; it does not imply that migrations or tables are already implemented.

Read this alongside [system overview](../architecture/system-overview.md), [data ownership](../architecture/data-ownership.md) and [ADR-0002](../decisions/0002-service-boundaries.md).
