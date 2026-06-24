# React Native Push Notification Strategy

## 1. Objective
Define a governed push notification model for airline operations across foreground, background, and terminated states.

This document expands **Layer 3: Push Notification Strategy** from [UI/REACT_NATIVE_FOUNDATION.md](UI/REACT_NATIVE_FOUNDATION.md).

---

## 2. Notification Channels
- iOS: APNS
- Android: FCM
- Provider abstraction via BFF notification service

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

---

## 9. Security and Compliance
- Permission prompts auditable
- Token storage in secure storage only
- Server-side authorization before target send
- Payload redaction policy (PII-safe)
- Notification click events audited where required

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
