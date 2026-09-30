# Part 27 — NgRx Effects

## NgRx Effects คืออะไร?

NgRx Effects เป็น Library สำหรับจัดการ Side Effects ใน NgRx โดยเฉพาะ Side Effects ที่เกี่ยวข้องกับการเรียก API, การจัดการ LocalStorage, การนำทาง หรือการดำเนินงานที่ไม่บริสุทธิ์ (Impure Operations)

### ทำไมต้องใช้ Effects?

Reducers ต้องเป็น Pure Functions ดังนั้นการเรียก API หรือทำ Side Effects จึงทำใน Reducers ไม่ได้ Effects แก้ปัญหานี้โดยการ:

1. รับฟัง Actions จาก Store
2. ทำ Side Effects (เช่น HTTP Request)
3. Dispatch Actions ใหม่กลับไปยัง Store

```
Component → dispatch(loadProducts)
              ↓
           Effects รับ loadProducts
              ↓
           ทำ HTTP Request
              ↓
           dispatch(loadProductsSuccess/Failure)
              ↓
           Reducer อัปเดต State
              ↓
           Component แสดงผลข้อมูลใหม่
```

---

## การติดตั้ง

```bash
npm install @ngrx/effects
# หรือ
ng add @ngrx/effects
```

---

## โครงสร้างพื้นฐานของ Effects

```typescript
import { Injectable } from '@angular/core';
import { Actions, createEffect, ofType } from '@ngrx/effects';
import { EMPTY } from 'rxjs';
import { catchError, map, switchMap } from 'rxjs/operators';
import { ProductService } from '../services/product.service';
import * as ProductActions from '../store/product/product.actions';

@Injectable()
export class ProductEffects {

  // สร้าง Effect ด้วย createEffect
  loadProducts$ = createEffect(() =>
    this.actions$.pipe(
      // กรองเฉพาะ Action ที่ต้องการ
      ofType(ProductActions.loadProducts),

      // ทำ Side Effect
      switchMap(() =>
        this.productService.getProducts().pipe(
          // กรณีสำเร็จ
          map((products) => ProductActions.loadProductsSuccess({ products })),

          // กรณีเกิดข้อผิดพลาด
          catchError((error) =>
            of(ProductActions.loadProductsFailure({ error: error.message }))
          )
        )
      )
    )
  );

  constructor(
    private actions$: Actions,
    private productService: ProductService
  ) {}
}
```

---

## Parameters ของ createEffect

```typescript
// Effect แบบ Dispatch (ค่าเริ่มต้น)
loadData$ = createEffect(() => /* ... */);

// Effect แบบไม่ Dispatch (สำหรับ Navigation, LocalStorage)
navigate$ = createEffect(
  () => /* ... */,
  { dispatch: false }
);

// Effect แบบไม่ Dispatch และจัดการ Error เอง
saveToStorage$ = createEffect(
  () => /* ... */,
  { dispatch: false, useEffectsErrorHandler: false }
);
```

---

## RxJS Operators ใน Effects

### switchMap — ใช้สำหรับ Cancel Previous Request

```typescript
// เมื่อ Action ใหม่มาถึง จะ Cancel Request เดิม (เหมาะสำหรับ Search)
searchProducts$ = createEffect(() =>
  this.actions$.pipe(
    ofType(ProductActions.searchProducts),
    debounceTime(300),  // รอให้ผู้ใช้หยุดพิมพ์
    switchMap(({ term }) =>
      this.productService.search(term).pipe(
        map((products) => ProductActions.searchProductsSuccess({ products })),
        catchError((error) => of(ProductActions.searchProductsFailure({ error: error.message })))
      )
    )
  )
);
```

### mergeMap — ใช้เมื่อต้องการทำ Concurrent Requests

```typescript
// ทำ Request หลายตัวพร้อมกัน (เหมาะสำหรับ Delete หลาย Items)
deleteProducts$ = createEffect(() =>
  this.actions$.pipe(
    ofType(ProductActions.deleteProduct),
    mergeMap(({ id }) =>
      this.productService.delete(id).pipe(
        map(() => ProductActions.deleteProductSuccess({ id })),
        catchError((error) => of(ProductActions.deleteProductFailure({ error: error.message })))
      )
    )
  )
);
```

### concatMap — ใช้เมื่อต้องทำตามลำดับ

```typescript
// ทำ Request ทีละตัวตามลำดับ (เหมาะสำหรับ Submit Form)
updateProduct$ = createEffect(() =>
  this.actions$.pipe(
    ofType(ProductActions.updateProduct),
    concatMap(({ product }) =>
      this.productService.update(product).pipe(
        map((updated) => ProductActions.updateProductSuccess({ product: updated })),
        catchError((error) => of(ProductActions.updateProductFailure({ error: error.message })))
      )
    )
  )
);
```

### exhaustMap — ใช้เมื่อไม่ต้องการ Request ซ้ำ

```typescript
// ป้องกันการ Submit Form ซ้ำ
submitForm$ = createEffect(() =>
  this.actions$.pipe(
    ofType(ProductActions.submitForm),
    exhaustMap(({ data }) =>
      this.productService.submit(data).pipe(
        map(() => ProductActions.submitFormSuccess()),
        catchError((error) => of(ProductActions.submitFormFailure({ error: error.message })))
      )
    )
  )
);
```

---

## Error Handling ใน Effects

### วิธีที่ 1: catchError ใน Inner Observable

```typescript
loadProducts$ = createEffect(() =>
  this.actions$.pipe(
    ofType(ProductActions.loadProducts),
    switchMap(() =>
      this.productService.getProducts().pipe(
        map((products) => ProductActions.loadProductsSuccess({ products })),
        // catchError ต้องอยู่ใน inner pipe เพื่อไม่ให้ Effect หยุดทำงาน
        catchError((error) => {
          console.error('Load products error:', error);
          return of(ProductActions.loadProductsFailure({
            error: error.message || 'เกิดข้อผิดพลาด'
          }));
        })
      )
    )
  )
);
```

### วิธีที่ 2: Global Error Handler

```typescript
// ถ้า catchError อยู่ด้านนอก Effect จะหยุดทำงาน
// ต้องใช้ tap เพื่อ log error เท่านั้น
loadProducts$ = createEffect(() =>
  this.actions$.pipe(
    ofType(ProductActions.loadProducts),
    switchMap(() =>
      this.productService.getProducts().pipe(
        map((products) => ProductActions.loadProductsSuccess({ products })),
        catchError((error) => of(ProductActions.loadProductsFailure({ error: error.message })))
      )
    )
  )
);
```

### วิธีที่ 3: Retry Logic

```typescript
import { retry, catchError } from 'rxjs/operators';

loadWithRetry$ = createEffect(() =>
  this.actions$.pipe(
    ofType(ProductActions.loadProducts),
    switchMap(() =>
      this.productService.getProducts().pipe(
        retry(3),  // ลองใหม่ 3 ครั้ง
        map((products) => ProductActions.loadProductsSuccess({ products })),
        catchError((error) =>
          of(ProductActions.loadProductsFailure({ error: 'ไม่สามารถโหลดข้อมูลได้' }))
        )
      )
    )
  )
);
```

---

## Workshop: Load Products Effect แบบสมบูรณ์

### Service

```typescript
// product.service.ts
import { Injectable } from '@angular/core';
import { HttpClient, HttpErrorResponse } from '@angular/common/http';
import { Observable, throwError } from 'rxjs';
import { catchError, map, retry } from 'rxjs/operators';
import { Product } from '../store/product/product.state';

@Injectable({
  providedIn: 'root',
})
export class ProductService {
  private apiUrl = 'https://api.example.com/products';

  constructor(private http: HttpClient) {}

  getAll(): Observable<Product[]> {
    return this.http.get<Product[]>(this.apiUrl).pipe(
      retry(2),
      catchError(this.handleError)
    );
  }

  getById(id: number): Observable<Product> {
    return this.http.get<Product>(`${this.apiUrl}/${id}`).pipe(
      catchError(this.handleError)
    );
  }

  create(product: Omit<Product, 'id'>): Observable<Product> {
    return this.http.post<Product>(this.apiUrl, product).pipe(
      catchError(this.handleError)
    );
  }

  update(product: Product): Observable<Product> {
    return this.http.put<Product>(`${this.apiUrl}/${product.id}`, product).pipe(
      catchError(this.handleError)
    );
  }

  delete(id: number): Observable<void> {
    return this.http.delete<void>(`${this.apiUrl}/${id}`).pipe(
      catchError(this.handleError)
    );
  }

  private handleError(error: HttpErrorResponse): Observable<never> {
    let message = 'เกิดข้อผิดพลาดที่ไม่ทราบสาเหตุ';

    if (error.status === 0) {
      message = 'ไม่สามารถเชื่อมต่อเซิร์ฟเวอร์ได้';
    } else if (error.status === 404) {
      message = 'ไม่พบข้อมูลที่ต้องการ';
    } else if (error.status === 403) {
      message = 'ไม่มีสิทธิ์ดำเนินการ';
    } else if (error.status >= 500) {
      message = 'เซิร์ฟเวอร์เกิดข้อผิดพลาด กรุณาลองใหม่';
    } else if (error.error?.message) {
      message = error.error.message;
    }

    return throwError(() => new Error(message));
  }
}
```

### Actions ที่ครบถ้วน

```typescript
// product.actions.ts (ฉบับสมบูรณ์)
import { createAction, props } from '@ngrx/store';
import { Product } from './product.state';

// Load All
export const loadProducts = createAction('[Product] Load Products');
export const loadProductsSuccess = createAction(
  '[Product] Load Products Success',
  props<{ products: Product[] }>()
);
export const loadProductsFailure = createAction(
  '[Product] Load Products Failure',
  props<{ error: string }>()
);

// Load Single
export const loadProduct = createAction(
  '[Product] Load Product',
  props<{ id: number }>()
);
export const loadProductSuccess = createAction(
  '[Product] Load Product Success',
  props<{ product: Product }>()
);
export const loadProductFailure = createAction(
  '[Product] Load Product Failure',
  props<{ error: string }>()
);

// Create
export const createProduct = createAction(
  '[Product] Create Product',
  props<{ product: Omit<Product, 'id'> }>()
);
export const createProductSuccess = createAction(
  '[Product] Create Product Success',
  props<{ product: Product }>()
);
export const createProductFailure = createAction(
  '[Product] Create Product Failure',
  props<{ error: string }>()
);

// Update
export const updateProduct = createAction(
  '[Product] Update Product',
  props<{ product: Product }>()
);
export const updateProductSuccess = createAction(
  '[Product] Update Product Success',
  props<{ product: Product }>()
);
export const updateProductFailure = createAction(
  '[Product] Update Product Failure',
  props<{ error: string }>()
);

// Delete
export const deleteProduct = createAction(
  '[Product] Delete Product',
  props<{ id: number }>()
);
export const deleteProductSuccess = createAction(
  '[Product] Delete Product Success',
  props<{ id: number }>()
);
export const deleteProductFailure = createAction(
  '[Product] Delete Product Failure',
  props<{ error: string }>()
);
```

### Effects แบบสมบูรณ์

```typescript
// product.effects.ts
import { Injectable } from '@angular/core';
import { Router } from '@angular/router';
import { Actions, createEffect, ofType } from '@ngrx/effects';
import { of } from 'rxjs';
import {
  catchError,
  concatMap,
  map,
  mergeMap,
  switchMap,
  tap,
} from 'rxjs/operators';
import { ProductService } from '../services/product.service';
import { NotificationService } from '../services/notification.service';
import * as ProductActions from './product.actions';

@Injectable()
export class ProductEffects {

  // โหลดสินค้าทั้งหมด
  loadProducts$ = createEffect(() =>
    this.actions$.pipe(
      ofType(ProductActions.loadProducts),
      switchMap(() =>
        this.productService.getAll().pipe(
          map((products) => ProductActions.loadProductsSuccess({ products })),
          catchError((error) =>
            of(ProductActions.loadProductsFailure({ error: error.message }))
          )
        )
      )
    )
  );

  // โหลดสินค้าชิ้นเดียว
  loadProduct$ = createEffect(() =>
    this.actions$.pipe(
      ofType(ProductActions.loadProduct),
      switchMap(({ id }) =>
        this.productService.getById(id).pipe(
          map((product) => ProductActions.loadProductSuccess({ product })),
          catchError((error) =>
            of(ProductActions.loadProductFailure({ error: error.message }))
          )
        )
      )
    )
  );

  // สร้างสินค้าใหม่
  createProduct$ = createEffect(() =>
    this.actions$.pipe(
      ofType(ProductActions.createProduct),
      concatMap(({ product }) =>
        this.productService.create(product).pipe(
          map((newProduct) => ProductActions.createProductSuccess({ product: newProduct })),
          catchError((error) =>
            of(ProductActions.createProductFailure({ error: error.message }))
          )
        )
      )
    )
  );

  // อัปเดตสินค้า
  updateProduct$ = createEffect(() =>
    this.actions$.pipe(
      ofType(ProductActions.updateProduct),
      concatMap(({ product }) =>
        this.productService.update(product).pipe(
          map((updated) => ProductActions.updateProductSuccess({ product: updated })),
          catchError((error) =>
            of(ProductActions.updateProductFailure({ error: error.message }))
          )
        )
      )
    )
  );

  // ลบสินค้า
  deleteProduct$ = createEffect(() =>
    this.actions$.pipe(
      ofType(ProductActions.deleteProduct),
      mergeMap(({ id }) =>
        this.productService.delete(id).pipe(
          map(() => ProductActions.deleteProductSuccess({ id })),
          catchError((error) =>
            of(ProductActions.deleteProductFailure({ error: error.message }))
          )
        )
      )
    )
  );

  // แสดงข้อความสำเร็จ (ไม่ Dispatch Action)
  showSuccessNotification$ = createEffect(
    () =>
      this.actions$.pipe(
        ofType(
          ProductActions.createProductSuccess,
          ProductActions.updateProductSuccess,
          ProductActions.deleteProductSuccess
        ),
        tap((action) => {
          const messages: Record<string, string> = {
            '[Product] Create Product Success': 'สร้างสินค้าสำเร็จ',
            '[Product] Update Product Success': 'อัปเดตสินค้าสำเร็จ',
            '[Product] Delete Product Success': 'ลบสินค้าสำเร็จ',
          };
          const message = messages[action.type] || 'ดำเนินการสำเร็จ';
          this.notificationService.showSuccess(message);
        })
      ),
    { dispatch: false }
  );

  // แสดงข้อความผิดพลาด (ไม่ Dispatch Action)
  showErrorNotification$ = createEffect(
    () =>
      this.actions$.pipe(
        ofType(
          ProductActions.loadProductsFailure,
          ProductActions.createProductFailure,
          ProductActions.updateProductFailure,
          ProductActions.deleteProductFailure
        ),
        tap(({ error }) => {
          this.notificationService.showError(error);
        })
      ),
    { dispatch: false }
  );

  // นำทางหลังสร้างสินค้า
  navigateAfterCreate$ = createEffect(
    () =>
      this.actions$.pipe(
        ofType(ProductActions.createProductSuccess),
        tap(({ product }) => {
          this.router.navigate(['/products', product.id]);
        })
      ),
    { dispatch: false }
  );

  constructor(
    private actions$: Actions,
    private productService: ProductService,
    private notificationService: NotificationService,
    private router: Router
  ) {}
}
```

### ลงทะเบียน Effects ใน Module

```typescript
// app.module.ts
import { NgModule } from '@angular/core';
import { EffectsModule } from '@ngrx/effects';
import { ProductEffects } from './store/product/product.effects';

@NgModule({
  imports: [
    EffectsModule.forRoot([ProductEffects]),
    // สำหรับ Feature Module:
    // EffectsModule.forFeature([ProductEffects])
  ],
})
export class AppModule {}
```

---

## Effect ขั้นสูง: การรวม Actions

```typescript
// Effect ที่ทำงานเมื่อมี Actions หลายอย่าง
refreshOnChanges$ = createEffect(() =>
  this.actions$.pipe(
    ofType(
      ProductActions.createProductSuccess,
      ProductActions.updateProductSuccess,
      ProductActions.deleteProductSuccess
    ),
    // โหลดข้อมูลใหม่หลังมีการเปลี่ยนแปลง
    map(() => ProductActions.loadProducts())
  )
);
```

---

## Effect ที่ใช้ Store Selector

```typescript
import { Store } from '@ngrx/store';
import { withLatestFrom } from 'rxjs/operators';
import * as ProductSelectors from './product.selectors';

@Injectable()
export class ProductEffects {

  // ใช้ข้อมูลจาก Store ใน Effect
  loadWithPagination$ = createEffect(() =>
    this.actions$.pipe(
      ofType(ProductActions.loadNextPage),
      withLatestFrom(this.store.select(ProductSelectors.selectCurrentPage)),
      switchMap(([action, currentPage]) =>
        this.productService.getPage(currentPage + 1).pipe(
          map((products) => ProductActions.loadNextPageSuccess({ products })),
          catchError((error) =>
            of(ProductActions.loadNextPageFailure({ error: error.message }))
          )
        )
      )
    )
  );

  constructor(
    private actions$: Actions,
    private productService: ProductService,
    private store: Store
  ) {}
}
```

---

## Effect สำหรับ LocalStorage

```typescript
@Injectable()
export class CartEffects {

  // บันทึก Cart ลง LocalStorage
  saveCart$ = createEffect(
    () =>
      this.actions$.pipe(
        ofType(
          CartActions.addItem,
          CartActions.removeItem,
          CartActions.updateQuantity
        ),
        withLatestFrom(this.store.select(CartSelectors.selectCartItems)),
        tap(([, items]) => {
          try {
            localStorage.setItem('cart', JSON.stringify(items));
          } catch (e) {
            console.error('ไม่สามารถบันทึกตะกร้าได้', e);
          }
        })
      ),
    { dispatch: false }
  );

  // โหลด Cart จาก LocalStorage เมื่อเริ่มต้นแอป
  loadCart$ = createEffect(() =>
    this.actions$.pipe(
      ofType(AppActions.appInit),
      map(() => {
        try {
          const saved = localStorage.getItem('cart');
          const items = saved ? JSON.parse(saved) : [];
          return CartActions.loadCartSuccess({ items });
        } catch (e) {
          return CartActions.loadCartSuccess({ items: [] });
        }
      })
    )
  );

  constructor(
    private actions$: Actions,
    private store: Store
  ) {}
}
```

---

## การทดสอบ Effects

```typescript
// product.effects.spec.ts
import { TestBed } from '@angular/core/testing';
import { provideMockActions } from '@ngrx/effects/testing';
import { Observable, of, throwError } from 'rxjs';
import { cold, hot } from 'jasmine-marbles';
import { ProductEffects } from './product.effects';
import { ProductService } from '../services/product.service';
import * as ProductActions from './product.actions';
import { Product } from './product.state';

describe('ProductEffects', () => {
  let actions$: Observable<any>;
  let effects: ProductEffects;
  let productService: jasmine.SpyObj<ProductService>;

  const mockProducts: Product[] = [
    { id: 1, name: 'สินค้า A', price: 100, category: 'A', stock: 10, description: 'คำอธิบาย A' },
    { id: 2, name: 'สินค้า B', price: 200, category: 'B', stock: 5, description: 'คำอธิบาย B' },
  ];

  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [
        ProductEffects,
        provideMockActions(() => actions$),
        {
          provide: ProductService,
          useValue: jasmine.createSpyObj('ProductService', ['getAll', 'create', 'delete']),
        },
      ],
    });

    effects = TestBed.inject(ProductEffects);
    productService = TestBed.inject(ProductService) as jasmine.SpyObj<ProductService>;
  });

  describe('loadProducts$', () => {
    it('ควร dispatch loadProductsSuccess เมื่อ API สำเร็จ', () => {
      const action = ProductActions.loadProducts();
      const outcome = ProductActions.loadProductsSuccess({ products: mockProducts });

      actions$ = hot('-a', { a: action });
      const response = cold('-b|', { b: mockProducts });
      productService.getAll.and.returnValue(response);

      const expected = cold('--c', { c: outcome });
      expect(effects.loadProducts$).toBeObservable(expected);
    });

    it('ควร dispatch loadProductsFailure เมื่อ API ล้มเหลว', () => {
      const action = ProductActions.loadProducts();
      const error = new Error('Network Error');
      const outcome = ProductActions.loadProductsFailure({ error: error.message });

      actions$ = hot('-a', { a: action });
      const response = cold('-#', {}, error);
      productService.getAll.and.returnValue(response);

      const expected = cold('--c', { c: outcome });
      expect(effects.loadProducts$).toBeObservable(expected);
    });
  });
});
```

---

## สรุป

| Operator | การใช้งาน |
|----------|-----------|
| **switchMap** | Cancel Request เดิม (Search, Load) |
| **mergeMap** | ทำหลาย Requests พร้อมกัน (Delete หลายชิ้น) |
| **concatMap** | ทำ Request ตามลำดับ (Create, Update) |
| **exhaustMap** | ป้องกัน Request ซ้ำ (Submit Form) |

| Configuration | ความหมาย |
|--------------|----------|
| `dispatch: true` | Effect จะ Dispatch Action (ค่าเริ่มต้น) |
| `dispatch: false` | Effect ไม่ Dispatch Action (Navigation, Storage) |
| `useEffectsErrorHandler: false` | จัดการ Error เอง |

ใน Part ถัดไปจะเรียน NgRx Selectors อย่างละเอียด รวมถึง Memoization และ Derived State
