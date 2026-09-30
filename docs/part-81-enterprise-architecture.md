# Part 81: Enterprise Architecture ใน Angular

## Enterprise Patterns คืออะไร

สถาปัตยกรรมสำหรับแอปขนาดใหญ่ ทีมใหญ่ ที่ต้องการ maintainability, scalability, testability

---

## 1. Domain-Driven Design (DDD) Structure

```
src/
├── core/                    ← Singleton services, guards, interceptors
│   ├── auth/
│   ├── http/
│   ├── error-handling/
│   └── core.module.ts
├── shared/                  ← Reusable components, pipes, directives
│   ├── components/
│   ├── directives/
│   ├── pipes/
│   └── shared.module.ts
├── features/                ← Feature modules (lazy loaded)
│   ├── products/
│   │   ├── data/            ← API, models, mappers
│   │   ├── domain/          ← Business logic, use cases
│   │   ├── presentation/    ← Components, containers, presenters
│   │   └── products.module.ts
│   ├── orders/
│   └── users/
└── layout/                  ← App shell components
    ├── header/
    ├── sidebar/
    └── footer/
```

---

## 2. Core Module

```typescript
// core/core.module.ts
import { NgModule, Optional, SkipSelf } from '@angular/core';
import { HTTP_INTERCEPTORS, HttpClientModule } from '@angular/common/http';
import { AuthInterceptor } from './http/auth.interceptor';
import { ErrorInterceptor } from './http/error.interceptor';
import { LoadingInterceptor } from './http/loading.interceptor';

@NgModule({
  imports: [HttpClientModule],
  providers: [
    { provide: HTTP_INTERCEPTORS, useClass: AuthInterceptor, multi: true },
    { provide: HTTP_INTERCEPTORS, useClass: ErrorInterceptor, multi: true },
    { provide: HTTP_INTERCEPTORS, useClass: LoadingInterceptor, multi: true }
  ]
})
export class CoreModule {
  constructor(@Optional() @SkipSelf() parentModule: CoreModule) {
    if (parentModule) {
      throw new Error('CoreModule ต้อง import ใน AppModule เท่านั้น!');
    }
  }
}
```

---

## 3. Repository Pattern

```typescript
// features/products/data/product.repository.ts
import { Injectable } from '@angular/core';
import { HttpClient, HttpParams } from '@angular/common/http';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';
import { Product, ProductFilter } from '../domain/product.model';
import { ProductApiDto } from './product.dto';
import { ProductMapper } from './product.mapper';

export abstract class ProductRepository {
  abstract getAll(filter?: ProductFilter): Observable<Product[]>;
  abstract getById(id: string): Observable<Product>;
  abstract create(product: Omit<Product, 'id'>): Observable<Product>;
  abstract update(id: string, product: Partial<Product>): Observable<Product>;
  abstract delete(id: string): Observable<void>;
}

@Injectable()
export class HttpProductRepository implements ProductRepository {
  private readonly baseUrl = '/api/v1/products';

  constructor(
    private http: HttpClient,
    private mapper: ProductMapper
  ) {}

  getAll(filter?: ProductFilter): Observable<Product[]> {
    let params = new HttpParams();
    
    if (filter?.category) params = params.set('category', filter.category);
    if (filter?.minPrice) params = params.set('minPrice', filter.minPrice.toString());
    if (filter?.maxPrice) params = params.set('maxPrice', filter.maxPrice.toString());
    if (filter?.search) params = params.set('q', filter.search);
    if (filter?.page) params = params.set('page', filter.page.toString());
    if (filter?.limit) params = params.set('limit', filter.limit.toString());

    return this.http.get<ProductApiDto[]>(this.baseUrl, { params }).pipe(
      map(dtos => dtos.map(dto => this.mapper.toDomain(dto)))
    );
  }

  getById(id: string): Observable<Product> {
    return this.http.get<ProductApiDto>(`${this.baseUrl}/${id}`).pipe(
      map(dto => this.mapper.toDomain(dto))
    );
  }

  create(product: Omit<Product, 'id'>): Observable<Product> {
    const dto = this.mapper.toCreateDto(product);
    return this.http.post<ProductApiDto>(this.baseUrl, dto).pipe(
      map(dto => this.mapper.toDomain(dto))
    );
  }

  update(id: string, product: Partial<Product>): Observable<Product> {
    const dto = this.mapper.toUpdateDto(product);
    return this.http.patch<ProductApiDto>(`${this.baseUrl}/${id}`, dto).pipe(
      map(dto => this.mapper.toDomain(dto))
    );
  }

  delete(id: string): Observable<void> {
    return this.http.delete<void>(`${this.baseUrl}/${id}`);
  }
}
```

---

## 4. Domain Model

```typescript
// features/products/domain/product.model.ts

export interface Product {
  id: string;
  name: string;
  description: string;
  price: Money;
  category: ProductCategory;
  inventory: Inventory;
  images: ProductImage[];
  status: ProductStatus;
  createdAt: Date;
  updatedAt: Date;
}

export interface Money {
  amount: number;
  currency: Currency;
}

export type Currency = 'THB' | 'USD' | 'EUR';

export interface ProductCategory {
  id: string;
  name: string;
  slug: string;
  parentId?: string;
}

export interface Inventory {
  quantity: number;
  reserved: number;
  location?: string;
}

export interface ProductImage {
  url: string;
  alt: string;
  isPrimary: boolean;
  order: number;
}

export type ProductStatus = 'active' | 'inactive' | 'draft' | 'archived';

export interface ProductFilter {
  category?: string;
  minPrice?: number;
  maxPrice?: number;
  search?: string;
  status?: ProductStatus;
  page?: number;
  limit?: number;
}

// Value Object
export class MoneyVO {
  constructor(
    private readonly amount: number,
    private readonly currency: Currency
  ) {
    if (amount < 0) throw new Error('Amount ต้องไม่ติดลบ');
  }

  add(other: MoneyVO): MoneyVO {
    if (this.currency !== other.currency) {
      throw new Error('ไม่สามารถรวม currency ต่างกันได้');
    }
    return new MoneyVO(this.amount + other.amount, this.currency);
  }

  format(): string {
    return new Intl.NumberFormat('th-TH', {
      style: 'currency',
      currency: this.currency
    }).format(this.amount);
  }

  toPlain(): Money {
    return { amount: this.amount, currency: this.currency };
  }
}
```

---

## 5. Use Cases

```typescript
// features/products/domain/use-cases/get-products.use-case.ts
import { Injectable } from '@angular/core';
import { Observable } from 'rxjs';
import { map, shareReplay } from 'rxjs/operators';
import { ProductRepository } from '../data/product.repository';
import { Product, ProductFilter } from './product.model';

@Injectable()
export class GetProductsUseCase {
  constructor(private productRepo: ProductRepository) {}

  execute(filter?: ProductFilter): Observable<Product[]> {
    return this.productRepo.getAll(filter).pipe(
      map(products => this.sortByRelevance(products, filter?.search)),
      shareReplay(1)
    );
  }

  private sortByRelevance(products: Product[], search?: string): Product[] {
    if (!search) return products;
    
    return [...products].sort((a, b) => {
      const scoreA = this.relevanceScore(a, search);
      const scoreB = this.relevanceScore(b, search);
      return scoreB - scoreA;
    });
  }

  private relevanceScore(product: Product, search: string): number {
    const term = search.toLowerCase();
    let score = 0;
    
    if (product.name.toLowerCase().startsWith(term)) score += 3;
    if (product.name.toLowerCase().includes(term)) score += 2;
    if (product.description.toLowerCase().includes(term)) score += 1;
    
    return score;
  }
}

// features/products/domain/use-cases/create-product.use-case.ts
@Injectable()
export class CreateProductUseCase {
  constructor(
    private productRepo: ProductRepository,
    private eventBus: EventBusService
  ) {}

  async execute(productData: Omit<Product, 'id' | 'createdAt' | 'updatedAt'>): Promise<Product> {
    // Validation
    this.validate(productData);
    
    // Create
    const product = await this.productRepo.create(productData).toPromise();
    
    // Publish event
    if (product) {
      this.eventBus.publish({
        type: 'PRODUCT_CREATED',
        payload: product,
        timestamp: new Date()
      });
    }
    
    return product!;
  }

  private validate(data: any): void {
    if (!data.name?.trim()) throw new Error('ชื่อสินค้าจำเป็นต้องกรอก');
    if (data.price?.amount < 0) throw new Error('ราคาต้องไม่ติดลบ');
    if (data.inventory?.quantity < 0) throw new Error('จำนวนต้องไม่ติดลบ');
  }
}
```

---

## 6. Facade Pattern

```typescript
// features/products/product.facade.ts
import { Injectable } from '@angular/core';
import { Observable } from 'rxjs';
import { GetProductsUseCase } from './domain/use-cases/get-products.use-case';
import { CreateProductUseCase } from './domain/use-cases/create-product.use-case';
import { ProductStateService } from './state/product.state';
import { Product, ProductFilter } from './domain/product.model';

// Facade เป็น single entry point สำหรับ feature
@Injectable()
export class ProductFacade {
  products$ = this.state.products$;
  loading$ = this.state.loading$;
  error$ = this.state.error$;
  selectedProduct$ = this.state.selectedProduct$;

  constructor(
    private getProductsUseCase: GetProductsUseCase,
    private createProductUseCase: CreateProductUseCase,
    private state: ProductStateService
  ) {}

  loadProducts(filter?: ProductFilter): void {
    this.state.setLoading(true);
    
    this.getProductsUseCase.execute(filter).subscribe({
      next: products => {
        this.state.setProducts(products);
        this.state.setLoading(false);
      },
      error: err => {
        this.state.setError(err.message);
        this.state.setLoading(false);
      }
    });
  }

  async createProduct(data: Omit<Product, 'id' | 'createdAt' | 'updatedAt'>): Promise<void> {
    this.state.setLoading(true);
    
    try {
      const product = await this.createProductUseCase.execute(data);
      this.state.addProduct(product);
    } catch (error: any) {
      this.state.setError(error.message);
    } finally {
      this.state.setLoading(false);
    }
  }

  selectProduct(id: string): void {
    this.state.setSelectedProduct(id);
  }

  clearError(): void {
    this.state.setError(null);
  }
}
```

---

## 7. Product Module Composition

```typescript
// features/products/products.module.ts
import { NgModule } from '@angular/core';
import { CommonModule } from '@angular/common';
import { RouterModule } from '@angular/router';
import { ProductRepository, HttpProductRepository } from './data/product.repository';
import { ProductMapper } from './data/product.mapper';
import { GetProductsUseCase } from './domain/use-cases/get-products.use-case';
import { CreateProductUseCase } from './domain/use-cases/create-product.use-case';
import { ProductStateService } from './state/product.state';
import { ProductFacade } from './product.facade';
import { ProductListComponent } from './presentation/list/product-list.component';
import { ProductDetailComponent } from './presentation/detail/product-detail.component';

@NgModule({
  declarations: [ProductListComponent, ProductDetailComponent],
  imports: [
    CommonModule,
    RouterModule.forChild([
      { path: '', component: ProductListComponent },
      { path: ':id', component: ProductDetailComponent }
    ])
  ],
  providers: [
    // Data layer
    ProductMapper,
    { provide: ProductRepository, useClass: HttpProductRepository },
    
    // Domain layer
    GetProductsUseCase,
    CreateProductUseCase,
    
    // State
    ProductStateService,
    
    // Facade
    ProductFacade
  ]
})
export class ProductsModule {}
```

---

## สรุป

| Layer | ความรับผิดชอบ |
|-------|-------------|
| Presentation | Components, UI, UX |
| Domain | Business logic, Use Cases |
| Data | API calls, Mappers, DTOs |
| Core | Cross-cutting concerns |
| Shared | Reusable UI components |

### Principles

1. **Single Responsibility** - แต่ละ class มีหน้าที่เดียว
2. **Dependency Inversion** - Depend on abstractions ไม่ใช่ implementations
3. **Open/Closed** - Open for extension, Closed for modification
4. **Separation of Concerns** - แยก business logic จาก UI
5. **Testability** - ง่ายต่อการ test แต่ละ layer
