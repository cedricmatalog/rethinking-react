# Chapter 18: Monitoring & Observability

## Introduction

Junior developers deploy code and hope it works. Senior developers instrument their applications comprehensively, monitoring performance, tracking errors, and gaining deep insights into user behavior and system health.

## Learning Objectives

- Implement error tracking and reporting
- Set up performance monitoring
- Track real user metrics (RUM)
- Configure application performance monitoring (APM)
- Implement structured logging
- Create custom metrics and dashboards
- Set up alerts and notifications
- Debug production issues effectively

## 18.1 Error Tracking

### Setting Up Sentry

```typescript
// main.tsx
import * as Sentry from '@sentry/react';
import { BrowserTracing } from '@sentry/tracing';

Sentry.init({
  dsn: import.meta.env.VITE_SENTRY_DSN,
  environment: import.meta.env.VITE_APP_ENV,
  integrations: [
    new BrowserTracing(),
    new Sentry.Replay({
      maskAllText: false,
      blockAllMedia: false,
    }),
  ],

  // Performance monitoring
  tracesSampleRate: import.meta.env.PROD ? 0.1 : 1.0,

  // Session replay
  replaysSessionSampleRate: 0.1,
  replaysOnErrorSampleRate: 1.0,

  // Release tracking
  release: import.meta.env.VITE_APP_VERSION,

  // Ignore specific errors
  ignoreErrors: [
    'ResizeObserver loop limit exceeded',
    'Non-Error promise rejection captured',
  ],

  beforeSend(event, hint) {
    // Filter out non-critical errors
    if (event.level === 'warning') {
      return null;
    }

    // Add custom context
    event.contexts = {
      ...event.contexts,
      app: {
        version: import.meta.env.VITE_APP_VERSION,
        buildTime: import.meta.env.VITE_BUILD_TIME,
      },
    };

    return event;
  },
});

// Error boundary with Sentry
function App() {
  return (
    <Sentry.ErrorBoundary
      fallback={<ErrorFallback />}
      showDialog
      dialogOptions={{
        title: 'Something went wrong',
        subtitle: 'Our team has been notified.',
        subtitle2: 'You can help by telling us what happened.',
      }}
    >
      <Router />
    </Sentry.ErrorBoundary>
  );
}
```

### Custom Error Tracking

```typescript
// services/errorTracking.ts
import * as Sentry from '@sentry/react';

export class ErrorTracker {
  static captureError(error: Error, context?: Record<string, any>) {
    Sentry.captureException(error, {
      contexts: {
        custom: context,
      },
    });
  }

  static captureMessage(message: string, level: Sentry.SeverityLevel = 'info') {
    Sentry.captureMessage(message, level);
  }

  static setUser(user: { id: string; email: string; username: string }) {
    Sentry.setUser(user);
  }

  static addBreadcrumb(breadcrumb: {
    message: string;
    category: string;
    level?: Sentry.SeverityLevel;
    data?: Record<string, any>;
  }) {
    Sentry.addBreadcrumb(breadcrumb);
  }

  static setContext(name: string, context: Record<string, any>) {
    Sentry.setContext(name, context);
  }
}

// Usage in components
function CheckoutButton() {
  const handleCheckout = async () => {
    try {
      ErrorTracker.addBreadcrumb({
        message: 'Checkout initiated',
        category: 'user-action',
        level: 'info',
        data: { items: cart.items.length },
      });

      await processCheckout();
    } catch (error) {
      ErrorTracker.captureError(error as Error, {
        cart: cart.items,
        total: cart.total,
      });
    }
  };

  return <button onClick={handleCheckout}>Checkout</button>;
}
```

### Error Boundaries

```typescript
// components/ErrorBoundary.tsx
import { Component, ReactNode } from 'react';
import { ErrorTracker } from '@/services/errorTracking';

interface Props {
  children: ReactNode;
  fallback?: ReactNode;
}

interface State {
  hasError: boolean;
  error?: Error;
}

export class ErrorBoundary extends Component<Props, State> {
  constructor(props: Props) {
    super(props);
    this.state = { hasError: false };
  }

  static getDerivedStateFromError(error: Error): State {
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, errorInfo: React.ErrorInfo) {
    ErrorTracker.captureError(error, {
      componentStack: errorInfo.componentStack,
      errorBoundary: 'global',
    });
  }

  render() {
    if (this.state.hasError) {
      return this.props.fallback || (
        <div className="error-fallback">
          <h1>Something went wrong</h1>
          <p>We've been notified and are working on a fix.</p>
          <button onClick={() => this.setState({ hasError: false })}>
            Try again
          </button>
        </div>
      );
    }

    return this.props.children;
  }
}
```

## 18.2 Performance Monitoring

### Web Vitals Tracking

```typescript
// services/performanceMonitoring.ts
import { onCLS, onFID, onFCP, onLCP, onTTFB } from 'web-vitals';
import * as Sentry from '@sentry/react';

export function initPerformanceMonitoring() {
  // Cumulative Layout Shift
  onCLS((metric) => {
    Sentry.setMeasurement('CLS', metric.value, 'ratio');
    sendToAnalytics('CLS', metric.value);
  });

  // First Input Delay
  onFID((metric) => {
    Sentry.setMeasurement('FID', metric.value, 'millisecond');
    sendToAnalytics('FID', metric.value);
  });

  // First Contentful Paint
  onFCP((metric) => {
    Sentry.setMeasurement('FCP', metric.value, 'millisecond');
    sendToAnalytics('FCP', metric.value);
  });

  // Largest Contentful Paint
  onLCP((metric) => {
    Sentry.setMeasurement('LCP', metric.value, 'millisecond');
    sendToAnalytics('LCP', metric.value);
  });

  // Time to First Byte
  onTTFB((metric) => {
    Sentry.setMeasurement('TTFB', metric.value, 'millisecond');
    sendToAnalytics('TTFB', metric.value);
  });
}

function sendToAnalytics(name: string, value: number) {
  if (window.gtag) {
    window.gtag('event', name, {
      event_category: 'Web Vitals',
      value: Math.round(value),
      metric_value: value,
      non_interaction: true,
    });
  }
}
```

### Custom Performance Marks

```typescript
// hooks/usePerformanceTracking.ts
import { useEffect } from 'react';
import * as Sentry from '@sentry/react';

export function usePerformanceTracking(componentName: string) {
  useEffect(() => {
    const startMark = `${componentName}-start`;
    const endMark = `${componentName}-end`;
    const measureName = `${componentName}-render`;

    performance.mark(startMark);

    return () => {
      performance.mark(endMark);
      performance.measure(measureName, startMark, endMark);

      const measure = performance.getEntriesByName(measureName)[0];
      if (measure) {
        Sentry.setMeasurement(
          `component.${componentName}`,
          measure.duration,
          'millisecond'
        );

        // Warn if component is slow
        if (measure.duration > 100) {
          console.warn(`Slow component render: ${componentName} (${measure.duration}ms)`);
        }
      }

      performance.clearMarks(startMark);
      performance.clearMarks(endMark);
      performance.clearMeasures(measureName);
    };
  }, [componentName]);
}

// Usage
function HeavyComponent() {
  usePerformanceTracking('HeavyComponent');

  // Component logic...
  return <div>...</div>;
}
```

### React Profiler Integration

```typescript
// components/ProfiledComponent.tsx
import { Profiler, ProfilerOnRenderCallback } from 'react';
import * as Sentry from '@sentry/react';

const onRenderCallback: ProfilerOnRenderCallback = (
  id,
  phase,
  actualDuration,
  baseDuration,
  startTime,
  commitTime
) => {
  // Send metrics to monitoring service
  Sentry.setMeasurement(`profiler.${id}.${phase}`, actualDuration, 'millisecond');

  // Log slow renders
  if (actualDuration > 100) {
    console.warn(`Slow ${phase} in ${id}: ${actualDuration}ms`);

    Sentry.addBreadcrumb({
      category: 'performance',
      message: `Slow render: ${id}`,
      level: 'warning',
      data: {
        phase,
        actualDuration,
        baseDuration,
      },
    });
  }
};

export function ProfiledComponent({ children }: { children: React.ReactNode }) {
  return (
    <Profiler id="ProfiledComponent" onRender={onRenderCallback}>
      {children}
    </Profiler>
  );
}
```

## 18.3 Real User Monitoring (RUM)

### Google Analytics 4

```typescript
// services/analytics.ts
declare global {
  interface Window {
    gtag: (...args: any[]) => void;
    dataLayer: any[];
  }
}

export class Analytics {
  static init(measurementId: string) {
    // Load GA4 script
    const script = document.createElement('script');
    script.src = `https://www.googletagmanager.com/gtag/js?id=${measurementId}`;
    script.async = true;
    document.head.appendChild(script);

    window.dataLayer = window.dataLayer || [];
    window.gtag = function() {
      window.dataLayer.push(arguments);
    };

    window.gtag('js', new Date());
    window.gtag('config', measurementId, {
      send_page_view: false, // We'll send manually
    });
  }

  static pageView(path: string, title?: string) {
    window.gtag('event', 'page_view', {
      page_path: path,
      page_title: title || document.title,
    });
  }

  static event(name: string, params?: Record<string, any>) {
    window.gtag('event', name, params);
  }

  static setUser(userId: string, properties?: Record<string, any>) {
    window.gtag('set', 'user_properties', {
      user_id: userId,
      ...properties,
    });
  }

  static timing(category: string, variable: string, value: number) {
    window.gtag('event', 'timing_complete', {
      event_category: category,
      event_label: variable,
      value: Math.round(value),
    });
  }
}

// Usage with React Router
function AppRoutes() {
  const location = useLocation();

  useEffect(() => {
    Analytics.pageView(location.pathname);
  }, [location]);

  return <Routes>...</Routes>;
}
```

### Custom User Analytics

```typescript
// hooks/useUserTracking.ts
import { useEffect } from 'react';
import { Analytics } from '@/services/analytics';

export function useUserTracking() {
  useEffect(() => {
    // Track session start
    Analytics.event('session_start', {
      timestamp: Date.now(),
      userAgent: navigator.userAgent,
      screenResolution: `${window.screen.width}x${window.screen.height}`,
    });

    // Track visibility changes
    const handleVisibilityChange = () => {
      if (document.hidden) {
        Analytics.event('session_hidden');
      } else {
        Analytics.event('session_visible');
      }
    };

    document.addEventListener('visibilitychange', handleVisibilityChange);

    // Track before unload
    const handleBeforeUnload = () => {
      Analytics.event('session_end', {
        duration: performance.now(),
      });
    };

    window.addEventListener('beforeunload', handleBeforeUnload);

    return () => {
      document.removeEventListener('visibilitychange', handleVisibilityChange);
      window.removeEventListener('beforeunload', handleBeforeUnload);
    };
  }, []);
}
```

### Feature Usage Tracking

```typescript
// hooks/useFeatureTracking.ts
export function useFeatureTracking(featureName: string) {
  useEffect(() => {
    Analytics.event('feature_view', {
      feature: featureName,
      timestamp: Date.now(),
    });
  }, [featureName]);

  const trackAction = useCallback((action: string, metadata?: Record<string, any>) => {
    Analytics.event('feature_action', {
      feature: featureName,
      action,
      ...metadata,
    });
  }, [featureName]);

  return { trackAction };
}

// Usage
function SearchComponent() {
  const { trackAction } = useFeatureTracking('search');

  const handleSearch = (query: string) => {
    trackAction('search_submitted', {
      query,
      resultsCount: results.length,
    });
  };

  return <SearchBar onSearch={handleSearch} />;
}
```

## 18.4 Application Performance Monitoring (APM)

### Datadog RUM

```typescript
// services/datadog.ts
import { datadogRum } from '@datadog/browser-rum';

export function initDatadog() {
  datadogRum.init({
    applicationId: import.meta.env.VITE_DATADOG_APP_ID,
    clientToken: import.meta.env.VITE_DATADOG_CLIENT_TOKEN,
    site: 'datadoghq.com',
    service: 'my-react-app',
    env: import.meta.env.VITE_APP_ENV,
    version: import.meta.env.VITE_APP_VERSION,
    sessionSampleRate: 100,
    sessionReplaySampleRate: 20,
    trackUserInteractions: true,
    trackResources: true,
    trackLongTasks: true,
    defaultPrivacyLevel: 'mask-user-input',
  });

  datadogRum.startSessionReplayRecording();
}

// Custom actions
export function trackUserAction(name: string, context?: Record<string, any>) {
  datadogRum.addAction(name, context);
}

// Custom timings
export function trackTiming(name: string, duration: number) {
  datadogRum.addTiming(name, duration);
}
```

### New Relic Browser Agent

```typescript
// services/newrelic.ts
declare global {
  interface Window {
    newrelic: any;
  }
}

export class NewRelic {
  static setCustomAttribute(name: string, value: string | number) {
    if (window.newrelic) {
      window.newrelic.setCustomAttribute(name, value);
    }
  }

  static addPageAction(name: string, attributes?: Record<string, any>) {
    if (window.newrelic) {
      window.newrelic.addPageAction(name, attributes);
    }
  }

  static noticeError(error: Error, customAttributes?: Record<string, any>) {
    if (window.newrelic) {
      window.newrelic.noticeError(error, customAttributes);
    }
  }

  static setCurrentRouteName(name: string) {
    if (window.newrelic) {
      window.newrelic.setCurrentRouteName(name);
    }
  }
}

// Usage with React Router
function AppRoutes() {
  const location = useLocation();

  useEffect(() => {
    NewRelic.setCurrentRouteName(location.pathname);
  }, [location]);

  return <Routes>...</Routes>;
}
```

## 18.5 Structured Logging

### Console Wrapper with Levels

```typescript
// services/logger.ts
enum LogLevel {
  DEBUG = 0,
  INFO = 1,
  WARN = 2,
  ERROR = 3,
}

class Logger {
  private level: LogLevel;
  private service: string;

  constructor(service: string, level: LogLevel = LogLevel.INFO) {
    this.service = service;
    this.level = level;
  }

  private shouldLog(level: LogLevel): boolean {
    return level >= this.level;
  }

  private formatMessage(level: string, message: string, meta?: Record<string, any>) {
    return {
      timestamp: new Date().toISOString(),
      level,
      service: this.service,
      message,
      ...meta,
    };
  }

  debug(message: string, meta?: Record<string, any>) {
    if (this.shouldLog(LogLevel.DEBUG)) {
      console.debug(this.formatMessage('DEBUG', message, meta));
    }
  }

  info(message: string, meta?: Record<string, any>) {
    if (this.shouldLog(LogLevel.INFO)) {
      console.info(this.formatMessage('INFO', message, meta));
    }
  }

  warn(message: string, meta?: Record<string, any>) {
    if (this.shouldLog(LogLevel.WARN)) {
      console.warn(this.formatMessage('WARN', message, meta));
    }
  }

  error(message: string, error?: Error, meta?: Record<string, any>) {
    if (this.shouldLog(LogLevel.ERROR)) {
      console.error(this.formatMessage('ERROR', message, {
        error: error?.message,
        stack: error?.stack,
        ...meta,
      }));

      // Also send to error tracking
      if (error) {
        ErrorTracker.captureError(error, meta);
      }
    }
  }
}

// Create loggers for different parts of the app
export const apiLogger = new Logger('api');
export const authLogger = new Logger('auth');
export const routerLogger = new Logger('router');

// Usage
apiLogger.info('API request started', {
  method: 'GET',
  url: '/api/users',
});
```

### Integration with External Logging Services

```typescript
// services/cloudWatchLogger.ts
import { CloudWatchLogs } from '@aws-sdk/client-cloudwatch-logs';

export class CloudWatchLogger {
  private client: CloudWatchLogs;
  private logGroupName: string;
  private logStreamName: string;
  private sequenceToken?: string;

  constructor(logGroupName: string, logStreamName: string) {
    this.client = new CloudWatchLogs({ region: 'us-east-1' });
    this.logGroupName = logGroupName;
    this.logStreamName = logStreamName;
  }

  async log(message: string, level: string, metadata?: Record<string, any>) {
    const logEvent = {
      message: JSON.stringify({
        timestamp: new Date().toISOString(),
        level,
        message,
        ...metadata,
      }),
      timestamp: Date.now(),
    };

    try {
      const response = await this.client.putLogEvents({
        logGroupName: this.logGroupName,
        logStreamName: this.logStreamName,
        logEvents: [logEvent],
        sequenceToken: this.sequenceToken,
      });

      this.sequenceToken = response.nextSequenceToken;
    } catch (error) {
      console.error('Failed to send logs to CloudWatch:', error);
    }
  }
}
```

## 18.6 Custom Metrics and Dashboards

### Creating Custom Metrics

```typescript
// services/metrics.ts
export class MetricsCollector {
  private metrics: Map<string, number[]> = new Map();

  record(name: string, value: number) {
    if (!this.metrics.has(name)) {
      this.metrics.set(name, []);
    }
    this.metrics.get(name)!.push(value);
  }

  increment(name: string, amount: number = 1) {
    const current = this.metrics.get(name)?.[0] || 0;
    this.metrics.set(name, [current + amount]);
  }

  gauge(name: string, value: number) {
    this.metrics.set(name, [value]);
  }

  flush() {
    const summary: Record<string, any> = {};

    this.metrics.forEach((values, name) => {
      summary[name] = {
        count: values.length,
        sum: values.reduce((a, b) => a + b, 0),
        avg: values.reduce((a, b) => a + b, 0) / values.length,
        min: Math.min(...values),
        max: Math.max(...values),
      };
    });

    // Send to monitoring service
    this.sendMetrics(summary);

    // Clear metrics
    this.metrics.clear();

    return summary;
  }

  private async sendMetrics(metrics: Record<string, any>) {
    try {
      await fetch('/api/metrics', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(metrics),
      });
    } catch (error) {
      console.error('Failed to send metrics:', error);
    }
  }
}

export const metrics = new MetricsCollector();

// Flush metrics every 60 seconds
setInterval(() => {
  metrics.flush();
}, 60000);
```

### Business Metrics Tracking

```typescript
// hooks/useBusinessMetrics.ts
export function useBusinessMetrics() {
  const trackCheckout = useCallback((amount: number, items: number) => {
    metrics.record('checkout.amount', amount);
    metrics.record('checkout.items', items);
    metrics.increment('checkout.total');

    Analytics.event('checkout_completed', {
      value: amount,
      items,
      currency: 'USD',
    });
  }, []);

  const trackSignup = useCallback((method: string) => {
    metrics.increment('signup.total');
    metrics.increment(`signup.${method}`);

    Analytics.event('sign_up', {
      method,
    });
  }, []);

  const trackFeatureUsage = useCallback((feature: string) => {
    metrics.increment(`feature.${feature}`);

    Analytics.event('feature_used', {
      feature,
    });
  }, []);

  return {
    trackCheckout,
    trackSignup,
    trackFeatureUsage,
  };
}
```

## 18.7 Alerts and Notifications

### Setting Up Alerts

```typescript
// services/alerting.ts
export class AlertManager {
  private static readonly ALERT_THRESHOLDS = {
    errorRate: 0.01, // 1% error rate
    responseTime: 3000, // 3 seconds
    memoryUsage: 0.9, // 90% memory usage
  };

  static checkErrorRate(errors: number, total: number) {
    const errorRate = errors / total;

    if (errorRate > this.ALERT_THRESHOLDS.errorRate) {
      this.sendAlert('High Error Rate', {
        errorRate: `${(errorRate * 100).toFixed(2)}%`,
        errors,
        total,
        severity: 'critical',
      });
    }
  }

  static checkResponseTime(responseTime: number) {
    if (responseTime > this.ALERT_THRESHOLDS.responseTime) {
      this.sendAlert('Slow Response Time', {
        responseTime: `${responseTime}ms`,
        threshold: `${this.ALERT_THRESHOLDS.responseTime}ms`,
        severity: 'warning',
      });
    }
  }

  private static async sendAlert(title: string, details: Record<string, any>) {
    // Send to Slack
    await fetch(import.meta.env.VITE_SLACK_WEBHOOK_URL, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        text: `🚨 ${title}`,
        blocks: [
          {
            type: 'section',
            text: {
              type: 'mrkdwn',
              text: `*${title}*`,
            },
          },
          {
            type: 'section',
            fields: Object.entries(details).map(([key, value]) => ({
              type: 'mrkdwn',
              text: `*${key}:*\n${value}`,
            })),
          },
        ],
      }),
    });

    // Send to PagerDuty for critical alerts
    if (details.severity === 'critical') {
      await this.sendToPagerDuty(title, details);
    }
  }

  private static async sendToPagerDuty(title: string, details: Record<string, any>) {
    await fetch('https://events.pagerduty.com/v2/enqueue', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        routing_key: import.meta.env.VITE_PAGERDUTY_KEY,
        event_action: 'trigger',
        payload: {
          summary: title,
          severity: 'critical',
          source: window.location.hostname,
          custom_details: details,
        },
      }),
    });
  }
}
```

## Real-World Scenario: Debugging a Production Issue

### The Challenge

Users report:
- Slow page loads
- Intermittent errors
- Features not working

### Senior Approach

```typescript
// 1. Check error tracking (Sentry)
// - Find error spike at 2:15 PM
// - Error: "Cannot read property 'map' of undefined"
// - Affected component: ProductList
// - Release: v2.3.0

// 2. Check performance monitoring
// - LCP increased from 1.2s to 4.5s
// - FID increased from 50ms to 200ms
// - Network requests timing out

// 3. Check session replays
// - Users clicking "Load More" button
// - Request to /api/products fails
// - State becomes undefined
// - Component crashes

// 4. Check APM (Datadog)
// - Database query taking 8 seconds
// - Missing index on products.category
// - 500 errors from API

// 5. Fix and deploy
// - Add database index
// - Add error boundary around ProductList
// - Add loading state handling
// - Deploy fix

// 6. Verify fix
// - Error rate drops to 0%
// - LCP back to 1.2s
// - Users can load products again

// 7. Post-mortem
// - Document incident
// - Add monitoring for slow queries
// - Add tests for error states
// - Set up alerts for similar issues
```

## Chapter Exercise: Implement Complete Monitoring

Set up comprehensive monitoring for your app:

**Requirements:**
1. Error tracking with Sentry
2. Performance monitoring (Web Vitals)
3. Real User Monitoring with analytics
4. Structured logging
5. Custom metrics and dashboards
6. Alerts for critical issues
7. Session replay for debugging

**Deliverables:**
- All monitoring services configured
- Custom metrics tracked
- Alerts set up
- Documentation for on-call team

## Review Checklist

- [ ] Error tracking configured
- [ ] Performance monitoring enabled
- [ ] Web Vitals tracked
- [ ] User analytics implemented
- [ ] Structured logging in place
- [ ] Custom metrics defined
- [ ] Dashboards created
- [ ] Alerts configured
- [ ] Session replay enabled
- [ ] On-call runbooks written

## Key Takeaways

1. **You can't fix what you can't see** - Comprehensive monitoring is essential
2. **Track errors before users report them** - Proactive error tracking
3. **Performance monitoring catches regressions** - Track Web Vitals
4. **Structured logging aids debugging** - Make logs searchable
5. **Custom metrics track business impact** - Not just technical metrics
6. **Alerts prevent outages** - Catch issues early
7. **Session replay saves hours of debugging** - See what users see

## Further Reading

- Sentry documentation
- Google Analytics 4 guide
- Datadog RUM documentation
- Web Vitals library
- Observability Engineering (book)

## Next Chapter

[Chapter 19: Authentication & Authorization](./19-authentication-authorization.md)
