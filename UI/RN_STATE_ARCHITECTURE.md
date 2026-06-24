# React Native State Architecture (Redux + TanStack Query)

## 1. Objective
Define an enterprise-grade state architecture for React Native where:
- `Redux Toolkit` manages **global client state**
- `TanStack Query` manages **server state**
- Persistent/offline records stay in local database (`Realm`) and not in Redux

This document expands **Layer 1: State Architecture (Redux + TanStack Query)** from [UI/REACT_NATIVE_FOUNDATION.md](UI/REACT_NATIVE_FOUNDATION.md).

---

## 2. Design Principles

1. **Single responsibility by state type**
   - Redux: app/session/UI orchestration
   - TanStack Query: API data cache lifecycle
   - Realm: durable offline data store

2. **No duplicated source of truth**
   - API entities are not copied into Redux unless required for UI orchestration

3. **Predictable and testable flows**
   - Redux slices for deterministic transitions
   - Query/mutation hooks for network and cache lifecycle

4. **Offline-first compatibility**
   - Transactional writes use local queue + sync strategy

5. **PII-safe architecture**
   - Minimize sensitive data in memory caches
   - Never persist secrets in Redux/TanStack cache

---

## 3. State Taxonomy

## 3.1 Redux (Global In-Memory State)
Use Redux for:
- Authentication/session status
- User identity and permissions (RBAC/ABAC outputs)
- App-level UI state (modals, banners, transient wizard state)
- Cross-feature orchestration flags
- Capability status snapshots (if needed globally)

Do not use Redux for:
- Long-lived API entity caches
- Persistent offline records (use Realm)
- Large lists directly from backend when Query already owns cache

## 3.2 TanStack Query (Server State)
Use Query for:
- API reads (flight status, passenger details, manifests)
- Cache invalidation and refetching
- Mutation status and optimistic updates
- Deduplication and stale-time policy

Do not use Query for:
- Global UI workflows
- Device policy/session tokens
- Local-only business workflow steps

## 3.3 Realm (Persistent Local Data)
Use Realm for:
- Offline manifest snapshots
- Boarding write-ahead logs
- Sync queue, reconciliation artifacts
- Last-known configuration for offline fallback

---

## 4. Recommended Folder Structure

```text
libs/
  features/
    boarding/
      src/
        state/
          boarding.slice.ts             # Redux slice (workflow/UI orchestration)
          boarding.selectors.ts         # Memoized selectors
          boarding.actions.ts           # Action creators/thunks if required
          boarding.types.ts             # State and action typings
        queries/
          boarding.query-keys.ts        # Query key factories
          use-boarding-list.query.ts    # Query hooks
          use-boarding-scan.mutation.ts # Mutation hooks
        services/
          boarding-api.service.ts       # API wrappers used by query/mutation hooks
        sync/
          boarding-sync.orchestrator.ts # Realm queue + sync trigger orchestration

  shared/
    state/
      store.ts                          # Redux store setup
      root-reducer.ts                   # Combined reducers
      middleware.ts                     # Logging, audit-safe middleware
      query-client.ts                   # TanStack Query client configuration
```

---

## 5. Ownership Model

| Concern | Owner | Technology | Storage Lifetime |
|---------|-------|------------|------------------|
| Session/auth state | Redux | Redux Toolkit | In-memory, cleared on logout |
| Permissions map | Redux | Redux Toolkit | In-memory, refreshed post-login |
| API read cache | Query | TanStack Query | TTL/stale policy based |
| API mutation lifecycle | Query | TanStack Query | Request lifecycle |
| Offline transactions | Offline layer | Realm | Durable until synced |
| Global modals/alerts | Redux | Redux Toolkit | In-memory transient |

---

## 6. Core Configuration Standards

## 6.1 Redux Standards
- Use `createSlice`
- Keep slice state normalized and minimal
- Use selectors for all screen consumption
- No network calls directly in reducers
- Clear all slices on logout via root reset action

## 6.2 TanStack Query Standards
- Centralized `queryKey` factory per domain
- Mandatory `staleTime` and `gcTime` per query type
- Mutations must define invalidation strategy
- Retry policy based on operation criticality
- Sensitive payloads must not be logged

## 6.3 Query Key Convention
```ts
['boarding', 'flight', flightId, 'manifest']
['boarding', 'passenger', passengerId]
['flights', 'status', airportCode]
```

---

## 7. Decision Matrix (Where should state live?)

| Example | Redux | Query | Realm | Notes |
|---------|------:|------:|------:|-------|
| Logged-in user ID | ✅ | ❌ | ❌ | Session/global concern |
| Boarding list API response | ❌ | ✅ | ⚠️ | Persist to Realm only if offline operation requires |
| Boarding scan pending sync | ❌ | ❌ | ✅ | Durable transactional data |
| Screen filter/sort UI state | ✅ | ❌ | ❌ | UI concern |
| Feature flag values (runtime) | ✅ | ❌ | ⚠️ | Cache fallback optional |
| Push notification token lifecycle | ✅ | ❌ | ❌ | App/global runtime concern |

---

## 8. Data Flow Patterns

## 8.1 Read Pattern
1. Screen calls query hook (`useBoardingListQuery`)
2. Query checks cache freshness
3. Fetch from API if stale/missing
4. Return data + loading/error state
5. Optional persist snapshot to Realm for offline fallback

## 8.2 Write Pattern (Online-first with offline safety)
1. User action triggers mutation hook
2. Mutation attempts API call
3. On network failure, write transaction to Realm queue
4. Redux updates local workflow status (e.g., `syncPending`)
5. Sync orchestrator replays queued events when online

## 8.3 Global UI Pattern
1. Feature emits Redux action (`showGlobalErrorBanner`)
2. Global UI slice updates
3. Shell renders banner/modal consistently across features

---

## 9. Error Handling Strategy

- Query errors: handled in query hooks and mapped to user-safe UI states
- Mutation errors: mapped to domain error categories (`retryable`, `business`, `auth`, `fatal`)
- Redux errors: no raw exception payloads in state
- All critical failures emit telemetry and correlation context

---

## 10. Security and Compliance Controls

- No secrets in Redux state
- No PII in query keys
- Avoid persisting restricted data in query cache snapshots
- Clear in-memory state on logout/session expiry
- Redact sensitive fields in logs and error boundaries

---

## 11. Testing Strategy

## 11.1 Unit Tests
- Slice reducers and selectors
- Query key factories
- Mutation invalidation logic

## 11.2 Integration Tests
- Query hooks with mocked API
- Offline fallback path with Realm queue simulation
- Redux + Query interaction on mutation failure/retry

## 11.3 E2E Coverage
- Login/logout state reset
- Boarding scan with online + offline transitions
- Cache refresh after background/foreground cycle

---

## 12. Performance Guidelines

- Keep Redux state small and serializable
- Use memoized selectors (`reselect`)
- Configure query stale time to reduce redundant calls
- Prefetch critical queries on navigation transitions
- Avoid unnecessary cache invalidation broad-casts

---

## 13. Anti-Patterns (Disallowed)

- Copying full API response objects into Redux slices
- Using Redux as offline database
- Persisting auth tokens in multiple stores without ownership
- Ad-hoc query keys not aligned with key factory
- Feature-to-feature direct state coupling without contracts

---

## 14. Example DoD for New Feature State

A feature is state-architecture compliant only if:
- Redux slice exists only for workflow/global UI needs
- Query hooks manage all API read/write lifecycle
- Query keys follow domain key factory
- Offline transactions are routed to Realm queue where required
- Logout reset behavior is tested
- PII handling and logging controls are verified

---

## 15. Implementation Checklist

- [ ] Define feature state ownership (Redux vs Query vs Realm)
- [ ] Add/extend query key factory
- [ ] Create query + mutation hooks
- [ ] Add minimal Redux slice for UI/session orchestration only
- [ ] Wire offline queue handling (if operationally required)
- [ ] Add tests (unit + integration)
- [ ] Validate telemetry and correlation propagation
- [ ] Review against governance and compliance controls
