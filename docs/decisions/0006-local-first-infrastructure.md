# ADR-0006: Free, local-first infrastructure

- **Status:** Accepted
- **Date:** 2026-11-26

## Context
Nexora must be reproducible without a paid cloud account while still demonstrating realistic infrastructure skills.

## Decision
Use free/open-source tools and make local development the default. Run dependencies locally, using Docker Compose when their implementation phase arrives. Keep cloud deployment optional. Kubernetes is a later target and must not block local execution.

## Consequences
- Prefer local PostgreSQL, Kafka, Redis and observability components over paid managed services for development.
- Keep workstation resource requirements reasonable and document them when the stack is introduced.
- Commit safe example configuration only; never commit secrets.
- Introduce dependencies incrementally, not all at initialization.
- Free local development does not imply production security; production still requires TLS, secret management, backups and access controls.
