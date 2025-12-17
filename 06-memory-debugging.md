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

### Junior vs Senior Perspective

**Junior Approach:**
"The app crashes after 30 minutes. I'll just add more try-catch blocks and maybe restart it every hour? Or add `window.location.reload()` as a workaround when memory gets high?"

**Senior Approach:**
"The app crashes after 30 minutes - that's a memory leak, not a random crash. Let me take a heap snapshot in Chrome DevTools, compare before/after navigating pages, and find which components aren't cleaning up. I'll bet it's an interval or event listener that wasn't removed on unmount."

**The Difference:**
- Junior: Treats symptoms (crashes) with workarounds (restarts, reloads)
- Senior: Diagnoses root cause (memory leak) using tools (heap snapshots), fixes permanently

**Key Insight:** Memory leaks are 100% preventable if you clean up every side effect. If it has setup, it needs cleanup.

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

---

## 💥 War Story: The Memory Leak That Crashed Black Friday

### The Disaster

**Company:** E-commerce platform (5M daily users)
**Team:** 25 frontend engineers
**Date:** Black Friday 2022, 11:47 PM (peak traffic)
**Impact:** Complete site outage, $4.2M in lost sales, 2.3M angry customers

### What Happened

Black Friday was going perfectly. Sales were 3× normal. Then at 11:47 PM (peak shopping time):

```
11:47 PM: First user reports: "Page won't load"
11:49 PM: 50 reports: "App frozen"
11:51 PM: 500 reports: "White screen"
11:52 PM: Load balancer health checks failing
11:53 PM: COMPLETE SITE OUTAGE

Engineers scrambled. Restarted servers. Site came back.
11:59 PM: Site down again.
12:04 AM: Down again.
12:08 AM: Down again.

Pattern: Site worked for 5-8 minutes, then crashed.
Duration of outage: 3 hours, 22 minutes
```

### The Investigation

**Initial confusion:**
- Backend was fine (API response times normal)
- Database was fine (CPU at 30%)
- CDN was fine (assets loading)
- **Frontend was the problem** - but why?

**The debug session (12:15 AM):**

Engineer took heap snapshot of the running app:

```
Memory Snapshot Analysis:

Initial load (fresh page):
  - Heap size: 52 MB ✅
  - Detached DOM nodes: 0 ✅

After 5 minutes of use:
  - Heap size: 380 MB ⚠️
  - Detached DOM nodes: 14,823 🚨
  - Growing objects: event listeners (45K+) 🚨

After 8 minutes:
  - Heap size: 512 MB 🚨 (browser limit reached)
  - Tab crashes
```

**The smoking gun:** Detached DOM nodes growing exponentially.

### The Root Cause

A seemingly innocent component was added 2 weeks before Black Friday:

```javascript
// ProductCard.tsx - Added Nov 10th, "Approved by 3 senior engineers"
function ProductCard({ product }) {
  const [isHovered, setIsHovered] = useState(false);

  // Track hover analytics
  useEffect(() => {
    const element = document.getElementById(`product-${product.id}`);

    const handleMouseEnter = () => {
      setIsHovered(true);
      analytics.track('product_hover', { id: product.id });
    };

    const handleMouseLeave = () => {
      setIsHovered(false);
    };

    element?.addEventListener('mouseenter', handleMouseEnter);
    element?.addEventListener('mouseleave', handleMouseLeave);

    // 🚨 MISSING CLEANUP! 🚨
    // return () => {
    //   element?.removeEventListener('mouseenter', handleMouseEnter);
    //   element?.removeEventListener('mouseleave', handleMouseLeave);
    // };
  }, [product.id]);

  return (
    <div id={`product-${product.id}`} className={isHovered ? 'hovered' : ''}>
      {/* ... */}
    </div>
  );
}
```

**What happened:**

```
User views product listing page (50 products):
  - 50 ProductCards mount
  - 100 event listeners added (mouseenter + mouseleave × 50)

User scrolls, new products render:
  - 50 more ProductCards mount
  - 100 MORE event listeners added
  - Old listeners NEVER removed (no cleanup!)

User navigates to /cart, then back to /products:
  - ProductCards unmount (components destroyed)
  - Event listeners stay attached to DOM 🚨
  - DOM nodes can't be garbage collected (listeners hold references)

After 10 page navigations (typical Black Friday shopping):
  - 1,000 ProductCards rendered total
  - 2,000 event listeners orphaned
  - 1,000 detached DOM nodes in memory

Heavy shoppers (50+ page views in 8 minutes):
  - 10,000+ detached DOM nodes
  - 20,000+ orphaned event listeners
  - 500+ MB memory
  - Browser crashes
```

### The Numbers

**Impact per user:**
- Light users (< 10 page views): No issue
- Medium users (10-30 views): Slowdown after 5 minutes
- Heavy users (30+ views): Crash after 7-8 minutes

**Black Friday traffic:**
- Normal day: 200K concurrent users
- Black Friday: 680K concurrent users
- Heavy shoppers: 35% (typical for sales events)
- Heavy shoppers affected: 680K × 35% = 238K users crashing

**Business Impact:**

```
Timeline of disaster:
11:47 PM - 12:15 AM: Intermittent crashes (28 min)
  Lost sales: $840K (est. $30K/min peak rate)

12:15 AM - 2:45 AM: Complete investigation + fix + deploy (2.5 hours)
  Lost sales: $2.25M ($15K/min after initial panic)

2:45 AM - 3:09 AM: Recovery period (24 min)
  Lost sales: $360K (reduced rate as customers left)

Total lost: $3.45M in direct sales
```

**Additional costs:**
- Customer support: 2,300 tickets filed
- Refunds/goodwill credits: $450K
- Engineering overtime (all hands): $120K
- Reputation damage: Trending on Twitter ("X site crashed on Black Friday")
- **Total: $4.2M**

### The Fix (Deployed at 2:45 AM)

**Emergency hotfix:**

```javascript
function ProductCard({ product }) {
  const [isHovered, setIsHovered] = useState(false);

  useEffect(() => {
    const element = document.getElementById(`product-${product.id}`);

    const handleMouseEnter = () => {
      setIsHovered(true);
      analytics.track('product_hover', { id: product.id });
    };

    const handleMouseLeave = () => {
      setIsHovered(false);
    };

    element?.addEventListener('mouseenter', handleMouseEnter);
    element?.addEventListener('mouseleave', handleMouseLeave);

    // ✅ THE FIX: Always cleanup event listeners
    return () => {
      element?.removeEventListener('mouseenter', handleMouseEnter);
      element?.removeEventListener('mouseleave', handleMouseLeave);
    };
  }, [product.id]);

  return (
    <div id={`product-${product.id}`} className={isHovered ? 'hovered' : ''}>
      {/* ... */}
    </div>
  );
}
```

**Result:**
- Memory growth: 52 MB → 62 MB (stable) ✅
- No detached DOM nodes ✅
- Site stable for hours ✅

### Why Code Review Missed This

**The PR (Nov 10th):**
```markdown
## Add hover analytics to product cards

**Changes:**
- Track when users hover over products
- Helps us understand engagement

**Files changed:** 1 file, +18 lines

**Reviewers:** @senior-dev-1 ✅, @senior-dev-2 ✅, @senior-dev-3 ✅
```

**Why 3 senior engineers approved broken code:**
1. **Small change bias** - "+18 lines" looked harmless
2. **No memory testing** - No one ran the app for 10+ minutes
3. **No cleanup checklist** - Code review template didn't ask "Did you cleanup all side effects?"
4. **Trust in `useEffect`** - Assumed junior dev knew to cleanup
5. **No automated detection** - ESLint rule `react-hooks/exhaustive-deps` doesn't catch missing cleanup

**The oversight:** Everyone focused on "does it work?" Nobody asked "does it clean up?"

### Long-Term Preventions

**1. ESLint rule to enforce cleanup (now mandatory):**

```javascript
// .eslintrc.js
rules: {
  'react-hooks/exhaustive-deps': 'error',
  'custom/require-cleanup-comment': 'error'  // Custom rule
}

// Custom rule: Require cleanup comment if useEffect has side effects
// If useEffect contains: addEventListener, setInterval, setTimeout, WebSocket
// Must have either:
//   - return () => { ... } cleanup, OR
//   - // @no-cleanup-needed: [justification]
```

**2. Memory leak testing in CI/CD:**

```javascript
// memory-leak-test.js (runs on every PR)
describe('Memory Leak Tests', () => {
  it('should not leak memory on repeated mounting', async () => {
    const { rerender, unmount } = render(<ProductList products={mockProducts} />);

    // Take initial heap snapshot
    const initialHeap = await getHeapSize();

    // Mount/unmount 100 times
    for (let i = 0; i < 100; i++) {
      unmount();
      rerender(<ProductList products={mockProducts} />);
    }

    // Check final heap size
    const finalHeap = await getHeapSize();
    const growth = finalHeap - initialHeap;

    // Should not grow more than 10% (allows for some variance)
    expect(growth / initialHeap).toBeLessThan(0.1);
  });
});
```

**3. Mandatory PR checklist:**

```markdown
## Memory Safety Checklist (required for all PRs)
- [ ] All `addEventListener` have matching `removeEventListener` in cleanup
- [ ] All `setInterval` have matching `clearInterval` in cleanup
- [ ] All `setTimeout` cleared or completed before unmount
- [ ] All WebSocket/SSE connections closed in cleanup
- [ ] No state updates after component unmount (use cleanup flag)
- [ ] Tested: Mount/unmount component 20+ times (no memory growth)
```

**4. Production memory monitoring:**

```javascript
// Added to all pages
function useMemoryMonitor() {
  useEffect(() => {
    const checkMemory = setInterval(() => {
      if (performance.memory) {
        const used = performance.memory.usedJSHeapSize / 1048576; // MB

        if (used > 300) {
          // Alert backend
          analytics.track('high_memory_usage', {
            heapSize: used,
            page: window.location.pathname,
            userId: user.id
          });
        }

        if (used > 450) {
          // Critical alert
          alert('App using high memory. Please refresh page.');
        }
      }
    }, 30000); // Check every 30s

    return () => clearInterval(checkMemory);
  }, []);
}
```

**5. Weekly "Cleanup Audit":**
- Every Monday, automated script scans codebase
- Finds all `useEffect` with side effects
- Checks if cleanup function exists
- Posts report to Slack with flagged components

### Lessons Learned

1. **Missing cleanup = memory leak**
   - `addEventListener` without `removeEventListener` = leak
   - `setInterval` without `clearInterval` = leak
   - **Every side effect needs cleanup**

2. **Small PRs can cause big disasters**
   - "+18 lines" crashed $4.2M in sales
   - Never assume small = safe
   - **Review for cleanup, not just correctness**

3. **Code reviews don't catch memory leaks**
   - 3 senior engineers missed it
   - Can't see memory leaks in code review
   - **Need automated testing**

4. **Black Friday traffic amplifies everything**
   - 1 leak × 680K users = disaster
   - Works fine in dev (low traffic)
   - **Always test at scale**

5. **useEffect is not fire-and-forget**
   - If you set up, you must clean up
   - Return function is not optional for side effects
   - **Cleanup is as important as setup**

### The Happy Ending

After this disaster:
- Zero memory leak incidents in next 12 months (automated testing caught 8 potential leaks before production)
- Memory monitoring caught 2 performance regressions early
- PR approval time increased (more thorough reviews)
- Engineers now paranoid about cleanup (good!)
- CTO wrote blog post "The $4.2M useEffect" (viral, 500K reads)

The disaster became a teaching moment for the entire React community. But it cost $4.2M to learn.

**Preventable?** 100% yes.
- ESLint rule: Free
- Memory leak test: 2 hours to write = $400
- Code review checklist: Free
- **Total prevention cost: $400**
- **ROI: 10,500× cheaper than fixing in production**

---

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

---

## 🚫 Common Mistakes Gallery

### Mistake 1: Forgetting Event Listener Cleanup
Adding `addEventListener` without `removeEventListener` in cleanup. **Every** listener needs cleanup or it leaks memory.

### Mistake 2: Interval Without Cleanup
`setInterval` in `useEffect` without `return () => clearInterval(id)`. App runs hundreds of intervals simultaneously, causing performance issues.

### Mistake 3: Async Operations After Unmount
Calling `setState` in a `.then()` after component unmounts causes "Can't perform state update on unmounted component" warnings. Use cancellation flags.

### Mistake 4: Mutable Dependencies in useEffect
Using objects/arrays in dependency array without memoization causes infinite re-renders. Dependencies must be stable or memoized.

### Mistake 5: No Error Boundaries
One component error crashes entire app. Always wrap route-level components in error boundaries to contain failures.

### Mistake 6: console.log Objects in Production
Logging large objects/arrays keeps them in memory (DevTools holds references). Remove or limit console logs in production.

---

## 📝 Cumulative Review

### Q1: Why did the ProductCard memory leak only crash heavy users?
<details><summary>Answer</summary>
Light users (< 10 page views) created few detached nodes (< 1,000) = under browser memory limit. Heavy users (50+ views) created 10,000+ detached nodes + 20,000 listeners = exceeded 512MB limit → crash. Memory leaks accumulate over time/usage.
</details>

### Q2: What's the difference between a detached DOM node and a regular DOM node?
<details><summary>Answer</summary>
Regular: Attached to document, displayed on page, can be garbage collected when removed. Detached: Removed from document but still in memory (event listeners/references hold it), never garbage collected → memory leak. Check with heap snapshot.
</details>

### Q3: How do you detect if your app has a memory leak?
<details><summary>Answer</summary>
1. Open Chrome DevTools → Memory tab
2. Take heap snapshot
3. Use app normally for 5-10 minutes (navigate pages, open/close modals)
4. Take another heap snapshot
5. Compare: Look for growing objects (detached DOM nodes, listeners)
6. If heap grows >20MB for same page → investigate
</details>

### Q4: What's the most common cause of memory leaks in React apps?
<details><summary>Answer</summary>
Missing cleanup in `useEffect`. Specifically: event listeners, intervals, timeouts, WebSocket connections, subscriptions not cleaned up when component unmounts. Rule: if `useEffect` has setup, it needs cleanup return function.
</details>

### Q5: Why don't code reviews catch memory leaks?
<details><summary>Answer</summary>
You can't see memory leaks in code - they only appear after prolonged use. A missing cleanup looks fine statically. Need: automated memory tests, heap snapshot comparisons, running app for 10+ minutes. Code review checks logic, not runtime memory behavior.
</details>

### Q6: You add `return () => clearInterval(id)` but still get warnings about state updates after unmount. What's wrong?
<details><summary>Answer</summary>
Cleanup stops new intervals, but ongoing async operations (fetch, setTimeout inside interval) still complete and try to update state. Solution: use cancellation flag:
```javascript
useEffect(() => {
  let cancelled = false;
  const id = setInterval(async () => {
    const data = await fetch('/api');
    if (!cancelled) setState(data);
  }, 1000);
  return () => {
    cancelled = true;
    clearInterval(id);
  };
}, []);
```
</details>

---

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
