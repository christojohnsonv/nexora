# Nexora documentation

## Reading order
1. [System overview](architecture/system-overview.md)
2. [Service boundaries and database ownership](decisions/0002-service-boundaries.md)
3. [Database design](database/README.md)
4. [Checkout workflow](architecture/checkout-workflow.md)
5. [Data ownership and consistency](architecture/data-ownership.md)
6. [Reliability principles](architecture/reliability.md)
7. [Architecture decisions](decisions/README.md)
8. [Development workflow](development/workflow.md) and [testing strategy](development/testing-strategy.md)

## Documentation rules
- The root README is recruiter-facing; these documents explain the intended design and engineering practices.
- ADRs record why decisions were made and their consequences.
- Keep planned behaviour clearly distinguished from implemented and verified behaviour.
- Add detailed API, Protobuf, event and database contracts when their implementation tickets introduce them.
- Never commit credentials, tokens or machine-specific configuration.
