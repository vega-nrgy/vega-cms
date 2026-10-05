# 03 — Key Flows

Sequence diagrams for the flows that cross component boundaries. Elements marked **(proposed)** are design details added during documentation that are not yet implemented and should be confirmed during Phase 1.

## Connection and version negotiation

The OCPP version is chosen during the WebSocket handshake via `Sec-WebSocket-Protocol`. This is the single point where a charger's version is decided, and it is persisted to `charge_points.ocpp_version`.

```mermaid
sequenceDiagram
    autonumber
    participant CP as Charger
    participant LB as Ingress / LB
    participant POD as cms-api pod
    participant R as Redis

    CP->>LB: GET /ocpp/{cp_id} Upgrade: websocket<br/>Sec-WebSocket-Protocol: ocpp2.0.1, ocpp1.6
    LB->>POD: forward upgrade
    POD->>POD: pick highest supported subprotocol
    POD-->>CP: 101 Switching Protocols<br/>Sec-WebSocket-Protocol: ocpp2.0.1
    POD->>R: SET charger:{cp_id}:pod = pod_id EX ttl
    POD->>POD: route to 1.6 or 2.0.1 handler class
    Note over POD: Reject (close) if no offered subprotocol is supported.<br/>Charger authentication (Basic auth / mTLS) is Phase 2.
```

## OCPP 1.6 — boot to stopped transaction

```mermaid
sequenceDiagram
    autonumber
    participant CP as Charger (1.6)
    participant POD as cms-api pod
    participant R as Redis
    participant PG as Postgres
    participant MQ as RabbitMQ
    participant W as Telemetry worker

    CP->>POD: BootNotification
    POD->>PG: upsert charge_points (vendor, model, firmware, ocpp_version)
    POD-->>CP: Accepted, interval

    CP->>POD: StatusNotification (connectorId, Available)
    POD->>PG: update connectors.status + insert status_notifications
    POD-->>CP: ack

    CP->>POD: Authorize (idTag)
    POD->>R: GET auth cache
    alt cache miss
        POD->>PG: SELECT id_tokens
        POD->>R: SET auth cache (short TTL)
    end
    POD-->>CP: idTagInfo (status, expiryDate)

    CP->>POD: StartTransaction (connectorId, idTag, meterStart)
    POD->>PG: insert transactions + transaction_events (Started)
    POD-->>CP: transactionId, idTagInfo

    loop every MeterValueSampleInterval
        CP->>POD: MeterValues (transactionId, sampledValue)
        POD->>MQ: publish
        POD-->>CP: ack
        MQ->>W: consume
        W->>PG: batch insert meter_values
    end

    CP->>POD: StopTransaction (transactionId, meterStop, reason)
    POD->>PG: update transactions + insert transaction_events (Ended)
    POD-->>CP: idTagInfo

    CP->>POD: Heartbeat
    POD->>R: refresh charger:{cp_id}:pod TTL
    POD-->>CP: currentTime
```

## OCPP 2.0.1 — transaction lifecycle

Same tables, one message type. `TransactionEvent` with `eventType` = `Started` / `Updated` / `Ended` replaces Start/Stop, and `Updated` events can carry `meterValue` inline.

```mermaid
sequenceDiagram
    autonumber
    participant CP as Charger (2.0.1)
    participant POD as cms-api pod
    participant PG as Postgres
    participant MQ as RabbitMQ

    CP->>POD: StatusNotification (evseId, connectorId, Occupied)
    POD->>PG: update connectors + insert status_notifications
    CP->>POD: TransactionEvent Started (transactionId, idToken, chargingState)
    POD->>PG: insert transactions + transaction_events (Started)
    POD-->>CP: idTokenInfo

    loop during session
        CP->>POD: TransactionEvent Updated (chargingState, meterValue)
        POD->>PG: insert transaction_events (Updated)
        POD->>MQ: publish meterValue
        POD-->>CP: ack
    end

    CP->>POD: TransactionEvent Ended (stoppedReason, meterValue)
    POD->>PG: close transactions + insert transaction_events (Ended)
    POD-->>CP: ack
```

Two differences from 1.6 that the handlers must respect:

- **The charger generates the transaction ID in 2.0.1.** In 1.6 the CSMS assigns `transactionId`. The `transactions` table therefore needs an internal surrogate key plus a version-agnostic `ocpp_transaction_id` text column, unique per charge point.
- **`seqNo` ordering.** 2.0.1 events carry `seqNo` and may arrive out of order after an offline period. Store it and order by it, not by arrival time.

## CSMS-initiated command routing

How a command issued on any pod reaches the pod holding the charger's socket.

```mermaid
sequenceDiagram
    autonumber
    participant A as Admin / app
    participant PA as Pod A (receives REST)
    participant R as Redis
    participant PB as Pod B (owns socket)
    participant CP as Charger

    A->>PA: POST /admin/charge-points/{cp_id}/commands/remote-start
    PA->>R: GET charger:{cp_id}:pod
    alt no entry
        PA-->>A: 409 charger offline
    else owner = pod B
        PA->>R: PUBLISH cmd:podB {request_id, action, payload}
        R->>PB: deliver
        PB->>PB: map action to version-specific call
        PB->>CP: 1.6 RemoteStartTransaction / 2.0.1 RequestStartTransaction
        CP-->>PB: Accepted / Rejected
        PB->>R: PUBLISH reply:{request_id} result
        R->>PA: deliver
        PA-->>A: 200 result
    end
    Note over PA: Timeout waiting for reply then 504.<br/>Reply channel and timeout are (proposed).
```

Commands are expressed internally in version-neutral terms and translated at the owning pod:

| Internal action | OCPP 1.6 | OCPP 2.0.1 |
|---|---|---|
| `remote_start` | `RemoteStartTransaction` | `RequestStartTransaction` |
| `remote_stop` | `RemoteStopTransaction` | `RequestStopTransaction` |
| `reset` | `Reset` (Hard / Soft) | `Reset` (Immediate / OnIdle, optional `evseId`) |
| `unlock_connector` | `UnlockConnector` (connectorId) | `UnlockConnector` (evseId + connectorId) |
| `change_availability` | `ChangeAvailability` (connectorId, Operative/Inoperative) | `ChangeAvailability` (optional evse{id, connectorId}, operationalStatus) |
| `clear_cache` | `ClearCache` | `ClearCache` |

## UPI payment to charging session

```mermaid
sequenceDiagram
    autonumber
    participant D as Driver app
    participant API as cms-api
    participant GW as UPI gateway
    participant PG as Postgres
    participant CP as Charger

    D->>API: scan QR, POST /sessions (connector, amount)
    API->>GW: create payment order
    API-->>D: order / intent
    D->>GW: pay via UPI app
    GW->>API: POST /payments/upi/webhook (signed)
    API->>API: verify signature
    API->>PG: insert payment keyed on gateway event id
    alt duplicate webhook
        API-->>GW: 200 (no-op)
    else first delivery
        API->>PG: mint pre-authorized id_token tied to payment
        API-->>GW: 200
        API->>CP: remote_start with minted token (routed via Redis)
        CP-->>API: Accepted
        opt charger configured to authorize remote starts
            CP->>API: Authorize (token)
            API-->>CP: Accepted
        end
        CP->>API: StartTransaction / TransactionEvent Started
    end
```

Notes:

- **Do not depend on the `Authorize` step.** Chargers only send it after a remote start if `AuthorizeRemoteTxRequests` (1.6) or `AuthCtrlr.AuthorizeRemoteStart` (2.0.1) is true, and many default to false. The token must be valid either way. Set the key explicitly at boot so behaviour is predictable fleet-wide.
- **Webhook idempotency** is enforced by a unique constraint on the gateway's event / payment ID, not by an application-level check.
- **Reconciliation job (required):** finds payments marked successful with no transaction started within N minutes and either retries the start or triggers a refund.
- Tariff calculation and receipt generation happen at transaction end and are not shown.

## Token revocation

Blocking a token in the database is not enough. Chargers may still accept it from their authorization cache or local list.

```mermaid
sequenceDiagram
    autonumber
    participant ADM as Admin
    participant API as cms-api
    participant PG as Postgres
    participant R as Redis
    participant CP as Affected chargers

    ADM->>API: POST /admin/id-tokens/{id}/block
    API->>PG: update id_tokens status = Blocked
    API->>R: DEL auth cache for token
    alt token is on a local auth list
        API->>CP: SendLocalList (differential update, token Blocked)
    end
    API->>CP: ClearCache
    Note over API,CP: Offline chargers receive the push on reconnect (proposed).<br/>Until then they can still honour the token up to its expiryDate.
```

Revocation lag is bounded by `idTokenInfo.expiryDate` for offline chargers. The expiry window is a security/resilience trade-off (see [open decisions](../open-decisions.md)).

## Graceful drain during a rolling deploy

```mermaid
sequenceDiagram
    autonumber
    participant K as Kubernetes
    participant P as Old pod
    participant LB as Load balancer
    participant R as Redis
    participant CP as Charger
    participant N as New pod

    K->>P: preStop hook (sleep 2–3 s)
    K->>P: SIGTERM
    P->>P: readiness = failing
    LB->>P: health check fails, stop routing new connections
    P->>P: wait ~5 s, then drain window 60–90 s
    P->>CP: close(1001 going away)
    P->>R: compare-and-delete charger:{cp_id}:pod
    CP->>LB: reconnect with backoff
    LB->>N: route
    N->>R: SET charger:{cp_id}:pod = new pod
```
