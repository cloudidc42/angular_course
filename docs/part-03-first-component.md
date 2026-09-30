# Part 03 — สร้าง Component แรก

## เนื้อหาในบทนี้
1. Component คืออะไร?
2. Anatomy ของ Component
3. สร้าง Component ด้วย CLI
4. Component Metadata
5. Interpolation และ Expressions
6. Property Binding
7. Event Binding
8. Two-Way Binding
9. Template Reference Variables
10. Workshop: Product Card Component

---

## 1. Component คืออะไร?

**Component** คือ building block พื้นฐานของ Angular แต่ละ Component ประกอบด้วย:
- **Template** — HTML ที่แสดงผล (View)
- **Class** — TypeScript ที่จัดการ logic
- **Styles** — CSS/SCSS สำหรับ Component นั้นๆ

```
┌─────────────────────────────────────────┐
│              Component                   │
│                                         │
│  ┌──────────────┐  ┌──────────────────┐ │
│  │   Template   │  │      Class       │ │
│  │   (View)     │◄─┤  (Controller)    │ │
│  │   HTML       │  │  TypeScript      │ │
│  └──────────────┘  └──────────────────┘ │
│         │                               │
│  ┌──────────────┐                       │
│  │    Styles    │                       │
│  │   CSS/SCSS   │                       │
│  └──────────────┘                       │
└─────────────────────────────────────────┘
```

### Component Tree

Angular App เป็น tree ของ Components:

```
AppComponent (root)
├── NavbarComponent
│   ├── LogoComponent
│   └── NavMenuComponent
├── MainContentComponent
│   ├── ProductListComponent
│   │   ├── ProductCardComponent (x10)
│   │   └── PaginationComponent
│   └── SidebarComponent
│       ├── FilterComponent
│       └── CategoryListComponent
└── FooterComponent
```

---

## 2. Anatomy ของ Component

```typescript
import { Component, OnInit } from '@angular/core';
import { CommonModule } from '@angular/common';

// @Component Decorator — ระบุ metadata
@Component({
  // ────────────────────────────────────────────────
  // selector: ชื่อ HTML tag ที่ใช้เรียก Component นี้
  selector: 'app-example',
  
  // standalone: ไม่ต้องอยู่ใน NgModule (Angular 14+)
  standalone: true,
  
  // imports: Modules/Components ที่ใช้ใน template
  imports: [CommonModule],
  
  // template: HTML inline
  template: `<h1>{{ title }}</h1>`,
  // หรือ
  // templateUrl: ชี้ไปที่ HTML file
  templateUrl: './example.component.html',
  
  // styles: CSS inline (array of strings)
  styles: [`h1 { color: red; }`],
  // หรือ
  // styleUrls: ชี้ไปที่ CSS/SCSS files
  styleUrl: './example.component.scss',
  
  // changeDetection: กลยุทธ์การตรวจจับการเปลี่ยนแปลง
  // changeDetection: ChangeDetectionStrategy.OnPush,
})
export class ExampleComponent implements OnInit {
  // Properties
  title = 'Example Component';
  
  // Constructor — ใช้สำหรับ Dependency Injection
  constructor() {}
  
  // ngOnInit — เรียกเมื่อ Component ถูก initialize
  ngOnInit(): void {
    // โค้ดที่ต้องรันเมื่อเริ่มต้น
  }
}
```

---

## 3. สร้าง Component ด้วย CLI

```bash
# สร้าง Component พื้นฐาน
ng generate component user-card
# หรือ
ng g c user-card

# สร้างในโฟลเดอร์ย่อย
ng g c components/user-card

# สร้างแบบ Standalone (Angular 14+)
ng g c user-card --standalone

# สร้างแบบ inline template และ styles
ng g c user-card --inline-template --inline-style

# สร้างโดยไม่มี test file
ng g c user-card --skip-tests

# สร้างโดยไม่เพิ่มใน module
ng g c user-card --skip-import

# ตัวอย่างการสร้าง Component หลายๆ อัน
ng g c components/header --standalone --skip-tests
ng g c components/footer --standalone --skip-tests
ng g c components/sidebar --standalone --skip-tests
ng g c features/products/product-list --standalone
ng g c features/products/product-card --standalone
ng g c features/products/product-detail --standalone
ng g c shared/ui/button --standalone
ng g c shared/ui/card --standalone
ng g c shared/ui/modal --standalone
```

### ไฟล์ที่ถูกสร้าง

```bash
CREATE src/app/user-card/user-card.component.html (24 bytes)
CREATE src/app/user-card/user-card.component.scss (0 bytes)
CREATE src/app/user-card/user-card.component.spec.ts (607 bytes)
CREATE src/app/user-card/user-card.component.ts (232 bytes)
```

---

## 4. Component Metadata

### Selector Types

```typescript
// Element Selector (แนะนำ)
@Component({ selector: 'app-button' })
// ใช้งาน: <app-button></app-button>

// Attribute Selector
@Component({ selector: '[appButton]' })
// ใช้งาน: <div appButton></div>

// Class Selector (ไม่แนะนำ)
@Component({ selector: '.app-button' })
// ใช้งาน: <div class="app-button"></div>
```

### Template Options

```typescript
// Inline Template — เหมาะกับ Component เล็กๆ
@Component({
  selector: 'app-badge',
  standalone: true,
  template: `
    <span class="badge" [class]="'badge-' + type">
      {{ label }}
    </span>
  `,
  styles: [`
    .badge {
      padding: 2px 8px;
      border-radius: 12px;
      font-size: 0.75rem;
    }
    .badge-success { background: #4caf50; color: white; }
    .badge-danger { background: #f44336; color: white; }
  `]
})
export class BadgeComponent {
  type = 'success';
  label = 'Active';
}

// External Files — เหมาะกับ Component ซับซ้อน
@Component({
  selector: 'app-product-list',
  standalone: true,
  imports: [CommonModule],
  templateUrl: './product-list.component.html',
  styleUrl: './product-list.component.scss'
})
export class ProductListComponent {}
```

---

## 5. Interpolation และ Expressions

**Interpolation** ใช้ `{{ expression }}` เพื่อแสดงค่าใน template

```typescript
@Component({
  selector: 'app-interpolation-demo',
  standalone: true,
  imports: [CommonModule],
  template: `
    <!-- Basic Interpolation -->
    <h1>{{ title }}</h1>
    <p>{{ description }}</p>
    
    <!-- Expressions -->
    <p>2 + 2 = {{ 2 + 2 }}</p>
    <p>{{ firstName + ' ' + lastName }}</p>
    
    <!-- Method Calls -->
    <p>{{ getFullName() }}</p>
    <p>{{ formatDate(birthDate) }}</p>
    
    <!-- Ternary -->
    <p>{{ isLoggedIn ? 'เข้าสู่ระบบแล้ว' : 'ยังไม่ได้เข้าสู่ระบบ' }}</p>
    
    <!-- Nullish Coalescing -->
    <p>{{ username ?? 'ผู้ใช้ทั่วไป' }}</p>
    
    <!-- Optional Chaining -->
    <p>{{ user?.profile?.avatar ?? 'ไม่มีรูป' }}</p>
    
    <!-- Array/Object -->
    <p>จำนวนสินค้า: {{ products.length }}</p>
    <p>ชื่อสินค้าแรก: {{ products[0]?.name }}</p>
    
    <!-- Pipe -->
    <p>วันที่: {{ today | date:'dd/MM/yyyy' }}</p>
    <p>ราคา: {{ price | currency:'THB':'symbol':'1.2-2' }}</p>
    <p>ตัวพิมพ์ใหญ่: {{ title | uppercase }}</p>
    
    <!-- ไม่สามารถใช้ใน Interpolation -->
    <!-- ❌ {{ let x = 5 }}          — ไม่มี assignment -->
    <!-- ❌ {{ if (x > 0) {} }}      — ไม่มี if statement -->
    <!-- ❌ {{ new Date() }}         — ไม่มี new keyword -->
    <!-- ❌ {{ console.log('hi') }}  — ไม่มี side effects -->
  `
})
export class InterpolationDemoComponent {
  title = 'Angular Interpolation';
  description = 'ตัวอย่างการใช้ Interpolation';
  firstName = 'สมชาย';
  lastName = 'ใจดี';
  isLoggedIn = true;
  username: string | null = null;
  user = { profile: { avatar: 'avatar.jpg' } };
  products = [
    { name: 'สินค้า A', price: 100 },
    { name: 'สินค้า B', price: 200 }
  ];
  today = new Date();
  price = 1234.56;
  birthDate = new Date('1990-01-15');
  
  getFullName(): string {
    return `${this.firstName} ${this.lastName}`;
  }
  
  formatDate(date: Date): string {
    return date.toLocaleDateString('th-TH');
  }
}
```

---

## 6. Property Binding

**Property Binding** ใช้ `[property]="expression"` เพื่อ bind ค่าไปยัง DOM property

```typescript
@Component({
  selector: 'app-property-binding',
  standalone: true,
  template: `
    <!-- src attribute -->
    <img [src]="imageUrl" [alt]="imageAlt" />
    
    <!-- href attribute -->
    <a [href]="websiteUrl">Visit</a>
    
    <!-- disabled state -->
    <button [disabled]="isLoading">{{ isLoading ? 'กำลังโหลด...' : 'บันทึก' }}</button>
    
    <!-- class binding -->
    <div [class]="containerClass">Content</div>
    <div [class.active]="isActive">Will have class 'active' when isActive=true</div>
    <div [class.hidden]="!isVisible">Hidden when !isVisible</div>
    
    <!-- style binding -->
    <div [style.color]="textColor">Colored text</div>
    <div [style.fontSize.px]="fontSize">Sized text</div>
    <div [style.background-color]="bgColor">Background</div>
    
    <!-- ngClass directive -->
    <div [ngClass]="{'active': isActive, 'disabled': isDisabled, 'highlight': isHighlighted}">
      Multiple classes
    </div>
    <div [ngClass]="getClasses()">Dynamic classes</div>
    
    <!-- ngStyle directive -->
    <div [ngStyle]="{'color': textColor, 'font-size': fontSize + 'px'}">
      Multiple styles
    </div>
    
    <!-- input value -->
    <input [value]="inputValue" />
    
    <!-- textarea value -->
    <textarea [value]="textContent"></textarea>
    
    <!-- select value -->
    <select [value]="selectedOption">
      <option value="a">A</option>
      <option value="b">B</option>
    </select>
    
    <!-- innerText vs innerHTML -->
    <span [innerText]="safeText"></span>
    <div [innerHTML]="htmlContent"></div>  <!-- ระวัง XSS! -->
    
    <!-- Component input -->
    <app-badge [type]="badgeType" [label]="badgeLabel" />
  `
})
export class PropertyBindingComponent {
  imageUrl = '/assets/images/product.jpg';
  imageAlt = 'Product Image';
  websiteUrl = 'https://angular.io';
  isLoading = false;
  containerClass = 'container main-content';
  isActive = true;
  isVisible = true;
  isDisabled = false;
  isHighlighted = true;
  textColor = '#1976d2';
  fontSize = 16;
  bgColor = '#e3f2fd';
  inputValue = 'Default value';
  textContent = 'Text content';
  selectedOption = 'a';
  safeText = 'Safe text content';
  htmlContent = '<strong>Bold</strong> text';
  badgeType = 'success';
  badgeLabel = 'Active';
  
  getClasses(): { [key: string]: boolean } {
    return {
      'active': this.isActive,
      'disabled': this.isDisabled,
      'highlight': this.isHighlighted
    };
  }
}
```

---

## 7. Event Binding

**Event Binding** ใช้ `(event)="handler()"` เพื่อ listen DOM events

```typescript
@Component({
  selector: 'app-event-binding',
  standalone: true,
  template: `
    <!-- Click Events -->
    <button (click)="onClick()">Click Me</button>
    <button (click)="onClickWithEvent($event)">Click with Event</button>
    <button (click)="count = count + 1">Inline: {{ count }}</button>
    
    <!-- Mouse Events -->
    <div
      (mouseenter)="onMouseEnter()"
      (mouseleave)="onMouseLeave()"
      (mousemove)="onMouseMove($event)"
      class="hover-area"
      [style.background]="isHovered ? '#e3f2fd' : 'white'"
    >
      Hover over me!
      <span *ngIf="isHovered">Mouse at: {{ mouseX }}, {{ mouseY }}</span>
    </div>
    
    <!-- Keyboard Events -->
    <input
      (keyup)="onKeyUp($event)"
      (keydown)="onKeyDown($event)"
      (keydown.enter)="onEnterPress()"
      (keydown.escape)="onEscapePress()"
      placeholder="พิมพ์บางอย่าง"
    />
    <p>พิมพ์ล่าสุด: {{ lastKey }}</p>
    
    <!-- Input/Change Events -->
    <input
      type="text"
      (input)="onInput($event)"
      placeholder="Input event"
    />
    <input
      type="checkbox"
      (change)="onCheckboxChange($event)"
    />
    <select (change)="onSelectChange($event)">
      <option value="">เลือก...</option>
      <option value="1">ตัวเลือก 1</option>
      <option value="2">ตัวเลือก 2</option>
    </select>
    
    <!-- Focus Events -->
    <input
      (focus)="onFocus()"
      (blur)="onBlur()"
      placeholder="Focus/Blur events"
    />
    
    <!-- Form Submit -->
    <form (submit)="onSubmit($event)">
      <input type="text" name="name" required />
      <button type="submit">Submit</button>
    </form>
    
    <!-- Stop Propagation -->
    <div (click)="onOuterClick()">
      Outer
      <button (click)="onInnerClick(); $event.stopPropagation()">Inner</button>
    </div>
    
    <!-- Prevent Default -->
    <a href="https://google.com" (click)="$event.preventDefault(); onLinkClick()">
      Prevented link
    </a>
  `
})
export class EventBindingComponent {
  count = 0;
  isHovered = false;
  mouseX = 0;
  mouseY = 0;
  lastKey = '';
  
  onClick(): void {
    console.log('Button clicked!');
    this.count++;
  }
  
  onClickWithEvent(event: MouseEvent): void {
    console.log('Click at:', event.clientX, event.clientY);
  }
  
  onMouseEnter(): void {
    this.isHovered = true;
  }
  
  onMouseLeave(): void {
    this.isHovered = false;
  }
  
  onMouseMove(event: MouseEvent): void {
    this.mouseX = event.clientX;
    this.mouseY = event.clientY;
  }
  
  onKeyUp(event: KeyboardEvent): void {
    this.lastKey = event.key;
  }
  
  onKeyDown(event: KeyboardEvent): void {
    if (event.ctrlKey && event.key === 's') {
      event.preventDefault();
      this.save();
    }
  }
  
  onEnterPress(): void {
    console.log('Enter pressed!');
  }
  
  onEscapePress(): void {
    console.log('Escape pressed!');
  }
  
  onInput(event: Event): void {
    const value = (event.target as HTMLInputElement).value;
    console.log('Input value:', value);
  }
  
  onCheckboxChange(event: Event): void {
    const checked = (event.target as HTMLInputElement).checked;
    console.log('Checked:', checked);
  }
  
  onSelectChange(event: Event): void {
    const value = (event.target as HTMLSelectElement).value;
    console.log('Selected:', value);
  }
  
  onFocus(): void {
    console.log('Input focused');
  }
  
  onBlur(): void {
    console.log('Input blurred');
  }
  
  onSubmit(event: SubmitEvent): void {
    event.preventDefault();
    console.log('Form submitted');
  }
  
  onOuterClick(): void {
    console.log('Outer clicked');
  }
  
  onInnerClick(): void {
    console.log('Inner clicked');
  }
  
  onLinkClick(): void {
    console.log('Link clicked (prevented)');
  }
  
  save(): void {
    console.log('Saved!');
  }
}
```

---

## 8. Two-Way Binding

**Two-Way Binding** ใช้ `[(ngModel)]` — ข้อมูลไหลสองทาง (Component ↔ Template)

```typescript
import { Component } from '@angular/core';
import { CommonModule } from '@angular/common';
import { FormsModule } from '@angular/forms';  // ต้อง import FormsModule

@Component({
  selector: 'app-two-way-binding',
  standalone: true,
  imports: [CommonModule, FormsModule],
  template: `
    <h2>Two-Way Binding Demo</h2>
    
    <!-- ngModel — ต้อง import FormsModule -->
    <input [(ngModel)]="username" placeholder="Username" />
    <p>Hello, {{ username }}!</p>
    
    <!-- เทียบเท่ากับ: -->
    <input
      [value]="username"
      (input)="username = $any($event.target).value"
      placeholder="Username (manual)"
    />
    
    <!-- Checkbox -->
    <label>
      <input type="checkbox" [(ngModel)]="isAgreed" />
      ฉันยอมรับเงื่อนไข
    </label>
    <p>ยอมรับ: {{ isAgreed }}</p>
    
    <!-- Select -->
    <select [(ngModel)]="selectedCity">
      <option *ngFor="let city of cities" [value]="city.value">
        {{ city.label }}
      </option>
    </select>
    <p>เมืองที่เลือก: {{ selectedCity }}</p>
    
    <!-- Radio -->
    <div>
      <label *ngFor="let gender of genders">
        <input type="radio" [(ngModel)]="selectedGender" [value]="gender.value" />
        {{ gender.label }}
      </label>
    </div>
    <p>เพศ: {{ selectedGender }}</p>
    
    <!-- Textarea -->
    <textarea [(ngModel)]="message" rows="4" cols="40"></textarea>
    <p>ข้อความ ({{ message.length }} ตัวอักษร):</p>
    <pre>{{ message }}</pre>
    
    <!-- Range -->
    <input type="range" [(ngModel)]="volume" min="0" max="100" />
    <p>ระดับเสียง: {{ volume }}%</p>
    
    <!-- Number -->
    <input type="number" [(ngModel)]="quantity" min="1" max="99" />
    <p>จำนวน: {{ quantity }}</p>
    
    <!-- สรุปข้อมูล -->
    <div class="summary">
      <h3>สรุปข้อมูล:</h3>
      <pre>{{ getSummary() }}</pre>
    </div>
  `
})
export class TwoWayBindingComponent {
  username = '';
  isAgreed = false;
  selectedCity = 'bangkok';
  selectedGender = 'male';
  message = '';
  volume = 50;
  quantity = 1;
  
  cities = [
    { value: 'bangkok', label: 'กรุงเทพฯ' },
    { value: 'chiangmai', label: 'เชียงใหม่' },
    { value: 'phuket', label: 'ภูเก็ต' },
    { value: 'pattaya', label: 'พัทยา' }
  ];
  
  genders = [
    { value: 'male', label: 'ชาย' },
    { value: 'female', label: 'หญิง' },
    { value: 'other', label: 'อื่นๆ' }
  ];
  
  getSummary(): string {
    return JSON.stringify({
      username: this.username,
      isAgreed: this.isAgreed,
      city: this.selectedCity,
      gender: this.selectedGender,
      volume: this.volume,
      quantity: this.quantity
    }, null, 2);
  }
}
```

---

## 9. Template Reference Variables

**Template Reference Variables** ใช้ `#name` เพื่อ reference element หรือ directive

```typescript
@Component({
  selector: 'app-template-ref',
  standalone: true,
  imports: [CommonModule, FormsModule],
  template: `
    <!-- Reference to DOM element -->
    <input #nameInput type="text" placeholder="ชื่อของคุณ" />
    <button (click)="greet(nameInput.value)">สวัสดี</button>
    <button (click)="nameInput.focus()">Focus</button>
    <button (click)="nameInput.select()">Select All</button>
    
    <!-- Reference to Component/Directive -->
    <input
      #myModel="ngModel"
      [(ngModel)]="email"
      type="email"
      required
      email
    />
    <div *ngIf="myModel.invalid && myModel.touched" class="error">
      กรุณาใส่ email ที่ถูกต้อง
    </div>
    
    <!-- Reference ใน Event Handler -->
    <div class="input-group">
      <input #searchInput type="text" placeholder="ค้นหา..." />
      <button (click)="search(searchInput.value)">ค้นหา</button>
      <button (click)="searchInput.value = ''; onClear()">ล้าง</button>
    </div>
    
    <!-- ใช้ใน ngIf -->
    <div *ngIf="showContent; else noContent">
      <p>มีเนื้อหา</p>
    </div>
    <ng-template #noContent>
      <p>ไม่มีเนื้อหา</p>
    </ng-template>
    
    <!-- Reference to ng-template -->
    <ng-container *ngTemplateOutlet="loadingTemplate"></ng-container>
    <ng-template #loadingTemplate>
      <div class="spinner">Loading...</div>
    </ng-template>
    
    <!-- ผลลัพธ์ -->
    <p *ngIf="greeting">{{ greeting }}</p>
    <p *ngIf="searchTerm">ค้นหา: {{ searchTerm }}</p>
  `
})
export class TemplateRefComponent {
  email = '';
  greeting = '';
  searchTerm = '';
  showContent = true;
  
  greet(name: string): void {
    this.greeting = name ? `สวัสดี, ${name}!` : 'กรุณาใส่ชื่อ';
  }
  
  search(term: string): void {
    this.searchTerm = term;
    console.log('Searching for:', term);
  }
  
  onClear(): void {
    this.searchTerm = '';
  }
}
```

---

## 10. Workshop: Product Card Component

มาสร้าง Product Card Component ที่สมบูรณ์!

```bash
ng g c features/products/product-card --standalone --skip-tests
```

**`product-card.component.ts`:**
```typescript
import { Component, Input, Output, EventEmitter, signal, computed } from '@angular/core';
import { CommonModule } from '@angular/common';

export interface Product {
  id: number;
  name: string;
  description: string;
  price: number;
  discountPrice?: number;
  imageUrl: string;
  category: string;
  rating: number;
  reviewCount: number;
  stock: number;
  tags: string[];
  isNew?: boolean;
  isFeatured?: boolean;
}

@Component({
  selector: 'app-product-card',
  standalone: true,
  imports: [CommonModule],
  templateUrl: './product-card.component.html',
  styleUrl: './product-card.component.scss'
})
export class ProductCardComponent {
  @Input({ required: true }) product!: Product;
  @Input() isInWishlist = false;
  
  @Output() addToCart = new EventEmitter<Product>();
  @Output() toggleWishlist = new EventEmitter<Product>();
  @Output() viewDetail = new EventEmitter<Product>();
  
  quantity = signal(1);
  imageError = signal(false);
  
  get discountPercent(): number | null {
    if (!this.product.discountPrice) return null;
    return Math.round((1 - this.product.discountPrice / this.product.price) * 100);
  }
  
  get isOutOfStock(): boolean {
    return this.product.stock === 0;
  }
  
  get isLowStock(): boolean {
    return this.product.stock > 0 && this.product.stock <= 5;
  }
  
  get stars(): number[] {
    return Array.from({ length: 5 }, (_, i) => i + 1);
  }
  
  onAddToCart(): void {
    if (this.isOutOfStock) return;
    this.addToCart.emit(this.product);
  }
  
  onToggleWishlist(): void {
    this.toggleWishlist.emit(this.product);
  }
  
  onViewDetail(): void {
    this.viewDetail.emit(this.product);
  }
  
  onImageError(): void {
    this.imageError.set(true);
  }
  
  increaseQty(): void {
    if (this.quantity() < this.product.stock) {
      this.quantity.update(q => q + 1);
    }
  }
  
  decreaseQty(): void {
    if (this.quantity() > 1) {
      this.quantity.update(q => q - 1);
    }
  }
}
```

**`product-card.component.html`:**
```html
<article class="product-card" [class.out-of-stock]="isOutOfStock">
  
  <!-- Image Container -->
  <div class="image-container" (click)="onViewDetail()">
    <img
      *ngIf="!imageError()"
      [src]="product.imageUrl"
      [alt]="product.name"
      (error)="onImageError()"
      class="product-image"
      loading="lazy"
    />
    
    <!-- Fallback Image -->
    <div *ngIf="imageError()" class="image-placeholder">
      <span>🖼️</span>
      <p>ไม่มีรูปภาพ</p>
    </div>
    
    <!-- Badges -->
    <div class="badges">
      <span *ngIf="product.isNew" class="badge badge-new">ใหม่</span>
      <span *ngIf="product.isFeatured" class="badge badge-featured">แนะนำ</span>
      <span *ngIf="discountPercent" class="badge badge-discount">
        -{{ discountPercent }}%
      </span>
    </div>
    
    <!-- Wishlist Button -->
    <button
      class="wishlist-btn"
      [class.active]="isInWishlist"
      (click)="onToggleWishlist(); $event.stopPropagation()"
      [attr.aria-label]="isInWishlist ? 'ลบออกจาก wishlist' : 'เพิ่มใน wishlist'"
    >
      {{ isInWishlist ? '❤️' : '🤍' }}
    </button>
  </div>
  
  <!-- Card Body -->
  <div class="card-body">
    
    <!-- Category -->
    <span class="category">{{ product.category }}</span>
    
    <!-- Name -->
    <h3 class="product-name" (click)="onViewDetail()">{{ product.name }}</h3>
    
    <!-- Description -->
    <p class="description">{{ product.description }}</p>
    
    <!-- Rating -->
    <div class="rating">
      <span class="stars">
        <span
          *ngFor="let star of stars"
          class="star"
          [class.filled]="star <= product.rating"
          [class.half]="star - 0.5 === product.rating"
        >★</span>
      </span>
      <span class="rating-value">{{ product.rating }}</span>
      <span class="review-count">({{ product.reviewCount }} รีวิว)</span>
    </div>
    
    <!-- Price -->
    <div class="price-container">
      <ng-container *ngIf="product.discountPrice; else regularPrice">
        <span class="price-original">฿{{ product.price | number:'1.0-0' }}</span>
        <span class="price-discount">฿{{ product.discountPrice | number:'1.0-0' }}</span>
      </ng-container>
      <ng-template #regularPrice>
        <span class="price-regular">฿{{ product.price | number:'1.0-0' }}</span>
      </ng-template>
    </div>
    
    <!-- Stock Status -->
    <div class="stock-status">
      <span *ngIf="isOutOfStock" class="out-of-stock">หมดสต็อก</span>
      <span *ngIf="isLowStock && !isOutOfStock" class="low-stock">
        เหลือเพียง {{ product.stock }} ชิ้น!
      </span>
      <span *ngIf="!isOutOfStock && !isLowStock" class="in-stock">
        มีสินค้า
      </span>
    </div>
    
    <!-- Tags -->
    <div class="tags" *ngIf="product.tags.length > 0">
      <span *ngFor="let tag of product.tags" class="tag">#{{ tag }}</span>
    </div>
    
    <!-- Quantity & Add to Cart -->
    <div class="actions" *ngIf="!isOutOfStock">
      <div class="quantity-selector">
        <button (click)="decreaseQty()" [disabled]="quantity() <= 1">-</button>
        <span>{{ quantity() }}</span>
        <button (click)="increaseQty()" [disabled]="quantity() >= product.stock">+</button>
      </div>
      
      <button class="btn-add-cart" (click)="onAddToCart()">
        🛒 เพิ่มลงตะกร้า
      </button>
    </div>
    
    <button *ngIf="isOutOfStock" class="btn-notify">
      🔔 แจ้งเตือนเมื่อมีสินค้า
    </button>
    
  </div>
</article>
```

**`product-card.component.scss`:**
```scss
.product-card {
  background: white;
  border-radius: 12px;
  overflow: hidden;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  transition: transform 0.3s ease, box-shadow 0.3s ease;
  display: flex;
  flex-direction: column;
  
  &:hover {
    transform: translateY(-4px);
    box-shadow: 0 8px 24px rgba(0, 0, 0, 0.15);
  }
  
  &.out-of-stock {
    opacity: 0.7;
  }
}

// Image Container
.image-container {
  position: relative;
  padding-top: 66%;  // 3:2 aspect ratio
  overflow: hidden;
  cursor: pointer;
  background: #f5f5f5;
  
  .product-image {
    position: absolute;
    inset: 0;
    width: 100%;
    height: 100%;
    object-fit: cover;
    transition: transform 0.3s ease;
  }
  
  &:hover .product-image {
    transform: scale(1.05);
  }
  
  .image-placeholder {
    position: absolute;
    inset: 0;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    color: #999;
    
    span { font-size: 3rem; }
    p { margin-top: 0.5rem; font-size: 0.875rem; }
  }
}

// Badges
.badges {
  position: absolute;
  top: 12px;
  left: 12px;
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.badge {
  padding: 3px 8px;
  border-radius: 4px;
  font-size: 0.7rem;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.5px;
  
  &-new { background: #4caf50; color: white; }
  &-featured { background: #ff9800; color: white; }
  &-discount { background: #f44336; color: white; }
}

// Wishlist Button
.wishlist-btn {
  position: absolute;
  top: 12px;
  right: 12px;
  background: white;
  border: none;
  border-radius: 50%;
  width: 36px;
  height: 36px;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  font-size: 1rem;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.15);
  transition: transform 0.2s ease;
  
  &:hover { transform: scale(1.15); }
}

// Card Body
.card-body {
  padding: 1rem;
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.category {
  font-size: 0.75rem;
  color: #1976d2;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

.product-name {
  font-size: 1rem;
  font-weight: 600;
  color: #212121;
  line-height: 1.4;
  cursor: pointer;
  
  &:hover { color: #1976d2; }
}

.description {
  font-size: 0.875rem;
  color: #757575;
  line-height: 1.5;
  display: -webkit-box;
  -webkit-line-clamp: 2;
  -webkit-box-orient: vertical;
  overflow: hidden;
}

// Rating
.rating {
  display: flex;
  align-items: center;
  gap: 0.25rem;
  
  .star {
    color: #e0e0e0;
    font-size: 1rem;
    
    &.filled { color: #ffc107; }
    &.half { color: #ffc107; }
  }
  
  .rating-value {
    font-weight: 600;
    font-size: 0.875rem;
    color: #212121;
  }
  
  .review-count {
    font-size: 0.75rem;
    color: #9e9e9e;
  }
}

// Price
.price-container {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  flex-wrap: wrap;
  
  .price-regular {
    font-size: 1.25rem;
    font-weight: 700;
    color: #212121;
  }
  
  .price-original {
    font-size: 0.875rem;
    text-decoration: line-through;
    color: #9e9e9e;
  }
  
  .price-discount {
    font-size: 1.25rem;
    font-weight: 700;
    color: #f44336;
  }
}

// Stock Status
.stock-status {
  font-size: 0.8rem;
  font-weight: 500;
  
  .out-of-stock { color: #f44336; }
  .low-stock { color: #ff9800; }
  .in-stock { color: #4caf50; }
}

// Tags
.tags {
  display: flex;
  flex-wrap: wrap;
  gap: 4px;
  
  .tag {
    padding: 2px 8px;
    background: #f5f5f5;
    border-radius: 12px;
    font-size: 0.75rem;
    color: #616161;
  }
}

// Actions
.actions {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  margin-top: auto;
  padding-top: 0.5rem;
}

.quantity-selector {
  display: flex;
  align-items: center;
  border: 1px solid #e0e0e0;
  border-radius: 6px;
  overflow: hidden;
  
  button {
    width: 32px;
    height: 32px;
    border: none;
    background: #f5f5f5;
    cursor: pointer;
    font-size: 1rem;
    transition: background 0.2s;
    
    &:hover:not(:disabled) { background: #e0e0e0; }
    &:disabled { opacity: 0.4; cursor: not-allowed; }
  }
  
  span {
    width: 36px;
    text-align: center;
    font-size: 0.875rem;
    font-weight: 600;
  }
}

.btn-add-cart {
  flex: 1;
  padding: 0.5rem 1rem;
  background: #1976d2;
  color: white;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  font-size: 0.875rem;
  font-weight: 600;
  transition: background 0.2s;
  white-space: nowrap;
  
  &:hover { background: #1565c0; }
  &:active { background: #0d47a1; }
}

.btn-notify {
  width: 100%;
  margin-top: auto;
  padding: 0.5rem;
  background: transparent;
  border: 2px solid #9e9e9e;
  border-radius: 6px;
  cursor: pointer;
  font-size: 0.875rem;
  color: #616161;
  transition: all 0.2s;
  
  &:hover {
    border-color: #1976d2;
    color: #1976d2;
  }
}
```

### สร้าง Product List เพื่อทดสอบ

```typescript
// app.component.ts
import { Component } from '@angular/core';
import { CommonModule } from '@angular/common';
import { ProductCardComponent, Product } from './features/products/product-card/product-card.component';

@Component({
  selector: 'app-root',
  standalone: true,
  imports: [CommonModule, ProductCardComponent],
  template: `
    <div class="page">
      <h1>🛍️ ร้านค้าออนไลน์</h1>
      
      <!-- Cart Summary -->
      <div class="cart-summary" *ngIf="cartItems.length > 0">
        🛒 ตะกร้า: {{ cartItems.length }} รายการ
        | ยอดรวม: ฿{{ cartTotal | number:'1.0-0' }}
      </div>
      
      <!-- Product Grid -->
      <div class="products-grid">
        <app-product-card
          *ngFor="let product of products"
          [product]="product"
          [isInWishlist]="wishlist.has(product.id)"
          (addToCart)="onAddToCart($event)"
          (toggleWishlist)="onToggleWishlist($event)"
          (viewDetail)="onViewDetail($event)"
        />
      </div>
    </div>
  `,
  styles: [`
    .page {
      max-width: 1200px;
      margin: 0 auto;
      padding: 2rem;
    }
    
    h1 {
      font-size: 2rem;
      margin-bottom: 1.5rem;
      color: #212121;
    }
    
    .cart-summary {
      background: #e3f2fd;
      padding: 0.75rem 1rem;
      border-radius: 8px;
      margin-bottom: 1.5rem;
      font-weight: 500;
    }
    
    .products-grid {
      display: grid;
      grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
      gap: 1.5rem;
    }
  `]
})
export class AppComponent {
  products: Product[] = [
    {
      id: 1,
      name: 'iPhone 15 Pro Max',
      description: 'สมาร์ทโฟนรุ่นใหม่ล่าสุด ชิป A17 Pro กล้อง 48MP',
      price: 49900,
      discountPrice: 44900,
      imageUrl: 'https://via.placeholder.com/400x266/1976d2/white?text=iPhone+15',
      category: 'สมาร์ทโฟน',
      rating: 4.8,
      reviewCount: 2341,
      stock: 15,
      tags: ['apple', 'smartphone', 'ios'],
      isNew: true,
      isFeatured: true
    },
    {
      id: 2,
      name: 'Samsung Galaxy S24 Ultra',
      description: 'Android flagship พร้อม S Pen ในตัว',
      price: 45900,
      imageUrl: 'https://via.placeholder.com/400x266/424242/white?text=Galaxy+S24',
      category: 'สมาร์ทโฟน',
      rating: 4.7,
      reviewCount: 1876,
      stock: 8,
      tags: ['samsung', 'android', 'spen'],
      isFeatured: true
    },
    {
      id: 3,
      name: 'MacBook Air M3',
      description: 'แล็ปท็อปที่บางเบา ชิป M3 แบตเตอรี่อยู่ได้ 18 ชั่วโมง',
      price: 42900,
      discountPrice: 39900,
      imageUrl: 'https://via.placeholder.com/400x266/607d8b/white?text=MacBook+Air',
      category: 'แล็ปท็อป',
      rating: 4.9,
      reviewCount: 987,
      stock: 0,  // out of stock
      tags: ['apple', 'laptop', 'macos'],
      isNew: true
    },
    {
      id: 4,
      name: 'Sony WH-1000XM5',
      description: 'หูฟัง Noise Cancelling ระดับโลก',
      price: 13900,
      discountPrice: 11900,
      imageUrl: 'https://via.placeholder.com/400x266/9c27b0/white?text=Sony+WH5',
      category: 'หูฟัง',
      rating: 4.6,
      reviewCount: 3210,
      stock: 3,  // low stock
      tags: ['sony', 'headphones', 'anc'],
    }
  ];
  
  cartItems: { product: Product; qty: number }[] = [];
  wishlist = new Set<number>();
  
  get cartTotal(): number {
    return this.cartItems.reduce((total, item) => {
      const price = item.product.discountPrice ?? item.product.price;
      return total + price * item.qty;
    }, 0);
  }
  
  onAddToCart(product: Product): void {
    const existing = this.cartItems.find(i => i.product.id === product.id);
    if (existing) {
      existing.qty++;
    } else {
      this.cartItems.push({ product, qty: 1 });
    }
    console.log(`เพิ่ม ${product.name} ลงตะกร้า`);
  }
  
  onToggleWishlist(product: Product): void {
    if (this.wishlist.has(product.id)) {
      this.wishlist.delete(product.id);
    } else {
      this.wishlist.add(product.id);
    }
  }
  
  onViewDetail(product: Product): void {
    console.log('ดูรายละเอียด:', product.name);
  }
}
```

---

## สรุปบทที่ 3

| หัวข้อ | สิ่งที่ได้เรียนรู้ |
|--------|------------------|
| Component | building block ของ Angular |
| Anatomy | Template, Class, Styles |
| CLI | ng g c, options |
| Metadata | selector, imports, template |
| Interpolation | {{ expression }} |
| Property Binding | [property]="value" |
| Event Binding | (event)="handler()" |
| Two-Way Binding | [(ngModel)]="value" |
| Template Ref | #varName |
| Workshop | Product Card Component |

---

## แบบฝึกหัด

1. **ง่าย**: สร้าง `UserAvatarComponent` รับ `name` และ `imageUrl` แสดงรูปหรือ initials ถ้าไม่มีรูป
2. **ปานกลาย**: สร้าง `CounterComponent` รับ `min`, `max`, `step` เป็น Input, emit `changed` event
3. **ท้าทาย**: สร้าง `StarRatingComponent` ที่ click เพื่อ rate ได้ ส่ง rating กลับ parent

---

## บทถัดไป

[Part 04 — Templates และ Data Binding →](part-04-templates-databinding.md)
