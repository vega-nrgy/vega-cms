# ADR-0003: Modular monolith, not microservices

**Status:** Accepted

## Context

The team is small, the domain boundaries are still settling, and the scale target (≤ 5,000 charge points) does not require independent scaling of subsystems.

## Decision

Build one FastAPI application, deployed as one image, internally separated into `app/ocpp/`, `app/api/admin/`, `app/api/driver/`, `app/models/`, and so on. The telemetry worker runs from the same codebase with a different entrypoint as a separate Deployment.

## Consequences

- No cross-service network calls, contracts or distributed transactions.
- Modules must not reach into each other's internals, so a split stays possible later.
- One bad deploy affects everything. Mitigated by graceful drain and staged rollouts.
