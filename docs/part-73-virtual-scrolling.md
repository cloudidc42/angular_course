# Part 73: Virtual Scrolling ใน Angular (CDK Virtual Scroll)

## ทำไมต้อง Virtual Scrolling

เมื่อ render list ขนาดใหญ่ (เช่น 100,000 รายการ) DOM จะหนักมาก Virtual Scrolling แก้ปัญหาโดย render เฉพาะ items ที่มองเห็นใน viewport

---

## 1. ติดตั้ง CDK

```bash
ng add @angular/cdk
# หรือ
npm install @angular/cdk
```

### นำเข้า Module

```typescript
// app.module.ts
import { ScrollingModule } from '@angular/cdk/scrolling';

@NgModule({
  imports: [ScrollingModule]
})
export class AppModule {}
```

---

## 2. Basic Virtual Scroll

```typescript
// app/components/virtual-list/virtual-list.component.ts
import { Component, OnInit, ChangeDetectionStrategy } from '@angular/core';

interface ListItem {
  id: number;
  name: string;
  description: string;
  avatar: string;
  status: 'active' | 'inactive';
}

@Component({
  selector: 'app-virtual-list',
  template: `
    <div class="container">
      <h2>Virtual Scroll - {{ items.length | number }} รายการ</h2>
      
      <div class="stats">
        <span>กำลังแสดงใน viewport: ประมาณ {{ visibleCount }} รายการ</span>
      </div>

      <cdk-virtual-scroll-viewport 
        itemSize="72" 
        class="viewport"
        (scrolledIndexChange)="onScrollIndexChange($event)"
      >
        <div 
          *cdkVirtualFor="let item of items; 
                          trackBy: trackById;
                          templateCacheSize: 20"
          class="list-item"
          [class.active]="item.status === 'active'"
          (click)="selectItem(item)"
        >
          <img [src]="item.avatar" [alt]="item.name" class="avatar">
          <div class="info">
            <strong>{{ item.name }}</strong>
            <p>{{ item.description }}</p>
          </div>
          <span class="status-badge" [class]="item.status">
            {{ item.status === 'active' ? 'ใช้งาน' : 'ปิด' }}
          </span>
        </div>
      </cdk-virtual-scroll-viewport>

      <div class="selected" *ngIf="selectedItem">
        เลือก: {{ selectedItem.name }}
      </div>
    </div>
  `,
  styles: [`
    .viewport {
      height: 500px;
      width: 100%;
      overflow-y: auto;
      border: 1px solid #ddd;
      border-radius: 8px;
    }
    .list-item {
      height: 72px;
      display: flex;
      align-items: center;
      padding: 8px 16px;
      border-bottom: 1px solid #f0f0f0;
      cursor: pointer;
      transition: background 0.2s;
    }
    .list-item:hover { background: #f5f5f5; }
    .avatar {
      width: 48px;
      height: 48px;
      border-radius: 50%;
      margin-right: 12px;
    }
    .info { flex: 1; }
    .info p { margin: 2px 0; font-size: 12px; color: #666; }
    .status-badge {
      padding: 4px 8px;
      border-radius: 12px;
      font-size: 12px;
    }
    .status-badge.active { background: #e8f5e9; color: #2e7d32; }
    .status-badge.inactive { background: #ffebee; color: #c62828; }
  `],
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class VirtualListComponent implements OnInit {
  items: ListItem[] = [];
  selectedItem: ListItem | null = null;
  currentScrollIndex = 0;
  visibleCount = 7;

  ngOnInit(): void {
    this.generateItems(100000);
  }

  private generateItems(count: number): void {
    const names = ['สมชาย', 'นิดา', 'วิชัย', 'สุภา', 'กิตติ', 'พิมพ์', 'ประยุทธ์', 'อรทัย'];
    const surnamese = ['ใจดี', 'มีสุข', 'รักงาน', 'ทำดี', 'มุ่งมั่น', 'ขยันเรียน'];
    
    this.items = Array.from({ length: count }, (_, i) => ({
      id: i + 1,
      name: `${names[i % names.length]} ${surnamese[i % surnamese.length]}`,
      description: `รายละเอียดที่ ${i + 1} - Email: user${i + 1}@example.com`,
      avatar: `https://i.pravatar.cc/48?img=${(i % 70) + 1}`,
      status: i % 3 === 0 ? 'inactive' : 'active'
    }));
  }

  trackById(index: number, item: ListItem): number {
    return item.id;
  }

  selectItem(item: ListItem): void {
    this.selectedItem = item;
  }

  onScrollIndexChange(index: number): void {
    this.currentScrollIndex = index;
  }
}
```

---

## 3. Variable Height Items

```typescript
// app/components/variable-scroll/variable-scroll.component.ts
import { Component, OnInit } from '@angular/core';
import { CdkVirtualScrollViewport } from '@angular/cdk/scrolling';
import { ViewChild } from '@angular/core';

interface Message {
  id: number;
  sender: string;
  content: string;
  timestamp: Date;
  type: 'text' | 'image' | 'file';
  height?: number;
}

// Custom Size Estimator
function estimateSize(item: Message): number {
  if (item.type === 'image') return 200;
  if (item.type === 'file') return 80;
  
  // คำนวณความสูงตามความยาว text
  const lines = Math.ceil(item.content.length / 50);
  return Math.max(60, 40 + lines * 20);
}

@Component({
  selector: 'app-variable-scroll',
  template: `
    <div class="chat-container">
      <div class="chat-header">
        <h3>Chat ({{ messages.length }} ข้อความ)</h3>
        <button (click)="scrollToBottom()">↓ ล่างสุด</button>
      </div>

      <cdk-virtual-scroll-viewport 
        autosize
        class="chat-viewport"
        #viewport
      >
        <div 
          *cdkVirtualFor="let msg of messages; trackBy: trackById"
          class="message"
          [class.own]="msg.sender === 'Me'"
        >
          <div class="message-header">
            <span class="sender">{{ msg.sender }}</span>
            <span class="time">{{ msg.timestamp | date:'HH:mm' }}</span>
          </div>
          
          <div class="message-body" [ngSwitch]="msg.type">
            <p *ngSwitchCase="'text'">{{ msg.content }}</p>
            <img *ngSwitchCase="'image'" 
                 [src]="msg.content" 
                 alt="รูปภาพ"
                 style="max-width: 200px; border-radius: 8px">
            <div *ngSwitchCase="'file'" class="file-preview">
              📎 {{ msg.content }}
            </div>
          </div>
        </div>
      </cdk-virtual-scroll-viewport>

      <div class="chat-input">
        <input 
          [(ngModel)]="newMessage" 
          placeholder="พิมพ์ข้อความ..."
          (keyup.enter)="sendMessage()"
        >
        <button (click)="sendMessage()">ส่ง</button>
      </div>
    </div>
  `,
  styles: [`
    .chat-container {
      display: flex;
      flex-direction: column;
      height: 600px;
      border: 1px solid #ddd;
      border-radius: 12px;
      overflow: hidden;
    }
    .chat-header {
      display: flex;
      justify-content: space-between;
      padding: 12px 16px;
      background: #f5f5f5;
      border-bottom: 1px solid #ddd;
    }
    .chat-viewport {
      flex: 1;
      overflow-y: auto;
      padding: 8px;
    }
    .message {
      margin: 4px 0;
      padding: 8px 12px;
      background: #fff;
      border-radius: 8px;
      box-shadow: 0 1px 2px rgba(0,0,0,0.1);
      max-width: 80%;
    }
    .message.own {
      margin-left: auto;
      background: #e3f2fd;
    }
    .message-header {
      display: flex;
      justify-content: space-between;
      font-size: 12px;
      margin-bottom: 4px;
    }
    .sender { font-weight: bold; }
    .time { color: #999; }
    .chat-input {
      display: flex;
      padding: 12px;
      border-top: 1px solid #ddd;
      gap: 8px;
    }
    .chat-input input {
      flex: 1;
      padding: 8px 12px;
      border: 1px solid #ddd;
      border-radius: 20px;
    }
  `]
})
export class VariableScrollComponent implements OnInit {
  @ViewChild('viewport') viewport!: CdkVirtualScrollViewport;
  
  messages: Message[] = [];
  newMessage = '';
  messageId = 1;

  ngOnInit(): void {
    this.generateMessages(500);
    setTimeout(() => this.scrollToBottom(), 100);
  }

  private generateMessages(count: number): void {
    const senders = ['Me', 'Alice', 'Bob', 'Charlie'];
    const contents = [
      'สวัสดีครับ มีเรื่องอยากปรึกษา',
      'โอเครับ ว่าไงครับ',
      'เรื่อง Angular ครับ เกี่ยวกับ Virtual Scroll',
      'อ๋อ เป็นเรื่องการ optimize performance ใช่ไหมครับ',
      'ใช่เลยครับ ข้อมูลเยอะมากเลย render ช้า',
      'ใช้ CDK Virtual Scroll ได้เลยครับ ประหยัด memory มากๆ',
    ];

    this.messages = Array.from({ length: count }, (_, i) => ({
      id: i + 1,
      sender: senders[i % senders.length],
      content: contents[i % contents.length] + (i % 5 === 0 ? ' ' + 'x'.repeat(Math.floor(Math.random() * 100)) : ''),
      timestamp: new Date(Date.now() - (count - i) * 60000),
      type: i % 20 === 0 ? 'image' : i % 15 === 0 ? 'file' : 'text'
    }));
  }

  trackById(index: number, msg: Message): number {
    return msg.id;
  }

  sendMessage(): void {
    if (!this.newMessage.trim()) return;
    
    this.messages = [...this.messages, {
      id: this.messages.length + 1,
      sender: 'Me',
      content: this.newMessage,
      timestamp: new Date(),
      type: 'text'
    }];
    
    this.newMessage = '';
    setTimeout(() => this.scrollToBottom(), 50);
  }

  scrollToBottom(): void {
    if (this.viewport) {
      this.viewport.scrollToIndex(this.messages.length - 1, 'smooth');
    }
  }
}
```

---

## 4. Infinite Scroll

```typescript
// app/components/infinite-scroll/infinite-scroll.component.ts
import { Component, OnInit } from '@angular/core';
import { CdkVirtualScrollViewport } from '@angular/cdk/scrolling';
import { ViewChild, ChangeDetectionStrategy, ChangeDetectorRef } from '@angular/core';
import { BehaviorSubject } from 'rxjs';
import { HttpClient } from '@angular/common/http';

interface Product {
  id: number;
  title: string;
  price: number;
  category: string;
  thumbnail: string;
}

@Component({
  selector: 'app-infinite-scroll',
  template: `
    <div class="infinite-container">
      <h2>Infinite Scroll Products</h2>
      
      <cdk-virtual-scroll-viewport 
        itemSize="120"
        class="product-viewport"
        (scrolledIndexChange)="checkLoadMore($event)"
        #viewport
      >
        <div 
          *cdkVirtualFor="let product of products$; trackBy: trackById; templateCacheSize: 10"
          class="product-card"
        >
          <ng-container *ngIf="product; else skeleton">
            <img [src]="product.thumbnail" [alt]="product.title" width="80" height="80">
            <div class="product-info">
              <h4>{{ product.title }}</h4>
              <span class="category">{{ product.category }}</span>
              <strong class="price">฿{{ product.price | number:'1.2-2' }}</strong>
            </div>
          </ng-container>
          
          <ng-template #skeleton>
            <div class="skeleton-card">
              <div class="skeleton skeleton-img"></div>
              <div class="skeleton-text">
                <div class="skeleton skeleton-line long"></div>
                <div class="skeleton skeleton-line short"></div>
              </div>
            </div>
          </ng-template>
        </div>
      </cdk-virtual-scroll-viewport>

      <div class="load-more" *ngIf="isLoading">
        กำลังโหลดข้อมูลเพิ่มเติม...
      </div>
      <div class="end-message" *ngIf="!hasMore && !isLoading">
        แสดงทั้งหมด {{ products$.getValue().length }} รายการ
      </div>
    </div>
  `,
  styles: [`
    .product-viewport { height: 600px; width: 100%; }
    .product-card {
      height: 120px;
      display: flex;
      align-items: center;
      padding: 12px;
      border-bottom: 1px solid #eee;
      gap: 16px;
    }
    .product-card img { object-fit: cover; border-radius: 8px; }
    .product-info { flex: 1; }
    .product-info h4 { margin: 0 0 4px; font-size: 14px; }
    .category { font-size: 12px; color: #999; display: block; margin-bottom: 4px; }
    .price { color: #e91e63; font-size: 16px; }
    .skeleton { background: linear-gradient(90deg, #f0f0f0 25%, #e0e0e0 50%, #f0f0f0 75%); animation: shimmer 1.5s infinite; }
    @keyframes shimmer { 0% { background-position: -1000px 0; } 100% { background-position: 1000px 0; } }
    .skeleton-img { width: 80px; height: 80px; border-radius: 8px; }
    .skeleton-line { height: 14px; border-radius: 4px; margin: 4px 0; }
    .long { width: 70%; }
    .short { width: 40%; }
  `],
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class InfiniteScrollComponent implements OnInit {
  @ViewChild('viewport') viewport!: CdkVirtualScrollViewport;
  
  products$ = new BehaviorSubject<(Product | null)[]>([]);
  isLoading = false;
  hasMore = true;
  page = 0;
  pageSize = 20;
  totalItems = 200; // สมมติมีทั้งหมด 200 รายการ

  constructor(
    private http: HttpClient,
    private cdr: ChangeDetectorRef
  ) {}

  ngOnInit(): void {
    this.loadMore();
  }

  async loadMore(): Promise<void> {
    if (this.isLoading || !this.hasMore) return;
    
    this.isLoading = true;
    
    try {
      // จำลอง API call
      const newItems = await this.fetchProducts(this.page, this.pageSize);
      
      const current = this.products$.getValue().filter(p => p !== null);
      this.products$.next([...current, ...newItems]);
      
      this.page++;
      this.hasMore = this.products$.getValue().length < this.totalItems;
    } finally {
      this.isLoading = false;
      this.cdr.markForCheck();
    }
  }

  private fetchProducts(page: number, size: number): Promise<Product[]> {
    // จำลองข้อมูล
    return new Promise(resolve => {
      setTimeout(() => {
        const categories = ['Electronics', 'Clothing', 'Books', 'Sports', 'Home'];
        resolve(Array.from({ length: size }, (_, i) => ({
          id: page * size + i + 1,
          title: `สินค้า ${page * size + i + 1}`,
          price: Math.round(Math.random() * 5000) / 100 * 100,
          category: categories[Math.floor(Math.random() * categories.length)],
          thumbnail: `https://picsum.photos/80/80?random=${page * size + i}`
        })));
      }, 500);
    });
  }

  checkLoadMore(index: number): void {
    const total = this.products$.getValue().length;
    if (index >= total - 10) {
      this.loadMore();
    }
  }

  trackById(index: number, item: Product | null): number {
    return item?.id ?? index;
  }
}
```

---

## 5. Virtual Scroll Grid

```typescript
// app/components/virtual-grid/virtual-grid.component.ts
import { Component, OnInit, HostListener } from '@angular/core';

interface GridItem {
  id: number;
  title: string;
  imageUrl: string;
  price: number;
}

@Component({
  selector: 'app-virtual-grid',
  template: `
    <div class="grid-container">
      <cdk-virtual-scroll-viewport 
        [itemSize]="rowHeight"
        class="grid-viewport"
      >
        <div 
          *cdkVirtualFor="let row of rows; trackBy: trackByRow"
          class="grid-row"
          [style.height.px]="rowHeight"
        >
          <div 
            *ngFor="let item of row; trackBy: trackById"
            class="grid-cell"
            [style.width.px]="cellWidth"
          >
            <ng-container *ngIf="item">
              <img [src]="item.imageUrl" [alt]="item.title">
              <div class="cell-info">
                <h5>{{ item.title }}</h5>
                <span>฿{{ item.price | number }}</span>
              </div>
            </ng-container>
          </div>
        </div>
      </cdk-virtual-scroll-viewport>
    </div>
  `,
  styles: [`
    .grid-viewport { height: 600px; width: 100%; }
    .grid-row { display: flex; gap: 8px; padding: 4px; }
    .grid-cell { border: 1px solid #eee; border-radius: 8px; overflow: hidden; }
    .grid-cell img { width: 100%; height: 120px; object-fit: cover; }
    .cell-info { padding: 8px; }
    .cell-info h5 { margin: 0; font-size: 13px; }
    .cell-info span { color: #e91e63; font-weight: bold; }
  `]
})
export class VirtualGridComponent implements OnInit {
  items: GridItem[] = [];
  rows: (GridItem | null)[][] = [];
  columns = 3;
  cellWidth = 200;
  rowHeight = 200;

  @HostListener('window:resize')
  onResize(): void {
    this.columns = Math.max(1, Math.floor(window.innerWidth / 220));
    this.buildRows();
  }

  ngOnInit(): void {
    this.items = Array.from({ length: 1000 }, (_, i) => ({
      id: i + 1,
      title: `สินค้า ${i + 1}`,
      imageUrl: `https://picsum.photos/200/120?random=${i}`,
      price: Math.floor(Math.random() * 10000) + 100
    }));
    
    this.onResize();
  }

  private buildRows(): void {
    this.rows = [];
    for (let i = 0; i < this.items.length; i += this.columns) {
      const row: (GridItem | null)[] = this.items.slice(i, i + this.columns);
      // เติม null ให้ครบ columns
      while (row.length < this.columns) row.push(null);
      this.rows.push(row);
    }
  }

  trackByRow(index: number): number { return index; }
  trackById(index: number, item: GridItem | null): number { return item?.id ?? index; }
}
```

---

## สรุป

| Feature | วิธีใช้ |
|---------|---------|
| Fixed Height | `itemSize="72"` |
| Variable Height | `autosize` directive |
| Grid | แบ่ง items เป็น rows |
| Infinite Scroll | ตรวจ `scrolledIndexChange` |

### Performance Tips

1. ใช้ `trackBy` เสมอ
2. ตั้ง `itemSize` ให้ถูกต้อง
3. ใช้ `ChangeDetectionStrategy.OnPush`
4. Preload ข้อมูลก่อน scroll ถึง
5. Cache rendered items ด้วย `templateCacheSize`
