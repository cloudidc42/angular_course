# Part 41: Role-Based Access Control (RBAC) ใน Angular

## บทนำ

Role-Based Access Control (RBAC) เป็นการควบคุมการเข้าถึงตาม Role ของผู้ใช้ ในบทนี้เราจะสร้างระบบ RBAC ที่สมบูรณ์พร้อม Role-based routing, Permission directives และ canActivate guards

---

## 1. Permission Model

```typescript
// app/models/permission.model.ts
export type Role = 'admin' | 'manager' | 'editor' | 'viewer' | 'user';

export type Permission =
  | 'users:read' | 'users:write' | 'users:delete'
  | 'products:read' | 'products:write' | 'products:delete'
  | 'orders:read' | 'orders:write' | 'orders:delete'
  | 'reports:read' | 'reports:export'
  | 'settings:read' | 'settings:write';

// กำหนด permissions สำหรับแต่ละ role
export const ROLE_PERMISSIONS: Record<Role, Permission[]> = {
  admin: [
    'users:read', 'users:write', 'users:delete',
    'products:read', 'products:write', 'products:delete',
    'orders:read', 'orders:write', 'orders:delete',
    'reports:read', 'reports:export',
    'settings:read', 'settings:write'
  ],
  manager: [
    'users:read',
    'products:read', 'products:write',
    'orders:read', 'orders:write',
    'reports:read', 'reports:export',
    'settings:read'
  ],
  editor: [
    'products:read', 'products:write',
    'orders:read',
    'reports:read'
  ],
  viewer: [
    'products:read',
    'orders:read',
    'reports:read'
  ],
  user: [
    'products:read',
    'orders:read'
  ]
};
```

---

## 2. Permission Service

```typescript
// app/services/permission.service.ts
import { Injectable, computed } from '@angular/core';
import { AuthService } from './auth.service';
import { Role, Permission, ROLE_PERMISSIONS } from '../models/permission.model';

@Injectable({ providedIn: 'root' })
export class PermissionService {

  constructor(private authService: AuthService) {}

  // ดึง permissions ทั้งหมดของผู้ใช้ปัจจุบัน
  getUserPermissions(): Permission[] {
    const user = this.authService.currentUser();
    if (!user) return [];

    const allPermissions = new Set<Permission>();
    user.roles.forEach((role: string) => {
      const rolePerms = ROLE_PERMISSIONS[role as Role] || [];
      rolePerms.forEach(p => allPermissions.add(p));
    });

    return Array.from(allPermissions);
  }

  // ตรวจสอบ single permission
  can(permission: Permission): boolean {
    return this.getUserPermissions().includes(permission);
  }

  // ตรวจสอบ permissions หลายตัว (ต้องมีทั้งหมด)
  canAll(permissions: Permission[]): boolean {
    const userPerms = this.getUserPermissions();
    return permissions.every(p => userPerms.includes(p));
  }

  // ตรวจสอบ permissions หลายตัว (มีอย่างน้อยหนึ่งตัว)
  canAny(permissions: Permission[]): boolean {
    const userPerms = this.getUserPermissions();
    return permissions.some(p => userPerms.includes(p));
  }

  // ตรวจสอบ role
  hasRole(role: Role): boolean {
    return this.authService.hasRole(role);
  }

  hasAnyRole(roles: Role[]): boolean {
    return roles.some(role => this.hasRole(role));
  }

  isAdmin(): boolean {
    return this.hasRole('admin');
  }
}
```

---

## 3. Role Guard

```typescript
// app/guards/role.guard.ts
import { inject } from '@angular/core';
import {
  CanActivateFn,
  ActivatedRouteSnapshot,
  RouterStateSnapshot,
  Router
} from '@angular/router';
import { AuthService } from '../services/auth.service';
import { PermissionService } from '../services/permission.service';
import { Role, Permission } from '../models/permission.model';

// Guard สำหรับตรวจสอบ roles
export const roleGuard: CanActivateFn = (
  route: ActivatedRouteSnapshot,
  state: RouterStateSnapshot
) => {
  const authService = inject(AuthService);
  const permissionService = inject(PermissionService);
  const router = inject(Router);

  if (!authService.isAuthenticated()) {
    return router.createUrlTree(['/login'], {
      queryParams: { returnUrl: state.url }
    });
  }

  const requiredRoles = route.data['roles'] as Role[] | undefined;
  const requiredPermissions = route.data['permissions'] as Permission[] | undefined;
  const requireAll = route.data['requireAll'] as boolean ?? true;

  // ตรวจสอบ roles
  if (requiredRoles?.length) {
    const hasRole = requireAll
      ? requiredRoles.every(role => permissionService.hasRole(role))
      : permissionService.hasAnyRole(requiredRoles);

    if (!hasRole) {
      return router.createUrlTree(['/forbidden']);
    }
  }

  // ตรวจสอบ permissions
  if (requiredPermissions?.length) {
    const hasPermission = requireAll
      ? permissionService.canAll(requiredPermissions)
      : permissionService.canAny(requiredPermissions);

    if (!hasPermission) {
      return router.createUrlTree(['/forbidden']);
    }
  }

  return true;
};
```

```typescript
// app/app-routing.module.ts
import { Routes } from '@angular/router';
import { authGuard } from './guards/auth.guard';
import { roleGuard } from './guards/role.guard';

export const routes: Routes = [
  // Admin เท่านั้น
  {
    path: 'admin',
    loadChildren: () => import('./features/admin/admin.module').then(m => m.AdminModule),
    canActivate: [authGuard, roleGuard],
    data: { roles: ['admin'] }
  },

  // Manager หรือ Admin
  {
    path: 'reports',
    loadComponent: () => import('./features/reports/reports.component').then(m => m.ReportsComponent),
    canActivate: [authGuard, roleGuard],
    data: { roles: ['admin', 'manager'], requireAll: false }
  },

  // ต้องมี permission
  {
    path: 'products',
    loadChildren: () => import('./features/products/products.module').then(m => m.ProductsModule),
    canActivate: [authGuard, roleGuard],
    data: { permissions: ['products:read'] }
  },

  // ต้องมีหลาย permissions
  {
    path: 'products/edit/:id',
    loadComponent: () => import('./features/products/edit.component').then(m => m.EditComponent),
    canActivate: [authGuard, roleGuard],
    data: {
      permissions: ['products:read', 'products:write'],
      requireAll: true
    }
  }
];
```

---

## 4. Permission Directives

```typescript
// app/directives/has-permission.directive.ts
import {
  Directive,
  Input,
  TemplateRef,
  ViewContainerRef,
  OnInit,
  OnDestroy,
  effect
} from '@angular/core';
import { PermissionService } from '../services/permission.service';
import { Permission } from '../models/permission.model';

@Directive({
  selector: '[appHasPermission]',
  standalone: true
})
export class HasPermissionDirective implements OnInit {
  @Input('appHasPermission') permission!: Permission | Permission[];
  @Input('appHasPermissionElse') elseTemplate?: TemplateRef<any>;

  private hasView = false;

  constructor(
    private templateRef: TemplateRef<any>,
    private viewContainer: ViewContainerRef,
    private permissionService: PermissionService
  ) {}

  ngOnInit() {
    this.updateView();
  }

  private updateView() {
    const hasPermission = Array.isArray(this.permission)
      ? this.permissionService.canAny(this.permission)
      : this.permissionService.can(this.permission);

    if (hasPermission && !this.hasView) {
      this.viewContainer.createEmbeddedView(this.templateRef);
      this.hasView = true;
    } else if (!hasPermission && this.hasView) {
      this.viewContainer.clear();
      this.hasView = false;

      if (this.elseTemplate) {
        this.viewContainer.createEmbeddedView(this.elseTemplate);
      }
    }
  }
}
```

```typescript
// app/directives/has-role.directive.ts
import {
  Directive,
  Input,
  TemplateRef,
  ViewContainerRef,
  OnInit
} from '@angular/core';
import { PermissionService } from '../services/permission.service';
import { Role } from '../models/permission.model';

@Directive({
  selector: '[appHasRole]',
  standalone: true
})
export class HasRoleDirective implements OnInit {
  @Input('appHasRole') roles!: Role | Role[];
  @Input('appHasRoleElse') elseTemplate?: TemplateRef<any>;

  constructor(
    private templateRef: TemplateRef<any>,
    private viewContainer: ViewContainerRef,
    private permissionService: PermissionService
  ) {}

  ngOnInit() {
    const roleArray = Array.isArray(this.roles) ? this.roles : [this.roles];
    const hasRole = this.permissionService.hasAnyRole(roleArray);

    if (hasRole) {
      this.viewContainer.createEmbeddedView(this.templateRef);
    } else if (this.elseTemplate) {
      this.viewContainer.createEmbeddedView(this.elseTemplate);
    }
  }
}
```

---

## 5. การใช้งาน Directives ใน Template

```typescript
// app/components/product-list/product-list.component.ts
import { Component } from '@angular/core';
import { HasPermissionDirective } from '../../directives/has-permission.directive';
import { HasRoleDirective } from '../../directives/has-role.directive';

@Component({
  selector: 'app-product-list',
  standalone: true,
  imports: [HasPermissionDirective, HasRoleDirective],
  template: `
    <div class="product-management">
      <h2>จัดการสินค้า</h2>

      <!-- แสดงเฉพาะผู้ที่มี permission เพิ่มสินค้า -->
      <ng-container *appHasPermission="'products:write'">
        <button class="btn btn-primary" (click)="addProduct()">
          + เพิ่มสินค้าใหม่
        </button>
      </ng-container>

      <!-- แสดงปุ่มตามสิทธิ์ -->
      <table>
        <thead>
          <tr>
            <th>ชื่อสินค้า</th>
            <th>ราคา</th>
            <th>สถานะ</th>
            <th>จัดการ</th>
          </tr>
        </thead>
        <tbody>
          <tr *ngFor="let product of products">
            <td>{{ product.name }}</td>
            <td>{{ product.price | currency:'THB' }}</td>
            <td>{{ product.status }}</td>
            <td>
              <!-- แก้ไข: ต้องมี products:write -->
              <button
                *appHasPermission="'products:write'"
                (click)="editProduct(product)"
              >แก้ไข</button>

              <!-- ลบ: ต้องมี products:delete -->
              <button
                *appHasPermission="'products:delete'"
                (click)="deleteProduct(product)"
                class="btn-danger"
              >ลบ</button>
            </td>
          </tr>
        </tbody>
      </table>

      <!-- แสดงสถิติเฉพาะ admin หรือ manager -->
      <div *appHasRole="['admin', 'manager']">
        <h3>สถิติสินค้า (เฉพาะ Admin/Manager)</h3>
        <p>สินค้าทั้งหมด: {{ products.length }}</p>
      </div>

      <!-- Else template -->
      <ng-template #noPermission>
        <p class="text-muted">คุณไม่มีสิทธิ์ดำเนินการนี้</p>
      </ng-template>

      <!-- แสดง menu เฉพาะ admin -->
      <div *appHasRole="'admin'; else notAdmin">
        <h3>Admin Menu</h3>
        <ul>
          <li><a routerLink="/admin/users">จัดการผู้ใช้</a></li>
          <li><a routerLink="/admin/settings">การตั้งค่า</a></li>
        </ul>
      </div>
      <ng-template #notAdmin>
        <p>คุณไม่ใช่ Admin</p>
      </ng-template>
    </div>
  `
})
export class ProductListComponent {
  products = [
    { id: 1, name: 'สินค้า A', price: 100, status: 'active' },
    { id: 2, name: 'สินค้า B', price: 200, status: 'inactive' }
  ];

  addProduct() { console.log('Add product'); }
  editProduct(p: any) { console.log('Edit:', p); }
  deleteProduct(p: any) { console.log('Delete:', p); }
}
```

---

## 6. Permission Pipe

```typescript
// app/pipes/has-permission.pipe.ts
import { Pipe, PipeTransform } from '@angular/core';
import { PermissionService } from '../services/permission.service';
import { Permission } from '../models/permission.model';

@Pipe({
  name: 'hasPermission',
  standalone: true,
  pure: false // impure เพราะ permission อาจเปลี่ยนได้
})
export class HasPermissionPipe implements PipeTransform {
  constructor(private permissionService: PermissionService) {}

  transform(permission: Permission | Permission[]): boolean {
    if (Array.isArray(permission)) {
      return this.permissionService.canAny(permission);
    }
    return this.permissionService.can(permission);
  }
}
```

```html
<!-- การใช้งาน pipe -->
<button [disabled]="!('products:write' | hasPermission)">
  แก้ไข
</button>

<div [class.hidden]="!(['products:read', 'reports:read'] | hasPermission)">
  ส่วนที่ซ่อน
</div>
```

---

## 7. Forbidden Page

```typescript
// app/components/forbidden/forbidden.component.ts
import { Component } from '@angular/core';
import { Router } from '@angular/router';
import { AuthService } from '../../services/auth.service';

@Component({
  selector: 'app-forbidden',
  template: `
    <div class="forbidden-page">
      <div class="forbidden-content">
        <h1>403</h1>
        <h2>ไม่มีสิทธิ์เข้าถึง</h2>
        <p>คุณไม่มีสิทธิ์ในการเข้าถึงหน้านี้</p>
        <p *ngIf="currentUser">
          บัญชี: {{ currentUser.email }}
          ({{ currentUser.roles.join(', ') }})
        </p>
        <div class="actions">
          <button (click)="goBack()">กลับ</button>
          <button (click)="goHome()">หน้าหลัก</button>
          <button *ngIf="!currentUser" (click)="login()">เข้าสู่ระบบ</button>
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
      padding: 40px;
    }
    h1 { font-size: 120px; color: #dc3545; margin: 0; }
    .actions { display: flex; gap: 12px; justify-content: center; margin-top: 24px; }
    button { padding: 10px 20px; border: none; border-radius: 4px; cursor: pointer; }
  `]
})
export class ForbiddenComponent {
  get currentUser() {
    return this.authService.currentUser();
  }

  constructor(
    private router: Router,
    private authService: AuthService
  ) {}

  goBack() { window.history.back(); }
  goHome() { this.router.navigate(['/']); }
  login() { this.router.navigate(['/login']); }
}
```

---

## สรุป

| องค์ประกอบ | หน้าที่ |
|----------|--------|
| Role Model | กำหนด roles และ permissions |
| PermissionService | ตรวจสอบสิทธิ์ของผู้ใช้ |
| roleGuard | ป้องกัน routes ตาม role/permission |
| HasPermissionDirective | ซ่อน/แสดง element ตาม permission |
| HasRoleDirective | ซ่อน/แสดง element ตาม role |
| HasPermissionPipe | ใช้ใน property binding |

RBAC ที่ดีควรตรวจสอบทั้งฝั่ง Frontend (UI) และ Backend (API) เสมอ เพราะ frontend เป็นแค่ UX enhancement ไม่ใช่การรักษาความปลอดภัยจริง
