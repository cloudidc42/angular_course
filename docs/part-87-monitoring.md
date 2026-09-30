# Part 87: Error Monitoring และ APM ด้วย Sentry

## ทำไมต้อง Monitor

Production errors ที่ไม่รู้จะทำให้ผู้ใช้เจอปัญหาโดยไม่มีใครทราบ Sentry ช่วยติดตาม errors แบบ real-time

---

## 1. ติดตั้ง Sentry

```bash
npm install @sentry/angular @sentry/tracing
```

### app.module.ts

```typescript
import { NgModule, ErrorHandler, APP_INITIALIZER } from '@angular/core';
import * as Sentry from '@sentry/angular';
import { BrowserTracing } from '@sentry/tracing';
import { Router } from '@angular/router';

// เริ่มต้น Sentry
Sentry.init({
  dsn: 'https://YOUR_KEY@sentry.io/YOUR_PROJECT',
  integrations: [
    new BrowserTracing({
      tracePropagationTargets: ['localhost', 'https://api.myapp.com'],
      routingInstrumentation: Sentry.routingInstrumentation
    })
  ],
  tracesSampleRate: 0.1,     // 10% of transactions
  replaysSessionSampleRate: 0.1,
  replaysOnErrorSampleRate: 1.0,
  environment: environment.production ? 'production' : 'development',
  release: 'my-app@1.0.0',
  beforeSend(event) {
    // กรอง events ที่ไม่ต้องการ
    if (event.exception?.values?.[0]?.type === 'ChunkLoadError') {
      return null;  // ไม่ส่ง chunk load errors
    }
    return event;
  }
});

@NgModule({
  providers: [
    {
      provide: ErrorHandler,
      useValue: Sentry.createErrorHandler({
        showDialog: false  // ไม่แสดง dialog ให้ user
      })
    },
    {
      provide: Sentry.TraceService,
      deps: [Router]
    },
    {
      provide: APP_INITIALIZER,
      useFactory: () => () => {},
      deps: [Sentry.TraceService],
      multi: true
    }
  ]
})
export class AppModule {}

declare const environment: { production: boolean };
```

---

## 2. Custom Error Handler

```typescript
// core/error-handling/global-error-handler.ts
import { Injectable, ErrorHandler, Injector, NgZone } from '@angular/core';
import { HttpErrorResponse } from '@angular/common/http';
import * as Sentry from '@sentry/angular';

@Injectable()
export class GlobalErrorHandler implements ErrorHandler {
  constructor(
    private injector: Injector,
    private ngZone: NgZone
  ) {}

  handleError(error: any): void {
    // ทำงานนอก NgZone เพื่อไม่ trigger change detection
    this.ngZone.runOutsideAngular(() => {
      if (error instanceof HttpErrorResponse) {
        this.handleHttpError(error);
      } else {
        this.handleClientError(error);
      }
    });
  }

  private handleHttpError(error: HttpErrorResponse): void {
    const status = error.status;
    
    // ไม่ log 4xx ที่คาดได้
    if (status === 401 || status === 403 || status === 404) {
      return;
    }

    Sentry.withScope(scope => {
      scope.setTag('error_type', 'http');
      scope.setTag('http_status', String(status));
      scope.setContext('http', {
        url: error.url,
        status,
        message: error.message
      });
      Sentry.captureException(error);
    });
  }

  private handleClientError(error: any): void {
    // แยก chunk load errors
    if (error?.name === 'ChunkLoadError' || error?.message?.includes('Loading chunk')) {
      console.warn('Chunk load error - reloading...', error);
      window.location.reload();
      return;
    }

    Sentry.withScope(scope => {
      scope.setTag('error_type', 'javascript');
      Sentry.captureException(error.originalError || error);
    });

    console.error('Unhandled error:', error);
  }
}
```

---

## 3. Logging Service

```typescript
// core/logging/logger.service.ts
import { Injectable } from '@angular/core';
import * as Sentry from '@sentry/angular';

export type LogLevel = 'debug' | 'info' | 'warning' | 'error' | 'fatal';

export interface LogEntry {
  level: LogLevel;
  message: string;
  data?: Record<string, any>;
  timestamp: Date;
}

@Injectable({ providedIn: 'root' })
export class LoggerService {
  private logs: LogEntry[] = [];
  private maxLogs = 1000;

  debug(message: string, data?: Record<string, any>): void {
    this.log('debug', message, data);
  }

  info(message: string, data?: Record<string, any>): void {
    this.log('info', message, data);
  }

  warn(message: string, data?: Record<string, any>): void {
    this.log('warning', message, data);
    
    Sentry.addBreadcrumb({
      type: 'default',
      category: 'warning',
      message,
      data,
      level: 'warning'
    });
  }

  error(message: string, error?: Error | any, data?: Record<string, any>): void {
    this.log('error', message, { error: error?.message, ...data });

    Sentry.withScope(scope => {
      if (data) scope.setContext('context', data);
      if (message) scope.setTag('error_message', message.slice(0, 100));
      
      if (error instanceof Error) {
        Sentry.captureException(error);
      } else {
        Sentry.captureMessage(message, 'error');
      }
    });
  }

  setUser(userId: string, email?: string, username?: string): void {
    Sentry.setUser({ id: userId, email, username });
  }

  addBreadcrumb(message: string, category: string, data?: Record<string, any>): void {
    Sentry.addBreadcrumb({
      message,
      category,
      data,
      timestamp: Date.now() / 1000
    });
  }

  private log(level: LogLevel, message: string, data?: Record<string, any>): void {
    const entry: LogEntry = { level, message, data, timestamp: new Date() };
    
    this.logs.push(entry);
    if (this.logs.length > this.maxLogs) {
      this.logs.shift();
    }

    const logFn = level === 'debug' ? 'debug' 
                : level === 'info' ? 'info'
                : level === 'warning' ? 'warn' 
                : 'error';
    
    console[logFn](`[${level.toUpperCase()}] ${message}`, data || '');
  }

  getRecentLogs(count = 50): LogEntry[] {
    return this.logs.slice(-count);
  }
}
```

---

## 4. Performance Monitoring

```typescript
// core/monitoring/performance-monitor.service.ts
import { Injectable } from '@angular/core';
import * as Sentry from '@sentry/angular';

@Injectable({ providedIn: 'root' })
export class PerformanceMonitorService {
  
  // เริ่ม transaction
  startTransaction(name: string, operation: string): any {
    return Sentry.startTransaction({ name, op: operation });
  }

  // วัดเวลา operation
  measureOperation<T>(
    name: string, 
    operation: () => Promise<T>,
    tags?: Record<string, string>
  ): Promise<T> {
    const transaction = this.startTransaction(name, 'operation');
    
    if (tags) {
      Object.entries(tags).forEach(([key, value]) => {
        transaction.setTag(key, value);
      });
    }

    return operation()
      .then(result => {
        transaction.finish();
        return result;
      })
      .catch(error => {
        transaction.setStatus('internal_error');
        transaction.finish();
        throw error;
      });
  }

  // Custom Web Vitals
  trackWebVital(name: string, value: number): void {
    const metric: Record<string, number> = {};
    metric[name] = value;
    
    Sentry.setMeasurement(name, value, 'millisecond');
  }

  // Performance Observer
  setupWebVitals(): void {
    if ('web-vital' in window) return;

    // Observe LCP
    const lcpObserver = new PerformanceObserver((list) => {
      const entries = list.getEntries();
      const lastEntry = entries[entries.length - 1] as any;
      this.trackWebVital('lcp', lastEntry.renderTime || lastEntry.loadTime);
    });
    lcpObserver.observe({ entryTypes: ['largest-contentful-paint'] });

    // Observe FID
    const fidObserver = new PerformanceObserver((list) => {
      list.getEntries().forEach((entry: any) => {
        this.trackWebVital('fid', entry.processingStart - entry.startTime);
      });
    });
    fidObserver.observe({ entryTypes: ['first-input'] });

    // CLS
    let clsValue = 0;
    const clsObserver = new PerformanceObserver((list) => {
      list.getEntries().forEach((entry: any) => {
        if (!entry.hadRecentInput) {
          clsValue += entry.value;
          this.trackWebVital('cls', clsValue);
        }
      });
    });
    clsObserver.observe({ entryTypes: ['layout-shift'] });
  }
}
```

---

## 5. Health Check Component

```typescript
// admin/health/health-check.component.ts
import { Component, OnInit } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { interval } from 'rxjs';
import { switchMap, catchError } from 'rxjs/operators';
import { of } from 'rxjs';

interface ServiceHealth {
  name: string;
  status: 'healthy' | 'degraded' | 'down';
  responseTime: number;
  lastChecked: Date;
  error?: string;
}

@Component({
  selector: 'app-health-check',
  template: `
    <div class="health-dashboard">
      <h2>System Health</h2>
      
      <div class="overall-status" [class]="overallStatus">
        <span class="status-icon">{{ getStatusIcon(overallStatus) }}</span>
        <span>{{ overallStatus === 'healthy' ? 'ระบบทำงานปกติ' : 'มีบางส่วนผิดปกติ' }}</span>
      </div>

      <div class="services-grid">
        <div 
          *ngFor="let service of services"
          class="service-card"
          [class]="service.status"
        >
          <div class="service-header">
            <h3>{{ service.name }}</h3>
            <span class="status-badge">{{ service.status }}</span>
          </div>
          
          <div class="service-metrics">
            <div class="metric">
              <span>Response Time</span>
              <strong [class.slow]="service.responseTime > 1000">
                {{ service.responseTime }}ms
              </strong>
            </div>
            <div class="metric">
              <span>Last Checked</span>
              <strong>{{ service.lastChecked | date:'HH:mm:ss' }}</strong>
            </div>
          </div>
          
          <div class="error-message" *ngIf="service.error">
            {{ service.error }}
          </div>
        </div>
      </div>
    </div>
  `,
  styles: [`
    .health-dashboard { padding: 24px; }
    .overall-status {
      padding: 16px 24px;
      border-radius: 8px;
      display: flex;
      align-items: center;
      gap: 12px;
      margin-bottom: 24px;
      font-size: 18px;
    }
    .overall-status.healthy { background: #e8f5e9; color: #2e7d32; }
    .overall-status.degraded { background: #fff3e0; color: #e65100; }
    .overall-status.down { background: #ffebee; color: #b71c1c; }
    .status-icon { font-size: 24px; }
    .services-grid { display: grid; grid-template-columns: repeat(3, 1fr); gap: 16px; }
    .service-card { background: white; border-radius: 8px; padding: 16px; box-shadow: 0 2px 8px rgba(0,0,0,0.08); border-left: 4px solid; }
    .service-card.healthy { border-left-color: #4caf50; }
    .service-card.degraded { border-left-color: #ff9800; }
    .service-card.down { border-left-color: #f44336; }
    .service-header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 12px; }
    .status-badge { padding: 3px 8px; border-radius: 4px; font-size: 12px; font-weight: 500; }
    .healthy .status-badge { background: #e8f5e9; color: #2e7d32; }
    .degraded .status-badge { background: #fff3e0; color: #e65100; }
    .down .status-badge { background: #ffebee; color: #b71c1c; }
    .service-metrics { display: flex; gap: 16px; }
    .metric { display: flex; flex-direction: column; gap: 2px; }
    .metric span { font-size: 12px; color: #999; }
    .metric strong { font-size: 14px; }
    .metric strong.slow { color: #f44336; }
    .error-message { margin-top: 8px; padding: 8px; background: #ffebee; border-radius: 4px; font-size: 12px; color: #c62828; }
  `]
})
export class HealthCheckComponent implements OnInit {
  services: ServiceHealth[] = [];

  ngOnInit(): void {
    this.checkHealth();
    
    // ตรวจสอบทุก 30 วินาที
    interval(30000).pipe(
      switchMap(() => this.runHealthChecks())
    ).subscribe();
  }

  private async checkHealth(): Promise<void> {
    await this.runHealthChecks().toPromise();
  }

  private runHealthChecks() {
    const checks = [
      { name: 'API Server', url: '/api/health' },
      { name: 'Database', url: '/api/health/db' },
      { name: 'Cache (Redis)', url: '/api/health/cache' },
      { name: 'Storage', url: '/api/health/storage' }
    ];

    return new (class {
      subscribe(callback: any) {
        Promise.all(checks.map(async check => {
          const start = Date.now();
          try {
            await fetch(check.url);
            return { name: check.name, status: 'healthy' as const, responseTime: Date.now() - start, lastChecked: new Date() };
          } catch (error: any) {
            return { name: check.name, status: 'down' as const, responseTime: Date.now() - start, lastChecked: new Date(), error: error.message };
          }
        })).then(results => {
          callback(results);
        });
        return { unsubscribe: () => {} };
      }
    })();
  }

  get overallStatus(): 'healthy' | 'degraded' | 'down' {
    if (this.services.some(s => s.status === 'down')) return 'down';
    if (this.services.some(s => s.status === 'degraded')) return 'degraded';
    return 'healthy';
  }

  getStatusIcon(status: string): string {
    return status === 'healthy' ? '✅' : status === 'degraded' ? '⚠️' : '❌';
  }
}
```

---

## สรุป

| Tool | ใช้สำหรับ |
|------|---------|
| Sentry | Error tracking, Performance |
| Datadog | APM, Infrastructure |
| New Relic | Full-stack observability |
| LogRocket | Session replay |

### Monitoring Checklist

- [ ] ติดตั้ง Sentry หรือ equivalent
- [ ] Custom ErrorHandler
- [ ] Logging service
- [ ] Health check endpoints
- [ ] Performance metrics
- [ ] Alert rules (error rate, p95 latency)
- [ ] On-call rotation
