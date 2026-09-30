# Part 100: Capstone Project - E-Commerce App ครบวงจร

## ภาพรวมโปรเจค

สร้าง E-Commerce App ครบถ้วนที่รวมทุกสิ่งที่เรียนมา ตั้งแต่ Angular core จนถึง advanced patterns

---

## 1. Project Structure

```
src/app/
├── core/
│   ├── auth/          # JWT, guards, interceptors
│   ├── http/          # base service, error handler
│   ├── store/         # NgRx store
│   └── services/      # shared singletons
├── shared/
│   ├── components/    # reusable UI
│   ├── directives/    # custom directives
│   └── pipes/         # custom pipes
├── features/
│   ├── home/          # landing page
│   ├── products/      # product listing & detail
│   ├── cart/          # shopping cart
│   ├── checkout/      # payment flow
│   ├── orders/        # order history
│   └── account/       # user profile
└── app-routing.module.ts
```

---

## 2. Domain Models

```typescript
// core/models/product.model.ts
export interface Product {
  id: number;
  name: string;
  description: string;
  price: number;
  salePrice?: number;
  images: string[];
  thumbnail: string;
  category: Category;
  tags: string[];
  stock: number;
  rating: number;
  reviewCount: number;
  sku: string;
  weight?: number;
  dimensions?: { width: number; height: number; depth: number };
  createdAt: Date;
  updatedAt: Date;
}

export interface Category {
  id: number;
  name: string;
  slug: string;
  parentId?: number;
  children?: Category[];
}

export interface CartItem {
  product: Product;
  quantity: number;
  variant?: ProductVariant;
}

export interface ProductVariant {
  id: number;
  name: string;
  price: number;
  stock: number;
}

export interface Order {
  id: string;
  orderNumber: string;
  items: OrderItem[];
  status: OrderStatus;
  subtotal: number;
  shipping: number;
  tax: number;
  discount: number;
  total: number;
  shippingAddress: Address;
  billingAddress: Address;
  paymentMethod: string;
  notes?: string;
  createdAt: Date;
  updatedAt: Date;
  trackingNumber?: string;
}

export type OrderStatus =
  | 'pending'
  | 'confirmed'
  | 'processing'
  | 'shipped'
  | 'delivered'
  | 'cancelled'
  | 'refunded';

export interface Address {
  firstName: string;
  lastName: string;
  phone: string;
  street: string;
  district: string;
  province: string;
  postalCode: string;
  country: string;
}

export interface OrderItem {
  productId: number;
  productName: string;
  thumbnail: string;
  quantity: number;
  price: number;
  total: number;
}
```

---

## 3. NgRx Store

```typescript
// core/store/cart/cart.state.ts
export interface CartState {
  items: CartItem[];
  isLoading: boolean;
  couponCode: string | null;
  discount: number;
}

export const initialCartState: CartState = {
  items: [],
  isLoading: false,
  couponCode: null,
  discount: 0
};

// cart.actions.ts
import { createAction, props } from '@ngrx/store';

export const addToCart = createAction('[Cart] Add Item', props<{ item: CartItem }>());
export const removeFromCart = createAction('[Cart] Remove Item', props<{ productId: number }>());
export const updateQuantity = createAction('[Cart] Update Quantity', props<{ productId: number; quantity: number }>());
export const clearCart = createAction('[Cart] Clear');
export const applyCoupon = createAction('[Cart] Apply Coupon', props<{ code: string }>());
export const applyCouponSuccess = createAction('[Cart] Apply Coupon Success', props<{ discount: number; code: string }>());
export const applyCouponFail = createAction('[Cart] Apply Coupon Fail', props<{ error: string }>());

// cart.reducer.ts
import { createReducer, on } from '@ngrx/store';

export const cartReducer = createReducer(
  initialCartState,
  on(addToCart, (state, { item }) => {
    const existing = state.items.find(i => i.product.id === item.product.id);
    if (existing) {
      return {
        ...state,
        items: state.items.map(i =>
          i.product.id === item.product.id
            ? { ...i, quantity: i.quantity + item.quantity }
            : i
        )
      };
    }
    return { ...state, items: [...state.items, item] };
  }),
  on(removeFromCart, (state, { productId }) => ({
    ...state,
    items: state.items.filter(i => i.product.id !== productId)
  })),
  on(updateQuantity, (state, { productId, quantity }) => ({
    ...state,
    items: state.items.map(i =>
      i.product.id === productId ? { ...i, quantity } : i
    ).filter(i => i.quantity > 0)
  })),
  on(clearCart, state => ({ ...state, items: [], couponCode: null, discount: 0 })),
  on(applyCouponSuccess, (state, { discount, code }) => ({
    ...state, discount, couponCode: code
  }))
);

// cart.selectors.ts
import { createSelector } from '@ngrx/store';

export const selectCart = (state: any) => state.cart;

export const selectCartItems = createSelector(selectCart, s => s.items);
export const selectCartCount = createSelector(selectCartItems, items =>
  items.reduce((sum: number, i: CartItem) => sum + i.quantity, 0)
);
export const selectCartSubtotal = createSelector(selectCartItems, items =>
  items.reduce((sum: number, i: CartItem) => sum + (i.product.price * i.quantity), 0)
);
export const selectCartDiscount = createSelector(selectCart, s => s.discount);
export const selectCartTotal = createSelector(
  selectCartSubtotal,
  selectCartDiscount,
  (subtotal, discount) => subtotal - discount
);
```

---

## 4. Product Listing Feature

```typescript
// features/products/product-list/product-list.component.ts
import { Component, OnInit } from '@angular/core';
import { ActivatedRoute, Router } from '@angular/router';
import { Store } from '@ngrx/store';
import { Observable, combineLatest } from 'rxjs';
import { map, switchMap, tap } from 'rxjs/operators';
import { ProductService } from '../../../core/services/product.service';
import { addToCart } from '../../../core/store/cart/cart.actions';

@Component({
  selector: 'app-product-list',
  template: `
    <div class="products-page">
      <!-- Filters Sidebar -->
      <aside class="filters-sidebar">
        <h3>กรองสินค้า</h3>
        
        <div class="filter-group">
          <h4>ราคา</h4>
          <div class="price-range">
            <input type="number" [(ngModel)]="minPrice" placeholder="ต่ำสุด">
            <span>-</span>
            <input type="number" [(ngModel)]="maxPrice" placeholder="สูงสุด">
            <button (click)="applyPriceFilter()">ใช้</button>
          </div>
        </div>

        <div class="filter-group">
          <h4>หมวดหมู่</h4>
          <label *ngFor="let cat of categories">
            <input 
              type="checkbox"
              [checked]="selectedCategories.has(cat.id)"
              (change)="toggleCategory(cat.id)"
            >
            {{ cat.name }}
          </label>
        </div>

        <div class="filter-group">
          <h4>คะแนนรีวิว</h4>
          <label *ngFor="let r of [4, 3, 2, 1]">
            <input 
              type="radio" 
              name="rating"
              [value]="r"
              [(ngModel)]="minRating"
              (change)="applyFilters()"
            >
            {{ r }}+ ดาว
          </label>
        </div>
      </aside>

      <!-- Products Main -->
      <main class="products-main">
        <!-- Sort & View Options -->
        <div class="products-toolbar">
          <span class="product-count">{{ totalCount }} สินค้า</span>
          <select [(ngModel)]="sortBy" (ngModelChange)="applyFilters()">
            <option value="newest">ใหม่ล่าสุด</option>
            <option value="price_asc">ราคาต่ำ-สูง</option>
            <option value="price_desc">ราคาสูง-ต่ำ</option>
            <option value="popular">ยอดนิยม</option>
            <option value="rating">คะแนนสูง</option>
          </select>
          <div class="view-toggle">
            <button (click)="viewMode = 'grid'" [class.active]="viewMode === 'grid'">⊞</button>
            <button (click)="viewMode = 'list'" [class.active]="viewMode === 'list'">☰</button>
          </div>
        </div>

        <!-- Loading -->
        <app-skeleton-grid *ngIf="isLoading" [count]="12"></app-skeleton-grid>

        <!-- Products -->
        <div [class]="'products-' + viewMode" *ngIf="!isLoading">
          <app-product-card
            *ngFor="let product of products; trackBy: trackById"
            [product]="product"
            [viewMode]="viewMode"
            (addToCart)="onAddToCart($event)"
            (viewProduct)="navigateToProduct($event)"
          ></app-product-card>
        </div>

        <!-- Empty State -->
        <div class="empty-state" *ngIf="!isLoading && products.length === 0">
          <div class="empty-icon">🔍</div>
          <h3>ไม่พบสินค้า</h3>
          <p>ลองเปลี่ยนตัวกรองค้นหาใหม่</p>
          <button (click)="resetFilters()">ล้างตัวกรอง</button>
        </div>

        <!-- Pagination -->
        <app-pagination
          *ngIf="totalCount > pageSize"
          [currentPage]="currentPage"
          [totalItems]="totalCount"
          [pageSize]="pageSize"
          (pageChange)="onPageChange($event)"
        ></app-pagination>
      </main>
    </div>
  `
})
export class ProductListComponent implements OnInit {
  products: Product[] = [];
  categories: Category[] = [];
  isLoading = false;
  totalCount = 0;
  currentPage = 1;
  pageSize = 20;
  sortBy = 'newest';
  viewMode: 'grid' | 'list' = 'grid';
  minPrice = 0;
  maxPrice = 0;
  minRating = 0;
  selectedCategories = new Set<number>();

  constructor(
    private productService: ProductService,
    private store: Store,
    private route: ActivatedRoute,
    private router: Router
  ) {}

  ngOnInit(): void {
    this.route.queryParams.subscribe(params => {
      this.currentPage = +params['page'] || 1;
      this.sortBy = params['sort'] || 'newest';
      this.loadProducts();
    });
    this.loadCategories();
  }

  loadProducts(): void {
    this.isLoading = true;
    this.productService.getProducts({
      page: this.currentPage,
      limit: this.pageSize,
      sort: this.sortBy,
      minPrice: this.minPrice || undefined,
      maxPrice: this.maxPrice || undefined,
      minRating: this.minRating || undefined,
      categories: [...this.selectedCategories]
    }).subscribe({
      next: (result) => {
        this.products = result.items;
        this.totalCount = result.total;
        this.isLoading = false;
      },
      error: () => { this.isLoading = false; }
    });
  }

  loadCategories(): void {
    this.productService.getCategories().subscribe(cats => this.categories = cats);
  }

  onAddToCart(product: Product): void {
    this.store.dispatch(addToCart({
      item: { product, quantity: 1 }
    }));
  }

  navigateToProduct(product: Product): void {
    this.router.navigate(['/products', product.id]);
  }

  toggleCategory(categoryId: number): void {
    if (this.selectedCategories.has(categoryId)) {
      this.selectedCategories.delete(categoryId);
    } else {
      this.selectedCategories.add(categoryId);
    }
    this.applyFilters();
  }

  applyPriceFilter(): void { this.applyFilters(); }

  applyFilters(): void {
    this.currentPage = 1;
    this.router.navigate([], { queryParams: { page: 1, sort: this.sortBy } });
    this.loadProducts();
  }

  resetFilters(): void {
    this.minPrice = 0;
    this.maxPrice = 0;
    this.minRating = 0;
    this.selectedCategories.clear();
    this.sortBy = 'newest';
    this.loadProducts();
  }

  onPageChange(page: number): void {
    this.currentPage = page;
    this.router.navigate([], { queryParams: { page, sort: this.sortBy } });
    window.scrollTo(0, 0);
    this.loadProducts();
  }

  trackById = (i: number, p: Product) => p.id;
}
```

---

## 5. Checkout Flow

```typescript
// features/checkout/checkout.component.ts
import { Component, OnInit } from '@angular/core';
import { FormBuilder, FormGroup, Validators } from '@angular/forms';
import { Router } from '@angular/router';
import { Store } from '@ngrx/store';
import { selectCartItems, selectCartTotal } from '../../core/store/cart/cart.selectors';
import { clearCart } from '../../core/store/cart/cart.actions';
import { OrderService } from '../../core/services/order.service';
import { StripeService } from '../../core/payment/stripe.service';

type CheckoutStep = 'address' | 'shipping' | 'payment' | 'review' | 'success';

@Component({
  selector: 'app-checkout',
  template: `
    <div class="checkout-container">
      <!-- Progress Steps -->
      <div class="checkout-steps">
        <div 
          *ngFor="let step of steps; let i = index"
          class="step"
          [class.active]="currentStep === step.id"
          [class.completed]="isStepCompleted(step.id)"
        >
          <div class="step-icon">
            <span *ngIf="!isStepCompleted(step.id)">{{ i + 1 }}</span>
            <span *ngIf="isStepCompleted(step.id)">✓</span>
          </div>
          <span class="step-label">{{ step.label }}</span>
        </div>
      </div>

      <div class="checkout-content">
        <!-- Step 1: Shipping Address -->
        <div *ngIf="currentStep === 'address'">
          <h2>ที่อยู่จัดส่ง</h2>
          <form [formGroup]="addressForm" (ngSubmit)="nextStep()">
            <div class="form-row">
              <label>
                ชื่อ
                <input formControlName="firstName" placeholder="ชื่อ">
                <span class="error" *ngIf="addressForm.get('firstName')?.invalid && addressForm.get('firstName')?.touched">
                  กรุณากรอกชื่อ
                </span>
              </label>
              <label>
                นามสกุล
                <input formControlName="lastName" placeholder="นามสกุล">
              </label>
            </div>
            <label>
              เบอร์โทรศัพท์
              <input formControlName="phone" placeholder="0xx-xxx-xxxx">
            </label>
            <label>
              ที่อยู่
              <input formControlName="street" placeholder="บ้านเลขที่ ซอย ถนน">
            </label>
            <div class="form-row">
              <label>
                แขวง/ตำบล
                <input formControlName="district" placeholder="แขวง/ตำบล">
              </label>
              <label>
                เขต/อำเภอ
                <input formControlName="province" placeholder="เขต/อำเภอ">
              </label>
            </div>
            <label>
              รหัสไปรษณีย์
              <input formControlName="postalCode" placeholder="10xxx">
            </label>
            <button type="submit" [disabled]="addressForm.invalid">ถัดไป</button>
          </form>
        </div>

        <!-- Step 2: Payment -->
        <div *ngIf="currentStep === 'payment'">
          <h2>วิธีชำระเงิน</h2>
          
          <div class="payment-methods">
            <label 
              *ngFor="let method of paymentMethods"
              class="payment-method"
              [class.selected]="selectedPayment === method.id"
            >
              <input 
                type="radio" 
                [value]="method.id"
                [(ngModel)]="selectedPayment"
              >
              <img [src]="method.icon" [alt]="method.name">
              {{ method.name }}
            </label>
          </div>

          <app-payment-form 
            *ngIf="selectedPayment === 'credit_card'"
            (paymentComplete)="onPaymentComplete($event)"
          ></app-payment-form>

          <button (click)="nextStep()" *ngIf="selectedPayment !== 'credit_card'">
            ยืนยันการชำระเงิน
          </button>
        </div>

        <!-- Step 3: Review -->
        <div *ngIf="currentStep === 'review'">
          <h2>ตรวจสอบคำสั่งซื้อ</h2>
          
          <div class="order-summary">
            <div *ngFor="let item of cartItems$ | async" class="order-item">
              <img [src]="item.product.thumbnail" [alt]="item.product.name">
              <div class="item-details">
                <p>{{ item.product.name }}</p>
                <p>{{ item.quantity }} x ฿{{ item.product.price | number }}</p>
              </div>
              <p class="item-total">฿{{ item.product.price * item.quantity | number }}</p>
            </div>
          </div>

          <div class="total-summary">
            <div class="row">
              <span>ราคาสินค้า</span>
              <span>฿{{ (cartTotal$ | async) | number }}</span>
            </div>
            <div class="row">
              <span>ค่าจัดส่ง</span>
              <span>฿{{ shippingCost | number }}</span>
            </div>
            <div class="row total">
              <strong>รวมทั้งหมด</strong>
              <strong>฿{{ ((cartTotal$ | async) || 0) + shippingCost | number }}</strong>
            </div>
          </div>

          <button (click)="placeOrder()" [disabled]="isPlacingOrder">
            {{ isPlacingOrder ? 'กำลังดำเนินการ...' : 'ยืนยันคำสั่งซื้อ' }}
          </button>
        </div>

        <!-- Step 4: Success -->
        <div class="success-page" *ngIf="currentStep === 'success'">
          <div class="success-icon">✅</div>
          <h2>สั่งซื้อสำเร็จ!</h2>
          <p>หมายเลขคำสั่งซื้อ: <strong>{{ orderId }}</strong></p>
          <p>อีเมลยืนยันจะถูกส่งไปที่อีเมลของคุณ</p>
          <div class="success-actions">
            <button routerLink="/orders">ติดตามสินค้า</button>
            <button routerLink="/products">ซื้อต่อ</button>
          </div>
        </div>
      </div>
    </div>
  `
})
export class CheckoutComponent implements OnInit {
  currentStep: CheckoutStep = 'address';
  steps = [
    { id: 'address', label: 'ที่อยู่' },
    { id: 'payment', label: 'ชำระเงิน' },
    { id: 'review', label: 'ตรวจสอบ' },
  ];
  completedSteps = new Set<CheckoutStep>();
  
  cartItems$ = this.store.select(selectCartItems);
  cartTotal$ = this.store.select(selectCartTotal);
  shippingCost = 50;
  
  addressForm: FormGroup;
  selectedPayment = 'credit_card';
  paymentMethods = [
    { id: 'credit_card', name: 'บัตรเครดิต/เดบิต', icon: '/assets/icons/credit-card.svg' },
    { id: 'promptpay', name: 'พร้อมเพย์', icon: '/assets/icons/promptpay.svg' },
    { id: 'cod', name: 'เก็บเงินปลายทาง', icon: '/assets/icons/cod.svg' },
  ];
  
  isPlacingOrder = false;
  orderId = '';

  constructor(
    private fb: FormBuilder,
    private store: Store,
    private orderService: OrderService,
    private router: Router
  ) {
    this.addressForm = this.fb.group({
      firstName: ['', Validators.required],
      lastName: ['', Validators.required],
      phone: ['', [Validators.required, Validators.pattern(/^0\d{8,9}$/)]],
      street: ['', Validators.required],
      district: ['', Validators.required],
      province: ['', Validators.required],
      postalCode: ['', [Validators.required, Validators.pattern(/^\d{5}$/)]]
    });
  }

  ngOnInit(): void {}

  nextStep(): void {
    this.completedSteps.add(this.currentStep);
    const stepOrder: CheckoutStep[] = ['address', 'payment', 'review', 'success'];
    const idx = stepOrder.indexOf(this.currentStep);
    if (idx < stepOrder.length - 1) {
      this.currentStep = stepOrder[idx + 1];
    }
  }

  isStepCompleted(stepId: string): boolean {
    return this.completedSteps.has(stepId as CheckoutStep);
  }

  onPaymentComplete(event: { success: boolean; paymentId?: string }): void {
    if (event.success) {
      this.nextStep();
    }
  }

  async placeOrder(): Promise<void> {
    this.isPlacingOrder = true;
    try {
      const order = await this.orderService.createOrder({
        shippingAddress: this.addressForm.value,
        paymentMethod: this.selectedPayment
      }).toPromise();
      
      this.orderId = order?.orderNumber || '';
      this.store.dispatch(clearCart());
      this.currentStep = 'success';
    } catch (error) {
      console.error('Order failed:', error);
    } finally {
      this.isPlacingOrder = false;
    }
  }
}
```

---

## 6. App Routing

```typescript
// app-routing.module.ts
const routes: Routes = [
  { path: '', loadChildren: () => import('./features/home/home.module').then(m => m.HomeModule) },
  { path: 'products', loadChildren: () => import('./features/products/products.module').then(m => m.ProductsModule) },
  { path: 'cart', loadChildren: () => import('./features/cart/cart.module').then(m => m.CartModule) },
  {
    path: 'checkout',
    loadChildren: () => import('./features/checkout/checkout.module').then(m => m.CheckoutModule),
    canActivate: [AuthGuard, CartNotEmptyGuard]
  },
  {
    path: 'orders',
    loadChildren: () => import('./features/orders/orders.module').then(m => m.OrdersModule),
    canActivate: [AuthGuard]
  },
  {
    path: 'account',
    loadChildren: () => import('./features/account/account.module').then(m => m.AccountModule),
    canActivate: [AuthGuard]
  },
  { path: 'login', loadChildren: () => import('./features/auth/auth.module').then(m => m.AuthModule) },
  { path: '**', redirectTo: '' }
];
```

---

## 7. สรุป Capstone Project

### Feature ที่ครอบคลุม

| Feature | Part ที่ใช้ |
|---------|-----------|
| Component Architecture | 71-76 |
| State Management (NgRx) | 81-83 |
| Authentication (JWT/SSO) | 91-92 |
| Payment Integration | 93 |
| Push Notifications | 94 |
| Accessibility | 71 |
| Performance Optimization | 79-80 |
| Analytics & Monitoring | 86-87 |
| Offline Support | 78 |
| Testing | 98 |

### Deployment Checklist

- [ ] Build production (`ng build --prod`)
- [ ] Environment variables ตั้งค่าถูกต้อง
- [ ] HTTPS enabled
- [ ] PWA configured
- [ ] Error monitoring (Sentry)
- [ ] Analytics configured (GA4)
- [ ] Performance budget ผ่าน
- [ ] Accessibility audit ผ่าน
- [ ] Security headers configured
- [ ] Lighthouse score > 90

---

## ยินดีด้วย! คุณเรียนจบหลักสูตร Angular แล้ว 🎉

จากหน้าแรกถึงหน้านี้ คุณได้เรียนรู้:
- Angular Fundamentals ถึง Advanced Patterns
- Performance, Security, Testing
- Real-world integrations (Auth, Payment, AI)
- Enterprise Architecture
- Mobile, Desktop, PWA

**ก้าวต่อไป**: ฝึกสร้าง project จริง, contribute open source, share knowledge
