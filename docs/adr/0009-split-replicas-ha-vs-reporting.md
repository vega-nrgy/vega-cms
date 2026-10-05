# ADR-0009: Separate replicas for HA and for reporting

**Status:** Accepted

## Context

Dashboard reporting runs wide scans. OCPP handlers need low-latency writes and read-your-writes consistency. A single replica cannot be both a lag-free failover target and a heavily queried reporting node.

## Decision

Run a synchronous standby for HA failover only (not queried), and an asynchronous read replica for admin reporting. Application code has two session factories: primary for OCPP and payments, replica for reporting endpoints only. PgBouncer in transaction pooling mode fronts the primary from day one.

## Consequences

- Reporting may lag by seconds, which is acceptable for revenue and analytics.
- Synchronous commit adds write latency. Measure it, and confirm the provider's behaviour if the standby becomes unavailable.
- With PgBouncer transaction pooling, asyncpg prepared-statement caching must be disabled.
