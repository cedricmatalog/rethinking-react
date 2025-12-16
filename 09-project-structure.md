# Chapter 9: Project Structure & Organization

## Introduction

The difference between a maintainable codebase and a mess often comes down to organization. Senior developers create structures that scale.

## Learning Objectives

- Design scalable folder structures
- Organize code by feature vs type
- Implement barrel exports effectively
- Create clear module boundaries
- Scale from small to large projects

## 9.1 Feature-First vs Type-First

### Type-First (Traditional - Doesn't Scale)

```
src/
├── components/
│   ├── Header.tsx
│   ├── Sidebar.tsx
│   ├── UserProfile.tsx
│   ├── UserSettings.tsx
│   ├── ProductCard.tsx
│   ├── ProductList.tsx
│   └── ...50 more files
├── hooks/
│   ├── useUser.ts
│   ├── useProducts.ts
│   └── ...20 more files
├── utils/
│   ├── formatters.ts
│   ├── validators.ts
│   └── ...15 more files
└── services/
    ├── userService.ts
    ├── productService.ts
    └── ...10 more files
```

Problems:
- Hard to find related code
- No clear boundaries
- Difficult to delete features
- Team conflicts

### Feature-First (Scalable)

```
src/
├── features/
│   ├── auth/
│   │   ├── components/
│   │   │   ├── LoginForm.tsx
│   │   │   └── RegisterForm.tsx
│   │   ├── hooks/
│   │   │   └── useAuth.ts
│   │   ├── services/
│   │   │   └── authService.ts
│   │   ├── types/
│   │   │   └── auth.types.ts
│   │   └── index.ts              # Public API
│   │
│   ├── products/
│   │   ├── components/
│   │   │   ├── ProductCard.tsx
│   │   │   ├── ProductList.tsx
│   │   │   └── ProductDetail.tsx
│   │   ├── hooks/
│   │   │   ├── useProducts.ts
│   │   │   └── useProductFilters.ts
│   │   ├── services/
│   │   │   └── productService.ts
│   │   └── index.ts
│   │
│   └── cart/
│       ├── components/
│       ├── hooks/
│       ├── store/
│       └── index.ts
│
├── shared/
│   ├── components/               # Truly shared UI
│   │   ├── Button/
│   │   ├── Modal/
│   │   └── Input/
│   ├── hooks/                    # Generic hooks
│   │   ├── useLocalStorage.ts
│   │   └── useDebounce.ts
│   ├── utils/                    # Pure utilities
│   │   ├── formatters.ts
│   │   └── validators.ts
│   └── types/                    # Global types
│       └── common.types.ts
│
├── pages/                        # Route components
│   ├── HomePage.tsx
│   ├── ProductsPage.tsx
│   └── CheckoutPage.tsx
│
└── app/                          # App-level
    ├── App.tsx
    ├── routes.tsx
    └── providers.tsx
```

Benefits:
- Easy to find code
- Clear boundaries
- Easy to delete features
- Team can work in parallel
- Enforces encapsulation

## 9.2 Barrel Exports and Public APIs

### Creating Feature APIs

```typescript
// features/products/index.ts - Public API
export { ProductCard } from './components/ProductCard';
export { ProductList } from './components/ProductList';
export { useProducts } from './hooks/useProducts';
export type { Product, ProductFilter } from './types';

// Internal components NOT exported
// ProductListItem.tsx is private to this feature

// Usage in other features
import { ProductCard, useProducts } from '@/features/products';
// ✓ Clean imports
// ✓ Clear what's public
// ✓ Easy to refactor internals
```

### Avoiding Barrel Export Pitfalls

```typescript
// BAD: Export everything
export * from './components';
export * from './hooks';
// Problems:
// - No clear API
// - Exports internal details
// - Hard to tree-shake

// GOOD: Explicit exports
export { Button } from './components/Button';
export { Input } from './components/Input';
export { useForm } from './hooks/useForm';
// Benefits:
// - Clear public API
// - Internal code stays private
// - Better tree-shaking
```

## 9.3 Scaling Patterns

### Small Project (< 10 components)

```
src/
├── components/
│   ├── Header.tsx
│   ├── TodoList.tsx
│   └── TodoItem.tsx
├── hooks/
│   └── useTodos.ts
├── App.tsx
└── index.tsx
```

### Medium Project (10-50 components)

```
src/
├── features/
│   ├── todos/
│   ├── auth/
│   └── settings/
├── shared/
│   ├── components/
│   └── hooks/
├── pages/
└── app/
```

### Large Project (50+ components)

```
src/
├── features/
│   ├── todos/
│   │   ├── components/
│   │   ├── hooks/
│   │   ├── services/
│   │   ├── store/
│   │   ├── utils/
│   │   ├── types/
│   │   └── __tests__/
│   └── ...more features
├── shared/
├── pages/
├── app/
└── lib/                          # Core utilities
```

### Enterprise Project (100+ components)

```
packages/
├── web-app/                      # Main application
│   └── src/
│       ├── features/
│       ├── pages/
│       └── app/
├── mobile-app/                   # Mobile version
├── shared-ui/                    # Shared component library
├── shared-utils/                 # Shared utilities
└── api-client/                   # API client library
```

## 9.4 Configuration and Constants

### Environment Variables

```typescript
// config/env.ts
export const config = {
  apiUrl: import.meta.env.VITE_API_URL || 'http://localhost:3000',
  environment: import.meta.env.MODE,
  isDevelopment: import.meta.env.DEV,
  isProduction: import.meta.env.PROD,
  features: {
    analytics: import.meta.env.VITE_ENABLE_ANALYTICS === 'true',
    newDashboard: import.meta.env.VITE_NEW_DASHBOARD === 'true'
  }
} as const;

// Usage
import { config } from '@/config/env';

if (config.features.analytics) {
  initAnalytics();
}
```

### Constants Organization

```typescript
// shared/constants/index.ts
export const ROUTES = {
  HOME: '/',
  PRODUCTS: '/products',
  PRODUCT_DETAIL: (id: string) => `/products/${id}`,
  CART: '/cart',
  CHECKOUT: '/checkout'
} as const;

export const API_ENDPOINTS = {
  USERS: '/api/users',
  PRODUCTS: '/api/products',
  ORDERS: '/api/orders'
} as const;

export const QUERY_KEYS = {
  users: ['users'],
  user: (id: string) => ['users', id],
  products: ['products'],
  product: (id: string) => ['products', id]
} as const;
```

## Review Checklist

- [ ] Organize code by feature, not type
- [ ] Create clear public APIs with barrel exports
- [ ] Scale structure as project grows
- [ ] Use path aliases for clean imports
- [ ] Separate truly shared code
- [ ] Establish naming conventions
- [ ] Document structure decisions

## Key Takeaways

1. **Feature-first scales** - Type-first doesn't
2. **Public APIs matter** - Not everything should be exported
3. **Start simple** - Add structure as you grow
4. **Colocation is good** - Keep related code together
5. **Shared ≠ reusable** - Only share what's truly generic

## Next Chapter

[Chapter 10: Building Design Systems](./10-design-systems.md)
