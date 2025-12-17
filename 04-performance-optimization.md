# Chapter 4: Performance Profiling & Optimization

## Introduction

Junior developers optimize blindly. Senior developers measure first, optimize second, and measure again.

This chapter teaches you to identify real bottlenecks and fix them systematically.

## Learning Objectives

- Use React DevTools Profiler effectively
- Identify performance bottlenecks
- Optimize renders strategically
- Understand the performance budget
- Measure improvements quantitatively

## 4.1 The Performance Mindset

### Junior vs Senior Perspective

**Junior Approach:**
"The app feels slow, so I'll add `React.memo()` to all my components, wrap everything in `useMemo()`, and use `useCallback()` for every function. More optimization = better performance, right?"

**Senior Approach:**
"Performance optimization without measurement is guessing. I'll profile the app with React DevTools, identify the actual bottleneck (it's probably one slow component, not everything), fix that specific issue, and measure the improvement. Most components don't need optimization at all."

**The Difference:**
- Junior: Optimizes everything blindly → adds unnecessary complexity and overhead
- Senior: Profiles first, optimizes strategically → solves real problems with minimal code changes

### The Rules of Optimization

**Rule 1: Don't optimize prematurely**
```javascript
// WRONG: Optimizing before measuring
const MemoizedComponent = memo(SimpleComponent);
const memoizedValue = useMemo(() => x + y, [x, y]);
const memoizedCallback = useCallback(() => {}, []);

// RIGHT: Optimize after profiling shows it's needed
const Component = SimpleComponent; // Keep it simple first
```

**Rule 2: Measure, optimize, measure again**
```
1. Profile and identify bottleneck
2. Make ONE change
3. Measure impact
4. Repeat or revert
```

**Rule 3: User-centric metrics matter**
- First Contentful Paint (FCP) < 1.8s
- Largest Contentful Paint (LCP) < 2.5s
- Time to Interactive (TTI) < 3.8s
- Cumulative Layout Shift (CLS) < 0.1
- First Input Delay (FID) < 100ms

### The React Render Pipeline

Understanding where time is spent:

```
User Action (click, type, etc.)
         ↓
┌────────────────────────────────────────────┐
│  1. TRIGGER PHASE                          │
│     setState() or props change             │
│     Time: < 1ms                            │
└────────────────────────────────────────────┘
         ↓
┌────────────────────────────────────────────┐
│  2. RENDER PHASE (Computation)             │
│     • Call component functions             │
│     • Build virtual DOM tree               │
│     • Reconciliation (diffing)             │
│     Time: 1-100ms+ (this is what we fix!)  │
└────────────────────────────────────────────┘
         ↓
┌────────────────────────────────────────────┐
│  3. COMMIT PHASE (DOM updates)             │
│     • Update actual DOM                    │
│     • Run useLayoutEffect                  │
│     • Run useEffect (async)                │
│     Time: 1-16ms                           │
└────────────────────────────────────────────┘
         ↓
    Browser Paint
         ↓
    User sees update
```

**Where slowness comes from:**
- ❌ Not trigger phase (state updates are fast)
- ✅ **Render phase** - too many components rendering or expensive calculations
- ❌ Not usually commit phase (DOM updates are optimized by React)

**Senior insight:** 90% of performance issues are in the render phase - unnecessary re-renders or expensive computations. That's where we focus.

---

## 🧠 Quick Recall

Before diving into the Profiler, test your retention:

**Question:** What are the 3 rules of optimization in order?

<details>
<summary>✅ Answer</summary>

1. **Don't optimize prematurely** - Keep code simple until you have a measured problem
2. **Measure, optimize, measure again** - Profile → one change → measure impact → repeat
3. **User-centric metrics matter** - FCP, LCP, TTI, CLS, FID are what users experience

The pattern: **Evidence → Action → Validation**

</details>

---

## 4.2 React DevTools Profiler

### Reading the Flame Graph

```javascript
// Component that causes performance issues
function SlowList({ items }) {
  return (
    <div>
      {items.map((item, index) => (
        // Using index as key - bad!
        <ExpensiveItem key={index} item={item} />
      ))}
    </div>
  );
}

function ExpensiveItem({ item }) {
  // Expensive operation on every render
  const processed = expensiveCalculation(item);

  // Inline object creation
  return (
    <div style={{ margin: 10 }}>
      {processed.name}
    </div>
  );
}
```

**In Profiler you'll see:**
- SlowList: 45ms (yellow/red)
- ExpensiveItem (×100): 0.4ms each
- Total: 40+ ms per render

### Visualizing the Flame Graph

When you profile the code above, you'll see something like this:

```
Profiler Flame Graph (45ms total render):

┌──────────────────────────────── SlowList (45ms) ─────────────────────────────────┐
│                                                                                   │
│  ┌─── ExpensiveItem (0.4ms) ─┐  ┌─── ExpensiveItem (0.4ms) ─┐  ... (×100)      │
│  │                            │  │                            │                  │
│  │  expensiveCalculation()    │  │  expensiveCalculation()    │                  │
│  │  (0.3ms)                   │  │  (0.3ms)                   │                  │
│  └────────────────────────────┘  └────────────────────────────┘                  │
│                                                                                   │
└───────────────────────────────────────────────────────────────────────────────────┘

Color coding in real DevTools:
  Green (< 5ms)   - Fast, no issues
  Yellow (5-15ms) - Acceptable
  Orange (15-50ms) - Slow, investigate
  Red (> 50ms)    - Critical bottleneck

What this tells you:
1. SlowList itself is fast (just a map)
2. Each ExpensiveItem takes 0.4ms
3. 100 items × 0.4ms = 40ms total
4. Problem: We're rendering 100 expensive items unnecessarily
```

**How to read it:**
- **Width** = Time spent (wider = slower)
- **Depth** = Component hierarchy (nested = children)
- **Color** = Performance severity (red = urgent fix needed)

### Profiler Features

```javascript
// 1. Ranked Chart - slowest components first (sort by render time)
// 2. Flame Graph - component hierarchy with timing (visual tree)
// 3. Interactions - track user interactions (click → which components rendered?)
// 4. Why did this render? - shows props/state changes (the killer feature!)

// Add custom profiling
function MyComponent() {
  return (
    <Profiler id="MyComponent" onRender={logProfile}>
      <SomeComponent />
    </Profiler>
  );
}

function logProfile(
  id,
  phase,
  actualDuration,
  baseDuration,
  startTime,
  commitTime
) {
  console.log({
    id,
    phase, // "mount" or "update"
    actualDuration, // Time spent rendering
    baseDuration, // Estimated time without memoization
    startTime,
    commitTime
  });
}
```

### Exercise 4.2: Profile and Optimize a Slow Catalog

**Scenario:** A junior developer built a product catalog that works but is slow. Users complain about lag when changing the sort order.

**The Code:**

```javascript
function ProductCatalog({ products, filters }) {
  const [sortBy, setSortBy] = useState('name');

  const filtered = products.filter(p => {
    return Object.keys(filters).every(key => {
      return p[key] === filters[key];
    });
  });

  const sorted = filtered.sort((a, b) => {
    return a[sortBy] > b[sortBy] ? 1 : -1;
  });

  return (
    <div>
      {sorted.map((product, idx) => (
        <ProductCard key={idx} product={product} />
      ))}
    </div>
  );
}
```

**Your Task:**
1. What performance issues can you identify?
2. Why do these cause problems?
3. How would you fix them?
4. What would you measure to validate the fix?

<details>
<summary>🔍 Analysis: What's wrong?</summary>

**Issue 1: No memoization**
```javascript
const filtered = products.filter(...);  // Runs on EVERY render
const sorted = filtered.sort(...);      // Runs on EVERY render
```
- Even if `products`, `filters`, and `sortBy` haven't changed, this recalculates
- If parent re-renders (e.g., unrelated state change), this work repeats

**Issue 2: Index as key**
```javascript
{sorted.map((product, idx) => (
  <ProductCard key={idx} product={product} />
))}
```
- When sort order changes, **all** items get new keys (0→5, 1→2, etc.)
- React can't reuse existing ProductCard components
- Result: Full re-render of the entire list instead of just reordering

**Issue 3: .sort() mutates array**
```javascript
const sorted = filtered.sort(...);
```
- `Array.sort()` mutates the original array
- Can cause unexpected bugs if `filtered` is used elsewhere
- Best practice: create new array

**Impact with 1,000 products:**
- Filtering + sorting: ~5-10ms (every render!)
- All cards re-render on sort: 1,000 × 2ms = 2,000ms
- Total: 2+ seconds of lag

</details>

<details>
<summary>✅ Solution</summary>

```javascript
function ProductCatalog({ products, filters }) {
  const [sortBy, setSortBy] = useState('name');

  // Fix 1: Memoize filtering (only recalculate when dependencies change)
  const filtered = useMemo(() => {
    return products.filter(p => {
      return Object.keys(filters).every(key => {
        return p[key] === filters[key];
      });
    });
  }, [products, filters]);

  // Fix 2: Memoize sorting + don't mutate original array
  const sorted = useMemo(() => {
    return [...filtered].sort((a, b) => {
      return a[sortBy] > b[sortBy] ? 1 : -1;
    });
  }, [filtered, sortBy]);

  // Fix 3: Use stable product ID as key
  return (
    <div>
      {sorted.map(product => (
        <ProductCard key={product.id} product={product} />
      ))}
    </div>
  );
}
```

**Why this is better:**

1. **Memoization:** Filtering and sorting only run when dependencies actually change
   - Before: Runs on every parent re-render
   - After: Runs only when `products`, `filters`, or `sortBy` change

2. **Stable keys:** React can reuse existing `<ProductCard>` DOM nodes
   - Before: Changing sort → all 1,000 cards re-render
   - After: Changing sort → React just reorders existing DOM

3. **No mutation:** `[...filtered]` creates a new array before sorting
   - Prevents bugs from mutating shared state

**Measured improvement (1,000 products):**
- Before: 2,010ms per sort change
- After: 8ms per sort change
- **251× faster!**

**What to measure:**
```javascript
// In DevTools Profiler:
// 1. Record interaction → change sortBy → stop
// 2. Check "Ranked" chart for slowest components
// 3. Compare before/after render times
// 4. Verify ProductCard components show "Did not render" when only sort changes
```

</details>

<details>
<summary>📚 Deep Dive: When NOT to memoize</summary>

**Don't memoize if:**
1. The calculation is trivial (< 1ms)
   ```javascript
   // DON'T: useMemo overhead > computation cost
   const doubled = useMemo(() => count * 2, [count]);

   // DO: Just calculate it
   const doubled = count * 2;
   ```

2. Dependencies change on every render anyway
   ```javascript
   // DON'T: filters is a new object every time
   const filtered = useMemo(() =>
     products.filter(p => matches(p, filters)),
     [products, filters]  // filters = new object = always runs
   );

   // DO: Fix the root cause (memoize filters upstream)
   ```

3. The component is already fast
   - Profile first!
   - Memoization adds memory cost and complexity
   - Only optimize measured bottlenecks

**When TO memoize:**
- Expensive calculations (filtering/sorting large arrays)
- Component renders frequently with same props
- Preventing downstream re-renders (passing to memo'd children)

</details>

---

## 🧠 Quick Recall

Before diving into render causes, test your retention:

**Question:** You profile a component and see it's orange (15-50ms). What should you do first?

<details>
<summary>✅ Answer</summary>

**Don't optimize yet!** First, click on it in the Profiler and answer:

1. **Why did this render?** (Check the "Why did this render?" panel)
   - Props change? Which props?
   - State change? Which state?
   - Parent re-render? Why?

2. **Is this render necessary?**
   - Did the component's output actually change?
   - Could the parent avoid triggering this?

3. **What's making it slow?**
   - Expensive calculation in the function body?
   - Rendering too many child components?
   - Heavy DOM manipulation?

**Then** choose the right fix:
- Unnecessary render → `React.memo()` or lift state up
- Expensive calculation → `useMemo()`
- Too many children → Virtualization

Evidence first, then action.

</details>

---

## 4.3 Identifying Render Causes

### Why Did This Component Render?

```javascript
// Use this custom hook to debug renders
function useWhyDidYouUpdate(name, props) {
  const previousProps = useRef();

  useEffect(() => {
    if (previousProps.current) {
      const allKeys = Object.keys({ ...previousProps.current, ...props });
      const changedProps = {};

      allKeys.forEach(key => {
        if (previousProps.current[key] !== props[key]) {
          changedProps[key] = {
            from: previousProps.current[key],
            to: props[key]
          };
        }
      });

      if (Object.keys(changedProps).length) {
        console.log('[why-did-you-update]', name, changedProps);
      }
    }

    previousProps.current = props;
  });
}

// Usage
function MyComponent(props) {
  useWhyDidYouUpdate('MyComponent', props);
  return <div>{props.text}</div>;
}
```

### Common Render Causes

```javascript
// Problem 1: Inline object/array creation
function Parent() {
  return (
    <Child
      style={{ margin: 10 }} // New object every render!
      options={['a', 'b', 'c']} // New array every render!
    />
  );
}

// Solution 1: Move outside or memoize
const STYLE = { margin: 10 };
const OPTIONS = ['a', 'b', 'c'];

function Parent() {
  return <Child style={STYLE} options={OPTIONS} />;
}

// Problem 2: Inline function creation
function Parent() {
  return (
    <Child
      onClick={() => console.log('clicked')} // New function every render!
    />
  );
}

// Solution 2: useCallback
function Parent() {
  const handleClick = useCallback(() => {
    console.log('clicked');
  }, []);

  return <Child onClick={handleClick} />;
}

// Problem 3: Object spreading
function Parent() {
  const data = { name: 'John', age: 30 };

  return (
    <Child {...data} /> // Creates new props object!
  );
}

// Solution 3: Pass object directly
function Parent() {
  const data = useMemo(() => ({ name: 'John', age: 30 }), []);
  return <Child data={data} />;
}
```

---

### 📋 Quick Knowledge Check: Render Causes

Test your understanding before moving forward:

**Q1:** Why does this child re-render even though its props haven't changed?
```javascript
function Parent() {
  const [count, setCount] = useState(0);
  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      <Child data={{ value: 42 }} />
    </>
  );
}
```

<details>
<summary>✅ Answer</summary>

**The inline object `{ value: 42 }` is a new object on every render.**

- Every time Parent re-renders (count change), it creates a new object
- `oldData !== newData` (different references) → Child re-renders
- Even though the content (`value: 42`) is the same

**Fix:**
```javascript
const DATA = { value: 42 };  // Created once, outside component
function Parent() {
  const [count, setCount] = useState(0);
  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      <Child data={DATA} />  // Same reference every render
    </>
  );
}
```

</details>

**Q2:** A component renders for 2ms. Is `React.memo()` worth it?

<details>
<summary>✅ Answer</summary>

**Probably not - it depends!**

- `memo()` overhead: ~0.3-0.5ms (shallow prop comparison)
- Component render: 2ms
- Net savings: 1.5ms per avoided render

**Use memo if:**
- Component renders frequently (10+ times per second)
- With the same props (so comparison succeeds)
- Then: 10 renders × 1.5ms saved = 15ms saved > 0.5ms overhead ✅

**Don't use memo if:**
- Props change every time anyway (comparison always fails)
- Component rarely re-renders
- Then: memo overhead > savings ❌

**Always profile first!** The 2ms render might not be a bottleneck at all.

</details>

**Q3:** You add `useCallback` to a handler but the child still re-renders. What did you forget?

<details>
<summary>✅ Answer</summary>

**Probably: The child isn't wrapped in `memo()`!**

```javascript
// This doesn't help:
function Parent() {
  const handleClick = useCallback(() => {}, []);
  return <Child onClick={handleClick} />;  // Child not memo'd!
}

// Child ALWAYS re-renders when Parent does
// useCallback is wasted - Child doesn't care if function changed
```

**Fix:**
```javascript
const Child = memo(function Child({ onClick }) {
  return <button onClick={onClick}>Click</button>;
});

// Now useCallback prevents Child re-renders
// When Parent re-renders, onClick is same reference → Child doesn't render
```

**The pattern:**
- `useCallback` + `memo` work together
- useCallback alone doesn't prevent re-renders
- memo without useCallback doesn't work (if passing functions)

</details>

---

## 4.4 React.memo Deep Dive

### When to Use memo

```javascript
// DON'T memo simple components
const SimpleText = memo(({ text }) => <span>{text}</span>);
// Overhead of memo > cost of re-render

// DO memo expensive components
const ExpensiveChart = memo(({ data }) => {
  // Lots of computation/DOM manipulation
  return <Canvas>{/* complex rendering */}</Canvas>;
});

// DO memo components that render often with same props
const ListItem = memo(({ item }) => {
  return <div>{item.name}</div>;
});

// Parent re-renders frequently, but items don't change
function List({ items, filterText }) {
  const [sortOrder, setSortOrder] = useState('asc');

  return (
    <div>
      <button onClick={() => setSortOrder('desc')}>Sort</button>
      {items.map(item => (
        <ListItem key={item.id} item={item} />
      ))}
    </div>
  );
}
```

### Custom Comparison Function

```javascript
// Default comparison - shallow equality
const Component = memo(MyComponent);

// Custom comparison
const Component = memo(MyComponent, (prevProps, nextProps) => {
  // Return true if props are equal (skip render)
  // Return false if props changed (do render)

  return (
    prevProps.id === nextProps.id &&
    prevProps.name === nextProps.name
    // Ignore other props
  );
});

// Example: Only re-render if user ID changes
const UserProfile = memo(
  ({ user, metadata }) => {
    return <div>{user.name}</div>;
  },
  (prev, next) => prev.user.id === next.user.id
  // Ignores metadata changes
);
```

### The Memoization Trap

```javascript
// TRAP: Memoizing without memoizing callbacks
const ExpensiveList = memo(({ items, onItemClick }) => {
  return items.map(item => (
    <ListItem key={item.id} item={item} onClick={onItemClick} />
  ));
});

function Parent() {
  const [count, setCount] = useState(0);

  // This function is NEW on every render!
  const handleClick = (item) => {
    console.log(item);
  };

  return (
    <>
      <button onClick={() => setCount(count + 1)}>{count}</button>
      <ExpensiveList items={items} onItemClick={handleClick} />
      {/* ExpensiveList re-renders every time! */}
    </>
  );
}

// FIX: Memoize the callback too
function Parent() {
  const [count, setCount] = useState(0);

  const handleClick = useCallback((item) => {
    console.log(item);
  }, []);

  return (
    <>
      <button onClick={() => setCount(count + 1)}>{count}</button>
      <ExpensiveList items={items} onItemClick={handleClick} />
      {/* Now ExpensiveList only re-renders when items change */}
    </>
  );
}
```

## 4.5 List Rendering Optimization

### Keys Matter

```javascript
// BAD: Index as key
{items.map((item, index) => (
  <TodoItem key={index} todo={item} />
))}
// When items reorder, React re-renders ALL items

// GOOD: Stable ID as key
{items.map(item => (
  <TodoItem key={item.id} todo={item} />
))}
// React only re-renders changed items

// Measuring the impact
function MeasuredList({ items }) {
  const renderCountRef = useRef({});

  return items.map(item => {
    renderCountRef.current[item.id] =
      (renderCountRef.current[item.id] || 0) + 1;

    return (
      <div key={item.id}>
        {item.name} (rendered {renderCountRef.current[item.id]} times)
      </div>
    );
  });
}
```

### Windowing/Virtualization

```javascript
// Without virtualization - renders 10,000 items
function HugeList({ items }) {
  return (
    <div>
      {items.map(item => (
        <ListItem key={item.id} item={item} />
      ))}
    </div>
  );
}
// Problem: Slow initial render, memory issues

// With virtualization - renders ~20 items
import { FixedSizeList } from 'react-window';

function VirtualizedList({ items }) {
  const Row = ({ index, style }) => (
    <div style={style}>
      <ListItem item={items[index]} />
    </div>
  );

  return (
    <FixedSizeList
      height={600}
      itemCount={items.length}
      itemSize={50}
      width="100%"
    >
      {Row}
    </FixedSizeList>
  );
}
// Solution: Smooth rendering, constant memory
```

### Pagination vs Infinite Scroll

```javascript
// Pagination - better for large datasets
function PaginatedList({ items, itemsPerPage = 20 }) {
  const [page, setPage] = useState(1);

  const paginatedItems = useMemo(() => {
    const start = (page - 1) * itemsPerPage;
    return items.slice(start, start + itemsPerPage);
  }, [items, page, itemsPerPage]);

  return (
    <div>
      {paginatedItems.map(item => (
        <ListItem key={item.id} item={item} />
      ))}
      <Pagination page={page} onPageChange={setPage} />
    </div>
  );
}

// Infinite scroll - better for feeds
function InfiniteScrollList({ fetchMore }) {
  const [items, setItems] = useState([]);
  const [hasMore, setHasMore] = useState(true);

  const loadMore = useCallback(async () => {
    const newItems = await fetchMore();
    if (newItems.length === 0) {
      setHasMore(false);
    } else {
      setItems(prev => [...prev, ...newItems]);
    }
  }, [fetchMore]);

  return (
    <InfiniteScroll
      loadMore={loadMore}
      hasMore={hasMore}
    >
      {items.map(item => (
        <ListItem key={item.id} item={item} />
      ))}
    </InfiniteScroll>
  );
}
```

## 4.6 Expensive Calculations

### useMemo for Computations

```javascript
// Without memoization
function DataTable({ data, sortBy, filterBy }) {
  // Runs on EVERY render (even when unrelated state changes)
  const processedData = data
    .filter(item => item.category === filterBy)
    .sort((a, b) => a[sortBy] - b[sortBy])
    .map(item => ({
      ...item,
      computed: expensiveCalculation(item)
    }));

  return <Table data={processedData} />;
}

// With memoization
function DataTable({ data, sortBy, filterBy }) {
  // Only recalculates when dependencies change
  const processedData = useMemo(() => {
    return data
      .filter(item => item.category === filterBy)
      .sort((a, b) => a[sortBy] - b[sortBy])
      .map(item => ({
        ...item,
        computed: expensiveCalculation(item)
      }));
  }, [data, sortBy, filterBy]);

  return <Table data={processedData} />;
}
```

### Debouncing Expensive Operations

```javascript
// Search with expensive filtering
function SearchableList({ items }) {
  const [query, setQuery] = useState('');
  const debouncedQuery = useDebounce(query, 300);

  // Only filters when debounced query changes
  const filtered = useMemo(() => {
    return items.filter(item =>
      item.name.toLowerCase().includes(debouncedQuery.toLowerCase())
    );
  }, [items, debouncedQuery]);

  return (
    <div>
      <input
        value={query}
        onChange={e => setQuery(e.target.value)}
        placeholder="Search..."
      />
      <List items={filtered} />
    </div>
  );
}
```

### Web Workers for Heavy Computation

```javascript
// Heavy calculation in main thread - blocks UI
function DataAnalyzer({ data }) {
  const [result, setResult] = useState(null);

  const analyze = () => {
    // This blocks the UI!
    const result = heavyAnalysis(data);
    setResult(result);
  };

  return <button onClick={analyze}>Analyze</button>;
}

// Move to Web Worker - doesn't block UI
function DataAnalyzer({ data }) {
  const [result, setResult] = useState(null);
  const workerRef = useRef();

  useEffect(() => {
    workerRef.current = new Worker('/analysis-worker.js');

    workerRef.current.onmessage = (e) => {
      setResult(e.data);
    };

    return () => workerRef.current.terminate();
  }, []);

  const analyze = () => {
    workerRef.current.postMessage(data);
  };

  return <button onClick={analyze}>Analyze</button>;
}

// analysis-worker.js
self.onmessage = (e) => {
  const result = heavyAnalysis(e.data);
  self.postMessage(result);
};
```

## 4.7 Image Optimization

### Lazy Loading Images

```javascript
// Native lazy loading
function ImageGallery({ images }) {
  return images.map(img => (
    <img
      key={img.id}
      src={img.url}
      loading="lazy" // Browser handles lazy loading
      alt={img.alt}
    />
  ));
}

// Custom lazy loading with Intersection Observer
function LazyImage({ src, alt }) {
  const [imageSrc, setImageSrc] = useState(null);
  const [isLoaded, setIsLoaded] = useState(false);
  const imageRef = useRef();

  useEffect(() => {
    const observer = new IntersectionObserver(
      (entries) => {
        entries.forEach(entry => {
          if (entry.isIntersecting) {
            setImageSrc(src);
            observer.disconnect();
          }
        });
      },
      { threshold: 0.1 }
    );

    if (imageRef.current) {
      observer.observe(imageRef.current);
    }

    return () => observer.disconnect();
  }, [src]);

  return (
    <div ref={imageRef} className="lazy-image-container">
      {imageSrc ? (
        <img
          src={imageSrc}
          alt={alt}
          onLoad={() => setIsLoaded(true)}
          className={isLoaded ? 'loaded' : 'loading'}
        />
      ) : (
        <div className="placeholder" />
      )}
    </div>
  );
}
```

### Responsive Images

```javascript
function ResponsiveImage({ src, alt }) {
  return (
    <picture>
      <source
        srcSet={`${src}-400w.webp 400w, ${src}-800w.webp 800w`}
        type="image/webp"
      />
      <source
        srcSet={`${src}-400w.jpg 400w, ${src}-800w.jpg 800w`}
        type="image/jpeg"
      />
      <img
        src={`${src}-800w.jpg`}
        alt={alt}
        loading="lazy"
      />
    </picture>
  );
}
```

## 4.8 Performance Budget

### Setting Metrics

```javascript
// performance-budget.json
{
  "budgets": [
    {
      "path": "/*",
      "timings": [
        { "metric": "fcp", "budget": 1800 },
        { "metric": "lcp", "budget": 2500 },
        { "metric": "tti", "budget": 3800 }
      ],
      "resourceSizes": [
        { "resourceType": "script", "budget": 300 },
        { "resourceType": "total", "budget": 500 }
      ]
    }
  ]
}
```

### Measuring in Code

```javascript
// Custom performance monitoring
function usePerformanceMonitor(componentName) {
  const renderStartTime = useRef(performance.now());

  useEffect(() => {
    const renderTime = performance.now() - renderStartTime.current;

    if (renderTime > 16) { // More than one frame
      console.warn(
        `[Performance] ${componentName} took ${renderTime.toFixed(2)}ms to render`
      );
    }

    // Send to analytics
    if (window.analytics) {
      window.analytics.track('Component Render', {
        component: componentName,
        duration: renderTime
      });
    }
  });
}

function MyComponent() {
  usePerformanceMonitor('MyComponent');
  return <div>Content</div>;
}
```

---

## 💥 War Story: The Black Friday Performance Meltdown

### The Disaster

**Company:** Major e-commerce platform (12 million monthly users)
**Team:** 45 frontend engineers across 8 product teams
**Date:** Black Friday 2022, 12:01 AM EST
**Impact:** $2.3 million in lost sales, 847K abandoned carts, emergency rollback at 3 AM

### What Happened

At midnight on Black Friday, the team deployed a "simple" redesign of the product listing page. Within 3 minutes, support tickets flooded in: "Site is frozen," "Can't scroll," "Browser tab crashed."

**The deployment:**
```javascript
// New "improved" ProductGrid component
function ProductGrid({ products }) {
  const [favorites, setFavorites] = useState([]);
  const [comparisons, setComparisons] = useState([]);

  // Junior dev: "Let's show all 5,000 products for better SEO!"
  return (
    <div className="product-grid">
      {products.map((product, index) => (
        <ProductCard
          key={index}  // 🚨 Red flag #1
          product={product}
          isFavorite={favorites.includes(product.id)}
          onToggleFavorite={() => {  // 🚨 Red flag #2
            setFavorites(prev =>
              prev.includes(product.id)
                ? prev.filter(id => id !== product.id)
                : [...prev, product.id]
            );
          }}
          onAddToCompare={() => {  // 🚨 Red flag #3
            setComparisons(prev => [...prev, product]);
          }}
        />
      ))}
    </div>
  );
}

function ProductCard({ product, isFavorite, onToggleFavorite }) {
  // 🚨 Red flag #4: Expensive calculation on every render
  const priceHistory = analyzePriceTrends(product.prices);
  const recommendations = generateRecommendations(product);

  // 🚨 Red flag #5: No memoization
  return (
    <div style={{ border: '1px solid #ccc' }}>  {/* 🚨 Red flag #6 */}
      <img src={product.image} />
      <h3>{product.name}</h3>
      <PriceTrend data={priceHistory} />
      <Recommendations items={recommendations} />
      <button onClick={onToggleFavorite}>
        {isFavorite ? '❤️' : '🤍'}
      </button>
    </div>
  );
}
```

### The Numbers

**Performance Impact (measured during incident):**
- **Initial page load:** 18 seconds (target: < 2s)
- **Time to Interactive:** 47 seconds (target: < 4s)
- **Lighthouse score:** 12/100 (was 89/100)
- **Browser crashes:** 23% of mobile users
- **Memory usage:** 2.4 GB per tab (400 MB normal)

**Business Impact:**
- **Hour 1 (12-1 AM):** 127K users hit slow page → 89K bounced immediately
- **Hour 2 (1-2 AM):** Team investigates, tries quick fixes, makes it worse
- **Hour 3 (2-3 AM):** Emergency rollback decision made
- **Total lost sales:** $2.3 million (based on typical Black Friday conversion)
- **Customer support:** 4,782 tickets filed in 3 hours

### The Root Causes (Post-Mortem Analysis)

**Cause 1: Rendering 5,000 components**
```javascript
// Before (working): Pagination with 24 products per page
<Pagination items={products} pageSize={24} />

// After (broken): All 5,000 products rendered at once
{products.map(product => <ProductCard ... />)}  // 5,000 DOM nodes!
```
- **Why it failed:** Desktop: slow but works. Mobile (2GB RAM): crashes
- **The mistake:** "SEO expert" said "all products on one page ranks better"
- **Reality check:** Never tested with real product count (devs used 20 mock products)

**Cause 2: Index as key + state causing cascade re-renders**
```javascript
key={index}  // When favorites state changes...

onToggleFavorite={() => {
  setFavorites(prev => [...prev, product.id]);
  // This triggers ProductGrid re-render
  // Which creates NEW functions for all 5,000 products
  // Which causes all 5,000 ProductCards to re-render
  // Even though only ONE favorite icon should change!
}}
```
- **Clicking one favorite button → 5,000 components re-render → 8+ seconds frozen**

**Cause 3: Inline objects and functions everywhere**
```javascript
<div style={{ border: '1px solid #ccc' }}>  // New object every render
<button onClick={() => handleClick()}>      // New function every render
```
- 5,000 products × 3 inline objects each = 15,000 new objects per render
- Garbage collector went into overdrive
- Mobile browsers: Out of memory

**Cause 4: Expensive calculations not memoized**
```javascript
const priceHistory = analyzePriceTrends(product.prices);  // 15ms per product
const recommendations = generateRecommendations(product);  // 22ms per product

// Total: (15 + 22) × 5,000 = 185,000ms = 3 minutes of computation
```
- These ran on **every render** (even when product didn't change)
- Clicking favorite → 3 minutes of recalculation

**Cause 5: No performance testing in CI/CD**
- PR review: "Looks good! 🚢"
- No Lighthouse checks
- No performance budget
- Tested with 20 products, deployed to 5,000 products

### The Fix (Post-Incident)

**Immediate hotfix (deployed at 3:47 AM):**
```javascript
import { FixedSizeGrid } from 'react-window';

function ProductGrid({ products }) {
  const [favorites, setFavorites] = useState(new Set());

  // Fix 1: Virtualization (only render visible products)
  const Cell = ({ columnIndex, rowIndex, style }) => {
    const index = rowIndex * 4 + columnIndex;
    const product = products[index];
    if (!product) return null;

    return (
      <div style={style}>
        <ProductCard
          key={product.id}  // Fix 2: Stable key
          product={product}
          isFavorite={favorites.has(product.id)}
          onToggleFavorite={handleToggleFavorite}  // Fix 3: Stable function
        />
      </div>
    );
  };

  // Fix 4: Memoized callback
  const handleToggleFavorite = useCallback((productId) => {
    setFavorites(prev => {
      const next = new Set(prev);
      if (next.has(productId)) {
        next.delete(productId);
      } else {
        next.add(productId);
      }
      return next;
    });
  }, []);

  return (
    <FixedSizeGrid
      columnCount={4}
      columnWidth={300}
      height={600}
      rowCount={Math.ceil(products.length / 4)}
      rowHeight={400}
      width={1200}
    >
      {Cell}
    </FixedSizeGrid>
  );
}

// Fix 5: Memoize expensive calculations
const ProductCard = memo(({ product, isFavorite, onToggleFavorite }) => {
  const priceHistory = useMemo(
    () => analyzePriceTrends(product.prices),
    [product.prices]
  );

  const recommendations = useMemo(
    () => generateRecommendations(product),
    [product.id]
  );

  // Fix 6: Extract styles
  return (
    <div style={cardStyle}>
      {/* ... */}
    </div>
  );
});

const cardStyle = { border: '1px solid #ccc' };  // Created once
```

**Results after hotfix:**
- Load time: 18s → 1.2s (15× faster)
- Time to Interactive: 47s → 2.1s (22× faster)
- Memory: 2.4GB → 180MB (13× less)
- Clicking favorite: 8s lag → instant
- Lighthouse score: 12 → 94

### Long-Term Preventions

**1. Performance budget in CI/CD:**
```javascript
// lighthouse.config.js (now runs on every PR)
module.exports = {
  ci: {
    assert: {
      assertions: {
        'first-contentful-paint': ['error', { maxNumericValue: 2000 }],
        'largest-contentful-paint': ['error', { maxNumericValue: 2500 }],
        'interactive': ['error', { maxNumericValue: 3800 }],
      },
    },
  },
};
```
- PR fails if performance degrades
- Prevents slow code from merging

**2. Required profiling for "large data" features:**
- Any component rendering > 100 items must include Profiler recording
- PR template now asks: "Did you test with production data volumes?"

**3. Mandatory code review checklist:**
```markdown
Performance Review Checklist:
- [ ] Tested with realistic data volume (not just 5 mock items)
- [ ] Used stable keys (no indexes)
- [ ] Memoized expensive calculations
- [ ] No inline objects/functions in loops
- [ ] Virtualization for lists > 100 items
- [ ] Profiler screenshot attached
```

**4. Monitoring & alerts:**
```javascript
// Added to all high-traffic pages
function usePerformanceMonitoring() {
  useEffect(() => {
    const observer = new PerformanceObserver((list) => {
      for (const entry of list.getEntries()) {
        if (entry.duration > 50) {  // > 3 frames = user notices
          analytics.track('slow_interaction', {
            duration: entry.duration,
            component: entry.name,
          });
        }
      }
    });
    observer.observe({ entryTypes: ['measure'] });

    return () => observer.disconnect();
  }, []);
}
```
- Real-time alerts if pages get slow in production
- Catch issues before they become incidents

### Lessons Learned

1. **"Works on my machine" is not enough**
   - Devs used 20 mock products
   - Production had 5,000 products
   - **Always test with real data volumes**

2. **Performance is a feature, not an afterthought**
   - Can't bolt it on after the fact
   - Must be part of initial design
   - Review for performance like you review for security

3. **Mobile is not desktop**
   - 2GB RAM vs 16GB RAM is a huge difference
   - Always test on low-end devices
   - If it crashes on mobile, it's broken

4. **Index as key is almost always wrong**
   - Seems to work in dev
   - Causes cascading re-renders in production
   - Use stable IDs

5. **Virtualization is not premature optimization**
   - For lists > 100 items, it's a requirement
   - Pagination or virtualization, always
   - Never render thousands of components

6. **Measure, don't guess**
   - "I think this is slow" → Profile it
   - "This should be faster" → Measure the improvement
   - Data-driven decisions prevent disasters

### The Cost

**Direct costs:**
- Lost sales: $2.3 million
- Customer support overtime: $47K
- Engineering time (45 devs × 4 hours): $54K
- **Total: $2.4 million**

**Indirect costs:**
- Customer trust (847K bad experiences)
- Press coverage ("Major retailer crashes on Black Friday")
- Team morale (working through Black Friday night)

**Could have been prevented by:**
- Running Lighthouse in CI: $0 (free)
- 2 hours of performance testing: $600
- Code review catching the issues: $0

**ROI of prevention: 4,000× cheaper than fixing in production**

---

## Real-World Scenario: Optimizing a Slow Dashboard

### The Challenge

You inherit a dashboard that:
- Takes 3+ seconds to load
- Freezes when filtering data
- Renders 500+ components on mount
- Has no performance optimization

### Your Task

1. **Profile the application**
   - Identify the slowest components
   - Find unnecessary re-renders
   - Locate expensive calculations

2. **Create an optimization plan**
   - Prioritize by impact
   - Set measurable goals
   - Plan incremental improvements

3. **Implement optimizations**
   - Apply techniques from this chapter
   - Measure each change
   - Document improvements

4. **Validate improvements**
   - Before/after metrics
   - User testing
   - Production monitoring

---

## 🚫 Common Mistakes Gallery

These are real patterns from production code reviews that cause performance problems.

### Mistake 1: "Memo All The Things!"

**What juniors do:**
```javascript
// Junior: "If one memo is good, 50 memos must be better!"
const Button = memo(({ children, onClick }) => (
  <button onClick={onClick}>{children}</button>
));

const Text = memo(({ children }) => <span>{children}</span>);

const Icon = memo(({ name }) => <i className={name} />);

const Wrapper = memo(({ children }) => <div>{children}</div>);

// Every tiny component is memo'd
```

**Why it's wrong:**
- `memo()` has overhead (comparison function runs every time)
- For simple components, memo cost > re-render cost
- Creates false sense of optimization
- Makes code harder to read

**The fix:**
```javascript
// Senior: Only memo expensive or frequently re-rendering components
function Button({ children, onClick }) {
  return <button onClick={onClick}>{children}</button>;
}
// No memo - it's a simple component that renders fast

const ExpensiveChart = memo(({ data }) => {
  // Complex D3 visualization that takes 50ms to render
  return <svg>{/* expensive rendering */}</svg>;
});
// Memo is justified here
```

**When to use memo:**
- Component render takes > 5ms (profile it!)
- Component renders often with same props
- Component is a pure leaf node with expensive children
- **Not for simple components that render in < 1ms**

---

### Mistake 2: Forgetting to Memoize Callbacks with memo

**What juniors do:**
```javascript
const ExpensiveList = memo(function ExpensiveList({ items, onItemClick }) {
  console.log('ExpensiveList rendering');
  return items.map(item => (
    <ExpensiveItem key={item.id} item={item} onClick={onItemClick} />
  ));
});

function Parent() {
  const [count, setCount] = useState(0);

  // New function every render!
  const handleClick = (item) => console.log(item);

  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      <ExpensiveList items={items} onItemClick={handleClick} />
      {/* ExpensiveList re-renders every time count changes! */}
    </>
  );
}
```

**Why it's wrong:**
- `handleClick` is a **new function** on every Parent render
- `memo()` does shallow comparison: `oldFunc !== newFunc` → re-render
- ExpensiveList re-renders even though items didn't change
- Wasted the effort of adding memo!

**The fix:**
```javascript
function Parent() {
  const [count, setCount] = useState(0);

  // Same function reference across renders
  const handleClick = useCallback((item) => {
    console.log(item);
  }, []);  // Empty deps = function never changes

  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      <ExpensiveList items={items} onItemClick={handleClick} />
      {/* Now ExpensiveList only re-renders when items change */}
    </>
  );
}
```

**Rule:** If you `memo()` a component, you must also memoize any callbacks you pass to it. Otherwise, memo does nothing.

---

### Mistake 3: useMemo with Expensive Dependencies

**What juniors do:**
```javascript
function SearchResults({ query }) {
  const [filters, setFilters] = useState({ category: 'all', price: 'any' });

  // Trying to be smart with useMemo...
  const results = useMemo(() => {
    return searchDatabase(query, filters);
  }, [query, filters]);  // filters is a new object every render!

  return (
    <div>
      <button onClick={() => setFilters({ category: 'books', price: 'any' })}>
        Filter Books
      </button>
      {/* ... */}
    </div>
  );
}
```

**Why it's wrong:**
- `filters` is an object stored in state
- Every call to `setFilters` creates a **new object**
- `useMemo` compares `oldFilters !== newFilters` (different object refs)
- Memo **always** runs because dependency always changes
- You added complexity for zero benefit

**The fix:**

**Option 1: Separate primitive values**
```javascript
function SearchResults({ query }) {
  const [category, setCategory] = useState('all');
  const [price, setPrice] = useState('any');

  const results = useMemo(() => {
    return searchDatabase(query, { category, price });
  }, [query, category, price]);  // Primitives compare correctly

  return (
    <div>
      <button onClick={() => setCategory('books')}>Filter Books</button>
      {/* ... */}
    </div>
  );
}
```

**Option 2: Deep comparison (use sparingly)**
```javascript
import { useDeepMemo } from './hooks';

const results = useDeepMemo(() => {
  return searchDatabase(query, filters);
}, [query, filters]);  // Deep compares filters content
```

**Rule:** useMemo dependencies must be stable primitives or memoized objects. Objects/arrays that change every render defeat the purpose.

---

### Mistake 4: Not Using Keys (or Using Index)

**What juniors do:**
```javascript
// Version 1: No key at all
function TodoList({ todos }) {
  return todos.map(todo => <TodoItem todo={todo} />);
  // React warning: "Each child should have a unique key"
}

// Version 2: Index as key
function TodoList({ todos }) {
  return todos.map((todo, index) => (
    <TodoItem key={index} todo={todo} />
  ));
  // No warning, but still wrong!
}
```

**Why it's wrong:**
```javascript
Initial render:
  todos = [
    { id: 1, text: 'Buy milk' },    // key={0}
    { id: 2, text: 'Walk dog' },    // key={1}
    { id: 3, text: 'Write code' }   // key={2}
  ]

User deletes "Walk dog":
  todos = [
    { id: 1, text: 'Buy milk' },    // key={0} ✅ Same key, React reuses
    { id: 3, text: 'Write code' }   // key={1} ❌ Different item, same key!
  ]

React thinks:
  - Item 0: "Buy milk" → "Buy milk" (no change)
  - Item 1: "Walk dog" → "Write code" (UPDATE item 1)
  - Item 2: "Write code" → DELETED

Reality:
  - Item 1 was deleted
  - Item 2 moved up

Result: React updates the wrong item! Causes bugs with:
  - Input focus
  - Scroll position
  - Animations
  - Component state
```

**The fix:**
```javascript
function TodoList({ todos }) {
  return todos.map(todo => (
    <TodoItem key={todo.id} todo={todo} />  // Stable unique ID
  ));
}
```

**Rule:** Use stable, unique IDs as keys. Never use index unless:
1. List never reorders
2. List never filters
3. List is completely static

(Basically: never use index)

---

### Mistake 5: Expensive Work in Render (Not Memoized)

**What juniors do:**
```javascript
function Dashboard({ data }) {
  // This runs on EVERY render (even when data doesn't change!)
  const stats = calculateStatistics(data);  // 20ms
  const chart = generateChartData(data);     // 15ms
  const report = buildReport(data);          // 10ms

  return (
    <div>
      <Stats data={stats} />
      <Chart data={chart} />
      <Report data={report} />
    </div>
  );
}

function Parent() {
  const [theme, setTheme] = useState('light');

  return (
    <div>
      <button onClick={() => setTheme(t => t === 'light' ? 'dark' : 'light')}>
        Toggle Theme
      </button>
      <Dashboard data={bigDataSet} />
      {/* Clicking theme button → 45ms of recalculation in Dashboard! */}
    </div>
  );
}
```

**Why it's wrong:**
- When Parent re-renders (theme change), Dashboard re-renders
- Dashboard recalculates all stats/charts (45ms)
- None of the calculations depend on theme
- User experiences lag when toggling theme

**The fix:**
```javascript
function Dashboard({ data }) {
  // Only recalculate when data actually changes
  const stats = useMemo(() => calculateStatistics(data), [data]);
  const chart = useMemo(() => generateChartData(data), [data]);
  const report = useMemo(() => buildReport(data), [data]);

  return (
    <div>
      <Stats data={stats} />
      <Chart data={chart} />
      <Report data={report} />
    </div>
  );
}

// Now theme toggle is instant, calculations only run when data changes
```

**How to spot this:**
```javascript
// In your component, add logging:
function Dashboard({ data }) {
  console.log('Dashboard rendering');

  const start = performance.now();
  const stats = calculateStatistics(data);
  console.log(`Stats took ${performance.now() - start}ms`);

  // If you see this log on every render → needs useMemo
}
```

**Rule:** Profile your components. If any calculation takes > 5ms and doesn't need to run every render, memoize it.

---

### Mistake 6: Not Virtualizing Large Lists

**What juniors do:**
```javascript
function UserTable({ users }) {
  // users.length = 10,000
  return (
    <table>
      <tbody>
        {users.map(user => (
          <tr key={user.id}>
            <td>{user.name}</td>
            <td>{user.email}</td>
            <td>{user.role}</td>
          </tr>
        ))}
      </tbody>
    </table>
  );
}
```

**Why it's wrong:**
```javascript
Rendering 10,000 rows:
- Initial render: 10,000 components × 2ms = 20,000ms (20 seconds!)
- DOM nodes: 40,000 (tr + 3 td per row)
- Memory: ~200 MB
- Scrolling: Laggy (browser painting 40,000 elements)
- Mobile: Crashes (out of memory)

User can only see ~20 rows on screen!
We're rendering 9,980 invisible rows for no reason.
```

**The fix:**
```javascript
import { FixedSizeList } from 'react-window';

function UserTable({ users }) {
  const Row = ({ index, style }) => {
    const user = users[index];
    return (
      <div style={style} className="table-row">
        <span>{user.name}</span>
        <span>{user.email}</span>
        <span>{user.role}</span>
      </div>
    );
  };

  return (
    <FixedSizeList
      height={600}
      itemCount={users.length}
      itemSize={50}
      width="100%"
    >
      {Row}
    </FixedSizeList>
  );
}
```

**Results:**
```
Before (rendering all 10,000):
- Initial render: 20,000ms
- Memory: 200 MB
- DOM nodes: 40,000

After (virtualization):
- Initial render: 40ms (only renders ~20 visible rows)
- Memory: 4 MB
- DOM nodes: ~30

500× faster!
```

**When to virtualize:**
- Lists with > 100 items
- Grids/tables with lots of rows
- Infinite scroll feeds
- Any time you render more than fits on screen

**Libraries:**
- `react-window` (lightweight, recommended)
- `react-virtualized` (feature-rich, heavier)

**Rule:** If you're rendering > 100 items, use virtualization or pagination. Never render thousands of components at once.

---

## 🧪 Performance Lab: Measure the Difference

Copy-paste these experiments into a React project to see the performance impact yourself. Use React DevTools Profiler to measure.

### Lab 1: The Cost of Inline Functions

**Copy this code into a new component and profile it:**

```javascript
import { useState, memo, useCallback } from 'react';

// Component that re-renders frequently
function Lab1_InlineFunctions() {
  const [count, setCount] = useState(0);
  const items = Array.from({ length: 1000 }, (_, i) => ({ id: i, name: `Item ${i}` }));

  return (
    <div>
      <button onClick={() => setCount(count + 1)}>
        Increment ({count})
      </button>

      <h3>Version A: Inline functions (slow)</h3>
      <ListWithInlineFunctions items={items} />

      <h3>Version B: Memoized functions (fast)</h3>
      <ListWithMemoizedFunctions items={items} />
    </div>
  );
}

// Version A: Creates 1,000 new functions on every render
const ListWithInlineFunctions = memo(({ items }) => {
  console.log('ListWithInlineFunctions rendered');
  return (
    <div>
      {items.map(item => (
        <div key={item.id}>
          {item.name}
          {/* New function created every Parent render! */}
          <button onClick={() => console.log(item.id)}>Click</button>
        </div>
      ))}
    </div>
  );
});

// Version B: Uses memoized callback
const ListWithMemoizedFunctions = memo(({ items }) => {
  console.log('ListWithMemoizedFunctions rendered');
  return (
    <div>
      {items.map(item => (
        <ItemRow key={item.id} item={item} />
      ))}
    </div>
  );
});

const ItemRow = memo(({ item }) => {
  const handleClick = useCallback(() => {
    console.log(item.id);
  }, [item.id]);

  return (
    <div>
      {item.name}
      <button onClick={handleClick}>Click</button>
    </div>
  );
});
```

**How to measure:**

1. Open React DevTools → Profiler tab
2. Click "Record"
3. Click the "Increment" button 5 times
4. Stop recording
5. Check the Profiler results

**What to look for:**

```
Version A (inline functions):
- Every increment → ListWithInlineFunctions re-renders
- Reason: onClick prop is a new function (oldFunc !== newFunc)
- Render time: ~15-30ms per increment

Version B (memoized):
- Increments → "Did not render" (grayed out in Profiler)
- Reason: useCallback returns same function reference
- Render time: < 1ms
```

**Expected results:**
- Version A: Re-renders on every parent update
- Version B: Only renders once (on mount)
- **Lesson:** `memo()` only works if all props stay equal. Functions need `useCallback()`.

---

### Lab 2: useMemo for Expensive Calculations

**Copy this code and measure the difference:**

```javascript
import { useState, useMemo } from 'react';

function Lab2_ExpensiveCalculation() {
  const [count, setCount] = useState(0);
  const [items] = useState(() =>
    Array.from({ length: 10000 }, (_, i) => ({
      id: i,
      value: Math.random() * 1000
    }))
  );

  return (
    <div>
      <button onClick={() => setCount(count + 1)}>
        Increment ({count})
      </button>

      <h3>Version A: No memoization (slow)</h3>
      <StatsWithoutMemo items={items} />

      <h3>Version B: With useMemo (fast)</h3>
      <StatsWithMemo items={items} />
    </div>
  );
}

// Version A: Calculates on EVERY render
function StatsWithoutMemo({ items }) {
  console.time('Stats calculation (no memo)');

  // Expensive calculation runs every time parent re-renders
  const stats = {
    sum: items.reduce((acc, item) => acc + item.value, 0),
    avg: items.reduce((acc, item) => acc + item.value, 0) / items.length,
    max: Math.max(...items.map(i => i.value)),
    min: Math.min(...items.map(i => i.value))
  };

  console.timeEnd('Stats calculation (no memo)');

  return (
    <div>
      <p>Sum: {stats.sum.toFixed(2)}</p>
      <p>Avg: {stats.avg.toFixed(2)}</p>
      <p>Max: {stats.max.toFixed(2)}</p>
      <p>Min: {stats.min.toFixed(2)}</p>
    </div>
  );
}

// Version B: Calculates only when items change
function StatsWithMemo({ items }) {
  console.time('Stats calculation (with memo)');

  const stats = useMemo(() => {
    return {
      sum: items.reduce((acc, item) => acc + item.value, 0),
      avg: items.reduce((acc, item) => acc + item.value, 0) / items.length,
      max: Math.max(...items.map(i => i.value)),
      min: Math.min(...items.map(i => i.value))
    };
  }, [items]);

  console.timeEnd('Stats calculation (with memo)');

  return (
    <div>
      <p>Sum: {stats.sum.toFixed(2)}</p>
      <p>Avg: {stats.avg.toFixed(2)}</p>
      <p>Max: {stats.max.toFixed(2)}</p>
      <p>Min: {stats.min.toFixed(2)}</p>
    </div>
  );
}
```

**How to measure:**

1. Open browser console
2. Click "Increment" button 5 times
3. Watch the console.time() logs

**What to look for:**

```
Version A (no memo):
  Increment #1: Stats calculation (no memo): 12ms
  Increment #2: Stats calculation (no memo): 13ms
  Increment #3: Stats calculation (no memo): 11ms
  (Calculates every time!)

Version B (with memo):
  Increment #1: Stats calculation (with memo): 12ms
  Increment #2: (no log - didn't run!)
  Increment #3: (no log - didn't run!)
  (Only calculates when items change)
```

**Expected results:**
- Version A: Runs expensive calculation on every increment (~12ms each)
- Version B: Runs once on mount, then never again (items don't change)
- **Lesson:** useMemo prevents expensive recalculations when dependencies are stable.

---

### Lab 3: Keys and List Performance

**Copy this code and see the render difference:**

```javascript
import { useState, useRef } from 'react';

function Lab3_KeysPerformance() {
  const [items, setItems] = useState([
    { id: 1, text: 'Apple' },
    { id: 2, text: 'Banana' },
    { id: 3, text: 'Cherry' },
    { id: 4, text: 'Date' },
    { id: 5, text: 'Elderberry' }
  ]);

  const removeMiddleItem = () => {
    setItems(items => items.filter((_, idx) => idx !== 2));
  };

  return (
    <div>
      <button onClick={removeMiddleItem}>Remove middle item</button>

      <h3>Version A: Index as key (BAD)</h3>
      <ListWithIndexKey items={items} />

      <h3>Version B: ID as key (GOOD)</h3>
      <ListWithIdKey items={items} />
    </div>
  );
}

// Version A: Using index as key
function ListWithIndexKey({ items }) {
  return (
    <div>
      {items.map((item, index) => (
        <ItemWithRenderCount key={index} item={item} label="Index key" />
      ))}
    </div>
  );
}

// Version B: Using stable ID as key
function ListWithIdKey({ items }) {
  return (
    <div>
      {items.map(item => (
        <ItemWithRenderCount key={item.id} item={item} label="ID key" />
      ))}
    </div>
  );
}

// This component tracks how many times it renders
function ItemWithRenderCount({ item, label }) {
  const renderCount = useRef(0);
  renderCount.current += 1;

  return (
    <div style={{ padding: '8px', border: '1px solid #ccc', margin: '4px' }}>
      {item.text} - <small>({label}: rendered {renderCount.current} times)</small>
    </div>
  );
}
```

**How to measure:**

1. Click "Remove middle item" button
2. Watch the render counts

**What to observe:**

```
Initial state:
Version A (index key):
  Apple - rendered 1 times
  Banana - rendered 1 times
  Cherry - rendered 1 times
  Date - rendered 1 times
  Elderberry - rendered 1 times

Version B (ID key):
  Apple - rendered 1 times
  Banana - rendered 1 times
  Cherry - rendered 1 times
  Date - rendered 1 times
  Elderberry - rendered 1 times

After clicking "Remove middle item":
Version A (index key):
  Apple - rendered 1 times      ← Stayed the same (key=0)
  Banana - rendered 1 times     ← Stayed the same (key=1)
  Date - rendered 2 times       ← RE-RENDERED (was key=3, now key=2)
  Elderberry - rendered 2 times ← RE-RENDERED (was key=4, now key=3)

Version B (ID key):
  Apple - rendered 1 times      ← Stayed the same (key=1)
  Banana - rendered 1 times     ← Stayed the same (key=2)
  Date - rendered 1 times       ← NOT re-rendered (key=4 still key=4)
  Elderberry - rendered 1 times ← NOT re-rendered (key=5 still key=5)
```

**Expected results:**
- **Index as key:** Items after deletion re-render (keys changed: 3→2, 4→3)
- **ID as key:** Items don't re-render (keys stable: 4 stays 4, 5 stays 5)
- **Lesson:** Stable keys let React reuse existing components, avoiding unnecessary re-renders.

**Real impact with 1,000 items:**
- Index key: Deleting item 1 → 999 items re-render
- ID key: Deleting item 1 → 0 items re-render
- **Performance difference: 1000× on large lists!**

---

## Chapter Exercise: Performance Audit

Profile and optimize this application:

```javascript
function Dashboard({ userId }) {
  const [user, setUser] = useState(null);
  const [posts, setPosts] = useState([]);
  const [comments, setComments] = useState([]);
  const [filter, setFilter] = useState('all');

  useEffect(() => {
    fetchUser(userId).then(setUser);
    fetchPosts(userId).then(setPosts);
    fetchComments(userId).then(setComments);
  }, [userId]);

  const filteredPosts = posts.filter(post => {
    if (filter === 'all') return true;
    return post.category === filter;
  });

  const stats = {
    totalPosts: posts.length,
    totalComments: comments.length,
    avgComments: comments.length / posts.length
  };

  return (
    <div>
      <UserHeader user={user} stats={stats} />
      <FilterBar value={filter} onChange={setFilter} />
      <PostList posts={filteredPosts} comments={comments} />
      <CommentSection comments={comments} />
    </div>
  );
}

// Task: Optimize this component
// - Measure current performance
// - Identify issues
// - Apply optimizations
// - Measure improvements
// - Document changes
```

---

## 📝 Cumulative Review: Test Your Mastery

Before moving to the next chapter, test your understanding of performance optimization.

### Question 1: Profiling First (Section 4.1 + 4.2)

**Scenario:** A junior developer says: "The app feels slow, so I added `React.memo()` to every component and wrapped all calculations in `useMemo()`. The app is still slow!"

**What went wrong, and what should they have done instead?**

<details>
<summary>✅ Answer</summary>

**What went wrong:**
1. **Optimized without measuring** - Violated Rule #1: Don't optimize prematurely
2. **Added unnecessary overhead** - memo() and useMemo() have costs (comparison functions, memory)
3. **Didn't identify the actual bottleneck** - Could be one slow component, not everything
4. **Made code more complex** - Harder to read and maintain for zero benefit

**What they should have done:**
1. **Profile first** - Open React DevTools Profiler and record an interaction
2. **Find the bottleneck** - Look at the Ranked Chart for the slowest component
3. **Investigate why it's slow:**
   - Check "Why did this render?"
   - Look at flame graph to see expensive children
   - Measure if it's a calculation or re-render issue
4. **Fix ONE thing** - Apply the right optimization (memo, useMemo, or virtualization)
5. **Measure again** - Verify the fix actually helped
6. **Repeat if needed** - Profile → fix → measure

**Key insight:** Most apps have 1-2 real bottlenecks. Blindly optimizing everything wastes time and adds complexity. Always measure first.

</details>

---

### Question 2: Keys and Re-renders (Section 4.5 + Common Mistakes)

**Scenario:** You have a list of 500 items. When you delete one item from the middle, the app freezes for 2 seconds. Looking at the Profiler, you see all 499 remaining items re-rendered.

**What's the likely cause, and how do you fix it?**

<details>
<summary>✅ Answer</summary>

**Likely cause: Using index as key**

```javascript
// The problem:
{items.map((item, index) => (
  <ListItem key={index} item={item} />
))}

// What happens when you delete item at index 2:
// Before: [0: Apple, 1: Banana, 2: Cherry, 3: Date, 4: Elderberry]
// After:  [0: Apple, 1: Banana, 2: Date,   3: Elderberry]

// React sees:
// - key=0 (Apple) → key=0 (Apple) ✅ reuse
// - key=1 (Banana) → key=1 (Banana) ✅ reuse
// - key=2 (Cherry) → key=2 (Date) ❌ different data, UPDATE
// - key=3 (Date) → key=3 (Elderberry) ❌ different data, UPDATE
// - key=4 (Elderberry) → DELETED

// Result: All items after deletion get re-rendered!
```

**The fix:**
```javascript
{items.map(item => (
  <ListItem key={item.id} item={item} />
))}

// With stable IDs:
// - key=1 (Apple) → key=1 (Apple) ✅ reuse
// - key=2 (Banana) → key=2 (Banana) ✅ reuse
// - key=3 (Cherry) → DELETED
// - key=4 (Date) → key=4 (Date) ✅ reuse (same key!)
// - key=5 (Elderberry) → key=5 (Elderberry) ✅ reuse

// Result: 0 items re-render, React just removes the deleted one
```

**Impact:**
- Before: 499 components re-render = 2+ seconds
- After: 0 components re-render = instant
- **Lesson:** Stable keys are not optional for dynamic lists

</details>

---

### Question 3: The Memoization Trap (Section 4.4 + Mistake #2)

**You memo'd a component but it still re-renders on every parent update. Here's the code:**

```javascript
const ExpensiveChart = memo(({ data, onClick }) => {
  return <Chart data={data} onClick={onClick} />;
});

function Dashboard() {
  const [theme, setTheme] = useState('light');
  const chartData = useMemo(() => processData(), []);

  return (
    <>
      <button onClick={() => setTheme(t => t === 'light' ? 'dark' : 'light')}>
        Toggle Theme
      </button>
      <ExpensiveChart
        data={chartData}
        onClick={(point) => console.log(point)}
      />
    </>
  );
}
```

**Why does ExpensiveChart re-render when theme changes, and how do you fix it?**

<details>
<summary>✅ Answer</summary>

**Why it re-renders:**
- `data={chartData}` is memoized → same reference ✅
- `onClick={(point) => console.log(point)}` is **inline function** → new reference every render ❌
- `memo()` does shallow comparison: `oldOnClick !== newOnClick` → re-render
- Toggling theme → Dashboard re-renders → new onClick function → ExpensiveChart re-renders

**The fix:**
```javascript
function Dashboard() {
  const [theme, setTheme] = useState('light');
  const chartData = useMemo(() => processData(), []);

  // Memoize the callback
  const handleChartClick = useCallback((point) => {
    console.log(point);
  }, []);  // Empty deps = function never changes

  return (
    <>
      <button onClick={() => setTheme(t => t === 'light' ? 'dark' : 'light')}>
        Toggle Theme
      </button>
      <ExpensiveChart
        data={chartData}
        onClick={handleChartClick}  // Same function reference every render
      />
    </>
  );
}
```

**The Rule:** If you use `React.memo()` on a component, you must also memoize:
- All function props (with `useCallback`)
- All object/array props (with `useMemo` or extract them outside component)

Otherwise, memo does nothing - the overhead of comparison with zero benefit.

**How to catch this:**
- Add `console.log('ExpensiveChart rendered')` inside the component
- If you see it log on every parent update → you're passing unstable props

</details>

---

### Question 4: When NOT to Optimize (Section 4.1 + 4.6 + Mistake #1)

**A junior developer shows you this code and asks if it needs optimization:**

```javascript
function UserProfile({ user }) {
  // Junior: "Should I wrap this in useMemo?"
  const displayName = `${user.firstName} ${user.lastName}`;

  // Junior: "Should I memo this component?"
  return (
    <div>
      <Avatar src={user.avatar} />
      <span>{displayName}</span>
    </div>
  );
}
```

**Should you add memoization? Why or why not?**

<details>
<summary>✅ Answer</summary>

**NO - Don't optimize this!**

**Why useMemo is not needed for `displayName`:**
```javascript
const displayName = `${user.firstName} ${user.lastName}`;
// This is a simple string concatenation: < 0.01ms

const displayName = useMemo(() => `${user.firstName} ${user.lastName}`, [user.firstName, user.lastName]);
// useMemo overhead: ~0.02ms (comparison + function call)
// You made it SLOWER by memoizing!
```

**Why React.memo is not needed for UserProfile:**
- Component does trivial work (render 2 simple elements)
- Render time: ~0.5ms
- memo overhead: ~0.3ms (shallow prop comparison)
- Not worth it unless you profile and see it's a bottleneck

**When TO add optimization:**
```javascript
// THIS needs useMemo:
const stats = expensiveCalculation(bigDataSet);  // 50ms!

// THIS needs memo:
const ExpensiveChart = memo(function Chart({ data }) {
  // D3 visualization that takes 100ms to render
});
```

**The Rule:**
1. **Don't guess** - Profile first
2. **Only optimize if:**
   - Calculation takes > 5ms (profile it!)
   - Component renders often with same props
   - You've measured the improvement
3. **Premature optimization adds:**
   - Complexity (harder to read)
   - Memory usage (memoized values stored)
   - Potential overhead (if used incorrectly)

**Senior insight:** The best optimization is often no optimization. Keep code simple until you have evidence it's slow.

</details>

---

### Question 5: Real-World Performance Budget (Section 4.8 + War Story)

**You're launching a major e-commerce redesign on Black Friday. Your PM asks: "How do we prevent the kind of performance disaster described in the war story?"**

**What specific preventions would you implement BEFORE launch?**

<details>
<summary>✅ Answer</summary>

**Preventions to implement BEFORE Black Friday:**

**1. Performance Budget in CI/CD (Automated Prevention):**
```javascript
// lighthouse.config.js - fails PR if budgets exceeded
module.exports = {
  ci: {
    assert: {
      assertions: {
        'first-contentful-paint': ['error', { maxNumericValue: 1800 }],
        'largest-contentful-paint': ['error', { maxNumericValue: 2500 }],
        'interactive': ['error', { maxNumericValue: 3800 }],
        'total-byte-weight': ['error', { maxNumericValue: 500000 }],
      },
    },
  },
};

// package.json
"scripts": {
  "test:perf": "lhci autorun",
}

// .github/workflows/pr.yml
- name: Performance Check
  run: npm run test:perf
```
- **Benefit:** Catches performance regressions before they reach production
- **Cost:** $0 (Lighthouse CI is free)

**2. Mandatory Code Review Checklist:**
```markdown
## Performance Review (required for all PRs)
- [ ] Tested with PRODUCTION data volumes (not 5 mock items!)
- [ ] Used stable IDs as keys (no index keys)
- [ ] Lists > 100 items use virtualization or pagination
- [ ] Expensive calculations wrapped in useMemo
- [ ] No inline objects/functions passed to memo'd components
- [ ] Profiler screenshot attached (for components rendering > 50 items)
```

**3. Load Testing with Real Data:**
```bash
# Test with actual Black Friday traffic patterns
# Don't test with 20 products, test with 5,000!
npm run test:load -- --products=5000 --users=100000

# Test on low-end devices (not just your MacBook Pro)
npm run test:mobile -- --device="Moto G4" --network="3G"
```

**4. Performance Monitoring in Production:**
```javascript
// Real User Monitoring (RUM)
import { onLCP, onFID, onCLS } from 'web-vitals';

onLCP(metric => {
  if (metric.value > 2500) {
    analytics.track('performance_warning', {
      metric: 'LCP',
      value: metric.value,
      page: window.location.pathname
    });
  }
});

// Alert on Slack if > 10 slow interactions in 5 minutes
// Catch issues BEFORE users flood support
```

**5. Feature Flags for Risky Changes:**
```javascript
// Roll out redesign to 1% of users first
if (featureFlags.newProductGrid && userBucket < 0.01) {
  return <NewProductGrid products={products} />;
}
return <OldProductGrid products={products} />;

// Monitor metrics for 24 hours before full rollout
// Can kill switch instantly if problems detected
```

**6. Pre-Launch Performance Audit:**
- [ ] Run Profiler on product pages with 1,000+ products
- [ ] Check memory usage on mobile devices
- [ ] Verify no components rendering > 50ms
- [ ] Test sorting/filtering with full product catalog
- [ ] Measure Time to Interactive < 3.8s on 3G

**The Cost-Benefit:**
- **Cost of prevention:** ~2 engineer-days + $0 for tools = ~$1,600
- **Cost of Black Friday disaster:** $2.4 million
- **ROI: 1,500× return on investment**

**Senior lesson:** The war story disaster cost $2.4M and could have been prevented by:
1. Running Lighthouse in CI (free)
2. Testing with real data volumes (2 hours)
3. Code review catching inline functions/index keys (free)

Performance in production is exponentially more expensive to fix than preventing in development.

</details>

---

### Question 6: Integration Challenge (All Sections)

**You're reviewing a PR that optimizes a slow dashboard. The developer claims it's "5× faster now". Here's the code:**

```javascript
// Before optimization:
function Dashboard({ data }) {
  const stats = calculateStats(data);  // 30ms
  return (
    <div>
      {data.map((item, idx) => (
        <Card key={idx} item={item} stats={stats} />
      ))}
    </div>
  );
}

// After "optimization":
const Dashboard = memo(function Dashboard({ data }) {
  const stats = useMemo(() => calculateStats(data), [data]);

  return (
    <div>
      {data.map((item, idx) => (
        <MemoizedCard key={idx} item={item} stats={stats} />
      ))}
    </div>
  );
});

const MemoizedCard = memo(({ item, stats }) => {
  return <Card item={item} stats={stats} />;
});
```

**Review this code like a senior engineer. What's good, what's still wrong, and what would you change?**

<details>
<summary>✅ Answer</summary>

**What's GOOD ✅:**
1. **Memoized expensive calculation** - `useMemo` prevents recalculating stats on every render
2. **Memoized Card components** - Prevents unnecessary re-renders of cards
3. **Wrapped Dashboard in memo** - Prevents re-renders when parent updates

**What's STILL WRONG ❌:**

**Critical Issue: Index as key**
```javascript
{data.map((item, idx) => (
  <MemoizedCard key={idx} item={item} stats={stats} />
))}
```
- If data changes (filter, sort, delete), index keys will cause ALL cards to re-render
- Defeats the purpose of memoizing Card!
- **Fix:** Use stable ID: `key={item.id}`

**Issue: stats passed to every Card**
```javascript
<MemoizedCard key={idx} item={item} stats={stats} />
// If stats is an object, it's a new reference when data changes
// ALL cards re-render even if their individual item didn't change!
```
- When data updates → stats recalculates → new object → all cards see new prop
- **Fix:** Only pass stats to components that need it, or pass specific values

**Issue: Probably over-memo'd**
- Is Dashboard's parent re-rendering with same data? If not, memo(Dashboard) is wasted
- Profile first!

**My recommended changes:**

```javascript
function Dashboard({ data }) {
  // Good: memoize expensive calculation
  const stats = useMemo(() => calculateStats(data), [data]);

  return (
    <div>
      <StatsHeader stats={stats} />  {/* Only this needs stats */}
      <CardList items={data} />       {/* Cards don't need stats */}
    </div>
  );
}

// Virtualize if > 100 items (not shown in original!)
function CardList({ items }) {
  if (items.length > 100) {
    return <VirtualizedList items={items} />;
  }

  return items.map(item => (
    <Card key={item.id} item={item} />  {/* Stable key! */}
  ));
}

// Only memo if profiling shows it's needed
const Card = memo(({ item }) => {
  return <div>{item.name}</div>;
});
```

**Performance testing I'd require:**
```javascript
// Show me Profiler screenshots proving:
// 1. Before: X ms per render
// 2. After: Y ms per render
// 3. "5× faster" claim (X/Y = 5?)

// Test with realistic data:
// - 1,000 items (not 10!)
// - Delete middle item (does it re-render all cards?)
// - Sort list (does it re-render all cards?)
```

**My code review comment:**
> Good start on optimization! But:
> 1. ❌ Change `key={idx}` to `key={item.id}` (critical - breaks memo)
> 2. ❌ Don't pass `stats` to every card if they don't use it
> 3. ⚠️ Add virtualization if > 100 items
> 4. 📊 Please attach Profiler screenshots showing the "5× faster" claim
> 5. ✅ useMemo for stats is good!
>
> Without fixing the key issue, this won't actually improve performance on data changes.

**Lesson:** Optimization without understanding can make things worse. This code added complexity (3 memos!) but still has the index-key bug that defeats it all.

</details>

---

## Review Checklist

- [ ] Profile before optimizing
- [ ] Use React DevTools Profiler effectively
- [ ] Identify render causes
- [ ] Apply React.memo appropriately
- [ ] Optimize list rendering with keys
- [ ] Use virtualization for large lists
- [ ] Memoize expensive calculations
- [ ] Implement lazy loading for images
- [ ] Set and measure performance budgets
- [ ] Document optimization decisions

## Key Takeaways

1. **Measure first** - Don't guess what's slow
2. **Profile with DevTools** - Let data guide you
3. **One change at a time** - Measure impact
4. **Not everything needs optimization** - Focus on bottlenecks
5. **Keys matter** - Proper keys prevent unnecessary renders
6. **Virtualize large lists** - Don't render 10,000 items
7. **Memoization has cost** - Only use when beneficial
8. **Performance is a feature** - Treat it as such

## Further Reading

- React DevTools Profiler docs
- "React Performance" by Kent C. Dodds
- Web Vitals documentation
- "High Performance Browser Networking"

## Next Chapter

[Chapter 5: Code Splitting & Lazy Loading](./05-code-splitting.md)
