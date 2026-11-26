# Nexora

**A distributed commerce platform built with Go**

Nexora is a backend e-commerce platform built around independently deployable services, clear domain boundaries and reliable communication. It brings together core commerce capabilities such as product catalogues, customer management, inventory, order processing and payments in a cohesive distributed system.

The project explores practical backend engineering and distributed-systems concepts through a connected business workflow.

## Architecture

```mermaid
flowchart TB
    Client[API Consumer] -->|REST / HTTPS| Gateway[API Gateway<br/>Gin]

    Gateway -->|gRPC| Catalog[Product Catalog Service]
    Gateway -->|gRPC| Customer[Customer Service]
    Gateway -->|gRPC| Order[Order Service]

    Order <-->|gRPC| Catalog
    Order <-->|gRPC| Customer
    Order <-->|gRPC| Inventory[Inventory Service]
    Order <-->|gRPC| Payment[Payment Service<br/>Simulated processor]

    Order -.->|Publish domain events| Kafka[(Apache Kafka)]
    Inventory -.->|Publish reservation result| Kafka
    Payment -.->|Publish payment result| Kafka
    Kafka -.->|Consume events| Order
    Kafka -.->|Consume events| Notification[Notification Service]

    Catalog --> CatalogDB[(Catalog PostgreSQL)]
    Customer --> CustomerDB[(Customer PostgreSQL)]
    Order --> OrderDB[(Order PostgreSQL)]
    Inventory --> InventoryDB[(Inventory PostgreSQL)]
    Payment --> PaymentDB[(Payment PostgreSQL)]
    Notification --> NotificationDB[(Notification PostgreSQL)]

    Gateway -.-> Redis[(Redis)]
    Catalog -.-> Redis
```

### Repository structure

The repository is organized as a monorepo. Each service has its own application entry point and internal packages, while service-to-service gRPC contracts are maintained separately under `api/proto/`.

```text
.
├── api/
│   └── proto/
│       ├── product-catalog/v1/
│       ├── customer/v1/
│       ├── order/v1/
│       ├── inventory/v1/
│       └── payment/v1/
├── docs/
│   ├── architecture/
│   ├── decisions/
│   ├── development/
│   └── runbooks/
├── infrastructure/
│   ├── local/
│   ├── observability/
│   └── deployment/
├── services/
│   ├── api-gateway/
│   │   ├── cmd/api-gateway/
│   │   ├── internal/
│   │   │   ├── clients/grpc/
│   │   │   ├── config/
│   │   │   ├── health/
│   │   │   └── transport/http/
│   │   │       ├── handlers/
│   │   │       ├── middleware/
│   │   │       └── router/
│   │   └── tests/integration/
│   ├── product-catalog/
│   │   ├── cmd/product-catalog/
│   │   ├── internal/
│   │   │   ├── config/
│   │   │   ├── domain/
│   │   │   ├── health/
│   │   │   ├── repository/postgres/
│   │   │   ├── service/
│   │   │   └── transport/grpc/
│   │   └── tests/integration/
│   ├── customer/
│   │   ├── cmd/customer/
│   │   ├── internal/
│   │   │   ├── config/
│   │   │   ├── domain/
│   │   │   ├── health/
│   │   │   ├── repository/postgres/
│   │   │   ├── service/
│   │   │   └── transport/grpc/
│   │   └── tests/integration/
│   ├── order/
│   │   ├── cmd/order/
│   │   ├── internal/
│   │   │   ├── clients/grpc/
│   │   │   ├── config/
│   │   │   ├── domain/
│   │   │   ├── health/
│   │   │   ├── messaging/kafka/
│   │   │   │   ├── consumer/
│   │   │   │   └── producer/
│   │   │   ├── repository/postgres/
│   │   │   ├── service/
│   │   │   ├── transport/grpc/
│   │   │   └── workflow/
│   │   └── tests/integration/
│   ├── inventory/
│   │   ├── cmd/inventory/
│   │   ├── internal/
│   │   │   ├── config/
│   │   │   ├── domain/
│   │   │   ├── health/
│   │   │   ├── messaging/kafka/producer/
│   │   │   ├── repository/postgres/
│   │   │   ├── service/
│   │   │   └── transport/grpc/
│   │   └── tests/integration/
│   ├── payment/
│   │   ├── cmd/payment/
│   │   ├── internal/
│   │   │   ├── config/
│   │   │   ├── domain/
│   │   │   ├── health/
│   │   │   ├── messaging/kafka/producer/
│   │   │   ├── repository/postgres/
│   │   │   ├── service/
│   │   │   └── transport/grpc/
│   │   └── tests/integration/
│   └── notification/
│       ├── cmd/notification/
│       ├── internal/
│       │   ├── config/
│       │   ├── domain/
│       │   ├── health/
│       │   ├── messaging/kafka/consumer/
│       │   ├── repository/postgres/
│       │   └── service/
│       └── tests/integration/
└── tests/
    └── e2e/
```

#### Package responsibilities

- `cmd/<service>/`: executable entry point for the service.
- `internal/domain/`: domain entities, value objects and business rules.
- `internal/service/`: application use cases and orchestration within the service.
- `internal/repository/`: persistence implementations owned by the service.
- `internal/transport/`: inbound protocols, such as gRPC servers and HTTP handlers.
- `internal/clients/`: outbound synchronous clients to other services.
- `internal/messaging/`: Kafka producers and consumers.
- `internal/config/`: service configuration loading and validation.
- `internal/health/`: service health and readiness checks.
- `tests/integration/`: service-level integration tests.
- `api/proto/`: versioned Protocol Buffer contracts used to generate gRPC code.
- `docs/`: architecture notes, technical decisions, development guidance and operational runbooks.
- `infrastructure/`: local environment, observability and deployment resources.
- `tests/e2e/`: cross-service workflow tests.

Services keep their implementation packages private under Go's `internal/` boundary. Shared service contracts live outside individual services; business logic and databases remain service-owned.

### Services

| Service | Responsibility |
|---|---|
| **API Gateway** | Provides a unified, versioned HTTP API and routes requests to the appropriate services. |
| **Product Catalog** | Manages products, categories, variants and catalogue information. |
| **Customer** | Manages customer profiles and delivery addresses. |
| **Order** | Owns order creation, purchase snapshots, lifecycle and checkout workflow coordination. |
| **Inventory** | Manages stock levels, availability and concurrency-safe reservations. |
| **Payment** | Handles simulated payment attempts, outcomes and refunds without connecting to a real payment provider. |
| **Notification** | Processes notification work from business events. |

Each service owns its domain logic and persistence. Services interact through defined contracts; no service directly accesses another service's database.

### Communication

| Mechanism | Purpose |
|---|---|
| REST/HTTPS | External client communication through the API Gateway |
| gRPC | Synchronous internal service-to-service communication |
| Apache Kafka | Asynchronous event-driven communication |

## Checkout workflow

1. A client submits an order request through the API Gateway.
2. The Order Service validates the request and retrieves required customer and product information through service contracts.
3. The Order Service creates the order and coordinates the checkout workflow.
4. The Inventory Service attempts to reserve the required stock and publishes the outcome.
5. When stock is reserved, the Payment Service performs a simulated payment attempt.
6. The Order Service processes the workflow outcome and updates the order lifecycle.
7. The Notification Service consumes relevant events and processes notification work.

The workflow uses asynchronous communication where appropriate and accounts for duplicate event delivery, transient failures and partial completion.

## Engineering focus

- **Service design:** bounded contexts, domain ownership and independent deployability.
- **Data ownership:** database-per-service and service-level transaction boundaries.
- **Distributed workflows:** Saga orchestration and compensating actions.
- **Messaging reliability:** transactional outbox, idempotent consumers, retries and dead-letter handling.
- **Concurrency:** safe inventory reservation and protection against overselling.
- **Consistency:** eventual consistency across service boundaries.
- **Resilience:** timeouts, graceful shutdown and controlled failure handling.
- **Observability:** structured logging, health checks, metrics and distributed tracing.
- **Testing:** unit, integration and end-to-end verification of business workflows.

## Technology stack

| Area | Technology |
|---|---|
| Language | Go |
| HTTP framework | Gin |
| Internal RPC | gRPC, Protocol Buffers |
| Event streaming | Apache Kafka |
| Databases | PostgreSQL |
| Caching | Redis |
| Containers | Docker, Docker Compose |
| Orchestration | Kubernetes |
| Metrics and dashboards | Prometheus, Grafana |
| Distributed tracing | OpenTelemetry |
| CI/CD | GitHub Actions |

## Design principles

- Each service owns its business domain and data.
- Cross-service communication happens through explicit APIs or events.
- No direct database access across service boundaries.
- Synchronous communication is used when an immediate response is required; asynchronous events are used to decouple workflows.
- Business logic remains independent of transport and persistence details.
- Reliability and failure handling are considered part of the system design.


## Project documentation

The architecture and engineering decisions are documented in the [docs directory](docs/README.md):

- [System overview](docs/architecture/system-overview.md)
- [Checkout workflow](docs/architecture/checkout-workflow.md)
- [Data ownership and consistency](docs/architecture/data-ownership.md)
- [Reliability principles](docs/architecture/reliability.md)
- [Architecture Decision Records](docs/decisions/README.md)
- [Development workflow](docs/development/workflow.md)
- [Testing strategy](docs/development/testing-strategy.md)

These documents describe the intended architecture. Individual components and controls are implemented incrementally through the project issues.

---

**Nexora — a Go-based distributed commerce platform exploring the engineering behind reliable, event-driven systems.**
