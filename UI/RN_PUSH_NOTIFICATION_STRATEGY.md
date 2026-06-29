# React Native Push Notification Strategy

## 1. Objective
Define a governed push notification model for airline operations across foreground, background, and terminated states.

This document expands **Layer 3: Push Notification Strategy** from [UI/REACT_NATIVE_FOUNDATION.md](UI/REACT_NATIVE_FOUNDATION.md).

---

## 2. Notification Channels
- iOS: APNS
- Android: FCM
- Provider abstraction via BFF notification service

## 2.1 Implementation Guidelines
- Keep provider specifics behind a single BFF notification abstraction.
- Use environment-isolated credentials/certificates per stage (`dev`, `sit`, `uat`, `prod`).
- Bind notification channel usage to role/airport policy where required.
- Never expose provider secrets or certificate metadata in client logs.

## 2.2 Folder Structure

```text
libs/
  notifications/
    src/
      channels/
        apns.channel.ts                  # iOS channel adapter contract
        fcm.channel.ts                   # Android channel adapter contract
        provider.types.ts
      config/
        notification-channel.config.ts   # env + provider config mapping
```

---

## 3. Notification Types
- Gate change
- Flight delay/cancel
- Boarding start/close
- Passenger missed
- Security alert
- Planned maintenance

Priority guidance:
- High: cancellation, security, boarding-close
- Medium: gate change, boarding-start
- Low: non-urgent operational info

## 3.1 Implementation Guidelines
- Define notification taxonomy as versioned constants and avoid ad-hoc strings.
- Map each type to delivery priority, TTL, and target UX behavior.
- Keep type ownership with domain teams and approval via platform governance.
- Enforce backward-compatible changes for existing types.

## 3.2 Folder Structure

```text
libs/
  notifications/
    src/
      taxonomy/
        notification-type.enum.ts
        notification-priority.map.ts
        notification-ttl.policy.ts
      governance/
        notification-type-ownership.md
```

---

## 4. Delivery Behavior Matrix

| App State | Behavior |
|---|---|
| Foreground | In-app presentation + optional local banner |
| Background | System notification |
| Terminated | System notification + deep-link entry |

---

## 5. Architecture

```text
Domain Event
  → BFF Notification Orchestrator
  → Push Provider (FCM/APNS)
  → Device Notification Handler
  → Deep Link Router / Feature Screen
```

## 5.1 Implementation Guidelines
- Keep orchestration logic server-side; client handles rendering/routing only.
- Correlation IDs must propagate from domain event to push payload.
- Enforce auth/session checks before processing deep links.
- Separate notification ingestion from navigation side effects.

## 5.2 Folder Structure

```text
libs/
  notifications/
    src/
      architecture/
        notification-orchestration.types.ts
        notification-processing.pipeline.ts
      ingestion/
        notification-ingress.service.ts
      navigation/
        notification-navigation.service.ts
```

---

## 6. Client Structure

```text
libs/
  notifications/
    src/
      registration/
        token-registration.service.ts
      handlers/
        foreground.handler.ts
        background.handler.ts
        opened-app.handler.ts
      routing/
        notification-deeplink.mapper.ts
      preferences/
        notification-preferences.slice.ts
      telemetry/
        notification-events.ts
      index.ts
```

## 6.1 Implementation Guidelines
- Keep client structure layered: registration -> handler -> routing -> preferences -> telemetry.
- Handlers must be side-effect controlled and return typed outcomes.
- Route resolution should be centralized in one mapping service.
- Preferences must be read/write through a single state boundary.

## 6.2 Folder Structure (Expanded)

```text
libs/
  notifications/
    src/
      registration/
        permission.service.ts
        token-registration.service.ts
        token-rotation.service.ts
      handlers/
        foreground.handler.ts
        background.handler.ts
        opened-app.handler.ts
        handler-result.types.ts
      routing/
        notification-deeplink.mapper.ts
        notification-route.guard.ts
      preferences/
        notification-preferences.slice.ts
        notification-preferences.selectors.ts
      payload/
        notification-payload.types.ts
        notification-payload.validator.ts
      telemetry/
        notification-events.ts
        notification-metrics.ts
      index.ts
```

---

## 7. Core Flows

## 7.1 Registration
1. Ask permission with rationale
2. Retrieve provider token
3. Bind token to `userId + deviceId + airport`
4. Rotate token on refresh/logout/device change

## 7.2 Receive (Foreground)
1. Parse payload
2. Validate schema/version
3. Show in-app UX
4. Route to relevant screen if user acts

## 7.3 Open from Notification
1. Validate payload signature/shape
2. Resolve deep link
3. Perform auth/session gate
4. Navigate to feature context

## 7.4 Implementation Guidelines
- Registration flow must support permission denial and re-prompt strategy by policy.
- Token lifecycle must handle refresh and invalidation atomically.
- Foreground handling must avoid navigation side effects unless user action is explicit.
- Open-flow must always pass through auth + permission guards before navigation.
- Store flow outcomes as telemetry events for operational diagnostics.

## 7.5 Folder Structure

```text
libs/
  notifications/
    src/
      flows/
        registration.flow.ts
        foreground-receive.flow.ts
        open-from-notification.flow.ts
      guards/
        notification-auth.guard.ts
        notification-permission.guard.ts
```

---

## 8. Payload Contract
Required fields:
- `notificationType`
- `title`
- `message`
- `airportId`
- `correlationId`
- `eventVersion`
- `deeplink`

Rules:
- No sensitive PII in title/body
- Stable versioned payload schema
- TTL defined by notification category

## 8.1 Implementation Guidelines
- Validate payloads at ingress using schema validators before UI handling.
- Keep contract version in payload and support `current` + `previous` during migration.
- Use typed mapping from payload -> domain notification model.
- Reject malformed payloads with explicit error reason and telemetry.

## 8.2 Folder Structure

```text
libs/
  notifications/
    src/
      payload/
        schemas/
          notification-payload.v1.schema.ts
          notification-payload.v2.schema.ts
        notification-payload.types.ts
        notification-payload.mapper.ts
        notification-payload.validator.ts
```

---

## 9. Security and Compliance
- Permission prompts auditable
- Token storage in secure storage only
- Server-side authorization before target send
- Payload redaction policy (PII-safe)
- Notification click events audited where required

## 9.1 Implementation Guidelines
- Store push tokens only in secure storage and rotate per session/device policy.
- Redact restricted fields before logs/telemetry emission.
- Capture audit-safe records for permission prompt outcomes and notification opens.
- Validate notification scopes against user/airport context before routing.
- Define incident handling for suspicious push payload patterns.

## 9.2 Folder Structure

```text
libs/
  notifications/
    src/
      security/
        token-secure-store.ts
        payload-redaction.policy.ts
        notification-scope.guard.ts
      compliance/
        permission-audit.service.ts
        notification-audit-events.ts
```

---

## 10. Reliability and SLAs
- Target delivery SLA aligned with foundation thresholds
- Retry strategy at orchestrator layer
- Dead-letter handling for failed sends
- Provider outage fallback: in-app polling for critical events

---

## 11. Observability
Mandatory events:
- `push_permission_prompted`
- `push_permission_result`
- `push_token_registered`
- `push_received`
- `push_opened`
- `push_delivery_failed`

Attributes:
- `platform`, `notificationType`, `airportId`, `correlationId`, `traceId`

## 11.1 Implementation Guidelines
- Emit stage-level telemetry (prompt, receive, open, route, fail).
- Use structured telemetry schema and avoid free-text payload dumps.
- Add alerts for token registration failures and delivery/open failure spikes.
- Correlate push telemetry with realtime/event traces via `correlationId`.

## 11.2 Folder Structure

```text
libs/
  notifications/
    src/
      telemetry/
        notification-events.ts
        notification-metrics.ts
        notification-observability.kpi.ts
        notification-alert-thresholds.ts
```

---

## 12. Testing
- Unit: payload validation, deep-link mapping
- Integration: registration and open-routing
- E2E: foreground/background/terminated scenarios
- Device farm: iOS/Android version matrix

---

## 13. DoD
- Payload schema versioned and approved
- Permission and token lifecycle tested
- Deep links validated for auth-safe navigation
- PII checks passed
- Monitoring dashboard and alerts enabled
