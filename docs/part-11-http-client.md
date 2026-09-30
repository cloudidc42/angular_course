# Part 11 — HttpClient และการเรียก API

## บทนำ

Angular HttpClient เป็น module สำหรับการสื่อสารกับ backend API ผ่าน HTTP protocol รองรับ methods หลัก ได้แก่ GET, POST, PUT, PATCH, DELETE พร้อมระบบ interceptor สำหรับจัดการ headers, authentication, error handling, และ logging

---

## 11.1 การ Setup HttpClient

### Angular 15 และเก่ากว่า (NgModule)

```typescript
// app.module.ts
import { NgModule } from '@angular/core';
import { BrowserModule } from '@angular/platform-browser';
import { HttpClientModule } from '@angular/common/http';

@NgModule({
  imports: [
    BrowserModule,
    HttpClientModule  // เพิ่มตรงนี้
  ],
  bootstrap: [AppComponent]
})
export class AppModule {}
```

### Angular 15+ (Standalone / provideHttpClient)

```typescript
// main.ts
import { bootstrapApplication } from '@angular/platform-browser';
import { provideHttpClient, withInterceptors } from '@angular/common/http';
import { AppComponent } from './app/app.component';

bootstrapApplication(AppComponent, {
  providers: [
    provideHttpClient()
    // หรือเพิ่ม interceptors:
    // provideHttpClient(withInterceptors([authInterceptor, loggingInterceptor]))
  ]
});
```

---

## 11.2 GET Request — ดึงข้อมูล

```typescript
// services/product.service.ts
import { Injectable } from '@angular/core';
import { HttpClient, HttpParams } from '@angular/common/http';
import { Observable } from 'rxjs';

export interface Product {
  id: number;
  name: string;
  price: number;
  category: string;
  description: string;
  stock: number;
}

export interface PaginatedResponse<T> {
  data: T[];
  total: number;
  page: number;
  limit: number;
}

@Injectable({ providedIn: 'root' })
export class ProductService {
  private readonly apiUrl = 'https://api.example.com';

  constructor(private http: HttpClient) {}

  // GET ทุกสินค้า
  getProducts(): Observable<Product[]> {
    return this.http.get<Product[]>(`${this.apiUrl}/products`);
  }

  // GET สินค้า 1 รายการ
  getProduct(id: number): Observable<Product> {
    return this.http.get<Product>(`${this.apiUrl}/products/${id}`);
  }

  // GET พร้อม query parameters
  getProductsWithFilters(page: number, limit: number, category?: string): Observable<PaginatedResponse<Product>> {
    // วิธีที่ 1: ใช้ HttpParams
    let params = new HttpParams()
      .set('page', page.toString())
      .set('limit', limit.toString());

    if (category) {
      params = params.set('category', category);
    }

    return this.http.get<PaginatedResponse<Product>>(`${this.apiUrl}/products`, { params });
  }

  // GET พร้อม search
  searchProducts(query: string): Observable<Product[]> {
    // วิธีที่ 2: object literal
    const params = { q: query, per_page: '10' };
    return this.http.get<Product[]>(`${this.apiUrl}/products/search`, { params });
  }
}
```

---

## 11.3 POST, PUT, PATCH, DELETE

```typescript
// services/product.service.ts (ต่อ)
import { HttpClient, HttpHeaders } from '@angular/common/http';

@Injectable({ providedIn: 'root' })
export class ProductService {
  private readonly apiUrl = 'https://api.example.com';

  constructor(private http: HttpClient) {}

  // POST — สร้างสินค้าใหม่
  createProduct(product: Omit<Product, 'id'>): Observable<Product> {
    return this.http.post<Product>(`${this.apiUrl}/products`, product);
  }

  // PUT — อัปเดตสินค้าทั้งหมด
  updateProduct(id: number, product: Product): Observable<Product> {
    return this.http.put<Product>(`${this.apiUrl}/products/${id}`, product);
  }

  // PATCH — อัปเดตบางฟิลด์
  patchProduct(id: number, updates: Partial<Product>): Observable<Product> {
    return this.http.patch<Product>(`${this.apiUrl}/products/${id}`, updates);
  }

  // DELETE — ลบสินค้า
  deleteProduct(id: number): Observable<void> {
    return this.http.delete<void>(`${this.apiUrl}/products/${id}`);
  }

  // DELETE ที่คืนค่า boolean
  deleteProducts(ids: number[]): Observable<{ success: boolean; count: number }> {
    return this.http.delete<{ success: boolean; count: number }>(
      `${this.apiUrl}/products/bulk`,
      { body: { ids } }  // ส่ง body ใน DELETE (บาง API รองรับ)
    );
  }

  // POST พร้อม custom headers
  createProductWithAuth(product: Omit<Product, 'id'>, token: string): Observable<Product> {
    const headers = new HttpHeaders({
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json',
      'X-Request-ID': crypto.randomUUID()
    });

    return this.http.post<Product>(`${this.apiUrl}/products`, product, { headers });
  }

  // Upload file
  uploadProductImage(productId: number, file: File): Observable<{ url: string }> {
    const formData = new FormData();
    formData.append('image', file);
    formData.append('productId', productId.toString());

    return this.http.post<{ url: string }>(
      `${this.apiUrl}/products/${productId}/images`,
      formData
      // ไม่ต้องตั้ง Content-Type สำหรับ FormData — browser จะจัดการเอง
    );
  }
}
```

---

## 11.4 HTTP Headers

```typescript
// การสร้างและจัดการ Headers
import { HttpHeaders } from '@angular/common/http';

// วิธีที่ 1: ผ่าน constructor
const headers = new HttpHeaders({
  'Authorization': 'Bearer my-token',
  'Content-Type': 'application/json'
});

// วิธีที่ 2: method chaining (immutable)
const headers2 = new HttpHeaders()
  .set('Authorization', 'Bearer my-token')
  .set('Content-Type', 'application/json')
  .append('X-Custom-Header', 'value');  // append เพิ่มค่า, set แทนที่ค่า

// การอ่าน header
const authValue = headers.get('Authorization');
const allHeaders = headers.keys();
const hasAuth = headers.has('Authorization');

// การลบ header
const headersWithoutAuth = headers.delete('Authorization');
```

### การส่ง Headers กับ Request

```typescript
// services/api.service.ts
getSecureData(): Observable<any> {
  const token = localStorage.getItem('auth_token');

  const headers = new HttpHeaders({
    'Authorization': `Bearer ${token}`,
    'Accept': 'application/json',
    'X-Client-Version': '1.0.0'
  });

  return this.http.get('/api/secure-data', { headers });
}
```

---

## 11.5 Response Types

```typescript
// สามารถกำหนด observe และ responseType ได้
import { HttpClient } from '@angular/common/http';

@Injectable({ providedIn: 'root' })
export class FileService {
  constructor(private http: HttpClient) {}

  // รับ response เป็น Blob (ไฟล์)
  downloadFile(fileId: string): Observable<Blob> {
    return this.http.get(`/api/files/${fileId}`, {
      responseType: 'blob'
    });
  }

  // รับ response เป็น ArrayBuffer
  downloadBinary(url: string): Observable<ArrayBuffer> {
    return this.http.get(url, {
      responseType: 'arraybuffer'
    });
  }

  // รับ response เป็น text
  getTextFile(url: string): Observable<string> {
    return this.http.get(url, {
      responseType: 'text'
    });
  }

  // รับ full HttpResponse (พร้อม headers, status code)
  getWithFullResponse(): Observable<import('@angular/common/http').HttpResponse<Product[]>> {
    return this.http.get<Product[]>('/api/products', {
      observe: 'response'  // 'body' (default), 'response', 'events'
    });
  }

  // ดู progress events ขณะ upload
  uploadWithProgress(file: File): Observable<import('@angular/common/http').HttpEvent<any>> {
    const formData = new FormData();
    formData.append('file', file);

    return this.http.post('/api/upload', formData, {
      observe: 'events',
      reportProgress: true
    });
  }
}
```

### การจัดการ Upload Progress

```typescript
// upload.component.ts
import { Component } from '@angular/core';
import { HttpEventType } from '@angular/common/http';
import { CommonModule } from '@angular/common';
import { FileService } from './file.service';

@Component({
  selector: 'app-upload',
  standalone: true,
  imports: [CommonModule],
  template: `
    <input type="file" (change)="onFileSelect($event)">
    <div *ngIf="uploadProgress !== null">
      <div class="progress">
        <div class="progress-bar" [style.width]="uploadProgress + '%'">
          {{ uploadProgress }}%
        </div>
      </div>
    </div>
    <div *ngIf="uploadedUrl">
      อัปโหลดสำเร็จ: <a [href]="uploadedUrl">{{ uploadedUrl }}</a>
    </div>
  `
})
export class UploadComponent {
  uploadProgress: number | null = null;
  uploadedUrl: string | null = null;

  constructor(private fileService: FileService) {}

  onFileSelect(event: Event): void {
    const input = event.target as HTMLInputElement;
    const file = input.files?.[0];
    if (!file) return;

    this.fileService.uploadWithProgress(file).subscribe(event => {
      switch (event.type) {
        case HttpEventType.UploadProgress:
          if (event.total) {
            this.uploadProgress = Math.round(100 * event.loaded / event.total);
          }
          break;
        case HttpEventType.Response:
          this.uploadedUrl = event.body?.url || null;
          this.uploadProgress = null;
          break;
      }
    });
  }
}
```

---

## 11.6 Error Handling

```typescript
// services/product.service.ts
import { HttpClient, HttpErrorResponse } from '@angular/common/http';
import { Observable, throwError } from 'rxjs';
import { catchError, retry } from 'rxjs/operators';

@Injectable({ providedIn: 'root' })
export class ProductService {
  constructor(private http: HttpClient) {}

  getProducts(): Observable<Product[]> {
    return this.http.get<Product[]>('/api/products').pipe(
      catchError(this.handleError)
    );
  }

  private handleError(error: HttpErrorResponse): Observable<never> {
    let errorMessage = 'เกิดข้อผิดพลาดที่ไม่ทราบสาเหตุ';

    if (error.status === 0) {
      // Client-side error (network, CORS, etc.)
      errorMessage = 'ไม่สามารถเชื่อมต่อกับ server ได้ กรุณาตรวจสอบการเชื่อมต่ออินเทอร์เน็ต';
      console.error('Client-side error:', error.error);
    } else {
      // Server-side error
      switch (error.status) {
        case 400:
          errorMessage = 'ข้อมูลที่ส่งไม่ถูกต้อง';
          break;
        case 401:
          errorMessage = 'กรุณาเข้าสู่ระบบก่อน';
          break;
        case 403:
          errorMessage = 'คุณไม่มีสิทธิ์เข้าถึงข้อมูลนี้';
          break;
        case 404:
          errorMessage = 'ไม่พบข้อมูลที่ต้องการ';
          break;
        case 409:
          errorMessage = 'ข้อมูลซ้ำกับที่มีอยู่แล้ว';
          break;
        case 422:
          errorMessage = `ข้อมูลไม่ถูกต้อง: ${JSON.stringify(error.error?.errors)}`;
          break;
        case 429:
          errorMessage = 'คำขอมากเกินไป กรุณารอสักครู่';
          break;
        case 500:
          errorMessage = 'เกิดข้อผิดพลาดภายใน server';
          break;
        default:
          errorMessage = `Error ${error.status}: ${error.statusText}`;
      }
    }

    return throwError(() => new Error(errorMessage));
  }
}
```

### Error Handling ใน Component

```typescript
// products.component.ts
import { Component, OnInit } from '@angular/core';
import { CommonModule } from '@angular/common';
import { ProductService, Product } from './product.service';

@Component({
  selector: 'app-products',
  standalone: true,
  imports: [CommonModule],
  template: `
    <div *ngIf="loading" class="d-flex justify-content-center">
      <div class="spinner-border"></div>
    </div>

    <div *ngIf="errorMessage" class="alert alert-danger">
      <strong>ข้อผิดพลาด:</strong> {{ errorMessage }}
      <button class="btn btn-sm btn-outline-danger ms-2" (click)="loadProducts()">
        ลองใหม่
      </button>
    </div>

    <div *ngIf="!loading && !errorMessage">
      <div *ngFor="let product of products" class="card mb-2">
        <div class="card-body">
          <h5>{{ product.name }}</h5>
          <p>฿{{ product.price | number }}</p>
        </div>
      </div>
    </div>
  `
})
export class ProductsComponent implements OnInit {
  products: Product[] = [];
  loading = false;
  errorMessage = '';

  constructor(private productService: ProductService) {}

  ngOnInit(): void {
    this.loadProducts();
  }

  loadProducts(): void {
    this.loading = true;
    this.errorMessage = '';

    this.productService.getProducts().subscribe({
      next: (products) => {
        this.products = products;
        this.loading = false;
      },
      error: (error: Error) => {
        this.errorMessage = error.message;
        this.loading = false;
      }
    });
  }
}
```

---

## 11.7 HttpInterceptor — Middleware สำหรับ HTTP

Interceptor ใช้สำหรับดักจับ request และ response เพื่อเพิ่ม headers, logging, หรือจัดการ errors แบบ global

### Auth Interceptor — เพิ่ม Token อัตโนมัติ

```typescript
// interceptors/auth.interceptor.ts
import { Injectable } from '@angular/core';
import {
  HttpInterceptor,
  HttpRequest,
  HttpHandler,
  HttpEvent
} from '@angular/common/http';
import { Observable } from 'rxjs';

@Injectable()
export class AuthInterceptor implements HttpInterceptor {
  intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    const token = localStorage.getItem('auth_token');

    if (token) {
      // clone request และเพิ่ม header
      const authReq = req.clone({
        headers: req.headers.set('Authorization', `Bearer ${token}`)
      });
      return next.handle(authReq);
    }

    return next.handle(req);
  }
}
```

### Logging Interceptor — Log ทุก Request

```typescript
// interceptors/logging.interceptor.ts
import { Injectable } from '@angular/core';
import {
  HttpInterceptor, HttpRequest, HttpHandler,
  HttpEvent, HttpResponse
} from '@angular/common/http';
import { Observable } from 'rxjs';
import { tap, finalize } from 'rxjs/operators';

@Injectable()
export class LoggingInterceptor implements HttpInterceptor {
  intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    const startTime = Date.now();
    const method = req.method;
    const url = req.url;

    console.log(`[HTTP] ${method} ${url} - started`);

    return next.handle(req).pipe(
      tap({
        next: (event) => {
          if (event instanceof HttpResponse) {
            const elapsed = Date.now() - startTime;
            console.log(`[HTTP] ${method} ${url} - ${event.status} (${elapsed}ms)`);
          }
        },
        error: (error) => {
          const elapsed = Date.now() - startTime;
          console.error(`[HTTP] ${method} ${url} - ERROR ${error.status} (${elapsed}ms)`);
        }
      }),
      finalize(() => {
        // เรียกเสมอ ไม่ว่าจะสำเร็จหรือไม่
      })
    );
  }
}
```

### Error Interceptor — จัดการ errors แบบ global

```typescript
// interceptors/error.interceptor.ts
import { Injectable } from '@angular/core';
import {
  HttpInterceptor, HttpRequest, HttpHandler,
  HttpEvent, HttpErrorResponse
} from '@angular/common/http';
import { Observable, throwError } from 'rxjs';
import { catchError } from 'rxjs/operators';
import { Router } from '@angular/router';

@Injectable()
export class ErrorInterceptor implements HttpInterceptor {
  constructor(private router: Router) {}

  intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    return next.handle(req).pipe(
      catchError((error: HttpErrorResponse) => {
        if (error.status === 401) {
          // token หมดอายุ — ล้าง token และ redirect ไป login
          localStorage.removeItem('auth_token');
          this.router.navigate(['/login']);
        }

        if (error.status === 503) {
          // Service unavailable
          this.router.navigate(['/maintenance']);
        }

        return throwError(() => error);
      })
    );
  }
}
```

### การ Register Interceptors

```typescript
// app.module.ts (NgModule style)
import { HTTP_INTERCEPTORS } from '@angular/common/http';

@NgModule({
  providers: [
    { provide: HTTP_INTERCEPTORS, useClass: AuthInterceptor, multi: true },
    { provide: HTTP_INTERCEPTORS, useClass: LoggingInterceptor, multi: true },
    { provide: HTTP_INTERCEPTORS, useClass: ErrorInterceptor, multi: true }
  ]
})
export class AppModule {}
```

```typescript
// main.ts (Standalone style — Angular 15+)
import { withInterceptors } from '@angular/common/http';

// สำหรับ Functional Interceptors (Angular 15+)
export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const token = localStorage.getItem('auth_token');

  if (token) {
    const authReq = req.clone({
      headers: req.headers.set('Authorization', `Bearer ${token}`)
    });
    return next(authReq);
  }

  return next(req);
};

bootstrapApplication(AppComponent, {
  providers: [
    provideHttpClient(withInterceptors([authInterceptor]))
  ]
});
```

---

## 11.8 Loading States

### Loading Service

```typescript
// services/loading.service.ts
import { Injectable } from '@angular/core';
import { BehaviorSubject, Observable } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class LoadingService {
  private loadingCount = 0;
  private loadingSubject = new BehaviorSubject<boolean>(false);

  loading$: Observable<boolean> = this.loadingSubject.asObservable();

  show(): void {
    this.loadingCount++;
    this.loadingSubject.next(true);
  }

  hide(): void {
    this.loadingCount = Math.max(0, this.loadingCount - 1);
    if (this.loadingCount === 0) {
      this.loadingSubject.next(false);
    }
  }
}
```

### Loading Interceptor

```typescript
// interceptors/loading.interceptor.ts
import { Injectable } from '@angular/core';
import {
  HttpInterceptor, HttpRequest, HttpHandler, HttpEvent
} from '@angular/common/http';
import { Observable } from 'rxjs';
import { finalize } from 'rxjs/operators';
import { LoadingService } from '../services/loading.service';

@Injectable()
export class LoadingInterceptor implements HttpInterceptor {
  constructor(private loadingService: LoadingService) {}

  intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    // ไม่แสดง loading สำหรับ background requests
    const skipLoading = req.headers.has('X-Skip-Loading');

    if (!skipLoading) {
      this.loadingService.show();
    }

    return next.handle(req).pipe(
      finalize(() => {
        if (!skipLoading) {
          this.loadingService.hide();
        }
      })
    );
  }
}
```

### Loading Spinner Component

```typescript
// components/loading-spinner/loading-spinner.component.ts
import { Component } from '@angular/core';
import { AsyncPipe } from '@angular/common';
import { LoadingService } from '../../services/loading.service';

@Component({
  selector: 'app-loading-spinner',
  standalone: true,
  imports: [AsyncPipe],
  template: `
    <div *ngIf="loadingService.loading$ | async" class="loading-overlay">
      <div class="spinner-container">
        <div class="spinner-border text-primary" role="status">
          <span class="visually-hidden">กำลังโหลด...</span>
        </div>
        <p class="mt-2">กำลังโหลดข้อมูล...</p>
      </div>
    </div>
  `,
  styles: [`
    .loading-overlay {
      position: fixed;
      top: 0; left: 0; right: 0; bottom: 0;
      background: rgba(0,0,0,0.3);
      display: flex;
      align-items: center;
      justify-content: center;
      z-index: 9999;
    }
    .spinner-container {
      background: white;
      padding: 2rem;
      border-radius: 8px;
      text-align: center;
    }
  `]
})
export class LoadingSpinnerComponent {
  constructor(public loadingService: LoadingService) {}
}
```

---

## 11.9 Retry Logic

```typescript
// services/resilient-api.service.ts
import { Injectable } from '@angular/core';
import { HttpClient, HttpErrorResponse } from '@angular/common/http';
import { Observable, throwError, timer } from 'rxjs';
import { retry, retryWhen, switchMap, catchError, take } from 'rxjs/operators';

@Injectable({ providedIn: 'root' })
export class ResilientApiService {
  constructor(private http: HttpClient) {}

  // Retry อย่างง่าย — ลองใหม่ 3 ครั้ง
  getDataWithRetry(): Observable<any> {
    return this.http.get('/api/data').pipe(
      retry(3),  // ลองใหม่สูงสุด 3 ครั้ง
      catchError(this.handleError)
    );
  }

  // Retry พร้อม exponential backoff
  getDataWithBackoff(): Observable<any> {
    return this.http.get('/api/data').pipe(
      retryWhen(errors =>
        errors.pipe(
          switchMap((error: HttpErrorResponse, index) => {
            // ไม่ retry ถ้า error เป็น 4xx (client error)
            if (error.status >= 400 && error.status < 500) {
              return throwError(() => error);
            }

            // ลองใหม่สูงสุด 3 ครั้ง
            if (index >= 3) {
              return throwError(() => error);
            }

            // รอก่อน retry: 1s, 2s, 4s (exponential backoff)
            const delayMs = Math.pow(2, index) * 1000;
            console.log(`Retrying in ${delayMs}ms... (attempt ${index + 1}/3)`);
            return timer(delayMs);
          })
        )
      ),
      catchError(this.handleError)
    );
  }

  private handleError(error: HttpErrorResponse) {
    return throwError(() => new Error(`HTTP Error: ${error.status}`));
  }
}
```

---

## 11.10 Workshop: Products API Integration

Workshop นี้จะสร้าง complete product management ที่เชื่อมต่อกับ REST API จริง

### Product Model และ DTOs

```typescript
// models/product.model.ts
export interface Product {
  id: number;
  name: string;
  description: string;
  price: number;
  category: string;
  stock: number;
  imageUrl: string;
  createdAt: string;
  updatedAt: string;
}

export interface CreateProductDto {
  name: string;
  description: string;
  price: number;
  category: string;
  stock: number;
  imageUrl?: string;
}

export interface UpdateProductDto extends Partial<CreateProductDto> {}

export interface ProductListResponse {
  products: Product[];
  total: number;
  page: number;
  limit: number;
  totalPages: number;
}

export interface ProductFilterParams {
  page?: number;
  limit?: number;
  category?: string;
  minPrice?: number;
  maxPrice?: number;
  search?: string;
  sortBy?: 'name' | 'price' | 'createdAt';
  sortOrder?: 'asc' | 'desc';
}
```

### Complete Product Service

```typescript
// services/product-api.service.ts
import { Injectable } from '@angular/core';
import { HttpClient, HttpParams, HttpErrorResponse } from '@angular/common/http';
import { Observable, throwError } from 'rxjs';
import { catchError, map } from 'rxjs/operators';
import {
  Product, CreateProductDto, UpdateProductDto,
  ProductListResponse, ProductFilterParams
} from '../models/product.model';
import { environment } from '../../environments/environment';

@Injectable({ providedIn: 'root' })
export class ProductApiService {
  private readonly baseUrl = `${environment.apiUrl}/products`;

  constructor(private http: HttpClient) {}

  // GET all products with filters
  getProducts(filters: ProductFilterParams = {}): Observable<ProductListResponse> {
    let params = new HttpParams();

    if (filters.page) params = params.set('page', filters.page);
    if (filters.limit) params = params.set('limit', filters.limit);
    if (filters.category) params = params.set('category', filters.category);
    if (filters.minPrice !== undefined) params = params.set('minPrice', filters.minPrice);
    if (filters.maxPrice !== undefined) params = params.set('maxPrice', filters.maxPrice);
    if (filters.search) params = params.set('search', filters.search);
    if (filters.sortBy) params = params.set('sortBy', filters.sortBy);
    if (filters.sortOrder) params = params.set('sortOrder', filters.sortOrder);

    return this.http.get<ProductListResponse>(this.baseUrl, { params }).pipe(
      catchError(this.handleError)
    );
  }

  // GET product by ID
  getProductById(id: number): Observable<Product> {
    return this.http.get<Product>(`${this.baseUrl}/${id}`).pipe(
      catchError(this.handleError)
    );
  }

  // POST create product
  createProduct(dto: CreateProductDto): Observable<Product> {
    return this.http.post<Product>(this.baseUrl, dto).pipe(
      catchError(this.handleError)
    );
  }

  // PUT update product
  updateProduct(id: number, dto: UpdateProductDto): Observable<Product> {
    return this.http.put<Product>(`${this.baseUrl}/${id}`, dto).pipe(
      catchError(this.handleError)
    );
  }

  // PATCH update stock only
  updateStock(id: number, stock: number): Observable<Product> {
    return this.http.patch<Product>(`${this.baseUrl}/${id}/stock`, { stock }).pipe(
      catchError(this.handleError)
    );
  }

  // DELETE product
  deleteProduct(id: number): Observable<{ message: string }> {
    return this.http.delete<{ message: string }>(`${this.baseUrl}/${id}`).pipe(
      catchError(this.handleError)
    );
  }

  // GET categories list
  getCategories(): Observable<string[]> {
    return this.http.get<string[]>(`${this.baseUrl}/categories`).pipe(
      catchError(this.handleError)
    );
  }

  // Error handler
  private handleError(error: HttpErrorResponse): Observable<never> {
    let message = 'เกิดข้อผิดพลาด';

    if (error.status === 0) {
      message = 'ไม่สามารถเชื่อมต่อกับ server';
    } else if (error.error?.message) {
      message = error.error.message;
    } else {
      message = `Server error: ${error.status}`;
    }

    console.error('API Error:', error);
    return throwError(() => new Error(message));
  }
}
```

### Product List Component

```typescript
// components/product-list/product-list.component.ts
import { Component, OnInit, OnDestroy } from '@angular/core';
import { CommonModule } from '@angular/common';
import { FormsModule } from '@angular/forms';
import { Subject } from 'rxjs';
import { takeUntil, debounceTime, distinctUntilChanged } from 'rxjs/operators';
import { ProductApiService } from '../../services/product-api.service';
import { Product, ProductFilterParams } from '../../models/product.model';

@Component({
  selector: 'app-product-list',
  standalone: true,
  imports: [CommonModule, FormsModule],
  template: `
    <div class="container mt-4">
      <h1>รายการสินค้า</h1>

      <!-- ส่วนค้นหาและกรอง -->
      <div class="row mb-3">
        <div class="col-md-4">
          <input type="text" class="form-control"
                 placeholder="ค้นหาสินค้า..."
                 [(ngModel)]="searchQuery"
                 (ngModelChange)="onSearchChange($event)">
        </div>
        <div class="col-md-3">
          <select class="form-select" [(ngModel)]="selectedCategory"
                  (ngModelChange)="onFilterChange()">
            <option value="">ทุกหมวดหมู่</option>
            <option *ngFor="let cat of categories" [value]="cat">{{ cat }}</option>
          </select>
        </div>
        <div class="col-md-3">
          <select class="form-select" [(ngModel)]="sortBy"
                  (ngModelChange)="onFilterChange()">
            <option value="createdAt">ล่าสุด</option>
            <option value="name">ชื่อ A-Z</option>
            <option value="price">ราคา</option>
          </select>
        </div>
        <div class="col-md-2">
          <button class="btn btn-primary w-100" (click)="openCreateForm()">
            + เพิ่มสินค้า
          </button>
        </div>
      </div>

      <!-- Loading state -->
      <div *ngIf="loading" class="text-center my-4">
        <div class="spinner-border text-primary"></div>
        <p>กำลังโหลดสินค้า...</p>
      </div>

      <!-- Error state -->
      <div *ngIf="errorMessage && !loading" class="alert alert-danger">
        {{ errorMessage }}
        <button class="btn btn-sm btn-link" (click)="loadProducts()">ลองใหม่</button>
      </div>

      <!-- สินค้า -->
      <div *ngIf="!loading && !errorMessage" class="row">
        <div *ngFor="let product of products" class="col-md-4 col-lg-3 mb-4">
          <div class="card h-100">
            <img [src]="product.imageUrl || 'assets/placeholder.jpg'"
                 class="card-img-top" style="height:200px; object-fit:cover"
                 [alt]="product.name">
            <div class="card-body">
              <h5 class="card-title">{{ product.name }}</h5>
              <p class="card-text text-muted small">{{ product.description | slice:0:80 }}...</p>
              <div class="d-flex justify-content-between align-items-center">
                <strong class="text-primary">฿{{ product.price | number:'1.0-0' }}</strong>
                <span class="badge" [class]="product.stock > 0 ? 'bg-success' : 'bg-danger'">
                  {{ product.stock > 0 ? 'มีสินค้า' : 'สินค้าหมด' }}
                </span>
              </div>
            </div>
            <div class="card-footer d-flex gap-2">
              <button class="btn btn-sm btn-outline-primary flex-grow-1"
                      (click)="editProduct(product)">แก้ไข</button>
              <button class="btn btn-sm btn-outline-danger"
                      (click)="confirmDelete(product)">ลบ</button>
            </div>
          </div>
        </div>

        <!-- Empty state -->
        <div *ngIf="products.length === 0" class="col-12 text-center py-5">
          <i class="bi bi-box-seam" style="font-size:3rem; color:#ccc"></i>
          <p class="text-muted mt-2">ไม่พบสินค้า</p>
        </div>
      </div>

      <!-- Pagination -->
      <nav *ngIf="totalPages > 1" class="mt-4">
        <ul class="pagination justify-content-center">
          <li class="page-item" [class.disabled]="currentPage === 1">
            <button class="page-link" (click)="goToPage(currentPage - 1)">«</button>
          </li>
          <li *ngFor="let p of getPageNumbers()" class="page-item"
              [class.active]="p === currentPage">
            <button class="page-link" (click)="goToPage(p)">{{ p }}</button>
          </li>
          <li class="page-item" [class.disabled]="currentPage === totalPages">
            <button class="page-link" (click)="goToPage(currentPage + 1)">»</button>
          </li>
        </ul>
        <p class="text-center text-muted">
          แสดง {{ products.length }} จาก {{ totalItems }} รายการ
        </p>
      </nav>
    </div>

    <!-- Delete Confirmation Modal -->
    <div *ngIf="productToDelete" class="modal d-block" style="background:rgba(0,0,0,.5)">
      <div class="modal-dialog">
        <div class="modal-content">
          <div class="modal-header">
            <h5 class="modal-title">ยืนยันการลบ</h5>
          </div>
          <div class="modal-body">
            คุณต้องการลบสินค้า <strong>{{ productToDelete.name }}</strong> ใช่หรือไม่?
          </div>
          <div class="modal-footer">
            <button class="btn btn-secondary" (click)="productToDelete = null">ยกเลิก</button>
            <button class="btn btn-danger" (click)="deleteProduct()"
                    [disabled]="deleting">
              {{ deleting ? 'กำลังลบ...' : 'ลบ' }}
            </button>
          </div>
        </div>
      </div>
    </div>
  `
})
export class ProductListComponent implements OnInit, OnDestroy {
  products: Product[] = [];
  categories: string[] = [];
  loading = false;
  errorMessage = '';
  deleting = false;
  productToDelete: Product | null = null;

  // Filters
  searchQuery = '';
  selectedCategory = '';
  sortBy: 'name' | 'price' | 'createdAt' = 'createdAt';

  // Pagination
  currentPage = 1;
  itemsPerPage = 12;
  totalItems = 0;
  totalPages = 0;

  private searchSubject = new Subject<string>();
  private destroy$ = new Subject<void>();

  constructor(private productService: ProductApiService) {}

  ngOnInit(): void {
    this.loadCategories();
    this.loadProducts();

    // debounce search
    this.searchSubject.pipe(
      debounceTime(400),
      distinctUntilChanged(),
      takeUntil(this.destroy$)
    ).subscribe(() => {
      this.currentPage = 1;
      this.loadProducts();
    });
  }

  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
  }

  loadProducts(): void {
    this.loading = true;
    this.errorMessage = '';

    const filters: ProductFilterParams = {
      page: this.currentPage,
      limit: this.itemsPerPage,
      category: this.selectedCategory || undefined,
      search: this.searchQuery || undefined,
      sortBy: this.sortBy,
      sortOrder: 'desc'
    };

    this.productService.getProducts(filters).pipe(
      takeUntil(this.destroy$)
    ).subscribe({
      next: (response) => {
        this.products = response.products;
        this.totalItems = response.total;
        this.totalPages = response.totalPages;
        this.loading = false;
      },
      error: (err: Error) => {
        this.errorMessage = err.message;
        this.loading = false;
      }
    });
  }

  loadCategories(): void {
    this.productService.getCategories().pipe(
      takeUntil(this.destroy$)
    ).subscribe(cats => this.categories = cats);
  }

  onSearchChange(query: string): void {
    this.searchSubject.next(query);
  }

  onFilterChange(): void {
    this.currentPage = 1;
    this.loadProducts();
  }

  goToPage(page: number): void {
    if (page < 1 || page > this.totalPages) return;
    this.currentPage = page;
    this.loadProducts();
  }

  getPageNumbers(): number[] {
    return Array.from({ length: this.totalPages }, (_, i) => i + 1);
  }

  openCreateForm(): void {
    // navigate หรือ open modal
    console.log('Open create form');
  }

  editProduct(product: Product): void {
    console.log('Edit product:', product);
  }

  confirmDelete(product: Product): void {
    this.productToDelete = product;
  }

  deleteProduct(): void {
    if (!this.productToDelete) return;

    this.deleting = true;
    this.productService.deleteProduct(this.productToDelete.id).pipe(
      takeUntil(this.destroy$)
    ).subscribe({
      next: () => {
        this.products = this.products.filter(p => p.id !== this.productToDelete!.id);
        this.totalItems--;
        this.productToDelete = null;
        this.deleting = false;
      },
      error: (err: Error) => {
        alert(`ลบไม่สำเร็จ: ${err.message}`);
        this.deleting = false;
      }
    });
  }
}
```

### Environment Configuration

```typescript
// environments/environment.ts
export const environment = {
  production: false,
  apiUrl: 'http://localhost:3000/api'
};

// environments/environment.prod.ts
export const environment = {
  production: true,
  apiUrl: 'https://api.myshop.com'
};
```

---

## สรุป Part 11

ใน Part นี้เราได้เรียนรู้:

- **HttpClientModule / provideHttpClient** — การ setup
- **GET, POST, PUT, PATCH, DELETE** — HTTP methods พร้อมตัวอย่าง
- **HttpHeaders, HttpParams** — การส่ง headers และ parameters
- **Response Types** — body, blob, text, arraybuffer, full response
- **Error Handling** — การจัดการ errors อย่างเป็นระบบ
- **HttpInterceptor** — auth, logging, error, loading interceptors
- **Loading States** — Loading service และ spinner component
- **Retry Logic** — retry และ exponential backoff
- **Workshop** — Products API Integration ครบวงจร

### Best Practices
1. ใช้ `environment.ts` เก็บ API URL
2. สร้าง service แยก สำหรับแต่ละ resource
3. จัดการ error ใน interceptor แบบ global
4. ใช้ `takeUntil(destroy$)` เพื่อป้องกัน memory leak
5. ใช้ `debounceTime` สำหรับ search input
