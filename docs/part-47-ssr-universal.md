# Part 47: Server-Side Rendering (SSR) และ Angular Universal

## บทนำ

Server-Side Rendering (SSR) คือการ render HTML บน server แทนที่จะ render บน client (browser) ช่วยให้แอปโหลดเร็วขึ้นและ SEO ดีขึ้น Angular 16+ มี built-in hydration ที่ช่วยให้ client สามารถ reuse DOM จาก server ได้

---

## 1. การติดตั้ง SSR

```bash
# Angular 17+ (แนะนำ)
ng add @angular/ssr

# Angular 16
ng add @nguniversal/express-engine

# คำสั่งนี้จะสร้าง:
# - server.ts (Express server)
# - app.server.module.ts (สำหรับ Angular < 17)
# - อัปเดต angular.json
```

---

## 2. Application Setup (Angular 17+)

```typescript
// src/app/app.config.ts
import { ApplicationConfig, provideZoneChangeDetection } from '@angular/core';
import { provideRouter } from '@angular/router';
import { provideClientHydration } from '@angular/platform-browser';
import { provideHttpClient, withFetch } from '@angular/common/http';
import { routes } from './app.routes';

export const appConfig: ApplicationConfig = {
  providers: [
    provideZoneChangeDetection({ eventCoalescing: true }),
    provideRouter(routes),
    provideClientHydration(),   // เปิด Hydration
    provideHttpClient(withFetch()) // ใช้ fetch API แทน XMLHttpRequest
  ]
};
```

```typescript
// src/app/app.config.server.ts
import { mergeApplicationConfig, ApplicationConfig } from '@angular/core';
import { provideServerRendering } from '@angular/platform-server';
import { appConfig } from './app.config';

const serverConfig: ApplicationConfig = {
  providers: [
    provideServerRendering()
  ]
};

export const config = mergeApplicationConfig(appConfig, serverConfig);
```

---

## 3. Express Server

```typescript
// server.ts
import { APP_BASE_HREF } from '@angular/common';
import { CommonEngine } from '@angular/ssr';
import express from 'express';
import { fileURLToPath } from 'node:url';
import { dirname, join, resolve } from 'node:path';
import bootstrap from './src/main.server';

export function app(): express.Express {
  const server = express();
  const serverDistFolder = dirname(fileURLToPath(import.meta.url));
  const browserDistFolder = resolve(serverDistFolder, '../browser');
  const indexHtml = join(serverDistFolder, 'index.server.html');

  const commonEngine = new CommonEngine();

  server.set('view engine', 'html');
  server.set('views', browserDistFolder);

  // Static files
  server.get('*.*', express.static(browserDistFolder, {
    maxAge: '1y'
  }));

  // SSR Routes
  server.get('*', (req, res, next) => {
    const { protocol, originalUrl, baseUrl, headers } = req;

    commonEngine
      .render({
        bootstrap,
        documentFilePath: indexHtml,
        url: `${protocol}://${headers.host}${originalUrl}`,
        publicPath: browserDistFolder,
        providers: [
          { provide: APP_BASE_HREF, useValue: baseUrl }
        ]
      })
      .then(html => res.send(html))
      .catch(err => next(err));
  });

  return server;
}

function run(): void {
  const port = process.env['PORT'] || 4000;
  const server = app();
  server.listen(port, () => {
    console.log(`Node Express server running at http://localhost:${port}`);
  });
}

run();
```

---

## 4. Platform Detection

```typescript
// app/utils/platform.utils.ts
import { inject, PLATFORM_ID } from '@angular/core';
import { isPlatformBrowser, isPlatformServer } from '@angular/common';

export function usePlatform() {
  const platformId = inject(PLATFORM_ID);
  return {
    isBrowser: isPlatformBrowser(platformId),
    isServer: isPlatformServer(platformId)
  };
}
```

```typescript
// app/services/storage.service.ts
import { Injectable, PLATFORM_ID, inject } from '@angular/core';
import { isPlatformBrowser } from '@angular/common';

@Injectable({ providedIn: 'root' })
export class StorageService {
  private isBrowser: boolean;

  constructor() {
    const platformId = inject(PLATFORM_ID);
    this.isBrowser = isPlatformBrowser(platformId);
  }

  getItem(key: string): string | null {
    if (!this.isBrowser) return null;
    try {
      return localStorage.getItem(key);
    } catch {
      return null;
    }
  }

  setItem(key: string, value: string): void {
    if (!this.isBrowser) return;
    try {
      localStorage.setItem(key, value);
    } catch { }
  }

  removeItem(key: string): void {
    if (!this.isBrowser) return;
    try {
      localStorage.removeItem(key);
    } catch { }
  }
}
```

---

## 5. SSR-Safe Component

```typescript
// app/components/ssr-safe/ssr-safe.component.ts
import {
  Component,
  OnInit,
  PLATFORM_ID,
  inject,
  signal,
  Inject
} from '@angular/core';
import { isPlatformBrowser } from '@angular/common';
import { HttpClient } from '@angular/common/http';
import { TransferState, makeStateKey } from '@angular/core';

// Key สำหรับ TransferState
const PRODUCTS_KEY = makeStateKey<any[]>('products');

@Component({
  selector: 'app-ssr-safe',
  template: `
    <div class="products-page">
      <h2>สินค้าของเรา</h2>

      <!-- Loading state -->
      <div *ngIf="isLoading()" class="loading">กำลังโหลด...</div>

      <!-- Products list -->
      <div class="products-grid">
        <div
          *ngFor="let product of products(); trackBy: trackProduct"
          class="product-card"
        >
          <img
            [src]="product.image"
            [alt]="product.name"
            loading="lazy"
            width="300"
            height="200"
          >
          <h3>{{ product.name }}</h3>
          <p>฿{{ product.price | number:'1.2-2' }}</p>
        </div>
      </div>

      <!-- Browser-only features -->
      <ng-container *ngIf="isBrowser">
        <div class="cart-summary">
          สินค้าในตะกร้า: {{ cartCount() }} ชิ้น
        </div>
      </ng-container>
    </div>
  `
})
export class SsrSafeComponent implements OnInit {
  products = signal<any[]>([]);
  isLoading = signal(true);
  cartCount = signal(0);
  isBrowser: boolean;

  constructor(
    private http: HttpClient,
    private transferState: TransferState,
    @Inject(PLATFORM_ID) platformId: Object
  ) {
    this.isBrowser = isPlatformBrowser(platformId);
  }

  ngOnInit() {
    // ตรวจสอบ TransferState ก่อน
    const cachedProducts = this.transferState.get(PRODUCTS_KEY, null);

    if (cachedProducts) {
      // ใช้ข้อมูลจาก server
      this.products.set(cachedProducts);
      this.isLoading.set(false);
      // ลบออกจาก state หลังใช้แล้ว
      this.transferState.remove(PRODUCTS_KEY);
    } else {
      this.fetchProducts();
    }

    // Browser-only
    if (this.isBrowser) {
      this.loadCartCount();
    }
  }

  private fetchProducts() {
    this.http.get<any[]>('/api/products').subscribe({
      next: products => {
        this.products.set(products);
        this.isLoading.set(false);

        // เก็บใน TransferState เพื่อส่งไปยัง client
        if (!this.isBrowser) {
          this.transferState.set(PRODUCTS_KEY, products);
        }
      },
      error: () => this.isLoading.set(false)
    });
  }

  private loadCartCount() {
    try {
      const cart = JSON.parse(localStorage.getItem('cart') || '[]');
      this.cartCount.set(cart.length);
    } catch { }
  }

  trackProduct(index: number, product: any): number {
    return product.id;
  }
}
```

---

## 6. HTTP State Transfer

```typescript
// app/interceptors/transfer-state.interceptor.ts
import { inject } from '@angular/core';
import {
  HttpInterceptorFn,
  HttpRequest,
  HttpHandlerFn,
  HttpResponse
} from '@angular/common/http';
import { TransferState, makeStateKey } from '@angular/core';
import { PLATFORM_ID } from '@angular/core';
import { isPlatformBrowser, isPlatformServer } from '@angular/common';
import { of } from 'rxjs';
import { tap } from 'rxjs/operators';

export const transferStateInterceptor: HttpInterceptorFn = (
  req: HttpRequest<any>,
  next: HttpHandlerFn
) => {
  const transferState = inject(TransferState);
  const platformId = inject(PLATFORM_ID);

  // เฉพาะ GET requests
  if (req.method !== 'GET') return next(req);

  const stateKey = makeStateKey<any>(`http_${req.url}`);

  if (isPlatformBrowser(platformId)) {
    // Client: ตรวจสอบ TransferState
    const cachedResponse = transferState.get(stateKey, null);
    if (cachedResponse) {
      transferState.remove(stateKey);
      return of(new HttpResponse({ body: cachedResponse, status: 200 }));
    }
  }

  return next(req).pipe(
    tap(event => {
      if (event instanceof HttpResponse && isPlatformServer(platformId)) {
        // Server: เก็บผลลัพธ์ใน TransferState
        transferState.set(stateKey, event.body);
      }
    })
  );
};
```

---

## 7. Prerendering (Static Site Generation)

```json
// angular.json - เพิ่ม prerender configuration
{
  "projects": {
    "myapp": {
      "architect": {
        "prerender": {
          "builder": "@angular/ssr:prerender",
          "options": {
            "routes": [
              "/",
              "/about",
              "/products",
              "/contact"
            ],
            "discoverRoutes": true
          }
        }
      }
    }
  }
}
```

```bash
# รัน prerendering
ng run myapp:prerender

# หรือ build พร้อม prerender
ng build --prerender
```

---

## 8. Common SSR Mistakes และวิธีแก้

```typescript
// ❌ ไม่ดี - เข้าถึง window/document โดยตรง
@Component({
  template: `<p>{{ width }}</p>`
})
export class BadComponent implements OnInit {
  width = 0;

  ngOnInit() {
    this.width = window.innerWidth; // จะ error บน server
  }
}

// ✅ ดี - ตรวจสอบ platform ก่อน
@Component({
  template: `<p>{{ width }}</p>`
})
export class GoodComponent implements OnInit {
  width = 0;

  constructor(@Inject(PLATFORM_ID) private platformId: Object) {}

  ngOnInit() {
    if (isPlatformBrowser(this.platformId)) {
      this.width = window.innerWidth;
    }
  }
}

// ✅ ดียิ่งขึ้น - ใช้ Angular CDK
import { BreakpointObserver } from '@angular/cdk/layout';

@Component({ template: `<p>{{ isMobile ? 'mobile' : 'desktop' }}</p>` })
export class BestComponent {
  isMobile = false;

  constructor(private breakpoint: BreakpointObserver) {
    this.breakpoint.observe('(max-width: 768px)').subscribe(
      result => this.isMobile = result.matches
    );
  }
}
```

---

## สรุป

SSR ใน Angular ต้องระวัง:

| ปัญหา | วิธีแก้ |
|------|--------|
| window/document ไม่มีบน server | ใช้ isPlatformBrowser() |
| localStorage ไม่มีบน server | ใช้ StorageService ที่ตรวจสอบ platform |
| Duplicate HTTP calls | ใช้ TransferState หรือ transferStateInterceptor |
| Hydration mismatch | ทำให้ server/client render เหมือนกัน |

Angular 16+ Hydration ช่วยให้ client reuse DOM จาก server แทนที่จะ re-render ใหม่ทั้งหมด ทำให้ Time to Interactive (TTI) ดีขึ้นอย่างมีนัยสำคัญ
