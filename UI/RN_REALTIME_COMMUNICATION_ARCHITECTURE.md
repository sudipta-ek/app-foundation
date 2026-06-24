# React Native Real-Time Communication Architecture (Critical)

## 1. Objective
Define a resilient real-time architecture for airline operational events (gate changes, delays, boarding state), with deterministic behavior under unstable airport networks.

This document expands **Layer 2: Real-Time Communication Architecture (Critical)** from [UI/REACT_NATIVE_FOUNDATION.md](UI/REACT_NATIVE_FOUNDATION.md).

---

## 2. Scope
- Mobile client real-time channel lifecycle
- Event subscription and routing model
- Connection resilience and offline replay behavior
- Security, observability, and governance controls

---

## 3. High-Level Topology

```text
React Native App
  ↕ (WSS)
WebSocket Gateway
  → Event Backbone (Solace)
  → BFF Services / Domain Producers
```

---

## 4. Event Categories

## 4.1 Critical Operational
- `gate-changed`
- `boarding-started`
- `boarding-closed`
- `flight-cancelled`

## 4.2 Informational Operational
- `flight-delayed`
- `passenger-boarded`
- `passenger-missed`

## 4.3 Control/Health
- `connection-state-changed`
- `subscription-acked`
- `heartbeat-timeout`

---

## 5. Client Architecture

## 5.1 Components
- Realtime transport client (`Socket.io`/WSS wrapper)
- Subscription manager
- Event router (domain dispatch)
- Deduplication/idempotency guard
- Offline event gap detector

## 5.2 Suggested Structure

```text
libs/
  realtime/
    src/
      client/
        realtime-client.ts
        connection-manager.ts
      subscriptions/
        subscription-registry.ts
        subscription-policy.ts
      routing/
        event-router.ts
        event-handlers.ts
      reliability/
        event-deduplicator.ts
        reconnect-backoff.ts
        missed-event-recovery.ts
      telemetry/
        realtime-telemetry.ts
      types/
        realtime-event.types.ts
      index.ts
```

---

## 6. Connection Lifecycle
1. Authenticate and mint realtime token
2. Open secure socket (`wss` only)
3. Subscribe using role/airport scoped topics
4. Process events with idempotency checks
5. On disconnect, retry with exponential backoff
6. On reconnect, perform missed-event recovery by cursor/timestamp

---

## 7. Reliability Standards
- Exponential backoff with jitter
- Max reconnect window configurable by environment
- Heartbeat monitoring and stale-connection reset
- Event ordering preserved per key (`flightId`, `gateId`)
- At-least-once delivery expected; client enforces idempotency

---

## 8. State Integration
- Realtime connection metadata in Redux (`status`, `lastConnectedAt`, `retryCount`)
- Event payload updates through feature reducers/actions
- Missed-event recovery may trigger TanStack Query invalidation/refetch

---

## 9. Security Controls
- Realtime auth token short-lived and scoped
- Topic authorization enforced server-side
- No PII in channel names or debug logs
- TLS 1.3 in transit
- Device identity context attached where required

---

## 10. Observability
Mandatory telemetry:
- `realtime_connect_start`
- `realtime_connected`
- `realtime_disconnected`
- `realtime_reconnect_attempt`
- `realtime_event_received`
- `realtime_event_dropped_duplicate`
- `realtime_recovery_invoked`

Mandatory attributes:
- `correlationId`
- `traceId`
- `airportId`
- `connectionId`
- `eventType`
- `latencyMs`

---

## 11. Testing Strategy
- Unit: router, deduplicator, backoff policy
- Integration: subscribe/reconnect/recovery flow with mock gateway
- E2E: network drop during boarding, event recovery verification
- Chaos: packet loss, delayed heartbeat, duplicate event bursts

---

## 12. DoD (Real-Time Feature)
- Subscription contract documented
- Idempotency behavior verified
- Reconnect + recovery tested
- Security review complete
- Telemetry visible in dashboard
- Operational runbook updated
