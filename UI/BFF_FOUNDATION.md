# BFF Foundation Specification (Mobile + Desktop, JDK 21 Baseline)

## 1. Objective
Define a production-grade **Backend for Frontend (BFF) foundation** aligned to [UI/REACT_NATIVE_FOUNDATION.md](UI/REACT_NATIVE_FOUNDATION.md), covering scope boundaries, platform capabilities, governance, and implementation structure.

## 1.1 Runtime Baseline (Enterprise Standard)
- **Language/Runtime**: Java on **JDK 21 (LTS)**
- **Framework**: Spring Boot 3.3+ (or enterprise-approved equivalent on Jakarta EE stack)
- **Build**: Maven 
- **API Contracts**: OpenAPI-first with generated DTO/client stubs where applicable
- **Observability**: OpenTelemetry + enterprise logging/metrics stack

> **Decision**: JDK 21 is the default BFF runtime for this foundation 

---

## 2. BFF Foundation Scope

The BFF layer is responsible for **experience-facing composition** and **channel-specific contracts** for mobile/desktop clients.

### In Scope
- API aggregation and response shaping for client views
- Channel-aware endpoint design (mobile/desktop)
- OAuth token validation and session context enrichment
- RBAC/ABAC policy evaluation hooks
- Event publishing/subscription integration for operational workflows
- Push notification orchestration hooks
- Feature-flag-aware response behavior
- Versioned API contracts and backward compatibility controls
- Request correlation, distributed tracing, and audit-safe logging
- Resilience controls (retry/circuit-breaker/timeout/fallback)

### Out of Scope
- Core domain business logic (owned by backend microservices)
- System-of-record data ownership
- Direct client access to secrets vaults
- Long-term immutable audit storage ownership (BFF forwards, does not own store)

---

## 3. Core Capabilities

## 3.1 Experience API Composition
- Compose data from multiple microservices into client-ready DTOs
- Reduce client-side orchestration and round trips
- Enforce pagination/filter/sort normalization

## 3.2 Contract Management
- OpenAPI-first contracts (versioned)
- Consumer-driven compatibility verification (Pact or Spring Cloud Contract)
- Deprecation lifecycle with sunset policy

## 3.3 Security Gateway Capabilities
- Access token validation/introspection
- Claims enrichment (`userId`, `role`, `airport`, `deviceId`)
- Scope enforcement per route/action
- Request signing/validation where required

## 3.4 Authorization Integration (RBAC/ABAC)
- Route and action-level policy checks
- Context-aware decisions (airport, gate, shift, device compliance)
- Consistent deny reason model for client UX and audit

## 3.5 Realtime/Event Bridge
- Publish operational events to Solace
- Consume selected events for response enrichment and async workflows
- Correlation propagation: `correlationId`, `traceId`

## 3.6 Notification Orchestration
- Server-side audience resolution (session / role / airport / device token registry queries)
- Provider HTTP integration (FCM HTTP v1 and APNS HTTP/2) with Vault-sourced credentials
- Fan-out cap enforcement (10,000 token limit per dispatch; ARB gate for oversized batches)
- Acknowledgement registry and per-policy suppression (`none` / `one-time` / `any-ack-suppresses-all` / `individual-ack`)
- All orchestration, dispatch, and ack decisions emitted to the audit store via `bff-audit-core`
- UI layer is responsible only for device token registration, notification handling, and calling the BFF `/v1/notifications/{id}/ack` endpoint

> Push notification ownership model: [UI/RN_PUSH_NOTIFICATION_STRATEGY.md — Section 1.1](UI/RN_PUSH_NOTIFICATION_STRATEGY.md)

## 3.7 Resilience & Reliability
- Per-dependency timeout budgets
- Retry with jitter for transient faults
- Circuit breaker and fallback response model
- Idempotency support for critical write endpoints

## 3.8 Performance Controls
- Response caching for safe read endpoints
- Request coalescing where applicable
- SLA-aware route budgets and SLO telemetry

## 3.9 Observability
- Structured logging (PII-safe)
- Distributed tracing (OpenTelemetry)
- Metrics per route/dependency/error class
- Audit-safe action records for regulated workflows

#### Metric Dimensions (Standard Label Set)

All BFF metrics must carry these dimensions for consistent dashboards and alerting:

| Dimension | Values |
|---|---|
| `service` | BFF application name (e.g. `bff-gateway`) |
| `endpoint` | Route path pattern (e.g. `/v1/boarding/{id}`) |
| `airport` | IATA airport code from `RequestContext` |
| `channel` | `mobile` \| `desktop` |
| `dependency` | Name of downstream service called |
| `errorCode` | Stable error taxonomy code |
| `userRole` | Resolved role from `RequestContext` |
| `clientVersion` | Client app version from `RequestContext` |

#### Standard Metric Names

| Metric | Type | Description |
|---|---|---|
| `bff.request.duration` | Histogram | End-to-end BFF request latency per endpoint |
| `bff.client.duration` | Histogram | Per-dependency downstream call latency |
| `bff.cache.hit` | Counter | Cache hits per endpoint and dependency |
| `bff.cache.miss` | Counter | Cache misses per endpoint and dependency |
| `bff.retry.count` | Counter | Retry attempts per dependency |
| `bff.circuitbreaker.open` | Gauge | Circuit breaker in OPEN state (1 = open, 0 = closed) |

## 3.10 Configuration & Feature Governance
- Runtime config and feature flag evaluation
- Airport-scoped behavior toggles
- Four-eye approval workflow for production config changes

---

## 4. BFF Ownership Model

| Area      | BFF Owns                                 | BFF Does NOT Own |
|-----------|------------------------------------------|------------------|
| API shape | Channel-specific DTOs and aggregation    | Domain entity source of truth |
| Security  | Token/claim/scope enforcement            | Enterprise IdP lifecycle ownership |
| Policy    | Route/action policy integration          | Authoring core business policy rules in domain services |
| Events    | Publishing/bridging for client workflows | Event backbone platform ownership |
| Audit     | Emitting audit-safe records              | Immutable audit storage platform |

---

## 5. Reference Architecture

```text
Client (Mobile/Desktop)
  → BFF API Gateway Layer
    → Auth/Policy Middleware
    → Experience Endpoints (composition)
    → Domain Service Clients
    → Event/Notification Integrations
    → Observability/Audit Hooks
  → Microservices / Solace / Notification Services
```

---

## 6. Recommended Folder Structure

```text
app-foundation/                              # Maven aggregator parent POM
  pom.xml                                    # parent POM — manages all module versions
  libs/                                      # non-deployable library modules (see Section 6.2)
    bff-context-core/                        # RequestContext model — foundation for all libs
    bff-middleware-core/                     # HTTP filters: correlation, request context population
    bff-auth-core/                           # Token validation, claims enrichment, session resolution
    bff-resilience-core/                     # Retry/timeout/circuit-breaker, BaseServiceClient
    bff-caching-core/                        # Cache policies, TTL governance, key conventions
    bff-events-core/                         # Solace publisher/subscriber abstractions
    bff-notifications-core/                  # Audience resolution, provider dispatch (FCM/APNS), ack registry
    bff-audit-core/                          # Audit emitter, PII-safe event mapper
  apps/                                      # deployable BFF applications
    bff-gateway/
      pom.xml                                # depends on all libs/ modules above
      src/
        main/
          java/
            com/airline/bff/
              bootstrap/
                AppBootstrapConfig.java      # Spring Boot startup/wiring config
              api/
                mobile/
                  BoardingController.java
                  FlightsController.java
                desktop/
                  OperationsController.java
              dto/
                mobile/
                  BoardingResponseDto.java
                desktop/
                  OperationsGridDto.java
              composition/
                BoardingComposer.java        # multi-service response shaping
                FlightsComposer.java
              authorization/
                RoutePolicyGuard.java        # uses bff-auth-core + bff-context-core
                ActionPolicyGuard.java
                PolicyDecisionMapper.java
              integrations/
                services/
                  DcsClient.java            # extends BaseServiceClient (bff-resilience-core)
                  OpsClient.java
                  CustomerClient.java
              mappers/
                PassengerMapper.java             # PassengerModel → mobile/desktop DTO
                FlightMapper.java                # FlightModel → response DTO
                BoardingMapper.java              # PassengerModel + FlightModel → BoardingPassDto
              errors/
                ErrorCatalog.java
                ErrorMapper.java
              observability/
                TracingFilter.java
                MetricsRegistry.java
                StructuredLoggingConfiguration.java  # structured JSON logging configuration
        main/resources/
          application.yml
          application-dev.yml
          application-sit.yml
          application-uat.yml
          application-prod.yml
          openapi/
            bff-mobile.v1.yaml
            bff-desktop.v1.yaml
        test/
          java/
            com/airline/bff/
              unit/
              integration/
              contract/
              performance/
```

### 6.1 Implementation Notes (JDK 21)
- Use `java.net.http` or enterprise-standard HTTP clients with connection pooling/timeouts.
- Use Spring Security OAuth2 resource server support for token validation and scope enforcement.
- Use Resilience4j (or enterprise equivalent) for circuit-breaker/retry/timeout patterns.
- Keep package/module boundaries aligned to domain and capability ownership.

#### Virtual Thread (VT) Guidance

| Use Virtual Threads ✔ | Avoid Virtual Threads ✖ |
|---|---|
| Parallel downstream REST calls in Composers | CPU-intensive aggregation or transformation logic |
| Blocking I/O in service client calls | `synchronized` blocks on shared mutable state |
| Parallel database or cache reads | Code with `ThreadLocal` assumptions |
| Aggregation requests with multiple blocking waits | Non-blocking reactive pipelines |

Enable globally in Spring Boot 3.2+: `spring.threads.virtual.enabled=true`

---

## 6.2 Cross-Cutting Library Module Strategy

Cross-cutting capabilities (`auth`, `events`, `notifications`, `resilience`, `caching`, `audit`, `middleware`) are extracted into **non-deployable Maven library modules** (`libs/`) owned by the Platform Team. Each library is independently versioned and plugged in as a compile-time dependency by any deployable BFF application.

### Merit

| Concern | In-App Co-located | Library Module |
|---|---|---|
| Reuse | Re-implemented per BFF deployable | Shared across all BFF apps |
| Versioning | Coupled to feature releases | Independent lifecycle |
| Governance | App team must implement correctly | Platform Team owns and enforces via dependency |
| Testability | Requires full app context to test | Isolated, independently testable artifact |
| Upgrade surface | Feature code touched on infra upgrades | Single library update; apps re-verify |
| Compliance | Each app re-certifies cross-cutting behavior | One certifiable artifact per lib |
| Multi-BFF readiness | Duplication if second BFF introduced | Zero-cost reuse for any future BFF deployable |

### Library Internal Structure

```text
libs/
  bff-context-core/                            # Foundation model — no BFF lib dependencies
    src/main/java/com/airline/bff/context/
      RequestContext.java                       # Immutable carrier: correlationId, userId, etc.
      RequestContextFactory.java               # Build context from incoming HTTP request
      RequestContextHolder.java                # VirtualThread-safe context propagation

  bff-middleware-core/                         # HTTP filter chain (depends: bff-context-core)
    src/main/java/com/airline/bff/middleware/
      CorrelationFilter.java                   # Inject/propagate correlationId
      RequestContextFilter.java                # Populate RequestContext per request

  bff-auth-core/                               # Auth framework (depends: bff-context-core)
    src/main/java/com/airline/bff/auth/
      TokenValidator.java                      # OAuth2/OIDC token validation
      ClaimsEnricher.java                      # Extract and augment claims into RequestContext
      SessionContextResolver.java              # Resolve session from validated token

  bff-resilience-core/                         # Resilience + client base (depends: bff-context-core)
    src/main/java/com/airline/bff/resilience/
      RetryPolicy.java                         # Configurable retry with exponential backoff/jitter
      TimeoutPolicy.java                       # Per-dependency timeout budgets
      CircuitBreakerPolicy.java                # Resilience4j circuit breaker configuration
      IdempotencyStore.java                    # Idempotency key tracking for write endpoints
      BaseServiceClient.java                   # Abstract base: timeout + retry + CB + metrics + tracing
      ServiceClientPolicyRegistry.java         # Registry of per-downstream-service policy configurations

  bff-caching-core/                            # Cache governance (depends: bff-context-core)
    src/main/java/com/airline/bff/caching/
      ResponseCachePolicy.java                 # TTL, invalidation trigger, stale-read rules
      CacheKeys.java                           # Canonical and consistent cache key conventions

  bff-events-core/                             # Solace event abstractions (independent)
    src/main/java/com/airline/bff/events/
      SolacePublisher.java                     # Publish events to Solace backbone
      SolaceSubscriber.java                    # Subscribe to Solace topic consumers

  bff-notifications-core/                      # Notification orchestration (depends: bff-context-core)
    src/main/java/com/airline/bff/notifications/
      audience/
        AudienceSpec.java                        # Sealed interface: 6 audience class variants
        AudienceResolver.java                    # Resolves device token set for a given AudienceSpec
        TokenRegistryClient.java                 # Queries token registry by session / role / airport
        AudienceFanOutGuard.java                 # Enforces 10k token cap; routes oversized dispatch to ARB
      dispatch/
        NotificationOrchestrator.java            # Entry point: drives full audience resolve + dispatch flow
        NotificationDispatcher.java              # Routes canonical payload to FCM or APNS provider client
        FcmProviderClient.java                   # FCM HTTP v1 API integration
        ApnsProviderClient.java                  # APNS HTTP/2 API integration
        PayloadBuilder.java                      # Builds provider-specific payload from canonical request
      acknowledgement/
        AcknowledgementRegistry.java             # Server-side ack persistence (per-user, per-notification)
        NotificationPolicyEvaluator.java         # Evaluates policy; suppresses pending sends on ack
        AckRequest.java                          # Ack endpoint request model
      config/
        NotificationProviderConfig.java          # Vault-sourced FCM/APNS credentials; per-env isolation

  bff-audit-core/                              # Audit framework (depends: bff-context-core)
    src/main/java/com/airline/bff/audit/
      AuditEmitter.java                        # Emit PII-safe structured audit records
      AuditEventMapper.java                    # Map domain actions to canonical audit events
```

### Library Dependency Graph

```text
bff-context-core                    ← no BFF library dependencies (foundation)
  ↑
  ├── bff-middleware-core
  ├── bff-auth-core
  ├── bff-resilience-core
  ├── bff-caching-core
  ├── bff-notifications-core            ← depends: bff-context-core (AudienceResolver uses RequestContext)
  └── bff-audit-core

bff-events-core                     ← independent (Solace abstractions only)

bff-gateway (app)
  → bff-context-core
  → bff-middleware-core
  → bff-auth-core
  → bff-resilience-core
  → bff-caching-core
  → bff-events-core
  → bff-notifications-core
  → bff-audit-core
```

### Governance Rules for Library Modules

- Feature teams must **not** re-implement or copy library behaviors in app-layer code.
- Library upgrades are managed by Platform Team; consuming apps must update within the agreed migration window.
- All `libs/` modules are published to the internal Maven artifact registry with semantic versioning.
- Breaking library API changes require ARB review and consumer impact assessment before release.
- Each library module ships its own unit and integration test suite; consuming apps run compatibility verification.

---

## 6.3 Repository Strategy

### The Three Options

| Dimension                            | Option A: Single Monorepo (libs + apps)                 | Option B: Dedicated Libs Repo (all 8 libs) | Option C: 8 Independent Repos |
|---------------------------------------|---------------------------------------------------------|---------------------------------|---|
| **Versioning**                        | Parent POM + BOM manages all versions atomically        | Single BOM release from libs repo; apps consume by BOM version | 8 independent versions; app must track 8 separate artifact coordinates |
| **Cross-lib refactoring**             | Atomic — single commit handles `bff-context-core` + all 5 dependents | Single repo, ordered Maven reactor — 1 PR covers all 8 | Requires 8 coordinated PRs across 8 repos in dependency order |
| **Lib + app change atomicity**        | Single commit, one PR — no coordination overhead        | Lib repo PR merged + artifact published → app repo PR | Lib repos must be released before app can be updated |
| **CI pipeline count**                 | 1 pipeline (path-filtered per module)                   | 1 pipeline (libs) + 1 pipeline (apps) | 8 lib pipelines + 1 app pipeline = 9 pipelines |
| **Team ownership enforcement**        | CODEOWNERS: `libs/**` → Platform Team only              | Repo-level access control (cleaner separation) | Repo-level access control per lib |
| **Sharing with other repos/teams**    | Requires extracting libs to separate repo at that point | Zero friction — libs repo is already standalone | Zero friction per lib |
| **Dependency graph visibility**       | Maven reactor enforces build order natively             | Maven reactor within libs repo | No native enforcement; manual coordination |
| **Onboarding / discoverability**      | Single clone, full context immediately                  | Two repos to clone and understand | Nine repos to navigate |
| **Operational overhead**              | Low — one repo, one branching strategy                  | Medium — two repos, coordinated releases | High — 8 branching strategies, 8 release processes |
| **Enterprise inner-source readiness** | Requires migration when sharing is needed               | Ready to share from day one | Each lib individually shareable but version matrix unmanageable |

### Key Constraint: `bff-context-core` Transitive Dependency

```text
bff-context-core  ←  bff-middleware-core
                  ←  bff-auth-core
                  ←  bff-resilience-core
                  ←  bff-caching-core
                  ←  bff-audit-core
```

A single breaking change to `RequestContext` (e.g., adding a required field) requires coordinated updates across **5 dependent libs and the consuming app**. This is the dominant constraint:

- **Option C** turns this into 7 coordinated cross-repo PRs in strict dependency order — unacceptable operational risk.
- **Option B** handles it in 1 PR within the libs repo Maven reactor.
- **Option A** handles it as 1 atomic commit across the full codebase.

### Recommendation: Option A → Evolve to Option B

**Start with Option A (Monorepo).** Appropriate for the current scope: single product, single Platform Team, `bff-gateway` as the only consumer.

```text
app-foundation/               # single git repository
  pom.xml                     # aggregator parent POM (revision property + BOM)
  libs/                       # Platform Team CODEOWNERS
    bff-context-core/
    bff-middleware-core/
    bff-auth-core/
    bff-resilience-core/
    bff-caching-core/
    bff-events-core/
    bff-notifications-core/
    bff-audit-core/
  apps/                       # Domain Teams CODEOWNERS (with Platform review for infra changes)
    bff-gateway/
```

**Evolve to Option B** (extract `libs/` to a dedicated repo) only when **one or more** of the following triggers is met:

| Evolution Trigger                                                                                            | Why It Justifies the Split |
|--------------------------------------------------------------------------------------------------------------|---|
| A second BFF application in a different repo needs to consume the libs                                       | Libs must be independently publishable and consumable |
| Platform Team and App Teams formally separate release cadences (e.g., libs on quarterly, app on fortnightly) | Repo boundary enforces the lifecycle separation |
| An enterprise inner-source programme formally adopts these libs for other airline platform teams             | Dedicated repo needed for external contributor governance |
| Libs reach a stable v1.x LTS milestone requiring long-term maintenance separate from app churn               | Repo isolation protects LTS from feature branch noise |

**Never Option C.** The 8-repo model provides no benefit that Option B does not, while adding 8× the operational overhead and creating an unmanageable version coordination problem for any `bff-context-core` breaking change.

### Versioning Approach for Monorepo (Option A)

Use the Maven **CI-friendly revision pattern** with a **BOM module**:

```text
app-foundation/
  pom.xml                          # parent: <revision>1.0.0</revision>
  bff-platform-bom/
    pom.xml                        # BOM: imports all lib versions from ${revision}
  libs/
    bff-context-core/pom.xml       # version: ${revision}
    bff-auth-core/pom.xml          # version: ${revision}
    ...
  apps/
    bff-gateway/pom.xml            # imports bff-platform-bom for consistent lib versions
```

Rules:
- All libs share the same `${revision}` — released together as a platform bundle.
- Consuming app (`bff-gateway`) imports `bff-platform-bom` and never specifies individual lib versions.
- `maven-flatten-plugin` resolves `${revision}` in published POMs so artifacts are consumable without parent POM.
- Individual patch releases for a single lib are allowed via a separate `<version>` override — requires ARB sign-off and a documented consumer impact assessment.

### Platform BOM Governance

| Rule | Detail |
|---|---|
| BOM ownership | Only Platform Team may modify `bff-platform-bom/pom.xml` |
| Application constraint | Applications import `bff-platform-bom` only; never specify individual `bff-*` library versions directly |
| Direct version prohibition | Specifying `<version>` for any `bff-*` library in an application POM is a build policy violation enforced by CI |
| CI enforcement | Dependency validation step fails the build if any `bff-*` dependency carries a version outside the BOM |
| BOM release cadence | BOM version increments follow platform release cadence; patch releases for critical security fixes only |
| Consumer communication | BOM releases published to internal artifact registry with release notes; consuming apps must upgrade within agreed migration window |

### CODEOWNERS Pattern

```text
# .github/CODEOWNERS  (or GitLab equivalent CODEOWNERS)
libs/**                   @platform-team
apps/bff-gateway/src/main/java/com/airline/bff/composition/**   @domain-teams
apps/bff-gateway/src/main/java/com/airline/bff/api/**           @domain-teams
apps/bff-gateway/src/main/java/com/airline/bff/authorization/**  @domain-teams @platform-team
apps/**                   @domain-teams @platform-team
```

---

## 6.4 Dependency Injection Rules

Architecture dependency direction — enforced via ArchUnit tests in CI:

```text
api/ (Controllers)
  → composition/ (Composers)
    → integrations/services/ (Service Clients)
    → mappers/
    → dto/

authorization/
  → bff-auth-core (library)

integrations/services/
  → BaseServiceClient (bff-resilience-core library)

dto/         ← no dependencies (pure data carriers)
errors/      ← no domain dependencies (shared utility)
```

Rules:
- **Controllers → Composers only.** Controllers never call service clients, mappers, or downstream services directly.
- **Composers → Service Clients + Mappers + DTOs.** A Composer must never call another Composer.
- **Service Clients → `BaseServiceClient` only** for resilience behavior. No REST calls outside the client class.
- **DTOs have no dependencies** on any other BFF package.
- **Dependency violations are detected by ArchUnit** architecture tests run as part of the CI build gate.

---

## 6.5 Composition Pattern

```text
Controller
  → Composer (orchestrate, aggregate, normalize, map)
    → [VT] ServiceClient.callA()   — parallel where independent
    → [VT] ServiceClient.callB()
    ← Canonical domain models
    → Mapper.toDto(model1, model2)
    ← DTO
  → HTTP Response (standard envelope)
```

**Composer responsibilities ✔**
- Orchestrate parallel or sequential downstream calls
- Aggregate results from multiple service clients
- Normalize response shapes across services
- Delegate all DTO mapping to dedicated Mapper classes

**Composer must NOT ✖**
- Contain business rules or domain logic
- Persist data or own transactions
- Call another Composer
- Call downstream REST directly (always via typed Client interface)

> `composition/` is the correct name when the primary responsibility is API aggregation and response shaping. Rename to `orchestration/` only if classes evolve to coordinate multi-step stateful workflows.

---

## 6.6 Service Client Standards

Each downstream service has exactly one typed client interface:

```java
// One client per service — typed interface — canonical models only
interface PassengerClient {
    PassengerModel getPassenger(String passengerId, RequestContext ctx);
}

interface FlightClient {
    FlightModel getFlight(String flightId, RequestContext ctx);
}
```

Rules:
- **One client interface per downstream service** — no shared multi-service clients.
- **No REST calls outside the client class.** HTTP calls are fully encapsulated within the client implementation.
- **Retry, timeout, and circuit breaker live in `BaseServiceClient` only** — client implementations must not add resilience logic.
- **Clients return canonical domain models** (`PassengerModel`, `FlightModel`), never DTOs.
- **No business mapping inside client classes** — mapping is the responsibility of the Mapper layer.
- All clients are registered in `ServiceClientPolicyRegistry` with per-service policy configuration.

---

## 6.7 DTO Governance

All BFF response DTOs must comply with:

| Rule | Detail |
|---|---|
| Immutable | Use Java records — no setters, no mutation after construction |
| Java records preferred | `record BoardingPassDto(...) {}` over class |
| No JPA annotations | DTOs are not persistence entities; no `@Entity`, `@Column`, etc. |
| No validation logic | Validation annotations (`@NotNull` etc.) on request input DTOs only; response DTOs are annotation-free |
| No business methods | DTOs carry data only — no domain logic, no computed fields with business meaning |
| No downstream entity reuse | Never expose internal domain entities, JPA entities, or downstream API models as API response |

```java
// Correct — Java record response DTO
public record BoardingPassDto(
    String passengerId,
    String flightNumber,
    String seatNumber,
    String boardingGroup,
    String gate
) {}
```

---

## 6.8 Mapping Layer

Mappers translate canonical service models into client-facing DTOs. Isolating mapping from Composers keeps Composers lean and makes mapping independently unit-testable.

```text
com/airline/bff/mappers/
  PassengerMapper.java       # PassengerModel → PassengerDto
  FlightMapper.java          # FlightModel → FlightDto
  BoardingMapper.java        # PassengerModel + FlightModel → BoardingPassDto
```

Rules:
- **Mappers are pure functions** — no I/O, no side effects, no shared state.
- **One mapper per domain concept or DTO family.**
- Mappers are **unit-tested independently** of Composers and Controllers.
- Mappers may accept **multiple canonical models** when a DTO aggregates data from multiple services (e.g., `BoardingMapper` accepts `PassengerModel` + `FlightModel`).
- **Mapping logic must not contain business rules.** Domain conditions belong in domain services, not mappers.

---

## 6.9 OpenAPI Generation Flow

```text
OpenAPI spec (bff-mobile.v1.yaml / bff-desktop.v1.yaml)
  ↓ openapi-generator-maven-plugin (build phase)
Generated request/response DTOs       → src/main/generated/
Generated downstream client stubs     → src/main/generated/ (from service-published specs)
  ↓
Composition layer consumes generated types
  ↓
Controller implements generated API interface
```

Rules:
- Generated sources live in `src/main/generated/` and are **excluded from manual editing**.
- Downstream service client stubs are generated from service-published OpenAPI specs where available.
- Manual DTO additions are prohibited when a generated equivalent exists.
- Generation is a **build-time step gated in CI** — build fails if spec and generated code diverge.
- Generated DTOs may be consumed by mappers but must never be modified in-place.

---

## 6.10 Request Flow Sequence Diagram

```text
Mobile Client
  ─→ BoardingController.getBoarding(passengerId)
       ─→ BoardingComposer.compose(passengerId, requestContext)
            ─→ [VT] PassengerClient.getPassenger(passengerId, ctx)
            ─→ [VT] FlightClient.getFlight(flightId, ctx)
            ←─ PassengerModel
            ←─ FlightModel
            ─→ BoardingMapper.toDto(passengerModel, flightModel)
            ←─ BoardingPassDto
       ←─ BoardingPassDto
  ←─ HTTP 200 { data: BoardingPassDto, correlationId, meta }
```

Sequence rules:
- Controller always returns the standard response envelope (`data`, `meta`, `errors`, `correlationId`).
- Parallel downstream calls are coordinated inside the Composer using Virtual Threads (`[VT]`).
- No domain models or downstream entities cross the Controller boundary — DTOs only.
- `correlationId` and `traceId` from `RequestContext` are included in every response envelope.

---

## 6.11 Configuration Hierarchy

Runtime configuration is resolved in strict precedence order (highest → lowest):

```text
Vault                         ← secrets: credentials, keys, tokens  (highest priority)
  ↓
Environment variables         ← container/platform-injected config
  ↓
application-{env}.yml         ← environment-specific overrides (prod / uat / sit / dev)
  ↓
application.yml               ← shared defaults
  ↓
Compiled defaults              ← code-level defaults                  (lowest priority)
```

Rules:
- **Secrets must come from Vault only.** Credentials, tokens, or keys in any yml file is a security policy violation.
- **Environment-specific overrides** live in `application-{env}.yml`; shared config in `application.yml`.
- **No hardcoded config values** in Java source code.
- Configuration model classes use `@ConfigurationProperties` with typed, validated POJOs.
- **Production config changes require four-eye approval** workflow before deployment.

---

## 7. API Capability Standards

- Versioning: `/v1`, `/v2` for major changes
- Response envelope standard:
  - `data`
  - `meta`
  - `errors`
  - `correlationId`
- Error taxonomy:
  - `validation_error`
  - `authorization_error`
  - `dependency_timeout`
  - `dependency_unavailable`
  - `business_conflict`

## 7.1 API Version Lifecycle (Mandatory)

```text
v1 Active
  ↓
Deprecated
  ↓
Sunset Header
  ↓
Retired
```

Rules:
- Semantic versioning is mandatory (`MAJOR.MINOR.PATCH` for contracts).
- Minimum support window after deprecation announcement: **6 months**.
- Breaking changes require Architecture Review Board approval.
- Consumer notification channels are mandatory (release notes + API portal + targeted team notice).
- `Deprecation` and `Sunset` headers are required once endpoint enters deprecation state.

## 7.2 Consumer Contract Testing Standard
- Approved tooling: **Pact** or **Spring Cloud Contract** (enterprise standard must select one primary).
- Required flow:

```text
Producer
  ↓ publishes contract
Contract Registry
  ↓
Consumer Verification
```

- Merge gate: producer/consumer verification must pass for changed contracts.
- BFF must test compatibility for `current` and `previous` major contract versions during migration windows.

## 7.3 Enterprise Error Contract

All BFF errors returned to clients must conform to:

```json
{
  "errorCode": "authorization_error",
  "message": "User is not authorized for this action",
  "correlationId": "9cbff51e-7ff5-4acd-b710-f1944fcbf001",
  "retryable": false,
  "severity": "HIGH",
  "category": "SECURITY"
}
```

Error fields:
- `errorCode` (stable machine-readable code)
- `message` (safe user/developer-facing message)
- `correlationId` (traceability)
- `retryable` (client behavior hint)
- `severity` (`LOW|MEDIUM|HIGH|CRITICAL`)
- `category` (`VALIDATION|SECURITY|DEPENDENCY|BUSINESS|SYSTEM`)

---

## 8. Security Baseline

- OAuth2/OIDC validation on all protected endpoints
- Mutual TLS/service auth for internal downstream calls (where applicable)
- PII redaction for logs and traces
- Rate limiting and abuse controls per client/device
- Secrets only from vault-backed runtime injection
- JVM baseline hardened to enterprise standard for JDK 21 runtime and container image policies

## 8.1 Security Threat Model (BFF)

| Threat | Mitigation |
|---|---|
| JWT replay | Short-lived tokens, nonce/session checks, token binding where applicable |
| API scraping | Rate limiting, API key/client fingerprint controls, anomaly detection |
| DoS | Gateway throttling, circuit breakers, autoscaling and backpressure |
| SSRF | Outbound allow-list, strict egress controls, URL validation |
| Header spoofing | Trusted proxy validation, canonical header policy |
| Injection | Bean validation, parameterized queries, strict input validation |
| Deserialization attacks | Safe Jackson configuration, allow-listed polymorphic types |

## 8.2 Request Context Model (Standard)

Every BFF layer must receive a single `RequestContext` object instead of scattered parameters.

Required fields:
- `correlationId`
- `traceId`
- `userId`
- `deviceId`
- `airport`
- `gate`
- `roles`
- `permissions`
- `locale`
- `clientVersion`
- `featureFlags`

---

## 9. Operational SLOs (Baseline)

| Metric | Target | Alert Threshold |
|---|---|---|
| P95 BFF response latency | < 400ms | > 700ms |
| Error rate (5xx) | < 0.5% | > 1.0% |
| Dependency timeout rate | < 0.3% | > 0.8% |
| Contract test pass rate | 100% | < 100% |

## 9.1 Performance Budget Decomposition (P95)

Total target: **< 400ms**

| Budget Area | Target |
|---|---|
| Authentication / authz | 30ms |
| Composition logic | 120ms |
| Dependency A | 100ms |
| Dependency B | 80ms |
| Serialization / response write | 20ms |
| Buffer (headroom) | 50ms |

Rules:
- Budget must be documented per endpoint family.
- Endpoints exceeding budget require either optimization, cache, or async redesign.

## 9.2 Dependency Fan-Out Limit
- Maximum synchronous downstream calls per request: **4** (hard cap **5** with ARB exception).
- Beyond the limit, teams must use one or more:
  - async processing
  - pre-computed read models
  - cached projections

## 9.3 Async Processing Pattern

Use async processing for long-running or multi-step orchestration.

```text
POST
  ↓
202 Accepted
  ↓
Solace
  ↓
Worker
  ↓
Notification / callback / polling endpoint
```

Classification:
- **Synchronous**: immediate, user-blocking operations within SLA budget
- **Asynchronous**: accepted and completed later, tracked by operation ID
- **Fire-and-forget**: non-critical side-effects with audit/telemetry only

## 9.4 Caching Governance

Never cache:
- auth/session responses
- permissions/role decisions
- real-time boarding state

Cache-eligible (read-mostly):
- airport configuration
- flight schedule metadata
- gate metadata

Baseline TTL guidance:
- Flight metadata: **30 sec**
- Airport config: **30 min**
- Passenger profile data: **No cache by default** (exception requires approval)

All cache policies must define:
- TTL
- invalidation trigger
- stale-read behavior
- data classification impact

---

## 10. Governance Gates (Mandatory)

A BFF release is compliant only if:
- OpenAPI contracts updated and validated
- Consumer compatibility tests pass
- Security policy checks pass (authz/authn/rate-limit)
- PII-safe logging verification complete
- Observability dashboards updated
- Rollback plan documented and tested
- Dependency fan-out and performance budget checks passed
- Contract verification tooling checks (Pact/SCC) passed
- Caching policy review completed for changed endpoints
- Threat model controls validated for changed security surfaces

## 10.1 Downstream Client Governance

Every downstream client (`OpsClient`, `CustomerClient`, etc.) must implement identical cross-cutting policies:
- timeout
- retry
- metrics
- tracing
- authentication propagation
- circuit breaker

No service-specific custom client may bypass base policy abstractions.

## 10.2 Domain Composition Governance

Guiding principle: compose for screen/use-case value, not data hoarding.

**Good**:
Passenger + Flight → Boarding screen DTO

**Bad**:
Passenger + Flight + Pricing + Loyalty + Billing + Payments → oversized DTO

Composition rules:
- maximum downstream synchronous dependencies: see fan-out limit
- prefer parallel calls where dependencies are independent
- avoid deep nested aggregation chains
- endpoint-specific composition ownership must be explicit

## 10.3 Multi-BFF Strategy

Default strategy:
- **One deployable BFF**
- **Multiple channel modules** (mobile, desktop, future channels)

Split into multiple deployables only when justified by:
- independent release cadence
- materially different scaling profiles
- hard isolation/compliance boundaries

## 10.4 Data Ownership Reminder
- BFF must never persist business entities as source of truth.
- BFF must not own passenger, flight, or boarding system-of-record data.
- Only temporary caches/projections are allowed under approved policy.

---

## 11. Implementation Checklist

- [ ] Define route ownership and channel scope (mobile/desktop)
- [ ] Define/approve OpenAPI contracts
- [ ] Implement composition layer + DTO mappers
- [ ] Integrate auth/session/policy middleware
- [ ] Add resilience policies for all dependencies
- [ ] Add telemetry + tracing + audit-safe events
- [ ] Add contract/integration/performance tests
- [ ] Validate release gates and publish compatibility notes
- [ ] Validate API lifecycle and deprecation headers for changed endpoints
- [ ] Validate contract verification tooling gates (Pact/SCC)
- [ ] Validate fan-out, cache, and async design rules for new endpoints
- [ ] Validate threat model controls for security-impacting changes
- [ ] Confirm no cross-cutting logic re-implemented in app layer (library dependency enforcement)
- [ ] Validate library module versions are aligned and compatible before release

---

## 12. Alignment References

- Mobile foundation: [UI/REACT_NATIVE_FOUNDATION.md](UI/REACT_NATIVE_FOUNDATION.md)
- State architecture: [UI/RN_STATE_ARCHITECTURE.md](UI/RN_STATE_ARCHITECTURE.md)
- Realtime architecture: [UI/RN_REALTIME_COMMUNICATION_ARCHITECTURE.md](UI/RN_REALTIME_COMMUNICATION_ARCHITECTURE.md)
- Event-driven architecture: [UI/RN_EVENT_DRIVEN_ARCHITECTURE.md](UI/RN_EVENT_DRIVEN_ARCHITECTURE.md)
- Enterprise authentication: [UI/RN_ENTERPRISE_AUTHENTICATION.md](UI/RN_ENTERPRISE_AUTHENTICATION.md)
- Push strategy: [UI/RN_PUSH_NOTIFICATION_STRATEGY.md](UI/RN_PUSH_NOTIFICATION_STRATEGY.md)

---

## 13.1 Platform Ownership Matrix

| Area | Owner |
|---|---|
| OpenAPI contracts | Platform Team |
| API Gateway/BFF runtime | Platform Team |
| AuthN/AuthZ standards | Security Team |
| Composition implementation | Domain Teams (with Platform governance) |
| Notification orchestration integration | Platform Team |
| Solace integration | Platform Team |
| Shared service client SDK/policies | Platform Team |
| `bff-context-core` library | Platform Team |
| `bff-middleware-core` library | Platform Team |
| `bff-auth-core` library | Platform Team + Security Team |
| `bff-resilience-core` library | Platform Team |
| `bff-caching-core` library | Platform Team |
| `bff-events-core` library | Platform Team |
| `bff-notifications-core` library | Platform Team |
| `bff-audit-core` library | Platform Team + Compliance Team |

---

## 14. JDK 21 Merit Summary for This Enterprise

| Dimension | Why JDK 21 is Preferred |
|---|---|
| Skill alignment | Existing teams are Java-strong, reducing onboarding and delivery risk |
| Platform maturity | Existing enterprise security, observability, and ops controls are Java-native |
| Service ecosystem | Backend domain services already Java-based, enabling reuse of patterns/libraries |
| Operational consistency | Unified runtime/tooling for build, deployment, support, and incident response |
| Long-term support | JDK 21 is LTS and suited for enterprise lifecycle management |

**Positioning**:
- Primary BFF runtime: **JDK 21**
- Non-Java BFF stacks may be used only by explicit architecture exception and governance approval.

---

## 15. BFF ADR Seed Index

| ADR | Decision | Status |
|---|---|---|
| ADR-101 | Spring Boot selected for BFF runtime | Proposed |
| ADR-102 | JDK 21 Virtual Threads policy for aggregation workloads | Proposed |
| ADR-103 | OpenAPI-first contract governance for BFF | Proposed |
| ADR-104 | Solace as async/event backbone integration | Proposed |
| ADR-105 | Resilience4j standard for retry/timeout/circuit-breaker | Proposed |
| ADR-106 | Cross-cutting BFF capabilities extracted as non-deployable shared library modules | Proposed |
| ADR-107 | Single monorepo (libs + apps) adopted; evolve to dedicated libs repo on explicit trigger criteria | Proposed |
| ADR-108 | Platform BOM is the sole library version management mechanism; direct version specification prohibited | Proposed |
| ADR-109 | Extension point plugin contract defined for AI, Offline, Payments, Vision, IoT future capabilities | Proposed |

---

## 16. Non-Functional Requirements

| NFR | Target | Notes |
|---|---|---|
| Availability | 99.95% monthly | Multi-AZ deployment; no single point of failure |
| Horizontal scalability | Stateless BFF instances | No in-process session or request state between requests |
| Zero-downtime deployments | Required | Rolling or blue/green; `maxUnavailable: 0` |
| Graceful shutdown | Required | Connection draining; 30 sec default grace period |
| Startup time | < 10 sec | Spring Boot optimized; GraalVM Native Image considered for sub-5s targets |
| Memory (steady state) | < 512 MB heap | JVM ergonomics tuned for container; G1GC or ZGC recommended |
| CPU (average utilization) | < 30% per core | Burst to 80% permitted under peak load |
| P95 latency | < 400ms | See performance budget decomposition (Section 9.1) |
| RTO | < 5 min | Automated health-check restart; autoscaling group failover |
| RPO | N/A | BFF is stateless; no persistent state; no data loss exposure |

#### Horizontal Scalability Rules
- BFF instances must be **completely stateless** — no in-process session or request state stored between requests.
- `RequestContext` is request-scoped and propagated via Virtual Thread carrier context, not static `ThreadLocal`.
- External cache (Redis / distributed cache) is used for any cross-request state (idempotency store, response cache).
- All configuration is externally sourced (Vault + environment variables); no per-instance config drift.

#### Zero-Downtime Deployment Requirements
- Kubernetes rolling update: `maxUnavailable: 0`, `maxSurge: 1`.
- Readiness probe returns healthy only after full startup (route registration complete, downstream health verified).
- Graceful shutdown sequence: stop accepting new connections → drain in-flight requests → release resources.
- Contract-compatible (non-breaking) changes only on rolling deployments; breaking changes require blue/green with an explicit migration window.

---

## 17. Extension Point Strategy

Future capabilities are isolated as independently adoptable extension modules. Extensions follow a typed plugin contract and must not modify core BFF framework code.

#### Extension Plugin Contract

```java
public interface BffExtension {
    String extensionId();                          // stable unique identifier
    void register(BffExtensionRegistry registry);  // register capabilities and hooks
    void onStartup(ApplicationContext ctx);         // post-startup initialization
    void onShutdown();                             // graceful cleanup
}
```

Extensions are registered via Spring `@Bean` discovery and loaded by `BffExtensionRegistry` at startup.

#### Planned Extension Modules

```text
extensions/
  ai/              # AI inference, recommendation, and decision support hooks
  offline/         # Offline data sync contracts and mutation queue management
  notifications/   # Advanced notification routing and template management
  payments/        # Payment gateway integration hooks (PCI scope — isolated module)
  vision/          # Webcam, barcode, and MRZ scanning integration
  iot/             # IoT device event ingestion and command bridge
```

#### Governance Rules
- Extensions are isolated Maven modules; core BFF depends only on extension API interfaces.
- Extensions must not modify `RequestContext` fields after population by `bff-context-core`.
- PCI-scoped extensions (`payments/`) run in an isolated module with restricted dependency access and a separate security review gate.
- Each extension module ships its own test suite and publishes a compatibility matrix against the BFF platform version.
- Extension registration and lifecycle events emit telemetry for observability coverage.
