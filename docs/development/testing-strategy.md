# Testing strategy

Testing is layered: fast deterministic unit tests first, then integration tests at infrastructure boundaries, then end-to-end business workflow tests.

## Unit tests
- Test domain rules and use cases without network/database dependencies where practical.
- Cover validation, state transitions, money calculations and error mapping.
- Use table-driven tests for edge cases.

## Integration tests
- Verify migrations, SQL queries, constraints and transaction boundaries against PostgreSQL.
- Verify gRPC serialization, deadlines, status codes and contract behaviour.
- Verify Kafka duplicate delivery, retry and offset handling when messaging is introduced.
- Use local containers/test dependencies; never require paid cloud services.

## End-to-end tests
Exercise complete journeys through the Gateway: successful checkout, insufficient stock, simulated payment decline, duplicate checkout, transient dependency failure, process interruption during checkout and notification processing independent of checkout availability.

## Quality gates
At minimum run gofmt, go test ./... and go vet ./... in each Go module. Add race detection for concurrency-sensitive code and integration tests for persistence/messaging boundaries. Assert observable behaviour and data invariants rather than incidental internal call order.
