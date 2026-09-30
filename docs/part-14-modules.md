# Part 14: Angular Modules

## บทนำ

Angular Modules หรือ NgModule เป็นกลไกสำคัญในการจัดระเบียบโค้ด Angular ให้เป็นหมวดหมู่ที่ชัดเจน แม้ว่าใน Angular 14+ จะมี Standalone Components ที่ลดความจำเป็นของ NgModule ลง แต่การเข้าใจ Modules ยังคงสำคัญสำหรับโปรเจคขนาดใหญ่และการดูแลรักษาโค้ดเดิม

---

## 1. NgModule คืออะไร

NgModule คือ decorator `@NgModule` ที่ใช้ประกาศกลุ่มของ Components, Directives, Pipes และ Services ที่เกี่ยวข้องกันให้ทำงานร่วมกันเป็นหน่วยเดียว

### โครงสร้างพื้นฐาน

```typescript
// app.module.ts
import { NgModule } from '@angular/core';
import { BrowserModule } from '@angular/platform-browser';
import { HttpClientModule } from '@angular/common/http';
import { FormsModule, ReactiveFormsModule } from '@angular/forms';

import { AppRoutingModule } from './app-routing.module';
import { AppComponent } from './app.component';
import { HomeComponent } from './home/home.component';
import { HeaderComponent } from './shared/header/header.component';

@NgModule({
  declarations: [
    // Components, Directives, Pipes ที่เป็นของ Module นี้
    AppComponent,
    HomeComponent,
    HeaderComponent,
  ],
  imports: [
    // Modules อื่นๆ ที่ Module นี้ต้องการใช้
    BrowserModule,
    HttpClientModule,
    FormsModule,
    ReactiveFormsModule,
    AppRoutingModule,
  ],
  exports: [
    // สิ่งที่ต้องการให้ Module อื่นนำไปใช้ได้
    HeaderComponent,
  ],
  providers: [
    // Services ที่ต้องการให้ Module นี้ inject ได้
    // (ส่วนใหญ่ใช้ providedIn: 'root' แทน)
  ],
  bootstrap: [
    // Component แรกที่จะ bootstrap (เฉพาะ AppModule)
    AppComponent,
  ],
})
export class AppModule {}
```

### ความหมายของแต่ละ property

| Property | ความหมาย |
|----------|----------|
| `declarations` | ประกาศ Components, Directives, Pipes ที่เป็นของ Module นี้ |
| `imports` | นำ Module อื่นมาใช้ภายใน Module นี้ |
| `exports` | เปิดเผย declarations ให้ Module อื่นนำไปใช้ได้ |
| `providers` | กำหนด Services สำหรับ Dependency Injection |
| `bootstrap` | กำหนด Root Component (ใช้ใน AppModule เท่านั้น) |

---

## 2. declarations, imports, exports, providers, bootstrap

### declarations — ประกาศสมาชิกของ Module

```typescript
// สิ่งที่ประกาศใน declarations ต้องเป็น:
// 1. Components
// 2. Directives
// 3. Pipes

@NgModule({
  declarations: [
    ProductListComponent,      // Component
    ProductCardComponent,      // Component
    HighlightDirective,        // Directive
    CurrencyThaiPipe,          // Pipe
  ],
})
export class ProductModule {}
```

**กฎสำคัญ**: แต่ละ Component/Directive/Pipe สามารถประกาศใน Module เดียวเท่านั้น

### imports — นำ Module อื่นมาใช้

```typescript
@NgModule({
  imports: [
    CommonModule,           // *ngIf, *ngFor, async pipe
    RouterModule,           // router-outlet, routerLink
    ReactiveFormsModule,    // FormGroup, FormControl
    MatButtonModule,        // Angular Material Button
    SharedModule,           // Shared components ของโปรเจค
  ],
})
export class ProductModule {}
```

### exports — แบ่งปันให้ Module อื่น

```typescript
@NgModule({
  declarations: [
    ButtonComponent,
    CardComponent,
    LoadingSpinnerComponent,
  ],
  exports: [
    // เฉพาะที่ต้องการให้ Module อื่นนำไปใช้
    ButtonComponent,
    CardComponent,
    LoadingSpinnerComponent,
    // สามารถ export Module ที่ import มาได้ด้วย
    CommonModule,
    ReactiveFormsModule,
  ],
})
export class SharedModule {}
```

### providers — Service Injection

```typescript
@NgModule({
  providers: [
    // แบบเก่า (scoped to module)
    ProductService,

    // กำหนด token เอง
    { provide: API_URL, useValue: 'https://api.example.com' },

    // ใช้ factory function
    {
      provide: LoggerService,
      useFactory: (config: AppConfig) => new LoggerService(config),
      deps: [AppConfig],
    },
  ],
})
export class ProductModule {}
```

### bootstrap — Root Component

```typescript
// เฉพาะ AppModule เท่านั้น
@NgModule({
  declarations: [AppComponent],
  bootstrap: [AppComponent],
})
export class AppModule {}
```

---

## 3. Feature Modules

Feature Module คือ Module ที่จัดกลุ่มฟีเจอร์เดียวกันเข้าด้วยกัน เช่น Products, Orders, Users

### โครงสร้างโปรเจค

```
src/app/
├── app.module.ts
├── app-routing.module.ts
├── features/
│   ├── products/
│   │   ├── products.module.ts
│   │   ├── products-routing.module.ts
│   │   ├── product-list/
│   │   ├── product-detail/
│   │   └── product-form/
│   ├── orders/
│   │   ├── orders.module.ts
│   │   └── ...
│   └── users/
│       ├── users.module.ts
│       └── ...
└── shared/
    ├── shared.module.ts
    └── ...
```

### สร้าง Feature Module

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
import { ProductCardComponent } from './product-card/product-card.component';

@NgModule({
  declarations: [
    ProductListComponent,
    ProductDetailComponent,
    ProductFormComponent,
    ProductCardComponent,
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

```typescript
// features/products/products-routing.module.ts
import { NgModule } from '@angular/core';
import { RouterModule, Routes } from '@angular/router';

import { ProductListComponent } from './product-list/product-list.component';
import { ProductDetailComponent } from './product-detail/product-detail.component';
import { ProductFormComponent } from './product-form/product-form.component';

const routes: Routes = [
  {
    path: '',
    component: ProductListComponent,
  },
  {
    path: ':id',
    component: ProductDetailComponent,
  },
  {
    path: 'new',
    component: ProductFormComponent,
  },
  {
    path: ':id/edit',
    component: ProductFormComponent,
  },
];

@NgModule({
  imports: [RouterModule.forChild(routes)],
  exports: [RouterModule],
})
export class ProductsRoutingModule {}
```

### ลงทะเบียน Feature Module ใน AppModule

```typescript
// app.module.ts
import { NgModule } from '@angular/core';
import { BrowserModule } from '@angular/platform-browser';
import { AppRoutingModule } from './app-routing.module';
import { AppComponent } from './app.component';
import { ProductsModule } from './features/products/products.module';
import { OrdersModule } from './features/orders/orders.module';

@NgModule({
  declarations: [AppComponent],
  imports: [
    BrowserModule,
    AppRoutingModule,
    ProductsModule,
    OrdersModule,
  ],
  bootstrap: [AppComponent],
})
export class AppModule {}
```

---

## 4. Shared Modules

Shared Module รวม Components, Directives, Pipes ที่ใช้ร่วมกันหลาย Feature Module

```typescript
// shared/shared.module.ts
import { NgModule } from '@angular/core';
import { CommonModule } from '@angular/common';
import { RouterModule } from '@angular/router';
import { ReactiveFormsModule, FormsModule } from '@angular/forms';

// Shared Components
import { ButtonComponent } from './components/button/button.component';
import { CardComponent } from './components/card/card.component';
import { ModalComponent } from './components/modal/modal.component';
import { LoadingSpinnerComponent } from './components/loading-spinner/loading-spinner.component';
import { AlertComponent } from './components/alert/alert.component';
import { PaginationComponent } from './components/pagination/pagination.component';
import { SearchBarComponent } from './components/search-bar/search-bar.component';
import { TableComponent } from './components/table/table.component';

// Shared Directives
import { ClickOutsideDirective } from './directives/click-outside.directive';
import { AutoFocusDirective } from './directives/auto-focus.directive';
import { LazyImageDirective } from './directives/lazy-image.directive';

// Shared Pipes
import { ThaiDatePipe } from './pipes/thai-date.pipe';
import { ThaiCurrencyPipe } from './pipes/thai-currency.pipe';
import { TruncatePipe } from './pipes/truncate.pipe';
import { SafeHtmlPipe } from './pipes/safe-html.pipe';

const SHARED_COMPONENTS = [
  ButtonComponent,
  CardComponent,
  ModalComponent,
  LoadingSpinnerComponent,
  AlertComponent,
  PaginationComponent,
  SearchBarComponent,
  TableComponent,
];

const SHARED_DIRECTIVES = [
  ClickOutsideDirective,
  AutoFocusDirective,
  LazyImageDirective,
];

const SHARED_PIPES = [
  ThaiDatePipe,
  ThaiCurrencyPipe,
  TruncatePipe,
  SafeHtmlPipe,
];

@NgModule({
  declarations: [
    ...SHARED_COMPONENTS,
    ...SHARED_DIRECTIVES,
    ...SHARED_PIPES,
  ],
  imports: [
    CommonModule,
    RouterModule,
    ReactiveFormsModule,
    FormsModule,
  ],
  exports: [
    // Export ทุก declarations
    ...SHARED_COMPONENTS,
    ...SHARED_DIRECTIVES,
    ...SHARED_PIPES,
    // Re-export Angular Modules ที่ใช้บ่อย
    CommonModule,
    RouterModule,
    ReactiveFormsModule,
    FormsModule,
  ],
})
export class SharedModule {}
```

### ตัวอย่าง Shared Components

```typescript
// shared/components/button/button.component.ts
import { Component, Input, Output, EventEmitter } from '@angular/core';

export type ButtonVariant = 'primary' | 'secondary' | 'danger' | 'outline';
export type ButtonSize = 'sm' | 'md' | 'lg';

@Component({
  selector: 'app-button',
  template: `
    <button
      [class]="buttonClass"
      [disabled]="disabled || loading"
      [type]="type"
      (click)="onClick($event)"
    >
      <app-loading-spinner *ngIf="loading" size="sm"></app-loading-spinner>
      <ng-content></ng-content>
    </button>
  `,
  styleUrls: ['./button.component.scss'],
})
export class ButtonComponent {
  @Input() variant: ButtonVariant = 'primary';
  @Input() size: ButtonSize = 'md';
  @Input() disabled = false;
  @Input() loading = false;
  @Input() type: 'button' | 'submit' | 'reset' = 'button';
  @Output() clicked = new EventEmitter<MouseEvent>();

  get buttonClass(): string {
    return `btn btn-${this.variant} btn-${this.size}`;
  }

  onClick(event: MouseEvent): void {
    if (!this.disabled && !this.loading) {
      this.clicked.emit(event);
    }
  }
}
```

```typescript
// shared/components/pagination/pagination.component.ts
import { Component, Input, Output, EventEmitter, OnChanges } from '@angular/core';

@Component({
  selector: 'app-pagination',
  template: `
    <nav class="pagination-nav" *ngIf="totalPages > 1">
      <button
        class="page-btn"
        [disabled]="currentPage === 1"
        (click)="changePage(currentPage - 1)"
      >
        &laquo; ก่อนหน้า
      </button>

      <button
        *ngFor="let page of visiblePages"
        class="page-btn"
        [class.active]="page === currentPage"
        (click)="changePage(page)"
      >
        {{ page }}
      </button>

      <button
        class="page-btn"
        [disabled]="currentPage === totalPages"
        (click)="changePage(currentPage + 1)"
      >
        ถัดไป &raquo;
      </button>
    </nav>
  `,
})
export class PaginationComponent implements OnChanges {
  @Input() currentPage = 1;
  @Input() totalItems = 0;
  @Input() pageSize = 10;
  @Output() pageChange = new EventEmitter<number>();

  totalPages = 0;
  visiblePages: number[] = [];

  ngOnChanges(): void {
    this.totalPages = Math.ceil(this.totalItems / this.pageSize);
    this.calculateVisiblePages();
  }

  private calculateVisiblePages(): void {
    const start = Math.max(1, this.currentPage - 2);
    const end = Math.min(this.totalPages, this.currentPage + 2);
    this.visiblePages = Array.from(
      { length: end - start + 1 },
      (_, i) => start + i
    );
  }

  changePage(page: number): void {
    if (page >= 1 && page <= this.totalPages) {
      this.pageChange.emit(page);
    }
  }
}
```

---

## 5. Core Module Pattern

Core Module มีไว้สำหรับ Services ที่ควรมี Instance เดียวตลอดแอปพลิเคชัน เช่น AuthService, LoggerService และ Components ที่แสดงผลครั้งเดียว เช่น Header, Footer, Sidebar

```typescript
// core/core.module.ts
import { NgModule, Optional, SkipSelf } from '@angular/core';
import { CommonModule } from '@angular/common';
import { HTTP_INTERCEPTORS } from '@angular/common/http';

// Core Components (แสดงครั้งเดียวใน AppComponent)
import { HeaderComponent } from './components/header/header.component';
import { FooterComponent } from './components/footer/footer.component';
import { SidebarComponent } from './components/sidebar/sidebar.component';
import { NavbarComponent } from './components/navbar/navbar.component';

// Core Services
import { AuthService } from './services/auth.service';
import { LoggerService } from './services/logger.service';
import { StorageService } from './services/storage.service';
import { NotificationService } from './services/notification.service';

// Interceptors
import { AuthInterceptor } from './interceptors/auth.interceptor';
import { ErrorInterceptor } from './interceptors/error.interceptor';
import { LoadingInterceptor } from './interceptors/loading.interceptor';

@NgModule({
  declarations: [
    HeaderComponent,
    FooterComponent,
    SidebarComponent,
    NavbarComponent,
  ],
  imports: [
    CommonModule,
  ],
  exports: [
    HeaderComponent,
    FooterComponent,
    SidebarComponent,
    NavbarComponent,
  ],
  providers: [
    AuthService,
    LoggerService,
    StorageService,
    NotificationService,
    {
      provide: HTTP_INTERCEPTORS,
      useClass: AuthInterceptor,
      multi: true,
    },
    {
      provide: HTTP_INTERCEPTORS,
      useClass: ErrorInterceptor,
      multi: true,
    },
    {
      provide: HTTP_INTERCEPTORS,
      useClass: LoadingInterceptor,
      multi: true,
    },
  ],
})
export class CoreModule {
  // Guard against importing CoreModule more than once
  constructor(@Optional() @SkipSelf() parentModule?: CoreModule) {
    if (parentModule) {
      throw new Error(
        'CoreModule is already loaded. Import it in the AppModule only.'
      );
    }
  }
}
```

### ใช้ CoreModule ใน AppModule

```typescript
// app.module.ts
import { NgModule } from '@angular/core';
import { BrowserModule } from '@angular/platform-browser';
import { CoreModule } from './core/core.module';
import { SharedModule } from './shared/shared.module';
import { AppRoutingModule } from './app-routing.module';
import { AppComponent } from './app.component';

@NgModule({
  declarations: [AppComponent],
  imports: [
    BrowserModule,
    CoreModule,       // import ครั้งเดียวที่นี่
    SharedModule,
    AppRoutingModule,
  ],
  bootstrap: [AppComponent],
})
export class AppModule {}
```

### AppComponent ใช้ Core Components

```typescript
// app.component.ts
import { Component } from '@angular/core';

@Component({
  selector: 'app-root',
  template: `
    <app-header></app-header>
    <app-navbar></app-navbar>
    <main class="main-content">
      <app-sidebar></app-sidebar>
      <div class="content">
        <router-outlet></router-outlet>
      </div>
    </main>
    <app-footer></app-footer>
  `,
})
export class AppComponent {}
```

---

## 6. Module ใน Standalone era (Angular 14+)

ใน Angular 14+ เราสามารถสร้าง Components โดยไม่ต้องใช้ NgModule ด้วย `standalone: true`

### Standalone Component

```typescript
// products/product-list/product-list.component.ts
import { Component, OnInit } from '@angular/core';
import { CommonModule } from '@angular/common';
import { RouterModule } from '@angular/router';
import { ProductService } from '../services/product.service';
import { ProductCardComponent } from '../product-card/product-card.component';
import { LoadingSpinnerComponent } from '../../shared/loading-spinner/loading-spinner.component';

@Component({
  selector: 'app-product-list',
  standalone: true,                    // ประกาศว่าเป็น Standalone
  imports: [                           // import สิ่งที่ต้องการโดยตรง
    CommonModule,
    RouterModule,
    ProductCardComponent,
    LoadingSpinnerComponent,
  ],
  template: `
    <div class="product-list">
      <app-loading-spinner *ngIf="loading"></app-loading-spinner>
      <div class="grid" *ngIf="!loading">
        <app-product-card
          *ngFor="let product of products"
          [product]="product"
        ></app-product-card>
      </div>
    </div>
  `,
})
export class ProductListComponent implements OnInit {
  products: any[] = [];
  loading = false;

  constructor(private productService: ProductService) {}

  ngOnInit(): void {
    this.loading = true;
    this.productService.getAll().subscribe({
      next: (products) => {
        this.products = products;
        this.loading = false;
      },
      error: () => {
        this.loading = false;
      },
    });
  }
}
```

### Bootstrapping Standalone Application

```typescript
// main.ts
import { bootstrapApplication } from '@angular/platform-browser';
import { provideRouter } from '@angular/router';
import { provideHttpClient, withInterceptors } from '@angular/common/http';
import { AppComponent } from './app/app.component';
import { routes } from './app/app.routes';
import { authInterceptor } from './app/core/interceptors/auth.interceptor';

bootstrapApplication(AppComponent, {
  providers: [
    provideRouter(routes),
    provideHttpClient(
      withInterceptors([authInterceptor])
    ),
  ],
}).catch(err => console.error(err));
```

### App Routes (Standalone)

```typescript
// app.routes.ts
import { Routes } from '@angular/router';

export const routes: Routes = [
  {
    path: '',
    redirectTo: '/home',
    pathMatch: 'full',
  },
  {
    path: 'home',
    loadComponent: () =>
      import('./features/home/home.component').then(m => m.HomeComponent),
  },
  {
    path: 'products',
    loadChildren: () =>
      import('./features/products/products.routes').then(m => m.productRoutes),
  },
];
```

### การผสม NgModule กับ Standalone Components

```typescript
// ใช้ Standalone Component ใน NgModule-based App
@NgModule({
  declarations: [
    // ไม่ต้อง declare Standalone Components
  ],
  imports: [
    // แต่ต้อง import เข้ามา
    StandaloneProductCardComponent,
    StandaloneSearchBarComponent,
  ],
})
export class ProductsModule {}
```

---

## 7. Workshop: E-commerce Module Structure

สร้างโครงสร้าง Module สำหรับระบบ E-commerce ที่สมบูรณ์

### โครงสร้างไฟล์

```
src/app/
├── core/
│   ├── core.module.ts
│   ├── components/
│   │   ├── header/
│   │   ├── footer/
│   │   └── nav/
│   ├── services/
│   │   ├── auth.service.ts
│   │   ├── cart.service.ts
│   │   └── api.service.ts
│   ├── guards/
│   │   └── auth.guard.ts
│   └── interceptors/
│       ├── auth.interceptor.ts
│       └── error.interceptor.ts
├── shared/
│   ├── shared.module.ts
│   ├── components/
│   │   ├── product-card/
│   │   ├── rating/
│   │   ├── price-tag/
│   │   └── badge/
│   ├── pipes/
│   │   ├── thai-currency.pipe.ts
│   │   └── discount.pipe.ts
│   └── directives/
│       └── add-to-cart.directive.ts
├── features/
│   ├── home/
│   │   └── home.module.ts
│   ├── products/
│   │   ├── products.module.ts
│   │   └── products-routing.module.ts
│   ├── cart/
│   │   ├── cart.module.ts
│   │   └── cart-routing.module.ts
│   ├── checkout/
│   │   ├── checkout.module.ts
│   │   └── checkout-routing.module.ts
│   ├── orders/
│   │   ├── orders.module.ts
│   │   └── orders-routing.module.ts
│   └── account/
│       ├── account.module.ts
│       └── account-routing.module.ts
├── app.module.ts
├── app-routing.module.ts
└── app.component.ts
```

### App Routing Module

```typescript
// app-routing.module.ts
import { NgModule } from '@angular/core';
import { RouterModule, Routes } from '@angular/router';
import { AuthGuard } from './core/guards/auth.guard';

const routes: Routes = [
  {
    path: '',
    redirectTo: 'home',
    pathMatch: 'full',
  },
  {
    path: 'home',
    loadChildren: () =>
      import('./features/home/home.module').then(m => m.HomeModule),
  },
  {
    path: 'products',
    loadChildren: () =>
      import('./features/products/products.module').then(m => m.ProductsModule),
  },
  {
    path: 'cart',
    loadChildren: () =>
      import('./features/cart/cart.module').then(m => m.CartModule),
    canActivate: [AuthGuard],
  },
  {
    path: 'checkout',
    loadChildren: () =>
      import('./features/checkout/checkout.module').then(m => m.CheckoutModule),
    canActivate: [AuthGuard],
  },
  {
    path: 'orders',
    loadChildren: () =>
      import('./features/orders/orders.module').then(m => m.OrdersModule),
    canActivate: [AuthGuard],
  },
  {
    path: 'account',
    loadChildren: () =>
      import('./features/account/account.module').then(m => m.AccountModule),
    canActivate: [AuthGuard],
  },
  {
    path: '**',
    loadChildren: () =>
      import('./features/not-found/not-found.module').then(m => m.NotFoundModule),
  },
];

@NgModule({
  imports: [RouterModule.forRoot(routes, {
    scrollPositionRestoration: 'top',
    anchorScrolling: 'enabled',
  })],
  exports: [RouterModule],
})
export class AppRoutingModule {}
```

### Products Module (Feature Module สมบูรณ์)

```typescript
// features/products/products.module.ts
import { NgModule } from '@angular/core';
import { CommonModule } from '@angular/common';
import { ReactiveFormsModule } from '@angular/forms';
import { InfiniteScrollModule } from 'ngx-infinite-scroll';

import { ProductsRoutingModule } from './products-routing.module';
import { SharedModule } from '../../shared/shared.module';

import { ProductListComponent } from './product-list/product-list.component';
import { ProductDetailComponent } from './product-detail/product-detail.component';
import { ProductSearchComponent } from './product-search/product-search.component';
import { ProductFilterComponent } from './product-filter/product-filter.component';
import { ProductReviewComponent } from './product-review/product-review.component';
import { ProductImageGalleryComponent } from './product-image-gallery/product-image-gallery.component';

import { ProductService } from './services/product.service';
import { ProductReviewService } from './services/product-review.service';
import { ProductSearchService } from './services/product-search.service';

@NgModule({
  declarations: [
    ProductListComponent,
    ProductDetailComponent,
    ProductSearchComponent,
    ProductFilterComponent,
    ProductReviewComponent,
    ProductImageGalleryComponent,
  ],
  imports: [
    CommonModule,
    ReactiveFormsModule,
    ProductsRoutingModule,
    SharedModule,
    InfiniteScrollModule,
  ],
  providers: [
    ProductService,
    ProductReviewService,
    ProductSearchService,
  ],
})
export class ProductsModule {}
```

### Cart Service (Core Service)

```typescript
// core/services/cart.service.ts
import { Injectable, signal, computed } from '@angular/core';

export interface CartItem {
  productId: string;
  name: string;
  price: number;
  quantity: number;
  imageUrl: string;
}

@Injectable({ providedIn: 'root' })
export class CartService {
  private items = signal<CartItem[]>([]);

  // Computed signals
  readonly cartItems = computed(() => this.items());
  readonly totalItems = computed(() =>
    this.items().reduce((sum, item) => sum + item.quantity, 0)
  );
  readonly totalPrice = computed(() =>
    this.items().reduce((sum, item) => sum + (item.price * item.quantity), 0)
  );

  addItem(product: Omit<CartItem, 'quantity'>): void {
    this.items.update(items => {
      const existing = items.find(i => i.productId === product.productId);
      if (existing) {
        return items.map(i =>
          i.productId === product.productId
            ? { ...i, quantity: i.quantity + 1 }
            : i
        );
      }
      return [...items, { ...product, quantity: 1 }];
    });
  }

  removeItem(productId: string): void {
    this.items.update(items => items.filter(i => i.productId !== productId));
  }

  updateQuantity(productId: string, quantity: number): void {
    if (quantity <= 0) {
      this.removeItem(productId);
      return;
    }
    this.items.update(items =>
      items.map(i => i.productId === productId ? { ...i, quantity } : i)
    );
  }

  clearCart(): void {
    this.items.set([]);
  }
}
```

---

## สรุป

| Pattern | เมื่อใช้ |
|---------|---------|
| **AppModule** | จุดเริ่มต้นของแอป |
| **Feature Module** | จัดกลุ่มฟีเจอร์เดียวกัน |
| **Shared Module** | Components/Pipes ที่ใช้ร่วมกัน |
| **Core Module** | Services ที่ใช้ตลอดแอป |
| **Standalone** | Angular 14+ ไม่ต้องใช้ NgModule |

### Best Practices

1. **ใช้ Feature Modules** สำหรับแต่ละฟีเจอร์ใหญ่
2. **Shared Module** ต้องไม่มี providers (ป้องกัน multiple instances)
3. **Core Module** import ครั้งเดียวใน AppModule
4. **Lazy Load** Feature Modules เพื่อประสิทธิภาพ
5. **Standalone Components** เหมาะสำหรับโปรเจคใหม่ใน Angular 14+
