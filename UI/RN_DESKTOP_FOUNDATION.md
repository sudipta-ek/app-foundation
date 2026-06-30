# RN Desktop Foundation (Aligned with React Native Foundation)

## 1. Objective
Define a desktop foundation aligned with [UI/REACT_NATIVE_FOUNDATION.md](UI/REACT_NATIVE_FOUNDATION.md), while supporting two implementation options:
- **Option A**: React Native Web (maximum shared UI with mobile)
- **Option B**: React (web-first desktop UI)

This foundation keeps the same enterprise architecture principles: modular monorepo, role/security governance, observability, offline-aware patterns, and contract-first API/event integration.

---

## 2. Scope
- Desktop shell and bootstrap model
- Routing and layout framework
- Grid and form framework choices
- Role-based rendering and permission gates
- Keyboard-first workflow support
- Shared component strategy with mobile

---

## 3. Technology Stack Options

## 3.1 Option A — React Native Web (Shared-UI First)
**When to choose:** maximize reuse between mobile and desktop, single design system primitives, fewer duplicate components.

| Area                            | Recommended Stack |
|---------------------------------|-------------------|
| App shell/bootstrap             | React Native Web + Nx desktop shell app |
| Routing/layout                  | React Navigation (web support) or thin router wrapper |
| State management                | Redux Toolkit + TanStack Query |
| API integration layer           | OpenAPI-generated SDK + Axios wrapper |
| Authentication/session handling | OIDC/OAuth2 via shared auth library + web token handler |
| Grid framework integration      | Responsive card/grid built in shared `ui-responsive` + optional data-grid wrapper |
| Form engine                     | React Hook Form + Zod/Yup adapters shared via platform libs |
| Role-based rendering            | Shared RBAC/ABAC permission resolver and guard components |
| Keyboard shortcuts/workflows    | `react-hotkeys-hook` + feature-level command map |
| Shared component system         | `ui-responsive` as default, desktop wrappers in `ui-desktop` |
| Logging/analytics hooks         | Shared observability SDK hooks (OpenTelemetry-compatible) |

## 3.2 Option B — React (Desktop-First UX)
**When to choose:** dense desktop workflows, advanced table/grid UX, keyboard-heavy operations with web-native ecosystem.

| Area                            | Recommended Stack                       |
|---------------------------------|-----------------------------------------|
| App shell/bootstrap             | React + Nx desktop shell app (Vite/Next-style host per org standard) |
| Routing/layout                  | React Router + layout shell composition |
| State management                | Redux Toolkit + TanStack Query |
| API integration layer           | OpenAPI-generated SDK + Axios wrapper |
| Authentication/session handling | OIDC/OAuth2 web SDK integrated via shared auth contracts |
| Grid framework integration      | AG Grid / MUI Data Grid / TanStack Table (based on licensing/perf policy) |
| Form engine                     | React Hook Form + Zod/Yup + enterprise field wrappers |
| Role-based rendering            | Shared RBAC/ABAC resolver + route/action guards |
| Keyboard shortcuts/workflows    | `react-hotkeys-hook` or command palette architecture |
| Shared component system         | Shared tokens/behavior contracts + desktop component library |
| Logging/analytics hooks         | Shared telemetry/events + web performance hooks |

---

## 4. Monorepo Structure (Desktop-Aligned)

```text
apps/
  desktop-shell/                         # Desktop entry app and shell wiring

libs/
  features/                              # Domain workflows (boarding/check-in/flights)
  ui-responsive/                         # Shared responsive components (mobile + desktop)
  ui-desktop/                            # Desktop-only components (dense table, split panes)
  auth/                                  # Shared SSO/RBAC/ABAC logic
  sdk/                                   # OpenAPI generated SDK
  realtime/                              # Realtime/event subscriptions
  analytics/                             # Logging and analytics hooks
  shared/                                # Utilities, config, contracts
```

Rules:
- `features` must not import app shell directly.
- Use `ui-responsive` by default.
- Use `ui-desktop` only for desktop-specific interaction/performance needs.

---

## 5. Desktop Shell Foundation

## 5.1 Shell Responsibilities
- App bootstrap and environment config
- Session bootstrap and token restore
- Global providers (Redux, QueryClient, Theme, i18n)
- Route registry initialization
- Global keyboard command registration
- Global notifications and error boundaries

## 5.2 Bootstrap Flow
1. Load runtime config and feature flags
2. Restore or establish session
3. Resolve permissions and airport context
4. Initialize telemetry and correlation context
5. Mount router/layout shell
6. Register keyboard workflows and command map

## 5.3 Implementation Guidelines
- Keep shell logic in `apps/desktop-shell` only; never move domain business logic into shell.
- Register providers in strict order: config → auth/session → state/query → routing → telemetry.
- Use a single startup orchestrator with explicit phases and fail-fast logging.
- Keep global error boundary and global toast/notification host at shell root.
- Expose a shell contract (`bootstrapContext`) for feature libraries (read-only).

## 5.4 Folder Structure

```text
apps/
  desktop-shell/
    src/
      bootstrap/
        bootstrap-app.ts                 # startup orchestrator
        bootstrap-config.ts              # runtime/env + remote config
        bootstrap-session.ts             # restore/login/session validation
        bootstrap-telemetry.ts           # tracing + analytics init
      providers/
        app-providers.tsx                # redux/query/theme/i18n composition
      shell/
        app-shell.tsx                    # root shell layout frame
        app-error-boundary.tsx           # global error boundary
        app-notification-host.tsx        # global notification host
      startup/
        startup-state.slice.ts           # startup phase state
        startup.selectors.ts
      index.tsx
```

---

## 6. Routing and Layout Framework

## 6.1 Route Governance
- Domain routes owned by domain teams
- Central route registry controls cross-domain navigation
- Role-aware and flag-aware route activation
- Typed route params and route contracts

## 6.1.1 Route Metadata Contract

Every registered route must declare a full typed metadata contract:

```typescript
interface RouteMetadata {
  path: string;
  title: string;              // browser tab title (i18n key or static string)
  analyticsId: string;        // stable, immutable telemetry screen identifier
  permissions: string[];      // all required permissions; empty = restricted by default
  featureFlag?: string;       // optional feature flag guard key
  layout: 'default' | 'split' | 'tabbed' | 'inspector';
  breadcrumb: string[];       // ordered breadcrumb segment labels
  deepLink: boolean;          // true = supports direct URL cold-start session restore
  ownership: string;          // owning domain team (see Domain Ownership Matrix, Section 20)
}
```

Rules:
- `analyticsId` is immutable once set; renaming requires analytics data backfill and ARB review.
- `ownership` must match a registered team in the Domain Ownership Matrix (Section 20).
- `deepLink: true` routes must implement session-restore-before-render on cold start.
- Routes with empty `permissions` array are treated as fully restricted by default.
- `featureFlag` evaluation in the client is UX convenience only; enforcement remains at BFF.

## 6.2 Layout Standards
- Shell layout: top app bar + side navigation + content region
- Desktop workspace layouts: split pane, inspector pane, tabbed work area
- Consistent breadcrumbs and command bar patterns

## 6.3 Implementation Guidelines
- Central route registry is the only cross-domain navigation entry.
- Enforce typed route contracts (`params`, `required permissions`, `feature flags`).
- Use route-level lazy loading for domain modules.
- Keep layout primitives in `ui-desktop/layout`; keep route declarations in feature libraries.
- Require audit-safe telemetry on route entry/denial.

## 6.4 Folder Structure

```text
apps/
  desktop-shell/
    src/
      routing/
        route-registry.ts                # central route map
        route-loader.ts                  # lazy loading + guards
        route-types.ts                   # typed route contracts
        navigation.service.ts            # shell-level navigation service
      layout/
        shell-layout.tsx                 # app bar + side nav + content
        workspace-layout.tsx             # split/tab/inspector layout
        breadcrumbs.tsx
      guards/
        route-permission.guard.tsx
        route-flag.guard.tsx

libs/
  features/
    boarding/
      src/
        routes/
          boarding.routes.ts             # domain route declarations
```

---

## 7. Grid Framework Integration

## 7.1 Requirements
- Virtualization for large datasets
- Server-side pagination/filter/sort support
- Keyboard navigation and bulk actions
- Column personalization and persistence

## 7.2 Governance
- One approved grid abstraction wrapper in `ui-desktop`
- Data access via query hooks only
- No direct API calls from cell renderer components

## 7.3 Implementation Guidelines
- Wrap selected grid engine behind one `enterprise-grid` adapter component.
- Standardize server-side query contract (`page`, `size`, `sort`, `filters`).
- Keep column definitions versioned and persist user preferences by role/user.
- Move all row actions through typed action callbacks with permission checks.
- Grid components are presentation-only; data retrieval lives in feature query hooks.

## 7.4 Folder Structure

```text
libs/
  ui-desktop/
    src/
      grid/
        enterprise-grid.component.tsx    # wrapper over selected grid engine
        grid-column.types.ts
        grid-toolbar.component.tsx
        grid-preferences.service.ts       # column visibility/order persistence
        grid-shortcuts.ts                 # keyboard navigation config

  features/
    flights/
      src/
        queries/
          use-flight-grid.query.ts
        mappers/
          flight-grid.mapper.ts
        components/
          flight-grid.container.tsx       # binds query data to enterprise-grid
```

---

## 8. Form Engine

Requirements:
- Schema-driven validation
- Async validation hooks
- Accessible labels/error semantics
- Dirty-state and unsaved-change guards

Implementation guidance:
- Shared form primitives across mobile/desktop where possible
- Desktop-specific field UX in `ui-desktop/forms`

## 8.1 Implementation Guidelines
- Use a single form abstraction (`useEnterpriseForm`) over React Hook Form.
- Validation schemas must be colocated with form domain model and versioned.
- Support async validations with debounced query hooks.
- Enforce dirty-state navigation guard and explicit discard/submit flows.
- Keep field components reusable and accessibility-compliant by default.

## 8.2 Folder Structure

```text
libs/
  ui-responsive/
    src/
      forms/
        text-field.component.tsx
        select-field.component.tsx
        form-error-summary.component.tsx

  ui-desktop/
    src/
      forms/
        desktop-date-time-field.component.tsx
        desktop-multi-select.component.tsx
        desktop-form-section.component.tsx

  features/
    checkin/
      src/
        forms/
          passenger-checkin.form.tsx
          passenger-checkin.schema.ts
          passenger-checkin.defaults.ts
        hooks/
          use-passenger-checkin-form.hook.ts
        guards/
          unsaved-changes.guard.ts
```

## 8.3 Form Telemetry

All form interactions must emit structured telemetry events. Field values are **prohibited** in all events; field names and structural metadata only.

| Event | Trigger | Key Properties |
|---|---|---|
| `form.started` | User interacts with first field | formId, screenId, fieldCount |
| `form.validation.failed` | Validation error surfaced to user | formId, fieldName (not value), errorType, attemptCount |
| `form.submitted` | Form submitted successfully | formId, durationMs, fieldCount, submitAttemptCount |
| `form.abandoned` | User navigates away with unsaved changes | formId, fieldsFilledRatio, fieldsTouched, timeSpentMs |
| `form.async.validation` | Async field validation triggered | formId, fieldName, durationMs, result |

PII rules:
- Field values: **prohibited** in all telemetry.
- Field names: allowed (e.g. `passportNumber` as a name identifier, never the value).
- All form events must carry `correlationId`, `sessionId`, `userId` (hashed).

---

## 9. Role-Based Rendering

## 9.1 Guard Types
- Route guard
- Feature guard
- Action guard
- Field-level guard

## 9.2 Policy Inputs
- User roles
- Airport/gate context
- Device/session compliance
- Feature flag state

All denied actions must be logged with audit-safe metadata.

## 9.3 Implementation Guidelines
- Resolve permissions once at session bootstrap and refresh on context change.
- Use composable guards: route guard + action guard + field guard.
- Keep authorization checks in shared auth lib; do not duplicate in features.
- Denied actions must emit audit-safe event with `userId`, `role`, `action`, `context`.
- Render fallback UX consistently (`hidden`, `disabled`, or `read-only`) per policy.

## 9.4 Folder Structure

```text
libs/
  auth/
    src/
      authorization/
        permission-resolver.ts
        rbac-engine.ts
        abac-engine.ts
        permission.types.ts
      guards/
        route.guard.tsx
        action.guard.tsx
        field.guard.tsx
      hooks/
        use-permission.hook.ts
        use-role-context.hook.ts

  features/
    boarding/
      src/
        permissions/
          boarding.permission-map.ts
        components/
          boarding-action-bar.component.tsx
```

---

## 10. Keyboard Workflow Support

## 10.1 Mandatory Keyboard Patterns
- Global command palette (search actions/screens)
- Domain shortcut groups (boarding/check-in/flight ops)
- Grid navigation shortcuts
- Form submission and navigation shortcuts

## 10.2 Keyboard Registry Governance

| Dimension | Policy |
|---|---|
| Shortcut versioning | All shortcuts carry a version tag in the registry with change history; breaking renames require ARB review |
| Conflict detection | Automated collision check runs in CI pipeline; conflicting registrations fail the build gate |
| Reserved enterprise shortcuts | `Ctrl+Shift+L` (force logout), `Ctrl+Shift+D` (diagnostics panel), `Ctrl+Shift+H` (help), `Ctrl+Shift+A` (accessibility panel) — reserved platform-wide; domain teams must not override these bindings |
| Airport-specific overrides | Airport operations may define override bindings via config service; override registry loaded at session init from airport ops config |
| Accessibility override | Accessibility mode shortcuts supersede all domain-level bindings; screen reader mode disables non-essential shortcuts; accessibility shortcuts take unconditional precedence |
| Scope levels | `global` — active at all times; `route` — active on matching route only; `modal` — active while a modal or dialog is open |

## 10.3 Implementation Guidelines
- Maintain a centralized shortcut registry with domain namespaces.
- Register global shortcuts in shell; register domain shortcuts on route mount.
- Validate collisions in CI using a static registry checker.
- Provide command palette fallback for discoverability.
- Disable or remap conflicting shortcuts based on platform/browser restrictions.

## 10.4 Folder Structure

```text
apps/
  desktop-shell/
    src/
      keyboard/
        shortcut-registry.ts             # source of truth for shortcuts
        shortcut-resolver.ts             # collision/precedence rules
        global-shortcuts.ts              # shell-level shortcuts
        command-palette.service.ts

libs/
  features/
    flights/
      src/
        keyboard/
          flights-shortcuts.ts           # domain-specific shortcuts
          use-flights-shortcuts.hook.ts

  ui-desktop/
    src/
      keyboard/
        shortcut-hint.component.tsx
        command-palette.component.tsx
```

---

## 10.5 Command Palette

The command palette is a global keyboard-accessible interface for discovering and executing all registered platform commands.

## 10.5.1 Command Registry Contract

```typescript
interface Command {
  id: string;                  // stable, immutable command identifier
  label: string;               // i18n display name
  group: string;               // category (boarding, check-in, navigation, system)
  permissions: string[];       // required permissions — hidden if not satisfied
  featureFlag?: string;        // optional feature flag guard
  shortcut?: string;           // associated shortcut label (display only)
  searchTerms?: string[];      // additional terms for search indexing
  action: () => void | Promise<void>;
}
```

## 10.5.2 Governance

| Dimension | Policy |
|---|---|
| Command registration | Global commands registered at shell init; domain commands registered on route mount and deregistered on unmount |
| Permissions | Command is **hidden** (not just disabled) if user lacks required permissions |
| Feature flags | Command hidden if feature flag evaluates false for the session context |
| Search provider | Commands indexed by `label`, `group`, and `searchTerms`; client-side fuzzy match |
| Telemetry | Every invocation and command selection emits `command.palette.invoked` event (see Section 15.1) |
| Audit | Security/compliance commands (force logout, config change) emit audit-safe event on execution |

## 10.5.3 Folder Structure

```text
apps/
  desktop-shell/
    src/
      keyboard/
        command-registry.ts              # typed command store
        command-palette.service.ts       # registration, deregistration, search

libs/
  ui-desktop/
    src/
      keyboard/
        command-palette.component.tsx    # presentational palette UI
        command-palette.hook.ts          # search state and selection logic
        command-palette.types.ts         # Command interface and related types
```

---

## 11. Shared Component System

## 11.1 Component Strategy
- **Shared-first:** `ui-responsive`
- **Desktop exception:** `ui-desktop`
- Design tokens are single source of truth

## 11.2 Component Classes
- Base primitives (buttons, inputs, chips)
- Data display (cards, list, table wrappers)
- Desktop advanced (data grid toolbar, inspector panel, split view)

---

## 12. Cross-Cutting Concerns

- **State:** Redux + Query architecture from [UI/RN_STATE_ARCHITECTURE.md](UI/RN_STATE_ARCHITECTURE.md)
- **Realtime:** align with [UI/RN_REALTIME_COMMUNICATION_ARCHITECTURE.md](UI/RN_REALTIME_COMMUNICATION_ARCHITECTURE.md)
- **Events:** align with [UI/RN_EVENT_DRIVEN_ARCHITECTURE.md](UI/RN_EVENT_DRIVEN_ARCHITECTURE.md)
- **Auth:** align with [UI/RN_ENTERPRISE_AUTHENTICATION.md](UI/RN_ENTERPRISE_AUTHENTICATION.md)
- **Native capability parity:** where relevant, provide desktop capability adapters aligned with [UI/RN_NATIVE_CAPABILITY_PLATFORM.md](UI/RN_NATIVE_CAPABILITY_PLATFORM.md)

---

## 13. Option Selection Matrix

| Criterion                        | Option A: React Native Web | Option B: React |
|----------------------------------|----------------------------|-----------------|
| Mobile UI reuse                  | Excellent                  | Moderate |
| Desktop UX depth                 | Moderate                   | Excellent |
| Grid ecosystem flexibility       | Moderate                   | Excellent |
| Keyboard workflow customization  | Good                       | Excellent |
| Team skill alignment (web-heavy) | Moderate                   | Excellent |
| Long-term unified UI strategy    | Excellent                  | Good |

**Recommended approach:**
- Start with **Option A** for reuse-driven programs.
- Use **Option B** for operations-heavy, dense-data desktop products.
- Hybrid model allowed: shared domain/state/auth + desktop-specialized UI shell.

---

## 14. Desktop Foundation DoD

A desktop foundation is considered ready when:
- Shell/bootstrap and route registry are operational
- Grid and form frameworks are standardized and wrapped
- RBAC/ABAC guards are enforced at route/action level
- Keyboard registry and conflict rules are implemented
- Shared component model (`ui-responsive`/`ui-desktop`) is enforced
- Observability and analytics hooks are active
- Architecture aligns with [UI/REACT_NATIVE_FOUNDATION.md](UI/REACT_NATIVE_FOUNDATION.md)

---

## 15. Desktop Observability Layer

## 15.1 Telemetry Event Taxonomy

| Event | Trigger | Key Properties |
|---|---|---|
| `app.started` | App fully bootstrapped and ready | startupDuration, sessionId, userId, airportCode, clientVersion |
| `screen.opened` | Route activated successfully | screenId, analyticsId, renderDuration, fromScreen, permissionsCount |
| `grid.loaded` | Grid receives and renders first data page | gridId, rowCount, renderDuration, pageSize, appliedFilters |
| `search.executed` | User submits search or applies filter | searchId, resultCount, durationMs, screenId |
| `keyboard.shortcut.used` | Registered shortcut triggered | shortcutId, action, screenId, scope, modifiers |
| `command.palette.invoked` | Command palette opened or command selected | invocationMethod, selectedCommandId, searchQuery (masked), durationMs |
| `api.request` | BFF API call initiated and completed | endpoint, method, statusCode, durationMs, correlationId, traceId |
| `route.denied` | Route guard blocks navigation | routePath, reason, userId, roles, featureFlag |
| `permission.failure` | Action or field guard denies operation | action, requiredPermission, userId, roles, airportCode |
| `realtime.event.received` | Solace or WebSocket event arrives | eventType, topic, processingDuration, correlationId |

All events must carry: `sessionId`, `correlationId`, `traceId`, `userId` (hashed), `clientVersion`, `airportCode`, `timestamp`.

## 15.2 OpenTelemetry Integration

- All telemetry emitted via OpenTelemetry SDK (web).
- **Traces**: Distributed trace propagation from BFF response `traceparent` header.
- **Spans**: One span per route activation, API call, and grid render cycle.
- **Metrics**: OTLP export to enterprise observability platform (Datadog / Dynatrace).
- **Logs**: Structured JSON via OTLP log exporter.

Trace propagation flow:
```text
BFF response (traceparent header)
  → Frontend: extract traceId + spanId
  → Create child spans for UI operations (route change, grid render, search)
  → Export via OTLP collector
```

## 15.3 CorrelationId / TraceId Governance

- `correlationId`: generated at session bootstrap; constant for the full session; sent as `X-Correlation-ID` on every API request.
- `traceId`: generated per user-initiated operation (route change, API call, search); propagated from BFF trace headers where available.
- Both must appear in all telemetry events, structured logs, and user-facing error diagnostics.
- `correlationId` must be surfaced in error UX and support diagnostics panels.

## 15.4 PII-Safe Logging

| Data Type | Policy |
|---|---|
| User name, email | Never log; use `userId` (hashed) only |
| Passport / ID numbers | Prohibited in all logs |
| Search query content | Log structure and result counts only; mask passenger identifiers |
| Form field values | Prohibited; log field name and validation result only |
| Session / auth tokens | Prohibited in all logging surfaces |
| Airport / gate context | Permitted |
| Error messages | Review before logging; mask any passenger data before emission |

Implementation rules:
- PII redaction applied at the structured-logger SDK level, not at each call site.
- New telemetry event fields touching user context require compliance sign-off.
- Log schema changes require Platform Team review and backward compatibility window.

## 15.5 Measurable Performance Targets

| Metric | Target | Alert Threshold |
|---|---|---|
| App startup (bootstrap to interactive) | < 2 sec | > 3 sec |
| Grid render (first page painted) | < 500ms | > 1 sec |
| Route change (transition complete) | < 300ms | > 600ms |
| Search response (results displayed) | < 1 sec | > 2 sec |
| Realtime update applied to UI | < 500ms | > 1 sec |
| Memory usage (steady state) | < 400MB | > 600MB |
| Crash-free session rate | > 99.8% | < 99.5% |

All targets are measured via Real User Monitoring (RUM) in production and synthetic monitoring in pre-production environments.

## 15.6 Folder Structure

```text
libs/
  observability/
    src/
      telemetry/
        telemetry.client.ts              # OpenTelemetry SDK init and OTLP export config
        event-schema.types.ts            # typed telemetry event contracts
        telemetry.emitter.ts             # emitEvent() with schema enforcement
        pii-redaction.policy.ts          # field-level PII scrubbing rules
      tracing/
        trace-context.service.ts         # extract and propagate traceparent header
        correlation.service.ts           # session-scoped correlationId management
      metrics/
        performance-observer.service.ts  # Web Vitals + custom performance metric hooks
        memory-monitor.service.ts        # memory usage sampling and alerting
      hooks/
        use-telemetry.hook.ts
        use-trace-context.hook.ts
      logging/
        structured-logger.ts             # PII-safe structured JSON logger
        log-level.policy.ts              # per-environment log level configuration
```

---

## 16. Accessibility Governance

## 16.1 Standards
- WCAG 2.1 Level AA mandatory for all desktop screens.
- Section 508 compliance where regulatory or contractual mandates apply.
- ARIA roles, landmarks, and live regions applied to all dynamic content areas.

## 16.2 Mandatory Requirements

| Area | Requirement |
|---|---|
| Focus management | Focus explicitly managed on route change, dialog open/close, and modal dismiss |
| Keyboard navigation | All interactions reachable and executable via keyboard alone |
| Screen reader | All interactive elements have descriptive ARIA labels or visible text labels |
| Grid accessibility | Row/cell ARIA roles, keyboard row navigation, sort/filter state announcements |
| Color contrast | Minimum 4.5:1 for body text; 3:1 for large text and UI components |
| Error announcement | Form errors announced via ARIA live region immediately on validation failure |
| Dynamic content | Asynchronous updates (realtime, polling) announced via `aria-live` where user-facing |
| Motion | Respect `prefers-reduced-motion`; no essential information conveyed through animation alone |
| Zoom | Layout must not break at 200% browser zoom |

## 16.3 Keyboard Accessibility Override Model
- Accessibility mode shortcut bindings supersede domain-level bindings (see Section 10.2).
- Screen reader mode activates enhanced ARIA narration for all grid and form interactions.
- High-contrast mode toggled via command palette or user profile preference.
- Font scaling support required at 150% and 200% without horizontal scrolling or layout breakage.

## 16.4 Governance Gates
- Automated accessibility scan (axe-core or Lighthouse CI) in every PR pipeline: zero critical violations gate.
- Manual accessibility review required for all new feature screens before UAT entry.
- Keyboard-only workflow test required for all grid and form screens.
- Screen reader test (NVDA/JAWS on Windows, VoiceOver on macOS) required before each major release.

---

## 17. Configuration Governance

## 17.1 Configuration Sources

| Layer | Source | Mutability |
|---|---|---|
| Environment config | Build-time env vars / runtime config endpoint | Per environment; immutable after deploy |
| Feature flags | Runtime feature flag service (LaunchDarkly or enterprise equivalent) | Runtime-mutable |
| Airport / gate context | Session-resolved from auth claims + ops service | Per-session |
| User preferences | Browser IndexedDB | Per-user; persisted |
| Keyboard / layout overrides | User profile service or browser storage | Per-user |
| Org-level policy overrides | Enterprise config service | Managed by Platform Team only |

## 17.2 Feature Flag Contract

```typescript
interface FeatureFlag {
  key: string;
  defaultValue: boolean;
  overrides?: {
    airportCode?: string[];
    role?: string[];
    environment?: string[];
  };
  owner: string;             // owning team responsible for lifecycle
  expiryDate?: string;       // planned flag retirement ISO date
}
```

## 17.3 Governance Rules
- Feature flags must have a registered owner and a defined expiry/retirement date.
- No flag may persist beyond 2 major releases without ARB review and justification.
- Config schema changes require a migration plan and backward compatibility window.
- All config consumed via typed config service only; direct `process.env` access prohibited in feature code.
- Four-eye approval required for production feature flag changes affecting security or compliance surfaces.

---

## 18. Security Threat Model (Desktop)

| Threat | Mitigation |
|---|---|
| XSS | React DOM escaping by default; CSP headers enforced; `dangerouslySetInnerHTML` prohibited |
| CSRF | `SameSite=Strict` cookies; per-request tokens for state-changing API calls |
| Token theft | Access tokens in memory only; refresh tokens in HttpOnly cookie only |
| Clickjacking | `X-Frame-Options: DENY`; CSP `frame-ancestors: none` |
| Session fixation | New session ID on login; existing session invalidated on logout |
| Open redirect | All navigation targets validated against allow-list; no external redirect from route params |
| Sensitive data leakage | PII-safe logging enforced at SDK level; no secrets or tokens in browser storage |
| Dependency supply chain | Snyk/Dependabot automated scanning; approved-package registry policy |
| Feature flag bypass | Security-relevant flags evaluated server-side; client flag is UX convenience only |
| Shared workstation exposure | Idle timeout clears memory token; forced clear on explicit logout |
| Extension injection | CSP nonces prevent unauthorized script execution |

## 18.1 Content Security Policy (CSP) Baseline

```text
default-src 'self';
script-src 'self' 'nonce-{nonce}';
style-src 'self';
connect-src 'self' https://api.airline.com https://telemetry.airline.com;
img-src 'self' data:;
font-src 'self';
frame-ancestors 'none';
```

## 18.2 Browser Storage Policy

| Storage | Permitted Use | Prohibited |
|---|---|---|
| `localStorage` | Non-sensitive user preferences, layout config | Tokens, credentials, PII, session secrets |
| `sessionStorage` | Transient UI state; cleared on tab close | Tokens, credentials, PII |
| `IndexedDB` | Offline data cache under data classification approval | PII without explicit data classification sign-off |
| Cookies | Refresh token in HttpOnly/SameSite=Strict only | Access tokens, business data |

---

## 19. Desktop Capability Platform

## 19.1 Capability Scope

| Capability | Web API | Fallback | Owner |
|---|---|---|---|
| Clipboard read/write | Clipboard API | `document.execCommand` (deprecated) | Platform Team |
| File system access | File Picker API / drag-and-drop | `<input type="file">` | Platform Team |
| Print | Window Print API + print CSS | Server-side PDF | Platform Team |
| PDF preview | PDF.js viewer | New browser tab download | Platform Team |
| Excel / CSV export | SheetJS (xlsx) or server-side export | CSV download | Platform Team |
| Browser notifications | Web Notifications API | In-app notification center | Platform Team |
| Offline support | Service Worker + IndexedDB (Workbox) | Online-only mode with offline banner | Platform Team |
| Deep linking | URL routing + session restore | Redirect to login then route | Platform Team |
| Webcam / barcode | `MediaDevices.getUserMedia` + ZXing-JS | Manual input fallback | Platform Team |
| WebAuthn / biometric | WebAuthn API (passkey) | Password fallback | Security + Platform |
| Tab synchronization | BroadcastChannel API | Polling-based sync | Platform Team |
| Drag and drop | HTML5 DnD API | Click-to-select fallback | Platform Team |

## 19.2 Capability Architecture

Each capability follows a four-stage contract:
1. **Availability check** — feature-detect before use; graceful degradation if unavailable.
2. **Permission request** — explicit browser permission prompt on user action, never on page load.
3. **Typed adapter** — all Web API calls go through the capability lib; feature code never calls Web APIs directly.
4. **Telemetry** — capability invoked, success/failure/fallback triggered events emitted.

```text
libs/
  capabilities/
    src/
      clipboard/
      file-access/
      print/
      pdf/
      export/
      notifications/
      offline/
      deep-link/
      scanner/
      webauthn/
      tab-sync/
      drag-drop/
```

## 19.3 Offline Capability Model

| Layer | Technology | Policy |
|---|---|---|
| Static asset cache | Service Worker (Workbox) | Cache-first for shell assets |
| API response cache | Workbox network-first with offline fallback | Per-endpoint TTL policy |
| Offline data store | IndexedDB (Dexie.js) | Read-only snapshot; no offline writes for regulated workflows |
| Sync on reconnect | Background Sync API | Queue failed mutations; replay on reconnect |
| Offline indicator | Global banner + route-level write guard | Always visible; blocks regulated write operations |

## 19.4 Capability Governance
- Feature libraries must not call Web APIs directly; all access through capability lib adapters.
- Capability unavailability must degrade gracefully with accessible fallback UX.
- Browser permission requests must be contextual (user-action-triggered, never on page load).
- All capability usage emits telemetry for adoption and failure monitoring.

---

## 20. Domain Ownership Matrix

| Domain | Feature Library | BFF Route Ownership | Domain Team |
|---|---|---|---|
| Boarding | `libs/features/boarding` | `/v1/boarding/**` | Boarding Squad |
| Check-in | `libs/features/checkin` | `/v1/checkin/**` | Check-in Squad |
| Flight Operations | `libs/features/flights` | `/v1/flights/**` | Ops Squad |
| Gate Management | `libs/features/gate` | `/v1/gate/**` | Ops Squad |
| Passenger Services | `libs/features/passenger` | `/v1/passenger/**` | Passenger Squad |
| Notifications | `libs/notifications` | N/A (push/realtime) | Platform Team |
| Auth / Session | `libs/auth` | N/A (auth lib) | Security + Platform |
| Desktop Shell | `apps/desktop-shell` | N/A (shell only) | Platform Team |
| Observability | `libs/observability` | N/A (cross-cutting lib) | Platform Team |
| Capabilities | `libs/capabilities` | N/A (cross-cutting lib) | Platform Team |

Rules:
- Domain teams own their `libs/features/<domain>` code without Platform approval for non-cross-cutting changes.
- Cross-domain navigation requires a central route registry entry with Platform Team review.
- Changes to `libs/auth`, `libs/observability`, `libs/capabilities`, `ui-responsive`, or `ui-desktop` require Platform Team PR approval.
- Breaking API route changes require consumer impact assessment and minimum 6-week deprecation window.

---

## 21. Browser Support Policy

## 21.1 Supported Browsers (Desktop)

| Browser | Minimum Version | Support Level |
|---|---|---|
| Google Chrome | Last 2 major releases | Full support (primary) |
| Microsoft Edge (Chromium) | Last 2 major releases | Full support (primary) |
| Mozilla Firefox | Last 2 major releases | Full support |
| Safari (macOS) | Last 2 major releases | Full support |
| Safari (iOS / iPadOS) | Last 2 major releases | Responsive desktop views only |
| Internet Explorer 11 | Not supported | Explicitly excluded |

## 21.2 Progressive Enhancement Policy
- Core functionality must operate without advanced Web API features; degrade gracefully when APIs unavailable.
- Advanced capabilities (WebAuthn, Clipboard, File Access) require explicit availability detection and accessible fallback.
- Polyfill strategy: approved polyfill set maintained by Platform Team; no ad-hoc polyfill imports in feature libraries.
- Browserslist config is the single source of truth for build target browser coverage.

## 21.3 Compatibility Review Cadence
- Browser compatibility review: **quarterly**, aligned with major browser release cadence.
- New Web API adoption requires Platform Team feasibility review before production use.
- Deprecated API usage detected in CI via `browserslist` + `eslint-plugin-compat`; warnings fail the lint gate.

---

## 22. Desktop Session Governance

## 22.1 Session Model

| Concern | Policy |
|---|---|
| Shared workstation | Mandatory explicit logout before each shift; session and in-memory tokens cleared on browser close |
| Idle timeout | 15-minute inactivity timeout (configurable per role via enterprise policy); warning shown at 13 minutes |
| Tab synchronization | BroadcastChannel API propagates session events (login, logout, token refresh, forced logout) across all open tabs |
| Multiple login detection | Second login from different device/browser triggers session conflict notification; policy: block or notify (configurable) |
| Forced logout | Backend-initiated via push notification or WebSocket event; client clears all state immediately on receipt |
| Token refresh | Silent refresh 60 seconds before access token expiry; refresh token in HttpOnly cookie only; access token in memory only |
| Browser storage | Tokens prohibited in localStorage / sessionStorage / IndexedDB; preferences only in localStorage |

## 22.2 Idle Timeout Flow

```text
User activity detected
  → Reset idle timer (15 min)
  ↓ 13 min idle → show warning modal with countdown
  ↓ 15 min idle → force logout, clear in-memory tokens, redirect to login
  ↓ Emit audit event: session_timeout { userId, airportCode, timestamp }
```

## 22.3 Tab Synchronization Events

| Event | BroadcastChannel Message | Consumer Behavior |
|---|---|---|
| Login completed | `{ type: 'session.login', sessionId }` | All tabs reload session state |
| Logout triggered | `{ type: 'session.logout' }` | All tabs clear state and redirect to login |
| Token refreshed | `{ type: 'session.token.refreshed' }` | All tabs update in-memory access token |
| Forced logout | `{ type: 'session.forced.logout', reason }` | All tabs clear state, display reason, redirect |

## 22.4 Governance Rules
- No token value (access or refresh) may be transmitted via BroadcastChannel messages.
- Idle timeout minimum is set by enterprise policy; UI configuration cannot override below the floor value.
- All session state transitions must emit an audit-safe telemetry event.
- Shared workstation policy must be surfaced in onboarding and enforced via auto-clear on browser close.

---

## 23. Error Handling Standard

## 23.1 Error Classification

| Class | Definition | User Impact | Recovery |
|---|---|---|---|
| Recoverable | Transient failure; retry expected to succeed | Inline retry prompt | Auto / manual retry |
| Fatal | Unrecoverable app or session error | Full error screen; session may be lost | Reload or re-login |
| Dependency failure | Downstream service unavailable or timed out | Partial screen failure with inline fallback | Retry with circuit breaker state |
| Validation error | User input invalid; no system failure | Inline field error; form remains active | User corrects and resubmits |
| Authorization error | Permission or policy denied | Denied message; no data revealed | Contact admin or change context |
| Offline | No network connectivity | Offline banner; regulated writes blocked | Reconnect and sync |

## 23.2 Error Boundary Model

```text
AppErrorBoundary (root)              ← catches fatal unhandled errors; renders full error page
  RouteErrorBoundary                 ← catches per-screen unhandled errors; renders screen error + retry
    FeatureErrorBoundary             ← catches per-widget/grid/form errors; renders inline fallback
```

Rules:
- `AppErrorBoundary`: displays full-page error with `correlationId` and support diagnostic link.
- `RouteErrorBoundary`: displays screen-level error with retry action and `correlationId`.
- `FeatureErrorBoundary`: renders inline fallback (skeleton or error state chip) without disrupting other screen sections.
- All error boundaries emit a `fatal` or `error` telemetry event on catch.

## 23.3 Retry UX Model
- 1st failure → automatic silent retry (1×, exponential backoff).
- 2nd failure → show retry button with countdown timer.
- 3rd failure → show error state with `correlationId` displayed and support diagnostic action.
- Retry behavior must respect circuit breaker state received from BFF response headers.

## 23.4 Offline Banner
- Always visible at top of shell when network connectivity is unavailable.
- Blocks all write operations in regulated workflows (boarding, check-in).
- Read-only operations from offline cache remain available where data classification permits.
- On reconnect: dismiss banner, replay queued mutations, refresh critical screen data.

## 23.5 CorrelationId in Error UX
- Every user-facing error state must display the `correlationId`.
- Support diagnostic link opens a read-only session event log filtered by `correlationId`.
- `correlationId` + `traceId` is the primary diagnostic key for incident investigation.

## 23.6 Support Diagnostics
- In-app diagnostic export: downloads a redacted (PII-safe) session log bundle on user consent.
- Log bundle contains: `correlationId`, `traceId`, event sequence, error details, `clientVersion`, `airportCode`.
- Diagnostic export is prohibited from including: token values, form field values, passenger PII.

---

## 24. Enterprise Logging Standard

## 24.1 Log Schema (Structured JSON)

All log entries must conform to the enterprise log schema:

```json
{
  "timestamp": "2026-06-30T12:00:00.000Z",
  "level": "INFO",
  "correlationId": "9cbff51e-7ff5-4acd-b710-f1944fcbf001",
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "sessionId": "anon-hashed-session-id",
  "userId": "hashed-user-id",
  "airportCode": "LHR",
  "clientVersion": "3.2.1",
  "environment": "prod",
  "component": "grid",
  "message": "Grid loaded: flights-grid",
  "metadata": {}
}
```

## 24.2 Log Levels

| Level | Usage |
|---|---|
| `DEBUG` | Development only; never enabled in production builds |
| `INFO` | Normal operational events: route changes, screen loads, user actions |
| `WARN` | Degraded behavior, retries, fallbacks, partial failures |
| `ERROR` | Handled errors, dependency failures, circuit breaker open |
| `FATAL` | Unrecoverable errors caught at error boundary level |

## 24.3 Governance Rules
- All production logging via OpenTelemetry OTLP log exporter.
- `console.log` / `console.error` prohibited in production builds; enforced via ESLint `no-console` rule.
- Log verbosity configured per environment: `DEBUG` (local), `INFO` (SIT/UAT), `WARN+` (prod).
- PII fields prohibited: see Section 15.4.
- Log schema changes require Platform Team review and backward compatibility window.
- Log retention policy owned by enterprise SIEM / observability platform team.

## 24.4 Folder Integration
- Structured logger lives in `libs/observability/src/logging/structured-logger.ts`.
- Feature libraries use `useLogger` hook or `Logger` service from `libs/observability`.
- No feature library may import a third-party logging library directly.

---

## 25. Platform Lifecycle Governance

## 25.1 Upgrade Review Cadence

| Component | Review Cadence | Owner |
|---|---|---|
| React | Annually + on LTS / major release | Platform Team |
| Node.js | Annually (LTS-aligned) | Platform Team |
| Nx | Quarterly (Nx migrate tool assisted) | Platform Team |
| TypeScript | Semi-annually | Platform Team |
| OpenAPI generator | Semi-annually | Platform Team |
| All npm dependencies | Monthly (automated Snyk / Dependabot) | Platform Team + Security |
| Browser compatibility | Quarterly | Platform Team |

## 25.2 Upgrade Process

```text
1. Platform Team evaluates release notes and breaking changes
2. Internal compatibility test against existing feature libraries
3. Migration plan documented with feature team impact assessment
4. Staged rollout: local → SIT → UAT → Prod
5. Feature teams given migration window: max 2 sprint cycles for minor; 1 quarter for major
6. Hard cutover enforced after migration window; older version support removed
```

## 25.3 Nx Upgrade Governance
- Use `nx migrate latest` with Platform Team review of all generated migration scripts before applying.
- Validate all affected workspace targets and custom generators post-migration.
- Custom Nx executors/generators require explicit compatibility validation after each Nx major release.

## 25.4 React Major Upgrade Gate
- Concurrent Mode and Suspense boundary compatibility review required.
- Third-party library compatibility matrix must be fully resolved before upgrade proceeds.
- Accessibility regression test suite must pass with zero new violations.
- Performance benchmarks (startup, render time, memory) must not regress beyond 5% threshold.
- Feature team sign-off required from each domain team before production rollout.

## 25.5 Dependency Health Policy
- No dependency with a known HIGH or CRITICAL CVE may be deployed to production.
- Automated SBOM (Software Bill of Materials) generated per release and archived.
- Licenses reviewed against enterprise approved-license list (MIT, Apache 2.0, BSD-2/3 preferred).
- Peer dependency conflicts resolved before merge; no suppression in `package.json` without Platform approval.
