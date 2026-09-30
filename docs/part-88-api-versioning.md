# Part 88: API Versioning Strategies ใน Angular

## ทำไมต้อง Version API

เมื่อ API เปลี่ยนแปลง client เก่าต้องยังทำงานได้ API versioning ช่วยให้ backward compatible

---

## 1. URL Path Versioning

```typescript
// core/api/api-config.service.ts
import { Injectable } from '@angular/core';
import { environment } from '../../environments/environment';

export type ApiVersion = 'v1' | 'v2' | 'v3';

@Injectable({ providedIn: 'root' })
export class ApiConfigService {
  readonly baseUrl = environment.apiUrl;
  readonly defaultVersion: ApiVersion = 'v2';

  buildUrl(path: string, version?: ApiVersion): string {
    const v = version || this.defaultVersion;
    const cleanPath = path.startsWith('/') ? path.slice(1) : path;
    return `${this.baseUrl}/api/${v}/${cleanPath}`;
  }

  // สำหรับ legacy endpoints ที่ไม่มี version
  buildLegacyUrl(path: string): string {
    const cleanPath = path.startsWith('/') ? path.slice(1) : path;
    return `${this.baseUrl}/api/${cleanPath}`;
  }
}
```

---

## 2. Header-Based Versioning

```typescript
// core/http/version.interceptor.ts
import { Injectable } from '@angular/core';
import { HttpInterceptor, HttpRequest, HttpHandler, HttpEvent } from '@angular/common/http';
import { Observable } from 'rxjs';

@Injectable()
export class VersionInterceptor implements HttpInterceptor {
  private readonly API_VERSION = '2024-01';  // Date-based versioning
  
  intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    // เพิ่ม version header เฉพาะ API requests
    if (req.url.includes('/api/')) {
      const cloned = req.clone({
        headers: req.headers
          .set('API-Version', this.API_VERSION)
          .set('X-App-Version', '1.5.0')
      });
      return next.handle(cloned);
    }
    return next.handle(req);
  }
}
```

---

## 3. Versioned HTTP Client

```typescript
// core/api/versioned-http.service.ts
import { Injectable } from '@angular/core';
import { HttpClient, HttpParams, HttpHeaders } from '@angular/common/http';
import { Observable } from 'rxjs';
import { ApiConfigService, ApiVersion } from './api-config.service';

export interface RequestOptions {
  version?: ApiVersion;
  params?: Record<string, string | number | boolean>;
  headers?: Record<string, string>;
}

@Injectable({ providedIn: 'root' })
export class VersionedHttpService {
  constructor(
    private http: HttpClient,
    private apiConfig: ApiConfigService
  ) {}

  get<T>(path: string, options: RequestOptions = {}): Observable<T> {
    const url = this.apiConfig.buildUrl(path, options.version);
    const params = this.buildParams(options.params);
    const headers = this.buildHeaders(options.headers, options.version);
    
    return this.http.get<T>(url, { params, headers });
  }

  post<T>(path: string, body: any, options: RequestOptions = {}): Observable<T> {
    const url = this.apiConfig.buildUrl(path, options.version);
    const headers = this.buildHeaders(options.headers, options.version);
    
    return this.http.post<T>(url, body, { headers });
  }

  put<T>(path: string, body: any, options: RequestOptions = {}): Observable<T> {
    const url = this.apiConfig.buildUrl(path, options.version);
    const headers = this.buildHeaders(options.headers, options.version);
    
    return this.http.put<T>(url, body, { headers });
  }

  patch<T>(path: string, body: any, options: RequestOptions = {}): Observable<T> {
    const url = this.apiConfig.buildUrl(path, options.version);
    const headers = this.buildHeaders(options.headers, options.version);
    
    return this.http.patch<T>(url, body, { headers });
  }

  delete<T>(path: string, options: RequestOptions = {}): Observable<T> {
    const url = this.apiConfig.buildUrl(path, options.version);
    const headers = this.buildHeaders(options.headers, options.version);
    
    return this.http.delete<T>(url, { headers });
  }

  private buildParams(params?: Record<string, string | number | boolean>): HttpParams {
    let httpParams = new HttpParams();
    if (params) {
      Object.entries(params).forEach(([key, value]) => {
        if (value !== undefined && value !== null) {
          httpParams = httpParams.set(key, String(value));
        }
      });
    }
    return httpParams;
  }

  private buildHeaders(
    headers?: Record<string, string>, 
    version?: ApiVersion
  ): HttpHeaders {
    let httpHeaders = new HttpHeaders();
    
    if (headers) {
      Object.entries(headers).forEach(([key, value]) => {
        httpHeaders = httpHeaders.set(key, value);
      });
    }
    
    return httpHeaders;
  }
}
```

---

## 4. API Migration Service

```typescript
// core/api/api-migration.service.ts
import { Injectable } from '@angular/core';
import { Observable, throwError } from 'rxjs';
import { catchError, map, switchMap } from 'rxjs/operators';
import { VersionedHttpService } from './versioned-http.service';

export interface ApiMigrationConfig {
  endpoint: string;
  fromVersion: string;
  toVersion: string;
  transformRequest?: (data: any) => any;
  transformResponse?: (data: any) => any;
}

@Injectable({ providedIn: 'root' })
export class ApiMigrationService {
  private migrations: ApiMigrationConfig[] = [];

  constructor(private http: VersionedHttpService) {}

  registerMigration(config: ApiMigrationConfig): void {
    this.migrations.push(config);
  }

  // ลอง v2 ก่อน fallback ไป v1
  getWithFallback<T>(
    path: string, 
    params?: Record<string, any>
  ): Observable<T> {
    return this.http.get<T>(path, { version: 'v2', params }).pipe(
      catchError(error => {
        if (error.status === 404 || error.status === 410) {
          console.warn(`v2 endpoint not found, falling back to v1: ${path}`);
          return this.http.get<T>(path, { version: 'v1', params });
        }
        return throwError(() => error);
      })
    );
  }

  // Migrate response format จาก v1 ไป v2
  transformV1ToV2Response<T>(data: any, endpoint: string): T {
    const migration = this.migrations.find(m => 
      m.endpoint === endpoint && m.fromVersion === 'v1'
    );
    
    if (migration?.transformResponse) {
      return migration.transformResponse(data) as T;
    }
    
    return data as T;
  }
}
```

---

## 5. Response Normalizer

```typescript
// core/api/response-normalizer.service.ts
import { Injectable } from '@angular/core';

// V1 Response format (เก่า)
interface V1ProductResponse {
  product_id: number;
  product_name: string;
  product_price: number;
  product_category: string;
  in_stock: boolean;
  stock_qty: number;
}

// V2 Response format (ใหม่)
interface V2ProductResponse {
  id: string;
  name: string;
  price: {
    amount: number;
    currency: string;
  };
  category: {
    id: string;
    name: string;
  };
  inventory: {
    available: boolean;
    quantity: number;
  };
}

// DTO ที่ app ใช้
export interface ProductDTO {
  id: string;
  name: string;
  price: number;
  currency: string;
  category: string;
  available: boolean;
  quantity: number;
}

@Injectable({ providedIn: 'root' })
export class ResponseNormalizerService {
  
  normalizeProduct(data: V1ProductResponse | V2ProductResponse, version: string): ProductDTO {
    if (version === 'v1') {
      return this.normalizeV1Product(data as V1ProductResponse);
    }
    return this.normalizeV2Product(data as V2ProductResponse);
  }

  private normalizeV1Product(data: V1ProductResponse): ProductDTO {
    return {
      id: String(data.product_id),
      name: data.product_name,
      price: data.product_price,
      currency: 'THB',
      category: data.product_category,
      available: data.in_stock,
      quantity: data.stock_qty
    };
  }

  private normalizeV2Product(data: V2ProductResponse): ProductDTO {
    return {
      id: data.id,
      name: data.name,
      price: data.price.amount,
      currency: data.price.currency,
      category: data.category.name,
      available: data.inventory.available,
      quantity: data.inventory.quantity
    };
  }

  // List normalization
  normalizeProductList(
    data: (V1ProductResponse | V2ProductResponse)[],
    version: string
  ): ProductDTO[] {
    return data.map(item => this.normalizeProduct(item, version));
  }
}
```

---

## 6. API Version Guard

```typescript
// features/products/guards/api-version.guard.ts
import { Injectable } from '@angular/core';
import { CanActivate, Router } from '@angular/router';
import { HttpClient } from '@angular/common/http';
import { Observable, of } from 'rxjs';
import { map, catchError } from 'rxjs/operators';

@Injectable({ providedIn: 'root' })
export class ApiVersionGuard implements CanActivate {
  private minRequiredVersion = '2.0.0';

  constructor(private http: HttpClient, private router: Router) {}

  canActivate(): Observable<boolean> {
    return this.http.get<{ version: string }>('/api/version').pipe(
      map(response => {
        if (this.isVersionCompatible(response.version, this.minRequiredVersion)) {
          return true;
        }
        this.router.navigate(['/upgrade-required']);
        return false;
      }),
      catchError(() => {
        // ถ้าไม่สามารถตรวจสอบได้ ให้ผ่านไปก่อน
        return of(true);
      })
    );
  }

  private isVersionCompatible(current: string, minimum: string): boolean {
    const currentParts = current.split('.').map(Number);
    const minimumParts = minimum.split('.').map(Number);
    
    for (let i = 0; i < 3; i++) {
      if (currentParts[i] > minimumParts[i]) return true;
      if (currentParts[i] < minimumParts[i]) return false;
    }
    return true;
  }
}
```

---

## สรุป

| Strategy | ข้อดี | ข้อเสีย |
|---------|-------|---------|
| URL Path `/api/v2/` | ชัดเจน, cacheable | URL เปลี่ยน |
| Header `API-Version` | URL สะอาด | Cache ซับซ้อน |
| Query `?version=2` | ง่าย test | Clutters URL |
| Date-based `2024-01` | Stripe ใช้ | ซับซ้อนกว่า |

### Best Practices

1. Maintain เก่าอย่างน้อย 2 versions
2. Deprecate อย่างน้อย 6 เดือนล่วงหน้า
3. Document breaking changes
4. ส่ง `Deprecation` header สำหรับ endpoints ที่จะลบ
5. ใช้ Semantic Versioning
6. Test backward compatibility อัตโนมัติ
