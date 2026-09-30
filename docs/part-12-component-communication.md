# Part 12 — การสื่อสารระหว่าง Components

## บทนำ

Component Communication เป็นหัวใจสำคัญของ Angular application การเลือกวิธีการสื่อสารที่เหมาะสมส่งผลต่อความสามารถในการดูแลรักษาและทดสอบ code ได้ง่าย Angular มีหลายกลไกสำหรับการสื่อสาร ขึ้นอยู่กับความสัมพันธ์ระหว่าง components

---

## 12.1 @Input — ส่งข้อมูลจาก Parent ลง Child

`@Input()` decorator ใช้รับข้อมูลจาก parent component ผ่าน property binding

### การใช้งานพื้นฐาน

```typescript
// child: product-card.component.ts
import { Component, Input } from '@angular/core';
import { CommonModule } from '@angular/common';

export interface Product {
  id: number;
  name: string;
  price: number;
  imageUrl: string;
  category: string;
}

@Component({
  selector: 'app-product-card',
  standalone: true,
  imports: [CommonModule],
  template: `
    <div class="card">
      <img [src]="product.imageUrl" [alt]="product.name" class="card-img-top">
      <div class="card-body">
        <h5 class="card-title">{{ product.name }}</h5>
        <p class="text-primary fw-bold">฿{{ product.price | number }}</p>
        <span class="badge bg-secondary">{{ product.category }}</span>
      </div>
    </div>
  `
})
export class ProductCardComponent {
  @Input() product!: Product;  // required input
  @Input() showCategory = true;  // optional input with default
}
```

```typescript
// parent: product-list.component.ts
import { Component } from '@angular/core';
import { CommonModule } from '@angular/common';
import { ProductCardComponent, Product } from './product-card.component';

@Component({
  selector: 'app-product-list',
  standalone: true,
  imports: [CommonModule, ProductCardComponent],
  template: `
    <div class="row">
      <!-- ส่งข้อมูลผ่าน property binding -->
      <div *ngFor="let product of products" class="col-md-4 mb-3">
        <app-product-card
          [product]="product"
          [showCategory]="true"
        ></app-product-card>
      </div>
    </div>
  `
})
export class ProductListComponent {
  products: Product[] = [
    { id: 1, name: 'iPhone 15', price: 35900, imageUrl: '/img/iphone.jpg', category: 'Electronics' },
    { id: 2, name: 'MacBook Air', price: 42900, imageUrl: '/img/mac.jpg', category: 'Electronics' }
  ];
}
```

### Input Transform (Angular 16+)

```typescript
// Angular 16+ รองรับ transform function สำหรับ @Input
import { Component, Input, booleanAttribute, numberAttribute } from '@angular/core';

@Component({
  selector: 'app-button',
  standalone: true,
  template: `
    <button [disabled]="disabled" [style.font-size]="size + 'px'">
      {{ label }}
    </button>
  `
})
export class ButtonComponent {
  @Input() label = 'Click me';

  // แปลง string "true"/"false" เป็น boolean อัตโนมัติ
  @Input({ transform: booleanAttribute }) disabled = false;

  // แปลง string เป็น number อัตโนมัติ
  @Input({ transform: numberAttribute }) size = 14;
}
```

```html
<!-- parent template — ใช้ attribute binding แบบ HTML -->
<app-button label="บันทึก" disabled size="16"></app-button>
```

### ngOnChanges — ตรวจจับการเปลี่ยนแปลงของ Input

```typescript
import { Component, Input, OnChanges, SimpleChanges } from '@angular/core';

@Component({
  selector: 'app-user-card',
  standalone: true,
  template: `
    <div class="user-card">
      <p>{{ user?.name }}</p>
      <p>Loading: {{ isLoading }}</p>
    </div>
  `
})
export class UserCardComponent implements OnChanges {
  @Input() userId!: number;
  @Input() isLoading = false;

  user: any = null;

  ngOnChanges(changes: SimpleChanges): void {
    // ตรวจสอบว่า userId เปลี่ยนหรือไม่
    if (changes['userId'] && !changes['userId'].firstChange) {
      const prev = changes['userId'].previousValue;
      const curr = changes['userId'].currentValue;
      console.log(`userId changed from ${prev} to ${curr}`);
      this.loadUser(curr);
    }
  }

  private loadUser(id: number): void {
    // โหลดข้อมูล user ใหม่
    console.log('Loading user:', id);
  }
}
```

---

## 12.2 @Input required (Angular 16+)

Angular 16 เพิ่ม `required` option สำหรับ @Input ทำให้ TypeScript และ Angular compiler บังคับให้ต้องส่งค่า

```typescript
// Angular 16+ syntax
import { Component, Input } from '@angular/core';

@Component({
  selector: 'app-product-detail',
  standalone: true,
  template: `<h2>{{ product.name }}</h2>`
})
export class ProductDetailComponent {
  // required: true — ต้องส่งค่าเสมอ ไม่มี default
  @Input({ required: true }) product!: { id: number; name: string; price: number };

  // optional (default behavior)
  @Input() showActions = true;
}
```

```html
<!-- compile error ถ้าไม่ส่ง product -->
<app-product-detail [product]="selectedProduct"></app-product-detail>

<!-- Error: Required input 'product' is not bound -->
<app-product-detail></app-product-detail>
```

---

## 12.3 @Output + EventEmitter — ส่งข้อมูลจาก Child ขึ้น Parent

`@Output()` + `EventEmitter` ใช้ส่ง event จาก child component ขึ้นไปยัง parent

```typescript
// child: product-card.component.ts
import { Component, Input, Output, EventEmitter } from '@angular/core';
import { CommonModule } from '@angular/common';

@Component({
  selector: 'app-product-card',
  standalone: true,
  imports: [CommonModule],
  template: `
    <div class="card">
      <div class="card-body">
        <h5>{{ product.name }}</h5>
        <p>฿{{ product.price | number }}</p>
        <div class="d-flex gap-2">
          <button class="btn btn-primary btn-sm"
                  (click)="onAddToCart()">
            เพิ่มลงตะกร้า
          </button>
          <button class="btn btn-outline-danger btn-sm"
                  (click)="onWishlist()">
            {{ isWishlisted ? '♥' : '♡' }}
          </button>
          <button class="btn btn-outline-secondary btn-sm"
                  (click)="onViewDetail()">
            ดูรายละเอียด
          </button>
        </div>
      </div>
    </div>
  `
})
export class ProductCardComponent {
  @Input({ required: true }) product!: { id: number; name: string; price: number };

  // EventEmitter — ส่ง event พร้อมข้อมูล
  @Output() addToCart = new EventEmitter<{ productId: number; quantity: number }>();
  @Output() wishlistToggle = new EventEmitter<number>();  // ส่ง productId
  @Output() viewDetail = new EventEmitter<void>();        // ไม่มีข้อมูล

  isWishlisted = false;

  onAddToCart(): void {
    this.addToCart.emit({ productId: this.product.id, quantity: 1 });
  }

  onWishlist(): void {
    this.isWishlisted = !this.isWishlisted;
    this.wishlistToggle.emit(this.product.id);
  }

  onViewDetail(): void {
    this.viewDetail.emit();
  }
}
```

```typescript
// parent: shop.component.ts
import { Component } from '@angular/core';
import { CommonModule } from '@angular/common';
import { ProductCardComponent } from './product-card.component';

@Component({
  selector: 'app-shop',
  standalone: true,
  imports: [CommonModule, ProductCardComponent],
  template: `
    <div class="container">
      <div class="row">
        <div *ngFor="let product of products" class="col-md-4 mb-3">
          <app-product-card
            [product]="product"
            (addToCart)="handleAddToCart($event)"
            (wishlistToggle)="handleWishlist($event)"
            (viewDetail)="handleViewDetail(product)"
          ></app-product-card>
        </div>
      </div>

      <div class="mt-3">
        <p>สินค้าในตะกร้า: {{ cartCount }}</p>
        <p>สินค้าใน Wishlist: {{ wishlistItems.join(', ') }}</p>
      </div>
    </div>
  `
})
export class ShopComponent {
  products = [
    { id: 1, name: 'iPhone 15', price: 35900 },
    { id: 2, name: 'Galaxy S24', price: 28900 }
  ];

  cartCount = 0;
  wishlistItems: number[] = [];

  handleAddToCart(event: { productId: number; quantity: number }): void {
    this.cartCount += event.quantity;
    console.log(`เพิ่มสินค้า ${event.productId} ลงตะกร้า จำนวน ${event.quantity}`);
  }

  handleWishlist(productId: number): void {
    const index = this.wishlistItems.indexOf(productId);
    if (index > -1) {
      this.wishlistItems.splice(index, 1);
    } else {
      this.wishlistItems.push(productId);
    }
  }

  handleViewDetail(product: any): void {
    console.log('View detail:', product);
  }
}
```

---

## 12.4 Content Projection — ng-content

Content Projection ช่วยให้ component รับ HTML content จากภายนอก ทำให้ component มีความยืดหยุ่นสูง

### Basic Content Projection

```typescript
// card.component.ts
import { Component } from '@angular/core';

@Component({
  selector: 'app-card',
  standalone: true,
  template: `
    <div class="card">
      <div class="card-body">
        <!-- ng-content คือที่ "ฉีด" content จาก parent -->
        <ng-content></ng-content>
      </div>
    </div>
  `
})
export class CardComponent {}
```

```html
<!-- parent template -->
<app-card>
  <h5>ชื่อสินค้า</h5>
  <p>รายละเอียดสินค้า</p>
  <button>ซื้อเลย</button>
</app-card>
```

### Multi-slot Content Projection

```typescript
// modal.component.ts
import { Component, Input } from '@angular/core';
import { CommonModule } from '@angular/common';

@Component({
  selector: 'app-modal',
  standalone: true,
  imports: [CommonModule],
  template: `
    <div *ngIf="isOpen" class="modal d-block" style="background:rgba(0,0,0,.5)">
      <div class="modal-dialog">
        <div class="modal-content">

          <!-- Header slot -->
          <div class="modal-header">
            <h5 class="modal-title">
              <ng-content select="[modal-title]"></ng-content>
            </h5>
            <button class="btn-close" (click)="close()"></button>
          </div>

          <!-- Body slot -->
          <div class="modal-body">
            <ng-content select="[modal-body]"></ng-content>
          </div>

          <!-- Footer slot -->
          <div class="modal-footer">
            <ng-content select="[modal-footer]"></ng-content>

            <!-- Default footer ถ้าไม่มี slot -->
            <ng-content select="[modal-footer]"></ng-content>
            <button *ngIf="showDefaultClose" class="btn btn-secondary" (click)="close()">
              ปิด
            </button>
          </div>
        </div>
      </div>
    </div>
  `
})
export class ModalComponent {
  @Input() isOpen = false;
  @Input() showDefaultClose = true;
  @Output() closed = new EventEmitter<void>();

  close(): void {
    this.closed.emit();
  }
}
```

```html
<!-- parent template — ใช้ multi-slot -->
<app-modal [isOpen]="showDeleteModal" (closed)="showDeleteModal = false">
  <span modal-title>ยืนยันการลบ</span>

  <div modal-body>
    <p>คุณต้องการลบ <strong>{{ selectedItem?.name }}</strong> ใช่หรือไม่?</p>
    <p class="text-danger">การกระทำนี้ไม่สามารถย้อนกลับได้</p>
  </div>

  <div modal-footer>
    <button class="btn btn-secondary me-2" (click)="showDeleteModal = false">ยกเลิก</button>
    <button class="btn btn-danger" (click)="confirmDelete()">ลบ</button>
  </div>
</app-modal>
```

### Conditional Content Projection

```typescript
// alert.component.ts
import { Component, Input } from '@angular/core';
import { CommonModule } from '@angular/common';

@Component({
  selector: 'app-alert',
  standalone: true,
  imports: [CommonModule],
  template: `
    <div class="alert" [class]="'alert-' + type" role="alert">
      <strong *ngIf="title">{{ title }}:</strong>

      <!-- ng-content พร้อม fallback -->
      <ng-content>
        <!-- fallback content ถ้า parent ไม่ส่ง content มา -->
        <span>กรุณาระบุข้อความ</span>
      </ng-content>
    </div>
  `
})
export class AlertComponent {
  @Input() type: 'success' | 'danger' | 'warning' | 'info' = 'info';
  @Input() title = '';
}
```

---

## 12.5 ViewChild และ ViewChildren

`@ViewChild` และ `@ViewChildren` ใช้เข้าถึง child component หรือ DOM element โดยตรงจาก parent

### ViewChild — เข้าถึง Child Component

```typescript
// parent-with-viewchild.component.ts
import { Component, ViewChild, AfterViewInit, ElementRef } from '@angular/core';
import { CommonModule } from '@angular/common';

// ตัวอย่าง child component ที่มี public methods
@Component({
  selector: 'app-countdown',
  standalone: true,
  template: `
    <div class="countdown">
      <h2>{{ count }}</h2>
    </div>
  `
})
export class CountdownComponent {
  count = 10;
  private timer: any;

  start(): void {
    this.timer = setInterval(() => {
      if (this.count > 0) {
        this.count--;
      } else {
        this.stop();
      }
    }, 1000);
  }

  stop(): void {
    clearInterval(this.timer);
  }

  reset(): void {
    this.stop();
    this.count = 10;
  }
}

// parent component
@Component({
  selector: 'app-game',
  standalone: true,
  imports: [CommonModule, CountdownComponent],
  template: `
    <h1>เกมนับถอยหลัง</h1>
    <app-countdown></app-countdown>

    <div class="controls mt-3">
      <button class="btn btn-success me-2" (click)="startGame()">เริ่ม</button>
      <button class="btn btn-warning me-2" (click)="pauseGame()">หยุด</button>
      <button class="btn btn-secondary" (click)="resetGame()">รีเซ็ต</button>
    </div>

    <!-- เข้าถึง DOM element โดยตรง -->
    <input #nameInput type="text" placeholder="ชื่อผู้เล่น" class="form-control mt-3">
    <button class="btn btn-primary mt-2" (click)="focusInput()">Focus ช่องชื่อ</button>
  `
})
export class GameComponent implements AfterViewInit {
  // เข้าถึง child component ผ่าน type
  @ViewChild(CountdownComponent) countdown!: CountdownComponent;

  // เข้าถึง DOM element ผ่าน template reference variable
  @ViewChild('nameInput') nameInput!: ElementRef<HTMLInputElement>;

  ngAfterViewInit(): void {
    // ViewChild พร้อมใช้หลัง AfterViewInit
    console.log('Countdown component:', this.countdown);
    console.log('Name input element:', this.nameInput.nativeElement);
  }

  startGame(): void {
    this.countdown.start();
  }

  pauseGame(): void {
    this.countdown.stop();
  }

  resetGame(): void {
    this.countdown.reset();
  }

  focusInput(): void {
    this.nameInput.nativeElement.focus();
  }
}
```

### ViewChildren — เข้าถึง Child Components หลายตัว

```typescript
// tabs.component.ts
import { Component, ViewChildren, QueryList, AfterViewInit } from '@angular/core';
import { CommonModule } from '@angular/common';

@Component({
  selector: 'app-tab',
  standalone: true,
  template: `
    <div [class]="active ? 'd-block' : 'd-none'">
      <ng-content></ng-content>
    </div>
  `
})
export class TabComponent {
  @Input() label = '';
  active = false;
}

@Component({
  selector: 'app-tabs',
  standalone: true,
  imports: [CommonModule, TabComponent],
  template: `
    <ul class="nav nav-tabs">
      <li *ngFor="let tab of tabs; let i = index" class="nav-item">
        <button class="nav-link" [class.active]="tab.active" (click)="selectTab(i)">
          {{ tab.label }}
        </button>
      </li>
    </ul>
    <div class="tab-content p-3 border border-top-0">
      <ng-content></ng-content>
    </div>
  `
})
export class TabsComponent implements AfterViewInit {
  // รวบรวม QueryList ของ TabComponent ทั้งหมด
  @ViewChildren(TabComponent) tabs!: QueryList<TabComponent>;

  ngAfterViewInit(): void {
    // activate tab แรก
    if (this.tabs.length > 0) {
      this.tabs.first.active = true;
    }

    // ติดตามการเปลี่ยนแปลง (เมื่อ tabs ถูก add/remove แบบ dynamic)
    this.tabs.changes.subscribe(() => {
      console.log('Tabs changed:', this.tabs.length);
    });
  }

  selectTab(index: number): void {
    this.tabs.forEach((tab, i) => {
      tab.active = i === index;
    });
  }
}
```

---

## 12.6 ContentChild และ ContentChildren

`@ContentChild` และ `@ContentChildren` คล้ายกับ ViewChild แต่เข้าถึง content ที่ฉีดเข้ามาผ่าน ng-content

```typescript
// expandable-panel.component.ts
import { Component, ContentChild, ContentChildren, QueryList, AfterContentInit } from '@angular/core';
import { CommonModule } from '@angular/common';

@Component({
  selector: 'panel-header',
  standalone: true,
  template: `<ng-content></ng-content>`
})
export class PanelHeaderComponent {
  @Input() icon = '';
}

@Component({
  selector: 'panel-body',
  standalone: true,
  template: `<ng-content></ng-content>`
})
export class PanelBodyComponent {}

@Component({
  selector: 'app-expandable-panel',
  standalone: true,
  imports: [CommonModule],
  template: `
    <div class="panel border rounded">
      <div class="panel-header p-3 bg-light d-flex justify-content-between align-items-center"
           (click)="toggle()" style="cursor:pointer">
        <!-- แสดง projected header -->
        <ng-content select="panel-header"></ng-content>
        <span>{{ isExpanded ? '▲' : '▼' }}</span>
      </div>
      <div *ngIf="isExpanded" class="panel-body p-3">
        <ng-content select="panel-body"></ng-content>
      </div>
    </div>
  `
})
export class ExpandablePanelComponent implements AfterContentInit {
  // เข้าถึง projected content
  @ContentChild(PanelHeaderComponent) header!: PanelHeaderComponent;
  @ContentChildren(PanelBodyComponent) bodies!: QueryList<PanelBodyComponent>;

  isExpanded = true;

  ngAfterContentInit(): void {
    // ContentChild พร้อมหลัง AfterContentInit
    console.log('Header:', this.header);
    console.log('Bodies count:', this.bodies.length);
  }

  toggle(): void {
    this.isExpanded = !this.isExpanded;
  }
}
```

```html
<!-- parent template -->
<app-expandable-panel>
  <panel-header icon="🛒">ตะกร้าสินค้า (3 รายการ)</panel-header>
  <panel-body>
    <ul>
      <li>iPhone 15 — ฿35,900</li>
      <li>AirPods — ฿9,900</li>
    </ul>
  </panel-body>
</app-expandable-panel>
```

---

## 12.7 Service-based Communication

Service เป็นวิธีที่นิยมมากสำหรับการสื่อสารระหว่าง components ที่ไม่มีความสัมพันธ์กันโดยตรง (sibling หรือ unrelated)

### Shared State Service

```typescript
// services/cart.service.ts
import { Injectable } from '@angular/core';
import { BehaviorSubject, Observable } from 'rxjs';
import { map } from 'rxjs/operators';

export interface CartItem {
  productId: number;
  name: string;
  price: number;
  quantity: number;
  imageUrl?: string;
}

@Injectable({ providedIn: 'root' })
export class CartService {
  // BehaviorSubject เก็บ state และ emit ค่าล่าสุดให้ subscribers ใหม่
  private cartItemsSubject = new BehaviorSubject<CartItem[]>([]);

  // Observable สำหรับ subscribe
  cartItems$ = this.cartItemsSubject.asObservable();

  // Computed observables
  totalItems$: Observable<number> = this.cartItems$.pipe(
    map(items => items.reduce((sum, item) => sum + item.quantity, 0))
  );

  totalPrice$: Observable<number> = this.cartItems$.pipe(
    map(items => items.reduce((sum, item) => sum + item.price * item.quantity, 0))
  );

  // เข้าถึงค่าปัจจุบัน (synchronous)
  get cartItems(): CartItem[] {
    return this.cartItemsSubject.getValue();
  }

  addItem(item: Omit<CartItem, 'quantity'>, quantity = 1): void {
    const current = this.cartItems;
    const existingIndex = current.findIndex(i => i.productId === item.productId);

    if (existingIndex > -1) {
      // เพิ่มจำนวนถ้ามีอยู่แล้ว
      const updated = [...current];
      updated[existingIndex] = {
        ...updated[existingIndex],
        quantity: updated[existingIndex].quantity + quantity
      };
      this.cartItemsSubject.next(updated);
    } else {
      // เพิ่มรายการใหม่
      this.cartItemsSubject.next([...current, { ...item, quantity }]);
    }
  }

  removeItem(productId: number): void {
    const updated = this.cartItems.filter(i => i.productId !== productId);
    this.cartItemsSubject.next(updated);
  }

  updateQuantity(productId: number, quantity: number): void {
    if (quantity <= 0) {
      this.removeItem(productId);
      return;
    }

    const updated = this.cartItems.map(item =>
      item.productId === productId ? { ...item, quantity } : item
    );
    this.cartItemsSubject.next(updated);
  }

  clearCart(): void {
    this.cartItemsSubject.next([]);
  }
}
```

### การใช้งาน Cart Service ใน Multiple Components

```typescript
// components/cart-icon/cart-icon.component.ts
import { Component } from '@angular/core';
import { AsyncPipe } from '@angular/common';
import { CartService } from '../../services/cart.service';

@Component({
  selector: 'app-cart-icon',
  standalone: true,
  imports: [AsyncPipe],
  template: `
    <button class="btn position-relative">
      🛒
      <span *ngIf="(cartService.totalItems$ | async)! > 0"
            class="position-absolute top-0 start-100 translate-middle badge rounded-pill bg-danger">
        {{ cartService.totalItems$ | async }}
      </span>
    </button>
  `
})
export class CartIconComponent {
  constructor(public cartService: CartService) {}
}
```

```typescript
// components/cart-sidebar/cart-sidebar.component.ts
import { Component } from '@angular/core';
import { CommonModule, AsyncPipe } from '@angular/common';
import { CartService, CartItem } from '../../services/cart.service';

@Component({
  selector: 'app-cart-sidebar',
  standalone: true,
  imports: [CommonModule, AsyncPipe],
  template: `
    <div class="cart-sidebar">
      <h4>ตะกร้าสินค้า</h4>

      <div *ngFor="let item of cartService.cartItems$ | async" class="cart-item mb-2">
        <div class="d-flex justify-content-between">
          <span>{{ item.name }}</span>
          <div>
            <button class="btn btn-sm btn-outline-secondary"
                    (click)="updateQty(item, item.quantity - 1)">-</button>
            <span class="mx-2">{{ item.quantity }}</span>
            <button class="btn btn-sm btn-outline-secondary"
                    (click)="updateQty(item, item.quantity + 1)">+</button>
            <button class="btn btn-sm btn-outline-danger ms-2"
                    (click)="removeItem(item)">×</button>
          </div>
        </div>
        <small class="text-muted">฿{{ item.price | number }} × {{ item.quantity }} = ฿{{ item.price * item.quantity | number }}</small>
      </div>

      <hr>
      <div class="d-flex justify-content-between fw-bold">
        <span>รวม:</span>
        <span>฿{{ cartService.totalPrice$ | async | number }}</span>
      </div>

      <button class="btn btn-danger w-100 mt-2" (click)="cartService.clearCart()">
        ล้างตะกร้า
      </button>
    </div>
  `
})
export class CartSidebarComponent {
  constructor(public cartService: CartService) {}

  updateQty(item: CartItem, qty: number): void {
    this.cartService.updateQuantity(item.productId, qty);
  }

  removeItem(item: CartItem): void {
    this.cartService.removeItem(item.productId);
  }
}
```

---

## 12.8 RxJS Subject สำหรับ Cross-component Communication

`Subject` ใช้สำหรับการสื่อสารระหว่าง components ที่ไม่เกี่ยวข้องกัน (fully decoupled) โดยผ่าน service

### Event Bus Pattern

```typescript
// services/event-bus.service.ts
import { Injectable } from '@angular/core';
import { Subject, Observable, filter } from 'rxjs';

// กำหนด event types
export type AppEventType =
  | 'product:added-to-cart'
  | 'product:added-to-wishlist'
  | 'user:logged-in'
  | 'user:logged-out'
  | 'notification:show'
  | 'cart:updated';

export interface AppEvent<T = any> {
  type: AppEventType;
  payload?: T;
}

@Injectable({ providedIn: 'root' })
export class EventBusService {
  private eventSubject = new Subject<AppEvent>();

  // ส่ง event
  emit<T>(type: AppEventType, payload?: T): void {
    this.eventSubject.next({ type, payload });
  }

  // รับ event ทั้งหมด
  on(): Observable<AppEvent> {
    return this.eventSubject.asObservable();
  }

  // รับ event เฉพาะ type
  onEvent<T = any>(type: AppEventType): Observable<AppEvent<T>> {
    return this.eventSubject.pipe(
      filter(event => event.type === type)
    ) as Observable<AppEvent<T>>;
  }
}
```

### Notification Service

```typescript
// services/notification.service.ts
import { Injectable } from '@angular/core';
import { Subject, Observable } from 'rxjs';

export interface Notification {
  id: string;
  type: 'success' | 'error' | 'warning' | 'info';
  message: string;
  duration?: number;
}

@Injectable({ providedIn: 'root' })
export class NotificationService {
  private notificationSubject = new Subject<Notification>();

  notifications$: Observable<Notification> = this.notificationSubject.asObservable();

  success(message: string, duration = 3000): void {
    this.show({ type: 'success', message, duration });
  }

  error(message: string, duration = 5000): void {
    this.show({ type: 'error', message, duration });
  }

  warning(message: string, duration = 4000): void {
    this.show({ type: 'warning', message, duration });
  }

  info(message: string, duration = 3000): void {
    this.show({ type: 'info', message, duration });
  }

  private show(notification: Omit<Notification, 'id'>): void {
    this.notificationSubject.next({
      ...notification,
      id: crypto.randomUUID()
    });
  }
}
```

### Notification Toast Component

```typescript
// components/toast-container/toast-container.component.ts
import { Component, OnInit, OnDestroy } from '@angular/core';
import { CommonModule } from '@angular/common';
import { Subscription } from 'rxjs';
import { NotificationService, Notification } from '../../services/notification.service';

@Component({
  selector: 'app-toast-container',
  standalone: true,
  imports: [CommonModule],
  template: `
    <div class="toast-container position-fixed bottom-0 end-0 p-3" style="z-index:9999">
      <div *ngFor="let toast of toasts"
           class="toast show"
           [class]="'bg-' + toastBgClass(toast.type)"
           role="alert">
        <div class="toast-body text-white d-flex justify-content-between">
          <span>{{ toast.message }}</span>
          <button type="button" class="btn-close btn-close-white"
                  (click)="removeToast(toast.id)"></button>
        </div>
      </div>
    </div>
  `
})
export class ToastContainerComponent implements OnInit, OnDestroy {
  toasts: Notification[] = [];
  private subscription!: Subscription;

  constructor(private notificationService: NotificationService) {}

  ngOnInit(): void {
    this.subscription = this.notificationService.notifications$.subscribe(notification => {
      this.toasts.push(notification);

      // ลบออกอัตโนมัติหลัง duration
      if (notification.duration) {
        setTimeout(() => {
          this.removeToast(notification.id);
        }, notification.duration);
      }
    });
  }

  ngOnDestroy(): void {
    this.subscription.unsubscribe();
  }

  removeToast(id: string): void {
    this.toasts = this.toasts.filter(t => t.id !== id);
  }

  toastBgClass(type: string): string {
    const map: Record<string, string> = {
      success: 'success',
      error: 'danger',
      warning: 'warning',
      info: 'info'
    };
    return map[type] || 'secondary';
  }
}
```

---

## 12.9 Workshop: Shopping Cart Cross-component Communication

Workshop นี้สร้างระบบ shopping cart ที่มีการสื่อสารระหว่างหลาย components

### Product Catalog Component

```typescript
// catalog/product-catalog.component.ts
import { Component, OnInit, OnDestroy } from '@angular/core';
import { CommonModule } from '@angular/common';
import { Subject } from 'rxjs';
import { takeUntil } from 'rxjs/operators';
import { CartService } from '../services/cart.service';
import { NotificationService } from '../services/notification.service';

interface CatalogProduct {
  id: number;
  name: string;
  price: number;
  imageUrl: string;
  category: string;
  stock: number;
}

@Component({
  selector: 'app-product-catalog',
  standalone: true,
  imports: [CommonModule],
  template: `
    <div class="container-fluid">
      <div class="row">
        <!-- Product Grid -->
        <div class="col-lg-9">
          <h2 class="mb-3">สินค้าทั้งหมด</h2>

          <div class="row">
            <div *ngFor="let product of products" class="col-sm-6 col-md-4 col-xl-3 mb-4">
              <div class="card h-100 product-card">
                <img [src]="product.imageUrl" class="card-img-top"
                     style="height:180px; object-fit:cover"
                     [alt]="product.name">
                <div class="card-body d-flex flex-column">
                  <h6 class="card-title">{{ product.name }}</h6>
                  <p class="text-primary fw-bold mb-1">฿{{ product.price | number:'1.0-0' }}</p>
                  <small class="text-muted mb-2">คงเหลือ: {{ product.stock }} ชิ้น</small>
                  <div class="mt-auto">
                    <button
                      class="btn btn-primary btn-sm w-100"
                      [disabled]="product.stock === 0 || isInCart(product.id)"
                      (click)="addToCart(product)">
                      {{ product.stock === 0 ? 'สินค้าหมด' :
                         isInCart(product.id) ? 'อยู่ในตะกร้าแล้ว' : 'เพิ่มลงตะกร้า' }}
                    </button>
                  </div>
                </div>
              </div>
            </div>
          </div>
        </div>

        <!-- Cart Sidebar -->
        <div class="col-lg-3">
          <div class="sticky-top" style="top: 20px">
            <app-cart-summary></app-cart-summary>
          </div>
        </div>
      </div>
    </div>
  `
})
export class ProductCatalogComponent implements OnInit, OnDestroy {
  products: CatalogProduct[] = [
    { id: 1, name: 'iPhone 15 Pro', price: 49900, imageUrl: 'assets/iphone.jpg', category: 'Electronics', stock: 10 },
    { id: 2, name: 'Samsung Galaxy S24', price: 29900, imageUrl: 'assets/samsung.jpg', category: 'Electronics', stock: 5 },
    { id: 3, name: 'AirPods Pro 2', price: 9900, imageUrl: 'assets/airpods.jpg', category: 'Electronics', stock: 20 },
    { id: 4, name: 'iPad Air', price: 25900, imageUrl: 'assets/ipad.jpg', category: 'Electronics', stock: 8 },
    { id: 5, name: 'Nike Air Max', price: 4900, imageUrl: 'assets/nike.jpg', category: 'Clothing', stock: 15 },
    { id: 6, name: 'Adidas Ultraboost', price: 5900, imageUrl: 'assets/adidas.jpg', category: 'Clothing', stock: 0 }
  ];

  cartProductIds: number[] = [];
  private destroy$ = new Subject<void>();

  constructor(
    private cartService: CartService,
    private notificationService: NotificationService
  ) {}

  ngOnInit(): void {
    // ติดตาม cart เพื่ออัปเดต UI
    this.cartService.cartItems$.pipe(
      takeUntil(this.destroy$)
    ).subscribe(items => {
      this.cartProductIds = items.map(item => item.productId);
    });
  }

  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
  }

  isInCart(productId: number): boolean {
    return this.cartProductIds.includes(productId);
  }

  addToCart(product: CatalogProduct): void {
    this.cartService.addItem({
      productId: product.id,
      name: product.name,
      price: product.price,
      imageUrl: product.imageUrl
    });

    this.notificationService.success(`เพิ่ม "${product.name}" ลงตะกร้าแล้ว`);
  }
}
```

### Cart Summary Component

```typescript
// cart/cart-summary.component.ts
import { Component } from '@angular/core';
import { CommonModule, AsyncPipe } from '@angular/common';
import { CartService, CartItem } from '../services/cart.service';
import { NotificationService } from '../services/notification.service';

@Component({
  selector: 'app-cart-summary',
  standalone: true,
  imports: [CommonModule, AsyncPipe],
  template: `
    <div class="cart-summary card">
      <div class="card-header d-flex justify-content-between align-items-center">
        <h5 class="mb-0">🛒 ตะกร้าสินค้า</h5>
        <span class="badge bg-primary">{{ cartService.totalItems$ | async }} ชิ้น</span>
      </div>

      <div class="card-body p-0">
        <div *ngIf="(cartService.cartItems$ | async)?.length === 0" class="text-center p-4 text-muted">
          <p>ตะกร้าว่างเปล่า</p>
        </div>

        <ul class="list-group list-group-flush">
          <li *ngFor="let item of cartService.cartItems$ | async"
              class="list-group-item px-3 py-2">
            <div class="d-flex justify-content-between align-items-center">
              <div class="flex-grow-1 me-2">
                <small class="fw-semibold d-block">{{ item.name }}</small>
                <small class="text-muted">฿{{ item.price | number }}</small>
              </div>

              <div class="d-flex align-items-center gap-1">
                <button class="btn btn-outline-secondary btn-sm"
                        style="width:28px; height:28px; padding:0"
                        (click)="updateQty(item, item.quantity - 1)">-</button>
                <span class="mx-1 fw-bold" style="min-width:20px; text-align:center">
                  {{ item.quantity }}
                </span>
                <button class="btn btn-outline-secondary btn-sm"
                        style="width:28px; height:28px; padding:0"
                        (click)="updateQty(item, item.quantity + 1)">+</button>
                <button class="btn btn-outline-danger btn-sm ms-1"
                        style="width:28px; height:28px; padding:0"
                        (click)="removeItem(item)">×</button>
              </div>
            </div>
            <small class="text-muted">
              รวม: ฿{{ item.price * item.quantity | number }}
            </small>
          </li>
        </ul>
      </div>

      <div class="card-footer">
        <div class="d-flex justify-content-between mb-2">
          <span>ยอดรวม:</span>
          <strong class="text-primary fs-5">
            ฿{{ cartService.totalPrice$ | async | number:'1.0-0' }}
          </strong>
        </div>
        <button class="btn btn-success w-100 mb-2"
                [disabled]="(cartService.totalItems$ | async) === 0"
                (click)="checkout()">
          สั่งซื้อ
        </button>
        <button class="btn btn-outline-secondary w-100 btn-sm"
                [disabled]="(cartService.totalItems$ | async) === 0"
                (click)="clearCart()">
          ล้างตะกร้า
        </button>
      </div>
    </div>
  `
})
export class CartSummaryComponent {
  constructor(
    public cartService: CartService,
    private notificationService: NotificationService
  ) {}

  updateQty(item: CartItem, qty: number): void {
    this.cartService.updateQuantity(item.productId, qty);
  }

  removeItem(item: CartItem): void {
    this.cartService.removeItem(item.productId);
    this.notificationService.info(`ลบ "${item.name}" ออกจากตะกร้าแล้ว`);
  }

  clearCart(): void {
    this.cartService.clearCart();
    this.notificationService.warning('ล้างตะกร้าสินค้าแล้ว');
  }

  checkout(): void {
    const items = this.cartService.cartItems;
    console.log('Checkout with items:', items);
    this.notificationService.success('ดำเนินการสั่งซื้อเรียบร้อย!');
    this.cartService.clearCart();
  }
}
```

### Header Component

```typescript
// header/header.component.ts
import { Component } from '@angular/core';
import { AsyncPipe } from '@angular/common';
import { CartService } from '../services/cart.service';

@Component({
  selector: 'app-header',
  standalone: true,
  imports: [AsyncPipe],
  template: `
    <nav class="navbar navbar-expand-lg navbar-dark bg-dark">
      <div class="container">
        <a class="navbar-brand" href="#">MyShop</a>

        <div class="ms-auto d-flex align-items-center gap-3">
          <span class="text-light">
            ฿{{ cartService.totalPrice$ | async | number:'1.0-0' }}
          </span>

          <button class="btn btn-outline-light position-relative">
            🛒 ตะกร้า
            <span *ngIf="(cartService.totalItems$ | async)! > 0"
                  class="position-absolute top-0 start-100 translate-middle badge rounded-pill bg-danger">
              {{ cartService.totalItems$ | async }}
            </span>
          </button>
        </div>
      </div>
    </nav>
  `
})
export class HeaderComponent {
  constructor(public cartService: CartService) {}
}
```

### App Component ประกอบทุกอย่างเข้าด้วยกัน

```typescript
// app.component.ts
import { Component } from '@angular/core';
import { CommonModule } from '@angular/common';
import { HeaderComponent } from './header/header.component';
import { ProductCatalogComponent } from './catalog/product-catalog.component';
import { CartSummaryComponent } from './cart/cart-summary.component';
import { ToastContainerComponent } from './components/toast-container/toast-container.component';

@Component({
  selector: 'app-root',
  standalone: true,
  imports: [
    CommonModule,
    HeaderComponent,
    ProductCatalogComponent,
    CartSummaryComponent,
    ToastContainerComponent
  ],
  template: `
    <!-- Header แสดงข้อมูลตะกร้า (ผ่าน CartService) -->
    <app-header></app-header>

    <!-- Product Catalog พร้อม CartSummary sidebar -->
    <main class="mt-4">
      <app-product-catalog></app-product-catalog>
    </main>

    <!-- Toast Notifications (ผ่าน NotificationService) -->
    <app-toast-container></app-toast-container>
  `
})
export class AppComponent {}
```

---

## สรุปวิธีการสื่อสารและเมื่อควรใช้

| วิธี | ใช้เมื่อ |
|------|----------|
| `@Input` | ส่งข้อมูลจาก parent → child |
| `@Output + EventEmitter` | ส่ง event จาก child → parent |
| `ng-content` | ฉีด HTML จาก parent เข้า child |
| `ViewChild / ViewChildren` | parent เข้าถึง child component โดยตรง |
| `ContentChild / ContentChildren` | เข้าถึง projected content |
| `Service + BehaviorSubject` | share state ระหว่าง components ต่างๆ |
| `Subject (Event Bus)` | broadcast event แบบ one-to-many |

---

## สรุป Part 12

ใน Part นี้เราได้เรียนรู้:

- **@Input** — ส่งข้อมูลจาก parent ลง child รวมถึง transform และ required option (Angular 16+)
- **@Output + EventEmitter** — ส่ง events จาก child ขึ้น parent พร้อมข้อมูล
- **ng-content** — Content Projection สำหรับสร้าง flexible components
- **ViewChild / ViewChildren** — เข้าถึง child components และ DOM elements โดยตรง
- **ContentChild / ContentChildren** — เข้าถึง projected content
- **Service-based Communication** — ใช้ BehaviorSubject สำหรับ shared state
- **RxJS Subject** — Event Bus pattern สำหรับ decoupled communication
- **Workshop** — Shopping Cart ที่ใช้หลายวิธีผสมกัน

### แนวทางการเลือกวิธีสื่อสาร
1. สำหรับ parent-child โดยตรง → ใช้ @Input / @Output
2. สำหรับ shared state ที่หลาย components ต้องใช้ → ใช้ Service + BehaviorSubject
3. สำหรับ global events (notifications, auth) → ใช้ Subject
4. สำหรับสร้าง reusable layout components → ใช้ ng-content
5. สำหรับการควบคุม child โดยตรง → ใช้ ViewChild
