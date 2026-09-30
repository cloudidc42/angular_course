# Part 15: Lazy Loading

## บทนำ

Lazy Loading คือเทคนิคการโหลด JavaScript Bundle เฉพาะเมื่อผู้ใช้ต้องการ แทนที่จะโหลดทุกอย่างพร้อมกันตั้งแต่เริ่มต้น ทำให้แอปพลิเคชันเริ่มต้นได้เร็วขึ้นอย่างมาก

---

## 1. Lazy Loading คืออะไรและทำไมต้องใช้

### ปัญหาของ Eager Loading

```
แอปโดยไม่ใช้ Lazy Loading:
Initial Bundle: 2.5 MB
├── app.js          (50 KB)
├── home.js         (100 KB)
├── products.js     (300 KB)  ← โหลดทั้งที่ผู้ใช้อาจไม่เข้าหน้านี้
├── admin.js        (800 KB)  ← โหลดทั้งที่ผู้ใช้ส่วนใหญ่ไม่ใช่ Admin
├── reports.js      (600 KB)  ← โหลดทั้งที่ผู้ใช้อาจไม่ดู Report
└── vendor.js       (650 KB)

เวลาโหลดครั้งแรก: ~5-8 วินาที บน 3G
```

### ด้วย Lazy Loading

```
Initial Bundle: 750 KB
├── app.js          (50 KB)
├── home.js         (100 KB)  ← โหลดเมื่อผู้ใช้เข้า /home
└── vendor.js       (600 KB)

Lazy Chunks (โหลดเมื่อต้องการ):
├── products.chunk.js  (300 KB) ← โหลดเมื่อเข้า /products
├── admin.chunk.js     (800 KB) ← โหลดเมื่อเข้า /admin
└── reports.chunk.js   (600 KB) ← โหลดเมื่อเข้า /reports

เวลาโหลดครั้งแรก: ~1.5 วินาที บน 3G (ลดลง 70%)
```

### ประโยชน์

- **เร็วขึ้น**: Initial bundle เล็กลง โหลดได้เร็วขึ้น
- **ประหยัด bandwidth**: ผู้ใช้โหลดเฉพาะสิ่งที่ต้องการ
- **Core Web Vitals ดีขึ้น**: LCP (Largest Contentful Paint) เร็วขึ้น
- **ประสบการณ์ผู้ใช้ดีขึ้น**: แอปรู้สึกเร็วกว่า

---

## 2. loadChildren ใน Routes

### การตั้งค่า Lazy Loading แบบ NgModule

```typescript
// app-routing.module.ts
import { NgModule } from '@angular/core';
import { RouterModule, Routes } from '@angular/router';

const routes: Routes = [
  {
    path: '',
    redirectTo: 'home',
    pathMatch: 'full',
  },
  {
    path: 'home',
    // Eager loading - โหลดทันที
    loadChildren: () =>
      import('./features/home/home.module').then(m => m.HomeModule),
  },
  {
    path: 'products',
    // Lazy loading - โหลดเมื่อผู้ใช้เข้า /products
    loadChildren: () =>
      import('./features/products/products.module').then(m => m.ProductsModule),
  },
  {
    path: 'admin',
    // Lazy loading พร้อม Guard
    loadChildren: () =>
      import('./features/admin/admin.module').then(m => m.AdminModule),
    canActivate: [AuthGuard],
    canMatch: [AdminRoleGuard],
  },
  {
    path: 'reports',
    loadChildren: () =>
      import('./features/reports/reports.module').then(m => m.ReportsModule),
  },
];

@NgModule({
  imports: [RouterModule.forRoot(routes)],
  exports: [RouterModule],
})
export class AppRoutingModule {}
```

### Feature Module ต้องใช้ `forChild`

```typescript
// features/products/products-routing.module.ts
import { NgModule } from '@angular/core';
import { RouterModule, Routes } from '@angular/router';
import { ProductListComponent } from './product-list/product-list.component';
import { ProductDetailComponent } from './product-detail/product-detail.component';
import { ProductFormComponent } from './product-form/product-form.component';

const routes: Routes = [
  {
    path: '',         // เทียบกับ '/products'
    component: ProductListComponent,
  },
  {
    path: 'new',      // เทียบกับ '/products/new'
    component: ProductFormComponent,
  },
  {
    path: ':id',      // เทียบกับ '/products/:id'
    component: ProductDetailComponent,
  },
  {
    path: ':id/edit', // เทียบกับ '/products/:id/edit'
    component: ProductFormComponent,
  },
];

@NgModule({
  imports: [RouterModule.forChild(routes)], // ใช้ forChild ไม่ใช่ forRoot
  exports: [RouterModule],
})
export class ProductsRoutingModule {}
```

### Products Module

```typescript
// features/products/products.module.ts
import { NgModule } from '@angular/core';
import { CommonModule } from '@angular/common';
import { ReactiveFormsModule } from '@angular/forms';
import { ProductsRoutingModule } from './products-routing.module';
import { SharedModule } from '../../shared/shared.module';
import { ProductListComponent } from './product-list/product-list.component';
import { ProductDetailComponent } from './product-detail/product-detail.component';
import { ProductFormComponent } from './product-form/product-form.component';

@NgModule({
  declarations: [
    ProductListComponent,
    ProductDetailComponent,
    ProductFormComponent,
  ],
  imports: [
    CommonModule,
    ReactiveFormsModule,
    ProductsRoutingModule,
    SharedModule,
  ],
})
export class ProductsModule {}
```

---

## 3. loadComponent สำหรับ Standalone

ใน Angular 14+ สามารถ Lazy Load ได้ระดับ Component โดยตรง

### Standalone Routes

```typescript
// app.routes.ts
import { Routes } from '@angular/router';

export const routes: Routes = [
  {
    path: '',
    redirectTo: 'home',
    pathMatch: 'full',
  },
  {
    path: 'home',
    loadComponent: () =>
      import('./features/home/home.component').then(c => c.HomeComponent),
  },
  {
    path: 'products',
    // Lazy load กลุ่ม routes
    loadChildren: () =>
      import('./features/products/products.routes').then(r => r.productRoutes),
  },
  {
    path: 'admin',
    loadChildren: () =>
      import('./features/admin/admin.routes').then(r => r.adminRoutes),
    canActivate: [authGuard],
  },
];
```

### Standalone Product Routes

```typescript
// features/products/products.routes.ts
import { Routes } from '@angular/router';

export const productRoutes: Routes = [
  {
    path: '',
    loadComponent: () =>
      import('./product-list/product-list.component')
        .then(c => c.ProductListComponent),
  },
  {
    path: 'new',
    loadComponent: () =>
      import('./product-form/product-form.component')
        .then(c => c.ProductFormComponent),
  },
  {
    path: ':id',
    loadComponent: () =>
      import('./product-detail/product-detail.component')
        .then(c => c.ProductDetailComponent),
  },
  {
    path: ':id/edit',
    loadComponent: () =>
      import('./product-form/product-form.component')
        .then(c => c.ProductFormComponent),
  },
];
```

### Standalone Component ที่ Lazy Load

```typescript
// features/products/product-list/product-list.component.ts
import { Component, OnInit, signal } from '@angular/core';
import { CommonModule } from '@angular/common';
import { RouterModule } from '@angular/router';
import { ProductService } from '../services/product.service';
import { Product } from '../models/product.model';
import { ProductCardComponent } from '../product-card/product-card.component';
import { SearchBarComponent } from '../../../shared/search-bar/search-bar.component';
import { PaginationComponent } from '../../../shared/pagination/pagination.component';

@Component({
  selector: 'app-product-list',
  standalone: true,
  imports: [
    CommonModule,
    RouterModule,
    ProductCardComponent,
    SearchBarComponent,
    PaginationComponent,
  ],
  template: `
    <div class="product-list-page">
      <h1>สินค้าทั้งหมด</h1>

      <app-search-bar
        (search)="onSearch($event)"
        placeholder="ค้นหาสินค้า..."
      ></app-search-bar>

      <div class="products-grid" *ngIf="!loading()">
        <app-product-card
          *ngFor="let product of products()"
          [product]="product"
        ></app-product-card>
      </div>

      <div class="loading" *ngIf="loading()">กำลังโหลด...</div>

      <app-pagination
        [currentPage]="currentPage()"
        [totalItems]="totalItems()"
        [pageSize]="pageSize"
        (pageChange)="onPageChange($event)"
      ></app-pagination>
    </div>
  `,
})
export class ProductListComponent implements OnInit {
  products = signal<Product[]>([]);
  loading = signal(false);
  currentPage = signal(1);
  totalItems = signal(0);
  pageSize = 12;

  constructor(private productService: ProductService) {}

  ngOnInit(): void {
    this.loadProducts();
  }

  loadProducts(): void {
    this.loading.set(true);
    this.productService
      .getProducts({ page: this.currentPage(), limit: this.pageSize })
      .subscribe({
        next: (response) => {
          this.products.set(response.data);
          this.totalItems.set(response.total);
          this.loading.set(false);
        },
        error: () => this.loading.set(false),
      });
  }

  onSearch(query: string): void {
    this.currentPage.set(1);
    this.loadProducts();
  }

  onPageChange(page: number): void {
    this.currentPage.set(page);
    this.loadProducts();
  }
}
```

---

## 4. Preloading Strategies

Preloading คือการโหลด Lazy Modules ล่วงหน้าเบื้องหลัง หลังจาก Initial Load เสร็จ

### PreloadAllModules

```typescript
// app-routing.module.ts
import { RouterModule, PreloadAllModules } from '@angular/router';

@NgModule({
  imports: [
    RouterModule.forRoot(routes, {
      preloadingStrategy: PreloadAllModules, // โหลดทุก Module ล่วงหน้า
    }),
  ],
  exports: [RouterModule],
})
export class AppRoutingModule {}
```

### Custom Preloading Strategy

```typescript
// core/strategies/selective-preloading.strategy.ts
import { Injectable } from '@angular/core';
import { PreloadingStrategy, Route } from '@angular/router';
import { Observable, of, timer } from 'rxjs';
import { mergeMap } from 'rxjs/operators';

@Injectable({ providedIn: 'root' })
export class SelectivePreloadingStrategy implements PreloadingStrategy {
  preload(route: Route, load: () => Observable<any>): Observable<any> {
    // โหลดล่วงหน้าเฉพาะ routes ที่มี data.preload: true
    if (route.data?.['preload'] === true) {
      // หน่วงเวลา 2 วินาทีก่อน preload (ให้ initial load เสร็จก่อน)
      return timer(2000).pipe(mergeMap(() => load()));
    }
    return of(null);
  }
}
```

```typescript
// ใช้ใน Routes
const routes: Routes = [
  {
    path: 'products',
    loadChildren: () =>
      import('./features/products/products.module').then(m => m.ProductsModule),
    data: { preload: true }, // จะ preload หน้านี้
  },
  {
    path: 'admin',
    loadChildren: () =>
      import('./features/admin/admin.module').then(m => m.AdminModule),
    // ไม่มี preload: true จะไม่ preload
  },
];

@NgModule({
  imports: [
    RouterModule.forRoot(routes, {
      preloadingStrategy: SelectivePreloadingStrategy,
    }),
  ],
})
export class AppRoutingModule {}
```

### Network-Aware Preloading

```typescript
// core/strategies/network-aware-preloading.strategy.ts
import { Injectable } from '@angular/core';
import { PreloadingStrategy, Route } from '@angular/router';
import { Observable, of } from 'rxjs';

declare const navigator: Navigator & {
  connection?: {
    effectiveType: string;
    saveData: boolean;
  };
};

@Injectable({ providedIn: 'root' })
export class NetworkAwarePreloadingStrategy implements PreloadingStrategy {
  preload(route: Route, load: () => Observable<any>): Observable<any> {
    // ตรวจสอบ Network Information API
    const connection = navigator.connection;

    if (connection) {
      // ไม่ preload ถ้าใช้ Data Saver mode
      if (connection.saveData) {
        return of(null);
      }

      // Preload เฉพาะ 4G/WiFi
      const slowConnections = ['slow-2g', '2g', '3g'];
      if (slowConnections.includes(connection.effectiveType)) {
        return of(null);
      }
    }

    // Preload ถ้าไม่มีข้อมูล Network หรือ Network เร็ว
    return route.data?.['preload'] ? load() : of(null);
  }
}
```

---

## 5. Route-level Code Splitting

### แยก Bundle ตาม Route

```typescript
// ตัวอย่าง Bundle Analysis ก่อน/หลัง
// ก่อน: main.bundle.js (3MB)
// หลัง:
//   main.bundle.js (500KB)
//   products-module.js (300KB)
//   admin-module.js (800KB)
//   reports-module.js (400KB)
```

### Nested Lazy Loading

```typescript
// features/admin/admin-routing.module.ts
const adminRoutes: Routes = [
  {
    path: '',
    component: AdminLayoutComponent,
    children: [
      {
        path: 'dashboard',
        component: AdminDashboardComponent,
      },
      {
        path: 'products',
        // Nested Lazy Loading ภายใน Admin
        loadChildren: () =>
          import('./products/admin-products.module')
            .then(m => m.AdminProductsModule),
      },
      {
        path: 'users',
        loadChildren: () =>
          import('./users/admin-users.module')
            .then(m => m.AdminUsersModule),
      },
      {
        path: 'reports',
        loadChildren: () =>
          import('./reports/admin-reports.module')
            .then(m => m.AdminReportsModule),
      },
    ],
  },
];
```

### Dynamic Import ตามเงื่อนไข

```typescript
// core/services/feature-flags.service.ts
import { Injectable } from '@angular/core';
import { Router } from '@angular/router';

@Injectable({ providedIn: 'root' })
export class FeatureFlagsService {
  private flags = {
    newCheckout: false,
    betaFeatures: false,
  };

  isEnabled(flag: string): boolean {
    return this.flags[flag as keyof typeof this.flags] ?? false;
  }
}
```

```typescript
// ใช้ canMatch เพื่อ load module ต่างกันตาม feature flag
const routes: Routes = [
  {
    path: 'checkout',
    loadChildren: () =>
      import('./features/checkout-v2/checkout-v2.module')
        .then(m => m.CheckoutV2Module),
    canMatch: [() => inject(FeatureFlagsService).isEnabled('newCheckout')],
  },
  {
    path: 'checkout',
    loadChildren: () =>
      import('./features/checkout/checkout.module')
        .then(m => m.CheckoutModule),
  },
];
```

### Preconnect และ Prefetch Hints

```html
<!-- index.html - บอก Browser ว่าจะ fetch อะไรล่วงหน้า -->
<head>
  <!-- Preconnect to API server -->
  <link rel="preconnect" href="https://api.yourapp.com">
  <link rel="preconnect" href="https://cdn.yourapp.com">

  <!-- DNS Prefetch -->
  <link rel="dns-prefetch" href="//fonts.googleapis.com">
</head>
```

---

## 6. Workshop: Admin Panel ที่ Lazy Load

### โครงสร้าง Admin Panel

```
src/app/features/admin/
├── admin.module.ts
├── admin-routing.module.ts
├── admin-layout/
│   └── admin-layout.component.ts
├── dashboard/
│   └── admin-dashboard.component.ts
├── products/
│   ├── admin-products.module.ts
│   ├── admin-products-routing.module.ts
│   ├── product-management/
│   └── product-categories/
├── users/
│   ├── admin-users.module.ts
│   └── user-management/
└── reports/
    ├── admin-reports.module.ts
    └── sales-report/
```

### Admin Module

```typescript
// features/admin/admin.module.ts
import { NgModule } from '@angular/core';
import { CommonModule } from '@angular/common';
import { AdminRoutingModule } from './admin-routing.module';
import { SharedModule } from '../../shared/shared.module';
import { AdminLayoutComponent } from './admin-layout/admin-layout.component';
import { AdminDashboardComponent } from './dashboard/admin-dashboard.component';
import { AdminSidebarComponent } from './admin-layout/admin-sidebar/admin-sidebar.component';
import { AdminHeaderComponent } from './admin-layout/admin-header/admin-header.component';

@NgModule({
  declarations: [
    AdminLayoutComponent,
    AdminDashboardComponent,
    AdminSidebarComponent,
    AdminHeaderComponent,
  ],
  imports: [
    CommonModule,
    AdminRoutingModule,
    SharedModule,
  ],
})
export class AdminModule {}
```

### Admin Routing

```typescript
// features/admin/admin-routing.module.ts
import { NgModule } from '@angular/core';
import { RouterModule, Routes } from '@angular/router';
import { AdminLayoutComponent } from './admin-layout/admin-layout.component';
import { AdminDashboardComponent } from './dashboard/admin-dashboard.component';
import { RoleGuard } from '../../core/guards/role.guard';

const routes: Routes = [
  {
    path: '',
    component: AdminLayoutComponent,
    canActivate: [RoleGuard],
    data: { role: 'admin' },
    children: [
      {
        path: '',
        redirectTo: 'dashboard',
        pathMatch: 'full',
      },
      {
        path: 'dashboard',
        component: AdminDashboardComponent,
      },
      {
        path: 'products',
        loadChildren: () =>
          import('./products/admin-products.module')
            .then(m => m.AdminProductsModule),
      },
      {
        path: 'users',
        loadChildren: () =>
          import('./users/admin-users.module')
            .then(m => m.AdminUsersModule),
      },
      {
        path: 'orders',
        loadChildren: () =>
          import('./orders/admin-orders.module')
            .then(m => m.AdminOrdersModule),
      },
      {
        path: 'reports',
        loadChildren: () =>
          import('./reports/admin-reports.module')
            .then(m => m.AdminReportsModule),
      },
      {
        path: 'settings',
        loadChildren: () =>
          import('./settings/admin-settings.module')
            .then(m => m.AdminSettingsModule),
      },
    ],
  },
];

@NgModule({
  imports: [RouterModule.forChild(routes)],
  exports: [RouterModule],
})
export class AdminRoutingModule {}
```

### Admin Layout Component

```typescript
// features/admin/admin-layout/admin-layout.component.ts
import { Component, signal } from '@angular/core';
import { CommonModule } from '@angular/common';
import { RouterModule } from '@angular/router';
import { AuthService } from '../../../core/services/auth.service';

interface NavItem {
  label: string;
  icon: string;
  path: string;
  badge?: number;
  children?: NavItem[];
}

@Component({
  selector: 'app-admin-layout',
  template: `
    <div class="admin-layout" [class.sidebar-collapsed]="sidebarCollapsed()">
      <!-- Sidebar -->
      <aside class="sidebar">
        <div class="brand">
          <span class="logo">🏪</span>
          <span class="brand-name" *ngIf="!sidebarCollapsed()">Admin Panel</span>
        </div>

        <nav class="nav">
          <a
            *ngFor="let item of navItems"
            [routerLink]="item.path"
            routerLinkActive="active"
            class="nav-item"
          >
            <span class="icon">{{ item.icon }}</span>
            <span class="label" *ngIf="!sidebarCollapsed()">{{ item.label }}</span>
            <span class="badge" *ngIf="item.badge">{{ item.badge }}</span>
          </a>
        </nav>
      </aside>

      <!-- Main Content -->
      <div class="main">
        <header class="top-bar">
          <button (click)="toggleSidebar()" class="toggle-btn">☰</button>
          <div class="user-info">
            <span>{{ (auth.currentUser$ | async)?.name }}</span>
            <button (click)="auth.logout()" class="logout-btn">ออกจากระบบ</button>
          </div>
        </header>

        <main class="content">
          <router-outlet></router-outlet>
        </main>
      </div>
    </div>
  `,
  styleUrls: ['./admin-layout.component.scss'],
})
export class AdminLayoutComponent {
  sidebarCollapsed = signal(false);

  navItems: NavItem[] = [
    { label: 'Dashboard', icon: '📊', path: '/admin/dashboard' },
    { label: 'สินค้า', icon: '📦', path: '/admin/products' },
    { label: 'คำสั่งซื้อ', icon: '🛒', path: '/admin/orders', badge: 5 },
    { label: 'ผู้ใช้งาน', icon: '👥', path: '/admin/users' },
    { label: 'รายงาน', icon: '📈', path: '/admin/reports' },
    { label: 'ตั้งค่า', icon: '⚙️', path: '/admin/settings' },
  ];

  constructor(public auth: AuthService) {}

  toggleSidebar(): void {
    this.sidebarCollapsed.update(v => !v);
  }
}
```

### Admin Dashboard (ข้อมูล Real-time)

```typescript
// features/admin/dashboard/admin-dashboard.component.ts
import { Component, OnInit, OnDestroy } from '@angular/core';
import { CommonModule } from '@angular/common';
import { Subject, interval, switchMap, takeUntil } from 'rxjs';
import { DashboardService } from '../services/dashboard.service';

interface DashboardStats {
  totalOrders: number;
  totalRevenue: number;
  newUsers: number;
  pendingOrders: number;
}

@Component({
  selector: 'app-admin-dashboard',
  template: `
    <div class="dashboard">
      <h1>Dashboard</h1>

      <!-- Stats Cards -->
      <div class="stats-grid">
        <div class="stat-card">
          <div class="stat-icon">🛒</div>
          <div class="stat-value">{{ stats?.totalOrders | number }}</div>
          <div class="stat-label">คำสั่งซื้อทั้งหมด</div>
        </div>
        <div class="stat-card revenue">
          <div class="stat-icon">💰</div>
          <div class="stat-value">{{ stats?.totalRevenue | currency:'THB':'symbol':'1.0-0' }}</div>
          <div class="stat-label">รายได้รวม</div>
        </div>
        <div class="stat-card">
          <div class="stat-icon">👥</div>
          <div class="stat-value">{{ stats?.newUsers }}</div>
          <div class="stat-label">ผู้ใช้ใหม่วันนี้</div>
        </div>
        <div class="stat-card warning">
          <div class="stat-icon">⏳</div>
          <div class="stat-value">{{ stats?.pendingOrders }}</div>
          <div class="stat-label">รอดำเนินการ</div>
        </div>
      </div>
    </div>
  `,
})
export class AdminDashboardComponent implements OnInit, OnDestroy {
  stats: DashboardStats | null = null;
  private destroy$ = new Subject<void>();

  constructor(private dashboardService: DashboardService) {}

  ngOnInit(): void {
    // อัปเดต Stats ทุก 30 วินาที
    interval(30000)
      .pipe(
        switchMap(() => this.dashboardService.getStats()),
        takeUntil(this.destroy$)
      )
      .subscribe(stats => {
        this.stats = stats;
      });

    // โหลดครั้งแรก
    this.dashboardService.getStats().subscribe(stats => {
      this.stats = stats;
    });
  }

  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

### Loading Indicator สำหรับ Lazy Loading

```typescript
// app.component.ts
import { Component, OnInit } from '@angular/core';
import { Router, NavigationStart, NavigationEnd, NavigationCancel, NavigationError } from '@angular/router';
import { signal } from '@angular/core';

@Component({
  selector: 'app-root',
  template: `
    <!-- แสดง Progress Bar ระหว่างโหลด Lazy Module -->
    <div class="route-loading-bar" *ngIf="isRouteLoading()">
      <div class="progress"></div>
    </div>

    <app-header></app-header>
    <router-outlet></router-outlet>
    <app-footer></app-footer>
  `,
})
export class AppComponent implements OnInit {
  isRouteLoading = signal(false);

  constructor(private router: Router) {}

  ngOnInit(): void {
    this.router.events.subscribe(event => {
      if (event instanceof NavigationStart) {
        this.isRouteLoading.set(true);
      } else if (
        event instanceof NavigationEnd ||
        event instanceof NavigationCancel ||
        event instanceof NavigationError
      ) {
        this.isRouteLoading.set(false);
      }
    });
  }
}
```

### angular.json — Bundle Analysis

```json
// angular.json - เพิ่ม budgets เพื่อเตือนเมื่อ bundle ใหญ่เกิน
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
      ]
    }
  }
}
```

---

## สรุป

| เทคนิค | ใช้เมื่อ |
|--------|---------|
| **loadChildren** | NgModule-based Feature Modules |
| **loadComponent** | Standalone Components ใน Angular 14+ |
| **PreloadAllModules** | ต้องการ preload ทุก Module หลัง initial load |
| **SelectivePreloading** | ต้องการ preload เฉพาะ Module ที่สำคัญ |
| **NetworkAwarePreloading** | ต้องการ preload ตาม Network speed |

### Best Practices

1. **Lazy Load ทุก Feature Module** โดยเฉพาะ Admin และฟีเจอร์ขนาดใหญ่
2. **ใช้ Preloading Strategy** เพื่อ preload ล่วงหน้า
3. **ตั้ง Budget** ใน angular.json เพื่อควบคุมขนาด bundle
4. **แสดง Loading Indicator** ระหว่างโหลด Module ใหม่
5. **วิเคราะห์ Bundle** ด้วย `ng build --stats-json` และ webpack-bundle-analyzer
