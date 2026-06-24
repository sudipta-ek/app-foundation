# React Native Enterprise Authentication (SSO + RBAC)

## 1. Objective
Define enterprise authentication and authorization for operational mobile usage, supporting SSO, role-based access, and policy-driven controls.

This document expands **Layer 5: Enterprise Authentication (SSO + RBAC)** from [UI/REACT_NATIVE_FOUNDATION.md](UI/REACT_NATIVE_FOUNDATION.md).

---

## 2. Supported Identity Providers
- Azure AD (Microsoft Entra ID)
- Okta
- Ping Identity
- SAML-based enterprise IdPs

Protocols:
- OAuth2
- OIDC
- SAML (through enterprise federation flow)

---

## 3. High-Level Auth Flow

```text
User Login
  → IdP Authentication (OIDC/SAML)
  → Token Exchange (BFF/Auth Service)
  → App Session Bootstrap
  → Role/Policy Resolution
  → Route Guard + Feature Access
```

---

## 4. Token and Session Model
- Access token: short-lived
- Refresh token: secure storage, rotation enabled
- Session bound to device identity where required
- Forced re-auth on high-risk transitions (policy-driven)

Storage:
- iOS Keychain
- Android secure storage/keystore-backed solution

---

## 5. Authorization Model

## 5.1 RBAC
- Role-based feature visibility and action entitlement
- Example roles: gate-agent, supervisor, operations-admin

## 5.2 ABAC
- Contextual constraints (airport, gate, shift, device compliance)
- Policy output evaluated at runtime before sensitive action

---

## 6. Suggested Structure

```text
libs/
  auth/
    src/
      providers/
        oidc.provider.ts
        saml.provider.ts
      session/
        session-manager.ts
        token-store.ts
        token-rotation.ts
      authorization/
        rbac-engine.ts
        abac-policy-engine.ts
        permission-resolver.ts
      guards/
        route.guard.tsx
        action.guard.ts
      telemetry/
        auth-events.ts
      index.ts
```

---

## 7. Runtime Enforcement Points
- App bootstrap: token validity and device compliance check
- Route entry: role + policy gate
- Action execution: permission + attribute verification
- API call: auth header + correlation context

---

## 8. Security Controls
- PKCE for public client auth flows
- Token rotation and revocation handling
- Device binding for operational roles
- Jailbreak/root-aware policy enforcement
- No token logging, no token in Redux persisted storage

---

## 9. Operational Policies
- Inactivity timeout and re-auth threshold
- Step-up authentication for high-risk actions
- Break-glass emergency access process (audited)
- Multi-airport policy scoping support

---

## 10. Observability and Audit
Mandatory events:
- `auth_login_started`
- `auth_login_succeeded`
- `auth_login_failed`
- `auth_token_refreshed`
- `auth_session_expired`
- `auth_access_denied`

Audit fields:
- `userId`, `deviceId`, `airportId`, `roleSet`, `correlationId`

---

## 11. Testing
- Unit: permission resolver, policy evaluation
- Integration: IdP login + token refresh + route guard
- E2E: role-based feature access, session expiry, re-auth
- Security: token leakage checks, revoked token behavior

---

## 12. DoD
- SSO flow validated against target IdP(s)
- RBAC/ABAC policy tests passing
- Route/action guards enforced
- Session/token lifecycle tested
- Audit and telemetry available in dashboards
