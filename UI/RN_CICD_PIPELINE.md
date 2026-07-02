# CI/CD Pipeline Architecture — Implementation Guide

**Aligned with**: [UI/REACT_NATIVE_FOUNDATION.md](UI/REACT_NATIVE_FOUNDATION.md) — Layer 19

---

## 1. Pipeline Overview

```text
Developer Push / MR
  ↓
[1] Static Analysis        lint + typecheck + dependency audit
  ↓
[2] Unit + Contract Tests  jest + pact consumer tests
  ↓
[3] Security Scan          SAST (SonarQube) + dependency CVE (Snyk/Trivy)
  ↓
[4] Build                  iOS (Bitrise) + Android (Bitrise) — Nx affected
  ↓
[5] Integration Tests      MSW-mocked BFF + Realm in-memory
  ↓
[6] E2E Tests              Detox on simulators + BrowserStack device matrix
  ↓
[7] Artifact Signing       Fastlane + Apple/Google credentials from vault
  ↓
[8] Deploy to Dev          Automatic on merge to main
  ↓
[9] SIT Gate               Automatic with smoke tests
  ↓
[10] UAT Gate              Manual approval + regression suite
  ↓
[11] PreProd Gate           ARB sign-off + release notes approved
  ↓
[12] Prod — Canary 5%      Staged rollout start
  ↓
[13] Prod — 25% → 100%     Auto-advance if error rate < threshold
```

---

## 2. Nx Affected Build Strategy

Only changed libraries and their downstream dependents are rebuilt and tested.

```yaml
# .gitlab-ci.yml — Nx affected pipeline
lint_affected:
  stage: static
  script:
    - npx nx affected --target=lint --base=origin/main --head=HEAD --parallel=4

test_affected:
  stage: test
  script:
    - npx nx affected --target=test --base=origin/main --head=HEAD --parallel=4 --coverage
  coverage: '/Statements\s*:\s*(\d+\.?\d*)%/'

build_affected:
  stage: build
  script:
    - npx nx affected --target=build --base=origin/main --head=HEAD --parallel=2
```

Rules:
- `--base=origin/main` ensures only code changed in this MR is tested.
- `--parallel=4` reduces pipeline time; set per runner capacity.
- Full baseline build runs on `main` branch merges and nightly schedule.
- Nx project graph (`nx graph`) must be reviewed when adding new library dependencies.

---

## 3. iOS Pipeline (Fastlane + Bitrise)

```ruby
# fastlane/Fastfile (iOS)
lane :build_release do
  # Certificates and provisioning via Fastlane Match
  match(
    type: 'appstore',
    app_identifier: 'com.airline.app',
    git_url: ENV['MATCH_GIT_URL'],              # private certs repo
    password: ENV['MATCH_ENCRYPTION_KEY'],
    readonly: true
  )

  # Increment build number from CI
  increment_build_number(
    build_number: ENV['CI_PIPELINE_IID']
  )

  # Build
  gym(
    scheme: 'AirlineApp',
    configuration: 'Release',
    export_method: 'app-store',
    output_directory: 'artifacts/ios',
    clean: true
  )

  # Upload to TestFlight for SIT/UAT
  pilot(
    api_key_path: ENV['APPLE_API_KEY_PATH'],
    skip_waiting_for_build_processing: true
  )
end

lane :deploy_prod do |options|
  # Staged rollout
  deliver(
    api_key_path: ENV['APPLE_API_KEY_PATH'],
    phased_release: true,
    submit_for_review: false,  # manual final submission
    automatic_release: false
  )
end
```

**Bitrise iOS workflow:**
```yaml
# bitrise.yml (excerpt)
workflows:
  release:
    steps:
    - activate-ssh-key: {}
    - git-clone: {}
    - node:
        inputs:
        - node_version: "20"
    - yarn:
        inputs:
        - command: install --frozen-lockfile
    - cocoapods-install:
        inputs:
        - podfile_path: ios/Podfile
    - fastlane:
        inputs:
        - lane: build_release
```

**Certificate management:**
- Fastlane `match` stores certificates in an encrypted Git repository.
- `MATCH_ENCRYPTION_KEY` stored in Bitrise Secrets (never in source).
- Certificates rotated annually via `fastlane match nuke` + `fastlane match` re-creation.

---

## 4. Android Pipeline

```ruby
# fastlane/Fastfile (Android)
lane :build_release_android do
  # Increment version
  android_set_version_code(version_code: ENV['CI_PIPELINE_IID'])

  # Build signed release AAB
  gradle(
    task: 'bundle',
    build_type: 'Release',
    project_dir: 'android',
    properties: {
      'android.injected.signing.store.file'     => ENV['KEYSTORE_PATH'],
      'android.injected.signing.store.password' => ENV['KEYSTORE_PASSWORD'],
      'android.injected.signing.key.alias'      => ENV['KEY_ALIAS'],
      'android.injected.signing.key.password'   => ENV['KEY_PASSWORD'],
    }
  )

  # Upload to Play Console internal track
  supply(
    track: 'internal',
    aab: 'android/app/build/outputs/bundle/release/app-release.aab',
    json_key_data: ENV['GOOGLE_PLAY_JSON_KEY'],
    skip_upload_apk: true
  )
end
```

**Android signing:**
- Keystore stored in CI secrets — never in repository.
- `KEYSTORE_PATH`: injected from vault-backed CI artifact store.
- Play App Signing used for production — upload key separate from signing key.

---

## 5. Quality Gates

All gates are mandatory. A single failure blocks the pipeline.

| Gate | Tool | Threshold | Block On |
|---|---|---|---|
| Lint | ESLint + Prettier | Zero errors | Any error |
| TypeScript | `tsc --noEmit` | Zero errors | Any error |
| Unit test coverage | Jest + Istanbul | ≥ 80% statements | Coverage drop below threshold |
| Unit tests | Jest | All pass | Any failure |
| Contract tests | Pact | All consumer contracts verified | Any verification failure |
| SAST | SonarQube Quality Gate | No blocker/critical issues | Blocker or critical code smell |
| Dependency CVE | Snyk / Trivy | No HIGH/CRITICAL CVEs | HIGH or CRITICAL CVE found |
| License compliance | `license-checker` | No GPL / AGPL in production | Prohibited license found |
| Nx boundary | `nx lint --all` | No module boundary violations | Any boundary violation |
| E2E (simulator) | Detox | All scenarios pass | Any scenario failure |

```yaml
# Quality gate enforcement in GitLab CI
quality_gate:
  stage: gate
  script:
    - npx nx affected --target=test --coverage --passWithNoTests
    - npx jest --coverageThreshold='{"global":{"statements":80}}'
    - npx nx run-many --target=lint --all
    - snyk test --severity-threshold=high
    - npx license-checker --onlyAllow 'MIT;Apache-2.0;BSD-2-Clause;BSD-3-Clause;ISC'
  allow_failure: false
```

---

## 6. Environment Promotion Gates

| Transition | Gate | Approver |
|---|---|---|
| main → Dev | Automatic (all quality gates green) | Pipeline |
| Dev → SIT | Automatic + smoke test suite pass | Pipeline |
| SIT → UAT | Manual approval | QA Lead |
| UAT → PreProd | Manual approval + release notes reviewed | Product Owner |
| PreProd → Prod (5%) | ARB sign-off + Platform Team approval | Release Manager |
| Prod 5% → 25% | Automatic if error rate < 0.5% for 24h | Pipeline (canary monitor) |
| Prod 25% → 100% | Automatic if error rate < 0.5% for 48h | Pipeline (canary monitor) |

**Canary abort trigger:**
```yaml
# Production canary monitoring job
canary_watch:
  stage: canary
  script:
    - ./scripts/check-canary-metrics.sh
      --error-rate-threshold=0.5
      --latency-p95-threshold=500ms
      --crash-free-threshold=99.8
      --halt-on-breach
```

---

## 7. Secret Injection in CI

```yaml
# GitLab CI — secret injection pattern
variables:
  # Masked CI/CD variables — never appear in logs
  APPLE_API_KEY_ID:        $APPLE_API_KEY_ID
  MATCH_ENCRYPTION_KEY:    $MATCH_ENCRYPTION_KEY
  SNYK_TOKEN:              $SNYK_TOKEN
  GOOGLE_PLAY_JSON_KEY:    $GOOGLE_PLAY_JSON_KEY
  BFF_CERT_HASH_PRIMARY:   $BFF_CERT_HASH_PRIMARY

# Vault-backed dynamic secrets (HashiCorp Vault + GitLab OIDC)
.vault_secrets: &vault_secrets
  id_tokens:
    VAULT_ID_TOKEN:
      aud: https://vault.airline.com
  secrets:
    KEYSTORE_PASSWORD:
      vault: secret/cicd/android/keystore@production
      file: false
```

Rules:
- All secrets are masked in GitLab — values never printed in job logs.
- Vault secrets use short-lived OIDC tokens — no static vault tokens in CI.
- Secret variables are scoped to protected branches and environments only.
- Secrets are audited on access via vault audit log.

---

## 8. Contract Test Gate

Consumer-driven contract tests (Pact) run in CI and verify BFF compatibility:

```yaml
contract_tests:
  stage: test
  script:
    - npx jest --testPathPattern='contract' --verbose
    # Publish Pacts to Pact Broker
    - npx pact-broker publish pacts/
        --broker-base-url=$PACT_BROKER_URL
        --consumer-app-version=$CI_COMMIT_SHA
        --tag=$CI_COMMIT_BRANCH
    # Verify provider compatibility (against current BFF)
    - npx pact-broker can-i-deploy
        --pacticipant=AirlineMobile
        --version=$CI_COMMIT_SHA
        --to-environment=production
        --broker-base-url=$PACT_BROKER_URL
```

---

## 9. Folder Structure (CI Configuration)

```text
.gitlab-ci.yml                     # Main pipeline definition
fastlane/
  Fastfile                         # Lane definitions (iOS + Android)
  Appfile                          # App identifiers
  Matchfile                        # Certificate configuration
bitrise.yml                        # Bitrise workflow definitions
scripts/
  check-canary-metrics.sh          # Canary error rate monitor
  check-coverage.sh                # Coverage threshold enforcement
  rotate-match-certs.sh            # Certificate rotation helper
docs/
  ci-cd-runbook.md                 # Incident response + manual steps
```

---

## 10. Rollback Procedures

| Scenario | Rollback Action | Time Target |
|---|---|---|
| App Store defect found | Halt canary, promote previous version in App Store Connect | < 30 min |
| Feature flag regression | Disable feature flag in Firebase Remote Config | < 1 min |
| Config regression | Revert config version in Azure App Config | < 5 min |
| Hotfix required | Emergency lane: lint + test + build + expedited review + deploy | < 4 hours |

```ruby
# Fastlane emergency rollback lane
lane :rollback_prod do |options|
  deliver(
    api_key_path: ENV['APPLE_API_KEY_PATH'],
    version: options[:rollback_to_version],
    phased_release: false,
    automatic_release: true
  )
end
```

---

## 11. Governance Gates (DoD)

A CI/CD pipeline change is compliant when:
- [ ] All quality gates green (lint, types, coverage ≥ 80%, tests, CVE scan, boundary)
- [ ] Contract tests pass and Pact broker updated
- [ ] No secrets committed to source or visible in pipeline logs
- [ ] Certificate rotation runbook updated if certificates changed
- [ ] Canary rollout configured with automatic halt on error rate breach
- [ ] Rollback procedure documented and tested in pre-prod
- [ ] Environment promotion approvals defined and enforced
- [ ] Nx affected builds used — full build only on main branch
