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

## 2.1 Implementation Guidelines
- Treat realtime as a platform capability (`libs/realtime`), not feature-owned transport code.
- Keep transport, subscription, routing, reliability, and telemetry separated.
- Define explicit non-goals: no business logic in transport/client layer.
- Enforce environment-specific configs and topic scopes.

## 2.2 Folder Structure

```text
libs/
  realtime/
    src/
      scope/
        realtime-scope.types.ts          # what realtime covers/does not cover
      config/
        realtime-env.config.ts           # env-specific realtime config
```

---

## 3. High-Level Topology

```text
React Native App
  ↕ (WSS)
WebSocket Gateway
  → Event Backbone (Solace)
  → BFF Services / Domain Producers
```

## 3.1 Implementation Guidelines
- Keep gateway URL discovery/config centralized.
- Use one authenticated socket per active session context.
- Propagate `correlationId`/`traceId` through inbound event metadata.
- Separate connectivity concerns from domain event handling.

## 3.2 Folder Structure

```text
libs/
  realtime/
    src/
      topology/
        topology.types.ts                # app/gateway/backbone metadata types
        gateway-endpoint.resolver.ts
      context/
        realtime-context.provider.tsx
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

## 4.4 Implementation Guidelines
- Keep event categories versioned and owned by platform governance.
- Define per-category priority and handling SLA.
- Map each category to handler pipeline and retry policy.
- Reject unknown categories with telemetry + safe fallback.

## 4.5 Folder Structure

```text
libs/
  realtime/
    src/
      categories/
        event-category.enum.ts
        event-category.policy.ts         # priority/SLA/retry mapping
        event-category.guard.ts
```

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

## 5.3 Implementation Guidelines
- `client` layer only handles connect/send/receive/close.
- `subscriptions` controls topic registration and scope checks.
- `routing` dispatches to domain-safe handlers only.
- `reliability` encapsulates deduplication, retry, replay logic.
- Use typed handler results (`accepted`, `duplicate`, `rejected`, `deferred`).

## 5.4 Folder Structure (Expanded)

```text
libs/
  realtime/
    src/
      client/
        realtime-client.ts
        connection-manager.ts
        socket-factory.ts
      subscriptions/
        subscription-registry.ts
        subscription-policy.ts
        subscription-scope.guard.ts
      routing/
        event-router.ts
        event-handlers.ts
        handler-result.types.ts
      reliability/
        event-deduplicator.ts
        reconnect-backoff.ts
        missed-event-recovery.ts
        replay-cursor.store.ts
      telemetry/
        realtime-telemetry.ts
      types/
        realtime-event.types.ts
        realtime-connection.types.ts
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

## 6.1 Implementation Guidelines
- Model lifecycle as finite states (`idle`, `connecting`, `connected`, `reconnecting`, `failed`).
- Gate connection by session validity and device compliance context.
- Re-subscribe idempotently after reconnect.
- Persist minimal recovery cursor for replay.
- Emit lifecycle telemetry on every state transition.

## 6.2 Folder Structure

```text
libs/
  realtime/
    src/
      lifecycle/
        connection-state.machine.ts
        lifecycle-orchestrator.ts
        reconnect-orchestrator.ts
      recovery/
        recovery-cursor.types.ts
        recovery-cursor.store.ts
```

---

## 7. Reliability Standards
- Exponential backoff with jitter
- Max reconnect window configurable by environment
- Heartbeat monitoring and stale-connection reset
- Event ordering preserved per key (`flightId`, `gateId`)
- At-least-once delivery expected; client enforces idempotency

## 7.1 Implementation Guidelines
- Keep retry strategy configurable by environment and release stage.
- Use per-entity ordering queues where ordering is business-critical.
- Apply deduplication with bounded TTL and storage constraints.
- Define stale heartbeat thresholds and hard reset policy.
- Track reliability KPIs (reconnect success rate, duplicate drop rate).

## 7.2 Folder Structure

```text
libs/
  realtime/
    src/
      reliability/
        reconnect-policy.ts
        heartbeat-monitor.ts
        ordering-queue.ts
        idempotency-store.ts
        reliability-kpi.ts
```

---

## 8. State Integration
- Realtime connection metadata in Redux (`status`, `lastConnectedAt`, `retryCount`)
- Event payload updates through feature reducers/actions
- Missed-event recovery may trigger TanStack Query invalidation/refetch

## 8.1 Implementation Guidelines
- Keep connection metadata in a dedicated realtime Redux slice.
- Route domain payloads to feature reducers; avoid generic global payload store.
- Trigger targeted query invalidation (not broad cache flush).
- Use selectors for UI indicators (`live`, `degraded`, `reconnecting`).

## 8.2 Folder Structure

```text
libs/
  realtime/
    src/
      state/
        realtime.slice.ts
        realtime.selectors.ts
        realtime.actions.ts

libs/
  features/
    boarding/
      src/
        events/
          boarding-realtime.reducer.ts
        queries/
          boarding-query-invalidation.ts
```

---

## 9. Security Controls
- Realtime auth token short-lived and scoped
- Topic authorization enforced server-side
- No PII in channel names or debug logs
- TLS 1.3 in transit
- Device identity context attached where required

## 9.1 Implementation Guidelines
- Mint short-lived realtime tokens with least-privilege scopes.
- Validate subscription scopes on client and server.
- Redact payload fragments from logs by policy.
- Bind session context (`userId`, `deviceId`, `airportId`) to connection metadata.
- Rotate/revoke tokens on logout, role change, or suspicious activity.

## 9.2 Folder Structure

```text
libs/
  realtime/
    src/
      security/
        realtime-token.service.ts
        subscription-authorization.guard.ts
        realtime-redaction.policy.ts
        security-events.ts
```

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

## 10.1 Implementation Guidelines
- Emit telemetry per stage: connect, subscribe, receive, dedupe, recover, fail.
- Keep structured logs only; no free-text payload dumps.
- Add alert thresholds for disconnect spikes and recovery failures.
- Correlate realtime metrics with API/event traces via shared IDs.

## 10.2 Folder Structure

```text
libs/
  realtime/
    src/
      telemetry/
        realtime-telemetry.ts
        realtime-metrics.ts
        realtime-alert-thresholds.ts
        realtime-observability.kpi.ts
```

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
