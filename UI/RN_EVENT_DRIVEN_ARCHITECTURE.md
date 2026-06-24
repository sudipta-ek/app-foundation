# React Native Event-Driven Architecture

## 1. Objective
Define how the mobile platform consumes and emits operational events with traceability, schema governance, and resilience.

This document expands **Layer 4: Event-Driven Architecture** from [UI/REACT_NATIVE_FOUNDATION.md](UI/REACT_NATIVE_FOUNDATION.md).

---

## 2. Event Backbone Context
- Primary event backbone: Solace PubSub+
- Events used for operational propagation, not as audit store
- Contracts governed by AsyncAPI + schema registry

---

## 3. Event Flow

```text
Operational Action (check-in/boarding)
  → Event Produced (BFF/service)
  → Solace Topic
  → Consumers (boarding, notifications, analytics, audit integration)
```

---

## 4. Event Contract Standards
- AsyncAPI-first before implementation
- Versioned naming: `{domain}.{entity}.{action}.v{major}`
- Backward-compatible evolution within same major
- Breaking changes require new major and migration plan

Required envelope:
- `eventId`
- `eventType`
- `eventVersion`
- `timestampUtc`
- `correlationId`
- `traceId`
- `airportId`
- `payload`

---

## 5. Client Event Consumption Model

```text
libs/
  realtime/
  events/
    src/
      contracts/
        boarding-passenger-boarded.v1.ts
      envelope/
        event-envelope.types.ts
      validation/
        event-schema-validator.ts
      handlers/
        boarding.events.handler.ts
        flight.events.handler.ts
      idempotency/
        processed-event-store.ts
      index.ts
```

---

## 6. Processing Rules
- Validate envelope + schema version before handling
- Enforce idempotency by `eventId`
- Preserve ordering per entity key when required
- Route unknown version to compatibility fallback or reject path
- Emit telemetry for accepted/rejected events

---

## 7. Event Persistence and Replay (Client-side)
- Optional short-term local checkpoint for recovery cursor
- No long-term event ledger stored in app for compliance (audit remains server-side)
- Recovery flow replays from last known cursor/timestamp

---

## 8. Governance Model

| Area | Rule | Owner |
|---|---|---|
| Contract lifecycle | AsyncAPI mandatory | Platform Team + Domain Owner |
| Versioning | Major-only break, minor additive | Event Owner Team |
| Consumer readiness | Compatibility tested before producer rollout | Consuming Team |
| Deprecation | Notice window and migration timeline | Platform Governance |

---

## 9. Observability
Mandatory metrics/events:
- `event_received_total`
- `event_validation_failed_total`
- `event_duplicate_dropped_total`
- `event_processing_latency_ms`
- `event_handler_failed_total`

All events include `correlationId` and `traceId`.

---

## 10. Security Controls
- Topic-level authorization
- Schema validation to block malformed payloads
- PII minimization in event payloads
- Sensitive fields masked before client logs

---

## 11. Testing
- Contract tests against schema registry
- Consumer compatibility tests per version
- Replay/recovery tests from synthetic outage windows
- Load tests for event burst handling

---

## 12. DoD
- Event contract merged and versioned
- Consumer handlers covered by integration tests
- Idempotency and ordering checks passed
- Telemetry and alerts configured
- Migration/rollback notes published
