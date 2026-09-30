# Part 17: HTTP Interceptors

## บทนำ

HTTP Interceptors คือ Middleware สำหรับ HttpClient ที่ช่วยให้เราดักจับ HTTP Requests และ Responses เพื่อเพิ่ม Logic พิเศษ เช่น การเพิ่ม Auth Token การจัดการ Error หรือการแสดง Loading State โดยไม่ต้องแก้ไขโค้ดในทุก Service

---

## 1. Interceptor คืออะไร

### Flow ของ HTTP Request

```
Component
    ↓ HttpClient.get('/api/products')
Interceptor 1 (Auth Token)
    ↓ เพิ่ม Authorization header
Interceptor 2 (Loading)
    ↓ แสดง Loading spinner
Interceptor 3 (Logger)
    ↓ บันทึก request log
HTTP Request → Server
    ↑
HTTP Response ← Server
    ↑ response กลับผ่าน interceptors เรียงกลับ
Interceptor 3 (Logger)
    ↑ บันทึก response log
Interceptor 2 (Loading)
    ↑ ซ่อน Loading spinner
Interceptor 1 (Auth Token)
    ↑ จัดการ 401 error
Component
```

### Class-based Interceptor (Angular 14-)

```typescript
// core/interceptors/auth.interceptor.ts
import { Injectable } from '@angular/core';
import {
  HttpInterceptor,
  HttpRequest,
  HttpHandler,
  HttpEvent,
} from '@angular/common/http';
import { Observable } from 'rxjs';
import { AuthService } from '../services/auth.service';

@Injectable()
export class AuthInterceptor implements HttpInterceptor {
  constructor(private authService: AuthService) {}

  intercept(
    req: HttpRequest<any>,
    next: HttpHandler
  ): Observable<HttpEvent<any>> {
    const token = this.authService.getAccessToken();

    if (token) {
      const authReq = req.clone({
        setHeaders: { Authorization: `Bearer ${token}` },
      });
      return next.handle(authReq);
    }

    return next.handle(req);
  }
}
```

### ลงทะเบียน Interceptors

```typescript
// app.module.ts
import { HTTP_INTERCEPTORS } from '@angular/common/http';

@NgModule({
  providers: [
    {
      provide: HTTP_INTERCEPTORS,
      useClass: AuthInterceptor,
      multi: true,    // multi: true สำคัญมาก ถ้าไม่ใส่จะแทนที่ interceptor อื่น
    },
    {
      provide: HTTP_INTERCEPTORS,
      useClass: ErrorInterceptor,
      multi: true,
    },
    {
      provide: HTTP_INTERCEPTORS,
      useClass: LoadingInterceptor,
      multi: true,
    },
  ],
})
export class AppModule {}
```

---

## 2. Authentication Token Interceptor

### Interceptor พร้อม Token Refresh

```typescript
// core/interceptors/auth.interceptor.ts
import { Injectable } from '@angular/core';
import {
  HttpInterceptor,
  HttpRequest,
  HttpHandler,
  HttpEvent,
  HttpErrorResponse,
} from '@angular/common/http';
import {
  Observable,
  throwError,
  BehaviorSubject,
  filter,
  take,
  switchMap,
  catchError,
} from 'rxjs';
import { AuthService } from '../services/auth.service';
import { TokenService } from '../services/token.service';

@Injectable()
export class AuthInterceptor implements HttpInterceptor {
  private isRefreshing = false;
  private refreshTokenSubject = new BehaviorSubject<string | null>(null);

  constructor(
    private authService: AuthService,
    private tokenService: TokenService
  ) {}

  intercept(
    req: HttpRequest<any>,
    next: HttpHandler
  ): Observable<HttpEvent<any>> {
    // ไม่เพิ่ม token สำหรับ auth endpoints
    if (this.isAuthEndpoint(req.url)) {
      return next.handle(req);
    }

    const token = this.tokenService.getAccessToken();
    const authReq = token ? this.addToken(req, token) : req;

    return next.handle(authReq).pipe(
      catchError(error => {
        if (error instanceof HttpErrorResponse && error.status === 401) {
          return this.handle401Error(req, next);
        }
        return throwError(() => error);
      })
    );
  }

  private addToken(req: HttpRequest<any>, token: string): HttpRequest<any> {
    return req.clone({
      setHeaders: {
        Authorization: `Bearer ${token}`,
      },
    });
  }

  private isAuthEndpoint(url: string): boolean {
    const authPaths = ['/api/auth/login', '/api/auth/register', '/api/auth/refresh'];
    return authPaths.some(path => url.includes(path));
  }

  private handle401Error(
    req: HttpRequest<any>,
    next: HttpHandler
  ): Observable<HttpEvent<any>> {
    if (this.isRefreshing) {
      // รอ token ใหม่จาก refresh ที่กำลังทำอยู่
      return this.refreshTokenSubject.pipe(
        filter(token => token !== null),
        take(1),
        switchMap(token => next.handle(this.addToken(req, token!)))
      );
    }

    this.isRefreshing = true;
    this.refreshTokenSubject.next(null);

    return this.authService.refreshToken().pipe(
      switchMap(response => {
        this.isRefreshing = false;
        this.refreshTokenSubject.next(response.accessToken);
        return next.handle(this.addToken(req, response.accessToken));
      }),
      catchError(error => {
        this.isRefreshing = false;
        // Refresh token หมดอายุ - logout
        this.authService.logout();
        return throwError(() => error);
      })
    );
  }
}
```

### Functional Auth Interceptor (Angular 15+)

```typescript
// core/interceptors/auth.interceptor.ts (functional style)
import { HttpInterceptorFn, HttpErrorResponse } from '@angular/common/http';
import { inject } from '@angular/core';
import { catchError, switchMap, throwError } from 'rxjs';
import { AuthService } from '../services/auth.service';
import { TokenService } from '../services/token.service';

export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const tokenService = inject(TokenService);
  const authService = inject(AuthService);

  const token = tokenService.getAccessToken();

  // ข้าม auth endpoints
  if (req.url.includes('/auth/')) {
    return next(req);
  }

  const authReq = token
    ? req.clone({ setHeaders: { Authorization: `Bearer ${token}` } })
    : req;

  return next(authReq).pipe(
    catchError(error => {
      if (error instanceof HttpErrorResponse && error.status === 401) {
        return authService.refreshToken().pipe(
          switchMap(response => {
            const newToken = response.accessToken;
            const retryReq = req.clone({
              setHeaders: { Authorization: `Bearer ${newToken}` },
            });
            return next(retryReq);
          }),
          catchError(() => {
            authService.logout();
            return throwError(() => error);
          })
        );
      }
      return throwError(() => error);
    })
  );
};
```

### ลงทะเบียน Functional Interceptors

```typescript
// main.ts (Standalone App)
import { bootstrapApplication } from '@angular/platform-browser';
import { provideHttpClient, withInterceptors } from '@angular/common/http';
import { AppComponent } from './app/app.component';
import { authInterceptor } from './app/core/interceptors/auth.interceptor';
import { errorInterceptor } from './app/core/interceptors/error.interceptor';
import { loadingInterceptor } from './app/core/interceptors/loading.interceptor';

bootstrapApplication(AppComponent, {
  providers: [
    provideHttpClient(
      withInterceptors([
        authInterceptor,
        errorInterceptor,
        loadingInterceptor,
      ])
    ),
  ],
});
```

---

## 3. Error Handling Interceptor

```typescript
// core/interceptors/error.interceptor.ts
import { Injectable } from '@angular/core';
import {
  HttpInterceptor,
  HttpRequest,
  HttpHandler,
  HttpEvent,
  HttpErrorResponse,
} from '@angular/common/http';
import { Observable, throwError } from 'rxjs';
import { catchError } from 'rxjs/operators';
import { Router } from '@angular/router';
import { NotificationService } from '../services/notification.service';
import { LoggerService } from '../services/logger.service';

export interface ApiError {
  status: number;
  message: string;
  errors?: Record<string, string[]>;
  timestamp?: string;
}

@Injectable()
export class ErrorInterceptor implements HttpInterceptor {
  constructor(
    private router: Router,
    private notification: NotificationService,
    private logger: LoggerService
  ) {}

  intercept(
    req: HttpRequest<any>,
    next: HttpHandler
  ): Observable<HttpEvent<any>> {
    return next.handle(req).pipe(
      catchError((error: HttpErrorResponse) => {
        const apiError = this.parseError(error);
        this.handleError(apiError);
        return throwError(() => apiError);
      })
    );
  }

  private parseError(error: HttpErrorResponse): ApiError {
    if (error.error instanceof ErrorEvent) {
      // Client-side error (Network error, timeout)
      return {
        status: 0,
        message: 'ไม่สามารถเชื่อมต่อกับเซิร์ฟเวอร์ได้ กรุณาตรวจสอบการเชื่อมต่ออินเทอร์เน็ต',
      };
    }

    // Server-side error
    return {
      status: error.status,
      message: error.error?.message || this.getDefaultMessage(error.status),
      errors: error.error?.errors,
      timestamp: new Date().toISOString(),
    };
  }

  private getDefaultMessage(status: number): string {
    const messages: Record<number, string> = {
      400: 'ข้อมูลที่ส่งไปไม่ถูกต้อง',
      401: 'กรุณาเข้าสู่ระบบ',
      403: 'คุณไม่มีสิทธิ์เข้าถึง',
      404: 'ไม่พบข้อมูลที่ต้องการ',
      408: 'การเชื่อมต่อหมดเวลา',
      409: 'ข้อมูลซ้ำกัน',
      422: 'ข้อมูลที่ส่งไปไม่ผ่านการตรวจสอบ',
      429: 'คำขอมากเกินไป กรุณารอสักครู่',
      500: 'เกิดข้อผิดพลาดในเซิร์ฟเวอร์',
      502: 'เซิร์ฟเวอร์ไม่ตอบสนอง',
      503: 'บริการไม่พร้อมให้บริการชั่วคราว',
    };
    return messages[status] || 'เกิดข้อผิดพลาด กรุณาลองใหม่อีกครั้ง';
  }

  private handleError(error: ApiError): void {
    // Log error
    this.logger.error('HTTP Error', error);

    switch (error.status) {
      case 401:
        // AuthInterceptor จัดการแล้ว
        break;
      case 403:
        this.router.navigate(['/forbidden']);
        break;
      case 404:
        // ไม่แสดง notification สำหรับ 404 บาง endpoint
        break;
      case 429:
        this.notification.warning('คำขอมากเกินไป กรุณารอ 1 นาทีแล้วลองใหม่');
        break;
      case 0:
        this.notification.error('ไม่สามารถเชื่อมต่อได้ กรุณาตรวจสอบอินเทอร์เน็ต');
        break;
      default:
        if (error.status >= 500) {
          this.notification.error('เกิดข้อผิดพลาดในระบบ กรุณาลองใหม่อีกครั้ง');
        }
    }
  }
}
```

---

## 4. Loading State Interceptor

```typescript
// core/services/loading.service.ts
import { Injectable, signal, computed } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class LoadingService {
  private activeRequests = signal(0);

  // Computed: true ถ้ามี request ที่กำลัง active อยู่
  isLoading = computed(() => this.activeRequests() > 0);

  // สำหรับแสดงจำนวน request
  requestCount = computed(() => this.activeRequests());

  startLoading(): void {
    this.activeRequests.update(count => count + 1);
  }

  stopLoading(): void {
    this.activeRequests.update(count => Math.max(0, count - 1));
  }
}
```

```typescript
// core/interceptors/loading.interceptor.ts
import { Injectable } from '@angular/core';
import {
  HttpInterceptor,
  HttpRequest,
  HttpHandler,
  HttpEvent,
} from '@angular/common/http';
import { Observable } from 'rxjs';
import { finalize } from 'rxjs/operators';
import { LoadingService } from '../services/loading.service';

@Injectable()
export class LoadingInterceptor implements HttpInterceptor {
  // URLs ที่ไม่ต้องแสดง loading (เช่น background polling)
  private skipUrls = [
    '/api/notifications/poll',
    '/api/health',
  ];

  constructor(private loadingService: LoadingService) {}

  intercept(
    req: HttpRequest<any>,
    next: HttpHandler
  ): Observable<HttpEvent<any>> {
    // ข้าม URLs ที่กำหนด
    if (this.shouldSkip(req)) {
      return next.handle(req);
    }

    this.loadingService.startLoading();

    return next.handle(req).pipe(
      finalize(() => {
        this.loadingService.stopLoading();
      })
    );
  }

  private shouldSkip(req: HttpRequest<any>): boolean {
    return (
      this.skipUrls.some(url => req.url.includes(url)) ||
      req.headers.has('X-Skip-Loading')
    );
  }
}
```

### Global Loading Component

```typescript
// shared/components/global-loading/global-loading.component.ts
import { Component } from '@angular/core';
import { CommonModule } from '@angular/common';
import { LoadingService } from '../../../core/services/loading.service';

@Component({
  selector: 'app-global-loading',
  standalone: true,
  imports: [CommonModule],
  template: `
    <!-- Top Progress Bar -->
    <div class="loading-bar" *ngIf="loadingService.isLoading()">
      <div class="progress-bar"></div>
    </div>

    <!-- Full Page Overlay (optional) -->
    <!-- <div class="loading-overlay" *ngIf="loadingService.isLoading()">
      <div class="spinner"></div>
    </div> -->
  `,
  styles: [`
    .loading-bar {
      position: fixed;
      top: 0;
      left: 0;
      right: 0;
      z-index: 9999;
      height: 3px;
    }
    .progress-bar {
      height: 100%;
      background: linear-gradient(90deg, #007bff, #00d4ff);
      animation: loading 1.5s ease-in-out infinite;
    }
    @keyframes loading {
      0% { width: 0%; margin-left: 0; }
      50% { width: 70%; margin-left: 0; }
      100% { width: 10%; margin-left: 90%; }
    }
  `],
})
export class GlobalLoadingComponent {
  constructor(public loadingService: LoadingService) {}
}
```

---

## 5. Retry Interceptor

```typescript
// core/interceptors/retry.interceptor.ts
import { Injectable } from '@angular/core';
import {
  HttpInterceptor,
  HttpRequest,
  HttpHandler,
  HttpEvent,
  HttpErrorResponse,
} from '@angular/common/http';
import { Observable, throwError, timer } from 'rxjs';
import { catchError, retry, switchMap } from 'rxjs/operators';

@Injectable()
export class RetryInterceptor implements HttpInterceptor {
  private readonly MAX_RETRIES = 3;
  private readonly RETRY_DELAY = 1000; // 1 วินาที
  private readonly RETRYABLE_STATUSES = [408, 429, 502, 503, 504];

  intercept(
    req: HttpRequest<any>,
    next: HttpHandler
  ): Observable<HttpEvent<any>> {
    // ไม่ retry สำหรับ POST/PUT/DELETE (ป้องกัน duplicate)
    if (!this.isRetryable(req)) {
      return next.handle(req);
    }

    return next.handle(req).pipe(
      catchError((error: HttpErrorResponse) => {
        if (this.shouldRetry(error)) {
          return this.retryWithBackoff(req, next, 1);
        }
        return throwError(() => error);
      })
    );
  }

  private retryWithBackoff(
    req: HttpRequest<any>,
    next: HttpHandler,
    attempt: number
  ): Observable<HttpEvent<any>> {
    const delay = this.RETRY_DELAY * Math.pow(2, attempt - 1); // Exponential backoff

    return timer(delay).pipe(
      switchMap(() => next.handle(req)),
      catchError((error: HttpErrorResponse) => {
        if (attempt < this.MAX_RETRIES && this.shouldRetry(error)) {
          console.log(`Retry attempt ${attempt + 1}/${this.MAX_RETRIES} after ${delay}ms`);
          return this.retryWithBackoff(req, next, attempt + 1);
        }
        return throwError(() => error);
      })
    );
  }

  private isRetryable(req: HttpRequest<any>): boolean {
    return req.method === 'GET';
  }

  private shouldRetry(error: HttpErrorResponse): boolean {
    return (
      error.status === 0 || // Network error
      this.RETRYABLE_STATUSES.includes(error.status)
    );
  }
}
```

---

## 6. Caching Interceptor

```typescript
// core/services/http-cache.service.ts
import { Injectable } from '@angular/core';
import { HttpResponse } from '@angular/common/http';

interface CacheEntry {
  response: HttpResponse<any>;
  expiry: number;
}

@Injectable({ providedIn: 'root' })
export class HttpCacheService {
  private cache = new Map<string, CacheEntry>();
  private readonly DEFAULT_TTL = 5 * 60 * 1000; // 5 นาที

  get(key: string): HttpResponse<any> | null {
    const entry = this.cache.get(key);
    if (!entry) return null;

    if (Date.now() > entry.expiry) {
      this.cache.delete(key);
      return null;
    }

    return entry.response;
  }

  set(key: string, response: HttpResponse<any>, ttl = this.DEFAULT_TTL): void {
    this.cache.set(key, {
      response,
      expiry: Date.now() + ttl,
    });
  }

  invalidate(pattern: string): void {
    // ลบ cache ที่ key ตรงกับ pattern
    for (const key of this.cache.keys()) {
      if (key.includes(pattern)) {
        this.cache.delete(key);
      }
    }
  }

  clear(): void {
    this.cache.clear();
  }
}
```

```typescript
// core/interceptors/cache.interceptor.ts
import { Injectable } from '@angular/core';
import {
  HttpInterceptor,
  HttpRequest,
  HttpHandler,
  HttpEvent,
  HttpResponse,
} from '@angular/common/http';
import { Observable, of } from 'rxjs';
import { tap } from 'rxjs/operators';
import { HttpCacheService } from '../services/http-cache.service';

@Injectable()
export class CacheInterceptor implements HttpInterceptor {
  constructor(private cacheService: HttpCacheService) {}

  intercept(
    req: HttpRequest<any>,
    next: HttpHandler
  ): Observable<HttpEvent<any>> {
    // Cache เฉพาะ GET requests
    if (req.method !== 'GET') {
      // Mutation: invalidate related cache
      this.invalidateRelatedCache(req);
      return next.handle(req);
    }

    // ตรวจสอบว่ามี cache header หรือเปล่า
    if (req.headers.has('X-No-Cache')) {
      return next.handle(req);
    }

    const cacheKey = this.getCacheKey(req);
    const cachedResponse = this.cacheService.get(cacheKey);

    if (cachedResponse) {
      console.log(`Cache HIT: ${req.url}`);
      return of(cachedResponse.clone());
    }

    console.log(`Cache MISS: ${req.url}`);

    // กำหนด TTL จาก header หรือใช้ default
    const ttl = this.getTTL(req);

    return next.handle(req).pipe(
      tap(event => {
        if (event instanceof HttpResponse && event.status === 200) {
          this.cacheService.set(cacheKey, event.clone(), ttl);
        }
      })
    );
  }

  private getCacheKey(req: HttpRequest<any>): string {
    return `${req.method}:${req.urlWithParams}`;
  }

  private getTTL(req: HttpRequest<any>): number {
    const ttlHeader = req.headers.get('X-Cache-TTL');
    return ttlHeader ? parseInt(ttlHeader, 10) * 1000 : 5 * 60 * 1000;
  }

  private invalidateRelatedCache(req: HttpRequest<any>): void {
    // Parse URL เพื่อหา base resource
    const url = new URL(req.url, window.location.origin);
    const pathParts = url.pathname.split('/').filter(Boolean);

    if (pathParts.length >= 2) {
      // invalidate /api/products เมื่อ POST/PUT/DELETE /api/products
      const resourcePath = `/${pathParts.slice(0, 2).join('/')}`;
      this.cacheService.invalidate(resourcePath);
    }
  }
}
```

---

## 7. Workshop: Full Interceptor Stack

### โครงสร้าง Interceptors

```
core/interceptors/
├── auth.interceptor.ts       # เพิ่ม Auth Token
├── error.interceptor.ts      # จัดการ Errors
├── loading.interceptor.ts    # แสดง Loading State
├── retry.interceptor.ts      # Retry requests ที่ fail
├── cache.interceptor.ts      # Cache GET requests
├── logger.interceptor.ts     # Log requests/responses
└── timeout.interceptor.ts    # Timeout requests
```

### Logger Interceptor

```typescript
// core/interceptors/logger.interceptor.ts
import { Injectable } from '@angular/core';
import {
  HttpInterceptor,
  HttpRequest,
  HttpHandler,
  HttpEvent,
  HttpResponse,
  HttpErrorResponse,
} from '@angular/common/http';
import { Observable } from 'rxjs';
import { tap, finalize } from 'rxjs/operators';
import { environment } from '../../../environments/environment';

@Injectable()
export class LoggerInterceptor implements HttpInterceptor {
  intercept(
    req: HttpRequest<any>,
    next: HttpHandler
  ): Observable<HttpEvent<any>> {
    if (environment.production) {
      return next.handle(req);
    }

    const startTime = Date.now();
    const requestId = this.generateRequestId();

    console.group(`🌐 HTTP ${req.method} ${req.url} [${requestId}]`);
    console.log('Request Headers:', req.headers.keys().reduce((acc, key) => {
      acc[key] = req.headers.get(key);
      return acc;
    }, {} as Record<string, any>));

    if (req.body) {
      console.log('Request Body:', req.body);
    }

    return next.handle(req).pipe(
      tap({
        next: (event) => {
          if (event instanceof HttpResponse) {
            const elapsed = Date.now() - startTime;
            console.log(`✅ Response ${event.status} (${elapsed}ms)`);
            console.log('Response Body:', event.body);
          }
        },
        error: (error: HttpErrorResponse) => {
          const elapsed = Date.now() - startTime;
          console.error(`❌ Error ${error.status} (${elapsed}ms):`, error.message);
        },
      }),
      finalize(() => {
        console.groupEnd();
      })
    );
  }

  private generateRequestId(): string {
    return Math.random().toString(36).substring(2, 8).toUpperCase();
  }
}
```

### Timeout Interceptor

```typescript
// core/interceptors/timeout.interceptor.ts
import { Injectable } from '@angular/core';
import {
  HttpInterceptor,
  HttpRequest,
  HttpHandler,
  HttpEvent,
} from '@angular/common/http';
import { Observable, timeout, TimeoutError, throwError } from 'rxjs';
import { catchError } from 'rxjs/operators';

@Injectable()
export class TimeoutInterceptor implements HttpInterceptor {
  private readonly DEFAULT_TIMEOUT = 30000; // 30 วินาที

  intercept(
    req: HttpRequest<any>,
    next: HttpHandler
  ): Observable<HttpEvent<any>> {
    const timeoutValue =
      Number(req.headers.get('X-Timeout')) || this.DEFAULT_TIMEOUT;

    return next.handle(req).pipe(
      timeout(timeoutValue),
      catchError(error => {
        if (error instanceof TimeoutError) {
          return throwError(() => ({
            status: 408,
            message: `Request timeout after ${timeoutValue}ms`,
          }));
        }
        return throwError(() => error);
      })
    );
  }
}
```

### ลงทะเบียนทั้งหมดใน AppModule

```typescript
// app.module.ts
import { NgModule } from '@angular/core';
import { BrowserModule } from '@angular/platform-browser';
import { HttpClientModule, HTTP_INTERCEPTORS } from '@angular/common/http';
import { AppRoutingModule } from './app-routing.module';
import { AppComponent } from './app.component';

// Interceptors (ลำดับสำคัญ!)
import { LoggerInterceptor } from './core/interceptors/logger.interceptor';
import { TimeoutInterceptor } from './core/interceptors/timeout.interceptor';
import { AuthInterceptor } from './core/interceptors/auth.interceptor';
import { LoadingInterceptor } from './core/interceptors/loading.interceptor';
import { CacheInterceptor } from './core/interceptors/cache.interceptor';
import { RetryInterceptor } from './core/interceptors/retry.interceptor';
import { ErrorInterceptor } from './core/interceptors/error.interceptor';

@NgModule({
  declarations: [AppComponent],
  imports: [
    BrowserModule,
    HttpClientModule,
    AppRoutingModule,
  ],
  providers: [
    // ลำดับ Interceptors มีความสำคัญ!
    // Request: บนลงล่าง | Response: ล่างขึ้นบน
    {
      provide: HTTP_INTERCEPTORS,
      useClass: LoggerInterceptor,      // 1. Log ก่อน
      multi: true,
    },
    {
      provide: HTTP_INTERCEPTORS,
      useClass: TimeoutInterceptor,     // 2. กำหนด Timeout
      multi: true,
    },
    {
      provide: HTTP_INTERCEPTORS,
      useClass: AuthInterceptor,        // 3. เพิ่ม Auth Token
      multi: true,
    },
    {
      provide: HTTP_INTERCEPTORS,
      useClass: LoadingInterceptor,     // 4. แสดง Loading
      multi: true,
    },
    {
      provide: HTTP_INTERCEPTORS,
      useClass: CacheInterceptor,       // 5. ตรวจสอบ Cache
      multi: true,
    },
    {
      provide: HTTP_INTERCEPTORS,
      useClass: RetryInterceptor,       // 6. Retry
      multi: true,
    },
    {
      provide: HTTP_INTERCEPTORS,
      useClass: ErrorInterceptor,       // 7. Handle Errors
      multi: true,
    },
  ],
  bootstrap: [AppComponent],
})
export class AppModule {}
```

### ใช้ Cache Control ใน Service

```typescript
// features/products/services/product.service.ts
import { Injectable } from '@angular/core';
import { HttpClient, HttpHeaders } from '@angular/common/http';
import { Observable } from 'rxjs';
import { Product } from '../models/product.model';

@Injectable({ providedIn: 'root' })
export class ProductService {
  private readonly apiUrl = '/api/products';

  constructor(private http: HttpClient) {}

  // GET พร้อม Cache (5 นาที)
  getAll(): Observable<Product[]> {
    return this.http.get<Product[]>(this.apiUrl);
  }

  // GET โดยไม่ใช้ Cache
  getAllFresh(): Observable<Product[]> {
    return this.http.get<Product[]>(this.apiUrl, {
      headers: new HttpHeaders({ 'X-No-Cache': 'true' }),
    });
  }

  // GET พร้อม Custom TTL (10 นาที)
  getCategories(): Observable<string[]> {
    return this.http.get<string[]>('/api/categories', {
      headers: new HttpHeaders({ 'X-Cache-TTL': '600' }),
    });
  }

  // GET พร้อม Custom Timeout (60 วินาที)
  generateReport(): Observable<Blob> {
    return this.http.get('/api/reports/generate', {
      responseType: 'blob',
      headers: new HttpHeaders({ 'X-Timeout': '60000' }),
    });
  }

  // POST - ไม่ Cache, invalidate related cache
  create(product: Partial<Product>): Observable<Product> {
    return this.http.post<Product>(this.apiUrl, product);
  }

  // PUT
  update(id: string, product: Partial<Product>): Observable<Product> {
    return this.http.put<Product>(`${this.apiUrl}/${id}`, product);
  }

  // DELETE
  delete(id: string): Observable<void> {
    return this.http.delete<void>(`${this.apiUrl}/${id}`);
  }
}
```

---

## สรุป

| Interceptor | หน้าที่ |
|-------------|---------|
| **Auth** | เพิ่ม Bearer token, refresh token อัตโนมัติ |
| **Error** | แปลง error เป็น user-friendly message, navigate |
| **Loading** | แสดง/ซ่อน global loading indicator |
| **Retry** | retry requests ที่ fail โดยอัตโนมัติ (exponential backoff) |
| **Cache** | cache GET responses, invalidate เมื่อ mutate |
| **Logger** | log requests/responses ใน development |
| **Timeout** | ยกเลิก requests ที่ใช้เวลานานเกินไป |

### Best Practices

1. **ลำดับ Interceptors สำคัญ** — บนลงล่างสำหรับ Request, ล่างขึ้นบนสำหรับ Response
2. **แยก Interceptor ตามหน้าที่** อย่ารวมทุกอย่างใน Interceptor เดียว
3. **ใช้ `finalize`** เพื่อให้มั่นใจว่า cleanup เกิดขึ้นเสมอ
4. **Retry เฉพาะ GET** เพื่อป้องกัน duplicate mutations
5. **Cache เฉพาะ GET** และ invalidate เมื่อมีการแก้ไขข้อมูล
6. **Token Refresh** ต้องจัดการ race condition (หลาย requests refresh พร้อมกัน)
