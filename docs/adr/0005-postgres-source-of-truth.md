# ADR-0005: Postgres is the source of truth; Redis is never authoritative

**Status:** Accepted

## Context

Stateless, disposable pods need shared state for connection routing and fast auth decisions, while transactions and payments need relational integrity.

## Decision

PostgreSQL holds all durable state. Redis holds only (a) the `charger:{cp_id}:pod` connection registry with TTL, (b) command pub/sub channels, and (c) short-TTL read caches of authorization decisions. Anything in Redis can be rebuilt from Postgres or from chargers reconnecting. Redis, RabbitMQ and Postgres run as managed services outside the cluster.

## Consequences

- Losing Redis degrades command routing and adds auth latency until rebuilt, but loses no data.
- Writes to tokens must invalidate the Redis cache explicitly.
- Registry deletes must be compare-and-delete to avoid wiping another pod's registration after a fast reconnect.
