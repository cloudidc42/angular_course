# Part 83: CQRS Pattern ใน Frontend Angular

## CQRS คืออะไร

**Command Query Responsibility Segregation** - แยก operations ที่อ่านข้อมูล (Query) ออกจาก operations ที่เปลี่ยนแปลงข้อมูล (Command)

---

## 1. โครงสร้าง CQRS

```typescript
// core/cqrs/command.ts
export interface Command {
  readonly type: string;
}

export interface CommandResult<T = void> {
  success: boolean;
  data?: T;
  error?: string;
}

export interface CommandHandler<TCommand extends Command, TResult = void> {
  execute(command: TCommand): Promise<CommandResult<TResult>>;
}
```

```typescript
// core/cqrs/query.ts
export interface Query<TResult> {
  readonly type: string;
}

export interface QueryHandler<TQuery extends Query<TResult>, TResult> {
  execute(query: TQuery): Observable<TResult>;
}
```

---

## 2. Command Bus

```typescript
// core/cqrs/command-bus.service.ts
import { Injectable, Type } from '@angular/core';
import { Command, CommandHandler, CommandResult } from './command';

@Injectable({ providedIn: 'root' })
export class CommandBus {
  private handlers = new Map<string, CommandHandler<any, any>>();

  register<T extends Command>(
    commandType: string, 
    handler: CommandHandler<T, any>
  ): void {
    this.handlers.set(commandType, handler);
  }

  async execute<TResult = void>(command: Command): Promise<CommandResult<TResult>> {
    const handler = this.handlers.get(command.type);
    
    if (!handler) {
      return {
        success: false,
        error: `ไม่พบ handler สำหรับ command: ${command.type}`
      };
    }

    try {
      const result = await handler.execute(command);
      return result;
    } catch (error: any) {
      return {
        success: false,
        error: error.message || 'เกิดข้อผิดพลาดที่ไม่ทราบสาเหตุ'
      };
    }
  }
}
```

---

## 3. Query Bus

```typescript
// core/cqrs/query-bus.service.ts
import { Injectable } from '@angular/core';
import { Observable, throwError } from 'rxjs';
import { Query, QueryHandler } from './query';

@Injectable({ providedIn: 'root' })
export class QueryBus {
  private handlers = new Map<string, QueryHandler<any, any>>();

  register<TQuery extends Query<TResult>, TResult>(
    queryType: string,
    handler: QueryHandler<TQuery, TResult>
  ): void {
    this.handlers.set(queryType, handler);
  }

  execute<TResult>(query: Query<TResult>): Observable<TResult> {
    const handler = this.handlers.get(query.type);
    
    if (!handler) {
      return throwError(() => new Error(`ไม่พบ handler สำหรับ query: ${query.type}`));
    }

    return handler.execute(query);
  }
}
```

---

## 4. Product Commands

```typescript
// features/products/commands/create-product.command.ts
import { Command, CommandResult } from '../../../core/cqrs/command';

export interface CreateProductCommand extends Command {
  type: 'CREATE_PRODUCT';
  payload: {
    name: string;
    price: number;
    category: string;
    description: string;
    stock: number;
  };
}

export interface UpdateProductCommand extends Command {
  type: 'UPDATE_PRODUCT';
  payload: {
    id: string;
    updates: Partial<{
      name: string;
      price: number;
      stock: number;
      status: string;
    }>;
  };
}

export interface DeleteProductCommand extends Command {
  type: 'DELETE_PRODUCT';
  payload: {
    id: string;
    reason?: string;
  };
}

export interface AddToCartCommand extends Command {
  type: 'ADD_TO_CART';
  payload: {
    productId: string;
    quantity: number;
  };
}
```

---

## 5. Command Handlers

```typescript
// features/products/commands/create-product.handler.ts
import { Injectable } from '@angular/core';
import { CommandHandler, CommandResult } from '../../../core/cqrs/command';
import { CreateProductCommand } from './create-product.command';
import { ProductRepository } from '../data/product.repository';
import { EventBusService } from '../../../core/event-bus/event-bus.service';
import { DomainEvents } from '../../../core/event-bus/domain-events';

@Injectable()
export class CreateProductCommandHandler 
  implements CommandHandler<CreateProductCommand, { id: string }> {
  
  constructor(
    private productRepo: ProductRepository,
    private eventBus: EventBusService
  ) {}

  async execute(command: CreateProductCommand): Promise<CommandResult<{ id: string }>> {
    const { payload } = command;

    // Validation
    if (!payload.name?.trim()) {
      return { success: false, error: 'ชื่อสินค้าจำเป็นต้องกรอก' };
    }

    if (payload.price < 0) {
      return { success: false, error: 'ราคาต้องไม่ติดลบ' };
    }

    try {
      const product = await this.productRepo.create({
        name: payload.name,
        price: { amount: payload.price, currency: 'THB' },
        category: { id: payload.category, name: '', slug: '' },
        description: payload.description,
        inventory: { quantity: payload.stock, reserved: 0 },
        images: [],
        status: 'active'
      }).toPromise();

      if (!product) throw new Error('ไม่สามารถสร้างสินค้าได้');

      // Publish domain event
      this.eventBus.publish({
        type: DomainEvents.PRODUCT_CREATED,
        payload: product,
        source: 'CreateProductCommandHandler'
      });

      return { success: true, data: { id: product.id } };
    } catch (error: any) {
      return { success: false, error: error.message };
    }
  }
}

// features/products/commands/add-to-cart.handler.ts
@Injectable()
export class AddToCartCommandHandler implements CommandHandler<AddToCartCommand> {
  constructor(
    private cartService: CartService,
    private productRepo: ProductRepository
  ) {}

  async execute(command: AddToCartCommand): Promise<CommandResult> {
    const { productId, quantity } = command.payload;

    if (quantity <= 0) {
      return { success: false, error: 'จำนวนต้องมากกว่า 0' };
    }

    try {
      const product = await this.productRepo.getById(productId).toPromise();
      if (!product) return { success: false, error: 'ไม่พบสินค้า' };
      
      if (product.inventory.quantity < quantity) {
        return { success: false, error: 'สินค้าไม่เพียงพอ' };
      }

      this.cartService.addItem({
        id: product.id,
        name: product.name,
        price: product.price.amount
      }, quantity);

      return { success: true };
    } catch (error: any) {
      return { success: false, error: error.message };
    }
  }
}
```

---

## 6. Product Queries

```typescript
// features/products/queries/get-products.query.ts
import { Query } from '../../../core/cqrs/query';

export interface GetProductsQuery extends Query<Product[]> {
  type: 'GET_PRODUCTS';
  filter?: {
    category?: string;
    minPrice?: number;
    maxPrice?: number;
    search?: string;
    page?: number;
    limit?: number;
  };
}

export interface GetProductByIdQuery extends Query<Product | null> {
  type: 'GET_PRODUCT_BY_ID';
  id: string;
}

export interface GetProductStatsQuery extends Query<ProductStats> {
  type: 'GET_PRODUCT_STATS';
  category?: string;
}

interface ProductStats {
  totalProducts: number;
  totalValue: number;
  lowStockCount: number;
  categoryBreakdown: { category: string; count: number }[];
}
```

---

## 7. Query Handlers

```typescript
// features/products/queries/get-products.handler.ts
import { Injectable } from '@angular/core';
import { Observable } from 'rxjs';
import { map, shareReplay } from 'rxjs/operators';
import { QueryHandler } from '../../../core/cqrs/query';
import { GetProductsQuery } from './get-products.query';
import { ProductRepository } from '../data/product.repository';

@Injectable()
export class GetProductsQueryHandler 
  implements QueryHandler<GetProductsQuery, Product[]> {
  
  constructor(private productRepo: ProductRepository) {}

  execute(query: GetProductsQuery): Observable<Product[]> {
    return this.productRepo.getAll(query.filter).pipe(
      map(products => {
        if (query.filter?.search) {
          return this.applySearch(products, query.filter.search);
        }
        return products;
      }),
      shareReplay(1)
    );
  }

  private applySearch(products: Product[], search: string): Product[] {
    const term = search.toLowerCase();
    return products.filter(p => 
      p.name.toLowerCase().includes(term) || 
      p.description.toLowerCase().includes(term)
    );
  }
}

// features/products/queries/get-product-stats.handler.ts
@Injectable()
export class GetProductStatsQueryHandler 
  implements QueryHandler<GetProductStatsQuery, ProductStats> {
  
  constructor(private productRepo: ProductRepository) {}

  execute(query: GetProductStatsQuery): Observable<ProductStats> {
    return this.productRepo.getAll({ category: query.category }).pipe(
      map(products => ({
        totalProducts: products.length,
        totalValue: products.reduce((sum, p) => sum + (p.price.amount * p.inventory.quantity), 0),
        lowStockCount: products.filter(p => p.inventory.quantity < 10).length,
        categoryBreakdown: this.groupByCategory(products)
      }))
    );
  }

  private groupByCategory(products: Product[]): { category: string; count: number }[] {
    const groups: Record<string, number> = {};
    products.forEach(p => {
      groups[p.category.name] = (groups[p.category.name] || 0) + 1;
    });
    return Object.entries(groups).map(([category, count]) => ({ category, count }));
  }
}
```

---

## 8. Component ใช้งาน CQRS

```typescript
// features/products/presentation/product-manager.component.ts
import { Component, OnInit } from '@angular/core';
import { Observable } from 'rxjs';
import { CommandBus } from '../../../core/cqrs/command-bus.service';
import { QueryBus } from '../../../core/cqrs/query-bus.service';
import { CreateProductCommand, AddToCartCommand } from '../commands';
import { GetProductsQuery, GetProductStatsQuery } from '../queries';

@Component({
  selector: 'app-product-manager',
  template: `
    <div class="product-manager">
      <!-- Stats (Query Result) -->
      <div class="stats-panel" *ngIf="stats$ | async as stats">
        <div class="stat">
          <h3>{{ stats.totalProducts }}</h3>
          <p>สินค้าทั้งหมด</p>
        </div>
        <div class="stat">
          <h3>{{ stats.totalValue | currency:'THB':'symbol':'1.0-0' }}</h3>
          <p>มูลค่ารวม</p>
        </div>
        <div class="stat warn" *ngIf="stats.lowStockCount > 0">
          <h3>{{ stats.lowStockCount }}</h3>
          <p>สินค้าใกล้หมด</p>
        </div>
      </div>

      <!-- Create Product Form (Command) -->
      <form (ngSubmit)="createProduct()" [formGroup]="createForm">
        <h3>เพิ่มสินค้าใหม่</h3>
        <input formControlName="name" placeholder="ชื่อสินค้า">
        <input formControlName="price" type="number" placeholder="ราคา">
        <input formControlName="stock" type="number" placeholder="จำนวน">
        <button type="submit" [disabled]="isCreating">
          {{ isCreating ? 'กำลังสร้าง...' : 'สร้างสินค้า' }}
        </button>
        <p class="error" *ngIf="createError">{{ createError }}</p>
        <p class="success" *ngIf="createSuccess">{{ createSuccess }}</p>
      </form>

      <!-- Product List (Query Result) -->
      <div class="products" *ngIf="products$ | async as products">
        <div *ngFor="let product of products" class="product-card">
          <h4>{{ product.name }}</h4>
          <p>{{ product.price.amount | currency:'THB' }}</p>
          <p>คงเหลือ: {{ product.inventory.quantity }}</p>
          <button (click)="addToCart(product.id)">เพิ่มในตะกร้า</button>
        </div>
      </div>
    </div>
  `
})
export class ProductManagerComponent implements OnInit {
  products$!: Observable<any[]>;
  stats$!: Observable<any>;
  isCreating = false;
  createError = '';
  createSuccess = '';

  createForm = this.fb.group({
    name: [''],
    price: [0],
    stock: [0],
    category: ['general'],
    description: ['']
  });

  constructor(
    private commandBus: CommandBus,
    private queryBus: QueryBus,
    private fb: FormBuilder
  ) {}

  ngOnInit(): void {
    // Execute query - รับ Observable
    const query: GetProductsQuery = { type: 'GET_PRODUCTS' };
    this.products$ = this.queryBus.execute(query);

    const statsQuery: GetProductStatsQuery = { type: 'GET_PRODUCT_STATS' };
    this.stats$ = this.queryBus.execute(statsQuery);
  }

  async createProduct(): Promise<void> {
    this.isCreating = true;
    this.createError = '';
    this.createSuccess = '';

    const command: CreateProductCommand = {
      type: 'CREATE_PRODUCT',
      payload: this.createForm.value as any
    };

    const result = await this.commandBus.execute(command);
    
    if (result.success) {
      this.createSuccess = `สร้างสินค้าสำเร็จ! ID: ${result.data?.id}`;
      this.createForm.reset();
    } else {
      this.createError = result.error || 'เกิดข้อผิดพลาด';
    }

    this.isCreating = false;
  }

  async addToCart(productId: string): Promise<void> {
    const command: AddToCartCommand = {
      type: 'ADD_TO_CART',
      payload: { productId, quantity: 1 }
    };

    const result = await this.commandBus.execute(command);
    if (!result.success) {
      alert(result.error);
    }
  }
}
```

---

## สรุป

| ส่วน | หน้าที่ |
|------|---------|
| Command | เปลี่ยนแปลง state |
| Query | อ่านข้อมูล |
| CommandBus | Route commands → handlers |
| QueryBus | Route queries → handlers |
| Handler | ทำงานจริง |

### ข้อดีของ CQRS

1. **Scalability**: Query side scale แยกจาก Command side
2. **Performance**: Query model optimize สำหรับ read
3. **Testability**: Test handlers แยกได้
4. **Single Responsibility**: แต่ละ handler มีหน้าที่เดียว
5. **Auditability**: ทุก command ถูกบันทึก
