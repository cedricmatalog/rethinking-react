# Chapter 6: Memory Management & Debugging

## Introduction

Junior developers fix bugs by trial and error. Senior developers systematically debug, understand root causes, and prevent future issues.

This chapter teaches you to debug like a detective and manage memory like a professional.

## Learning Objectives

- Identify and fix memory leaks
- Use Chrome DevTools effectively
- Debug complex React issues
- Understand React internals for better debugging
- Build debugging tools and strategies

## 6.1 Memory Leaks in React

### Common Causes

```javascript
// Leak 1: Forgotten cleanup in useEffect
function Component() {
  const [data, setData] = useState([]);

  useEffect(() => {
    const interval = setInterval(() => {
      fetch('/api/data').then(r => r.json()).then(setData);
    }, 1000);

    // Missing cleanup! Interval keeps running after unmount
  }, []);

  return <div>{data.length} items</div>;
}

// Fix: Always cleanup
function Component() {
  const [data, setData] = useState([]);

  useEffect(() => {
    const interval = setInterval(() => {
      fetch('/api/data').then(r => r.json()).then(setData);
    }, 1000);

    return () => clearInterval(interval); // ✓ Cleanup
  }, []);

  return <div>{data.length} items</div>;
}

// Leak 2: Event listeners
function Component() {
  useEffect(() => {
    const handleResize = () => console.log('resized');
    window.addEventListener('resize', handleResize);

    // Missing cleanup!
  }, []);
}

// Fix
function Component() {
  useEffect(() => {
    const handleResize = () => console.log('resized');
    window.addEventListener('resize', handleResize);

    return () => window.removeEventListener('resize', handleResize);
  }, []);
}

// Leak 3: Async operations after unmount
function Component() {
  const [data, setData] = useState(null);

  useEffect(() => {
    fetchData().then(result => {
      setData(result); // Called even after unmount!
    });
  }, []);
}

// Fix: Cancel flag
function Component() {
  const [data, setData] = useState(null);

  useEffect(() => {
    let cancelled = false;

    fetchData().then(result => {
      if (!cancelled) {
        setData(result);
      }
    });

    return () => {
      cancelled = true;
    };
  }, []);
}
```

### Detecting Memory Leaks

```javascript
// Use Chrome DevTools Memory Profiler
// 1. Open DevTools > Memory tab
// 2. Take heap snapshot
// 3. Perform actions (navigate, open modals, etc.)
// 4. Take another snapshot
// 5. Compare snapshots
// 6. Look for growing arrays, listeners, timers

// Custom memory monitor
function useMemoryMonitor(componentName) {
  useEffect(() => {
    const initialMemory = performance.memory?.usedJSHeapSize;

    return () => {
      const finalMemory = performance.memory?.usedJSHeapSize;
      const diff = finalMemory - initialMemory;

      if (diff > 1000000) { // 1MB
        console.warn(
          `${componentName} may have memory leak: +${(diff / 1024 / 1024).toFixed(2)}MB`
        );
      }
    };
  }, [componentName]);
}

// Usage
function MyComponent() {
  useMemoryMonitor('MyComponent');
  // component code
}
```

## 6.2 Chrome DevTools Mastery

### React DevTools

```javascript
// Profiler tab shows:
// - Component render times
// - Why components rendered
// - Props/state changes

// Components tab shows:
// - Component tree
// - Props and state
// - Hooks values
// - Source code location

// Useful for debugging:
// 1. Highlight updates when components render
// 2. Record why each component rendered
// 3. Filter components by name
// 4. Inspect hooks in detail
```

### Performance Tab

```javascript
// Record performance:
// 1. Open Performance tab
// 2. Click Record
// 3. Perform slow action
// 4. Stop recording
// 5. Analyze timeline

// Look for:
// - Long tasks (>50ms)
// - Forced reflows
// - Layout thrashing
// - Paint operations
// - JavaScript execution time

// Example: Finding layout thrashing
function Component({ items }) {
  const updateHeights = () => {
    items.forEach(item => {
      // Reading layout
      const height = item.ref.offsetHeight;

      // Writing layout - causes reflow!
      item.ref.style.height = height + 'px';
    });
  };

  // Fix: Batch reads and writes
  const updateHeightsBetter = () => {
    // Read all
    const heights = items.map(item => item.ref.offsetHeight);

    // Write all
    items.forEach((item, i) => {
      item.ref.style.height = heights[i] + 'px';
    });
  };
}
```

### Memory Tab

```javascript
// Heap Snapshot
// - Shows all objects in memory
// - Find retained objects
// - Identify detached DOM nodes

// Allocation Timeline
// - Shows memory allocations over time
// - Find memory spikes
// - Correlate with user actions

// Allocation Sampling
// - Lightweight profiling
// - Shows function call stacks
// - Find allocation hotspots
```

## 6.3 Debugging Strategies

### The Systematic Approach

```javascript
// 1. Reproduce the bug consistently
// 2. Isolate the problem
// 3. Form a hypothesis
// 4. Test the hypothesis
// 5. Fix and verify
// 6. Prevent future occurrences

// Example: Debugging state not updating
function BuggyComponent() {
  const [count, setCount] = useState(0);

  const increment = () => {
    setCount(count + 1);
    setCount(count + 1); // Why doesn't this work?
  };

  // Debug process:
  // 1. Add console.log
  console.log('Render with count:', count);

  // 2. Add debugger
  const increment = () => {
    debugger; // Pause here
    setCount(count + 1);
    setCount(count + 1);
  };

  // 3. Check React DevTools
  // - See count value
  // - See render count

  // 4. Hypothesis: Closure over stale count
  // 5. Test: Use functional update
  const incrementFixed = () => {
    setCount(c => c + 1);
    setCount(c => c + 1); // Works!
  };

  return <button onClick={incrementFixed}>{count}</button>;
}
```

### Debug Hooks

```javascript
// useDebugValue - show in DevTools
function useCustomHook(value) {
  const processed = expensiveOperation(value);

  useDebugValue(processed, value => {
    return `Processed: ${value}`;
  });

  return processed;
}

// useWhyDidYouUpdate - find unnecessary renders
function useWhyDidYouUpdate(name, props) {
  const previousProps = useRef();

  useEffect(() => {
    if (previousProps.current) {
      const allKeys = Object.keys({ ...previousProps.current, ...props });
      const changes = {};

      allKeys.forEach(key => {
        if (previousProps.current[key] !== props[key]) {
          changes[key] = {
            from: previousProps.current[key],
            to: props[key]
          };
        }
      });

      if (Object.keys(changes).length > 0) {
        console.log('[why-did-you-update]', name, changes);
      }
    }

    previousProps.current = props;
  });
}

// useTraceUpdate - track prop changes
function useTraceUpdate(props) {
  const prev = useRef(props);

  useEffect(() => {
    const changedProps = Object.entries(props).reduce((acc, [key, val]) => {
      if (prev.current[key] !== val) {
        acc[key] = [prev.current[key], val];
      }
      return acc;
    }, {});

    if (Object.keys(changedProps).length > 0) {
      console.log('Changed props:', changedProps);
    }

    prev.current = props;
  });
}
```

### Error Boundaries

```javascript
class ErrorBoundary extends React.Component {
  state = { hasError: false, error: null, errorInfo: null };

  static getDerivedStateFromError(error) {
    return { hasError: true };
  }

  componentDidCatch(error, errorInfo) {
    // Log to error reporting service
    console.error('Error caught:', error, errorInfo);

    // Log component stack
    console.log('Component stack:', errorInfo.componentStack);

    this.setState({ error, errorInfo });

    // Send to monitoring service
    if (window.Sentry) {
      window.Sentry.captureException(error, {
        contexts: {
          react: {
            componentStack: errorInfo.componentStack
          }
        }
      });
    }
  }

  render() {
    if (this.state.hasError) {
      return (
        <div>
          <h2>Something went wrong</h2>
          <details>
            <summary>Error details</summary>
            <pre>{this.state.error?.toString()}</pre>
            <pre>{this.state.errorInfo?.componentStack}</pre>
          </details>
          <button onClick={() => this.setState({ hasError: false })}>
            Try again
          </button>
        </div>
      );
    }

    return this.props.children;
  }
}

// Usage with granular boundaries
function App() {
  return (
    <ErrorBoundary>
      <Header />

      <ErrorBoundary>
        <Sidebar />
      </ErrorBoundary>

      <ErrorBoundary>
        <MainContent />
      </ErrorBoundary>
    </ErrorBoundary>
  );
}
```

## 6.4 React Internals for Debugging

### Understanding Reconciliation

```javascript
// React compares:
// 1. Element type
// 2. Key
// 3. Props

// Different types = unmount + mount
<div>Hello</div>  →  <span>Hello</span>
// Unmounts div, mounts span

// Same type = update
<div className="before">Hello</div>  →  <div className="after">Hello</div>
// Updates className

// Understanding keys
const items = ['a', 'b', 'c'];

// Without keys - reuses elements
items.map((item, index) => <Item key={index}>{item}</Item>)
// If 'a' removed, 'b' updates to 'a', 'c' updates to 'b', last removed

// With keys - updates correctly
items.map(item => <Item key={item}>{item}</Item>)
// If 'a' removed, removes 'a' element, keeps 'b' and 'c'
```

### Fiber Architecture

```javascript
// React Fiber enables:
// - Incremental rendering
// - Pause, abort, reuse work
// - Priority to different updates
// - Concurrent features

// Debugging with Fiber
// React DevTools shows fiber tree
// Each fiber has:
// - type (function/class/host)
// - key
// - props
// - state
// - effectTag (what changed)

// Understanding update priorities
// - Immediate: User input, clicks
// - User-blocking: Hover, scrolling
// - Normal: Network responses
// - Low: Analytics, logging
// - Idle: Background work
```

## 6.5 Common Bug Patterns

### Stale Closures

```javascript
// Bug: Stale closure
function Component() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const interval = setInterval(() => {
      console.log(count); // Always logs 0!
      setCount(count + 1); // Only increments once!
    }, 1000);

    return () => clearInterval(interval);
  }, []); // Empty deps - closure over initial count

  // Fix 1: Functional update
  useEffect(() => {
    const interval = setInterval(() => {
      setCount(c => c + 1); // Uses current count
    }, 1000);

    return () => clearInterval(interval);
  }, []);

  // Fix 2: Add to dependencies
  useEffect(() => {
    const interval = setInterval(() => {
      console.log(count); // Logs current count
      setCount(count + 1);
    }, 1000);

    return () => clearInterval(interval);
  }, [count]); // Re-run when count changes

  // Fix 3: Use ref
  const countRef = useRef(count);
  countRef.current = count;

  useEffect(() => {
    const interval = setInterval(() => {
      console.log(countRef.current); // Always current
      setCount(c => c + 1);
    }, 1000);

    return () => clearInterval(interval);
  }, []);
}
```

### Infinite Loops

```javascript
// Bug: Infinite loop
function Component() {
  const [data, setData] = useState([]);

  useEffect(() => {
    fetch('/api/data')
      .then(r => r.json())
      .then(setData); // Triggers re-render
  }); // No deps - runs after every render!

  // Fix: Add dependencies
  useEffect(() => {
    fetch('/api/data')
      .then(r => r.json())
      .then(setData);
  }, []); // Only on mount

  // Bug: Object/array in dependencies
  useEffect(() => {
    console.log('Effect ran');
  }, [{ userId: 123 }]); // New object every render!

  // Fix: Primitive values or memoized objects
  const userId = 123;
  useEffect(() => {
    console.log('Effect ran');
  }, [userId]); // Primitive

  const user = useMemo(() => ({ userId: 123 }), []);
  useEffect(() => {
    console.log('Effect ran');
  }, [user]); // Memoized
}
```

### Race Conditions

```javascript
// Bug: Race condition
function Component({ userId }) {
  const [user, setUser] = useState(null);

  useEffect(() => {
    fetchUser(userId).then(setUser);
    // If userId changes quickly, responses may arrive out of order
  }, [userId]);

  // Fix: Ignore stale responses
  useEffect(() => {
    let cancelled = false;

    fetchUser(userId).then(data => {
      if (!cancelled) {
        setUser(data);
      }
    });

    return () => {
      cancelled = true;
    };
  }, [userId]);

  // Better: Use abort controller
  useEffect(() => {
    const controller = new AbortController();

    fetch(`/api/users/${userId}`, {
      signal: controller.signal
    })
      .then(r => r.json())
      .then(setUser)
      .catch(err => {
        if (err.name !== 'AbortError') {
          console.error(err);
        }
      });

    return () => controller.abort();
  }, [userId]);
}
```

## 6.6 Production Debugging

### Source Maps

```javascript
// Enable in production (securely)
// webpack.config.js
module.exports = {
  devtool: 'hidden-source-map', // or 'source-map'
  // Upload source maps to error tracking service
  // Don't serve them publicly
};

// Access source maps in Sentry, DataDog, etc.
// See original code in error stack traces
```

### Logging Strategy

```javascript
// Development: Verbose
if (process.env.NODE_ENV === 'development') {
  console.log('Component rendered with props:', props);
}

// Production: Structured logging
const logger = {
  info: (message, context) => {
    if (window.analytics) {
      window.analytics.track('Log', {
        level: 'info',
        message,
        context,
        timestamp: Date.now()
      });
    }
  },

  error: (error, context) => {
    console.error(error);

    if (window.Sentry) {
      window.Sentry.captureException(error, {
        contexts: { custom: context }
      });
    }
  },

  warn: (message, context) => {
    console.warn(message, context);

    if (window.analytics) {
      window.analytics.track('Warning', {
        message,
        context
      });
    }
  }
};

// Usage
function Component() {
  useEffect(() => {
    logger.info('Component mounted', {
      componentName: 'Component',
      props: { userId: props.userId }
    });
  }, []);

  const handleError = (error) => {
    logger.error(error, {
      action: 'fetchData',
      userId: props.userId
    });
  };
}
```

### Feature Flags for Debugging

```javascript
// Enable debug features in production
function useDebugMode() {
  const [isDebug, setIsDebug] = useState(() => {
    return localStorage.getItem('debug') === 'true';
  });

  useEffect(() => {
    const handler = (e) => {
      if (e.key === 'd' && e.ctrlKey && e.shiftKey) {
        const newValue = !isDebug;
        setIsDebug(newValue);
        localStorage.setItem('debug', String(newValue));
      }
    };

    window.addEventListener('keydown', handler);
    return () => window.removeEventListener('keydown', handler);
  }, [isDebug]);

  return isDebug;
}

function Component() {
  const isDebug = useDebugMode();

  if (isDebug) {
    return (
      <div>
        <DebugPanel />
        <ActualComponent />
      </div>
    );
  }

  return <ActualComponent />;
}
```

## Real-World Scenario: Memory Leak Hunt

### The Challenge

Production app starts slow after 30 minutes of use:
- Memory grows from 50MB to 500MB
- App becomes unresponsive
- No obvious errors in console

### Your Investigation

1. **Reproduce locally**
2. **Take memory snapshots**
3. **Compare snapshots**
4. **Find growing objects**
5. **Trace to source**
6. **Fix and verify**

### Common Culprits

```javascript
// Found: Event listeners not cleaned up
// Found: setInterval without clearInterval
// Found: WebSocket connections not closed
// Found: Large arrays growing unbounded
// Found: Cache without size limit
```

## Chapter Exercise: Debug This App

```javascript
// This app has multiple bugs and memory leaks
function BuggyApp() {
  const [users, setUsers] = useState([]);
  const [filter, setFilter] = useState('');

  useEffect(() => {
    setInterval(() => {
      fetch('/api/users')
        .then(r => r.json())
        .then(data => setUsers(users.concat(data)));
    }, 5000);
  }, []);

  const filtered = users.filter(u =>
    u.name.includes(filter)
  );

  return (
    <div>
      <input value={filter} onChange={e => setFilter(e.target.value)} />
      {filtered.map((user, idx) => (
        <UserCard key={idx} user={user} />
      ))}
    </div>
  );
}

// Tasks:
// 1. Find ALL bugs
// 2. Fix each one
// 3. Explain why it was a bug
// 4. Add safeguards to prevent similar bugs
```

## Review Checklist

- [ ] Identify and fix memory leaks
- [ ] Use Chrome DevTools effectively
- [ ] Debug with React DevTools
- [ ] Understand common bug patterns
- [ ] Implement error boundaries
- [ ] Set up production debugging
- [ ] Create debugging utilities
- [ ] Monitor memory usage

## Key Takeaways

1. **Always cleanup** - useEffect cleanup is critical
2. **Debug systematically** - Don't guess
3. **Use the tools** - DevTools are powerful
4. **Understand internals** - Helps debugging
5. **Prevent don't just fix** - Add safeguards
6. **Log strategically** - In production too
7. **Memory leaks are common** - Watch for them
8. **Test for leaks** - Before production

## Further Reading

- Chrome DevTools documentation
- React Fiber architecture
- "Debugging JavaScript" by Paul Irish
- Memory leak patterns and prevention

## Next Chapter

[Chapter 7: Component Architecture](./07-component-architecture.md)
