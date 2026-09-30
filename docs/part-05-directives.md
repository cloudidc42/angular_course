# Part 05 — Built-in Directives

## สารบัญ

1. [Directives คืออะไร](#directives-intro)
2. [Structural Directives: *ngIf](#ngif)
3. [Structural Directives: *ngFor](#ngfor)
4. [Structural Directives: *ngSwitch](#ngswitch)
5. [New Syntax Angular 17+: @if](#new-if)
6. [New Syntax Angular 17+: @for](#new-for)
7. [New Syntax Angular 17+: @switch](#new-switch)
8. [Attribute Directives: ngClass](#ngclass)
9. [Attribute Directives: ngStyle](#ngstyle)
10. [ngModel และ ngForm](#ngmodel-form)
11. [Built-in Pipes ที่ใช้กับ Directives](#pipes-with-directives)
12. [Custom Structural Directive พื้นฐาน](#custom-directive)
13. [Workshop: Product Filter UI](#workshop)

---

## 1. Directives คืออะไร {#directives-intro}

Directive คือ class ที่เพิ่ม behavior พิเศษให้กับ elements ใน Angular template โดยมี 3 ประเภทหลัก:

| ประเภท | คำอธิบาย | ตัวอย่าง |
|--------|----------|----------|
| **Component** | Directive ที่มี template | `@Component(...)` |
| **Structural** | เปลี่ยนโครงสร้าง DOM (เพิ่ม/ลบ elements) | `*ngIf`, `*ngFor`, `*ngSwitch` |
| **Attribute** | เปลี่ยน appearance หรือ behavior | `ngClass`, `ngStyle` |

```typescript
// การ import directives
import { NgIf, NgFor, NgSwitch, NgClass, NgStyle } from '@angular/common';
import { FormsModule } from '@angular/forms';

@Component({
  selector: 'app-demo',
  standalone: true,
  imports: [NgIf, NgFor, NgSwitch, NgClass, NgStyle, FormsModule],
  template: `...`
})
export class DemoComponent {}
```

---

## 2. Structural Directives: *ngIf {#ngif}

`*ngIf` ใช้สำหรับแสดงหรือซ่อน element ตามเงื่อนไข โดยจะ เพิ่ม/ลบ element จาก DOM จริงๆ

### การใช้งานพื้นฐาน

```typescript
// ngif-demo.component.ts
import { Component } from '@angular/core';
import { NgIf } from '@angular/common';
import { FormsModule } from '@angular/forms';

@Component({
  selector: 'app-ngif-demo',
  standalone: true,
  imports: [NgIf, FormsModule],
  template: `
    <div>
      <!-- *ngIf พื้นฐาน -->
      <p *ngIf="isVisible">ฉันแสดงเมื่อ isVisible = true</p>
      
      <!-- *ngIf กับ else -->
      <div *ngIf="isLoggedIn; else loginTemplate">
        <p>ยินดีต้อนรับ, {{ username }}!</p>
        <button (click)="logout()">ออกจากระบบ</button>
      </div>
      
      <ng-template #loginTemplate>
        <p>กรุณาเข้าสู่ระบบ</p>
        <button (click)="login()">เข้าสู่ระบบ</button>
      </ng-template>
      
      <!-- *ngIf กับ then และ else -->
      <ng-container
        *ngIf="userStatus === 'loading'; then loadingTpl; else contentTpl">
      </ng-container>
      
      <ng-template #loadingTpl>
        <div class="loading">⏳ กำลังโหลด...</div>
      </ng-template>
      
      <ng-template #contentTpl>
        <div class="content">✅ โหลดเสร็จแล้ว</div>
      </ng-template>
      
      <!-- *ngIf กับ as (เก็บค่าใน local variable) -->
      <div *ngIf="user$ | async as user">
        <p>ชื่อ: {{ user.name }}</p>
        <p>อีเมล: {{ user.email }}</p>
      </div>
      
      <!-- *ngIf กับ object -->
      <div *ngIf="selectedProduct">
        <h3>{{ selectedProduct?.name }}</h3>
        <p>ราคา: {{ selectedProduct?.price | number }}</p>
      </div>
      
      <!-- *ngIf กับ method -->
      <div *ngIf="hasPermission('admin')">
        <p>คุณมีสิทธิ์ผู้ดูแลระบบ</p>
      </div>
      
      <!-- ซ้อนกัน (Nested *ngIf) -->
      <div *ngIf="showSection">
        <p *ngIf="showDetails">รายละเอียดเพิ่มเติม</p>
        <p *ngIf="!showDetails">คลิกเพื่อดูรายละเอียด</p>
      </div>
      
      <!-- Toggle controls -->
      <div class="controls">
        <button (click)="isVisible = !isVisible">Toggle Visible</button>
        <button (click)="isLoggedIn = !isLoggedIn">Toggle Login</button>
        <select [(ngModel)]="userStatus">
          <option value="loading">Loading</option>
          <option value="ready">Ready</option>
        </select>
      </div>
    </div>
  `
})
export class NgIfDemoComponent {
  isVisible = true;
  isLoggedIn = false;
  username = 'สมชาย';
  userStatus = 'ready';
  showSection = true;
  showDetails = false;
  
  selectedProduct = {
    name: 'iPhone 15 Pro',
    price: 45000
  };
  
  user$ = Promise.resolve({
    name: 'สมชาย ใจดี',
    email: 'somchai@example.com'
  });

  login(): void {
    this.isLoggedIn = true;
  }

  logout(): void {
    this.isLoggedIn = false;
  }

  hasPermission(role: string): boolean {
    const userRoles = ['admin', 'editor'];
    return userRoles.includes(role);
  }
}
```

### ความแตกต่างระหว่าง *ngIf และ [hidden]

```html
<!-- *ngIf: ลบ element ออกจาก DOM จริงๆ (ประหยัด memory แต่ช้ากว่าเมื่อ toggle บ่อย) -->
<div *ngIf="isVisible">เนื้อหา</div>

<!-- [hidden]: ซ่อน element ด้วย CSS display:none (element ยังอยู่ใน DOM) -->
<div [hidden]="!isVisible">เนื้อหา</div>

<!-- ใช้ *ngIf เมื่อ: component มี lifecycle hooks สำคัญ, data ใหญ่, แสดงน้อยครั้ง -->
<!-- ใช้ [hidden] เมื่อ: toggle บ่อยมาก, ต้องการ preserve state -->
```

---

## 3. Structural Directives: *ngFor {#ngfor}

`*ngFor` ใช้สำหรับวน loop สร้าง elements จาก array

### การใช้งานพื้นฐาน

```typescript
// ngfor-demo.component.ts
import { Component } from '@angular/core';
import { NgFor, NgIf } from '@angular/common';

interface Product {
  id: number;
  name: string;
  price: number;
  category: string;
  inStock: boolean;
  tags: string[];
}

@Component({
  selector: 'app-ngfor-demo',
  standalone: true,
  imports: [NgFor, NgIf],
  template: `
    <div>
      <!-- *ngFor พื้นฐาน -->
      <ul>
        <li *ngFor="let item of simpleItems">{{ item }}</li>
      </ul>
      
      <!-- *ngFor กับ index -->
      <ul>
        <li *ngFor="let item of simpleItems; let i = index">
          {{ i + 1 }}. {{ item }}
        </li>
      </ul>
      
      <!-- *ngFor กับ built-in variables ทั้งหมด -->
      <div *ngFor="let product of products;
                   let i = index;
                   let first = first;
                   let last = last;
                   let even = even;
                   let odd = odd;
                   let count = count;
                   trackBy: trackByProductId">
        
        <div class="product-item"
          [class.first]="first"
          [class.last]="last"
          [class.even]="even"
          [class.odd]="odd">
          
          <span class="index">{{ i + 1 }}/{{ count }}</span>
          <strong>{{ product.name }}</strong>
          <span>{{ product.price | number:'1.0-0' }} บาท</span>
          
          <ng-container *ngIf="first">
            <span class="badge">⭐ สินค้าแรก</span>
          </ng-container>
          <ng-container *ngIf="last">
            <span class="badge">🏁 สินค้าสุดท้าย</span>
          </ng-container>
        </div>
      </div>
      
      <!-- *ngFor ซ้อนกัน (Nested) -->
      <div *ngFor="let category of categories">
        <h3>{{ category.name }}</h3>
        <ul>
          <li *ngFor="let product of getProductsByCategory(category.id)">
            {{ product.name }} - {{ product.price | number:'1.0-0' }} บาท
          </li>
        </ul>
      </div>
      
      <!-- *ngFor กับ Object (ใช้ keyvalue pipe) -->
      <div *ngFor="let entry of productMap | keyvalue">
        <strong>{{ entry.key }}:</strong> {{ entry.value }}
      </div>
      
      <!-- *ngFor กับ range (สร้าง array จาก numbers) -->
      <div class="stars">
        <span *ngFor="let star of [1,2,3,4,5]; let i = index"
          [class.filled]="i < rating">
          ★
        </span>
      </div>
      
      <!-- *ngFor กับ async -->
      <div *ngFor="let item of asyncItems$ | async">
        {{ item.name }}
      </div>
      
      <!-- Empty state -->
      <ng-container *ngIf="products.length > 0; else emptyState">
        <p>มีสินค้า {{ products.length }} รายการ</p>
      </ng-container>
      <ng-template #emptyState>
        <p>ไม่มีสินค้า</p>
      </ng-template>
    </div>
  `,
  styles: [`
    .product-item { padding: 8px; border: 1px solid #ddd; margin: 4px 0; }
    .even { background: #f5f5f5; }
    .odd { background: white; }
    .badge { margin-left: 8px; font-size: 12px; }
    .stars span { color: #ccc; font-size: 24px; }
    .stars .filled { color: #ffd700; }
  `]
})
export class NgForDemoComponent {
  simpleItems = ['Angular', 'React', 'Vue', 'Svelte'];
  rating = 4;

  products: Product[] = [
    { id: 1, name: 'iPhone 15', price: 35000, category: 'phone', inStock: true, tags: ['apple', 'smartphone'] },
    { id: 2, name: 'Samsung S24', price: 28000, category: 'phone', inStock: true, tags: ['samsung', 'android'] },
    { id: 3, name: 'MacBook Pro', price: 75000, category: 'laptop', inStock: false, tags: ['apple', 'laptop'] },
    { id: 4, name: 'Dell XPS 15', price: 55000, category: 'laptop', inStock: true, tags: ['dell', 'windows'] },
    { id: 5, name: 'iPad Pro', price: 32000, category: 'tablet', inStock: true, tags: ['apple', 'tablet'] }
  ];

  categories = [
    { id: 'phone', name: 'โทรศัพท์' },
    { id: 'laptop', name: 'แล็ปท็อป' },
    { id: 'tablet', name: 'แท็บเล็ต' }
  ];

  productMap = {
    brand: 'Apple',
    model: 'iPhone 15 Pro',
    color: 'Black Titanium',
    storage: '256GB'
  };

  asyncItems$ = Promise.resolve([
    { name: 'Async Item 1' },
    { name: 'Async Item 2' }
  ]);

  trackByProductId(index: number, product: Product): number {
    return product.id;
  }

  getProductsByCategory(categoryId: string): Product[] {
    return this.products.filter(p => p.category === categoryId);
  }
}
```

### trackBy - การ Optimize *ngFor

```typescript
// ทำไม trackBy ถึงสำคัญ?
// โดยปกติ Angular จะ re-render ทุก item เมื่อ array เปลี่ยน
// trackBy บอก Angular ว่า item ไหน unique เพื่อ re-render เฉพาะที่เปลี่ยน

@Component({
  template: `
    <!-- ไม่ดี: Angular ลบและสร้าง DOM ใหม่ทุกครั้ง -->
    <div *ngFor="let item of items">{{ item.name }}</div>
    
    <!-- ดี: Angular update เฉพาะ item ที่เปลี่ยน -->
    <div *ngFor="let item of items; trackBy: trackById">{{ item.name }}</div>
    
    <!-- ดีมาก: ใช้ arrow function (Angular 17+) -->
    <div *ngFor="let item of items; trackBy: trackByIdFn">{{ item.name }}</div>
  `
})
export class TrackByDemoComponent {
  items = [
    { id: 1, name: 'Item 1' },
    { id: 2, name: 'Item 2' }
  ];

  // Method-based trackBy
  trackById(index: number, item: { id: number }): number {
    return item.id;
  }

  // สามารถ track ด้วย index ถ้าไม่มี unique id
  trackByIndex(index: number): number {
    return index;
  }

  // Track ด้วยหลาย fields
  trackByComposite(index: number, item: any): string {
    return `${item.id}-${item.version}`;
  }

  trackByIdFn = (index: number, item: { id: number }) => item.id;

  // เพิ่ม/ลบ/เรียงลำดับโดยไม่ destroy ทุก element
  addItem(): void {
    this.items.push({ id: Date.now(), name: `Item ${this.items.length + 1}` });
  }

  removeItem(id: number): void {
    this.items = this.items.filter(item => item.id !== id);
  }

  shuffleItems(): void {
    this.items = [...this.items].sort(() => Math.random() - 0.5);
  }
}
```

---

## 4. Structural Directives: *ngSwitch {#ngswitch}

`*ngSwitch` ใช้สำหรับเลือกแสดง element ตามค่าที่กำหนด คล้าย switch statement ใน JavaScript

```typescript
// ngswitch-demo.component.ts
import { Component } from '@angular/core';
import { NgSwitch, NgSwitchCase, NgSwitchDefault, NgFor } from '@angular/common';
import { FormsModule } from '@angular/forms';

type UserRole = 'admin' | 'editor' | 'viewer' | 'guest';
type OrderStatus = 'pending' | 'processing' | 'shipped' | 'delivered' | 'cancelled';

@Component({
  selector: 'app-ngswitch-demo',
  standalone: true,
  imports: [NgSwitch, NgSwitchCase, NgSwitchDefault, NgFor, FormsModule],
  template: `
    <div>
      <!-- *ngSwitch พื้นฐาน -->
      <div [ngSwitch]="currentDay">
        <p *ngSwitchCase="'Monday'">วันจันทร์: เริ่มสัปดาห์ใหม่!</p>
        <p *ngSwitchCase="'Friday'">วันศุกร์: ใกล้จะหยุดแล้ว!</p>
        <p *ngSwitchCase="'Saturday'">วันเสาร์: วันหยุด!</p>
        <p *ngSwitchCase="'Sunday'">วันอาทิตย์: พักผ่อนให้เต็มที่!</p>
        <p *ngSwitchDefault>วันทำงานปกติ</p>
      </div>
      
      <!-- *ngSwitch กับ User Role -->
      <div [ngSwitch]="userRole">
        <ng-container *ngSwitchCase="'admin'">
          <div class="admin-panel">
            <h3>แดชบอร์ดผู้ดูแล</h3>
            <button>จัดการผู้ใช้</button>
            <button>ตั้งค่าระบบ</button>
            <button>ดูรายงาน</button>
            <button>สำรองข้อมูล</button>
          </div>
        </ng-container>
        
        <ng-container *ngSwitchCase="'editor'">
          <div class="editor-panel">
            <h3>เครื่องมือบรรณาธิการ</h3>
            <button>สร้างเนื้อหา</button>
            <button>แก้ไขเนื้อหา</button>
            <button>อัพโหลดสื่อ</button>
          </div>
        </ng-container>
        
        <ng-container *ngSwitchCase="'viewer'">
          <div class="viewer-panel">
            <h3>พื้นที่ผู้ชม</h3>
            <button>ดูเนื้อหา</button>
            <button>บันทึกรายการโปรด</button>
          </div>
        </ng-container>
        
        <ng-container *ngSwitchDefault>
          <div class="guest-panel">
            <h3>ยินดีต้อนรับ</h3>
            <button>เข้าสู่ระบบ</button>
            <button>สมัครสมาชิก</button>
          </div>
        </ng-container>
      </div>
      
      <!-- *ngSwitch กับ Order Status - แสดงเป็น step progress -->
      <div class="order-status" [ngSwitch]="orderStatus">
        <div class="status-card" *ngSwitchCase="'pending'">
          <span class="status-icon">⏳</span>
          <h4>รอการยืนยัน</h4>
          <p>คำสั่งซื้อของคุณกำลังรอการยืนยัน</p>
          <button (click)="cancelOrder()">ยกเลิกคำสั่งซื้อ</button>
        </div>
        
        <div class="status-card processing" *ngSwitchCase="'processing'">
          <span class="status-icon">🔄</span>
          <h4>กำลังดำเนินการ</h4>
          <p>คำสั่งซื้อกำลังถูกดำเนินการจัดส่ง</p>
        </div>
        
        <div class="status-card shipped" *ngSwitchCase="'shipped'">
          <span class="status-icon">🚚</span>
          <h4>กำลังจัดส่ง</h4>
          <p>พัสดุอยู่ระหว่างการจัดส่ง</p>
          <p>เลขพัสดุ: TH123456789</p>
          <a href="#">ติดตามพัสดุ</a>
        </div>
        
        <div class="status-card delivered" *ngSwitchCase="'delivered'">
          <span class="status-icon">✅</span>
          <h4>ส่งสำเร็จ</h4>
          <p>ได้รับสินค้าเรียบร้อยแล้ว</p>
          <button>รีวิวสินค้า</button>
        </div>
        
        <div class="status-card cancelled" *ngSwitchCase="'cancelled'">
          <span class="status-icon">❌</span>
          <h4>ยกเลิกแล้ว</h4>
          <p>คำสั่งซื้อถูกยกเลิก</p>
          <button>สั่งซื้อใหม่</button>
        </div>
      </div>
      
      <!-- Controls -->
      <div class="controls">
        <select [(ngModel)]="userRole">
          <option value="admin">Admin</option>
          <option value="editor">Editor</option>
          <option value="viewer">Viewer</option>
          <option value="guest">Guest</option>
        </select>
        
        <select [(ngModel)]="orderStatus">
          <option value="pending">Pending</option>
          <option value="processing">Processing</option>
          <option value="shipped">Shipped</option>
          <option value="delivered">Delivered</option>
          <option value="cancelled">Cancelled</option>
        </select>
      </div>
    </div>
  `
})
export class NgSwitchDemoComponent {
  currentDay = 'Monday';
  userRole: UserRole = 'admin';
  orderStatus: OrderStatus = 'pending';

  cancelOrder(): void {
    this.orderStatus = 'cancelled';
  }
}
```

---

## 5. New Syntax Angular 17+: @if {#new-if}

Angular 17 แนะนำ built-in control flow syntax ใหม่ที่ไม่ต้อง import directives

```typescript
// new-syntax.component.ts
import { Component } from '@angular/core';

@Component({
  selector: 'app-new-syntax',
  standalone: true,
  template: `
    <!-- @if แบบง่าย -->
    @if (isLoggedIn) {
      <p>ยินดีต้อนรับ!</p>
    }
    
    <!-- @if / @else -->
    @if (isLoggedIn) {
      <div class="welcome">
        <h2>ยินดีต้อนรับ, {{ username }}</h2>
        <button (click)="logout()">ออกจากระบบ</button>
      </div>
    } @else {
      <div class="login">
        <h2>กรุณาเข้าสู่ระบบ</h2>
        <button (click)="login()">เข้าสู่ระบบ</button>
      </div>
    }
    
    <!-- @if / @else if / @else -->
    @if (score >= 80) {
      <p class="grade-a">เกรด A - ยอดเยี่ยม!</p>
    } @else if (score >= 70) {
      <p class="grade-b">เกรด B - ดี</p>
    } @else if (score >= 60) {
      <p class="grade-c">เกรด C - พอใช้</p>
    } @else if (score >= 50) {
      <p class="grade-d">เกรด D - ต้องปรับปรุง</p>
    } @else {
      <p class="grade-f">เกรด F - สอบตก</p>
    }
    
    <!-- @if กับ expression และ as -->
    @if (user$ | async; as user) {
      <div>
        <h3>{{ user.name }}</h3>
        <p>{{ user.email }}</p>
      </div>
    } @else {
      <p>กำลังโหลดข้อมูลผู้ใช้...</p>
    }
    
    <!-- เปรียบเทียบกับ *ngIf เดิม -->
    <!-- เดิม: -->
    <!-- <div *ngIf="isLoggedIn; else loginTemplate">...</div>
         <ng-template #loginTemplate>...</ng-template> -->
    
    <!-- ใหม่: ชัดเจนและอ่านง่ายกว่า -->
    @if (isLoggedIn) {
      <div>logged in content</div>
    } @else {
      <div>login form</div>
    }
    
    <!-- @if กับ null checks -->
    @if (selectedItem !== null && selectedItem !== undefined) {
      <div>{{ selectedItem.name }}</div>
    }
    
    <!-- @if กับ permission check -->
    @if (hasRole('admin')) {
      <button>ลบข้อมูล</button>
    }
  `
})
export class NewSyntaxComponent {
  isLoggedIn = false;
  username = 'สมชาย';
  score = 85;
  selectedItem: { name: string } | null = { name: 'Test Item' };

  user$ = Promise.resolve({
    name: 'สมชาย ใจดี',
    email: 'somchai@example.com'
  });

  login(): void { this.isLoggedIn = true; }
  logout(): void { this.isLoggedIn = false; }

  hasRole(role: string): boolean {
    return ['admin'].includes(role);
  }
}
```

---

## 6. New Syntax Angular 17+: @for {#new-for}

```typescript
@Component({
  template: `
    <!-- @for พื้นฐาน - ต้องมี track เสมอ -->
    @for (item of items; track item.id) {
      <div>{{ item.name }}</div>
    }
    
    <!-- @for กับ @empty (แสดงเมื่อ array ว่าง) -->
    @for (product of products; track product.id) {
      <div class="product-card">
        <h3>{{ product.name }}</h3>
        <p>{{ product.price | number:'1.0-0' }} บาท</p>
      </div>
    } @empty {
      <div class="empty-state">
        <p>ไม่มีสินค้าในรายการ</p>
        <button>เพิ่มสินค้า</button>
      </div>
    }
    
    <!-- @for กับ built-in variables -->
    @for (item of items; track item.id; let i = $index; let first = $first; let last = $last; let even = $even; let odd = $odd; let count = $count) {
      <div
        [class.first-item]="first"
        [class.last-item]="last"
        [class.even-row]="even">
        
        {{ i + 1 }}/{{ count }}: {{ item.name }}
        
        @if (first) { <span>⭐ รายการแรก</span> }
        @if (last) { <span>🏁 รายการสุดท้าย</span> }
      </div>
    }
    
    <!-- @for ซ้อนกัน -->
    @for (category of categories; track category.id) {
      <div class="category">
        <h3>{{ category.name }}</h3>
        @for (product of category.products; track product.id) {
          <div class="product">{{ product.name }}</div>
        } @empty {
          <p>ไม่มีสินค้าในหมวดนี้</p>
        }
      </div>
    }
    
    <!-- @for กับ pipe -->
    @for (user of users | sortBy:'name'; track user.id) {
      <p>{{ user.name }}</p>
    }
    
    <!-- track ด้วย index (ถ้าไม่มี unique id) -->
    @for (item of simpleList; track $index) {
      <li>{{ item }}</li>
    }
    
    <!-- track ด้วย expression -->
    @for (item of items; track item.id + '-' + item.version) {
      <div>{{ item.name }} v{{ item.version }}</div>
    }
  `
})
export class NewForComponent {
  items = [
    { id: 1, name: 'Angular', version: '17' },
    { id: 2, name: 'React', version: '18' },
    { id: 3, name: 'Vue', version: '3' }
  ];

  products: any[] = [];

  categories = [
    {
      id: 1,
      name: 'โทรศัพท์',
      products: [
        { id: 101, name: 'iPhone 15' },
        { id: 102, name: 'Samsung S24' }
      ]
    },
    {
      id: 2,
      name: 'แล็ปท็อป',
      products: []
    }
  ];

  users = [
    { id: 1, name: 'Charlie' },
    { id: 2, name: 'Alice' },
    { id: 3, name: 'Bob' }
  ];

  simpleList = ['Angular', 'TypeScript', 'RxJS'];
}
```

---

## 7. New Syntax Angular 17+: @switch {#new-switch}

```typescript
@Component({
  template: `
    <!-- @switch พื้นฐาน -->
    @switch (currentTheme) {
      @case ('light') {
        <div class="theme-light">🌞 โหมดสว่าง</div>
      }
      @case ('dark') {
        <div class="theme-dark">🌙 โหมดมืด</div>
      }
      @case ('system') {
        <div class="theme-system">⚙️ ตามระบบ</div>
      }
      @default {
        <div>ธีมไม่รู้จัก</div>
      }
    }
    
    <!-- @switch กับ user role -->
    @switch (userRole) {
      @case ('super-admin') {
        <div class="role-panel">
          <h3>Super Admin Panel</h3>
          <p>คุณมีสิทธิ์สูงสุด</p>
          <button>จัดการระบบ</button>
          <button>ดูสถิติทั้งหมด</button>
          <button>ลบข้อมูล</button>
        </div>
      }
      @case ('admin') {
        <div class="role-panel">
          <h3>Admin Panel</h3>
          <button>จัดการผู้ใช้</button>
          <button>ดูรายงาน</button>
        </div>
      }
      @case ('editor') {
        <div class="role-panel">
          <h3>Editor Panel</h3>
          <button>เขียนบทความ</button>
          <button>จัดการสื่อ</button>
        </div>
      }
      @default {
        <div class="role-panel">
          <h3>Viewer</h3>
          <p>คุณสามารถอ่านเนื้อหาได้เท่านั้น</p>
        </div>
      }
    }
    
    <!-- @switch กับ สถานะ Loading/Error/Success -->
    @switch (loadingState) {
      @case ('loading') {
        <div class="state-loading">
          <div class="spinner"></div>
          <p>กำลังโหลด...</p>
        </div>
      }
      @case ('error') {
        <div class="state-error">
          <span>⚠️</span>
          <p>{{ errorMessage }}</p>
          <button (click)="retry()">ลองใหม่</button>
        </div>
      }
      @case ('success') {
        <div class="state-success">
          <span>✅</span>
          <p>โหลดข้อมูลสำเร็จ!</p>
        </div>
      }
      @default {
        <div class="state-idle">
          <button (click)="loadData()">โหลดข้อมูล</button>
        </div>
      }
    }
    
    <!-- @switch ใน @for -->
    @for (step of steps; track step.id) {
      <div class="step">
        @switch (step.type) {
          @case ('input') {
            <input [placeholder]="step.label">
          }
          @case ('select') {
            <select>
              @for (opt of step.options; track opt.value) {
                <option [value]="opt.value">{{ opt.label }}</option>
              }
            </select>
          }
          @case ('checkbox') {
            <label>
              <input type="checkbox"> {{ step.label }}
            </label>
          }
          @default {
            <p>{{ step.label }}</p>
          }
        }
      </div>
    }
    
    <!-- Controls -->
    <select [(ngModel)]="userRole">
      <option value="super-admin">Super Admin</option>
      <option value="admin">Admin</option>
      <option value="editor">Editor</option>
      <option value="viewer">Viewer</option>
    </select>
    
    <select [(ngModel)]="loadingState">
      <option value="idle">Idle</option>
      <option value="loading">Loading</option>
      <option value="success">Success</option>
      <option value="error">Error</option>
    </select>
  `
})
export class NewSwitchComponent {
  currentTheme = 'dark';
  userRole = 'admin';
  loadingState = 'idle';
  errorMessage = 'ไม่สามารถเชื่อมต่อกับเซิร์ฟเวอร์ได้';

  steps = [
    { id: 1, type: 'input', label: 'ชื่อ' },
    { id: 2, type: 'select', label: 'จังหวัด', options: [
      { value: 'bkk', label: 'กรุงเทพฯ' },
      { value: 'cnx', label: 'เชียงใหม่' }
    ]},
    { id: 3, type: 'checkbox', label: 'ยอมรับเงื่อนไข' },
    { id: 4, type: 'textarea', label: 'หมายเหตุ' }
  ];

  retry(): void {
    this.loadingState = 'loading';
    setTimeout(() => this.loadingState = 'success', 2000);
  }

  loadData(): void {
    this.loadingState = 'loading';
    setTimeout(() => this.loadingState = 'success', 1500);
  }
}
```

---

## 8. Attribute Directives: ngClass {#ngclass}

`ngClass` ช่วยให้เราจัดการ CSS classes ได้อย่างยืดหยุ่น

```typescript
// ngclass-demo.component.ts
import { Component } from '@angular/core';
import { NgClass, NgFor, NgIf } from '@angular/common';
import { FormsModule } from '@angular/forms';

type AlertType = 'info' | 'success' | 'warning' | 'danger';
type ButtonSize = 'sm' | 'md' | 'lg';
type ButtonVariant = 'primary' | 'secondary' | 'outline' | 'ghost';

@Component({
  selector: 'app-ngclass-demo',
  standalone: true,
  imports: [NgClass, NgFor, NgIf, FormsModule],
  template: `
    <div>
      <!-- ngClass กับ string -->
      <div [ngClass]="'alert alert-info'">Alert Info</div>
      
      <!-- ngClass กับ array -->
      <div [ngClass]="['alert', currentAlertClass]">Dynamic Alert</div>
      
      <!-- ngClass กับ object (แนะนำ) -->
      <div [ngClass]="{
        'is-loading': isLoading,
        'is-active': isActive,
        'has-error': hasError,
        'is-disabled': isDisabled
      }">Object-based ngClass</div>
      
      <!-- ngClass กับ method ที่ return object -->
      <button [ngClass]="getButtonClasses()" (click)="onBtnClick()">
        {{ isLoading ? 'กำลังดำเนินการ...' : 'คลิก' }}
      </button>
      
      <!-- Alert Component ด้วย ngClass -->
      <div
        class="alert"
        [ngClass]="{
          'alert-info': alertType === 'info',
          'alert-success': alertType === 'success',
          'alert-warning': alertType === 'warning',
          'alert-danger': alertType === 'danger'
        }"
        role="alert">
        <span class="alert-icon">
          <ng-container [ngSwitch]="alertType">
            <span *ngSwitchCase="'info'">ℹ️</span>
            <span *ngSwitchCase="'success'">✅</span>
            <span *ngSwitchCase="'warning'">⚠️</span>
            <span *ngSwitchCase="'danger'">❌</span>
          </ng-container>
        </span>
        {{ alertMessage }}
        <button class="close-btn" (click)="closeAlert()">×</button>
      </div>
      
      <!-- Button Builder - ตัวอย่างการใช้งานจริง -->
      <div class="button-builder">
        <h3>Button Builder</h3>
        
        <div class="controls">
          <select [(ngModel)]="buttonVariant">
            <option value="primary">Primary</option>
            <option value="secondary">Secondary</option>
            <option value="outline">Outline</option>
            <option value="ghost">Ghost</option>
          </select>
          
          <select [(ngModel)]="buttonSize">
            <option value="sm">Small</option>
            <option value="md">Medium</option>
            <option value="lg">Large</option>
          </select>
          
          <label>
            <input type="checkbox" [(ngModel)]="isLoading"> Loading
          </label>
          <label>
            <input type="checkbox" [(ngModel)]="isDisabled"> Disabled
          </label>
          <label>
            <input type="checkbox" [(ngModel)]="isFullWidth"> Full Width
          </label>
        </div>
        
        <!-- Preview -->
        <button
          [ngClass]="getDynamicButtonClasses()"
          [disabled]="isDisabled || isLoading">
          @if (isLoading) {
            <span class="spinner"></span>
            กำลังโหลด...
          } @else {
            คลิกฉัน
          }
        </button>
        
        <!-- Classes ที่ถูกใช้ -->
        <p class="classes-used">
          Classes: <code>{{ getCurrentClasses() }}</code>
        </p>
      </div>
      
      <!-- Table Row Highlighting -->
      <table>
        <tr
          *ngFor="let row of tableData; let i = index"
          [ngClass]="{
            'row-even': i % 2 === 0,
            'row-odd': i % 2 !== 0,
            'row-selected': selectedRowId === row.id,
            'row-error': row.hasError,
            'row-new': row.isNew
          }"
          (click)="selectRow(row.id)">
          <td>{{ row.name }}</td>
          <td>{{ row.value }}</td>
        </tr>
      </table>
    </div>
  `,
  styles: [`
    .alert {
      padding: 12px 16px;
      border-radius: 8px;
      margin: 8px 0;
      display: flex;
      align-items: center;
      gap: 8px;
    }
    .alert-info { background: #e3f2fd; border-left: 4px solid #2196f3; color: #0d47a1; }
    .alert-success { background: #e8f5e9; border-left: 4px solid #4caf50; color: #1b5e20; }
    .alert-warning { background: #fff3e0; border-left: 4px solid #ff9800; color: #e65100; }
    .alert-danger { background: #ffebee; border-left: 4px solid #f44336; color: #b71c1c; }
    .close-btn { margin-left: auto; background: none; border: none; cursor: pointer; font-size: 18px; }
    
    .btn { padding: 8px 16px; border: none; cursor: pointer; border-radius: 4px; }
    .btn-sm { padding: 4px 10px; font-size: 13px; }
    .btn-md { padding: 8px 16px; font-size: 14px; }
    .btn-lg { padding: 12px 24px; font-size: 16px; }
    .btn-primary { background: #007bff; color: white; }
    .btn-secondary { background: #6c757d; color: white; }
    .btn-outline { background: transparent; border: 2px solid #007bff; color: #007bff; }
    .btn-ghost { background: transparent; color: #007bff; }
    .btn-loading { opacity: 0.7; cursor: wait; }
    .btn-full-width { width: 100%; }
    
    .row-even { background: #f9f9f9; }
    .row-odd { background: white; }
    .row-selected { background: #e3f2fd !important; }
    .row-error { color: red; }
    .row-new { font-weight: bold; }
  `]
})
export class NgClassDemoComponent {
  isLoading = false;
  isActive = true;
  hasError = false;
  isDisabled = false;
  isFullWidth = false;
  alertType: AlertType = 'info';
  alertMessage = 'นี่คือข้อความแจ้งเตือน';
  buttonVariant: ButtonVariant = 'primary';
  buttonSize: ButtonSize = 'md';
  selectedRowId: number | null = null;

  get currentAlertClass(): string {
    return `alert-${this.alertType}`;
  }

  tableData = [
    { id: 1, name: 'สินค้า A', value: 100, hasError: false, isNew: true },
    { id: 2, name: 'สินค้า B', value: 200, hasError: true, isNew: false },
    { id: 3, name: 'สินค้า C', value: 300, hasError: false, isNew: false }
  ];

  getButtonClasses(): Record<string, boolean> {
    return {
      'btn': true,
      'btn-primary': true,
      'btn-loading': this.isLoading,
      'btn-disabled': this.isDisabled
    };
  }

  getDynamicButtonClasses(): Record<string, boolean> {
    return {
      'btn': true,
      [`btn-${this.buttonVariant}`]: true,
      [`btn-${this.buttonSize}`]: true,
      'btn-loading': this.isLoading,
      'btn-full-width': this.isFullWidth
    };
  }

  getCurrentClasses(): string {
    const classes = this.getDynamicButtonClasses();
    return Object.entries(classes)
      .filter(([_, active]) => active)
      .map(([cls]) => cls)
      .join(' ');
  }

  closeAlert(): void {
    this.alertMessage = '';
  }

  onBtnClick(): void {
    this.isLoading = true;
    setTimeout(() => this.isLoading = false, 2000);
  }

  selectRow(id: number): void {
    this.selectedRowId = this.selectedRowId === id ? null : id;
  }
}
```

---

## 9. Attribute Directives: ngStyle {#ngstyle}

```typescript
// ngstyle-demo.component.ts
import { Component } from '@angular/core';
import { NgStyle, NgFor } from '@angular/common';
import { FormsModule } from '@angular/forms';

@Component({
  selector: 'app-ngstyle-demo',
  standalone: true,
  imports: [NgStyle, NgFor, FormsModule],
  template: `
    <div>
      <!-- ngStyle พื้นฐาน -->
      <div [ngStyle]="{'color': textColor, 'font-size': fontSize + 'px'}">
        ข้อความที่ปรับได้
      </div>
      
      <!-- ngStyle กับ method -->
      <div [ngStyle]="getCardStyles()">
        Card Style
      </div>
      
      <!-- Color Picker Demo -->
      <div class="color-picker-demo">
        <h3>ตัวสร้างสีแบบ Dynamic</h3>
        
        <div class="sliders">
          <label>R: <input type="range" min="0" max="255" [(ngModel)]="red"> {{ red }}</label>
          <label>G: <input type="range" min="0" max="255" [(ngModel)]="green"> {{ green }}</label>
          <label>B: <input type="range" min="0" max="255" [(ngModel)]="blue"> {{ blue }}</label>
          <label>Alpha: <input type="range" min="0" max="100" [(ngModel)]="alpha"> {{ alpha }}%</label>
        </div>
        
        <!-- Preview Box -->
        <div
          class="color-preview"
          [ngStyle]="{
            'background-color': getRgbaColor(),
            'border': '3px solid ' + getDarkerColor(),
            'color': getContrastColor(),
            'padding': '20px',
            'border-radius': '8px',
            'text-align': 'center',
            'margin-top': '16px'
          }">
          <p>สี: {{ getRgbaColor() }}</p>
          <p>Contrast color: {{ getContrastColor() }}</p>
        </div>
      </div>
      
      <!-- Progress Bars -->
      <div class="progress-demo">
        <h3>Progress Bars</h3>
        <div *ngFor="let item of progressItems">
          <div class="progress-label">
            {{ item.label }} ({{ item.value }}%)
          </div>
          <div class="progress-track">
            <div
              class="progress-fill"
              [ngStyle]="{
                'width': item.value + '%',
                'background-color': getProgressColor(item.value),
                'transition': 'width 0.5s ease'
              }">
            </div>
          </div>
        </div>
      </div>
      
      <!-- CSS Variable Override -->
      <div
        [ngStyle]="getCssVarStyles()"
        class="themed-component">
        Themed Component
      </div>
    </div>
  `,
  styles: [`
    .progress-track {
      height: 12px;
      background: #e0e0e0;
      border-radius: 6px;
      overflow: hidden;
      margin-bottom: 8px;
    }
    .progress-fill { height: 100%; border-radius: 6px; }
    .themed-component {
      --primary: #007bff;
      --bg: #f5f5f5;
      background: var(--bg);
      color: var(--primary);
      padding: 16px;
      border-radius: 8px;
    }
  `]
})
export class NgStyleDemoComponent {
  textColor = '#333';
  fontSize = 16;
  red = 100;
  green = 150;
  blue = 200;
  alpha = 100;

  progressItems = [
    { label: 'HTML', value: 90 },
    { label: 'CSS', value: 75 },
    { label: 'TypeScript', value: 85 },
    { label: 'Angular', value: 70 }
  ];

  getRgbaColor(): string {
    return `rgba(${this.red}, ${this.green}, ${this.blue}, ${this.alpha / 100})`;
  }

  getDarkerColor(): string {
    return `rgb(${Math.max(0, this.red - 40)}, ${Math.max(0, this.green - 40)}, ${Math.max(0, this.blue - 40)})`;
  }

  getContrastColor(): string {
    const luminance = (0.299 * this.red + 0.587 * this.green + 0.114 * this.blue) / 255;
    return luminance > 0.5 ? '#000' : '#fff';
  }

  getProgressColor(value: number): string {
    if (value < 30) return '#f44336';
    if (value < 60) return '#ff9800';
    if (value < 80) return '#2196f3';
    return '#4caf50';
  }

  getCardStyles(): Record<string, string> {
    return {
      'padding': '16px',
      'background': '#fff',
      'border-radius': '8px',
      'box-shadow': '0 2px 8px rgba(0,0,0,0.1)',
      'border-left': `4px solid ${this.textColor}`
    };
  }

  getCssVarStyles(): Record<string, string> {
    return {
      '--primary': this.getRgbaColor(),
      '--bg': this.getDarkerColor()
    };
  }
}
```

---

## 10. ngModel และ ngForm {#ngmodel-form}

```typescript
// ngmodel-form.component.ts
import { Component } from '@angular/core';
import { FormsModule, NgForm } from '@angular/forms';
import { NgIf, NgFor, NgClass } from '@angular/common';

interface ContactForm {
  name: string;
  email: string;
  phone: string;
  subject: string;
  message: string;
  priority: string;
  newsletter: boolean;
}

@Component({
  selector: 'app-contact-form',
  standalone: true,
  imports: [FormsModule, NgIf, NgFor, NgClass],
  template: `
    <div class="contact-form-wrapper">
      <h2>แบบฟอร์มติดต่อ</h2>
      
      <form
        #contactForm="ngForm"
        (ngSubmit)="onSubmit(contactForm)"
        class="contact-form"
        novalidate>
        
        <!-- ชื่อ -->
        <div class="form-group" [ngClass]="getFieldClass(name)">
          <label for="name">ชื่อ-นามสกุล *</label>
          <input
            id="name"
            type="text"
            name="name"
            #name="ngModel"
            [(ngModel)]="formData.name"
            required
            minlength="2"
            maxlength="100"
            placeholder="กรอกชื่อ-นามสกุล">
          
          @if (name.invalid && (name.dirty || name.touched)) {
            <div class="error-messages">
              @if (name.errors?.['required']) { <span>กรุณากรอกชื่อ</span> }
              @if (name.errors?.['minlength']) { <span>ชื่อต้องมีอย่างน้อย 2 ตัวอักษร</span> }
            </div>
          }
        </div>
        
        <!-- อีเมล -->
        <div class="form-group" [ngClass]="getFieldClass(email)">
          <label for="email">อีเมล *</label>
          <input
            id="email"
            type="email"
            name="email"
            #email="ngModel"
            [(ngModel)]="formData.email"
            required
            email
            placeholder="example@email.com">
          
          @if (email.invalid && (email.dirty || email.touched)) {
            <div class="error-messages">
              @if (email.errors?.['required']) { <span>กรุณากรอกอีเมล</span> }
              @if (email.errors?.['email']) { <span>รูปแบบอีเมลไม่ถูกต้อง</span> }
            </div>
          }
        </div>
        
        <!-- เบอร์โทร -->
        <div class="form-group">
          <label for="phone">เบอร์โทรศัพท์</label>
          <input
            id="phone"
            type="tel"
            name="phone"
            #phone="ngModel"
            [(ngModel)]="formData.phone"
            pattern="^[0-9]{10}$"
            placeholder="0812345678">
          
          @if (phone.invalid && (phone.dirty || phone.touched)) {
            <div class="error-messages">
              @if (phone.errors?.['pattern']) { <span>เบอร์โทรต้องเป็นตัวเลข 10 หลัก</span> }
            </div>
          }
        </div>
        
        <!-- หัวข้อ -->
        <div class="form-group">
          <label for="subject">หัวข้อ *</label>
          <select
            id="subject"
            name="subject"
            #subject="ngModel"
            [(ngModel)]="formData.subject"
            required>
            <option value="">-- เลือกหัวข้อ --</option>
            <option *ngFor="let opt of subjectOptions" [value]="opt.value">
              {{ opt.label }}
            </option>
          </select>
        </div>
        
        <!-- Priority -->
        <div class="form-group">
          <label>ความสำคัญ</label>
          <div class="radio-group">
            <label *ngFor="let opt of priorityOptions">
              <input
                type="radio"
                name="priority"
                [(ngModel)]="formData.priority"
                [value]="opt.value">
              {{ opt.label }}
            </label>
          </div>
        </div>
        
        <!-- ข้อความ -->
        <div class="form-group" [ngClass]="getFieldClass(message)">
          <label for="message">ข้อความ *</label>
          <textarea
            id="message"
            name="message"
            #message="ngModel"
            [(ngModel)]="formData.message"
            required
            minlength="10"
            maxlength="1000"
            rows="5"
            placeholder="กรอกข้อความที่ต้องการติดต่อ">
          </textarea>
          <div class="char-count">
            {{ formData.message.length }}/1000 ตัวอักษร
          </div>
          
          @if (message.invalid && (message.dirty || message.touched)) {
            <div class="error-messages">
              @if (message.errors?.['required']) { <span>กรุณากรอกข้อความ</span> }
              @if (message.errors?.['minlength']) { <span>ข้อความต้องมีอย่างน้อย 10 ตัวอักษร</span> }
            </div>
          }
        </div>
        
        <!-- Newsletter -->
        <div class="form-group">
          <label class="checkbox-label">
            <input
              type="checkbox"
              name="newsletter"
              [(ngModel)]="formData.newsletter">
            สมัครรับข่าวสาร newsletter
          </label>
        </div>
        
        <!-- Form Status -->
        <div class="form-status">
          <p>สถานะ: 
            <span [ngClass]="contactForm.valid ? 'status-valid' : 'status-invalid'">
              {{ contactForm.valid ? '✅ พร้อมส่ง' : '❌ ยังไม่ครบถ้วน' }}
            </span>
          </p>
        </div>
        
        <!-- Buttons -->
        <div class="form-actions">
          <button
            type="submit"
            class="btn-submit"
            [disabled]="contactForm.invalid || isSubmitting">
            {{ isSubmitting ? '⏳ กำลังส่ง...' : '📤 ส่งข้อความ' }}
          </button>
          <button type="button" class="btn-reset" (click)="resetForm(contactForm)">
            🔄 รีเซ็ต
          </button>
        </div>
      </form>
      
      <!-- Success Message -->
      @if (isSubmitted) {
        <div class="success-message">
          <h3>✅ ส่งข้อความสำเร็จ!</h3>
          <p>ขอบคุณที่ติดต่อเรา เราจะตอบกลับภายใน 24 ชั่วโมง</p>
          <button (click)="isSubmitted = false">ส่งข้อความอีกครั้ง</button>
        </div>
      }
    </div>
  `,
  styles: [`
    .contact-form-wrapper { max-width: 600px; margin: 0 auto; padding: 24px; }
    .form-group { margin-bottom: 20px; }
    .form-group label { display: block; margin-bottom: 6px; font-weight: 500; color: #374151; }
    .form-group input, .form-group select, .form-group textarea {
      width: 100%; padding: 10px 14px;
      border: 1px solid #d1d5db; border-radius: 6px;
      font-size: 14px; outline: none; transition: border-color 0.2s;
    }
    .form-group input:focus, .form-group select:focus, .form-group textarea:focus {
      border-color: #3b82f6; box-shadow: 0 0 0 3px rgba(59,130,246,0.1);
    }
    .form-group.ng-invalid.ng-touched input,
    .form-group.ng-invalid.ng-touched textarea { border-color: #ef4444; }
    .form-group.ng-valid.ng-touched input,
    .form-group.ng-valid.ng-touched textarea { border-color: #10b981; }
    .error-messages { margin-top: 4px; }
    .error-messages span { color: #ef4444; font-size: 12px; display: block; }
    .char-count { text-align: right; font-size: 12px; color: #6b7280; margin-top: 4px; }
    .radio-group { display: flex; gap: 16px; }
    .checkbox-label { display: flex; align-items: center; gap: 8px; cursor: pointer; }
    .form-status { padding: 12px; background: #f9fafb; border-radius: 8px; margin-bottom: 16px; }
    .status-valid { color: #059669; font-weight: bold; }
    .status-invalid { color: #dc2626; font-weight: bold; }
    .form-actions { display: flex; gap: 12px; }
    .btn-submit {
      flex: 1; padding: 12px; background: #3b82f6; color: white;
      border: none; border-radius: 8px; cursor: pointer; font-size: 15px;
    }
    .btn-submit:disabled { background: #93c5fd; cursor: not-allowed; }
    .btn-reset {
      padding: 12px 20px; background: white; color: #6b7280;
      border: 1px solid #d1d5db; border-radius: 8px; cursor: pointer;
    }
    .success-message {
      text-align: center; padding: 32px; background: #f0fdf4;
      border-radius: 12px; border: 1px solid #86efac;
    }
    .success-message h3 { color: #166534; }
  `]
})
export class ContactFormComponent {
  isSubmitting = false;
  isSubmitted = false;

  formData: ContactForm = {
    name: '',
    email: '',
    phone: '',
    subject: '',
    message: '',
    priority: 'normal',
    newsletter: false
  };

  subjectOptions = [
    { value: 'general', label: 'คำถามทั่วไป' },
    { value: 'technical', label: 'ปัญหาทางเทคนิค' },
    { value: 'billing', label: 'การชำระเงิน' },
    { value: 'feedback', label: 'ข้อเสนอแนะ' },
    { value: 'other', label: 'อื่นๆ' }
  ];

  priorityOptions = [
    { value: 'low', label: 'ต่ำ' },
    { value: 'normal', label: 'ปกติ' },
    { value: 'high', label: 'สูง' },
    { value: 'urgent', label: 'เร่งด่วน' }
  ];

  getFieldClass(field: any): Record<string, boolean> {
    return {
      'ng-valid': field.valid && (field.dirty || field.touched),
      'ng-invalid': field.invalid && (field.dirty || field.touched)
    };
  }

  async onSubmit(form: NgForm): Promise<void> {
    if (form.invalid) {
      Object.keys(form.controls).forEach(key => {
        form.controls[key].markAsTouched();
      });
      return;
    }

    this.isSubmitting = true;
    try {
      await new Promise(resolve => setTimeout(resolve, 2000));
      console.log('Form submitted:', this.formData);
      this.isSubmitted = true;
      form.resetForm();
    } finally {
      this.isSubmitting = false;
    }
  }

  resetForm(form: NgForm): void {
    form.resetForm();
    this.formData = {
      name: '', email: '', phone: '', subject: '',
      message: '', priority: 'normal', newsletter: false
    };
  }
}
```

---

## 11. Built-in Pipes ที่ใช้กับ Directives {#pipes-with-directives}

```typescript
// pipes-with-directives.component.ts
import { Component } from '@angular/core';
import { NgFor, NgIf, AsyncPipe, DatePipe, CurrencyPipe, DecimalPipe, UpperCasePipe } from '@angular/common';
import { Observable, of, interval } from 'rxjs';
import { map, take } from 'rxjs/operators';

@Component({
  selector: 'app-pipes-directives',
  standalone: true,
  imports: [NgFor, NgIf, AsyncPipe, DatePipe, CurrencyPipe, DecimalPipe, UpperCasePipe],
  template: `
    <div>
      <!-- async pipe กับ *ngFor -->
      <ul>
        <li *ngFor="let item of products$ | async">
          {{ item.name }} - {{ item.price | currency:'THB':'symbol':'1.0-0' }}
        </li>
      </ul>
      
      <!-- async กับ @for -->
      @for (item of (products$ | async) ?? []; track item.id) {
        <div>{{ item.name }}</div>
      }
      
      <!-- keyvalue pipe กับ *ngFor -->
      <div *ngFor="let entry of userProfile | keyvalue">
        <strong>{{ entry.key | titlecase }}:</strong>
        {{ entry.value }}
      </div>
      
      <!-- slice pipe กับ *ngFor -->
      <p>แสดง 3 รายการแรก:</p>
      <ul>
        <li *ngFor="let item of items | slice:0:3">{{ item }}</li>
      </ul>
      
      <!-- filter ด้วย pipe custom -->
      <ul>
        <li *ngFor="let product of products | filterBy:'category':'phone'">
          {{ product.name }}
        </li>
      </ul>
      
      <!-- async pipe กับ loading state -->
      <ng-container *ngIf="data$ | async as data; else loading">
        <div>{{ data.title }}</div>
      </ng-container>
      <ng-template #loading>กำลังโหลด...</ng-template>
      
      <!-- Timer ด้วย async pipe -->
      <p>เวลา: {{ timer$ | async }} วินาที</p>
    </div>
  `
})
export class PipesDirectivesComponent {
  products$ = of([
    { id: 1, name: 'iPhone 15', price: 35000, category: 'phone' },
    { id: 2, name: 'MacBook Pro', price: 75000, category: 'laptop' },
    { id: 3, name: 'Samsung S24', price: 28000, category: 'phone' }
  ]);

  data$ = of({ title: 'ข้อมูลจาก API', content: 'รายละเอียด' });

  timer$ = interval(1000).pipe(take(60));

  userProfile = {
    firstName: 'สมชาย',
    lastName: 'ใจดี',
    email: 'somchai@example.com',
    role: 'admin'
  };

  items = ['Angular', 'React', 'Vue', 'Svelte', 'Next.js', 'Nuxt.js'];

  products = [
    { name: 'iPhone 15', category: 'phone' },
    { name: 'MacBook Pro', category: 'laptop' },
    { name: 'Samsung S24', category: 'phone' }
  ];
}
```

---

## 12. Custom Structural Directive พื้นฐาน {#custom-directive}

```typescript
// unless.directive.ts - ตรงข้ามกับ *ngIf
import {
  Directive,
  Input,
  TemplateRef,
  ViewContainerRef,
  OnChanges,
  SimpleChanges
} from '@angular/core';

@Directive({
  selector: '[appUnless]',
  standalone: true
})
export class UnlessDirective implements OnChanges {
  @Input() appUnless = false;

  constructor(
    private templateRef: TemplateRef<any>,
    private vcr: ViewContainerRef
  ) {}

  ngOnChanges(changes: SimpleChanges): void {
    if (changes['appUnless']) {
      if (this.appUnless) {
        // เงื่อนไขเป็น true: ซ่อน (ตรงข้ามกับ ngIf)
        this.vcr.clear();
      } else {
        // เงื่อนไขเป็น false: แสดง
        this.vcr.createEmbeddedView(this.templateRef);
      }
    }
  }
}

// การใช้งาน: <div *appUnless="isLoggedIn">กรุณาเข้าสู่ระบบ</div>
```

```typescript
// repeat.directive.ts - วน loop ตามจำนวนที่กำหนด
import { Directive, Input, TemplateRef, ViewContainerRef, OnInit } from '@angular/core';

@Directive({
  selector: '[appRepeat]',
  standalone: true
})
export class RepeatDirective implements OnInit {
  @Input() appRepeat = 1;

  constructor(
    private templateRef: TemplateRef<{ $implicit: number; index: number }>,
    private vcr: ViewContainerRef
  ) {}

  ngOnInit(): void {
    this.vcr.clear();
    for (let i = 0; i < this.appRepeat; i++) {
      this.vcr.createEmbeddedView(this.templateRef, {
        $implicit: i,
        index: i
      });
    }
  }
}

// การใช้งาน:
// <div *appRepeat="5; let i">ดาว {{ i + 1 }}</div>
// หรือ <div *appRepeat="5; let i; index as idx">{{ i }}</div>
```

```typescript
// lazy.directive.ts - Lazy render content
import {
  Directive,
  Input,
  TemplateRef,
  ViewContainerRef,
  OnInit,
  OnDestroy
} from '@angular/core';

@Directive({
  selector: '[appLazy]',
  standalone: true
})
export class LazyDirective implements OnInit, OnDestroy {
  @Input() appLazyDelay = 0; // milliseconds

  private timeoutId?: ReturnType<typeof setTimeout>;

  constructor(
    private templateRef: TemplateRef<any>,
    private vcr: ViewContainerRef
  ) {}

  ngOnInit(): void {
    this.timeoutId = setTimeout(() => {
      this.vcr.createEmbeddedView(this.templateRef);
    }, this.appLazyDelay);
  }

  ngOnDestroy(): void {
    if (this.timeoutId) {
      clearTimeout(this.timeoutId);
    }
  }
}

// การใช้งาน:
// <div *appLazy [appLazyDelay]="500">
//   เนื้อหาที่จะ render หลัง 500ms
// </div>
```

```typescript
// permission.directive.ts - แสดง/ซ่อนตาม permission
import { Directive, Input, TemplateRef, ViewContainerRef, OnInit, Inject } from '@angular/core';

export interface AuthService {
  hasPermission(permission: string): boolean;
  hasAnyPermission(permissions: string[]): boolean;
}

export const AUTH_SERVICE = 'AuthService';

@Directive({
  selector: '[appPermission]',
  standalone: true
})
export class PermissionDirective implements OnInit {
  @Input() appPermission = '';
  @Input() appPermissionElse?: TemplateRef<any>;

  constructor(
    private templateRef: TemplateRef<any>,
    private vcr: ViewContainerRef
  ) {}

  ngOnInit(): void {
    // จำลองการ check permission
    const hasPermission = this.checkPermission(this.appPermission);

    if (hasPermission) {
      this.vcr.createEmbeddedView(this.templateRef);
    } else if (this.appPermissionElse) {
      this.vcr.createEmbeddedView(this.appPermissionElse);
    }
  }

  private checkPermission(permission: string): boolean {
    // ในงานจริงควร inject AuthService มาใช้
    const userPermissions = ['read', 'write', 'delete'];
    return userPermissions.includes(permission);
  }
}

// การใช้งาน:
// <button *appPermission="'delete'; else noPermTpl">ลบ</button>
// <ng-template #noPermTpl><span>ไม่มีสิทธิ์</span></ng-template>
```

---

## 13. Workshop: Product Filter UI {#workshop}

Workshop สร้าง Product Filter UI ที่ครบสมบูรณ์

### โครงสร้างโปรเจค

```
src/app/product-filter/
├── product-filter.component.ts
├── product-filter.component.html
├── product-filter.component.css
├── models/
│   └── product.model.ts
├── components/
│   ├── filter-panel/
│   │   └── filter-panel.component.ts
│   └── product-grid/
│       └── product-grid.component.ts
└── pipes/
    └── filter-products.pipe.ts
```

### Product Model

```typescript
// product.model.ts
export interface Product {
  id: number;
  name: string;
  description: string;
  price: number;
  originalPrice?: number;
  category: string;
  brand: string;
  rating: number;
  reviewCount: number;
  inStock: boolean;
  isNew: boolean;
  isSale: boolean;
  tags: string[];
  imageUrl: string;
  colors: string[];
}

export interface FilterOptions {
  search: string;
  categories: string[];
  brands: string[];
  minPrice: number;
  maxPrice: number;
  minRating: number;
  inStock: boolean;
  isNew: boolean;
  isSale: boolean;
  sortBy: 'name' | 'price-asc' | 'price-desc' | 'rating' | 'newest';
}
```

### Filter Products Pipe

```typescript
// filter-products.pipe.ts
import { Pipe, PipeTransform } from '@angular/core';
import { Product, FilterOptions } from '../models/product.model';

@Pipe({
  name: 'filterProducts',
  standalone: true,
  pure: false // Impure เพราะต้อง re-evaluate เมื่อ object เปลี่ยน
})
export class FilterProductsPipe implements PipeTransform {
  transform(products: Product[], filters: FilterOptions): Product[] {
    if (!products) return [];

    let filtered = [...products];

    // Filter by search
    if (filters.search) {
      const search = filters.search.toLowerCase();
      filtered = filtered.filter(p =>
        p.name.toLowerCase().includes(search) ||
        p.description.toLowerCase().includes(search) ||
        p.brand.toLowerCase().includes(search) ||
        p.tags.some(t => t.toLowerCase().includes(search))
      );
    }

    // Filter by categories
    if (filters.categories.length > 0) {
      filtered = filtered.filter(p => filters.categories.includes(p.category));
    }

    // Filter by brands
    if (filters.brands.length > 0) {
      filtered = filtered.filter(p => filters.brands.includes(p.brand));
    }

    // Filter by price range
    filtered = filtered.filter(p =>
      p.price >= filters.minPrice && p.price <= filters.maxPrice
    );

    // Filter by rating
    if (filters.minRating > 0) {
      filtered = filtered.filter(p => p.rating >= filters.minRating);
    }

    // Filter by stock
    if (filters.inStock) {
      filtered = filtered.filter(p => p.inStock);
    }

    // Filter by new
    if (filters.isNew) {
      filtered = filtered.filter(p => p.isNew);
    }

    // Filter by sale
    if (filters.isSale) {
      filtered = filtered.filter(p => p.isSale);
    }

    // Sort
    switch (filters.sortBy) {
      case 'price-asc':
        filtered.sort((a, b) => a.price - b.price);
        break;
      case 'price-desc':
        filtered.sort((a, b) => b.price - a.price);
        break;
      case 'rating':
        filtered.sort((a, b) => b.rating - a.rating);
        break;
      case 'newest':
        filtered.sort((a, b) => (b.isNew ? 1 : 0) - (a.isNew ? 1 : 0));
        break;
      default:
        filtered.sort((a, b) => a.name.localeCompare(b.name));
    }

    return filtered;
  }
}
```

### Product Filter Component หลัก

```typescript
// product-filter.component.ts
import { Component, OnInit } from '@angular/core';
import { NgFor, NgIf, NgClass, DecimalPipe, CurrencyPipe } from '@angular/common';
import { FormsModule } from '@angular/forms';
import { Product, FilterOptions } from './models/product.model';
import { FilterProductsPipe } from './pipes/filter-products.pipe';

@Component({
  selector: 'app-product-filter',
  standalone: true,
  imports: [NgFor, NgIf, NgClass, FormsModule, DecimalPipe, CurrencyPipe, FilterProductsPipe],
  template: `
    <div class="product-filter-page">
      <!-- Page Header -->
      <div class="page-header">
        <h1>สินค้าทั้งหมด</h1>
        <p>{{ filteredProducts.length }} รายการ</p>
      </div>
      
      <div class="layout">
        <!-- Filter Panel (Sidebar) -->
        <aside class="filter-sidebar" [class.mobile-open]="isMobileFilterOpen">
          <div class="filter-header">
            <h3>กรองสินค้า</h3>
            <button class="btn-clear-all" (click)="clearAllFilters()">ล้างทั้งหมด</button>
          </div>
          
          <!-- Search -->
          <div class="filter-section">
            <h4>ค้นหา</h4>
            <div class="search-input-wrapper">
              <span class="search-icon">🔍</span>
              <input
                type="search"
                [(ngModel)]="filters.search"
                placeholder="ค้นหาสินค้า..."
                class="search-input">
            </div>
          </div>
          
          <!-- Categories -->
          <div class="filter-section">
            <h4>หมวดหมู่</h4>
            <div class="checkbox-list">
              <label *ngFor="let cat of availableCategories" class="checkbox-item">
                <input
                  type="checkbox"
                  [value]="cat.id"
                  [checked]="filters.categories.includes(cat.id)"
                  (change)="toggleFilter('categories', cat.id, $event)">
                <span>{{ cat.name }}</span>
                <span class="count">({{ getCategoryCount(cat.id) }})</span>
              </label>
            </div>
          </div>
          
          <!-- Brands -->
          <div class="filter-section">
            <h4>แบรนด์</h4>
            <div class="checkbox-list">
              <label *ngFor="let brand of availableBrands" class="checkbox-item">
                <input
                  type="checkbox"
                  [value]="brand"
                  [checked]="filters.brands.includes(brand)"
                  (change)="toggleFilter('brands', brand, $event)">
                <span>{{ brand }}</span>
              </label>
            </div>
          </div>
          
          <!-- Price Range -->
          <div class="filter-section">
            <h4>ช่วงราคา</h4>
            <div class="price-range">
              <div class="price-inputs">
                <input
                  type="number"
                  [(ngModel)]="filters.minPrice"
                  min="0"
                  placeholder="ต่ำสุด">
                <span>-</span>
                <input
                  type="number"
                  [(ngModel)]="filters.maxPrice"
                  placeholder="สูงสุด">
              </div>
              <div class="price-presets">
                <button
                  *ngFor="let preset of pricePresets"
                  [ngClass]="{'active': filters.minPrice === preset.min && filters.maxPrice === preset.max}"
                  (click)="setPriceRange(preset.min, preset.max)">
                  {{ preset.label }}
                </button>
              </div>
            </div>
          </div>
          
          <!-- Rating -->
          <div class="filter-section">
            <h4>คะแนนขั้นต่ำ</h4>
            <div class="rating-filter">
              <button
                *ngFor="let star of [1,2,3,4,5]"
                [ngClass]="{'active': filters.minRating === star}"
                (click)="setMinRating(star)">
                <span *ngFor="let s of [1,2,3,4,5]" [class.filled]="s <= star">★</span>
                <span *ngIf="star < 5"> ขึ้นไป</span>
                <span *ngIf="star === 5"> เท่านั้น</span>
              </button>
            </div>
          </div>
          
          <!-- Checkboxes Filters -->
          <div class="filter-section">
            <h4>ตัวกรองเพิ่มเติม</h4>
            <label class="checkbox-item">
              <input type="checkbox" [(ngModel)]="filters.inStock">
              <span>มีสินค้าในสต็อก</span>
            </label>
            <label class="checkbox-item">
              <input type="checkbox" [(ngModel)]="filters.isNew">
              <span>สินค้าใหม่</span>
            </label>
            <label class="checkbox-item">
              <input type="checkbox" [(ngModel)]="filters.isSale">
              <span>สินค้าลดราคา</span>
            </label>
          </div>
        </aside>
        
        <!-- Product Grid -->
        <main class="products-main">
          <!-- Sort & View Options -->
          <div class="toolbar">
            <div class="sort-section">
              <label>เรียงตาม:</label>
              <select [(ngModel)]="filters.sortBy">
                <option value="name">ชื่อ A-Z</option>
                <option value="price-asc">ราคา: ต่ำ-สูง</option>
                <option value="price-desc">ราคา: สูง-ต่ำ</option>
                <option value="rating">คะแนนสูงสุด</option>
                <option value="newest">สินค้าใหม่</option>
              </select>
            </div>
            
            <div class="view-toggle">
              <button
                [ngClass]="{'active': viewMode === 'grid'}"
                (click)="viewMode = 'grid'"
                title="Grid View">⊞</button>
              <button
                [ngClass]="{'active': viewMode === 'list'}"
                (click)="viewMode = 'list'"
                title="List View">☰</button>
            </div>
          </div>
          
          <!-- Active Filters Tags -->
          <div class="active-filters" *ngIf="hasActiveFilters">
            <span>กรองด้วย:</span>
            <span
              *ngFor="let tag of getActiveFilterTags()"
              class="filter-tag">
              {{ tag.label }}
              <button (click)="removeFilter(tag)">×</button>
            </span>
            <button class="clear-all" (click)="clearAllFilters()">ล้างทั้งหมด</button>
          </div>
          
          <!-- Products Grid -->
          <div
            [ngClass]="{
              'products-grid': viewMode === 'grid',
              'products-list': viewMode === 'list'
            }">
            
            <ng-container *ngFor="let product of filteredProducts; trackBy: trackByProductId">
              <!-- Grid Card -->
              <div
                *ngIf="viewMode === 'grid'"
                class="product-card"
                [class.out-of-stock]="!product.inStock"
                [class.on-sale]="product.isSale">
                
                <!-- Badges -->
                <div class="badges">
                  <span class="badge badge-new" *ngIf="product.isNew">ใหม่</span>
                  <span class="badge badge-sale" *ngIf="product.isSale">
                    -{{ getDiscountPercent(product) }}%
                  </span>
                </div>
                
                <!-- Image -->
                <div class="product-image" [style.background]="getImagePlaceholderColor(product.category)">
                  <span class="category-icon">{{ getCategoryIcon(product.category) }}</span>
                </div>
                
                <!-- Info -->
                <div class="product-info">
                  <p class="brand">{{ product.brand }}</p>
                  <h3 class="name">{{ product.name }}</h3>
                  
                  <!-- Rating -->
                  <div class="rating">
                    <span *ngFor="let s of [1,2,3,4,5]"
                      [class.filled]="s <= product.rating">★</span>
                    <span class="review-count">({{ product.reviewCount }})</span>
                  </div>
                  
                  <!-- Price -->
                  <div class="price">
                    <span class="current-price">
                      {{ product.price | currency:'THB':'symbol':'1.0-0' }}
                    </span>
                    <span
                      *ngIf="product.originalPrice"
                      class="original-price">
                      {{ product.originalPrice | currency:'THB':'symbol':'1.0-0' }}
                    </span>
                  </div>
                  
                  <!-- Stock Status -->
                  <p
                    class="stock-status"
                    [ngClass]="{
                      'in-stock': product.inStock,
                      'out-of-stock': !product.inStock
                    }">
                    {{ product.inStock ? '✓ มีสินค้า' : '✗ สินค้าหมด' }}
                  </p>
                  
                  <!-- Colors -->
                  <div class="colors" *ngIf="product.colors.length">
                    <span
                      *ngFor="let color of product.colors"
                      class="color-dot"
                      [style.background]="color"
                      [title]="color">
                    </span>
                  </div>
                  
                  <!-- Tags -->
                  <div class="tags">
                    <span
                      *ngFor="let tag of product.tags | slice:0:3"
                      class="tag">
                      {{ tag }}
                    </span>
                  </div>
                </div>
                
                <!-- Actions -->
                <div class="product-actions">
                  <button
                    class="btn-add-cart"
                    [disabled]="!product.inStock"
                    (click)="addToCart(product)">
                    {{ product.inStock ? '🛒 เพิ่มในตะกร้า' : 'สินค้าหมด' }}
                  </button>
                  <button class="btn-wishlist" (click)="toggleWishlist(product)">
                    {{ wishlist.has(product.id) ? '❤️' : '🤍' }}
                  </button>
                </div>
              </div>
              
              <!-- List Row -->
              <div
                *ngIf="viewMode === 'list'"
                class="product-list-item"
                [class.out-of-stock]="!product.inStock">
                
                <div class="list-image" [style.background]="getImagePlaceholderColor(product.category)">
                  <span>{{ getCategoryIcon(product.category) }}</span>
                </div>
                
                <div class="list-info">
                  <p class="brand">{{ product.brand }}</p>
                  <h3>{{ product.name }}</h3>
                  <p class="description">{{ product.description }}</p>
                  <div class="rating">
                    <span *ngFor="let s of [1,2,3,4,5]" [class.filled]="s <= product.rating">★</span>
                    ({{ product.reviewCount }})
                  </div>
                </div>
                
                <div class="list-price">
                  <p class="current-price">{{ product.price | currency:'THB':'symbol':'1.0-0' }}</p>
                  <p class="original-price" *ngIf="product.originalPrice">
                    {{ product.originalPrice | currency:'THB':'symbol':'1.0-0' }}
                  </p>
                  <p [class.in-stock]="product.inStock" [class.out-of-stock]="!product.inStock">
                    {{ product.inStock ? 'มีสินค้า' : 'หมด' }}
                  </p>
                  <button
                    class="btn-add-cart"
                    [disabled]="!product.inStock"
                    (click)="addToCart(product)">
                    {{ product.inStock ? 'เพิ่มในตะกร้า' : 'สินค้าหมด' }}
                  </button>
                </div>
              </div>
            </ng-container>
          </div>
          
          <!-- Empty State -->
          <div *ngIf="filteredProducts.length === 0" class="empty-state">
            <p>🔍 ไม่พบสินค้าที่ตรงกับเงื่อนไข</p>
            <p>ลองปรับเงื่อนไขการค้นหา</p>
            <button (click)="clearAllFilters()">ล้างตัวกรองทั้งหมด</button>
          </div>
          
          <!-- Load More -->
          <div class="pagination-area" *ngIf="filteredProducts.length > 0">
            <p>แสดง {{ filteredProducts.length }} จาก {{ allProducts.length }} รายการ</p>
          </div>
        </main>
      </div>
      
      <!-- Mobile Filter Toggle -->
      <button class="mobile-filter-btn" (click)="isMobileFilterOpen = !isMobileFilterOpen">
        🔧 ตัวกรอง
        <span class="active-count" *ngIf="activeFilterCount > 0">{{ activeFilterCount }}</span>
      </button>
    </div>
  `,
  styles: [`
    .product-filter-page { min-height: 100vh; background: #f8fafc; }
    .page-header { padding: 24px; background: white; border-bottom: 1px solid #e2e8f0; }
    .page-header h1 { margin: 0; font-size: 24px; }
    .layout { display: flex; gap: 24px; padding: 24px; }
    
    .filter-sidebar {
      width: 260px;
      flex-shrink: 0;
      background: white;
      border-radius: 12px;
      padding: 20px;
      height: fit-content;
      box-shadow: 0 2px 8px rgba(0,0,0,0.06);
    }
    .filter-header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 20px; }
    .filter-header h3 { margin: 0; font-size: 16px; }
    .btn-clear-all { background: none; border: none; color: #3b82f6; cursor: pointer; font-size: 13px; }
    
    .filter-section { margin-bottom: 20px; padding-bottom: 20px; border-bottom: 1px solid #f1f5f9; }
    .filter-section h4 { margin: 0 0 12px; font-size: 14px; color: #374151; }
    
    .search-input-wrapper { position: relative; }
    .search-icon { position: absolute; left: 10px; top: 50%; transform: translateY(-50%); }
    .search-input { width: 100%; padding: 8px 12px 8px 32px; border: 1px solid #e2e8f0; border-radius: 8px; box-sizing: border-box; }
    
    .checkbox-list { display: flex; flex-direction: column; gap: 8px; }
    .checkbox-item { display: flex; align-items: center; gap: 8px; cursor: pointer; font-size: 14px; }
    .checkbox-item .count { color: #9ca3af; font-size: 12px; }
    
    .price-inputs { display: flex; gap: 8px; align-items: center; margin-bottom: 8px; }
    .price-inputs input { width: 80px; padding: 6px 8px; border: 1px solid #e2e8f0; border-radius: 6px; }
    .price-presets { display: flex; flex-wrap: wrap; gap: 6px; }
    .price-presets button { padding: 4px 8px; border: 1px solid #e2e8f0; border-radius: 6px; background: white; cursor: pointer; font-size: 12px; }
    .price-presets button.active { background: #3b82f6; color: white; border-color: #3b82f6; }
    
    .rating-filter { display: flex; flex-direction: column; gap: 4px; }
    .rating-filter button { display: flex; align-items: center; gap: 4px; padding: 6px 10px; border: 1px solid #e2e8f0; border-radius: 6px; background: white; cursor: pointer; font-size: 13px; }
    .rating-filter button.active { background: #fef3c7; border-color: #f59e0b; }
    .rating-filter span { color: #d1d5db; }
    .rating-filter .filled { color: #f59e0b; }
    
    .products-main { flex: 1; }
    .toolbar { display: flex; justify-content: space-between; align-items: center; margin-bottom: 16px; }
    .sort-section { display: flex; align-items: center; gap: 8px; }
    .sort-section select { padding: 6px 10px; border: 1px solid #e2e8f0; border-radius: 6px; }
    .view-toggle button { padding: 6px 10px; border: 1px solid #e2e8f0; background: white; cursor: pointer; }
    .view-toggle .active { background: #3b82f6; color: white; border-color: #3b82f6; }
    
    .active-filters { display: flex; flex-wrap: wrap; gap: 8px; align-items: center; margin-bottom: 16px; padding: 12px; background: #f0f9ff; border-radius: 8px; font-size: 13px; }
    .filter-tag { display: flex; align-items: center; gap: 4px; padding: 4px 10px; background: white; border: 1px solid #bfdbfe; border-radius: 20px; }
    .filter-tag button { background: none; border: none; cursor: pointer; color: #6b7280; }
    
    .products-grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(220px, 1fr)); gap: 20px; }
    
    .product-card { background: white; border-radius: 12px; overflow: hidden; box-shadow: 0 2px 8px rgba(0,0,0,0.06); transition: transform 0.2s, box-shadow 0.2s; position: relative; }
    .product-card:hover { transform: translateY(-4px); box-shadow: 0 8px 24px rgba(0,0,0,0.12); }
    .product-card.out-of-stock { opacity: 0.7; }
    .product-card.on-sale { border: 2px solid #ef4444; }
    
    .badges { position: absolute; top: 10px; left: 10px; display: flex; gap: 4px; z-index: 1; }
    .badge { padding: 3px 8px; border-radius: 4px; font-size: 11px; font-weight: bold; }
    .badge-new { background: #10b981; color: white; }
    .badge-sale { background: #ef4444; color: white; }
    
    .product-image { height: 160px; display: flex; align-items: center; justify-content: center; font-size: 48px; }
    .product-info { padding: 12px; }
    .brand { color: #6b7280; font-size: 12px; margin: 0 0 4px; }
    .name { font-size: 15px; font-weight: 600; margin: 0 0 8px; line-height: 1.3; }
    .rating { display: flex; gap: 2px; align-items: center; margin-bottom: 8px; }
    .rating span { color: #d1d5db; font-size: 16px; }
    .rating .filled { color: #f59e0b; }
    .review-count { font-size: 12px; color: #6b7280; margin-left: 4px; }
    .price { display: flex; align-items: baseline; gap: 8px; margin-bottom: 6px; }
    .current-price { font-size: 18px; font-weight: 700; color: #1f2937; }
    .original-price { font-size: 13px; color: #9ca3af; text-decoration: line-through; }
    .stock-status { font-size: 12px; margin: 4px 0; }
    .in-stock { color: #059669; }
    .out-of-stock { color: #dc2626; }
    .colors { display: flex; gap: 4px; margin: 6px 0; }
    .color-dot { width: 16px; height: 16px; border-radius: 50%; border: 1px solid #ddd; }
    .tags { display: flex; flex-wrap: wrap; gap: 4px; }
    .tag { padding: 2px 8px; background: #f1f5f9; border-radius: 12px; font-size: 11px; color: #475569; }
    
    .product-actions { display: flex; gap: 8px; padding: 0 12px 12px; }
    .btn-add-cart { flex: 1; padding: 8px; background: #3b82f6; color: white; border: none; border-radius: 8px; cursor: pointer; font-size: 13px; }
    .btn-add-cart:disabled { background: #e2e8f0; color: #94a3b8; cursor: not-allowed; }
    .btn-wishlist { padding: 8px 12px; border: 1px solid #e2e8f0; background: white; border-radius: 8px; cursor: pointer; font-size: 18px; }
    
    .products-list { display: flex; flex-direction: column; gap: 12px; }
    .product-list-item { display: flex; gap: 16px; background: white; border-radius: 12px; padding: 16px; box-shadow: 0 2px 8px rgba(0,0,0,0.06); }
    .list-image { width: 80px; height: 80px; border-radius: 8px; display: flex; align-items: center; justify-content: center; font-size: 32px; flex-shrink: 0; }
    .list-info { flex: 1; }
    .list-info .description { font-size: 13px; color: #6b7280; }
    .list-price { text-align: right; }
    .list-price .current-price { font-size: 20px; font-weight: 700; }
    
    .empty-state { text-align: center; padding: 60px; color: #6b7280; }
    .empty-state button { margin-top: 16px; padding: 10px 20px; background: #3b82f6; color: white; border: none; border-radius: 8px; cursor: pointer; }
    .mobile-filter-btn { display: none; }
    
    @media (max-width: 768px) {
      .layout { flex-direction: column; }
      .filter-sidebar { width: 100%; display: none; }
      .filter-sidebar.mobile-open { display: block; }
      .mobile-filter-btn { display: block; position: fixed; bottom: 20px; right: 20px; padding: 12px 20px; background: #3b82f6; color: white; border: none; border-radius: 24px; cursor: pointer; box-shadow: 0 4px 12px rgba(0,0,0,0.2); z-index: 100; }
      .products-grid { grid-template-columns: repeat(2, 1fr); }
    }
  `]
})
export class ProductFilterComponent implements OnInit {
  allProducts: Product[] = [];
  viewMode: 'grid' | 'list' = 'grid';
  isMobileFilterOpen = false;
  wishlist = new Set<number>();

  filters: FilterOptions = {
    search: '',
    categories: [],
    brands: [],
    minPrice: 0,
    maxPrice: 999999,
    minRating: 0,
    inStock: false,
    isNew: false,
    isSale: false,
    sortBy: 'name'
  };

  availableCategories = [
    { id: 'phone', name: 'โทรศัพท์' },
    { id: 'laptop', name: 'แล็ปท็อป' },
    { id: 'tablet', name: 'แท็บเล็ต' },
    { id: 'accessories', name: 'อุปกรณ์เสริม' }
  ];

  availableBrands = ['Apple', 'Samsung', 'Dell', 'Lenovo', 'Sony'];

  pricePresets = [
    { label: 'ทั้งหมด', min: 0, max: 999999 },
    { label: 'ต่ำกว่า 10,000', min: 0, max: 10000 },
    { label: '10,000-30,000', min: 10000, max: 30000 },
    { label: '30,000-50,000', min: 30000, max: 50000 },
    { label: 'มากกว่า 50,000', min: 50000, max: 999999 }
  ];

  get filteredProducts(): Product[] {
    let products = [...this.allProducts];
    if (this.filters.search) {
      const s = this.filters.search.toLowerCase();
      products = products.filter(p =>
        p.name.toLowerCase().includes(s) ||
        p.brand.toLowerCase().includes(s)
      );
    }
    if (this.filters.categories.length) {
      products = products.filter(p => this.filters.categories.includes(p.category));
    }
    if (this.filters.brands.length) {
      products = products.filter(p => this.filters.brands.includes(p.brand));
    }
    products = products.filter(p =>
      p.price >= this.filters.minPrice && p.price <= this.filters.maxPrice
    );
    if (this.filters.minRating) {
      products = products.filter(p => p.rating >= this.filters.minRating);
    }
    if (this.filters.inStock) products = products.filter(p => p.inStock);
    if (this.filters.isNew) products = products.filter(p => p.isNew);
    if (this.filters.isSale) products = products.filter(p => p.isSale);
    return products;
  }

  get hasActiveFilters(): boolean {
    return !!(this.filters.search ||
      this.filters.categories.length ||
      this.filters.brands.length ||
      this.filters.minPrice > 0 ||
      this.filters.maxPrice < 999999 ||
      this.filters.minRating > 0 ||
      this.filters.inStock ||
      this.filters.isNew ||
      this.filters.isSale);
  }

  get activeFilterCount(): number {
    return this.getActiveFilterTags().length;
  }

  ngOnInit(): void {
    this.loadProducts();
  }

  loadProducts(): void {
    this.allProducts = [
      {
        id: 1, name: 'iPhone 15 Pro', description: 'สมาร์ทโฟนจาก Apple',
        price: 45000, originalPrice: 50000, category: 'phone', brand: 'Apple',
        rating: 4.8, reviewCount: 2341, inStock: true, isNew: true, isSale: true,
        tags: ['smartphone', '5G', 'Pro'], imageUrl: '', colors: ['#1a1a1a', '#e8d5c4', '#f5f5f0']
      },
      {
        id: 2, name: 'Samsung Galaxy S24', description: 'สมาร์ทโฟน Android ล่าสุด',
        price: 28000, category: 'phone', brand: 'Samsung',
        rating: 4.5, reviewCount: 1823, inStock: true, isNew: true, isSale: false,
        tags: ['android', '5G', 'Galaxy'], imageUrl: '', colors: ['#1a1a2e', '#c0c0c0', '#e8b4d0']
      },
      {
        id: 3, name: 'MacBook Pro M3', description: 'แล็ปท็อปสำหรับงานหนัก',
        price: 85000, category: 'laptop', brand: 'Apple',
        rating: 4.9, reviewCount: 892, inStock: false, isNew: true, isSale: false,
        tags: ['macOS', 'M3', 'Pro'], imageUrl: '', colors: ['#6e6e73', '#e8e8e8']
      },
      {
        id: 4, name: 'Dell XPS 15', description: 'แล็ปท็อป Windows ระดับมือโปร',
        price: 55000, originalPrice: 62000, category: 'laptop', brand: 'Dell',
        rating: 4.3, reviewCount: 456, inStock: true, isNew: false, isSale: true,
        tags: ['windows', 'OLED', '4K'], imageUrl: '', colors: ['#1a1a1a', '#f5f5f5']
      },
      {
        id: 5, name: 'iPad Pro M4', description: 'แท็บเล็ตที่ทรงพลัง',
        price: 38000, category: 'tablet', brand: 'Apple',
        rating: 4.7, reviewCount: 1204, inStock: true, isNew: true, isSale: false,
        tags: ['iPadOS', 'M4', 'Pencil'], imageUrl: '', colors: ['#6e6e73', '#f5f5f0']
      }
    ];
  }

  toggleFilter(type: 'categories' | 'brands', value: string, event: Event): void {
    const checkbox = event.target as HTMLInputElement;
    const arr = this.filters[type] as string[];
    if (checkbox.checked) {
      arr.push(value);
    } else {
      const idx = arr.indexOf(value);
      if (idx > -1) arr.splice(idx, 1);
    }
  }

  setPriceRange(min: number, max: number): void {
    this.filters.minPrice = min;
    this.filters.maxPrice = max;
  }

  setMinRating(rating: number): void {
    this.filters.minRating = this.filters.minRating === rating ? 0 : rating;
  }

  clearAllFilters(): void {
    this.filters = {
      search: '', categories: [], brands: [],
      minPrice: 0, maxPrice: 999999, minRating: 0,
      inStock: false, isNew: false, isSale: false, sortBy: 'name'
    };
  }

  getActiveFilterTags(): { label: string; type: string; value: any }[] {
    const tags: { label: string; type: string; value: any }[] = [];
    if (this.filters.search) tags.push({ label: `"${this.filters.search}"`, type: 'search', value: '' });
    this.filters.categories.forEach(c => {
      const cat = this.availableCategories.find(x => x.id === c);
      tags.push({ label: cat?.name || c, type: 'category', value: c });
    });
    this.filters.brands.forEach(b => tags.push({ label: b, type: 'brand', value: b }));
    if (this.filters.inStock) tags.push({ label: 'มีสินค้า', type: 'inStock', value: true });
    if (this.filters.isNew) tags.push({ label: 'สินค้าใหม่', type: 'isNew', value: true });
    if (this.filters.isSale) tags.push({ label: 'ลดราคา', type: 'isSale', value: true });
    if (this.filters.minRating) tags.push({ label: `${this.filters.minRating}★ ขึ้นไป`, type: 'rating', value: 0 });
    return tags;
  }

  removeFilter(tag: { type: string; value: any }): void {
    switch (tag.type) {
      case 'search': this.filters.search = ''; break;
      case 'category':
        this.filters.categories = this.filters.categories.filter(c => c !== tag.value);
        break;
      case 'brand':
        this.filters.brands = this.filters.brands.filter(b => b !== tag.value);
        break;
      case 'inStock': this.filters.inStock = false; break;
      case 'isNew': this.filters.isNew = false; break;
      case 'isSale': this.filters.isSale = false; break;
      case 'rating': this.filters.minRating = 0; break;
    }
  }

  getCategoryCount(categoryId: string): number {
    return this.allProducts.filter(p => p.category === categoryId).length;
  }

  getDiscountPercent(product: Product): number {
    if (!product.originalPrice) return 0;
    return Math.round((1 - product.price / product.originalPrice) * 100);
  }

  getImagePlaceholderColor(category: string): string {
    const colors: Record<string, string> = {
      phone: '#e3f2fd', laptop: '#f3e5f5',
      tablet: '#e8f5e9', accessories: '#fff3e0'
    };
    return colors[category] || '#f5f5f5';
  }

  getCategoryIcon(category: string): string {
    const icons: Record<string, string> = {
      phone: '📱', laptop: '💻', tablet: '📲', accessories: '🎧'
    };
    return icons[category] || '📦';
  }

  addToCart(product: Product): void {
    alert(`เพิ่ม "${product.name}" ในตะกร้าแล้ว`);
  }

  toggleWishlist(product: Product): void {
    if (this.wishlist.has(product.id)) {
      this.wishlist.delete(product.id);
    } else {
      this.wishlist.add(product.id);
    }
  }

  trackByProductId(index: number, product: Product): number {
    return product.id;
  }
}
```

---

## สรุป Part 05

| Directive | Old Syntax | New Syntax (Angular 17+) |
|-----------|-----------|--------------------------|
| Conditional | `*ngIf="cond"` | `@if (cond) { }` |
| Loop | `*ngFor="let x of arr"` | `@for (x of arr; track x.id) { }` |
| Switch | `[ngSwitch]`, `*ngSwitchCase` | `@switch (val) { @case ('x') {} }` |
| Empty State | ต้องใช้ `ng-template` | `@empty { }` ใน `@for` |
| Else | `else templateRef` | `@else { }` |

### ประโยชน์ของ New Syntax
- ไม่ต้อง import directives
- อ่านง่ายและเข้าใจง่ายกว่า
- Performance ดีกว่า (ไม่ต้องสร้าง directive instances)
- Type checking ที่ดีกว่า
- `@empty` ใน `@for` สะดวกกว่าการใช้ `ngIf` แยก

---

*ก่อนหน้า: [Part 04 — Templates และ Data Binding](part-04-templates-databinding.md)*  
*ต่อไป: [Part 06 — Pipes และการจัดการข้อมูล](part-06-pipes.md)*
