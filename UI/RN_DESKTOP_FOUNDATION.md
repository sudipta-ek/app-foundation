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

## 10.2 Governance
- Shortcut registry versioned per release
- Conflict detection required in CI checks
- Accessibility parity with screen reader and focus management

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
