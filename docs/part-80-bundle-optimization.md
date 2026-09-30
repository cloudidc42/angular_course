# Part 80: Bundle Optimization - Tree Shaking, Code Splitting, Preloading

## ทำไม Bundle Size ถึงสำคัญ

Bundle ใหญ่ = โหลดช้า = ผู้ใช้หนี ทุก 100KB เพิ่มเวลาโหลด ~1 วินาทีบน 3G

---

## 1. Tree Shaking

Tree shaking ลบ code ที่ไม่ได้ใช้ออกจาก bundle

### ทำให้ Tree Shaking ได้ผลดีขึ้น

```typescript
// ❌ แบบนี้ import ทั้ง library
import * as _ from 'lodash';
const result = _.chunk(arr, 2);

// ✅ แบบนี้ import เฉพาะที่ใช้
import chunk from 'lodash/chunk';
const result = chunk(arr, 2);
```

```typescript
// ❌ แบบนี้ barrel import อาจทำให้ tree shaking ไม่ทำงาน
// index.ts
export * from './product.component';
export * from './product-list.component';
export * from './product-detail.component';

// ✅ แบบนี้ดีกว่า - import โดยตรง
import { ProductComponent } from './product/product.component';
```

### Pure Functions สำหรับ Tree Shaking

```typescript
// app/utils/math.utils.ts

// ใส่ /*@__PURE__*/ เพื่อบอก bundler ว่าไม่มี side effects
export const calculateTax = /*@__PURE__*/ (amount: number, rate: number): number => {
  return amount * rate;
};

export const formatCurrency = /*@__PURE__*/ (amount: number, currency = 'THB'): string => {
  return new Intl.NumberFormat('th-TH', { style: 'currency', currency }).format(amount);
};

// ถ้าไม่ใช้ฟังก์ชันเหล่านี้ จะถูก tree shake ออก
```

---

## 2. Lazy Loading

### Route-based Code Splitting

```typescript
// app/app-routing.module.ts
import { NgModule } from '@angular/core';
import { RouterModule, Routes } from '@angular/router';

const routes: Routes = [
  {
    path: '',
    loadComponent: () => import('./home/home.component').then(m => m.HomeComponent)
  },
  {
    path: 'products',
    loadChildren: () => import('./products/products.module').then(m => m.ProductsModule)
  },
  {
    path: 'admin',
    loadChildren: () => import('./admin/admin.module').then(m => m.AdminModule),
    canLoad: [AdminGuard]
  },
  {
    path: 'reports',
    loadComponent: () => import('./reports/reports.component').then(m => m.ReportsComponent),
    data: { preload: true }  // Preload hint
  }
];

@NgModule({
  imports: [RouterModule.forRoot(routes, {
    preloadingStrategy: CustomPreloadingStrategy
  })],
  exports: [RouterModule]
})
export class AppRoutingModule {}
```

### Custom Preloading Strategy

```typescript
// app/strategies/custom-preloading.strategy.ts
import { Injectable } from '@angular/core';
import { PreloadingStrategy, Route } from '@angular/router';
import { Observable, of, timer } from 'rxjs';
import { switchMap } from 'rxjs/operators';

@Injectable({ providedIn: 'root' })
export class CustomPreloadingStrategy implements PreloadingStrategy {
  
  preload(route: Route, load: () => Observable<any>): Observable<any> {
    // Preload ถ้า route มี data.preload = true
    if (route.data?.['preload']) {
      // รอ 2 วินาทีก่อน preload เพื่อไม่ block main bundle
      return timer(2000).pipe(switchMap(() => load()));
    }
    return of(null);
  }
}
```

---

## 3. Dynamic Imports

### Lazy Load Heavy Libraries

```typescript
// app/services/chart.service.ts
import { Injectable } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class ChartService {
  private chartLib: any;

  async loadChartLibrary(): Promise<void> {
    if (!this.chartLib) {
      // Dynamic import - โหลดเมื่อต้องการ
      const module = await import('chart.js');
      this.chartLib = module.Chart;
    }
  }

  async createChart(canvas: HTMLCanvasElement, data: any): Promise<any> {
    await this.loadChartLibrary();
    return new this.chartLib(canvas, data);
  }
}
```

### Lazy Load Dialogs

```typescript
// app/components/product-dialog/product-dialog.lazy.ts
import { Component, OnInit } from '@angular/core';

@Component({
  selector: 'app-product-dialog-trigger',
  template: `
    <button (click)="openDialog()">เปิด Dialog</button>
    <ng-container #dialogHost></ng-container>
  `
})
export class ProductDialogTriggerComponent {
  async openDialog(): Promise<void> {
    // Lazy load dialog component เมื่อคลิก
    const { ProductDialogComponent } = await import('./product-dialog.component');
    // สร้าง component dynamically...
  }
}
```

---

## 4. Standalone Components (Angular 14+)

```typescript
// app/features/dashboard/dashboard.component.ts
import { Component } from '@angular/core';
import { CommonModule } from '@angular/common';
import { RouterModule } from '@angular/router';
import { ChartModule } from './chart/chart.module';

@Component({
  selector: 'app-dashboard',
  standalone: true,
  imports: [
    CommonModule,
    RouterModule,
    ChartModule
    // import เฉพาะที่ต้องการ ไม่ต้องผ่าน NgModule
  ],
  template: `<h1>Dashboard</h1>`
})
export class DashboardComponent {}

// Lazy load standalone component
// routes.ts
const routes = [
  {
    path: 'dashboard',
    loadComponent: () => 
      import('./dashboard/dashboard.component')
        .then(m => m.DashboardComponent)
  }
];
```

---

## 5. Image Optimization

```typescript
// app/directives/lazy-image.directive.ts
import { Directive, ElementRef, Input, OnInit, OnDestroy } from '@angular/core';

@Directive({
  selector: '[appLazyImage]'
})
export class LazyImageDirective implements OnInit, OnDestroy {
  @Input('appLazyImage') src = '';
  @Input() placeholder = 'data:image/svg+xml,%3Csvg...%3E';
  
  private observer!: IntersectionObserver;

  constructor(private el: ElementRef<HTMLImageElement>) {}

  ngOnInit(): void {
    const img = this.el.nativeElement;
    img.src = this.placeholder;
    
    this.observer = new IntersectionObserver(
      (entries) => {
        entries.forEach(entry => {
          if (entry.isIntersecting) {
            this.loadImage();
            this.observer.unobserve(img);
          }
        });
      },
      { rootMargin: '50px' }
    );

    this.observer.observe(img);
  }

  private loadImage(): void {
    const img = this.el.nativeElement;
    img.src = this.src;
    img.classList.add('loading');
    
    img.onload = () => {
      img.classList.remove('loading');
      img.classList.add('loaded');
    };
  }

  ngOnDestroy(): void {
    this.observer?.disconnect();
  }
}
```

### NgOptimizedImage (Angular 15+)

```typescript
// app.module.ts
import { NgOptimizedImage } from '@angular/common';

@NgModule({
  imports: [NgOptimizedImage]
})
export class AppModule {}
```

```html
<!-- ใน template -->
<img 
  ngSrc="/assets/hero.jpg"
  width="800"
  height="400"
  priority          ← Preload สำหรับ LCP image
  [ngSrcset]="'800w, 400w, 200w'"
  sizes="(max-width: 768px) 100vw, 800px"
>

<!-- Lazy load ธรรมดา -->
<img 
  ngSrc="/assets/product.jpg"
  width="200"
  height="200"
  loading="lazy"
>
```

---

## 6. Angular Build Configuration

```json
// angular.json - Production config
{
  "configurations": {
    "production": {
      "budgets": [
        {
          "type": "initial",
          "maximumWarning": "500kb",
          "maximumError": "1mb"
        },
        {
          "type": "anyComponentStyle",
          "maximumWarning": "2kb",
          "maximumError": "4kb"
        }
      ],
      "outputHashing": "all",
      "optimization": {
        "scripts": true,
        "styles": {
          "minify": true,
          "inlineCritical": true
        },
        "fonts": {
          "inline": true
        }
      }
    }
  }
}
```

---

## 7. HTTP Caching

```typescript
// app/interceptors/cache.interceptor.ts
import { Injectable } from '@angular/core';
import { HttpInterceptor, HttpRequest, HttpHandler, HttpResponse } from '@angular/common/http';
import { Observable, of } from 'rxjs';
import { tap } from 'rxjs/operators';

interface CacheEntry {
  response: HttpResponse<any>;
  timestamp: number;
  ttl: number;
}

@Injectable()
export class CacheInterceptor implements HttpInterceptor {
  private cache = new Map<string, CacheEntry>();

  intercept(req: HttpRequest<any>, next: HttpHandler): Observable<any> {
    // Cache เฉพาะ GET requests
    if (req.method !== 'GET') return next.handle(req);

    const ttl = req.headers.get('X-Cache-TTL');
    if (!ttl) return next.handle(req);

    const cacheKey = req.urlWithParams;
    const cached = this.cache.get(cacheKey);
    
    if (cached && Date.now() - cached.timestamp < cached.ttl) {
      return of(cached.response.clone());
    }

    return next.handle(req).pipe(
      tap(event => {
        if (event instanceof HttpResponse) {
          this.cache.set(cacheKey, {
            response: event.clone(),
            timestamp: Date.now(),
            ttl: parseInt(ttl) * 1000
          });
        }
      })
    );
  }

  clearCache(pattern?: string): void {
    if (!pattern) {
      this.cache.clear();
      return;
    }
    
    this.cache.forEach((_, key) => {
      if (key.includes(pattern)) this.cache.delete(key);
    });
  }
}
```

---

## สรุป - Optimization Checklist

| เทคนิค | ผลประหยัด |
|--------|-----------|
| Lazy loading routes | 30-60% smaller initial bundle |
| Tree shaking | ลบ unused code |
| Image lazy loading | ลด initial network |
| Bundle splitting | ปรับปรุง cache hit rate |
| HTTP caching | ลด API calls |
| NgOptimizedImage | LCP improvement |

### Angular Build Flags

```bash
# Production build (recommended)
ng build --configuration production

# Analyze bundle
ng build --stats-json
npx webpack-bundle-analyzer dist/app/stats.json

# Source map (สำหรับ debug production)
ng build --source-map
```
