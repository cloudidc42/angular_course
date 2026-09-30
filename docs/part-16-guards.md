# Part 16: Route Guards

## บทนำ

Route Guards คือ Interfaces ที่ควบคุมการเข้าถึง Routes ใน Angular ใช้สำหรับการตรวจสอบสิทธิ์ (Authentication) การตรวจสอบบทบาท (Authorization) และการป้องกันการออกจากหน้า (Unsaved Changes)

---

## 1. AuthGuard, RoleGuard

### AuthGuard — ตรวจสอบการล็อกอิน

```typescript
// core/guards/auth.guard.ts (แบบ Class-based Angular 14-)
import { Injectable } from '@angular/core';
import {
  CanActivate,
  ActivatedRouteSnapshot,
  RouterStateSnapshot,
  Router,
  UrlTree,
} from '@angular/router';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';
import { AuthService } from '../services/auth.service';

@Injectable({ providedIn: 'root' })
export class AuthGuard implements CanActivate {
  constructor(
    private authService: AuthService,
    private router: Router
  ) {}

  canActivate(
    route: ActivatedRouteSnapshot,
    state: RouterStateSnapshot
  ): Observable<boolean | UrlTree> {
    return this.authService.isLoggedIn$.pipe(
      map(isLoggedIn => {
        if (isLoggedIn) {
          return true;
        }
        // Redirect ไป login พร้อมบันทึก URL ที่ต้องการไป
        return this.router.createUrlTree(['/auth/login'], {
          queryParams: { returnUrl: state.url },
        });
      })
    );
  }
}
```

### RoleGuard — ตรวจสอบบทบาท

```typescript
// core/guards/role.guard.ts
import { Injectable } from '@angular/core';
import {
  CanActivate,
  ActivatedRouteSnapshot,
  RouterStateSnapshot,
  Router,
  UrlTree,
} from '@angular/router';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';
import { AuthService } from '../services/auth.service';

export type UserRole = 'admin' | 'manager' | 'staff' | 'customer';

@Injectable({ providedIn: 'root' })
export class RoleGuard implements CanActivate {
  constructor(
    private authService: AuthService,
    private router: Router
  ) {}

  canActivate(
    route: ActivatedRouteSnapshot,
    state: RouterStateSnapshot
  ): Observable<boolean | UrlTree> {
    const requiredRole = route.data['role'] as UserRole;
    const requiredRoles = route.data['roles'] as UserRole[] | undefined;

    return this.authService.currentUser$.pipe(
      map(user => {
        if (!user) {
          return this.router.createUrlTree(['/auth/login']);
        }

        // ตรวจสอบ role เดียว
        if (requiredRole && !this.hasRole(user.roles, requiredRole)) {
          return this.router.createUrlTree(['/forbidden']);
        }

        // ตรวจสอบหลาย roles (ต้องมีอย่างน้อยหนึ่ง)
        if (requiredRoles && !requiredRoles.some(r => this.hasRole(user.roles, r))) {
          return this.router.createUrlTree(['/forbidden']);
        }

        return true;
      })
    );
  }

  private hasRole(userRoles: UserRole[], requiredRole: UserRole): boolean {
    return userRoles.includes(requiredRole);
  }
}
```

### PermissionGuard — ตรวจสอบสิทธิ์ละเอียด

```typescript
// core/guards/permission.guard.ts
import { Injectable } from '@angular/core';
import { CanActivate, ActivatedRouteSnapshot, Router } from '@angular/router';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';
import { PermissionService } from '../services/permission.service';

@Injectable({ providedIn: 'root' })
export class PermissionGuard implements CanActivate {
  constructor(
    private permissionService: PermissionService,
    private router: Router
  ) {}

  canActivate(route: ActivatedRouteSnapshot): Observable<boolean> {
    const requiredPermission = route.data['permission'] as string;

    return this.permissionService.hasPermission(requiredPermission).pipe(
      map(hasPermission => {
        if (hasPermission) return true;
        this.router.navigate(['/forbidden']);
        return false;
      })
    );
  }
}
```

---

## 2. canActivate, canDeactivate, canMatch

### canActivate — ควบคุมการเข้า Route

```typescript
// ใช้ใน Routes
const routes: Routes = [
  {
    path: 'dashboard',
    component: DashboardComponent,
    canActivate: [AuthGuard],
  },
  {
    path: 'admin',
    component: AdminComponent,
    canActivate: [AuthGuard, RoleGuard],
    data: { role: 'admin' },
  },
  {
    path: 'reports',
    component: ReportsComponent,
    canActivate: [AuthGuard, PermissionGuard],
    data: { permission: 'reports.view' },
  },
];
```

### canDeactivate — ป้องกันการออกจากหน้า

```typescript
// core/guards/unsaved-changes.guard.ts
import { Injectable } from '@angular/core';
import { CanDeactivate } from '@angular/router';
import { Observable } from 'rxjs';

// Interface สำหรับ Components ที่ต้องการใช้ Guard นี้
export interface CanComponentDeactivate {
  canDeactivate: () => boolean | Observable<boolean>;
}

@Injectable({ providedIn: 'root' })
export class UnsavedChangesGuard implements CanDeactivate<CanComponentDeactivate> {
  canDeactivate(
    component: CanComponentDeactivate
  ): boolean | Observable<boolean> {
    return component.canDeactivate ? component.canDeactivate() : true;
  }
}
```

```typescript
// features/products/product-form/product-form.component.ts
import { Component, OnInit } from '@angular/core';
import { FormBuilder, FormGroup, Validators } from '@angular/forms';
import { CanComponentDeactivate } from '../../../core/guards/unsaved-changes.guard';
import { Observable, of } from 'rxjs';

@Component({
  selector: 'app-product-form',
  template: `
    <form [formGroup]="productForm" (ngSubmit)="onSubmit()">
      <div class="form-group">
        <label>ชื่อสินค้า</label>
        <input formControlName="name" class="form-control">
      </div>
      <div class="form-group">
        <label>ราคา</label>
        <input formControlName="price" type="number" class="form-control">
      </div>
      <div class="form-group">
        <label>รายละเอียด</label>
        <textarea formControlName="description" class="form-control"></textarea>
      </div>
      <div class="form-actions">
        <button type="submit" [disabled]="productForm.invalid">บันทึก</button>
        <button type="button" (click)="cancel()">ยกเลิก</button>
      </div>
    </form>
  `,
})
export class ProductFormComponent implements OnInit, CanComponentDeactivate {
  productForm!: FormGroup;
  private savedSuccessfully = false;

  constructor(private fb: FormBuilder) {}

  ngOnInit(): void {
    this.productForm = this.fb.group({
      name: ['', Validators.required],
      price: [0, [Validators.required, Validators.min(0)]],
      description: [''],
    });
  }

  canDeactivate(): boolean | Observable<boolean> {
    if (this.savedSuccessfully || this.productForm.pristine) {
      return true;
    }

    // แสดง Dialog ถามผู้ใช้
    const confirm = window.confirm(
      'คุณมีการเปลี่ยนแปลงที่ยังไม่ได้บันทึก\nต้องการออกจากหน้านี้หรือไม่?'
    );
    return confirm;
  }

  onSubmit(): void {
    if (this.productForm.valid) {
      // บันทึกข้อมูล...
      this.savedSuccessfully = true;
    }
  }

  cancel(): void {
    // Router จะถาม canDeactivate ก่อน navigate
  }
}
```

### canMatch — ควบคุมว่า Route จะ match หรือไม่

```typescript
// canMatch ต่างจาก canActivate ตรงที่
// canActivate: Route match แล้วค่อย guard (redirect ได้)
// canMatch: guard ก่อน match (ถ้า false จะหา route ถัดไป)

const routes: Routes = [
  {
    path: 'dashboard',
    // Admin Dashboard
    loadComponent: () =>
      import('./admin-dashboard/admin-dashboard.component')
        .then(c => c.AdminDashboardComponent),
    canMatch: [() => inject(AuthService).hasRole('admin')],
  },
  {
    path: 'dashboard',
    // User Dashboard (fallback)
    loadComponent: () =>
      import('./user-dashboard/user-dashboard.component')
        .then(c => c.UserDashboardComponent),
    canMatch: [() => inject(AuthService).isLoggedIn()],
  },
  {
    path: 'dashboard',
    // Guest Dashboard (fallback)
    redirectTo: '/home',
  },
];
```

### canActivateChild — ควบคุม Child Routes

```typescript
// core/guards/admin.guard.ts
import { Injectable } from '@angular/core';
import { CanActivateChild, ActivatedRouteSnapshot, RouterStateSnapshot, Router } from '@angular/router';

@Injectable({ providedIn: 'root' })
export class AdminGuard implements CanActivateChild {
  constructor(
    private authService: AuthService,
    private router: Router
  ) {}

  canActivateChild(
    childRoute: ActivatedRouteSnapshot,
    state: RouterStateSnapshot
  ): boolean {
    if (this.authService.isAdmin()) {
      return true;
    }
    this.router.navigate(['/forbidden']);
    return false;
  }
}
```

```typescript
// ใช้กับ parent route
const routes: Routes = [
  {
    path: 'admin',
    component: AdminLayoutComponent,
    canActivateChild: [AdminGuard],
    children: [
      { path: 'products', component: AdminProductsComponent },
      { path: 'users', component: AdminUsersComponent },
      { path: 'settings', component: AdminSettingsComponent },
    ],
  },
];
```

---

## 3. Functional Guards (Angular 14+)

ใน Angular 14+ สามารถใช้ Guard เป็น Function แทน Class ได้ ทำให้โค้ดกระชับขึ้น

### Functional AuthGuard

```typescript
// core/guards/auth.guard.ts (Angular 14+ functional style)
import { inject } from '@angular/core';
import { CanActivateFn, Router } from '@angular/router';
import { map } from 'rxjs/operators';
import { AuthService } from '../services/auth.service';

export const authGuard: CanActivateFn = (route, state) => {
  const authService = inject(AuthService);
  const router = inject(Router);

  return authService.isLoggedIn$.pipe(
    map(isLoggedIn => {
      if (isLoggedIn) return true;
      return router.createUrlTree(['/auth/login'], {
        queryParams: { returnUrl: state.url },
      });
    })
  );
};
```

### Functional RoleGuard

```typescript
// core/guards/role.guard.ts (functional)
import { inject } from '@angular/core';
import { CanActivateFn, Router } from '@angular/router';
import { map } from 'rxjs/operators';
import { AuthService } from '../services/auth.service';

export const roleGuard = (requiredRole: string): CanActivateFn => {
  return (route, state) => {
    const authService = inject(AuthService);
    const router = inject(Router);

    return authService.currentUser$.pipe(
      map(user => {
        if (!user) {
          return router.createUrlTree(['/auth/login']);
        }
        if (user.roles.includes(requiredRole)) {
          return true;
        }
        return router.createUrlTree(['/forbidden']);
      })
    );
  };
};
```

### Functional canDeactivate Guard

```typescript
// core/guards/unsaved-changes.guard.ts (functional)
import { CanDeactivateFn } from '@angular/router';
import { CanComponentDeactivate } from '../interfaces/can-deactivate.interface';

export const unsavedChangesGuard: CanDeactivateFn<CanComponentDeactivate> = (
  component
) => {
  return component.canDeactivate ? component.canDeactivate() : true;
};
```

### ใช้ Functional Guards ใน Routes

```typescript
// app.routes.ts
import { Routes } from '@angular/router';
import { authGuard } from './core/guards/auth.guard';
import { roleGuard } from './core/guards/role.guard';
import { unsavedChangesGuard } from './core/guards/unsaved-changes.guard';

export const routes: Routes = [
  {
    path: 'home',
    loadComponent: () =>
      import('./features/home/home.component').then(c => c.HomeComponent),
  },
  {
    path: 'products',
    loadChildren: () =>
      import('./features/products/products.routes').then(r => r.productRoutes),
    canActivate: [authGuard],
  },
  {
    path: 'admin',
    loadChildren: () =>
      import('./features/admin/admin.routes').then(r => r.adminRoutes),
    canActivate: [authGuard, roleGuard('admin')],
  },
  {
    path: 'products/edit/:id',
    loadComponent: () =>
      import('./features/products/product-form/product-form.component')
        .then(c => c.ProductFormComponent),
    canActivate: [authGuard],
    canDeactivate: [unsavedChangesGuard],
  },
];
```

---

## 4. Redirect Logic

### แบบ Simple Redirect

```typescript
const routes: Routes = [
  // Redirect / ไป /home
  { path: '', redirectTo: '/home', pathMatch: 'full' },

  // Redirect ทุก path ที่ไม่ตรงไป /not-found
  { path: '**', redirectTo: '/not-found' },
];
```

### Redirect หลัง Login

```typescript
// core/services/auth.service.ts
import { Injectable } from '@angular/core';
import { Router } from '@angular/router';
import { HttpClient } from '@angular/common/http';
import { BehaviorSubject, Observable, tap } from 'rxjs';

export interface User {
  id: string;
  email: string;
  name: string;
  roles: string[];
}

export interface LoginResponse {
  user: User;
  accessToken: string;
  refreshToken: string;
}

@Injectable({ providedIn: 'root' })
export class AuthService {
  private currentUserSubject = new BehaviorSubject<User | null>(null);

  currentUser$ = this.currentUserSubject.asObservable();
  isLoggedIn$ = this.currentUser$.pipe(map(user => !!user));

  constructor(
    private http: HttpClient,
    private router: Router
  ) {
    // Restore session จาก localStorage
    const saved = localStorage.getItem('currentUser');
    if (saved) {
      this.currentUserSubject.next(JSON.parse(saved));
    }
  }

  login(email: string, password: string): Observable<LoginResponse> {
    return this.http.post<LoginResponse>('/api/auth/login', { email, password }).pipe(
      tap(response => {
        localStorage.setItem('accessToken', response.accessToken);
        localStorage.setItem('refreshToken', response.refreshToken);
        localStorage.setItem('currentUser', JSON.stringify(response.user));
        this.currentUserSubject.next(response.user);
      })
    );
  }

  logout(): void {
    localStorage.removeItem('accessToken');
    localStorage.removeItem('refreshToken');
    localStorage.removeItem('currentUser');
    this.currentUserSubject.next(null);
    this.router.navigate(['/auth/login']);
  }

  isAdmin(): boolean {
    const user = this.currentUserSubject.getValue();
    return user?.roles.includes('admin') ?? false;
  }

  hasRole(role: string): boolean {
    const user = this.currentUserSubject.getValue();
    return user?.roles.includes(role) ?? false;
  }

  isLoggedIn(): boolean {
    return !!this.currentUserSubject.getValue();
  }
}
```

```typescript
// features/auth/login/login.component.ts
import { Component } from '@angular/core';
import { FormBuilder, Validators } from '@angular/forms';
import { Router, ActivatedRoute } from '@angular/router';
import { AuthService } from '../../../core/services/auth.service';

@Component({
  selector: 'app-login',
  template: `
    <div class="login-page">
      <div class="login-card">
        <h2>เข้าสู่ระบบ</h2>

        <form [formGroup]="loginForm" (ngSubmit)="onSubmit()">
          <div class="form-group">
            <label>อีเมล</label>
            <input
              type="email"
              formControlName="email"
              class="form-control"
              placeholder="email@example.com"
            >
          </div>
          <div class="form-group">
            <label>รหัสผ่าน</label>
            <input
              type="password"
              formControlName="password"
              class="form-control"
            >
          </div>

          <div class="error-message" *ngIf="errorMessage">
            {{ errorMessage }}
          </div>

          <button
            type="submit"
            class="btn-login"
            [disabled]="loginForm.invalid || loading"
          >
            {{ loading ? 'กำลังเข้าสู่ระบบ...' : 'เข้าสู่ระบบ' }}
          </button>
        </form>
      </div>
    </div>
  `,
})
export class LoginComponent {
  loginForm = this.fb.group({
    email: ['', [Validators.required, Validators.email]],
    password: ['', [Validators.required, Validators.minLength(6)]],
  });

  loading = false;
  errorMessage = '';

  constructor(
    private fb: FormBuilder,
    private authService: AuthService,
    private router: Router,
    private route: ActivatedRoute
  ) {}

  onSubmit(): void {
    if (this.loginForm.invalid) return;

    this.loading = true;
    this.errorMessage = '';

    const { email, password } = this.loginForm.value;

    this.authService.login(email!, password!).subscribe({
      next: () => {
        // Redirect ไปยัง returnUrl ที่บันทึกไว้ หรือ dashboard
        const returnUrl = this.route.snapshot.queryParams['returnUrl'] || '/dashboard';
        this.router.navigateByUrl(returnUrl);
      },
      error: (err) => {
        this.errorMessage = err.error?.message || 'เข้าสู่ระบบไม่สำเร็จ';
        this.loading = false;
      },
    });
  }
}
```

---

## 5. Workshop: Protected Routes

### โครงสร้าง Protected Routes

```typescript
// app.routes.ts (Standalone App)
import { Routes } from '@angular/router';
import { authGuard } from './core/guards/auth.guard';
import { roleGuard } from './core/guards/role.guard';
import { unsavedChangesGuard } from './core/guards/unsaved-changes.guard';
import { loggedOutGuard } from './core/guards/logged-out.guard';

export const routes: Routes = [
  // Public Routes (ไม่ต้อง Login)
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
    path: 'auth',
    canActivate: [loggedOutGuard], // ถ้า Login แล้ว redirect ไป dashboard
    children: [
      {
        path: 'login',
        loadComponent: () =>
          import('./features/auth/login/login.component').then(c => c.LoginComponent),
      },
      {
        path: 'register',
        loadComponent: () =>
          import('./features/auth/register/register.component').then(c => c.RegisterComponent),
      },
      {
        path: 'forgot-password',
        loadComponent: () =>
          import('./features/auth/forgot-password/forgot-password.component')
            .then(c => c.ForgotPasswordComponent),
      },
    ],
  },

  // Protected Routes (ต้อง Login)
  {
    path: 'dashboard',
    canActivate: [authGuard],
    loadComponent: () =>
      import('./features/dashboard/dashboard.component').then(c => c.DashboardComponent),
  },
  {
    path: 'products',
    loadChildren: () =>
      import('./features/products/products.routes').then(r => r.productRoutes),
  },
  {
    path: 'orders',
    canActivate: [authGuard],
    loadChildren: () =>
      import('./features/orders/orders.routes').then(r => r.orderRoutes),
  },
  {
    path: 'profile',
    canActivate: [authGuard],
    children: [
      {
        path: '',
        loadComponent: () =>
          import('./features/profile/profile.component').then(c => c.ProfileComponent),
      },
      {
        path: 'edit',
        loadComponent: () =>
          import('./features/profile/profile-edit/profile-edit.component')
            .then(c => c.ProfileEditComponent),
        canDeactivate: [unsavedChangesGuard],
      },
    ],
  },

  // Admin Routes (ต้องเป็น Admin)
  {
    path: 'admin',
    canActivate: [authGuard, roleGuard('admin')],
    loadChildren: () =>
      import('./features/admin/admin.routes').then(r => r.adminRoutes),
  },

  // Manager Routes
  {
    path: 'manager',
    canActivate: [authGuard, roleGuard('manager')],
    loadChildren: () =>
      import('./features/manager/manager.routes').then(r => r.managerRoutes),
  },

  // 404
  {
    path: 'not-found',
    loadComponent: () =>
      import('./features/not-found/not-found.component').then(c => c.NotFoundComponent),
  },
  {
    path: 'forbidden',
    loadComponent: () =>
      import('./features/forbidden/forbidden.component').then(c => c.ForbiddenComponent),
  },
  {
    path: '**',
    redirectTo: 'not-found',
  },
];
```

### LoggedOut Guard (ป้องกันไม่ให้ User ที่ Login แล้วเข้า Login Page)

```typescript
// core/guards/logged-out.guard.ts
import { inject } from '@angular/core';
import { CanActivateFn, Router } from '@angular/router';
import { map } from 'rxjs/operators';
import { AuthService } from '../services/auth.service';

export const loggedOutGuard: CanActivateFn = () => {
  const authService = inject(AuthService);
  const router = inject(Router);

  return authService.isLoggedIn$.pipe(
    map(isLoggedIn => {
      if (!isLoggedIn) return true;
      // ถ้า Login อยู่แล้ว redirect ไป dashboard
      return router.createUrlTree(['/dashboard']);
    })
  );
};
```

### Forbidden Component

```typescript
// features/forbidden/forbidden.component.ts
import { Component } from '@angular/core';
import { CommonModule } from '@angular/common';
import { RouterModule } from '@angular/router';
import { AuthService } from '../../core/services/auth.service';

@Component({
  selector: 'app-forbidden',
  standalone: true,
  imports: [CommonModule, RouterModule],
  template: `
    <div class="forbidden-page">
      <div class="forbidden-content">
        <div class="error-code">403</div>
        <h1>ไม่มีสิทธิ์เข้าถึง</h1>
        <p>คุณไม่มีสิทธิ์เข้าถึงหน้านี้</p>
        <div class="actions">
          <a routerLink="/home" class="btn-home">กลับหน้าหลัก</a>
          <button (click)="auth.logout()" class="btn-logout">
            ออกจากระบบ และเข้าสู่ระบบใหม่
          </button>
        </div>
      </div>
    </div>
  `,
  styles: [`
    .forbidden-page {
      display: flex;
      justify-content: center;
      align-items: center;
      min-height: 100vh;
      background: #f8f9fa;
    }
    .forbidden-content {
      text-align: center;
      padding: 2rem;
    }
    .error-code {
      font-size: 8rem;
      font-weight: bold;
      color: #dc3545;
      line-height: 1;
    }
    .actions {
      display: flex;
      gap: 1rem;
      justify-content: center;
      margin-top: 2rem;
    }
  `],
})
export class ForbiddenComponent {
  constructor(public auth: AuthService) {}
}
```

### Token Refresh Guard

```typescript
// core/guards/token-refresh.guard.ts
import { inject } from '@angular/core';
import { CanActivateFn, Router } from '@angular/router';
import { catchError, map, of, switchMap } from 'rxjs';
import { AuthService } from '../services/auth.service';
import { TokenService } from '../services/token.service';

export const tokenRefreshGuard: CanActivateFn = (route, state) => {
  const tokenService = inject(TokenService);
  const authService = inject(AuthService);
  const router = inject(Router);

  // ถ้า token หมดอายุให้ refresh ก่อน
  if (tokenService.isAccessTokenExpired() && tokenService.hasRefreshToken()) {
    return authService.refreshToken().pipe(
      map(() => true),
      catchError(() => {
        // Refresh ไม่สำเร็จ - redirect to login
        return of(
          router.createUrlTree(['/auth/login'], {
            queryParams: { returnUrl: state.url },
          })
        );
      })
    );
  }

  return of(true);
};
```

### Guard Composition

```typescript
// รวม Guards หลายตัวด้วย combineLatest
import { inject } from '@angular/core';
import { CanActivateFn, Router } from '@angular/router';
import { combineLatest, map } from 'rxjs';
import { AuthService } from '../services/auth.service';
import { FeatureFlagsService } from '../services/feature-flags.service';

export const featureGuard = (featureFlag: string): CanActivateFn => {
  return () => {
    const authService = inject(AuthService);
    const featureFlags = inject(FeatureFlagsService);
    const router = inject(Router);

    return combineLatest([
      authService.isLoggedIn$,
      featureFlags.isEnabled$(featureFlag),
    ]).pipe(
      map(([isLoggedIn, isEnabled]) => {
        if (!isLoggedIn) {
          return router.createUrlTree(['/auth/login']);
        }
        if (!isEnabled) {
          return router.createUrlTree(['/not-found']);
        }
        return true;
      })
    );
  };
};
```

---

## สรุป

| Guard | Interface | ใช้เมื่อ |
|-------|-----------|---------|
| **canActivate** | `CanActivateFn` | ตรวจสอบก่อนเข้า Route |
| **canActivateChild** | `CanActivateChildFn` | ตรวจสอบก่อนเข้า Child Route |
| **canDeactivate** | `CanDeactivateFn` | ตรวจสอบก่อนออกจาก Route |
| **canMatch** | `CanMatchFn` | ตรวจสอบว่า Route จะ match |
| **resolve** | `ResolveFn` | โหลดข้อมูลก่อนเข้า Route |

### Best Practices

1. **ใช้ Functional Guards** แทน Class-based ใน Angular 14+
2. **แยก Guard ตามหน้าที่** อย่ารวมทุกอย่างในก้อนเดียว
3. **บันทึก returnUrl** เพื่อ redirect กลับหลัง login
4. **ใช้ canDeactivate** สำหรับฟอร์มที่มีการแก้ไข
5. **ใช้ canMatch** เมื่อต้องการแสดง Component ต่างกันตาม Role
