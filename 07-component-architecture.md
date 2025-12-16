# Chapter 7: Component Architecture

## Introduction

The difference between junior and senior developers is most visible in how they structure components. Junior developers organize by file type. Senior developers organize by feature, domain, and coupling.

This chapter teaches you to think architecturally about components.

## Learning Objectives

- Design scalable component hierarchies
- Apply SOLID principles to React components
- Master component coupling and cohesion
- Create maintainable folder structures
- Build components that last

## 7.1 Component Responsibility

### Single Responsibility Principle in React

```javascript
// Junior: God component doing everything
function UserDashboard() {
  const [user, setUser] = useState(null);
  const [posts, setPosts] = useState([]);
  const [loading, setLoading] = useState(true);
  const [theme, setTheme] = useState('light');

  useEffect(() => {
    // Fetch user
    fetch('/api/user').then(r => r.json()).then(setUser);
    // Fetch posts
    fetch('/api/posts').then(r => r.json()).then(setPosts);
    // Load theme from localStorage
    const savedTheme = localStorage.getItem('theme');
    setTheme(savedTheme || 'light');
  }, []);

  const handleThemeChange = (newTheme) => {
    setTheme(newTheme);
    localStorage.setItem('theme', newTheme);
  };

  const handleDeletePost = (postId) => {
    fetch(`/api/posts/${postId}`, { method: 'DELETE' })
      .then(() => setPosts(posts.filter(p => p.id !== postId)));
  };

  return (
    <div className={`dashboard theme-${theme}`}>
      <header>
        <h1>Welcome {user?.name}</h1>
        <button onClick={() => handleThemeChange(theme === 'light' ? 'dark' : 'light')}>
          Toggle Theme
        </button>
      </header>
      <div className="posts">
        {posts.map(post => (
          <div key={post.id}>
            <h2>{post.title}</h2>
            <p>{post.content}</p>
            <button onClick={() => handleDeletePost(post.id)}>Delete</button>
          </div>
        ))}
      </div>
    </div>
  );
}
```

```javascript
// Senior: Each component has one clear responsibility

// 1. Theme management
function useTheme() {
  const [theme, setTheme] = useState(() =>
    localStorage.getItem('theme') || 'light'
  );

  const toggleTheme = () => {
    const newTheme = theme === 'light' ? 'dark' : 'light';
    setTheme(newTheme);
    localStorage.setItem('theme', newTheme);
  };

  return { theme, toggleTheme };
}

// 2. Data fetching
function useUser() {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetch('/api/user')
      .then(r => r.json())
      .then(setUser)
      .finally(() => setLoading(false));
  }, []);

  return { user, loading };
}

function usePosts() {
  const [posts, setPosts] = useState([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetch('/api/posts')
      .then(r => r.json())
      .then(setPosts)
      .finally(() => setLoading(false));
  }, []);

  const deletePost = async (postId) => {
    await fetch(`/api/posts/${postId}`, { method: 'DELETE' });
    setPosts(posts.filter(p => p.id !== postId));
  };

  return { posts, loading, deletePost };
}

// 3. Presentation components
function DashboardHeader({ userName, onThemeToggle }) {
  return (
    <header>
      <h1>Welcome {userName}</h1>
      <button onClick={onThemeToggle}>Toggle Theme</button>
    </header>
  );
}

function PostList({ posts, onDeletePost }) {
  return (
    <div className="posts">
      {posts.map(post => (
        <Post key={post.id} post={post} onDelete={onDeletePost} />
      ))}
    </div>
  );
}

function Post({ post, onDelete }) {
  return (
    <article>
      <h2>{post.title}</h2>
      <p>{post.content}</p>
      <button onClick={() => onDelete(post.id)}>Delete</button>
    </article>
  );
}

// 4. Container component - orchestration only
function UserDashboard() {
  const { theme, toggleTheme } = useTheme();
  const { user, loading: userLoading } = useUser();
  const { posts, loading: postsLoading, deletePost } = usePosts();

  if (userLoading || postsLoading) return <LoadingSpinner />;

  return (
    <div className={`dashboard theme-${theme}`}>
      <DashboardHeader userName={user.name} onThemeToggle={toggleTheme} />
      <PostList posts={posts} onDeletePost={deletePost} />
    </div>
  );
}
```

### Benefits of SRP
- Easy to test
- Easy to understand
- Easy to change
- Reusable pieces

### Hands-On Exercise 7.1

Refactor a shopping cart component that handles:
- Product fetching
- Cart state
- Checkout process
- Analytics tracking
- UI rendering

Break it into appropriate components and hooks with single responsibilities.

## 7.2 Container vs Presentation Pattern

### The Pattern

```javascript
// Presentation Component - Pure UI
function ProductCard({
  product,
  onAddToCart,
  onToggleFavorite,
  isFavorite
}) {
  return (
    <div className="product-card">
      <img src={product.image} alt={product.name} />
      <h3>{product.name}</h3>
      <p>${product.price}</p>
      <button onClick={() => onAddToCart(product)}>Add to Cart</button>
      <button onClick={() => onToggleFavorite(product.id)}>
        {isFavorite ? '❤️' : '🤍'}
      </button>
    </div>
  );
}

// Container Component - Logic and data
function ProductCardContainer({ productId }) {
  const product = useProduct(productId);
  const { addToCart } = useCart();
  const { isFavorite, toggleFavorite } = useFavorites();

  if (!product) return <Skeleton />;

  return (
    <ProductCard
      product={product}
      onAddToCart={addToCart}
      onToggleFavorite={toggleFavorite}
      isFavorite={isFavorite(productId)}
    />
  );
}
```

### When to Use This Pattern

| Use Container/Presentation | Don't Use |
|---------------------------|-----------|
| Complex data fetching | Simple components |
| Multiple data sources | Single purpose components |
| Need to test UI separately | Tightly coupled logic/UI |
| Building design system | Prototype/MVP phase |

### Modern Alternative: Colocation

```javascript
// Modern approach - colocate related concerns
function ProductCard({ productId }) {
  // Data fetching
  const product = useProduct(productId);

  // Business logic
  const { addToCart } = useCart();
  const { isFavorite, toggleFavorite } = useFavorites();

  // Early returns for loading states
  if (!product) return <Skeleton />;

  // UI
  return (
    <div className="product-card">
      <img src={product.image} alt={product.name} />
      <h3>{product.name}</h3>
      <p>${product.price}</p>
      <button onClick={() => addToCart(product)}>Add to Cart</button>
      <button onClick={() => toggleFavorite(product.id)}>
        {isFavorite(productId) ? '❤️' : '🤍'}
      </button>
    </div>
  );
}
```

### Decision Framework

Ask yourself:
1. **Do I need to test UI separately?** → Container/Presentation
2. **Will this UI be reused with different data?** → Presentation component
3. **Is this a one-off component?** → Colocated
4. **Am I building a component library?** → Presentation components

## 7.3 Component Coupling and Cohesion

### High Cohesion, Low Coupling

```javascript
// BAD: Low cohesion - unrelated concerns together
function UserProfile() {
  const [user, setUser] = useState(null);
  const [weather, setWeather] = useState(null); // Why is this here?
  const [stockPrice, setStockPrice] = useState(null); // And this?

  return (/* ... */);
}

// GOOD: High cohesion - related concerns together
function UserProfile() {
  const user = useUser();
  const userSettings = useUserSettings();
  const userStats = useUserStats();

  return (/* ... */);
}
```

```javascript
// BAD: Tight coupling - child depends on parent's internals
function Parent() {
  const [user, setUser] = useState(null);
  const [posts, setPosts] = useState([]);

  return <Child user={user} posts={posts} setUser={setUser} />;
}

function Child({ user, posts, setUser }) {
  // Child knows about parent's state structure
  const handleUpdate = () => {
    setUser({ ...user, name: 'New Name' }); // Brittle!
  };
}

// GOOD: Loose coupling - child emits events, parent handles state
function Parent() {
  const [user, setUser] = useState(null);
  const [posts, setPosts] = useState([]);

  const handleUserUpdate = (updates) => {
    setUser(current => ({ ...current, ...updates }));
  };

  return <Child user={user} posts={posts} onUserUpdate={handleUserUpdate} />;
}

function Child({ user, posts, onUserUpdate }) {
  // Child doesn't know how parent manages state
  const handleUpdate = () => {
    onUserUpdate({ name: 'New Name' });
  };
}
```

### Measuring Coupling

Ask these questions:
1. Can I change this component without changing others?
2. Can I test this component in isolation?
3. Can I understand this component without reading others?
4. Can I reuse this component elsewhere?

If the answer is "no" to any, you have tight coupling.

### Hands-On Exercise 7.3

You have a tightly coupled component tree:
```
App (has all state)
├── Sidebar (accesses App's state directly)
│   └── Navigation (accesses Sidebar's props)
│       └── NavItem (accesses Navigation's props)
└── MainContent (accesses App's state directly)
    └── PostList (accesses MainContent's props)
        └── Post (accesses PostList's props)
```

Refactor to reduce coupling while maintaining cohesion.

## 7.4 Component Layering

### The Layered Architecture

```
┌─────────────────────────────────────┐
│         Pages (Routes)              │  - Route components
│         - /products                 │  - Compose features
│         - /checkout                 │
├─────────────────────────────────────┤
│         Features                    │  - Business features
│         - ProductCatalog            │  - Domain logic
│         - ShoppingCart              │  - Orchestration
├─────────────────────────────────────┤
│         Shared Components           │  - Reusable components
│         - Button, Input             │  - No business logic
│         - Modal, Dropdown           │  - Pure UI
├─────────────────────────────────────┤
│         Hooks & State               │  - Custom hooks
│         - useAuth, useCart          │  - State management
│         - useApi, useLocalStorage   │  - Side effects
├─────────────────────────────────────┤
│         Services & Utils            │  - API calls
│         - api.js                    │  - Utilities
│         - formatters.js             │  - Pure functions
└─────────────────────────────────────┘
```

### Rules of Layering

1. **Layers can only depend on layers below them**
   - Pages can use Features ✓
   - Features can use Shared Components ✓
   - Shared Components cannot use Features ✗

2. **No circular dependencies**
   - If A imports B, B cannot import A

3. **Each layer has a clear purpose**

### Example Structure

```
src/
├── pages/
│   ├── ProductsPage.jsx
│   └── CheckoutPage.jsx
│
├── features/
│   ├── products/
│   │   ├── ProductCatalog.jsx
│   │   ├── ProductFilters.jsx
│   │   ├── useProducts.js
│   │   └── productsApi.js
│   │
│   └── cart/
│       ├── ShoppingCart.jsx
│       ├── CartItem.jsx
│       ├── useCart.js
│       └── cartApi.js
│
├── components/
│   ├── ui/
│   │   ├── Button.jsx
│   │   ├── Input.jsx
│   │   └── Modal.jsx
│   │
│   └── layout/
│       ├── Header.jsx
│       └── Sidebar.jsx
│
├── hooks/
│   ├── useAuth.js
│   ├── useLocalStorage.js
│   └── useDebounce.js
│
└── services/
    ├── api.js
    └── analytics.js
```

### Hands-On Exercise 7.4

Given a flat component structure:
```
src/components/
├── App.jsx
├── ProductCard.jsx
├── ProductList.jsx
├── Cart.jsx
├── CartItem.jsx
├── Button.jsx
├── Modal.jsx
├── Header.jsx
└── ... (50 more files)
```

Reorganize into a proper layered architecture.

## 7.5 Feature-Based Organization

### From Technical to Feature-Based

```javascript
// Junior: Organized by technical type
src/
├── components/
│   ├── ProductCard.jsx
│   ├── ProductList.jsx
│   ├── CartButton.jsx
│   └── CheckoutForm.jsx
├── hooks/
│   ├── useProducts.js
│   ├── useCart.js
│   └── useCheckout.js
├── services/
│   ├── productService.js
│   ├── cartService.js
│   └── checkoutService.js
└── types/
    ├── product.ts
    ├── cart.ts
    └── checkout.ts

// Senior: Organized by feature (domain)
src/
├── features/
│   ├── products/
│   │   ├── components/
│   │   │   ├── ProductCard.jsx
│   │   │   └── ProductList.jsx
│   │   ├── hooks/
│   │   │   └── useProducts.js
│   │   ├── services/
│   │   │   └── productService.js
│   │   ├── types/
│   │   │   └── product.ts
│   │   └── index.js  // Public API
│   │
│   ├── cart/
│   │   ├── components/
│   │   │   ├── CartButton.jsx
│   │   │   └── CartItem.jsx
│   │   ├── hooks/
│   │   │   └── useCart.js
│   │   ├── services/
│   │   │   └── cartService.js
│   │   └── index.js
│   │
│   └── checkout/
│       ├── components/
│       │   └── CheckoutForm.jsx
│       ├── hooks/
│       │   └── useCheckout.js
│       └── index.js
│
└── shared/  // Truly shared across features
    ├── components/
    ├── hooks/
    └── utils/
```

### Benefits
- Easy to find related code
- Easy to delete features
- Clear boundaries
- Better for team scalability

### Public APIs for Features

```javascript
// features/products/index.js - Public API
export { ProductCatalog } from './components/ProductCatalog';
export { ProductCard } from './components/ProductCard';
export { useProducts } from './hooks/useProducts';
// Internal components NOT exported

// Usage in other features
import { ProductCard } from '@/features/products';
// NOT: import { ProductCard } from '@/features/products/components/ProductCard';
```

## 7.6 Dependency Inversion

### The Problem: Direct Dependencies

```javascript
// BAD: Direct coupling to implementation
function ProductList() {
  const [products, setProducts] = useState([]);

  useEffect(() => {
    // Directly coupled to fetch API
    fetch('/api/products')
      .then(r => r.json())
      .then(setProducts);
  }, []);

  return products.map(p => <ProductCard key={p.id} product={p} />);
}
```

### Solution: Depend on Abstractions

```javascript
// GOOD: Depend on abstraction (hook)
function ProductList() {
  const { products, loading } = useProducts();

  if (loading) return <Spinner />;
  return products.map(p => <ProductCard key={p.id} product={p} />);
}

// Implementation can change without affecting ProductList
function useProducts() {
  const [products, setProducts] = useState([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    // Can swap fetch for axios, GraphQL, etc.
    productService.getAll()
      .then(setProducts)
      .finally(() => setLoading(false));
  }, []);

  return { products, loading };
}
```

### Hands-On Exercise 7.6

Refactor a component that:
- Uses fetch directly
- Manages its own loading/error states
- Has validation logic inline
- Transforms data in the component

Create proper abstractions using hooks and services.

## Real-World Scenario: Refactoring a Monolithic Component

### The Challenge

You have a 2000-line `Dashboard.jsx` that:
- Fetches 5 different data sources
- Renders 20+ sub-components
- Has 15 useState calls
- Has mixed concerns (data, UI, business logic)
- Is impossible to test

### Your Task

Create a refactoring plan that:
1. Identifies responsibilities
2. Extracts custom hooks
3. Creates feature modules
4. Establishes clear boundaries
5. Makes it testable

Document your architectural decisions.

## Chapter Exercise: Build an E-Commerce Feature

Design and implement a product catalog feature with proper architecture:

**Requirements:**
1. Feature-based organization
2. Layered architecture
3. Clear separation of concerns
4. Proper coupling/cohesion
5. Public API for the feature
6. Testable components

**Evaluation:**
- Can features be extracted to separate packages?
- Can components be tested in isolation?
- Is business logic separate from UI?
- Are dependencies pointing downward?

## Review Checklist

- [ ] Apply single responsibility to components
- [ ] Choose appropriate patterns for component structure
- [ ] Measure and reduce coupling
- [ ] Organize code by feature/domain
- [ ] Create proper abstraction layers
- [ ] Design public APIs for modules
- [ ] Make architectural trade-off decisions

## Key Takeaways

1. **Architecture emerges from requirements** - Don't over-engineer early
2. **Coupling is the enemy** - Optimize for change
3. **Cohesion is your friend** - Keep related things together
4. **Layers create maintainability** - Clear boundaries reduce complexity
5. **Feature-based scales better** - Organize by domain, not technology
6. **Depend on abstractions** - Makes code flexible and testable

## Further Reading

- Clean Architecture by Robert C. Martin
- Domain-Driven Design by Eric Evans
- Bulletproof React - Code organization guide
- React folder structure best practices

## Next Chapter

[Chapter 8: State Management at Scale](./08-state-management.md)
