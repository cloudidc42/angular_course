# Part 58: Nx Monorepo ใน Angular

## บทนำ

Nx เป็นเครื่องมือสำหรับจัดการ Monorepo ที่ทรงพลัง ช่วยให้หลายแอปพลิเคชันและ libraries อยู่ใน repository เดียวกันได้อย่างมีประสิทธิภาพ

## 1. การติดตั้ง Nx Workspace

```bash
# สร้าง Nx workspace ใหม่
npx create-nx-workspace@latest my-company --preset=angular-monorepo

# หรือเพิ่ม Nx ใน Angular project ที่มีอยู่
ng add @nx/angular
```

### โครงสร้าง Nx Workspace

```
my-company/
├── apps/
│   ├── shell/                    # Host/Shell Application
│   ├── shell-e2e/                # E2E tests สำหรับ shell
│   ├── products-mfe/             # Products Micro-frontend
│   └── orders-mfe/               # Orders Micro-frontend
├── libs/
│   ├── shared/
│   │   ├── ui/                   # Shared UI Components
│   │   ├── util-auth/            # Authentication utilities
│   │   ├── util-http/            # HTTP utilities
│   │   └── data-access/          # Shared data services
│   ├── products/
│   │   ├── feature-product-list/ # Feature: Product List
│   │   ├── feature-product-detail/ # Feature: Product Detail
│   │   ├── data-access/          # Products API services
│   │   └── ui/                   # Products-specific UI
│   └── orders/
│       ├── feature-order-list/
│       ├── feature-checkout/
│       └── data-access/
├── nx.json                       # Nx configuration
├── project.json                  # Root project config
└── tsconfig.base.json            # Base TypeScript config
```

## 2. สร้าง Applications และ Libraries

```bash
# สร้าง Angular Application
nx generate @nx/angular:application products-mfe --routing --style=scss

# สร้าง Angular Library
nx generate @nx/angular:library shared/ui --publishable --importPath=@my-company/shared/ui

# สร้าง Feature Library
nx generate @nx/angular:library products/feature-product-list --lazy

# สร้าง Data Access Library
nx generate @nx/angular:library products/data-access

# สร้าง Utility Library
nx generate @nx/js:library shared/util-format --unitTestRunner=jest
```

## 3. ตั้งค่า nx.json

```json
{
  "$schema": "./node_modules/nx/schemas/nx-schema.json",
  "defaultBase": "main",
  "namedInputs": {
    "default": ["{projectRoot}/**/*", "sharedGlobals"],
    "production": [
      "default",
      "!{projectRoot}/**/*.spec.ts",
      "!{projectRoot}/tsconfig.spec.json"
    ],
    "sharedGlobals": []
  },
  "targetDefaults": {
    "build": {
      "cache": true,
      "dependsOn": ["^build"],
      "inputs": ["production", "^production"]
    },
    "test": {
      "cache": true,
      "inputs": ["default", "^production", "{workspaceRoot}/jest.preset.js"]
    },
    "lint": {
      "cache": true,
      "inputs": [
        "default",
        "{workspaceRoot}/.eslintrc.json",
        "{workspaceRoot}/.eslintignore"
      ]
    }
  },
  "generators": {
    "@nx/angular:application": {
      "style": "scss",
      "linter": "eslint",
      "unitTestRunner": "jest"
    },
    "@nx/angular:library": {
      "style": "scss",
      "linter": "eslint",
      "unitTestRunner": "jest"
    },
    "@nx/angular:component": {
      "style": "scss"
    }
  }
}
```

## 4. Shared UI Library

```typescript
// libs/shared/ui/src/lib/button/button.component.ts
import { Component, Input, Output, EventEmitter } from '@angular/core';
import { CommonModule } from '@angular/common';

@Component({
  selector: 'my-company-button',
  standalone: true,
  imports: [CommonModule],
  template: `
    <button
      [type]="type"
      [disabled]="disabled || loading"
      [class]="buttonClasses"
      (click)="handleClick($event)"
    >
      <span *ngIf="loading" class="spinner"></span>
      <ng-content></ng-content>
    </button>
  `,
  styles: [`
    button {
      display: inline-flex;
      align-items: center;
      gap: 0.5rem;
      padding: 0.5rem 1.25rem;
      border: none;
      border-radius: 6px;
      font-size: 0.9rem;
      font-weight: 500;
      cursor: pointer;
      transition: all 0.2s;
    }
    button:disabled { opacity: 0.6; cursor: not-allowed; }
    .btn-primary { background: #007bff; color: white; }
    .btn-primary:hover:not(:disabled) { background: #0056b3; }
    .btn-secondary { background: #6c757d; color: white; }
    .btn-outline { background: transparent; border: 2px solid #007bff; color: #007bff; }
    .btn-danger { background: #dc3545; color: white; }
    .btn-sm { padding: 0.25rem 0.75rem; font-size: 0.8rem; }
    .btn-lg { padding: 0.75rem 2rem; font-size: 1.1rem; }
    .spinner {
      width: 16px;
      height: 16px;
      border: 2px solid rgba(255,255,255,0.3);
      border-top-color: white;
      border-radius: 50%;
      animation: spin 0.6s linear infinite;
    }
    @keyframes spin { to { transform: rotate(360deg); } }
  `]
})
export class ButtonComponent {
  @Input() variant: 'primary' | 'secondary' | 'outline' | 'danger' = 'primary';
  @Input() size: 'sm' | 'md' | 'lg' = 'md';
  @Input() type: 'button' | 'submit' | 'reset' = 'button';
  @Input() disabled = false;
  @Input() loading = false;
  @Output() clicked = new EventEmitter<MouseEvent>();

  get buttonClasses(): string {
    return [
      `btn-${this.variant}`,
      this.size !== 'md' ? `btn-${this.size}` : ''
    ].filter(Boolean).join(' ');
  }

  handleClick(event: MouseEvent): void {
    if (!this.disabled && !this.loading) {
      this.clicked.emit(event);
    }
  }
}
```

```typescript
// libs/shared/ui/src/index.ts
export * from './lib/button/button.component';
export * from './lib/card/card.component';
export * from './lib/modal/modal.component';
export * from './lib/table/table.component';
export * from './lib/form/input/input.component';
export * from './lib/notification/notification.component';
```

## 5. Data Access Library

```typescript
// libs/products/data-access/src/lib/products.service.ts
import { Injectable } from '@angular/core';
import { HttpClient, HttpParams } from '@angular/common/http';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';

export interface Product {
  id: string;
  name: string;
  description: string;
  price: number;
  stock: number;
  categoryId: string;
  images: string[];
  createdAt: Date;
}

export interface ProductsResponse {
  items: Product[];
  total: number;
  page: number;
  limit: number;
}

export interface ProductFilters {
  search?: string;
  categoryId?: string;
  minPrice?: number;
  maxPrice?: number;
  inStock?: boolean;
  page?: number;
  limit?: number;
}

@Injectable({ providedIn: 'root' })
export class ProductsService {
  private readonly apiUrl = '/api/products';

  constructor(private http: HttpClient) {}

  getProducts(filters: ProductFilters = {}): Observable<ProductsResponse> {
    let params = new HttpParams();
    
    Object.entries(filters).forEach(([key, value]) => {
      if (value !== undefined && value !== null) {
        params = params.set(key, String(value));
      }
    });

    return this.http.get<ProductsResponse>(this.apiUrl, { params });
  }

  getProduct(id: string): Observable<Product> {
    return this.http.get<Product>(`${this.apiUrl}/${id}`);
  }

  createProduct(product: Omit<Product, 'id' | 'createdAt'>): Observable<Product> {
    return this.http.post<Product>(this.apiUrl, product);
  }

  updateProduct(id: string, changes: Partial<Product>): Observable<Product> {
    return this.http.patch<Product>(`${this.apiUrl}/${id}`, changes);
  }

  deleteProduct(id: string): Observable<void> {
    return this.http.delete<void>(`${this.apiUrl}/${id}`);
  }

  uploadProductImage(productId: string, file: File): Observable<{ url: string }> {
    const formData = new FormData();
    formData.append('image', file);
    return this.http.post<{ url: string }>(
      `${this.apiUrl}/${productId}/images`,
      formData
    );
  }
}
```

## 6. Feature Library

```typescript
// libs/products/feature-product-list/src/lib/product-list.component.ts
import { Component, OnInit } from '@angular/core';
import { CommonModule } from '@angular/common';
import { RouterLink } from '@angular/router';
import { FormControl, ReactiveFormsModule } from '@angular/forms';
import { debounceTime, distinctUntilChanged, switchMap, startWith } from 'rxjs/operators';

// Import จาก libraries
import { ButtonComponent } from '@my-company/shared/ui';
import { ProductsService, Product } from '@my-company/products/data-access';

@Component({
  selector: 'my-company-product-list',
  standalone: true,
  imports: [CommonModule, RouterLink, ReactiveFormsModule, ButtonComponent],
  template: `
    <div class="product-list-page">
      <div class="page-header">
        <h1>รายการสินค้า</h1>
        <my-company-button variant="primary" [routerLink]="['/products/new']">
          + เพิ่มสินค้า
        </my-company-button>
      </div>

      <div class="search-bar">
        <input
          [formControl]="searchControl"
          placeholder="ค้นหาสินค้า..."
          class="search-input"
        />
      </div>

      <div *ngIf="loading" class="loading">กำลังโหลด...</div>

      <div class="products-grid" *ngIf="!loading">
        <div *ngFor="let product of products" class="product-card">
          <img [src]="product.images[0]" [alt]="product.name" />
          <div class="product-info">
            <h3>{{ product.name }}</h3>
            <p class="price">฿{{ product.price | number }}</p>
            <p class="stock" [class.low-stock]="product.stock < 10">
              คงเหลือ: {{ product.stock }} ชิ้น
            </p>
          </div>
          <div class="product-actions">
            <my-company-button
              variant="outline"
              size="sm"
              [routerLink]="['/products', product.id]"
            >แก้ไข</my-company-button>
            <my-company-button
              variant="danger"
              size="sm"
              (clicked)="deleteProduct(product.id)"
            >ลบ</my-company-button>
          </div>
        </div>
      </div>

      <div class="pagination">
        <my-company-button
          variant="outline"
          size="sm"
          [disabled]="currentPage <= 1"
          (clicked)="changePage(currentPage - 1)"
        >ก่อนหน้า</my-company-button>
        <span>หน้า {{ currentPage }} / {{ totalPages }}</span>
        <my-company-button
          variant="outline"
          size="sm"
          [disabled]="currentPage >= totalPages"
          (clicked)="changePage(currentPage + 1)"
        >ถัดไป</my-company-button>
      </div>
    </div>
  `,
  styles: [`
    .product-list-page { padding: 1rem; }
    .page-header {
      display: flex;
      justify-content: space-between;
      align-items: center;
      margin-bottom: 1rem;
    }
    .search-input {
      width: 100%;
      padding: 0.75rem;
      border: 1px solid #ddd;
      border-radius: 6px;
      font-size: 1rem;
      margin-bottom: 1rem;
    }
    .products-grid {
      display: grid;
      grid-template-columns: repeat(auto-fill, minmax(250px, 1fr));
      gap: 1rem;
    }
    .product-card {
      border: 1px solid #eee;
      border-radius: 8px;
      overflow: hidden;
      background: white;
    }
    .product-card img { width: 100%; height: 200px; object-fit: cover; }
    .product-info { padding: 1rem; }
    .price { font-size: 1.2rem; font-weight: bold; color: #007bff; }
    .low-stock { color: #dc3545; }
    .product-actions {
      padding: 0.75rem;
      border-top: 1px solid #eee;
      display: flex;
      gap: 0.5rem;
    }
    .pagination {
      display: flex;
      justify-content: center;
      align-items: center;
      gap: 1rem;
      margin-top: 1.5rem;
    }
  `]
})
export class ProductListComponent implements OnInit {
  products: Product[] = [];
  loading = false;
  currentPage = 1;
  totalPages = 1;
  searchControl = new FormControl('');

  constructor(private productsService: ProductsService) {}

  ngOnInit(): void {
    this.searchControl.valueChanges.pipe(
      startWith(''),
      debounceTime(300),
      distinctUntilChanged(),
      switchMap(search => {
        this.loading = true;
        return this.productsService.getProducts({ search: search || '', page: 1 });
      })
    ).subscribe({
      next: (response) => {
        this.products = response.items;
        this.totalPages = Math.ceil(response.total / 20);
        this.loading = false;
      }
    });
  }

  changePage(page: number): void {
    this.currentPage = page;
    this.loading = true;
    this.productsService.getProducts({
      search: this.searchControl.value || '',
      page
    }).subscribe(response => {
      this.products = response.items;
      this.loading = false;
    });
  }

  deleteProduct(id: string): void {
    if (confirm('ต้องการลบสินค้านี้ใช่หรือไม่?')) {
      this.productsService.deleteProduct(id).subscribe(() => {
        this.products = this.products.filter(p => p.id !== id);
      });
    }
  }
}
```

## 7. Nx Commands ที่สำคัญ

```bash
# รัน application
nx serve shell
nx serve products-mfe

# Build
nx build shell
nx build products-mfe --configuration=production

# Test
nx test shared-ui
nx test products-data-access

# Lint
nx lint shell
nx lint --all

# Affected commands (เฉพาะที่เปลี่ยนแปลง)
nx affected:test
nx affected:build
nx affected:lint

# Dependency graph
nx graph

# Format code
nx format:write

# Generate components
nx generate @nx/angular:component button --project=shared-ui --standalone

# Run multiple targets
nx run-many --target=test --all
nx run-many --target=build --projects=shell,products-mfe
```

## 8. Tags และ Constraints

```json
// project.json ของแต่ละ project
{
  "name": "products-feature-product-list",
  "tags": ["scope:products", "type:feature"]
}

// .eslintrc.json (root)
{
  "rules": {
    "@nx/enforce-module-boundaries": [
      "error",
      {
        "enforceBuildableLibDependency": true,
        "allow": [],
        "depConstraints": [
          {
            "sourceTag": "type:app",
            "onlyDependOnLibsWithTags": ["type:feature", "type:ui", "type:data-access", "type:util"]
          },
          {
            "sourceTag": "type:feature",
            "onlyDependOnLibsWithTags": ["type:ui", "type:data-access", "type:util"]
          },
          {
            "sourceTag": "scope:products",
            "onlyDependOnLibsWithTags": ["scope:products", "scope:shared"]
          }
        ]
      }
    ]
  }
}
```

## สรุป Nx Monorepo Benefits

| ประโยชน์ | รายละเอียด |
|---------|-----------|
| Code Sharing | ใช้ libraries ร่วมกันได้ง่าย |
| Affected Commands | Build/Test เฉพาะที่เปลี่ยน |
| Dependency Graph | เห็น dependencies ชัดเจน |
| Caching | Build cache ทั้ง local และ cloud |
| Code Generation | Generators ช่วยสร้าง boilerplate |
| Constraints | บังคับ module boundaries |

Nx เหมาะสำหรับองค์กรที่มีหลาย applications และต้องการ share code ระหว่างกัน
