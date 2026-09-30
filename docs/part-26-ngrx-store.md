# Part 26 — NgRx Store

## NgRx คืออะไร?

NgRx เป็น State Management Library สำหรับ Angular ที่ได้รับแรงบันดาลใจจาก Redux Pattern โดยใช้ Reactive Extensions (RxJS) เป็นพื้นฐาน ช่วยจัดการ State ของแอปพลิเคชันให้มีระเบียบและคาดเดาได้

### ทำไมต้องใช้ NgRx?

- **Single Source of Truth** — State ทั้งหมดอยู่ในที่เดียว
- **Predictable State Changes** — State เปลี่ยนได้ผ่าน Actions เท่านั้น
- **Time-travel Debugging** — ย้อนดู State ในแต่ละช่วงเวลาได้
- **Performance** — ใช้ Memoization ลดการคำนวณซ้ำ
- **Testability** — ทดสอบได้ง่ายเพราะ Pure Functions

---

## Core Concepts

### 1. Store

Store คือ Single State Tree ของแอปพลิเคชัน ทุก Component อ่านข้อมูลจาก Store ผ่าน Selectors

```
                    ┌─────────────┐
                    │   Component │
                    └──────┬──────┘
                           │ dispatch(action)
                    ┌──────▼──────┐
                    │   Actions   │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │   Reducers  │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │    Store    │◄── Selectors ── Component
                    └─────────────┘
```

### 2. Actions

Actions คือ Events ที่บอก Store ว่าเกิดอะไรขึ้น ประกอบด้วย `type` และ optional `props`

```typescript
// การสร้าง Action ด้วย createAction
import { createAction, props } from '@ngrx/store';

// Action ไม่มี payload
export const increment = createAction('[Counter] Increment');
export const decrement = createAction('[Counter] Decrement');
export const reset = createAction('[Counter] Reset');

// Action มี payload
export const incrementBy = createAction(
  '[Counter] Increment By',
  props<{ amount: number }>()
);

// Action สำหรับ API
export const loadProducts = createAction('[Product] Load Products');
export const loadProductsSuccess = createAction(
  '[Product] Load Products Success',
  props<{ products: Product[] }>()
);
export const loadProductsFailure = createAction(
  '[Product] Load Products Failure',
  props<{ error: string }>()
);
```

### 3. Reducers

Reducers คือ Pure Functions ที่รับ State เดิมกับ Action แล้วคืน State ใหม่

```typescript
import { createReducer, on } from '@ngrx/store';
import { increment, decrement, reset, incrementBy } from './counter.actions';

// กำหนด Interface ของ State
export interface CounterState {
  count: number;
}

// กำหนดค่าเริ่มต้น
export const initialState: CounterState = {
  count: 0,
};

// สร้าง Reducer
export const counterReducer = createReducer(
  initialState,
  on(increment, (state) => ({ ...state, count: state.count + 1 })),
  on(decrement, (state) => ({ ...state, count: state.count - 1 })),
  on(reset, (state) => ({ ...state, count: 0 })),
  on(incrementBy, (state, { amount }) => ({
    ...state,
    count: state.count + amount,
  }))
);
```

### 4. Selectors

Selectors คือ Functions ที่ดึงข้อมูลจาก Store

```typescript
import { createSelector, createFeatureSelector } from '@ngrx/store';
import { CounterState } from './counter.reducer';

// Feature Selector
export const selectCounterState =
  createFeatureSelector<CounterState>('counter');

// Selector
export const selectCount = createSelector(
  selectCounterState,
  (state) => state.count
);

// Derived Selector
export const selectIsPositive = createSelector(
  selectCount,
  (count) => count > 0
);
```

---

## การติดตั้ง NgRx

```bash
# ติดตั้ง NgRx Store
npm install @ngrx/store

# ติดตั้งพร้อม Schematics (แนะนำ)
ng add @ngrx/store

# ติดตั้ง DevTools
npm install @ngrx/store-devtools
```

---

## สร้าง Counter Store แบบสมบูรณ์

### โครงสร้างไฟล์

```
src/
└── app/
    └── store/
        └── counter/
            ├── counter.actions.ts
            ├── counter.reducer.ts
            ├── counter.selectors.ts
            └── counter.state.ts
```

### Step 1: กำหนด State Interface

```typescript
// counter.state.ts
export interface CounterState {
  count: number;
  history: number[];
  lastUpdated: Date | null;
}
```

### Step 2: สร้าง Actions

```typescript
// counter.actions.ts
import { createAction, props } from '@ngrx/store';

export const increment = createAction('[Counter] Increment');
export const decrement = createAction('[Counter] Decrement');
export const reset = createAction('[Counter] Reset');
export const setCount = createAction(
  '[Counter] Set Count',
  props<{ value: number }>()
);
export const undo = createAction('[Counter] Undo');
```

### Step 3: สร้าง Reducer

```typescript
// counter.reducer.ts
import { createReducer, on } from '@ngrx/store';
import * as CounterActions from './counter.actions';
import { CounterState } from './counter.state';

export const initialState: CounterState = {
  count: 0,
  history: [],
  lastUpdated: null,
};

export const counterReducer = createReducer(
  initialState,

  on(CounterActions.increment, (state) => ({
    ...state,
    count: state.count + 1,
    history: [...state.history, state.count],
    lastUpdated: new Date(),
  })),

  on(CounterActions.decrement, (state) => ({
    ...state,
    count: state.count - 1,
    history: [...state.history, state.count],
    lastUpdated: new Date(),
  })),

  on(CounterActions.reset, (state) => ({
    ...initialState,
    lastUpdated: new Date(),
  })),

  on(CounterActions.setCount, (state, { value }) => ({
    ...state,
    count: value,
    history: [...state.history, state.count],
    lastUpdated: new Date(),
  })),

  on(CounterActions.undo, (state) => {
    if (state.history.length === 0) return state;
    const previousCount = state.history[state.history.length - 1];
    return {
      ...state,
      count: previousCount,
      history: state.history.slice(0, -1),
      lastUpdated: new Date(),
    };
  })
);
```

### Step 4: สร้าง Selectors

```typescript
// counter.selectors.ts
import { createSelector, createFeatureSelector } from '@ngrx/store';
import { CounterState } from './counter.state';

export const selectCounterState =
  createFeatureSelector<CounterState>('counter');

export const selectCount = createSelector(
  selectCounterState,
  (state) => state.count
);

export const selectHistory = createSelector(
  selectCounterState,
  (state) => state.history
);

export const selectLastUpdated = createSelector(
  selectCounterState,
  (state) => state.lastUpdated
);

export const selectCanUndo = createSelector(
  selectHistory,
  (history) => history.length > 0
);

export const selectIsNegative = createSelector(
  selectCount,
  (count) => count < 0
);

export const selectAbsoluteCount = createSelector(
  selectCount,
  (count) => Math.abs(count)
);
```

### Step 5: ลงทะเบียน Store ใน AppModule

```typescript
// app.module.ts
import { NgModule } from '@angular/core';
import { BrowserModule } from '@angular/platform-browser';
import { StoreModule } from '@ngrx/store';
import { StoreDevtoolsModule } from '@ngrx/store-devtools';
import { counterReducer } from './store/counter/counter.reducer';
import { environment } from '../environments/environment';

@NgModule({
  imports: [
    BrowserModule,
    StoreModule.forRoot({
      counter: counterReducer,
    }),
    StoreDevtoolsModule.instrument({
      maxAge: 25,
      logOnly: environment.production,
    }),
  ],
  // ...
})
export class AppModule {}
```

### Step 6: ใช้งานใน Component

```typescript
// counter.component.ts
import { Component, OnInit } from '@angular/core';
import { Store } from '@ngrx/store';
import { Observable } from 'rxjs';
import * as CounterActions from '../store/counter/counter.actions';
import * as CounterSelectors from '../store/counter/counter.selectors';

@Component({
  selector: 'app-counter',
  template: `
    <div class="counter-container">
      <h2>NgRx Counter</h2>

      <div class="count-display" [class.negative]="isNegative$ | async">
        {{ count$ | async }}
      </div>

      <div class="controls">
        <button (click)="increment()">+</button>
        <button (click)="decrement()">-</button>
        <button (click)="reset()">Reset</button>
        <button (click)="undo()" [disabled]="!(canUndo$ | async)">Undo</button>
      </div>

      <div class="set-count">
        <input #countInput type="number" placeholder="Set value" />
        <button (click)="setCount(countInput.value)">Set</button>
      </div>

      <div class="history" *ngIf="(history$ | async)?.length">
        <h4>History:</h4>
        <span *ngFor="let h of history$ | async">{{ h }} → </span>
        <strong>{{ count$ | async }}</strong>
      </div>

      <div class="last-updated" *ngIf="lastUpdated$ | async">
        Last updated: {{ lastUpdated$ | async | date:'medium' }}
      </div>
    </div>
  `,
  styles: [`
    .counter-container { text-align: center; padding: 20px; }
    .count-display {
      font-size: 4rem;
      font-weight: bold;
      color: #333;
      padding: 20px;
    }
    .count-display.negative { color: red; }
    button { margin: 5px; padding: 8px 16px; cursor: pointer; }
    button:disabled { opacity: 0.5; cursor: not-allowed; }
  `]
})
export class CounterComponent implements OnInit {
  count$: Observable<number>;
  history$: Observable<number[]>;
  lastUpdated$: Observable<Date | null>;
  canUndo$: Observable<boolean>;
  isNegative$: Observable<boolean>;

  constructor(private store: Store) {}

  ngOnInit(): void {
    this.count$ = this.store.select(CounterSelectors.selectCount);
    this.history$ = this.store.select(CounterSelectors.selectHistory);
    this.lastUpdated$ = this.store.select(CounterSelectors.selectLastUpdated);
    this.canUndo$ = this.store.select(CounterSelectors.selectCanUndo);
    this.isNegative$ = this.store.select(CounterSelectors.selectIsNegative);
  }

  increment(): void {
    this.store.dispatch(CounterActions.increment());
  }

  decrement(): void {
    this.store.dispatch(CounterActions.decrement());
  }

  reset(): void {
    this.store.dispatch(CounterActions.reset());
  }

  undo(): void {
    this.store.dispatch(CounterActions.undo());
  }

  setCount(value: string): void {
    const numValue = parseInt(value, 10);
    if (!isNaN(numValue)) {
      this.store.dispatch(CounterActions.setCount({ value: numValue }));
    }
  }
}
```

---

## Workshop: Product State Management

ในส่วนนี้เราจะสร้าง Product State Management ที่ครบถ้วน

### State Interface

```typescript
// product.state.ts
export interface Product {
  id: number;
  name: string;
  price: number;
  category: string;
  stock: number;
  description: string;
}

export interface ProductState {
  products: Product[];
  selectedProductId: number | null;
  loading: boolean;
  error: string | null;
  filter: {
    category: string | null;
    minPrice: number | null;
    maxPrice: number | null;
    searchTerm: string;
  };
}
```

### Actions

```typescript
// product.actions.ts
import { createAction, props } from '@ngrx/store';
import { Product } from './product.state';

// Load
export const loadProducts = createAction('[Product] Load Products');
export const loadProductsSuccess = createAction(
  '[Product] Load Products Success',
  props<{ products: Product[] }>()
);
export const loadProductsFailure = createAction(
  '[Product] Load Products Failure',
  props<{ error: string }>()
);

// CRUD
export const addProduct = createAction(
  '[Product] Add Product',
  props<{ product: Omit<Product, 'id'> }>()
);
export const updateProduct = createAction(
  '[Product] Update Product',
  props<{ product: Product }>()
);
export const deleteProduct = createAction(
  '[Product] Delete Product',
  props<{ id: number }>()
);

// Selection
export const selectProduct = createAction(
  '[Product] Select Product',
  props<{ id: number }>()
);
export const clearSelectedProduct = createAction(
  '[Product] Clear Selected Product'
);

// Filter
export const setFilter = createAction(
  '[Product] Set Filter',
  props<{ filter: Partial<ProductState['filter']> }>()
);
export const clearFilter = createAction('[Product] Clear Filter');
```

### Reducer

```typescript
// product.reducer.ts
import { createReducer, on } from '@ngrx/store';
import * as ProductActions from './product.actions';
import { ProductState } from './product.state';

export const initialState: ProductState = {
  products: [],
  selectedProductId: null,
  loading: false,
  error: null,
  filter: {
    category: null,
    minPrice: null,
    maxPrice: null,
    searchTerm: '',
  },
};

let nextId = 1;

export const productReducer = createReducer(
  initialState,

  // Load
  on(ProductActions.loadProducts, (state) => ({
    ...state,
    loading: true,
    error: null,
  })),

  on(ProductActions.loadProductsSuccess, (state, { products }) => ({
    ...state,
    products,
    loading: false,
  })),

  on(ProductActions.loadProductsFailure, (state, { error }) => ({
    ...state,
    loading: false,
    error,
  })),

  // Add
  on(ProductActions.addProduct, (state, { product }) => ({
    ...state,
    products: [
      ...state.products,
      { ...product, id: nextId++ },
    ],
  })),

  // Update
  on(ProductActions.updateProduct, (state, { product }) => ({
    ...state,
    products: state.products.map((p) =>
      p.id === product.id ? product : p
    ),
  })),

  // Delete
  on(ProductActions.deleteProduct, (state, { id }) => ({
    ...state,
    products: state.products.filter((p) => p.id !== id),
    selectedProductId:
      state.selectedProductId === id ? null : state.selectedProductId,
  })),

  // Selection
  on(ProductActions.selectProduct, (state, { id }) => ({
    ...state,
    selectedProductId: id,
  })),

  on(ProductActions.clearSelectedProduct, (state) => ({
    ...state,
    selectedProductId: null,
  })),

  // Filter
  on(ProductActions.setFilter, (state, { filter }) => ({
    ...state,
    filter: { ...state.filter, ...filter },
  })),

  on(ProductActions.clearFilter, (state) => ({
    ...state,
    filter: initialState.filter,
  }))
);
```

### Selectors

```typescript
// product.selectors.ts
import { createSelector, createFeatureSelector } from '@ngrx/store';
import { ProductState } from './product.state';

export const selectProductState =
  createFeatureSelector<ProductState>('product');

export const selectAllProducts = createSelector(
  selectProductState,
  (state) => state.products
);

export const selectLoading = createSelector(
  selectProductState,
  (state) => state.loading
);

export const selectError = createSelector(
  selectProductState,
  (state) => state.error
);

export const selectFilter = createSelector(
  selectProductState,
  (state) => state.filter
);

export const selectSelectedProductId = createSelector(
  selectProductState,
  (state) => state.selectedProductId
);

// Derived: ดึงสินค้าที่เลือก
export const selectSelectedProduct = createSelector(
  selectAllProducts,
  selectSelectedProductId,
  (products, selectedId) =>
    selectedId ? products.find((p) => p.id === selectedId) : null
);

// Derived: กรองสินค้าตาม filter
export const selectFilteredProducts = createSelector(
  selectAllProducts,
  selectFilter,
  (products, filter) => {
    return products.filter((product) => {
      const matchCategory =
        !filter.category || product.category === filter.category;
      const matchMinPrice =
        filter.minPrice === null || product.price >= filter.minPrice;
      const matchMaxPrice =
        filter.maxPrice === null || product.price <= filter.maxPrice;
      const matchSearch =
        !filter.searchTerm ||
        product.name.toLowerCase().includes(filter.searchTerm.toLowerCase()) ||
        product.description.toLowerCase().includes(filter.searchTerm.toLowerCase());

      return matchCategory && matchMinPrice && matchMaxPrice && matchSearch;
    });
  }
);

// Derived: รายชื่อหมวดหมู่ทั้งหมด
export const selectCategories = createSelector(
  selectAllProducts,
  (products) => [...new Set(products.map((p) => p.category))]
);

// Derived: สถิติ
export const selectProductStats = createSelector(
  selectAllProducts,
  (products) => ({
    total: products.length,
    totalValue: products.reduce((sum, p) => sum + p.price * p.stock, 0),
    averagePrice:
      products.length > 0
        ? products.reduce((sum, p) => sum + p.price, 0) / products.length
        : 0,
    lowStock: products.filter((p) => p.stock < 10).length,
  })
);
```

### Product List Component

```typescript
// product-list.component.ts
import { Component, OnInit } from '@angular/core';
import { Store } from '@ngrx/store';
import { Observable } from 'rxjs';
import { Product } from '../store/product/product.state';
import * as ProductActions from '../store/product/product.actions';
import * as ProductSelectors from '../store/product/product.selectors';

@Component({
  selector: 'app-product-list',
  template: `
    <div class="product-list">
      <div class="stats" *ngIf="stats$ | async as stats">
        <div class="stat">สินค้าทั้งหมด: {{ stats.total }}</div>
        <div class="stat">มูลค่ารวม: {{ stats.totalValue | currency:'THB' }}</div>
        <div class="stat">ราคาเฉลี่ย: {{ stats.averagePrice | currency:'THB' }}</div>
        <div class="stat warn">สต็อกต่ำ: {{ stats.lowStock }} รายการ</div>
      </div>

      <div class="filter-panel">
        <input
          type="text"
          placeholder="ค้นหาสินค้า..."
          (input)="onSearch($event)"
        />
        <select (change)="onCategoryFilter($event)">
          <option value="">ทุกหมวดหมู่</option>
          <option *ngFor="let cat of categories$ | async" [value]="cat">
            {{ cat }}
          </option>
        </select>
      </div>

      <div *ngIf="loading$ | async" class="loading">กำลังโหลด...</div>
      <div *ngIf="error$ | async as error" class="error">{{ error }}</div>

      <div class="products-grid">
        <div
          *ngFor="let product of filteredProducts$ | async"
          class="product-card"
          [class.selected]="(selectedProduct$ | async)?.id === product.id"
          (click)="selectProduct(product.id)"
        >
          <h3>{{ product.name }}</h3>
          <p>{{ product.description }}</p>
          <div class="price">{{ product.price | currency:'THB' }}</div>
          <div class="stock" [class.low]="product.stock < 10">
            สต็อก: {{ product.stock }}
          </div>
          <div class="category">{{ product.category }}</div>
          <button (click)="deleteProduct(product.id, $event)">ลบ</button>
        </div>
      </div>
    </div>
  `
})
export class ProductListComponent implements OnInit {
  filteredProducts$: Observable<Product[]>;
  selectedProduct$: Observable<Product | undefined | null>;
  loading$: Observable<boolean>;
  error$: Observable<string | null>;
  categories$: Observable<string[]>;
  stats$: Observable<any>;

  constructor(private store: Store) {}

  ngOnInit(): void {
    this.filteredProducts$ = this.store.select(
      ProductSelectors.selectFilteredProducts
    );
    this.selectedProduct$ = this.store.select(
      ProductSelectors.selectSelectedProduct
    );
    this.loading$ = this.store.select(ProductSelectors.selectLoading);
    this.error$ = this.store.select(ProductSelectors.selectError);
    this.categories$ = this.store.select(ProductSelectors.selectCategories);
    this.stats$ = this.store.select(ProductSelectors.selectProductStats);

    this.store.dispatch(ProductActions.loadProducts());
  }

  selectProduct(id: number): void {
    this.store.dispatch(ProductActions.selectProduct({ id }));
  }

  deleteProduct(id: number, event: Event): void {
    event.stopPropagation();
    if (confirm('ยืนยันการลบสินค้า?')) {
      this.store.dispatch(ProductActions.deleteProduct({ id }));
    }
  }

  onSearch(event: Event): void {
    const searchTerm = (event.target as HTMLInputElement).value;
    this.store.dispatch(
      ProductActions.setFilter({ filter: { searchTerm } })
    );
  }

  onCategoryFilter(event: Event): void {
    const category = (event.target as HTMLSelectElement).value || null;
    this.store.dispatch(
      ProductActions.setFilter({ filter: { category } })
    );
  }
}
```

---

## AppState และ Feature States

เมื่อแอปพลิเคชันมีหลาย Feature ควรรวม State ทั้งหมดไว้ใน AppState:

```typescript
// app.state.ts
import { CounterState } from './store/counter/counter.state';
import { ProductState } from './store/product/product.state';

export interface AppState {
  counter: CounterState;
  product: ProductState;
}
```

```typescript
// app.module.ts
import { NgModule } from '@angular/core';
import { StoreModule } from '@ngrx/store';
import { counterReducer } from './store/counter/counter.reducer';
import { productReducer } from './store/product/product.reducer';

@NgModule({
  imports: [
    StoreModule.forRoot({
      counter: counterReducer,
      product: productReducer,
    }),
  ],
})
export class AppModule {}
```

---

## Feature Module (Lazy Loading)

สำหรับ Lazy Loaded Modules ใช้ `StoreModule.forFeature()`:

```typescript
// admin/admin.module.ts
import { NgModule } from '@angular/core';
import { StoreModule } from '@ngrx/store';
import { adminReducer } from './store/admin.reducer';

@NgModule({
  imports: [
    StoreModule.forFeature('admin', adminReducer),
  ],
})
export class AdminModule {}
```

---

## การทดสอบ Reducers

```typescript
// counter.reducer.spec.ts
import { counterReducer, initialState } from './counter.reducer';
import * as CounterActions from './counter.actions';

describe('CounterReducer', () => {
  it('ควรคืนค่า initialState เมื่อไม่มี action', () => {
    const state = counterReducer(undefined, { type: '@@INIT' } as any);
    expect(state).toEqual(initialState);
  });

  it('ควรเพิ่มค่า count เมื่อ dispatch increment', () => {
    const state = counterReducer(initialState, CounterActions.increment());
    expect(state.count).toBe(1);
  });

  it('ควรลดค่า count เมื่อ dispatch decrement', () => {
    const startState = { ...initialState, count: 5 };
    const state = counterReducer(startState, CounterActions.decrement());
    expect(state.count).toBe(4);
  });

  it('ควรบันทึก history เมื่อเปลี่ยนค่า', () => {
    let state = counterReducer(initialState, CounterActions.increment());
    state = counterReducer(state, CounterActions.increment());
    expect(state.history).toEqual([0, 1]);
    expect(state.count).toBe(2);
  });

  it('ควร reset ค่าเป็น initialState', () => {
    const startState = { count: 10, history: [1, 2, 3], lastUpdated: new Date() };
    const state = counterReducer(startState, CounterActions.reset());
    expect(state.count).toBe(0);
    expect(state.history).toEqual([]);
  });
});
```

---

## สรุป

| Concept | หน้าที่ |
|---------|---------|
| **Store** | Container กลางที่เก็บ State ทั้งหมด |
| **Action** | Event ที่บอกว่าจะทำอะไร |
| **Reducer** | Pure Function ที่สร้าง State ใหม่ |
| **Selector** | Function ที่ดึงข้อมูลจาก Store |

NgRx Store เป็นรากฐานสำคัญ ใน Part ถัดไปจะเรียน NgRx Effects สำหรับจัดการ Side Effects เช่น HTTP Requests
