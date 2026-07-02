# Navigation Architecture — Implementation Guide

**Aligned with**: [UI/REACT_NATIVE_FOUNDATION.md](UI/REACT_NATIVE_FOUNDATION.md) — Layer 23

---

## 1. Stack Architecture

```text
Root Navigator  (Platform Team — apps/mobile-shell)
  ├── Auth Stack              # login, MFA, session restore
  ├── Boarding Stack          # Boarding Team domain
  │   ├── BoardingListScreen
  │   ├── PassengerScanScreen
  │   └── BoardingSummaryScreen
  ├── Checkin Stack           # Checkin Team domain
  ├── Flight Ops Stack        # Flights Team domain
  ├── Baggage Stack           # Baggage Team domain
  └── Settings Stack          # Platform Team
```

Rules:
- Platform Team owns the Root Navigator wiring only.
- Each domain team owns its own Stack navigator and screen list.
- No domain stack may import screens from another domain stack directly.
- Stack navigator files live in `apps/mobile-shell/src/navigation/`; screen components live in `libs/features/{domain}/src/screens/`.

---

## 2. Route Contract (TypeScript)

Every navigable screen must declare a typed route contract. Contracts live in `libs/navigation-contracts/`.

```typescript
// libs/navigation-contracts/src/boarding.routes.ts
export type BoardingRoutes = {
  BoardingList: {
    flightId: string;
    airportCode: string;
  };
  PassengerScan: {
    flightId: string;
    passengerId?: string;   // optional: pre-populated from list
  };
  BoardingSummary: {
    flightId: string;
    closedAt: string;
  };
};

// libs/navigation-contracts/src/root-routes.ts
import type { BoardingRoutes } from './boarding.routes';
import type { CheckinRoutes } from './checkin.routes';
import type { FlightOpsRoutes } from './flight-ops.routes';

export type RootRoutes = BoardingRoutes & CheckinRoutes & FlightOpsRoutes & {
  Login: undefined;
  MFAChallenge: { challengeType: 'totp' | 'push' };
};
```

Rules:
- Route params are `undefined` (no params) or a typed object — never `any`.
- Optional params must be explicitly typed as `T | undefined`.
- Route names are PascalCase and unique across all stacks.
- Contract changes require owning domain team PR + Platform Team review.

---

## 3. Route Registry

The central route registry is the **only** way to perform cross-domain navigation. It abstracts the navigator so domain libraries do not import from each other or from `apps/`.

```typescript
// libs/navigation-contracts/src/route-registry.ts
import type { RootRoutes } from './root-routes';

type RouteEntry<T extends keyof RootRoutes> = {
  name: T;
  requiredPermissions: string[];
  featureFlag?: string;
  analyticsId: string;       // immutable telemetry screen ID
  owner: string;             // domain team name
  deepLink?: string;         // airline://domain/action/:param
};

export const ROUTE_REGISTRY: { [K in keyof RootRoutes]?: RouteEntry<K> } = {
  BoardingList: {
    name: 'BoardingList',
    requiredPermissions: ['boarding:view'],
    analyticsId: 'boarding_list',
    owner: 'boarding-team',
    deepLink: 'airline://boarding/list/:flightId',
  },
  PassengerScan: {
    name: 'PassengerScan',
    requiredPermissions: ['boarding:scan'],
    featureFlag: 'BOARDING_SCAN_ENABLED',
    analyticsId: 'boarding_scan',
    owner: 'boarding-team',
  },
};
```

---

## 4. Cross-Domain Navigation Service

```typescript
// libs/navigation-contracts/src/navigation.service.ts
import { createNavigationContainerRef } from '@react-navigation/native';
import type { RootRoutes } from './root-routes';
import { ROUTE_REGISTRY } from './route-registry';
import { permissionStore } from '@airline/auth';
import { featureFlagStore } from '@airline/shared';
import { auditEmitter } from '@airline/analytics';

export const navigationRef = createNavigationContainerRef<RootRoutes>();

export function navigateTo<T extends keyof RootRoutes>(
  route: T,
  params: RootRoutes[T],
  context: { userId: string; correlationId: string }
): void {
  const entry = ROUTE_REGISTRY[route];
  if (!entry) throw new Error(`Route ${String(route)} not registered`);

  // Permission check
  const denied = entry.requiredPermissions.filter(
    p => !permissionStore.has(p)
  );
  if (denied.length > 0) {
    auditEmitter.emit('navigation.denied', {
      route: String(route),
      missingPermissions: denied,
      ...context,
    });
    return;
  }

  // Feature flag check
  if (entry.featureFlag && !featureFlagStore.isEnabled(entry.featureFlag)) {
    return;
  }

  // Emit navigation telemetry
  auditEmitter.emit('navigation.initiated', {
    route: String(route),
    analyticsId: entry.analyticsId,
    ...context,
  });

  navigationRef.navigate(route as any, params as any);
}
```

Rules:
- All cross-domain navigation goes through `navigateTo()` only.
- Within a domain, React Navigation's `useNavigation()` may be used for intra-stack navigation.
- `navigateTo()` silently blocks and emits an audit event on permission failure — it never throws to the caller.

---

## 5. Deep Link Handling

```typescript
// apps/mobile-shell/src/navigation/deep-link.config.ts
export const deepLinkConfig = {
  prefixes: ['airline://', 'https://app.airline.com'],
  config: {
    screens: {
      BoardingList:    'boarding/list/:flightId',
      PassengerScan:   'boarding/scan/:flightId/:passengerId',
      BoardingSummary: 'boarding/summary/:flightId',
      CheckinFlow:     'checkin/:bookingRef',
      FlightStatus:    'flights/:flightId/status',
    },
  },
};
```

**Cold-start deep link flow:**

```text
App not running
  → OS activates app with deep link URL
  → bootstrap-session.ts: restore or establish session first
  → If session valid: navigate to deep link target
  → If session invalid: navigate to Login, preserve deep link intent
  → Post-login: resolve preserved intent and navigate
```

**Warm-start deep link flow:**

```text
App backgrounded
  → OS delivers URL via onOpenURL
  → NavigationService intercepts
  → Permission check via ROUTE_REGISTRY
  → Navigate directly if permitted
```

```typescript
// apps/mobile-shell/src/navigation/deep-link.handler.ts
import { Linking } from 'react-native';
import { navigateTo } from '@airline/navigation-contracts';

export function registerDeepLinkHandler(sessionReady: boolean): void {
  Linking.addEventListener('url', ({ url }) => {
    if (!sessionReady) {
      pendingDeepLink = url;  // preserve for post-login resolution
      return;
    }
    resolveDeepLink(url);
  });
}

export function resolvePendingDeepLink(): void {
  if (pendingDeepLink) {
    resolveDeepLink(pendingDeepLink);
    pendingDeepLink = null;
  }
}
```

---

## 6. Navigation Guards

```typescript
// apps/mobile-shell/src/navigation/guards/auth.guard.ts
import { useEffect } from 'react';
import { useNavigation } from '@react-navigation/native';
import { useSelector } from 'react-redux';
import { selectIsAuthenticated } from '@airline/auth';

export function useAuthGuard(): void {
  const navigation = useNavigation();
  const isAuthenticated = useSelector(selectIsAuthenticated);

  useEffect(() => {
    if (!isAuthenticated) {
      navigation.reset({ index: 0, routes: [{ name: 'Login' }] });
    }
  }, [isAuthenticated, navigation]);
}
```

```typescript
// libs/features/boarding/src/guards/boarding-permission.guard.tsx
import { usePermission } from '@airline/auth';

export function BoardingPermissionGuard({ children }: { children: React.ReactNode }) {
  const canViewBoarding = usePermission('boarding:view');
  if (!canViewBoarding) return <AccessDeniedScreen />;
  return <>{children}</>;
}
```

Guard hierarchy:
1. `AuthGuard` — root level, managed by shell
2. `PermissionGuard` — per-stack, managed by domain
3. `FeatureFlagGuard` — per-screen, managed by domain

---

## 7. Feature Flag-Aware Route Registration

```typescript
// apps/mobile-shell/src/navigation/stacks/boarding.stack.tsx
import { useFeatureFlag } from '@airline/shared';

export function BoardingStack() {
  const scanEnabled = useFeatureFlag('BOARDING_SCAN_ENABLED');

  return (
    <Stack.Navigator>
      <Stack.Screen name="BoardingList" component={BoardingListScreen} />
      {scanEnabled && (
        <Stack.Screen name="PassengerScan" component={PassengerScanScreen} />
      )}
      <Stack.Screen name="BoardingSummary" component={BoardingSummaryScreen} />
    </Stack.Navigator>
  );
}
```

Rules:
- Screens hidden by feature flag are never registered in the navigator — not just hidden in UI.
- Flag-gated routes must have fallback UX if a deep link arrives while flag is off.
- Flag state is evaluated once on stack mount; changes require stack re-render.

---

## 8. Folder Structure

```text
libs/
  navigation-contracts/
    src/
      root-routes.ts               # merged RootRoutes type
      boarding.routes.ts           # BoardingRoutes type
      checkin.routes.ts
      flight-ops.routes.ts
      route-registry.ts            # ROUTE_REGISTRY with permissions + analytics IDs
      navigation.service.ts        # navigateTo() — cross-domain entry point
      deep-link.handler.ts         # intent preservation, cold/warm start
      __tests__/
        navigation.service.spec.ts
        route-registry.spec.ts

apps/
  mobile-shell/
    src/
      navigation/
        root.navigator.tsx         # Root stack wiring
        stacks/
          boarding.stack.tsx
          checkin.stack.tsx
          flight-ops.stack.tsx
          auth.stack.tsx
        guards/
          auth.guard.ts
          session.guard.ts
        deep-link.config.ts        # Linking config + prefixes
        deep-link.handler.ts       # Warm start handler
```

---

## 9. Testing Navigation

```typescript
// libs/navigation-contracts/src/__tests__/navigation.service.spec.ts
import { navigateTo } from '../navigation.service';
import { navigationRef } from '../navigation.service';

describe('navigateTo', () => {
  it('blocks navigation when permission is missing', () => {
    mockPermissionStore(['boarding:view']); // no boarding:scan
    const auditSpy = jest.spyOn(auditEmitter, 'emit');

    navigateTo('PassengerScan', { flightId: 'EK001' }, mockContext);

    expect(navigationRef.navigate).not.toHaveBeenCalled();
    expect(auditSpy).toHaveBeenCalledWith('navigation.denied', expect.objectContaining({
      route: 'PassengerScan',
      missingPermissions: ['boarding:scan'],
    }));
  });

  it('blocks navigation when feature flag is off', () => {
    mockPermissionStore(['boarding:view', 'boarding:scan']);
    mockFeatureFlag('BOARDING_SCAN_ENABLED', false);

    navigateTo('PassengerScan', { flightId: 'EK001' }, mockContext);

    expect(navigationRef.navigate).not.toHaveBeenCalled();
  });
});
```

---

## 10. Governance Gates (DoD)

Every navigation change is compliant when:
- [ ] Route contract declared in `libs/navigation-contracts/` with typed params
- [ ] Route registered in `ROUTE_REGISTRY` with `requiredPermissions`, `analyticsId`, `owner`
- [ ] Cross-domain navigation uses `navigateTo()` — no direct screen imports
- [ ] Deep link pattern registered if screen supports cold-start URL
- [ ] Permission-denied path covered by unit test
- [ ] Feature flag guard in place if screen is flag-gated
- [ ] Telemetry events emitting on navigation initiated and denied
- [ ] Nx boundary tag enforcement: `libs/features/*` does not import another feature lib directly
