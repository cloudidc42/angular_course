# Part 52: Caching ใน Angular

## บทนำ

Caching ช่วยลดจำนวน HTTP requests และทำให้แอปพลิเคชันตอบสนองได้เร็วขึ้น ใน Angular มีหลายวิธีในการทำ caching ตั้งแต่ HTTP Cache ไปจนถึง Memory Cache

## 1. HTTP Caching Headers

เมื่อ server ส่ง cache headers กลับมา browser จะ cache response ให้อัตโนมัติ

```typescript
// cache-aware.service.ts
import { Injectable } from '@angular/core';
import { HttpClient, HttpHeaders } from '@angular/common/http';
import { Observable } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class CacheAwareService {
  constructor(private http: HttpClient) {}

  // ขอข้อมูลพร้อม cache headers
  getWithCacheControl(url: string): Observable<any> {
    const headers = new HttpHeaders({
      'Cache-Control': 'max-age=3600',
      'Pragma': 'cache'
    });
    
    return this.http.get(url, { headers });
  }

  // ข้ามการ cache
  getNoCache(url: string): Observable<any> {
    const headers = new HttpHeaders({
      'Cache-Control': 'no-cache, no-store, must-revalidate',
      'Pragma': 'no-cache'
    });
    
    return this.http.get(url, { headers });
  }
}
```

## 2. Memory Cache Service

```typescript
// cache.service.ts
import { Injectable } from '@angular/core';

interface CacheEntry<T> {
  data: T;
  timestamp: number;
  ttl: number; // Time To Live in milliseconds
}

@Injectable({ providedIn: 'root' })
export class CacheService {
  private cache = new Map<string, CacheEntry<any>>();
  private readonly DEFAULT_TTL = 5 * 60 * 1000; // 5 นาที

  set<T>(key: string, data: T, ttl = this.DEFAULT_TTL): void {
    this.cache.set(key, {
      data,
      timestamp: Date.now(),
      ttl
    });
  }

  get<T>(key: string): T | null {
    const entry = this.cache.get(key);
    
    if (!entry) {
      return null;
    }

    if (this.isExpired(entry)) {
      this.cache.delete(key);
      return null;
    }

    return entry.data as T;
  }

  has(key: string): boolean {
    const entry = this.cache.get(key);
    if (!entry) return false;
    
    if (this.isExpired(entry)) {
      this.cache.delete(key);
      return false;
    }
    
    return true;
  }

  delete(key: string): void {
    this.cache.delete(key);
  }

  clear(): void {
    this.cache.clear();
  }

  clearPattern(pattern: RegExp): void {
    for (const key of this.cache.keys()) {
      if (pattern.test(key)) {
        this.cache.delete(key);
      }
    }
  }

  getStats(): { size: number; keys: string[] } {
    return {
      size: this.cache.size,
      keys: Array.from(this.cache.keys())
    };
  }

  private isExpired(entry: CacheEntry<any>): boolean {
    return Date.now() - entry.timestamp > entry.ttl;
  }
}
```

## 3. Cache Interceptor

```typescript
// cache.interceptor.ts
import { Injectable } from '@angular/core';
import {
  HttpInterceptor,
  HttpRequest,
  HttpHandler,
  HttpEvent,
  HttpResponse
} from '@angular/common/http';
import { Observable, of } from 'rxjs';
import { tap } from 'rxjs/operators';
import { CacheService } from './cache.service';

@Injectable()
export class CacheInterceptor implements HttpInterceptor {
  private readonly CACHEABLE_METHODS = ['GET'];
  private readonly CACHE_TTL = 5 * 60 * 1000; // 5 นาที

  constructor(private cacheService: CacheService) {}

  intercept(
    request: HttpRequest<unknown>,
    next: HttpHandler
  ): Observable<HttpEvent<unknown>> {
    // Cache เฉพาะ GET requests
    if (!this.CACHEABLE_METHODS.includes(request.method)) {
      return next.handle(request);
    }

    // ข้าม cache ถ้ามี header บอก
    if (request.headers.has('X-No-Cache')) {
      return next.handle(request);
    }

    const cacheKey = this.getCacheKey(request);
    const cachedResponse = this.cacheService.get<HttpResponse<unknown>>(cacheKey);

    if (cachedResponse) {
      console.log(`[Cache HIT] ${request.url}`);
      return of(cachedResponse);
    }

    console.log(`[Cache MISS] ${request.url}`);
    
    return next.handle(request).pipe(
      tap(event => {
        if (event instanceof HttpResponse && event.status === 200) {
          this.cacheService.set(cacheKey, event, this.CACHE_TTL);
        }
      })
    );
  }

  private getCacheKey(request: HttpRequest<unknown>): string {
    const params = request.params.toString();
    return `${request.method}:${request.url}${params ? '?' + params : ''}`;
  }
}
```

### ลงทะเบียน Interceptor

```typescript
// app.module.ts
import { HTTP_INTERCEPTORS } from '@angular/common/http';
import { CacheInterceptor } from './cache.interceptor';

@NgModule({
  providers: [
    {
      provide: HTTP_INTERCEPTORS,
      useClass: CacheInterceptor,
      multi: true
    }
  ]
})
export class AppModule {}
```

## 4. Observable Cache Pattern

```typescript
// cached-http.service.ts
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable, of } from 'rxjs';
import { shareReplay, tap } from 'rxjs/operators';

@Injectable({ providedIn: 'root' })
export class CachedHttpService {
  private cache = new Map<string, Observable<any>>();

  constructor(private http: HttpClient) {}

  // Cache observable ด้วย shareReplay
  get<T>(url: string, forceRefresh = false): Observable<T> {
    if (forceRefresh) {
      this.cache.delete(url);
    }

    if (!this.cache.has(url)) {
      const request$ = this.http.get<T>(url).pipe(
        shareReplay(1) // cache ใน memory และ share กับ subscriber
      );
      this.cache.set(url, request$);
    }

    return this.cache.get(url) as Observable<T>;
  }

  invalidate(url: string): void {
    this.cache.delete(url);
  }

  invalidateAll(): void {
    this.cache.clear();
  }
}
```

## 5. Advanced Cache Service พร้อม LRU

```typescript
// lru-cache.service.ts
import { Injectable } from '@angular/core';

class LRUNode<T> {
  key: string;
  value: T;
  prev: LRUNode<T> | null = null;
  next: LRUNode<T> | null = null;
  expiresAt: number;

  constructor(key: string, value: T, ttl: number) {
    this.key = key;
    this.value = value;
    this.expiresAt = Date.now() + ttl;
  }
}

@Injectable({ providedIn: 'root' })
export class LRUCacheService<T = any> {
  private map = new Map<string, LRUNode<T>>();
  private head: LRUNode<T> | null = null; // Most Recently Used
  private tail: LRUNode<T> | null = null; // Least Recently Used
  private readonly maxSize: number;

  constructor(maxSize = 100) {
    this.maxSize = maxSize;
  }

  set(key: string, value: T, ttl = 300000): void {
    if (this.map.has(key)) {
      const node = this.map.get(key)!;
      node.value = value;
      node.expiresAt = Date.now() + ttl;
      this.moveToHead(node);
      return;
    }

    const node = new LRUNode<T>(key, value, ttl);
    this.map.set(key, node);
    this.addToHead(node);

    if (this.map.size > this.maxSize) {
      const removed = this.removeTail();
      if (removed) {
        this.map.delete(removed.key);
      }
    }
  }

  get(key: string): T | null {
    if (!this.map.has(key)) return null;

    const node = this.map.get(key)!;
    
    if (Date.now() > node.expiresAt) {
      this.remove(node);
      this.map.delete(key);
      return null;
    }

    this.moveToHead(node);
    return node.value;
  }

  private addToHead(node: LRUNode<T>): void {
    node.prev = null;
    node.next = this.head;
    
    if (this.head) {
      this.head.prev = node;
    }
    
    this.head = node;
    
    if (!this.tail) {
      this.tail = node;
    }
  }

  private remove(node: LRUNode<T>): void {
    if (node.prev) {
      node.prev.next = node.next;
    } else {
      this.head = node.next;
    }

    if (node.next) {
      node.next.prev = node.prev;
    } else {
      this.tail = node.prev;
    }
  }

  private moveToHead(node: LRUNode<T>): void {
    this.remove(node);
    this.addToHead(node);
  }

  private removeTail(): LRUNode<T> | null {
    const node = this.tail;
    if (node) {
      this.remove(node);
    }
    return node;
  }
}
```

## 6. Cache ด้วย localStorage/sessionStorage

```typescript
// persistent-cache.service.ts
import { Injectable } from '@angular/core';

interface PersistentCacheEntry<T> {
  data: T;
  expiresAt: number;
  version: string;
}

@Injectable({ providedIn: 'root' })
export class PersistentCacheService {
  private readonly CACHE_VERSION = '1.0';
  private readonly PREFIX = 'app_cache_';

  setSession<T>(key: string, data: T, ttlMs = 30 * 60 * 1000): void {
    const entry: PersistentCacheEntry<T> = {
      data,
      expiresAt: Date.now() + ttlMs,
      version: this.CACHE_VERSION
    };
    
    try {
      sessionStorage.setItem(this.PREFIX + key, JSON.stringify(entry));
    } catch (e) {
      console.warn('SessionStorage write failed:', e);
    }
  }

  getSession<T>(key: string): T | null {
    try {
      const raw = sessionStorage.getItem(this.PREFIX + key);
      if (!raw) return null;

      const entry: PersistentCacheEntry<T> = JSON.parse(raw);
      
      if (entry.version !== this.CACHE_VERSION || Date.now() > entry.expiresAt) {
        sessionStorage.removeItem(this.PREFIX + key);
        return null;
      }

      return entry.data;
    } catch {
      return null;
    }
  }

  setLocal<T>(key: string, data: T, ttlMs = 24 * 60 * 60 * 1000): void {
    const entry: PersistentCacheEntry<T> = {
      data,
      expiresAt: Date.now() + ttlMs,
      version: this.CACHE_VERSION
    };
    
    try {
      localStorage.setItem(this.PREFIX + key, JSON.stringify(entry));
    } catch (e) {
      console.warn('LocalStorage write failed:', e);
    }
  }

  getLocal<T>(key: string): T | null {
    try {
      const raw = localStorage.getItem(this.PREFIX + key);
      if (!raw) return null;

      const entry: PersistentCacheEntry<T> = JSON.parse(raw);
      
      if (entry.version !== this.CACHE_VERSION || Date.now() > entry.expiresAt) {
        localStorage.removeItem(this.PREFIX + key);
        return null;
      }

      return entry.data;
    } catch {
      return null;
    }
  }

  clearAll(): void {
    const keys = Object.keys(localStorage).filter(k => k.startsWith(this.PREFIX));
    keys.forEach(key => localStorage.removeItem(key));
    
    const sessionKeys = Object.keys(sessionStorage).filter(k => k.startsWith(this.PREFIX));
    sessionKeys.forEach(key => sessionStorage.removeItem(key));
  }
}
```

## 7. Cache ด้วย RxJS

```typescript
// reactive-cache.service.ts
import { Injectable } from '@angular/core';
import { Observable, BehaviorSubject, timer, EMPTY } from 'rxjs';
import { switchMap, tap, shareReplay, takeUntil } from 'rxjs/operators';
import { HttpClient } from '@angular/common/http';

interface CachedData<T> {
  data: T | null;
  loading: boolean;
  error: string | null;
  lastUpdated: Date | null;
}

@Injectable({ providedIn: 'root' })
export class ReactiveCacheService {
  constructor(private http: HttpClient) {}

  createAutoRefresh<T>(
    url: string,
    intervalMs = 60000
  ): Observable<CachedData<T>> {
    const subject = new BehaviorSubject<CachedData<T>>({
      data: null,
      loading: true,
      error: null,
      lastUpdated: null
    });

    const fetch$ = this.http.get<T>(url).pipe(
      tap({
        next: (data) => {
          subject.next({
            data,
            loading: false,
            error: null,
            lastUpdated: new Date()
          });
        },
        error: (err) => {
          subject.next({
            data: null,
            loading: false,
            error: err.message,
            lastUpdated: null
          });
        }
      })
    );

    // auto-refresh ทุก intervalMs
    timer(0, intervalMs).pipe(
      switchMap(() => fetch$)
    ).subscribe();

    return subject.asObservable();
  }
}
```

## 8. การใช้งาน Cache ใน Component

```typescript
// products.component.ts
import { Component, OnInit } from '@angular/core';
import { CommonModule } from '@angular/common';
import { HttpClient } from '@angular/common/http';
import { Observable, of } from 'rxjs';
import { tap } from 'rxjs/operators';
import { CacheService } from './cache.service';

interface Product {
  id: number;
  name: string;
  price: number;
}

@Component({
  selector: 'app-products',
  standalone: true,
  imports: [CommonModule],
  template: `
    <div *ngIf="loading">กำลังโหลดสินค้า...</div>
    <div *ngFor="let product of products">
      <h3>{{ product.name }}</h3>
      <p>ราคา: {{ product.price | currency:'THB':'symbol' }}</p>
    </div>
    <button (click)="refresh()">รีเฟรช</button>
    <p *ngIf="fromCache" class="cache-info">📦 ข้อมูลจาก Cache</p>
  `
})
export class ProductsComponent implements OnInit {
  products: Product[] = [];
  loading = false;
  fromCache = false;
  
  private readonly CACHE_KEY = 'products_list';
  private readonly CACHE_TTL = 10 * 60 * 1000; // 10 นาที

  constructor(
    private http: HttpClient,
    private cacheService: CacheService
  ) {}

  ngOnInit(): void {
    this.loadProducts();
  }

  loadProducts(forceRefresh = false): void {
    if (!forceRefresh) {
      const cached = this.cacheService.get<Product[]>(this.CACHE_KEY);
      if (cached) {
        this.products = cached;
        this.fromCache = true;
        return;
      }
    }

    this.loading = true;
    this.fromCache = false;

    this.http.get<Product[]>('/api/products').pipe(
      tap(products => {
        this.cacheService.set(this.CACHE_KEY, products, this.CACHE_TTL);
      })
    ).subscribe({
      next: (products) => {
        this.products = products;
        this.loading = false;
      },
      error: () => {
        this.loading = false;
      }
    });
  }

  refresh(): void {
    this.loadProducts(true);
  }
}
```

## 9. Service Worker Cache (PWA)

```json
// ngsw-config.json
{
  "$schema": "./node_modules/@angular/service-worker/config/schema.json",
  "index": "/index.html",
  "assetGroups": [
    {
      "name": "app",
      "installMode": "prefetch",
      "resources": {
        "files": [
          "/favicon.ico",
          "/index.html",
          "/manifest.webmanifest",
          "/*.css",
          "/*.js"
        ]
      }
    }
  ],
  "dataGroups": [
    {
      "name": "api-freshness",
      "urls": ["/api/products/**"],
      "cacheConfig": {
        "strategy": "freshness",
        "maxSize": 100,
        "maxAge": "3d",
        "timeout": "10s"
      }
    },
    {
      "name": "api-performance",
      "urls": ["/api/categories/**"],
      "cacheConfig": {
        "strategy": "performance",
        "maxSize": 50,
        "maxAge": "1d"
      }
    }
  ]
}
```

## 10. Cache Invalidation Strategy

```typescript
// cache-manager.service.ts
import { Injectable } from '@angular/core';
import { CacheService } from './cache.service';

@Injectable({ providedIn: 'root' })
export class CacheManagerService {
  constructor(private cacheService: CacheService) {}

  // Invalidate cache เมื่อสร้าง/แก้ไข/ลบข้อมูล
  invalidateOnWrite(entity: string, id?: number): void {
    // ล้าง cache ของ entity นั้น
    if (id) {
      this.cacheService.delete(`GET:/api/${entity}/${id}`);
    }
    // ล้าง list cache
    this.cacheService.clearPattern(new RegExp(`GET:/api/${entity}`));
  }

  // Tag-based invalidation
  private tags = new Map<string, Set<string>>();

  setWithTags(key: string, data: any, tags: string[], ttl?: number): void {
    this.cacheService.set(key, data, ttl);
    
    tags.forEach(tag => {
      if (!this.tags.has(tag)) {
        this.tags.set(tag, new Set());
      }
      this.tags.get(tag)!.add(key);
    });
  }

  invalidateByTag(tag: string): void {
    const keys = this.tags.get(tag);
    if (keys) {
      keys.forEach(key => this.cacheService.delete(key));
      this.tags.delete(tag);
    }
  }
}
```

## สรุป

| กลยุทธ์ | เหมาะกับ | TTL แนะนำ |
|---------|---------|----------|
| Memory Cache | ข้อมูลชั่วคราว | 5-15 นาที |
| SessionStorage | ข้อมูลต่อ session | 30-60 นาที |
| localStorage | ข้อมูลถาวร | 1-7 วัน |
| HTTP Cache | Static resources | 1 ชั่วโมง+ |
| Service Worker | PWA offline | ตามความต้องการ |

การเลือก caching strategy ที่เหมาะสมช่วยลด latency และปรับปรุง UX ได้อย่างมาก
