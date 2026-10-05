# ADR-0002: FastAPI with mobilityhouse/ocpp

**Status:** Accepted

## Context

We need an OCPP-J implementation for both 1.6 and 2.0.1, WebSocket handling, and a REST API. Options considered were Python (FastAPI) and Node (Express / NestJS).

## Decision

Use FastAPI with the `mobilityhouse/ocpp` library. It is the most mature OCPP library across either ecosystem, supports both versions, handles framing and snake_case/camelCase translation, and runs on FastAPI's native ASGI WebSockets. Pydantic provides request validation, and SQLAlchemy 2.0 + asyncpg gives mature async Postgres access.

## Consequences

- One language across OCPP handling, REST API, workers and the ML analytics pipeline.
- Python per-connection memory overhead is higher than Go or Node. The per-pod connection ceiling must be measured under load and drives autoscaling targets (ADR-0006).
- We depend on an external library's release cadence for protocol fixes.
