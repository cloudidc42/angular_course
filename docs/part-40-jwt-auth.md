# Part 40: JWT Authentication ใน Angular

## บทนำ

JWT (JSON Web Token) เป็น standard สำหรับการส่งข้อมูลระหว่าง client และ server อย่างปลอดภัย ในบทนี้เราจะเรียนรู้การ decode JWT, จัดการ refresh token และสร้าง HttpInterceptor สำหรับ auth

---

## 1. JWT Structure และการ Decode

JWT ประกอบด้วย 3 ส่วนคั่นด้วย `.`:
- **Header**: algorithm และ type
- **Payload**: claims (ข้อมูล)
- **Signature**: การยืนยันความถูกต้อง

```typescript
// app/utils/jwt.utils.ts

export interface JwtPayload {
  sub: string;        // Subject (user ID)
  email: string;
  name: string;
  roles: string[];
  iat: number;        // Issued At
  exp: number;        // Expiration
  jti?: string;       // JWT ID
}

export class JwtUtils {
  // Decode JWT โดยไม่ verify signature (ทำฝั่ง client เพื่ออ่านข้อมูลเท่านั้น)
  static decode(token: string): JwtPayload | null {
    try {
      const parts = token.split('.');
      if (parts.length !== 3) return null;

      // Decode Base64URL
      const payload = parts[1]
        .replace(/-/g, '+')
        .replace(/_/g, '/');

      // เติม padding ถ้าจำเป็น
      const padded = payload.padEnd(
        payload.length + (4 - payload.length % 4) % 4,
        '='
      );

      return JSON.parse(atob(padded)) as JwtPayload;
    } catch {
      return null;
    }
  }

  static isExpired(token: string): boolean {
    const payload = this.decode(token);
    if (!payload) return true;

    // เพิ่ม 30 วินาที buffer เพื่อป้องกัน token หมดอายุระหว่างส่ง request
    const bufferSeconds = 30;
    return Date.now() >= (payload.exp - bufferSeconds) * 1000;
  }

  static getExpiryDate(token: string): Date | null {
    const payload = this.decode(token);
    if (!payload) return null;
    return new Date(payload.exp * 1000);
  }

  static getTimeUntilExpiry(token: string): number {
    const payload = this.decode(token);
    if (!payload) return 0;
    return Math.max(0, payload.exp * 1000 - Date.now());
  }

  static getRoles(token: string): string[] {
    return this.decode(token)?.roles ?? [];
  }
}
```

---

## 2. Refresh Token Service

```typescript
// app/services/refresh-token.service.ts
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable, throwError, BehaviorSubject } from 'rxjs';
import { tap, catchError, filter, take, switchMap } from 'rxjs/operators';
import { TokenService } from './token.service';

interface RefreshResponse {
  accessToken: string;
  refreshToken: string;
}

@Injectable({ providedIn: 'root' })
export class RefreshTokenService {
  private readonly API_URL = '/api/auth';

  // ป้องกัน multiple refresh calls พร้อมกัน
  private isRefreshing = false;
  private refreshTokenSubject = new BehaviorSubject<string | null>(null);

  constructor(
    private http: HttpClient,
    private tokenService: TokenService
  ) {}

  refresh(): Observable<string> {
    if (this.isRefreshing) {
      // รอ refresh ที่กำลังทำอยู่
      return this.refreshTokenSubject.pipe(
        filter(token => token !== null),
        take(1),
        switchMap(token => new Observable<string>(obs => {
          obs.next(token!);
          obs.complete();
        }))
      );
    }

    this.isRefreshing = true;
    this.refreshTokenSubject.next(null);

    const refreshToken = this.tokenService.getRefreshToken();
    if (!refreshToken) {
      this.isRefreshing = false;
      return throwError(() => new Error('No refresh token'));
    }

    return this.http.post<RefreshResponse>(`${this.API_URL}/refresh`, { refreshToken }).pipe(
      tap(response => {
        this.tokenService.setAccessToken(response.accessToken);
        this.tokenService.setRefreshToken(response.refreshToken);
        this.refreshTokenSubject.next(response.accessToken);
        this.isRefreshing = false;
      }),
      switchMap(response => new Observable<string>(obs => {
        obs.next(response.accessToken);
        obs.complete();
      })),
      catchError(error => {
        this.isRefreshing = false;
        this.refreshTokenSubject.next(null);
        return throwError(() => error);
      })
    );
  }
}
```

---

## 3. Auth HttpInterceptor

```typescript
// app/interceptors/auth.interceptor.ts
import { inject } from '@angular/core';
import {
  HttpInterceptorFn,
  HttpRequest,
  HttpHandlerFn,
  HttpEvent,
  HttpErrorResponse
} from '@angular/common/http';
import { Observable, throwError } from 'rxjs';
import { catchError, switchMap } from 'rxjs/operators';
import { TokenService } from '../services/token.service';
import { RefreshTokenService } from '../services/refresh-token.service';
import { AuthService } from '../services/auth.service';

// URLs ที่ไม่ต้องใส่ token
const PUBLIC_URLS = ['/api/auth/login', '/api/auth/register', '/api/auth/refresh'];

export const authInterceptor: HttpInterceptorFn = (
  req: HttpRequest<any>,
  next: HttpHandlerFn
): Observable<HttpEvent<any>> => {
  const tokenService = inject(TokenService);
  const refreshTokenService = inject(RefreshTokenService);
  const authService = inject(AuthService);

  // ข้าม public URLs
  if (PUBLIC_URLS.some(url => req.url.includes(url))) {
    return next(req);
  }

  const token = tokenService.getAccessToken();

  // เพิ่ม token ถ้ามี
  const authReq = token ? addTokenToRequest(req, token) : req;

  return next(authReq).pipe(
    catchError((error: HttpErrorResponse) => {
      if (error.status === 401) {
        return handleUnauthorized(req, next, tokenService, refreshTokenService, authService);
      }
      return throwError(() => error);
    })
  );
};

function addTokenToRequest(req: HttpRequest<any>, token: string): HttpRequest<any> {
  return req.clone({
    headers: req.headers.set('Authorization', `Bearer ${token}`)
  });
}

function handleUnauthorized(
  req: HttpRequest<any>,
  next: HttpHandlerFn,
  tokenService: TokenService,
  refreshTokenService: RefreshTokenService,
  authService: AuthService
): Observable<HttpEvent<any>> {
  return refreshTokenService.refresh().pipe(
    switchMap(newToken => {
      const authReq = addTokenToRequest(req, newToken);
      return next(authReq);
    }),
    catchError(refreshError => {
      // Refresh token หมดอายุหรือ invalid - logout
      authService.logout();
      return throwError(() => refreshError);
    })
  );
}
```

### Functional Interceptor (Angular 15+)

```typescript
// app/app.config.ts
import { ApplicationConfig } from '@angular/core';
import { provideRouter } from '@angular/router';
import { provideHttpClient, withInterceptors } from '@angular/common/http';
import { authInterceptor } from './interceptors/auth.interceptor';
import { loggingInterceptor } from './interceptors/logging.interceptor';
import { errorInterceptor } from './interceptors/error.interceptor';
import { routes } from './app-routing.module';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes),
    provideHttpClient(
      withInterceptors([
        authInterceptor,
        loggingInterceptor,
        errorInterceptor
      ])
    )
  ]
};
```

---

## 4. Logging Interceptor

```typescript
// app/interceptors/logging.interceptor.ts
import { inject } from '@angular/core';
import {
  HttpInterceptorFn,
  HttpRequest,
  HttpHandlerFn,
  HttpEvent,
  HttpResponse
} from '@angular/common/http';
import { Observable } from 'rxjs';
import { tap, finalize } from 'rxjs/operators';

export const loggingInterceptor: HttpInterceptorFn = (
  req: HttpRequest<any>,
  next: HttpHandlerFn
): Observable<HttpEvent<any>> => {
  const startTime = Date.now();
  const reqId = Math.random().toString(36).substr(2, 9);

  console.group(`[HTTP ${reqId}] ${req.method} ${req.url}`);
  console.log('Request:', { headers: req.headers.keys(), body: req.body });

  return next(req).pipe(
    tap({
      next: event => {
        if (event instanceof HttpResponse) {
          console.log('Response:', { status: event.status, body: event.body });
        }
      },
      error: err => {
        console.error('Error:', err);
      }
    }),
    finalize(() => {
      const elapsed = Date.now() - startTime;
      console.log(`Duration: ${elapsed}ms`);
      console.groupEnd();
    })
  );
};
```

---

## 5. Error Interceptor

```typescript
// app/interceptors/error.interceptor.ts
import { inject } from '@angular/core';
import {
  HttpInterceptorFn,
  HttpRequest,
  HttpHandlerFn,
  HttpEvent,
  HttpErrorResponse
} from '@angular/common/http';
import { Observable, throwError } from 'rxjs';
import { catchError, retry } from 'rxjs/operators';
import { Router } from '@angular/router';

interface ApiError {
  message: string;
  code: string;
  details?: any;
}

export const errorInterceptor: HttpInterceptorFn = (
  req: HttpRequest<any>,
  next: HttpHandlerFn
): Observable<HttpEvent<any>> => {
  const router = inject(Router);

  return next(req).pipe(
    // retry GET requests เมื่อเกิด network error
    req.method === 'GET' ? retry({ count: 2, delay: 1000 }) : (obs) => obs,
    catchError((error: HttpErrorResponse) => {
      const apiError = parseError(error);

      switch (error.status) {
        case 0:
          console.error('Network error - ไม่สามารถเชื่อมต่อ server ได้');
          break;

        case 400:
          console.error('Bad Request:', apiError.message);
          break;

        case 403:
          console.error('Forbidden - ไม่มีสิทธิ์เข้าถึง');
          router.navigate(['/403']);
          break;

        case 404:
          console.error('Not Found:', req.url);
          break;

        case 422:
          console.error('Validation Error:', apiError.details);
          break;

        case 500:
          console.error('Server Error:', apiError.message);
          router.navigate(['/500']);
          break;
      }

      return throwError(() => apiError);
    })
  );
};

function parseError(error: HttpErrorResponse): ApiError {
  if (error.error instanceof ErrorEvent) {
    // Client-side error
    return { message: error.error.message, code: 'CLIENT_ERROR' };
  }

  // Server-side error
  return {
    message: error.error?.message || `HTTP Error ${error.status}`,
    code: error.error?.code || `HTTP_${error.status}`,
    details: error.error?.details
  };
}
```

---

## 6. JWT Timer (Auto Refresh)

```typescript
// app/services/jwt-timer.service.ts
import { Injectable, OnDestroy } from '@angular/core';
import { Subject, timer, Subscription } from 'rxjs';
import { takeUntil, switchMap } from 'rxjs/operators';
import { TokenService } from './token.service';
import { RefreshTokenService } from './refresh-token.service';
import { JwtUtils } from '../utils/jwt.utils';

@Injectable({ providedIn: 'root' })
export class JwtTimerService implements OnDestroy {
  private destroy$ = new Subject<void>();
  private timerSubscription?: Subscription;

  constructor(
    private tokenService: TokenService,
    private refreshTokenService: RefreshTokenService
  ) {}

  startTimer() {
    this.stopTimer();

    const token = this.tokenService.getAccessToken();
    if (!token) return;

    // refresh token 5 นาทีก่อนหมดอายุ
    const timeUntilExpiry = JwtUtils.getTimeUntilExpiry(token);
    const refreshIn = Math.max(0, timeUntilExpiry - 5 * 60 * 1000);

    console.log(`JWT จะ refresh ใน ${Math.round(refreshIn / 1000)} วินาที`);

    this.timerSubscription = timer(refreshIn)
      .pipe(
        switchMap(() => this.refreshTokenService.refresh()),
        takeUntil(this.destroy$)
      )
      .subscribe({
        next: () => {
          console.log('JWT refreshed successfully');
          this.startTimer(); // เริ่มนับถอยหลังใหม่
        },
        error: () => {
          console.warn('JWT refresh failed');
        }
      });
  }

  stopTimer() {
    this.timerSubscription?.unsubscribe();
  }

  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

---

## 7. ตัวอย่าง API Service ที่ใช้ Auth

```typescript
// app/services/api.service.ts
import { Injectable } from '@angular/core';
import { HttpClient, HttpParams } from '@angular/common/http';
import { Observable } from 'rxjs';

interface PaginatedResponse<T> {
  data: T[];
  total: number;
  page: number;
  limit: number;
}

interface QueryParams {
  page?: number;
  limit?: number;
  search?: string;
  sort?: string;
  order?: 'asc' | 'desc';
  [key: string]: any;
}

@Injectable({ providedIn: 'root' })
export class ApiService {
  private readonly BASE_URL = '/api';

  constructor(private http: HttpClient) {}

  get<T>(endpoint: string, params?: QueryParams): Observable<T> {
    let httpParams = new HttpParams();

    if (params) {
      Object.entries(params).forEach(([key, value]) => {
        if (value !== null && value !== undefined) {
          httpParams = httpParams.set(key, String(value));
        }
      });
    }

    return this.http.get<T>(`${this.BASE_URL}/${endpoint}`, { params: httpParams });
  }

  getPaginated<T>(endpoint: string, params?: QueryParams): Observable<PaginatedResponse<T>> {
    return this.get<PaginatedResponse<T>>(endpoint, params);
  }

  post<T>(endpoint: string, body: any): Observable<T> {
    return this.http.post<T>(`${this.BASE_URL}/${endpoint}`, body);
  }

  put<T>(endpoint: string, body: any): Observable<T> {
    return this.http.put<T>(`${this.BASE_URL}/${endpoint}`, body);
  }

  patch<T>(endpoint: string, body: Partial<any>): Observable<T> {
    return this.http.patch<T>(`${this.BASE_URL}/${endpoint}`, body);
  }

  delete<T>(endpoint: string): Observable<T> {
    return this.http.delete<T>(`${this.BASE_URL}/${endpoint}`);
  }
}
```

---

## สรุป

| ส่วนประกอบ | หน้าที่ |
|-----------|--------|
| JwtUtils | Decode JWT และตรวจสอบ expiry |
| TokenService | เก็บ/ดึง token จาก storage |
| RefreshTokenService | จัดการการ refresh token โดยป้องกัน race condition |
| AuthInterceptor | แนบ token ทุก request และจัดการ 401 |
| JwtTimerService | Auto-refresh ก่อน token หมดอายุ |

การใช้ Interceptor ทำให้ทุก HTTP request มี auth header อัตโนมัติ โดยไม่ต้องเขียน code ซ้ำในแต่ละ service
