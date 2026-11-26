# ADR-0003: REST externally and gRPC internally

- **Status:** Accepted
- **Date:** 2026-11-26

## Context
External clients need a broadly compatible API; internal calls benefit from typed contracts and request/response semantics.

## Decision
- Expose external APIs through REST over HTTPS at the API Gateway.
- Use Gin for HTTP services where appropriate.
- Use gRPC for synchronous internal calls.
- Define contracts in versioned Protocol Buffer files under api/proto/<service>/v1/.
- Generate bindings; do not hand-maintain duplicate request/response types.

## Consequences
The Gateway translates external HTTP requests into internal calls. Internal gRPC servers do not use Gin. Follow Protobuf compatibility rules: never reuse field numbers, reserve removed fields and make breaking changes through a new version. Exact methods and error mapping are decided with service tickets.
