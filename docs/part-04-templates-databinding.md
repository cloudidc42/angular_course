# Part 04 — Templates และ Data Binding

## สารบัญ

1. [Template Syntax ทั้งหมด](#template-syntax)
2. [Interpolation ขั้นสูง](#interpolation)
3. [Property Binding](#property-binding)
4. [Attribute Binding](#attribute-binding)
5. [Class Binding](#class-binding)
6. [Style Binding](#style-binding)
7. [Event Binding ขั้นสูง](#event-binding)
8. [Two-Way Binding](#two-way-binding)
9. [ngModel ใน Reactive Forms](#ngmodel-reactive-forms)
10. [Template Reference Variables ขั้นสูง](#template-reference-variables)
11. [ng-template](#ng-template)
12. [ng-container](#ng-container)
13. [ng-content และ Content Projection](#ng-content)
14. [Workshop: Dashboard Layout Component](#workshop)

---

## 1. Template Syntax ทั้งหมด {#template-syntax}

Angular Template คือ HTML ที่ถูกเสริมด้วย syntax พิเศษของ Angular ซึ่งช่วยให้เราสามารถแสดงข้อมูล, ตอบสนองต่อ event และ control การแสดงผลได้อย่างมีประสิทธิภาพ

### ประเภทหลักของ Template Syntax

| Syntax | รูปแบบ | ความหมาย |
|--------|--------|-----------|
| Interpolation | `{{ expression }}` | แสดงค่าของ expression |
| Property Binding | `[property]="expression"` | ผูก property กับ expression |
| Attribute Binding | `[attr.name]="expression"` | ผูก attribute กับ expression |
| Class Binding | `[class.name]="boolean"` | เพิ่ม/ลบ CSS class |
| Style Binding | `[style.prop]="value"` | กำหนด inline style |
| Event Binding | `(event)="handler()"` | ฟังก์ event จาก DOM |
| Two-Way Binding | `[(ngModel)]="property"` | binding แบบ 2 ทิศทาง |
| Template Variable | `#refName` | อ้างอิงไปยัง element หรือ component |
| Structural Directive | `*ngIf`, `*ngFor` | เปลี่ยนโครงสร้าง DOM |

### ตัวอย่างโครงสร้างพื้นฐาน

```typescript
// app.component.ts
import { Component } from '@angular/core';

@Component({
  selector: 'app-root',
  template: `
    <div class="container">
      <h1>{{ title }}</h1>
      <p [class.highlight]="isActive">{{ message }}</p>
      <button (click)="toggleActive()">Toggle</button>
    </div>
  `,
  styles: [`
    .highlight { color: blue; font-weight: bold; }
  `]
})
export class AppComponent {
  title = 'Angular Course';
  message = 'Hello, Angular!';
  isActive = false;

  toggleActive(): void {
    this.isActive = !this.isActive;
  }
}
```

---

## 2. Interpolation ขั้นสูง {#interpolation}

Interpolation ใช้ `{{ }}` เพื่อแสดงค่าใน template โดยสามารถใส่ expression ที่ซับซ้อนได้

### การใช้งานพื้นฐาน

```typescript
// interpolation-demo.component.ts
import { Component } from '@angular/core';

@Component({
  selector: 'app-interpolation-demo',
  template: `
    <div>
      <!-- แสดงค่าตัวแปรธรรมดา -->
      <p>ชื่อ: {{ firstName }}</p>
      <p>นามสกุล: {{ lastName }}</p>
      
      <!-- การต่อ string -->
      <p>ชื่อเต็ม: {{ firstName + ' ' + lastName }}</p>
      
      <!-- การเรียก method -->
      <p>ชื่อตัวพิมพ์ใหญ่: {{ getFullName().toUpperCase() }}</p>
      
      <!-- การคำนวณ -->
      <p>ผลรวม: {{ price * quantity }}</p>
      <p>ราคารวม (พร้อม VAT): {{ calculateTotal() | number:'1.2-2' }}</p>
      
      <!-- Ternary operator -->
      <p>สถานะ: {{ isActive ? 'ใช้งานอยู่' : 'ไม่ได้ใช้งาน' }}</p>
      
      <!-- Optional chaining (Angular 14+) -->
      <p>เมือง: {{ user?.address?.city ?? 'ไม่ระบุ' }}</p>
      
      <!-- การใช้กับ object -->
      <p>อีเมล: {{ user.email }}</p>
    </div>
  `
})
export class InterpolationDemoComponent {
  firstName = 'สมชาย';
  lastName = 'ใจดี';
  price = 100;
  quantity = 3;
  isActive = true;
  
  user = {
    email: 'somchai@example.com',
    address: {
      city: 'กรุงเทพฯ'
    }
  };

  getFullName(): string {
    return `${this.firstName} ${this.lastName}`;
  }

  calculateTotal(): number {
    return this.price * this.quantity * 1.07; // รวม VAT 7%
  }
}
```

### สิ่งที่ห้ามใช้ใน Interpolation

```html
<!-- ห้ามใช้: assignment operator -->
{{ x = 10 }}

<!-- ห้ามใช้: new keyword -->
{{ new Date() }}

<!-- ห้ามใช้: chaining expressions ด้วย ; -->
{{ a = 1; b = 2 }}

<!-- ใช้ได้: การเรียก method ปกติ -->
{{ formatDate(date) }}
{{ items.length > 0 ? 'มีสินค้า' : 'ไม่มีสินค้า' }}
```

### Interpolation กับ HTML Encoding

```typescript
@Component({
  template: `
    <!-- Angular จะ encode HTML โดยอัตโนมัติเพื่อป้องกัน XSS -->
    <p>{{ htmlContent }}</p>
    <!-- Output: &lt;strong&gt;Bold&lt;/strong&gt; (ไม่ render เป็น HTML) -->
    
    <!-- ถ้าต้องการ render HTML ต้องใช้ [innerHTML] binding แทน -->
    <p [innerHTML]="trustedHtmlContent"></p>
  `
})
export class HtmlDemoComponent {
  htmlContent = '<strong>Bold Text</strong>';
  trustedHtmlContent = '<em>Italic Text</em>';
}
```

---

## 3. Property Binding {#property-binding}

Property Binding ใช้ `[property]="expression"` เพื่อผูกค่าของ JavaScript expression เข้ากับ property ของ DOM element หรือ component

### การใช้งานพื้นฐาน

```typescript
// property-binding.component.ts
import { Component } from '@angular/core';

@Component({
  selector: 'app-property-binding',
  template: `
    <div>
      <!-- ผูก src ของรูปภาพ -->
      <img [src]="imageUrl" [alt]="imageAlt" [width]="imageWidth">
      
      <!-- ผูก disabled state ของ button -->
      <button [disabled]="isLoading">
        {{ isLoading ? 'กำลังโหลด...' : 'คลิกที่นี่' }}
      </button>
      
      <!-- ผูก href ของ link -->
      <a [href]="websiteUrl" target="_blank">เว็บไซต์</a>
      
      <!-- ผูก value ของ input -->
      <input [value]="currentValue" type="text">
      
      <!-- ผูก checked ของ checkbox -->
      <input type="checkbox" [checked]="isChecked">
      
      <!-- ผูก title attribute (tooltip) -->
      <span [title]="tooltipText">Hover me</span>
      
      <!-- ผูกหลาย property พร้อมกัน -->
      <video
        [src]="videoUrl"
        [width]="videoWidth"
        [height]="videoHeight"
        [autoplay]="shouldAutoplay"
        [muted]="isMuted"
        controls>
      </video>
    </div>
  `
})
export class PropertyBindingComponent {
  imageUrl = 'https://example.com/image.jpg';
  imageAlt = 'ภาพตัวอย่าง';
  imageWidth = 300;
  isLoading = false;
  websiteUrl = 'https://angular.io';
  currentValue = 'ค่าเริ่มต้น';
  isChecked = true;
  tooltipText = 'นี่คือ tooltip';
  videoUrl = 'https://example.com/video.mp4';
  videoWidth = 640;
  videoHeight = 360;
  shouldAutoplay = false;
  isMuted = true;
}
```

### Property Binding กับ Component Inputs

```typescript
// parent.component.ts
import { Component } from '@angular/core';

interface Product {
  id: number;
  name: string;
  price: number;
  inStock: boolean;
}

@Component({
  selector: 'app-parent',
  template: `
    <div>
      <h2>รายการสินค้า</h2>
      
      <!-- ส่ง data ไปยัง child component ผ่าน property binding -->
      <app-product-card
        *ngFor="let product of products"
        [product]="product"
        [showPrice]="showPrices"
        [currency]="'THB'"
        [discountPercent]="10">
      </app-product-card>
      
      <button (click)="togglePrices()">
        {{ showPrices ? 'ซ่อนราคา' : 'แสดงราคา' }}
      </button>
    </div>
  `
})
export class ParentComponent {
  showPrices = true;
  
  products: Product[] = [
    { id: 1, name: 'iPhone 15', price: 35000, inStock: true },
    { id: 2, name: 'Samsung Galaxy S24', price: 28000, inStock: false },
    { id: 3, name: 'Google Pixel 8', price: 22000, inStock: true }
  ];

  togglePrices(): void {
    this.showPrices = !this.showPrices;
  }
}

// product-card.component.ts
import { Component, Input } from '@angular/core';

@Component({
  selector: 'app-product-card',
  template: `
    <div class="card" [class.out-of-stock]="!product.inStock">
      <h3>{{ product.name }}</h3>
      <p *ngIf="showPrice">
        ราคา: {{ product.price * (1 - discountPercent/100) | number:'1.0-0' }} {{ currency }}
        <span class="discount">(ลด {{ discountPercent }}%)</span>
      </p>
      <p [class.available]="product.inStock">
        {{ product.inStock ? 'มีสินค้า' : 'สินค้าหมด' }}
      </p>
    </div>
  `
})
export class ProductCardComponent {
  @Input() product!: Product;
  @Input() showPrice = true;
  @Input() currency = 'THB';
  @Input() discountPercent = 0;
}
```

---

## 4. Attribute Binding {#attribute-binding}

ใช้สำหรับ HTML attributes ที่ไม่มี DOM property counterpart โดยตรง (เช่น `aria-*`, `colspan`, `rowspan`)

```typescript
// attribute-binding.component.ts
import { Component } from '@angular/core';

@Component({
  selector: 'app-attribute-binding',
  template: `
    <div>
      <!-- Aria attributes สำหรับ accessibility -->
      <button
        [attr.aria-label]="buttonLabel"
        [attr.aria-expanded]="isExpanded"
        [attr.aria-controls]="'panel-' + panelId"
        (click)="togglePanel()">
        เมนู
      </button>
      
      <div [attr.id]="'panel-' + panelId" *ngIf="isExpanded">
        เนื้อหาของ panel
      </div>
      
      <!-- Table colspan/rowspan -->
      <table>
        <tr>
          <td [attr.colspan]="colSpan">รวมคอลัมน์</td>
        </tr>
        <tr>
          <td [attr.rowspan]="rowSpan">รวมแถว</td>
          <td>เซลล์ปกติ</td>
        </tr>
      </table>
      
      <!-- Data attributes -->
      <div
        [attr.data-user-id]="userId"
        [attr.data-role]="userRole">
        ข้อมูลผู้ใช้
      </div>
      
      <!-- SVG attributes -->
      <svg width="100" height="100">
        <circle
          cx="50" cy="50"
          [attr.r]="circleRadius"
          [attr.fill]="circleColor"
          [attr.stroke]="strokeColor"
          [attr.stroke-width]="strokeWidth">
        </circle>
      </svg>
    </div>
  `
})
export class AttributeBindingComponent {
  buttonLabel = 'เปิด/ปิดเมนู';
  isExpanded = false;
  panelId = 1;
  colSpan = 3;
  rowSpan = 2;
  userId = 12345;
  userRole = 'admin';
  circleRadius = 40;
  circleColor = '#4CAF50';
  strokeColor = '#333';
  strokeWidth = 2;

  togglePanel(): void {
    this.isExpanded = !this.isExpanded;
  }
}
```

---

## 5. Class Binding {#class-binding}

Class Binding ช่วยให้เราสามารถเพิ่ม/ลบ CSS class ตามเงื่อนไขได้อย่างสะดวก

```typescript
// class-binding.component.ts
import { Component } from '@angular/core';

@Component({
  selector: 'app-class-binding',
  template: `
    <div>
      <!-- Single class binding -->
      <p [class.active]="isActive">ข้อความที่ active</p>
      <p [class.error]="hasError">ข้อความ error</p>
      
      <!-- Multiple class binding ด้วย object -->
      <div [ngClass]="buttonClasses">
        Button ที่มีหลาย class
      </div>
      
      <!-- ngClass กับ method -->
      <div [ngClass]="getStatusClasses()">
        สถานะปัจจุบัน
      </div>
      
      <!-- ngClass กับ array -->
      <div [ngClass]="['base-class', 'rounded', isHighlighted ? 'highlighted' : '']">
        Array-based classes
      </div>
      
      <!-- ผสมกัน: static class + dynamic binding -->
      <button
        class="btn"
        [class.btn-primary]="isPrimary"
        [class.btn-secondary]="!isPrimary"
        [class.btn-lg]="isLarge"
        [class.disabled]="isDisabled">
        Mixed Binding Button
      </button>
      
      <!-- Class binding กับ string -->
      <div [class]="activeClasses">Dynamic Class String</div>
    </div>
  `,
  styles: [`
    .active { color: green; font-weight: bold; }
    .error { color: red; }
    .loading { opacity: 0.5; cursor: wait; }
    .success { background-color: #e8f5e9; }
    .warning { background-color: #fff3e0; }
    .danger { background-color: #ffebee; }
    .highlighted { box-shadow: 0 0 10px rgba(0,0,0,0.3); }
    .btn { padding: 8px 16px; border: none; cursor: pointer; }
    .btn-primary { background-color: #007bff; color: white; }
    .btn-secondary { background-color: #6c757d; color: white; }
    .btn-lg { padding: 12px 24px; font-size: 1.2em; }
    .disabled { opacity: 0.6; cursor: not-allowed; }
  `]
})
export class ClassBindingComponent {
  isActive = true;
  hasError = false;
  isHighlighted = true;
  isPrimary = true;
  isLarge = false;
  isDisabled = false;
  
  status: 'loading' | 'success' | 'warning' | 'danger' = 'success';

  get buttonClasses(): Record<string, boolean> {
    return {
      'btn': true,
      'loading': this.status === 'loading',
      'success': this.status === 'success',
      'warning': this.status === 'warning',
      'danger': this.status === 'danger'
    };
  }

  get activeClasses(): string {
    const classes: string[] = ['base'];
    if (this.isActive) classes.push('active');
    if (this.isHighlighted) classes.push('highlighted');
    return classes.join(' ');
  }

  getStatusClasses(): Record<string, boolean> {
    return {
      'status-indicator': true,
      [`status-${this.status}`]: true,
      'pulse': this.status === 'loading'
    };
  }
}
```

---

## 6. Style Binding {#style-binding}

Style Binding ช่วยกำหนด inline styles ได้อย่างยืดหยุ่น

```typescript
// style-binding.component.ts
import { Component } from '@angular/core';

@Component({
  selector: 'app-style-binding',
  template: `
    <div>
      <!-- Single style binding -->
      <p [style.color]="textColor">ข้อความสี {{ textColor }}</p>
      <p [style.font-size.px]="fontSize">ขนาด font {{ fontSize }}px</p>
      <p [style.font-size.em]="fontSizeEm">ขนาด font {{ fontSizeEm }}em</p>
      <p [style.opacity]="opacity">Opacity: {{ opacity }}</p>
      
      <!-- Style binding กับ unit -->
      <div
        [style.width.px]="boxWidth"
        [style.height.px]="boxHeight"
        [style.background-color]="backgroundColor"
        [style.border-radius.px]="borderRadius">
        กล่องตัวอย่าง
      </div>
      
      <!-- ngStyle กับ object -->
      <div [ngStyle]="cardStyles">
        Card ที่มีหลาย styles
      </div>
      
      <!-- ngStyle กับ method -->
      <div [ngStyle]="getDynamicStyles()">
        Dynamic Styles
      </div>
      
      <!-- Transition animation -->
      <div
        [style.transform]="'scale(' + scale + ')'"
        [style.transition]="'transform 0.3s ease'"
        (mouseenter)="scale = 1.1"
        (mouseleave)="scale = 1">
        Hover to scale
      </div>
      
      <!-- Progress bar -->
      <div class="progress-container">
        <div
          class="progress-bar"
          [style.width.%]="progressPercent"
          [style.background-color]="progressColor">
        </div>
        <span>{{ progressPercent }}%</span>
      </div>
    </div>
  `,
  styles: [`
    .progress-container { 
      width: 300px; 
      height: 20px; 
      background: #e0e0e0; 
      border-radius: 10px;
      overflow: hidden;
      position: relative;
    }
    .progress-bar { 
      height: 100%; 
      border-radius: 10px;
      transition: width 0.5s ease;
    }
  `]
})
export class StyleBindingComponent {
  textColor = '#e91e63';
  fontSize = 18;
  fontSizeEm = 1.2;
  opacity = 0.8;
  boxWidth = 200;
  boxHeight = 150;
  backgroundColor = '#bbdefb';
  borderRadius = 8;
  scale = 1;
  progressPercent = 65;

  get progressColor(): string {
    if (this.progressPercent < 30) return '#f44336';
    if (this.progressPercent < 70) return '#ff9800';
    return '#4caf50';
  }

  get cardStyles(): Record<string, string> {
    return {
      'padding': '16px',
      'margin': '8px',
      'background-color': '#f5f5f5',
      'border': '1px solid #ddd',
      'border-radius': '8px',
      'box-shadow': '0 2px 4px rgba(0,0,0,0.1)'
    };
  }

  getDynamicStyles(): Record<string, string> {
    const baseSize = 16;
    return {
      'font-size': `${baseSize}px`,
      'line-height': '1.5',
      'color': this.opacity > 0.5 ? '#333' : '#999',
      'text-decoration': this.opacity > 0.5 ? 'none' : 'line-through'
    };
  }
}
```

---

## 7. Event Binding ขั้นสูง {#event-binding}

Event Binding ใช้ `(eventName)="handler()"` เพื่อฟัง DOM events และตอบสนอง

### Event พื้นฐาน

```typescript
// event-binding.component.ts
import { Component } from '@angular/core';

@Component({
  selector: 'app-event-binding',
  template: `
    <div>
      <!-- Click events -->
      <button (click)="onClick()">คลิก</button>
      <button (click)="onClickWithEvent($event)">คลิกพร้อม event object</button>
      <button (click)="onClickWithParam('hello', $event)">คลิกพร้อม params</button>
      
      <!-- Mouse events -->
      <div
        (mouseenter)="onMouseEnter($event)"
        (mouseleave)="onMouseLeave($event)"
        (mousemove)="onMouseMove($event)"
        (mousedown)="onMouseDown($event)"
        (mouseup)="onMouseUp($event)"
        class="mouse-area">
        พื้นที่ mouse events
        <p>X: {{ mouseX }}, Y: {{ mouseY }}</p>
      </div>
      
      <!-- Keyboard events -->
      <input
        (keydown)="onKeyDown($event)"
        (keyup)="onKeyUp($event)"
        (keypress)="onKeyPress($event)"
        placeholder="พิมพ์ที่นี่">
      <p>Key ล่าสุด: {{ lastKey }}</p>
      
      <!-- Focus events -->
      <input
        (focus)="onFocus()"
        (blur)="onBlur()"
        [class.focused]="isFocused"
        placeholder="Click to focus">
      
      <!-- Form events -->
      <input
        type="text"
        (input)="onInput($event)"
        (change)="onChange($event)">
      <p>ค่าปัจจุบัน: {{ inputValue }}</p>
      
      <!-- Submit event -->
      <form (submit)="onSubmit($event)">
        <input type="text" name="name" [(ngModel)]="formName">
        <button type="submit">ส่ง</button>
      </form>
      
      <!-- Drag events -->
      <div
        draggable="true"
        (dragstart)="onDragStart($event)"
        (dragend)="onDragEnd($event)"
        class="draggable">
        ลากฉัน
      </div>
      
      <div
        (dragover)="onDragOver($event)"
        (drop)="onDrop($event)"
        class="drop-zone">
        วางที่นี่
      </div>
    </div>
  `
})
export class EventBindingComponent {
  mouseX = 0;
  mouseY = 0;
  lastKey = '';
  isFocused = false;
  inputValue = '';
  formName = '';

  onClick(): void {
    console.log('คลิกแล้ว!');
    alert('คลิกแล้ว!');
  }

  onClickWithEvent(event: MouseEvent): void {
    console.log('Mouse position:', event.clientX, event.clientY);
    console.log('Button clicked:', event.button);
    // 0 = left click, 1 = middle, 2 = right click
  }

  onClickWithParam(param: string, event: MouseEvent): void {
    console.log('Param:', param);
    console.log('Event:', event);
  }

  onMouseEnter(event: MouseEvent): void {
    console.log('Mouse entered');
  }

  onMouseLeave(event: MouseEvent): void {
    console.log('Mouse left');
  }

  onMouseMove(event: MouseEvent): void {
    this.mouseX = event.offsetX;
    this.mouseY = event.offsetY;
  }

  onMouseDown(event: MouseEvent): void {
    console.log('Mouse down at:', event.clientX, event.clientY);
  }

  onMouseUp(event: MouseEvent): void {
    console.log('Mouse up');
  }

  onKeyDown(event: KeyboardEvent): void {
    this.lastKey = event.key;
    if (event.key === 'Enter') {
      console.log('Enter pressed!');
    }
    if (event.ctrlKey && event.key === 's') {
      event.preventDefault(); // ป้องกัน browser save
      console.log('Ctrl+S pressed - Custom save!');
    }
  }

  onKeyUp(event: KeyboardEvent): void {
    console.log('Key up:', event.key);
  }

  onKeyPress(event: KeyboardEvent): void {
    console.log('Key press:', event.key);
  }

  onFocus(): void {
    this.isFocused = true;
    console.log('Input focused');
  }

  onBlur(): void {
    this.isFocused = false;
    console.log('Input blurred');
  }

  onInput(event: Event): void {
    const input = event.target as HTMLInputElement;
    this.inputValue = input.value;
  }

  onChange(event: Event): void {
    const input = event.target as HTMLInputElement;
    console.log('Changed to:', input.value);
  }

  onSubmit(event: Event): void {
    event.preventDefault(); // ป้องกัน page reload
    console.log('Form submitted with name:', this.formName);
  }

  onDragStart(event: DragEvent): void {
    event.dataTransfer?.setData('text/plain', 'drag data');
    console.log('Drag started');
  }

  onDragEnd(event: DragEvent): void {
    console.log('Drag ended');
  }

  onDragOver(event: DragEvent): void {
    event.preventDefault(); // จำเป็นต้องมีเพื่อให้ drop ทำงานได้
  }

  onDrop(event: DragEvent): void {
    event.preventDefault();
    const data = event.dataTransfer?.getData('text/plain');
    console.log('Dropped:', data);
  }
}
```

### Event Filtering และ Key Events

```typescript
@Component({
  selector: 'app-key-events',
  template: `
    <div>
      <!-- Angular key event filter (เฉพาะ key ที่กำหนด) -->
      <input (keyup.enter)="onEnterKey()">
      <input (keyup.escape)="onEscapeKey()">
      <input (keydown.space)="onSpaceKey($event)">
      <input (keydown.arrowup)="onArrowUp()">
      <input (keydown.arrowdown)="onArrowDown()">
      
      <!-- Modifier keys -->
      <input (keydown.ctrl.s)="onCtrlS($event)">
      <input (keydown.shift.enter)="onShiftEnter()">
      <input (keydown.alt.f4)="onAltF4($event)">
      
      <!-- Custom shortcut handler -->
      <div (keydown)="handleShortcuts($event)" tabindex="0">
        กด Ctrl+K เพื่อ search, Ctrl+N เพื่อสร้างใหม่
      </div>
    </div>
  `
})
export class KeyEventsComponent {
  onEnterKey(): void {
    console.log('Enter pressed - submit form');
  }

  onEscapeKey(): void {
    console.log('Escape pressed - close modal');
  }

  onSpaceKey(event: KeyboardEvent): void {
    event.preventDefault();
    console.log('Space pressed - toggle selection');
  }

  onArrowUp(): void {
    console.log('Arrow up - move selection up');
  }

  onArrowDown(): void {
    console.log('Arrow down - move selection down');
  }

  onCtrlS(event: KeyboardEvent): void {
    event.preventDefault();
    console.log('Ctrl+S - Save');
  }

  onShiftEnter(): void {
    console.log('Shift+Enter - new line');
  }

  onAltF4(event: KeyboardEvent): void {
    event.preventDefault();
    console.log('Alt+F4 blocked');
  }

  handleShortcuts(event: KeyboardEvent): void {
    if (event.ctrlKey) {
      switch (event.key.toLowerCase()) {
        case 'k':
          event.preventDefault();
          console.log('Open search');
          break;
        case 'n':
          event.preventDefault();
          console.log('Create new');
          break;
      }
    }
  }
}
```

---

## 8. Two-Way Binding {#two-way-binding}

Two-Way Binding ทำให้ข้อมูลไหลสองทิศทาง: ทั้งจาก component ไปยัง template และจาก template กลับมายัง component

```typescript
// two-way-binding.component.ts
import { Component } from '@angular/core';
import { FormsModule } from '@angular/forms';

@Component({
  selector: 'app-two-way',
  imports: [FormsModule],
  template: `
    <div>
      <!-- Two-way binding กับ ngModel -->
      <input [(ngModel)]="username" placeholder="กรอกชื่อผู้ใช้">
      <p>ชื่อผู้ใช้: {{ username }}</p>
      
      <!-- Two-way binding กับ select -->
      <select [(ngModel)]="selectedCity">
        <option value="">เลือกจังหวัด</option>
        <option *ngFor="let city of cities" [value]="city.id">
          {{ city.name }}
        </option>
      </select>
      <p>จังหวัดที่เลือก: {{ selectedCity }}</p>
      
      <!-- Two-way binding กับ checkbox -->
      <label>
        <input type="checkbox" [(ngModel)]="isAgree">
        ยอมรับเงื่อนไข
      </label>
      <p>ยอมรับ: {{ isAgree }}</p>
      
      <!-- Two-way binding กับ radio -->
      <label *ngFor="let option of genderOptions">
        <input
          type="radio"
          [(ngModel)]="selectedGender"
          [value]="option.value">
        {{ option.label }}
      </label>
      <p>เพศ: {{ selectedGender }}</p>
      
      <!-- Two-way binding กับ textarea -->
      <textarea
        [(ngModel)]="description"
        rows="4"
        placeholder="กรอกคำอธิบาย">
      </textarea>
      <p>จำนวนตัวอักษร: {{ description.length }}</p>
      
      <!-- Custom two-way binding ด้วย EventEmitter -->
      <app-counter [(count)]="counterValue"></app-counter>
      <p>Counter value: {{ counterValue }}</p>
    </div>
  `
})
export class TwoWayBindingComponent {
  username = '';
  selectedCity = '';
  isAgree = false;
  selectedGender = '';
  description = '';
  counterValue = 0;

  cities = [
    { id: 'bkk', name: 'กรุงเทพมหานคร' },
    { id: 'cnx', name: 'เชียงใหม่' },
    { id: 'hkt', name: 'ภูเก็ต' },
    { id: 'kbi', name: 'กระบี่' }
  ];

  genderOptions = [
    { value: 'male', label: 'ชาย' },
    { value: 'female', label: 'หญิง' },
    { value: 'other', label: 'อื่นๆ' }
  ];
}
```

### Custom Two-Way Binding Component

```typescript
// counter.component.ts
import { Component, Input, Output, EventEmitter } from '@angular/core';

@Component({
  selector: 'app-counter',
  template: `
    <div class="counter">
      <button (click)="decrement()">-</button>
      <span>{{ count }}</span>
      <button (click)="increment()">+</button>
    </div>
  `
})
export class CounterComponent {
  @Input() count = 0;
  // ต้องตั้งชื่อ Output เป็น inputName + 'Change'
  @Output() countChange = new EventEmitter<number>();

  increment(): void {
    this.count++;
    this.countChange.emit(this.count);
  }

  decrement(): void {
    this.count--;
    this.countChange.emit(this.count);
  }
}

// การใช้งาน:
// <app-counter [(count)]="myValue"></app-counter>
// เทียบเท่ากับ:
// <app-counter [count]="myValue" (countChange)="myValue = $event"></app-counter>
```

---

## 9. ngModel ใน Reactive Forms {#ngmodel-reactive-forms}

### Template-driven Forms

```typescript
// template-form.component.ts
import { Component } from '@angular/core';
import { FormsModule, NgForm } from '@angular/forms';

interface UserForm {
  username: string;
  email: string;
  password: string;
  role: string;
  notifications: boolean;
}

@Component({
  selector: 'app-template-form',
  imports: [FormsModule],
  template: `
    <form #userForm="ngForm" (ngSubmit)="onSubmit(userForm)">
      <div class="form-group">
        <label>ชื่อผู้ใช้</label>
        <input
          type="text"
          name="username"
          [(ngModel)]="formData.username"
          #username="ngModel"
          required
          minlength="3"
          maxlength="20"
          [class.error]="username.invalid && username.touched">
        
        <div *ngIf="username.invalid && username.touched">
          <span *ngIf="username.errors?.['required']">กรุณากรอกชื่อผู้ใช้</span>
          <span *ngIf="username.errors?.['minlength']">ชื่อต้องมีอย่างน้อย 3 ตัวอักษร</span>
          <span *ngIf="username.errors?.['maxlength']">ชื่อต้องไม่เกิน 20 ตัวอักษร</span>
        </div>
      </div>
      
      <div class="form-group">
        <label>อีเมล</label>
        <input
          type="email"
          name="email"
          [(ngModel)]="formData.email"
          #email="ngModel"
          required
          email
          [class.error]="email.invalid && email.touched">
        
        <div *ngIf="email.invalid && email.touched">
          <span *ngIf="email.errors?.['required']">กรุณากรอกอีเมล</span>
          <span *ngIf="email.errors?.['email']">รูปแบบอีเมลไม่ถูกต้อง</span>
        </div>
      </div>
      
      <div class="form-group">
        <label>รหัสผ่าน</label>
        <input
          type="password"
          name="password"
          [(ngModel)]="formData.password"
          #password="ngModel"
          required
          minlength="8"
          pattern="^(?=.*[A-Za-z])(?=.*\d)[A-Za-z\d]{8,}$"
          [class.error]="password.invalid && password.touched">
        
        <div *ngIf="password.invalid && password.touched">
          <span *ngIf="password.errors?.['required']">กรุณากรอกรหัสผ่าน</span>
          <span *ngIf="password.errors?.['minlength']">รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร</span>
          <span *ngIf="password.errors?.['pattern']">รหัสผ่านต้องมีทั้งตัวอักษรและตัวเลข</span>
        </div>
      </div>
      
      <div class="form-group">
        <label>บทบาท</label>
        <select
          name="role"
          [(ngModel)]="formData.role"
          #role="ngModel"
          required>
          <option value="">-- เลือกบทบาท --</option>
          <option value="admin">ผู้ดูแลระบบ</option>
          <option value="editor">บรรณาธิการ</option>
          <option value="viewer">ผู้ชม</option>
        </select>
      </div>
      
      <div class="form-group">
        <label>
          <input
            type="checkbox"
            name="notifications"
            [(ngModel)]="formData.notifications">
          รับการแจ้งเตือน
        </label>
      </div>
      
      <div class="form-status">
        <p>สถานะฟอร์ม: {{ userForm.valid ? 'ถูกต้อง' : 'ไม่ถูกต้อง' }}</p>
        <p>ข้อมูลสกปรก: {{ userForm.dirty }}</p>
        <p>ข้อมูลถูกแตะ: {{ userForm.touched }}</p>
      </div>
      
      <button
        type="submit"
        [disabled]="userForm.invalid || isSubmitting">
        {{ isSubmitting ? 'กำลังส่ง...' : 'ส่งข้อมูล' }}
      </button>
      
      <button type="button" (click)="resetForm(userForm)">รีเซ็ต</button>
    </form>
  `,
  styles: [`
    .form-group { margin-bottom: 16px; }
    .form-group label { display: block; margin-bottom: 4px; font-weight: bold; }
    .form-group input, .form-group select, .form-group textarea {
      width: 100%; padding: 8px; border: 1px solid #ddd; border-radius: 4px;
    }
    .form-group input.error, .form-group select.error { border-color: red; }
    .form-group div span { color: red; font-size: 12px; display: block; }
    button[type="submit"] { 
      background: #007bff; color: white; padding: 10px 20px; 
      border: none; border-radius: 4px; cursor: pointer; 
    }
    button[type="submit"]:disabled { background: #ccc; cursor: not-allowed; }
  `]
})
export class TemplateFormComponent {
  isSubmitting = false;

  formData: UserForm = {
    username: '',
    email: '',
    password: '',
    role: '',
    notifications: false
  };

  async onSubmit(form: NgForm): Promise<void> {
    if (form.invalid) return;
    
    this.isSubmitting = true;
    try {
      // จำลองการส่งข้อมูล
      await new Promise(resolve => setTimeout(resolve, 2000));
      console.log('Form data:', this.formData);
      alert('ส่งข้อมูลสำเร็จ!');
      form.resetForm();
    } catch (error) {
      console.error('Error:', error);
    } finally {
      this.isSubmitting = false;
    }
  }

  resetForm(form: NgForm): void {
    form.resetForm();
    this.formData = {
      username: '',
      email: '',
      password: '',
      role: '',
      notifications: false
    };
  }
}
```

---

## 10. Template Reference Variables ขั้นสูง {#template-reference-variables}

Template Reference Variables ใช้ `#varName` เพื่ออ้างอิงถึง element, component หรือ directive ใน template

```typescript
// template-ref.component.ts
import { Component, ViewChild, ElementRef, AfterViewInit } from '@angular/core';

@Component({
  selector: 'app-template-ref',
  template: `
    <div>
      <!-- อ้างอิงถึง DOM element -->
      <input #emailInput type="email" placeholder="กรอกอีเมล">
      <button (click)="focusEmail()">Focus Email</button>
      <button (click)="clearEmail()">Clear Email</button>
      <p>ค่าปัจจุบัน: {{ emailInput.value }}</p>
      
      <!-- อ้างอิงถึง component -->
      <app-child-component #childComp></app-child-component>
      <button (click)="callChildMethod()">เรียก method ของ child</button>
      <button (click)="getChildData()">ดูข้อมูล child</button>
      
      <!-- อ้างอิงถึง NgForm -->
      <form #myForm="ngForm" (ngSubmit)="onSubmit(myForm)">
        <input name="name" ngModel required>
        <button type="submit" [disabled]="myForm.invalid">ส่ง</button>
        <p>Form valid: {{ myForm.valid }}</p>
      </form>
      
      <!-- อ้างอิงถึง NgModel -->
      <input
        #nameField="ngModel"
        name="fullName"
        ngModel
        required
        minlength="2">
      <p>Field status: {{ nameField.status }}</p>
      <p>Field dirty: {{ nameField.dirty }}</p>
      
      <!-- ใช้กับ ng-template -->
      <ng-template #loadingTemplate>
        <div class="spinner">กำลังโหลด...</div>
      </ng-template>
      
      <ng-template #errorTemplate let-message>
        <div class="error">เกิดข้อผิดพลาด: {{ message }}</div>
      </ng-template>
      
      <div *ngIf="isLoading; else contentOrError">
        <ng-container [ngTemplateOutlet]="loadingTemplate"></ng-container>
      </div>
      
      <ng-template #contentOrError>
        <div *ngIf="!hasError; else errorContent">
          <p>เนื้อหาปกติ</p>
        </div>
        <ng-template #errorContent>
          <ng-container
            [ngTemplateOutlet]="errorTemplate"
            [ngTemplateOutletContext]="{ $implicit: errorMessage }">
          </ng-container>
        </ng-template>
      </ng-template>
    </div>
  `
})
export class TemplateRefComponent implements AfterViewInit {
  @ViewChild('emailInput') emailInputRef!: ElementRef<HTMLInputElement>;
  @ViewChild('childComp') childComponent!: any;
  
  isLoading = false;
  hasError = false;
  errorMessage = 'ไม่พบข้อมูล';

  ngAfterViewInit(): void {
    // สามารถเข้าถึง ViewChild ได้หลังจาก view initialized
    console.log('Email input element:', this.emailInputRef.nativeElement);
  }

  focusEmail(): void {
    this.emailInputRef.nativeElement.focus();
  }

  clearEmail(): void {
    this.emailInputRef.nativeElement.value = '';
    this.emailInputRef.nativeElement.focus();
  }

  callChildMethod(): void {
    if (this.childComponent) {
      this.childComponent.doSomething();
    }
  }

  getChildData(): void {
    if (this.childComponent) {
      console.log('Child data:', this.childComponent.data);
    }
  }

  onSubmit(form: any): void {
    console.log('Form submitted:', form.value);
  }
}
```

---

## 11. ng-template {#ng-template}

`ng-template` ใช้สำหรับสร้าง template ที่ไม่ render โดยตรง แต่สามารถนำมาใช้ซ้ำได้

```typescript
// ng-template.component.ts
import { Component, TemplateRef, ViewContainerRef, ViewChild } from '@angular/core';

@Component({
  selector: 'app-ng-template-demo',
  template: `
    <div>
      <!-- ng-template พื้นฐาน - ไม่ render โดยตรง -->
      <ng-template #greetingTemplate>
        <p>สวัสดี! นี่คือ template</p>
      </ng-template>
      
      <!-- ใช้ ngTemplateOutlet เพื่อ render template -->
      <ng-container [ngTemplateOutlet]="greetingTemplate"></ng-container>
      <ng-container [ngTemplateOutlet]="greetingTemplate"></ng-container>
      
      <!-- ng-template กับ context (ส่งข้อมูลเข้าไป) -->
      <ng-template #userCard let-user let-index="index">
        <div class="user-card">
          <h3>#{{ index + 1 }}: {{ user.name }}</h3>
          <p>อีเมล: {{ user.email }}</p>
          <p>บทบาท: {{ user.role }}</p>
        </div>
      </ng-template>
      
      <!-- Render template พร้อม context -->
      <ng-container
        *ngFor="let user of users; let i = index"
        [ngTemplateOutlet]="userCard"
        [ngTemplateOutletContext]="{ $implicit: user, index: i }">
      </ng-container>
      
      <!-- ng-template สำหรับ loading/error/empty states -->
      <ng-template #loading>
        <div class="skeleton-loader">
          <div class="skeleton-line"></div>
          <div class="skeleton-line"></div>
          <div class="skeleton-line short"></div>
        </div>
      </ng-template>
      
      <ng-template #error let-msg>
        <div class="error-state">
          <span class="icon">⚠️</span>
          <p>{{ msg || 'เกิดข้อผิดพลาด' }}</p>
          <button (click)="retry()">ลองใหม่</button>
        </div>
      </ng-template>
      
      <ng-template #empty>
        <div class="empty-state">
          <p>ไม่มีข้อมูล</p>
          <button (click)="loadData()">โหลดข้อมูล</button>
        </div>
      </ng-template>
      
      <!-- Conditional rendering ด้วย ng-template -->
      <div [ngSwitch]="dataState">
        <ng-container *ngSwitchCase="'loading'"
          [ngTemplateOutlet]="loading">
        </ng-container>
        <ng-container *ngSwitchCase="'error'"
          [ngTemplateOutlet]="error"
          [ngTemplateOutletContext]="{ $implicit: errorMsg }">
        </ng-container>
        <ng-container *ngSwitchCase="'empty'"
          [ngTemplateOutlet]="empty">
        </ng-container>
        <ng-container *ngSwitchDefault>
          <div>เนื้อหาข้อมูล</div>
        </ng-container>
      </div>
      
      <!-- Dynamic template rendering -->
      <ng-container [ngTemplateOutlet]="currentTemplate"></ng-container>
      
      <ng-template #template1>
        <div>Template 1: หน้าแรก</div>
      </ng-template>
      
      <ng-template #template2>
        <div>Template 2: หน้าที่สอง</div>
      </ng-template>
      
      <button (click)="showTemplate(template1)">แสดง Template 1</button>
      <button (click)="showTemplate(template2)">แสดง Template 2</button>
    </div>
  `
})
export class NgTemplateDemoComponent {
  dataState: 'loading' | 'error' | 'empty' | 'data' = 'data';
  errorMsg = 'การเชื่อมต่อล้มเหลว';
  currentTemplate!: TemplateRef<any>;

  users = [
    { name: 'สมชาย ใจดี', email: 'somchai@example.com', role: 'Admin' },
    { name: 'สมหญิง รักดี', email: 'somying@example.com', role: 'Editor' },
    { name: 'ประชา ชูใจ', email: 'pracha@example.com', role: 'Viewer' }
  ];

  showTemplate(template: TemplateRef<any>): void {
    this.currentTemplate = template;
  }

  retry(): void {
    this.dataState = 'loading';
    setTimeout(() => {
      this.dataState = 'data';
    }, 2000);
  }

  loadData(): void {
    this.dataState = 'loading';
    setTimeout(() => {
      this.dataState = 'data';
    }, 1500);
  }
}
```

---

## 12. ng-container {#ng-container}

`ng-container` เป็น logical container ที่ไม่ render HTML element ใดๆ เหมาะสำหรับจัดกลุ่ม elements โดยไม่เพิ่ม DOM node

```typescript
// ng-container.component.ts
import { Component } from '@angular/core';

@Component({
  selector: 'app-ng-container-demo',
  template: `
    <div>
      <!-- ปัญหา: ต้องการใช้หลาย directive บน element เดียว -->
      <!-- ไม่ถูกต้อง: ใส่ *ngIf และ *ngFor บน element เดียวกัน -->
      <!-- <div *ngIf="isVisible" *ngFor="let item of items">{{ item }}</div> -->
      
      <!-- ถูกต้อง: ใช้ ng-container -->
      <ng-container *ngIf="isVisible">
        <div *ngFor="let item of items; trackBy: trackById">
          {{ item.name }}
        </div>
      </ng-container>
      
      <!-- ng-container สำหรับจัดกลุ่มโดยไม่เพิ่ม wrapper -->
      <table>
        <tr>
          <ng-container *ngFor="let col of columns">
            <th>{{ col.header }}</th>
          </ng-container>
        </tr>
        <tr *ngFor="let row of rows">
          <ng-container *ngFor="let col of columns">
            <td>{{ row[col.field] }}</td>
          </ng-container>
        </tr>
      </table>
      
      <!-- ng-container กับ ngSwitch -->
      <ng-container [ngSwitch]="userRole">
        <ng-container *ngSwitchCase="'admin'">
          <button>จัดการผู้ใช้</button>
          <button>ตั้งค่าระบบ</button>
          <button>ดูรายงาน</button>
        </ng-container>
        <ng-container *ngSwitchCase="'editor'">
          <button>แก้ไขเนื้อหา</button>
          <button>อัพโหลดไฟล์</button>
        </ng-container>
        <ng-container *ngSwitchDefault>
          <button>ดูเนื้อหา</button>
        </ng-container>
      </ng-container>
      
      <!-- ng-container กับ async pipe -->
      <ng-container *ngIf="data$ | async as data; else loading">
        <p>ข้อมูล: {{ data.title }}</p>
        <p>รายละเอียด: {{ data.description }}</p>
      </ng-container>
      
      <ng-template #loading>
        <p>กำลังโหลด...</p>
      </ng-template>
      
      <!-- Nested ng-container -->
      <ng-container *ngIf="showSection">
        <h2>ส่วนที่ 1</h2>
        <ng-container *ngFor="let group of groups">
          <h3>{{ group.title }}</h3>
          <ng-container *ngFor="let item of group.items">
            <div [class.active]="item.active">{{ item.name }}</div>
          </ng-container>
        </ng-container>
      </ng-container>
    </div>
  `
})
export class NgContainerDemoComponent {
  isVisible = true;
  userRole = 'admin';
  showSection = true;
  data$ = Promise.resolve({ title: 'ข้อมูลตัวอย่าง', description: 'รายละเอียด' });

  items = [
    { id: 1, name: 'สินค้า A' },
    { id: 2, name: 'สินค้า B' },
    { id: 3, name: 'สินค้า C' }
  ];

  columns = [
    { header: 'ชื่อ', field: 'name' },
    { header: 'ราคา', field: 'price' },
    { header: 'จำนวน', field: 'quantity' }
  ];

  rows = [
    { name: 'สินค้า A', price: 100, quantity: 5 },
    { name: 'สินค้า B', price: 200, quantity: 3 }
  ];

  groups = [
    {
      title: 'กลุ่ม A',
      items: [
        { name: 'รายการ 1', active: true },
        { name: 'รายการ 2', active: false }
      ]
    },
    {
      title: 'กลุ่ม B',
      items: [
        { name: 'รายการ 3', active: true }
      ]
    }
  ];

  trackById(index: number, item: { id: number }): number {
    return item.id;
  }
}
```

---

## 13. ng-content และ Content Projection {#ng-content}

Content Projection ช่วยให้ component สามารถรับและแสดง HTML content จากภายนอกได้

```typescript
// card.component.ts - Component ที่รับ content projection
import { Component } from '@angular/core';

@Component({
  selector: 'app-card',
  template: `
    <div class="card">
      <!-- Single slot projection -->
      <div class="card-header">
        <ng-content select="[card-title]"></ng-content>
      </div>
      
      <div class="card-body">
        <ng-content select="[card-body]"></ng-content>
      </div>
      
      <div class="card-footer" *ngIf="hasFooter">
        <ng-content select="[card-footer]"></ng-content>
      </div>
      
      <!-- Default slot สำหรับ content ที่ไม่มี selector -->
      <ng-content></ng-content>
    </div>
  `,
  styles: [`
    .card { 
      border: 1px solid #ddd; 
      border-radius: 8px; 
      overflow: hidden;
      margin: 16px 0;
      box-shadow: 0 2px 4px rgba(0,0,0,0.1);
    }
    .card-header { 
      background: #f5f5f5; 
      padding: 16px; 
      border-bottom: 1px solid #ddd; 
    }
    .card-body { padding: 16px; }
    .card-footer { 
      background: #f9f9f9; 
      padding: 12px 16px; 
      border-top: 1px solid #ddd; 
    }
  `]
})
export class CardComponent {
  hasFooter = true;
}

// การใช้งาน card component
@Component({
  selector: 'app-card-usage',
  template: `
    <app-card>
      <!-- ส่ง content เข้าไปยัง slot ต่างๆ -->
      <h2 card-title>หัวข้อของ Card</h2>
      
      <div card-body>
        <p>นี่คือเนื้อหาหลักของ card</p>
        <p>สามารถใส่ HTML ใดๆ ก็ได้</p>
        <ul>
          <li>รายการที่ 1</li>
          <li>รายการที่ 2</li>
        </ul>
      </div>
      
      <div card-footer>
        <button>ยืนยัน</button>
        <button>ยกเลิก</button>
      </div>
    </app-card>
    
    <!-- Card แบบง่ายๆ ไม่มี slot -->
    <app-card>
      <p>เนื้อหาใน default slot</p>
    </app-card>
  `
})
export class CardUsageComponent {}
```

### Advanced Content Projection

```typescript
// tabs.component.ts - Complex content projection
import { Component, ContentChildren, QueryList, AfterContentInit } from '@angular/core';
import { TabComponent } from './tab.component';

@Component({
  selector: 'app-tabs',
  template: `
    <div class="tabs">
      <!-- Tab headers -->
      <div class="tab-header">
        <button
          *ngFor="let tab of tabs; let i = index"
          class="tab-btn"
          [class.active]="activeIndex === i"
          (click)="selectTab(i)">
          {{ tab.title }}
        </button>
      </div>
      
      <!-- Tab content (projected) -->
      <div class="tab-content">
        <ng-content></ng-content>
      </div>
    </div>
  `
})
export class TabsComponent implements AfterContentInit {
  @ContentChildren(TabComponent) tabs!: QueryList<TabComponent>;
  activeIndex = 0;

  ngAfterContentInit(): void {
    this.tabs.forEach((tab, index) => {
      tab.isActive = index === 0;
    });
  }

  selectTab(index: number): void {
    this.activeIndex = index;
    this.tabs.forEach((tab, i) => {
      tab.isActive = i === index;
    });
  }
}

// tab.component.ts
import { Component, Input } from '@angular/core';

@Component({
  selector: 'app-tab',
  template: `
    <div class="tab-pane" [class.active]="isActive" [hidden]="!isActive">
      <ng-content></ng-content>
    </div>
  `
})
export class TabComponent {
  @Input() title = '';
  isActive = false;
}

// การใช้งาน
@Component({
  selector: 'app-tabs-demo',
  template: `
    <app-tabs>
      <app-tab title="ข้อมูลส่วนตัว">
        <form>
          <input placeholder="ชื่อ">
          <input placeholder="นามสกุล">
        </form>
      </app-tab>
      
      <app-tab title="ที่อยู่">
        <form>
          <textarea placeholder="ที่อยู่"></textarea>
        </form>
      </app-tab>
      
      <app-tab title="ข้อมูลติดต่อ">
        <form>
          <input type="tel" placeholder="เบอร์โทรศัพท์">
          <input type="email" placeholder="อีเมล">
        </form>
      </app-tab>
    </app-tabs>
  `
})
export class TabsDemoComponent {}
```

---

## 14. Workshop: Dashboard Layout Component {#workshop}

ในส่วน Workshop นี้จะสร้าง Dashboard Layout Component ที่ครบสมบูรณ์ โดยใช้ความรู้ทั้งหมดจาก Part 04

### โครงสร้างโปรเจค

```
src/
├── app/
│   ├── dashboard/
│   │   ├── dashboard.component.ts
│   │   ├── dashboard.component.html
│   │   ├── dashboard.component.css
│   │   ├── components/
│   │   │   ├── sidebar/
│   │   │   │   └── sidebar.component.ts
│   │   │   ├── header/
│   │   │   │   └── header.component.ts
│   │   │   ├── stat-card/
│   │   │   │   └── stat-card.component.ts
│   │   │   └── data-table/
│   │   │       └── data-table.component.ts
│   │   └── models/
│   │       └── dashboard.model.ts
```

### Dashboard Models

```typescript
// dashboard.model.ts
export interface StatCard {
  id: string;
  title: string;
  value: number | string;
  unit?: string;
  icon: string;
  trend: 'up' | 'down' | 'neutral';
  trendPercent?: number;
  color: 'blue' | 'green' | 'orange' | 'red' | 'purple';
}

export interface TableRow {
  id: number;
  name: string;
  email: string;
  status: 'active' | 'inactive' | 'pending';
  role: string;
  lastLogin: Date;
  actions: string[];
}

export interface MenuItem {
  id: string;
  label: string;
  icon: string;
  route: string;
  badge?: number;
  children?: MenuItem[];
  isExpanded?: boolean;
}

export interface DashboardUser {
  name: string;
  email: string;
  avatar?: string;
  role: string;
  notifications: number;
}
```

### Stat Card Component

```typescript
// stat-card.component.ts
import { Component, Input } from '@angular/core';
import { StatCard } from '../models/dashboard.model';

@Component({
  selector: 'app-stat-card',
  template: `
    <div class="stat-card" [ngClass]="'stat-card--' + card.color">
      <div class="stat-card__icon">
        <span>{{ card.icon }}</span>
      </div>
      
      <div class="stat-card__content">
        <p class="stat-card__title">{{ card.title }}</p>
        <p class="stat-card__value">
          {{ card.value | number }}
          <span *ngIf="card.unit" class="stat-card__unit">{{ card.unit }}</span>
        </p>
      </div>
      
      <div class="stat-card__trend" *ngIf="card.trendPercent !== undefined">
        <span
          class="trend-indicator"
          [class.trend-up]="card.trend === 'up'"
          [class.trend-down]="card.trend === 'down'"
          [class.trend-neutral]="card.trend === 'neutral'">
          {{ card.trend === 'up' ? '↑' : card.trend === 'down' ? '↓' : '→' }}
          {{ card.trendPercent | number:'1.1-1' }}%
        </span>
        <span class="trend-period">เทียบเดือนที่แล้ว</span>
      </div>
    </div>
  `,
  styles: [`
    .stat-card {
      background: white;
      border-radius: 12px;
      padding: 20px;
      box-shadow: 0 2px 8px rgba(0,0,0,0.08);
      display: flex;
      flex-direction: column;
      gap: 12px;
      transition: transform 0.2s, box-shadow 0.2s;
    }
    .stat-card:hover {
      transform: translateY(-2px);
      box-shadow: 0 4px 16px rgba(0,0,0,0.12);
    }
    .stat-card__icon { font-size: 32px; }
    .stat-card__title { color: #666; font-size: 14px; margin: 0; }
    .stat-card__value { font-size: 28px; font-weight: 700; margin: 4px 0 0; }
    .stat-card__unit { font-size: 14px; font-weight: 400; color: #666; }
    .trend-up { color: #4caf50; }
    .trend-down { color: #f44336; }
    .trend-neutral { color: #ff9800; }
    .trend-period { font-size: 12px; color: #999; margin-left: 8px; }
    .stat-card--blue .stat-card__value { color: #1976d2; }
    .stat-card--green .stat-card__value { color: #388e3c; }
    .stat-card--orange .stat-card__value { color: #f57c00; }
    .stat-card--red .stat-card__value { color: #d32f2f; }
    .stat-card--purple .stat-card__value { color: #7b1fa2; }
  `]
})
export class StatCardComponent {
  @Input() card!: StatCard;
}
```

### Sidebar Component

```typescript
// sidebar.component.ts
import { Component, Input, Output, EventEmitter } from '@angular/core';
import { MenuItem } from '../models/dashboard.model';

@Component({
  selector: 'app-sidebar',
  template: `
    <aside class="sidebar" [class.collapsed]="isCollapsed">
      <!-- Logo -->
      <div class="sidebar__logo">
        <img *ngIf="!isCollapsed" src="assets/logo.svg" alt="Logo">
        <img *ngIf="isCollapsed" src="assets/logo-small.svg" alt="Logo">
      </div>
      
      <!-- Navigation Menu -->
      <nav class="sidebar__nav">
        <ul class="menu">
          <li
            *ngFor="let item of menuItems; trackBy: trackByItem"
            class="menu-item"
            [class.active]="item.id === activeItemId"
            [class.has-children]="item.children?.length">
            
            <a
              class="menu-link"
              [attr.href]="item.children ? null : item.route"
              (click)="onMenuClick(item, $event)">
              
              <span class="menu-icon">{{ item.icon }}</span>
              
              <ng-container *ngIf="!isCollapsed">
                <span class="menu-label">{{ item.label }}</span>
                
                <span class="menu-badge" *ngIf="item.badge">
                  {{ item.badge > 99 ? '99+' : item.badge }}
                </span>
                
                <span
                  class="menu-arrow"
                  *ngIf="item.children?.length"
                  [class.rotated]="item.isExpanded">
                  ›
                </span>
              </ng-container>
            </a>
            
            <!-- Submenu -->
            <ul
              class="submenu"
              *ngIf="item.children?.length && item.isExpanded && !isCollapsed">
              <li
                *ngFor="let child of item.children"
                class="submenu-item"
                [class.active]="child.id === activeItemId">
                <a
                  [attr.href]="child.route"
                  (click)="onMenuClick(child, $event)"
                  class="submenu-link">
                  {{ child.label }}
                </a>
              </li>
            </ul>
          </li>
        </ul>
      </nav>
      
      <!-- Collapse Button -->
      <button
        class="sidebar__toggle"
        (click)="toggleSidebar()"
        [title]="isCollapsed ? 'ขยาย sidebar' : 'ย่อ sidebar'">
        {{ isCollapsed ? '»' : '«' }}
      </button>
    </aside>
  `,
  styles: [`
    .sidebar {
      width: 256px;
      min-height: 100vh;
      background: #1e293b;
      color: white;
      display: flex;
      flex-direction: column;
      transition: width 0.3s ease;
    }
    .sidebar.collapsed { width: 64px; }
    .sidebar__logo { padding: 20px 16px; border-bottom: 1px solid #334155; }
    .sidebar__logo img { height: 40px; }
    .sidebar__nav { flex: 1; padding: 16px 0; overflow-y: auto; }
    .menu { list-style: none; margin: 0; padding: 0; }
    .menu-link {
      display: flex;
      align-items: center;
      padding: 10px 16px;
      color: #94a3b8;
      text-decoration: none;
      gap: 12px;
      cursor: pointer;
      transition: background 0.2s, color 0.2s;
    }
    .menu-item.active .menu-link,
    .menu-link:hover { background: #334155; color: white; }
    .menu-icon { font-size: 20px; min-width: 24px; text-align: center; }
    .menu-label { flex: 1; font-size: 14px; }
    .menu-badge {
      background: #ef4444;
      color: white;
      border-radius: 12px;
      padding: 2px 8px;
      font-size: 12px;
    }
    .menu-arrow {
      font-size: 18px;
      transition: transform 0.3s;
    }
    .menu-arrow.rotated { transform: rotate(90deg); }
    .submenu { list-style: none; margin: 0; padding: 0; background: #0f172a; }
    .submenu-link {
      display: block;
      padding: 8px 16px 8px 52px;
      color: #64748b;
      text-decoration: none;
      font-size: 13px;
      transition: color 0.2s;
    }
    .submenu-item.active .submenu-link,
    .submenu-link:hover { color: white; }
    .sidebar__toggle {
      margin: 16px;
      padding: 8px;
      background: #334155;
      border: none;
      color: white;
      border-radius: 6px;
      cursor: pointer;
    }
  `]
})
export class SidebarComponent {
  @Input() menuItems: MenuItem[] = [];
  @Input() activeItemId = '';
  @Input() isCollapsed = false;
  @Output() itemSelected = new EventEmitter<MenuItem>();
  @Output() collapsedChange = new EventEmitter<boolean>();

  onMenuClick(item: MenuItem, event: MouseEvent): void {
    if (item.children?.length) {
      event.preventDefault();
      item.isExpanded = !item.isExpanded;
    } else {
      event.preventDefault();
      this.itemSelected.emit(item);
    }
  }

  toggleSidebar(): void {
    this.isCollapsed = !this.isCollapsed;
    this.collapsedChange.emit(this.isCollapsed);
  }

  trackByItem(index: number, item: MenuItem): string {
    return item.id;
  }
}
```

### Header Component

```typescript
// header.component.ts
import { Component, Input, Output, EventEmitter } from '@angular/core';
import { DashboardUser } from '../models/dashboard.model';

@Component({
  selector: 'app-header',
  template: `
    <header class="header">
      <!-- Page Title -->
      <div class="header__left">
        <h1 class="page-title">{{ pageTitle }}</h1>
        <nav class="breadcrumb" *ngIf="breadcrumbs.length">
          <ng-container *ngFor="let crumb of breadcrumbs; let last = last">
            <a *ngIf="!last" [href]="crumb.route">{{ crumb.label }}</a>
            <span *ngIf="!last"> / </span>
            <span *ngIf="last">{{ crumb.label }}</span>
          </ng-container>
        </nav>
      </div>
      
      <!-- Actions -->
      <div class="header__right">
        <!-- Search -->
        <div class="search-box" [class.active]="isSearchActive">
          <input
            #searchInput
            type="search"
            placeholder="ค้นหา..."
            [(ngModel)]="searchQuery"
            (focus)="isSearchActive = true"
            (blur)="onSearchBlur()"
            (keyup.enter)="onSearch()">
          <span class="search-icon">🔍</span>
        </div>
        
        <!-- Notifications -->
        <button
          class="icon-btn notification-btn"
          (click)="toggleNotifications()"
          [title]="'การแจ้งเตือน ' + user.notifications + ' รายการ'">
          🔔
          <span class="badge" *ngIf="user.notifications > 0">
            {{ user.notifications > 99 ? '99+' : user.notifications }}
          </span>
        </button>
        
        <!-- Notification Panel -->
        <div class="notification-panel" *ngIf="showNotifications">
          <h4>การแจ้งเตือน</h4>
          <div class="notification-item" *ngFor="let notif of notifications">
            <span>{{ notif.icon }}</span>
            <div>
              <p>{{ notif.message }}</p>
              <small>{{ notif.time }}</small>
            </div>
          </div>
          <button (click)="clearNotifications()">ล้างทั้งหมด</button>
        </div>
        
        <!-- User Menu -->
        <div class="user-menu">
          <button class="user-btn" (click)="toggleUserMenu()">
            <div class="user-avatar" [style.background-color]="avatarColor">
              {{ user.name.charAt(0) }}
            </div>
            <div class="user-info" *ngIf="showUserInfo">
              <span class="user-name">{{ user.name }}</span>
              <span class="user-role">{{ user.role }}</span>
            </div>
            <span>▾</span>
          </button>
          
          <div class="user-dropdown" *ngIf="showUserDropdown">
            <a href="/profile">โปรไฟล์</a>
            <a href="/settings">ตั้งค่า</a>
            <hr>
            <button (click)="onLogout()">ออกจากระบบ</button>
          </div>
        </div>
      </div>
    </header>
  `,
  styles: [`
    .header {
      display: flex;
      align-items: center;
      justify-content: space-between;
      padding: 0 24px;
      height: 64px;
      background: white;
      border-bottom: 1px solid #e2e8f0;
      box-shadow: 0 1px 4px rgba(0,0,0,0.06);
      position: sticky;
      top: 0;
      z-index: 100;
    }
    .page-title { font-size: 20px; font-weight: 600; color: #1e293b; margin: 0; }
    .breadcrumb { font-size: 13px; color: #64748b; }
    .breadcrumb a { color: #3b82f6; text-decoration: none; }
    .header__right { display: flex; align-items: center; gap: 12px; }
    .search-box {
      position: relative;
      border: 1px solid #e2e8f0;
      border-radius: 8px;
      overflow: hidden;
    }
    .search-box input {
      border: none;
      padding: 8px 36px 8px 12px;
      outline: none;
      width: 200px;
    }
    .search-icon {
      position: absolute;
      right: 10px;
      top: 50%;
      transform: translateY(-50%);
      pointer-events: none;
    }
    .icon-btn {
      position: relative;
      background: none;
      border: none;
      cursor: pointer;
      font-size: 20px;
      padding: 8px;
    }
    .badge {
      position: absolute;
      top: 2px;
      right: 2px;
      background: #ef4444;
      color: white;
      border-radius: 10px;
      padding: 1px 5px;
      font-size: 10px;
      font-weight: bold;
    }
    .notification-panel {
      position: absolute;
      right: 80px;
      top: 64px;
      width: 320px;
      background: white;
      border: 1px solid #e2e8f0;
      border-radius: 12px;
      padding: 16px;
      box-shadow: 0 8px 24px rgba(0,0,0,0.12);
      z-index: 200;
    }
    .user-btn {
      display: flex;
      align-items: center;
      gap: 8px;
      background: none;
      border: none;
      cursor: pointer;
    }
    .user-avatar {
      width: 36px;
      height: 36px;
      border-radius: 50%;
      display: flex;
      align-items: center;
      justify-content: center;
      color: white;
      font-weight: bold;
    }
    .user-dropdown {
      position: absolute;
      right: 24px;
      top: 64px;
      background: white;
      border: 1px solid #e2e8f0;
      border-radius: 8px;
      padding: 8px 0;
      min-width: 160px;
      box-shadow: 0 8px 24px rgba(0,0,0,0.12);
      z-index: 200;
    }
    .user-dropdown a, .user-dropdown button {
      display: block;
      padding: 8px 16px;
      text-decoration: none;
      color: #334155;
      font-size: 14px;
      background: none;
      border: none;
      cursor: pointer;
      width: 100%;
      text-align: left;
    }
    .user-dropdown a:hover, .user-dropdown button:hover { background: #f8fafc; }
  `]
})
export class HeaderComponent {
  @Input() pageTitle = 'Dashboard';
  @Input() user!: DashboardUser;
  @Input() showUserInfo = true;
  @Input() breadcrumbs: { label: string; route: string }[] = [];
  @Output() logout = new EventEmitter<void>();
  @Output() search = new EventEmitter<string>();

  searchQuery = '';
  isSearchActive = false;
  showNotifications = false;
  showUserDropdown = false;

  get avatarColor(): string {
    const colors = ['#3b82f6', '#10b981', '#f59e0b', '#ef4444', '#8b5cf6'];
    const index = this.user?.name.charCodeAt(0) % colors.length;
    return colors[index];
  }

  notifications = [
    { icon: '📧', message: 'คุณมีอีเมลใหม่ 3 ฉบับ', time: '5 นาทีที่แล้ว' },
    { icon: '✅', message: 'Order #1234 ได้รับการยืนยันแล้ว', time: '1 ชั่วโมงที่แล้ว' },
    { icon: '⚠️', message: 'เซิร์ฟเวอร์ใช้งาน CPU สูง', time: '2 ชั่วโมงที่แล้ว' }
  ];

  onSearchBlur(): void {
    setTimeout(() => {
      this.isSearchActive = false;
    }, 200);
  }

  onSearch(): void {
    this.search.emit(this.searchQuery);
  }

  toggleNotifications(): void {
    this.showNotifications = !this.showNotifications;
    if (this.showNotifications) {
      this.showUserDropdown = false;
    }
  }

  toggleUserMenu(): void {
    this.showUserDropdown = !this.showUserDropdown;
    if (this.showUserDropdown) {
      this.showNotifications = false;
    }
  }

  clearNotifications(): void {
    this.notifications = [];
    this.user.notifications = 0;
    this.showNotifications = false;
  }

  onLogout(): void {
    this.logout.emit();
  }
}
```

### Dashboard Component หลัก

```typescript
// dashboard.component.ts
import { Component, OnInit } from '@angular/core';
import { StatCard, TableRow, MenuItem, DashboardUser } from './models/dashboard.model';

@Component({
  selector: 'app-dashboard',
  template: `
    <div class="dashboard-layout" [class.sidebar-collapsed]="isSidebarCollapsed">
      <!-- Sidebar -->
      <app-sidebar
        [menuItems]="menuItems"
        [activeItemId]="activeMenuId"
        [(isCollapsed)]="isSidebarCollapsed"
        (itemSelected)="onMenuItemSelected($event)">
      </app-sidebar>
      
      <!-- Main Content -->
      <div class="dashboard-main">
        <!-- Header -->
        <app-header
          [pageTitle]="currentPageTitle"
          [user]="currentUser"
          [breadcrumbs]="breadcrumbs"
          (logout)="onLogout()"
          (search)="onSearch($event)">
        </app-header>
        
        <!-- Content Area -->
        <main class="dashboard-content">
          <!-- Stat Cards Grid -->
          <section class="stats-grid">
            <app-stat-card
              *ngFor="let card of statCards; trackBy: trackById"
              [card]="card">
            </app-stat-card>
          </section>
          
          <!-- Data Table Section -->
          <section class="table-section">
            <div class="section-header">
              <h2>รายการผู้ใช้ล่าสุด</h2>
              <div class="section-actions">
                <input
                  type="search"
                  [(ngModel)]="tableSearch"
                  placeholder="ค้นหาในตาราง..."
                  class="table-search">
                <button class="btn btn-primary" (click)="addUser()">
                  + เพิ่มผู้ใช้ใหม่
                </button>
              </div>
            </div>
            
            <!-- Table -->
            <div class="table-wrapper">
              <table class="data-table">
                <thead>
                  <tr>
                    <th>
                      <input
                        type="checkbox"
                        (change)="toggleSelectAll($event)"
                        [checked]="allSelected">
                    </th>
                    <th (click)="sortBy('name')" class="sortable">
                      ชื่อ-นามสกุล
                      <span *ngIf="sortField === 'name'">
                        {{ sortDirection === 'asc' ? '↑' : '↓' }}
                      </span>
                    </th>
                    <th>อีเมล</th>
                    <th (click)="sortBy('status')" class="sortable">
                      สถานะ
                    </th>
                    <th>บทบาท</th>
                    <th>เข้าสู่ระบบล่าสุด</th>
                    <th>จัดการ</th>
                  </tr>
                </thead>
                <tbody>
                  <tr
                    *ngFor="let row of filteredAndSortedRows; trackBy: trackByRowId"
                    [class.selected]="selectedRows.has(row.id)"
                    (click)="toggleRowSelect(row.id)">
                    <td>
                      <input
                        type="checkbox"
                        [checked]="selectedRows.has(row.id)"
                        (change)="toggleRowSelect(row.id)"
                        (click)="$event.stopPropagation()">
                    </td>
                    <td>{{ row.name }}</td>
                    <td>{{ row.email }}</td>
                    <td>
                      <span class="badge" [ngClass]="'badge-' + row.status">
                        {{ getStatusLabel(row.status) }}
                      </span>
                    </td>
                    <td>{{ row.role }}</td>
                    <td>{{ row.lastLogin | date:'dd/MM/yyyy HH:mm' }}</td>
                    <td>
                      <button
                        *ngFor="let action of row.actions"
                        class="action-btn"
                        (click)="onAction(action, row, $event)">
                        {{ action }}
                      </button>
                    </td>
                  </tr>
                </tbody>
              </table>
              
              <!-- Empty state -->
              <div *ngIf="filteredAndSortedRows.length === 0" class="empty-state">
                <p>ไม่พบข้อมูลที่ค้นหา</p>
              </div>
            </div>
            
            <!-- Pagination -->
            <div class="pagination">
              <span>แสดง {{ filteredAndSortedRows.length }} จาก {{ tableRows.length }} รายการ</span>
              <div class="page-controls">
                <button [disabled]="currentPage === 1" (click)="currentPage = currentPage - 1">«</button>
                <button
                  *ngFor="let page of pageNumbers"
                  [class.active]="page === currentPage"
                  (click)="currentPage = page">
                  {{ page }}
                </button>
                <button [disabled]="currentPage === totalPages" (click)="currentPage = currentPage + 1">»</button>
              </div>
            </div>
          </section>
        </main>
      </div>
    </div>
  `,
  styles: [`
    .dashboard-layout {
      display: flex;
      min-height: 100vh;
      background: #f1f5f9;
    }
    .dashboard-main {
      flex: 1;
      display: flex;
      flex-direction: column;
      overflow: hidden;
    }
    .dashboard-content {
      padding: 24px;
      flex: 1;
      overflow-y: auto;
    }
    .stats-grid {
      display: grid;
      grid-template-columns: repeat(auto-fill, minmax(240px, 1fr));
      gap: 20px;
      margin-bottom: 28px;
    }
    .table-section {
      background: white;
      border-radius: 12px;
      padding: 24px;
      box-shadow: 0 2px 8px rgba(0,0,0,0.06);
    }
    .section-header {
      display: flex;
      justify-content: space-between;
      align-items: center;
      margin-bottom: 20px;
    }
    .section-header h2 { font-size: 18px; font-weight: 600; margin: 0; }
    .section-actions { display: flex; gap: 12px; align-items: center; }
    .table-search {
      padding: 8px 12px;
      border: 1px solid #e2e8f0;
      border-radius: 8px;
      outline: none;
    }
    .btn-primary {
      background: #3b82f6;
      color: white;
      border: none;
      padding: 8px 16px;
      border-radius: 8px;
      cursor: pointer;
    }
    .data-table { width: 100%; border-collapse: collapse; }
    .data-table th, .data-table td {
      padding: 12px 16px;
      text-align: left;
      border-bottom: 1px solid #f1f5f9;
    }
    .data-table th {
      background: #f8fafc;
      font-weight: 600;
      font-size: 13px;
      color: #64748b;
    }
    .data-table tr:hover td { background: #f8fafc; }
    .data-table tr.selected td { background: #eff6ff; }
    .sortable { cursor: pointer; }
    .badge {
      padding: 4px 10px;
      border-radius: 20px;
      font-size: 12px;
      font-weight: 500;
    }
    .badge-active { background: #dcfce7; color: #166534; }
    .badge-inactive { background: #f1f5f9; color: #475569; }
    .badge-pending { background: #fef3c7; color: #92400e; }
    .action-btn {
      background: none;
      border: 1px solid #e2e8f0;
      padding: 4px 10px;
      border-radius: 6px;
      cursor: pointer;
      font-size: 12px;
      margin-right: 4px;
    }
    .action-btn:hover { background: #f8fafc; }
    .pagination {
      display: flex;
      justify-content: space-between;
      align-items: center;
      margin-top: 16px;
      padding-top: 16px;
      border-top: 1px solid #f1f5f9;
      font-size: 14px;
      color: #64748b;
    }
    .page-controls { display: flex; gap: 4px; }
    .page-controls button {
      width: 32px;
      height: 32px;
      border: 1px solid #e2e8f0;
      background: white;
      border-radius: 6px;
      cursor: pointer;
    }
    .page-controls button.active { background: #3b82f6; color: white; border-color: #3b82f6; }
    .page-controls button:disabled { opacity: 0.5; cursor: not-allowed; }
    .empty-state { text-align: center; padding: 40px; color: #94a3b8; }
  `]
})
export class DashboardComponent implements OnInit {
  isSidebarCollapsed = false;
  activeMenuId = 'dashboard';
  currentPageTitle = 'Dashboard';
  tableSearch = '';
  sortField = 'name';
  sortDirection: 'asc' | 'desc' = 'asc';
  selectedRows = new Set<number>();
  currentPage = 1;
  itemsPerPage = 10;

  currentUser: DashboardUser = {
    name: 'สมชาย ใจดี',
    email: 'admin@example.com',
    role: 'ผู้ดูแลระบบ',
    notifications: 3
  };

  breadcrumbs = [
    { label: 'หน้าหลัก', route: '/' },
    { label: 'Dashboard', route: '/dashboard' }
  ];

  menuItems: MenuItem[] = [
    { id: 'dashboard', label: 'Dashboard', icon: '📊', route: '/dashboard' },
    { id: 'users', label: 'ผู้ใช้', icon: '👥', route: '/users', badge: 12 },
    {
      id: 'products',
      label: 'สินค้า',
      icon: '📦',
      route: '/products',
      isExpanded: false,
      children: [
        { id: 'product-list', label: 'รายการสินค้า', icon: '', route: '/products/list' },
        { id: 'product-add', label: 'เพิ่มสินค้า', icon: '', route: '/products/add' },
        { id: 'product-cat', label: 'หมวดหมู่', icon: '', route: '/products/categories' }
      ]
    },
    { id: 'orders', label: 'คำสั่งซื้อ', icon: '🛒', route: '/orders', badge: 5 },
    { id: 'reports', label: 'รายงาน', icon: '📈', route: '/reports' },
    { id: 'settings', label: 'ตั้งค่า', icon: '⚙️', route: '/settings' }
  ];

  statCards: StatCard[] = [
    {
      id: '1', title: 'ผู้ใช้ทั้งหมด', value: 12458, unit: 'คน',
      icon: '👥', trend: 'up', trendPercent: 12.5, color: 'blue'
    },
    {
      id: '2', title: 'รายได้รวม', value: 2456789, unit: 'บาท',
      icon: '💰', trend: 'up', trendPercent: 8.3, color: 'green'
    },
    {
      id: '3', title: 'คำสั่งซื้อวันนี้', value: 156, unit: 'รายการ',
      icon: '🛒', trend: 'down', trendPercent: 3.2, color: 'orange'
    },
    {
      id: '4', title: 'สินค้าหมด', value: 23, unit: 'รายการ',
      icon: '⚠️', trend: 'up', trendPercent: 15, color: 'red'
    }
  ];

  tableRows: TableRow[] = [
    {
      id: 1, name: 'สมชาย ใจดี', email: 'somchai@example.com',
      status: 'active', role: 'Admin', lastLogin: new Date(),
      actions: ['แก้ไข', 'ลบ']
    },
    {
      id: 2, name: 'สมหญิง รักดี', email: 'somying@example.com',
      status: 'inactive', role: 'Editor', lastLogin: new Date('2024-01-15'),
      actions: ['แก้ไข', 'เปิดใช้งาน', 'ลบ']
    },
    {
      id: 3, name: 'ประชา ชูใจ', email: 'pracha@example.com',
      status: 'pending', role: 'Viewer', lastLogin: new Date('2024-01-10'),
      actions: ['อนุมัติ', 'ปฏิเสธ']
    }
  ];

  get filteredAndSortedRows(): TableRow[] {
    let rows = this.tableRows;
    
    if (this.tableSearch) {
      const search = this.tableSearch.toLowerCase();
      rows = rows.filter(r =>
        r.name.toLowerCase().includes(search) ||
        r.email.toLowerCase().includes(search)
      );
    }
    
    return [...rows].sort((a, b) => {
      const aVal = String(a[this.sortField as keyof TableRow]);
      const bVal = String(b[this.sortField as keyof TableRow]);
      return this.sortDirection === 'asc'
        ? aVal.localeCompare(bVal)
        : bVal.localeCompare(aVal);
    });
  }

  get allSelected(): boolean {
    return this.tableRows.length > 0 &&
      this.tableRows.every(r => this.selectedRows.has(r.id));
  }

  get totalPages(): number {
    return Math.ceil(this.filteredAndSortedRows.length / this.itemsPerPage);
  }

  get pageNumbers(): number[] {
    return Array.from({ length: this.totalPages }, (_, i) => i + 1);
  }

  ngOnInit(): void {
    console.log('Dashboard initialized');
  }

  onMenuItemSelected(item: MenuItem): void {
    this.activeMenuId = item.id;
    this.currentPageTitle = item.label;
  }

  sortBy(field: string): void {
    if (this.sortField === field) {
      this.sortDirection = this.sortDirection === 'asc' ? 'desc' : 'asc';
    } else {
      this.sortField = field;
      this.sortDirection = 'asc';
    }
  }

  toggleSelectAll(event: Event): void {
    const checkbox = event.target as HTMLInputElement;
    if (checkbox.checked) {
      this.tableRows.forEach(r => this.selectedRows.add(r.id));
    } else {
      this.selectedRows.clear();
    }
  }

  toggleRowSelect(id: number): void {
    if (this.selectedRows.has(id)) {
      this.selectedRows.delete(id);
    } else {
      this.selectedRows.add(id);
    }
  }

  getStatusLabel(status: string): string {
    const labels: Record<string, string> = {
      active: 'ใช้งาน',
      inactive: 'ไม่ใช้งาน',
      pending: 'รอดำเนินการ'
    };
    return labels[status] || status;
  }

  onAction(action: string, row: TableRow, event: MouseEvent): void {
    event.stopPropagation();
    console.log(`Action: ${action} on row:`, row);
    alert(`${action} ผู้ใช้: ${row.name}`);
  }

  addUser(): void {
    console.log('Add new user');
    alert('เปิดฟอร์มเพิ่มผู้ใช้ใหม่');
  }

  onLogout(): void {
    if (confirm('ต้องการออกจากระบบ?')) {
      console.log('Logging out...');
    }
  }

  onSearch(query: string): void {
    console.log('Search:', query);
  }

  trackById(index: number, card: StatCard): string {
    return card.id;
  }

  trackByRowId(index: number, row: TableRow): number {
    return row.id;
  }
}
```

---

## สรุป Part 04

ในส่วนนี้เราได้เรียนรู้:

| หัวข้อ | สิ่งที่สำคัญ |
|--------|-------------|
| Template Syntax | `{{ }}`, `[]`, `()`, `[()]`, `#ref` |
| Interpolation | expression, method calls, optional chaining |
| Property Binding | DOM properties, component inputs |
| Attribute Binding | `[attr.name]` สำหรับ aria, colspan, data-* |
| Class Binding | `[class.name]`, `[ngClass]` |
| Style Binding | `[style.prop]`, `[ngStyle]` |
| Event Binding | `(event)`, `$event`, key filters |
| Two-Way Binding | `[(ngModel)]`, custom `[(prop)]` |
| Template Variables | `#ref`, `@ViewChild` |
| ng-template | reusable templates, context |
| ng-container | logical grouping |
| ng-content | content projection, named slots |

---

*ต่อไป: [Part 05 — Built-in Directives](part-05-directives.md)*
