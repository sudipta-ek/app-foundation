# React Native Native Capability Platform

## 1. Objective
Define the governed platform for native hardware capability integration (camera, NFC, biometrics, passport/MRZ), including contracts, adapters, bridge implementations, and operational controls.

This document expands **Layer 12: Native Capability Platform** from [UI/REACT_NATIVE_FOUNDATION.md](UI/REACT_NATIVE_FOUNDATION.md).

---

## 2. Capability Scope
Current:
- Camera
- NFC
- Biometric authentication
- Passport scanner (MRZ)

Roadmap:
- Barcode/QR
- Passport OCR
- Bluetooth printer
- RFID
- Face recognition
- eID / mobile printing

---

## 3. Architecture Model

```text
Feature/UI
  → native-capabilities public API
  → platform adapter (ios/android/mock)
  → TurboModule spec
  → native bridge implementation
  → OS framework
```

---

## 4. High-Level Structure

```text
libs/
  native-capabilities/
    src/
      contracts/
      public-api/
        hooks/
        services/
      registry/
      adapters/
        ios/
        android/
        mock/
      telemetry/
      errors/

  ui-native/
    src/
      camera/
      nfc/
      biometric/
      passport/

apps/
  mobile-shell/
    src/native-specs/
    ios/
    android/
```

---

## 5. Folder Responsibilities
- `contracts`: TS-first interface definitions, DTOs, versioned signatures
- `public-api/hooks`: feature-facing hooks (`useCamera`, `useNfc`, `useBiometric`)
- `public-api/services`: orchestration facade, fallback routing
- `registry`: capability availability, policy checks, version compatibility
- `adapters`: platform implementations of shared contracts
- `telemetry`: latency/success/failure events
- `errors`: typed capability error taxonomy and safe mappings
- `ui-native`: native-only presentation components, no domain business logic
- `native-specs`: TurboModule JS/TS bridge specs
- `ios/android`: native implementations in Swift/Kotlin

---

## 6. Governance Model

| Area | Rule | Owner |
|---|---|---|
| Contract-first | Native work starts only after TS contract approval | Platform Team |
| Parity | iOS/Android parity required or exception approved | Platform Team |
| Versioning | Breaking signature = major version bump | Platform Team |
| Security | Permission + data handling review mandatory | Security + Platform |
| Rollout | Feature flag for new capability rollout | Platform + Domain Teams |

---

## 7. Lifecycle for New Capability
1. Define TS contract in `contracts`
2. Add mock adapter
3. Implement iOS and Android adapters
4. Add TurboModule spec + native impl
5. Add policy/registry integration
6. Add telemetry and typed errors
7. Validate through tests and release gates

---

## 8. Quality Gates
- Contract and adapter tests pass (`ios/android/mock`)
- Permission denied/unavailable paths validated
- P95 latency tracked (`capability_latency_ms`)
- No unsafe native errors leaked to UI
- Accessibility review for native UI wrappers
- Security review for PII/biometric handling

---

## 9. Testing Strategy
- Unit: contract mapping, registry logic, error mapping
- Integration: adapter + spec alignment tests
- Device tests: iOS/Android hardware behavior parity
- E2E: capability-enabled and capability-unavailable user flows

---

## 10. DoD
- Contract approved and versioned
- Platform parity complete or exception signed off
- Telemetry + audit-safe errors implemented
- Security and accessibility checks completed
- Rollout/rollback plan documented
