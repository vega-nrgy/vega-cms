# Architecture Decision Records

One file per decision. To change a decision, add a new ADR that supersedes the old one and update the old one's status. Do not rewrite history.

| ADR | Decision | Status |
|---|---|---|
| [ADR-0001](0001-dual-stack-version-agnostic-model.md) | Dual-stack OCPP with a version-agnostic data model | Accepted |
| [ADR-0002](0002-fastapi-mobilityhouse-ocpp.md) | FastAPI with mobilityhouse/ocpp | Accepted |
| [ADR-0003](0003-modular-monolith.md) | Modular monolith, not microservices | Accepted |
| [ADR-0004](0004-rabbitmq-for-telemetry.md) | RabbitMQ (not Kafka) to buffer telemetry | Accepted |
| [ADR-0005](0005-postgres-source-of-truth.md) | Postgres is the source of truth; Redis is never authoritative | Accepted |
| [ADR-0006](0006-autoscale-on-connections.md) | Autoscale on WebSocket connection count, not CPU | Accepted |
| [ADR-0007](0007-vega-only-mvp-sites-placeholder.md) | Vega-only MVP with a `sites` placeholder | Accepted |
| [ADR-0008](0008-monthly-partitioning.md) | Monthly range partitioning for high-volume tables | Accepted |
| [ADR-0009](0009-split-replicas-ha-vs-reporting.md) | Separate replicas for HA and for reporting | Accepted |

Template: copy any ADR and keep the Status / Context / Decision / Consequences headings.
