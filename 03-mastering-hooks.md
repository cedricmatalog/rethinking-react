# Chapter 3: Mastering React Hooks

## Introduction

Junior developers use hooks to manage state. Senior developers use hooks to create reusable logic, optimize performance, and build elegant abstractions.

This chapter goes beyond basic useState and useEffect to master the entire hooks ecosystem.

## Learning Objectives

- Master all built-in React hooks
- Create powerful custom hooks
- Understand hooks implementation details
- Avoid common hooks pitfalls
- Design hooks-based architectures

## 3.1 useState Deep Dive

### Beyond Basic State

```javascript
// Junior: Separate state for everything
function Form() {
  const [firstName, setFirstName] = useState('');
  const [lastName, setLastName] = useState('');
  const [email, setEmail] = useState('');
  const [phone, setPhone] = useState('');
  const [address, setAddress] = useState('');
  // ... 10 more fields

  return (/* form */);
}

// Senior: Grouped related state
function Form() {
  const [formData, setFormData] = useState({
    firstName: '',
    lastName: '',
    email: '',
    phone: '',
    address: ''
  });

  const updateField = (field, value) => {
    setFormData(prev => ({ ...prev, [field]: value }));
  };

  return (/* form */);
}
```

### Functional Updates

```javascript
// WRONG: Using stale state
function Counter() {
  const [count, setCount] = useState(0);

  const handleClick = () => {
    setCount(count + 1);
    setCount(count + 1); // Still increments by 1!
    setCount(count + 1);
  };

  return <button onClick={handleClick}>{count}</button>;
}

// CORRECT: Functional updates
function Counter() {
  const [count, setCount] = useState(0);

  const handleClick = () => {
    setCount(c => c + 1); // Increments by 3
    setCount(c => c + 1);
    setCount(c => c + 1);
  };

  return <button onClick={handleClick}>{count}</button>;
}
```

### Lazy Initialization

```javascript
// BAD: Expensive calculation on every render
function Component() {
  const [data, setData] = useState(expensiveCalculation());
  // expensiveCalculation runs on every render!
}

// GOOD: Lazy initialization
function Component() {
  const [data, setData] = useState(() => expensiveCalculation());
  // Only runs once on mount
}

// Example: Reading from localStorage
function useLocalStorage(key, defaultValue) {
  const [value, setValue] = useState(() => {
    try {
      const stored = localStorage.getItem(key);
      return stored ? JSON.parse(stored) : defaultValue;
    } catch {
      return defaultValue;
    }
  });

  return [value, setValue];
}
```

### Hands-On Exercise 3.1

Build a `useUndo` hook that provides undo/redo functionality:

```javascript
function useUndo(initialState) {
  // Implement:
  // - state: current state
  // - setState: update state
  // - undo: go back
  // - redo: go forward
  // - canUndo: boolean
  // - canRedo: boolean
  // - reset: clear history
}

// Usage
function DrawingApp() {
  const [drawing, setDrawing, { undo, redo, canUndo, canRedo }] =
    useUndo([]);

  return (
    <>
      <button onClick={undo} disabled={!canUndo}>Undo</button>
      <button onClick={redo} disabled={!canRedo}>Redo</button>
    </>
  );
}
```

## 3.2 useEffect Mastery

### Understanding Dependencies

```javascript
// Junior: Missing dependencies (bugs!)
function Profile({ userId }) {
  const [user, setUser] = useState(null);

  useEffect(() => {
    fetchUser(userId).then(setUser);
  }, []); // Missing userId dependency!

  // Won't update when userId changes
}

// Senior: Correct dependencies
function Profile({ userId }) {
  const [user, setUser] = useState(null);

  useEffect(() => {
    fetchUser(userId).then(setUser);
  }, [userId]); // ✓ Correct
}
```

### Cleanup Functions

```javascript
// Without cleanup - memory leak!
function Timer() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const id = setInterval(() => {
      setCount(c => c + 1);
    }, 1000);
    // No cleanup - interval keeps running after unmount!
  }, []);

  return <div>{count}</div>;
}

// With cleanup
function Timer() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const id = setInterval(() => {
      setCount(c => c + 1);
    }, 1000);

    return () => clearInterval(id); // Cleanup
  }, []);

  return <div>{count}</div>;
}
```

### Async Effects Pattern

```javascript
// WRONG: async useEffect
useEffect(async () => {
  const data = await fetchData(); // ❌ Don't do this!
  setData(data);
}, []);

// CORRECT: IIFE pattern
useEffect(() => {
  let cancelled = false;

  (async () => {
    try {
      const data = await fetchData();
      if (!cancelled) {
        setData(data);
      }
    } catch (error) {
      if (!cancelled) {
        setError(error);
      }
    }
  })();

  return () => {
    cancelled = true;
  };
}, []);

// BETTER: Custom hook
function useAsync(asyncFunction, dependencies) {
  const [state, setState] = useState({
    loading: true,
    error: null,
    data: null
  });

  useEffect(() => {
    let cancelled = false;

    setState({ loading: true, error: null, data: null });

    asyncFunction()
      .then(data => {
        if (!cancelled) {
          setState({ loading: false, error: null, data });
        }
      })
      .catch(error => {
        if (!cancelled) {
          setState({ loading: false, error, data: null });
        }
      });

    return () => {
      cancelled = true;
    };
  }, dependencies);

  return state;
}
```

### When NOT to Use useEffect

```javascript
// ANTI-PATTERN: Deriving state with useEffect
function Cart({ items }) {
  const [total, setTotal] = useState(0);

  useEffect(() => {
    setTotal(items.reduce((sum, item) => sum + item.price, 0));
  }, [items]);

  return <div>Total: ${total}</div>;
}

// CORRECT: Compute during render
function Cart({ items }) {
  const total = items.reduce((sum, item) => sum + item.price, 0);
  return <div>Total: ${total}</div>;
}

// ANTI-PATTERN: Event handlers in effects
function SearchBox() {
  const [query, setQuery] = useState('');

  useEffect(() => {
    if (query.length > 3) {
      search(query);
    }
  }, [query]);

  return <input value={query} onChange={e => setQuery(e.target.value)} />;
}

// CORRECT: Handle in event
function SearchBox() {
  const [query, setQuery] = useState('');

  const handleSearch = (value) => {
    setQuery(value);
    if (value.length > 3) {
      search(value);
    }
  };

  return <input value={query} onChange={e => handleSearch(e.target.value)} />;
}
```

### Hands-On Exercise 3.2

Build a `useDebounce` hook:

```javascript
// Should debounce any value
function useDebounce(value, delay) {
  // Your implementation
}

// Usage
function SearchBox() {
  const [search, setSearch] = useState('');
  const debouncedSearch = useDebounce(search, 500);

  useEffect(() => {
    if (debouncedSearch) {
      searchAPI(debouncedSearch);
    }
  }, [debouncedSearch]);

  return <input value={search} onChange={e => setSearch(e.target.value)} />;
}
```

## 3.3 useCallback and useMemo

### When to Memoize

```javascript
// DON'T: Over-optimization
function Component() {
  // Unnecessary - string is cheap
  const title = useMemo(() => 'Hello World', []);

  // Unnecessary - simple calculation
  const doubled = useMemo(() => count * 2, [count]);

  return <div>{title}</div>;
}

// DO: Expensive calculations
function DataGrid({ data }) {
  // Good - expensive operation
  const sortedAndFiltered = useMemo(() => {
    return data
      .filter(item => item.active)
      .sort((a, b) => a.name.localeCompare(b.name));
  }, [data]);

  return <Table data={sortedAndFiltered} />;
}

// DO: Referential equality matters
function Parent() {
  const [count, setCount] = useState(0);

  // Without useCallback, Child re-renders every time
  const handleClick = () => {
    console.log('Clicked');
  };

  // With useCallback, Child only re-renders when needed
  const memoizedHandleClick = useCallback(() => {
    console.log('Clicked');
  }, []);

  return (
    <>
      <button onClick={() => setCount(count + 1)}>{count}</button>
      <ExpensiveChild onClick={memoizedHandleClick} />
    </>
  );
}

const ExpensiveChild = memo(({ onClick }) => {
  // Expensive rendering
  return <button onClick={onClick}>Click me</button>;
});
```

### The Optimization Triad

```javascript
// All three must work together
const Parent = () => {
  const [count, setCount] = useState(0);

  // 1. Memoize the function
  const handleClick = useCallback(() => {
    console.log('Clicked');
  }, []);

  return (
    <>
      <button onClick={() => setCount(count + 1)}>{count}</button>
      {/* 2. Memoize the component */}
      <Child onClick={handleClick} />
    </>
  );
};

// 3. Use React.memo
const Child = memo(({ onClick }) => {
  console.log('Child rendered');
  return <button onClick={onClick}>Click me</button>;
});

// Without any of these three, optimization doesn't work!
```

### Hands-On Exercise 3.3

Profile and optimize this component:

```javascript
function ProductList({ products, category }) {
  const filteredProducts = products.filter(p => p.category === category);
  const sortedProducts = filteredProducts.sort((a, b) => b.price - a.price);

  const handleAddToCart = (productId) => {
    addToCart(productId);
  };

  return (
    <div>
      {sortedProducts.map(product => (
        <ProductCard
          key={product.id}
          product={product}
          onAddToCart={handleAddToCart}
        />
      ))}
    </div>
  );
}

// Task: Optimize using useMemo and useCallback where appropriate
```

## 3.4 useReducer for Complex State

### When to Use useReducer

| useState | useReducer |
|----------|------------|
| Simple state | Complex state logic |
| Few updates | Many state transitions |
| Independent values | Related state updates |
| Simple logic | Complex validation |

### Example: Form with Validation

```javascript
// Complex state with useState - messy
function Form() {
  const [name, setName] = useState('');
  const [email, setEmail] = useState('');
  const [errors, setErrors] = useState({});
  const [touched, setTouched] = useState({});
  const [isSubmitting, setIsSubmitting] = useState(false);

  const handleNameChange = (value) => {
    setName(value);
    if (touched.name) {
      setErrors(prev => ({
        ...prev,
        name: value ? null : 'Required'
      }));
    }
  };

  // ... more complex update logic
}

// Clean state with useReducer
const initialState = {
  values: { name: '', email: '' },
  errors: {},
  touched: {},
  isSubmitting: false
};

function formReducer(state, action) {
  switch (action.type) {
    case 'UPDATE_FIELD':
      return {
        ...state,
        values: {
          ...state.values,
          [action.field]: action.value
        },
        errors: {
          ...state.errors,
          [action.field]: validate(action.field, action.value)
        }
      };

    case 'TOUCH_FIELD':
      return {
        ...state,
        touched: { ...state.touched, [action.field]: true }
      };

    case 'SUBMIT_START':
      return { ...state, isSubmitting: true };

    case 'SUBMIT_SUCCESS':
      return initialState;

    case 'SUBMIT_ERROR':
      return { ...state, isSubmitting: false, errors: action.errors };

    default:
      return state;
  }
}

function Form() {
  const [state, dispatch] = useReducer(formReducer, initialState);

  const handleChange = (field, value) => {
    dispatch({ type: 'UPDATE_FIELD', field, value });
  };

  return (/* clean component code */);
}
```

### State Machine Pattern

```javascript
// Modal state machine
const modalStates = {
  CLOSED: 'CLOSED',
  OPENING: 'OPENING',
  OPEN: 'OPEN',
  CLOSING: 'CLOSING'
};

function modalReducer(state, action) {
  switch (state.status) {
    case modalStates.CLOSED:
      if (action.type === 'OPEN') {
        return { status: modalStates.OPENING };
      }
      return state;

    case modalStates.OPENING:
      if (action.type === 'ANIMATION_END') {
        return { status: modalStates.OPEN };
      }
      return state;

    case modalStates.OPEN:
      if (action.type === 'CLOSE') {
        return { status: modalStates.CLOSING };
      }
      return state;

    case modalStates.CLOSING:
      if (action.type === 'ANIMATION_END') {
        return { status: modalStates.CLOSED };
      }
      return state;

    default:
      return state;
  }
}

function Modal() {
  const [state, dispatch] = useReducer(modalReducer, {
    status: modalStates.CLOSED
  });

  const isOpen = state.status === modalStates.OPEN ||
                 state.status === modalStates.OPENING;

  return (
    <AnimatedModal
      isOpen={isOpen}
      onAnimationEnd={() => dispatch({ type: 'ANIMATION_END' })}
    />
  );
}
```

### Hands-On Exercise 3.4

Build a shopping cart with useReducer:

```javascript
// Actions: ADD_ITEM, REMOVE_ITEM, UPDATE_QUANTITY, APPLY_COUPON, CLEAR_CART
// State: items, total, discount, coupon

function cartReducer(state, action) {
  // Your implementation
}

function ShoppingCart() {
  const [cart, dispatch] = useReducer(cartReducer, initialState);
  // Use the cart
}
```

## 3.5 useRef Beyond DOM Access

### Storing Mutable Values

```javascript
// Use case 1: Previous value
function Component({ value }) {
  const previousValue = useRef();

  useEffect(() => {
    previousValue.current = value;
  }, [value]);

  const delta = value - (previousValue.current || 0);

  return <div>Changed by: {delta}</div>;
}

// Use case 2: Avoiding stale closures
function Timer() {
  const [count, setCount] = useState(0);
  const savedCallback = useRef();

  useEffect(() => {
    savedCallback.current = () => {
      console.log('Count:', count);
    };
  });

  useEffect(() => {
    const tick = () => savedCallback.current();
    const id = setInterval(tick, 1000);
    return () => clearInterval(id);
  }, []); // Empty deps - but always logs current count!

  return <div>{count}</div>;
}

// Use case 3: Instance variables
function VideoPlayer({ src }) {
  const playerRef = useRef(null);
  const isPlayingRef = useRef(false);

  const play = () => {
    playerRef.current?.play();
    isPlayingRef.current = true;
  };

  const pause = () => {
    playerRef.current?.pause();
    isPlayingRef.current = false;
  };

  return (
    <div>
      <video ref={playerRef} src={src} />
      <button onClick={play}>Play</button>
      <button onClick={pause}>Pause</button>
    </div>
  );
}
```

### Hands-On Exercise 3.5

Build a `useInterval` hook that doesn't have stale closure issues:

```javascript
function useInterval(callback, delay) {
  // Should always call latest callback
  // Should handle null delay (pause)
  // Should cleanup on unmount
}

// Usage
function Counter() {
  const [count, setCount] = useState(0);
  const [delay, setDelay] = useState(1000);

  useInterval(() => {
    setCount(count + 1); // Should use current count
  }, delay);

  return <div>{count}</div>;
}
```

## 3.6 Advanced Custom Hooks

### Composition Pattern

```javascript
// Building blocks
function useLocalStorage(key, initialValue) {
  const [value, setValue] = useState(() => {
    const stored = localStorage.getItem(key);
    return stored ? JSON.parse(stored) : initialValue;
  });

  useEffect(() => {
    localStorage.setItem(key, JSON.stringify(value));
  }, [key, value]);

  return [value, setValue];
}

function useDebounce(value, delay) {
  const [debouncedValue, setDebouncedValue] = useState(value);

  useEffect(() => {
    const timer = setTimeout(() => setDebouncedValue(value), delay);
    return () => clearTimeout(timer);
  }, [value, delay]);

  return debouncedValue;
}

// Composed hook
function useDebouncedLocalStorage(key, initialValue, delay = 500) {
  const [value, setValue] = useLocalStorage(key, initialValue);
  const debouncedValue = useDebounce(value, delay);

  return [value, setValue, debouncedValue];
}

// Usage
function SearchBox() {
  const [search, setSearch, debouncedSearch] =
    useDebouncedLocalStorage('search', '', 500);

  useEffect(() => {
    if (debouncedSearch) {
      searchAPI(debouncedSearch);
    }
  }, [debouncedSearch]);

  return <input value={search} onChange={e => setSearch(e.target.value)} />;
}
```

### Advanced useAsync Hook

```javascript
function useAsync(asyncFunction, immediate = true) {
  const [status, setStatus] = useState('idle');
  const [data, setData] = useState(null);
  const [error, setError] = useState(null);

  const execute = useCallback(() => {
    setStatus('pending');
    setData(null);
    setError(null);

    return asyncFunction()
      .then(response => {
        setData(response);
        setStatus('success');
        return response;
      })
      .catch(error => {
        setError(error);
        setStatus('error');
        throw error;
      });
  }, [asyncFunction]);

  useEffect(() => {
    if (immediate) {
      execute();
    }
  }, [execute, immediate]);

  return {
    execute,
    status,
    data,
    error,
    isIdle: status === 'idle',
    isPending: status === 'pending',
    isSuccess: status === 'success',
    isError: status === 'error'
  };
}

// Usage
function UserProfile({ userId }) {
  const {
    data: user,
    error,
    isPending,
    execute
  } = useAsync(() => fetchUser(userId));

  if (isPending) return <Spinner />;
  if (error) return <Error error={error} onRetry={execute} />;
  return <div>{user.name}</div>;
}
```

## 3.7 useImperativeHandle (Rare but Powerful)

```javascript
// Exposing imperative methods from child
const FancyInput = forwardRef((props, ref) => {
  const inputRef = useRef();
  const [isFocused, setIsFocused] = useState(false);

  useImperativeHandle(ref, () => ({
    focus: () => {
      inputRef.current.focus();
      setIsFocused(true);
    },
    blur: () => {
      inputRef.current.blur();
      setIsFocused(false);
    },
    getValue: () => inputRef.current.value
  }));

  return (
    <input
      ref={inputRef}
      {...props}
      className={isFocused ? 'focused' : ''}
    />
  );
});

// Parent usage
function Form() {
  const inputRef = useRef();

  const handleSubmit = () => {
    console.log(inputRef.current.getValue());
  };

  useEffect(() => {
    inputRef.current.focus();
  }, []);

  return (
    <form onSubmit={handleSubmit}>
      <FancyInput ref={inputRef} />
    </form>
  );
}
```

## Real-World Scenario: Building a Data Fetching Hook

### The Challenge

Build a production-ready data fetching hook that handles:
- Loading, error, and success states
- Request cancellation
- Retries with exponential backoff
- Caching
- Optimistic updates
- Pagination
- Polling

### Your Task

```javascript
function useQuery(key, fetcher, options = {}) {
  // Implement all the features above
}

// Should work like this:
function UserList() {
  const {
    data,
    error,
    isLoading,
    refetch,
    fetchMore
  } = useQuery(
    'users',
    () => fetchUsers(),
    {
      retry: 3,
      cacheTime: 5 * 60 * 1000,
      staleTime: 1 * 60 * 1000,
      refetchOnWindowFocus: true
    }
  );
}
```

## Chapter Exercise: Build a Form Library

Create a custom hooks-based form library with:

**Requirements:**
1. `useForm` hook for form state
2. `useField` hook for individual fields
3. Validation (sync and async)
4. Error handling
5. Touched/dirty tracking
6. Submit handling
7. Reset functionality

**Evaluation:**
- Clean API design
- TypeScript support
- Proper cleanup
- No unnecessary re-renders
- Comprehensive examples

## Review Checklist

- [ ] Understand when to use functional updates with useState
- [ ] Know when to use lazy initialization
- [ ] Master useEffect dependencies and cleanup
- [ ] Avoid common useEffect anti-patterns
- [ ] Use useCallback/useMemo appropriately
- [ ] Choose between useState and useReducer correctly
- [ ] Leverage useRef for mutable values
- [ ] Compose custom hooks effectively
- [ ] Design clean hook APIs

## Key Takeaways

1. **Hooks are about logic reuse** - Extract and compose
2. **Dependencies matter** - Never lie to the dependencies array
3. **Cleanup is critical** - Always cleanup effects
4. **Don't over-optimize** - Measure before memoizing
5. **useReducer for complexity** - Multiple related state updates
6. **Custom hooks are powerful** - Build your own abstractions
7. **Composition > Configuration** - Combine simple hooks

## Further Reading

- React docs: Hooks API Reference
- "A Complete Guide to useEffect" by Dan Abramov
- "usehooks.com" - Collection of custom hooks
- React Query documentation (advanced data fetching)

## Next Chapter

[Chapter 4: Performance Profiling & Optimization](./04-performance-optimization.md)
