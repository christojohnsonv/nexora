# Nexora documentation

## Reading order
1. [System overview](architecture/system-overview.md)
2. [Checkout workflow](architecture/checkout-workflow.md)
3. [Data ownership](architecture/data-ownership.md)
4. [Reliability principles](architecture/reliability.md)
5. [Architecture decisions](decisions/README.md)
6. [Development workflow](development/workflow.md) and [testing strategy](development/testing-strategy.md)

## Documentation rules
- The root README is recruiter-facing; these documents explain the intended design and engineering practices.
- ADRs record why decisions were made and their consequences.
- Keep planned behaviour clearly distinguished from implemented and verified behaviour.
- Add detailed API, Protobuf, event and database contracts when their implementation tickets introduce them.
- Never commit credentials, tokens or machine-specific configuration.
