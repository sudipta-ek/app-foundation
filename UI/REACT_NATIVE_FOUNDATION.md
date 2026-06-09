# React Native Foundation - Technical Details

## 1.1 React Native Foundation Scope
**Objective**: Establish a robust, scalable, and maintainable React Native foundation supporting iOS, iPad, and future Android platforms with BFF-based architecture integration.

---

## UI FOUNDATION COMPONENTS - Technical Specifications

### 1. App Bootstrap/Setup
**Purpose**: Initialize application with essential configurations and context setup

**Technical Details:**
- **Entry Point**: Configure `index.js` / `App.tsx` with platform-specific initialization
- **Initialization Flow**:
  - Runtime permissions setup (iOS/Android specific)
  - Deep linking configuration
  - Firebase/Analytics initialization
  - Theme/localization initialization
  - Global error boundary setup
- **React Native Version**: >= 0.73.x for latest improvements
- **TypeScript**: Strict mode enabled for type safety
- **Module Resolution**: Path aliases for clean imports (`@/components`, `@/services`, etc.)
- **Environment Configuration**: Environment variables per build flavor (dev, staging, prod)
- **Key Libraries**:
  - `react-native`: Core framework
  - `react-native-cli`: Build tooling
  - `@react-native-community/cli-*`: Extended CLI capabilities
  - `dotenv`: Environment variable management
  - `react-native-config`: Build-time configuration

**Implementation Pattern:**
```typescript
// App.tsx
import React, { useEffect } from 'react'
import { GestureHandlerRootView } from 'react-native-gesture-handler'
import { Provider } from 'react-redux'
import { PersistGate } from 'redux-persist/integration/react'
import { ErrorBoundary } from 'react-error-boundary'
import store, { persistor } from './store'
import RootNavigator from './navigation/RootNavigator'
import ErrorFallback from './components/ErrorFallback'

const App: React.FC = () => {
  useEffect(() => {
    initializeApp()
  }, [])

  const initializeApp = async () => {
    // Initialize Firebase
    // Initialize Analytics
    // Setup deep linking
    // Load cached data
  }

  return (
    <ErrorBoundary FallbackComponent={ErrorFallback}>
      <GestureHandlerRootView style={{ flex: 1 }}>
        <Provider store={store}>
          <PersistGate loading={null} persistor={persistor}>
            <RootNavigator />
          </PersistGate>
        </Provider>
      </GestureHandlerRootView>
    </ErrorBoundary>
  )
}

export default App
```

**Deliverables:**
- Configured `App.tsx` with hooks initialization
- Environment configuration files (`.env.dev`, `.env.staging`, `.env.prod`)
- Platform-specific configuration files (Podfile, build.gradle)
- Startup performance metrics setup

---

### 2. Navigation Framework
**Purpose**: Implement navigation system supporting deep linking, stack/tab navigation, and platform-specific UX patterns

**Technical Details:**
- **Primary Library**: React Navigation v6.x
- **Navigation Structure**:
  ```
  RootNavigator
  ├── AuthStack (Login, Register, ForgotPassword)
  ├── AppStack
  │   ├── BottomTabNavigator (iOS/iPad)
  │   │   ├── HomeStack
  │   │   ├── SearchStack
  │   │   └── ProfileStack
  │   └── ModalNavigator (iPad specific overlays)
  └── SplashScreen
  ```
- **Implementation Pattern**:
  - Navigation state managed at root level
  - Type-safe navigation params using TypeScript generics
  - Deep linking configuration with URL schemes and universal links
  - Navigation persistence (save/restore state)
- **Platform-Specific Navigation**:
  - **iPhone**: Bottom Tab Navigation + Stack Navigation
  - **iPad**: Split View Controller pattern (Master-Detail)
  - **Android** (future): Drawer Navigation + Bottom Tab Navigation
- **Key Libraries**:
  - `@react-navigation/native`: Core navigation
  - `@react-navigation/bottom-tabs`: Tab navigation
  - `@react-navigation/stack`: Stack navigation
  - `@react-navigation/drawer`: Drawer navigation (optional)
  - `react-native-screens`: Native screen containers for performance
  - `react-native-safe-area-context`: Safe area handling

**Implementation Pattern:**
```typescript
// navigation/RootNavigator.tsx
import React from 'react'
import { NavigationContainer, NavigatorScreenParams } from '@react-navigation/native'
import { createNativeStackNavigator } from '@react-navigation/native-stack'
import { useAppSelector } from '@/hooks/redux'
import AuthStack from './AuthStack'
import AppStack from './AppStack'
import SplashScreen from '@/screens/SplashScreen'

type RootStackParamList = {
  Splash: undefined
  Auth: NavigatorScreenParams<AuthStackParamList>
  App: NavigatorScreenParams<AppStackParamList>
}

const Stack = createNativeStackNavigator<RootStackParamList>()

const RootNavigator: React.FC = () => {
  const { isLoading, isSignedIn } = useAppSelector(state => state.auth)

  if (isLoading) {
    return <SplashScreen />
  }

  return (
    <NavigationContainer linking={linking}>
      <Stack.Navigator screenOptions={{ headerShown: false }}>
        {isSignedIn ? (
          <Stack.Screen name="App" component={AppStack} />
        ) : (
          <Stack.Screen name="Auth" component={AuthStack} />
        )}
      </Stack.Navigator>
    </NavigationContainer>
  )
}

// Deep linking configuration
const linking = {
  prefixes: ['myapp://', 'https://myapp.com'],
  config: {
    screens: {
      Auth: {
        screens: {
          Login: 'login',
          Register: 'register',
          ForgotPassword: 'forgot-password',
        },
      },
      App: {
        screens: {
          Home: 'home',
          Details: 'details/:id',
          Profile: 'profile/:userId',
        },
      },
    },
  },
}
```

**Deliverables:**
- Navigation structure and routing configuration
- Deep linking implementation with URL schemes
- Type-safe navigation params
- Navigation testing suite

---

### 3. State Management Setup
**Purpose**: Establish centralized, predictable state management across the application

**Technical Details:**
- **Primary Solution**: Redux Toolkit + Redux Thunk
- **Store Architecture**:
  ```
  store/
  ├── slices/
  │   ├── authSlice.ts (user, token, session)
  │   ├── userSlice.ts (profile, preferences)
  │   ├── uiSlice.ts (theme, locale, navigation state)
  │   └── appSlice.ts (initialization, version)
  ├── middlewares/
  │   ├── persistMiddleware.ts (rehydrate from storage)
  │   └── loggerMiddleware.ts (development logging)
  ├── selectors/
  │   └── Memoized selectors using reselect
  └── hooks/
      ├── useAppDispatch.ts
      ├── useAppSelector.ts
      └── Custom hooks for domain logic
  ```
- **Persistence Strategy**:
  - Redux Persist for state rehydration
  - Selective persistence (auth, UI preferences)
  - Encryption for sensitive state
- **Performance Optimization**:
  - Memoized selectors (reselect)
  - Normalized state shape
  - Lazy loading reducers
- **Key Libraries**:
  - `@reduxjs/toolkit`: Redux boilerplate reduction
  - `react-redux`: React bindings
  - `redux-thunk`: Async actions
  - `redux-persist`: State persistence
  - `reselect`: Selector memoization

**Implementation Pattern:**
```typescript
// store/slices/authSlice.ts
import { createSlice, createAsyncThunk, PayloadAction } from '@reduxjs/toolkit'

interface AuthState {
  user: User | null
  accessToken: string | null
  refreshToken: string | null
  isLoading: boolean
  error: string | null
}

const initialState: AuthState = {
  user: null,
  accessToken: null,
  refreshToken: null,
  isLoading: false,
  error: null,
}

export const loginUser = createAsyncThunk(
  'auth/loginUser',
  async (credentials: { email: string; password: string }, { rejectWithValue }) => {
    try {
      const response = await authService.login(credentials)
      return response.data
    } catch (error) {
      return rejectWithValue(error.response.data)
    }
  }
)

const authSlice = createSlice({
  name: 'auth',
  initialState,
  reducers: {
    logout: (state) => {
      state.user = null
      state.accessToken = null
      state.refreshToken = null
    },
    setUser: (state, action: PayloadAction<User>) => {
      state.user = action.payload
    },
  },
  extraReducers: (builder) => {
    builder
      .addCase(loginUser.pending, (state) => {
        state.isLoading = true
        state.error = null
      })
      .addCase(loginUser.fulfilled, (state, action) => {
        state.isLoading = false
        state.user = action.payload.user
        state.accessToken = action.payload.accessToken
        state.refreshToken = action.payload.refreshToken
      })
      .addCase(loginUser.rejected, (state, action) => {
        state.isLoading = false
        state.error = action.payload?.message || 'Login failed'
      })
  },
})

export const { logout, setUser } = authSlice.actions
export default authSlice.reducer
```

**Deliverables:**
- Redux store configuration with slices
- Custom hooks for state access
- Middleware pipeline configuration
- State persistence logic with encryption

---

### 4. API Client Abstraction
**Purpose**: Create a unified, interceptor-based HTTP client for BFF and microservice communication

**Technical Details:**
- **HTTP Client**: Axios with custom interceptors
- **Architecture Pattern**:
  ```
  api/
  ├── client.ts (Axios instance configuration)
  ├── interceptors/
  │   ├── requestInterceptor.ts (auth header, request tracking)
  │   ├── responseInterceptor.ts (error handling, retry logic)
  │   └── errorInterceptor.ts (standardized error transformation)
  ├── services/
  │   ├── authService.ts
  │   ├── userService.ts
  │   ├── dataService.ts
  │   └── ...
  └── types/
      └── apiTypes.ts (request/response interfaces)
  ```
- **Key Features**:
  - Automatic token refresh (JWT)
  - Request/Response logging
  - Retry mechanism with exponential backoff
  - Request timeout handling
  - Request cancellation
  - Request deduplication
- **Base URL Management**: Environment-based configuration
- **Error Handling**: Standardized error response format
- **Timeout Configuration**:
  - Default: 30 seconds
  - Customizable per endpoint
- **Key Libraries**:
  - `axios`: HTTP client
  - `axios-retry`: Retry logic

**Implementation Pattern:**
```typescript
// api/client.ts
import axios, { AxiosInstance, AxiosRequestConfig } from 'axios'
import { store } from '@/store'
import { getAuthToken, refreshAuthToken } from '@/services/authService'

const API_BASE_URL = process.env.REACT_APP_API_BASE_URL

const createApiClient = (): AxiosInstance => {
  const client = axios.create({
    baseURL: API_BASE_URL,
    timeout: 30000,
    headers: {
      'Content-Type': 'application/json',
    },
  })

  // Request interceptor
  client.interceptors.request.use(
    async (config) => {
      const token = await getAuthToken()
      if (token) {
        config.headers.Authorization = `Bearer ${token}`
      }
      config.headers['X-Request-ID'] = generateRequestId()
      return config
    },
    (error) => Promise.reject(error)
  )

  // Response interceptor
  client.interceptors.response.use(
    (response) => response,
    async (error) => {
      const originalRequest = error.config

      // Handle 401 - Token expired
      if (error.response?.status === 401 && !originalRequest._retry) {
        originalRequest._retry = true
        try {
          await refreshAuthToken()
          const token = await getAuthToken()
          originalRequest.headers.Authorization = `Bearer ${token}`
          return client(originalRequest)
        } catch (refreshError) {
          store.dispatch(logout())
          return Promise.reject(refreshError)
        }
      }

      // Transform error
      return Promise.reject(transformError(error))
    }
  )

  return client
}

const apiClient = createApiClient()

export default apiClient
```

**Deliverables:**
- Axios instance with interceptors
- API service layer (abstraction)
- Error transformation utilities
- Request/Response type definitions
- Mock API adapter for testing

---

### 5. Authentication/Session Handling
**Purpose**: Manage user authentication lifecycle, tokens, and session persistence

**Technical Details:**
- **Authentication Flow**:
  ```
  1. Login → BFF (OAuth2/Bearer Token)
  2. Store tokens (Access Token + Refresh Token)
  3. Auto-refresh on expiry
  4. Handle 401 responses → Logout & navigate to login
  ```
- **Token Storage**:
  - Access Token: Secure storage (iOS Keychain, Android Keystore)
  - Refresh Token: Secure storage
  - User metadata: Encrypted shared preferences
- **Session Management**:
  - Session validation on app resume
  - Logout action: Clear tokens & state
  - Biometric authentication (Face ID, Touch ID)
- **Implementation Details**:
  - JWT payload parsing
  - Token expiration monitoring
  - Token refresh queue (prevent multiple refresh calls)
  - Deep link authentication (SSO/OAuth callback)
- **Key Libraries**:
  - `react-native-keychain`: Secure token storage
  - `react-native-jwtdecode`: JWT parsing
  - `redux-persist-sensitive-storage`: Encrypted persistence

**Implementation Pattern:**
```typescript
// services/authService.ts
import * as SecureStore from 'react-native-secure-storage'
import jwtDecode from 'jwt-decode'
import apiClient from '@/api/client'

const TOKEN_KEY = 'auth_access_token'
const REFRESH_TOKEN_KEY = 'auth_refresh_token'

let tokenRefreshPromise: Promise<string> | null = null

export const getAuthToken = async (): Promise<string | null> => {
  return SecureStore.getItem(TOKEN_KEY)
}

export const setAuthToken = async (token: string, refreshToken: string) => {
  await SecureStore.setItem(TOKEN_KEY, token)
  await SecureStore.setItem(REFRESH_TOKEN_KEY, refreshToken)
}

export const isTokenExpired = async (): Promise<boolean> => {
  const token = await getAuthToken()
  if (!token) return true

  try {
    const decoded: any = jwtDecode(token)
    const currentTime = Date.now() / 1000
    return decoded.exp < currentTime
  } catch {
    return true
  }
}

export const refreshAuthToken = async (): Promise<string> => {
  // Prevent multiple refresh calls
  if (tokenRefreshPromise) {
    return tokenRefreshPromise
  }

  tokenRefreshPromise = (async () => {
    try {
      const refreshToken = await SecureStore.getItem(REFRESH_TOKEN_KEY)
      if (!refreshToken) throw new Error('No refresh token')

      const response = await apiClient.post('/auth/refresh', {
        refreshToken,
      })

      const { accessToken, refreshToken: newRefreshToken } = response.data
      await setAuthToken(accessToken, newRefreshToken)
      return accessToken
    } finally {
      tokenRefreshPromise = null
    }
  })()

  return tokenRefreshPromise
}

export const logout = async () => {
  await SecureStore.removeItem(TOKEN_KEY)
  await SecureStore.removeItem(REFRESH_TOKEN_KEY)
}
```

**Deliverables:**
- Authentication context/Redux slice
- Session management service
- Token refresh mechanism
- Biometric authentication integration
- Auth guard component

---

### 6. Offline Framework
**Purpose**: Enable app functionality when network is unavailable with sync strategy

**Technical Details:**
- **Offline Detection**:
  - `@react-native-community/netinfo`: Network state monitoring
  - Real-time network status in Redux
- **Offline Data Strategy**:
  - **Read Operations**: Serve from cache (SQLite/Realm)
  - **Write Operations**: Queue in local database with sync on reconnection
- **Implementation Pattern**:
  ```
  Offline Queue Manager
  ├── Queue Storage (SQLite/Realm)
  ├── Sync Engine (process queue when online)
  ├── Conflict Resolution (last-write-wins / user prompt)
  └── Status Indicators (UI feedback)
  ```
- **Cache Strategy**:
  - API response caching with TTL
  - Selective cache invalidation
  - Cache versioning for migrations
- **Local Database**:
  - **Primary**: Realm (cross-platform, powerful querying)
  - **Alternative**: SQLite (WatermelonDB for performance)
- **Sync Implementation**:
  - Background sync using background tasks
  - Incremental sync (delta sync)
  - Conflict resolution strategy
- **Key Libraries**:
  - `@react-native-community/netinfo`: Network info
  - `realm`: Local database
  - `watermelondb`: Database (alternative)
  - `redux-offline`: Offline-first state management

**Implementation Pattern:**
```typescript
// services/offlineService.ts
import NetInfo from '@react-native-community/netinfo'
import Realm from 'realm'

interface OfflineAction {
  id: string
  type: 'CREATE' | 'UPDATE' | 'DELETE'
  endpoint: string
  payload: any
  timestamp: number
  status: 'pending' | 'syncing' | 'synced' | 'failed'
}

class OfflineQueueManager {
  private realm: Realm
  private isOnline: boolean = true

  constructor() {
    this.realm = new Realm({
      schema: [
        {
          name: 'OfflineAction',
          properties: {
            id: 'string',
            type: 'string',
            endpoint: 'string',
            payload: 'string',
            timestamp: 'int',
            status: 'string',
          },
          primaryKey: 'id',
        },
      ],
    })

    // Monitor network state
    NetInfo.addEventListener((state) => {
      this.isOnline = state.isConnected ?? false
      if (this.isOnline) {
        this.syncQueue()
      }
    })
  }

  async addToQueue(type: string, endpoint: string, payload: any) {
    const action: OfflineAction = {
      id: generateId(),
      type,
      endpoint,
      payload,
      timestamp: Date.now(),
      status: 'pending',
    }

    this.realm.write(() => {
      this.realm.create('OfflineAction', action)
    })

    if (this.isOnline) {
      this.syncQueue()
    }
  }

  async syncQueue() {
    const pendingActions = this.realm
      .objects<OfflineAction>('OfflineAction')
      .filtered('status = "pending"')

    for (const action of pendingActions) {
      try {
        this.realm.write(() => {
          action.status = 'syncing'
        })

        const response = await apiClient({
          method: action.type === 'CREATE' ? 'POST' : action.type === 'UPDATE' ? 'PUT' : 'DELETE',
          url: action.endpoint,
          data: action.payload,
        })

        this.realm.write(() => {
          action.status = 'synced'
        })
      } catch (error) {
        this.realm.write(() => {
          action.status = 'failed'
        })
      }
    }
  }

  getQueueStatus() {
    const pending = this.realm.objects('OfflineAction').filtered('status = "pending"').length
    const failed = this.realm.objects('OfflineAction').filtered('status = "failed"').length
    return { pending, failed }
  }
}
```

**Deliverables:**
- Network state monitoring
- Offline queue management system
- Local database schema
- Sync engine implementation
- Offline UI indicators

---

### 7. Secure Storage Integration
**Purpose**: Implement encrypted storage for sensitive data (tokens, credentials, PII)

**Technical Details:**
- **Storage Hierarchy**:
  ```
  Sensitive Data (Tokens, Passwords)
  └── Platform Keychain (iOS Keychain, Android Keystore)
  
  Encrypted Data (User Preferences, Cached Data)
  └── Encrypted SharedPreferences
  
  Non-Sensitive Data
  └── AsyncStorage / MMKV
  ```
- **Implementation Approach**:
  - **iOS**: Keychain via `react-native-keychain`
  - **Android**: Android Keystore via `react-native-secure-storage`
  - **Cross-Platform**: Encryption library (TweetNaCl.js, libsodium)
- **Encryption Standards**:
  - AES-256-GCM for data encryption
  - Key derivation (PBKDF2) for master key
- **Best Practices**:
  - Never log sensitive data
  - Clear sensitive data on logout
  - Use device biometric for additional security layer
- **Key Libraries**:
  - `react-native-keychain`: Secure storage
  - `react-native-encrypted-storage`: Encrypted storage
  - `tweetnacl.js`: Cryptography

**Implementation Pattern:**
```typescript
// services/secureStorageService.ts
import * as Keychain from 'react-native-keychain'
import EncryptedStorage from 'react-native-encrypted-storage'

export const SecureStorageService = {
  async setToken(key: string, value: string): Promise<void> {
    await Keychain.setGenericPassword(key, value, {
      service: `com.myapp.${key}`,
      securityLevel: Keychain.SECURITY_LEVEL.VERY_STRONG,
    })
  },

  async getToken(key: string): Promise<string | null> {
    const credentials = await Keychain.getGenericPassword({
      service: `com.myapp.${key}`,
    })
    return credentials ? credentials.password : null
  },

  async removeToken(key: string): Promise<void> {
    await Keychain.resetGenericPassword({
      service: `com.myapp.${key}`,
    })
  },

  async setEncryptedData(key: string, value: any): Promise<void> {
    await EncryptedStorage.setItem(key, JSON.stringify(value))
  },

  async getEncryptedData<T>(key: string): Promise<T | null> {
    const data = await EncryptedStorage.getItem(key)
    return data ? JSON.parse(data) : null
  },

  async removeEncryptedData(key: string): Promise<void> {
    await EncryptedStorage.removeItem(key)
  },

  async clearAll(): Promise<void> {
    await Keychain.resetGenericPassword()
    await EncryptedStorage.clear()
  },
}
```

**Deliverables:**
- Secure storage service wrapper
- Encryption/decryption utilities
- Key management strategy
- Data migration for encryption

---

### 8. Shared UI Component Base
**Purpose**: Build reusable, accessible, and themeable component library

**Technical Details:**
- **Component Architecture**:
  ```
  components/
  ├── primitives/
  │   ├── Button.tsx (variants: primary, secondary, tertiary)
  │   ├── Text.tsx (sizes: h1-h6, body, caption)
  │   ├── Input.tsx (text, number, email, password)
  │   ├── Card.tsx
  │   ├── Modal.tsx
  │   ├── Touchable.tsx (press feedback handling)
  │   └── Separator.tsx
  ├── layouts/
  │   ├── SafeAreaLayout.tsx
  │   ├── ScrollableLayout.tsx
  │   ├── FlexLayout.tsx
  │   └── Grid.tsx
  ├── forms/
  │   ├── FormField.tsx
  │   ├── Checkbox.tsx
  │   ├── RadioButton.tsx
  │   ├── Switch.tsx
  │   └── Picker.tsx
  ├── feedback/
  │   ├── Toast.tsx
  │   ├── Loading.tsx
  │   ├── Skeleton.tsx
  │   └── EmptyState.tsx
  └── navigation/
      ├── Header.tsx
      ├── TabBar.tsx
      └── BottomSheet.tsx
  ```
- **Accessibility Standards**:
  - WCAG 2.1 Level AA compliance
  - Screen reader support (VoiceOver, TalkBack)
  - Proper accessibility labels
  - Keyboard navigation
- **Responsive Design**:
  - Device-aware components (useWindowDimensions)
  - Adaptive layouts (iPhone vs iPad)
  - Orientation change handling
  - Safe area consideration
- **Key Libraries**:
  - `react-native`: Core
  - `react-native-svg`: SVG support
  - `lottie-react-native`: Animations
  - `react-native-reanimated`: High-performance animations
  - `react-native-gesture-handler`: Gesture handling

**Implementation Pattern:**
```typescript
// components/primitives/Button.tsx
import React from 'react'
import { TouchableOpacity, Text, StyleSheet, ViewStyle, TextStyle } from 'react-native'
import { useTheme } from '@/hooks/useTheme'

interface ButtonProps {
  variant?: 'primary' | 'secondary' | 'tertiary'
  size?: 'small' | 'medium' | 'large'
  onPress: () => void
  title: string
  disabled?: boolean
  accessibilityLabel?: string
}

const Button: React.FC<ButtonProps> = ({
  variant = 'primary',
  size = 'medium',
  onPress,
  title,
  disabled = false,
  accessibilityLabel,
}) => {
  const { colors, spacing, typography } = useTheme()

  const getButtonStyle = (): ViewStyle => {
    const baseStyle: ViewStyle = {
      borderRadius: spacing.radius.md,
      justifyContent: 'center',
      alignItems: 'center',
    }

    const variants = {
      primary: { backgroundColor: colors.primary },
      secondary: { backgroundColor: colors.secondary },
      tertiary: { backgroundColor: colors.tertiary },
    }

    const sizes = {
      small: { paddingVertical: spacing.sm, paddingHorizontal: spacing.md },
      medium: { paddingVertical: spacing.md, paddingHorizontal: spacing.lg },
      large: { paddingVertical: spacing.lg, paddingHorizontal: spacing.xl },
    }

    return {
      ...baseStyle,
      ...variants[variant],
      ...sizes[size],
      opacity: disabled ? 0.6 : 1,
    }
  }

  const getTextStyle = (): TextStyle => {
    return {
      color: variant === 'tertiary' ? colors.text : colors.white,
      fontSize: typography.sizes.body,
      fontWeight: '600',
    }
  }

  return (
    <TouchableOpacity
      style={getButtonStyle()}
      onPress={onPress}
      disabled={disabled}
      accessibilityLabel={accessibilityLabel || title}
      accessibilityRole="button"
    >
      <Text style={getTextStyle()}>{title}</Text>
    </TouchableOpacity>
  )
}

export default Button
```

**Deliverables:**
- Component library with 20+ reusable components
- Component documentation with examples
- Accessibility guidelines documentation
- Component testing suite

---

### 9. Theming/Design System Integration
**Purpose**: Implement consistent, dynamically switchable theming system

**Technical Details:**
- **Design System Structure**:
  ```
  theme/
  ├── colors.ts
  │   ├── Light theme palette
  │   ├── Dark theme palette
  │   └── Semantic colors (error, success, warning)
  ├── typography.ts
  │   ├── Font families
  │   ├── Font sizes (10px - 40px)
  │   ├── Font weights
  │   └── Line heights
  ├── spacing.ts
  │   └── Scale (4px, 8px, 12px, 16px, 24px, 32px...)
  ├── shadows.ts
  │   └── Elevation levels
  ├── borderRadius.ts
  ├── animations.ts
  │   └── Duration, easing
  ├── zIndex.ts
  └── index.ts (theme object)
  ```
- **Theme Provider Implementation**:
  - Context API for theme distribution
  - Redux slice for theme persistence
  - Appearance listener for system theme changes
- **Dynamic Theme Switching**:
  - Light/Dark mode toggle
  - Custom theme support
  - CSS-in-JS or styled-components pattern
- **Platform Adaptability**:
  - iPhone specific overrides
  - iPad specific overrides
  - Screen size breakpoints

**Implementation Pattern:**
```typescript
// theme/index.ts
import { useColorScheme } from 'react-native'

export const colors = {
  light: {
    primary: '#007AFF',
    secondary: '#5AC8FA',
    tertiary: '#F5F5F5',
    background: '#FFFFFF',
    surface: '#F9F9F9',
    text: '#000000',
    textSecondary: '#666666',
    error: '#FF3B30',
    success: '#34C759',
    warning: '#FF9500',
    border: '#E0E0E0',
  },
  dark: {
    primary: '#0A84FF',
    secondary: '#30B0C0',
    tertiary: '#1C1C1E',
    background: '#000000',
    surface: '#1C1C1E',
    text: '#FFFFFF',
    textSecondary: '#999999',
    error: '#FF453A',
    success: '#32D74B',
    warning: '#FF9500',
    border: '#333333',
  },
}

export const spacing = {
  xs: 4,
  sm: 8,
  md: 12,
  lg: 16,
  xl: 24,
  xxl: 32,
  radius: {
    sm: 4,
    md: 8,
    lg: 12,
    full: 999,
  },
}

export const typography = {
  sizes: {
    h1: 32,
    h2: 28,
    h3: 24,
    h4: 20,
    h5: 18,
    body: 16,
    bodySmall: 14,
    caption: 12,
  },
  weights: {
    light: '300',
    regular: '400',
    medium: '500',
    semibold: '600',
    bold: '700',
  },
}

export type Theme = typeof colors.light

// hooks/useTheme.ts
import { useColorScheme } from 'react-native'
import { useAppSelector } from './redux'
import { colors, spacing, typography } from '@/theme'

export const useTheme = () => {
  const systemColorScheme = useColorScheme()
  const { theme: userTheme } = useAppSelector(state => state.ui)

  const isDark = userTheme === 'dark' || (userTheme === 'system' && systemColorScheme === 'dark')
  const currentColors = isDark ? colors.dark : colors.light

  return {
    colors: currentColors,
    spacing,
    typography,
    isDark,
  }
}
```

**Deliverables:**
- Complete design system specification
- Theme provider component
- Design tokens documentation
- Theme switching implementation

---

### 10. Error Handling/Retry Framework
**Purpose**: Implement comprehensive error handling with intelligent retry strategies

**Technical Details:**
- **Error Hierarchy**:
  ```
  AppError (base)
  ├── NetworkError
  │   ├── TimeoutError
  │   └── ConnectionError
  ├── ApiError
  │   ├── 4xx ClientError
  │   │   ├── 401 AuthenticationError (trigger logout)
  │   │   ├── 403 AuthorizationError
  │   │   └── 400 ValidationError
  │   └── 5xx ServerError
  │       ├── 500 InternalServerError
  │       └── 503 ServiceUnavailableError
  ├── ValidationError
  ├── AuthenticationError
  └── ProgrammingError
  ```
- **Error Handling Pattern**:
  - Global error handler with Redux integration
  - Error boundaries for React components
  - User-friendly error messages
  - Retry buttons for transient failures
- **Retry Strategy**:
  - **Exponential Backoff**: 1s, 2s, 4s, 8s (max 3 retries)
  - **Idempotency**: Only retry safe operations (GET, HEAD, DELETE with idempotency keys)
  - **No Retry**: 4xx errors (except specific ones), auth errors
  - **Circuit Breaker**: Prevent cascading failures
- **Key Libraries**:
  - `axios-retry`: Automatic retry
  - `react-error-boundary`: Error boundaries
  - `sentry-react-native`: Error tracking

**Implementation Pattern:**
```typescript
// services/errorService.ts
export class AppError extends Error {
  constructor(
    public code: string,
    public message: string,
    public statusCode?: number,
    public originalError?: Error
  ) {
    super(message)
  }
}

export const errorTransformer = (error: any): AppError => {
  if (error.response) {
    // Server responded with error status
    const { status, data } = error.response
    const message = data?.message || 'An error occurred'

    if (status === 401) {
      return new AppError('AUTH_ERROR', 'Session expired. Please login again.', status, error)
    } else if (status === 403) {
      return new AppError('FORBIDDEN', 'You do not have permission to perform this action.', status, error)
    } else if (status >= 500) {
      return new AppError('SERVER_ERROR', 'Server error. Please try again later.', status, error)
    }

    return new AppError('API_ERROR', message, status, error)
  } else if (error.request) {
    // Request made but no response
    return new AppError('NETWORK_ERROR', 'Network request failed. Check your connection.', undefined, error)
  }

  return new AppError('UNKNOWN_ERROR', 'An unexpected error occurred.', undefined, error)
}

// middleware/errorMiddleware.ts
export const errorMiddleware = (store: any) => (next: any) => (action: any) => {
  try {
    return next(action)
  } catch (error) {
    const appError = errorTransformer(error)
    store.dispatch(setError(appError))
    logError(appError)
    return Promise.reject(appError)
  }
}

// components/ErrorBoundary.tsx
import React from 'react'
import { View, Text, TouchableOpacity } from 'react-native'
import { useDispatch } from 'react-redux'
import { clearError } from '@/store/slices/errorSlice'

interface ErrorBoundaryProps {
  children: React.ReactNode
}

interface ErrorBoundaryState {
  hasError: boolean
  error?: Error
}

class ErrorBoundary extends React.Component<ErrorBoundaryProps, ErrorBoundaryState> {
  constructor(props: ErrorBoundaryProps) {
    super(props)
    this.state = { hasError: false }
  }

  static getDerivedStateFromError(error: Error) {
    return { hasError: true, error }
  }

  componentDidCatch(error: Error, errorInfo: React.ErrorInfo) {
    console.error('Error caught by boundary:', error, errorInfo)
    logError(error)
  }

  render() {
    if (this.state.hasError) {
      return (
        <View style={{ flex: 1, justifyContent: 'center', alignItems: 'center' }}>
          <Text>Something went wrong!</Text>
          <TouchableOpacity
            onPress={() => this.setState({ hasError: false })}
          >
            <Text>Try again</Text>
          </TouchableOpacity>
        </View>
      )
    }

    return this.props.children
  }
}

export default ErrorBoundary
```

**Deliverables:**
- Error type definitions and hierarchy
- Global error handler
- Retry mechanism implementation
- Error boundary components
- Error logging service

---

### 11. Logging/Analytics Hooks
**Purpose**: Implement comprehensive logging and analytics tracking throughout the app

**Technical Details:**
- **Logging Levels**:
  ```
  DEBUG: Development-only detailed information
  INFO: General informational messages
  WARN: Warning messages (potential issues)
  ERROR: Error messages with stack traces
  CRITICAL: Critical failures
  ```
- **Logging Service Architecture**:
  ```
  Logger
  ├── Console Logger (development)
  ├── File Logger (local storage)
  ├── Remote Logger (backend)
  └── Analytics Logger (Firebase Analytics)
  ```
- **Analytics Tracking**:
  ```
  Events:
  ├── User Actions (button clicks, form submissions)
  ├── Screen Views (screen navigation)
  ├── User Properties (demographics, preferences)
  ├── Performance Metrics (app load time, API latency)
  ├── Errors (crash events, network errors)
  └── Conversions (sign-up, purchase, goal completion)
  ```
- **Performance Monitoring**:
  - App startup time
  - Screen load time
  - API response time
  - Frame rate (FPS) monitoring
  - Memory usage
- **Key Libraries**:
  - `@react-native-firebase/analytics`: Firebase Analytics
  - `@react-native-firebase/crashlytics`: Crash reporting
  - `redux-logger`: Redux action logging

**Implementation Pattern:**
```typescript
// services/analyticsService.ts
import analytics from '@react-native-firebase/analytics'
import crashlytics from '@react-native-firebase/crashlytics'

export const analyticsService = {
  async logScreenView(screenName: string, screenClass?: string) {
    await analytics().logScreenView({
      screen_name: screenName,
      screen_class: screenClass || screenName,
    })
  },

  async logEvent(eventName: string, parameters?: { [key: string]: any }) {
    await analytics().logEvent(eventName, parameters)
  },

  async setUserProperties(userId: string, properties: { [key: string]: string }) {
    await analytics().setUserId(userId)
    await analytics().setUserProperties(properties)
  },

  async logError(error: Error, context?: string) {
    await crashlytics().recordError(error)
    await analytics().logEvent('app_error', {
      error_message: error.message,
      error_context: context,
      stack_trace: error.stack,
    })
  },

  async logPerformance(metricName: string, duration: number) {
    await analytics().logEvent('performance_metric', {
      metric_name: metricName,
      duration_ms: duration,
    })
  },

  async logConversion(conversionName: string, value?: number) {
    await analytics().logEvent('conversion', {
      conversion_name: conversionName,
      value: value || 0,
    })
  },
}

// hooks/useAnalytics.ts
import { useEffect } from 'react'
import { useRoute } from '@react-navigation/native'
import { analyticsService } from '@/services/analyticsService'

export const useAnalytics = () => {
  const route = useRoute()

  useEffect(() => {
    analyticsService.logScreenView(route.name)
  }, [route.name])

  return analyticsService
}

// hooks/usePerformance.ts
import { useEffect, useRef } from 'react'
import { analyticsService } from '@/services/analyticsService'

export const usePerformance = (metricName: string) => {
  const startTime = useRef<number>(Date.now())

  useEffect(() => {
    return () => {
      const duration = Date.now() - startTime.current
      analyticsService.logPerformance(metricName, duration)
    }
  }, [metricName])
}
```

**Deliverables:**
- Logger service implementation
- Analytics event definitions
- Performance monitoring setup
- Log aggregation configuration
- Privacy policy integration

---

## Implementation Priority & Timeline

### Phase 1 (Week 1-2): Core Foundation
- [ ] App Bootstrap/Setup
- [ ] Navigation Framework
- [ ] State Management Setup

### Phase 2 (Week 3-4): Communication & Storage
- [ ] API Client Abstraction
- [ ] Authentication/Session Handling
- [ ] Secure Storage Integration

### Phase 3 (Week 5-6): UI & Theming
- [ ] Shared UI Component Base
- [ ] Theming/Design System Integration

### Phase 4 (Week 7-8): Resilience & Monitoring
- [ ] Offline Framework
- [ ] Error Handling/Retry Framework
- [ ] Logging/Analytics Hooks

---

## Technology Stack Summary

| Component | Technology | Version |
|-----------|-----------|---------|
| Framework | React Native | >= 0.73.x |
| Language | TypeScript | 5.x |
| Navigation | React Navigation | 6.x |
| State Management | Redux Toolkit | 1.9.x |
| HTTP Client | Axios | 1.x |
| Local Database | Realm | 12.x |
| Secure Storage | react-native-keychain | 8.x |
| Error Tracking | Sentry | 5.x |
| Analytics | Firebase Analytics | Latest |
| Testing | Jest + Detox | Latest |

---

## Success Criteria

- ✅ Type-safe codebase (TypeScript strict mode)
- ✅ 90%+ code coverage for core utilities
- ✅ App startup time < 2 seconds
- ✅ Network requests with automatic retry and offline support
- ✅ Accessible components (WCAG 2.1 Level AA)
- ✅ Smooth animations with 60 FPS
- ✅ Comprehensive error tracking and logging
- ✅ Theme switching without app restart
