# Part 25 — โครงสร้างโปรเจ็กต์ที่ดี

## บทนำ

โครงสร้างโปรเจ็กต์ที่ดีทำให้โค้ด:
- อ่านง่ายและเข้าใจง่าย
- Scale ได้เมื่อโปรเจ็กต์โต
- ทำงานร่วมกันใน team ได้ดี
- Test ได้ง่าย
- Maintain ได้ในระยะยาว

---

## 1. Feature-based vs Type-based Structure

### 1.1 Type-based Structure (ไม่แนะนำสำหรับ app ใหญ่)

จัดกลุ่มตามประเภทของไฟล์

```
src/app/
├── components/
│   ├── header/
│   ├── footer/
│   ├── product-list/
│   └── user-profile/
├── services/
│   ├── auth.service.ts
│   ├── product.service.ts
│   └── user.service.ts
├── models/
│   ├── user.model.ts
│   └── product.model.ts
├── pipes/
│   └── currency.pipe.ts
└── directives/
    └── highlight.directive.ts
```

**ข้อดี:** ง่ายสำหรับโปรเจ็กต์เล็ก
**ข้อเสีย:** ยากเมื่อ app โต (ต้องเปิดหลายโฟลเดอร์เพื่อดูหนึ่ง feature)

### 1.2 Feature-based Structure (แนะนำ)

จัดกลุ่มตาม feature/domain

```
src/app/
├── core/                      # Singleton services, guards, interceptors
│   ├── auth/
│   │   ├── auth.service.ts
│   │   ├── auth.guard.ts
│   │   └── auth.interceptor.ts
│   ├── http/
│   │   └── http-error.interceptor.ts
│   └── core.module.ts         # หรือ providers array สำหรับ standalone
│
├── shared/                    # Reusable components, pipes, directives
│   ├── components/
│   │   ├── button/
│   │   ├── modal/
│   │   └── table/
│   ├── pipes/
│   │   ├── currency-thai.pipe.ts
│   │   └── truncate.pipe.ts
│   ├── directives/
│   │   └── tooltip.directive.ts
│   └── index.ts               # Barrel export
│
├── features/                  # Feature modules
│   ├── auth/                  # Authentication feature
│   │   ├── components/
│   │   │   ├── login/
│   │   │   └── register/
│   │   ├── services/
│   │   │   └── auth-feature.service.ts
│   │   ├── guards/
│   │   │   └── login.guard.ts
│   │   ├── models/
│   │   │   └── user.model.ts
│   │   ├── auth.routes.ts
│   │   └── index.ts
│   │
│   ├── products/              # Products feature
│   │   ├── components/
│   │   │   ├── product-list/
│   │   │   ├── product-detail/
│   │   │   └── product-form/
│   │   ├── services/
│   │   │   └── product.service.ts
│   │   ├── store/
│   │   │   └── product.store.ts
│   │   ├── models/
│   │   │   └── product.model.ts
│   │   ├── products.routes.ts
│   │   └── index.ts
│   │
│   └── cart/                  # Shopping cart feature
│       ├── components/
│       ├── services/
│       ├── store/
│       ├── cart.routes.ts
│       └── index.ts
│
├── layout/                    # Layout components
│   ├── header/
│   ├── footer/
│   ├── sidebar/
│   └── layout.component.ts
│
├── app.component.ts
├── app.config.ts              # หรือ app.module.ts
└── app.routes.ts
```

---

## 2. Core, Shared, Features Modules

### 2.1 Core Module/Config

Core ควรมี services ที่เป็น singleton และใช้ทั้ง application

```typescript
// src/app/core/core.providers.ts
import { Provider, APP_INITIALIZER } from '@angular/core';
import { HTTP_INTERCEPTORS } from '@angular/common/http';
import { AuthInterceptor } from './auth/auth.interceptor';
import { HttpErrorInterceptor } from './http/http-error.interceptor';
import { AuthService } from './auth/auth.service';

// App initializer — รันก่อน app เริ่ม
function initializeApp(authService: AuthService): () => Promise<void> {
  return () => authService.initialize();
}

export const coreProviders: Provider[] = [
  // HTTP Interceptors
  {
    provide: HTTP_INTERCEPTORS,
    useClass: AuthInterceptor,
    multi: true
  },
  {
    provide: HTTP_INTERCEPTORS,
    useClass: HttpErrorInterceptor,
    multi: true
  },
  // App Initializer
  {
    provide: APP_INITIALIZER,
    useFactory: initializeApp,
    deps: [AuthService],
    multi: true
  }
];
```

```typescript
// src/app/core/auth/auth.interceptor.ts
import {
  HttpInterceptor,
  HttpRequest,
  HttpHandler,
  HttpEvent
} from '@angular/common/http';
import { Injectable } from '@angular/core';
import { Observable } from 'rxjs';
import { AuthService } from './auth.service';

@Injectable()
export class AuthInterceptor implements HttpInterceptor {

  constructor(private authService: AuthService) {}

  intercept(
    req: HttpRequest<unknown>,
    next: HttpHandler
  ): Observable<HttpEvent<unknown>> {
    const token = this.authService.getToken();

    if (token) {
      const authReq = req.clone({
        headers: req.headers.set('Authorization', `Bearer ${token}`)
      });
      return next.handle(authReq);
    }

    return next.handle(req);
  }
}
```

```typescript
// src/app/core/http/http-error.interceptor.ts
import {
  HttpInterceptor,
  HttpRequest,
  HttpHandler,
  HttpEvent,
  HttpErrorResponse
} from '@angular/common/http';
import { Injectable } from '@angular/core';
import { Observable, throwError } from 'rxjs';
import { catchError, retry } from 'rxjs/operators';
import { Router } from '@angular/router';

@Injectable()
export class HttpErrorInterceptor implements HttpInterceptor {

  constructor(private router: Router) {}

  intercept(
    req: HttpRequest<unknown>,
    next: HttpHandler
  ): Observable<HttpEvent<unknown>> {
    return next.handle(req).pipe(
      retry(1),  // retry อีก 1 ครั้งหากเกิด error
      catchError((error: HttpErrorResponse) => {
        if (error.status === 401) {
          this.router.navigate(['/login']);
        } else if (error.status === 403) {
          this.router.navigate(['/forbidden']);
        } else if (error.status === 404) {
          this.router.navigate(['/not-found']);
        }

        console.error('HTTP Error:', error.status, error.message);
        return throwError(() => error);
      })
    );
  }
}
```

### 2.2 Shared Module/Components

Shared ควรมี components, pipes, directives ที่ใช้ร่วมกันหลาย features

```typescript
// src/app/shared/components/button/button.component.ts
import { Component, Input, Output, EventEmitter } from '@angular/core';
import { CommonModule } from '@angular/common';

type ButtonVariant = 'primary' | 'secondary' | 'danger' | 'outline';
type ButtonSize = 'sm' | 'md' | 'lg';

@Component({
  selector: 'app-button',
  standalone: true,
  imports: [CommonModule],
  template: `
    <button
      [class]="buttonClasses"
      [disabled]="disabled || loading"
      [type]="type"
      (click)="onClick.emit($event)"
    >
      <span *ngIf="loading" class="spinner"></span>
      <ng-content></ng-content>
    </button>
  `,
  styles: [`
    .btn { padding: 8px 16px; border-radius: 4px; cursor: pointer; border: none; }
    .btn-primary { background: #007bff; color: white; }
    .btn-secondary { background: #6c757d; color: white; }
    .btn-danger { background: #dc3545; color: white; }
    .btn-outline { background: transparent; border: 1px solid #007bff; color: #007bff; }
    .btn-sm { padding: 4px 8px; font-size: 14px; }
    .btn-lg { padding: 12px 24px; font-size: 18px; }
    .btn:disabled { opacity: 0.65; cursor: not-allowed; }
    .spinner { display: inline-block; width: 16px; height: 16px; border: 2px solid rgba(255,255,255,0.3); border-top: 2px solid white; border-radius: 50%; animation: spin 0.8s linear infinite; margin-right: 8px; }
    @keyframes spin { to { transform: rotate(360deg); } }
  `]
})
export class ButtonComponent {
  @Input() variant: ButtonVariant = 'primary';
  @Input() size: ButtonSize = 'md';
  @Input() disabled = false;
  @Input() loading = false;
  @Input() type: 'button' | 'submit' | 'reset' = 'button';

  @Output() onClick = new EventEmitter<MouseEvent>();

  get buttonClasses(): string {
    return `btn btn-${this.variant} btn-${this.size}`;
  }
}
```

```typescript
// src/app/shared/components/modal/modal.component.ts
import {
  Component,
  Input,
  Output,
  EventEmitter,
  HostListener
} from '@angular/core';
import { CommonModule } from '@angular/common';

@Component({
  selector: 'app-modal',
  standalone: true,
  imports: [CommonModule],
  template: `
    <div
      *ngIf="isOpen"
      class="modal-overlay"
      (click)="onOverlayClick($event)"
    >
      <div class="modal-container" [style.maxWidth]="width">
        <!-- Header -->
        <div class="modal-header">
          <h3>{{ title }}</h3>
          <button (click)="close()" class="close-btn">✕</button>
        </div>

        <!-- Body -->
        <div class="modal-body">
          <ng-content></ng-content>
        </div>

        <!-- Footer -->
        <div class="modal-footer" *ngIf="showFooter">
          <ng-content select="[slot=footer]"></ng-content>
        </div>
      </div>
    </div>
  `,
  styles: [`
    .modal-overlay {
      position: fixed;
      top: 0; left: 0;
      width: 100%; height: 100%;
      background: rgba(0,0,0,0.5);
      display: flex;
      align-items: center;
      justify-content: center;
      z-index: 1000;
    }
    .modal-container {
      background: white;
      border-radius: 8px;
      width: 90%;
      max-height: 90vh;
      overflow-y: auto;
    }
    .modal-header {
      display: flex;
      justify-content: space-between;
      align-items: center;
      padding: 16px 20px;
      border-bottom: 1px solid #eee;
    }
    .modal-body { padding: 20px; }
    .modal-footer { padding: 16px 20px; border-top: 1px solid #eee; }
    .close-btn { background: none; border: none; font-size: 20px; cursor: pointer; }
  `]
})
export class ModalComponent {
  @Input() isOpen = false;
  @Input() title = '';
  @Input() width = '500px';
  @Input() showFooter = true;
  @Input() closeOnOverlayClick = true;

  @Output() closed = new EventEmitter<void>();

  @HostListener('document:keydown.escape')
  onEscape(): void {
    this.close();
  }

  close(): void {
    this.closed.emit();
  }

  onOverlayClick(event: MouseEvent): void {
    if (
      this.closeOnOverlayClick &&
      (event.target as HTMLElement).classList.contains('modal-overlay')
    ) {
      this.close();
    }
  }
}
```

---

## 3. Barrel exports (index.ts)

Barrel exports ช่วยให้ import โค้ดได้สะดวกขึ้น

### 3.1 สร้าง index.ts

```typescript
// src/app/shared/index.ts — Barrel export สำหรับ shared

// Components
export { ButtonComponent } from './components/button/button.component';
export { ModalComponent } from './components/modal/modal.component';
export { TableComponent } from './components/table/table.component';
export { LoadingComponent } from './components/loading/loading.component';
export { EmptyStateComponent } from './components/empty-state/empty-state.component';

// Pipes
export { CurrencyThaiPipe } from './pipes/currency-thai.pipe';
export { TruncatePipe } from './pipes/truncate.pipe';
export { TimeAgoPipe } from './pipes/time-ago.pipe';

// Directives
export { TooltipDirective } from './directives/tooltip.directive';
export { HighlightDirective } from './directives/highlight.directive';
export { ClickOutsideDirective } from './directives/click-outside.directive';

// Models / Interfaces
export type { TableColumn, TableConfig } from './components/table/table.types';

// Utils
export { formatDate, formatCurrency } from './utils/format.utils';
export { validateEmail, validatePhone } from './utils/validators.utils';
```

```typescript
// src/app/features/products/index.ts — Feature barrel export

// Public API ของ products feature
export { ProductListComponent } from './components/product-list/product-list.component';
export { ProductDetailComponent } from './components/product-detail/product-detail.component';
export { ProductService } from './services/product.service';
export type { Product, ProductFilter, ProductState } from './models/product.model';
export { productRoutes } from './products.routes';
```

### 3.2 การ import ด้วย Barrel

```typescript
// ก่อนใช้ Barrel (ยุ่งยาก)
import { ButtonComponent } from '../../shared/components/button/button.component';
import { ModalComponent } from '../../shared/components/modal/modal.component';
import { CurrencyThaiPipe } from '../../shared/pipes/currency-thai.pipe';
import { TooltipDirective } from '../../shared/directives/tooltip.directive';

// หลังใช้ Barrel (สะอาด)
import {
  ButtonComponent,
  ModalComponent,
  CurrencyThaiPipe,
  TooltipDirective
} from '../../shared';
```

---

## 4. Naming Conventions

### 4.1 ชื่อไฟล์ (kebab-case)

```
ชื่อไฟล์: <feature>.<type>.ts

Components:
  product-list.component.ts
  user-profile.component.ts

Services:
  product.service.ts
  auth.service.ts

Modules:
  products.module.ts
  shared.module.ts

Routes:
  products.routes.ts
  app.routes.ts

Models/Interfaces:
  product.model.ts
  user.interface.ts

Pipes:
  currency-thai.pipe.ts
  truncate.pipe.ts

Directives:
  tooltip.directive.ts
  highlight.directive.ts

Guards:
  auth.guard.ts
  admin.guard.ts

Interceptors:
  auth.interceptor.ts
  http-error.interceptor.ts

Resolvers:
  product-detail.resolver.ts

Tests:
  product.service.spec.ts
  product-list.component.spec.ts

Store:
  product.store.ts
  product.actions.ts (NgRx)
  product.reducer.ts (NgRx)
  product.effects.ts (NgRx)
  product.selectors.ts (NgRx)
```

### 4.2 ชื่อ Class (PascalCase)

```typescript
// Components
export class ProductListComponent {}
export class UserProfileComponent {}

// Services
export class ProductService {}
export class AuthService {}

// Modules
export class ProductsModule {}
export class SharedModule {}

// Pipes
export class CurrencyThaiPipe {}
export class TruncatePipe {}

// Directives
export class TooltipDirective {}
export class HighlightDirective {}

// Guards
export class AuthGuard {}
export class AdminGuard {}

// Models/Interfaces (ไม่ต้องมี prefix I)
export interface Product {}
export interface User {}
export class ProductModel {}  // หรือเป็น class ถ้าต้องการ methods
```

### 4.3 ชื่อ Methods และ Properties

```typescript
// Methods: camelCase + verb
loadProducts(): void {}
getUserById(id: number): User {}
isAuthenticated(): boolean {}
hasPermission(permission: string): boolean {}

// Private: prefix ด้วย _ (optional, บางทีไม่ใส่ก็ได้)
private _cachedData: Product[] = [];
// หรือแค่
private cachedData: Product[] = [];

// Subjects: ลงท้ายด้วย Subject
private userSubject = new BehaviorSubject<User | null>(null);

// Observables: ลงท้ายด้วย $
user$ = this.userSubject.asObservable();
products$: Observable<Product[]>;
loading$: Observable<boolean>;

// Booleans: prefix ด้วย is, has, can, should
isLoading = false;
hasError = false;
canDelete = false;
shouldShowModal = false;

// Constants: SCREAMING_SNAKE_CASE
const MAX_RETRY_COUNT = 3;
const API_BASE_URL = 'https://api.example.com';

// Enums: PascalCase สำหรับ enum, UPPER_CASE สำหรับ values
enum UserRole {
  ADMIN = 'ADMIN',
  USER = 'USER',
  GUEST = 'GUEST'
}
```

### 4.4 Selector Naming

```typescript
// Components: app-feature-name (kebab-case)
@Component({ selector: 'app-product-list' })
@Component({ selector: 'app-user-profile' })
@Component({ selector: 'app-shared-button' })

// Directives (Attribute): appDirectiveName (camelCase)
@Directive({ selector: '[appTooltip]' })
@Directive({ selector: '[appHighlight]' })
@Directive({ selector: '[appPermission]' })

// Structural Directives: *appDirectiveName
@Directive({ selector: '[appRepeat]' })  // ใช้ *appRepeat

// Pipes: camelCase
@Pipe({ name: 'currencyThai' })
@Pipe({ name: 'truncate' })
@Pipe({ name: 'timeAgo' })
```

---

## 5. Workshop: Refactor โปรเจ็กต์ให้สะอาด

### 5.1 โครงสร้างสมบูรณ์ที่แนะนำ

```
my-angular-app/
├── src/
│   ├── app/
│   │   │
│   │   ├── core/                         # Singleton providers
│   │   │   ├── auth/
│   │   │   │   ├── auth.service.ts
│   │   │   │   ├── auth.guard.ts
│   │   │   │   ├── auth.interceptor.ts
│   │   │   │   └── index.ts
│   │   │   ├── http/
│   │   │   │   └── http-error.interceptor.ts
│   │   │   ├── layout/
│   │   │   │   ├── header/
│   │   │   │   ├── footer/
│   │   │   │   └── sidebar/
│   │   │   └── core.providers.ts
│   │   │
│   │   ├── shared/                        # Reusable pieces
│   │   │   ├── components/
│   │   │   │   ├── button/
│   │   │   │   ├── modal/
│   │   │   │   ├── table/
│   │   │   │   └── loading-spinner/
│   │   │   ├── directives/
│   │   │   │   ├── tooltip.directive.ts
│   │   │   │   └── permission.directive.ts
│   │   │   ├── pipes/
│   │   │   │   └── currency-thai.pipe.ts
│   │   │   ├── utils/
│   │   │   │   ├── format.utils.ts
│   │   │   │   └── validators.utils.ts
│   │   │   └── index.ts                   # Barrel export
│   │   │
│   │   ├── features/                      # Feature modules
│   │   │   ├── dashboard/
│   │   │   │   ├── components/
│   │   │   │   │   └── dashboard.component.ts
│   │   │   │   ├── dashboard.routes.ts
│   │   │   │   └── index.ts
│   │   │   │
│   │   │   ├── products/
│   │   │   │   ├── components/
│   │   │   │   │   ├── product-list/
│   │   │   │   │   │   ├── product-list.component.ts
│   │   │   │   │   │   └── product-list.component.spec.ts
│   │   │   │   │   ├── product-detail/
│   │   │   │   │   │   ├── product-detail.component.ts
│   │   │   │   │   │   └── product-detail.component.spec.ts
│   │   │   │   │   └── product-form/
│   │   │   │   │       ├── product-form.component.ts
│   │   │   │   │       └── product-form.component.spec.ts
│   │   │   │   ├── services/
│   │   │   │   │   ├── product.service.ts
│   │   │   │   │   └── product.service.spec.ts
│   │   │   │   ├── store/
│   │   │   │   │   └── product.store.ts
│   │   │   │   ├── models/
│   │   │   │   │   └── product.model.ts
│   │   │   │   ├── products.routes.ts
│   │   │   │   └── index.ts
│   │   │   │
│   │   │   └── cart/
│   │   │       ├── components/
│   │   │       ├── services/
│   │   │       ├── store/
│   │   │       ├── cart.routes.ts
│   │   │       └── index.ts
│   │   │
│   │   ├── app.component.ts
│   │   ├── app.config.ts
│   │   └── app.routes.ts
│   │
│   ├── environments/
│   │   ├── environment.ts
│   │   └── environment.production.ts
│   │
│   ├── assets/
│   │   ├── images/
│   │   ├── icons/
│   │   └── i18n/
│   │
│   ├── styles/
│   │   ├── _variables.scss
│   │   ├── _mixins.scss
│   │   ├── _components.scss
│   │   └── styles.scss
│   │
│   └── main.ts
│
├── .angular/
├── .github/
├── node_modules/
├── angular.json
├── package.json
├── tsconfig.json
├── tsconfig.app.json
└── tsconfig.spec.json
```

### 5.2 TypeScript Path Aliases

```json
// tsconfig.json — กำหนด path aliases
{
  "compilerOptions": {
    "baseUrl": "src",
    "paths": {
      "@core/*": ["app/core/*"],
      "@shared/*": ["app/shared/*"],
      "@features/*": ["app/features/*"],
      "@env/*": ["environments/*"]
    }
  }
}
```

```typescript
// ก่อน path alias (ยุ่งยาก)
import { AuthService } from '../../../core/auth/auth.service';
import { ButtonComponent } from '../../../shared/components/button/button.component';

// หลัง path alias (สะอาด)
import { AuthService } from '@core/auth/auth.service';
import { ButtonComponent } from '@shared/components/button/button.component';
```

### 5.3 Angular.json Configuration

```json
{
  "projects": {
    "my-app": {
      "architect": {
        "build": {
          "options": {
            "styles": [
              "src/styles/styles.scss"
            ],
            "assets": [
              "src/favicon.ico",
              "src/assets"
            ]
          }
        }
      }
    }
  }
}
```

### 5.4 Environment Configuration

```typescript
// src/environments/environment.ts
export const environment = {
  production: false,
  apiUrl: 'http://localhost:3000/api',
  googleMapsKey: 'dev-key',
  features: {
    darkMode: true,
    notifications: true,
    betaFeatures: false
  }
};

// src/environments/environment.production.ts
export const environment = {
  production: true,
  apiUrl: 'https://api.myapp.com/api',
  googleMapsKey: 'prod-key',
  features: {
    darkMode: true,
    notifications: true,
    betaFeatures: false
  }
};
```

```typescript
// การใช้งาน environment
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { environment } from '@env/environment';

@Injectable({ providedIn: 'root' })
export class ProductService {
  private baseUrl = `${environment.apiUrl}/products`;

  constructor(private http: HttpClient) {}

  getProducts() {
    return this.http.get(this.baseUrl);
  }
}
```

### 5.5 Lazy Loading Routes

```typescript
// src/app/app.routes.ts
import { Routes } from '@angular/router';
import { AuthGuard } from '@core/auth/auth.guard';

export const routes: Routes = [
  {
    path: '',
    redirectTo: '/dashboard',
    pathMatch: 'full'
  },
  {
    path: 'auth',
    loadChildren: () =>
      import('@features/auth/auth.routes').then(m => m.authRoutes)
  },
  {
    path: 'dashboard',
    canActivate: [AuthGuard],
    loadChildren: () =>
      import('@features/dashboard/dashboard.routes')
        .then(m => m.dashboardRoutes)
  },
  {
    path: 'products',
    canActivate: [AuthGuard],
    loadChildren: () =>
      import('@features/products/products.routes')
        .then(m => m.productRoutes)
  },
  {
    path: 'cart',
    canActivate: [AuthGuard],
    loadChildren: () =>
      import('@features/cart/cart.routes').then(m => m.cartRoutes)
  },
  {
    path: '**',
    loadComponent: () =>
      import('./core/layout/not-found/not-found.component')
        .then(m => m.NotFoundComponent)
  }
];
```

### 5.6 App Config (Standalone)

```typescript
// src/app/app.config.ts
import { ApplicationConfig, importProvidersFrom } from '@angular/core';
import { provideRouter, withViewTransitions } from '@angular/router';
import { provideHttpClient, withInterceptors } from '@angular/common/http';
import { provideAnimations } from '@angular/platform-browser/animations';
import { routes } from './app.routes';
import { authInterceptorFn } from '@core/auth/auth.interceptor';
import { httpErrorInterceptorFn } from '@core/http/http-error.interceptor';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes, withViewTransitions()),
    provideHttpClient(
      withInterceptors([
        authInterceptorFn,
        httpErrorInterceptorFn
      ])
    ),
    provideAnimations()
  ]
};
```

---

## สรุปบทที่ 25

### Checklist โครงสร้างโปรเจ็กต์ที่ดี

- [ ] ใช้ Feature-based structure
- [ ] แยก core, shared, features ออกจากกัน
- [ ] สร้าง barrel exports (index.ts) ทุก module
- [ ] ตั้งชื่อไฟล์แบบ kebab-case + suffix
- [ ] ตั้งชื่อ class แบบ PascalCase
- [ ] ตั้งชื่อ observables ลงท้ายด้วย `$`
- [ ] ตั้งค่า TypeScript path aliases
- [ ] ใช้ environment files สำหรับ config
- [ ] ใช้ lazy loading ทุก feature module
- [ ] แยก styles ออกเป็น partials (_variables, _mixins)
- [ ] มี test files คู่กับทุกไฟล์

### สิ่งที่ควรทำใน Core

| ไฟล์ | ใช้สำหรับ |
|------|---------|
| AuthService | จัดการ authentication |
| AuthGuard | ป้องกัน routes ที่ต้อง login |
| AuthInterceptor | แนบ token ใน HTTP headers |
| HttpErrorInterceptor | จัดการ HTTP errors |
| Layout components | Header, Footer, Sidebar |

### สิ่งที่ควรทำใน Shared

| ไฟล์ | ใช้สำหรับ |
|------|---------|
| ButtonComponent | ปุ่มที่ใช้ทั้ง app |
| ModalComponent | Modal dialogs |
| TableComponent | Data tables |
| LoadingComponent | Loading states |
| CurrencyThaiPipe | Format ราคา |
| TooltipDirective | Tooltips |

### Anti-patterns ที่ควรหลีกเลี่ยง

1. **God Service** — service เดียวทำทุกอย่าง
2. **Feature ใน Shared** — ใส่ feature-specific ไว้ใน shared
3. **Circular dependencies** — A import B, B import A
4. **Import จาก deep paths** — ควรใช้ barrel หรือ path alias
5. **ไม่มี index.ts** — ทำให้ import ยุ่ง
6. **ชื่อไม่สื่อความหมาย** — component1, service2
