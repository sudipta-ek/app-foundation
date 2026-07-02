# React Native Push Notification Strategy

## 1. Objective
Define a governed push notification model for airline operations across foreground, background, and terminated states.

This document expands **Layer 3: Push Notification Strategy** from [UI/REACT_NATIVE_FOUNDATION.md](UI/REACT_NATIVE_FOUNDATION.md).

---

## 1.1 Layer Ownership Model

The push notification capability spans both the BFF (`as-ops-bff`) and the UI (`as-ops-ui`). The table below is the authoritative boundary — no responsibility crosses layers without explicit justification.

| Responsibility | Layer | Location |
|---|---|---|
| FCM/APNS provider HTTP integration | **BFF** | `bff-notifications-core/dispatch/` |
| Provider credential management (Vault-sourced) | **BFF** | `bff-notifications-core/config/` |
| Audience resolution (token registry queries) | **BFF** | `bff-notifications-core/audience/` |
| Fan-out cap enforcement (10,000 tokens) | **BFF** | `bff-notifications-core/audience/AudienceFanOutGuard` |
| Notification orchestration & dispatch | **BFF** | `bff-notifications-core/dispatch/NotificationOrchestrator` |
| Server-side acknowledgement registry | **BFF** | `bff-notifications-core/acknowledgement/AcknowledgementRegistry` |
| Acknowledgement policy evaluation & suppression | **BFF** | `bff-notifications-core/acknowledgement/NotificationPolicyEvaluator` |
| Audit emission (dispatch, ack, errors) | **BFF** | `bff-audit-core` |
| Device token registration with BFF | **UI** | `libs/notifications/registration/` |
| Push permission prompt & lifecycle | **UI** | `libs/notifications/registration/permission.service.ts` |
| Token rotation on session change / logout | **UI** | `libs/notifications/registration/token-rotation.service.ts` |
| Foreground / background / terminated handlers | **UI** | `libs/notifications/handlers/` |
| Deep link routing from notification tap | **UI** | `libs/notifications/routing/` |
| Client-side acknowledgement (POST /v1/…/ack) | **UI** | `libs/notifications/acknowledgement/notification-ack.service.ts` |
| Local one-time deduplication (Realm-backed) | **UI** | `libs/notifications/acknowledgement/ack-store.ts` |
| Notification preferences state | **UI** | `libs/notifications/preferences/` |
| Payload type contracts & schema validation | **UI** | `libs/notifications/payload/` |
| Push telemetry & observability events | **UI** | `libs/notifications/telemetry/` |
| Notification type taxonomy (versioned constants) | **Shared Contract** | `libs/notifications/taxonomy/` |

> **Rule**: The UI layer never calls FCM/APNS HTTP APIs directly. All provider dispatch, audience resolution, and acknowledgement enforcement are BFF responsibilities. The UI only registers tokens, handles incoming messages, and calls the BFF acknowledgement endpoint.

---

## 2. Notification Channels
- iOS: APNS
- Android: FCM
- Provider HTTP integration is a **BFF concern** (`bff-notifications-core`); the UI layer receives device tokens and handles incoming push messages via the native OS push infrastructure — it does not call FCM/APNS HTTP APIs directly.

## 2.1 Implementation Guidelines
- Keep provider specifics behind a single BFF notification abstraction.
- Use environment-isolated credentials/certificates per stage (`dev`, `sit`, `uat`, `prod`).
- Bind notification channel usage to role/airport policy where required.
- Never expose provider secrets or certificate metadata in client logs.

## 2.2 BFF Layer — Provider Integration (bff-notifications-core)

FCM and APNS HTTP API integration, credential management, and dispatch routing are BFF responsibilities. The UI layer never calls provider HTTP APIs directly.

```text
bff-notifications-core/                          # Maven module — as-ops-bff
  src/main/java/com/as/ops/bff/notifications/
    dispatch/
      NotificationOrchestrator.java              # Entry point: receives trigger, drives full dispatch flow
      NotificationDispatcher.java                # Routes payload to FCM or APNS provider client
      FcmProviderClient.java                     # FCM HTTP v1 API integration
      ApnsProviderClient.java                    # APNS HTTP/2 API integration
      PayloadBuilder.java                        # Builds provider-specific payload from canonical request
    config/
      NotificationProviderConfig.java            # Vault-sourced FCM/APNS credentials; per-env isolation
```

## 2.3 UI Layer — Device Channel Setup (libs/notifications)

The UI registers the device with the OS push infrastructure, captures the device token, and registers it with the BFF token registry. Android notification channels (visual grouping, priority) are configured here.

```text
libs/
  notifications/
    src/
      registration/
        permission.service.ts                    # Request push permission from OS; track grant/deny outcome
        token-registration.service.ts            # POST device token + userId + deviceId to BFF token registry
        token-rotation.service.ts                # Re-register on logout / session change / token refresh
      config/
        notification-channel.config.ts           # Android notification channel IDs, iOS categories
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

## 3.2 Folder Structure (UI — libs/notifications)

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

## 3.3 Audience Targeting Taxonomy

Every notification dispatch must declare a typed `AudienceClass`. The BFF `NotificationOrchestrator` resolves the full device token set before dispatching to FCM/APNS.

| Audience Class | Description | Session Required | Typical Use Case |
|----------------|-------------|------------------|------------------|
| `all-authenticated` | All users with an active session token at time of dispatch | Yes | System-wide operational alert |
| `user-active` | Specific user — active session only | Yes | Personal action confirmation, assignment |
| `user-any-device` | Specific user on any registered device, regardless of session state | No | Re-engagement, recall from break |
| `role-all` | All users holding a specific operational role | Yes | Boarding agents, gate supervisors |
| `role-exclude` | All users of a role minus an explicit exclusion list | Yes | Notify all agents except those already actioned |
| `airport-role` | All users of a role scoped to a specific airport | Yes | DXB boarding agents only |
| `device` | Specific device token — not user-scoped | No | Dedicated gate terminal notification |

**Not-logged-in delivery (`user-any-device` / `device`):**
- Device token remains registered after logout, marked `session-detached` in the token registry.
- Notification is delivered as a system push to the device.
- On notification tap: app launches → login screen displayed → pending deep-link intent preserved → post-login navigation resolves the intent.
- `requiresActiveSession: false` must be explicitly set in the payload; absent = default `true`.

## 3.4 BFF Audience Resolution Model

The BFF resolves the target token set server-side before dispatch. Feature code never constructs token lists.

```java
// bff-notifications-core / AudienceSpec.java
public sealed interface AudienceSpec {
    record AllAuthenticated() implements AudienceSpec {}
    record SpecificUser(String userId, boolean requireActiveSession) implements AudienceSpec {}
    record Role(String role) implements AudienceSpec {}
    record RoleExclude(String role, List<String> excludeUserIds) implements AudienceSpec {}
    record AirportRole(String airportCode, String role) implements AudienceSpec {}
    record SpecificDevice(String deviceId) implements AudienceSpec {}
}
```

```java
// bff-notifications-core / AudienceResolver.java
public interface AudienceResolver {
    List<DeviceToken> resolve(AudienceSpec spec, RequestContext ctx);
}
```

**Resolution rules:**
- `AllAuthenticated`: query token registry for all tokens with `sessionStatus = ACTIVE`.
- `SpecificUser` + `requireActiveSession = true`: return tokens only if user has an active session; no active session = no dispatch.
- `SpecificUser` + `requireActiveSession = false`: return all registered device tokens for `userId`, regardless of session state.
- `RoleExclude`: resolve `Role` set first, then subtract tokens matching `excludeUserIds`.
- `AirportRole`: `Role` resolution filtered by `airportCode` claim in session context.
- Maximum fan-out: resolved token list capped at **10,000** per dispatch; batches above this require explicit ARB approval.

**Server-side audience authorization:**
- Caller must have `notifications:dispatch:{audienceClass}` permission.
- `role-exclude` and `all-authenticated` dispatches require elevated `notifications:broadcast` permission.
- All audience resolution and dispatch decisions are emitted to the audit store.

```text
bff-notifications-core/                          # Maven module — as-ops-bff (BFF layer)
  src/main/java/com/as/ops/bff/notifications/
    audience/
      AudienceSpec.java                          # Sealed interface: 6 audience class variants
      AudienceResolver.java                      # Resolves device token set for a given AudienceSpec
      TokenRegistryClient.java                   # Queries token registry by session / role / airport
      AudienceFanOutGuard.java                   # Enforces 10k token cap; routes oversized dispatch to ARB
```

## 3.5 Notification Lifecycle & Acknowledgement Model

Each notification carries an `AcknowledgementPolicy` that governs how delivery is suppressed after action.

| Policy | Behaviour | Use Case |
|---|---|---|
| `none` | Fire-and-forget; no tracking | Informational update, flight status |
| `one-time` | Sent once per `notificationId`; not re-sent on re-login, token refresh, or device change | Maintenance window, shift briefing |
| `any-ack-suppresses-all` | First acknowledgement by **any** recipient suppresses further delivery to all remaining recipients | Gate door open — one agent acts, others silenced |
| `individual-ack` | Each recipient must independently acknowledge; unacknowledged recipients continue to receive reminders until they ack | Safety checklist, compliance confirmation, mandatory briefing |

**Acknowledgement flow:**

```text
User taps "Acknowledge" in app
  → NotificationAckService.acknowledge(notificationId, userId, deviceId)
      → POST /v1/notifications/{notificationId}/ack  (BFF)
          → Acknowledgement stored in ack-registry
          → Policy evaluation:
              any-ack-suppresses-all: mark notification as globally suppressed → cancel pending sends for all remaining tokens
              individual-ack:        mark suppressed for this userId only → other recipients unaffected
              one-time:              no ack needed; suppression handled at dispatch time
          → Audit event emitted: notification.acknowledged
```

**Client-side acknowledgement service:**

```typescript
// libs/notifications/src/acknowledgement/notification-ack.service.ts
export async function acknowledgeNotification(
  notificationId: string,
  policy: AcknowledgementPolicy,
  context: { userId: string; deviceId: string; correlationId: string }
): Promise<void> {
  if (policy === 'none') return;

  // Optimistic local suppression — prevent duplicate display
  ackStore.markAcknowledged(notificationId, context.userId);

  // Sync to BFF — server enforces cross-device/cross-user suppression
  await notificationsApi.acknowledge({ notificationId, ...context });

  // Emit audit-safe telemetry
  auditClient.emit('notification.acknowledged', {
    notificationId,
    policy,
    deviceId: context.deviceId,
    userId: context.userId,
    correlationId: context.correlationId,
  });
}
```

**One-time deduplication:**

```typescript
// libs/notifications/src/acknowledgement/ack-store.ts
// Persisted in Realm — survives app restart, re-login, token refresh
export function isAlreadyDelivered(notificationId: string): boolean {
  return realm.objectForPrimaryKey('AcknowledgedNotification', notificationId) !== null;
}

export function markDelivered(notificationId: string): void {
  realm.write(() => {
    realm.create('AcknowledgedNotification', {
      notificationId,
      deliveredAt: new Date().toISOString(),
    }, Realm.UpdateMode.Never);
  });
}
```

```text
libs/
  notifications/
    src/
      acknowledgement/
        notification-ack.service.ts      # client-side ack entry point
        ack-store.ts                     # Realm-backed local ack deduplication
        ack-policy.types.ts              # AcknowledgementPolicy type union
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
─────────────────── BFF Layer (as-ops-bff) ────────────────────────────────────────
Domain Event (Solace / internal trigger)
  → NotificationOrchestrator              [bff-notifications-core/dispatch]
      → AudienceResolver.resolve(spec)    [bff-notifications-core/audience]
          → TokenRegistryClient           (queries session / role / airport registry)
      → AudienceFanOutGuard               (enforces 10k token cap)
      → NotificationDispatcher            [bff-notifications-core/dispatch]
          → FcmProviderClient             (FCM HTTP v1 API)
          → ApnsProviderClient            (APNS HTTP/2 API)
  → AuditEmitter.emit(notification.dispatched)   [bff-audit-core]

POST /v1/notifications/{notificationId}/ack   ← called by UI acknowledgement service
  → AcknowledgementRegistry              [bff-notifications-core/acknowledgement]
  → NotificationPolicyEvaluator          (suppress pending sends per policy)
  → AuditEmitter.emit(notification.acknowledged)

─────────────────── UI Layer (as-ops-ui) ──────────────────────────────────────────
Device receives push (native OS — FCM/APNS SDK)
  → NotificationHandler (foreground / background / terminated)
      → PayloadValidator                  [libs/notifications/payload]
      → AckStore.isAlreadyDelivered()     [libs/notifications/acknowledgement] ← one-time guard
      → foreground: in-app banner UX
      → background / terminated: system notification
  → User taps notification
      → NotificationRouteGuard            (auth + permission check)
      → DeepLinkMapper                    (map deepLink field → typed route)
      → Navigate to Feature Screen
  → User acknowledges
      → NotificationAckService            [libs/notifications/acknowledgement]
          → AckStore.markAcknowledged()   (optimistic local suppression)
          → POST /v1/notifications/{id}/ack → BFF
```

## 5.1 Implementation Guidelines
- Keep orchestration logic server-side; client handles rendering/routing only.
- Correlation IDs must propagate from domain event to push payload.
- Enforce auth/session checks before processing deep links.
- Separate notification ingestion from navigation side effects.

## 5.2 BFF Folder Structure (bff-notifications-core)

```text
bff-notifications-core/                          # Maven module — as-ops-bff
  src/main/java/com/as/ops/bff/notifications/
    audience/
      AudienceSpec.java                          # Sealed interface for audience targeting
      AudienceResolver.java                      # Resolves device token set for AudienceSpec
      TokenRegistryClient.java                   # Queries token registry by session / role / airport
      AudienceFanOutGuard.java                   # Enforces 10k cap; gates large dispatches via ARB
    dispatch/
      NotificationOrchestrator.java              # Entry point: drives full dispatch flow
      NotificationDispatcher.java                # Routes payload to FCM or APNS provider client
      FcmProviderClient.java                     # FCM HTTP v1 API integration
      ApnsProviderClient.java                    # APNS HTTP/2 API integration
      PayloadBuilder.java                        # Builds provider-specific payload from canonical request
    acknowledgement/
      AcknowledgementRegistry.java               # Server-side ack persistence (per-user, per-notification)
      NotificationPolicyEvaluator.java           # Evaluates policy; suppresses pending sends on ack
      AckRequest.java                            # Ack endpoint request model
    config/
      NotificationProviderConfig.java            # Vault-sourced FCM/APNS credentials; per-env isolation
```

## 5.3 UI Folder Structure (libs/notifications)

Authoritative UI-layer folder structure for `as-ops-ui`. All subfolders listed here are UI concerns only.

```text
libs/
  notifications/
    src/
      registration/
        permission.service.ts                    # Request/track push permission from OS
        token-registration.service.ts            # Register device token with BFF token registry
        token-rotation.service.ts                # Rotate token on session change / logout
      handlers/
        foreground.handler.ts                    # Handle push when app is in foreground
        background.handler.ts                    # Handle push in background state
        opened-app.handler.ts                    # Handle notification tap (app launch entry)
        handler-result.types.ts
      routing/
        notification-deeplink.mapper.ts          # Map deepLink field → typed route
        notification-route.guard.ts              # Auth + permission check before navigation
      acknowledgement/
        notification-ack.service.ts              # Client ack entry point → POST /v1/…/ack
        ack-store.ts                             # Realm-backed local dedup for one-time policy
        ack-policy.types.ts                      # AcknowledgementPolicy TypeScript union
      payload/
        notification-payload.types.ts            # Payload type contract (mirrors BFF canonical model)
        notification-payload.validator.ts        # Schema validation at ingress
      preferences/
        notification-preferences.slice.ts        # Redux slice for per-user notification preferences
        notification-preferences.selectors.ts
      taxonomy/
        notification-type.enum.ts                # Versioned type constants (shared contract)
        notification-priority.map.ts
        notification-ttl.policy.ts
      telemetry/
        notification-events.ts                   # Telemetry event definitions
        notification-metrics.ts                  # Metric counters and histograms
      index.ts                                   # Barrel exports for controlled consumption
```

---

## 6. Client Structure (UI — libs/notifications)

> The authoritative UI folder structure is in **Section 5.3**. The sections below cover implementation guidelines and submodule detail.

## 6.1 Implementation Guidelines
- Keep client structure layered: registration → handler → routing → acknowledgement → preferences → telemetry.
- Handlers must be side-effect controlled and return typed outcomes.
- Route resolution must be centralized in `notification-deeplink.mapper.ts` — no inline navigation inside handlers.
- Acknowledgement must always route through `notification-ack.service.ts` — never call the BFF ack endpoint directly from screens or handlers.
- Preferences must be read/write through the Redux slice only — no direct AsyncStorage or Realm access from feature code.

## 6.2 Folder Structure — Expanded (UI — libs/notifications)

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
      acknowledgement/
        notification-ack.service.ts
        ack-store.ts
        ack-policy.types.ts
      payload/
        notification-payload.types.ts
        notification-payload.validator.ts
      preferences/
        notification-preferences.slice.ts
        notification-preferences.selectors.ts
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

## 7.5 Folder Structure (UI — libs/notifications)

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
- `notificationId`          — stable unique ID; used for deduplication + acknowledgement tracking
- `notificationType`        — versioned type from notification taxonomy
- `title`
- `message`
- `airportId`
- `correlationId`
- `traceId`
- `eventVersion`
- `deeplink`
- `audienceClass`           — the audience class used at dispatch (for client-side awareness)
- `acknowledgementPolicy`  — `none` | `one-time` | `any-ack-suppresses-all` | `individual-ack`
- `requiresActiveSession`  — `true` (default) | `false` (allow delivery to session-detached device)
- `ttl`                    — seconds until notification expires; 0 = no expiry

Rules:
- No sensitive PII in title/body
- Stable versioned payload schema
- TTL defined by notification category
- `notificationId` must be a stable, idempotent ID — regenerating it on retry must produce the same value for the same business event (use deterministic UUID from event correlation)
- Client must check `ackStore.isAlreadyDelivered(notificationId)` before displaying one-time notifications

## 8.1 Implementation Guidelines
- Validate payloads at ingress using schema validators before UI handling.
- Keep contract version in payload and support `current` + `previous` during migration.
- Use typed mapping from payload -> domain notification model.
- Reject malformed payloads with explicit error reason and telemetry.

## 8.2 Folder Structure (UI — libs/notifications)

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

## 9.2 Folder Structure (UI — libs/notifications)

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

## 11.2 Folder Structure (UI — libs/notifications)

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
- `AudienceClass` declared for every notification type in taxonomy
- BFF audience resolution tested for all 7 audience classes
- `role-exclude` and `all-authenticated` dispatches require `notifications:broadcast` permission — verified in auth tests
- `AcknowledgementPolicy` defined per notification type — no unclassified notifications
- `any-ack-suppresses-all` suppression verified: remaining recipients receive no further sends after first ack
- `individual-ack` reminder loop tested: unacknowledged recipients continue to receive until individually acked
- `one-time` deduplication verified: re-login and token refresh do not re-deliver the same `notificationId`
- Not-logged-in delivery path tested: session-detached device receives push; post-login deep-link intent resolved correctly
- Acknowledgement audit events emitting for all policies
- Fan-out cap (10,000 tokens) enforced and tested in `AudienceFanOutGuard`
