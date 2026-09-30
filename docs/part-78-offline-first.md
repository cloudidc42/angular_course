# Part 78: Offline-First ด้วย Service Worker และ IndexedDB

## Offline-First คืออะไร

แอปที่ทำงานได้แม้ไม่มีอินเทอร์เน็ต โดยใช้ Service Worker cache ทรัพยากร และ IndexedDB เก็บข้อมูล

---

## 1. ติดตั้ง Angular PWA

```bash
ng add @angular/pwa

# ผลลัพธ์:
# - ngsw-config.json  ← Service Worker config
# - manifest.webmanifest ← PWA manifest
# - icons/
```

### ngsw-config.json

```json
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
    },
    {
      "name": "assets",
      "installMode": "lazy",
      "updateMode": "prefetch",
      "resources": {
        "files": ["/assets/**", "/*.(svg|cur|jpg|jpeg|png|apng|webp|avif|gif|otf|ttf|woff|woff2|ani)"]
      }
    }
  ],
  "dataGroups": [
    {
      "name": "api-freshness",
      "urls": ["/api/products", "/api/categories"],
      "cacheConfig": {
        "strategy": "freshness",
        "maxSize": 100,
        "maxAge": "3d",
        "timeout": "10s"
      }
    },
    {
      "name": "api-performance",
      "urls": ["/api/settings", "/api/config"],
      "cacheConfig": {
        "strategy": "performance",
        "maxSize": 10,
        "maxAge": "7d"
      }
    }
  ]
}
```

---

## 2. IndexedDB Service

```typescript
// app/services/indexed-db.service.ts
import { Injectable } from '@angular/core';
import { Observable, from, throwError } from 'rxjs';
import { catchError } from 'rxjs/operators';

interface DBConfig {
  name: string;
  version: number;
  stores: {
    name: string;
    keyPath: string;
    indexes?: { name: string; keyPath: string; unique?: boolean }[];
  }[];
}

@Injectable({ providedIn: 'root' })
export class IndexedDBService {
  private db: IDBDatabase | null = null;
  private dbReady: Promise<IDBDatabase>;

  private config: DBConfig = {
    name: 'OfflineAppDB',
    version: 1,
    stores: [
      {
        name: 'products',
        keyPath: 'id',
        indexes: [
          { name: 'category', keyPath: 'category' },
          { name: 'synced', keyPath: 'synced' }
        ]
      },
      {
        name: 'pendingOperations',
        keyPath: 'id',
        indexes: [
          { name: 'timestamp', keyPath: 'timestamp' }
        ]
      },
      {
        name: 'userProfile',
        keyPath: 'userId'
      }
    ]
  };

  constructor() {
    this.dbReady = this.initDB();
  }

  private initDB(): Promise<IDBDatabase> {
    return new Promise((resolve, reject) => {
      const request = indexedDB.open(this.config.name, this.config.version);

      request.onerror = () => reject(request.error);
      request.onsuccess = () => {
        this.db = request.result;
        resolve(this.db);
      };

      request.onupgradeneeded = (event) => {
        const db = (event.target as IDBOpenDBRequest).result;

        this.config.stores.forEach(storeConfig => {
          if (!db.objectStoreNames.contains(storeConfig.name)) {
            const store = db.createObjectStore(storeConfig.name, {
              keyPath: storeConfig.keyPath
            });

            storeConfig.indexes?.forEach(index => {
              store.createIndex(index.name, index.keyPath, {
                unique: index.unique || false
              });
            });
          }
        });
      };
    });
  }

  async getAll<T>(storeName: string): Promise<T[]> {
    const db = await this.dbReady;
    return new Promise((resolve, reject) => {
      const transaction = db.transaction(storeName, 'readonly');
      const store = transaction.objectStore(storeName);
      const request = store.getAll();

      request.onsuccess = () => resolve(request.result);
      request.onerror = () => reject(request.error);
    });
  }

  async get<T>(storeName: string, key: any): Promise<T | undefined> {
    const db = await this.dbReady;
    return new Promise((resolve, reject) => {
      const transaction = db.transaction(storeName, 'readonly');
      const store = transaction.objectStore(storeName);
      const request = store.get(key);

      request.onsuccess = () => resolve(request.result);
      request.onerror = () => reject(request.error);
    });
  }

  async put<T>(storeName: string, item: T): Promise<void> {
    const db = await this.dbReady;
    return new Promise((resolve, reject) => {
      const transaction = db.transaction(storeName, 'readwrite');
      const store = transaction.objectStore(storeName);
      const request = store.put(item);

      request.onsuccess = () => resolve();
      request.onerror = () => reject(request.error);
    });
  }

  async delete(storeName: string, key: any): Promise<void> {
    const db = await this.dbReady;
    return new Promise((resolve, reject) => {
      const transaction = db.transaction(storeName, 'readwrite');
      const store = transaction.objectStore(storeName);
      const request = store.delete(key);

      request.onsuccess = () => resolve();
      request.onerror = () => reject(request.error);
    });
  }

  async getByIndex<T>(storeName: string, indexName: string, value: any): Promise<T[]> {
    const db = await this.dbReady;
    return new Promise((resolve, reject) => {
      const transaction = db.transaction(storeName, 'readonly');
      const store = transaction.objectStore(storeName);
      const index = store.index(indexName);
      const request = index.getAll(value);

      request.onsuccess = () => resolve(request.result);
      request.onerror = () => reject(request.error);
    });
  }

  async bulkPut<T>(storeName: string, items: T[]): Promise<void> {
    const db = await this.dbReady;
    return new Promise((resolve, reject) => {
      const transaction = db.transaction(storeName, 'readwrite');
      const store = transaction.objectStore(storeName);

      items.forEach(item => store.put(item));
      
      transaction.oncomplete = () => resolve();
      transaction.onerror = () => reject(transaction.error);
    });
  }

  async clear(storeName: string): Promise<void> {
    const db = await this.dbReady;
    return new Promise((resolve, reject) => {
      const transaction = db.transaction(storeName, 'readwrite');
      const store = transaction.objectStore(storeName);
      const request = store.clear();

      request.onsuccess = () => resolve();
      request.onerror = () => reject(request.error);
    });
  }
}
```

---

## 3. Offline Sync Service

```typescript
// app/services/offline-sync.service.ts
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { BehaviorSubject, fromEvent, merge, Observable } from 'rxjs';
import { map, switchMap, retry, catchError } from 'rxjs/operators';
import { IndexedDBService } from './indexed-db.service';

interface PendingOperation {
  id: string;
  type: 'CREATE' | 'UPDATE' | 'DELETE';
  endpoint: string;
  data: any;
  timestamp: number;
  retries: number;
}

@Injectable({ providedIn: 'root' })
export class OfflineSyncService {
  private isOnline$ = new BehaviorSubject<boolean>(navigator.onLine);
  private syncStatus$ = new BehaviorSubject<'idle' | 'syncing' | 'error'>('idle');
  private pendingCount$ = new BehaviorSubject<number>(0);

  constructor(
    private http: HttpClient,
    private db: IndexedDBService
  ) {
    this.setupOnlineListener();
    this.updatePendingCount();
  }

  private setupOnlineListener(): void {
    merge(
      fromEvent(window, 'online').pipe(map(() => true)),
      fromEvent(window, 'offline').pipe(map(() => false))
    ).subscribe(isOnline => {
      this.isOnline$.next(isOnline);
      if (isOnline) {
        this.syncPendingOperations();
      }
    });
  }

  get isOnline(): boolean {
    return this.isOnline$.value;
  }

  get onlineStatus$(): Observable<boolean> {
    return this.isOnline$.asObservable();
  }

  get syncStatus(): Observable<string> {
    return this.syncStatus$.asObservable();
  }

  // Queue operation สำหรับ sync ทีหลัง
  async queueOperation(operation: Omit<PendingOperation, 'id' | 'timestamp' | 'retries'>): Promise<void> {
    const pending: PendingOperation = {
      id: `op-${Date.now()}-${Math.random().toString(36).slice(2)}`,
      ...operation,
      timestamp: Date.now(),
      retries: 0
    };

    await this.db.put('pendingOperations', pending);
    await this.updatePendingCount();

    if (this.isOnline) {
      await this.syncPendingOperations();
    }
  }

  async syncPendingOperations(): Promise<void> {
    if (this.syncStatus$.value === 'syncing') return;

    const operations = await this.db.getAll<PendingOperation>('pendingOperations');
    if (operations.length === 0) return;

    this.syncStatus$.next('syncing');

    // Sort by timestamp
    const sorted = operations.sort((a, b) => a.timestamp - b.timestamp);

    let hasError = false;
    for (const op of sorted) {
      try {
        await this.executeOperation(op);
        await this.db.delete('pendingOperations', op.id);
      } catch (error) {
        hasError = true;
        
        // Increment retry count
        op.retries++;
        if (op.retries >= 3) {
          console.error('Operation failed after 3 retries, removing:', op);
          await this.db.delete('pendingOperations', op.id);
        } else {
          await this.db.put('pendingOperations', op);
        }
      }
    }

    this.syncStatus$.next(hasError ? 'error' : 'idle');
    await this.updatePendingCount();
  }

  private async executeOperation(op: PendingOperation): Promise<any> {
    const { type, endpoint, data } = op;

    switch (type) {
      case 'CREATE':
        return this.http.post(endpoint, data).toPromise();
      case 'UPDATE':
        return this.http.put(endpoint, data).toPromise();
      case 'DELETE':
        return this.http.delete(endpoint).toPromise();
    }
  }

  private async updatePendingCount(): Promise<void> {
    const ops = await this.db.getAll<PendingOperation>('pendingOperations');
    this.pendingCount$.next(ops.length);
  }

  get pendingOperations$(): Observable<number> {
    return this.pendingCount$.asObservable();
  }
}
```

---

## 4. Offline-Aware Product Service

```typescript
// app/services/product-offline.service.ts
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable, from } from 'rxjs';
import { tap, catchError, switchMap } from 'rxjs/operators';
import { IndexedDBService } from './indexed-db.service';
import { OfflineSyncService } from './offline-sync.service';

interface Product {
  id: string;
  name: string;
  price: number;
  category: string;
  stock: number;
  synced: boolean;
  updatedAt: number;
}

@Injectable({ providedIn: 'root' })
export class ProductOfflineService {
  private apiUrl = '/api/products';

  constructor(
    private http: HttpClient,
    private db: IndexedDBService,
    private syncService: OfflineSyncService
  ) {}

  // ดึงข้อมูล: ลอง API ก่อน, ถ้าไม่ได้ใช้ cache
  getProducts(): Observable<Product[]> {
    if (this.syncService.isOnline) {
      return this.http.get<Product[]>(this.apiUrl).pipe(
        tap(async products => {
          // บันทึก cache ใน IndexedDB
          const cached = products.map(p => ({ ...p, synced: true }));
          await this.db.bulkPut('products', cached);
        }),
        catchError(async () => {
          // Fallback ใช้ cache
          console.warn('API ล้มเหลว ใช้ข้อมูล cache');
          return await this.db.getAll<Product>('products');
        })
      );
    } else {
      // Offline: ใช้ cache ทันที
      return from(this.db.getAll<Product>('products'));
    }
  }

  async createProduct(product: Omit<Product, 'id' | 'synced' | 'updatedAt'>): Promise<Product> {
    const newProduct: Product = {
      ...product,
      id: `local-${Date.now()}`,
      synced: false,
      updatedAt: Date.now()
    };

    // บันทึกใน IndexedDB ก่อนเสมอ
    await this.db.put('products', newProduct);

    if (this.syncService.isOnline) {
      try {
        const saved = await this.http.post<Product>(this.apiUrl, product).toPromise();
        if (saved) {
          // อัปเดต local record ด้วย server ID
          await this.db.delete('products', newProduct.id);
          await this.db.put('products', { ...saved, synced: true });
          return saved;
        }
      } catch {
        // Queue สำหรับ sync ทีหลัง
        await this.syncService.queueOperation({
          type: 'CREATE',
          endpoint: this.apiUrl,
          data: product
        });
      }
    } else {
      await this.syncService.queueOperation({
        type: 'CREATE',
        endpoint: this.apiUrl,
        data: product
      });
    }

    return newProduct;
  }

  async updateProduct(id: string, updates: Partial<Product>): Promise<void> {
    const current = await this.db.get<Product>('products', id);
    if (!current) throw new Error('Product not found');

    const updated = { ...current, ...updates, synced: false, updatedAt: Date.now() };
    await this.db.put('products', updated);

    if (this.syncService.isOnline) {
      try {
        await this.http.put(`${this.apiUrl}/${id}`, updates).toPromise();
        await this.db.put('products', { ...updated, synced: true });
      } catch {
        await this.syncService.queueOperation({
          type: 'UPDATE',
          endpoint: `${this.apiUrl}/${id}`,
          data: updates
        });
      }
    } else {
      await this.syncService.queueOperation({
        type: 'UPDATE',
        endpoint: `${this.apiUrl}/${id}`,
        data: updates
      });
    }
  }
}
```

---

## 5. Offline Status Component

```typescript
// app/components/offline-banner/offline-banner.component.ts
import { Component, OnInit } from '@angular/core';
import { Observable, combineLatest } from 'rxjs';
import { map } from 'rxjs/operators';
import { OfflineSyncService } from '../../services/offline-sync.service';

@Component({
  selector: 'app-offline-banner',
  template: `
    <ng-container *ngIf="status$ | async as status">
      <!-- Offline Banner -->
      <div class="offline-banner" *ngIf="!status.isOnline" role="alert">
        <span>📵 คุณกำลังใช้งานแบบออฟไลน์</span>
        <span *ngIf="status.pending > 0">
          มีการเปลี่ยนแปลง {{ status.pending }} รายการรอ sync
        </span>
      </div>

      <!-- Syncing Banner -->
      <div class="sync-banner" *ngIf="status.isOnline && status.syncStatus === 'syncing'">
        <div class="spinner-small"></div>
        กำลัง sync ข้อมูล...
      </div>

      <!-- Back Online Banner -->
      <div class="online-banner" *ngIf="status.isOnline && status.syncStatus === 'idle' && status.pending === 0">
        ✅ เชื่อมต่ออินเทอร์เน็ตแล้ว
      </div>
    </ng-container>
  `,
  styles: [`
    .offline-banner {
      position: fixed; top: 0; left: 0; right: 0;
      background: #333; color: white;
      padding: 8px 16px;
      display: flex; justify-content: space-between;
      z-index: 9999; font-size: 14px;
    }
    .sync-banner {
      position: fixed; bottom: 16px; right: 16px;
      background: #2196f3; color: white;
      padding: 8px 16px; border-radius: 24px;
      display: flex; align-items: center; gap: 8px;
      z-index: 9999; font-size: 13px;
    }
    .online-banner {
      position: fixed; bottom: 16px; right: 16px;
      background: #4caf50; color: white;
      padding: 8px 16px; border-radius: 24px;
      z-index: 9999; font-size: 13px;
      animation: fadeOut 3s forwards;
    }
    @keyframes fadeOut { 0% { opacity: 1; } 70% { opacity: 1; } 100% { opacity: 0; } }
    .spinner-small {
      width: 16px; height: 16px;
      border: 2px solid rgba(255,255,255,0.3);
      border-top-color: white;
      border-radius: 50%;
      animation: spin 0.8s linear infinite;
    }
    @keyframes spin { to { transform: rotate(360deg); } }
  `]
})
export class OfflineBannerComponent {
  status$ = combineLatest([
    this.syncService.onlineStatus$,
    this.syncService.syncStatus,
    this.syncService.pendingOperations$
  ]).pipe(
    map(([isOnline, syncStatus, pending]) => ({
      isOnline,
      syncStatus,
      pending
    }))
  );

  constructor(private syncService: OfflineSyncService) {}
}
```

---

## สรุป

| เทคโนโลยี | ใช้สำหรับ |
|-----------|----------|
| Service Worker | Cache static assets & API |
| IndexedDB | เก็บข้อมูล structured |
| Background Sync | Retry failed requests |
| Online/Offline events | ตรวจสอบ connectivity |

### Best Practices

1. Cache-first สำหรับ static assets
2. Network-first สำหรับ API data
3. Queue operations เมื่อ offline
4. แจ้ง user เมื่อ offline
5. Handle conflicts เมื่อ sync
6. Test offline ด้วย Chrome DevTools
