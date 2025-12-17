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

### Junior vs Senior Perspective

**Junior Approach:**
"I'll put all the logic in one big component - easier to find everything in one place! If it gets too big, I'll just split the JSX into smaller components and pass everything as props."

**Senior Approach:**
"I'll separate concerns from the start: custom hooks for data/logic, presentation components for UI, container components for composition. Each piece has one clear responsibility. When requirements change, I only touch one file."

**The Difference:**
- Junior: Organizes by file size (split when file gets big), couples everything together
- Senior: Organizes by responsibility (each piece does one thing), decouples from day one

**Key Insight:** Good architecture isn't about small files, it's about clear boundaries. A 200-line component with one responsibility is better than 5 components with tangled responsibilities.

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

### Visual: Single Responsibility in Action

```
BEFORE (God Component)          AFTER (Single Responsibilities)
┌────────────────────┐
│  UserDashboard     │          ┌────────────────┐
│  (650 lines)       │          │ useTheme()     │  ← Theme logic
│                    │          └────────────────┘
│ ├─ Theme logic     │
│ ├─ User fetching   │          ┌────────────────┐
│ ├─ Posts fetching  │          │ useUser()      │  ← User data
│ ├─ Analytics       │          └────────────────┘
│ ├─ UI rendering    │
│ └─ Event handlers  │          ┌────────────────┐
│                    │          │ usePosts()     │  ← Posts data
│ (One reason to     │          └────────────────┘
│  change? NO!       │
│  7 reasons!)       │          ┌────────────────┐
└────────────────────┘          │ DashboardHeader│  ← UI
                                 └────────────────┘
Change theme?
  → Edit 650-line file           ┌────────────────┐
Change posts API?                │ PostList       │  ← UI
  → Edit 650-line file           └────────────────┘
Add analytics?
  → Edit 650-line file           ┌────────────────┐
                                 │ UserDashboard  │  ← Orchestration
Testing?                         │ (80 lines)     │
  → Mock everything              └────────────────┘
  → 45 test setup lines
                                 Change theme?
                                   → Edit useTheme() (20 lines)
                                 Change posts API?
                                   → Edit usePosts() (40 lines)
                                 Add analytics?
                                   → New hook, zero changes elsewhere

                                 Testing?
                                   → Test each piece independently
                                   → 5 lines per test
```

**Key insight:** The number of lines doesn't matter. What matters is the number of reasons to change.

### Hands-On Exercise 7.1: Refactor a God Component

**Scenario:** You have a `ShoppingCart` component (650 lines) that handles:
- Product fetching from API
- Cart state management (add, remove, update quantities)
- Checkout process and payment
- Analytics tracking (page views, add-to-cart events, purchases)
- UI rendering (cart items, totals, checkout form)

**Task:** Refactor into components and hooks with single responsibilities.

<details>
<summary>💡 Hint 1: Identify responsibilities</summary>

Ask "what are the reasons this component might change?"

1. **Product data changes** (API endpoint changes)
2. **Cart logic changes** (new features like save-for-later)
3. **Checkout flow changes** (new payment methods)
4. **Analytics requirements change** (new tracking events)
5. **UI design changes** (new styles, layout)

Each reason to change = separate responsibility.

</details>

<details>
<summary>💡 Hint 2: Extract hooks first</summary>

Start with data and logic:
- `useProducts(ids)` - fetch product details
- `useCart()` - cart state and operations
- `useCheckout()` - checkout process
- `useAnalytics()` - tracking events

Then UI components.

</details>

<details>
<summary>✅ Solution</summary>

**Step 1: Extract custom hooks**

```javascript
// hooks/useProducts.js - Product data responsibility
function useProducts(productIds) {
  const [products, setProducts] = useState([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetch(`/api/products?ids=${productIds.join(',')}`)
      .then(r => r.json())
      .then(setProducts)
      .finally(() => setLoading(false));
  }, [productIds.join(',')]);

  return { products, loading };
}

// hooks/useCart.js - Cart state responsibility
function useCart() {
  const [items, setItems] = useState([]);

  const addItem = (product, quantity = 1) => {
    setItems(current => {
      const existing = current.find(item => item.id === product.id);
      if (existing) {
        return current.map(item =>
          item.id === product.id
            ? { ...item, quantity: item.quantity + quantity }
            : item
        );
      }
      return [...current, { ...product, quantity }];
    });
  };

  const removeItem = (productId) => {
    setItems(current => current.filter(item => item.id !== productId));
  };

  const updateQuantity = (productId, quantity) => {
    setItems(current =>
      current.map(item =>
        item.id === productId ? { ...item, quantity } : item
      )
    );
  };

  const total = items.reduce((sum, item) => sum + item.price * item.quantity, 0);

  return { items, addItem, removeItem, updateQuantity, total };
}

// hooks/useCheckout.js - Checkout process responsibility
function useCheckout() {
  const [isProcessing, setIsProcessing] = useState(false);
  const [error, setError] = useState(null);

  const processCheckout = async (cartItems, paymentInfo) => {
    setIsProcessing(true);
    setError(null);
    try {
      const response = await fetch('/api/checkout', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ items: cartItems, payment: paymentInfo })
      });
      if (!response.ok) throw new Error('Checkout failed');
      return await response.json();
    } catch (err) {
      setError(err.message);
      throw err;
    } finally {
      setIsProcessing(false);
    }
  };

  return { processCheckout, isProcessing, error };
}

// hooks/useAnalytics.js - Analytics responsibility
function useAnalytics() {
  const trackEvent = (eventName, data) => {
    // Send to analytics service
    fetch('/api/analytics', {
      method: 'POST',
      body: JSON.stringify({ event: eventName, data, timestamp: Date.now() })
    });
  };

  const trackPageView = (page) => trackEvent('page_view', { page });
  const trackAddToCart = (product) => trackEvent('add_to_cart', product);
  const trackPurchase = (total, items) => trackEvent('purchase', { total, items });

  return { trackPageView, trackAddToCart, trackPurchase };
}
```

**Step 2: Create presentation components**

```javascript
// components/CartItem.jsx - Single cart item UI
function CartItem({ item, onUpdateQuantity, onRemove }) {
  return (
    <div className="cart-item">
      <img src={item.image} alt={item.name} />
      <div>
        <h3>{item.name}</h3>
        <p>${item.price}</p>
      </div>
      <input
        type="number"
        value={item.quantity}
        onChange={(e) => onUpdateQuantity(item.id, Number(e.target.value))}
        min="1"
      />
      <button onClick={() => onRemove(item.id)}>Remove</button>
    </div>
  );
}

// components/CartSummary.jsx - Totals display
function CartSummary({ items, total }) {
  return (
    <div className="cart-summary">
      <h3>Order Summary</h3>
      <div>Items: {items.length}</div>
      <div>Subtotal: ${total.toFixed(2)}</div>
      <div>Tax: ${(total * 0.1).toFixed(2)}</div>
      <div><strong>Total: ${(total * 1.1).toFixed(2)}</strong></div>
    </div>
  );
}

// components/CheckoutForm.jsx - Payment form UI
function CheckoutForm({ onSubmit, isProcessing }) {
  const [payment, setPayment] = useState({ cardNumber: '', cvv: '', expiry: '' });

  const handleSubmit = (e) => {
    e.preventDefault();
    onSubmit(payment);
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        placeholder="Card Number"
        value={payment.cardNumber}
        onChange={(e) => setPayment({ ...payment, cardNumber: e.target.value })}
      />
      <input
        placeholder="CVV"
        value={payment.cvv}
        onChange={(e) => setPayment({ ...payment, cvv: e.target.value })}
      />
      <input
        placeholder="Expiry (MM/YY)"
        value={payment.expiry}
        onChange={(e) => setPayment({ ...payment, expiry: e.target.value })}
      />
      <button type="submit" disabled={isProcessing}>
        {isProcessing ? 'Processing...' : 'Complete Purchase'}
      </button>
    </form>
  );
}
```

**Step 3: Container component - orchestration only**

```javascript
// ShoppingCart.jsx - Now just 80 lines of orchestration
function ShoppingCart() {
  const { items, addItem, removeItem, updateQuantity, total } = useCart();
  const { products } = useProducts(items.map(item => item.id));
  const { processCheckout, isProcessing, error } = useCheckout();
  const { trackPageView, trackAddToCart, trackPurchase } = useAnalytics();

  useEffect(() => {
    trackPageView('shopping_cart');
  }, []);

  const handleCheckout = async (paymentInfo) => {
    try {
      await processCheckout(items, paymentInfo);
      trackPurchase(total, items);
      // Redirect to success page
    } catch (err) {
      console.error('Checkout failed:', err);
    }
  };

  return (
    <div className="shopping-cart">
      <h1>Shopping Cart</h1>

      {items.length === 0 ? (
        <p>Your cart is empty</p>
      ) : (
        <>
          <div className="cart-items">
            {items.map(item => (
              <CartItem
                key={item.id}
                item={item}
                onUpdateQuantity={updateQuantity}
                onRemove={removeItem}
              />
            ))}
          </div>

          <CartSummary items={items} total={total} />

          <CheckoutForm
            onSubmit={handleCheckout}
            isProcessing={isProcessing}
          />

          {error && <div className="error">{error}</div>}
        </>
      )}
    </div>
  );
}
```

**Result:**
- 650 lines → 80 lines in main component
- 4 custom hooks (testable independently)
- 3 presentation components (reusable)
- Each piece has **one reason to change**
- Easy to test, easy to modify

**Testing becomes trivial:**
```javascript
// Test cart logic without UI
test('useCart adds items correctly', () => {
  const { result } = renderHook(() => useCart());
  act(() => result.current.addItem({ id: 1, price: 10 }));
  expect(result.current.items).toHaveLength(1);
});

// Test UI without business logic
test('CartItem renders', () => {
  render(<CartItem item={mockItem} onRemove={jest.fn()} />);
  expect(screen.getByText('Remove')).toBeInTheDocument();
});
```

</details>



---

## 🧠 Quick Recall: Before Moving to 7.2

Before learning about container vs presentation patterns, test your retention:

**Question:** What does Single Responsibility Principle mean for React components, and how would you split a component that fetches data, manages state, AND renders UI?

<details>
<summary>✅ Answer</summary>

Each component should have **one reason to change**. Split by:
1. **Custom hooks** for data fetching and state management
2. **Presentation components** for UI rendering
3. **Container components** for orchestration

Example: `UserDashboard` → `useUser()` + `useTheme()` + `DashboardHeader` + `PostList`

This makes each piece easy to test, understand, and change independently.

</details>

---

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

---

## 🧠 Quick Recall: Before Moving to 7.3

Before diving into coupling and cohesion, test your understanding:

**Question:** What's the difference between a Container component and a Presentation component? When would you choose colocated pattern instead?

<details>
<summary>✅ Answer</summary>

**Container:** Handles data fetching, state, business logic. Passes data as props to presentation components.

**Presentation:** Receives data via props, renders UI. No data fetching, no business logic. Pure and testable.

**Modern colocated approach:** Combine both when:
- It's a one-off component (not reused)
- You don't need to test UI separately
- Custom hooks already abstract the complexity

Use container/presentation when building design systems or when UI needs multiple data sources.

</details>

---

## 7.3 Component Coupling and Cohesion

### Understanding Coupling and Cohesion

**Visual representation:**

```
HIGH COUPLING (BAD)              LOW COUPLING (GOOD)
┌──────────┐                     ┌──────────┐
│  Parent  │                     │  Parent  │
│ ┌──────┐ │                     └────┬─────┘
│ │State │ │                          │ event
│ │All   │ │                          ↓
│ │Data  │ │                     ┌──────────┐
│ └──┬───┘ │                     │  Child   │
│    │knows│                     │  (props  │
│    │state│                     │   only)  │
└────┼─────┘                     └──────────┘
     │setState
     ↓
┌──────────┐
│  Child   │
│ (coupled │
│ to parent│
│ internals│
└──────────┘


LOW COHESION (BAD)              HIGH COHESION (GOOD)
┌─────────────────┐             ┌──────────────┐
│ UserProfile     │             │ UserProfile  │
│ ├─ User data    │             │ ├─ User data │
│ ├─ Weather API  │             │ ├─ User prefs│
│ ├─ Stock prices │             │ └─ User stats│
│ └─ Random stuff │             └──────────────┘
└─────────────────┘             (All related!)
(Unrelated!)
```

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

### Hands-On Exercise 7.3: Reduce Coupling in Component Tree

**Scenario:** You have a tightly coupled component tree where every child depends on its parent's internal structure:

```
App (has all state: user, posts, sidebar, theme)
├── Sidebar (accesses App's state directly via props)
│   └── Navigation (knows Sidebar's props structure)
│       └── NavItem (knows Navigation's props structure)
└── MainContent (accesses App's state directly)
    └── PostList (knows MainContent's props)
        └── Post (knows PostList's props)
```

**Problems:**
- Changing App's state structure breaks 6 components
- Can't test Sidebar without mocking App
- Can't reuse Navigation elsewhere
- Prop drilling 3+ levels deep

**Task:** Refactor to reduce coupling while maintaining cohesion.

<details>
<summary>💡 Hint 1: Identify what each component really needs</summary>

Most components don't need parent's entire state:
- `Sidebar` needs: `isOpen`, `onToggle` (not entire App state)
- `Navigation` needs: `items[]`, `activeItem` (not Sidebar internals)
- `NavItem` needs: `label`, `href`, `isActive` (just its own data)

**Principle:** Components should receive only what they need, not everything the parent has.

</details>

<details>
<summary>💡 Hint 2: Use composition and Context for deep data</summary>

For data needed many levels deep:
- **Context** for truly global data (user, theme)
- **Composition** to avoid prop drilling
- **Custom hooks** to encapsulate business logic

</details>

<details>
<summary>✅ Solution</summary>

**Step 1: Extract Context for global data**

```javascript
// contexts/UserContext.js
const UserContext = createContext();

export function UserProvider({ children }) {
  const [user, setUser] = useState(null);

  useEffect(() => {
    fetch('/api/user')
      .then(r => r.json())
      .then(setUser);
  }, []);

  return (
    <UserContext.Provider value={{ user, setUser }}>
      {children}
    </UserContext.Provider>
  );
}

export function useUser() {
  const context = useContext(UserContext);
  if (!context) throw new Error('useUser must be within UserProvider');
  return context;
}

// contexts/ThemeContext.js
const ThemeContext = createContext();

export function ThemeProvider({ children }) {
  const [theme, setTheme] = useState('light');
  return (
    <ThemeContext.Provider value={{ theme, setTheme }}>
      {children}
    </ThemeContext.Provider>
  );
}

export function useTheme() {
  return useContext(ThemeContext);
}
```

**Step 2: Decouple components - make them self-sufficient**

```javascript
// components/NavItem.jsx - Knows nothing about parents
function NavItem({ label, href, isActive }) {
  return (
    <a
      href={href}
      className={isActive ? 'nav-item active' : 'nav-item'}
    >
      {label}
    </a>
  );
}

// components/Navigation.jsx - Knows nothing about Sidebar
function Navigation({ items, currentPath }) {
  return (
    <nav>
      {items.map(item => (
        <NavItem
          key={item.href}
          label={item.label}
          href={item.href}
          isActive={item.href === currentPath}
        />
      ))}
    </nav>
  );
}

// components/Sidebar.jsx - Self-contained state
function Sidebar() {
  const [isOpen, setIsOpen] = useState(false);
  const { user } = useUser(); // Get user from Context, not props
  const navItems = [
    { label: 'Dashboard', href: '/dashboard' },
    { label: 'Posts', href: '/posts' },
    { label: 'Settings', href: '/settings' }
  ];

  return (
    <aside className={isOpen ? 'sidebar open' : 'sidebar'}>
      <button onClick={() => setIsOpen(!isOpen)}>Toggle</button>
      <div className="user-info">
        {user?.name}
      </div>
      <Navigation items={navItems} currentPath={window.location.pathname} />
    </aside>
  );
}
```

**Step 3: Decouple Post components**

```javascript
// components/Post.jsx - Pure presentation
function Post({ post }) {
  const { user } = useUser(); // Get user from Context
  const canEdit = user?.id === post.authorId;

  return (
    <article>
      <h2>{post.title}</h2>
      <p>{post.content}</p>
      <span>By {post.author}</span>
      {canEdit && <button>Edit</button>}
    </article>
  );
}

// components/PostList.jsx - Handles its own data
function PostList() {
  const [posts, setPosts] = useState([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetch('/api/posts')
      .then(r => r.json())
      .then(setPosts)
      .finally(() => setLoading(false));
  }, []);

  if (loading) return <div>Loading...</div>;

  return (
    <div className="post-list">
      {posts.map(post => (
        <Post key={post.id} post={post} />
      ))}
    </div>
  );
}

// components/MainContent.jsx - Simple composition
function MainContent() {
  return (
    <main>
      <h1>Posts</h1>
      <PostList />
    </main>
  );
}
```

**Step 4: App becomes simple orchestration**

```javascript
// App.jsx - Just composition, minimal state
function App() {
  return (
    <UserProvider>
      <ThemeProvider>
        <div className="app">
          <Sidebar />
          <MainContent />
        </div>
      </ThemeProvider>
    </UserProvider>
  );
}
```

**Before vs After:**

| Before | After |
|--------|-------|
| App has all state (user, posts, sidebar, theme) | Context manages global state |
| Prop drilling 3+ levels | No prop drilling |
| Post needs MainContent → PostList → Post props | Post gets user from Context |
| NavItem coupled to Navigation to Sidebar to App | NavItem is pure, receives only needed props |
| Can't test Sidebar (needs App) | Can test Sidebar independently |
| Changing App state breaks 6 components | Changing Context only affects consumers |

**Testing before:**
```javascript
// Had to mock entire App structure
test('NavItem', () => {
  const mockApp = { /* all app state */ };
  const mockSidebar = { /* sidebar props */ };
  const mockNav = { /* navigation props */ };
  render(<NavItem {...mockNav} />); // Still coupled!
});
```

**Testing after:**
```javascript
// Pure, testable components
test('NavItem', () => {
  render(<NavItem label="Home" href="/" isActive={true} />);
  expect(screen.getByText('Home')).toHaveClass('active');
});

test('Sidebar', () => {
  render(
    <UserProvider>
      <Sidebar />
    </UserProvider>
  );
  expect(screen.getByText('Toggle')).toBeInTheDocument();
});
```

**Key principles applied:**
1. **Context for global data** - User, theme available everywhere without props
2. **Components own their state** - Sidebar manages `isOpen`, PostList fetches posts
3. **Minimal props** - Each component receives only what it needs
4. **Pure components** - NavItem, Post are pure (props in → UI out)
5. **No prop drilling** - Deep components get data from Context, not through 3 layers

Result: **Low coupling** (components independent), **high cohesion** (related logic together).

</details>



---

### 📊 Quick Knowledge Check

Test your understanding of coupling, cohesion, and separation of concerns:

**Question 1:** You have a `UserProfile` component with 800 lines that handles user data fetching, profile editing, password change, subscription management, and theme preferences. What architectural problem is this, and how would you fix it?

<details>
<summary>✅ Answer</summary>

**Problem:** Violates Single Responsibility Principle - multiple reasons to change (God component).

**Fix:** Split by domain responsibility:
1. `useUser()` - user data fetching
2. `useSubscription()` - subscription logic
3. `useTheme()` - theme preferences
4. `ProfileEditor` - editing UI
5. `PasswordForm` - password change UI
6. `SubscriptionPanel` - subscription UI
7. `UserProfile` - orchestration only (composes everything)

Each piece now has one reason to change.

</details>

**Question 2:** Your `Button` component accepts 23 props including `user`, `onUserUpdate`, `cart`, `onCartUpdate`, `theme`, `analytics`. What principle is violated and how does this affect reusability?

<details>
<summary>✅ Answer</summary>

**Problem:** Tight coupling + low cohesion. Button is coupled to user, cart, theme, and analytics systems. It's not reusable because it requires all these dependencies.

**Fix:** Button should only have UI props: `onClick`, `disabled`, `variant`, `children`. Move business logic to parent:
```javascript
// ❌ Tightly coupled
<Button user={user} cart={cart} onUserUpdate={...} />

// ✅ Loosely coupled
<Button onClick={handleClick} variant="primary">
  Add to Cart
</Button>
```

Parent handles business logic, Button handles UI only.

</details>

**Question 3 (Interleaved):** You're building a product catalog feature. You have `ProductCard`, `ProductList`, `useProducts` hook, and `productService` API. A junior dev proposes organizing it like this:

```
src/
├── components/ProductCard.jsx, ProductList.jsx
├── hooks/useProducts.js
└── services/productService.js
```

What's wrong with this approach as the app grows to 15+ features? How would you organize it using principles from 7.1-7.3?

<details>
<summary>✅ Answer</summary>

**Problem:** File-type organization breaks down at scale:
- Hard to find related code (4 folders to understand products)
- Hard to delete feature (files scattered)
- Coupling unclear (which components use useProducts?)
- Low cohesion (all components mixed together)

**Better approach - Feature-based organization:**
```
src/features/products/
├── components/
│   ├── ProductCard.jsx
│   └── ProductList.jsx
├── hooks/useProducts.js
├── services/productService.js
└── index.js  // Public API
```

**Applies principles:**
- **SRP (7.1):** Each folder has one domain responsibility
- **Cohesion (7.3):** Related code together
- **Low coupling (7.3):** Public API (`index.js`) controls what other features can access

This scales to 100+ features. All product code in one place.

</details>

---

## 🧠 Quick Recall: Before Moving to 7.4

Before learning about layering, test your memory:

**Question:** What's the difference between coupling and cohesion? Give an example of code with high coupling and low cohesion.

<details>
<summary>✅ Answer</summary>

**Coupling:** How much components depend on each other. **Lower is better.**
- High coupling: Child calls parent's `setState` directly, knows parent's state structure

**Cohesion:** How related things within a component are. **Higher is better.**
- Low cohesion: `UserProfile` component also fetching weather and stock prices

**Example of both problems:**
```javascript
// High coupling + Low cohesion
function UserProfile() {
  const [user, setUser] = useState(null);
  const [weather, setWeather] = useState(null); // Unrelated!
  return <Child setUser={setUser} />; // Coupling!
}
```

**Fix:** High cohesion (group related things) + Low coupling (use events, not state manipulation)

</details>

---

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

### Hands-On Exercise 7.4: Reorganize Flat Structure into Layers

**Scenario:** You have a flat component structure with 50+ files in one folder:

```
src/components/
├── App.jsx
├── ProductCard.jsx
├── ProductList.jsx
├── ProductFilters.jsx
├── Cart.jsx
├── CartItem.jsx
├── CartSummary.jsx
├── CheckoutForm.jsx
├── Button.jsx
├── Input.jsx
├── Modal.jsx
├── Dropdown.jsx
├── Header.jsx
├── Footer.jsx
├── Sidebar.jsx
├── UserProfile.jsx
├── LoginForm.jsx
├── ... (35 more files)
```

**Problems:**
- Can't tell which components are reusable vs feature-specific
- No clear dependencies (Button imports Cart imports Product?)
- Hard to find related components
- Circular dependencies possible

**Task:** Reorganize into a proper layered architecture with clear dependency rules.

<details>
<summary>💡 Hint: Identify component categories</summary>

Ask "what is this component's purpose?"

1. **Pages** - Route components (`HomePage`, `ProductsPage`, `CheckoutPage`)
2. **Features** - Business features (`ProductCatalog`, `ShoppingCart`, `UserAuth`)
3. **Shared UI** - Reusable components (`Button`, `Modal`, `Input`)
4. **Layout** - Page structure (`Header`, `Footer`, `Sidebar`)
5. **Hooks** - Custom hooks (`useCart`, `useAuth`, `useLocalStorage`)
6. **Services** - API/business logic (`api.js`, `analytics.js`)

</details>

<details>
<summary>✅ Solution</summary>

**Step 1: Analyze and categorize existing components**

```
Pages (routes):
- App.jsx → HomePage
- (Create) ProductsPage, CheckoutPage

Features (domain-specific):
Products:
- ProductCard.jsx
- ProductList.jsx
- ProductFilters.jsx

Cart:
- Cart.jsx
- CartItem.jsx
- CartSummary.jsx
- CheckoutForm.jsx

User:
- UserProfile.jsx
- LoginForm.jsx

Shared Components (reusable UI):
- Button.jsx
- Input.jsx
- Modal.jsx
- Dropdown.jsx

Layout:
- Header.jsx
- Footer.jsx
- Sidebar.jsx

Hooks (to be created):
- useProducts.js
- useCart.js
- useAuth.js

Services (to be created):
- productService.js
- cartService.js
- authService.js
```

**Step 2: Create new layered structure**

```
src/
├── pages/
│   ├── HomePage.jsx
│   ├── ProductsPage.jsx
│   └── CheckoutPage.jsx
│
├── features/
│   ├── products/
│   │   ├── components/
│   │   │   ├── ProductCard.jsx
│   │   │   ├── ProductList.jsx
│   │   │   └── ProductFilters.jsx
│   │   ├── hooks/
│   │   │   └── useProducts.js
│   │   ├── services/
│   │   │   └── productService.js
│   │   └── index.js  // Public API
│   │
│   ├── cart/
│   │   ├── components/
│   │   │   ├── Cart.jsx
│   │   │   ├── CartItem.jsx
│   │   │   ├── CartSummary.jsx
│   │   │   └── CheckoutForm.jsx
│   │   ├── hooks/
│   │   │   └── useCart.js
│   │   ├── services/
│   │   │   └── cartService.js
│   │   └── index.js
│   │
│   └── user/
│       ├── components/
│       │   ├── UserProfile.jsx
│       │   └── LoginForm.jsx
│       ├── hooks/
│       │   └── useAuth.js
│       ├── services/
│       │   └── authService.js
│       └── index.js
│
├── components/
│   ├── ui/
│   │   ├── Button.jsx
│   │   ├── Input.jsx
│   │   ├── Modal.jsx
│   │   └── Dropdown.jsx
│   │
│   └── layout/
│       ├── Header.jsx
│       ├── Footer.jsx
│       └── Sidebar.jsx
│
├── hooks/
│   ├── useLocalStorage.js
│   ├── useDebounce.js
│   └── useMediaQuery.js
│
└── services/
    ├── api.js
    └── analytics.js
```

**Step 3: Define dependency rules**

```
┌──────────────────────────────┐
│  Pages (Route components)    │  ← Can import from Features, Components
├──────────────────────────────┤
│  Features (Business logic)   │  ← Can import from Components, Hooks, Services
├──────────────────────────────┤
│  Components (Shared UI)      │  ← Can import from Hooks (but NOT Features)
├──────────────────────────────┤
│  Hooks (Reusable logic)      │  ← Can import from Services
├──────────────────────────────┤
│  Services (API, utilities)   │  ← No dependencies (pure functions)
└──────────────────────────────┘

Dependencies flow DOWNWARD only. No circular dependencies.
```

**Step 4: Create public APIs for features**

```javascript
// features/products/index.js
export { ProductCard } from './components/ProductCard';
export { ProductList } from './components/ProductList';
export { ProductFilters } from './components/ProductFilters';
export { useProducts } from './hooks/useProducts';
// Internal components NOT exported

// features/cart/index.js
export { Cart } from './components/Cart';
export { useCart } from './hooks/useCart';
// CartItem is internal, not exported

// Usage in pages:
import { ProductList } from '@/features/products';
import { Cart } from '@/features/cart';
// NOT: import { ProductList } from '@/features/products/components/ProductList';
```

**Step 5: Example page composition**

```javascript
// pages/ProductsPage.jsx - Composes features
import { ProductList, ProductFilters } from '@/features/products';
import { Cart } from '@/features/cart';
import { Header, Footer } from '@/components/layout';

function ProductsPage() {
  return (
    <>
      <Header />
      <main>
        <aside>
          <ProductFilters />
        </aside>
        <section>
          <ProductList />
        </section>
        <aside>
          <Cart />
        </aside>
      </main>
      <Footer />
    </>
  );
}
```

**Benefits of layered structure:**

| Before (Flat) | After (Layered) |
|---------------|-----------------|
| 50 files in one folder | ~10 files per folder (easy to navigate) |
| No clear dependencies | Clear dependency rules (downward only) |
| Circular dependencies possible | Impossible by design |
| Can't tell what's reusable | Shared components clearly identified |
| Features mixed with UI | Features separated, self-contained |
| Hard to delete features | Delete entire `features/xyz` folder |
| Button might import Cart (wrong!) | UI components can't import features |
| New devs confused | Clear structure, obvious where to add code |

**Example of prevented bad dependency:**

```javascript
// ❌ BEFORE: Flat structure allowed this mistake
// components/Button.jsx
import { useCart } from './Cart'; // Button depends on Cart!?

function Button() {
  const { itemCount } = useCart();
  return <button>Cart ({itemCount})</button>;
}
```

```javascript
// ✅ AFTER: Layered structure prevents this
// components/ui/Button.jsx
import { useCart } from '@/features/cart'; // ❌ ERROR: UI can't import features!

// Correct approach: Make Button pure
function Button({ children, onClick }) {
  return <button onClick={onClick}>{children}</button>;
}

// Parent (feature) handles business logic
function CartButton() {
  const { itemCount } = useCart();
  return <Button>Cart ({itemCount})</Button>;
}
```

**Migration steps:**

1. Create new folder structure
2. Move files to appropriate locations
3. Update imports (use absolute imports with `@/`)
4. Create feature `index.js` public APIs
5. Add ESLint rule to prevent layer violations:
   ```javascript
   // .eslintrc.js
   'no-restricted-imports': ['error', {
     patterns: [
       {
         group: ['@/features/*'],
         importNames: ['*'],
         message: 'UI components cannot import features. Pass data as props.'
       }
     ]
   }]
   ```

**Result:** Clear architecture that scales to 100+ components with no confusion about dependencies.

</details>

---

## 🧠 Quick Recall: Before Moving to 7.5

Before exploring feature-based organization, recall:

**Question:** In a layered architecture, why can't Shared Components import from Features? What's the rule for dependencies between layers?

<details>
<summary>✅ Answer</summary>

**Rule:** Layers can **only depend on layers below them**. Dependencies flow downward.

```
Pages → Features → Shared → Hooks → Services
  ↓        ↓          ↓        ↓        ✓
 Can use  Can use   Can use  Can use  No deps
```

**Why can't Shared Components import Features?**
- Shared Components are **below** Features in the hierarchy
- If shared components depend on features, you get **circular dependencies**
- Shared components become **not actually reusable** (tied to specific features)

Example violation:
```javascript
// ❌ BAD: Button (shared) importing from features
import { useCart } from '@/features/cart';

function Button() {
  const { itemCount } = useCart(); // Coupling!
}
```

This Button is no longer "shared" - it only works with cart feature.

</details>

---

## 7.5 Feature-Based Organization

### The Evolution: From Chaos to Clarity

```
FLAT STRUCTURE (0-20 components)
src/components/
├── App.jsx
├── Button.jsx
├── ProductCard.jsx
├── Cart.jsx
└── ... (works fine!)


TECHNICAL ORGANIZATION (20-50 components)
src/
├── components/     ← 25 files (mixed purposes)
├── hooks/          ← 8 files (which component uses which?)
├── services/       ← 5 files (hard to find related code)
└── types/          ← 12 files (scattered domain logic)

Problem: Related code scattered across 4 folders


FEATURE-BASED (50+ components, scales to 500+)
src/
├── features/
│   ├── products/       ← Everything product-related
│   │   ├── components/
│   │   ├── hooks/
│   │   ├── services/
│   │   └── index.js   ← Public API
│   │
│   ├── cart/           ← Everything cart-related
│   │   ├── components/
│   │   ├── hooks/
│   │   └── index.js
│   │
│   └── checkout/       ← Everything checkout-related
│       ├── components/
│       └── index.js
│
└── shared/            ← Truly reusable across features
    ├── components/
    └── hooks/

Benefits:
✓ All related code in one place
✓ Easy to delete entire feature
✓ Teams can own features
✓ Clear boundaries (public APIs)
✓ Scales to 100+ features
```

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

---

## 🧠 Senior Think-Aloud: Architecting a New Feature

**Scenario:** Product manager says "We need to add a wishlist feature. Users can save products, create multiple lists, and share them with friends."

**Watch a senior developer think through the architecture:**

### Initial Analysis (First 5 minutes)

"Okay, wishlist feature. Let me think through the scope before writing any code..."

**Questions I'm asking myself:**

1. **Is this a new feature or modification?**
   - New feature → Create `features/wishlist/`
   - Modifies existing → Add to `features/products/` or `features/user/`?

2. **What are the responsibilities?**
   - Wishlist data management (CRUD operations)
   - Multiple lists per user
   - Sharing functionality
   - UI for displaying/managing lists

3. **What are the dependencies?**
   - Needs user authentication (`features/user/`)
   - Needs product data (`features/products/`)
   - Might need sharing/social features (new?)

4. **How will this scale?**
   - 10 products per list? Easy.
   - 1000 products per list? Need virtualization.
   - Real-time sync across devices? Need WebSocket.

### Decision Process

**Decision 1: Where does this live?**

```
Option A: features/user/wishlist/
  ✗ Wishlist isn't really a "user" concern
  ✗ User feature is about authentication/profile

Option B: features/products/wishlist/
  ✗ Wishlist isn't a product concern
  ✗ Products should be about catalog/search

Option C: features/wishlist/  ✓
  ✓ It's a distinct business domain
  ✓ Will likely grow (multiple lists, sharing, recommendations)
  ✓ Team can own this feature independently
```

**Decision: Create `features/wishlist/`**

**Decision 2: What's the component structure?**

"Let me think about responsibilities..."

```javascript
// Data & Logic Layer (hooks + services)
- useWishlist()          → Fetch user's wishlists
- useWishlistItems()     → Fetch items in a specific list
- useWishlistMutations() → Add/remove items, create/delete lists
- wishlistService.js     → API calls

// UI Layer (components)
- WishlistPage.jsx           → Route component (orchestration)
- WishlistList.jsx           → Shows all user's lists
- WishlistDetail.jsx         → Shows items in one list
- WishlistItem.jsx           → Individual product in list
- CreateWishlistModal.jsx    → Modal for creating new list
- ShareWishlistDialog.jsx    → Sharing functionality
```

"Wait, should WishlistItem use ProductCard from `features/products`?"

```javascript
// Option A: Reuse ProductCard
import { ProductCard } from '@/features/products';

function WishlistItem({ product }) {
  return <ProductCard product={product} />;
}
// ✗ Problem: ProductCard might have "Add to Cart" button
// ✗ Wishlist needs different actions (remove from list, move to another list)
// ✗ Coupling: Wishlist depends on Products feature internals

// Option B: Compose with shared components
import { Button } from '@/components/ui';
import { ProductImage } from '@/features/products'; // If exported

function WishlistItem({ product, onRemove, onMove }) {
  return (
    <div className="wishlist-item">
      <ProductImage src={product.image} alt={product.name} />
      <h3>{product.name}</h3>
      <p>${product.price}</p>
      <Button onClick={onRemove}>Remove</Button>
      <Button onClick={onMove}>Move to...</Button>
    </div>
  );
}
// ✓ Wishlist-specific UI
// ✓ No coupling to Products feature
// ✓ Can reuse ProductImage if Products exports it
```

**Decision: Create wishlist-specific components, reuse shared UI components only**

**Decision 3: How to handle product data?**

"Wishlist needs product details (name, price, image). Do I fetch from wishlist API or products API?"

```javascript
// Option A: Wishlist API returns full product data
// GET /api/wishlist/123
// Response: { id: 123, items: [{ product: { id, name, price, image } }] }
// ✓ One request
// ✗ Product data might be stale
// ✗ If product price changes, wishlist shows old price

// Option B: Wishlist API returns product IDs, fetch details from products API
// GET /api/wishlist/123 → { id: 123, items: [{ productId: 1 }, { productId: 2 }] }
// GET /api/products?ids=1,2 → [{ id: 1, name, price }, { id: 2, name, price }]
// ✓ Always fresh product data
// ✗ Two requests
// ✗ More complex state management

// Option C: Wishlist returns product IDs, use existing useProducts() hook
function WishlistDetail({ listId }) {
  const { items } = useWishlistItems(listId); // [{ productId: 1 }, { productId: 2 }]
  const productIds = items.map(item => item.productId);
  const { products } = useProducts(productIds); // Reuse existing hook!

  // Merge: items + products
  const enrichedItems = items.map(item => ({
    ...item,
    product: products.find(p => p.id === item.productId)
  }));

  return <WishlistItems items={enrichedItems} />;
}
// ✓ Reuses existing useProducts hook
// ✓ Fresh product data
// ✓ Leverages any caching in useProducts
// ✗ Slightly more complex
```

**Decision: Wishlist API returns IDs, reuse `useProducts()` for details**

**Decision 4: Public API - what do we export?**

```javascript
// features/wishlist/index.js

// What should other features access?

// ✓ Export: Main components for pages
export { WishlistPage } from './components/WishlistPage';
export { WishlistButton } from './components/WishlistButton'; // "Add to Wishlist" button

// ✓ Export: Hooks for data
export { useWishlist } from './hooks/useWishlist';

// ✗ Don't export: Internal components
// WishlistItem, WishlistDetail are internal implementation details

// Usage in ProductCard (products feature):
import { WishlistButton } from '@/features/wishlist';

function ProductCard({ product }) {
  return (
    <div>
      <h3>{product.name}</h3>
      <WishlistButton productId={product.id} />  {/* Simple! */}
    </div>
  );
}
```

### Final Architecture

```
features/wishlist/
├── components/
│   ├── WishlistPage.jsx           (route component)
│   ├── WishlistList.jsx           (internal)
│   ├── WishlistDetail.jsx         (internal)
│   ├── WishlistItem.jsx           (internal)
│   ├── WishlistButton.jsx         (public - other features use this)
│   ├── CreateWishlistModal.jsx    (internal)
│   └── ShareWishlistDialog.jsx    (internal)
├── hooks/
│   ├── useWishlist.js
│   ├── useWishlistItems.js
│   └── useWishlistMutations.js
├── services/
│   └── wishlistService.js
└── index.js  // Public API: WishlistPage, WishlistButton, useWishlist
```

### Key Decisions Summary

| Decision | Rationale |
|----------|-----------|
| Create `features/wishlist/` | Distinct domain, will grow, team ownership |
| Wishlist-specific components | Avoid coupling to Products feature |
| Reuse `useProducts()` hook | Fresh data, leverage existing caching |
| Export WishlistButton | Other features need "Add to Wishlist" functionality |
| Keep most components internal | Prevents coupling, freedom to refactor |

### What I avoided

❌ Putting wishlist code in `features/products/` - Wrong domain
❌ Reusing ProductCard directly - Creates coupling
❌ Fetching all product data in wishlist API - Stale data problem
❌ Exporting every component - Creates coupling
❌ Starting with code - Thought through architecture first

### Time spent

- Analysis: 5 minutes
- Decision making: 10 minutes
- Writing architecture doc: 5 minutes
- **Total: 20 minutes before writing any code**

**Result:** Clear architecture, no rework needed. Junior might spend 2 hours coding, then 4 hours refactoring. Senior spends 20 minutes planning, 2 hours coding, zero refactoring.

**Senior principle:** "Measure twice, cut once." Think through architecture before coding.

---

## 🧠 Quick Recall: Before Moving to 7.6

Before learning about dependency inversion, test yourself:

**Question:** What are the benefits of feature-based organization compared to organizing by file type (components/, hooks/, services/)? When does file-type organization break down?

<details>
<summary>✅ Answer</summary>

**Feature-based benefits:**
1. **Easy to find related code** - All cart code in `features/cart/`
2. **Easy to delete features** - Delete entire folder
3. **Clear boundaries** - Features don't depend on each other
4. **Team scalability** - Teams can own features independently
5. **Better for large codebases** - Scales to hundreds of components

**When file-type breaks down:**
- Around 50+ components, everything related is scattered
- Hard to understand dependencies (is `useProducts` used by cart? checkout?)
- Hard to delete features (files scattered across components/, hooks/, services/)
- Team conflicts (everyone editing components/ folder)

**Example pain:**
```
// File-type: To understand cart, read 8 scattered files
components/CartButton.jsx
components/CartItem.jsx
hooks/useCart.js
services/cartService.js
types/cart.ts

// Feature-based: Everything in one place
features/cart/*
```

</details>

---

## 7.6 Dependency Inversion

### Understanding Dependency Direction

```
TIGHT COUPLING (Direct Dependencies)
┌─────────────────────────────────────┐
│ ProductList Component               │
│                                     │
│  directly depends on ↓              │
│  ┌─────────────────┐                │
│  │ fetch() API     │  ← concrete!   │
│  └─────────────────┘                │
│  ┌─────────────────┐                │
│  │ JSON parsing    │  ← concrete!   │
│  └─────────────────┘                │
│  ┌─────────────────┐                │
│  │ Error handling  │  ← concrete!   │
│  └─────────────────┘                │
└─────────────────────────────────────┘

Problem: Can't swap implementations
Can't test without real HTTP
Changes to fetch affect component


LOOSE COUPLING (Dependency Inversion)
┌─────────────────────────────────────┐
│ ProductList Component               │
│                                     │
│  depends on ↓                       │
│  ┌─────────────────┐                │
│  │ useProducts()   │  ← abstraction!│
│  │   (hook)        │                │
│  └────────┬────────┘                │
└───────────┼─────────────────────────┘
            │ depends on
            ↓
    ┌───────────────┐
    │ productService│  ← abstraction!
    └───────┬───────┘
            │ implementation can be:
            ├→ FetchService
            ├→ AxiosService
            ├→ GraphQLService
            └→ MockService (for tests)

Benefit: Component unchanged when implementation changes
Easy to test (mock useProducts hook)
Can swap fetch → axios → GraphQL without touching component
```

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

### Hands-On Exercise 7.6: Apply Dependency Inversion

**Scenario:** You have a `UserList` component that violates dependency inversion by depending on concrete implementations:

```javascript
function UserList() {
  const [users, setUsers] = useState([]);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);

  useEffect(() => {
    setLoading(true);
    // Directly coupled to fetch API
    fetch('/api/users')
      .then(response => {
        if (!response.ok) throw new Error('Failed to fetch');
        return response.json();
      })
      .then(data => {
        // Data transformation inline
        const transformed = data.map(user => ({
          id: user.user_id,
          name: `${user.first_name} ${user.last_name}`,
          email: user.email_address,
          status: user.is_active ? 'Active' : 'Inactive'
        }));

        // Validation inline
        const valid = transformed.filter(user => {
          return user.email && user.email.includes('@') && user.name.length > 0;
        });

        setUsers(valid);
      })
      .catch(err => setError(err.message))
      .finally(() => setLoading(false));
  }, []);

  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error}</div>;

  return (
    <ul>
      {users.map(user => (
        <li key={user.id}>{user.name} - {user.email}</li>
      ))}
    </ul>
  );
}
```

**Problems:**
- Tightly coupled to `fetch` (can't swap for axios, GraphQL, mock data)
- Can't test component without making real HTTP requests
- Can't reuse data fetching, transformation, or validation logic
- Hard to change API structure (transformation scattered in component)

**Task:** Apply dependency inversion - make the component depend on abstractions (hooks/services), not concrete implementations.

<details>
<summary>💡 Hint 1: Create layers of abstraction</summary>

**Abstraction layers from bottom to top:**

1. **Service layer** - API calls (pure functions, no React)
   - `userService.js` - `fetchUsers()`, `createUser()`, etc.

2. **Transform/validation layer** - Business logic (pure functions)
   - `userTransforms.js` - `transformUserData()`, `validateUser()`

3. **Hook layer** - React integration (state + service)
   - `useUsers.js` - Calls service, manages state, returns data

4. **Component layer** - UI only (no business logic)
   - `UserList.jsx` - Calls hook, renders UI

Each layer depends on the abstraction below it, not concrete details.

</details>

<details>
<summary>💡 Hint 2: Identify what can be tested separately</summary>

Ask "what needs to be tested?"
- ✅ API calls (service) → test with mock fetch
- ✅ Data transformation → test with sample data
- ✅ Validation rules → test with valid/invalid users
- ✅ Loading/error states (hook) → test with renderHook
- ✅ UI rendering (component) → test with mock hook data

Each piece should be testable **independently**.

</details>

<details>
<summary>✅ Solution</summary>

**Step 1: Create service layer (API abstraction)**

```javascript
// services/userService.js - Pure functions, no React
export const userService = {
  async fetchUsers() {
    const response = await fetch('/api/users');
    if (!response.ok) {
      throw new Error(`HTTP ${response.status}: Failed to fetch users`);
    }
    return response.json();
  },

  async createUser(userData) {
    const response = await fetch('/api/users', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(userData)
    });
    if (!response.ok) throw new Error('Failed to create user');
    return response.json();
  },

  async deleteUser(userId) {
    const response = await fetch(`/api/users/${userId}`, {
      method: 'DELETE'
    });
    if (!response.ok) throw new Error('Failed to delete user');
  }
};

// Easy to swap implementations:
// export const userService = new AxiosUserService();
// export const userService = new GraphQLUserService();
// export const userService = new MockUserService(); // for tests
```

**Step 2: Create business logic layer (pure functions)**

```javascript
// utils/userTransforms.js - Pure functions, easily testable
export function transformUserData(apiUser) {
  return {
    id: apiUser.user_id,
    name: `${apiUser.first_name} ${apiUser.last_name}`,
    email: apiUser.email_address,
    status: apiUser.is_active ? 'Active' : 'Inactive'
  };
}

export function validateUser(user) {
  if (!user.email || !user.email.includes('@')) {
    return { valid: false, error: 'Invalid email' };
  }
  if (!user.name || user.name.length === 0) {
    return { valid: false, error: 'Name required' };
  }
  return { valid: true };
}

export function processUsers(apiUsers) {
  return apiUsers
    .map(transformUserData)
    .filter(user => validateUser(user).valid);
}
```

**Step 3: Create hook layer (React integration)**

```javascript
// hooks/useUsers.js - Manages state, calls service
import { useState, useEffect } from 'react';
import { userService } from '../services/userService';
import { processUsers } from '../utils/userTransforms';

export function useUsers() {
  const [users, setUsers] = useState([]);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);

  useEffect(() => {
    let cancelled = false;

    async function loadUsers() {
      setLoading(true);
      setError(null);
      try {
        const data = await userService.fetchUsers();
        if (!cancelled) {
          const processed = processUsers(data);
          setUsers(processed);
        }
      } catch (err) {
        if (!cancelled) {
          setError(err.message);
        }
      } finally {
        if (!cancelled) {
          setLoading(false);
        }
      }
    }

    loadUsers();

    return () => {
      cancelled = true; // Cleanup to prevent state updates after unmount
    };
  }, []);

  const createUser = async (userData) => {
    try {
      await userService.createUser(userData);
      // Refresh list
      const data = await userService.fetchUsers();
      setUsers(processUsers(data));
    } catch (err) {
      setError(err.message);
      throw err;
    }
  };

  return { users, loading, error, createUser };
}
```

**Step 4: Component layer (UI only)**

```javascript
// components/UserList.jsx - Pure UI, no business logic
import { useUsers } from '../hooks/useUsers';

function UserList() {
  const { users, loading, error } = useUsers();

  if (loading) return <LoadingSpinner />;
  if (error) return <ErrorMessage message={error} />;
  if (users.length === 0) return <EmptyState message="No users found" />;

  return (
    <ul className="user-list">
      {users.map(user => (
        <UserListItem key={user.id} user={user} />
      ))}
    </ul>
  );
}

function UserListItem({ user }) {
  return (
    <li className="user-item">
      <span className="user-name">{user.name}</span>
      <span className="user-email">{user.email}</span>
      <span className={`user-status ${user.status.toLowerCase()}`}>
        {user.status}
      </span>
    </li>
  );
}
```

**Benefits - Each layer is now independently testable:**

```javascript
// Test service (no React needed)
test('userService.fetchUsers', async () => {
  global.fetch = jest.fn(() =>
    Promise.resolve({
      ok: true,
      json: () => Promise.resolve([{ user_id: 1, first_name: 'John' }])
    })
  );
  const users = await userService.fetchUsers();
  expect(users).toHaveLength(1);
});

// Test business logic (pure functions)
test('transformUserData', () => {
  const apiUser = {
    user_id: 1,
    first_name: 'John',
    last_name: 'Doe',
    email_address: 'john@example.com',
    is_active: true
  };
  const result = transformUserData(apiUser);
  expect(result).toEqual({
    id: 1,
    name: 'John Doe',
    email: 'john@example.com',
    status: 'Active'
  });
});

test('validateUser', () => {
  expect(validateUser({ name: 'John', email: 'invalid' })).toEqual({
    valid: false,
    error: 'Invalid email'
  });
  expect(validateUser({ name: 'John', email: 'john@example.com' })).toEqual({
    valid: true
  });
});

// Test hook (with mock service)
test('useUsers', async () => {
  jest.spyOn(userService, 'fetchUsers').mockResolvedValue([
    { user_id: 1, first_name: 'John', last_name: 'Doe', email_address: 'john@example.com', is_active: true }
  ]);

  const { result, waitForNextUpdate } = renderHook(() => useUsers());
  expect(result.current.loading).toBe(true);

  await waitForNextUpdate();
  expect(result.current.loading).toBe(false);
  expect(result.current.users).toHaveLength(1);
  expect(result.current.users[0].name).toBe('John Doe');
});

// Test component (with mock hook)
test('UserList', () => {
  jest.mock('../hooks/useUsers');
  useUsers.mockReturnValue({
    users: [{ id: 1, name: 'John Doe', email: 'john@example.com', status: 'Active' }],
    loading: false,
    error: null
  });

  render(<UserList />);
  expect(screen.getByText('John Doe')).toBeInTheDocument();
});
```

**Dependency Inversion achieved:**

| Before | After |
|--------|-------|
| Component depends on `fetch` | Component depends on `useUsers` hook (abstraction) |
| Can't test without real HTTP | Each layer tests independently |
| Can't swap `fetch` for axios | Swap `userService` implementation easily |
| Transformation logic in component | Pure `transformUserData` function |
| Validation inline in useEffect | Pure `validateUser` function |
| 60-line component | 15-line component (just UI) |

**Now you can easily:**
- Swap `fetch` for `axios`: Change `userService` only
- Add GraphQL: Create `GraphQLUserService` implementing same interface
- Mock for tests: Use `MockUserService`
- Change transformation: Edit `userTransforms.js` only
- Reuse validation: Import `validateUser` anywhere
- Add caching: Modify `useUsers` hook, component unchanged

**Key principle:** High-level components (UI) depend on abstractions (hooks), not concrete implementations (fetch, axios). This makes code flexible and testable.

</details>



---

## 💥 War Story: The 8,000-Line God Component

### The Disaster

**Company:** SaaS platform (18M users, 120 engineers)
**Team:** 15 frontend engineers working on "Workspace" feature
**Date:** Q2 2023 (feature freeze lasted 6 weeks)
**Impact:** $1.8M in delayed revenue, 6 engineers quit, complete rewrite required

### What Happened

The company's flagship "Workspace" feature was their competitive advantage. One massive file controlled everything:

```
Workspace.tsx: 8,247 lines
Last modified: Every single day by multiple engineers
Git blame: 47 different authors
Merge conflicts: 12-15 per week
Test coverage: 8% (impossible to test)
```

**The file structure:**
```javascript
// Workspace.tsx (8,247 lines of nightmare)
function Workspace() {
  // STATE (lines 1-450): 89 useState calls
  const [users, setUsers] = useState([]);
  const [projects, setProjects] = useState([]);
  const [tasks, setTasks] = useState([]);
  const [comments, setComments] = useState([]);
  const [notifications, setNotifications] = useState([]);
  const [settings, setSettings] = useState({});
  const [theme, setTheme] = useState('light');
  const [sidebar, setSidebar] = useState(true);
  const [modal, setModal] = useState(null);
  // ... 80 more useState calls

  // DATA FETCHING (lines 451-1200): 15 useEffect hooks
  useEffect(() => { /* fetch users */ }, []);
  useEffect(() => { /* fetch projects */ }, [userId]);
  useEffect(() => { /* fetch tasks */ }, [projectId]);
  useEffect(() => { /* fetch comments */ }, [taskId]);
  useEffect(() => { /* sync with WebSocket */ }, []);
  useEffect(() => { /* poll for updates */ }, []);
  // ... 9 more useEffects

  // BUSINESS LOGIC (lines 1201-3500): 180 functions
  const handleCreateProject = () => { /* 45 lines */ };
  const handleUpdateTask = () => { /* 67 lines */ };
  const handleDeleteComment = () => { /* 23 lines */ };
  const handleAssignUser = () => { /* 89 lines */ };
  const validateTaskData = () => { /* 156 lines */ };
  const calculatePermissions = () => { /* 234 lines */ };
  // ... 174 more functions

  // RENDER (lines 3501-8247): Giant JSX nightmare
  return (
    <div>
      {/* 4,746 lines of inline JSX */}
      {/* No component extraction */}
      {/* Everything inline */}
    </div>
  );
}
```

### The Crisis Point (May 15, 2023)

**The request:** "Add team collaboration feature - 2 week sprint"

**What happened:**
- Week 1: 3 engineers started working simultaneously
- 127 merge conflicts in Workspace.tsx
- Spent 18 hours resolving conflicts, introduced 8 bugs
- Week 2: Fixed bugs, introduced 6 new bugs
- Week 3: Gave up. Feature incomplete.

**Engineering manager's report:**
> "We can no longer ship features. Every change to Workspace.tsx breaks something else. We have 3 engineers spending 100% of their time fixing bugs introduced by other changes. The codebase has become unmaintainable."

### The Numbers

**Developer productivity collapse:**
```
Time to add simple feature:
  Before (2022): 2-3 days
  Now (May 2023): 2-3 weeks (if at all)

Merge conflict time per PR:
  Before: 5 minutes
  Now: 2-4 hours

Bug introduction rate:
  Before: 1 bug per 10 features
  Now: 3 bugs per 1 feature

Test coverage:
  Target: 80%
  Reality: 8% (Workspace.tsx untestable)
```

**Business Impact:**
- 6 planned features delayed (Q2 roadmap missed)
- Lost deals: 3 enterprise customers walked away ($1.8M annual contract value)
- Engineering morale: 6 senior engineers quit in 2 months
- Hiring impact: Candidates rejected offers after seeing codebase

### The Root Cause

**How did this happen?**

2020: Workspace.tsx created, 300 lines
```javascript
// Workspace.tsx v1 (300 lines - reasonable)
function Workspace() {
  const [projects, setProjects] = useState([]);

  useEffect(() => {
    fetch('/api/projects').then(r => r.json()).then(setProjects);
  }, []);

  return (
    <div>
      {projects.map(p => <ProjectCard key={p.id} project={p} />)}
    </div>
  );
}
```

**Then feature creep:**
- 2020 Q3: Added tasks (400 lines → 700 lines)
- 2020 Q4: Added comments (700 → 1,200)
- 2021 Q1: Added notifications (1,200 → 1,800)
- 2021 Q2: Added real-time sync (1,800 → 2,500)
- 2021 Q3: Added permissions (2,500 → 3,400)
- 2021 Q4: Added team features (3,400 → 4,800)
- 2022: More features (4,800 → 6,500)
- 2023: Breaking point (6,500 → 8,247)

**Nobody refactored because:**
1. "Too risky to touch - it works"
2. "We don't have time" (always in sprint mode)
3. "I'll just add my feature and leave" (tragedy of the commons)
4. No architectural reviews (code reviews only checked "does it work?")
5. No file size limits in CI/CD

### The 6-Week Rewrite (June-July 2023)

**The decision:** Complete architectural rewrite. Feature freeze.

**New architecture:**
```
features/
├── workspace/
│   ├── components/          # Presentation (20 files, avg 80 lines)
│   │   ├── WorkspaceLayout.tsx
│   │   ├── Sidebar.tsx
│   │   └── TopBar.tsx
│   ├── hooks/               # Data & logic (12 hooks, avg 60 lines)
│   │   ├── useWorkspace.ts
│   │   ├── useProjects.ts
│   │   ├── useTasks.ts
│   │   └── useComments.ts
│   ├── services/            # API layer (5 files)
│   │   ├── workspaceService.ts
│   │   ├── projectService.ts
│   │   └── taskService.ts
│   ├── utils/               # Pure functions (testable!)
│   │   ├── permissions.ts
│   │   ├── validation.ts
│   │   └── calculations.ts
│   └── index.ts            # Public API

  ├── projects/              # Feature module
  ├── tasks/                 # Feature module
  └── comments/              # Feature module
```

**Result:**
- 8,247 lines → 2,100 lines (across 45 files)
- Avg file size: 47 lines
- Test coverage: 8% → 76%
- Merge conflicts: 12/week → 1/week

### After the Rewrite

**Productivity comparison:**

| Metric | Before (May) | After (Aug) | Improvement |
|--------|--------------|-------------|-------------|
| Feature velocity | 0.3 features/week | 4 features/week | **13× faster** |
| Bug rate | 300% (3 bugs per feature) | 15% | **20× better** |
| Merge conflicts | 12 per week | 1 per week | **12× fewer** |
| Time to onboard new dev | 4 weeks | 3 days | **9× faster** |
| Test coverage | 8% | 76% | **9.5× more tested** |

**The delayed team collaboration feature?**
- Before: 3 weeks, failed
- After: 2 days, shipped ✅

### Lessons Learned

1. **File size is a smell**
   - 8,000 lines = architectural failure
   - Set CI limits: max 300 lines per file
   - Force architecture discussions

2. **"Works" ≠ good code**
   - Workspace.tsx "worked" but killed productivity
   - Focus on maintainability, not just functionality
   - **Code is read 10× more than written**

3. **Refactoring isn't optional**
   - "No time to refactor" → eventually no time to ship features
   - Allocate 20% of sprint to paying down tech debt
   - **Prevention is 10× cheaper than rewrite**

4. **Coupling kills teams**
   - 15 engineers editing 1 file = guaranteed conflicts
   - Feature modules = independent work
   - **Good architecture enables parallel development**

5. **Architecture decay is gradual**
   - 300 lines → 8,000 lines over 3 years
   - Nobody noticed until it was too late
   - **Monitor file size, complexity metrics**

6. **Rewrites are expensive**
   - 6-week feature freeze
   - $1.8M in lost revenue
   - 6 engineers quit
   - **But necessary when architecture fails completely**

### The Cost

**Direct costs:**
- Lost deals: $1.8M annual contract value
- 6-week feature freeze opportunity cost: $500K
- Rewrite engineering time (15 devs × 6 weeks): $720K
- **Total: $3M+**

**Indirect costs:**
- Recruiting/training replacements for 6 engineers who quit: $900K
- Damaged team morale (eng satisfaction score: 7.2 → 4.1)
- Technical debt in other components (copied the bad pattern)
- Customer trust (delayed features, buggy releases)

**Could have been prevented by:**
- File size limit in CI: Free (1 hour to set up)
- Monthly architecture reviews: 2 hours/month = $12K/year
- 20% time for refactoring: Built into sprint planning
- **Total prevention cost: ~$15K/year**
- **ROI: 200× cheaper than rewrite**

### The Happy Ending

After the rewrite:
- Feature velocity back to normal (actually 13× better)
- Zero merge conflicts in Workspace feature for 3 months
- Team morale recovered (eng satisfaction: 4.1 → 7.8)
- Hired 8 new engineers (attracted by clean codebase)
- Won back 2 of the 3 lost customers
- CTO wrote blog post "How we refactored 8,000 lines into maintainable architecture" (HackerNews front page)

The rewrite was painful but necessary. But it never should have gotten that bad.

---

## 🚫 Common Mistakes Gallery

### Mistake 1: God Component
Putting everything in one massive component. Split by responsibility: hooks for data, components for UI, services for API.

### Mistake 2: Organizing by File Type
```
components/
hooks/
utils/
```
Scale breaks this. Use feature-based organization.

### Mistake 3: Passing Props Through Multiple Layers
Prop drilling 5+ levels deep. Use Context, composition, or component co-location instead.

### Mistake 4: No Abstraction Layers
Components calling fetch directly. Create service layer → hooks → components for clean boundaries.

### Mistake 5: Mixing Business Logic with UI
Validation, calculations, transformations in JSX. Extract to pure functions for testability.

### Mistake 6: No Public API for Features
Exposing internal structure. Create index.ts that exports only what other features need.

---

## 🧪 Architecture Lab: Measure Developer Productivity

These experiments demonstrate how architecture impacts development speed and maintainability.

### Lab 1: Test Writing Speed

**Hypothesis:** Well-architected code is faster to test.

**Experiment:**

```javascript
// Setup A: God Component (tightly coupled)
function UserDashboard() {
  const [user, setUser] = useState(null);
  const [posts, setPosts] = useState([]);

  useEffect(() => {
    fetch('/api/user').then(r => r.json()).then(setUser);
    fetch('/api/posts').then(r => r.json()).then(setPosts);
  }, []);

  return (
    <div>
      <h1>Welcome {user?.name}</h1>
      {posts.map(post => <div key={post.id}>{post.title}</div>)}
    </div>
  );
}

// Time yourself: Write a test that verifies user name is displayed
// Start timer...
```

<details>
<summary>⏱️ Reveal test code (Setup A)</summary>

```javascript
// You probably wrote something like this (12-15 lines):
test('displays user name', async () => {
  global.fetch = jest.fn((url) => {
    if (url === '/api/user') {
      return Promise.resolve({
        json: () => Promise.resolve({ name: 'John' })
      });
    }
    if (url === '/api/posts') {
      return Promise.resolve({
        json: () => Promise.resolve([])
      });
    }
  });

  render(<UserDashboard />);
  await waitFor(() => {
    expect(screen.getByText('Welcome John')).toBeInTheDocument();
  });
});

// Problem: Had to mock BOTH user AND posts API, even though we're only testing user name
// Time: ~3-5 minutes to write
```

</details>

```javascript
// Setup B: Well-Architected (separated concerns)
function UserDashboard() {
  const { user } = useUser();
  const { posts } = usePosts();

  return (
    <div>
      <UserHeader user={user} />
      <PostList posts={posts} />
    </div>
  );
}

function UserHeader({ user }) {
  return <h1>Welcome {user?.name}</h1>;
}

// Time yourself: Write a test that verifies user name is displayed
// Start timer...
```

<details>
<summary>⏱️ Reveal test code (Setup B)</summary>

```javascript
// Much simpler (3 lines):
test('displays user name', () => {
  render(<UserHeader user={{ name: 'John' }} />);
  expect(screen.getByText('Welcome John')).toBeInTheDocument();
});

// No mocking fetch, no async, no other API concerns
// Time: ~30 seconds to write
```

</details>

**Results:**
- Setup A (God Component): 3-5 minutes, 12-15 lines
- Setup B (Well-Architected): 30 seconds, 3 lines
- **Speedup: 6-10× faster to test**

**Lesson:** Good architecture makes testing trivial. Bad architecture makes testing painful.

---

### Lab 2: Change Impact Analysis

**Hypothesis:** Good architecture localizes changes. Bad architecture creates cascading changes.

**Experiment:** Track how many files you need to modify for this requirement change:

**Requirement:** "Change the user API endpoint from `/api/user` to `/api/v2/users/me`"

```javascript
// Setup A: Direct fetch() calls in components
// File 1: components/UserDashboard.jsx
function UserDashboard() {
  useEffect(() => {
    fetch('/api/user').then(...);  // ← Change here
  }, []);
}

// File 2: components/UserProfile.jsx
function UserProfile() {
  useEffect(() => {
    fetch('/api/user').then(...);  // ← Change here
  }, []);
}

// File 3: components/UserSettings.jsx
function UserSettings() {
  useEffect(() => {
    fetch('/api/user').then(...);  // ← Change here
  }, []);
}

// Files modified: 3
// Lines changed: 3
// Risk: Might miss one, causing runtime error
```

**Now try Setup B:**

```javascript
// Setup B: Abstraction layer
// File 1: services/userService.js
export const userService = {
  async getUser() {
    return fetch('/api/user').then(...);  // ← Change ONLY here to '/api/v2/users/me'
  }
};

// All components unchanged:
function UserDashboard() {
  useEffect(() => {
    userService.getUser().then(...);  // No change needed
  }, []);
}

// Files modified: 1
// Lines changed: 1
// Risk: Zero (all components use same service)
```

**Measure yourself:**
1. Fork your own codebase
2. Time how long it takes to change API endpoints
3. Count how many files you touched

**Results:**
- Setup A: Touch 3+ files, risk missing some, 5-10 minutes
- Setup B: Touch 1 file, zero risk, 30 seconds
- **Speedup: 10-20× faster to change**

**Lesson:** Abstraction layers (services, hooks) make changes easy. Direct dependencies make changes risky.

---

### Lab 3: Feature Deletion Speed

**Hypothesis:** Good organization makes features easy to delete. Bad organization makes deletion scary.

**Experiment:** Time yourself deleting a "wishlist" feature.

**Setup A: File-type organization**

```
src/
├── components/
│   ├── ProductCard.jsx       (uses wishlist?)
│   ├── WishlistButton.jsx    ← Delete this
│   ├── WishlistPage.jsx      ← Delete this
│   ├── WishlistItem.jsx      ← Delete this
│   └── ... (48 other files - which use wishlist?)
├── hooks/
│   ├── useWishlist.js        ← Delete this
│   ├── useProducts.js        (uses wishlist?)
│   └── ... (12 other files - which use wishlist?)
├── services/
│   ├── wishlistService.js    ← Delete this
│   └── ... (8 other files - do any import wishlistService?)
└── types/
    ├── wishlist.ts           ← Delete this
    └── ... (15 other files - do any import Wishlist type?)
```

**Questions you must answer:**
1. Which files implement wishlist? (grep through 70+ files)
2. Which files import wishlist? (check dependencies)
3. Can I safely delete without breaking other features?

**Time:** 20-30 minutes searching + 10 minutes deleting = **30-40 minutes**

**Setup B: Feature-based organization**

```
src/features/
├── wishlist/        ← Delete entire folder
│   ├── components/
│   ├── hooks/
│   ├── services/
│   └── types/
├── products/
├── cart/
└── user/
```

**Steps:**
1. Delete `features/wishlist/` folder
2. Search for imports: `grep -r "from '@/features/wishlist'" src/`
3. Remove those imports (likely just WishlistButton in ProductCard)

**Time:** 2 minutes to delete + 1 minute to remove imports = **3 minutes**

**Results:**
- Setup A: 30-40 minutes, uncertain if safe
- Setup B: 3 minutes, confident
- **Speedup: 10-13× faster**

**Lesson:** Feature-based organization makes deletion safe and fast. File-type organization makes it scary.

---

### Summary: Architecture Performance Metrics

| Metric | God Component / Flat Structure | Well-Architected |
|--------|-------------------------------|------------------|
| **Test writing time** | 3-5 minutes per test | 30 seconds per test |
| **Change impact** | 3-10 files modified | 1 file modified |
| **Feature deletion** | 30-40 minutes, uncertain | 3 minutes, confident |
| **Onboarding new dev** | 2-4 weeks to understand | 2-3 days to understand |
| **Bug fixing time** | Hours (finding the bug) | Minutes (clear boundaries) |

**Key insight:** Good architecture is a productivity multiplier. The Workspace.tsx war story showed 13× productivity improvement just from better architecture.

**Try it yourself:** Take a feature in your codebase and measure:
1. How long does it take to write a test?
2. How many files do you touch for a simple change?
3. How long would it take to delete the feature?

If the numbers are high, you have an architecture problem.

---

## 📝 Cumulative Review

### Q1: Why did the 8,000-line Workspace.tsx kill productivity?
<details><summary>Answer</summary>
Merge conflicts (12/week), impossible to test (8% coverage), every change broke something else, 15 engineers editing same file simultaneously. Good architecture enables parallel work; bad architecture forces serial conflicts.
</details>

### Q2: What's the difference between coupling and cohesion?
<details><summary>Answer</summary>
Coupling: how much components depend on each other (lower is better). Cohesion: how related things within a component are (higher is better). Goal: low coupling, high cohesion. Example: feature modules have high cohesion (related code together), low coupling (features don't depend on each other).
</details>

### Q3: When should you extract code into a custom hook?
<details><summary>Answer</summary>
When you have: data fetching logic, complex state management, or side effects that could be reused or tested separately. Don't extract for every useState - only when there's actual logic to abstract. Rule: if it has business logic or side effects, extract it.
</details>

### Q4: How do you know if a component has too many responsibilities?
<details><summary>Answer</summary>
Signs: hard to name (needs "and" in name like "UserListAndEditorAndSettings"), hard to test (too many mocks needed), changes for multiple reasons, 500+ lines. Fix: apply Single Responsibility - each component does one thing.
</details>

### Q5: What's the benefit of feature-based organization vs file-type organization?
<details><summary>Answer</summary>
Feature-based: all related code together, easy to delete features, teams can own features independently, scales to large codebases. File-type (components/, hooks/): works up to ~50 components, then everything related is scattered, hard to understand dependencies.
</details>

### Q6: After the Workspace rewrite, feature velocity increased 13×. Why so dramatic?
<details><summary>Answer</summary>
Before: one giant file, merge conflicts, couldn't test, bugs everywhere, fear of breaking things. After: small focused files, parallel work (no conflicts), 76% test coverage (confidence), clear boundaries. Architecture directly impacts team velocity - bad architecture is a productivity tax.
</details>

---

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
