# Part 49: Performance Optimization ใน Angular

## บทนำ

การเพิ่มประสิทธิภาพ Angular application มีหลายมิติ ตั้งแต่ Change Detection, TrackBy, Lazy Loading Images ไปจนถึง Bundle Analysis ในบทนี้เราจะครอบคลุมทุกเทคนิคที่สำคัญ

---

## 1. OnPush Change Detection

```typescript
// app/components/product-card/product-card.component.ts
import {
  Component,
  Input,
  Output,
  EventEmitter,
  ChangeDetectionStrategy
} from '@angular/core';

interface Product {
  id: number;
  name: string;
  price: number;
  image: string;
  stock: number;
}

@Component({
  selector: 'app-product-card',
  changeDetection: ChangeDetectionStrategy.OnPush, // เปิด OnPush
  template: `
    <div class="card" (click)="select.emit(product)">
      <img
        [src]="product.image"
        [alt]="product.name"
        loading="lazy"
        width="300"
        height="200"
      >
      <h3>{{ product.name }}</h3>
      <p class="price">฿{{ product.price | number:'1.0-0' }}</p>
      <span class="stock" [class.low]="product.stock < 10">
        {{ product.stock > 0 ? 'มีสินค้า (' + product.stock + ')' : 'หมด' }}
      </span>
      <button
        (click)="addToCart.emit(product); $event.stopPropagation()"
        [disabled]="product.stock === 0"
      >
        เพิ่มลงตะกร้า
      </button>
    </div>
  `
})
export class ProductCardComponent {
  @Input() product!: Product;
  @Output() select = new EventEmitter<Product>();
  @Output() addToCart = new EventEmitter<Product>();
}
```

---

## 2. TrackBy Function

```typescript
// app/components/product-list/product-list.component.ts
import {
  Component,
  ChangeDetectionStrategy,
  signal,
  computed
} from '@angular/core';

interface Product {
  id: number;
  name: string;
  price: number;
  category: string;
}

@Component({
  selector: 'app-product-list',
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <div class="filters">
      <button
        *ngFor="let cat of categories()"
        (click)="selectCategory(cat)"
        [class.active]="selectedCategory() === cat"
      >
        {{ cat }}
      </button>
    </div>

    <!-- ❌ ไม่ดี - Angular จะ re-render ทั้งหมดเมื่อ array เปลี่ยน -->
    <!-- <div *ngFor="let product of filteredProducts()"> -->

    <!-- ✅ ดี - ใช้ trackBy เพื่อบอก Angular ว่า item ไหนเหมือนเดิม -->
    <div class="product-grid">
      <app-product-card
        *ngFor="let product of filteredProducts(); trackBy: trackByProductId"
        [product]="product"
        (addToCart)="onAddToCart($event)"
      ></app-product-card>
    </div>

    <p>แสดง {{ filteredProducts().length }} จาก {{ products().length }} รายการ</p>
  `
})
export class ProductListComponent {
  products = signal<Product[]>([
    { id: 1, name: 'สินค้า A', price: 100, category: 'อิเล็กทรอนิกส์' },
    { id: 2, name: 'สินค้า B', price: 200, category: 'เสื้อผ้า' },
    { id: 3, name: 'สินค้า C', price: 300, category: 'อิเล็กทรอนิกส์' }
  ]);

  selectedCategory = signal<string>('ทั้งหมด');

  categories = computed(() => ['ทั้งหมด', ...new Set(this.products().map(p => p.category))]);

  filteredProducts = computed(() => {
    const cat = this.selectedCategory();
    return cat === 'ทั้งหมด'
      ? this.products()
      : this.products().filter(p => p.category === cat);
  });

  // trackBy ลด DOM manipulation ที่ไม่จำเป็น
  trackByProductId(index: number, product: Product): number {
    return product.id;
  }

  selectCategory(category: string) {
    this.selectedCategory.set(category);
  }

  onAddToCart(product: Product) {
    console.log('Add to cart:', product);
  }
}
```

---

## 3. Lazy Loading Images

```typescript
// app/directives/lazy-image.directive.ts
import {
  Directive,
  ElementRef,
  Input,
  OnInit,
  OnDestroy
} from '@angular/core';

@Directive({
  selector: 'img[appLazyLoad]',
  standalone: true
})
export class LazyImageDirective implements OnInit, OnDestroy {
  @Input('appLazyLoad') src!: string;
  @Input() placeholder = '/assets/images/placeholder.webp';
  @Input() errorImage = '/assets/images/error.webp';

  private observer?: IntersectionObserver;

  constructor(private el: ElementRef<HTMLImageElement>) {}

  ngOnInit() {
    const img = this.el.nativeElement;

    // แสดง placeholder ก่อน
    img.src = this.placeholder;
    img.loading = 'lazy';

    if ('IntersectionObserver' in window) {
      this.observer = new IntersectionObserver(
        (entries) => {
          entries.forEach(entry => {
            if (entry.isIntersecting) {
              this.loadImage(img);
              this.observer?.disconnect();
            }
          });
        },
        { rootMargin: '200px 0px' } // โหลดก่อน 200px
      );
      this.observer.observe(img);
    } else {
      // Fallback สำหรับ browsers เก่า
      this.loadImage(img);
    }
  }

  private loadImage(img: HTMLImageElement) {
    const tempImg = new Image();

    tempImg.onload = () => {
      img.src = this.src;
      img.classList.add('loaded');
    };

    tempImg.onerror = () => {
      img.src = this.errorImage;
    };

    tempImg.src = this.src;
  }

  ngOnDestroy() {
    this.observer?.disconnect();
  }
}
```

```html
<!-- การใช้งาน -->
<img
  appLazyLoad="/api/products/1/image"
  placeholder="/assets/placeholder.webp"
  alt="สินค้า"
  width="300"
  height="200"
>

<!-- หรือใช้ loading="lazy" ที่ built-in (modern browsers) -->
<img
  [src]="product.image"
  [alt]="product.name"
  loading="lazy"
  width="300"
  height="200"
>
```

---

## 4. Pure Pipes สำหรับ Expensive Computation

```typescript
// app/pipes/filter.pipe.ts
import { Pipe, PipeTransform } from '@angular/core';

// Pure pipe - คำนวณใหม่เฉพาะเมื่อ input เปลี่ยน
@Pipe({
  name: 'filterProducts',
  standalone: true,
  pure: true // default คือ true
})
export class FilterProductsPipe implements PipeTransform {
  transform(products: any[], searchTerm: string, category: string): any[] {
    if (!products) return [];

    return products.filter(product => {
      const matchesSearch = !searchTerm ||
        product.name.toLowerCase().includes(searchTerm.toLowerCase());
      const matchesCategory = !category || category === 'ทั้งหมด' ||
        product.category === category;

      return matchesSearch && matchesCategory;
    });
  }
}
```

```html
<!-- ใช้ pipe แทนการ filter ใน component -->
<!-- ✅ Pure pipe จะคำนวณใหม่เฉพาะเมื่อ products, searchTerm, หรือ category เปลี่ยน -->
<app-product-card
  *ngFor="let p of products | filterProducts:searchTerm:selectedCategory; trackBy: trackById"
  [product]="p"
></app-product-card>
```

---

## 5. Memoization

```typescript
// app/utils/memoize.ts
export function memoize<T extends (...args: any[]) => any>(fn: T): T {
  const cache = new Map<string, ReturnType<T>>();

  return ((...args: any[]) => {
    const key = JSON.stringify(args);
    if (cache.has(key)) {
      return cache.get(key);
    }
    const result = fn(...args);
    cache.set(key, result);
    return result;
  }) as T;
}

// ตัวอย่างการใช้งาน
export const calculateDiscount = memoize((price: number, discountPercent: number): number => {
  // Heavy calculation
  return price * (1 - discountPercent / 100);
});

export const formatCurrency = memoize((amount: number, currency = 'THB'): string => {
  return new Intl.NumberFormat('th-TH', {
    style: 'currency',
    currency
  }).format(amount);
});
```

---

## 6. Bundle Analysis

```bash
# ติดตั้ง webpack-bundle-analyzer
npm install --save-dev webpack-bundle-analyzer

# Build พร้อม stats
ng build --stats-json

# วิเคราะห์ bundle
npx webpack-bundle-analyzer dist/myapp/browser/stats.json

# หรือใช้ source-map-explorer
npm install --save-dev source-map-explorer
ng build --source-map
npx source-map-explorer 'dist/myapp/browser/*.js'
```

### การอ่าน Bundle Analysis

```typescript
// angular.json - ตั้งค่า budget เพื่อเตือนเมื่อ bundle ใหญ่เกินไป
{
  "configurations": {
    "production": {
      "budgets": [
        {
          "type": "initial",
          "maximumWarning": "2mb",
          "maximumError": "5mb"
        },
        {
          "type": "anyComponentStyle",
          "maximumWarning": "6kb",
          "maximumError": "10kb"
        }
      ]
    }
  }
}
```

---

## 7. Code Splitting และ Lazy Loading

```typescript
// app/app.routes.ts - Lazy Loading routes
export const routes: Routes = [
  { path: '', loadComponent: () => import('./pages/home/home.component').then(m => m.HomeComponent) },
  {
    path: 'products',
    loadChildren: () => import('./features/products/products.routes').then(m => m.PRODUCT_ROUTES)
  },
  {
    path: 'admin',
    loadChildren: () => import('./features/admin/admin.routes').then(m => m.ADMIN_ROUTES),
    canActivate: [authGuard]
  }
];
```

```typescript
// app/features/products/products.routes.ts
export const PRODUCT_ROUTES: Routes = [
  {
    path: '',
    loadComponent: () => import('./product-list/product-list.component').then(m => m.ProductListComponent)
  },
  {
    path: ':id',
    loadComponent: () => import('./product-detail/product-detail.component').then(m => m.ProductDetailComponent)
  }
];
```

---

## 8. Virtual Scrolling สำหรับ Large Lists

```typescript
// app/components/large-list/large-list.component.ts
import { Component, OnInit } from '@angular/core';
import { ScrollingModule } from '@angular/cdk/scrolling';
import { ChangeDetectionStrategy } from '@angular/core';

@Component({
  selector: 'app-large-list',
  standalone: true,
  imports: [ScrollingModule],
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <h3>รายการสินค้า ({{ items.length }} รายการ)</h3>

    <!-- Virtual Scroll - render เฉพาะ items ที่มองเห็น -->
    <cdk-virtual-scroll-viewport
      itemSize="80"
      class="list-viewport"
    >
      <div
        *cdkVirtualFor="let item of items; trackBy: trackItem; templateCacheSize: 20"
        class="list-item"
      >
        <span class="item-id">#{{ item.id }}</span>
        <span class="item-name">{{ item.name }}</span>
        <span class="item-price">฿{{ item.price | number }}</span>
      </div>
    </cdk-virtual-scroll-viewport>
  `,
  styles: [`
    .list-viewport { height: 500px; }
    .list-item { height: 80px; display: flex; align-items: center; gap: 16px; padding: 0 16px; border-bottom: 1px solid #eee; }
  `]
})
export class LargeListComponent implements OnInit {
  items: any[] = [];

  ngOnInit() {
    // 50,000 items - ไม่มี Virtual Scroll จะ freeze browser
    this.items = Array.from({ length: 50000 }, (_, i) => ({
      id: i + 1,
      name: `สินค้า ${i + 1}`,
      price: Math.round(Math.random() * 10000)
    }));
  }

  trackItem(index: number, item: any): number {
    return item.id;
  }
}
```

---

## 9. Preloading Strategies

```typescript
// app/strategies/custom-preload.strategy.ts
import { Injectable } from '@angular/core';
import { PreloadingStrategy, Route } from '@angular/router';
import { Observable, of, timer } from 'rxjs';
import { switchMap } from 'rxjs/operators';

@Injectable({ providedIn: 'root' })
export class SelectivePreloadingStrategy implements PreloadingStrategy {
  preload(route: Route, load: () => Observable<any>): Observable<any> {
    if (route.data?.['preload']) {
      const delay = route.data?.['preloadDelay'] || 0;
      return timer(delay).pipe(switchMap(() => load()));
    }
    return of(null);
  }
}
```

```typescript
// app/app.routes.ts
{
  path: 'products',
  loadChildren: () => import('./features/products/products.routes').then(m => m.PRODUCT_ROUTES),
  data: { preload: true, preloadDelay: 1000 } // preload หลังจาก 1 วินาที
}
```

```typescript
// app/app.config.ts
import { provideRouter, withPreloading } from '@angular/router';
import { SelectivePreloadingStrategy } from './strategies/custom-preload.strategy';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes, withPreloading(SelectivePreloadingStrategy))
  ]
};
```

---

## 10. Performance Monitoring

```typescript
// app/services/performance.service.ts
import { Injectable } from '@angular/core';

interface PerformanceMetric {
  name: string;
  value: number;
  rating: 'good' | 'needs-improvement' | 'poor';
}

@Injectable({ providedIn: 'root' })
export class PerformanceMonitorService {

  measureWebVitals() {
    // First Contentful Paint (FCP)
    this.observePaint('first-contentful-paint', (value) => {
      const metric: PerformanceMetric = {
        name: 'FCP',
        value,
        rating: value < 1800 ? 'good' : value < 3000 ? 'needs-improvement' : 'poor'
      };
      this.reportMetric(metric);
    });

    // Largest Contentful Paint (LCP) - via PerformanceObserver
    if ('PerformanceObserver' in window) {
      const observer = new PerformanceObserver(list => {
        const entries = list.getEntries();
        const lastEntry = entries[entries.length - 1];
        const value = lastEntry.startTime;

        this.reportMetric({
          name: 'LCP',
          value,
          rating: value < 2500 ? 'good' : value < 4000 ? 'needs-improvement' : 'poor'
        });
      });

      try {
        observer.observe({ entryTypes: ['largest-contentful-paint'] });
      } catch { }
    }
  }

  measureComponentRender(componentName: string): () => void {
    const start = performance.now();
    return () => {
      const duration = performance.now() - start;
      console.log(`[Performance] ${componentName} rendered in ${duration.toFixed(2)}ms`);
    };
  }

  private observePaint(paintName: string, callback: (value: number) => void) {
    if (!('PerformanceObserver' in window)) return;

    const observer = new PerformanceObserver(list => {
      const entries = list.getEntries();
      const entry = entries.find(e => e.name === paintName);
      if (entry) {
        callback(entry.startTime);
        observer.disconnect();
      }
    });

    try {
      observer.observe({ entryTypes: ['paint'] });
    } catch { }
  }

  private reportMetric(metric: PerformanceMetric) {
    console.log(`[Web Vitals] ${metric.name}: ${metric.value.toFixed(0)}ms (${metric.rating})`);
    // ส่งไป analytics
  }
}
```

---

## สรุป

| เทคนิค | ผลกระทบ | ความยาก |
|--------|---------|---------|
| OnPush Strategy | สูง | ต่ำ |
| TrackBy | กลาง | ต่ำ |
| Lazy Loading Routes | สูง | ต่ำ |
| Lazy Loading Images | กลาง | ต่ำ |
| Virtual Scroll | สูงมาก (สำหรับ large lists) | กลาง |
| Pure Pipes | กลาง | ต่ำ |
| Bundle Analysis | สูง | กลาง |
| Preloading Strategy | กลาง | กลาง |

การ Optimize ควรทำตาม data จาก profiler และ Web Vitals ไม่ใช่ optimise ทุกอย่างโดยไม่มีหลักฐาน (Premature Optimization)
