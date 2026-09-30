# Part 51: Error Handling ใน Angular

## บทนำ

การจัดการข้อผิดพลาด (Error Handling) เป็นส่วนสำคัญของแอปพลิเคชันที่ดี ใน Angular มีหลายวิธีในการจัดการ error ตั้งแต่ระดับ Global จนถึงระดับ Component

## 1. Global Error Handler

Angular มี `ErrorHandler` service ที่สามารถ override ได้เพื่อจัดการ error ทั่วทั้งแอปพลิเคชัน

```typescript
// global-error-handler.ts
import { ErrorHandler, Injectable, NgZone } from '@angular/core';
import { HttpErrorResponse } from '@angular/common/http';
import { Router } from '@angular/router';

@Injectable()
export class GlobalErrorHandler implements ErrorHandler {
  constructor(
    private router: Router,
    private ngZone: NgZone
  ) {}

  handleError(error: Error | HttpErrorResponse): void {
    console.error('Global Error Handler caught:', error);

    if (error instanceof HttpErrorResponse) {
      // จัดการ HTTP Error
      this.handleHttpError(error);
    } else {
      // จัดการ Client Error
      this.handleClientError(error);
    }
  }

  private handleHttpError(error: HttpErrorResponse): void {
    const message = this.getHttpErrorMessage(error);
    console.error('HTTP Error:', message);

    this.ngZone.run(() => {
      switch (error.status) {
        case 401:
          this.router.navigate(['/login']);
          break;
        case 403:
          this.router.navigate(['/forbidden']);
          break;
        case 404:
          this.router.navigate(['/not-found']);
          break;
        case 500:
          this.router.navigate(['/server-error']);
          break;
        default:
          console.error('Unhandled HTTP error:', error);
      }
    });
  }

  private handleClientError(error: Error): void {
    console.error('Client Error:', error.message);
    // บันทึก error ไปยัง logging service
  }

  private getHttpErrorMessage(error: HttpErrorResponse): string {
    if (error.error instanceof ErrorEvent) {
      return `Client Error: ${error.error.message}`;
    }
    return `Server Error: ${error.status} - ${error.message}`;
  }
}
```

### การลงทะเบียน Global Error Handler

```typescript
// app.module.ts
import { NgModule, ErrorHandler } from '@angular/core';
import { BrowserModule } from '@angular/platform-browser';
import { GlobalErrorHandler } from './global-error-handler';

@NgModule({
  imports: [BrowserModule],
  providers: [
    {
      provide: ErrorHandler,
      useClass: GlobalErrorHandler
    }
  ],
  bootstrap: [AppComponent]
})
export class AppModule {}
```

### สำหรับ Standalone Application (Angular 17+)

```typescript
// main.ts
import { bootstrapApplication } from '@angular/platform-browser';
import { provideRouter } from '@angular/router';
import { ErrorHandler } from '@angular/core';
import { AppComponent } from './app/app.component';
import { GlobalErrorHandler } from './app/global-error-handler';

bootstrapApplication(AppComponent, {
  providers: [
    provideRouter([]),
    {
      provide: ErrorHandler,
      useClass: GlobalErrorHandler
    }
  ]
});
```

## 2. HTTP Error Interceptor

```typescript
// error.interceptor.ts
import { Injectable } from '@angular/core';
import {
  HttpInterceptor,
  HttpRequest,
  HttpHandler,
  HttpEvent,
  HttpErrorResponse
} from '@angular/common/http';
import { Observable, throwError } from 'rxjs';
import { catchError, retry } from 'rxjs/operators';
import { Router } from '@angular/router';
import { NotificationService } from './notification.service';

@Injectable()
export class ErrorInterceptor implements HttpInterceptor {
  constructor(
    private router: Router,
    private notificationService: NotificationService
  ) {}

  intercept(
    request: HttpRequest<unknown>,
    next: HttpHandler
  ): Observable<HttpEvent<unknown>> {
    return next.handle(request).pipe(
      retry(1), // ลองใหม่ 1 ครั้งก่อน error
      catchError((error: HttpErrorResponse) => {
        let errorMessage = '';

        if (error.error instanceof ErrorEvent) {
          // Client-side error
          errorMessage = `Error: ${error.error.message}`;
        } else {
          // Server-side error
          errorMessage = this.getServerErrorMessage(error);
        }

        this.notificationService.showError(errorMessage);
        return throwError(() => new Error(errorMessage));
      })
    );
  }

  private getServerErrorMessage(error: HttpErrorResponse): string {
    switch (error.status) {
      case 400:
        return 'คำขอไม่ถูกต้อง (Bad Request)';
      case 401:
        return 'กรุณาเข้าสู่ระบบ (Unauthorized)';
      case 403:
        return 'ไม่มีสิทธิ์เข้าถึง (Forbidden)';
      case 404:
        return 'ไม่พบข้อมูลที่ต้องการ (Not Found)';
      case 409:
        return 'ข้อมูลซ้ำกัน (Conflict)';
      case 422:
        return 'ข้อมูลไม่ถูกต้อง (Unprocessable Entity)';
      case 500:
        return 'เซิร์ฟเวอร์มีปัญหา (Internal Server Error)';
      case 503:
        return 'เซิร์ฟเวอร์ไม่พร้อมใช้งาน (Service Unavailable)';
      default:
        return `เกิดข้อผิดพลาด (${error.status})`;
    }
  }
}
```

## 3. ErrorBoundary Pattern

Angular ไม่มี ErrorBoundary แบบ React แต่สามารถสร้าง pattern ที่คล้ายกันได้

```typescript
// error-boundary.component.ts
import {
  Component,
  OnInit,
  Input,
  ChangeDetectorRef,
  TemplateRef
} from '@angular/core';
import { CommonModule } from '@angular/common';

@Component({
  selector: 'app-error-boundary',
  standalone: true,
  imports: [CommonModule],
  template: `
    <ng-container *ngIf="!hasError; else errorTemplate">
      <ng-content></ng-content>
    </ng-container>
    <ng-template #errorTemplate>
      <div class="error-boundary">
        <div class="error-icon">⚠️</div>
        <h2>เกิดข้อผิดพลาด</h2>
        <p>{{ errorMessage }}</p>
        <button (click)="retry()">ลองใหม่</button>
      </div>
    </ng-template>
  `,
  styles: [`
    .error-boundary {
      display: flex;
      flex-direction: column;
      align-items: center;
      padding: 2rem;
      text-align: center;
    }
    .error-icon { font-size: 3rem; margin-bottom: 1rem; }
    button {
      margin-top: 1rem;
      padding: 0.5rem 1rem;
      background: #007bff;
      color: white;
      border: none;
      border-radius: 4px;
      cursor: pointer;
    }
  `]
})
export class ErrorBoundaryComponent {
  @Input() fallback?: TemplateRef<any>;
  
  hasError = false;
  errorMessage = '';

  constructor(private cdr: ChangeDetectorRef) {}

  setError(error: Error): void {
    this.hasError = true;
    this.errorMessage = error.message;
    this.cdr.detectChanges();
  }

  retry(): void {
    this.hasError = false;
    this.errorMessage = '';
    this.cdr.detectChanges();
  }
}
```

### ใช้งาน ErrorBoundary

```typescript
// parent.component.ts
import { Component, ViewChild } from '@angular/core';
import { ErrorBoundaryComponent } from './error-boundary.component';
import { ChildComponent } from './child.component';

@Component({
  selector: 'app-parent',
  standalone: true,
  imports: [ErrorBoundaryComponent, ChildComponent],
  template: `
    <app-error-boundary #boundary>
      <app-child (error)="boundary.setError($event)"></app-child>
    </app-error-boundary>
  `
})
export class ParentComponent {}
```

## 4. Notification Service สำหรับแสดง Error

```typescript
// notification.service.ts
import { Injectable } from '@angular/core';
import { Subject, Observable } from 'rxjs';

export interface Notification {
  type: 'success' | 'error' | 'warning' | 'info';
  message: string;
  duration?: number;
}

@Injectable({ providedIn: 'root' })
export class NotificationService {
  private notificationSubject = new Subject<Notification>();
  
  notifications$: Observable<Notification> = this.notificationSubject.asObservable();

  showSuccess(message: string, duration = 3000): void {
    this.show({ type: 'success', message, duration });
  }

  showError(message: string, duration = 5000): void {
    this.show({ type: 'error', message, duration });
  }

  showWarning(message: string, duration = 4000): void {
    this.show({ type: 'warning', message, duration });
  }

  showInfo(message: string, duration = 3000): void {
    this.show({ type: 'info', message, duration });
  }

  private show(notification: Notification): void {
    this.notificationSubject.next(notification);
  }
}
```

```typescript
// toast.component.ts
import { Component, OnInit, OnDestroy } from '@angular/core';
import { CommonModule } from '@angular/common';
import { Subject } from 'rxjs';
import { takeUntil } from 'rxjs/operators';
import { NotificationService, Notification } from './notification.service';

@Component({
  selector: 'app-toast',
  standalone: true,
  imports: [CommonModule],
  template: `
    <div class="toast-container">
      <div
        *ngFor="let toast of toasts; trackBy: trackByFn"
        class="toast"
        [class]="'toast--' + toast.type"
        [@fadeInOut]
      >
        <span class="toast__icon">{{ getIcon(toast.type) }}</span>
        <span class="toast__message">{{ toast.message }}</span>
        <button class="toast__close" (click)="remove(toast)">×</button>
      </div>
    </div>
  `,
  styles: [`
    .toast-container {
      position: fixed;
      top: 1rem;
      right: 1rem;
      z-index: 9999;
      display: flex;
      flex-direction: column;
      gap: 0.5rem;
    }
    .toast {
      display: flex;
      align-items: center;
      padding: 0.75rem 1rem;
      border-radius: 4px;
      min-width: 300px;
      box-shadow: 0 2px 8px rgba(0,0,0,0.2);
    }
    .toast--success { background: #28a745; color: white; }
    .toast--error { background: #dc3545; color: white; }
    .toast--warning { background: #ffc107; color: black; }
    .toast--info { background: #17a2b8; color: white; }
    .toast__message { flex: 1; margin: 0 0.5rem; }
    .toast__close { background: none; border: none; color: inherit; cursor: pointer; font-size: 1.2rem; }
  `]
})
export class ToastComponent implements OnInit, OnDestroy {
  toasts: (Notification & { id: number })[] = [];
  private destroy$ = new Subject<void>();
  private nextId = 0;

  constructor(private notificationService: NotificationService) {}

  ngOnInit(): void {
    this.notificationService.notifications$
      .pipe(takeUntil(this.destroy$))
      .subscribe(notification => {
        this.addToast(notification);
      });
  }

  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
  }

  private addToast(notification: Notification): void {
    const toast = { ...notification, id: this.nextId++ };
    this.toasts.push(toast);

    if (notification.duration) {
      setTimeout(() => this.remove(toast), notification.duration);
    }
  }

  remove(toast: { id: number }): void {
    this.toasts = this.toasts.filter(t => t.id !== toast.id);
  }

  getIcon(type: string): string {
    const icons: Record<string, string> = {
      success: '✓',
      error: '✗',
      warning: '⚠',
      info: 'ℹ'
    };
    return icons[type] || '';
  }

  trackByFn(index: number, item: { id: number }): number {
    return item.id;
  }
}
```

## 5. Sentry Integration

```bash
npm install @sentry/angular @sentry/tracing
```

```typescript
// main.ts
import * as Sentry from '@sentry/angular';
import { BrowserTracing } from '@sentry/tracing';

Sentry.init({
  dsn: 'https://your-sentry-dsn@sentry.io/project-id',
  integrations: [
    new BrowserTracing({
      tracePropagationTargets: ['localhost', 'https://yourapi.com'],
      routingInstrumentation: Sentry.routingInstrumentation,
    }),
  ],
  tracesSampleRate: 1.0,
  environment: 'production',
  release: '1.0.0',
});

bootstrapApplication(AppComponent, {
  providers: [
    {
      provide: ErrorHandler,
      useValue: Sentry.createErrorHandler({
        showDialog: false,
        logErrors: true
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
});
```

### Sentry Error Handler แบบ Custom

```typescript
// sentry-error-handler.ts
import { ErrorHandler, Injectable } from '@angular/core';
import * as Sentry from '@sentry/angular';

@Injectable()
export class SentryErrorHandler implements ErrorHandler {
  handleError(error: any): void {
    const chunkFailedMessage = /Loading chunk [\d]+ failed/;
    
    if (chunkFailedMessage.test(error.message)) {
      // Reload หน้าเมื่อ chunk load ไม่สำเร็จ
      window.location.reload();
      return;
    }

    Sentry.captureException(error.originalError || error);
    console.error(error);
  }
}
```

## 6. การจัดการ Error ใน Component

```typescript
// data.service.ts
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable, throwError } from 'rxjs';
import { catchError, map } from 'rxjs/operators';

export interface ApiError {
  code: string;
  message: string;
  details?: string[];
}

@Injectable({ providedIn: 'root' })
export class DataService {
  constructor(private http: HttpClient) {}

  getUsers(): Observable<any[]> {
    return this.http.get<any[]>('/api/users').pipe(
      map(response => response),
      catchError(error => {
        const apiError: ApiError = {
          code: error.status?.toString() || 'UNKNOWN',
          message: error.error?.message || 'เกิดข้อผิดพลาดที่ไม่ทราบสาเหตุ',
          details: error.error?.details
        };
        return throwError(() => apiError);
      })
    );
  }
}
```

```typescript
// users.component.ts
import { Component, OnInit } from '@angular/core';
import { CommonModule } from '@angular/common';
import { DataService, ApiError } from './data.service';

@Component({
  selector: 'app-users',
  standalone: true,
  imports: [CommonModule],
  template: `
    <div *ngIf="loading" class="loading">กำลังโหลด...</div>
    
    <div *ngIf="error" class="error-container">
      <h3>เกิดข้อผิดพลาด</h3>
      <p>{{ error.message }}</p>
      <ul *ngIf="error.details?.length">
        <li *ngFor="let detail of error.details">{{ detail }}</li>
      </ul>
      <button (click)="loadUsers()">ลองใหม่</button>
    </div>

    <div *ngIf="!loading && !error">
      <div *ngFor="let user of users" class="user-card">
        {{ user.name }}
      </div>
    </div>
  `
})
export class UsersComponent implements OnInit {
  users: any[] = [];
  loading = false;
  error: ApiError | null = null;

  constructor(private dataService: DataService) {}

  ngOnInit(): void {
    this.loadUsers();
  }

  loadUsers(): void {
    this.loading = true;
    this.error = null;

    this.dataService.getUsers().subscribe({
      next: (users) => {
        this.users = users;
        this.loading = false;
      },
      error: (err: ApiError) => {
        this.error = err;
        this.loading = false;
      }
    });
  }
}
```

## 7. Custom Error Classes

```typescript
// errors.ts
export class AppError extends Error {
  constructor(
    message: string,
    public code: string,
    public details?: any
  ) {
    super(message);
    this.name = 'AppError';
    Object.setPrototypeOf(this, AppError.prototype);
  }
}

export class ValidationError extends AppError {
  constructor(
    public fields: Record<string, string[]>,
    message = 'ข้อมูลไม่ถูกต้อง'
  ) {
    super(message, 'VALIDATION_ERROR', fields);
    this.name = 'ValidationError';
    Object.setPrototypeOf(this, ValidationError.prototype);
  }
}

export class NetworkError extends AppError {
  constructor(message = 'ไม่สามารถเชื่อมต่อเครือข่ายได้') {
    super(message, 'NETWORK_ERROR');
    this.name = 'NetworkError';
    Object.setPrototypeOf(this, NetworkError.prototype);
  }
}

export class AuthenticationError extends AppError {
  constructor(message = 'กรุณาเข้าสู่ระบบ') {
    super(message, 'AUTH_ERROR');
    this.name = 'AuthenticationError';
    Object.setPrototypeOf(this, AuthenticationError.prototype);
  }
}
```

## 8. Error Logging Service

```typescript
// error-logging.service.ts
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable, of } from 'rxjs';
import { catchError } from 'rxjs/operators';

export interface ErrorLog {
  timestamp: string;
  level: 'error' | 'warning' | 'info';
  message: string;
  stack?: string;
  context?: Record<string, any>;
  userAgent: string;
  url: string;
}

@Injectable({ providedIn: 'root' })
export class ErrorLoggingService {
  private readonly logEndpoint = '/api/logs/errors';

  constructor(private http: HttpClient) {}

  logError(error: Error, context?: Record<string, any>): void {
    const log: ErrorLog = {
      timestamp: new Date().toISOString(),
      level: 'error',
      message: error.message,
      stack: error.stack,
      context,
      userAgent: navigator.userAgent,
      url: window.location.href
    };

    this.sendLog(log).subscribe();
    console.error('[Error Log]', log);
  }

  private sendLog(log: ErrorLog): Observable<void> {
    return this.http.post<void>(this.logEndpoint, log).pipe(
      catchError(err => {
        console.error('Failed to send error log:', err);
        return of(void 0);
      })
    );
  }
}
```

## 9. Retry Strategy

```typescript
// retry.utils.ts
import { Observable, throwError, timer } from 'rxjs';
import { mergeMap, retryWhen } from 'rxjs/operators';

export function retryWithDelay(
  maxRetries = 3,
  delayMs = 1000,
  backoffFactor = 2
) {
  return (attempts: Observable<any>) => {
    return attempts.pipe(
      mergeMap((error, index) => {
        const retryAttempt = index + 1;
        
        if (retryAttempt > maxRetries) {
          return throwError(() => error);
        }

        const delay = delayMs * Math.pow(backoffFactor, retryAttempt - 1);
        console.log(`ลองใหม่ครั้งที่ ${retryAttempt} ใน ${delay}ms`);
        
        return timer(delay);
      })
    );
  };
}

// การใช้งาน
// this.http.get('/api/data').pipe(
//   retryWhen(retryWithDelay(3, 1000, 2))
// )
```

## 10. สรุป

| เทคนิค | ใช้เมื่อไหร่ |
|--------|------------|
| Global ErrorHandler | จัดการ error ทั้งหมดในแอป |
| HTTP Interceptor | จัดการ HTTP error โดยเฉพาะ |
| ErrorBoundary | แยก error ไม่ให้กระทบส่วนอื่น |
| Component Error State | แสดง error UI ใน component |
| Sentry | Monitoring และ tracking ใน production |
| Custom Error Classes | จัดหมวดหมู่ error ให้ชัดเจน |

การจัดการ error ที่ดีทำให้แอปพลิเคชันมีความน่าเชื่อถือ และผู้ใช้งานได้รับประสบการณ์ที่ดี แม้เมื่อเกิดปัญหา
