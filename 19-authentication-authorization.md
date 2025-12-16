# Chapter 19: Authentication & Authorization

## Introduction

Junior developers implement basic login forms and store tokens in localStorage. Senior developers build secure, scalable authentication systems with proper token management, role-based access control, and defense against common security vulnerabilities.

## Learning Objectives

- Implement JWT-based authentication
- Set up OAuth 2.0 / OIDC flows
- Manage authentication state securely
- Implement role-based access control (RBAC)
- Handle token refresh and expiration
- Secure sensitive routes and API calls
- Prevent common security vulnerabilities
- Integrate third-party auth providers

## 19.1 JWT Authentication

### Basic JWT Implementation

```typescript
// types/auth.ts
export interface User {
  id: string;
  email: string;
  name: string;
  roles: string[];
}

export interface AuthTokens {
  accessToken: string;
  refreshToken: string;
}

export interface AuthState {
  user: User | null;
  tokens: AuthTokens | null;
  isAuthenticated: boolean;
  isLoading: boolean;
}

// services/authService.ts
export class AuthService {
  private static readonly ACCESS_TOKEN_KEY = 'access_token';
  private static readonly REFRESH_TOKEN_KEY = 'refresh_token';

  static async login(email: string, password: string): Promise<AuthTokens> {
    const response = await fetch('/api/auth/login', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ email, password }),
    });

    if (!response.ok) {
      throw new Error('Login failed');
    }

    const tokens: AuthTokens = await response.json();
    this.setTokens(tokens);
    return tokens;
  }

  static async register(email: string, password: string, name: string): Promise<AuthTokens> {
    const response = await fetch('/api/auth/register', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ email, password, name }),
    });

    if (!response.ok) {
      throw new Error('Registration failed');
    }

    const tokens: AuthTokens = await response.json();
    this.setTokens(tokens);
    return tokens;
  }

  static async refreshAccessToken(): Promise<string> {
    const refreshToken = this.getRefreshToken();

    if (!refreshToken) {
      throw new Error('No refresh token available');
    }

    const response = await fetch('/api/auth/refresh', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ refreshToken }),
    });

    if (!response.ok) {
      this.clearTokens();
      throw new Error('Token refresh failed');
    }

    const { accessToken } = await response.json();
    this.setAccessToken(accessToken);
    return accessToken;
  }

  static async logout(): Promise<void> {
    const refreshToken = this.getRefreshToken();

    if (refreshToken) {
      try {
        await fetch('/api/auth/logout', {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({ refreshToken }),
        });
      } catch (error) {
        console.error('Logout request failed:', error);
      }
    }

    this.clearTokens();
  }

  static getAccessToken(): string | null {
    return sessionStorage.getItem(this.ACCESS_TOKEN_KEY);
  }

  static getRefreshToken(): string | null {
    return localStorage.getItem(this.REFRESH_TOKEN_KEY);
  }

  static setTokens(tokens: AuthTokens): void {
    sessionStorage.setItem(this.ACCESS_TOKEN_KEY, tokens.accessToken);
    localStorage.setItem(this.REFRESH_TOKEN_KEY, tokens.refreshToken);
  }

  static setAccessToken(token: string): void {
    sessionStorage.setItem(this.ACCESS_TOKEN_KEY, token);
  }

  static clearTokens(): void {
    sessionStorage.removeItem(this.ACCESS_TOKEN_KEY);
    localStorage.removeItem(this.REFRESH_TOKEN_KEY);
  }

  static decodeToken(token: string): any {
    try {
      const base64Url = token.split('.')[1];
      const base64 = base64Url.replace(/-/g, '+').replace(/_/g, '/');
      const jsonPayload = decodeURIComponent(
        atob(base64)
          .split('')
          .map((c) => '%' + ('00' + c.charCodeAt(0).toString(16)).slice(-2))
          .join('')
      );
      return JSON.parse(jsonPayload);
    } catch (error) {
      return null;
    }
  }

  static isTokenExpired(token: string): boolean {
    const decoded = this.decodeToken(token);
    if (!decoded || !decoded.exp) return true;

    const currentTime = Date.now() / 1000;
    return decoded.exp < currentTime;
  }
}
```

### Auth Context Provider

```typescript
// contexts/AuthContext.tsx
import { createContext, useContext, useEffect, useState, ReactNode } from 'react';
import { AuthService } from '@/services/authService';
import type { User, AuthState } from '@/types/auth';

interface AuthContextType extends AuthState {
  login: (email: string, password: string) => Promise<void>;
  register: (email: string, password: string, name: string) => Promise<void>;
  logout: () => Promise<void>;
  refreshAuth: () => Promise<void>;
}

const AuthContext = createContext<AuthContextType | undefined>(undefined);

export function AuthProvider({ children }: { children: ReactNode }) {
  const [state, setState] = useState<AuthState>({
    user: null,
    tokens: null,
    isAuthenticated: false,
    isLoading: true,
  });

  useEffect(() => {
    // Check for existing session on mount
    const initAuth = async () => {
      const accessToken = AuthService.getAccessToken();

      if (accessToken && !AuthService.isTokenExpired(accessToken)) {
        const decoded = AuthService.decodeToken(accessToken);
        setState({
          user: decoded.user,
          tokens: {
            accessToken,
            refreshToken: AuthService.getRefreshToken()!,
          },
          isAuthenticated: true,
          isLoading: false,
        });
      } else if (AuthService.getRefreshToken()) {
        // Try to refresh the token
        try {
          await refreshAuth();
        } catch (error) {
          setState((prev) => ({ ...prev, isLoading: false }));
        }
      } else {
        setState((prev) => ({ ...prev, isLoading: false }));
      }
    };

    initAuth();
  }, []);

  const login = async (email: string, password: string) => {
    setState((prev) => ({ ...prev, isLoading: true }));

    try {
      const tokens = await AuthService.login(email, password);
      const decoded = AuthService.decodeToken(tokens.accessToken);

      setState({
        user: decoded.user,
        tokens,
        isAuthenticated: true,
        isLoading: false,
      });
    } catch (error) {
      setState((prev) => ({ ...prev, isLoading: false }));
      throw error;
    }
  };

  const register = async (email: string, password: string, name: string) => {
    setState((prev) => ({ ...prev, isLoading: true }));

    try {
      const tokens = await AuthService.register(email, password, name);
      const decoded = AuthService.decodeToken(tokens.accessToken);

      setState({
        user: decoded.user,
        tokens,
        isAuthenticated: true,
        isLoading: false,
      });
    } catch (error) {
      setState((prev) => ({ ...prev, isLoading: false }));
      throw error;
    }
  };

  const logout = async () => {
    await AuthService.logout();
    setState({
      user: null,
      tokens: null,
      isAuthenticated: false,
      isLoading: false,
    });
  };

  const refreshAuth = async () => {
    try {
      const accessToken = await AuthService.refreshAccessToken();
      const decoded = AuthService.decodeToken(accessToken);

      setState((prev) => ({
        ...prev,
        user: decoded.user,
        tokens: {
          accessToken,
          refreshToken: AuthService.getRefreshToken()!,
        },
        isAuthenticated: true,
        isLoading: false,
      }));
    } catch (error) {
      setState({
        user: null,
        tokens: null,
        isAuthenticated: false,
        isLoading: false,
      });
      throw error;
    }
  };

  return (
    <AuthContext.Provider
      value={{
        ...state,
        login,
        register,
        logout,
        refreshAuth,
      }}
    >
      {children}
    </AuthContext.Provider>
  );
}

export function useAuth() {
  const context = useContext(AuthContext);
  if (!context) {
    throw new Error('useAuth must be used within AuthProvider');
  }
  return context;
}
```

### Protected Routes

```typescript
// components/ProtectedRoute.tsx
import { Navigate, useLocation } from 'react-router-dom';
import { useAuth } from '@/contexts/AuthContext';

interface ProtectedRouteProps {
  children: React.ReactNode;
  requiredRoles?: string[];
}

export function ProtectedRoute({ children, requiredRoles }: ProtectedRouteProps) {
  const { isAuthenticated, isLoading, user } = useAuth();
  const location = useLocation();

  if (isLoading) {
    return <LoadingSpinner />;
  }

  if (!isAuthenticated) {
    // Redirect to login, preserving the intended destination
    return <Navigate to="/login" state={{ from: location }} replace />;
  }

  if (requiredRoles && user) {
    const hasRequiredRole = requiredRoles.some((role) =>
      user.roles.includes(role)
    );

    if (!hasRequiredRole) {
      return <Navigate to="/unauthorized" replace />;
    }
  }

  return <>{children}</>;
}

// Usage in router
function App() {
  return (
    <Routes>
      <Route path="/login" element={<LoginPage />} />
      <Route path="/register" element={<RegisterPage />} />

      {/* Protected routes */}
      <Route
        path="/dashboard"
        element={
          <ProtectedRoute>
            <Dashboard />
          </ProtectedRoute>
        }
      />

      {/* Admin-only routes */}
      <Route
        path="/admin"
        element={
          <ProtectedRoute requiredRoles={['admin']}>
            <AdminPanel />
          </ProtectedRoute>
        }
      />
    </Routes>
  );
}
```

## 19.2 Axios Interceptors for Token Management

### Automatic Token Injection and Refresh

```typescript
// services/apiClient.ts
import axios, { AxiosError, InternalAxiosRequestConfig } from 'axios';
import { AuthService } from './authService';

const apiClient = axios.create({
  baseURL: import.meta.env.VITE_API_URL,
  headers: {
    'Content-Type': 'application/json',
  },
});

// Request interceptor - inject access token
apiClient.interceptors.request.use(
  (config: InternalAxiosRequestConfig) => {
    const token = AuthService.getAccessToken();

    if (token && config.headers) {
      config.headers.Authorization = `Bearer ${token}`;
    }

    return config;
  },
  (error) => Promise.reject(error)
);

// Response interceptor - handle token refresh
let isRefreshing = false;
let failedQueue: Array<{
  resolve: (value?: any) => void;
  reject: (reason?: any) => void;
}> = [];

const processQueue = (error: any = null, token: string | null = null) => {
  failedQueue.forEach((prom) => {
    if (error) {
      prom.reject(error);
    } else {
      prom.resolve(token);
    }
  });

  failedQueue = [];
};

apiClient.interceptors.response.use(
  (response) => response,
  async (error: AxiosError) => {
    const originalRequest = error.config as InternalAxiosRequestConfig & {
      _retry?: boolean;
    };

    // If error is 401 and we haven't retried yet
    if (error.response?.status === 401 && !originalRequest._retry) {
      if (isRefreshing) {
        // Queue the request while refresh is in progress
        return new Promise((resolve, reject) => {
          failedQueue.push({ resolve, reject });
        })
          .then((token) => {
            if (originalRequest.headers) {
              originalRequest.headers.Authorization = `Bearer ${token}`;
            }
            return apiClient(originalRequest);
          })
          .catch((err) => Promise.reject(err));
      }

      originalRequest._retry = true;
      isRefreshing = true;

      try {
        const newToken = await AuthService.refreshAccessToken();
        processQueue(null, newToken);

        if (originalRequest.headers) {
          originalRequest.headers.Authorization = `Bearer ${newToken}`;
        }

        return apiClient(originalRequest);
      } catch (refreshError) {
        processQueue(refreshError, null);
        AuthService.clearTokens();
        window.location.href = '/login';
        return Promise.reject(refreshError);
      } finally {
        isRefreshing = false;
      }
    }

    return Promise.reject(error);
  }
);

export default apiClient;
```

## 19.3 OAuth 2.0 / OpenID Connect

### OAuth with Google

```typescript
// services/oauthService.ts
export class OAuthService {
  private static readonly GOOGLE_CLIENT_ID = import.meta.env.VITE_GOOGLE_CLIENT_ID;
  private static readonly REDIRECT_URI = `${window.location.origin}/auth/callback`;

  static initiateGoogleLogin() {
    const params = new URLSearchParams({
      client_id: this.GOOGLE_CLIENT_ID,
      redirect_uri: this.REDIRECT_URI,
      response_type: 'code',
      scope: 'openid email profile',
      state: this.generateState(),
      nonce: this.generateNonce(),
    });

    // Store state for verification
    sessionStorage.setItem('oauth_state', params.get('state')!);

    window.location.href = `https://accounts.google.com/o/oauth2/v2/auth?${params}`;
  }

  static async handleCallback(code: string, state: string): Promise<AuthTokens> {
    // Verify state to prevent CSRF
    const savedState = sessionStorage.getItem('oauth_state');
    if (state !== savedState) {
      throw new Error('Invalid state parameter');
    }

    sessionStorage.removeItem('oauth_state');

    // Exchange code for tokens
    const response = await fetch('/api/auth/oauth/callback', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ code, provider: 'google' }),
    });

    if (!response.ok) {
      throw new Error('OAuth callback failed');
    }

    return response.json();
  }

  private static generateState(): string {
    const array = new Uint8Array(32);
    crypto.getRandomValues(array);
    return Array.from(array, (byte) => byte.toString(16).padStart(2, '0')).join('');
  }

  private static generateNonce(): string {
    const array = new Uint8Array(16);
    crypto.getRandomValues(array);
    return Array.from(array, (byte) => byte.toString(16).padStart(2, '0')).join('');
  }
}

// components/OAuthCallback.tsx
function OAuthCallback() {
  const navigate = useNavigate();
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    const handleCallback = async () => {
      const params = new URLSearchParams(window.location.search);
      const code = params.get('code');
      const state = params.get('state');
      const errorParam = params.get('error');

      if (errorParam) {
        setError(errorParam);
        return;
      }

      if (!code || !state) {
        setError('Missing required parameters');
        return;
      }

      try {
        const tokens = await OAuthService.handleCallback(code, state);
        AuthService.setTokens(tokens);
        navigate('/dashboard');
      } catch (err) {
        setError('Authentication failed');
      }
    };

    handleCallback();
  }, [navigate]);

  if (error) {
    return <div>Error: {error}</div>;
  }

  return <LoadingSpinner />;
}
```

### Using Auth0

```typescript
// Install: npm install @auth0/auth0-react

// main.tsx
import { Auth0Provider } from '@auth0/auth0-react';

ReactDOM.createRoot(document.getElementById('root')!).render(
  <Auth0Provider
    domain={import.meta.env.VITE_AUTH0_DOMAIN}
    clientId={import.meta.env.VITE_AUTH0_CLIENT_ID}
    authorizationParams={{
      redirect_uri: window.location.origin,
      audience: import.meta.env.VITE_AUTH0_AUDIENCE,
      scope: 'openid profile email',
    }}
    cacheLocation="localstorage"
  >
    <App />
  </Auth0Provider>
);

// components/LoginButton.tsx
import { useAuth0 } from '@auth0/auth0-react';

export function LoginButton() {
  const { loginWithRedirect, logout, isAuthenticated, user } = useAuth0();

  if (isAuthenticated) {
    return (
      <div>
        <span>Welcome, {user?.name}</span>
        <button onClick={() => logout({ logoutParams: { returnTo: window.location.origin } })}>
          Logout
        </button>
      </div>
    );
  }

  return <button onClick={() => loginWithRedirect()}>Login</button>;
}

// hooks/useAuthToken.ts
import { useAuth0 } from '@auth0/auth0-react';
import { useEffect, useState } from 'react';

export function useAuthToken() {
  const { getAccessTokenSilently } = useAuth0();
  const [token, setToken] = useState<string | null>(null);

  useEffect(() => {
    const getToken = async () => {
      try {
        const accessToken = await getAccessTokenSilently();
        setToken(accessToken);
      } catch (error) {
        console.error('Error getting token:', error);
      }
    };

    getToken();
  }, [getAccessTokenSilently]);

  return token;
}
```

## 19.4 Role-Based Access Control (RBAC)

### Permission System

```typescript
// types/permissions.ts
export enum Permission {
  // User permissions
  USER_READ = 'user:read',
  USER_WRITE = 'user:write',
  USER_DELETE = 'user:delete',

  // Post permissions
  POST_READ = 'post:read',
  POST_WRITE = 'post:write',
  POST_DELETE = 'post:delete',
  POST_PUBLISH = 'post:publish',

  // Admin permissions
  ADMIN_ACCESS = 'admin:access',
  ADMIN_USERS = 'admin:users',
  ADMIN_SETTINGS = 'admin:settings',
}

export interface Role {
  name: string;
  permissions: Permission[];
}

export const ROLES: Record<string, Role> = {
  viewer: {
    name: 'viewer',
    permissions: [Permission.USER_READ, Permission.POST_READ],
  },
  editor: {
    name: 'editor',
    permissions: [
      Permission.USER_READ,
      Permission.POST_READ,
      Permission.POST_WRITE,
      Permission.POST_PUBLISH,
    ],
  },
  admin: {
    name: 'admin',
    permissions: Object.values(Permission),
  },
};

// services/permissionService.ts
export class PermissionService {
  static hasPermission(userRoles: string[], permission: Permission): boolean {
    return userRoles.some((roleName) => {
      const role = ROLES[roleName];
      return role?.permissions.includes(permission);
    });
  }

  static hasAnyPermission(userRoles: string[], permissions: Permission[]): boolean {
    return permissions.some((permission) =>
      this.hasPermission(userRoles, permission)
    );
  }

  static hasAllPermissions(userRoles: string[], permissions: Permission[]): boolean {
    return permissions.every((permission) =>
      this.hasPermission(userRoles, permission)
    );
  }
}

// hooks/usePermission.ts
import { useAuth } from '@/contexts/AuthContext';

export function usePermission() {
  const { user } = useAuth();

  const hasPermission = (permission: Permission): boolean => {
    if (!user) return false;
    return PermissionService.hasPermission(user.roles, permission);
  };

  const hasAnyPermission = (permissions: Permission[]): boolean => {
    if (!user) return false;
    return PermissionService.hasAnyPermission(user.roles, permissions);
  };

  const hasAllPermissions = (permissions: Permission[]): boolean => {
    if (!user) return false;
    return PermissionService.hasAllPermissions(user.roles, permissions);
  };

  return {
    hasPermission,
    hasAnyPermission,
    hasAllPermissions,
  };
}

// components/Can.tsx - Declarative permission checking
interface CanProps {
  perform: Permission | Permission[];
  requireAll?: boolean;
  children: React.ReactNode;
  fallback?: React.ReactNode;
}

export function Can({ perform, requireAll = false, children, fallback = null }: CanProps) {
  const { hasPermission, hasAnyPermission, hasAllPermissions } = usePermission();

  const isAllowed = Array.isArray(perform)
    ? requireAll
      ? hasAllPermissions(perform)
      : hasAnyPermission(perform)
    : hasPermission(perform);

  return isAllowed ? <>{children}</> : <>{fallback}</>;
}

// Usage
function PostActions({ post }: { post: Post }) {
  return (
    <div>
      <Can perform={Permission.POST_WRITE}>
        <button>Edit</button>
      </Can>

      <Can perform={Permission.POST_DELETE}>
        <button>Delete</button>
      </Can>

      <Can perform={Permission.POST_PUBLISH} fallback={<span>Draft</span>}>
        <button>Publish</button>
      </Can>
    </div>
  );
}
```

## 19.5 Secure Token Storage

### Using HTTP-Only Cookies (Most Secure)

```typescript
// Server-side: Set HTTP-only cookie
// Backend should set cookies instead of sending tokens in response body

app.post('/api/auth/login', async (req, res) => {
  const { email, password } = req.body;

  // Validate credentials
  const user = await validateUser(email, password);

  const accessToken = generateAccessToken(user);
  const refreshToken = generateRefreshToken(user);

  // Set HTTP-only cookies
  res.cookie('accessToken', accessToken, {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'strict',
    maxAge: 15 * 60 * 1000, // 15 minutes
  });

  res.cookie('refreshToken', refreshToken, {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'strict',
    maxAge: 7 * 24 * 60 * 60 * 1000, // 7 days
    path: '/api/auth/refresh',
  });

  res.json({ user });
});

// Client-side: axios automatically sends cookies
const apiClient = axios.create({
  baseURL: import.meta.env.VITE_API_URL,
  withCredentials: true, // Important: send cookies with requests
});
```

### Using Secure Storage Library

```typescript
// Install: npm install secure-web-storage

import SecureStorage from 'secure-web-storage';
import CryptoJS from 'crypto-js';

const SECRET_KEY = import.meta.env.VITE_STORAGE_SECRET_KEY;

const secureStorage = new SecureStorage(localStorage, {
  hash: function hash(key: string) {
    return CryptoJS.SHA256(key, SECRET_KEY).toString();
  },
  encrypt: function encrypt(data: string) {
    return CryptoJS.AES.encrypt(data, SECRET_KEY).toString();
  },
  decrypt: function decrypt(data: string) {
    return CryptoJS.AES.decrypt(data, SECRET_KEY).toString(CryptoJS.enc.Utf8);
  },
});

// Usage
secureStorage.setItem('refreshToken', token);
const token = secureStorage.getItem('refreshToken');
```

## 19.6 Security Best Practices

### CSRF Protection

```typescript
// services/csrfService.ts
export class CSRFService {
  private static readonly CSRF_TOKEN_KEY = 'csrf_token';

  static async getToken(): Promise<string> {
    // Get CSRF token from server
    const response = await fetch('/api/auth/csrf-token');
    const { token } = await response.json();

    sessionStorage.setItem(this.CSRF_TOKEN_KEY, token);
    return token;
  }

  static getStoredToken(): string | null {
    return sessionStorage.getItem(this.CSRF_TOKEN_KEY);
  }
}

// Add to axios interceptor
apiClient.interceptors.request.use(async (config) => {
  // Only add CSRF token for state-changing requests
  if (['POST', 'PUT', 'DELETE', 'PATCH'].includes(config.method?.toUpperCase() || '')) {
    let csrfToken = CSRFService.getStoredToken();

    if (!csrfToken) {
      csrfToken = await CSRFService.getToken();
    }

    if (config.headers) {
      config.headers['X-CSRF-Token'] = csrfToken;
    }
  }

  return config;
});
```

### Content Security Policy

```html
<!-- index.html -->
<meta
  http-equiv="Content-Security-Policy"
  content="
    default-src 'self';
    script-src 'self' 'unsafe-inline' 'unsafe-eval' https://accounts.google.com;
    style-src 'self' 'unsafe-inline';
    img-src 'self' data: https:;
    font-src 'self' data:;
    connect-src 'self' https://api.example.com;
    frame-src 'self' https://accounts.google.com;
  "
/>
```

### Rate Limiting on Client

```typescript
// hooks/useRateLimitedAuth.ts
import { useState, useCallback } from 'react';

const MAX_ATTEMPTS = 5;
const LOCKOUT_DURATION = 5 * 60 * 1000; // 5 minutes

export function useRateLimitedAuth() {
  const [attempts, setAttempts] = useState(0);
  const [lockedUntil, setLockedUntil] = useState<number | null>(null);

  const isLocked = useCallback(() => {
    if (!lockedUntil) return false;

    if (Date.now() < lockedUntil) {
      return true;
    }

    // Lockout expired, reset
    setLockedUntil(null);
    setAttempts(0);
    return false;
  }, [lockedUntil]);

  const recordAttempt = useCallback((success: boolean) => {
    if (success) {
      setAttempts(0);
      setLockedUntil(null);
      return;
    }

    const newAttempts = attempts + 1;
    setAttempts(newAttempts);

    if (newAttempts >= MAX_ATTEMPTS) {
      setLockedUntil(Date.now() + LOCKOUT_DURATION);
    }
  }, [attempts]);

  const getRemainingTime = useCallback(() => {
    if (!lockedUntil) return 0;
    return Math.max(0, lockedUntil - Date.now());
  }, [lockedUntil]);

  return {
    isLocked: isLocked(),
    remainingAttempts: Math.max(0, MAX_ATTEMPTS - attempts),
    remainingTime: getRemainingTime(),
    recordAttempt,
  };
}
```

## Real-World Scenario: Implementing Secure Auth

### The Challenge

Build authentication for a SaaS application:
- Multiple user roles (admin, manager, user)
- Social login (Google, GitHub)
- Multi-factor authentication
- Session management
- Secure token handling

### Senior Approach

```typescript
// Complete auth implementation with all best practices

// 1. Auth context with comprehensive features
const AuthProvider = () => {
  // JWT with refresh tokens
  // HTTP-only cookies for storage
  // Automatic token refresh
  // Role-based permissions
  // Session timeout
  // Multi-device logout
};

// 2. Protected routes with permission checks
<Route
  path="/admin"
  element={
    <ProtectedRoute requiredPermissions={[Permission.ADMIN_ACCESS]}>
      <AdminPanel />
    </ProtectedRoute>
  }
/>

// 3. OAuth integration
const handleGoogleLogin = async () => {
  await OAuthService.initiateGoogleLogin();
};

// 4. API client with security
// - Automatic token injection
// - CSRF protection
// - Request signing
// - Rate limiting

// 5. Security headers
// - Content Security Policy
// - HSTS
// - X-Frame-Options
// - X-Content-Type-Options

// Results:
// - Secure authentication
// - No XSS vulnerabilities
// - CSRF protected
// - Token theft prevented
// - Proper session management
```

## Chapter Exercise: Build Auth System

Create a complete authentication system:

**Requirements:**
1. JWT-based authentication
2. Protected routes with role checking
3. Token refresh mechanism
4. OAuth with Google or GitHub
5. Permission-based UI rendering
6. Secure token storage
7. CSRF protection
8. Rate limiting

**Bonus:**
- Multi-factor authentication
- Session management
- Device tracking
- Audit logging

## Review Checklist

- [ ] JWT implementation secure
- [ ] Tokens stored securely (HTTP-only cookies preferred)
- [ ] Automatic token refresh
- [ ] Protected routes configured
- [ ] RBAC implemented
- [ ] OAuth integration working
- [ ] CSRF protection enabled
- [ ] Rate limiting in place
- [ ] Security headers configured
- [ ] Logout clears all tokens

## Key Takeaways

1. **Never store tokens in localStorage** - Use HTTP-only cookies
2. **Always refresh tokens automatically** - Better UX
3. **Implement proper RBAC** - Not just admin/user
4. **Use OAuth for social login** - Don't roll your own
5. **Protect against CSRF** - Especially with cookies
6. **Rate limit authentication** - Prevent brute force
7. **Security is layered** - Multiple defenses

## Further Reading

- OAuth 2.0 specification
- JWT best practices
- OWASP Authentication Cheat Sheet
- Auth0 documentation
- The Web Application Hacker's Handbook

## Next Chapter

[Chapter 20: Data Fetching & Caching](./20-data-fetching-caching.md)
