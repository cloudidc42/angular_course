# Part 31 — Standalone Components

## Standalone Components คืออะไร?

Standalone Components เป็น Feature ที่เพิ่มมาใน Angular 14 และกลายเป็น Default ใน Angular 17 ช่วยให้ Components ทำงานได้โดยไม่ต้องพึ่งพา NgModule ทำให้โครงสร้างแอปง่ายขึ้นและโค้ดอ่านง่ายขึ้น

### ความแตกต่างระหว่าง NgModule กับ Standalone

**NgModule (วิธีเดิม):**
```typescript
// ต้องประกาศใน Module
@NgModule({
  declarations: [MyComponent],
  imports: [CommonModule, FormsModule],
  exports: [MyComponent],
})
export class MyModule {}
```

**Standalone (วิธีใหม่):**
```typescript
// Component จัดการ Dependencies ของตัวเอง
@Component({
  standalone: true,
  imports: [CommonModule, FormsModule],
  // ...
})
export class MyComponent {}
```

---

## สร้าง Standalone Component

```typescript
import { Component } from '@angular/core';
import { CommonModule } from '@angular/common';
import { FormsModule } from '@angular/forms';
import { RouterModule } from '@angular/router';

@Component({
  selector: 'app-hello',
  standalone: true,                          // บอก Angular ว่าเป็น Standalone
  imports: [CommonModule, FormsModule, RouterModule],  // Import สิ่งที่ต้องการ
  template: `
    <div>
      <h1>สวัสดี {{ name }}</h1>
      <input [(ngModel)]="name" placeholder="ชื่อของคุณ" />
      <a routerLink="/home">กลับหน้าหลัก</a>
    </div>
  `,
})
export class HelloComponent {
  name = 'Angular';
}
```

### Standalone Pipe และ Directive

```typescript
// Standalone Pipe
@Pipe({
  name: 'thaiDate',
  standalone: true,
})
export class ThaiDatePipe implements PipeTransform {
  transform(date: Date | string): string {
    const d = new Date(date);
    return d.toLocaleDateString('th-TH', {
      year: 'numeric',
      month: 'long',
      day: 'numeric',
    });
  }
}

// Standalone Directive
@Directive({
  selector: '[appHighlight]',
  standalone: true,
})
export class HighlightDirective {
  @Input() appHighlight = 'yellow';

  constructor(private el: ElementRef) {}

  @HostListener('mouseenter')
  onMouseEnter(): void {
    this.el.nativeElement.style.backgroundColor = this.appHighlight;
  }

  @HostListener('mouseleave')
  onMouseLeave(): void {
    this.el.nativeElement.style.backgroundColor = '';
  }
}
```

---

## bootstrapApplication

แทน `AppModule` ด้วย `bootstrapApplication`:

```typescript
// main.ts
import { bootstrapApplication } from '@angular/platform-browser';
import { AppComponent } from './app/app.component';
import { appConfig } from './app/app.config';

bootstrapApplication(AppComponent, appConfig).catch((err) =>
  console.error(err)
);
```

```typescript
// app.config.ts
import { ApplicationConfig } from '@angular/core';
import { provideRouter } from '@angular/router';
import { provideHttpClient } from '@angular/common/http';
import { provideAnimations } from '@angular/platform-browser/animations';
import { routes } from './app.routes';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes),
    provideHttpClient(),
    provideAnimations(),
  ],
};
```

```typescript
// app.component.ts
import { Component } from '@angular/core';
import { RouterOutlet } from '@angular/router';
import { HeaderComponent } from './header/header.component';
import { FooterComponent } from './footer/footer.component';

@Component({
  selector: 'app-root',
  standalone: true,
  imports: [RouterOutlet, HeaderComponent, FooterComponent],
  template: `
    <app-header />
    <main>
      <router-outlet />
    </main>
    <app-footer />
  `,
})
export class AppComponent {}
```

---

## provideRouter

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
      import('./pages/home/home.component').then((m) => m.HomeComponent),
  },
  {
    path: 'products',
    loadComponent: () =>
      import('./pages/products/products.component').then(
        (m) => m.ProductsComponent
      ),
  },
  {
    path: 'products/:id',
    loadComponent: () =>
      import('./pages/product-detail/product-detail.component').then(
        (m) => m.ProductDetailComponent
      ),
  },
  {
    path: 'admin',
    // Lazy load ทั้ง Route Group
    loadChildren: () =>
      import('./pages/admin/admin.routes').then((m) => m.adminRoutes),
  },
  {
    path: '**',
    loadComponent: () =>
      import('./pages/not-found/not-found.component').then(
        (m) => m.NotFoundComponent
      ),
  },
];
```

```typescript
// app.config.ts — provideRouter พร้อม Options
import { provideRouter, withPreloading, PreloadAllModules, withComponentInputBinding, withViewTransitions } from '@angular/router';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(
      routes,
      withPreloading(PreloadAllModules),     // Preload ทุก Lazy Module
      withComponentInputBinding(),            // Route Params เป็น @Input อัตโนมัติ
      withViewTransitions(),                  // Animate ระหว่าง Routes
    ),
    provideHttpClient(),
    provideAnimations(),
  ],
};
```

---

## provideHttpClient

```typescript
// app.config.ts — HttpClient พร้อม Interceptors
import {
  provideHttpClient,
  withInterceptors,
  withInterceptorsFromDi,
} from '@angular/common/http';

// Functional Interceptor (Angular 15+)
const authInterceptor = (req: HttpRequest<unknown>, next: HttpHandlerFn) => {
  const token = localStorage.getItem('token');
  if (token) {
    const authReq = req.clone({
      headers: req.headers.set('Authorization', `Bearer ${token}`),
    });
    return next(authReq);
  }
  return next(req);
};

const loggingInterceptor = (req: HttpRequest<unknown>, next: HttpHandlerFn) => {
  console.log(`[HTTP] ${req.method} ${req.url}`);
  const start = Date.now();
  return next(req).pipe(
    tap((event) => {
      if (event instanceof HttpResponse) {
        console.log(`[HTTP] Response ${event.status} in ${Date.now() - start}ms`);
      }
    })
  );
};

export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient(
      withInterceptors([authInterceptor, loggingInterceptor])
    ),
  ],
};
```

---

## Workshop: Convert App to Standalone

ขั้นตอนการแปลงแอปที่ใช้ NgModule เป็น Standalone

### ขั้นตอนที่ 1: ใช้ Angular Migration Schematic

```bash
# Migration อัตโนมัติ (Angular 15.2+)
ng generate @angular/core:standalone

# เลือก Mode:
# 1. Convert all components, directives and pipes to standalone
# 2. Remove unnecessary NgModule classes
# 3. Bootstrap the application using standalone APIs
```

### ขั้นตอนที่ 2: แปลง Component ทีละตัว (Manual)

**ก่อนแปลง:**
```typescript
// header.component.ts
@Component({
  selector: 'app-header',
  templateUrl: './header.component.html',
  styleUrls: ['./header.component.scss'],
})
export class HeaderComponent {
  isMenuOpen = false;
}

// shared.module.ts
@NgModule({
  declarations: [HeaderComponent],
  imports: [CommonModule, RouterModule],
  exports: [HeaderComponent],
})
export class SharedModule {}
```

**หลังแปลง:**
```typescript
// header.component.ts
import { Component } from '@angular/core';
import { CommonModule } from '@angular/common';
import { RouterModule } from '@angular/router';

@Component({
  selector: 'app-header',
  standalone: true,
  imports: [CommonModule, RouterModule],
  templateUrl: './header.component.html',
  styleUrls: ['./header.component.scss'],
})
export class HeaderComponent {
  isMenuOpen = false;
}
```

### ขั้นตอนที่ 3: แปลง AppModule

**ก่อน:**
```typescript
// app.module.ts
@NgModule({
  declarations: [AppComponent],
  imports: [
    BrowserModule,
    HttpClientModule,
    RouterModule.forRoot(routes),
    StoreModule.forRoot(reducers),
    EffectsModule.forRoot(effects),
  ],
  bootstrap: [AppComponent],
})
export class AppModule {}
```

**หลัง:**
```typescript
// app.config.ts
import { provideStore } from '@ngrx/store';
import { provideEffects } from '@ngrx/effects';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes),
    provideHttpClient(),
    provideAnimations(),
    provideStore(reducers),
    provideEffects(effects),
  ],
};

// main.ts
bootstrapApplication(AppComponent, appConfig);
```

---

## ตัวอย่างแอปสมบูรณ์แบบ Standalone

### โครงสร้าง

```
src/app/
├── app.component.ts          (Standalone Root)
├── app.config.ts             (Application Config)
├── app.routes.ts             (Routes)
├── core/
│   ├── guards/
│   │   └── auth.guard.ts
│   ├── interceptors/
│   │   └── auth.interceptor.ts
│   └── services/
│       └── auth.service.ts
├── shared/
│   ├── components/
│   │   ├── button/
│   │   │   └── button.component.ts
│   │   └── card/
│   │       └── card.component.ts
│   ├── directives/
│   │   └── highlight.directive.ts
│   └── pipes/
│       └── thai-date.pipe.ts
└── pages/
    ├── home/
    │   └── home.component.ts
    └── products/
        └── products.component.ts
```

### Shared Button Component

```typescript
// shared/components/button/button.component.ts
import { Component, Input, Output, EventEmitter } from '@angular/core';
import { CommonModule } from '@angular/common';

@Component({
  selector: 'app-button',
  standalone: true,
  imports: [CommonModule],
  template: `
    <button
      [type]="type"
      [class]="'btn btn-' + variant"
      [disabled]="disabled || loading"
      (click)="onClick()"
    >
      <span *ngIf="loading" class="spinner"></span>
      <ng-content></ng-content>
    </button>
  `,
  styles: [`
    .btn { padding: 8px 16px; border-radius: 4px; cursor: pointer; border: none; }
    .btn:disabled { opacity: 0.6; cursor: not-allowed; }
    .btn-primary { background: #1976d2; color: white; }
    .btn-secondary { background: #757575; color: white; }
    .btn-danger { background: #d32f2f; color: white; }
    .spinner { display: inline-block; width: 16px; height: 16px;
               border: 2px solid rgba(255,255,255,0.3);
               border-top-color: white; border-radius: 50%;
               animation: spin 0.8s linear infinite; }
    @keyframes spin { to { transform: rotate(360deg); } }
  `]
})
export class ButtonComponent {
  @Input() variant: 'primary' | 'secondary' | 'danger' = 'primary';
  @Input() type: 'button' | 'submit' | 'reset' = 'button';
  @Input() disabled = false;
  @Input() loading = false;
  @Output() clicked = new EventEmitter<void>();

  onClick(): void {
    if (!this.disabled && !this.loading) {
      this.clicked.emit();
    }
  }
}
```

### Products Page (Standalone)

```typescript
// pages/products/products.component.ts
import { Component, OnInit, inject } from '@angular/core';
import { CommonModule } from '@angular/common';
import { FormsModule } from '@angular/forms';
import { RouterModule } from '@angular/router';
import { ProductService } from '../../core/services/product.service';
import { ButtonComponent } from '../../shared/components/button/button.component';
import { CardComponent } from '../../shared/components/card/card.component';
import { ThaiDatePipe } from '../../shared/pipes/thai-date.pipe';
import { HighlightDirective } from '../../shared/directives/highlight.directive';

@Component({
  selector: 'app-products',
  standalone: true,
  imports: [
    CommonModule,
    FormsModule,
    RouterModule,
    ButtonComponent,
    CardComponent,
    ThaiDatePipe,
    HighlightDirective,
  ],
  template: `
    <div class="products-page">
      <h1>สินค้าทั้งหมด</h1>

      <div class="search-bar">
        <input [(ngModel)]="searchTerm" placeholder="ค้นหา..." />
        <app-button (clicked)="search()">ค้นหา</app-button>
        <app-button variant="secondary" (clicked)="reset()">รีเซต</app-button>
      </div>

      <div *ngIf="loading" class="loading">กำลังโหลด...</div>
      <div *ngIf="error" class="error">{{ error }}</div>

      <div class="products-grid">
        <app-card
          *ngFor="let product of filteredProducts"
          appHighlight="lightyellow"
        >
          <h3>{{ product.name }}</h3>
          <p>{{ product.description }}</p>
          <p>ราคา: {{ product.price | currency:'THB' }}</p>
          <p>เพิ่มเมื่อ: {{ product.createdAt | thaiDate }}</p>
          <app-button (clicked)="viewDetail(product.id)">
            ดูรายละเอียด
          </app-button>
        </app-card>
      </div>
    </div>
  `
})
export class ProductsComponent implements OnInit {
  private productService = inject(ProductService);

  products: any[] = [];
  filteredProducts: any[] = [];
  searchTerm = '';
  loading = false;
  error = '';

  ngOnInit(): void {
    this.loadProducts();
  }

  loadProducts(): void {
    this.loading = true;
    this.productService.getAll().subscribe({
      next: (products) => {
        this.products = products;
        this.filteredProducts = products;
        this.loading = false;
      },
      error: (err) => {
        this.error = err.message;
        this.loading = false;
      },
    });
  }

  search(): void {
    const term = this.searchTerm.toLowerCase();
    this.filteredProducts = this.products.filter((p) =>
      p.name.toLowerCase().includes(term)
    );
  }

  reset(): void {
    this.searchTerm = '';
    this.filteredProducts = this.products;
  }

  viewDetail(id: number): void {
    // นำทางไปหน้ารายละเอียด
  }
}
```

---

## Feature Flag สำหรับ Migration

```typescript
// สำหรับแอปที่ค่อยๆ แปลง สามารถใช้ NgModule ร่วมกับ Standalone ได้

// Standalone Component ใน NgModule
@NgModule({
  declarations: [LegacyComponent],
  imports: [
    CommonModule,
    // ใช้ Standalone Component ใน Module ได้ตรงๆ
    NewStandaloneComponent,
    NewStandalonePipe,
  ],
})
export class LegacyModule {}

// หรือ NgModule Component ใน Standalone Component
@Component({
  standalone: true,
  imports: [
    LegacyModule,  // import Module เดิมได้เลย
  ],
})
export class NewStandaloneComponent {}
```

---

## สรุป

| แบบเดิม (NgModule) | แบบใหม่ (Standalone) |
|-------------------|---------------------|
| `@NgModule({ declarations: [...] })` | `@Component({ standalone: true })` |
| `@NgModule({ imports: [...] })` | `@Component({ imports: [...] })` |
| `AppModule` + `BrowserModule` | `bootstrapApplication` + `appConfig` |
| `RouterModule.forRoot(routes)` | `provideRouter(routes)` |
| `HttpClientModule` | `provideHttpClient()` |
| `BrowserAnimationsModule` | `provideAnimations()` |

Standalone Components ทำให้โค้ด Angular สะอาดขึ้น Bundle เล็กลง และเข้าใจง่ายขึ้น ใน Part ถัดไปจะเรียน Advanced Routing ที่รวม Resolvers, Preloading และฟีเจอร์ขั้นสูงอื่นๆ
