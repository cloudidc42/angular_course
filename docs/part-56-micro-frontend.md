# Part 56: Micro-Frontend Concepts ใน Angular

## บทนำ

Micro-frontend เป็นสถาปัตยกรรมที่แบ่งแอปพลิเคชันขนาดใหญ่ออกเป็นส่วนย่อยๆ ที่แต่ละทีมสามารถพัฒนาและ deploy ได้อิสระ

## 1. แนวคิด Micro-Frontend

### ปัญหาของ Monolithic Frontend

```
┌─────────────────────────────────┐
│      Monolithic Angular App     │
│                                 │
│  ┌────────┐  ┌────────────────┐ │
│  │ Header │  │   Navigation   │ │
│  ├────────┴──┴────────────────┤ │
│  │                            │ │
│  │   Dashboard Module         │ │
│  │   Products Module          │ │
│  │   Orders Module            │ │
│  │   Users Module             │ │
│  │   Reports Module           │ │
│  │                            │ │
│  └────────────────────────────┘ │
└─────────────────────────────────┘
  - ทีมเดียวรับผิดชอบทั้งหมด
  - Deploy ทั้งแอปทุกครั้ง
  - Codebase ใหญ่และซับซ้อน
  - Technology lock-in
```

### Micro-Frontend Architecture

```
┌─────────────────────────────────────────┐
│           Shell/Host Application        │
│                                         │
│  ┌────────────────────────────────────┐ │
│  │         Shared Header/Nav          │ │
│  └────────────────────────────────────┘ │
│                                         │
│  ┌──────────┐  ┌───────────┐  ┌──────┐ │
│  │  Team A  │  │  Team B   │  │  C   │ │
│  │Dashboard │  │ Products  │  │Orders│ │
│  │  MFE     │  │   MFE     │  │ MFE  │ │
│  └──────────┘  └───────────┘  └──────┘ │
└─────────────────────────────────────────┘
  + แต่ละทีม deploy อิสระ
  + Technology agnostic
  + Codebase เล็กลง
  + Scale ได้ตามทีม
```

## 2. Micro-Frontend Patterns

### Pattern 1: iFrame Integration

วิธีที่ง่ายที่สุด แต่ UX ไม่ดี

```typescript
// iframe-mfe.component.ts
import { Component, Input } from '@angular/core';
import { DomSanitizer, SafeResourceUrl } from '@angular/platform-browser';

@Component({
  selector: 'app-iframe-mfe',
  standalone: true,
  template: `
    <iframe
      [src]="safeUrl"
      [style.height.px]="height"
      style="width: 100%; border: none;"
      (load)="onLoad()"
    ></iframe>
  `
})
export class IframeMfeComponent {
  @Input() set url(value: string) {
    this.safeUrl = this.sanitizer.bypassSecurityTrustResourceUrl(value);
  }
  @Input() height = 600;

  safeUrl!: SafeResourceUrl;

  constructor(private sanitizer: DomSanitizer) {}

  onLoad(): void {
    console.log('MFE loaded');
  }
}
```

### Pattern 2: Web Components

```typescript
// web-component-mfe.component.ts
import { Component, Input, OnInit, ElementRef, ViewChild } from '@angular/core';

@Component({
  selector: 'app-web-component-mfe',
  standalone: true,
  template: `
    <div #container></div>
  `
})
export class WebComponentMfeComponent implements OnInit {
  @ViewChild('container', { static: true }) container!: ElementRef;
  @Input() mfeName!: string;
  @Input() props: Record<string, any> = {};

  ngOnInit(): void {
    this.loadMfe();
  }

  private loadMfe(): void {
    const element = document.createElement(this.mfeName);
    
    // ส่ง props ไปยัง Web Component
    Object.entries(this.props).forEach(([key, value]) => {
      element.setAttribute(key, typeof value === 'object' ? JSON.stringify(value) : String(value));
    });

    this.container.nativeElement.appendChild(element);
  }
}
```

### Pattern 3: Module Federation (แนะนำ)

```
┌─────────────────────────────────────────────────────┐
│                    Host App (Shell)                  │
│  port: 4200                                         │
│                                                     │
│  ┌─────────────────────────────────────────────┐    │
│  │  Lazy Loads Remote Modules at Runtime        │    │
│  │                                             │    │
│  │  /dashboard → Remote: DashboardMFE:4201     │    │
│  │  /products  → Remote: ProductsMFE:4202      │    │
│  │  /orders    → Remote: OrdersMFE:4203        │    │
│  └─────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────┘
```

## 3. Communication Between MFEs

### Event Bus Pattern

```typescript
// mfe-event-bus.service.ts
import { Injectable } from '@angular/core';
import { Subject, Observable, filter, map } from 'rxjs';

export interface MfeEvent {
  type: string;
  source: string;
  payload: any;
  timestamp: number;
}

@Injectable({ providedIn: 'root' })
export class MfeEventBusService {
  private eventBus = new Subject<MfeEvent>();

  // Publish event
  publish(type: string, payload: any, source = 'unknown'): void {
    this.eventBus.next({
      type,
      source,
      payload,
      timestamp: Date.now()
    });
  }

  // Subscribe to specific event type
  on<T>(eventType: string): Observable<T> {
    return this.eventBus.asObservable().pipe(
      filter(event => event.type === eventType),
      map(event => event.payload as T)
    );
  }

  // Subscribe to events from specific source
  fromSource(source: string): Observable<MfeEvent> {
    return this.eventBus.asObservable().pipe(
      filter(event => event.source === source)
    );
  }
}
```

### Shared State ผ่าน Custom Events

```typescript
// cross-mfe-state.service.ts
import { Injectable, OnDestroy } from '@angular/core';
import { BehaviorSubject } from 'rxjs';

export interface SharedState {
  user: { id: string; name: string; role: string } | null;
  theme: 'light' | 'dark';
  language: string;
  permissions: string[];
}

// Service ที่ใช้ร่วมกันระหว่าง MFEs ผ่าน window object
@Injectable({ providedIn: 'root' })
export class CrossMfeStateService implements OnDestroy {
  private state$ = new BehaviorSubject<SharedState>({
    user: null,
    theme: 'light',
    language: 'th',
    permissions: []
  });

  readonly state = this.state$.asObservable();
  private eventHandler: ((e: CustomEvent) => void) | null = null;

  constructor() {
    this.listenToExternalEvents();
    this.registerWithWindow();
  }

  updateUser(user: SharedState['user']): void {
    this.updateState({ user });
  }

  updateTheme(theme: SharedState['theme']): void {
    this.updateState({ theme });
    document.documentElement.setAttribute('data-theme', theme);
  }

  private updateState(partial: Partial<SharedState>): void {
    const current = this.state$.value;
    const newState = { ...current, ...partial };
    this.state$.next(newState);
    this.broadcastState(newState);
  }

  private broadcastState(state: SharedState): void {
    window.dispatchEvent(new CustomEvent('mfe:state-changed', {
      detail: state,
      bubbles: true
    }));
  }

  private listenToExternalEvents(): void {
    this.eventHandler = (e: CustomEvent) => {
      if (e.detail && e.type === 'mfe:state-changed') {
        this.state$.next(e.detail);
      }
    };
    window.addEventListener('mfe:state-changed', this.eventHandler as EventListener);
  }

  private registerWithWindow(): void {
    (window as any).__mfeState = {
      get: () => this.state$.value,
      update: (partial: Partial<SharedState>) => this.updateState(partial)
    };
  }

  ngOnDestroy(): void {
    if (this.eventHandler) {
      window.removeEventListener('mfe:state-changed', this.eventHandler as EventListener);
    }
  }
}
```

## 4. Shared Libraries

```typescript
// shared/auth/auth.service.ts (ใช้ร่วมกันทุก MFE)
export interface AuthToken {
  accessToken: string;
  refreshToken: string;
  expiresAt: number;
}

export interface User {
  id: string;
  email: string;
  name: string;
  roles: string[];
  permissions: string[];
}

// แชร์ผ่าน npm package หรือ Module Federation shared
export class SharedAuthService {
  private readonly TOKEN_KEY = 'auth_token';

  getToken(): AuthToken | null {
    const raw = localStorage.getItem(this.TOKEN_KEY);
    if (!raw) return null;
    
    try {
      return JSON.parse(raw);
    } catch {
      return null;
    }
  }

  setToken(token: AuthToken): void {
    localStorage.setItem(this.TOKEN_KEY, JSON.stringify(token));
  }

  clearToken(): void {
    localStorage.removeItem(this.TOKEN_KEY);
  }

  isAuthenticated(): boolean {
    const token = this.getToken();
    if (!token) return false;
    return Date.now() < token.expiresAt;
  }

  hasPermission(permission: string): boolean {
    const user = this.getCurrentUser();
    return user?.permissions.includes(permission) ?? false;
  }

  getCurrentUser(): User | null {
    const token = this.getToken();
    if (!token) return null;
    
    try {
      const payload = JSON.parse(atob(token.accessToken.split('.')[1]));
      return payload.user;
    } catch {
      return null;
    }
  }
}
```

## 5. Loading Remote MFE Component

```typescript
// remote-loader.component.ts
import {
  Component, Input, OnInit, ViewContainerRef,
  Compiler, NgModuleFactory, ComponentFactory
} from '@angular/core';
import { CommonModule } from '@angular/common';

interface RemoteConfig {
  url: string;       // URL ของ remote entry
  moduleName: string; // ชื่อ module
  componentName: string; // ชื่อ component
}

@Component({
  selector: 'app-remote-loader',
  standalone: true,
  imports: [CommonModule],
  template: `
    <div *ngIf="loading" class="loading-mfe">
      กำลังโหลด {{ config?.componentName }}...
    </div>
    <div *ngIf="error" class="error-mfe">
      ไม่สามารถโหลด MFE: {{ error }}
    </div>
  `
})
export class RemoteLoaderComponent implements OnInit {
  @Input() config!: RemoteConfig;
  
  loading = true;
  error: string | null = null;

  constructor(
    private viewContainerRef: ViewContainerRef
  ) {}

  ngOnInit(): void {
    this.loadRemote();
  }

  private async loadRemote(): Promise<void> {
    try {
      // ใช้ Dynamic import สำหรับ Module Federation
      const module = await this.loadRemoteModule(this.config);
      const componentClass = module[this.config.componentName];
      
      if (!componentClass) {
        throw new Error(`Component ${this.config.componentName} not found`);
      }

      this.viewContainerRef.createComponent(componentClass);
      this.loading = false;
    } catch (err: any) {
      this.error = err.message;
      this.loading = false;
    }
  }

  private loadRemoteModule(config: RemoteConfig): Promise<any> {
    return new Promise((resolve, reject) => {
      // สร้าง script tag สำหรับ remote entry
      const script = document.createElement('script');
      script.src = `${config.url}/remoteEntry.js`;
      
      script.onload = async () => {
        try {
          // เข้าถึง exposed module จาก remote
          const container = (window as any)[config.moduleName];
          await container.init({});
          const factory = await container.get(`./${config.componentName}`);
          resolve(factory());
        } catch (err) {
          reject(err);
        }
      };
      
      script.onerror = () => reject(new Error(`Failed to load ${config.url}`));
      document.head.appendChild(script);
    });
  }
}
```

## 6. การจัดการ Routing ใน Micro-Frontend

```typescript
// shell-routing.ts
import { Routes } from '@angular/router';
import { RemoteLoaderComponent } from './remote-loader.component';
import { authGuard } from './guards/auth.guard';

export const routes: Routes = [
  {
    path: '',
    redirectTo: 'dashboard',
    pathMatch: 'full'
  },
  {
    path: 'dashboard',
    loadChildren: () =>
      // Module Federation: load from remote
      import('dashboardMfe/DashboardModule').then(m => m.DashboardModule)
  },
  {
    path: 'products',
    canActivate: [authGuard],
    loadChildren: () =>
      import('productsMfe/ProductsModule').then(m => m.ProductsModule)
  },
  {
    path: 'orders',
    canActivate: [authGuard],
    component: RemoteLoaderComponent,
    data: {
      remoteConfig: {
        url: 'http://localhost:4203',
        moduleName: 'ordersMfe',
        componentName: 'OrdersComponent'
      }
    }
  }
];
```

## 7. Design System แบบ Shared

```typescript
// shared-ui/button/button.component.ts
// เป็น Web Component ที่ใช้ได้ทุก Framework
import { Component, Input, Output, EventEmitter } from '@angular/core';

@Component({
  selector: 'mfe-button',
  standalone: true,
  template: `
    <button
      [type]="type"
      [disabled]="disabled"
      [class]="buttonClass"
      (click)="onClick.emit($event)"
    >
      <ng-content></ng-content>
    </button>
  `,
  styles: [`
    button {
      padding: 0.5rem 1rem;
      border: none;
      border-radius: 4px;
      cursor: pointer;
      font-size: 1rem;
      transition: opacity 0.2s;
    }
    button:disabled { opacity: 0.5; cursor: not-allowed; }
    .btn-primary { background: #007bff; color: white; }
    .btn-secondary { background: #6c757d; color: white; }
    .btn-danger { background: #dc3545; color: white; }
    .btn-outline {
      background: transparent;
      border: 1px solid #007bff;
      color: #007bff;
    }
  `]
})
export class SharedButtonComponent {
  @Input() variant: 'primary' | 'secondary' | 'danger' | 'outline' = 'primary';
  @Input() type: 'button' | 'submit' | 'reset' = 'button';
  @Input() disabled = false;
  @Output() onClick = new EventEmitter<MouseEvent>();

  get buttonClass(): string {
    return `btn-${this.variant}`;
  }
}
```

## สรุป

| Pattern | ความง่าย | Integration | Performance |
|---------|---------|-------------|-------------|
| iFrame | ง่ายมาก | แยกส่วนสมบูรณ์ | ช้า |
| Web Components | ปานกลาง | ดี | ปานกลาง |
| Module Federation | ซับซ้อน | ดีมาก | ดีมาก |
| npm packages | ง่าย | ต้อง redeploy | ดี |

Micro-frontend เหมาะสำหรับองค์กรขนาดใหญ่ที่มีหลายทีมพัฒนา frontend พร้อมกัน
