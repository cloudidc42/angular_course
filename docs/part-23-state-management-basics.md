# Part 23 — State Management พื้นฐาน

## บทนำ

State Management คือการจัดการข้อมูล (state) ของแอปพลิเคชันอย่างมีระบบ เพื่อให้ predictable, maintainable และ testable

### ปัญหาที่ State Management แก้ได้
- Components หลายตัวต้องการข้อมูลเดียวกัน
- Data flow สับสน (props drilling)
- ยาก debug เมื่อ state เปลี่ยน
- ยาก test component ที่มี state ซับซ้อน

---

## 1. State Management คืออะไร

### 1.1 ประเภทของ State

```
State ใน Angular แบ่งเป็น:

1. Local State
   - ข้อมูลที่ใช้เฉพาะใน component เดียว
   - เช่น: isOpen, formValue, tempData
   - จัดการใน component ด้วย class properties

2. Shared State
   - ข้อมูลที่ใช้ร่วมกันระหว่าง components
   - เช่น: currentUser, cartItems, notifications
   - จัดการใน services

3. Server State
   - ข้อมูลที่มาจาก API
   - ต้องจัดการ loading, error, cache
   - จัดการใน services หรือ NgRx

4. URL State
   - ข้อมูลที่อยู่ใน URL (route params, query params)
   - จัดการด้วย Angular Router
```

### 1.2 หลักการ Flux Architecture

```
Action → Dispatcher → Store → View
   ↑___________________________|

- Action: ชื่อของสิ่งที่เกิดขึ้น (ADD_TO_CART, REMOVE_ITEM)
- Dispatcher: กระจาย action ไปยัง stores
- Store: เก็บ state และ update ตาม action
- View: แสดงผลตาม state และส่ง actions กลับ
```

---

## 2. Service with BehaviorSubject

Pattern นี้เป็นวิธีที่ง่ายที่สุดสำหรับ shared state

### 2.1 โครงสร้างพื้นฐาน

```typescript
// src/app/stores/counter.store.ts
import { Injectable } from '@angular/core';
import { BehaviorSubject, Observable } from 'rxjs';
import { map } from 'rxjs/operators';

@Injectable({ providedIn: 'root' })
export class CounterStore {

  // Private state — ไม่ให้ component เข้าถึงโดยตรง
  private countSubject = new BehaviorSubject<number>(0);

  // Public observable — read only
  count$ = this.countSubject.asObservable();

  // Derived state
  isEven$ = this.count$.pipe(map(count => count % 2 === 0));
  doubled$ = this.count$.pipe(map(count => count * 2));

  // Synchronous getter
  get currentCount(): number {
    return this.countSubject.getValue();
  }

  // Actions
  increment(): void {
    this.countSubject.next(this.currentCount + 1);
  }

  decrement(): void {
    const current = this.currentCount;
    if (current > 0) {
      this.countSubject.next(current - 1);
    }
  }

  reset(): void {
    this.countSubject.next(0);
  }

  setCount(value: number): void {
    if (value >= 0) {
      this.countSubject.next(value);
    }
  }
}
```

```typescript
// การใช้งานใน Component
@Component({
  selector: 'app-counter',
  standalone: true,
  imports: [CommonModule],
  template: `
    <div>
      <h2>Counter: {{ counter.count$ | async }}</h2>
      <p>คู่/คี่: {{ (counter.isEven$ | async) ? 'คู่' : 'คี่' }}</p>
      <p>x2: {{ counter.doubled$ | async }}</p>

      <button (click)="counter.decrement()">-</button>
      <button (click)="counter.increment()">+</button>
      <button (click)="counter.reset()">Reset</button>
    </div>
  `
})
export class CounterComponent {
  constructor(public counter: CounterStore) {}
}
```

### 2.2 ตัวอย่าง TodoList Store

```typescript
// src/app/stores/todo.store.ts
import { Injectable } from '@angular/core';
import { BehaviorSubject, Observable } from 'rxjs';
import { map } from 'rxjs/operators';

export interface Todo {
  id: number;
  title: string;
  completed: boolean;
  createdAt: Date;
}

export type FilterType = 'all' | 'active' | 'completed';

export interface TodoState {
  todos: Todo[];
  filter: FilterType;
  loading: boolean;
  error: string | null;
}

const initialState: TodoState = {
  todos: [],
  filter: 'all',
  loading: false,
  error: null
};

@Injectable({ providedIn: 'root' })
export class TodoStore {

  private stateSubject = new BehaviorSubject<TodoState>(initialState);
  private nextId = 1;

  // State observable
  state$ = this.stateSubject.asObservable();

  // Derived selectors
  todos$ = this.state$.pipe(map(state => state.todos));
  filter$ = this.state$.pipe(map(state => state.filter));
  loading$ = this.state$.pipe(map(state => state.loading));
  error$ = this.state$.pipe(map(state => state.error));

  filteredTodos$ = this.state$.pipe(
    map(state => {
      switch (state.filter) {
        case 'active':
          return state.todos.filter(todo => !todo.completed);
        case 'completed':
          return state.todos.filter(todo => todo.completed);
        default:
          return state.todos;
      }
    })
  );

  activeCount$ = this.todos$.pipe(
    map(todos => todos.filter(t => !t.completed).length)
  );

  completedCount$ = this.todos$.pipe(
    map(todos => todos.filter(t => t.completed).length)
  );

  allCompleted$ = this.todos$.pipe(
    map(todos => todos.length > 0 && todos.every(t => t.completed))
  );

  // Private state update helper
  private updateState(partial: Partial<TodoState>): void {
    this.stateSubject.next({
      ...this.stateSubject.getValue(),
      ...partial
    });
  }

  private get state(): TodoState {
    return this.stateSubject.getValue();
  }

  // Actions
  addTodo(title: string): void {
    if (!title.trim()) return;

    const todo: Todo = {
      id: this.nextId++,
      title: title.trim(),
      completed: false,
      createdAt: new Date()
    };

    this.updateState({
      todos: [...this.state.todos, todo]
    });
  }

  removeTodo(id: number): void {
    this.updateState({
      todos: this.state.todos.filter(todo => todo.id !== id)
    });
  }

  toggleTodo(id: number): void {
    this.updateState({
      todos: this.state.todos.map(todo =>
        todo.id === id
          ? { ...todo, completed: !todo.completed }
          : todo
      )
    });
  }

  updateTitle(id: number, newTitle: string): void {
    if (!newTitle.trim()) return;

    this.updateState({
      todos: this.state.todos.map(todo =>
        todo.id === id
          ? { ...todo, title: newTitle.trim() }
          : todo
      )
    });
  }

  setFilter(filter: FilterType): void {
    this.updateState({ filter });
  }

  clearCompleted(): void {
    this.updateState({
      todos: this.state.todos.filter(todo => !todo.completed)
    });
  }

  toggleAll(): void {
    const allCompleted = this.state.todos.every(t => t.completed);
    this.updateState({
      todos: this.state.todos.map(todo => ({
        ...todo,
        completed: !allCompleted
      }))
    });
  }
}
```

---

## 3. Simple Store Pattern

### 3.1 Generic Store Class

```typescript
// src/app/core/store/generic.store.ts
import { BehaviorSubject, Observable } from 'rxjs';
import { map, distinctUntilChanged } from 'rxjs/operators';

export class Store<T> {

  private stateSubject: BehaviorSubject<T>;

  constructor(initialState: T) {
    this.stateSubject = new BehaviorSubject<T>(initialState);
  }

  // Get full state as observable
  protected get state$(): Observable<T> {
    return this.stateSubject.asObservable();
  }

  // Get current state synchronously
  protected get state(): T {
    return this.stateSubject.getValue();
  }

  // Select a slice of state
  protected select<K>(selector: (state: T) => K): Observable<K> {
    return this.state$.pipe(
      map(selector),
      distinctUntilChanged()
    );
  }

  // Update state (immutably)
  protected setState(partial: Partial<T>): void {
    this.stateSubject.next({
      ...this.state,
      ...partial
    });
  }

  // Replace entire state
  protected replaceState(newState: T): void {
    this.stateSubject.next(newState);
  }
}
```

```typescript
// src/app/stores/product.store.ts
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { tap, catchError } from 'rxjs/operators';
import { of } from 'rxjs';
import { Store } from '../core/store/generic.store';

interface Product {
  id: number;
  name: string;
  price: number;
  category: string;
  stock: number;
}

interface ProductState {
  products: Product[];
  selectedProduct: Product | null;
  loading: boolean;
  error: string | null;
  filter: {
    category: string;
    minPrice: number;
    maxPrice: number;
    search: string;
  };
}

const initialState: ProductState = {
  products: [],
  selectedProduct: null,
  loading: false,
  error: null,
  filter: {
    category: '',
    minPrice: 0,
    maxPrice: Infinity,
    search: ''
  }
};

@Injectable({ providedIn: 'root' })
export class ProductStore extends Store<ProductState> {

  constructor(private http: HttpClient) {
    super(initialState);
  }

  // Selectors (public)
  products$ = this.select(state => state.products);
  loading$ = this.select(state => state.loading);
  error$ = this.select(state => state.error);
  selectedProduct$ = this.select(state => state.selectedProduct);

  filteredProducts$ = this.select(state => {
    const { products, filter } = state;
    return products
      .filter(p => !filter.category || p.category === filter.category)
      .filter(p => p.price >= filter.minPrice && p.price <= filter.maxPrice)
      .filter(p =>
        !filter.search ||
        p.name.toLowerCase().includes(filter.search.toLowerCase())
      );
  });

  categories$ = this.select(state =>
    [...new Set(state.products.map(p => p.category))]
  );

  // Actions
  loadProducts(): void {
    this.setState({ loading: true, error: null });

    this.http.get<Product[]>('/api/products').pipe(
      tap(products => {
        this.setState({ products, loading: false });
      }),
      catchError(error => {
        this.setState({
          loading: false,
          error: 'ไม่สามารถโหลดสินค้าได้'
        });
        return of([]);
      })
    ).subscribe();
  }

  selectProduct(id: number): void {
    const product = this.state.products.find(p => p.id === id) ?? null;
    this.setState({ selectedProduct: product });
  }

  updateFilter(filterUpdate: Partial<ProductState['filter']>): void {
    this.setState({
      filter: { ...this.state.filter, ...filterUpdate }
    });
  }

  clearFilter(): void {
    this.setState({
      filter: initialState.filter
    });
  }
}
```

---

## 4. ก่อน NgRx: State Management แบบง่าย

### 4.1 เปรียบเทียบแนวทาง

```
1. Component State (ง่ายที่สุด)
   - ใช้เมื่อ: state ใช้แค่ใน component เดียว
   - วิธี: class properties ปกติ

2. Service + BehaviorSubject (แนะนำสำหรับ medium apps)
   - ใช้เมื่อ: share state ระหว่างไม่กี่ components
   - วิธี: Injectable service พร้อม BehaviorSubject

3. Generic Store Pattern (ก่อน NgRx)
   - ใช้เมื่อ: ต้องการ structure แต่ยังไม่อยากใช้ NgRx
   - วิธี: Base Store class ที่ extend ได้

4. NgRx (สำหรับ large apps)
   - ใช้เมื่อ: app ใหญ่, team ใหญ่, ต้องการ DevTools
   - วิธี: Actions, Reducers, Effects, Selectors
```

---

## 5. Workshop: Shopping Cart State

```typescript
// src/app/stores/cart.store.ts
import { Injectable } from '@angular/core';
import { BehaviorSubject } from 'rxjs';
import { map, distinctUntilChanged } from 'rxjs/operators';

export interface CartItem {
  productId: number;
  name: string;
  price: number;
  quantity: number;
  image: string;
}

export interface CartState {
  items: CartItem[];
  couponCode: string | null;
  discountPercent: number;
  shippingCost: number;
}

const initialState: CartState = {
  items: [],
  couponCode: null,
  discountPercent: 0,
  shippingCost: 50
};

@Injectable({ providedIn: 'root' })
export class CartStore {

  private stateSubject = new BehaviorSubject<CartState>(initialState);
  state$ = this.stateSubject.asObservable();

  // Selectors
  items$ = this.state$.pipe(
    map(state => state.items),
    distinctUntilChanged()
  );

  itemCount$ = this.items$.pipe(
    map(items => items.reduce((sum, item) => sum + item.quantity, 0))
  );

  subtotal$ = this.items$.pipe(
    map(items => items.reduce((sum, item) => sum + (item.price * item.quantity), 0))
  );

  discount$ = this.state$.pipe(
    map(state => {
      const subtotal = state.items.reduce(
        (sum, item) => sum + (item.price * item.quantity), 0
      );
      return subtotal * (state.discountPercent / 100);
    })
  );

  shipping$ = this.state$.pipe(
    map(state => {
      const subtotal = state.items.reduce(
        (sum, item) => sum + (item.price * item.quantity), 0
      );
      // ฟรีค่าส่งเมื่อซื้อเกิน 1000 บาท
      return subtotal >= 1000 ? 0 : state.shippingCost;
    })
  );

  total$ = this.state$.pipe(
    map(state => {
      const subtotal = state.items.reduce(
        (sum, item) => sum + (item.price * item.quantity), 0
      );
      const discount = subtotal * (state.discountPercent / 100);
      const shipping = subtotal >= 1000 ? 0 : state.shippingCost;
      return subtotal - discount + shipping;
    })
  );

  isEmpty$ = this.items$.pipe(map(items => items.length === 0));

  private get state(): CartState {
    return this.stateSubject.getValue();
  }

  private updateState(partial: Partial<CartState>): void {
    this.stateSubject.next({ ...this.state, ...partial });
  }

  // Actions
  addItem(product: { id: number; name: string; price: number; image: string }): void {
    const existingItem = this.state.items.find(
      item => item.productId === product.id
    );

    if (existingItem) {
      this.updateQuantity(product.id, existingItem.quantity + 1);
    } else {
      this.updateState({
        items: [
          ...this.state.items,
          {
            productId: product.id,
            name: product.name,
            price: product.price,
            quantity: 1,
            image: product.image
          }
        ]
      });
    }
  }

  removeItem(productId: number): void {
    this.updateState({
      items: this.state.items.filter(item => item.productId !== productId)
    });
  }

  updateQuantity(productId: number, quantity: number): void {
    if (quantity <= 0) {
      this.removeItem(productId);
      return;
    }

    this.updateState({
      items: this.state.items.map(item =>
        item.productId === productId
          ? { ...item, quantity }
          : item
      )
    });
  }

  applyCoupon(code: string): boolean {
    const coupons: Record<string, number> = {
      'SAVE10': 10,
      'SAVE20': 20,
      'HALFOFF': 50
    };

    const discount = coupons[code.toUpperCase()];

    if (discount !== undefined) {
      this.updateState({
        couponCode: code.toUpperCase(),
        discountPercent: discount
      });
      return true;
    }

    return false;
  }

  removeCoupon(): void {
    this.updateState({
      couponCode: null,
      discountPercent: 0
    });
  }

  clearCart(): void {
    this.stateSubject.next(initialState);
  }
}
```

```typescript
// src/app/components/shopping-cart/shopping-cart.component.ts
import { Component, OnInit } from '@angular/core';
import { CommonModule, CurrencyPipe } from '@angular/common';
import { ReactiveFormsModule, FormControl } from '@angular/forms';
import { CartStore, CartItem } from '../../stores/cart.store';

@Component({
  selector: 'app-shopping-cart',
  standalone: true,
  imports: [CommonModule, ReactiveFormsModule, CurrencyPipe],
  template: `
    <div class="cart-container">
      <h2>ตะกร้าสินค้า ({{ cart.itemCount$ | async }} ชิ้น)</h2>

      <!-- Empty State -->
      <div *ngIf="cart.isEmpty$ | async" class="empty-cart">
        <p>ตะกร้าสินค้าว่างเปล่า</p>
      </div>

      <!-- Cart Items -->
      <div *ngIf="!(cart.isEmpty$ | async)" class="cart-content">
        <div
          *ngFor="let item of cart.items$ | async"
          class="cart-item"
        >
          <img [src]="item.image" [alt]="item.name" class="item-image">

          <div class="item-info">
            <h4>{{ item.name }}</h4>
            <p>{{ item.price | currency:'THB':'symbol':'1.0-0' }} / ชิ้น</p>
          </div>

          <div class="item-quantity">
            <button (click)="cart.updateQuantity(item.productId, item.quantity - 1)">-</button>
            <span>{{ item.quantity }}</span>
            <button (click)="cart.updateQuantity(item.productId, item.quantity + 1)">+</button>
          </div>

          <div class="item-total">
            {{ item.price * item.quantity | currency:'THB':'symbol':'1.0-0' }}
          </div>

          <button
            class="remove-btn"
            (click)="cart.removeItem(item.productId)"
          >✕</button>
        </div>

        <!-- Coupon Section -->
        <div class="coupon-section">
          <div *ngIf="!(cart.state$ | async)?.couponCode; else couponApplied">
            <input
              [formControl]="couponControl"
              placeholder="รหัสส่วนลด"
              class="coupon-input"
            >
            <button (click)="applyCoupon()" class="btn btn-outline">
              ใช้รหัส
            </button>
            <span *ngIf="couponError" class="error-text">{{ couponError }}</span>
          </div>
          <ng-template #couponApplied>
            <span class="coupon-badge">
              🏷 {{ (cart.state$ | async)?.couponCode }}
              (-{{ (cart.state$ | async)?.discountPercent }}%)
            </span>
            <button (click)="cart.removeCoupon()" class="btn-remove-coupon">ลบ</button>
          </ng-template>
        </div>

        <!-- Order Summary -->
        <div class="order-summary">
          <div class="summary-row">
            <span>ราคาสินค้า</span>
            <span>{{ cart.subtotal$ | async | currency:'THB':'symbol':'1.0-0' }}</span>
          </div>
          <div class="summary-row" *ngIf="(cart.state$ | async)?.couponCode">
            <span>ส่วนลด</span>
            <span class="discount-text">
              -{{ cart.discount$ | async | currency:'THB':'symbol':'1.0-0' }}
            </span>
          </div>
          <div class="summary-row">
            <span>ค่าจัดส่ง</span>
            <span>
              <ng-container *ngIf="(cart.shipping$ | async) === 0; else shippingCost">
                ฟรี 🎉
              </ng-container>
              <ng-template #shippingCost>
                {{ cart.shipping$ | async | currency:'THB':'symbol':'1.0-0' }}
              </ng-template>
            </span>
          </div>
          <div class="summary-row total-row">
            <span><strong>รวมทั้งหมด</strong></span>
            <span><strong>{{ cart.total$ | async | currency:'THB':'symbol':'1.0-0' }}</strong></span>
          </div>
        </div>

        <!-- Actions -->
        <div class="cart-actions">
          <button (click)="cart.clearCart()" class="btn btn-outline">
            ล้างตะกร้า
          </button>
          <button class="btn btn-primary">
            ชำระเงิน
          </button>
        </div>
      </div>
    </div>
  `,
  styles: [`
    .cart-container { max-width: 800px; margin: 0 auto; padding: 20px; }
    .cart-item {
      display: flex;
      align-items: center;
      gap: 16px;
      padding: 12px 0;
      border-bottom: 1px solid #eee;
    }
    .item-image { width: 80px; height: 80px; object-fit: cover; border-radius: 4px; }
    .item-info { flex: 1; }
    .item-quantity {
      display: flex;
      align-items: center;
      gap: 8px;
    }
    .item-quantity button {
      width: 28px;
      height: 28px;
      border: 1px solid #ddd;
      background: white;
      cursor: pointer;
      border-radius: 4px;
    }
    .remove-btn { background: none; border: none; cursor: pointer; color: #999; }
    .coupon-section { padding: 16px 0; }
    .coupon-input { padding: 8px; border: 1px solid #ddd; border-radius: 4px; margin-right: 8px; }
    .order-summary { background: #f8f9fa; padding: 16px; border-radius: 8px; margin: 16px 0; }
    .summary-row { display: flex; justify-content: space-between; padding: 4px 0; }
    .total-row { border-top: 1px solid #ddd; padding-top: 8px; margin-top: 8px; }
    .discount-text { color: #28a745; }
    .coupon-badge {
      background: #d4edda;
      color: #155724;
      padding: 4px 8px;
      border-radius: 12px;
    }
    .error-text { color: #dc3545; font-size: 14px; margin-left: 8px; }
    .cart-actions { display: flex; justify-content: flex-end; gap: 12px; }
    .btn { padding: 10px 20px; border-radius: 4px; cursor: pointer; }
    .btn-primary { background: #007bff; color: white; border: none; }
    .btn-outline { background: white; border: 1px solid #007bff; color: #007bff; }
  `]
})
export class ShoppingCartComponent {

  couponControl = new FormControl('');
  couponError = '';

  constructor(public cart: CartStore) {}

  applyCoupon(): void {
    const code = this.couponControl.value ?? '';
    const success = this.cart.applyCoupon(code);

    if (success) {
      this.couponControl.setValue('');
      this.couponError = '';
    } else {
      this.couponError = 'รหัสส่วนลดไม่ถูกต้อง';
    }
  }
}
```

---

## สรุปบทที่ 23

### เมื่อไหรควรใช้แนวทางไหน?

| สถานการณ์ | แนวทาง |
|-----------|--------|
| State ใน component เดียว | Component class properties |
| Share state ระหว่าง parent-child | @Input/@Output |
| Share state ระหว่าง siblings | Service + BehaviorSubject |
| App ขนาดกลาง | Generic Store Pattern |
| App ขนาดใหญ่, team ใหญ่ | NgRx หรือ Elf/Akita |

### Best Practices

1. **ใช้ `asObservable()`** — ซ่อน Subject ไม่ให้ components เข้าถึง `.next()` โดยตรง
2. **Immutable state updates** — ใช้ spread operator เสมอ
3. **Derived state ใน store** — คำนวณ computed values ใน store ไม่ใช่ใน component
4. **ใช้ `async pipe`** — จัดการ subscription อัตโนมัติ
5. **Single source of truth** — เก็บ state ไว้ที่เดียว
