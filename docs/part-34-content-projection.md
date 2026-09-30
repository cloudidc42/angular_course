# Part 34 — Content Projection

## Content Projection คืออะไร?

Content Projection เป็นวิธีการส่ง HTML Content จาก Parent Component เข้าไปแสดงใน Child Component ทำให้ Component มีความยืดหยุ่นสูงและนำกลับมาใช้ใหม่ได้

---

## ng-content พื้นฐาน

```typescript
// card.component.ts
@Component({
  selector: 'app-card',
  standalone: true,
  template: `
    <div class="card">
      <div class="card-body">
        <ng-content></ng-content>  <!-- Content จาก Parent จะแสดงที่นี่ -->
      </div>
    </div>
  `,
  styles: [`
    .card { border: 1px solid #ddd; border-radius: 8px; padding: 16px; }
  `]
})
export class CardComponent {}
```

```html
<!-- การใช้งาน -->
<app-card>
  <h2>ชื่อการ์ด</h2>
  <p>เนื้อหาการ์ด</p>
  <button>คลิก</button>
</app-card>
```

---

## select attribute — Multi-slot Projection

`select` ใช้กรอง Content ที่จะแสดงในแต่ละ Slot

```typescript
// panel.component.ts
@Component({
  selector: 'app-panel',
  standalone: true,
  template: `
    <div class="panel">
      <!-- Slot สำหรับ Header -->
      <div class="panel-header">
        <ng-content select="[panel-header]"></ng-content>
      </div>

      <!-- Slot สำหรับ Body -->
      <div class="panel-body">
        <ng-content select="[panel-body]"></ng-content>
      </div>

      <!-- Slot สำหรับ Footer -->
      <div class="panel-footer">
        <ng-content select="[panel-footer]"></ng-content>
      </div>
    </div>
  `,
  styles: [`
    .panel { border: 1px solid #ddd; border-radius: 8px; overflow: hidden; }
    .panel-header { background: #f5f5f5; padding: 12px 16px; border-bottom: 1px solid #ddd; }
    .panel-body { padding: 16px; }
    .panel-footer { background: #f5f5f5; padding: 12px 16px; border-top: 1px solid #ddd; }
  `]
})
export class PanelComponent {}
```

```html
<!-- การใช้งาน -->
<app-panel>
  <div panel-header>
    <h3>หัวข้อ Panel</h3>
  </div>
  <div panel-body>
    <p>เนื้อหาหลัก</p>
  </div>
  <div panel-footer>
    <button>ยืนยัน</button>
    <button>ยกเลิก</button>
  </div>
</app-panel>
```

### selector แบบต่างๆ

```html
<!-- กรองด้วย Attribute -->
<ng-content select="[myAttr]"></ng-content>

<!-- กรองด้วย CSS Class -->
<ng-content select=".my-class"></ng-content>

<!-- กรองด้วย Element Tag -->
<ng-content select="h1"></ng-content>
<ng-content select="app-header"></ng-content>

<!-- กรองด้วย Attribute + Value -->
<ng-content select="[slot='header']"></ng-content>

<!-- รับเนื้อหาที่เหลือทั้งหมด (ต้องอยู่ท้ายสุด) -->
<ng-content></ng-content>
```

---

## ngTemplateOutlet

`ngTemplateOutlet` ใช้ Render Template ที่ส่งมาจากภายนอก

### พื้นฐาน

```typescript
// list.component.ts
@Component({
  selector: 'app-list',
  standalone: true,
  imports: [CommonModule],
  template: `
    <ul>
      <li *ngFor="let item of items">
        <!-- Render Template ที่ส่งมา พร้อม Context -->
        <ng-container
          *ngTemplateOutlet="itemTemplate; context: { $implicit: item, index: i }"
          *ngFor="let item of items; let i = index"
        >
        </ng-container>
      </li>
    </ul>
  `
})
export class ListComponent<T> {
  @Input() items: T[] = [];
  @Input() itemTemplate!: TemplateRef<{ $implicit: T; index: number }>;
}
```

```html
<!-- การใช้งาน -->
<app-list [items]="products" [itemTemplate]="productTpl">
  <!-- กำหนด Template -->
</app-list>

<ng-template #productTpl let-product let-i="index">
  <span>{{ i + 1 }}. {{ product.name }} - {{ product.price | currency:'THB' }}</span>
</ng-template>
```

---

## Workshop: Flexible Card Component แบบสมบูรณ์

### Card Component

```typescript
// flexible-card.component.ts
import {
  Component,
  Input,
  Output,
  EventEmitter,
  ContentChild,
  ContentChildren,
  QueryList,
  TemplateRef,
  AfterContentInit,
} from '@angular/core';
import { CommonModule } from '@angular/common';

// Sub-components สำหรับ Named Slots
@Component({
  selector: 'app-card-header',
  standalone: true,
  template: '<ng-content></ng-content>',
})
export class CardHeaderComponent {}

@Component({
  selector: 'app-card-body',
  standalone: true,
  template: '<ng-content></ng-content>',
})
export class CardBodyComponent {}

@Component({
  selector: 'app-card-footer',
  standalone: true,
  template: '<ng-content></ng-content>',
})
export class CardFooterComponent {}

// Main Card Component
@Component({
  selector: 'app-flexible-card',
  standalone: true,
  imports: [CommonModule],
  template: `
    <div class="card" [class]="variant" [class.loading]="loading" [class.selected]="selected">
      <!-- Loading Overlay -->
      <div class="card-loading" *ngIf="loading">
        <div class="spinner"></div>
      </div>

      <!-- Header Slot -->
      <div class="card-header" *ngIf="hasHeader">
        <ng-content select="app-card-header"></ng-content>
      </div>

      <!-- Image Slot -->
      <div class="card-image" *ngIf="imageUrl || hasImageContent">
        <img *ngIf="imageUrl" [src]="imageUrl" [alt]="imageAlt" />
        <ng-content select="[card-image]"></ng-content>
      </div>

      <!-- Body -->
      <div class="card-body">
        <ng-content select="app-card-body"></ng-content>
        <!-- Default Content ถ้าไม่มี card-body -->
        <ng-content></ng-content>
      </div>

      <!-- Footer Slot -->
      <div class="card-footer" *ngIf="hasFooter">
        <ng-content select="app-card-footer"></ng-content>
      </div>
    </div>
  `,
  styles: [`
    .card {
      border: 1px solid #e0e0e0;
      border-radius: 8px;
      overflow: hidden;
      background: white;
      box-shadow: 0 2px 4px rgba(0,0,0,0.08);
      transition: box-shadow 0.2s, transform 0.2s;
      position: relative;
    }
    .card:hover { box-shadow: 0 4px 12px rgba(0,0,0,0.15); }
    .card.selected { border-color: #1976d2; box-shadow: 0 0 0 2px #1976d240; }
    .card.flat { box-shadow: none; }
    .card.outlined { border-width: 2px; }
    .card.elevated { box-shadow: 0 8px 24px rgba(0,0,0,0.12); }

    .card-header {
      padding: 16px 16px 0;
      font-size: 18px;
      font-weight: 600;
    }
    .card-image img { width: 100%; height: auto; display: block; }
    .card-body { padding: 16px; }
    .card-footer {
      padding: 12px 16px;
      background: #fafafa;
      border-top: 1px solid #e0e0e0;
      display: flex;
      gap: 8px;
      justify-content: flex-end;
    }

    .card-loading {
      position: absolute;
      inset: 0;
      background: rgba(255,255,255,0.8);
      display: flex;
      align-items: center;
      justify-content: center;
      z-index: 1;
    }
    .spinner {
      width: 32px;
      height: 32px;
      border: 3px solid #e0e0e0;
      border-top-color: #1976d2;
      border-radius: 50%;
      animation: spin 0.8s linear infinite;
    }
    @keyframes spin { to { transform: rotate(360deg); } }
  `]
})
export class FlexibleCardComponent implements AfterContentInit {
  @Input() variant: 'flat' | 'outlined' | 'elevated' | '' = '';
  @Input() imageUrl = '';
  @Input() imageAlt = '';
  @Input() loading = false;
  @Input() selected = false;
  @Output() cardClick = new EventEmitter<void>();

  @ContentChild(CardHeaderComponent) headerContent?: CardHeaderComponent;
  @ContentChild(CardFooterComponent) footerContent?: CardFooterComponent;

  hasHeader = false;
  hasFooter = false;
  hasImageContent = false;

  ngAfterContentInit(): void {
    this.hasHeader = !!this.headerContent;
    this.hasFooter = !!this.footerContent;
  }
}
```

### Table Component พร้อม ngTemplateOutlet

```typescript
// data-table.component.ts
import {
  Component,
  Input,
  Output,
  EventEmitter,
  ContentChildren,
  QueryList,
  AfterContentInit,
  TemplateRef,
} from '@angular/core';
import { CommonModule } from '@angular/common';

@Component({
  selector: 'app-column',
  standalone: true,
  template: '',
})
export class ColumnComponent {
  @Input() field = '';
  @Input() header = '';
  @Input() sortable = false;
  @Input() cellTemplate?: TemplateRef<any>;
}

@Component({
  selector: 'app-data-table',
  standalone: true,
  imports: [CommonModule],
  template: `
    <div class="table-container">
      <!-- Search -->
      <div class="table-toolbar">
        <input
          *ngIf="searchable"
          type="text"
          placeholder="ค้นหา..."
          (input)="onSearch($event)"
        />
        <ng-content select="[table-actions]"></ng-content>
      </div>

      <!-- Table -->
      <table>
        <thead>
          <tr>
            <th
              *ngFor="let col of columns"
              [class.sortable]="col.sortable"
              (click)="col.sortable && sort(col.field)"
            >
              {{ col.header }}
              <span *ngIf="col.sortable && sortField === col.field">
                {{ sortDirection === 'asc' ? '↑' : '↓' }}
              </span>
            </th>
            <th *ngIf="actions">การดำเนินการ</th>
          </tr>
        </thead>
        <tbody>
          <tr
            *ngFor="let row of displayedData; let i = index"
            (click)="rowClick.emit(row)"
          >
            <td *ngFor="let col of columns">
              <ng-container
                *ngIf="col.cellTemplate; else defaultCell"
              >
                <ng-container
                  *ngTemplateOutlet="col.cellTemplate; context: { $implicit: row[col.field], row: row, index: i }"
                ></ng-container>
              </ng-container>
              <ng-template #defaultCell>
                {{ row[col.field] }}
              </ng-template>
            </td>
            <td *ngIf="actions">
              <ng-container
                *ngTemplateOutlet="actions; context: { $implicit: row, index: i }"
              ></ng-container>
            </td>
          </tr>
          <tr *ngIf="displayedData.length === 0">
            <td [attr.colspan]="columns.length + (actions ? 1 : 0)" class="empty">
              ไม่พบข้อมูล
            </td>
          </tr>
        </tbody>
      </table>

      <!-- Pagination -->
      <div class="pagination" *ngIf="paginated">
        <button [disabled]="currentPage === 1" (click)="changePage(currentPage - 1)">
          ←
        </button>
        <span>{{ currentPage }} / {{ totalPages }}</span>
        <button [disabled]="currentPage === totalPages" (click)="changePage(currentPage + 1)">
          →
        </button>
      </div>
    </div>
  `,
  styles: [`
    .table-container { overflow: auto; }
    .table-toolbar { display: flex; justify-content: space-between; padding: 8px; }
    table { width: 100%; border-collapse: collapse; }
    th, td { padding: 10px 12px; text-align: left; border-bottom: 1px solid #e0e0e0; }
    th { background: #f5f5f5; font-weight: 600; }
    th.sortable { cursor: pointer; user-select: none; }
    th.sortable:hover { background: #eeeeee; }
    tr:hover td { background: #f9f9f9; }
    .empty { text-align: center; color: #999; padding: 32px; }
    .pagination { display: flex; align-items: center; gap: 16px; padding: 8px; justify-content: center; }
  `]
})
export class DataTableComponent<T> implements AfterContentInit {
  @Input() data: T[] = [];
  @Input() searchable = true;
  @Input() paginated = true;
  @Input() pageSize = 10;
  @Input() actions?: TemplateRef<any>;
  @Output() rowClick = new EventEmitter<T>();

  @ContentChildren(ColumnComponent) columns!: QueryList<ColumnComponent>;

  displayedData: T[] = [];
  currentPage = 1;
  totalPages = 1;
  sortField = '';
  sortDirection: 'asc' | 'desc' = 'asc';
  searchTerm = '';

  ngAfterContentInit(): void {
    this.processData();
  }

  onSearch(event: Event): void {
    this.searchTerm = (event.target as HTMLInputElement).value.toLowerCase();
    this.currentPage = 1;
    this.processData();
  }

  sort(field: string): void {
    if (this.sortField === field) {
      this.sortDirection = this.sortDirection === 'asc' ? 'desc' : 'asc';
    } else {
      this.sortField = field;
      this.sortDirection = 'asc';
    }
    this.processData();
  }

  changePage(page: number): void {
    this.currentPage = page;
    this.processData();
  }

  private processData(): void {
    let filtered = [...this.data] as any[];

    // Search
    if (this.searchTerm) {
      filtered = filtered.filter((row) =>
        Object.values(row).some((val) =>
          String(val).toLowerCase().includes(this.searchTerm)
        )
      );
    }

    // Sort
    if (this.sortField) {
      filtered.sort((a, b) => {
        const aVal = a[this.sortField];
        const bVal = b[this.sortField];
        const comparison = aVal < bVal ? -1 : aVal > bVal ? 1 : 0;
        return this.sortDirection === 'asc' ? comparison : -comparison;
      });
    }

    // Paginate
    this.totalPages = Math.max(1, Math.ceil(filtered.length / this.pageSize));
    this.currentPage = Math.min(this.currentPage, this.totalPages);
    const start = (this.currentPage - 1) * this.pageSize;
    this.displayedData = this.paginated
      ? filtered.slice(start, start + this.pageSize)
      : filtered;
  }
}
```

### การใช้งาน DataTable

```html
<!-- products-table.component.html -->
<app-data-table
  [data]="products"
  [searchable]="true"
  [paginated]="true"
  [pageSize]="5"
  [actions]="rowActions"
  (rowClick)="onRowClick($event)"
>
  <!-- Action Buttons ใน Toolbar -->
  <div table-actions>
    <button (click)="addProduct()">+ เพิ่มสินค้า</button>
  </div>

  <!-- Column Definitions -->
  <app-column field="id" header="รหัส" [sortable]="true"></app-column>
  <app-column field="name" header="ชื่อสินค้า" [sortable]="true"></app-column>
  <app-column
    field="price"
    header="ราคา"
    [sortable]="true"
    [cellTemplate]="priceTemplate"
  ></app-column>
  <app-column
    field="isActive"
    header="สถานะ"
    [cellTemplate]="statusTemplate"
  ></app-column>
</app-data-table>

<!-- Custom Cell Templates -->
<ng-template #priceTemplate let-price>
  <strong>{{ price | currency:'THB' }}</strong>
</ng-template>

<ng-template #statusTemplate let-isActive let-row="row">
  <span [class.active]="isActive" [class.inactive]="!isActive">
    {{ isActive ? '✅ Active' : '❌ Inactive' }}
  </span>
</ng-template>

<!-- Row Actions Template -->
<ng-template #rowActions let-product let-i="index">
  <button (click)="editProduct(product)">แก้ไข</button>
  <button (click)="deleteProduct(product.id)" class="danger">ลบ</button>
</ng-template>
```

---

## ContentChild และ ContentChildren

```typescript
// accordion.component.ts
import {
  Component,
  ContentChildren,
  QueryList,
  AfterContentInit,
} from '@angular/core';
import { CommonModule } from '@angular/common';

@Component({
  selector: 'app-accordion-item',
  standalone: true,
  imports: [CommonModule],
  template: `
    <div class="accordion-item" [class.open]="isOpen">
      <div class="accordion-header" (click)="toggle()">
        <ng-content select="[slot='title']"></ng-content>
        <span class="icon">{{ isOpen ? '▲' : '▼' }}</span>
      </div>
      <div class="accordion-body" *ngIf="isOpen">
        <ng-content></ng-content>
      </div>
    </div>
  `,
  styles: [`
    .accordion-header {
      padding: 12px 16px;
      cursor: pointer;
      display: flex;
      justify-content: space-between;
      background: #f5f5f5;
    }
    .accordion-body { padding: 16px; }
  `]
})
export class AccordionItemComponent {
  isOpen = false;

  toggle(): void {
    this.isOpen = !this.isOpen;
  }

  open(): void {
    this.isOpen = true;
  }

  close(): void {
    this.isOpen = false;
  }
}

@Component({
  selector: 'app-accordion',
  standalone: true,
  imports: [CommonModule],
  template: `<ng-content></ng-content>`,
})
export class AccordionComponent implements AfterContentInit {
  @Input() multiOpen = false;
  @ContentChildren(AccordionItemComponent) items!: QueryList<AccordionItemComponent>;

  ngAfterContentInit(): void {
    if (!this.multiOpen) {
      this.items.forEach((item) => {
        item.toggle = () => {
          const wasOpen = item.isOpen;
          if (!this.multiOpen) {
            this.items.forEach((i) => i.close());
          }
          if (!wasOpen) item.open();
        };
      });
    }
  }
}
```

```html
<!-- การใช้งาน Accordion -->
<app-accordion [multiOpen]="false">
  <app-accordion-item>
    <span slot="title">หัวข้อที่ 1</span>
    เนื้อหาของหัวข้อที่ 1
  </app-accordion-item>
  <app-accordion-item>
    <span slot="title">หัวข้อที่ 2</span>
    เนื้อหาของหัวข้อที่ 2
  </app-accordion-item>
  <app-accordion-item>
    <span slot="title">หัวข้อที่ 3</span>
    เนื้อหาของหัวข้อที่ 3
  </app-accordion-item>
</app-accordion>
```

---

## สรุป

| Feature | การใช้งาน |
|---------|-----------|
| `<ng-content>` | Single-slot projection |
| `<ng-content select="...">` | Multi-slot projection |
| `<ng-template #ref>` | กำหนด Template |
| `*ngTemplateOutlet="ref; context: {...}"` | Render Template |
| `@ContentChild(Type)` | ดึง Single Projected Component |
| `@ContentChildren(Type)` | ดึง All Projected Components |
| `$implicit` ใน Context | ค่า default ของ let variable |

Content Projection ทำให้ Components มีความยืดหยุ่นสูง ผู้ใช้กำหนด Content ได้เอง ใน Part ถัดไปจะเรียน View Encapsulation
