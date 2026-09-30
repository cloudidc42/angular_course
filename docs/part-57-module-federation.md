# Part 57: Webpack Module Federation ใน Angular

## บทนำ

Module Federation คือฟีเจอร์ของ Webpack 5 ที่ช่วยให้แอปพลิเคชันหนึ่งสามารถโหลด code จากแอปพลิเคชันอื่นได้ตอน runtime นี่คือรากฐานของ Micro-frontend สมัยใหม่

## 1. การติดตั้ง

```bash
# ใช้ @angular-architects/module-federation
npm install @angular-architects/module-federation

# สร้าง host app
ng add @angular-architects/module-federation --project shell --type host --port 4200

# สร้าง remote app
ng add @angular-architects/module-federation --project products-mfe --type remote --port 4201
```

## 2. ตั้งค่า Host Application

```javascript
// shell/webpack.config.js
const { shareAll, withModuleFederationPlugin } = require('@angular-architects/module-federation/webpack');

module.exports = withModuleFederationPlugin({
  remotes: {
    // ชื่อ: URL ของ remote entry
    'productsMfe': 'http://localhost:4201/remoteEntry.js',
    'ordersMfe': 'http://localhost:4202/remoteEntry.js',
    'dashboardMfe': 'http://localhost:4203/remoteEntry.js',
  },
  shared: {
    ...shareAll({
      singleton: true,
      strictVersion: true,
      requiredVersion: 'auto'
    })
  }
});
```

```typescript
// shell/src/app/app.routes.ts
import { Routes } from '@angular/router';
import { loadRemoteModule } from '@angular-architects/module-federation';

export const routes: Routes = [
  {
    path: '',
    redirectTo: 'dashboard',
    pathMatch: 'full'
  },
  {
    path: 'dashboard',
    loadChildren: () =>
      loadRemoteModule({
        type: 'module',
        remoteEntry: 'http://localhost:4203/remoteEntry.js',
        exposedModule: './DashboardModule'
      }).then(m => m.DashboardModule)
  },
  {
    path: 'products',
    loadChildren: () =>
      loadRemoteModule({
        type: 'module',
        remoteEntry: 'http://localhost:4201/remoteEntry.js',
        exposedModule: './ProductsModule'
      }).then(m => m.ProductsModule)
  },
  {
    path: 'orders',
    loadChildren: () =>
      loadRemoteModule({
        type: 'module',
        remoteEntry: 'http://localhost:4202/remoteEntry.js',
        exposedModule: './OrdersModule'
      }).then(m => m.OrdersModule)
  }
];
```

## 3. ตั้งค่า Remote Application

```javascript
// products-mfe/webpack.config.js
const { shareAll, withModuleFederationPlugin } = require('@angular-architects/module-federation/webpack');

module.exports = withModuleFederationPlugin({
  name: 'productsMfe',
  exposes: {
    // ชื่อที่ expose: path ของ file
    './ProductsModule': './src/app/products/products.module.ts',
    './ProductDetailComponent': './src/app/products/product-detail/product-detail.component.ts',
  },
  shared: {
    ...shareAll({
      singleton: true,
      strictVersion: true,
      requiredVersion: 'auto'
    })
  }
});
```

```typescript
// products-mfe/src/app/products/products.module.ts
import { NgModule } from '@angular/core';
import { CommonModule } from '@angular/common';
import { RouterModule, Routes } from '@angular/router';
import { ProductsListComponent } from './products-list/products-list.component';
import { ProductDetailComponent } from './product-detail/product-detail.component';

const routes: Routes = [
  { path: '', component: ProductsListComponent },
  { path: ':id', component: ProductDetailComponent }
];

@NgModule({
  imports: [
    CommonModule,
    RouterModule.forChild(routes)
  ],
  declarations: [ProductsListComponent, ProductDetailComponent]
})
export class ProductsModule {}
```

## 4. Shell Component

```typescript
// shell/src/app/shell/shell.component.ts
import { Component } from '@angular/core';
import { RouterOutlet, RouterLink, RouterLinkActive } from '@angular/router';
import { CommonModule } from '@angular/common';

interface NavItem {
  path: string;
  label: string;
  icon: string;
}

@Component({
  selector: 'app-shell',
  standalone: true,
  imports: [RouterOutlet, RouterLink, RouterLinkActive, CommonModule],
  template: `
    <div class="shell-layout">
      <header class="shell-header">
        <div class="logo">
          <h1>🏢 Enterprise App</h1>
        </div>
        <nav class="main-nav">
          <a
            *ngFor="let item of navItems"
            [routerLink]="item.path"
            routerLinkActive="active"
            [routerLinkActiveOptions]="{ exact: item.path === '/' }"
            class="nav-link"
          >
            {{ item.icon }} {{ item.label }}
          </a>
        </nav>
        <div class="header-actions">
          <span>{{ userName }}</span>
          <button (click)="logout()">ออกจากระบบ</button>
        </div>
      </header>

      <div class="shell-content">
        <aside class="shell-sidebar">
          <nav>
            <a
              *ngFor="let item of navItems"
              [routerLink]="item.path"
              routerLinkActive="active"
              class="sidebar-link"
            >
              <span class="icon">{{ item.icon }}</span>
              <span class="label">{{ item.label }}</span>
            </a>
          </nav>
        </aside>

        <main class="shell-main">
          <router-outlet></router-outlet>
        </main>
      </div>
    </div>
  `,
  styles: [`
    .shell-layout {
      display: flex;
      flex-direction: column;
      height: 100vh;
    }
    .shell-header {
      display: flex;
      align-items: center;
      justify-content: space-between;
      padding: 0 1rem;
      height: 60px;
      background: #1a1a2e;
      color: white;
    }
    .shell-content {
      display: flex;
      flex: 1;
      overflow: hidden;
    }
    .shell-sidebar {
      width: 220px;
      background: #16213e;
      padding: 1rem 0;
    }
    .sidebar-link {
      display: flex;
      align-items: center;
      gap: 0.75rem;
      padding: 0.75rem 1.5rem;
      color: #adb5bd;
      text-decoration: none;
      transition: all 0.2s;
    }
    .sidebar-link:hover, .sidebar-link.active {
      background: rgba(255,255,255,0.1);
      color: white;
    }
    .shell-main {
      flex: 1;
      overflow: auto;
      padding: 1.5rem;
      background: #f8f9fa;
    }
  `]
})
export class ShellComponent {
  userName = 'สมชาย ใจดี';
  
  navItems: NavItem[] = [
    { path: '/dashboard', label: 'Dashboard', icon: '📊' },
    { path: '/products', label: 'สินค้า', icon: '📦' },
    { path: '/orders', label: 'ออเดอร์', icon: '🛒' },
    { path: '/customers', label: 'ลูกค้า', icon: '👥' },
    { path: '/reports', label: 'รายงาน', icon: '📈' }
  ];

  logout(): void {
    console.log('Logging out...');
  }
}
```

## 5. Dynamic Remote Loading

```typescript
// dynamic-remote-loader.service.ts
import { Injectable } from '@angular/core';
import { loadRemoteModule } from '@angular-architects/module-federation';

interface RemoteDefinition {
  remoteEntry: string;
  remoteName: string;
  exposedModule: string;
  displayName: string;
}

@Injectable({ providedIn: 'root' })
export class DynamicRemoteLoaderService {
  // โหลด config จาก API (Dynamic Module Federation)
  async loadRemoteDefinitions(configUrl: string): Promise<RemoteDefinition[]> {
    const response = await fetch(configUrl);
    return response.json();
  }

  async loadComponent(def: RemoteDefinition): Promise<any> {
    const module = await loadRemoteModule({
      type: 'module',
      remoteEntry: def.remoteEntry,
      exposedModule: def.exposedModule
    });
    
    return module[Object.keys(module)[0]];
  }
}
```

### config.json (เก็บบน server)

```json
[
  {
    "remoteEntry": "http://localhost:4201/remoteEntry.js",
    "remoteName": "productsMfe",
    "exposedModule": "./ProductsModule",
    "displayName": "สินค้า"
  },
  {
    "remoteEntry": "http://localhost:4202/remoteEntry.js",
    "remoteName": "ordersMfe",
    "exposedModule": "./OrdersModule",
    "displayName": "ออเดอร์"
  }
]
```

## 6. Error Handling สำหรับ Remote Loading

```typescript
// safe-remote-loader.component.ts
import { Component, Input, OnInit, ViewContainerRef } from '@angular/core';
import { CommonModule } from '@angular/common';
import { loadRemoteModule } from '@angular-architects/module-federation';
import { Router } from '@angular/router';

@Component({
  selector: 'app-safe-remote-loader',
  standalone: true,
  imports: [CommonModule],
  template: `
    <div *ngIf="loading" class="loading-state">
      <div class="spinner"></div>
      <p>กำลังโหลดโมดูล...</p>
    </div>
    
    <div *ngIf="error" class="error-state">
      <h3>⚠️ ไม่สามารถโหลดโมดูลได้</h3>
      <p>{{ error }}</p>
      <div class="error-actions">
        <button (click)="retry()">ลองใหม่</button>
        <button (click)="goHome()">กลับหน้าแรก</button>
      </div>
    </div>
  `,
  styles: [`
    .loading-state, .error-state {
      display: flex;
      flex-direction: column;
      align-items: center;
      justify-content: center;
      min-height: 300px;
      gap: 1rem;
    }
    .spinner {
      width: 40px;
      height: 40px;
      border: 4px solid #f0f0f0;
      border-top-color: #007bff;
      border-radius: 50%;
      animation: spin 0.8s linear infinite;
    }
    @keyframes spin { to { transform: rotate(360deg); } }
    .error-actions { display: flex; gap: 1rem; }
    button {
      padding: 0.5rem 1rem;
      border: none;
      border-radius: 4px;
      cursor: pointer;
    }
    button:first-child { background: #007bff; color: white; }
    button:last-child { background: #6c757d; color: white; }
  `]
})
export class SafeRemoteLoaderComponent implements OnInit {
  @Input() remoteEntry!: string;
  @Input() exposedModule!: string;
  @Input() componentName!: string;

  loading = true;
  error: string | null = null;
  private retryCount = 0;
  private readonly MAX_RETRIES = 3;

  constructor(
    private viewContainerRef: ViewContainerRef,
    private router: Router
  ) {}

  ngOnInit(): void {
    this.loadModule();
  }

  async retry(): Promise<void> {
    if (this.retryCount < this.MAX_RETRIES) {
      this.retryCount++;
      this.error = null;
      this.loading = true;
      await this.loadModule();
    }
  }

  goHome(): void {
    this.router.navigate(['/']);
  }

  private async loadModule(): Promise<void> {
    try {
      const module = await loadRemoteModule({
        type: 'module',
        remoteEntry: this.remoteEntry,
        exposedModule: this.exposedModule
      });

      const component = module[this.componentName];
      
      if (!component) {
        throw new Error(`Component "${this.componentName}" not found in module`);
      }

      this.viewContainerRef.createComponent(component);
      this.loading = false;
    } catch (err: any) {
      console.error('Failed to load remote module:', err);
      this.error = err.message || 'เกิดข้อผิดพลาดที่ไม่ทราบสาเหตุ';
      this.loading = false;
    }
  }
}
```

## 7. Shared State ระหว่าง Host และ Remote

```typescript
// shared/state/app-state.service.ts
// ต้อง share ใน webpack config
import { Injectable } from '@angular/core';
import { BehaviorSubject } from 'rxjs';

export interface AppState {
  currentUser: {
    id: string;
    name: string;
    email: string;
    role: string;
  } | null;
  cart: { productId: string; quantity: number }[];
  notifications: number;
}

@Injectable({ providedIn: 'root' })
export class AppStateService {
  private state = new BehaviorSubject<AppState>({
    currentUser: null,
    cart: [],
    notifications: 0
  });

  readonly state$ = this.state.asObservable();

  get currentState(): AppState {
    return this.state.value;
  }

  setUser(user: AppState['currentUser']): void {
    this.updateState({ currentUser: user });
  }

  addToCart(item: { productId: string; quantity: number }): void {
    const cart = [...this.currentState.cart];
    const existingIndex = cart.findIndex(c => c.productId === item.productId);
    
    if (existingIndex >= 0) {
      cart[existingIndex] = {
        ...cart[existingIndex],
        quantity: cart[existingIndex].quantity + item.quantity
      };
    } else {
      cart.push(item);
    }
    
    this.updateState({ cart });
  }

  private updateState(partial: Partial<AppState>): void {
    this.state.next({ ...this.currentState, ...partial });
  }
}
```

```javascript
// webpack.config.js (ต้อง share service นี้)
const { shareAll, withModuleFederationPlugin } = require('@angular-architects/module-federation/webpack');

module.exports = withModuleFederationPlugin({
  shared: {
    ...shareAll({ singleton: true, strictVersion: true, requiredVersion: 'auto' }),
    // เพิ่ม shared service โดยเฉพาะ
    'src/app/shared/state/app-state.service': {
      singleton: true,
      eager: true
    }
  }
});
```

## 8. การ Deploy Micro-frontends

```yaml
# docker-compose.yml
version: '3.8'
services:
  shell:
    build: ./shell
    ports: ['4200:80']
    environment:
      - PRODUCTS_MFE_URL=http://products-mfe:80
      - ORDERS_MFE_URL=http://orders-mfe:80

  products-mfe:
    build: ./products-mfe
    ports: ['4201:80']

  orders-mfe:
    build: ./orders-mfe
    ports: ['4202:80']
```

## สรุปขั้นตอนการสร้าง Module Federation

1. **สร้าง Host App** - ทำหน้าที่เป็น shell ที่โหลด remotes
2. **สร้าง Remote Apps** - แต่ละ MFE expose modules/components
3. **ตั้งค่า webpack.config.js** - ทั้ง host และ remote
4. **แชร์ dependencies** - Angular, RxJS, ฯลฯ เป็น singleton
5. **ตั้งค่า routing** - ใน host ให้ load จาก remote
6. **Deploy แยกกัน** - แต่ละ MFE เป็น Docker container แยก

Module Federation ทำให้ทีมสามารถ deploy อิสระโดยไม่กระทบกัน แต่ผู้ใช้เห็นเป็นแอปเดียว
