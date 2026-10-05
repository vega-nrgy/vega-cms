# ADR-0006: Autoscale on WebSocket connection count, not CPU

**Status:** Accepted

## Context

Each charger holds one long-lived, mostly idle WebSocket. CPU stays low until memory or file-descriptor limits are reached, so CPU-based autoscaling would not trigger before pods fail.

## Decision

Each pod exports an `ocpp_active_connections` gauge. The HPA scales on that metric (Prometheus Adapter or KEDA), with a per-pod target set below the measured ceiling. Run 3–6 pods across at least two availability zones.

## Consequences

- The per-pod ceiling must be measured with a load simulator before the target is set.
- New pods only take new connections. Existing chargers rebalance as they reconnect, so scale out early.
- Requires a metrics pipeline in the cluster from the start.
