# Vega CMS Documentation

Engineering documentation for the Vega Charge Charger Management System: an OCPP 1.6-J / 2.0.1 CSMS for a fleet of 3,000–5,000 charge points, built as a FastAPI modular monolith on Kubernetes.

Diagrams are written in Mermaid and render directly on GitHub.

## Architecture

| Doc | Covers |
|---|---|
| [01 — System overview](architecture/01-system-overview.md) | Context diagram, request paths, component responsibilities, codebase layout, roles |
| [02 — Deployment & infrastructure](architecture/02-deployment.md) | Kubernetes topology, connection registry, autoscaling, graceful drain, Postgres pooling / replication |
| [03 — Key flows](architecture/03-flows.md) | Sequence diagrams: version negotiation, 1.6 and 2.0.1 transactions, command routing, UPI payment, token revocation, drain |
| [04 — Data model](architecture/04-data-model.md) | ER diagram, version normalization, partitioned tables |
| [05 — OCPP protocol notes](architecture/05-ocpp.md) | State machines, status normalization, auth cache vs local list, command names per version |

## Decisions

- [Architecture Decision Records](adr/README.md): why the system is shaped this way
- [Open decisions & risks](open-decisions.md): what still needs a call

## Development

- [Local setup](development/setup.md): WSL2, Docker, uv, migrations, simulator, troubleshooting

## Related files at repo root

| File | Contents |
|---|---|
| `PROJECT_PLAN.md` | MVP feature scope, API surface, testing strategy, timeline |
| `PHASE_1.md` | Task breakdown for the current phase (core OCPP engine) |
| `vega_cms_schema.sql` | Authoritative PostgreSQL DDL |
| `CONTEXT.md` | Session history: corrections and reasoning behind decisions |

## Keeping docs current

- Schema change → update the DDL first, then [04 — Data model](architecture/04-data-model.md).
- New architectural decision → new ADR. Never edit an accepted ADR's decision. Supersede it.
- Items marked **(proposed)** in these docs are not yet implemented. Confirm or remove them as Phase 1 lands.
