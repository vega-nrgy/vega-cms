# ADR-0008: Monthly range partitioning for high-volume tables

**Status:** Accepted

## Context

`meter_values` grows by ~1.5–2 M rows/day (~100 GB/year). Deleting old rows from a single huge table causes bloat and long vacuums, and time-range queries scan far more than needed.

## Decision

Range-partition `meter_values` and `status_notifications` by month from the first migration. Partitions are created ahead of need by `pg_partman` or a scheduled job, with a `DEFAULT` partition as a safety net. Partitioned tables are created with hand-written Alembic DDL. Primary keys include the partition column.

## Consequences

- Retention becomes `DROP` / `DETACH` of old partitions.
- Queries filtered by time get partition pruning.
- No foreign keys can point into these tables.
- Partition creation becomes an operational job that must be monitored.
