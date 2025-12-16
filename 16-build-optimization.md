# Chapter 16: Build Optimization

## Introduction

Junior developers accept default build configurations and wonder why their apps are slow. Senior developers understand the build pipeline intimately, optimizing bundle sizes, leveraging caching, and ensuring fast load times for users worldwide.

## Learning Objectives

- Analyze and optimize bundle sizes
- Configure webpack/Vite effectively
- Implement tree shaking and dead code elimination
- Optimize assets (images, fonts, CSS)
- Leverage browser caching strategies
- Implement compression and minification
- Use bundle analyzers to identify bottlenecks
- Configure production builds properly

## 16.1 Bundle Analysis

### Using Webpack Bundle Analyzer

```bash
# Install bundle analyzer
npm install --save-dev webpack-bundle-analyzer

# webpack.config.js
const BundleAnalyzerPlugin = require('webpack-bundle-analyzer').BundleAnalyzerPlugin;

module.exports = {
  plugins: [
    new BundleAnalyzerPlugin({
      analyzerMode: 'static',
      openAnalyzer: false,
      reportFilename: 'bundle-report.html'
    })
  ]
};
```

### Using Vite Bundle Analyzer

```typescript
// vite.config.ts
import { defineConfig } from 'vite';
import { visualizer } from 'rollup-plugin-visualizer';

export default defineConfig({
  plugins: [
    visualizer({
      open: true,
      gzipSize: true,
      brotliSize: true,
      filename: 'bundle-analysis.html'
    })
  ]
});
```

### Analyzing Bundle Output

```typescript
// package.json
{
  "scripts": {
    "build": "vite build",
    "build:analyze": "vite build && rollup-plugin-visualizer",
    "analyze": "source-map-explorer 'dist/assets/*.js'"
  }
}
```

### Reading Bundle Reports

```typescript
// Example issues found in bundle analysis:

// 1. Large third-party libraries
// ❌ Bad: moment.js (289 KB)
import moment from 'moment';

// ✅ Good: date-fns (13 KB with tree shaking)
import { format, parseISO } from 'date-fns';

// 2. Duplicate dependencies
// ❌ Bad: Multiple versions of React
// Check package-lock.json for duplicates

// 3. Unused code
// ❌ Bad: Importing entire library
import _ from 'lodash';

// ✅ Good: Import only what you need
import debounce from 'lodash/debounce';
import throttle from 'lodash/throttle';
```

## 16.2 Tree Shaking

### Enabling Tree Shaking

```typescript
// vite.config.ts
export default defineConfig({
  build: {
    rollupOptions: {
      output: {
        manualChunks: undefined
      }
    },
    // Enable tree shaking
    minify: 'terser',
    terserOptions: {
      compress: {
        dead_code: true,
        drop_console: true,
        drop_debugger: true,
        pure_funcs: ['console.log', 'console.info']
      }
    }
  }
});
```

### Writing Tree-Shakeable Code

```typescript
// ❌ Bad: Side effects prevent tree shaking
// utils.ts
export const unused = () => {
  console.log('This has side effects');
};

export const used = () => 'used';

// ✅ Good: Pure functions are tree-shakeable
// utils.ts
export const unused = () => 'unused'; // Will be removed if not imported

export const used = () => 'used';

// package.json - Mark package as side-effect free
{
  "sideEffects": false
}

// Or specify files with side effects
{
  "sideEffects": [
    "*.css",
    "*.scss",
    "./src/polyfills.ts"
  ]
}
```

### Tree Shaking CSS

```typescript
// Install PurgeCSS
npm install --save-dev @fullhuman/postcss-purgecss

// postcss.config.js
module.exports = {
  plugins: [
    require('@fullhuman/postcss-purgecss')({
      content: [
        './src/**/*.html',
        './src/**/*.tsx',
        './src/**/*.ts'
      ],
      defaultExtractor: content => content.match(/[\w-/:]+(?<!:)/g) || [],
      safelist: [
        /^data-/,
        /^aria-/,
        'html',
        'body'
      ]
    })
  ]
};
```

## 16.3 Code Splitting Strategies

### Route-Based Code Splitting

```typescript
// App.tsx
import { lazy, Suspense } from 'react';
import { BrowserRouter, Routes, Route } from 'react-router-dom';

// Lazy load route components
const Home = lazy(() => import('./pages/Home'));
const Dashboard = lazy(() => import('./pages/Dashboard'));
const Profile = lazy(() => import('./pages/Profile'));
const Settings = lazy(() => import('./pages/Settings'));

function App() {
  return (
    <BrowserRouter>
      <Suspense fallback={<PageLoader />}>
        <Routes>
          <Route path="/" element={<Home />} />
          <Route path="/dashboard" element={<Dashboard />} />
          <Route path="/profile" element={<Profile />} />
          <Route path="/settings" element={<Settings />} />
        </Routes>
      </Suspense>
    </BrowserRouter>
  );
}
```

### Component-Based Code Splitting

```typescript
// Heavy component loaded only when needed
const HeavyChart = lazy(() => import('./components/HeavyChart'));
const VideoPlayer = lazy(() => import('./components/VideoPlayer'));
const RichTextEditor = lazy(() => import('./components/RichTextEditor'));

function Dashboard() {
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

### Vendor Chunk Splitting

```typescript
// vite.config.ts
export default defineConfig({
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          // React ecosystem
          'react-vendor': ['react', 'react-dom', 'react-router-dom'],

          // UI libraries
          'ui-vendor': ['@mui/material', '@emotion/react', '@emotion/styled'],

          // Data fetching
          'data-vendor': ['@tanstack/react-query', 'axios'],

          // Utilities
          'utils-vendor': ['lodash-es', 'date-fns', 'zod']
        }
      }
    },
    chunkSizeWarningLimit: 1000
  }
});

// webpack.config.js (for webpack users)
module.exports = {
  optimization: {
    splitChunks: {
      chunks: 'all',
      cacheGroups: {
        react: {
          test: /[\\/]node_modules[\\/](react|react-dom|react-router-dom)[\\/]/,
          name: 'react-vendor',
          priority: 10
        },
        vendors: {
          test: /[\\/]node_modules[\\/]/,
          name: 'vendors',
          priority: 5
        },
        common: {
          minChunks: 2,
          priority: 1,
          reuseExistingChunk: true
        }
      }
    }
  }
};
```

## 16.4 Asset Optimization

### Image Optimization

```typescript
// vite.config.ts
import viteImagemin from 'vite-plugin-imagemin';

export default defineConfig({
  plugins: [
    viteImagemin({
      gifsicle: {
        optimizationLevel: 7,
        interlaced: false
      },
      optipng: {
        optimizationLevel: 7
      },
      mozjpeg: {
        quality: 80
      },
      pngquant: {
        quality: [0.8, 0.9],
        speed: 4
      },
      svgo: {
        plugins: [
          {
            name: 'removeViewBox',
            active: false
          },
          {
            name: 'removeEmptyAttrs',
            active: true
          }
        ]
      }
    })
  ]
});
```

### Using Modern Image Formats

```typescript
// ImageComponent.tsx
interface ImageProps {
  src: string;
  alt: string;
  width?: number;
  height?: number;
}

function OptimizedImage({ src, alt, width, height }: ImageProps) {
  // Generate WebP and AVIF versions
  const webpSrc = src.replace(/\.(jpg|png)$/, '.webp');
  const avifSrc = src.replace(/\.(jpg|png)$/, '.avif');

  return (
    <picture>
      <source srcSet={avifSrc} type="image/avif" />
      <source srcSet={webpSrc} type="image/webp" />
      <img
        src={src}
        alt={alt}
        width={width}
        height={height}
        loading="lazy"
        decoding="async"
      />
    </picture>
  );
}
```

### Font Optimization

```css
/* fonts.css */

/* Preload critical fonts */
/* Add to index.html:
<link rel="preload" href="/fonts/inter.woff2" as="font" type="font/woff2" crossorigin>
*/

@font-face {
  font-family: 'Inter';
  font-style: normal;
  font-weight: 400;
  font-display: swap; /* Prevent invisible text flash */
  src: url('/fonts/inter.woff2') format('woff2'),
       url('/fonts/inter.woff') format('woff');
  /* Subset fonts for performance */
  unicode-range: U+0000-00FF, U+0131, U+0152-0153;
}

/* Load additional weights asynchronously */
@font-face {
  font-family: 'Inter';
  font-style: normal;
  font-weight: 700;
  font-display: swap;
  src: url('/fonts/inter-bold.woff2') format('woff2');
}
```

### SVG Optimization

```typescript
// Install SVGR for React
npm install --save-dev @svgr/rollup

// vite.config.ts
import svgr from '@svgr/rollup';

export default defineConfig({
  plugins: [
    svgr({
      svgoConfig: {
        plugins: [
          {
            name: 'removeViewBox',
            active: false
          },
          {
            name: 'cleanupIDs',
            active: true
          }
        ]
      }
    })
  ]
});

// Usage - import SVG as React component
import { ReactComponent as Logo } from './logo.svg';

function Header() {
  return <Logo className="logo" />;
}
```

## 16.5 Compression and Minification

### Gzip and Brotli Compression

```typescript
// vite.config.ts
import viteCompression from 'vite-plugin-compression';

export default defineConfig({
  plugins: [
    // Gzip compression
    viteCompression({
      algorithm: 'gzip',
      ext: '.gz',
      threshold: 10240, // Only compress files > 10KB
      deleteOriginFile: false
    }),

    // Brotli compression (better than gzip)
    viteCompression({
      algorithm: 'brotliCompress',
      ext: '.br',
      threshold: 10240,
      deleteOriginFile: false
    })
  ]
});
```

### Server Configuration for Compression

```nginx
# nginx.conf
http {
  # Enable Gzip
  gzip on;
  gzip_vary on;
  gzip_min_length 1024;
  gzip_types
    text/plain
    text/css
    text/javascript
    application/javascript
    application/json
    application/xml
    image/svg+xml;

  # Enable Brotli (requires ngx_brotli module)
  brotli on;
  brotli_types
    text/plain
    text/css
    text/javascript
    application/javascript
    application/json
    application/xml
    image/svg+xml;
  brotli_comp_level 6;
}
```

### CSS Minification

```typescript
// vite.config.ts
export default defineConfig({
  css: {
    preprocessorOptions: {
      scss: {
        additionalData: `@import "@/styles/variables.scss";`
      }
    },
    modules: {
      localsConvention: 'camelCase',
      generateScopedName: '[hash:base64:5]' // Short class names in production
    }
  },
  build: {
    cssCodeSplit: true, // Split CSS into chunks
    cssMinify: 'lightningcss' // Fast CSS minifier
  }
});
```

## 16.6 Caching Strategies

### Long-Term Caching with Content Hashing

```typescript
// vite.config.ts
export default defineConfig({
  build: {
    rollupOptions: {
      output: {
        // Content-based hashing for cache busting
        entryFileNames: 'assets/[name].[hash].js',
        chunkFileNames: 'assets/[name].[hash].js',
        assetFileNames: 'assets/[name].[hash].[ext]'
      }
    }
  }
});
```

### Service Worker Caching

```typescript
// sw.ts (Service Worker)
const CACHE_NAME = 'app-v1';
const STATIC_ASSETS = [
  '/',
  '/index.html',
  '/assets/main.js',
  '/assets/styles.css'
];

// Install event - cache static assets
self.addEventListener('install', (event: ExtendableEvent) => {
  event.waitUntil(
    caches.open(CACHE_NAME)
      .then(cache => cache.addAll(STATIC_ASSETS))
  );
});

// Fetch event - serve from cache, fallback to network
self.addEventListener('fetch', (event: FetchEvent) => {
  event.respondWith(
    caches.match(event.request)
      .then(response => {
        if (response) {
          return response;
        }

        return fetch(event.request).then(response => {
          // Cache successful responses
          if (response.status === 200) {
            const responseClone = response.clone();
            caches.open(CACHE_NAME)
              .then(cache => cache.put(event.request, responseClone));
          }
          return response;
        });
      })
  );
});

// Activate event - clean old caches
self.addEventListener('activate', (event: ExtendableEvent) => {
  event.waitUntil(
    caches.keys().then(cacheNames => {
      return Promise.all(
        cacheNames
          .filter(name => name !== CACHE_NAME)
          .map(name => caches.delete(name))
      );
    })
  );
});
```

### HTTP Cache Headers

```typescript
// express server example
import express from 'express';
import path from 'path';

const app = express();

// Serve static files with cache headers
app.use('/assets', express.static(
  path.join(__dirname, 'dist/assets'),
  {
    maxAge: '1y', // Cache for 1 year (assets have content hash)
    immutable: true
  }
));

// Serve HTML with no-cache (always revalidate)
app.get('*', (req, res) => {
  res.set({
    'Cache-Control': 'no-cache, no-store, must-revalidate',
    'Pragma': 'no-cache',
    'Expires': '0'
  });
  res.sendFile(path.join(__dirname, 'dist/index.html'));
});
```

## 16.7 Module Federation (Micro-Frontends)

### Setting Up Module Federation

```typescript
// vite.config.ts (Host App)
import federation from '@originjs/vite-plugin-federation';

export default defineConfig({
  plugins: [
    federation({
      name: 'host-app',
      remotes: {
        remoteApp: 'http://localhost:3001/assets/remoteEntry.js'
      },
      shared: ['react', 'react-dom', 'react-router-dom']
    })
  ]
});

// vite.config.ts (Remote App)
export default defineConfig({
  plugins: [
    federation({
      name: 'remote-app',
      filename: 'remoteEntry.js',
      exposes: {
        './Button': './src/components/Button',
        './Dashboard': './src/pages/Dashboard'
      },
      shared: ['react', 'react-dom']
    })
  ],
  build: {
    target: 'esnext'
  }
});
```

### Using Remote Components

```typescript
// App.tsx (Host)
import { lazy, Suspense } from 'react';

// Import remote component
const RemoteDashboard = lazy(() => import('remoteApp/Dashboard'));

function App() {
  return (
    <Suspense fallback={<Loading />}>
      <RemoteDashboard />
    </Suspense>
  );
}
```

## 16.8 Build Performance

### Speeding Up Development Builds

```typescript
// vite.config.ts
export default defineConfig({
  server: {
    warmup: {
      // Pre-transform commonly used files
      clientFiles: [
        './src/components/**/*.tsx',
        './src/pages/**/*.tsx'
      ]
    }
  },
  optimizeDeps: {
    include: [
      'react',
      'react-dom',
      'react-router-dom',
      '@tanstack/react-query'
    ],
    // Force re-optimization
    force: false
  }
});
```

### Parallel Builds

```typescript
// package.json
{
  "scripts": {
    "build": "npm-run-all --parallel build:*",
    "build:client": "vite build",
    "build:server": "tsc -p tsconfig.server.json",
    "build:types": "tsc -p tsconfig.types.json --emitDeclarationOnly"
  }
}
```

### Using esbuild for Faster Builds

```typescript
// vite.config.ts
export default defineConfig({
  build: {
    minify: 'esbuild', // Faster than terser
    target: 'esnext'
  },
  esbuild: {
    logOverride: { 'this-is-undefined-in-esm': 'silent' }
  }
});
```

## 16.9 Environment-Specific Builds

### Production vs Development

```typescript
// vite.config.ts
import { defineConfig } from 'vite';

export default defineConfig(({ mode }) => ({
  define: {
    __DEV__: mode === 'development',
    __PROD__: mode === 'production'
  },

  build: {
    sourcemap: mode === 'development',
    minify: mode === 'production' ? 'esbuild' : false,

    rollupOptions: {
      plugins: mode === 'production' ? [
        // Production-only plugins
      ] : []
    }
  }
}));
```

### Feature Flags

```typescript
// config/featureFlags.ts
export const featureFlags = {
  enableNewDashboard: import.meta.env.VITE_ENABLE_NEW_DASHBOARD === 'true',
  enableBetaFeatures: import.meta.env.VITE_ENABLE_BETA === 'true',
  apiEndpoint: import.meta.env.VITE_API_ENDPOINT
};

// Usage
import { featureFlags } from './config/featureFlags';

function App() {
  return (
    <>
      {featureFlags.enableNewDashboard ? (
        <NewDashboard />
      ) : (
        <OldDashboard />
      )}
    </>
  );
}

// .env.production
VITE_ENABLE_NEW_DASHBOARD=true
VITE_API_ENDPOINT=https://api.production.com

// .env.development
VITE_ENABLE_NEW_DASHBOARD=false
VITE_API_ENDPOINT=http://localhost:3000
```

## Real-World Scenario: Optimizing a Slow Build

### The Challenge

Your app has:
- 5MB bundle size
- 45-second build time
- Slow page loads
- Large node_modules

### Junior Approach

```typescript
// Just accept slow builds
"Builds are slow, that's normal"
```

### Senior Approach

```typescript
// 1. Analyze the bundle
npm run build:analyze

// Findings:
// - moment.js: 289KB (switch to date-fns)
// - lodash: 500KB (use individual imports)
// - Duplicate React versions
// - No code splitting

// 2. Replace heavy dependencies
// Before
import moment from 'moment';
import _ from 'lodash';

// After
import { format } from 'date-fns';
import debounce from 'lodash-es/debounce';

// 3. Implement code splitting
const Dashboard = lazy(() => import('./pages/Dashboard'));
const Reports = lazy(() => import('./pages/Reports'));

// 4. Configure chunk splitting
// vite.config.ts
export default defineConfig({
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          'react-vendor': ['react', 'react-dom'],
          'ui-vendor': ['@mui/material'],
          'utils': ['date-fns', 'zod']
        }
      }
    }
  }
});

// 5. Enable compression
plugins: [
  viteCompression({ algorithm: 'brotliCompress' })
]

// Results:
// - Bundle: 5MB → 800KB (84% reduction)
// - Build time: 45s → 8s (82% faster)
// - Load time: 6s → 1.2s (80% faster)
```

### Optimization Checklist

```typescript
// package.json - Add bundle size checks
{
  "scripts": {
    "build": "vite build",
    "build:check": "npm run build && bundlesize"
  },
  "bundlesize": [
    {
      "path": "./dist/assets/*.js",
      "maxSize": "200 kB"
    },
    {
      "path": "./dist/assets/*.css",
      "maxSize": "50 kB"
    }
  ]
}
```

## Chapter Exercise: Optimize Your Build

Optimize a React application's build:

**Requirements:**
1. Run bundle analyzer and identify issues
2. Reduce bundle size by at least 40%
3. Implement code splitting for routes
4. Set up vendor chunk splitting
5. Enable compression (Gzip + Brotli)
6. Configure caching headers
7. Optimize images and fonts
8. Set up bundle size monitoring

**Target Metrics:**
- Initial bundle < 200KB
- Total assets < 1MB
- Build time < 30s
- Lighthouse score > 90

## Review Checklist

- [ ] Bundle analyzer configured
- [ ] Tree shaking enabled
- [ ] Code splitting implemented
- [ ] Vendor chunks separated
- [ ] Assets optimized (images, fonts)
- [ ] Compression enabled
- [ ] Caching strategy configured
- [ ] Bundle size monitoring in place
- [ ] Source maps in development only
- [ ] Environment variables configured

## Key Takeaways

1. **Measure first, optimize second** - Use bundle analyzers
2. **Dependencies matter** - Choose lightweight alternatives
3. **Code splitting is essential** - Don't ship everything upfront
4. **Compression is free performance** - Enable Brotli + Gzip
5. **Caching prevents re-downloads** - Use content hashing
6. **Assets are often the biggest culprit** - Optimize images
7. **Monitor bundle size** - Prevent regressions

## Further Reading

- Webpack documentation
- Vite optimization guide
- Web.dev: Fast load times
- Bundle size tools comparison
- Module Federation architecture

## Next Chapter

[Chapter 17: CI/CD for React Apps](./17-cicd-react-apps.md)
