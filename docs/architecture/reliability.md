# Reliability principles

These are design requirements for a production-oriented system, not a claim that every control is already implemented.

## Requests and RPC
- Set explicit deadlines/timeouts and propagate caller deadlines where appropriate.
- Retry only transient failures, with bounded attempts and backoff; retry only operations safe to repeat or protected by idempotency.
- Do not retry validation or authorization failures.
- Bound concurrency and database connection pools.
- Return stable error codes; never expose stack traces or secrets.

## Messaging
- Assume at-least-once delivery; consumers must be idempotent.
- Commit offsets only after effects are durably handled.
- Use a transactional outbox to atomically persist local state and events.
- Define bounded retry and dead-letter handling for poison messages.
- Include event ID, type, schema version, occurrence time and correlation/causation identifiers.
- Do not promise global ordering; use aggregate keys where per-aggregate ordering matters.

## Lifecycle
- Handle termination signals and stop accepting new work during graceful shutdown.
- Drain in-flight requests and safely finish or abandon consumer work before exit.
- Readiness indicates ability to serve; liveness detects a process that cannot recover by itself.

## Security
- Keep secrets out of source control and logs.
- Use least-privilege database and CI credentials.
- Validate external input at trust boundaries.
- Finalize authentication and authorization before exposing customer/order operations beyond trusted local development.
- Production requires TLS, secret management, backups and a documented service-to-service trust model.

## Observability
Use structured logs with service, severity, timestamp and request/trace/correlation IDs. Add health/readiness checks, metrics for latency/errors/dependencies/consumer lag/business outcomes, and OpenTelemetry propagation across HTTP, gRPC and Kafka.

## Medium-scale guardrails
Avoid premature sharding, multi-region active-active, service mesh and custom platform operators. Prefer a small, understandable topology with measurable resource limits.
