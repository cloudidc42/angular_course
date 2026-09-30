# Part 08 — Routing พื้นฐาน

## สารบัญ

1. [Router Module Setup](#router-module-setup)
2. [Routes Configuration](#routes-configuration)
3. [RouterOutlet](#routeroutlet)
4. [RouterLink และ RouterLinkActive](#routerlink-และ-routerlinkactive)
5. [Programmatic Navigation](#programmatic-navigation)
6. [Route Parameters (:id)](#route-parameters-id)
7. [Query Parameters](#query-parameters)
8. [Child Routes](#child-routes)
9. [Named Outlets](#named-outlets)
10. [Workshop: Multi-page Product App](#workshop-multi-page-product-app)

---

## Router Module Setup

Angular Router เป็น Module ที่จัดการการนำทางระหว่างหน้าต่าง ๆ ใน Single Page Application (SPA)

### ติดตั้งและตั้งค่าเบื้องต้น

```bash
# สร้าง Project ใหม่พร้อม Routing
ng new my-app --routing

# หรือสร้าง Routing Module ทีหลัง
ng generate module app-routing --flat --module=app
```

### โครงสร้างไฟล์

```
src/
  app/
    app-routing.module.ts   ← กำหนด Routes ทั้งหมด
    app.module.ts           ← Import AppRoutingModule
    app.component.ts
    app.component.html      ← มี <router-outlet>
```

### app-routing.module.ts

```typescript
// src/app/app-routing.module.ts
import { NgModule } from '@angular/core';
import { RouterModule, Routes } from '@angular/router';

// Import Components
import { HomeComponent } from './pages/home/home.component';
import { AboutComponent } from './pages/about/about.component';
import { NotFoundComponent } from './pages/not-found/not-found.component';

const routes: Routes = [
  { path: '', component: HomeComponent },        // หน้าหลัก
  { path: 'about', component: AboutComponent },  // หน้า About
  { path: '**', component: NotFoundComponent }   // 404 — ต้องอยู่ท้ายสุด
];

@NgModule({
  imports: [RouterModule.forRoot(routes)],  // forRoot ใช้เพียงครั้งเดียว
  exports: [RouterModule]                   // Export เพื่อให้ใช้ directive ได้
})
export class AppRoutingModule { }
```

### app.module.ts

```typescript
// src/app/app.module.ts
import { NgModule } from '@angular/core';
import { BrowserModule } from '@angular/platform-browser';
import { AppRoutingModule } from './app-routing.module';
import { AppComponent } from './app.component';

@NgModule({
  declarations: [AppComponent],
  imports: [
    BrowserModule,
    AppRoutingModule  // ← ต้อง Import ด้วย
  ],
  bootstrap: [AppComponent]
})
export class AppModule { }
```

### RouterModule Options

```typescript
RouterModule.forRoot(routes, {
  // กำหนดว่า URL Hash จะใช้ '#' หรือ HTML5 History API
  useHash: false,  // default: false (HTML5 mode: /products)
                   // true = Hash mode: /#/products

  // เลื่อนหน้าไปบนสุดเมื่อ Navigate
  scrollPositionRestoration: 'top',  // 'disabled' | 'top' | 'enabled'

  // แสดง Router Events ใน Console (Debug)
  enableTracing: false,

  // ตัวเลือก Initial Navigation
  initialNavigation: 'enabledBlocking',  // สำหรับ Server-Side Rendering
})
```

---

## Routes Configuration

### รูปแบบ Route Object

```typescript
interface Route {
  path: string;                    // เส้นทาง URL
  component?: Type<any>;           // Component ที่จะแสดง
  redirectTo?: string;             // เปลี่ยนทางไป path อื่น
  pathMatch?: 'full' | 'prefix';   // วิธีจับคู่ URL
  children?: Route[];              // Routes ลูก
  loadChildren?: () => Promise;    // Lazy Loading
  canActivate?: any[];             // Guards
  data?: any;                      // ข้อมูลเพิ่มเติม
  title?: string;                  // Page Title (Angular 14+)
}
```

### ตัวอย่าง Routes ครบถ้วน

```typescript
// src/app/app-routing.module.ts
import { NgModule } from '@angular/core';
import { RouterModule, Routes } from '@angular/router';

import { HomeComponent } from './pages/home/home.component';
import { ProductListComponent } from './pages/products/product-list.component';
import { ProductDetailComponent } from './pages/products/product-detail.component';
import { ProductCreateComponent } from './pages/products/product-create.component';
import { ProductEditComponent } from './pages/products/product-edit.component';
import { CategoryComponent } from './pages/category/category.component';
import { SearchComponent } from './pages/search/search.component';
import { NotFoundComponent } from './pages/not-found/not-found.component';

const routes: Routes = [
  // หน้าหลัก
  {
    path: '',
    component: HomeComponent,
    title: 'หน้าหลัก'
  },

  // หน้าสินค้า
  {
    path: 'products',
    component: ProductListComponent,
    title: 'รายการสินค้า'
  },

  // สินค้าตาม ID
  {
    path: 'products/:id',
    component: ProductDetailComponent,
    title: 'รายละเอียดสินค้า'
  },

  // สร้างสินค้าใหม่
  {
    path: 'products/new',
    component: ProductCreateComponent,
    title: 'เพิ่มสินค้าใหม่'
  },

  // แก้ไขสินค้า
  {
    path: 'products/:id/edit',
    component: ProductEditComponent,
    title: 'แก้ไขสินค้า'
  },

  // หมวดหมู่
  {
    path: 'categories/:slug',
    component: CategoryComponent
  },

  // ค้นหา
  {
    path: 'search',
    component: SearchComponent
  },

  // Redirect
  {
    path: 'shop',
    redirectTo: 'products',
    pathMatch: 'full'  // ต้องตรงทั้ง path ไม่ใช่แค่ prefix
  },

  // 404 — ต้องอยู่ท้ายสุดเสมอ
  {
    path: '**',
    component: NotFoundComponent,
    title: 'ไม่พบหน้าที่ต้องการ'
  }
];

@NgModule({
  imports: [RouterModule.forRoot(routes)],
  exports: [RouterModule]
})
export class AppRoutingModule { }
```

### Route Data — ส่งข้อมูลพิเศษ

```typescript
const routes: Routes = [
  {
    path: 'admin',
    component: AdminComponent,
    data: {
      role: 'admin',
      breadcrumb: 'ผู้ดูแลระบบ',
      animation: 'AdminPage'
    }
  }
];
```

```typescript
// ดึงข้อมูลใน Component
import { ActivatedRoute } from '@angular/router';

@Component({ selector: 'app-admin', template: `...` })
export class AdminComponent {
  constructor(private route: ActivatedRoute) {
    const data = this.route.snapshot.data;
    console.log(data['role']);       // 'admin'
    console.log(data['breadcrumb']); // 'ผู้ดูแลระบบ'
  }
}
```

---

## RouterOutlet

`<router-outlet>` คือ Placeholder ที่บอก Angular ว่าจะแสดง Component ที่ตรงกับ Route ตรงไหน

### การใช้งานพื้นฐาน

```html
<!-- src/app/app.component.html -->
<nav>
  <a routerLink="/">หน้าหลัก</a>
  <a routerLink="/products">สินค้า</a>
  <a routerLink="/about">เกี่ยวกับเรา</a>
</nav>

<!-- Component ที่ตรงกับ Route ปัจจุบันจะแสดงตรงนี้ -->
<router-outlet></router-outlet>

<footer>Footer Content</footer>
```

### Router Events

```typescript
// ติดตาม Navigation Events
import { Router, NavigationStart, NavigationEnd, NavigationError } from '@angular/router';

@Component({ selector: 'app-root', template: `...` })
export class AppComponent implements OnInit {
  isLoading = false;

  constructor(private router: Router) { }

  ngOnInit(): void {
    this.router.events.subscribe(event => {
      if (event instanceof NavigationStart) {
        this.isLoading = true;
      }
      if (event instanceof NavigationEnd) {
        this.isLoading = false;
      }
      if (event instanceof NavigationError) {
        this.isLoading = false;
        console.error('Navigation Error:', event.error);
      }
    });
  }
}
```

---

## RouterLink และ RouterLinkActive

### RouterLink — สร้าง Link ไปยัง Route

```html
<!-- Static Path -->
<a routerLink="/products">ดูสินค้าทั้งหมด</a>

<!-- Dynamic Path ด้วย Array -->
<a [routerLink]="['/products', product.id]">{{ product.name }}</a>

<!-- Path พร้อม Query Parameters -->
<a [routerLink]="['/search']" [queryParams]="{ q: 'iphone', page: 1 }">
  ค้นหา iPhone
</a>

<!-- Path พร้อม Fragment (#section) -->
<a [routerLink]="['/about']" fragment="team">ทีมงาน</a>

<!-- เปิดใน Tab เดิม (default) -->
<a routerLink="/products" routerLinkActive="active">สินค้า</a>
```

### RouterLinkActive — เพิ่ม Class เมื่อ Active

```html
<!-- เพิ่ม class="active" เมื่อ URL ตรงกับ /products -->
<a routerLink="/products" routerLinkActive="active">สินค้า</a>

<!-- หลาย Classes -->
<a routerLink="/about" routerLinkActive="active highlighted">เกี่ยวกับ</a>

<!-- exactMatch — ต้องตรง URL ทั้งหมด (ไม่ใช่แค่ prefix) -->
<a routerLink="/" routerLinkActive="active" [routerLinkActiveOptions]="{ exact: true }">
  หน้าหลัก
</a>
```

### Navigation Menu ที่สมบูรณ์

```typescript
// src/app/components/navbar/navbar.component.ts
import { Component } from '@angular/core';

interface NavItem {
  label: string;
  path: string;
  exact?: boolean;
  icon?: string;
}

@Component({
  selector: 'app-navbar',
  templateUrl: './navbar.component.html',
  styleUrls: ['./navbar.component.css']
})
export class NavbarComponent {
  navItems: NavItem[] = [
    { label: 'หน้าหลัก', path: '/', exact: true, icon: '🏠' },
    { label: 'สินค้า', path: '/products', icon: '📦' },
    { label: 'หมวดหมู่', path: '/categories', icon: '📂' },
    { label: 'เกี่ยวกับ', path: '/about', icon: 'ℹ️' },
    { label: 'ติดต่อ', path: '/contact', icon: '📧' }
  ];
}
```

```html
<!-- src/app/components/navbar/navbar.component.html -->
<nav class="navbar">
  <div class="navbar-brand">
    <a routerLink="/">My Shop</a>
  </div>

  <ul class="navbar-nav">
    <li *ngFor="let item of navItems">
      <a
        [routerLink]="item.path"
        routerLinkActive="nav-active"
        [routerLinkActiveOptions]="{ exact: item.exact || false }"
        class="nav-link"
      >
        <span *ngIf="item.icon">{{ item.icon }}</span>
        {{ item.label }}
      </a>
    </li>
  </ul>
</nav>
```

---

## Programmatic Navigation

### การ Navigate ด้วย Code (Router.navigate)

```typescript
import { Router } from '@angular/router';

@Component({ selector: 'app-login', template: `...` })
export class LoginComponent {
  constructor(private router: Router) { }

  // Navigate ไปยัง Path แบบ Absolute
  goToHome(): void {
    this.router.navigate(['/']);
  }

  // Navigate พร้อม Parameters
  goToProduct(id: number): void {
    this.router.navigate(['/products', id]);
  }

  // Navigate พร้อม Query Parameters
  goToSearch(keyword: string): void {
    this.router.navigate(['/search'], {
      queryParams: { q: keyword, page: 1 }
    });
  }

  // Navigate แบบ Relative (ใช้ ActivatedRoute)
  goBack(): void {
    this.router.navigate(['../'], { relativeTo: this.route });
  }

  // Navigate พร้อม State (ไม่แสดงใน URL)
  goWithState(): void {
    this.router.navigate(['/products'], {
      state: { fromLogin: true, timestamp: Date.now() }
    });
  }
}
```

### navigateByUrl — Navigate ด้วย URL String

```typescript
// ใช้เมื่อมี URL string เต็ม ๆ
this.router.navigateByUrl('/products?sort=price&order=asc');

// Navigate ไปยังหน้าก่อนหน้า
this.router.navigateByUrl(this.previousUrl || '/');
```

### ตัวอย่าง Login with Redirect

```typescript
// src/app/pages/login/login.component.ts
import { Component } from '@angular/core';
import { Router, ActivatedRoute } from '@angular/router';
import { AuthService } from '../../services/auth.service';

@Component({
  selector: 'app-login',
  template: `
    <form (ngSubmit)="onLogin()">
      <input type="text" [(ngModel)]="username" name="username" placeholder="ชื่อผู้ใช้">
      <input type="password" [(ngModel)]="password" name="password" placeholder="รหัสผ่าน">
      <button type="submit" [disabled]="loading">
        {{ loading ? 'กำลังเข้าสู่ระบบ...' : 'เข้าสู่ระบบ' }}
      </button>
    </form>
    <p *ngIf="error" class="error">{{ error }}</p>
  `
})
export class LoginComponent {
  username = '';
  password = '';
  loading = false;
  error = '';

  private returnUrl: string;

  constructor(
    private router: Router,
    private route: ActivatedRoute,
    private authService: AuthService
  ) {
    // ดึง Return URL จาก Query Parameter
    this.returnUrl = this.route.snapshot.queryParams['returnUrl'] || '/';
  }

  onLogin(): void {
    this.loading = true;
    this.error = '';

    this.authService.login(this.username, this.password).subscribe({
      next: () => {
        // Login สำเร็จ → กลับไปหน้าที่ต้องการ
        this.router.navigateByUrl(this.returnUrl);
      },
      error: (err) => {
        this.error = 'ชื่อผู้ใช้หรือรหัสผ่านไม่ถูกต้อง';
        this.loading = false;
      }
    });
  }
}
```

---

## Route Parameters (:id)

### อ่าน Parameter จาก URL

```typescript
// URL: /products/42

import { Component, OnInit, OnDestroy } from '@angular/core';
import { ActivatedRoute, ParamMap } from '@angular/router';
import { Subscription } from 'rxjs';
import { switchMap } from 'rxjs/operators';
import { ProductService } from '../../services/product.service';
import { Product } from '../../models/product.model';

@Component({
  selector: 'app-product-detail',
  templateUrl: './product-detail.component.html'
})
export class ProductDetailComponent implements OnInit, OnDestroy {
  product: Product | null = null;
  loading = true;
  error = '';
  private sub?: Subscription;

  constructor(
    private route: ActivatedRoute,
    private productService: ProductService
  ) { }

  ngOnInit(): void {
    // วิธีที่ 1: Snapshot (ใช้เมื่อ Component ไม่ถูก Reuse)
    const id = +this.route.snapshot.paramMap.get('id')!;
    this.loadProduct(id);

    // วิธีที่ 2: Observable (แนะนำ — ใช้เมื่อ Component ถูก Reuse)
    this.sub = this.route.paramMap.pipe(
      switchMap((params: ParamMap) => {
        const id = +params.get('id')!;
        this.loading = true;
        return this.productService.getById(id);
      })
    ).subscribe({
      next: (product) => {
        this.product = product;
        this.loading = false;
      },
      error: (err) => {
        this.error = err.message;
        this.loading = false;
      }
    });
  }

  ngOnDestroy(): void {
    this.sub?.unsubscribe();
  }

  private loadProduct(id: number): void {
    this.productService.getById(id).subscribe({
      next: (p) => { this.product = p; this.loading = false; },
      error: (e) => { this.error = e.message; this.loading = false; }
    });
  }
}
```

```html
<!-- product-detail.component.html -->
<div *ngIf="loading" class="loading">กำลังโหลด...</div>

<div *ngIf="error" class="error-message">
  <p>{{ error }}</p>
  <a routerLink="/products">กลับไปรายการสินค้า</a>
</div>

<div *ngIf="product && !loading" class="product-detail">
  <img [src]="product.imageUrl" [alt]="product.name">
  <h1>{{ product.name }}</h1>
  <p class="price">{{ product.price | currency:'THB':'symbol':'1.2-2' }}</p>
  <p class="description">{{ product.description }}</p>
  <div class="actions">
    <button class="btn btn-primary">เพิ่มลงตะกร้า</button>
    <a [routerLink]="['/products', product.id, 'edit']" class="btn btn-secondary">
      แก้ไข
    </a>
  </div>
</div>
```

### Multiple Parameters

```typescript
// URL: /categories/electronics/products/42
{
  path: 'categories/:categorySlug/products/:productId',
  component: ProductDetailComponent
}
```

```typescript
ngOnInit(): void {
  this.route.paramMap.subscribe(params => {
    const categorySlug = params.get('categorySlug');
    const productId = +params.get('productId')!;
    console.log('Category:', categorySlug);
    console.log('Product ID:', productId);
  });
}
```

---

## Query Parameters

Query Parameters คือข้อมูลที่ต่อท้าย URL ด้วย `?key=value` เช่น `/products?page=2&sort=price`

### ส่ง Query Parameters

```html
<!-- ใน Template -->
<a [routerLink]="['/products']"
   [queryParams]="{ page: currentPage, sort: 'price', order: 'asc' }">
  สินค้าราคาต่ำไปสูง
</a>
```

```typescript
// ใน Component
this.router.navigate(['/products'], {
  queryParams: { page: 2, sort: 'name', order: 'desc' }
});

// รักษา Query Params เดิมและเพิ่มเติม
this.router.navigate(['/products'], {
  queryParams: { page: 3 },
  queryParamsHandling: 'merge'  // 'merge' | 'preserve'
});
```

### อ่าน Query Parameters

```typescript
// src/app/pages/products/product-list.component.ts
import { Component, OnInit } from '@angular/core';
import { ActivatedRoute, Router } from '@angular/router';
import { combineLatest } from 'rxjs';
import { map, switchMap } from 'rxjs/operators';
import { ProductService } from '../../services/product.service';

@Component({
  selector: 'app-product-list',
  templateUrl: './product-list.component.html'
})
export class ProductListComponent implements OnInit {
  products: any[] = [];
  currentPage = 1;
  sortBy = 'name';
  order = 'asc';
  searchQuery = '';
  loading = false;

  constructor(
    private route: ActivatedRoute,
    private router: Router,
    private productService: ProductService
  ) { }

  ngOnInit(): void {
    // ติดตาม Query Params แบบ Reactive
    this.route.queryParamMap.subscribe(params => {
      this.currentPage = +( params.get('page') || 1 );
      this.sortBy = params.get('sort') || 'name';
      this.order = params.get('order') || 'asc';
      this.searchQuery = params.get('q') || '';

      this.loadProducts();
    });
  }

  loadProducts(): void {
    this.loading = true;
    this.productService.getAll().subscribe(products => {
      this.products = products;
      this.loading = false;
    });
  }

  onSearch(query: string): void {
    this.router.navigate(['/products'], {
      queryParams: { q: query, page: 1 },
      queryParamsHandling: 'merge'
    });
  }

  onSortChange(sort: string): void {
    const order = this.sortBy === sort && this.order === 'asc' ? 'desc' : 'asc';
    this.router.navigate(['.'], {
      relativeTo: this.route,
      queryParams: { sort, order, page: 1 },
      queryParamsHandling: 'merge'
    });
  }

  onPageChange(page: number): void {
    this.router.navigate(['.'], {
      relativeTo: this.route,
      queryParams: { page },
      queryParamsHandling: 'merge'
    });
  }
}
```

---

## Child Routes

Child Routes ใช้เมื่อต้องการ Nested Routing เช่น หน้า Dashboard ที่มีหน้าย่อย

### การกำหนด Child Routes

```typescript
// app-routing.module.ts
const routes: Routes = [
  {
    path: 'dashboard',
    component: DashboardComponent,
    children: [
      { path: '', redirectTo: 'overview', pathMatch: 'full' },
      { path: 'overview', component: OverviewComponent },
      { path: 'analytics', component: AnalyticsComponent },
      { path: 'settings', component: SettingsComponent },
      { path: 'profile', component: ProfileComponent }
    ]
  }
];
```

### DashboardComponent — ต้องมี router-outlet

```typescript
// src/app/pages/dashboard/dashboard.component.ts
@Component({
  selector: 'app-dashboard',
  template: `
    <div class="dashboard">
      <aside class="sidebar">
        <nav>
          <a routerLink="overview" routerLinkActive="active">ภาพรวม</a>
          <a routerLink="analytics" routerLinkActive="active">วิเคราะห์</a>
          <a routerLink="settings" routerLinkActive="active">ตั้งค่า</a>
          <a routerLink="profile" routerLinkActive="active">โปรไฟล์</a>
        </nav>
      </aside>

      <main class="content">
        <!-- Child Component แสดงตรงนี้ -->
        <router-outlet></router-outlet>
      </main>
    </div>
  `
})
export class DashboardComponent { }
```

### Nested Child Routes (หลายระดับ)

```typescript
const routes: Routes = [
  {
    path: 'admin',
    component: AdminLayoutComponent,
    children: [
      {
        path: 'products',
        component: AdminProductsComponent,
        children: [
          { path: '', component: ProductListComponent },
          { path: 'new', component: ProductCreateComponent },
          { path: ':id', component: ProductDetailComponent },
          { path: ':id/edit', component: ProductEditComponent }
        ]
      },
      {
        path: 'users',
        component: AdminUsersComponent,
        children: [
          { path: '', component: UserListComponent },
          { path: ':id', component: UserDetailComponent }
        ]
      }
    ]
  }
];
```

---

## Named Outlets

Named Outlets ใช้เมื่อต้องการแสดง Component หลายตัวพร้อมกันใน Router

```typescript
// ตัวอย่าง: หน้าที่มี Sidebar แยกกับ Content หลัก
const routes: Routes = [
  {
    path: 'products',
    component: ProductListComponent
  },
  {
    path: 'product-detail',
    component: ProductDetailComponent,
    outlet: 'detail'  // Named Outlet ชื่อ 'detail'
  }
];
```

```html
<!-- app.component.html -->
<div class="layout">
  <div class="main-content">
    <router-outlet></router-outlet>           <!-- Primary outlet -->
  </div>
  <aside class="sidebar">
    <router-outlet name="detail"></router-outlet>  <!-- Named outlet -->
  </aside>
</div>
```

```typescript
// Navigate ไปยัง Named Outlet
this.router.navigate([
  { outlets: { primary: ['products'], detail: ['product-detail'] } }
]);

// ปิด Named Outlet
this.router.navigate([{ outlets: { detail: null } }]);
```

---

## Workshop: Multi-page Product App

### โครงสร้างโปรเจกต์

```
src/app/
  models/
    product.model.ts
  services/
    product.service.ts
  pages/
    home/
      home.component.ts
      home.component.html
      home.component.css
    products/
      product-list/
        product-list.component.ts
        product-list.component.html
      product-detail/
        product-detail.component.ts
        product-detail.component.html
      product-create/
        product-create.component.ts
        product-create.component.html
      product-edit/
        product-edit.component.ts
        product-edit.component.html
    not-found/
      not-found.component.ts
  components/
    navbar/
      navbar.component.ts
      navbar.component.html
    product-card/
      product-card.component.ts
      product-card.component.html
```

### Step 1: สร้าง Routes

```typescript
// src/app/app-routing.module.ts
import { NgModule } from '@angular/core';
import { RouterModule, Routes } from '@angular/router';

import { HomeComponent } from './pages/home/home.component';
import { ProductListComponent } from './pages/products/product-list/product-list.component';
import { ProductDetailComponent } from './pages/products/product-detail/product-detail.component';
import { ProductCreateComponent } from './pages/products/product-create/product-create.component';
import { ProductEditComponent } from './pages/products/product-edit/product-edit.component';
import { NotFoundComponent } from './pages/not-found/not-found.component';

const routes: Routes = [
  { path: '', component: HomeComponent, title: 'My Shop — หน้าหลัก' },
  { path: 'products', component: ProductListComponent, title: 'สินค้าทั้งหมด' },
  { path: 'products/new', component: ProductCreateComponent, title: 'เพิ่มสินค้า' },
  { path: 'products/:id', component: ProductDetailComponent },
  { path: 'products/:id/edit', component: ProductEditComponent },
  { path: '**', component: NotFoundComponent, title: '404 — ไม่พบหน้า' }
];

@NgModule({
  imports: [RouterModule.forRoot(routes, { scrollPositionRestoration: 'top' })],
  exports: [RouterModule]
})
export class AppRoutingModule { }
```

### Step 2: App Component

```typescript
// src/app/app.component.ts
import { Component } from '@angular/core';

@Component({
  selector: 'app-root',
  template: `
    <app-navbar></app-navbar>
    <main class="container">
      <router-outlet></router-outlet>
    </main>
    <footer class="footer">
      <p>© 2024 My Shop. สงวนลิขสิทธิ์</p>
    </footer>
  `,
  styles: [`
    .container { min-height: 80vh; padding: 2rem 1rem; }
    .footer { background: #333; color: white; text-align: center; padding: 1rem; }
  `]
})
export class AppComponent { }
```

### Step 3: Home Component

```typescript
// src/app/pages/home/home.component.ts
import { Component, OnInit } from '@angular/core';
import { Router } from '@angular/router';
import { ProductService } from '../../services/product.service';
import { Product } from '../../models/product.model';

@Component({
  selector: 'app-home',
  templateUrl: './home.component.html'
})
export class HomeComponent implements OnInit {
  featuredProducts: Product[] = [];
  searchQuery = '';
  loading = false;

  constructor(
    private productService: ProductService,
    private router: Router
  ) { }

  ngOnInit(): void {
    this.loading = true;
    this.productService.getAll().subscribe(products => {
      this.featuredProducts = products.slice(0, 6);
      this.loading = false;
    });
  }

  onSearch(): void {
    if (this.searchQuery.trim()) {
      this.router.navigate(['/products'], {
        queryParams: { q: this.searchQuery.trim() }
      });
    }
  }
}
```

```html
<!-- src/app/pages/home/home.component.html -->
<section class="hero">
  <h1>ยินดีต้อนรับสู่ My Shop</h1>
  <p>สินค้าคุณภาพ ราคาเป็นมิตร</p>

  <div class="search-box">
    <input
      type="text"
      [(ngModel)]="searchQuery"
      placeholder="ค้นหาสินค้า..."
      (keyup.enter)="onSearch()"
    >
    <button (click)="onSearch()" class="btn btn-primary">ค้นหา</button>
  </div>

  <a routerLink="/products" class="btn btn-secondary">ดูสินค้าทั้งหมด</a>
</section>

<section class="featured-products">
  <h2>สินค้าแนะนำ</h2>

  <div *ngIf="loading">กำลังโหลด...</div>

  <div class="product-grid" *ngIf="!loading">
    <app-product-card
      *ngFor="let product of featuredProducts"
      [product]="product"
    ></app-product-card>
  </div>
</section>
```

### Step 4: Product List Component

```typescript
// src/app/pages/products/product-list/product-list.component.ts
import { Component, OnInit } from '@angular/core';
import { ActivatedRoute, Router } from '@angular/router';
import { ProductService } from '../../../services/product.service';
import { Product } from '../../../models/product.model';

@Component({
  selector: 'app-product-list',
  templateUrl: './product-list.component.html'
})
export class ProductListComponent implements OnInit {
  products: Product[] = [];
  filteredProducts: Product[] = [];
  loading = false;
  searchQuery = '';
  sortBy = 'name';

  constructor(
    private productService: ProductService,
    private route: ActivatedRoute,
    private router: Router
  ) { }

  ngOnInit(): void {
    this.loading = true;
    this.productService.getAll().subscribe(products => {
      this.products = products;
      this.loading = false;

      // อ่าน Query Params
      this.route.queryParamMap.subscribe(params => {
        this.searchQuery = params.get('q') || '';
        this.sortBy = params.get('sort') || 'name';
        this.applyFilters();
      });
    });
  }

  applyFilters(): void {
    let result = [...this.products];

    // กรองตาม Search
    if (this.searchQuery) {
      const q = this.searchQuery.toLowerCase();
      result = result.filter(p =>
        p.name.toLowerCase().includes(q) ||
        p.description.toLowerCase().includes(q)
      );
    }

    // เรียงลำดับ
    result.sort((a, b) => {
      if (this.sortBy === 'price') return a.price - b.price;
      if (this.sortBy === 'name') return a.name.localeCompare(b.name);
      return 0;
    });

    this.filteredProducts = result;
  }

  onSortChange(sort: string): void {
    this.router.navigate(['.'], {
      relativeTo: this.route,
      queryParams: { sort },
      queryParamsHandling: 'merge'
    });
  }

  deleteProduct(id: number): void {
    if (confirm('ต้องการลบสินค้านี้?')) {
      this.productService.delete(id).subscribe(() => {
        this.products = this.products.filter(p => p.id !== id);
        this.applyFilters();
      });
    }
  }
}
```

```html
<!-- product-list.component.html -->
<div class="page-header">
  <h1>สินค้าทั้งหมด</h1>
  <a routerLink="/products/new" class="btn btn-primary">+ เพิ่มสินค้า</a>
</div>

<div class="filters">
  <span>เรียงตาม:</span>
  <button (click)="onSortChange('name')"
          [class.active]="sortBy === 'name'">ชื่อ</button>
  <button (click)="onSortChange('price')"
          [class.active]="sortBy === 'price'">ราคา</button>
</div>

<p *ngIf="searchQuery" class="search-info">
  ผลการค้นหา "{{ searchQuery }}": {{ filteredProducts.length }} รายการ
</p>

<div *ngIf="loading">กำลังโหลดสินค้า...</div>

<div class="product-grid" *ngIf="!loading">
  <div *ngIf="filteredProducts.length === 0" class="empty-state">
    ไม่พบสินค้าที่ตรงกับการค้นหา
  </div>

  <app-product-card
    *ngFor="let product of filteredProducts"
    [product]="product"
    [showActions]="true"
    (onDelete)="deleteProduct($event)"
  ></app-product-card>
</div>
```

### Step 5: Product Card Component

```typescript
// src/app/components/product-card/product-card.component.ts
import { Component, Input, Output, EventEmitter } from '@angular/core';
import { Product } from '../../models/product.model';

@Component({
  selector: 'app-product-card',
  templateUrl: './product-card.component.html',
  styleUrls: ['./product-card.component.css']
})
export class ProductCardComponent {
  @Input() product!: Product;
  @Input() showActions = false;
  @Output() onDelete = new EventEmitter<number>();

  deleteProduct(): void {
    this.onDelete.emit(this.product.id);
  }
}
```

```html
<!-- product-card.component.html -->
<div class="product-card">
  <a [routerLink]="['/products', product.id]">
    <img [src]="product.imageUrl || 'assets/default.jpg'"
         [alt]="product.name"
         class="product-image">
    <div class="product-info">
      <h3>{{ product.name }}</h3>
      <p class="price">฿{{ product.price | number:'1.2-2' }}</p>
      <p class="description">{{ product.description | slice:0:80 }}...</p>
    </div>
  </a>

  <div *ngIf="showActions" class="product-actions">
    <a [routerLink]="['/products', product.id]" class="btn btn-sm btn-info">ดู</a>
    <a [routerLink]="['/products', product.id, 'edit']" class="btn btn-sm btn-warning">แก้ไข</a>
    <button (click)="deleteProduct()" class="btn btn-sm btn-danger">ลบ</button>
  </div>
</div>
```

### Step 6: Product Detail Component

```typescript
// src/app/pages/products/product-detail/product-detail.component.ts
import { Component, OnInit } from '@angular/core';
import { ActivatedRoute, Router } from '@angular/router';
import { ProductService } from '../../../services/product.service';
import { Product } from '../../../models/product.model';

@Component({
  selector: 'app-product-detail',
  templateUrl: './product-detail.component.html'
})
export class ProductDetailComponent implements OnInit {
  product: Product | null = null;
  loading = true;
  error = '';

  constructor(
    private route: ActivatedRoute,
    private router: Router,
    private productService: ProductService
  ) { }

  ngOnInit(): void {
    const id = +this.route.snapshot.paramMap.get('id')!;
    this.productService.getById(id).subscribe({
      next: (product) => { this.product = product; this.loading = false; },
      error: (err) => { this.error = err.message; this.loading = false; }
    });
  }

  goBack(): void {
    this.router.navigate(['/products']);
  }

  deleteProduct(): void {
    if (!this.product) return;
    if (confirm(`ต้องการลบ "${this.product.name}"?`)) {
      this.productService.delete(this.product.id).subscribe(() => {
        this.router.navigate(['/products']);
      });
    }
  }
}
```

```html
<!-- product-detail.component.html -->
<div *ngIf="loading">กำลังโหลดข้อมูลสินค้า...</div>

<div *ngIf="error" class="alert alert-danger">
  {{ error }}
  <button (click)="goBack()">กลับ</button>
</div>

<div *ngIf="product && !loading" class="product-detail">
  <nav class="breadcrumb">
    <a routerLink="/">หน้าหลัก</a> /
    <a routerLink="/products">สินค้า</a> /
    <span>{{ product.name }}</span>
  </nav>

  <div class="product-layout">
    <div class="product-image">
      <img [src]="product.imageUrl || 'assets/default.jpg'" [alt]="product.name">
    </div>

    <div class="product-info">
      <h1>{{ product.name }}</h1>
      <p class="price">฿{{ product.price | number:'1.2-2' }}</p>
      <p class="stock">คลัง: {{ product.stock }} ชิ้น</p>
      <p class="description">{{ product.description }}</p>

      <div class="actions">
        <button class="btn btn-primary" [disabled]="product.stock === 0">
          {{ product.stock > 0 ? 'เพิ่มลงตะกร้า' : 'สินค้าหมด' }}
        </button>
        <a [routerLink]="['/products', product.id, 'edit']" class="btn btn-secondary">
          แก้ไข
        </a>
        <button (click)="deleteProduct()" class="btn btn-danger">ลบสินค้า</button>
      </div>
    </div>
  </div>

  <div class="back-link">
    <a (click)="goBack()" style="cursor: pointer">← กลับไปรายการสินค้า</a>
  </div>
</div>
```

### Step 7: Not Found Component

```typescript
// src/app/pages/not-found/not-found.component.ts
import { Component } from '@angular/core';
import { Location } from '@angular/common';

@Component({
  selector: 'app-not-found',
  template: `
    <div class="not-found">
      <h1>404</h1>
      <p>ไม่พบหน้าที่คุณต้องการ</p>
      <div class="actions">
        <button (click)="goBack()" class="btn btn-secondary">กลับหน้าก่อน</button>
        <a routerLink="/" class="btn btn-primary">ไปหน้าหลัก</a>
      </div>
    </div>
  `,
  styles: [`
    .not-found { text-align: center; padding: 4rem; }
    h1 { font-size: 6rem; color: #ccc; margin: 0; }
    p { font-size: 1.5rem; color: #666; }
    .actions { display: flex; gap: 1rem; justify-content: center; margin-top: 2rem; }
  `]
})
export class NotFoundComponent {
  constructor(private location: Location) { }

  goBack(): void {
    this.location.back();
  }
}
```

### Step 8: ลงทะเบียนทุก Component ใน Module

```typescript
// src/app/app.module.ts
import { NgModule } from '@angular/core';
import { BrowserModule } from '@angular/platform-browser';
import { FormsModule } from '@angular/forms';
import { HttpClientModule } from '@angular/common/http';

import { AppRoutingModule } from './app-routing.module';
import { AppComponent } from './app.component';
import { NavbarComponent } from './components/navbar/navbar.component';
import { ProductCardComponent } from './components/product-card/product-card.component';
import { HomeComponent } from './pages/home/home.component';
import { ProductListComponent } from './pages/products/product-list/product-list.component';
import { ProductDetailComponent } from './pages/products/product-detail/product-detail.component';
import { ProductCreateComponent } from './pages/products/product-create/product-create.component';
import { ProductEditComponent } from './pages/products/product-edit/product-edit.component';
import { NotFoundComponent } from './pages/not-found/not-found.component';

@NgModule({
  declarations: [
    AppComponent,
    NavbarComponent,
    ProductCardComponent,
    HomeComponent,
    ProductListComponent,
    ProductDetailComponent,
    ProductCreateComponent,
    ProductEditComponent,
    NotFoundComponent
  ],
  imports: [
    BrowserModule,
    FormsModule,
    HttpClientModule,
    AppRoutingModule
  ],
  bootstrap: [AppComponent]
})
export class AppModule { }
```

---

## สรุป

| Concept | คำอธิบาย |
|---------|---------|
| `RouterModule.forRoot()` | ตั้งค่า Router สำหรับ Root Module |
| `RouterModule.forChild()` | ตั้งค่า Router สำหรับ Feature Module |
| `<router-outlet>` | Placeholder สำหรับแสดง Component |
| `routerLink` | Directive สำหรับสร้าง Link |
| `routerLinkActive` | เพิ่ม Class เมื่อ Route Active |
| `Router.navigate()` | Programmatic Navigation |
| `ActivatedRoute` | ดึงข้อมูล Route ปัจจุบัน |
| `paramMap` | อ่าน Route Parameters |
| `queryParamMap` | อ่าน Query Parameters |
| Child Routes | Routes ซ้อนกัน (Nested) |
| Named Outlets | แสดงหลาย Component พร้อมกัน |

---

*เอกสารนี้เป็นส่วนหนึ่งของหลักสูตร Angular — Part 08*
