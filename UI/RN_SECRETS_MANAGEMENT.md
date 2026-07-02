# Enterprise Secrets Management — Implementation Guide

**Aligned with**: [UI/REACT_NATIVE_FOUNDATION.md](UI/REACT_NATIVE_FOUNDATION.md) — Layer 18

---

## 1. Vault Hierarchy

```text
Secrets precedence (highest → lowest):

Azure Key Vault (primary — Azure-hosted environments)
  ↑ synced from
HashiCorp Vault (AWS cloud — synced to on-prem OpenShift Vault cluster)
  ↓
CI/CD Pipeline (injects build-time secrets as env vars)
  ↓
BFF Runtime (fetches runtime secrets from vault on startup)
  ↓
Mobile Device (receives ONLY what BFF exposes — never vault-direct)
```

**Environment isolation:**
- Separate vault instance per environment: `Dev`, `SIT`, `UAT`, `Prod`.
- Cross-environment secret access is prohibited.
- Prod vault requires MFA + just-in-time access approval.

---

## 2. Build-Time Secret Injection (CI/CD)

Secrets required at build time (code signing, push credentials) are injected by the CI pipeline — never committed to source.

```yaml
# .gitlab-ci.yml (excerpt)
build_ios:
  stage: build
  variables:
    MATCH_PASSWORD: $MATCH_ENCRYPTION_KEY         # from GitLab CI/CD variables (masked)
    APPLE_API_KEY_ID: $APPLE_API_KEY_ID
    FASTLANE_APPLE_APPLICATION_SPECIFIC_PASSWORD: $APP_SPECIFIC_PASSWORD
  script:
    - bundle exec fastlane ios build_release
  environment:
    name: production
```

Rules:
- All CI secrets stored in GitLab CI/CD variables (masked + protected).
- No secret value ever appears in `.gitlab-ci.yml`, `Fastfile`, or any source file.
- Build logs are scrubbed to prevent accidental secret exposure.
- Secret rotation in CI variables does not require source code changes.

---

## 3. Runtime Secret Pattern (BFF-Only)

**The device never accesses secrets vaults directly.** All runtime secrets are fetched and managed by the BFF.

```text
Mobile App
  → POST /v1/session/bootstrap (with auth token)
  ← BFF returns session context (short-lived tokens, endpoint config)

Mobile App
  → BFF /v1/config/runtime (with session token)
  ← BFF returns: { apiBaseUrl, featureFlagKey, cdnUrl }
    (no credentials, no vault secrets)

Device-specific secrets (FCM registration token, biometric keys):
  → Stored in iOS Keychain / Android Keystore — not in BFF
  → Sent to BFF only when required for push registration
```

The mobile app must **never**:
- Call Azure Key Vault, HashiCorp Vault, or AWS Secrets Manager directly.
- Embed API keys, credentials, or signing keys in the app bundle.
- Store credentials in `AsyncStorage`, SQLite plain text, or `MMKV` without encryption.

---

## 4. iOS Keychain Implementation

```typescript
// libs/secrets/src/ios/keychain-storage.ts
import * as Keychain from 'react-native-keychain';

const ACCESSIBILITY = Keychain.ACCESSIBLE.WHEN_UNLOCKED_THIS_DEVICE_ONLY;

export async function storeSecureValue(
  key: string,
  value: string,
  requiresBiometric = false
): Promise<void> {
  const options: Keychain.Options = {
    accessible: ACCESSIBILITY,
    accessControl: requiresBiometric
      ? Keychain.ACCESS_CONTROL.BIOMETRY_CURRENT_SET
      : Keychain.ACCESS_CONTROL.USER_PRESENCE,
    securityLevel: Keychain.SECURITY_LEVEL.SECURE_HARDWARE, // Secure Enclave
    service: `com.airline.app.${key}`,
  };
  await Keychain.setGenericPassword(key, value, options);
}

export async function retrieveSecureValue(
  key: string,
  requiresBiometric = false
): Promise<string | null> {
  const options: Keychain.Options = {
    service: `com.airline.app.${key}`,
    authenticationPrompt: requiresBiometric
      ? { title: 'Authenticate to continue', cancel: 'Cancel' }
      : undefined,
  };
  const result = await Keychain.getGenericPassword(options);
  return result ? result.password : null;
}

export async function deleteSecureValue(key: string): Promise<void> {
  await Keychain.resetGenericPassword({ service: `com.airline.app.${key}` });
}
```

**iOS Keychain item attributes used:**

| Attribute | Value | Reason |
|---|---|---|
| `kSecAttrAccessible` | `kSecAttrAccessibleWhenUnlockedThisDeviceOnly` | Not backed up to iCloud; device-bound |
| `kSecAttrAccessControl` | `biometryCurrentSet` (biometric keys) | Requires Face ID/Touch ID for sensitive keys |
| `kSecAttrSynchronizable` | `false` | Keys never synchronised to other devices |
| Storage backend | Secure Enclave (A-series chip) | Hardware-isolated key storage |

**Keychain access groups:**
```
com.airline.app.shared   → keys shared between app and extensions (e.g. App Clip)
com.airline.app.private  → keys accessible to main app only
```

---

## 5. Android Keystore Implementation

```typescript
// libs/secrets/src/android/keystore-storage.ts
import * as Keychain from 'react-native-keychain';
import EncryptedStorage from 'react-native-encrypted-storage';

// For auth tokens and session credentials — Android Keystore backed
export async function storeSessionToken(token: string): Promise<void> {
  await Keychain.setGenericPassword('session', token, {
    storage: Keychain.STORAGE_TYPE.RSA,   // Android Keystore RSA-wrapped AES
    accessible: Keychain.ACCESSIBLE.WHEN_UNLOCKED,
    securityLevel: Keychain.SECURITY_LEVEL.SECURE_HARDWARE,
    service: 'com.airline.app.session',
  });
}

// For larger encrypted data (offline config, certificates)
export async function storeEncryptedData(key: string, data: string): Promise<void> {
  await EncryptedStorage.setItem(key, data);
  // EncryptedStorage wraps Android EncryptedSharedPreferences
  // which uses AES-256-GCM with keys in Android Keystore
}
```

**Android Keystore key properties:**

| Property | Value | Reason |
|---|---|---|
| Key algorithm | AES-256-GCM | Authenticated encryption |
| Key storage | Android Keystore System | Hardware-backed on TEE/StrongBox |
| User auth required | `true` for biometric keys | Requires fingerprint/face unlock |
| Key validity duration | Session-scoped | Keys invalidated on biometric changes |

**StrongBox (Android 9+ with dedicated security chip):**
```typescript
const canUseStrongBox = await Keychain.getSupportedBiometryType();
const securityLevel = canUseStrongBox
  ? Keychain.SECURITY_LEVEL.SECURE_HARDWARE
  : Keychain.SECURITY_LEVEL.SECURE_SOFTWARE;
```

---

## 6. React Native Abstraction Layer

Feature code must never call platform-specific Keychain/Keystore APIs directly. Use the platform-agnostic abstraction:

```typescript
// libs/secrets/src/secure-storage.ts
import { Platform } from 'react-native';
import { storeSecureValue as iosStore } from './ios/keychain-storage';
import { storeSessionToken as androidStore } from './android/keystore-storage';

export interface SecureStorage {
  store(key: SecretKey, value: string): Promise<void>;
  retrieve(key: SecretKey): Promise<string | null>;
  delete(key: SecretKey): Promise<void>;
}

export type SecretKey =
  | 'session.accessToken'
  | 'session.refreshToken'
  | 'biometric.enrollmentKey'
  | 'realm.encryptionKey'
  | 'certificate.deviceCert';

class PlatformSecureStorage implements SecureStorage {
  async store(key: SecretKey, value: string): Promise<void> {
    const requiresBiometric = key === 'biometric.enrollmentKey';
    if (Platform.OS === 'ios') {
      return iosStore(key, value, requiresBiometric);
    }
    return androidStore(value);  // simplified — expand per key type
  }

  async retrieve(key: SecretKey): Promise<string | null> {
    // Platform-specific retrieval...
  }

  async delete(key: SecretKey): Promise<void> {
    await deleteSecureValue(key);
  }
}

export const secureStorage = new PlatformSecureStorage();
```

Rules:
- `SecretKey` is a typed union — no arbitrary string keys.
- New secret types require Platform Team review before adding to `SecretKey`.
- `secureStorage` is the **only** secure storage entry point for feature code.

---

## 7. Biometric Key Protection

Certain keys require biometric authentication on every read (not just device unlock):

```typescript
// libs/secrets/src/biometric-key.service.ts
export async function storeBiometricProtectedKey(
  key: string,
  value: string
): Promise<void> {
  await secureStorage.store('biometric.enrollmentKey', value);
  // Stored with ACCESS_CONTROL.BIOMETRY_CURRENT_SET
  // If user re-enrolls biometrics, this key is automatically invalidated
}

export async function retrieveWithBiometric(
  key: string,
  promptMessage: string
): Promise<string | null> {
  // Triggers Face ID / Touch ID / Fingerprint prompt
  return Keychain.getGenericPassword({
    service: `com.airline.app.${key}`,
    authenticationPrompt: {
      title: promptMessage,
      cancel: 'Cancel',
      fallbackLabel: 'Use passcode',
    },
  }).then(r => r ? r.password : null);
}
```

**Biometric key invalidation rules:**
- Key is automatically invalidated when the user adds/removes biometric enrollments.
- App must detect invalidation and re-prompt the user to re-establish the key.
- Biometric key invalidation is treated as a security event and logged to audit.

---

## 8. Certificate Pinning Storage

```typescript
// libs/secrets/src/certificate-pinning.service.ts
// Certificates delivered via MDM profile or injected at build time
// Stored as public key hashes — not full certificates (avoids rotation coupling)

export const PINNED_KEY_HASHES = {
  bffPrimary: process.env.BFF_CERT_HASH_PRIMARY!,
  bffBackup:  process.env.BFF_CERT_HASH_BACKUP!,
};

// Used in HTTP client configuration (never store private keys on device)
export function getPinningConfig(): PinningConfig {
  return {
    'api.airline.com': [PINNED_KEY_HASHES.bffPrimary, PINNED_KEY_HASHES.bffBackup],
  };
}
```

Rules:
- Always configure **two pins** (primary + backup) to enable certificate rotation without app update.
- Pin public key hashes (SPKI), not full certificate fingerprints.
- Certificate rotation plan must exist before deploying pinning.
- Backup pin pre-deployed minimum 4 weeks before primary certificate rotation.

---

## 9. MDM-Distributed Certificates

For enterprise deployments, device certificates are distributed via MDM (Intune/Jamf):

```text
MDM Server (Intune / Jamf)
  → Pushes device certificate profile (PKCS#12 or SCEP)
  → Installed in iOS Keychain / Android Keystore under system profile
  → App accesses via MDM-managed keychain access group
  → BFF validates device certificate on session establishment
```

```typescript
// libs/secrets/src/mdm-certificate.service.ts
export async function getMdmDeviceCertificate(): Promise<string | null> {
  // Access MDM-pushed certificate from managed keychain group
  return Keychain.getGenericPassword({
    service: 'com.airline.mdm.devicecert',
    accessGroup: 'com.airline.app.mdm',  // MDM-managed access group
  }).then(r => r ? r.password : null);
}
```

---

## 10. Secret Rotation

| Secret Type | Rotation Frequency | Method | Zero-Downtime |
|---|---|---|---|
| BFF API keys | Every 90 days | Automated vault rotation | Blue/green with overlap period |
| TLS certificates | 60-day cycle (90-day cert) | ACME / cert-manager | Backup pin pre-deployed |
| FCM/APNS credentials | On provider requirement | Manual + CI re-inject | Build-time re-injection |
| Device certificates | Annual or on device replacement | MDM re-push | MDM overlap deployment |
| Realm encryption key | On device replacement | Re-provisioned via session | Fresh Realm DB on new device |
| JWT signing keys | Every 30 days | BFF-side vault rotation | Key overlap (accept current + previous) |

Rotation process:
1. New secret provisioned in vault.
2. BFF or CI updated to consume new secret (overlap period: both old and new accepted).
3. Old secret marked deprecated.
4. After 48-hour overlap window, old secret revoked.
5. Audit event emitted for secret rotation.

---

## 11. Folder Structure

```text
libs/
  secrets/
    src/
      secure-storage.ts              # Platform-agnostic facade (SecureStorage interface)
      secret-key.types.ts            # SecretKey typed union
      ios/
        keychain-storage.ts          # iOS Keychain implementation
        keychain-access-groups.ts    # Access group configuration
      android/
        keystore-storage.ts          # Android Keystore implementation
        encrypted-storage.ts         # EncryptedSharedPreferences wrapper
      biometric-key.service.ts       # Biometric-protected key operations
      certificate-pinning.service.ts # SPKI pin configuration
      mdm-certificate.service.ts     # MDM-distributed certificate access
      utils/
        key-invalidation.handler.ts  # Biometric change detection
      __tests__/
        secure-storage.spec.ts
        biometric-key.service.spec.ts
```

---

## 12. Governance Gates (DoD)

A secrets implementation is compliant when:
- [ ] No secrets, credentials, or keys committed to source code or repository
- [ ] All build-time secrets injected via CI/CD variables (masked + protected)
- [ ] Device stores only what is strictly required — no raw credentials on device
- [ ] iOS: `kSecAttrAccessibleWhenUnlockedThisDeviceOnly` on all Keychain items
- [ ] Android: Android Keystore + `SECURITY_LEVEL.SECURE_HARDWARE` where available
- [ ] Backup pin deployed before certificate rotation
- [ ] Secret rotation plan documented with zero-downtime overlap window
- [ ] MDM certificate distribution verified for all enrolled device types
- [ ] Biometric key invalidation on enrollment change handled and audited
- [ ] `SecretKey` typed — no raw string keys in `secureStorage` calls
- [ ] Security Team review for any new secret type or rotation policy change
