# Logging Angular Applications to ELK

## Frontend Logging Strategies

```
┌─────────────────────────────────────────────────────────────┐
│              ANGULAR LOGGING TO ELK                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Strategy 1: Via Backend API                                │
│  ┌─────────────┐   ┌─────────────┐   ┌───────────────┐    │
│  │   Angular   │──▶│  .NET API   │──▶│ Elasticsearch │    │
│  │   App       │   │ /api/logs   │   └───────────────┘    │
│  └─────────────┘   └─────────────┘                         │
│                                                             │
│  Strategy 2: Direct to Elasticsearch (Not Recommended)     │
│  ┌─────────────┐        ┌───────────────┐                 │
│  │   Angular   │───────▶│ Elasticsearch │                 │
│  │   App       │        └───────────────┘                 │
│  └─────────────┘   (Exposes ES to public!)                │
│                                                             │
│  Strategy 3: Via Log Aggregation Service                   │
│  ┌─────────────┐   ┌───────────┐   ┌───────────────┐     │
│  │   Angular   │──▶│  Logstash │──▶│ Elasticsearch │     │
│  │   App       │   │  (HTTP)   │   └───────────────┘     │
│  └─────────────┘   └───────────┘                          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Logging Service

### Create Logger Service

```typescript
// src/app/services/logger.service.ts
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { environment } from '../../environments/environment';

export enum LogLevel {
  DEBUG = 'DEBUG',
  INFO = 'INFO',
  WARN = 'WARN',
  ERROR = 'ERROR'
}

export interface LogEntry {
  timestamp: string;
  level: LogLevel;
  message: string;
  data?: any;
  context?: {
    url?: string;
    userAgent?: string;
    userId?: string;
    sessionId?: string;
    correlationId?: string;
  };
  error?: {
    name?: string;
    message?: string;
    stack?: string;
  };
}

@Injectable({
  providedIn: 'root'
})
export class LoggerService {
  private buffer: LogEntry[] = [];
  private flushInterval = 5000; // 5 seconds
  private bufferSize = 20;
  private sessionId: string;

  constructor(private http: HttpClient) {
    this.sessionId = this.generateSessionId();
    this.startFlushTimer();
    this.setupUnhandledErrorCapture();
  }

  debug(message: string, data?: any): void {
    if (!environment.production) {
      this.log(LogLevel.DEBUG, message, data);
    }
  }

  info(message: string, data?: any): void {
    this.log(LogLevel.INFO, message, data);
  }

  warn(message: string, data?: any): void {
    this.log(LogLevel.WARN, message, data);
  }

  error(message: string, error?: Error, data?: any): void {
    this.log(LogLevel.ERROR, message, data, error);
  }

  private log(level: LogLevel, message: string, data?: any, error?: Error): void {
    const entry: LogEntry = {
      timestamp: new Date().toISOString(),
      level,
      message,
      data: this.sanitize(data),
      context: {
        url: window.location.href,
        userAgent: navigator.userAgent,
        sessionId: this.sessionId,
        correlationId: this.getCorrelationId()
      }
    };

    if (error) {
      entry.error = {
        name: error.name,
        message: error.message,
        stack: error.stack
      };
    }

    // Console output in development
    if (!environment.production) {
      this.consoleLog(entry);
    }

    // Buffer for batch sending
    this.buffer.push(entry);

    // Immediate flush for errors
    if (level === LogLevel.ERROR || this.buffer.length >= this.bufferSize) {
      this.flush();
    }
  }

  private sanitize(data: any): any {
    if (!data) return undefined;

    // Remove sensitive fields
    const sanitized = { ...data };
    const sensitiveFields = ['password', 'token', 'secret', 'creditCard', 'ssn'];

    for (const field of sensitiveFields) {
      if (sanitized[field]) {
        sanitized[field] = '***REDACTED***';
      }
    }

    return sanitized;
  }

  private consoleLog(entry: LogEntry): void {
    const style = this.getConsoleStyle(entry.level);
    console.log(
      `%c[${entry.level}] ${entry.message}`,
      style,
      entry.data || '',
      entry.error || ''
    );
  }

  private getConsoleStyle(level: LogLevel): string {
    const styles: Record<LogLevel, string> = {
      [LogLevel.DEBUG]: 'color: gray',
      [LogLevel.INFO]: 'color: blue',
      [LogLevel.WARN]: 'color: orange',
      [LogLevel.ERROR]: 'color: red; font-weight: bold'
    };
    return styles[level];
  }

  private flush(): void {
    if (this.buffer.length === 0) return;

    const logsToSend = [...this.buffer];
    this.buffer = [];

    this.http.post(`${environment.apiUrl}/api/logs`, { logs: logsToSend })
      .subscribe({
        error: (err) => {
          console.error('Failed to send logs:', err);
          // Re-add to buffer on failure (with limit)
          if (this.buffer.length < 100) {
            this.buffer.push(...logsToSend);
          }
        }
      });
  }

  private startFlushTimer(): void {
    setInterval(() => this.flush(), this.flushInterval);

    // Flush on page unload
    window.addEventListener('beforeunload', () => {
      this.flush();
    });
  }

  private setupUnhandledErrorCapture(): void {
    window.onerror = (message, source, lineno, colno, error) => {
      this.error('Unhandled error', error, { source, lineno, colno });
      return false;
    };

    window.onunhandledrejection = (event) => {
      this.error('Unhandled promise rejection', undefined, { reason: event.reason });
    };
  }

  private generateSessionId(): string {
    return 'sess_' + Math.random().toString(36).substring(2, 15);
  }

  private getCorrelationId(): string {
    // Get from localStorage or generate new
    let correlationId = sessionStorage.getItem('correlationId');
    if (!correlationId) {
      correlationId = 'corr_' + Math.random().toString(36).substring(2, 15);
      sessionStorage.setItem('correlationId', correlationId);
    }
    return correlationId;
  }
}
```

---

## Global Error Handler

```typescript
// src/app/services/global-error-handler.service.ts
import { ErrorHandler, Injectable, Injector } from '@angular/core';
import { HttpErrorResponse } from '@angular/common/http';
import { LoggerService } from './logger.service';

@Injectable()
export class GlobalErrorHandler implements ErrorHandler {
  constructor(private injector: Injector) {}

  handleError(error: Error | HttpErrorResponse): void {
    const logger = this.injector.get(LoggerService);

    if (error instanceof HttpErrorResponse) {
      // HTTP error
      logger.error('HTTP Error', undefined, {
        status: error.status,
        statusText: error.statusText,
        url: error.url,
        message: error.message
      });
    } else {
      // Application error
      logger.error('Application Error', error, {
        componentStack: (error as any).ngDebugContext?.component?.constructor?.name
      });
    }

    // Rethrow in development
    if (!(error instanceof HttpErrorResponse)) {
      console.error(error);
    }
  }
}
```

### Register Error Handler

```typescript
// app.module.ts
import { ErrorHandler, NgModule } from '@angular/core';
import { GlobalErrorHandler } from './services/global-error-handler.service';

@NgModule({
  providers: [
    { provide: ErrorHandler, useClass: GlobalErrorHandler }
  ]
})
export class AppModule {}
```

---

## HTTP Interceptor for API Logging

```typescript
// src/app/interceptors/logging.interceptor.ts
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
import { LoggerService } from '../services/logger.service';

@Injectable()
export class LoggingInterceptor implements HttpInterceptor {
  constructor(private logger: LoggerService) {}

  intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    const startTime = Date.now();
    const requestId = this.generateRequestId();

    // Log request
    this.logger.debug('HTTP Request', {
      requestId,
      method: req.method,
      url: req.url,
      body: this.sanitizeBody(req.body)
    });

    return next.handle(req).pipe(
      tap({
        next: (event) => {
          if (event instanceof HttpResponse) {
            this.logger.debug('HTTP Response', {
              requestId,
              status: event.status,
              duration: Date.now() - startTime
            });
          }
        },
        error: (error: HttpErrorResponse) => {
          this.logger.error('HTTP Error', undefined, {
            requestId,
            method: req.method,
            url: req.url,
            status: error.status,
            statusText: error.statusText,
            duration: Date.now() - startTime,
            error: error.error
          });
        }
      })
    );
  }

  private generateRequestId(): string {
    return 'req_' + Math.random().toString(36).substring(2, 9);
  }

  private sanitizeBody(body: any): any {
    if (!body) return undefined;
    const sanitized = { ...body };
    delete sanitized.password;
    delete sanitized.token;
    return sanitized;
  }
}
```

### Register Interceptor

```typescript
// app.module.ts
import { HTTP_INTERCEPTORS } from '@angular/common/http';
import { LoggingInterceptor } from './interceptors/logging.interceptor';

@NgModule({
  providers: [
    { provide: HTTP_INTERCEPTORS, useClass: LoggingInterceptor, multi: true }
  ]
})
export class AppModule {}
```

---

## Router Event Logging

```typescript
// src/app/services/router-logger.service.ts
import { Injectable } from '@angular/core';
import { Router, NavigationStart, NavigationEnd, NavigationError } from '@angular/router';
import { filter } from 'rxjs/operators';
import { LoggerService } from './logger.service';

@Injectable({
  providedIn: 'root'
})
export class RouterLoggerService {
  private navigationStart: number = 0;

  constructor(private router: Router, private logger: LoggerService) {
    this.subscribeToRouterEvents();
  }

  private subscribeToRouterEvents(): void {
    this.router.events.pipe(
      filter(event =>
        event instanceof NavigationStart ||
        event instanceof NavigationEnd ||
        event instanceof NavigationError
      )
    ).subscribe(event => {
      if (event instanceof NavigationStart) {
        this.navigationStart = Date.now();
        this.logger.debug('Navigation started', { url: event.url });
      }

      if (event instanceof NavigationEnd) {
        const duration = Date.now() - this.navigationStart;
        this.logger.info('Page view', {
          url: event.urlAfterRedirects,
          duration
        });
      }

      if (event instanceof NavigationError) {
        this.logger.error('Navigation error', event.error, {
          url: event.url
        });
      }
    });
  }
}
```

---

## Backend Log Endpoint (.NET 10)

### Log Controller

```csharp
// Controllers/LogsController.cs
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.RateLimiting;

[ApiController]
[Route("api/[controller]")]
[EnableRateLimiting("logs")]
public class LogsController : ControllerBase
{
    private readonly ILogger<LogsController> _logger;

    public LogsController(ILogger<LogsController> logger)
    {
        _logger = logger;
    }

    [HttpPost]
    public IActionResult ReceiveLogs([FromBody] FrontendLogsRequest request)
    {
        foreach (var log in request.Logs)
        {
            var level = log.Level.ToUpper() switch
            {
                "ERROR" => LogLevel.Error,
                "WARN" => LogLevel.Warning,
                "INFO" => LogLevel.Information,
                _ => LogLevel.Debug
            };

            using (_logger.BeginScope(new Dictionary<string, object>
            {
                ["Source"] = "Frontend",
                ["SessionId"] = log.Context?.SessionId ?? "unknown",
                ["CorrelationId"] = log.Context?.CorrelationId ?? "unknown",
                ["Url"] = log.Context?.Url ?? "unknown",
                ["UserAgent"] = log.Context?.UserAgent ?? "unknown"
            }))
            {
                if (log.Error != null)
                {
                    _logger.Log(level, "Frontend: {Message} - Error: {ErrorMessage}\n{StackTrace}",
                        log.Message, log.Error.Message, log.Error.Stack);
                }
                else
                {
                    _logger.Log(level, "Frontend: {Message} {@Data}",
                        log.Message, log.Data);
                }
            }
        }

        return Ok();
    }
}

public class FrontendLogsRequest
{
    public List<FrontendLogEntry> Logs { get; set; } = new();
}

public class FrontendLogEntry
{
    public string Timestamp { get; set; } = string.Empty;
    public string Level { get; set; } = string.Empty;
    public string Message { get; set; } = string.Empty;
    public object? Data { get; set; }
    public FrontendLogContext? Context { get; set; }
    public FrontendLogError? Error { get; set; }
}

public class FrontendLogContext
{
    public string? Url { get; set; }
    public string? UserAgent { get; set; }
    public string? UserId { get; set; }
    public string? SessionId { get; set; }
    public string? CorrelationId { get; set; }
}

public class FrontendLogError
{
    public string? Name { get; set; }
    public string? Message { get; set; }
    public string? Stack { get; set; }
}
```

### Rate Limiting

```csharp
// Program.cs
builder.Services.AddRateLimiter(options =>
{
    options.AddFixedWindowLimiter("logs", opt =>
    {
        opt.PermitLimit = 100;
        opt.Window = TimeSpan.FromMinutes(1);
        opt.QueueProcessingOrder = QueueProcessingOrder.OldestFirst;
        opt.QueueLimit = 10;
    });
});

app.UseRateLimiter();
```

---

## Usage in Components

```typescript
// src/app/components/checkout/checkout.component.ts
import { Component } from '@angular/core';
import { LoggerService } from '../../services/logger.service';

@Component({
  selector: 'app-checkout',
  template: `...`
})
export class CheckoutComponent {
  constructor(private logger: LoggerService) {}

  async submitOrder(): Promise<void> {
    this.logger.info('Checkout started', { cartItems: this.cart.items.length });

    try {
      const order = await this.orderService.submit(this.cart);

      this.logger.info('Order submitted successfully', {
        orderId: order.id,
        total: order.total
      });

      this.router.navigate(['/order-confirmation', order.id]);
    } catch (error) {
      this.logger.error('Order submission failed', error as Error, {
        cartItems: this.cart.items.length
      });

      this.showErrorMessage('Failed to submit order');
    }
  }

  onPaymentMethodChange(method: string): void {
    this.logger.info('Payment method changed', { method });
  }
}
```

---

## Performance Logging

```typescript
// src/app/services/performance-logger.service.ts
import { Injectable } from '@angular/core';
import { LoggerService } from './logger.service';

@Injectable({
  providedIn: 'root'
})
export class PerformanceLoggerService {
  constructor(private logger: LoggerService) {
    this.logWebVitals();
  }

  private logWebVitals(): void {
    // First Contentful Paint
    const paintObserver = new PerformanceObserver((list) => {
      for (const entry of list.getEntries()) {
        if (entry.name === 'first-contentful-paint') {
          this.logger.info('Web Vital: FCP', { value: entry.startTime });
        }
      }
    });
    paintObserver.observe({ type: 'paint', buffered: true });

    // Largest Contentful Paint
    const lcpObserver = new PerformanceObserver((list) => {
      const entries = list.getEntries();
      const lastEntry = entries[entries.length - 1];
      this.logger.info('Web Vital: LCP', { value: lastEntry.startTime });
    });
    lcpObserver.observe({ type: 'largest-contentful-paint', buffered: true });

    // Cumulative Layout Shift
    let clsValue = 0;
    const clsObserver = new PerformanceObserver((list) => {
      for (const entry of list.getEntries()) {
        if (!(entry as any).hadRecentInput) {
          clsValue += (entry as any).value;
        }
      }
      this.logger.info('Web Vital: CLS', { value: clsValue });
    });
    clsObserver.observe({ type: 'layout-shift', buffered: true });
  }
}
```

---

## Summary

| Component | Purpose |
|-----------|---------|
| **LoggerService** | Core logging functionality |
| **GlobalErrorHandler** | Catch unhandled errors |
| **LoggingInterceptor** | HTTP request/response logging |
| **RouterLoggerService** | Page navigation tracking |
| **Backend Endpoint** | Receive and forward logs |

### Best Practices

```
✓ Buffer logs and send in batches
✓ Flush immediately on errors
✓ Include correlation IDs
✓ Sanitize sensitive data
✓ Rate limit the log endpoint
✓ Log user actions for analytics
```
