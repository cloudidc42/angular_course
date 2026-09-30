# Part 43: Angular CDK (Component Dev Kit)

## บทนำ

Angular CDK (Component Dev Kit) เป็น library ที่ให้ building blocks สำหรับสร้าง custom components ที่มีความสามารถสูง ในบทนี้เราจะเรียนรู้ Overlay, DragDrop, Virtual Scroll และ Clipboard

---

## 1. การติดตั้ง CDK

```bash
npm install @angular/cdk
```

---

## 2. CDK Overlay

Overlay ใช้สำหรับสร้าง floating panels, tooltips, dropdowns ที่กำหนดเองได้

```typescript
// app/services/overlay.service.ts
import {
  Injectable,
  ComponentRef,
  Injector
} from '@angular/core';
import {
  Overlay,
  OverlayRef,
  OverlayConfig,
  ConnectedPosition
} from '@angular/cdk/overlay';
import {
  ComponentPortal,
  TemplatePortal
} from '@angular/cdk/portal';

@Injectable({ providedIn: 'root' })
export class OverlayService {
  private overlayRefs = new Map<string, OverlayRef>();

  constructor(
    private overlay: Overlay,
    private injector: Injector
  ) {}

  // สร้าง overlay แบบ centered (modal-like)
  openCentered<T>(
    component: any,
    id: string,
    data?: any
  ): ComponentRef<T> {
    const overlayRef = this.overlay.create({
      hasBackdrop: true,
      backdropClass: 'cdk-overlay-dark-backdrop',
      positionStrategy: this.overlay.position().global().centerHorizontally().centerVertically(),
      scrollStrategy: this.overlay.scrollStrategies.block()
    });

    this.overlayRefs.set(id, overlayRef);

    // ปิดเมื่อคลิก backdrop
    overlayRef.backdropClick().subscribe(() => this.close(id));

    const portal = new ComponentPortal(component, null, this.injector);
    return overlayRef.attach(portal);
  }

  // สร้าง overlay ที่ติดกับ element (dropdown-like)
  openConnected<T>(
    component: any,
    id: string,
    origin: HTMLElement
  ): ComponentRef<T> {
    const positions: ConnectedPosition[] = [
      {
        originX: 'start',
        originY: 'bottom',
        overlayX: 'start',
        overlayY: 'top',
        offsetY: 4
      },
      {
        originX: 'start',
        originY: 'top',
        overlayX: 'start',
        overlayY: 'bottom',
        offsetY: -4
      }
    ];

    const overlayRef = this.overlay.create({
      hasBackdrop: true,
      backdropClass: 'cdk-overlay-transparent-backdrop',
      positionStrategy: this.overlay.position()
        .flexibleConnectedTo(origin)
        .withPositions(positions),
      scrollStrategy: this.overlay.scrollStrategies.reposition()
    });

    this.overlayRefs.set(id, overlayRef);
    overlayRef.backdropClick().subscribe(() => this.close(id));

    const portal = new ComponentPortal(component);
    return overlayRef.attach(portal);
  }

  close(id: string) {
    const ref = this.overlayRefs.get(id);
    if (ref?.hasAttached()) {
      ref.detach();
      ref.dispose();
    }
    this.overlayRefs.delete(id);
  }

  closeAll() {
    this.overlayRefs.forEach((ref, id) => this.close(id));
  }
}
```

---

## 3. Custom Dropdown Component

```typescript
// app/components/custom-dropdown/custom-dropdown.component.ts
import {
  Component,
  Input,
  Output,
  EventEmitter,
  ElementRef,
  TemplateRef,
  ViewChild,
  ViewContainerRef,
  OnDestroy
} from '@angular/core';
import {
  Overlay,
  OverlayRef,
  ConnectedPosition
} from '@angular/cdk/overlay';
import { TemplatePortal } from '@angular/cdk/portal';
import { Subject } from 'rxjs';
import { takeUntil } from 'rxjs/operators';

export interface DropdownOption {
  value: any;
  label: string;
  icon?: string;
  disabled?: boolean;
}

@Component({
  selector: 'app-custom-dropdown',
  template: `
    <button
      #trigger
      class="dropdown-trigger"
      (click)="toggle()"
      [class.open]="isOpen"
    >
      <span>{{ selectedLabel || placeholder }}</span>
      <mat-icon>{{ isOpen ? 'expand_less' : 'expand_more' }}</mat-icon>
    </button>

    <ng-template #dropdownTemplate>
      <div class="dropdown-panel mat-elevation-z4">
        <div class="dropdown-search" *ngIf="searchable">
          <input
            [(ngModel)]="searchText"
            placeholder="ค้นหา..."
            (click)="$event.stopPropagation()"
          >
        </div>
        <ul class="dropdown-options">
          <li
            *ngFor="let option of filteredOptions"
            class="dropdown-option"
            [class.selected]="option.value === value"
            [class.disabled]="option.disabled"
            (click)="selectOption(option)"
          >
            <mat-icon *ngIf="option.icon">{{ option.icon }}</mat-icon>
            <span>{{ option.label }}</span>
            <mat-icon *ngIf="option.value === value" class="check">check</mat-icon>
          </li>
          <li *ngIf="filteredOptions.length === 0" class="no-options">
            ไม่พบข้อมูล
          </li>
        </ul>
      </div>
    </ng-template>
  `,
  styles: [`
    .dropdown-trigger {
      display: flex;
      align-items: center;
      gap: 8px;
      padding: 8px 12px;
      border: 1px solid #ddd;
      border-radius: 4px;
      background: white;
      cursor: pointer;
      min-width: 200px;
    }
    .dropdown-trigger.open { border-color: #1976d2; }
    .dropdown-panel {
      background: white;
      border-radius: 4px;
      min-width: 200px;
      max-height: 300px;
      overflow: hidden;
    }
    .dropdown-options { list-style: none; margin: 0; padding: 4px 0; overflow-y: auto; max-height: 240px; }
    .dropdown-option { display: flex; align-items: center; gap: 8px; padding: 8px 16px; cursor: pointer; }
    .dropdown-option:hover { background: #f5f5f5; }
    .dropdown-option.selected { background: #e3f2fd; color: #1976d2; }
    .dropdown-option.disabled { opacity: 0.5; pointer-events: none; }
    .dropdown-search { padding: 8px; border-bottom: 1px solid #eee; }
    .dropdown-search input { width: 100%; padding: 4px 8px; border: 1px solid #ddd; border-radius: 4px; }
    .no-options { padding: 16px; text-align: center; color: #999; }
  `]
})
export class CustomDropdownComponent implements OnDestroy {
  @Input() options: DropdownOption[] = [];
  @Input() value: any = null;
  @Input() placeholder = '-- เลือก --';
  @Input() searchable = false;
  @Output() valueChange = new EventEmitter<any>();

  @ViewChild('trigger') triggerRef!: ElementRef;
  @ViewChild('dropdownTemplate') dropdownTemplate!: TemplateRef<any>;

  isOpen = false;
  searchText = '';
  private overlayRef?: OverlayRef;
  private destroy$ = new Subject<void>();

  constructor(
    private overlay: Overlay,
    private viewContainerRef: ViewContainerRef
  ) {}

  get selectedLabel(): string {
    return this.options.find(o => o.value === this.value)?.label || '';
  }

  get filteredOptions(): DropdownOption[] {
    if (!this.searchText) return this.options;
    return this.options.filter(o =>
      o.label.toLowerCase().includes(this.searchText.toLowerCase())
    );
  }

  toggle() {
    this.isOpen ? this.close() : this.open();
  }

  open() {
    const positions: ConnectedPosition[] = [
      { originX: 'start', originY: 'bottom', overlayX: 'start', overlayY: 'top', offsetY: 4 },
      { originX: 'start', originY: 'top', overlayX: 'start', overlayY: 'bottom', offsetY: -4 }
    ];

    this.overlayRef = this.overlay.create({
      hasBackdrop: true,
      backdropClass: 'cdk-overlay-transparent-backdrop',
      positionStrategy: this.overlay.position()
        .flexibleConnectedTo(this.triggerRef)
        .withPositions(positions),
      minWidth: this.triggerRef.nativeElement.offsetWidth
    });

    const portal = new TemplatePortal(this.dropdownTemplate, this.viewContainerRef);
    this.overlayRef.attach(portal);
    this.overlayRef.backdropClick()
      .pipe(takeUntil(this.destroy$))
      .subscribe(() => this.close());

    this.isOpen = true;
  }

  close() {
    this.overlayRef?.detach();
    this.overlayRef?.dispose();
    this.isOpen = false;
    this.searchText = '';
  }

  selectOption(option: DropdownOption) {
    if (!option.disabled) {
      this.value = option.value;
      this.valueChange.emit(option.value);
      this.close();
    }
  }

  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
    this.overlayRef?.dispose();
  }
}
```

---

## 4. CDK Drag and Drop

```typescript
// app/components/kanban-board/kanban-board.component.ts
import { Component } from '@angular/core';
import {
  CdkDragDrop,
  moveItemInArray,
  transferArrayItem,
  CdkDrag,
  CdkDropList,
  CdkDropListGroup
} from '@angular/cdk/drag-drop';

interface Task {
  id: number;
  title: string;
  priority: 'low' | 'medium' | 'high';
  assignee: string;
}

interface Column {
  id: string;
  title: string;
  tasks: Task[];
  color: string;
}

@Component({
  selector: 'app-kanban-board',
  standalone: true,
  imports: [CdkDrag, CdkDropList, CdkDropListGroup],
  template: `
    <div class="kanban-board" cdkDropListGroup>
      <div
        *ngFor="let column of columns"
        class="kanban-column"
        [style.border-top-color]="column.color"
      >
        <div class="column-header" [style.color]="column.color">
          <h3>{{ column.title }}</h3>
          <span class="count">{{ column.tasks.length }}</span>
        </div>

        <div
          class="task-list"
          cdkDropList
          [cdkDropListData]="column.tasks"
          [id]="column.id"
          (cdkDropListDropped)="onDrop($event)"
        >
          <div
            *ngFor="let task of column.tasks"
            class="task-card"
            cdkDrag
            [cdkDragData]="task"
          >
            <!-- Drag preview -->
            <div class="task-drag-preview" *cdkDragPreview>
              <p>{{ task.title }}</p>
            </div>

            <!-- Placeholder -->
            <div class="task-placeholder" *cdkDragPlaceholder></div>

            <div class="task-content">
              <p class="task-title">{{ task.title }}</p>
              <div class="task-meta">
                <span class="priority" [class]="'priority-' + task.priority">
                  {{ task.priority }}
                </span>
                <span class="assignee">{{ task.assignee }}</span>
              </div>
            </div>
          </div>

          <!-- Empty state -->
          <div *ngIf="column.tasks.length === 0" class="empty-column">
            ลากงานมาวางที่นี่
          </div>
        </div>

        <button class="add-task" (click)="addTask(column)">+ เพิ่มงาน</button>
      </div>
    </div>
  `,
  styles: [`
    .kanban-board { display: flex; gap: 16px; padding: 16px; overflow-x: auto; }
    .kanban-column { width: 280px; min-width: 280px; background: #f5f5f5; border-radius: 8px; border-top: 4px solid; padding: 12px; }
    .column-header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 12px; }
    .count { background: #ddd; border-radius: 12px; padding: 2px 8px; font-size: 12px; }
    .task-list { min-height: 100px; }
    .task-card { background: white; border-radius: 6px; padding: 12px; margin-bottom: 8px; cursor: grab; box-shadow: 0 1px 3px rgba(0,0,0,0.1); }
    .task-card:active { cursor: grabbing; }
    .task-card.cdk-drag-animating { transition: transform 250ms; }
    .task-list.cdk-drop-list-dragging .task-card:not(.cdk-drag-placeholder) { transition: transform 250ms; }
    .task-drag-preview { background: white; border-radius: 6px; padding: 12px; box-shadow: 0 4px 12px rgba(0,0,0,0.2); }
    .task-placeholder { border: 2px dashed #ddd; border-radius: 6px; height: 60px; margin-bottom: 8px; }
    .task-title { margin: 0 0 8px; }
    .task-meta { display: flex; gap: 8px; font-size: 12px; }
    .priority { padding: 2px 6px; border-radius: 4px; }
    .priority-low { background: #e8f5e9; color: #2e7d32; }
    .priority-medium { background: #fff3e0; color: #e65100; }
    .priority-high { background: #ffebee; color: #c62828; }
    .empty-column { text-align: center; padding: 20px; color: #999; }
    .add-task { width: 100%; padding: 8px; background: none; border: 1px dashed #ddd; border-radius: 4px; cursor: pointer; margin-top: 8px; }
  `]
})
export class KanbanBoardComponent {
  columns: Column[] = [
    {
      id: 'todo',
      title: 'งานที่รอ',
      color: '#9e9e9e',
      tasks: [
        { id: 1, title: 'ออกแบบหน้า Login', priority: 'high', assignee: 'สมชาย' },
        { id: 2, title: 'เขียน API Documentation', priority: 'medium', assignee: 'สมหญิง' }
      ]
    },
    {
      id: 'in-progress',
      title: 'กำลังทำ',
      color: '#1976d2',
      tasks: [
        { id: 3, title: 'พัฒนา Dashboard', priority: 'high', assignee: 'สมชาย' }
      ]
    },
    {
      id: 'review',
      title: 'รอตรวจสอบ',
      color: '#f57c00',
      tasks: [
        { id: 4, title: 'Unit Tests สำหรับ Auth', priority: 'medium', assignee: 'วิชัย' }
      ]
    },
    {
      id: 'done',
      title: 'เสร็จแล้ว',
      color: '#388e3c',
      tasks: [
        { id: 5, title: 'Setup Project Structure', priority: 'low', assignee: 'สมชาย' }
      ]
    }
  ];

  onDrop(event: CdkDragDrop<Task[]>) {
    if (event.previousContainer === event.container) {
      moveItemInArray(event.container.data, event.previousIndex, event.currentIndex);
    } else {
      transferArrayItem(
        event.previousContainer.data,
        event.container.data,
        event.previousIndex,
        event.currentIndex
      );
    }
  }

  addTask(column: Column) {
    const title = prompt('ชื่องาน:');
    if (title) {
      column.tasks.push({
        id: Date.now(),
        title,
        priority: 'medium',
        assignee: 'ไม่ระบุ'
      });
    }
  }
}
```

---

## 5. CDK Virtual Scroll

```typescript
// app/components/virtual-list/virtual-list.component.ts
import { Component, OnInit } from '@angular/core';
import {
  ScrollingModule,
  FixedSizeVirtualScrollStrategy,
  VIRTUAL_SCROLL_STRATEGY
} from '@angular/cdk/scrolling';

interface ListItem {
  id: number;
  name: string;
  email: string;
  status: 'active' | 'inactive';
}

@Component({
  selector: 'app-virtual-list',
  standalone: true,
  imports: [ScrollingModule],
  template: `
    <div class="virtual-scroll-demo">
      <h3>Virtual Scroll Demo ({{ items.length }} รายการ)</h3>
      <p>ใช้ Virtual Scroll เพื่อ render เฉพาะ items ที่มองเห็นได้</p>

      <cdk-virtual-scroll-viewport
        itemSize="72"
        class="scroll-viewport"
      >
        <div
          *cdkVirtualFor="let item of items; trackBy: trackItem"
          class="list-item"
          [class.inactive]="item.status === 'inactive'"
        >
          <div class="item-avatar">{{ item.name[0] }}</div>
          <div class="item-info">
            <p class="item-name">{{ item.name }}</p>
            <p class="item-email">{{ item.email }}</p>
          </div>
          <span class="item-status" [class]="'status-' + item.status">
            {{ item.status === 'active' ? 'ใช้งาน' : 'ปิดใช้' }}
          </span>
        </div>
      </cdk-virtual-scroll-viewport>
    </div>
  `,
  styles: [`
    .virtual-scroll-demo { padding: 16px; }
    .scroll-viewport { height: 400px; border: 1px solid #ddd; border-radius: 4px; }
    .list-item {
      display: flex;
      align-items: center;
      gap: 12px;
      padding: 12px 16px;
      border-bottom: 1px solid #eee;
      height: 72px;
      box-sizing: border-box;
    }
    .list-item.inactive { opacity: 0.6; }
    .item-avatar {
      width: 40px;
      height: 40px;
      border-radius: 50%;
      background: #1976d2;
      color: white;
      display: flex;
      align-items: center;
      justify-content: center;
      font-weight: bold;
    }
    .item-info { flex: 1; }
    .item-name { margin: 0; font-weight: 500; }
    .item-email { margin: 0; font-size: 12px; color: #666; }
    .status-active { color: green; font-size: 12px; }
    .status-inactive { color: red; font-size: 12px; }
  `]
})
export class VirtualListComponent implements OnInit {
  items: ListItem[] = [];

  ngOnInit() {
    // สร้างข้อมูล 10,000 รายการ
    this.items = Array.from({ length: 10000 }, (_, i) => ({
      id: i + 1,
      name: `ผู้ใช้ ${i + 1}`,
      email: `user${i + 1}@example.com`,
      status: i % 3 === 0 ? 'inactive' : 'active'
    }));
  }

  trackItem(index: number, item: ListItem): number {
    return item.id;
  }
}
```

---

## 6. CDK Clipboard

```typescript
// app/components/clipboard-demo/clipboard-demo.component.ts
import { Component } from '@angular/core';
import { Clipboard } from '@angular/cdk/clipboard';

@Component({
  selector: 'app-clipboard-demo',
  template: `
    <div class="clipboard-demo">
      <h3>Clipboard Demo</h3>

      <!-- Copy สั้น ๆ -->
      <div class="copy-item">
        <span>{{ shortText }}</span>
        <button
          [cdkCopyToClipboard]="shortText"
          (cdkCopyToClipboardCopied)="onCopied($event)"
        >
          {{ copied ? '✓ คัดลอกแล้ว' : 'คัดลอก' }}
        </button>
      </div>

      <!-- Copy code block -->
      <div class="code-block">
        <pre><code>{{ codeExample }}</code></pre>
        <button (click)="copyCode()">คัดลอก Code</button>
      </div>
    </div>
  `,
  styles: [`
    .copy-item { display: flex; gap: 12px; align-items: center; padding: 12px; border: 1px solid #ddd; border-radius: 4px; margin-bottom: 12px; }
    .code-block { position: relative; background: #1e1e1e; color: #d4d4d4; border-radius: 4px; padding: 16px; }
    .code-block button { position: absolute; top: 8px; right: 8px; }
  `]
})
export class ClipboardDemoComponent {
  shortText = 'https://angular.io';
  copied = false;
  codeExample = `const greeting = 'สวัสดี Angular CDK!';
console.log(greeting);`;

  constructor(private clipboard: Clipboard) {}

  onCopied(success: boolean) {
    this.copied = success;
    if (success) {
      setTimeout(() => this.copied = false, 2000);
    }
  }

  copyCode() {
    const success = this.clipboard.copy(this.codeExample);
    if (success) {
      alert('คัดลอก code แล้ว!');
    }
  }
}
```

---

## สรุป

| CDK Feature | ใช้เมื่อ |
|------------|---------|
| Overlay | สร้าง popup, dropdown, tooltip เอง |
| DragDrop | Kanban board, sortable list, drag-to-reorder |
| Virtual Scroll | List ขนาดใหญ่ (1000+ items) |
| Clipboard | Copy text, code snippets |

CDK เป็น foundation ที่ Angular Material ใช้สร้าง components ของตัวเอง การเรียนรู้ CDK ช่วยให้สร้าง custom components ที่ซับซ้อนได้อย่างมีประสิทธิภาพ
