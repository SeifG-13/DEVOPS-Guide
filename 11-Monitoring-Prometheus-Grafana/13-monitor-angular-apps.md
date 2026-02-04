# Monitoring Angular Applications

## Frontend Monitoring Overview

Frontend monitoring focuses on user experience metrics that can't be captured by backend monitoring alone.

```
┌─────────────────────────────────────────────────────────────┐
│              FRONTEND MONITORING STRATEGY                    │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                Angular Application                   │   │
│  │                                                     │   │
│  │  ┌───────────────────┐  ┌───────────────────┐      │   │
│  │  │   Performance     │  │     Errors        │      │   │
│  │  │   • Load time     │  │   • JS errors     │      │   │
│  │  │   • FCP, LCP      │  │   • API failures  │      │   │
│  │  │   • Navigation    │  │   • Unhandled     │      │   │
│  │  └───────────────────┘  └───────────────────┘      │   │
│  │                                                     │   │
│  │  ┌───────────────────┐  ┌───────────────────┐      │   │
│  │  │   User Actions    │  │   API Calls       │      │   │
│  │  │   • Page views    │  │   • Response time │      │   │
│  │  │   • Clicks        │  │   • Error rate    │      │   │
│  │  │   • Form submits  │  │   • Status codes  │      │   │
│  │  └───────────────────┘  └───────────────────┘      │   │
│  └─────────────────────────────────────────────────────┘   │
│                            │                                │
│                            ▼                                │
│  ┌─────────────────────────────────────────────────────┐   │
│  │         Backend Metrics Collector                    │   │
│  │         (Send metrics to backend → Prometheus)       │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Monitoring Approaches

### 1. Real User Monitoring (RUM)
Capture actual user experience in production.

### 2. Synthetic Monitoring
Automated tests simulating user journeys (Blackbox Exporter).

### 3. Backend-Forwarded Metrics
Send frontend metrics to backend for Prometheus collection.

---

## Angular Performance Service

### Create Metrics Service

```typescript
// src/app/services/metrics.service.ts
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { environment } from '../../environments/environment';

interface MetricPayload {
  name: string;
  value: number;
  labels: Record<string, string>;
  timestamp: number;
}

@Injectable({
  providedIn: 'root'
})
export class MetricsService {
  private metricsBuffer: MetricPayload[] = [];
  private flushInterval = 10000; // 10 seconds

  constructor(private http: HttpClient) {
    this.startPeriodicFlush();
    this.initWebVitals();
  }

  // Record a counter increment
  incrementCounter(name: string, labels: Record<string, string> = {}): void {
    this.addMetric(name, 1, labels);
  }

  // Record a gauge value
  setGauge(name: string, value: number, labels: Record<string, string> = {}): void {
    this.addMetric(name, value, labels);
  }

  // Record a timing/histogram observation
  recordTiming(name: string, durationMs: number, labels: Record<string, string> = {}): void {
    this.addMetric(name, durationMs / 1000, labels); // Convert to seconds
  }

  private addMetric(name: string, value: number, labels: Record<string, string>): void {
    this.metricsBuffer.push({
      name: `frontend_${name}`,
      value,
      labels: {
        ...labels,
        app: 'angular-app',
        environment: environment.production ? 'production' : 'development'
      },
      timestamp: Date.now()
    });
  }

  private startPeriodicFlush(): void {
    setInterval(() => {
      this.flushMetrics();
    }, this.flushInterval);

    // Flush on page unload
    window.addEventListener('beforeunload', () => {
      this.flushMetrics();
    });
  }

  private async flushMetrics(): Promise<void> {
    if (this.metricsBuffer.length === 0) return;

    const metricsToSend = [...this.metricsBuffer];
    this.metricsBuffer = [];

    try {
      await this.http.post(
        `${environment.apiUrl}/api/metrics`,
        { metrics: metricsToSend }
      ).toPromise();
    } catch (error) {
      console.error('Failed to send metrics:', error);
      // Re-add failed metrics to buffer (with limit)
      if (this.metricsBuffer.length < 1000) {
        this.metricsBuffer.push(...metricsToSend);
      }
    }
  }

  // Web Vitals tracking
  private initWebVitals(): void {
    // First Contentful Paint
    this.observePaint('first-contentful-paint', 'fcp');

    // Largest Contentful Paint
    this.observeLCP();

    // First Input Delay
    this.observeFID();

    // Cumulative Layout Shift
    this.observeCLS();
  }

  private observePaint(type: string, metricName: string): void {
    const observer = new PerformanceObserver((list) => {
      for (const entry of list.getEntries()) {
        if (entry.name === type) {
          this.recordTiming(`web_vitals_${metricName}_seconds`, entry.startTime, {});
        }
      }
    });
    observer.observe({ type: 'paint', buffered: true });
  }

  private observeLCP(): void {
    const observer = new PerformanceObserver((list) => {
      const entries = list.getEntries();
      const lastEntry = entries[entries.length - 1];
      this.recordTiming('web_vitals_lcp_seconds', lastEntry.startTime, {});
    });
    observer.observe({ type: 'largest-contentful-paint', buffered: true });
  }

  private observeFID(): void {
    const observer = new PerformanceObserver((list) => {
      for (const entry of list.getEntries()) {
        const fidEntry = entry as PerformanceEventTiming;
        this.recordTiming('web_vitals_fid_seconds', fidEntry.processingStart - fidEntry.startTime, {});
      }
    });
    observer.observe({ type: 'first-input', buffered: true });
  }

  private observeCLS(): void {
    let clsValue = 0;
    const observer = new PerformanceObserver((list) => {
      for (const entry of list.getEntries()) {
        if (!(entry as any).hadRecentInput) {
          clsValue += (entry as any).value;
        }
      }
      this.setGauge('web_vitals_cls', clsValue, {});
    });
    observer.observe({ type: 'layout-shift', buffered: true });
  }
}
```

---

## HTTP Interceptor for API Metrics

```typescript
// src/app/interceptors/metrics.interceptor.ts
import { Injectable } from '@angular/core';
import {
  HttpInterceptor,
  HttpRequest,
  HttpHandler,
  HttpEvent,
  HttpResponse,
  HttpErrorResponse
} from '@angular/common/http';
import { Observable, tap, finalize } from 'rxjs';
import { MetricsService } from '../services/metrics.service';

@Injectable()
export class MetricsInterceptor implements HttpInterceptor {
  constructor(private metrics: MetricsService) {}

  intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    const startTime = performance.now();
    const url = new URL(req.url, window.location.origin);
    const endpoint = url.pathname;

    return next.handle(req).pipe(
      tap({
        next: (event) => {
          if (event instanceof HttpResponse) {
            this.recordSuccess(req.method, endpoint, event.status, startTime);
          }
        },
        error: (error: HttpErrorResponse) => {
          this.recordError(req.method, endpoint, error.status, startTime);
        }
      }),
      finalize(() => {
        // Track in-progress requests
        this.metrics.incrementCounter('http_requests_total', {
          method: req.method,
          endpoint: this.normalizeEndpoint(endpoint)
        });
      })
    );
  }

  private recordSuccess(method: string, endpoint: string, status: number, startTime: number): void {
    const duration = performance.now() - startTime;

    this.metrics.recordTiming('http_request_duration_seconds', duration, {
      method,
      endpoint: this.normalizeEndpoint(endpoint),
      status: status.toString()
    });
  }

  private recordError(method: string, endpoint: string, status: number, startTime: number): void {
    const duration = performance.now() - startTime;

    this.metrics.recordTiming('http_request_duration_seconds', duration, {
      method,
      endpoint: this.normalizeEndpoint(endpoint),
      status: status.toString()
    });

    this.metrics.incrementCounter('http_errors_total', {
      method,
      endpoint: this.normalizeEndpoint(endpoint),
      status: status.toString()
    });
  }

  // Normalize endpoint to reduce cardinality
  private normalizeEndpoint(path: string): string {
    // Replace IDs with placeholders
    return path
      .replace(/\/\d+/g, '/:id')
      .replace(/\/[a-f0-9-]{36}/g, '/:uuid');
  }
}
```

### Register Interceptor

```typescript
// app.module.ts
import { HTTP_INTERCEPTORS } from '@angular/common/http';
import { MetricsInterceptor } from './interceptors/metrics.interceptor';

@NgModule({
  providers: [
    {
      provide: HTTP_INTERCEPTORS,
      useClass: MetricsInterceptor,
      multi: true
    }
  ]
})
export class AppModule {}
```

---

## Error Handler for Frontend Errors

```typescript
// src/app/services/error-handler.service.ts
import { ErrorHandler, Injectable, Injector } from '@angular/core';
import { MetricsService } from './metrics.service';
import { HttpErrorResponse } from '@angular/common/http';

@Injectable()
export class GlobalErrorHandler implements ErrorHandler {
  constructor(private injector: Injector) {}

  handleError(error: Error | HttpErrorResponse): void {
    const metricsService = this.injector.get(MetricsService);

    if (error instanceof HttpErrorResponse) {
      // HTTP error
      metricsService.incrementCounter('errors_total', {
        type: 'http',
        status: error.status.toString(),
        url: error.url || 'unknown'
      });
    } else {
      // JavaScript error
      metricsService.incrementCounter('errors_total', {
        type: 'javascript',
        message: this.sanitizeErrorMessage(error.message),
        stack: this.sanitizeStack(error.stack)
      });
    }

    // Still log to console
    console.error(error);
  }

  private sanitizeErrorMessage(message: string): string {
    // Remove sensitive data, truncate
    return message.substring(0, 100);
  }

  private sanitizeStack(stack?: string): string {
    if (!stack) return 'no-stack';
    // Return only the first line
    return stack.split('\n')[0].substring(0, 100);
  }
}
```

### Register Error Handler

```typescript
// app.module.ts
import { ErrorHandler } from '@angular/core';
import { GlobalErrorHandler } from './services/error-handler.service';

@NgModule({
  providers: [
    { provide: ErrorHandler, useClass: GlobalErrorHandler }
  ]
})
export class AppModule {}
```

---

## Router Metrics

```typescript
// src/app/services/router-metrics.service.ts
import { Injectable } from '@angular/core';
import { Router, NavigationStart, NavigationEnd, NavigationError } from '@angular/router';
import { filter } from 'rxjs/operators';
import { MetricsService } from './metrics.service';

@Injectable({
  providedIn: 'root'
})
export class RouterMetricsService {
  private navigationStartTime: number = 0;

  constructor(
    private router: Router,
    private metrics: MetricsService
  ) {
    this.initRouterMetrics();
  }

  private initRouterMetrics(): void {
    this.router.events.pipe(
      filter(event =>
        event instanceof NavigationStart ||
        event instanceof NavigationEnd ||
        event instanceof NavigationError
      )
    ).subscribe(event => {
      if (event instanceof NavigationStart) {
        this.navigationStartTime = performance.now();
      }

      if (event instanceof NavigationEnd) {
        const duration = performance.now() - this.navigationStartTime;

        this.metrics.recordTiming('page_navigation_duration_seconds', duration, {
          route: this.normalizeRoute(event.url)
        });

        this.metrics.incrementCounter('page_views_total', {
          route: this.normalizeRoute(event.url)
        });
      }

      if (event instanceof NavigationError) {
        this.metrics.incrementCounter('navigation_errors_total', {
          route: this.normalizeRoute(event.url),
          error: event.error?.message || 'unknown'
        });
      }
    });
  }

  private normalizeRoute(url: string): string {
    // Remove query params and normalize IDs
    return url
      .split('?')[0]
      .replace(/\/\d+/g, '/:id')
      .replace(/\/[a-f0-9-]{36}/g, '/:uuid');
  }
}
```

### Initialize in App Component

```typescript
// app.component.ts
import { Component } from '@angular/core';
import { RouterMetricsService } from './services/router-metrics.service';

@Component({
  selector: 'app-root',
  template: '<router-outlet></router-outlet>'
})
export class AppComponent {
  // Inject to initialize
  constructor(private routerMetrics: RouterMetricsService) {}
}
```

---

## Backend Metrics Endpoint (.NET 10)

### Metrics Controller

```csharp
// Controllers/MetricsController.cs
using Microsoft.AspNetCore.Mvc;
using Prometheus;

[ApiController]
[Route("api/[controller]")]
public class MetricsController : ControllerBase
{
    private static readonly Counter FrontendCounter = Metrics.CreateCounter(
        "frontend_events_total",
        "Frontend events from Angular app",
        new CounterConfiguration
        {
            LabelNames = new[] { "name", "app", "environment" }
        });

    private static readonly Histogram FrontendTiming = Metrics.CreateHistogram(
        "frontend_timing_seconds",
        "Frontend timing metrics",
        new HistogramConfiguration
        {
            LabelNames = new[] { "name", "app" },
            Buckets = new[] { 0.1, 0.25, 0.5, 1, 2.5, 5, 10, 30 }
        });

    [HttpPost]
    public IActionResult RecordMetrics([FromBody] MetricsPayload payload)
    {
        foreach (var metric in payload.Metrics)
        {
            var labels = metric.Labels ?? new Dictionary<string, string>();

            if (metric.Name.Contains("duration") || metric.Name.Contains("timing"))
            {
                // Histogram observation
                FrontendTiming
                    .WithLabels(metric.Name, labels.GetValueOrDefault("app", "unknown"))
                    .Observe(metric.Value);
            }
            else
            {
                // Counter increment
                FrontendCounter
                    .WithLabels(
                        metric.Name,
                        labels.GetValueOrDefault("app", "unknown"),
                        labels.GetValueOrDefault("environment", "unknown"))
                    .Inc(metric.Value);
            }
        }

        return Ok();
    }
}

public class MetricsPayload
{
    public List<MetricItem> Metrics { get; set; } = new();
}

public class MetricItem
{
    public string Name { get; set; } = string.Empty;
    public double Value { get; set; }
    public Dictionary<string, string>? Labels { get; set; }
    public long Timestamp { get; set; }
}
```

---

## Synthetic Monitoring with Blackbox Exporter

### Configure Blackbox for Frontend

```yaml
# blackbox.yml
modules:
  http_angular_app:
    prober: http
    timeout: 10s
    http:
      valid_http_versions: ["HTTP/1.1", "HTTP/2.0"]
      valid_status_codes: [200]
      method: GET
      headers:
        Accept: text/html
      fail_if_not_ssl: true
      fail_if_body_not_matches_regexp:
        - "<app-root>"  # Angular app selector
```

### Prometheus Config

```yaml
scrape_configs:
  - job_name: 'blackbox-frontend'
    metrics_path: /probe
    params:
      module: [http_angular_app]
    static_configs:
      - targets:
          - https://myapp.example.com
          - https://myapp.example.com/login
          - https://myapp.example.com/dashboard
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: blackbox-exporter:9115
```

---

## Grafana Dashboard Queries

```promql
# Page load time
histogram_quantile(0.95, sum by (le) (rate(frontend_timing_seconds_bucket{name="web_vitals_lcp_seconds"}[5m])))

# Error rate
sum(rate(frontend_events_total{name="errors_total"}[5m])) by (type)

# Page views
sum(rate(frontend_events_total{name="page_views_total"}[5m])) by (route)

# API response time from frontend perspective
histogram_quantile(0.95, sum by (le, endpoint) (rate(frontend_timing_seconds_bucket{name="http_request_duration_seconds"}[5m])))

# Synthetic uptime
probe_success{job="blackbox-frontend"}
```

---

## Alert Rules for Frontend

```yaml
groups:
  - name: frontend-alerts
    rules:
      - alert: HighFrontendErrorRate
        expr: |
          sum(rate(frontend_events_total{name="errors_total"}[5m])) > 0.1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High frontend error rate"
          description: "Frontend error rate is {{ $value }} errors/sec"

      - alert: SlowPageLoad
        expr: |
          histogram_quantile(0.95, sum by (le) (rate(frontend_timing_seconds_bucket{name="web_vitals_lcp_seconds"}[5m]))) > 4
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Slow page load times"
          description: "95th percentile LCP is {{ $value }}s"

      - alert: FrontendDown
        expr: probe_success{job="blackbox-frontend"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Frontend is down"
          description: "{{ $labels.instance }} is not responding"
```

---

## Summary

| Metric Type | What to Track |
|-------------|---------------|
| **Web Vitals** | FCP, LCP, FID, CLS |
| **Navigation** | Page load time, route changes |
| **API Calls** | Response time, error rate |
| **Errors** | JS errors, HTTP errors |
| **User Actions** | Clicks, form submissions |

### Implementation Checklist

```
✓ Install metrics service
✓ Configure HTTP interceptor
✓ Setup error handler
✓ Track router events
✓ Setup backend metrics endpoint
✓ Configure synthetic monitoring
✓ Create Grafana dashboard
✓ Setup alerts
```
