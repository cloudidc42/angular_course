# Part 46: Service Worker และ Caching Strategies

## บทนำ

Service Worker เป็น JavaScript ที่ run บน background thread แยกจาก browser window ช่วยให้แอปทำงาน offline ได้, cache resources, และรับ push notifications ในบทนี้เราจะเรียนรู้ caching strategies และ push notifications อย่างละเอียด

---

## 1. ngsw-config.json Configuration

```json
// ngsw-config.json
{
  "$schema": "./node_modules/@angular/service-worker/config/schema.json",
  "index": "/index.html",
  "assetGroups": [
    {
      "name": "app-shell",
      "installMode": "prefetch",
      "updateMode": "prefetch",
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
        "files": [
          "/assets/**",
          "/*.(svg|cur|jpg|jpeg|png|apng|webp|avif|gif|otf|ttf|woff|woff2)"
        ]
      }
    }
  ],
  "dataGroups": [
    {
      "name": "api-performance",
      "urls": ["/api/products", "/api/categories"],
      "cacheConfig": {
        "strategy": "performance",
        "maxSize": 100,
        "maxAge": "1d",
        "timeout": "5s"
      }
    },
    {
      "name": "api-freshness",
      "urls": ["/api/orders", "/api/user/**"],
      "cacheConfig": {
        "strategy": "freshness",
        "maxSize": 50,
        "maxAge": "1h",
        "timeout": "3s"
      }
    },
    {
      "name": "api-static",
      "urls": ["/api/config", "/api/constants"],
      "cacheConfig": {
        "strategy": "performance",
        "maxSize": 10,
        "maxAge": "7d"
      }
    }
  ],
  "navigationUrls": [
    "/**",
    "!/**/*.*",
    "!/**/*__*",
    "!/**/*__*/**"
  ]
}
```

### Cache Strategies อธิบาย

- **performance**: Cache-first, เร็วแต่อาจ stale
- **freshness**: Network-first, ข้อมูลใหม่แต่ต้องใช้ network

---

## 2. Custom Service Worker

```javascript
// src/sw-custom.js
// Custom Service Worker ที่เพิ่มเติมจาก ngsw

const CACHE_NAME = 'myshop-custom-v1';
const OFFLINE_PAGE = '/offline.html';

// Resources ที่ต้อง cache ทันที
const PRECACHE_URLS = [
  '/',
  '/offline.html',
  '/assets/images/placeholder.png'
];

// Install event
self.addEventListener('install', event => {
  event.waitUntil(
    caches.open(CACHE_NAME).then(cache => {
      return cache.addAll(PRECACHE_URLS);
    })
  );
  self.skipWaiting();
});

// Activate event - ล้าง cache เก่า
self.addEventListener('activate', event => {
  event.waitUntil(
    caches.keys().then(cacheNames => {
      return Promise.all(
        cacheNames
          .filter(name => name !== CACHE_NAME)
          .map(name => caches.delete(name))
      );
    })
  );
  self.clients.claim();
});

// Fetch event - ดักจับ requests
self.addEventListener('fetch', event => {
  const url = new URL(event.request.url);

  // จัดการ API requests แบบ Stale-While-Revalidate
  if (url.pathname.startsWith('/api/')) {
    event.respondWith(staleWhileRevalidate(event.request));
    return;
  }

  // จัดการ images แบบ Cache-First
  if (event.request.destination === 'image') {
    event.respondWith(cacheFirst(event.request));
    return;
  }

  // Navigate requests - ส่ง offline page ถ้าไม่มี network
  if (event.request.mode === 'navigate') {
    event.respondWith(networkFirstWithFallback(event.request));
    return;
  }
});

// Cache-First Strategy
async function cacheFirst(request) {
  const cached = await caches.match(request);
  if (cached) return cached;

  try {
    const response = await fetch(request);
    if (response.ok) {
      const cache = await caches.open(CACHE_NAME);
      cache.put(request, response.clone());
    }
    return response;
  } catch {
    return new Response('Image not available', { status: 503 });
  }
}

// Network-First with Cache Fallback
async function networkFirst(request) {
  try {
    const response = await fetch(request);
    if (response.ok) {
      const cache = await caches.open(CACHE_NAME);
      cache.put(request, response.clone());
    }
    return response;
  } catch {
    const cached = await caches.match(request);
    return cached || new Response('{"error": "offline"}', {
      status: 503,
      headers: { 'Content-Type': 'application/json' }
    });
  }
}

// Network-First with Offline Page Fallback
async function networkFirstWithFallback(request) {
  try {
    return await fetch(request);
  } catch {
    const cached = await caches.match(request);
    if (cached) return cached;
    return caches.match(OFFLINE_PAGE);
  }
}

// Stale-While-Revalidate
async function staleWhileRevalidate(request) {
  const cache = await caches.open(CACHE_NAME);
  const cached = await cache.match(request);

  // Fetch ใน background เพื่ออัปเดต cache
  const fetchPromise = fetch(request).then(response => {
    if (response.ok) {
      cache.put(request, response.clone());
    }
    return response;
  }).catch(() => null);

  // ส่ง cache ทันที หรือรอ network
  return cached || fetchPromise;
}
```

---

## 3. Push Notifications

### 3.1 Push Notification Service

```typescript
// app/services/push-notification.service.ts
import { Injectable, signal } from '@angular/core';
import { SwPush } from '@angular/service-worker';
import { HttpClient } from '@angular/common/http';
import { Observable, from, of } from 'rxjs';
import { switchMap, catchError, tap } from 'rxjs/operators';

export interface PushNotificationPayload {
  title: string;
  body: string;
  icon?: string;
  badge?: string;
  image?: string;
  actions?: { action: string; title: string; icon?: string }[];
  data?: any;
  tag?: string;
  requireInteraction?: boolean;
}

@Injectable({ providedIn: 'root' })
export class PushNotificationService {
  private readonly VAPID_PUBLIC_KEY = 'YOUR_VAPID_PUBLIC_KEY_HERE';
  private readonly API_URL = '/api/notifications';

  isSupported = signal(this.checkSupport());
  isSubscribed = signal(false);
  permissionStatus = signal<NotificationPermission>('default');

  constructor(
    private swPush: SwPush,
    private http: HttpClient
  ) {
    if (this.isSupported()) {
      this.permissionStatus.set(Notification.permission);
      this.checkExistingSubscription();
      this.listenForMessages();
    }
  }

  private checkSupport(): boolean {
    return 'Notification' in window &&
      'serviceWorker' in navigator &&
      'PushManager' in window;
  }

  private async checkExistingSubscription() {
    const sub = await this.swPush.subscription.toPromise();
    this.isSubscribed.set(!!sub);
  }

  async requestPermission(): Promise<NotificationPermission> {
    if (!this.isSupported()) return 'denied';

    const permission = await Notification.requestPermission();
    this.permissionStatus.set(permission);
    return permission;
  }

  subscribe(): Observable<PushSubscription> {
    return from(this.swPush.requestSubscription({
      serverPublicKey: this.VAPID_PUBLIC_KEY
    })).pipe(
      tap(subscription => {
        this.isSubscribed.set(true);
        // ส่ง subscription ไปยัง server
        this.saveSubscriptionToServer(subscription).subscribe();
      }),
      catchError(error => {
        console.error('Push subscription failed:', error);
        throw error;
      })
    );
  }

  unsubscribe(): Observable<void> {
    return from(this.swPush.unsubscribe()).pipe(
      tap(() => this.isSubscribed.set(false))
    );
  }

  private saveSubscriptionToServer(subscription: PushSubscription): Observable<any> {
    return this.http.post(`${this.API_URL}/subscribe`, {
      subscription: subscription.toJSON()
    });
  }

  private listenForMessages() {
    // รับ notification ที่คลิก
    this.swPush.notificationClicks.subscribe(({ action, notification }) => {
      console.log('Notification clicked:', action, notification);

      // จัดการ action
      switch (action) {
        case 'view-order':
          window.open(notification.data?.orderUrl, '_blank');
          break;
        case 'dismiss':
          // ไม่ต้องทำอะไร
          break;
        default:
          // เปิดแอป
          window.open('/', '_blank');
      }
    });

    // รับ messages จาก service worker
    this.swPush.messages.subscribe(message => {
      console.log('Push message received:', message);
    });
  }
}
```

### 3.2 Push Notification Component

```typescript
// app/components/push-settings/push-settings.component.ts
import { Component } from '@angular/core';
import { PushNotificationService } from '../../services/push-notification.service';

@Component({
  selector: 'app-push-settings',
  template: `
    <div class="push-settings">
      <h3>การแจ้งเตือน</h3>

      <!-- ไม่รองรับ -->
      <div *ngIf="!pushService.isSupported()" class="not-supported">
        <p>Browser ของคุณไม่รองรับ Push Notifications</p>
      </div>

      <!-- รองรับ -->
      <ng-container *ngIf="pushService.isSupported()">

        <!-- Permission denied -->
        <div *ngIf="pushService.permissionStatus() === 'denied'" class="permission-denied">
          <mat-icon>notifications_off</mat-icon>
          <p>คุณได้บล็อกการแจ้งเตือนไว้</p>
          <p>กรุณาเปิดอนุญาตใน Settings ของ Browser</p>
        </div>

        <!-- Ready to subscribe -->
        <ng-container *ngIf="pushService.permissionStatus() !== 'denied'">
          <div class="notification-status">
            <mat-icon [color]="pushService.isSubscribed() ? 'primary' : 'disabled'">
              {{ pushService.isSubscribed() ? 'notifications_active' : 'notifications' }}
            </mat-icon>
            <span>
              {{ pushService.isSubscribed() ? 'เปิดการแจ้งเตือนแล้ว' : 'ปิดการแจ้งเตือน' }}
            </span>
          </div>

          <mat-slide-toggle
            [checked]="pushService.isSubscribed()"
            (change)="toggleNotifications($event.checked)"
            [disabled]="isLoading"
          >
            รับการแจ้งเตือน
          </mat-slide-toggle>

          <!-- Notification Categories -->
          <div *ngIf="pushService.isSubscribed()" class="notification-categories">
            <h4>ประเภทการแจ้งเตือน</h4>
            <mat-checkbox
              *ngFor="let category of notificationCategories"
              [(ngModel)]="category.enabled"
              (change)="updatePreferences()"
            >
              {{ category.label }}
            </mat-checkbox>
          </div>
        </ng-container>
      </ng-container>

      <!-- Test button -->
      <button
        *ngIf="pushService.isSubscribed()"
        mat-stroked-button
        (click)="sendTestNotification()"
      >
        ทดสอบการแจ้งเตือน
      </button>
    </div>
  `
})
export class PushSettingsComponent {
  isLoading = false;
  notificationCategories = [
    { key: 'orders', label: 'คำสั่งซื้อใหม่', enabled: true },
    { key: 'promotions', label: 'โปรโมชั่นพิเศษ', enabled: false },
    { key: 'system', label: 'การแจ้งเตือนระบบ', enabled: true },
    { key: 'messages', label: 'ข้อความใหม่', enabled: true }
  ];

  constructor(public pushService: PushNotificationService) {}

  async toggleNotifications(enable: boolean) {
    this.isLoading = true;
    try {
      if (enable) {
        const permission = await this.pushService.requestPermission();
        if (permission === 'granted') {
          await this.pushService.subscribe().toPromise();
        }
      } else {
        await this.pushService.unsubscribe().toPromise();
      }
    } catch (error) {
      console.error('Toggle notifications failed:', error);
    } finally {
      this.isLoading = false;
    }
  }

  updatePreferences() {
    const prefs = this.notificationCategories
      .filter(c => c.enabled)
      .map(c => c.key);
    console.log('Notification preferences:', prefs);
    // ส่งไป API
  }

  sendTestNotification() {
    if (Notification.permission === 'granted') {
      new Notification('ทดสอบการแจ้งเตือน', {
        body: 'นี่คือการแจ้งเตือนทดสอบ',
        icon: '/assets/icons/icon-192x192.png'
      });
    }
  }
}
```

---

## 4. Background Sync

```typescript
// app/services/background-sync.service.ts
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { from, Observable } from 'rxjs';

interface PendingRequest {
  id: string;
  url: string;
  method: string;
  body: any;
  timestamp: number;
}

@Injectable({ providedIn: 'root' })
export class BackgroundSyncService {
  private readonly QUEUE_KEY = 'pending_requests';

  constructor(private http: HttpClient) {
    // ฟัง sync event จาก service worker
    navigator.serviceWorker?.addEventListener('message', event => {
      if (event.data?.type === 'SYNC_COMPLETE') {
        this.processQueue();
      }
    });
  }

  // เพิ่ม request เข้า queue สำหรับ offline
  queueRequest(url: string, method: string, body: any): void {
    const pending: PendingRequest = {
      id: Date.now().toString(),
      url,
      method,
      body,
      timestamp: Date.now()
    };

    const queue = this.getQueue();
    queue.push(pending);
    this.saveQueue(queue);

    // ลงทะเบียน background sync ถ้ารองรับ
    if ('serviceWorker' in navigator && 'SyncManager' in window) {
      navigator.serviceWorker.ready.then(registration => {
        (registration as any).sync.register('pending-requests');
      });
    }
  }

  // ส่ง requests ที่ค้างอยู่
  async processQueue(): Promise<void> {
    const queue = this.getQueue();
    if (queue.length === 0) return;

    const successIds: string[] = [];

    for (const request of queue) {
      try {
        await this.http[request.method.toLowerCase() as 'post' | 'put' | 'patch' | 'delete'](
          request.url,
          request.body
        ).toPromise();
        successIds.push(request.id);
      } catch {
        // ยังไม่สำเร็จ ปล่อยไว้ใน queue
      }
    }

    // ลบ requests ที่สำเร็จแล้ว
    const remaining = queue.filter(r => !successIds.includes(r.id));
    this.saveQueue(remaining);
  }

  getPendingCount(): number {
    return this.getQueue().length;
  }

  private getQueue(): PendingRequest[] {
    try {
      return JSON.parse(localStorage.getItem(this.QUEUE_KEY) || '[]');
    } catch {
      return [];
    }
  }

  private saveQueue(queue: PendingRequest[]): void {
    localStorage.setItem(this.QUEUE_KEY, JSON.stringify(queue));
  }
}
```

---

## 5. Cache Management Component

```typescript
// app/components/cache-info/cache-info.component.ts
import { Component, OnInit, signal } from '@angular/core';

interface CacheInfo {
  name: string;
  size: number;
  count: number;
}

@Component({
  selector: 'app-cache-info',
  template: `
    <div class="cache-info">
      <h3>ข้อมูล Cache</h3>

      <div *ngFor="let cache of cacheInfoList()" class="cache-item">
        <span>{{ cache.name }}</span>
        <span>{{ cache.count }} รายการ</span>
        <span>{{ formatSize(cache.size) }}</span>
      </div>

      <div class="total">
        <strong>รวม: {{ formatSize(totalSize()) }}</strong>
      </div>

      <button (click)="clearAllCaches()" class="btn-clear">
        ล้าง Cache ทั้งหมด
      </button>
    </div>
  `
})
export class CacheInfoComponent implements OnInit {
  cacheInfoList = signal<CacheInfo[]>([]);

  get totalSize(): () => number {
    return () => this.cacheInfoList().reduce((sum, c) => sum + c.size, 0);
  }

  async ngOnInit() {
    await this.loadCacheInfo();
  }

  private async loadCacheInfo() {
    if (!('caches' in window)) return;

    const cacheNames = await caches.keys();
    const infos = await Promise.all(
      cacheNames.map(async name => {
        const cache = await caches.open(name);
        const keys = await cache.keys();
        let size = 0;

        for (const request of keys) {
          const response = await cache.match(request);
          const blob = await response?.blob();
          size += blob?.size || 0;
        }

        return { name, size, count: keys.length };
      })
    );

    this.cacheInfoList.set(infos);
  }

  async clearAllCaches() {
    const cacheNames = await caches.keys();
    await Promise.all(cacheNames.map(name => caches.delete(name)));
    this.cacheInfoList.set([]);
    alert('ล้าง Cache เรียบร้อยแล้ว');
  }

  formatSize(bytes: number): string {
    if (bytes === 0) return '0 B';
    const k = 1024;
    const sizes = ['B', 'KB', 'MB', 'GB'];
    const i = Math.floor(Math.log(bytes) / Math.log(k));
    return `${parseFloat((bytes / Math.pow(k, i)).toFixed(1))} ${sizes[i]}`;
  }
}
```

---

## สรุป

| Caching Strategy | ใช้กับ |
|-----------------|-------|
| Cache-First | Static assets, images, fonts |
| Network-First | API endpoints ที่ต้องการข้อมูลใหม่ |
| Stale-While-Revalidate | ข้อมูลที่ยอมรับ stale เล็กน้อยได้ |
| Cache-Only | Offline-only resources |
| Network-Only | ไม่ต้องการ cache เลย |

Service Worker ช่วยให้แอปทำงาน offline ได้อย่างราบรื่น แต่ต้องวางแผน cache strategy ให้เหมาะกับแต่ละ endpoint
