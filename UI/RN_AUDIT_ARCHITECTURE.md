# Audit Architecture — Implementation Guide

**Aligned with**: [UI/REACT_NATIVE_FOUNDATION.md](UI/REACT_NATIVE_FOUNDATION.md) — Layer 15

---

## 1. Audit vs Analytics vs Logging

| Dimension | Audit | Analytics | Logging |
|---|---|---|---|
| Purpose | Regulatory accountability — who did what | Business/UX insights | Operational diagnostics |
| Mutability | Immutable — append-only WORM storage | Aggregated / archivable | Mutable / rolling retention |
| Retention | 7 years minimum | 30–90 days | 30–90 days |
| PII | Hashed IDs only — no raw PII | No PII | No PII |
| Destination | Immutable Audit Store (WORM) | Analytics platform (Mixpanel/Amplitude) | Log aggregation (Datadog/Azure Monitor) |
| Deletion | Cannot be deleted | Can be archived | Rolling TTL |
| Required by | Regulatory compliance | Product/business | Engineering ops |
| Owner | Platform + Compliance | Analytics team | Platform Team |

Rules:
- Audit SDK, Analytics SDK, and Logger are **three separate libraries** — never co-mingled.
- Feature code emits to all three through typed interfaces; never calls transport APIs directly.
- Audit events must not be conditionally emitted — every regulated action must produce an audit record.

---

## 2. Audit Event Taxonomy

```typescript
// libs/audit/src/events/audit-event.types.ts
export type AuditEventType =
  | 'session.login'
  | 'session.logout'
  | 'session.timeout'
  | 'session.forced_logout'
  | 'boarding.scan'
  | 'boarding.reversal'
  | 'boarding.manual_override'
  | 'checkin.completed'
  | 'checkin.document_check'
  | 'passenger.search'
  | 'document.scan'              // passport, ID scan
  | 'config.change'
  | 'security.alert'
  | 'permission.denied'
  | 'sync.conflict_resolution';

export interface AuditEvent {
  eventType: AuditEventType;
  timestamp: string;             // ISO 8601 UTC
  correlationId: string;
  traceId: string;
  userId: string;                // hashed — never raw
  deviceId: string;
  airportCode: string;
  gateId?: string;
  flightId?: string;
  passengerId?: string;          // hashed — never raw
  action: string;                // human-readable action description
  result: 'success' | 'failure' | 'denied';
  metadata?: Record<string, unknown>; // additional context — PII-free
  appVersion: string;
  sessionId: string;
}
```

PII rules for audit:
- `userId`, `passengerId`: SHA-256 hash — raw values prohibited.
- Passenger name, passport number, DOB: prohibited in all audit fields.
- `metadata` object: reviewed by Platform Team before new fields added.

---

## 3. Audit SDK Implementation

```typescript
// libs/audit/src/audit.client.ts
import type { AuditEvent, AuditEventType } from './events/audit-event.types';
import { AuditLocalBuffer } from './buffer/audit-local-buffer';
import { AuditTransport } from './transport/audit.transport';
import { hashUserId } from './utils/pii-hash';
import { correlationService } from '@airline/observability';

class AuditClient {
  private buffer = new AuditLocalBuffer();
  private transport = new AuditTransport();

  emit(
    eventType: AuditEventType,
    payload: Omit<AuditEvent, 'eventType' | 'timestamp' | 'correlationId' | 'traceId' | 'appVersion'>
  ): void {
    const event: AuditEvent = {
      eventType,
      timestamp: new Date().toISOString(),
      correlationId: correlationService.getCorrelationId(),
      traceId: correlationService.getTraceId(),
      appVersion: APP_VERSION,
      // Enforce PII hashing at SDK boundary
      userId: hashUserId(payload.userId),
      passengerId: payload.passengerId ? hashUserId(payload.passengerId) : undefined,
      ...payload,
    };

    // Always buffer locally first (offline safety)
    this.buffer.append(event);

    // Attempt immediate transport
    this.transport.send(event).catch(() => {
      // Buffered — will retry on next sync cycle
    });
  }
}

export const auditClient = new AuditClient();
```

---

## 4. Audit Record Schema (Transport Payload)

```json
{
  "eventType": "boarding.scan",
  "timestamp": "2026-07-01T08:42:00.000Z",
  "correlationId": "9cbff51e-7ff5-4acd-b710-f1944fcbf001",
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "sessionId": "sess-abc123",
  "userId": "sha256:a3f8c2...",
  "deviceId": "ipad-DXB-B12-001",
  "deviceType": "ipad",
  "airportCode": "DXB",
  "worksite":"checkin",
  "location":"Terminal-3",
  "gateId": "B12",
  "flightId": "EK0001-02072026-DXB",
  "passengerId": "sha256:9d1f44...",
  "action": "Passenger boarding scan",
  "result": "success",
  "appVersion": "2026.07.0",
  "metadata": {
    "scanMethod": "barcode",
    "boardingSequence": 42
  }
}
```

---

## 5. Transport & Storage Pipeline

```text
Feature Code
  → auditClient.emit(eventType, payload)
      → PII hashing enforced at SDK boundary
      → AuditLocalBuffer.append(event)         [Realm — encrypted, offline-safe]
      → AuditTransport.send(event)
            → BFF /v1/audit/events              [HTTP POST + correlationId + authToken]
                → Audit BFF Service
                    → Solace audit topic        [airline/*/audit/event/v1]
                        → Audit Consumer Service
                            → WORM Storage      [Azure Immutable Blob / S3 Object Lock]
```

**BFF Audit Endpoint:**
- `POST /v1/audit/events` — accepts batch of up to 50 events.
- Returns `202 Accepted` — async write to WORM.
- Correlation ID echoed back in response for tracing.

**WORM Storage:**
- Azure Immutable Blob Storage with time-based immutability policy (7 years).
- S3 Object Lock (COMPLIANCE mode) on AWS environments.
- Encrypted at rest (AES-256).
- Access restricted to Compliance Team (read-only) and Audit Consumer Service (write-only).

---

## 6. Offline Audit Buffer

When network is unavailable, audit events are queued in Realm and flushed on reconnect.

```typescript
// libs/audit/src/buffer/audit-local-buffer.ts
import Realm from 'realm';

class AuditEventSchema extends Realm.Object {
  static schema = {
    name: 'AuditEvent',
    primaryKey: 'correlationId',
    properties: {
      correlationId: 'string',
      payload: 'string',         // JSON serialised AuditEvent
      createdAt: 'date',
      synced: { type: 'bool', default: false },
    },
  };
}

export class AuditLocalBuffer {
  private realm = openAuditRealm();  // separate encrypted Realm for audit

  append(event: AuditEvent): void {
    this.realm.write(() => {
      this.realm.create('AuditEvent', {
        correlationId: event.correlationId,
        payload: JSON.stringify(event),
        createdAt: new Date(),
        synced: false,
      });
    });
  }

  getPending(): AuditEvent[] {
    return this.realm.objects('AuditEvent')
      .filtered('synced = false')
      .map(r => JSON.parse(r.payload));
  }

  markSynced(correlationIds: string[]): void {
    this.realm.write(() => {
      correlationIds.forEach(id => {
        const record = this.realm.objectForPrimaryKey('AuditEvent', id);
        if (record) record.synced = true;
      });
    });
  }
}
```

Sync trigger:
```typescript
// libs/audit/src/sync/audit-sync.service.ts
export async function flushAuditBuffer(): Promise<void> {
  const pending = auditLocalBuffer.getPending();
  if (pending.length === 0) return;

  const batches = chunk(pending, 50);
  for (const batch of batches) {
    const result = await auditTransport.sendBatch(batch);
    if (result.ok) {
      auditLocalBuffer.markSynced(batch.map(e => e.correlationId));
    }
  }
}
// Called by network restore handler and on app foreground
```

---

## 7. Analytics Hooks

Analytics tracks user behaviour for product/UX insights — distinct from audit.

```typescript
// libs/analytics/src/analytics.client.ts
export type AnalyticsEvent =
  | 'screen_view'
  | 'boarding_scan_initiated'
  | 'boarding_scan_completed'
  | 'search_executed'
  | 'feature_used'
  | 'error_displayed';

export interface AnalyticsPayload {
  event: AnalyticsEvent;
  screenId?: string;
  duration?: number;
  success?: boolean;
  [key: string]: unknown;  // additional dimensions — PII-free only
}

class AnalyticsClient {
  track(payload: AnalyticsPayload): void {
    // PII strip applied before emission
    const sanitised = piiStripper.strip(payload);
    analyticsProvider.track(sanitised);  // Mixpanel / Amplitude adapter
  }

  screenView(screenId: string, properties?: Record<string, unknown>): void {
    this.track({ event: 'screen_view', screenId, ...properties });
  }
}

export const analyticsClient = new AnalyticsClient();
```

```typescript
// Usage in feature screen (via hook — never direct)
// libs/analytics/src/hooks/use-screen-analytics.hook.ts
import { useFocusEffect } from '@react-navigation/native';
import { analyticsClient } from '../analytics.client';

export function useScreenAnalytics(screenId: string): void {
  useFocusEffect(() => {
    analyticsClient.screenView(screenId);
  });
}
```

Rules:
- `analyticsClient.track()` is the only analytics entry point for feature code.
- Feature code must not import analytics provider SDKs (Mixpanel, Amplitude) directly.
- Every new analytics event schema must be reviewed by Platform Team.

---

## 8. Logging Integration

```typescript
// libs/observability/src/logging/structured-logger.ts
export type LogLevel = 'DEBUG' | 'INFO' | 'WARN' | 'ERROR' | 'FATAL';

export interface LogEntry {
  level: LogLevel;
  message: string;
  correlationId: string;
  traceId: string;
  component: string;
  metadata?: Record<string, unknown>;  // PII-free
}

class StructuredLogger {
  private minLevel: LogLevel;

  constructor(component: string) {
    this.component = component;
    this.minLevel = getEnvironmentLogLevel();
  }

  info(message: string, metadata?: Record<string, unknown>): void {
    this.log('INFO', message, metadata);
  }

  warn(message: string, metadata?: Record<string, unknown>): void {
    this.log('WARN', message, metadata);
  }

  error(message: string, error: Error, metadata?: Record<string, unknown>): void {
    this.log('ERROR', message, {
      errorMessage: error.message,
      errorName: error.name,
      // stack only in non-prod
      ...(isDev() ? { stack: error.stack } : {}),
      ...metadata,
    });
  }

  private log(level: LogLevel, message: string, metadata?: Record<string, unknown>): void {
    if (!this.shouldLog(level)) return;

    const entry: LogEntry = {
      level,
      message,
      correlationId: correlationService.getCorrelationId(),
      traceId: correlationService.getTraceId(),
      component: this.component,
      metadata: piiStripper.strip(metadata ?? {}),
    };

    // In production: OTLP export. In dev: console.
    logExporter.export(entry);
  }
}

export function createLogger(component: string): StructuredLogger {
  return new StructuredLogger(component);
}
```

Usage:
```typescript
// In a feature service — never console.log
const logger = createLogger('boarding.scan-service');
logger.info('Scan initiated', { flightId, gateId });
logger.error('Scan failed', error, { flightId });
```

---

## 9. PII-Safe Rules

| Data | Audit | Analytics | Log |
|---|---|---|---|
| userId | SHA-256 hash only | Prohibited | Prohibited |
| passengerId | SHA-256 hash only | Prohibited | Prohibited |
| Passenger name | Prohibited | Prohibited | Prohibited |
| Passport / doc number | Prohibited | Prohibited | Prohibited |
| Date of birth | Prohibited | Prohibited | Prohibited |
| Seat number | Prohibited | Prohibited | Prohibited |
| flightId | Permitted | Permitted | Permitted |
| airportCode | Permitted | Permitted | Permitted |
| gateId | Permitted | Permitted | Permitted |
| deviceId | Permitted | Prohibited | Permitted |
| Action result | Permitted | Permitted | Permitted |

PII stripper is applied at the SDK boundary — not at each call site.

---

## 10. Folder Structure

```text
libs/
  audit/
    src/
      events/
        audit-event.types.ts       # AuditEvent interface + AuditEventType union
      audit.client.ts              # auditClient singleton — feature code entry point
      buffer/
        audit-local-buffer.ts      # Realm-backed offline buffer
      transport/
        audit.transport.ts         # BFF HTTP transport + batch logic
      sync/
        audit-sync.service.ts      # flush on reconnect / foreground
      utils/
        pii-hash.ts                # SHA-256 user/passenger ID hashing
      __tests__/
        audit.client.spec.ts
        audit-local-buffer.spec.ts
        pii-hash.spec.ts

  analytics/
    src/
      analytics.client.ts          # analyticsClient singleton
      providers/
        mixpanel.adapter.ts
        amplitude.adapter.ts
      hooks/
        use-screen-analytics.hook.ts
        use-feature-analytics.hook.ts
      utils/
        pii-stripper.ts
      __tests__/

  observability/
    src/
      logging/
        structured-logger.ts       # createLogger() factory
        log-exporter.ts            # OTLP / console adapter
        log-level.policy.ts        # per-environment level config
      correlation/
        correlation.service.ts     # correlationId + traceId management
      __tests__/
```

---

## 11. Governance Gates (DoD)

An audit/analytics/logging implementation is compliant when:
- [ ] `auditClient.emit()` called for all regulated operational actions
- [ ] No raw PII in any audit, analytics, or log payload (PII-stripper test passing)
- [ ] Offline buffer verified — audit events survive network loss and flush on reconnect
- [ ] Analytics events declared in `AnalyticsEvent` type — no untyped strings
- [ ] Structured logger used — no `console.log` in feature code (ESLint `no-console` gate)
- [ ] Correlation ID present on all error and fatal log entries
- [ ] WORM storage immutability policy verified in target environment
- [ ] Audit retention (7 years) policy documented and applied
- [ ] Compliance Team sign-off for new PII-adjacent audit fields
