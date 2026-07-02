# Comprehensive Testing Pyramid — Implementation Guide

**Aligned with**: [UI/REACT_NATIVE_FOUNDATION.md](UI/REACT_NATIVE_FOUNDATION.md) — Layer 8

---

## 1. Pyramid Overview & Coverage Targets

```text
                        ┌─────────────────┐
                        │   E2E (Detox)   │  5%  — real device flows
                        ├─────────────────┤
                        │  Integration    │  15% — MSW + Realm in-memory
                        │  (MSW)          │
                        ├─────────────────┤
                        │  Component      │  30% — RNTL + interaction
                        │  (RNTL)         │
                        ├─────────────────┤
                        │  Unit (Jest)    │  50% — logic, hooks, slices
                        └─────────────────┘

Overall target: 80%+ statement coverage
Contract tests: 100% pass rate (CI gate — non-negotiable)
```

**What each level proves:**

| Level | Validates | Scope | Speed |
|---|---|---|---|
| Unit | Pure logic, hooks, reducers, mappers, utilities | Single function/class | < 5ms per test |
| Component | Rendering, user interaction, accessibility | Single component tree | < 50ms per test |
| Integration | Feature slice: screen + hooks + MSW + state | Feature workflow | < 500ms per test |
| Contract | API contract compatibility with BFF | Consumer/producer contract | < 2s per test |
| E2E | Full user journey on real device/simulator | Full app stack | 1–30s per test |

---

## 2. Unit Tests (Jest)

**What to unit test:**
- Redux slices (actions, reducers, selectors)
- Custom hooks (via `renderHook` from RNTL)
- Mapper / transformer functions
- Validation rules
- Utility functions
- Audit/analytics emission logic

```typescript
// libs/features/boarding/src/state/__tests__/boarding.slice.spec.ts
import { boardingReducer, scanPassenger, setScanResult } from '../boarding.slice';
import type { BoardingState } from '../boarding.types';

const initialState: BoardingState = {
  scannedPassengers: [],
  scanStatus: 'idle',
};

describe('boarding.slice', () => {
  it('should set scan status to scanning on scanPassenger', () => {
    const state = boardingReducer(initialState, scanPassenger({ passengerId: 'P001' }));
    expect(state.scanStatus).toBe('scanning');
  });

  it('should append passenger on successful scan result', () => {
    const scanning = { ...initialState, scanStatus: 'scanning' as const };
    const state = boardingReducer(scanning, setScanResult({ passengerId: 'P001', result: 'success' }));
    expect(state.scannedPassengers).toHaveLength(1);
    expect(state.scanStatus).toBe('idle');
  });
});
```

```typescript
// libs/features/boarding/src/hooks/__tests__/use-boarding-list.hook.spec.ts
import { renderHook, waitFor } from '@testing-library/react-native';
import { useBoardingList } from '../use-boarding-list.hook';
import { createTestWrapper } from '@airline/test-utils';
import { server } from '@airline/mock-server';
import { boardingHandlers } from '../../__mocks__/boarding.handlers';

describe('useBoardingList', () => {
  beforeEach(() => server.use(...boardingHandlers));

  it('returns flight passengers on success', async () => {
    const { result } = renderHook(
      () => useBoardingList({ flightId: 'EK001' }),
      { wrapper: createTestWrapper() }
    );

    await waitFor(() => expect(result.current.isSuccess).toBe(true));
    expect(result.current.data).toHaveLength(3);
  });

  it('returns error state when BFF returns 500', async () => {
    server.use(boardingHandlers.error500);
    const { result } = renderHook(
      () => useBoardingList({ flightId: 'EK001' }),
      { wrapper: createTestWrapper() }
    );

    await waitFor(() => expect(result.current.isError).toBe(true));
  });
});
```

**Jest configuration:**
```javascript
// jest.config.js
module.exports = {
  preset: 'react-native',
  setupFilesAfterFramework: ['./jest.setup.ts'],
  moduleNameMapper: {
    '@airline/(.*)': '<rootDir>/libs/$1/src/index.ts',
  },
  collectCoverageFrom: [
    'libs/**/*.{ts,tsx}',
    '!libs/**/*.spec.{ts,tsx}',
    '!libs/**/*.types.ts',
    '!libs/**/index.ts',
  ],
  coverageThreshold: {
    global: { statements: 80, branches: 75, functions: 80, lines: 80 },
  },
};
```

---

## 3. Component Tests (React Native Testing Library)

**What to component test:**
- Rendering with different props/state
- User interactions (press, input, scroll)
- Loading / error / empty states
- Accessibility attributes
- Conditional rendering based on permissions or flags

```typescript
// libs/features/boarding/src/components/__tests__/boarding-card.component.spec.tsx
import { render, screen, fireEvent } from '@testing-library/react-native';
import { BoardingCard } from '../boarding-card.component';

const mockPassenger = {
  id: 'P001', name: 'Test Passenger', seat: '12A', boardingGroup: 'A', status: 'not-boarded',
};

describe('BoardingCard', () => {
  it('renders passenger details', () => {
    render(<BoardingCard passenger={mockPassenger} onBoard={jest.fn()} />);
    expect(screen.getByText('12A')).toBeTruthy();
    expect(screen.getByText('Group A')).toBeTruthy();
  });

  it('calls onBoard when board button pressed', () => {
    const onBoard = jest.fn();
    render(<BoardingCard passenger={mockPassenger} onBoard={onBoard} />);
    fireEvent.press(screen.getByRole('button', { name: /board/i }));
    expect(onBoard).toHaveBeenCalledWith('P001');
  });

  it('shows boarded badge when status is boarded', () => {
    render(<BoardingCard passenger={{ ...mockPassenger, status: 'boarded' }} onBoard={jest.fn()} />);
    expect(screen.getByAccessibilityLabel('Boarded')).toBeTruthy();
  });

  it('disables board button when status is boarded', () => {
    render(<BoardingCard passenger={{ ...mockPassenger, status: 'boarded' }} onBoard={jest.fn()} />);
    expect(screen.getByRole('button', { name: /board/i })).toBeDisabled();
  });
});
```

**Accessibility testing in components:**
```typescript
it('has correct accessibility role and label', () => {
  const { getByRole } = render(<BoardingCard passenger={mockPassenger} onBoard={jest.fn()} />);
  expect(getByRole('button', { name: /board passenger/i })).toBeTruthy();
});

it('announces boarded status to screen reader', () => {
  const { getByA11yState } = render(
    <BoardingCard passenger={{ ...mockPassenger, status: 'boarded' }} onBoard={jest.fn()} />
  );
  expect(getByA11yState({ checked: true })).toBeTruthy();
});
```

---

## 4. Integration Tests (MSW)

Integration tests cover a complete feature slice: screen renders, hooks fire, MSW intercepts BFF calls, and state is correctly updated.

```typescript
// libs/features/boarding/src/__tests__/boarding-list-screen.integration.spec.tsx
import { render, screen, waitFor, fireEvent } from '@testing-library/react-native';
import { server } from '@airline/mock-server';
import { boardingHandlers } from '../__mocks__/boarding.handlers';
import { BoardingListScreen } from '../screens/boarding-list.screen';
import { createIntegrationWrapper } from '@airline/test-utils';

describe('BoardingListScreen integration', () => {
  beforeAll(() => server.listen());
  afterEach(() => server.resetHandlers());
  afterAll(() => server.close());

  it('loads and displays passenger list from BFF', async () => {
    server.use(...boardingHandlers.success);

    render(<BoardingListScreen route={{ params: { flightId: 'EK001' } }} />, {
      wrapper: createIntegrationWrapper(),
    });

    expect(screen.getByTestId('loading-skeleton')).toBeTruthy();
    await waitFor(() => expect(screen.getByText('Passenger 1')).toBeTruthy());
    expect(screen.getAllByRole('listitem')).toHaveLength(3);
  });

  it('shows retry UI on BFF error', async () => {
    server.use(boardingHandlers.error500);

    render(<BoardingListScreen route={{ params: { flightId: 'EK001' } }} />, {
      wrapper: createIntegrationWrapper(),
    });

    await waitFor(() => expect(screen.getByText(/failed to load/i)).toBeTruthy());
    expect(screen.getByRole('button', { name: /retry/i })).toBeTruthy();
  });
});
```

**MSW handler pattern:**
```typescript
// libs/features/boarding/src/__mocks__/boarding.handlers.ts
import { http, HttpResponse } from 'msw';

export const boardingHandlers = {
  success: [
    http.get('*/v1/boarding/EK001/passengers', () =>
      HttpResponse.json({
        data: [
          { id: 'P001', name: 'Passenger 1', seat: '1A', status: 'not-boarded' },
          { id: 'P002', name: 'Passenger 2', seat: '1B', status: 'not-boarded' },
          { id: 'P003', name: 'Passenger 3', seat: '2A', status: 'boarded' },
        ],
        meta: { total: 3, correlationId: 'test-corr-id' },
      })
    ),
  ],
  error500: http.get('*/v1/boarding/:flightId/passengers', () =>
    HttpResponse.json({ error: 'Internal Server Error' }, { status: 500 })
  ),
};
```

---

## 5. Contract Tests (Pact)

Consumer-driven contract tests verify that the mobile app's expectations of the BFF API are met.

```typescript
// libs/features/boarding/src/contract/__tests__/boarding-api.contract.spec.ts
import { PactV3, MatchersV3 } from '@pact-foundation/pact';

const { like, string, integer, eachLike } = MatchersV3;

const provider = new PactV3({
  consumer: 'AirlineMobile',
  provider: 'BFFGateway',
  logLevel: 'warn',
  dir: 'pacts/',
});

describe('Boarding API Contract', () => {
  it('fetches passenger list for a flight', async () => {
    await provider
      .addInteraction({
        states: [{ description: 'Flight EK001 has 3 passengers' }],
        uponReceiving: 'a request for boarding passengers',
        withRequest: {
          method: 'GET',
          path: '/v1/boarding/EK001/passengers',
          headers: { Authorization: like('Bearer token') },
        },
        willRespondWith: {
          status: 200,
          body: {
            data: eachLike({
              id: string('P001'),
              seat: string('1A'),
              status: string('not-boarded'),
              boardingGroup: string('A'),
            }),
            meta: { total: integer(3), correlationId: string() },
          },
        },
      })
      .executeTest(async (mockServer) => {
        const client = new BoardingApiClient({ baseUrl: mockServer.url });
        const result = await client.getPassengers('EK001');
        expect(result.data).toHaveLength(1);  // Pact eachLike returns 1 example
        expect(result.data[0].id).toBeDefined();
      });
  });
});
```

Rules:
- Pact files published to Pact Broker on every CI run.
- `can-i-deploy` gate must pass before promoting to production.
- Consumer contracts must cover: success, error 400, error 500, and auth 401 responses.

---

## 6. E2E Tests (Detox)

E2E tests run against the full app on iOS simulator / Android emulator. Focus on critical user journeys only.

```typescript
// e2e/boarding/boarding-scan.e2e.ts
describe('Boarding Scan Flow', () => {
  beforeAll(async () => {
    await device.launchApp({ newInstance: true });
    await loginAsTestAgent(); // helper: fills credentials, submits
  });

  it('should scan a passenger and update boarding status', async () => {
    await element(by.id('boarding-list-screen')).tap();
    await waitFor(element(by.id('passenger-list'))).toBeVisible().withTimeout(5000);

    // Tap scan button
    await element(by.id('scan-passenger-button')).tap();
    await waitFor(element(by.id('camera-preview'))).toBeVisible().withTimeout(3000);

    // Simulate barcode scan (Detox mock)
    await device.sendUserNotification({
      trigger: { type: 'push' },
      payload: { scanResult: { rawValue: 'P001', format: 'barcode' } },
    });

    // Verify status update
    await waitFor(element(by.id('passenger-P001-boarded-badge')))
      .toBeVisible()
      .withTimeout(3000);
  });

  it('should show error banner on duplicate scan', async () => {
    await element(by.id('scan-passenger-button')).tap();
    await simulateScan('P001'); // already boarded

    await waitFor(element(by.text('Passenger already boarded')))
      .toBeVisible()
      .withTimeout(2000);
  });
});
```

**Detox configuration:**
```javascript
// .detoxrc.js
module.exports = {
  testRunner: { $0: 'jest', args: { config: 'e2e/jest.config.js' } },
  apps: {
    'ios.release': {
      type: 'ios.app',
      binaryPath: 'ios/build/Release/AirlineApp.app',
      build: 'xcodebuild -scheme AirlineApp -configuration Release -sdk iphonesimulator ...',
    },
    'android.release': {
      type: 'android.apk',
      binaryPath: 'android/app/build/outputs/apk/release/app-release.apk',
    },
  },
  devices: {
    'ios.simulator': {
      type: 'ios.simulator',
      device: { type: 'iPhone 16 Pro' },
    },
    'android.emulator': {
      type: 'android.emulator',
      device: { avdName: 'Pixel_8_API_34' },
    },
  },
  configurations: {
    'ios.sim.release': { device: 'ios.simulator', app: 'ios.release' },
    'android.emu.release': { device: 'android.emulator', app: 'android.release' },
  },
};
```

**E2E test scope — what to cover:**
- [ ] Login → session restore
- [ ] Critical boarding scan flow (success + duplicate)
- [ ] Offline mode banner + read-only access
- [ ] Push notification tap → deep link navigation
- [ ] Permission denial UX (scan without permission)
- [ ] Session timeout and re-login

---

## 7. Test Utilities and Shared Mocks

```typescript
// libs/test-utils/src/wrappers/create-test-wrapper.tsx
export function createTestWrapper(
  overrides?: Partial<TestWrapperOptions>
): React.ComponentType {
  return ({ children }) => (
    <Provider store={createTestStore(overrides?.storeState)}>
      <QueryClientProvider client={createTestQueryClient()}>
        <ThemeProvider>
          <NavigationContainer>
            {children}
          </NavigationContainer>
        </ThemeProvider>
      </QueryClientProvider>
    </Provider>
  );
}

export function createIntegrationWrapper(): React.ComponentType {
  return createTestWrapper({
    storeState: {
      auth: { isAuthenticated: true, userId: 'test-user', permissions: ['boarding:view', 'boarding:scan'] },
    },
  });
}
```

---

## 8. Folder Structure

```text
libs/
  features/
    boarding/
      src/
        __tests__/
          boarding-list-screen.integration.spec.tsx
        components/
          __tests__/
            boarding-card.component.spec.tsx
        hooks/
          __tests__/
            use-boarding-list.hook.spec.ts
        state/
          __tests__/
            boarding.slice.spec.ts
        contract/
          __tests__/
            boarding-api.contract.spec.ts
        __mocks__/
          boarding.handlers.ts             # MSW handlers for this domain

  test-utils/
    src/
      wrappers/
        create-test-wrapper.tsx
        create-integration-wrapper.tsx
      factories/
        passenger.factory.ts              # test data factories
        flight.factory.ts
      store/
        create-test-store.ts
      mock-server/
        server.ts                         # shared MSW server instance

e2e/
  boarding/
    boarding-scan.e2e.ts
  checkin/
    checkin-flow.e2e.ts
  helpers/
    login.helper.ts
    simulate-scan.helper.ts
  jest.config.js
  .detoxrc.js
```

---

## 9. Coverage Enforcement

```typescript
// jest.config.js — coverage thresholds
coverageThreshold: {
  global: {
    statements: 80,
    branches:   75,
    functions:  80,
    lines:      80,
  },
  // Per-library minimum — stricter for shared platform libs
  './libs/auth/**': { statements: 90 },
  './libs/audit/**': { statements: 90 },
  './libs/navigation-contracts/**': { statements: 85 },
},
```

**Coverage exclusions (explicitly excluded from threshold):**
- `*.types.ts` — type definitions, no runtime code
- `*/index.ts` — barrel exports
- `*.mock.ts` / `__mocks__/**` — test mocks
- `*/stories/**` — Storybook stories
- Generated code in `src/generated/`

---

## 10. Governance Gates (DoD)

A feature is test-compliant when:
- [ ] Unit tests for all slices, hooks, mappers, and validators
- [ ] Component tests cover: success, loading, error, empty, and disabled states
- [ ] Integration test covers the primary user-facing flow end-to-end
- [ ] Pact contract test written for every new BFF endpoint consumed
- [ ] E2E scenario added for new critical user journey
- [ ] Coverage ≥ 80% — CI gate blocks merge on coverage drop
- [ ] Accessibility assertions in component tests for all interactive elements
- [ ] Offline/error state covered in at least integration or E2E level
- [ ] MSW handlers added to shared mock library (not duplicated per test)
- [ ] All tests deterministic — no random data, no time-dependent assertions without freeze
