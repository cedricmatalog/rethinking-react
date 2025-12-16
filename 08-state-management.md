# Chapter 8: State Management at Scale

## Introduction

Junior developers reach for Redux for everything. Senior developers understand state categories and choose the right tool for each.

This chapter teaches you to manage state at scale with confidence.

## Learning Objectives

- Categorize different types of state
- Choose appropriate state management solutions
- Implement scalable state architectures
- Avoid common state management pitfalls
- Know when NOT to use global state

## 8.1 Types of State

### The State Spectrum

```javascript
// 1. Local State - useState
function Counter() {
  const [count, setCount] = useState(0); // Component-specific
  return <button onClick={() => setCount(count + 1)}>{count}</button>;
}

// 2. Lifted State - Props
function Parent() {
  const [filter, setFilter] = useState('');
  return (
    <>
      <FilterInput value={filter} onChange={setFilter} />
      <FilteredList filter={filter} />
    </>
  );
}

// 3. Shared State - Context
const ThemeContext = createContext();

function App() {
  const [theme, setTheme] = useState('light');
  return (
    <ThemeContext.Provider value={{ theme, setTheme }}>
      <Header />
      <Main />
    </ThemeContext.Provider>
  );
}

// 4. Server State - React Query/SWR
function UserProfile({ userId }) {
  const { data: user } = useQuery(['user', userId], () =>
    fetchUser(userId)
  );
  return <div>{user?.name}</div>;
}

// 5. URL State - Router
function ProductList() {
  const [searchParams] = useSearchParams();
  const page = searchParams.get('page') || 1;
  const filter = searchParams.get('filter') || '';

  return <List page={page} filter={filter} />;
}

// 6. Global State - Redux/Zustand
const useStore = create((set) => ({
  cart: [],
  addItem: (item) => set((state) => ({
    cart: [...state.cart, item]
  }))
}));
```

### Decision Framework

```
Ask yourself:

1. Is it used in one component only?
   → Local state (useState)

2. Is it used by siblings/parent-child?
   → Lift state up

3. Is it used across the app but changes rarely?
   → Context

4. Is it fetched from server?
   → React Query / SWR (don't store in Redux!)

5. Should it persist across page refreshes?
   → URL state or localStorage

6. Is it complex, frequently updated, accessed everywhere?
   → Global state (Redux/Zustand)
```

## 8.2 Context API at Scale

### The Performance Problem

```javascript
// ANTI-PATTERN: One context for everything
const AppContext = createContext();

function AppProvider({ children }) {
  const [user, setUser] = useState(null);
  const [theme, setTheme] = useState('light');
  const [notifications, setNotifications] = useState([]);
  const [settings, setSettings] = useState({});

  // Every state change re-renders ALL consumers!
  return (
    <AppContext.Provider
      value={{ user, theme, notifications, settings, /* ...setters */ }}
    >
      {children}
    </AppContext.Provider>
  );
}

// Component only needs theme, but re-renders for everything
function ThemeToggle() {
  const { theme, setTheme } = useContext(AppContext);
  return <button onClick={() => setTheme(theme === 'light' ? 'dark' : 'light')}>
    {theme}
  </button>;
}
```

### The Solution: Split Contexts

```javascript
// Split by concern
const UserContext = createContext();
const ThemeContext = createContext();
const NotificationContext = createContext();

function Providers({ children }) {
  return (
    <UserProvider>
      <ThemeProvider>
        <NotificationProvider>
          {children}
        </NotificationProvider>
      </ThemeProvider>
    </UserProvider>
  );
}

// Now components only re-render when their data changes
function ThemeToggle() {
  const { theme, setTheme } = useContext(ThemeContext);
  // Only re-renders when theme changes
}
```

### Context with Selectors

```javascript
// Advanced: Context with fine-grained subscriptions
function createStore(initialState) {
  const subscribers = new Set();
  let state = initialState;

  const store = {
    getState: () => state,

    setState: (newState) => {
      state = typeof newState === 'function'
        ? newState(state)
        : newState;
      subscribers.forEach(callback => callback());
    },

    subscribe: (callback) => {
      subscribers.add(callback);
      return () => subscribers.delete(callback);
    }
  };

  return store;
}

const StoreContext = createContext();

function useSelector(selector) {
  const store = useContext(StoreContext);
  const [state, setState] = useState(() => selector(store.getState()));

  useEffect(() => {
    const checkForUpdates = () => {
      const newState = selector(store.getState());
      setState(prevState => {
        // Only update if selected data changed
        return Object.is(prevState, newState) ? prevState : newState;
      });
    };

    const unsubscribe = store.subscribe(checkForUpdates);
    return unsubscribe;
  }, [store, selector]);

  return state;
}

// Usage
function UserName() {
  const name = useSelector(state => state.user.name);
  // Only re-renders when name changes
}
```

## 8.3 Redux vs Zustand vs Jotai

### Redux: The Battle-Tested Giant

```javascript
// Redux with Redux Toolkit
import { createSlice, configureStore } from '@reduxjs/toolkit';

const cartSlice = createSlice({
  name: 'cart',
  initialState: { items: [], total: 0 },
  reducers: {
    addItem: (state, action) => {
      state.items.push(action.payload);
      state.total += action.payload.price;
    },
    removeItem: (state, action) => {
      const index = state.items.findIndex(i => i.id === action.payload);
      if (index !== -1) {
        state.total -= state.items[index].price;
        state.items.splice(index, 1);
      }
    }
  }
});

const store = configureStore({
  reducer: {
    cart: cartSlice.reducer
  }
});

// Usage
function CartButton() {
  const items = useSelector(state => state.cart.items);
  const dispatch = useDispatch();

  return (
    <button onClick={() => dispatch(addItem({ id: 1, price: 10 }))}>
      Cart ({items.length})
    </button>
  );
}
```

### Zustand: The Simple Alternative

```javascript
// Zustand - much simpler
import create from 'zustand';

const useCartStore = create((set) => ({
  items: [],
  total: 0,

  addItem: (item) => set((state) => ({
    items: [...state.items, item],
    total: state.total + item.price
  })),

  removeItem: (id) => set((state) => {
    const item = state.items.find(i => i.id === id);
    return {
      items: state.items.filter(i => i.id !== id),
      total: state.total - (item?.price || 0)
    };
  })
}));

// Usage - even simpler
function CartButton() {
  const items = useCartStore(state => state.items);
  const addItem = useCartStore(state => state.addItem);

  return (
    <button onClick={() => addItem({ id: 1, price: 10 })}>
      Cart ({items.length})
    </button>
  );
}
```

### Jotai: Atomic State

```javascript
// Jotai - atomic approach
import { atom, useAtom } from 'jotai';

const cartItemsAtom = atom([]);
const cartTotalAtom = atom((get) => {
  const items = get(cartItemsAtom);
  return items.reduce((sum, item) => sum + item.price, 0);
});

const addItemAtom = atom(
  null,
  (get, set, item) => {
    set(cartItemsAtom, [...get(cartItemsAtom), item]);
  }
);

// Usage
function CartButton() {
  const [items] = useAtom(cartItemsAtom);
  const [, addItem] = useAtom(addItemAtom);

  return (
    <button onClick={() => addItem({ id: 1, price: 10 })}>
      Cart ({items.length})
    </button>
  );
}
```

### Comparison

| Feature | Redux | Zustand | Jotai |
|---------|-------|---------|-------|
| Bundle Size | 11KB | 1.2KB | 3KB |
| Boilerplate | High | Low | Medium |
| DevTools | Excellent | Good | Good |
| Learning Curve | Steep | Gentle | Medium |
| Middleware | Extensive | Simple | Plugins |
| Best For | Large apps | Most apps | Fine-grained |

## 8.4 Server State Management

### React Query

```javascript
// Perfect for server state
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';

function UserProfile({ userId }) {
  // Automatic caching, refetching, background updates
  const { data: user, isLoading, error } = useQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId),
    staleTime: 5 * 60 * 1000, // 5 minutes
    cacheTime: 10 * 60 * 1000 // 10 minutes
  });

  const queryClient = useQueryClient();

  const updateUserMutation = useMutation({
    mutationFn: (updates) => updateUser(userId, updates),
    onSuccess: () => {
      // Invalidate and refetch
      queryClient.invalidateQueries({ queryKey: ['user', userId] });
    }
  });

  if (isLoading) return <Spinner />;
  if (error) return <Error error={error} />;

  return (
    <div>
      <h1>{user.name}</h1>
      <button onClick={() => updateUserMutation.mutate({ name: 'New Name' })}>
        Update
      </button>
    </div>
  );
}
```

### Optimistic Updates

```javascript
function TodoList() {
  const { data: todos } = useQuery({
    queryKey: ['todos'],
    queryFn: fetchTodos
  });

  const queryClient = useQueryClient();

  const addTodoMutation = useMutation({
    mutationFn: createTodo,

    // Optimistic update
    onMutate: async (newTodo) => {
      // Cancel outgoing refetches
      await queryClient.cancelQueries({ queryKey: ['todos'] });

      // Snapshot previous value
      const previousTodos = queryClient.getQueryData(['todos']);

      // Optimistically update
      queryClient.setQueryData(['todos'], (old) => [
        ...old,
        { ...newTodo, id: 'temp-id', status: 'pending' }
      ]);

      return { previousTodos };
    },

    // Rollback on error
    onError: (err, newTodo, context) => {
      queryClient.setQueryData(['todos'], context.previousTodos);
      toast.error('Failed to add todo');
    },

    // Refetch on success
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['todos'] });
    }
  });

  return (
    <div>
      {todos.map(todo => (
        <TodoItem key={todo.id} todo={todo} />
      ))}
      <button onClick={() => addTodoMutation.mutate({ title: 'New Todo' })}>
        Add Todo
      </button>
    </div>
  );
}
```

## 8.5 URL as State

### Why URL State?

```javascript
// Benefits:
// - Shareable links
// - Browser back/forward
// - Bookmarkable
// - SSR friendly
// - No prop drilling

// What belongs in URL?
// - Filters
// - Search queries
// - Pagination
// - Sort order
// - Modal/dialog state
// - Tab selection
```

### Implementation

```javascript
function ProductList() {
  const [searchParams, setSearchParams] = useSearchParams();

  // Read from URL
  const page = parseInt(searchParams.get('page') || '1');
  const filter = searchParams.get('filter') || '';
  const sort = searchParams.get('sort') || 'name';

  // Update URL
  const setFilter = (value) => {
    setSearchParams({
      ...Object.fromEntries(searchParams),
      filter: value,
      page: '1' // Reset page when filtering
    });
  };

  const setPage = (value) => {
    setSearchParams({
      ...Object.fromEntries(searchParams),
      page: String(value)
    });
  };

  return (
    <div>
      <FilterInput value={filter} onChange={setFilter} />
      <ProductGrid
        products={products}
        sort={sort}
      />
      <Pagination
        page={page}
        onPageChange={setPage}
      />
    </div>
  );
}

// URL: /products?filter=electronics&page=2&sort=price
// Users can share this exact view!
```

### Custom Hook

```javascript
function useQueryState(key, defaultValue) {
  const [searchParams, setSearchParams] = useSearchParams();

  const value = searchParams.get(key) || defaultValue;

  const setValue = (newValue) => {
    setSearchParams(prev => {
      const params = new URLSearchParams(prev);
      if (newValue === defaultValue) {
        params.delete(key);
      } else {
        params.set(key, newValue);
      }
      return params;
    });
  };

  return [value, setValue];
}

// Usage
function Component() {
  const [filter, setFilter] = useQueryState('filter', '');
  const [page, setPage] = useQueryState('page', '1');

  return (
    <>
      <input value={filter} onChange={(e) => setFilter(e.target.value)} />
      <button onClick={() => setPage(String(parseInt(page) + 1))}>
        Next Page
      </button>
    </>
  );
}
```

## 8.6 State Architecture Patterns

### Layered State

```
┌─────────────────────────────────────┐
│         URL State                   │  ← Filters, pagination
├─────────────────────────────────────┤
│         Server State                │  ← React Query/SWR
├─────────────────────────────────────┤
│         Global App State            │  ← Redux/Zustand
├─────────────────────────────────────┤
│         Context State               │  ← Theme, auth
├─────────────────────────────────────┤
│         Local Component State       │  ← Form inputs, UI
└─────────────────────────────────────┘
```

### Module Pattern

```javascript
// features/cart/store.ts
export const useCartStore = create((set) => ({
  items: [],
  addItem: (item) => set((state) => ({
    items: [...state.items, item]
  }))
}));

// features/auth/store.ts
export const useAuthStore = create((set) => ({
  user: null,
  login: (user) => set({ user }),
  logout: () => set({ user: null })
}));

// Each feature owns its state
// No giant central store
```

### Event-Driven Updates

```javascript
// Event bus for cross-feature communication
const eventBus = {
  events: {},

  on(event, callback) {
    if (!this.events[event]) {
      this.events[event] = [];
    }
    this.events[event].push(callback);
  },

  emit(event, data) {
    if (this.events[event]) {
      this.events[event].forEach(callback => callback(data));
    }
  }
};

// Feature A
function CartButton() {
  const items = useCartStore(state => state.items);

  const handlePurchase = () => {
    // Complete purchase
    eventBus.emit('purchase:completed', { items });
  };
}

// Feature B (doesn't know about Feature A)
function Analytics() {
  useEffect(() => {
    eventBus.on('purchase:completed', (data) => {
      trackPurchase(data.items);
    });
  }, []);
}
```

## Real-World Scenario: Refactoring State Management

### The Challenge

Legacy app with problems:
- Everything in Redux (even form inputs!)
- 50+ actions and reducers
- Props drilling through 5+ levels
- API data in Redux (stale data issues)
- Performance problems

### Your Refactoring Plan

1. **Identify state categories**
2. **Extract server state to React Query**
3. **Move UI state to local/URL**
4. **Keep only app state in Redux**
5. **Measure improvements**

## Chapter Exercise: State Management Audit

Analyze and refactor this app:

```javascript
// Everything in Redux - bad!
const appSlice = createSlice({
  name: 'app',
  initialState: {
    users: [],
    currentUser: null,
    theme: 'light',
    searchQuery: '',
    page: 1,
    filter: '',
    modalOpen: false
  },
  reducers: {
    // 20+ reducers...
  }
});

// Task: Refactor to appropriate state management
// - Server state → React Query
// - UI state → Local
// - Filters/page → URL
// - Theme → Context
// - Only keep global app state
```

## Review Checklist

- [ ] Categorize different types of state
- [ ] Choose appropriate state solution for each type
- [ ] Avoid putting everything in global state
- [ ] Use React Query for server state
- [ ] Leverage URL for shareable state
- [ ] Split contexts by concern
- [ ] Understand Redux vs Zustand vs Jotai
- [ ] Implement optimistic updates

## Key Takeaways

1. **Not all state is equal** - Categorize first
2. **Server state ≠ client state** - Use React Query
3. **Context can be slow** - Split by concern
4. **URL is underused** - Great for shareable state
5. **Local state is fine** - Don't lift unnecessarily
6. **Choose the right tool** - Redux isn't always the answer
7. **Architecture matters** - Plan state management

## Further Reading

- React Query documentation
- Zustand vs Redux comparison
- "Application State Management with React" by Kent C. Dodds
- Jotai documentation

## Next Chapter

[Chapter 9: Project Structure & Organization](./09-project-structure.md)
