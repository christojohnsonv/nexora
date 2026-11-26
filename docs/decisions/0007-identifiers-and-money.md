# ADR-0007: Opaque identifiers and integer monetary values

- **Status:** Proposed
- **Date:** 2026-11-26

## Context
Identifiers cross service boundaries and monetary values must remain exact through order, payment and refund calculations.

## Decision
Recommend UUIDs as opaque cross-service identifiers. Represent money as an integer amount in the currency's minor unit plus an ISO 4217 currency code. Never use binary floating-point for money.

## Consequences
- Public APIs do not expose sequential database primary keys.
- Currency is explicit; foreign-exchange conversion is outside initial scope.
- Confirm UUID version/library and database representation in the first persistence ticket.
