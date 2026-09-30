# Part 28 — NgRx Selectors

## Selectors คืออะไร?

Selectors คือ Pure Functions ที่ดึงข้อมูลจาก Store NgRx มีฟีเจอร์ Memoization ที่ช่วยให้ Selectors คำนวณใหม่เฉพาะเมื่อ Input เปลี่ยน ทำให้แอปพลิเคชันมีประสิทธิภาพสูงขึ้น

### ประโยชน์ของ Selectors

1. **Memoization** — คำนวณใหม่เฉพาะเมื่อ Input เปลี่ยน
2. **Composable** — รวม Selectors หลายตัวเป็นตัวใหม่ได้
3. **Testable** — ทดสอบเป็น Pure Functions ได้ง่าย
4. **Reusable** — ใช้ซ้ำใน Component หลายที่
5. **Type-safe** — TypeScript รองรับ Type ให้อัตโนมัติ

---

## createFeatureSelector

ใช้ดึง Feature State จาก Root State

```typescript
import { createFeatureSelector } from '@ngrx/store';
import { ProductState } from './product.state';

// ดึง State ของ Feature 'product' จาก Root State
export const selectProductFeature =
  createFeatureSelector<ProductState>('product');

// Root State มีหน้าตาแบบนี้:
// {
//   product: ProductState,   <-- selectProductFeature ดึงส่วนนี้
//   counter: CounterState,
//   cart: CartState,
// }
```

---

## createSelector

```typescript
import { createSelector } from '@ngrx/store';

// Selector พื้นฐาน — รับ Feature State เป็น Input
export const selectAllProducts = createSelector(
  selectProductFeature,
  (state) => state.products
);

export const selectLoading = createSelector(
  selectProductFeature,
  (state) => state.loading
);

export const selectError = createSelector(
  selectProductFeature,
  (state) => state.error
);

// Selector รับ Input หลายตัว
export const selectProductById = (id: number) =>
  createSelector(
    selectAllProducts,
    (products) => products.find((p) => p.id === id)
  );
```

---

## Memoization ทำงานอย่างไร?

```typescript
// Selector นี้จะคำนวณใหม่เฉพาะเมื่อ products หรือ filter เปลี่ยน
export const selectFilteredProducts = createSelector(
  selectAllProducts,     // Input 1
  selectCurrentFilter,   // Input 2
  (products, filter) => {
    // ฟังก์ชันนี้จะถูกเรียกใหม่เฉพาะเมื่อ products หรือ filter เปลี่ยน
    // ถ้า Input ไม่เปลี่ยน จะคืนค่าเดิมจาก Cache
    return products.filter((p) => matchesFilter(p, filter));
  }
);
```

### ตัวอย่างการทดสอบ Memoization

```typescript
const state = {
  products: [
    { id: 1, name: 'สินค้า A', price: 100, category: 'Food' },
    { id: 2, name: 'สินค้า B', price: 200, category: 'Electronics' },
  ],
  filter: { category: 'Food' }
};

// เรียกครั้งแรก — คำนวณจริง
const result1 = selectFilteredProducts.projector(state.products, state.filter);

// เรียกครั้งที่สอง ด้วย Reference เดิม — ใช้ Cache
const result2 = selectFilteredProducts.projector(state.products, state.filter);

console.log(result1 === result2); // true — เป็น Object เดิม!
```

---

## Derived State

Derived State คือข้อมูลที่คำนวณจาก State อื่น ไม่ต้องเก็บไว้ใน Store

```typescript
export interface CartItem {
  productId: number;
  quantity: number;
  product?: Product;
}

export interface CartState {
  items: CartItem[];
}

// Selectors สำหรับ Cart
export const selectCartFeature = createFeatureSelector<CartState>('cart');
export const selectCartItems = createSelector(
  selectCartFeature,
  (state) => state.items
);

// Derived: จำนวนสินค้าทั้งหมดในตะกร้า
export const selectCartItemCount = createSelector(
  selectCartItems,
  (items) => items.reduce((sum, item) => sum + item.quantity, 0)
);

// Derived: ราคารวม (รวม Product ด้วย)
export const selectCartTotal = createSelector(
  selectCartItems,
  selectAllProducts,  // ดึงจาก Product Feature
  (items, products) => {
    return items.reduce((total, item) => {
      const product = products.find((p) => p.id === item.productId);
      return total + (product ? product.price * item.quantity : 0);
    }, 0);
  }
);

// Derived: ข้อมูล Cart พร้อมรายละเอียดสินค้า
export const selectCartItemsWithProducts = createSelector(
  selectCartItems,
  selectAllProducts,
  (items, products) =>
    items.map((item) => ({
      ...item,
      product: products.find((p) => p.id === item.productId),
    }))
);

// Derived: ตะกร้าว่างหรือไม่
export const selectIsCartEmpty = createSelector(
  selectCartItemCount,
  (count) => count === 0
);
```

---

## Workshop: Cart Selectors แบบสมบูรณ์

### State และ Models

```typescript
// cart.state.ts
export interface CartItem {
  productId: number;
  quantity: number;
  addedAt: Date;
}

export interface CartState {
  items: CartItem[];
  couponCode: string | null;
  discount: number;  // เปอร์เซ็นต์ส่วนลด
}

// product.state.ts
export interface Product {
  id: number;
  name: string;
  price: number;
  originalPrice: number;
  category: string;
  stock: number;
  imageUrl: string;
  isOnSale: boolean;
}
```

### Cart Selectors ทั้งหมด

```typescript
// cart.selectors.ts
import { createSelector, createFeatureSelector } from '@ngrx/store';
import { CartState } from './cart.state';
import { selectAllProducts } from '../product/product.selectors';

// Feature Selector
export const selectCartFeature = createFeatureSelector<CartState>('cart');

// Basic Selectors
export const selectCartItems = createSelector(
  selectCartFeature,
  (state) => state.items
);

export const selectCouponCode = createSelector(
  selectCartFeature,
  (state) => state.couponCode
);

export const selectDiscount = createSelector(
  selectCartFeature,
  (state) => state.discount
);

// Derived: จำนวนชิ้นทั้งหมด (รวม quantity)
export const selectTotalQuantity = createSelector(
  selectCartItems,
  (items) => items.reduce((sum, item) => sum + item.quantity, 0)
);

// Derived: จำนวนประเภทสินค้า (ไม่นับ quantity)
export const selectUniqueItemCount = createSelector(
  selectCartItems,
  (items) => items.length
);

// Derived: รายการ Cart พร้อมข้อมูลสินค้า
export const selectEnrichedCartItems = createSelector(
  selectCartItems,
  selectAllProducts,
  (items, products) =>
    items.map((item) => {
      const product = products.find((p) => p.id === item.productId);
      return {
        ...item,
        product,
        subtotal: product ? product.price * item.quantity : 0,
        isAvailable: product ? product.stock >= item.quantity : false,
      };
    }).filter((item) => item.product !== undefined)
);

// Derived: ราคาก่อนส่วนลด
export const selectSubtotal = createSelector(
  selectEnrichedCartItems,
  (items) => items.reduce((sum, item) => sum + item.subtotal, 0)
);

// Derived: จำนวนส่วนลด (บาท)
export const selectDiscountAmount = createSelector(
  selectSubtotal,
  selectDiscount,
  (subtotal, discount) => Math.round(subtotal * (discount / 100))
);

// Derived: ราคาหลังส่วนลด
export const selectTotal = createSelector(
  selectSubtotal,
  selectDiscountAmount,
  (subtotal, discount) => subtotal - discount
);

// Derived: ค่าจัดส่ง (ฟรีถ้าซื้อเกิน 500)
export const selectShippingFee = createSelector(
  selectTotal,
  (total) => total >= 500 ? 0 : 50
);

// Derived: ราคารวมทั้งหมด
export const selectGrandTotal = createSelector(
  selectTotal,
  selectShippingFee,
  (total, shipping) => total + shipping
);

// Derived: ตะกร้าว่างหรือไม่
export const selectIsCartEmpty = createSelector(
  selectUniqueItemCount,
  (count) => count === 0
);

// Derived: สินค้าที่สต็อกไม่พอ
export const selectUnavailableItems = createSelector(
  selectEnrichedCartItems,
  (items) => items.filter((item) => !item.isAvailable)
);

// Derived: สามารถ Checkout ได้หรือไม่
export const selectCanCheckout = createSelector(
  selectIsCartEmpty,
  selectUnavailableItems,
  (isEmpty, unavailable) => !isEmpty && unavailable.length === 0
);

// Derived: สรุปข้อมูล Cart ทั้งหมด
export const selectCartSummary = createSelector(
  selectEnrichedCartItems,
  selectSubtotal,
  selectDiscountAmount,
  selectTotal,
  selectShippingFee,
  selectGrandTotal,
  selectCouponCode,
  selectCanCheckout,
  (items, subtotal, discountAmount, total, shipping, grandTotal, coupon, canCheckout) => ({
    items,
    subtotal,
    discountAmount,
    total,
    shipping,
    grandTotal,
    coupon,
    canCheckout,
    itemCount: items.reduce((sum, item) => sum + item.quantity, 0),
  })
);
```

### Cart Component ที่ใช้ Selectors

```typescript
// cart.component.ts
import { Component, OnInit } from '@angular/core';
import { Store } from '@ngrx/store';
import { Observable } from 'rxjs';
import * as CartActions from '../store/cart/cart.actions';
import * as CartSelectors from '../store/cart/cart.selectors';

@Component({
  selector: 'app-cart',
  template: `
    <ng-container *ngIf="cartSummary$ | async as cart">
      <div class="cart-container">
        <h2>ตะกร้าสินค้า ({{ cart.itemCount }} ชิ้น)</h2>

        <div *ngIf="cart.items.length === 0" class="empty-cart">
          ตะกร้าว่างเปล่า
          <a routerLink="/products">ไปช้อปปิ้ง</a>
        </div>

        <div *ngFor="let item of cart.items" class="cart-item">
          <img [src]="item.product?.imageUrl" [alt]="item.product?.name" />
          <div class="item-info">
            <h3>{{ item.product?.name }}</h3>
            <div class="price">{{ item.product?.price | currency:'THB' }}</div>
            <div class="unavailable" *ngIf="!item.isAvailable">
              ⚠️ สต็อกไม่เพียงพอ
            </div>
          </div>
          <div class="quantity">
            <button (click)="decreaseQty(item.productId)">-</button>
            <span>{{ item.quantity }}</span>
            <button (click)="increaseQty(item.productId)">+</button>
          </div>
          <div class="subtotal">{{ item.subtotal | currency:'THB' }}</div>
          <button (click)="removeItem(item.productId)" class="remove">✕</button>
        </div>

        <div class="cart-summary" *ngIf="cart.items.length > 0">
          <div class="row">
            <span>ราคารวม</span>
            <span>{{ cart.subtotal | currency:'THB' }}</span>
          </div>
          <div class="row discount" *ngIf="cart.discountAmount > 0">
            <span>ส่วนลด ({{ cart.coupon }})</span>
            <span>-{{ cart.discountAmount | currency:'THB' }}</span>
          </div>
          <div class="row">
            <span>ค่าจัดส่ง</span>
            <span>{{ cart.shipping === 0 ? 'ฟรี' : (cart.shipping | currency:'THB') }}</span>
          </div>
          <div class="row total">
            <strong>ยอดรวมทั้งหมด</strong>
            <strong>{{ cart.grandTotal | currency:'THB' }}</strong>
          </div>

          <div class="coupon">
            <input #couponInput type="text" placeholder="รหัสคูปอง" />
            <button (click)="applyCoupon(couponInput.value)">ใช้คูปอง</button>
          </div>

          <button
            class="checkout-btn"
            [disabled]="!cart.canCheckout"
            (click)="checkout()"
          >
            ชำระเงิน
          </button>
        </div>
      </div>
    </ng-container>
  `
})
export class CartComponent implements OnInit {
  cartSummary$: Observable<any>;

  constructor(private store: Store) {}

  ngOnInit(): void {
    this.cartSummary$ = this.store.select(CartSelectors.selectCartSummary);
  }

  increaseQty(productId: number): void {
    this.store.dispatch(CartActions.increaseQuantity({ productId }));
  }

  decreaseQty(productId: number): void {
    this.store.dispatch(CartActions.decreaseQuantity({ productId }));
  }

  removeItem(productId: number): void {
    this.store.dispatch(CartActions.removeItem({ productId }));
  }

  applyCoupon(code: string): void {
    if (code) {
      this.store.dispatch(CartActions.applyCoupon({ code }));
    }
  }

  checkout(): void {
    this.store.dispatch(CartActions.checkout());
  }
}
```

---

## Parameterized Selectors

Selector ที่รับ Parameter เพิ่มเติม

```typescript
// วิธีที่ 1: Factory Function
export const selectProductById = (productId: number) =>
  createSelector(
    selectAllProducts,
    (products) => products.find((p) => p.id === productId)
  );

// การใช้งาน
this.product$ = this.store.select(selectProductById(this.productId));

// วิธีที่ 2: Props (เก่ากว่า ไม่แนะนำใน NgRx v15+)
// ใช้วิธีที่ 1 แทน

// วิธีที่ 3: ใช้ combineLatest กับ select
this.product$ = combineLatest([
  this.store.select(selectAllProducts),
  this.productId$,
]).pipe(
  map(([products, id]) => products.find((p) => p.id === id))
);
```

---

## Selector ที่ซับซ้อน

```typescript
// Selector ที่รับ Input มากถึง 8 ตัว
export const selectDashboardData = createSelector(
  selectAllProducts,
  selectAllOrders,
  selectAllUsers,
  selectCartSummary,
  selectCurrentUser,
  (products, orders, users, cart, currentUser) => ({
    totalProducts: products.length,
    totalOrders: orders.length,
    totalUsers: users.length,
    totalRevenue: orders.reduce((sum, o) => sum + o.total, 0),
    cartItemCount: cart.itemCount,
    userName: currentUser?.name || 'Guest',
    recentOrders: orders
      .filter((o) => o.userId === currentUser?.id)
      .slice(0, 5),
  })
);
```

---

## การ Reset Memoization

```typescript
// Reset Cache ของ Selector
selectFilteredProducts.release();

// ใช้ประโยชน์เมื่อ State เดิมกลับมาแต่ต้องการ Recalculate
```

---

## การทดสอบ Selectors

```typescript
// cart.selectors.spec.ts
import * as CartSelectors from './cart.selectors';
import { CartState } from './cart.state';

describe('CartSelectors', () => {
  const mockProducts = [
    { id: 1, name: 'สินค้า A', price: 100, stock: 10, category: 'A', originalPrice: 120, imageUrl: '', isOnSale: false },
    { id: 2, name: 'สินค้า B', price: 200, stock: 5, category: 'B', originalPrice: 200, imageUrl: '', isOnSale: false },
  ];

  const mockCartItems = [
    { productId: 1, quantity: 3, addedAt: new Date() },
    { productId: 2, quantity: 1, addedAt: new Date() },
  ];

  const mockState = {
    cart: {
      items: mockCartItems,
      couponCode: null,
      discount: 0,
    } as CartState,
    product: {
      products: mockProducts,
      selectedProductId: null,
      loading: false,
      error: null,
      filter: { category: null, minPrice: null, maxPrice: null, searchTerm: '' },
    },
  };

  it('selectTotalQuantity ควรคืนจำนวนชิ้นรวม', () => {
    const result = CartSelectors.selectTotalQuantity.projector(mockCartItems);
    expect(result).toBe(4); // 3 + 1
  });

  it('selectSubtotal ควรคำนวณราคารวมถูกต้อง', () => {
    const enriched = [
      { ...mockCartItems[0], product: mockProducts[0], subtotal: 300, isAvailable: true },
      { ...mockCartItems[1], product: mockProducts[1], subtotal: 200, isAvailable: true },
    ];
    const result = CartSelectors.selectSubtotal.projector(enriched);
    expect(result).toBe(500); // 300 + 200
  });

  it('selectShippingFee ควรเป็น 0 เมื่อซื้อเกิน 500', () => {
    expect(CartSelectors.selectShippingFee.projector(500)).toBe(0);
    expect(CartSelectors.selectShippingFee.projector(499)).toBe(50);
  });

  it('selectCanCheckout ควรเป็น false เมื่อตะกร้าว่าง', () => {
    const result = CartSelectors.selectCanCheckout.projector(true, []);
    expect(result).toBe(false);
  });

  it('selectCanCheckout ควรเป็น false เมื่อมีสินค้าที่ stock ไม่พอ', () => {
    const unavailableItems = [{ productId: 1, quantity: 100, isAvailable: false }];
    const result = CartSelectors.selectCanCheckout.projector(false, unavailableItems);
    expect(result).toBe(false);
  });

  it('selectCanCheckout ควรเป็น true เมื่อสินค้าพร้อม', () => {
    const result = CartSelectors.selectCanCheckout.projector(false, []);
    expect(result).toBe(true);
  });
});
```

---

## Best Practices

### 1. แยกไฟล์ Selectors

```
store/
├── product/
│   ├── product.actions.ts
│   ├── product.reducer.ts
│   ├── product.selectors.ts   ← แยกไฟล์เสมอ
│   └── product.state.ts
└── cart/
    ├── cart.selectors.ts
    └── ...
```

### 2. ตั้งชื่อตาม Convention

```typescript
// Feature Selector: selectXxxFeature หรือ selectXxxState
export const selectProductFeature = createFeatureSelector<ProductState>('product');

// Selector: selectXxx
export const selectAllProducts = createSelector(/* ... */);

// Selector ที่มี Derived Logic: selectXxx (ไม่ต้องเติม "derived")
export const selectFilteredProducts = createSelector(/* ... */);
```

### 3. ไม่ควร Subscribe ใน Component โดยตรง

```typescript
// ❌ หลีกเลี่ยง
this.store.select(selectAllProducts).subscribe((products) => {
  this.products = products;
});

// ✅ แนะนำ — ใช้ async pipe ใน template
products$ = this.store.select(selectAllProducts);
// ใน template: *ngFor="let p of products$ | async"
```

### 4. ใช้ Selector แทนการคำนวณใน Component

```typescript
// ❌ หลีกเลี่ยง
ngOnInit() {
  this.products$.subscribe((products) => {
    this.total = products.reduce((sum, p) => sum + p.price, 0);
  });
}

// ✅ แนะนำ — ย้ายไปอยู่ใน Selector
total$ = this.store.select(selectTotalPrice);
```

---

## สรุป

| API | การใช้งาน |
|----|-----------|
| `createFeatureSelector<T>(key)` | ดึง Feature State จาก Root State |
| `createSelector(...inputs, projector)` | สร้าง Derived Selector |
| `selector.projector(...)` | เรียกใช้ Projection Function โดยตรง (สำหรับ Test) |
| `selector.release()` | ล้าง Memoization Cache |

Selectors เป็นเครื่องมือสำคัญที่ช่วยให้ Components ไม่ต้องรู้โครงสร้างของ Store และยังช่วยเพิ่มประสิทธิภาพด้วย Memoization ใน Part ถัดไปจะเรียน NgRx Entity ซึ่งช่วยจัดการ Collections ได้ง่ายขึ้น
