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

### Junior vs Senior Perspective

**Junior Approach:**
"My app loads slow, so I'll minify the code and enable gzip. That should fix it, right? Oh, and I read about lazy loading, so I'll lazy load everything! Every single component will be `lazy()`!"

**Senior Approach:**
"Let me first analyze the bundle with webpack-bundle-analyzer to see what's actually taking space. Ah, I see moment.js is 200KB and we only use it once. That's the real problem. I'll replace it with date-fns (2KB) and add route-based code splitting for the 5 major routes. Component-level lazy loading only for the heavy chart library and PDF viewer - not everything."

**The Difference:**
- Junior: Optimizes blindly without data, over-optimizes trivial components
- Senior: Analyzes first, fixes the biggest wins (heavy libraries, route splits), measures improvement

**Key Insight:** 80% of your bundle size usually comes from 20% of your dependencies. Find those heavy libraries first.

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

What webpack-bundle-analyzer shows you:

```
Bundle Composition (2.5MB uncompressed, 800KB gzipped):

┌────────────────────────────────────────────────────────────────┐
│                      main.bundle.js (2.5MB)                    │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  ┌────────────────────────────────────────────┐  node_modules │
│  │  moment.js (200KB) 🚨 REPLACE THIS!        │  (1.6MB)      │
│  │  moment-timezone (180KB) 🚨                │  64%          │
│  ├────────────────────────────────────────────┤               │
│  │  lodash (70KB) ⚠️  Import specific funcs   │               │
│  │  react (40KB)                              │               │
│  │  react-dom (100KB)                         │               │
│  │  recharts (400KB) 💡 Lazy load this        │               │
│  │  react-pdf (300KB) 💡 Lazy load this       │               │
│  │  other deps (310KB)                        │               │
│  └────────────────────────────────────────────┘               │
│                                                                │
│  ┌────────────────────────────────────────────┐  Your Code    │
│  │  src/ (500KB)                              │  (500KB)      │
│  │    - routes/ (200KB)                       │  20%          │
│  │    - components/ (180KB)                   │               │
│  │    - utils/ (80KB)                         │               │
│  │    - other/ (40KB)                         │               │
│  └────────────────────────────────────────────┘               │
│                                                                │
│  ┌────────────────────────────────────────────┐  Vendor       │
│  │  webpack runtime (15KB)                    │  (400KB)      │
│  │  polyfills (50KB)                          │  16%          │
│  │  CSS (80KB)                                │               │
│  │  other (255KB)                             │               │
│  └────────────────────────────────────────────┘               │
│                                                                │
└────────────────────────────────────────────────────────────────┘

🚨 RED FLAGS (fix these first):
1. moment.js + moment-timezone = 380KB → Replace with date-fns (2KB)
2. Entire lodash (70KB) → Import specific functions (debounce = 2KB)
3. recharts (400KB) → Lazy load (only used on /analytics page)
4. react-pdf (300KB) → Lazy load (only used in /documents)

💡 QUICK WINS:
Replace moment.js → save 378KB  (-15% total bundle!)
Lazy load recharts → save 400KB  (-16% initial load!)
Lazy load react-pdf → save 300KB (-12% initial load!)

Total potential savings: 1,078KB / 2,500KB = 43% smaller bundle!
```

**How to read this:**
- **Size** = Actual file size (bigger boxes = more bytes)
- **Red flags (🚨)** = Heavy libraries that can be replaced
- **Lazy load (💡)** = Heavy components used on specific routes
- **Your code** = Usually not the problem (20-30% of bundle)

**Key insight:** Your code is rarely the problem. It's the dependencies you added without checking size.

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

---

## 🧠 Quick Recall

Before learning about code splitting, test your retention:

**Question:** In the bundle map above, what percentage of the bundle came from dependencies (node_modules) vs your own code?

<details>
<summary>✅ Answer</summary>

**node_modules: 64% (1.6MB) | Your code: 20% (500KB)**

This is typical! Most React apps have:
- 60-80% dependencies
- 20-30% your code
- 10-20% vendor/webpack runtime

**The lesson:** Optimizing your own code rarely has big impact. The wins are:
1. Replacing heavy libraries (moment → date-fns saves 378KB)
2. Lazy loading big dependencies (recharts, PDF viewers)
3. Tree-shaking (import specific functions, not whole libraries)

Don't spend hours optimizing 500KB of your code when you have 1.6MB of dependencies to tackle first.

</details>

---

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

---

## 💥 War Story: The 18MB Bundle That Killed Mobile Signups

### The Disaster

**Company:** SaaS startup (250K users, Series B funding)
**Team:** 8 frontend engineers
**Date:** Product launch week, Q4 2023
**Impact:** 73% mobile signup drop, $890K in lost MRR projections, emergency "code freeze"

### What Happened

The startup was launching their mobile-optimized web app to capture the growing mobile market. Marketing had spent $500K on ads driving mobile traffic. Launch day: 12,000 mobile visitors, only 340 signups (2.8% conversion, expected 12%).

**The investigation revealed:**

```
Mobile users (iPhone 12, 4G connection):
- Page load: 47 seconds  (!!!)
- 94% bounced before page loaded
- Only 6% saw the signup form

Desktop users (same day, for comparison):
- Page load: 4.2 seconds
- 32% signup conversion (normal)
```

**The bundle analysis:**

```javascript
// What the bundle analyzer showed:
main.bundle.js: 18.2 MB (!!!)
  ├─ node_modules/: 16.8 MB (92%)
  │   ├─ moment.js: 525 KB
  │   ├─ moment-timezone + all locales: 1.2 MB
  │   ├─ lodash (entire library): 530 KB
  │   ├─ chart.js + dependencies: 3.8 MB
  │   ├─ pdf-lib + pdfjs-dist: 4.2 MB
  │   ├─ @monaco-editor/react: 2.9 MB  // Code editor!
  │   ├─ THREE.js: 1.8 MB  // 3D graphics!
  │   ├─ antd (entire UI library): 1.2 MB
  │   └─ other dependencies: 0.7 MB
  └─ src/: 1.4 MB (8%)

// Mobile download time:
// 18.2 MB ÷ 1.5 Mbps (4G) = 97 seconds
// Gzipped: 4.8 MB ÷ 1.5 Mbps = 25 seconds
// Parse + execute: +22 seconds
// Total: 47 seconds

The app was loading:
- A PDF viewer (only used on /documents page)
- A code editor (only used by devs in /admin)
- 3D graphics library (removed feature, code never deleted)
- ALL 150+ moment-timezone locales (only needed en-US)
- Full antd library (used 8 components)
```

### The Root Cause

**How did this happen?**

A junior developer was tasked with "add a code editor to the admin panel" (used by 5 internal users). They ran:

```bash
npm install @monaco-editor/react
```

**The code:**
```javascript
// AdminPanel.tsx
import Editor from '@monaco-editor/react';  // 2.9 MB added to main bundle!

function AdminPanel() {
  return (
    <div>
      {/* Editor loaded for ALL users, even though only 5 people access /admin */}
      <Editor language="javascript" value={code} />
    </div>
  );
}
```

**No code review caught it because:**
- PR description: "Add code editor to admin panel"
- Changed files: 1 file, +15 lines
- Reviewer thought: "Looks fine, small change"
- **No bundle size check in CI/CD**
- **No performance testing**

**The bundle grew from 2.1 MB → 18.2 MB** and nobody noticed until launch day.

### The Numbers

**Business Impact:**

```
Expected mobile conversions: 12,000 visitors × 12% = 1,440 signups
Actual mobile conversions: 12,000 visitors × 2.8% = 340 signups
Lost signups: 1,100

Average MRR per customer: $49/month
Lost MRR: 1,100 × $49 = $53,900/month
Annual impact: $646,800
```

**Marketing waste:**
- $500K spent on mobile ads
- Only 340 conversions = $1,470 per signup (should be $120)
- **$500K mostly wasted**

**Team Impact:**
- All-hands emergency meeting at 2 AM
- 3-day "code freeze" (no new features)
- CEO personally apologizing to investors
- CTO writing post-mortem for board

### The Fix (Emergency 48-Hour Sprint)

**Step 1: Remove the obvious bloat (Hour 1-4)**

```javascript
// BEFORE: 18.2 MB bundle
import moment from 'moment';
import 'moment-timezone';  // All 150 locales!
import _ from 'lodash';  // Entire library
import Editor from '@monaco-editor/react';
import * as THREE from 'three';  // Not even used anymore!

// AFTER: Basic cleanup
import { format } from 'date-fns';  // 2 KB vs 1.7 MB
import debounce from 'lodash/debounce';  // 2 KB vs 530 KB

// Removed THREE.js entirely (dead code)
// Removed Editor (moved to lazy load - see below)

Savings: 5.2 MB → Down to 13 MB
```

**Step 2: Implement route-based code splitting (Hour 5-12)**

```javascript
// BEFORE: Everything loaded upfront
import Dashboard from './pages/Dashboard';
import Documents from './pages/Documents';  // Has PDF viewer
import AdminPanel from './pages/AdminPanel';  // Has code editor
import Analytics from './pages/Analytics';  // Has charts

// AFTER: Route-based splitting
import { lazy, Suspense } from 'react';

const Dashboard = lazy(() => import('./pages/Dashboard'));
const Documents = lazy(() => import('./pages/Documents'));  // PDF viewer only loads here
const AdminPanel = lazy(() => import('./pages/AdminPanel'));  // Editor only loads here
const Analytics = lazy(() => import('./pages/Analytics'));  // Charts only loads here

function App() {
  return (
    <Suspense fallback={<PageLoader />}>
      <Routes>
        <Route path="/" element={<Dashboard />} />
        <Route path="/documents" element={<Documents />} />
        <Route path="/admin" element={<AdminPanel />} />
        <Route path="/analytics" element={<Analytics />} />
      </Routes>
    </Suspense>
  );
}

// Result:
// Initial bundle: 13 MB → 380 KB (just the dashboard code)
// Documents page: +4.2 MB (PDF viewer loads when needed)
// Admin page: +2.9 MB (Editor loads when needed)
// Analytics page: +3.8 MB (Charts load when needed)
```

**Step 3: Tree-shake antd (Hour 13-18)**

```javascript
// BEFORE: Entire antd library (1.2 MB)
import { Button, Modal, Input, Select, DatePicker, Table, Form, Card } from 'antd';
import 'antd/dist/antd.css';

// AFTER: Import only what we use
import Button from 'antd/es/button';
import Modal from 'antd/es/modal';
import Input from 'antd/es/input';
import 'antd/es/button/style/css';
import 'antd/es/modal/style/css';
import 'antd/es/input/style/css';

Savings: 1.2 MB → 180 KB
```

**Final Results (Hour 48):**

```
BEFORE (launch day disaster):
- Main bundle: 18.2 MB
- Mobile load time: 47 seconds
- Mobile conversion: 2.8%

AFTER (emergency fix):
- Initial bundle: 280 KB  (-98.5%!)
- Additional chunks loaded on-demand
- Mobile load time: 1.8 seconds (-96%)
- Mobile conversion: 11.2% (back to normal)
```

**Performance Metrics:**

| Metric | Before | After | Improvement |
|--------|--------|-------|-------------|
| Bundle size | 18.2 MB | 280 KB | **98.5% smaller** |
| FCP | 38s | 0.9s | **42× faster** |
| LCP | 45s | 1.6s | **28× faster** |
| TTI | 89s | 2.3s | **39× faster** |
| Mobile bounce rate | 94% | 14% | **85% reduction** |
| Mobile conversion | 2.8% | 11.2% | **4× improvement** |

### Long-Term Preventions

**1. Bundle Size Budget in CI/CD (now blocks PRs):**

```javascript
// budgets.json
{
  "budgets": [
    {
      "path": "dist/main.*.js",
      "maxSize": "300kb",  // PR fails if exceeded
      "baseline": "250kb"
    },
    {
      "path": "dist/*.chunk.js",
      "maxSize": "500kb"  // Individual chunks
    }
  ]
}

// .github/workflows/bundle-check.yml
- name: Bundle Size Check
  run: npm run build && npx bundlesize
  # Fails PR if bundle exceeds budget
```

**2. Dependency Size Check (runs on npm install):**

```javascript
// .npmrc
package-lock=true

// package.json
"scripts": {
  "postinstall": "npx cost-of-modules --less 500"
}

// Now when someone runs: npm install @monaco-editor/react
// Output:
//   ⚠️  @monaco-editor/react: 2.9 MB
//   ⚠️  This exceeds the 500 KB warning threshold
//   ⚠️  Consider alternatives or lazy-load this dependency
```

**3. Mandatory PR Template Update:**

```markdown
## Bundle Impact Checklist
- [ ] Ran `npm run build` and checked bundle size
- [ ] If adding dependency > 50KB, justify why and show lazy-loading plan
- [ ] Attached screenshot of bundle analyzer diff
- [ ] Tested on mobile 4G connection (throttled)

**Bundle size change:** +XX KB / -XX KB
```

**4. Weekly Bundle Reports:**

```javascript
// Automated Slack message every Monday:
📦 Bundle Size Report - Week of 12/18

Current: 285 KB
Last week: 280 KB (+5 KB, +1.8%)

Largest chunks:
  1. dashboard.chunk.js: 120 KB
  2. analytics.chunk.js: 89 KB
  3. documents.chunk.js: 76 KB

Recent additions:
  ⚠️  lodash-es (+15 KB) - Consider tree-shaking

🎯 Staying under 300 KB budget ✅
```

### Lessons Learned

1. **One unreviewed dependency can kill your app**
   - 2.9 MB editor added in a "small PR"
   - No bundle size checking = disaster
   - **Always check bundle impact**

2. **Mobile users are different**
   - 18 MB was "annoying" on desktop (4s)
   - 18 MB was "unusable" on mobile (47s)
   - **Test on real mobile connections (4G, not WiFi)**

3. **Dead code accumulates**
   - THREE.js (1.8 MB) from removed feature still in bundle
   - No automated dead code detection
   - **Regular bundle audits are essential**

4. **Lazy loading is not premature optimization**
   - "We'll optimize later" turned into $500K waste
   - Route splitting takes 30 minutes to implement
   - **Do it from day one**

5. **Code reviews don't catch bundle bloat**
   - "+15 lines" looked innocent
   - Bundle grew 8× and nobody noticed
   - **Automated checks are mandatory**

6. **Marketing spend is wasted if app doesn't load**
   - $500K on ads, 94% bounced before seeing content
   - No amount of marketing fixes a 47-second load time
   - **Performance is a business metric, not just engineering**

### The Cost

**Direct costs:**
- Lost MRR: $646K annually
- Wasted marketing: $500K
- Engineering time (all hands, 48 hours): $80K
- **Total: $1.23 million**

**Indirect costs:**
- Investor confidence hit (explained in board meeting)
- Delayed feature roadmap (3-day code freeze)
- Team morale (working through weekend emergency)
- Reputation damage (users shared load time screenshots on Twitter)

**Could have been prevented by:**
- Bundle size check in CI: **Free** (bundlesize npm package)
- Code review asking "what's the bundle impact?": **Free**
- Testing on mobile before launch: 2 hours = **$400**

**ROI of prevention: 3,075× cheaper than fixing in production**

### The Happy Ending

After the fix, the startup:
- Recovered mobile conversion to 11.2%
- Hit MRR targets within 2 weeks (extended campaign)
- Made bundle size checks mandatory (prevented 8 future incidents in next 6 months)
- Open sourced their bundle monitoring setup (got HackerNews front page)
- CTO wrote a viral blog post "How an 18MB bundle almost killed our startup" (200K reads)

The disaster became a learning opportunity. But it didn't have to happen.

---

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

---

## 🚫 Common Mistakes Gallery

### Mistake 1: Lazy Loading Everything
Junior devs over-apply lazy loading to tiny components, adding overhead for no gain. Only lazy load heavy components (> 50KB) or route-level components.

### Mistake 2: No Loading State
Using `<Suspense fallback={<div>Loading...</div>}>` creates jarring UX. Use skeleton screens that match the loaded content's layout.

### Mistake 3: Importing Entire Libraries
`import _ from 'lodash'` (70KB) vs `import debounce from 'lodash/debounce'` (2KB). Always import specific functions for tree-shaking.

### Mistake 4: No Bundle Size Monitoring
Bundle grows from 500KB → 3MB over 6 months without anyone noticing. Set up automated bundle size checks in CI/CD.

### Mistake 5: Loading Heavy Libraries Upfront
PDF viewers, code editors, and chart libraries should be lazy loaded - they're only used on specific pages.

### Mistake 6: Keeping Dead Code
Removed features leave behind dependencies (THREE.js, unused UI libraries). Audit and remove unused dependencies regularly.

---

## 📝 Cumulative Review

### Q1: Why did the 18MB bundle only affect mobile users severely?
<details><summary>Answer</summary>
Mobile connections (4G = 1.5 Mbps) are 20-40× slower than home WiFi. 18MB takes 47s on 4G vs 4s on WiFi. Plus mobile devices have less memory/CPU, so parsing/executing JS takes longer.
</details>

### Q2: What's the difference between `React.lazy()` and regular imports?
<details><summary>Answer</summary>
Regular: `import Foo from './Foo'` loads immediately (part of main bundle).
Lazy: `const Foo = lazy(() => import('./Foo'))` creates a separate chunk that loads only when component renders. Use for routes and heavy components.
</details>

### Q3: When should you NOT lazy load a component?
<details><summary>Answer</summary>
Don't lazy load if: component is small (< 20KB), loads on every page, or critical for initial render. Lazy loading adds overhead (network request, Suspense boundary). Only worth it for heavy/conditional components.
</details>

### Q4: How do you prevent bundle size from creeping up over time?
<details><summary>Answer</summary>
1. Bundle size budget in CI (PR fails if exceeded)
2. Weekly bundle reports
3. Dependency size checks on npm install
4. Regular audits with webpack-bundle-analyzer
5. Code review checklist requiring bundle impact assessment
</details>

### Q5: What's tree-shaking and why does it matter?
<details><summary>Answer</summary>
Tree-shaking removes unused code from bundles. Matters because `import {Button} from 'ui-lib'` can pull in the whole library (1.2MB) if not tree-shakeable. Solution: import specific paths (`import Button from 'ui-lib/Button'`) or use libraries marked `"sideEffects": false`.
</details>

### Q6: You add route-based code splitting but initial load is still slow. What else should you check?
<details><summary>Answer</summary>
Check for:
1. Heavy libraries still in main bundle (moment, lodash, charts)
2. CSS bloat (entire UI library styles loaded)
3. Polyfills loading for all browsers (use differential serving)
4. Large images not optimized
5. No gzip/brotli compression
Analyze bundle first, then fix biggest chunks.
</details>

---

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
