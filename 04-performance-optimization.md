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

### Profiler Features

```javascript
// 1. Ranked Chart - slowest components first
// 2. Flame Graph - component hierarchy with timing
// 3. Interactions - track user interactions
// 4. Why did this render? - shows props/state changes

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

### Hands-On Exercise 4.2

Profile and fix this slow component:

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

// Tasks:
// 1. Profile and identify issues
// 2. Fix the issues
// 3. Measure improvement
// 4. Document what you changed and why
```

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
