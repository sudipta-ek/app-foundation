# React Native Event-Driven Architecture

## 1. Objective
Define how the mobile platform consumes and emits operational events with traceability, schema governance, and resilience.

This document expands **Layer 4: Event-Driven Architecture** from [UI/REACT_NATIVE_FOUNDATION.md](UI/REACT_NATIVE_FOUNDATION.md).

---

## 2. Event Backbone Context
- Primary event backbone: Solace PubSub+
- Events used for operational propagation, not as audit store
- Contracts governed by AsyncAPI + schema registry

## 2.1 Implementation Guidelines
- Keep Solace topic taxonomy and event naming aligned with AsyncAPI definitions.
- Treat mobile as event consumer/edge producer only where explicitly approved.
- Do not use event streams as an audit source of truth; audit must remain server-side.
- Enforce environment isolation (`dev`, `sit`, `uat`, `prod`) with separate topic domains and credentials.
- Define domain ownership for each event family before implementation starts.

## 2.2 Folder Structure

```text
docs/
  events/
    asyncapi/
      boarding.asyncapi.yaml
      flights.asyncapi.yaml
    ownership/
      event-ownership-matrix.md
    taxonomy/
      solace-topic-taxonomy.md

libs/
  events/
    src/
      config/
        event-backbone.config.ts         # broker endpoints, env mapping, topic roots
```

---

## 3. Event Flow

```text
Operational Action (check-in/boarding)
  → Event Produced (BFF/service)
  → Solace Topic
  → Consumers (boarding, notifications, analytics, audit integration)
```

## 3.1 Implementation Guidelines
- Model each flow as `producer -> topic -> consumer` with explicit owner and fallback behavior.
- Use correlation propagation from source action to all downstream handlers.
- Define failure path for each consumer: retry, dead-letter, or reject.
- Keep consumer side-effects idempotent and independently recoverable.
- Document event ordering guarantees per entity (`flightId`, `passengerId`) in runbooks.

## 3.2 Folder Structure

```text
docs/
  events/
    flows/
      boarding-scan.event-flow.md
      gate-change.event-flow.md
      passenger-missed.event-flow.md

libs/
  events/
    src/
      flow/
        event-flow.types.ts              # producer/consumer metadata contracts
        event-routing-map.ts             # maps eventType -> handler pipeline
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

## 4.1 Implementation Guidelines
- Define contract first in AsyncAPI, then generate/implement TypeScript types.
- Keep envelope mandatory and immutable across all domains.
- Allow additive fields only within same major version.
- Reject malformed or unknown-required-field payloads at validation boundary.
- Maintain compatibility tests for `current` and `previous` versions during migration windows.

## 4.2 Folder Structure

```text
libs/
  events/
    src/
      contracts/
        v1/
          boarding-passenger-boarded.contract.ts
          flight-gate-changed.contract.ts
        v2/
          boarding-passenger-boarded.contract.ts
      envelope/
        event-envelope.types.ts
        event-envelope.schema.ts
      versioning/
        contract-version.policy.ts
        compatibility-matrix.ts

docs/
  events/
    schemas/
      boarding-passenger-boarded.v1.json
      flight-gate-changed.v1.json
```

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

## 5.1 Implementation Guidelines
- Keep consumption pipeline ordered: receive -> validate -> deduplicate -> route -> handle -> observe.
- Separate transport concerns (`realtime`) from domain event logic (`events`).
- Use one handler per domain event group; avoid monolithic handler files.
- Keep feature state updates inside domain reducers/actions, not inside transport layer.
- Ensure handler outputs are typed (`accepted`, `rejected`, `deferred`) for operational metrics.

## 5.2 Folder Structure

```text
libs/
  realtime/
    src/
      client/
        realtime-client.ts
      subscriptions/
        subscription-registry.ts

  events/
    src/
      ingestion/
        event-ingress.service.ts         # receives raw event from realtime
      validation/
        event-schema-validator.ts
      routing/
        event-router.ts
      handlers/
        boarding.events.handler.ts
        checkin.events.handler.ts
        flight.events.handler.ts
      results/
        handler-result.types.ts
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

## 6.1 Implementation Guidelines
- Implement processing as deterministic middleware stages.
- Use per-entity ordering queues only where business-critical; avoid global serialization bottlenecks.
- Apply idempotency TTL retention policy to balance correctness and memory/storage usage.
- Send reject reasons with taxonomy (`schema_invalid`, `duplicate`, `unsupported_version`, `handler_error`).
- Define retry policy separately for transient vs non-transient failures.

## 6.2 Folder Structure

```text
libs/
  events/
    src/
      pipeline/
        process-event.pipeline.ts        # stage orchestration
        process-event.context.ts
      ordering/
        entity-ordering-queue.ts
      idempotency/
        idempotency-policy.ts
      retry/
        retry-policy.ts
      rejects/
        reject-reason.types.ts
```

---

## 7. Event Persistence and Replay (Client-side)
- Optional short-term local checkpoint for recovery cursor
- No long-term event ledger stored in app for compliance (audit remains server-side)
- Recovery flow replays from last known cursor/timestamp

## 7.1 Implementation Guidelines
- Persist only lightweight recovery checkpoints (cursor, timestamp, topic).
- Keep replay window bounded and configurable by environment.
- Use replay only for recovery; avoid using it as normal state synchronization path.
- On replay completion, force reconciliation check with server state for critical entities.
- Capture replay outcome metrics (`replayed`, `skipped`, `failed`, `duration`).

## 7.2 Folder Structure

```text
libs/
  events/
    src/
      recovery/
        replay-controller.ts
        replay-request.builder.ts
        replay-reconciliation.ts
      checkpoint/
        event-checkpoint.store.ts
        event-checkpoint.types.ts
      persistence/
        checkpoint-realm.adapter.ts
```

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

## 9.1 Implementation Guidelines
- Emit one telemetry event per processing stage and aggregate counters per event type.
- Use shared observability SDK; do not log free-text event payloads.
- Include environment, app version, and handler identity on failures.
- Set alerts for validation spikes, handler failure rates, and replay error threshold.
- Correlate event telemetry with API/realtime telemetry using common `correlationId`.

## 9.2 Folder Structure

```text
libs/
  events/
    src/
      telemetry/
        event-telemetry.ts
        event-metrics.ts
        event-logging.policy.ts
      dashboards/
        event-observability.kpi.ts       # KPI definitions used by dashboards
```

---

## 10. Security Controls
- Topic-level authorization
- Schema validation to block malformed payloads
- PII minimization in event payloads
- Sensitive fields masked before client logs

## 10.1 Implementation Guidelines
- Enforce allow-list topic subscription policy per role/airport context.
- Validate schemas before deserialization into domain models.
- Apply field-level redaction policies before emitting logs/telemetry.
- Reject unauthorized event scopes and trigger security telemetry.
- Rotate credentials/tokens for event transport by environment policy.

## 10.2 Folder Structure

```text
libs/
  events/
    src/
      security/
        topic-authorization.policy.ts
        payload-redaction.policy.ts
        schema-security.validator.ts
        security-events.ts

  realtime/
    src/
      security/
        realtime-auth-token.service.ts
        subscription-scope.guard.ts
```

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
