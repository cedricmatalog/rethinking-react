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

### Junior Perspective
"I use useState for everything! Just add more useState calls when I need more state."

### Senior Perspective
"useState is powerful but requires discipline: group related state, use functional updates when reading previous state, and lazy-initialize expensive computations. Know when to switch to useReducer for complex state logic."

### Beyond Basic State: Grouping Related Values

**The Problem:** Managing many related state variables

```javascript
// Junior: Separate state for everything
function Form() {
  const [firstName, setFirstName] = useState('');
  const [lastName, setLastName] = useState('');
  const [email, setEmail] = useState('');
  const [phone, setPhone] = useState('');
  const [address, setAddress] = useState('');
  const [city, setCity] = useState('');
  const [zipCode, setZipCode] = useState('');
  const [country, setCountry] = useState('');
  // ... 10 more fields

  return (/* form */);
}
```

**What's wrong:**
- 15+ state variables for one form
- Hard to reset form (need 15 setters)
- Difficult to validate across fields
- Can't easily pass form data to API

**Visual representation:**

```
Junior Approach - State Explosion:
┌────────────────────────────────┐
│        Form Component          │
├────────────────────────────────┤
│  useState('') → firstName      │
│  useState('') → lastName       │
│  useState('') → email          │
│  useState('') → phone          │
│  useState('') → address        │
│  useState('') → city           │
│  useState('') → zipCode        │
│  useState('') → country        │
│  useState('') → ...10 more     │
├────────────────────────────────┤
│  15 state variables!           │
│  15 setters to manage!         │
│  Hard to sync!                 │
└────────────────────────────────┘
```

**Senior Solution: Grouped State**

```javascript
// Senior: Grouped related state
function Form() {
  const [formData, setFormData] = useState({
    firstName: '',
    lastName: '',
    email: '',
    phone: '',
    address: '',
    city: '',
    zipCode: '',
    country: ''
  });

  // Single reusable update function
  const updateField = (field, value) => {
    setFormData(prev => ({ ...prev, [field]: value }));
  };

  // Easy reset
  const resetForm = () => {
    setFormData({
      firstName: '',
      lastName: '',
      email: '',
      phone: '',
      address: '',
      city: '',
      zipCode: '',
      country: ''
    });
  };

  // Easy API submission
  const handleSubmit = () => {
    api.post('/submit', formData); // One object!
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        value={formData.firstName}
        onChange={(e) => updateField('firstName', e.target.value)}
      />
      {/* ... other fields */}
    </form>
  );
}
```

**Visual representation:**

```
Senior Approach - Grouped State:
┌────────────────────────────────┐
│        Form Component          │
├────────────────────────────────┤
│  useState({                    │
│    firstName: '',              │
│    lastName: '',               │
│    email: '',                  │
│    ...all fields               │
│  }) → formData                 │
├────────────────────────────────┤
│  ✅ 1 state variable            │
│  ✅ 1 setter (updateField)      │
│  ✅ Easy to reset               │
│  ✅ Easy to validate            │
│  ✅ Easy to submit              │
└────────────────────────────────┘
```

**When to group vs separate:**
- **Group:** Related data that changes together (form fields, user profile)
- **Separate:** Independent concerns (isLoading, selectedTab, theme)

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

### Exercise 3.1: Building useUndo Hook

**Challenge:** Build a `useUndo` hook that provides undo/redo functionality for any state.

**Requirements:**
1. Maintain history of state changes
2. Provide undo/redo functions
3. Track whether undo/redo is possible
4. Limit history size (prevent memory leaks)
5. Reset history function

**Think About:**
- How do you store history? Array? Two arrays (past/future)?
- What happens when you undo then make a new change?
- How do you prevent unlimited history growth?

<details>
<summary>💡 Hint: State structure</summary>

**Consider this structure:**
```javascript
{
  past: [state1, state2, state3],
  present: state4,
  future: [state5, state6]
}
```

When you **undo:**
- Move present to future
- Pop from past to present

When you **set:**
- Move present to past
- Clear future (can't redo after new change!)

</details>

<details>
<summary>✅ Solution</summary>

```javascript
function useUndo(initialState, maxHistorySize = 50) {
  const [state, setState] = useState({
    past: [],
    present: initialState,
    future: []
  });

  const canUndo = state.past.length > 0;
  const canRedo = state.future.length > 0;

  const set = useCallback((newPresent) => {
    setState(currentState => {
      const newPast = [...currentState.past, currentState.present];

      // Limit history size
      if (newPast.length > maxHistorySize) {
        newPast.shift();
      }

      return {
        past: newPast,
        present: newPresent,
        future: [] // Clear future on new change
      };
    });
  }, [maxHistorySize]);

  const undo = useCallback(() => {
    setState(currentState => {
      if (currentState.past.length === 0) return currentState;

      const previous = currentState.past[currentState.past.length - 1];
      const newPast = currentState.past.slice(0, currentState.past.length - 1);

      return {
        past: newPast,
        present: previous,
        future: [currentState.present, ...currentState.future]
      };
    });
  }, []);

  const redo = useCallback(() => {
    setState(currentState => {
      if (currentState.future.length === 0) return currentState;

      const next = currentState.future[0];
      const newFuture = currentState.future.slice(1);

      return {
        past: [...currentState.past, currentState.present],
        present: next,
        future: newFuture
      };
    });
  }, []);

  const reset = useCallback((newPresent = initialState) => {
    setState({
      past: [],
      present: newPresent,
      future: []
    });
  }, [initialState]);

  return [
    state.present,
    set,
    {
      undo,
      redo,
      canUndo,
      canRedo,
      reset,
      history: state.past.length
    }
  ];
}

// Usage
function DrawingApp() {
  const [drawing, setDrawing, { undo, redo, canUndo, canRedo, history }] =
    useUndo([]);

  const addPoint = (point) => {
    setDrawing([...drawing, point]);
  };

  return (
    <>
      <div>History: {history} changes</div>
      <button onClick={undo} disabled={!canUndo}>
        Undo
      </button>
      <button onClick={redo} disabled={!canRedo}>
        Redo
      </button>
      <Canvas points={drawing} onAddPoint={addPoint} />
    </>
  );
}
```

**Key Observations:**
- Uses single useState with past/present/future structure
- `useCallback` prevents recreating functions on every render
- Setting new state clears future (standard undo/redo behavior)
- `maxHistorySize` prevents memory leaks
- Returns helpers object for metadata (canUndo, canRedo, etc.)

**What You Learned:**
- Complex state can be managed with useState
- Functional updates ensure you read latest state
- useCallback for stable function references
- Limit array growth to prevent memory issues

</details>

---

## 3.2 useEffect Mastery

**🧠 Quick Recall (from 3.1):** Before diving in, test your memory: When should you use functional updates with setState? What's the risk of not using them?

<details>
<summary>Check your answer</summary>

**When to use functional updates:**
```javascript
// Use when you need to read previous state
setCount(prev => prev + 1);  // ✓ Always gets latest
setCount(count + 1);          // ✗ Might use stale value
```

**Risk of not using them:**
If you call setState multiple times in one event handler, without functional updates, they all read the same "count" value:
```javascript
// All read count=0, so count ends up as 1
setCount(count + 1);  // 0 + 1
setCount(count + 1);  // 0 + 1
setCount(count + 1);  // 0 + 1
```

With functional updates, each reads the result of the previous:
```javascript
setCount(c => c + 1);  // 0 + 1 = 1
setCount(c => c + 1);  // 1 + 1 = 2
setCount(c => c + 1);  // 2 + 1 = 3
```

Ready to master useEffect!
</details>

---

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

### Exercise 3.2: Building useDebounce Hook

**Challenge:** Build a `useDebounce` hook that delays updating a value until after a specified delay.

<details>
<summary>✅ Solution</summary>

```javascript
function useDebounce(value, delay) {
  const [debouncedValue, setDebouncedValue] = useState(value);

  useEffect(() => {
    // Set up timer
    const timer = setTimeout(() => {
      setDebouncedValue(value);
    }, delay);

    // Cleanup: cancel timer if value changes before delay
    return () => clearTimeout(timer);
  }, [value, delay]);

  return debouncedValue;
}

// Usage
function SearchBox() {
  const [search, setSearch] = useState('');
  const debouncedSearch = useDebounce(search, 500);

  useEffect(() => {
    if (debouncedSearch) {
      searchAPI(debouncedSearch);  // Only calls API after 500ms of no typing
    }
  }, [debouncedSearch]);

  return <input value={search} onChange={e => setSearch(e.target.value)} />;
}
```

**What You Learned:**
- useEffect cleanup runs when dependencies change
- Debouncing prevents excessive API calls
- Custom hooks encapsulate reusable logic

</details>

---

## 💥 Real War Story: The useEffect Dependency Hell

**Company:** E-commerce platform (8M users)
**Date:** March 2023
**Hook Misuse:** Missing dependencies causing data corruption

### What Happened

A team built a product recommendation feature that fetched personalized suggestions based on user browsing history.

```javascript
// Their code - looks innocent!
function RecommendationsWidget({ userId }) {
  const [recommendations, setRecommendations] = useState([]);
  const [preferences, setPreferences] = useState(null);

  // Fetch user preferences
  useEffect(() => {
    fetchPreferences(userId).then(setPreferences);
  }, [userId]); // ✓ Correct

  // Fetch recommendations based on preferences
  useEffect(() => {
    if (preferences) {
      fetchRecommendations(userId, preferences).then(setRecommendations);
    }
  }, [preferences]); // ❌ MISSING userId!
}
```

**The Bug:**
- User A (ID=123) visits → fetches preferences for 123
- Recommendations load for user 123
- User navigates to user B's profile (ID=456)
- Preferences fetch for user 456
- **BUT:** Recommendations still use old fetchRecommendations with userId=123!
- User B sees user A's recommendations!

**Why It Happened:**
```javascript
useEffect(() => {
  if (preferences) {
    fetchRecommendations(userId, preferences).then(setRecommendations);
    //                  ^^^^^^ This userId is from CLOSURE!
  }
}, [preferences]); // userId not in deps → stale closure bug!
```

When `preferences` changes but `userId` doesn't change yet, the effect doesn't re-run.
When `userId` changes, `preferences` is still the old user's preferences!

### The Impact

**Data Leak Discovered:**
- Running for 3 weeks before caught
- 47,000 users saw wrong recommendations
- Some saw products from users in different countries (wrong currency/language)
- 12 customer support tickets about "weird recommendations"
- GDPR concern: showing user A's data to user B

**Revenue Impact:**
- Recommendation click-through rate dropped 34%
- Estimated $180,000 in lost sales over 3 weeks
- 2 days of engineering time to fix + audit similar issues

### The Fix

```javascript
function RecommendationsWidget({ userId }) {
  const [recommendations, setRecommendations] = useState([]);
  const [preferences, setPreferences] = useState(null);

  useEffect(() => {
    fetchPreferences(userId).then(setPreferences);
  }, [userId]);

  useEffect(() => {
    if (preferences) {
      fetchRecommendations(userId, preferences).then(setRecommendations);
    }
  }, [userId, preferences]); // ✅ BOTH dependencies!
  //    ^^^^^^ ADDED THIS!
}
```

**Better: Single Effect**
```javascript
useEffect(() => {
  let cancelled = false;

  (async () => {
    const prefs = await fetchPreferences(userId);
    if (cancelled) return;

    const recs = await fetchRecommendations(userId, prefs);
    if (cancelled) return;

    setRecommendations(recs);
  })();

  return () => { cancelled = true; };
}, [userId]); // Single source of truth!
```

### Lessons Learned

From their post-mortem:

> "ESLint's exhaustive-deps rule would have caught this. We had it disabled because it was 'annoying'. That $180k mistake taught us to never disable linter rules without understanding why they exist."

**What They Changed:**

1. **Enabled react-hooks/exhaustive-deps** (required in CI)
   ```json
   {
     "rules": {
       "react-hooks/exhaustive-deps": "error"  // Was "off"
     }
   }
   ```

2. **Code Review Checklist:**
   - [ ] All useEffect dependencies declared?
   - [ ] Using values from closure? Add to deps!
   - [ ] Cleanup function needed?

3. **Testing:**
   - Added tests that change props mid-test
   - Caught 8 more similar bugs before production

4. **Training:**
   - Mandatory "Hooks Deep Dive" training for all React devs
   - Quiz on closures and dependencies (must score 100%)

---

**The Key Rule:**

```
Every value used inside useEffect that can change
between re-renders MUST be in the dependency array.

No exceptions. No "it works fine without it".
The linter is right. You are wrong.
```

**Common Excuses (All Wrong):**
- ❌ "But it works in testing" → Works until it doesn't
- ❌ "userId won't change" → Props can always change
- ❌ "Too many re-renders" → Fix the cause, don't hide it
- ❌ "Linter is annoying" → $180k mistake says otherwise

---

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

---

## 🚫 Common Hooks Mistakes Gallery

**Real patterns from thousands of code reviews.**

### Mistake #1: Forgetting useCallback Deps

```javascript
// ❌ BAD - useCallback with missing deps
function Parent({ userId }) {
  const fetchData = useCallback(() => {
    api.get(`/users/${userId}`);  // userId from closure!
  }, []); // Empty deps - fetchData never updates!

  return <Child onFetch={fetchData} />;
}
```

**Fix:**
```javascript
const fetchData = useCallback(() => {
  api.get(`/users/${userId}`);
}, [userId]); // ✅ Include all closure values
```

---

### Mistake #2: useState for Derived Values

```javascript
// ❌ BAD - Unnecessary state + useEffect
function Cart({ items }) {
  const [total, setTotal] = useState(0);

  useEffect(() => {
    setTotal(items.reduce((sum, item) => sum + item.price, 0));
  }, [items]);

  return <div>{total}</div>;
}
```

**Fix:**
```javascript
// ✅ GOOD - Compute during render
function Cart({ items }) {
  const total = items.reduce((sum, item) => sum + item.price, 0);
  return <div>{total}</div>;
}
```

---

### Mistake #3: Not Cleaning Up Subscriptions

```javascript
// ❌ BAD - Memory leak!
function Chat({ roomId }) {
  useEffect(() => {
    const connection = createConnection(roomId);
    connection.on('message', handleMessage);
    // No cleanup - connection stays open!
  }, [roomId]);
}
```

**Fix:**
```javascript
useEffect(() => {
  const connection = createConnection(roomId);
  connection.on('message', handleMessage);

  return () => {
    connection.off('message', handleMessage);
    connection.close();
  };
}, [roomId]);
```

---

### Mistake #4: useMemo for Everything

```javascript
// ❌ BAD - Over-optimization
function Component({ name }) {
  const greeting = useMemo(() => `Hello, ${name}`, [name]);
  const doubled = useMemo(() => count * 2, [count]);

  // useMemo costs MORE than the operation!
}
```

**Fix:**
```javascript
// ✅ GOOD - Simple operations don't need memoization
function Component({ name }) {
  const greeting = `Hello, ${name}`;
  const doubled = count * 2;
}
```

---

### Mistake #5: Async useEffect Without Cancel

```javascript
// ❌ BAD - Race condition!
function Profile({ userId }) {
  const [user, setUser] = useState(null);

  useEffect(() => {
    fetchUser(userId).then(setUser);
    // If userId changes quickly, old fetch might finish last!
  }, [userId]);
}
```

**Fix:**
```javascript
useEffect(() => {
  let cancelled = false;

  fetchUser(userId).then(data => {
    if (!cancelled) setUser(data);
  });

  return () => { cancelled = true; };
}, [userId]);
```

---

### Mistake #6: setState in Render

```javascript
// ❌ BAD - Causes infinite loop!
function Component({ externalValue }) {
  const [value, setValue] = useState(externalValue);

  if (externalValue !== value) {
    setValue(externalValue); // setState during render!
  }

  return <div>{value}</div>;
}
```

**Fix:**
```javascript
// ✅ GOOD - Use the prop directly or useEffect
function Component({ externalValue }) {
  return <div>{externalValue}</div>;
}

// Or if you need to transform it:
function Component({ externalValue }) {
  const [value, setValue] = useState(externalValue);

  useEffect(() => {
    setValue(externalValue);
  }, [externalValue]);

  return <div>{value}</div>;
}
```

---

## 🧠 Cumulative Review: Master All Hooks

**Test yourself on all 7 sections:**

### Question 1: When would you use useReducer instead of useState?

<details>
<summary>✅ Answer</summary>

**Use useReducer when:**
- Multiple related state values that update together
- Complex state transitions with validation
- Next state depends on previous state in complex ways
- State machine patterns
- Multiple ways to update the same state

**Example:**
```javascript
// useState - gets messy
const [name, setName] = useState('');
const [email, setEmail] = useState('');
const [errors, setErrors] = useState({});
const [touched, setTouched] = useState({});
const [isSubmitting, setIsSubmitting] = useState(false);

// useReducer - cleaner
const [state, dispatch] = useReducer(formReducer, initialState);
```

</details>

---

### Question 2: This code has a memory leak. Where?

```javascript
function Timer() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const id = setInterval(() => {
      setCount(count + 1);
    }, 1000);
  }, []);

  return <div>{count}</div>;
}
```

<details>
<summary>✅ Answer</summary>

**Two bugs:**

1. **Memory leak:** No cleanup function
2. **Stale closure:** `count` in closure won't update

**Fix:**
```javascript
useEffect(() => {
  const id = setInterval(() => {
    setCount(c => c + 1);  // Functional update
  }, 1000);

  return () => clearInterval(id);  // Cleanup!
}, []);
```

</details>

---

### Question 3: When should you use useCallback?

<details>
<summary>✅ Answer</summary>

**Use useCallback when:**
1. Passing function to memoized child component
2. Function is a dependency of useEffect/useMemo
3. Function is used as a prop in React.memo component

**Example:**
```javascript
const Parent = () => {
  const [count, setCount] = useState(0);

  // Without useCallback, Child re-renders on every Parent render
  const handleClick = useCallback(() => {
    console.log('clicked');
  }, []);

  return (
    <>
      <button onClick={() => setCount(count + 1)}>{count}</button>
      <ExpensiveChild onClick={handleClick} />
    </>
  );
};

const ExpensiveChild = memo(({ onClick }) => {
  // Only re-renders if onClick changes
});
```

**Don't use when:**
- Function isn't passed to other components
- Child isn't memoized
- Performance doesn't matter

</details>

---

### Question 4: What's wrong with this useEffect?

```javascript
function SearchResults({ query, filters }) {
  const [results, setResults] = useState([]);

  useEffect(() => {
    search(query, filters).then(setResults);
  }, [query]); // Missing filters!

  return <List items={results} />;
}
```

<details>
<summary>✅ Answer</summary>

**Problem:** Missing `filters` in dependency array!

When `filters` change but `query` doesn't, the effect won't re-run.
Results will be stale - searched with old filters.

**Fix:**
```javascript
useEffect(() => {
  search(query, filters).then(setResults);
}, [query, filters]); // ✅ Both dependencies
```

**Rule:** Every value from component scope used in effect must be in deps.

</details>

---

### Question 5: How do you prevent this race condition?

```javascript
function Profile({ userId }) {
  const [user, setUser] = useState(null);

  useEffect(() => {
    fetchUser(userId).then(setUser);
  }, [userId]);
}
// User clicks fast: user1 → user2 → user1
// Fetch order: fetch(1), fetch(2), fetch(1)
// But finish order might be: fetch(2) finishes last!
// Shows user2 when userId is 1!
```

<details>
<summary>✅ Answer</summary>

**Fix with cancellation flag:**
```javascript
useEffect(() => {
  let cancelled = false;

  fetchUser(userId).then(user => {
    if (!cancelled) setUser(user);
  });

  return () => { cancelled = true; };
}, [userId]);
```

When userId changes, cleanup runs and sets `cancelled = true`.
If old fetch finishes after cleanup, it won't call setUser.

</details>

---

### Question 6: Design a custom hook for this logic

You need to fetch data, handle loading/error states, and allow manual refetch. Design the hook API.

<details>
<summary>✅ Answer</summary>

**API Design:**
```javascript
function useFetch(url) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);

  const execute = useCallback(async () => {
    setLoading(true);
    setError(null);

    try {
      const response = await fetch(url);
      const json = await response.json();
      setData(json);
    } catch (err) {
      setError(err);
    } finally {
      setLoading(false);
    }
  }, [url]);

  useEffect(() => {
    execute();
  }, [execute]);

  return { data, loading, error, refetch: execute };
}

// Usage
const { data, loading, error, refetch } = useFetch('/api/users');
```

**Key decisions:**
- Returns object (easier to add new fields later)
- `execute` is useCallback (stable reference)
- Exposes `refetch` for manual trigger
- Auto-fetches on mount via useEffect

</details>

---

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
