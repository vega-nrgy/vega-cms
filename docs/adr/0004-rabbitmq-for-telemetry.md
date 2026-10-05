# ADR-0004: RabbitMQ (not Kafka) to buffer telemetry

**Status:** Accepted

## Context

`MeterValues` is by far the highest-volume message type, and bursts (many sessions starting together) should not hit Postgres directly. Average load across the full fleet is roughly 20–25 messages/second.

## Decision

Pods publish `MeterValues` to RabbitMQ after parsing. A telemetry worker consumes and batch-inserts into `meter_values`. Critical-path messages (`BootNotification`, `StatusNotification`, transaction start/stop) bypass the queue and are written synchronously before the charger is acked. Chargers never connect to RabbitMQ.

## Consequences

- Kafka's operational overhead (KRaft, partition tuning, retention management) is not justified at this volume.
- RabbitMQ does not offer replayable event history. Revisit Kafka only if an analytics pipeline needs independent replay.
- Telemetry is eventually consistent: a meter reading is acked before it is in Postgres. Queues must be durable with publisher confirms so acked data is not lost.
