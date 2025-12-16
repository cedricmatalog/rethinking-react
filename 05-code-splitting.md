# Chapter 5: Code Splitting & Lazy Loading

## Introduction

Junior developers ship one massive JavaScript bundle. Senior developers ship what users need, when they need it.

Code splitting is one of the most impactful optimizations you can make.

## Learning Objectives

- Understand bundle analysis
- Implement route-based code splitting
- Use component-based lazy loading
- Optimize third-party libraries
- Measure and improve load times

## 5.1 Understanding Your Bundle

### Bundle Analysis

```bash
# Install bundle analyzer
npm install --save-dev webpack-bundle-analyzer

# For Create React App
npm install --save-dev cra-bundle-analyzer
npx cra-bundle-analyzer

# For Next.js (built-in)
npm run build -- --analyze

# For Vite
npm install --save-dev rollup-plugin-visualizer
```

### Reading the Bundle Map

```javascript
// What you'll see:
// - Main bundle: 2.5MB (800KB gzipped)
//   - React: 140KB
//   - moment.js: 200KB (!!)
//   - lodash: 70KB
//   - Your code: 500KB
//   - node_modules: 1.6MB

// Red flags:
// 1. Large unused libraries
// 2. Duplicate dependencies
// 3. Moment.js (use date-fns or day.js instead)
// 4. Entire lodash instead of individual functions
```

### Measuring Real Impact

```javascript
// Check bundle size impact
import { useState } from 'react';

// Bad: Imports entire lodash (70KB)
import _ from 'lodash';

// Good: Import only what you need
import debounce from 'lodash/debounce';

// Better: Use native when possible
const debounce = (fn, ms) => {
  let timeout;
  return (...args) => {
    clearTimeout(timeout);
    timeout = setTimeout(() => fn(...args), ms);
  };
};
```

## 5.2 Route-Based Code Splitting

### React.lazy and Suspense

```javascript
// Before: All routes loaded upfront
import Home from './pages/Home';
import About from './pages/About';
import Dashboard from './pages/Dashboard';
import Settings from './pages/Settings';

// After: Routes loaded on demand
import { lazy, Suspense } from 'react';

const Home = lazy(() => import('./pages/Home'));
const About = lazy(() => import('./pages/About'));
const Dashboard = lazy(() => import('./pages/Dashboard'));
const Settings = lazy(() => import('./pages/Settings'));

function App() {
  return (
    <Router>
      <Suspense fallback={<PageLoader />}>
        <Routes>
          <Route path="/" element={<Home />} />
          <Route path="/about" element={<About />} />
          <Route path="/dashboard" element={<Dashboard />} />
          <Route path="/settings" element={<Settings />} />
        </Routes>
      </Suspense>
    </Router>
  );
}
```

### Loading States

```javascript
// Basic loading
<Suspense fallback={<div>Loading...</div>}>
  <LazyComponent />
</Suspense>

// Better loading with skeleton
<Suspense fallback={<PageSkeleton />}>
  <LazyComponent />
</Suspense>

// Best: Context-aware loading
function RouteLoader() {
  return (
    <div className="route-loader">
      <Spinner />
      <p>Loading page...</p>
    </div>
  );
}

<Suspense fallback={<RouteLoader />}>
  <Routes />
</Suspense>
```

### Nested Loading States

```javascript
function Dashboard() {
  return (
    <div>
      <Header /> {/* Always loaded */}

      <Suspense fallback={<Spinner />}>
        <DashboardContent /> {/* Lazy loaded */}
      </Suspense>

      <Suspense fallback={<ChartSkeleton />}>
        <Analytics /> {/* Separately lazy loaded */}
      </Suspense>
    </div>
  );
}
```

## 5.3 Component-Based Code Splitting

### Heavy Components

```javascript
// Heavy chart library - load only when needed
const HeavyChart = lazy(() => import('./HeavyChart'));

function DataVisualization({ data }) {
  const [showChart, setShowChart] = useState(false);

  return (
    <div>
      <button onClick={() => setShowChart(true)}>
        Show Chart
      </button>

      {showChart && (
        <Suspense fallback={<ChartSkeleton />}>
          <HeavyChart data={data} />
        </Suspense>
      )}
    </div>
  );
}
```

### Modal Content

```javascript
// Don't load modal content until modal opens
function App() {
  const [isModalOpen, setIsModalOpen] = useState(false);

  // Lazy load modal content
  const ModalContent = lazy(() => import('./ModalContent'));

  return (
    <>
      <button onClick={() => setIsModalOpen(true)}>
        Open Modal
      </button>

      {isModalOpen && (
        <Modal onClose={() => setIsModalOpen(false)}>
          <Suspense fallback={<Spinner />}>
            <ModalContent />
          </Suspense>
        </Modal>
      )}
    </>
  );
}
```

### Conditional Features

```javascript
// Load editor only for premium users
function DocumentPage({ user, document }) {
  const Editor = useMemo(() => {
    if (user.isPremium) {
      return lazy(() => import('./RichTextEditor'));
    }
    return lazy(() => import('./SimpleTextarea'));
  }, [user.isPremium]);

  return (
    <Suspense fallback={<EditorSkeleton />}>
      <Editor content={document.content} />
    </Suspense>
  );
}
```

## 5.4 Preloading Strategies

### Link Prefetching

```javascript
// Prefetch on hover
function NavigationLink({ to, children }) {
  const prefetchRoute = () => {
    // Dynamically import the route component
    const route = routeMap[to];
    if (route) {
      route(); // This triggers the import
    }
  };

  return (
    <Link
      to={to}
      onMouseEnter={prefetchRoute}
      onTouchStart={prefetchRoute}
    >
      {children}
    </Link>
  );
}

// Route map for prefetching
const routeMap = {
  '/dashboard': () => import('./pages/Dashboard'),
  '/settings': () => import('./pages/Settings'),
  '/profile': () => import('./pages/Profile')
};
```

### Intelligent Prefetching

```javascript
// Prefetch based on user behavior
function useIntelligentPrefetch() {
  const location = useLocation();

  useEffect(() => {
    // After 2 seconds on a page, prefetch likely next pages
    const timer = setTimeout(() => {
      const likelyNextRoutes = predictNextRoute(location.pathname);

      likelyNextRoutes.forEach(route => {
        const component = routeMap[route];
        if (component) {
          component(); // Prefetch
        }
      });
    }, 2000);

    return () => clearTimeout(timer);
  }, [location]);
}

function predictNextRoute(currentPath) {
  // Based on analytics/common patterns
  const patterns = {
    '/': ['/dashboard', '/products'],
    '/products': ['/products/:id', '/cart'],
    '/cart': ['/checkout']
  };

  return patterns[currentPath] || [];
}
```

### Resource Hints

```javascript
// In your HTML head
<head>
  {/* DNS prefetch */}
  <link rel="dns-prefetch" href="https://api.example.com" />

  {/* Preconnect */}
  <link rel="preconnect" href="https://api.example.com" />

  {/* Prefetch */}
  <link rel="prefetch" href="/dashboard-chunk.js" />

  {/* Preload (high priority) */}
  <link rel="preload" href="/critical.js" as="script" />
</head>

// In React component
function useResourceHints() {
  useEffect(() => {
    // Add prefetch link dynamically
    const link = document.createElement('link');
    link.rel = 'prefetch';
    link.href = '/next-page-chunk.js';
    document.head.appendChild(link);

    return () => document.head.removeChild(link);
  }, []);
}
```

## 5.5 Third-Party Library Optimization

### Lazy Load Heavy Libraries

```javascript
// Bad: Load PDF viewer upfront
import PDFViewer from 'react-pdf-viewer';

// Good: Load when needed
function DocumentViewer({ url }) {
  const [showViewer, setShowViewer] = useState(false);

  const PDFViewer = lazy(() =>
    import('react-pdf-viewer').then(module => ({
      default: module.PDFViewer
    }))
  );

  if (!showViewer) {
    return (
      <button onClick={() => setShowViewer(true)}>
        View PDF
      </button>
    );
  }

  return (
    <Suspense fallback={<Loading />}>
      <PDFViewer url={url} />
    </Suspense>
  );
}
```

### Tree Shaking

```javascript
// Bad: Imports entire library
import { Button, Modal, Input, Select, DatePicker } from 'ui-library';

// Good: Import specific components (if library supports tree-shaking)
import Button from 'ui-library/Button';
import Modal from 'ui-library/Modal';

// Check package.json for:
{
  "sideEffects": false // Enables tree shaking
}
```

### Dynamic Imports with Named Exports

```javascript
// When importing named exports
const { BarChart } = await import('recharts');

// Or with React.lazy
const BarChart = lazy(() =>
  import('recharts').then(module => ({
    default: module.BarChart
  }))
);
```

## 5.6 Advanced Patterns

### Error Boundaries with Retry

```javascript
class LazyLoadErrorBoundary extends React.Component {
  state = { hasError: false };

  static getDerivedStateFromError() {
    return { hasError: true };
  }

  retry = () => {
    this.setState({ hasError: false });
  };

  render() {
    if (this.state.hasError) {
      return (
        <div>
          <p>Failed to load component</p>
          <button onClick={this.retry}>Retry</button>
        </div>
      );
    }

    return this.props.children;
  }
}

// Usage
<LazyLoadErrorBoundary>
  <Suspense fallback={<Loading />}>
    <LazyComponent />
  </Suspense>
</LazyLoadErrorBoundary>
```

### Progressive Enhancement

```javascript
// Load features progressively
function Editor() {
  const [features, setFeatures] = useState({
    basic: true,
    spellcheck: false,
    grammar: false,
    ai: false
  });

  useEffect(() => {
    // Load spell check after 1s
    setTimeout(() => {
      import('./features/spellcheck').then(() => {
        setFeatures(f => ({ ...f, spellcheck: true }));
      });
    }, 1000);

    // Load grammar after 3s
    setTimeout(() => {
      import('./features/grammar').then(() => {
        setFeatures(f => ({ ...f, grammar: true }));
      });
    }, 3000);

    // Load AI features after 5s
    setTimeout(() => {
      import('./features/ai').then(() => {
        setFeatures(f => ({ ...f, ai: true }));
      });
    }, 5000);
  }, []);

  return (
    <div>
      <BasicEditor />
      {features.spellcheck && <SpellCheckIndicator />}
      {features.grammar && <GrammarSuggestions />}
      {features.ai && <AISuggestions />}
    </div>
  );
}
```

### Chunking Strategies

```javascript
// webpack.config.js or next.config.js

module.exports = {
  optimization: {
    splitChunks: {
      chunks: 'all',
      cacheGroups: {
        // Vendor chunk
        vendor: {
          test: /[\\/]node_modules[\\/]/,
          name: 'vendors',
          priority: 10
        },
        // React chunk
        react: {
          test: /[\\/]node_modules[\\/](react|react-dom)[\\/]/,
          name: 'react',
          priority: 20
        },
        // UI library chunk
        ui: {
          test: /[\\/]node_modules[\\/](antd|@mui)[\\/]/,
          name: 'ui-library',
          priority: 15
        },
        // Common code chunk
        common: {
          minChunks: 2,
          priority: 5,
          reuseExistingChunk: true
        }
      }
    }
  }
};
```

## 5.7 Measuring Impact

### Performance Metrics

```javascript
function measureLoadTime(componentName, importFn) {
  return async () => {
    const start = performance.now();

    try {
      const module = await importFn();
      const end = performance.now();

      console.log(
        `${componentName} loaded in ${(end - start).toFixed(2)}ms`
      );

      // Send to analytics
      analytics.track('Lazy Load', {
        component: componentName,
        duration: end - start
      });

      return module;
    } catch (error) {
      console.error(`Failed to load ${componentName}`, error);
      throw error;
    }
  };
}

// Usage
const Dashboard = lazy(
  measureLoadTime('Dashboard', () => import('./Dashboard'))
);
```

### Lighthouse Audits

```javascript
// Key metrics to track:
// - First Contentful Paint (FCP)
// - Largest Contentful Paint (LCP)
// - Time to Interactive (TTI)
// - Total Blocking Time (TBT)
// - Speed Index

// Before code splitting:
// FCP: 2.5s
// LCP: 4.2s
// TTI: 5.8s
// Bundle: 2.5MB

// After code splitting:
// FCP: 0.8s (-68%)
// LCP: 1.6s (-62%)
// TTI: 2.3s (-60%)
// Initial bundle: 350KB (-86%)
```

## Real-World Scenario: Splitting a Monolithic App

### The Challenge

Your React app has:
- One 3MB bundle
- 15 seconds load time on 3G
- 50+ routes
- Multiple heavy libraries (charts, PDF viewer, rich text editor)

### Your Task

1. **Analyze the bundle**
   - Identify largest chunks
   - Find unnecessary code
   - Locate optimization opportunities

2. **Create splitting strategy**
   - Route-based splits
   - Component-based splits
   - Library splits

3. **Implement progressively**
   - Start with routes
   - Add component splits
   - Optimize libraries

4. **Measure and iterate**
   - Track load times
   - Monitor user experience
   - Optimize further

## Chapter Exercise: Optimize an App

Given this application:

```javascript
// App with no code splitting
import Dashboard from './Dashboard';
import Analytics from './Analytics';
import Reports from './Reports';
import Settings from './Settings';
import RichTextEditor from 'heavy-editor-lib';
import ChartLibrary from 'heavy-chart-lib';
import PDFViewer from 'heavy-pdf-lib';

function App() {
  return (
    <Router>
      <Routes>
        <Route path="/" element={<Dashboard />} />
        <Route path="/analytics" element={<Analytics />} />
        <Route path="/reports" element={<Reports />} />
        <Route path="/settings" element={<Settings />} />
      </Routes>
    </Router>
  );
}
```

**Tasks:**
1. Analyze current bundle size
2. Implement route-based code splitting
3. Add component-based lazy loading
4. Optimize third-party libraries
5. Add preloading for common paths
6. Measure improvements
7. Document strategy

**Success Criteria:**
- 70%+ reduction in initial bundle
- FCP < 1.5s
- LCP < 2.5s
- No visible loading jank

## Review Checklist

- [ ] Analyze bundle with visualizer
- [ ] Implement route-based code splitting
- [ ] Lazy load heavy components
- [ ] Optimize third-party libraries
- [ ] Add intelligent prefetching
- [ ] Handle loading states gracefully
- [ ] Implement error boundaries for lazy components
- [ ] Measure performance impact
- [ ] Set up bundle size monitoring

## Key Takeaways

1. **Analyze first** - Know what's in your bundle
2. **Route splitting is easiest** - Start there
3. **Lazy load heavy features** - Charts, editors, PDFs
4. **Prefetch intelligently** - Based on user behavior
5. **Handle errors gracefully** - Lazy loading can fail
6. **Measure impact** - Use Lighthouse and real metrics
7. **Monitor over time** - Bundle size creeps up
8. **Progressive enhancement** - Load features as needed

## Further Reading

- "Code Splitting" - React docs
- "Import Cost" VS Code extension
- "webpack-bundle-analyzer" documentation
- "Next.js Automatic Static Optimization"

## Next Chapter

[Chapter 6: Memory Management & Debugging](./06-memory-debugging.md)
