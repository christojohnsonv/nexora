# System overview

## Purpose and constraints

Nexora is a backend-only distributed commerce platform built with Go. It targets production-oriented engineering practices at a medium scale while remaining practical to develop and demonstrate with free, locally runnable infrastructure. Avoid enterprise complexity without a measured need.

## Target architecture

~~~mermaid
flowchart TB
 Client[API Consumer] -->|REST / HTTPS| Gateway[API Gateway - Gin]
 Gateway -->|gRPC| Catalog[Product Catalog]
 Gateway -->|gRPC| Customer[Customer]
 Gateway -->|gRPC| Order[Order]
 Order -->|gRPC| Catalog
 Order -->|gRPC| Customer
 Order -->|gRPC commands| Inventory[Inventory]
 Order -->|gRPC commands| Payment[Payment]
 Order -.-> Kafka[(Kafka)]
 Inventory -.-> Kafka
 Payment -.-> Kafka
 Kafka -.-> Order
 Kafka -.-> Notification[Notification]
 Catalog --> CatalogDB[(Catalog PostgreSQL)]
 Customer --> CustomerDB[(Customer PostgreSQL)]
 Order --> OrderDB[(Order PostgreSQL)]
 Inventory --> InventoryDB[(Inventory PostgreSQL)]
 Payment --> PaymentDB[(Payment PostgreSQL)]
 Notification --> NotificationDB[(Notification PostgreSQL)]
 Gateway -.-> Redis[(Redis - later)]
~~~

The diagram describes the target, not a claim that every component is implemented. Introduce components only through their planned tickets.

## Service responsibilities

| Service | Responsibility |
|---|---|
| API Gateway | Versioned external HTTP API, routing, edge validation and security enforcement |
| Product Catalog | Products, categories, variants and catalogue descriptions; not authoritative stock |
| Customer | Customer profiles and delivery addresses |
| Order | Order lifecycle, purchase-time snapshots and checkout Saga coordination |
| Inventory | Authoritative stock and concurrency-safe reservations |
| Payment | Simulated payment attempts, outcomes and refunds; no real provider |
| Notification | Asynchronous notification processing from business events |

Each service owns its domain logic and persistence. No service reads or writes another service's database.

## Communication

- External clients use REST over HTTPS through the Gateway.
- Gin is the HTTP framework for HTTP services where appropriate.
- Internal synchronous calls use gRPC and versioned Protocol Buffers.
- Kafka carries asynchronous domain events.
- Do not force Gin into gRPC servers, Kafka consumers, workers or domain packages.

## Scale and infrastructure guardrails

Use bounded resource pools, explicit timeouts, indexed queries, observable queues and controlled retries. Prefer local PostgreSQL, Kafka, Redis and observability components when introduced. Kubernetes is a later target, not a prerequisite for early development. Do not introduce sharding, multi-region active-active, service meshes or custom operators without evidence.

## Non-goals

No frontend, real payment gateway, cross-service database access or paid managed infrastructure requirement.
