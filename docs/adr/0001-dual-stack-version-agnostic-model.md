# ADR-0001: Dual-stack OCPP with a version-agnostic data model

**Status:** Accepted

## Context

The fleet will include chargers speaking OCPP 1.6-J and OCPP 2.0.1. The two versions model the same things differently: 1.6 has connectors and Start/StopTransaction, 2.0.1 has EVSEs, a single `TransactionEvent` message, a 5-value status plus `chargingState`, and charger-assigned transaction IDs. Adding 2.0.1 later to a 1.6-shaped system would mean rewriting the data model and every report.

## Decision

Support both versions from day one. Normalize both into one schema: `charge_points → evses → connectors` (1.6 gets one synthetic EVSE per connector), one `transactions` table with a surrogate key plus `ocpp_transaction_id`, and `transaction_events` as the native 2.0.1 event log (1.6 synthesizes Started and Ended rows). Version is negotiated via `Sec-WebSocket-Protocol` and stored on `charge_points.ocpp_version`, which is the only per-version branch point and is used only by the OCPP layer.

## Consequences

- Reporting and APIs never branch on protocol version.
- Two handler classes and a command translation layer are needed (`remote_start` → `RemoteStartTransaction` or `RequestStartTransaction`, etc.).
- Some 1.6 detail is lossy when normalized (for example Preparing vs Finishing), so raw values are kept alongside normalized ones.
- Every OCPP feature is built and tested twice, which is the main ongoing cost.
