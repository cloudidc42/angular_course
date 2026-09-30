# Part 32 — Advanced Routing

## ทบทวน Router พื้นฐาน

```typescript
// app.routes.ts
const routes: Routes = [
  { path: '', component: HomeComponent },
  { path: 'products', component: ProductsComponent },
  { path: 'products/:id', component: ProductDetailComponent },
  { path: '**', component: NotFoundComponent },
];
```

ใน Part นี้จะเรียนฟีเจอร์ขั้นสูงที่ใช้ในแอปจริง

---

## Resolver

Resolver ใช้โหลดข้อมูลล่วงหน้าก่อนที่ Route จะ Activate ทำให้ Component ได้รับข้อมูลพร้อมทันที ไม่ต้องแสดง Loading State

### Functional Resolver (Angular 14+)

```typescript
// product.resolver.ts
import { ResolveFn, ActivatedRouteSnapshot } from '@angular/router';
import { inject } from '@angular/core';
import { Router } from '@angular/router';
import { catchError, EMPTY } from 'rxjs';
import { ProductService } from '../services/product.service';
import { Product } from '../models/product.model';

export const productResolver: ResolveFn<Product> = (
  route: ActivatedRouteSnapshot
) => {
  const productService = inject(ProductService);
  const router = inject(Router);
  const id = Number(route.paramMap.get('id'));

  return productService.getById(id).pipe(
    catchError(() => {
      router.navigate(['/not-found']);
      return EMPTY;
    })
  );
};
```

```typescript
// app.routes.ts
{
  path: 'products/:id',
  component: ProductDetailComponent,
  resolve: {
    product: productResolver,  // ข้อมูลจะอยู่ใน route.data['product']
  },
}
```

```typescript
// product-detail.component.ts
import { Component, OnInit } from '@angular/core';
import { ActivatedRoute } from '@angular/router';

@Component({ /* ... */ })
export class ProductDetailComponent implements OnInit {
  product: Product;

  constructor(private route: ActivatedRoute) {}

  ngOnInit(): void {
    // ข้อมูลพร้อมใช้งานเลย ไม่ต้องรอ
    this.product = this.route.snapshot.data['product'];

    // หรือ Observable สำหรับ Re-use Component
    this.route.data.subscribe((data) => {
      this.product = data['product'];
    });
  }
}
```

### Class-based Resolver (เก่ากว่า)

```typescript
@Injectable({ providedIn: 'root' })
export class ProductsResolver implements Resolve<Product[]> {
  constructor(
    private productService: ProductService,
    private router: Router
  ) {}

  resolve(route: ActivatedRouteSnapshot): Observable<Product[]> {
    return this.productService.getAll().pipe(
      catchError(() => {
        this.router.navigate(['/error']);
        return EMPTY;
      })
    );
  }
}
```

---

## Preloading Strategies

Preloading โหลด Lazy Modules ล่วงหน้าในพื้นหลัง

### PreloadAllModules

```typescript
// app.config.ts
import { provideRouter, withPreloading, PreloadAllModules } from '@angular/router';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes, withPreloading(PreloadAllModules)),
  ],
};
```

### Custom Preloading Strategy

```typescript
// selective-preload.strategy.ts
import { Injectable } from '@angular/core';
import { PreloadingStrategy, Route } from '@angular/router';
import { Observable, of } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class SelectivePreloadStrategy implements PreloadingStrategy {
  preload(route: Route, load: () => Observable<any>): Observable<any> {
    // โหลดเฉพาะ Route ที่มี data.preload = true
    return route.data?.['preload'] ? load() : of(null);
  }
}
```

```typescript
// app.routes.ts — ระบุว่า Route ไหนต้อง Preload
const routes: Routes = [
  {
    path: 'products',
    loadComponent: () => import('./pages/products.component').then(m => m.ProductsComponent),
    data: { preload: true },  // จะถูก Preload
  },
  {
    path: 'admin',
    loadChildren: () => import('./admin/admin.routes').then(m => m.adminRoutes),
    data: { preload: false },  // ไม่ Preload (โหลดตอน Navigate เท่านั้น)
  },
];
```

---

## Router Events

Router Events ให้ติดตาม Navigation Lifecycle

```typescript
// app.component.ts
import { Component, OnInit } from '@angular/core';
import { Router, NavigationStart, NavigationEnd, NavigationError, NavigationCancel } from '@angular/router';
import { filter } from 'rxjs/operators';

@Component({
  selector: 'app-root',
  standalone: true,
  template: `
    <div class="loading-bar" [class.visible]="isLoading"></div>
    <router-outlet />
  `
})
export class AppComponent implements OnInit {
  isLoading = false;

  constructor(private router: Router) {}

  ngOnInit(): void {
    // ติดตามทุก Navigation Event
    this.router.events.subscribe((event) => {
      if (event instanceof NavigationStart) {
        this.isLoading = true;
        console.log('Navigation Started:', event.url);
      }

      if (event instanceof NavigationEnd) {
        this.isLoading = false;
        console.log('Navigation Ended:', event.url);
        // ส่ง Page View ไปยัง Analytics
        this.trackPageView(event.urlAfterRedirects);
      }

      if (event instanceof NavigationError) {
        this.isLoading = false;
        console.error('Navigation Error:', event.error);
      }

      if (event instanceof NavigationCancel) {
        this.isLoading = false;
        console.warn('Navigation Cancelled:', event.reason);
      }
    });
  }

  private trackPageView(url: string): void {
    // Google Analytics หรือ Analytics ของตัวเอง
    console.log('Page view:', url);
  }
}
```

### Progress Bar Service

```typescript
// router-progress.service.ts
import { Injectable } from '@angular/core';
import { Router, NavigationStart, NavigationEnd, NavigationError } from '@angular/router';
import { BehaviorSubject } from 'rxjs';
import { filter } from 'rxjs/operators';

@Injectable({ providedIn: 'root' })
export class RouterProgressService {
  private loading$ = new BehaviorSubject<boolean>(false);
  isLoading$ = this.loading$.asObservable();

  constructor(private router: Router) {
    this.router.events.pipe(
      filter((e) =>
        e instanceof NavigationStart ||
        e instanceof NavigationEnd ||
        e instanceof NavigationError
      )
    ).subscribe((event) => {
      if (event instanceof NavigationStart) {
        this.loading$.next(true);
      } else {
        this.loading$.next(false);
      }
    });
  }
}
```

---

## Scroll Restoration

```typescript
// app.config.ts
import { provideRouter, withInMemoryScrolling } from '@angular/router';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(
      routes,
      withInMemoryScrolling({
        // คืนตำแหน่ง Scroll เมื่อ Navigate กลับ
        scrollPositionRestoration: 'enabled',
        // Scroll ไปที่ Anchor ที่ระบุใน URL
        anchorScrolling: 'enabled',
      })
    ),
  ],
};
```

### Manual Scroll Control

```typescript
// scroll.service.ts
import { Injectable } from '@angular/core';
import { ViewportScroller } from '@angular/common';
import { Router, Scroll } from '@angular/router';
import { filter } from 'rxjs/operators';

@Injectable({ providedIn: 'root' })
export class ScrollService {
  constructor(
    private router: Router,
    private viewportScroller: ViewportScroller
  ) {
    this.router.events.pipe(
      filter((event): event is Scroll => event instanceof Scroll)
    ).subscribe((event) => {
      if (event.position) {
        // Backward Navigation — คืนตำแหน่งเดิม
        this.viewportScroller.scrollToPosition(event.position);
      } else if (event.anchor) {
        // Anchor Navigation
        this.viewportScroller.scrollToAnchor(event.anchor);
      } else {
        // Forward Navigation — Scroll ไปบนสุด
        this.viewportScroller.scrollToPosition([0, 0]);
      }
    });
  }
}
```

---

## Workshop: Advanced Navigation

### Route Configuration สมบูรณ์

```typescript
// app.routes.ts
import { Routes } from '@angular/router';
import { authGuard } from './guards/auth.guard';
import { roleGuard } from './guards/role.guard';
import { productResolver } from './resolvers/product.resolver';
import { unsavedChangesGuard } from './guards/unsaved-changes.guard';

export const routes: Routes = [
  // หน้าแรก
  {
    path: '',
    redirectTo: '/home',
    pathMatch: 'full',
  },

  // Home
  {
    path: 'home',
    loadComponent: () =>
      import('./pages/home/home.component').then((m) => m.HomeComponent),
    title: 'หน้าหลัก',
  },

  // Products
  {
    path: 'products',
    children: [
      {
        path: '',
        loadComponent: () =>
          import('./pages/products/products.component').then((m) => m.ProductsComponent),
        title: 'สินค้าทั้งหมด',
        data: { preload: true },
        resolve: {
          products: () => inject(ProductService).getAll(),
        },
      },
      {
        path: ':id',
        loadComponent: () =>
          import('./pages/product-detail/product-detail.component').then(
            (m) => m.ProductDetailComponent
          ),
        title: (route) => `สินค้า #${route.params['id']}`,
        resolve: { product: productResolver },
      },
      {
        path: ':id/edit',
        loadComponent: () =>
          import('./pages/product-edit/product-edit.component').then(
            (m) => m.ProductEditComponent
          ),
        canActivate: [authGuard],
        canDeactivate: [unsavedChangesGuard],  // ป้องกันออกเมื่อมี Unsaved Changes
        resolve: { product: productResolver },
      },
    ],
  },

  // Admin (Protected)
  {
    path: 'admin',
    canActivate: [authGuard, () => roleGuard('admin')],
    loadChildren: () =>
      import('./pages/admin/admin.routes').then((m) => m.adminRoutes),
    data: { preload: false },
  },

  // Auth
  {
    path: 'login',
    loadComponent: () =>
      import('./pages/login/login.component').then((m) => m.LoginComponent),
    canActivate: [() => {
      const auth = inject(AuthService);
      return auth.isLoggedIn() ? inject(Router).parseUrl('/home') : true;
    }],
  },

  // 404
  {
    path: '**',
    loadComponent: () =>
      import('./pages/not-found/not-found.component').then(
        (m) => m.NotFoundComponent
      ),
    title: 'ไม่พบหน้าที่ต้องการ',
  },
];
```

### Guards

```typescript
// auth.guard.ts
import { inject } from '@angular/core';
import { CanActivateFn, Router } from '@angular/router';
import { AuthService } from '../services/auth.service';

export const authGuard: CanActivateFn = (route, state) => {
  const auth = inject(AuthService);
  const router = inject(Router);

  if (auth.isLoggedIn()) {
    return true;
  }

  // เก็บ URL ที่ต้องการเข้า แล้วนำทางไปหน้า Login
  return router.createUrlTree(['/login'], {
    queryParams: { returnUrl: state.url },
  });
};
```

```typescript
// role.guard.ts
import { inject } from '@angular/core';
import { CanActivateFn, Router } from '@angular/router';
import { AuthService } from '../services/auth.service';

export const roleGuard = (requiredRole: string): CanActivateFn => {
  return (route, state) => {
    const auth = inject(AuthService);
    const router = inject(Router);

    if (auth.hasRole(requiredRole)) {
      return true;
    }

    return router.createUrlTree(['/forbidden']);
  };
};
```

```typescript
// unsaved-changes.guard.ts
import { CanDeactivateFn } from '@angular/router';

// Interface สำหรับ Component ที่ต้องการป้องกัน
export interface CanDeactivateComponent {
  hasUnsavedChanges(): boolean;
}

export const unsavedChangesGuard: CanDeactivateFn<CanDeactivateComponent> = (
  component
) => {
  if (component.hasUnsavedChanges()) {
    return confirm('คุณมีการเปลี่ยนแปลงที่ยังไม่ได้บันทึก ต้องการออกจากหน้านี้หรือไม่?');
  }
  return true;
};
```

### Component ที่ใช้ Guard

```typescript
// product-edit.component.ts
import { Component } from '@angular/core';
import { CanDeactivateComponent } from '../../guards/unsaved-changes.guard';

@Component({ /* ... */ })
export class ProductEditComponent implements CanDeactivateComponent {
  form = { /* ... */ };
  isSaved = false;

  hasUnsavedChanges(): boolean {
    return this.form.dirty && !this.isSaved;
  }

  save(): void {
    // บันทึกข้อมูล
    this.isSaved = true;
  }
}
```

### Route Title Strategy

```typescript
// custom-title.strategy.ts
import { Injectable } from '@angular/core';
import { Title } from '@angular/platform-browser';
import { RouterStateSnapshot, TitleStrategy } from '@angular/router';

@Injectable({ providedIn: 'root' })
export class CustomTitleStrategy extends TitleStrategy {
  constructor(private readonly title: Title) {
    super();
  }

  override updateTitle(routerState: RouterStateSnapshot): void {
    const title = this.buildTitle(routerState);
    if (title) {
      this.title.setTitle(`${title} | ร้านค้าออนไลน์`);
    } else {
      this.title.setTitle('ร้านค้าออนไลน์');
    }
  }
}

// ลงทะเบียนใน app.config.ts
export const appConfig: ApplicationConfig = {
  providers: [
    { provide: TitleStrategy, useClass: CustomTitleStrategy },
    provideRouter(routes),
  ],
};
```

### Breadcrumb Service

```typescript
// breadcrumb.service.ts
import { Injectable } from '@angular/core';
import { ActivatedRouteSnapshot, NavigationEnd, Router } from '@angular/router';
import { BehaviorSubject } from 'rxjs';
import { filter } from 'rxjs/operators';

export interface Breadcrumb {
  label: string;
  url: string;
}

@Injectable({ providedIn: 'root' })
export class BreadcrumbService {
  private breadcrumbs$ = new BehaviorSubject<Breadcrumb[]>([]);
  breadcrumbs = this.breadcrumbs$.asObservable();

  constructor(private router: Router) {
    this.router.events.pipe(
      filter((event) => event instanceof NavigationEnd)
    ).subscribe(() => {
      const root = this.router.routerState.snapshot.root;
      this.breadcrumbs$.next(this.buildBreadcrumbs(root));
    });
  }

  private buildBreadcrumbs(
    route: ActivatedRouteSnapshot,
    url = '',
    breadcrumbs: Breadcrumb[] = []
  ): Breadcrumb[] {
    const path = route.url.map((s) => s.path).join('/');
    const nextUrl = path ? `${url}/${path}` : url;

    const breadcrumb = route.data?.['breadcrumb'];
    if (breadcrumb) {
      breadcrumbs.push({ label: breadcrumb, url: nextUrl });
    }

    if (route.firstChild) {
      return this.buildBreadcrumbs(route.firstChild, nextUrl, breadcrumbs);
    }

    return breadcrumbs;
  }
}
```

---

## Route Animations

```typescript
// route-animation.ts
import { trigger, transition, style, animate, query } from '@angular/animations';

export const routeAnimations = trigger('routeAnimations', [
  transition('* <=> *', [
    query(':enter, :leave', [
      style({ position: 'fixed', width: '100%' })
    ], { optional: true }),
    query(':enter', [
      style({ opacity: 0, transform: 'translateX(20px)' })
    ], { optional: true }),
    query(':leave', [
      animate('200ms ease-out', style({ opacity: 0, transform: 'translateX(-20px)' }))
    ], { optional: true }),
    query(':enter', [
      animate('200ms ease-in', style({ opacity: 1, transform: 'translateX(0)' }))
    ], { optional: true }),
  ])
]);
```

```typescript
// app.component.ts
@Component({
  selector: 'app-root',
  standalone: true,
  imports: [RouterOutlet],
  template: `
    <main [@routeAnimations]="getAnimationData()">
      <router-outlet #outlet="outlet" />
    </main>
  `,
  animations: [routeAnimations],
})
export class AppComponent {
  getAnimationData(): string {
    return Math.random().toString();
  }
}
```

---

## สรุป

| Feature | การใช้งาน |
|---------|-----------|
| **Resolver** | โหลดข้อมูลก่อน Route Activate |
| **PreloadAllModules** | Preload ทุก Lazy Module |
| **SelectivePreload** | Preload เฉพาะ Route ที่กำหนด |
| **Router Events** | ติดตาม Navigation Lifecycle |
| **Scroll Restoration** | คืนตำแหน่ง Scroll อัตโนมัติ |
| **CanActivate** | ป้องกันการเข้าถึง Route |
| **CanDeactivate** | ป้องกันการออกจาก Route |
| **TitleStrategy** | กำหนด Page Title อัตโนมัติ |

ใน Part ถัดไปจะเรียน Dynamic Components สำหรับการสร้าง Components แบบ Programmatic
