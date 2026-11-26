# Development workflow

## Guiding constraints
- Implement one GitHub issue at a time and follow its scope and acceptance criteria.
- Read prerequisites first; study only concepts needed for the current ticket.
- Do not add future-phase infrastructure or optional refactors.
- Use Gin for HTTP services where appropriate, not for gRPC servers, Kafka consumers, workers or domain logic.
- Update API, event, schema or architecture docs when a ticket changes a contract or established behaviour.

## Local development
- Services should be independently buildable and testable.
- Load configuration from environment variables or ignored local files; commit safe examples only.
- Provide deterministic tests that do not require paid services.
- Introduce local containers only when the relevant issue adds an external dependency.
- Never log credentials, tokens or payment secrets.

## Change checklist
1. Confirm issue scope and acceptance criteria.
2. Run formatting, unit tests, static analysis and required integration tests.
3. Verify failure paths and idempotency where applicable.
4. Update relevant docs and examples.
5. Check for secrets, local files and generated artifacts.
6. Record new architectural decisions as ADRs.
