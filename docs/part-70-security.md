# Part 70: Security ใน Angular

## บทนำ

Security เป็นสิ่งสำคัญที่สุดในการพัฒนาแอปพลิเคชัน บทนี้ครอบคลุม XSS, CSRF, CSP, Sanitization, HTTPS และ Secrets management

## 1. Cross-Site Scripting (XSS)

Angular มีการป้องกัน XSS อัตโนมัติ แต่ต้องระวังเมื่อ bypass security

### ❌ อันตราย

```typescript
// อย่าทำแบบนี้!
@Component({
  template: `<div [innerHTML]="userInput"></div>`  // XSS risk!
})
export class BadComponent {
  userInput = '<script>alert("XSS")</script>';
}
```

### ✅ ปลอดภัย

```typescript
// Angular sanitize innerHTML โดยอัตโนมัติ
@Component({
  template: `<div [innerHTML]="safeHtml"></div>`
})
export class SafeComponent {
  constructor(private sanitizer: DomSanitizer) {}

  // เมื่อต้องการ HTML จริงๆ - ใช้ sanitizeHtml
  get safeHtml(): SafeHtml {
    const trustedContent = '<p>Safe content <strong>here</strong></p>';
    return this.sanitizer.sanitize(SecurityContext.HTML, trustedContent)!;
  }

  // สำหรับ URL
  getSafeUrl(url: string): SafeUrl {
    // ตรวจสอบก่อนว่าเป็น URL ที่เชื่อถือได้
    if (!url.startsWith('https://trusted-domain.com')) {
      throw new Error('Untrusted URL');
    }
    return this.sanitizer.bypassSecurityTrustUrl(url);
  }
}
```

### XSS Prevention Service

```typescript
// security/xss-prevention.service.ts
import { Injectable } from '@angular/core';
import { DomSanitizer, SafeHtml, SafeUrl } from '@angular/platform-browser';

@Injectable({ providedIn: 'root' })
export class XssPreventionService {
  private readonly ALLOWED_HTML_TAGS = ['p', 'b', 'i', 'em', 'strong', 'a', 'ul', 'ol', 'li', 'br'];
  private readonly ALLOWED_ATTRS = ['href', 'title', 'class'];

  constructor(private sanitizer: DomSanitizer) {}

  sanitizeText(text: string): string {
    // ลบ HTML tags ทั้งหมด
    return text.replace(/<[^>]*>/g, '');
  }

  sanitizeHtml(html: string): SafeHtml {
    // Angular จะ sanitize HTML ให้
    return this.sanitizer.sanitize(1 /* SecurityContext.HTML */, html) || '';
  }

  sanitizeUrl(url: string): SafeUrl | null {
    const allowedProtocols = ['https:', 'http:', 'mailto:'];
    
    try {
      const parsedUrl = new URL(url);
      if (!allowedProtocols.includes(parsedUrl.protocol)) {
        console.warn('Blocked potentially unsafe URL:', url);
        return null;
      }
      return this.sanitizer.bypassSecurityTrustUrl(url);
    } catch {
      return null;
    }
  }

  escapeHtml(text: string): string {
    const map: Record<string, string> = {
      '&': '&amp;',
      '<': '&lt;',
      '>': '&gt;',
      '"': '&quot;',
      "'": '&#x27;',
      '/': '&#x2F;'
    };
    return text.replace(/[&<>"'/]/g, (s) => map[s]);
  }
}
```

## 2. Cross-Site Request Forgery (CSRF)

```typescript
// security/csrf.interceptor.ts
import { Injectable } from '@angular/core';
import {
  HttpInterceptor,
  HttpRequest,
  HttpHandler,
  HttpEvent,
  HttpXsrfTokenExtractor
} from '@angular/common/http';
import { Observable } from 'rxjs';

@Injectable()
export class CsrfInterceptor implements HttpInterceptor {
  // Methods ที่ต้องการ CSRF token
  private readonly MUTATION_METHODS = ['POST', 'PUT', 'PATCH', 'DELETE'];

  constructor(private tokenExtractor: HttpXsrfTokenExtractor) {}

  intercept(
    request: HttpRequest<unknown>,
    next: HttpHandler
  ): Observable<HttpEvent<unknown>> {
    // เพิ่ม CSRF token สำหรับ mutation requests
    if (this.MUTATION_METHODS.includes(request.method)) {
      const token = this.tokenExtractor.getToken();
      
      if (token && !request.headers.has('X-XSRF-TOKEN')) {
        request = request.clone({
          headers: request.headers.set('X-XSRF-TOKEN', token)
        });
      }
    }

    return next.handle(request);
  }
}
```

```typescript
// app.config.ts
import { provideHttpClient, withXsrfConfiguration } from '@angular/common/http';

export const appConfig = {
  providers: [
    provideHttpClient(
      withXsrfConfiguration({
        cookieName: 'XSRF-TOKEN',
        headerName: 'X-XSRF-TOKEN'
      })
    )
  ]
};
```

## 3. Content Security Policy (CSP)

### HTTP Header (Server-side)

```nginx
# nginx/security-headers.conf
add_header Content-Security-Policy "
  default-src 'self';
  script-src 'self' 'nonce-{NONCE}';
  style-src 'self' 'unsafe-inline' https://fonts.googleapis.com;
  font-src 'self' https://fonts.gstatic.com;
  img-src 'self' data: https:;
  connect-src 'self' https://api.example.com wss://api.example.com;
  frame-src 'none';
  object-src 'none';
  base-uri 'self';
  form-action 'self';
  upgrade-insecure-requests;
  block-all-mixed-content;
" always;
```

### CSP Nonce สำหรับ Angular

```typescript
// security/csp.service.ts
import { Injectable, Inject } from '@angular/core';
import { DOCUMENT } from '@angular/common';

@Injectable({ providedIn: 'root' })
export class CspService {
  constructor(@Inject(DOCUMENT) private document: Document) {}

  getNonce(): string | null {
    // อ่าน nonce จาก meta tag
    const metaNonce = this.document.querySelector('meta[name="csp-nonce"]');
    return metaNonce?.getAttribute('content') || null;
  }

  addNonceToScript(scriptElement: HTMLScriptElement): void {
    const nonce = this.getNonce();
    if (nonce) {
      scriptElement.nonce = nonce;
    }
  }

  createTrustedScript(content: string): HTMLScriptElement {
    const script = this.document.createElement('script');
    this.addNonceToScript(script);
    script.textContent = content;
    return script;
  }
}
```

## 4. Authentication & Authorization

```typescript
// security/auth.guard.ts
import { inject } from '@angular/core';
import { Router, CanActivateFn } from '@angular/router';
import { AuthService } from './auth.service';

export const authGuard: CanActivateFn = (route, state) => {
  const authService = inject(AuthService);
  const router = inject(Router);

  if (!authService.isAuthenticated()) {
    router.navigate(['/login'], {
      queryParams: { returnUrl: state.url }
    });
    return false;
  }

  // ตรวจสอบ permission ถ้ามีกำหนดใน route
  const requiredPermission = route.data?.['permission'] as string;
  if (requiredPermission && !authService.hasPermission(requiredPermission)) {
    router.navigate(['/forbidden']);
    return false;
  }

  return true;
};

export const roleGuard: CanActivateFn = (route) => {
  const authService = inject(AuthService);
  const router = inject(Router);
  
  const allowedRoles = route.data?.['roles'] as string[];
  if (!allowedRoles?.length) return true;

  const userRole = authService.getCurrentUser()?.role;
  if (!userRole || !allowedRoles.includes(userRole)) {
    router.navigate(['/forbidden']);
    return false;
  }

  return true;
};
```

```typescript
// security/token.service.ts
import { Injectable } from '@angular/core';

export interface TokenPair {
  accessToken: string;
  refreshToken: string;
  expiresAt: number;
}

@Injectable({ providedIn: 'root' })
export class TokenService {
  private readonly ACCESS_TOKEN_KEY = 'access_token';
  private readonly REFRESH_TOKEN_KEY = 'refresh_token';

  // เก็บ tokens อย่างปลอดภัย
  setTokens(tokens: TokenPair): void {
    // Access token ใน memory (ไม่เก็บใน localStorage ถ้าเป็นไปได้)
    sessionStorage.setItem(this.ACCESS_TOKEN_KEY, tokens.accessToken);
    // Refresh token ใน httpOnly cookie (ต้องทำที่ server)
    // localStorage สำหรับ expiry เท่านั้น
    localStorage.setItem('token_expires_at', String(tokens.expiresAt));
  }

  getAccessToken(): string | null {
    return sessionStorage.getItem(this.ACCESS_TOKEN_KEY);
  }

  isTokenExpired(): boolean {
    const expiresAt = Number(localStorage.getItem('token_expires_at') || 0);
    return Date.now() >= expiresAt;
  }

  clearTokens(): void {
    sessionStorage.removeItem(this.ACCESS_TOKEN_KEY);
    localStorage.removeItem('token_expires_at');
  }

  // Parse JWT payload (ไม่ verify signature - ทำที่ server)
  parseToken(token: string): Record<string, any> | null {
    try {
      const payload = token.split('.')[1];
      return JSON.parse(atob(payload));
    } catch {
      return null;
    }
  }
}
```

## 5. Input Validation และ Sanitization

```typescript
// security/input-sanitizer.pipe.ts
import { Pipe, PipeTransform } from '@angular/core';

@Pipe({ name: 'sanitize', standalone: true, pure: true })
export class SanitizePipe implements PipeTransform {
  transform(value: string, type: 'text' | 'email' | 'url' = 'text'): string {
    switch (type) {
      case 'email':
        return value.toLowerCase().trim().replace(/[^a-z0-9@._\-+]/g, '');
      case 'url':
        // Allow only safe URL characters
        return value.replace(/[^a-zA-Z0-9\-._~:/?#\[\]@!$&'()*+,;=%]/g, '');
      default:
        // Remove HTML tags and dangerous characters
        return value.replace(/<[^>]*>/g, '').trim();
    }
  }
}
```

```typescript
// security/secure-form.validator.ts
import { AbstractControl, ValidatorFn } from '@angular/forms';

export class SecureFormValidators {
  // ป้องกัน SQL Injection patterns
  static noSqlInjection(): ValidatorFn {
    return (control: AbstractControl) => {
      const sqlPatterns = [
        /(\bselect\b|\binsert\b|\bupdate\b|\bdelete\b|\bdrop\b|\bcreate\b)/i,
        /(--|;|'|"|\/\*|\*\/)/,
        /(\bor\b|\band\b)\s+\d+\s*=\s*\d+/i
      ];

      if (control.value && sqlPatterns.some(p => p.test(control.value))) {
        return { sqlInjection: true };
      }
      return null;
    };
  }

  // ป้องกัน Script Injection
  static noScriptTags(): ValidatorFn {
    return (control: AbstractControl) => {
      if (control.value && /<script[\s\S]*?>[\s\S]*?<\/script>/gi.test(control.value)) {
        return { scriptInjection: true };
      }
      return null;
    };
  }

  // ตรวจสอบ URL ปลอดภัย
  static safeUrl(): ValidatorFn {
    return (control: AbstractControl) => {
      if (!control.value) return null;
      
      try {
        const url = new URL(control.value);
        if (!['http:', 'https:'].includes(url.protocol)) {
          return { unsafeProtocol: true };
        }
      } catch {
        return { invalidUrl: true };
      }
      
      return null;
    };
  }

  // ป้องกัน Path Traversal
  static noPathTraversal(): ValidatorFn {
    return (control: AbstractControl) => {
      if (control.value && /\.\.[/\\]/.test(control.value)) {
        return { pathTraversal: true };
      }
      return null;
    };
  }
}
```

## 6. HTTPS และ Secure Headers

```typescript
// security/https.guard.ts
import { inject } from '@angular/core';
import { CanActivateFn } from '@angular/router';
import { DOCUMENT } from '@angular/common';

export const httpsGuard: CanActivateFn = () => {
  const document = inject(DOCUMENT);
  
  if (document.location.protocol !== 'https:' && 
      document.location.hostname !== 'localhost') {
    document.location.href = document.location.href.replace('http:', 'https:');
    return false;
  }
  
  return true;
};
```

```typescript
// security/security-interceptor.ts
import { Injectable } from '@angular/core';
import {
  HttpInterceptor,
  HttpRequest,
  HttpHandler,
  HttpEvent
} from '@angular/common/http';
import { Observable } from 'rxjs';

@Injectable()
export class SecurityInterceptor implements HttpInterceptor {
  intercept(
    request: HttpRequest<unknown>,
    next: HttpHandler
  ): Observable<HttpEvent<unknown>> {
    // เพิ่ม security headers
    const secureRequest = request.clone({
      headers: request.headers
        .set('X-Content-Type-Options', 'nosniff')
        .set('X-Requested-With', 'XMLHttpRequest')
    });

    return next.handle(secureRequest);
  }
}
```

## 7. Secrets Management

```typescript
// config/secrets.service.ts
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable, of } from 'rxjs';
import { map, catchError } from 'rxjs/operators';

// ❌ อย่าทำแบบนี้ - Secrets ใน source code
const BAD_CONFIG = {
  apiKey: 'sk-1234567890abcdef',  // อันตราย!
  secretKey: 'my-secret-123'       // อันตราย!
};

// ✅ ดี - อ่าน config จาก environment ที่ปลอดภัย
@Injectable({ providedIn: 'root' })
export class SecretsService {
  private config: Record<string, string> = {};

  constructor(private http: HttpClient) {}

  // โหลด config จาก secure endpoint (ที่ server จัดการ)
  loadConfig(): Observable<void> {
    return this.http.get<Record<string, string>>('/api/client-config').pipe(
      map(config => {
        this.config = config;
      }),
      catchError(() => of(void 0))
    );
  }

  get(key: string): string | null {
    return this.config[key] || null;
  }
}
```

### Environment Variables ที่ปลอดภัย

```typescript
// ❌ อย่า hardcode secrets
const apiKey = 'sk-secret-key-123';

// ✅ ใช้ environment variables ที่ inject ตอน build
// angular.json
// "fileReplacements": [{ "replace": "src/environments/environment.ts", "with": "src/environments/environment.prod.ts" }]

// environment.ts (non-sensitive only)
export const environment = {
  production: false,
  apiUrl: 'http://localhost:3000',
  // ไม่เก็บ secrets ใน environment files!
};
```

## 8. Rate Limiting ใน Angular

```typescript
// security/rate-limiter.service.ts
import { Injectable } from '@angular/core';

interface RateLimitEntry {
  count: number;
  resetTime: number;
}

@Injectable({ providedIn: 'root' })
export class RateLimiterService {
  private limits = new Map<string, RateLimitEntry>();

  // ป้องกัน brute force สำหรับ login
  checkRateLimit(
    action: string,
    maxAttempts = 5,
    windowMs = 15 * 60 * 1000 // 15 นาที
  ): { allowed: boolean; remainingAttempts: number; resetIn: number } {
    const now = Date.now();
    const entry = this.limits.get(action);

    if (!entry || now > entry.resetTime) {
      this.limits.set(action, { count: 1, resetTime: now + windowMs });
      return { allowed: true, remainingAttempts: maxAttempts - 1, resetIn: windowMs };
    }

    if (entry.count >= maxAttempts) {
      return {
        allowed: false,
        remainingAttempts: 0,
        resetIn: entry.resetTime - now
      };
    }

    entry.count++;
    return {
      allowed: true,
      remainingAttempts: maxAttempts - entry.count,
      resetIn: entry.resetTime - now
    };
  }

  reset(action: string): void {
    this.limits.delete(action);
  }
}
```

## 9. Security Audit Checklist

```typescript
// security/security-audit.service.ts
@Injectable({ providedIn: 'root' })
export class SecurityAuditService {
  runBasicChecks(): SecurityCheckResult[] {
    return [
      this.checkHttps(),
      this.checkLocalStorageSensitiveData(),
      this.checkConsoleLeaks(),
      this.checkInlineEventHandlers()
    ];
  }

  private checkHttps(): SecurityCheckResult {
    const isHttps = location.protocol === 'https:' || location.hostname === 'localhost';
    return {
      name: 'HTTPS',
      passed: isHttps,
      severity: 'critical',
      message: isHttps ? 'Using HTTPS' : 'WARNING: Not using HTTPS'
    };
  }

  private checkLocalStorageSensitiveData(): SecurityCheckResult {
    const sensitiveKeys = ['password', 'secret', 'token', 'private_key', 'api_key'];
    const found = sensitiveKeys.filter(key =>
      Object.keys(localStorage).some(k => k.toLowerCase().includes(key))
    );

    return {
      name: 'LocalStorage Sensitive Data',
      passed: found.length === 0,
      severity: 'high',
      message: found.length > 0
        ? `Found potentially sensitive keys: ${found.join(', ')}`
        : 'No sensitive data in localStorage'
    };
  }

  private checkConsoleLeaks(): SecurityCheckResult {
    // ตรวจสอบว่า console.log ถูกปิดใน production
    const isProduction = (window as any)['__ng_environment__']?.production;
    
    return {
      name: 'Console Logging',
      passed: !isProduction || (typeof console.log === 'function'),
      severity: 'low',
      message: 'Check console logging in production'
    };
  }

  private checkInlineEventHandlers(): SecurityCheckResult {
    const hasInlineHandlers = document.querySelectorAll('[onclick], [onload], [onerror]').length > 0;
    return {
      name: 'Inline Event Handlers',
      passed: !hasInlineHandlers,
      severity: 'medium',
      message: hasInlineHandlers
        ? 'Found inline event handlers - potential XSS vector'
        : 'No inline event handlers found'
    };
  }
}

interface SecurityCheckResult {
  name: string;
  passed: boolean;
  severity: 'critical' | 'high' | 'medium' | 'low';
  message: string;
}
```

## สรุป Security Checklist

| ภัยคุกคาม | การป้องกัน | Angular Feature |
|-----------|-----------|----------------|
| XSS | DomSanitizer, Angular escaping | Built-in |
| CSRF | XSRF Token | HttpClientXsrfModule |
| Clickjacking | X-Frame-Options header | Nginx config |
| SQL Injection | Input validation | Custom validators |
| Sensitive Data | ไม่เก็บใน code/localStorage | Secure token service |
| HTTPS | Force HTTPS, HSTS | HTTPS Guard + Nginx |
| CSP | Content-Security-Policy header | Meta tag + Nginx |
| Brute Force | Rate limiting | RateLimiterService |

Security ต้องทำทั้งใน Frontend และ Backend ควรทำ security audit สม่ำเสมอ
