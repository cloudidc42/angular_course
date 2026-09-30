# Part 33 — Dynamic Components

## Dynamic Components คืออะไร?

Dynamic Components คือการสร้างและจัดการ Components แบบ Programmatic ที่ Runtime แทนที่จะเขียนใน Template โดยตรง มีประโยชน์สำหรับ:

- Modal / Dialog Systems
- Toast Notifications
- Dynamic Wizards / Multi-step Forms
- Plugin-based UIs
- Tab Management

---

## ViewContainerRef

`ViewContainerRef` คือ Reference ไปยัง Container ที่สามารถ Insert Components ได้

```typescript
import { Component, ViewChild, ViewContainerRef } from '@angular/core';

@Component({
  selector: 'app-host',
  template: `
    <div>
      <!-- ng-container ไม่สร้าง DOM Element จริง -->
      <ng-container #dynamicHost></ng-container>
    </div>
  `
})
export class HostComponent {
  // @ViewChild ดึง ViewContainerRef ของ ng-container
  @ViewChild('dynamicHost', { read: ViewContainerRef })
  dynamicHost!: ViewContainerRef;
}
```

---

## createComponent()

```typescript
import { Component, ViewChild, ViewContainerRef, ComponentRef } from '@angular/core';
import { AlertComponent } from './alert.component';

@Component({
  selector: 'app-host',
  template: `
    <button (click)="showAlert()">แสดง Alert</button>
    <ng-container #alertContainer></ng-container>
  `
})
export class HostComponent {
  @ViewChild('alertContainer', { read: ViewContainerRef })
  container!: ViewContainerRef;

  private alertRef: ComponentRef<AlertComponent> | null = null;

  showAlert(): void {
    // ล้าง Container ก่อน
    this.container.clear();

    // สร้าง Component
    this.alertRef = this.container.createComponent(AlertComponent);

    // กำหนด Inputs
    this.alertRef.setInput('message', 'นี่คือ Alert Message!');
    this.alertRef.setInput('type', 'success');

    // รับ Output Events
    this.alertRef.instance.closed.subscribe(() => {
      this.closeAlert();
    });
  }

  closeAlert(): void {
    this.alertRef?.destroy();
    this.alertRef = null;
  }
}
```

---

## Alert Component

```typescript
// alert.component.ts
import { Component, Input, Output, EventEmitter } from '@angular/core';
import { CommonModule } from '@angular/common';

@Component({
  selector: 'app-alert',
  standalone: true,
  imports: [CommonModule],
  template: `
    <div class="alert" [class]="'alert-' + type">
      <span class="icon">{{ icon }}</span>
      <span class="message">{{ message }}</span>
      <button class="close" (click)="close()">✕</button>
    </div>
  `,
  styles: [`
    .alert {
      display: flex;
      align-items: center;
      gap: 8px;
      padding: 12px 16px;
      border-radius: 4px;
      margin: 8px 0;
    }
    .alert-success { background: #e8f5e9; border-left: 4px solid #4caf50; color: #2e7d32; }
    .alert-error { background: #ffebee; border-left: 4px solid #f44336; color: #c62828; }
    .alert-warning { background: #fff3e0; border-left: 4px solid #ff9800; color: #e65100; }
    .alert-info { background: #e3f2fd; border-left: 4px solid #2196f3; color: #0d47a1; }
    .close { margin-left: auto; background: none; border: none; cursor: pointer; font-size: 16px; }
  `]
})
export class AlertComponent {
  @Input() message = '';
  @Input() type: 'success' | 'error' | 'warning' | 'info' = 'info';
  @Output() closed = new EventEmitter<void>();

  get icon(): string {
    const icons: Record<string, string> = {
      success: '✅',
      error: '❌',
      warning: '⚠️',
      info: 'ℹ️',
    };
    return icons[this.type] || 'ℹ️';
  }

  close(): void {
    this.closed.emit();
  }
}
```

---

## Workshop: Toast Notification System

ระบบ Toast Notification แบบสมบูรณ์ที่ใช้ Dynamic Components

### Toast Model

```typescript
// toast.model.ts
export type ToastType = 'success' | 'error' | 'warning' | 'info';

export interface Toast {
  id: string;
  message: string;
  title?: string;
  type: ToastType;
  duration: number;   // milliseconds (0 = ไม่ปิดอัตโนมัติ)
  createdAt: Date;
}
```

### Toast Component

```typescript
// toast.component.ts
import {
  Component,
  Input,
  Output,
  EventEmitter,
  OnInit,
  OnDestroy,
  ChangeDetectorRef,
} from '@angular/core';
import { CommonModule } from '@angular/common';
import { Toast } from './toast.model';
import {
  trigger,
  state,
  style,
  transition,
  animate,
} from '@angular/animations';

@Component({
  selector: 'app-toast',
  standalone: true,
  imports: [CommonModule],
  template: `
    <div
      class="toast"
      [class]="'toast-' + toast.type"
      [@toastState]="animationState"
      (@toastState.done)="onAnimationDone($event)"
    >
      <div class="toast-icon">{{ icon }}</div>
      <div class="toast-content">
        <div class="toast-title" *ngIf="toast.title">{{ toast.title }}</div>
        <div class="toast-message">{{ toast.message }}</div>
      </div>
      <button class="toast-close" (click)="dismiss()">✕</button>
      <div
        class="toast-progress"
        *ngIf="toast.duration > 0"
        [style.animation-duration.ms]="toast.duration"
      ></div>
    </div>
  `,
  styles: [`
    .toast {
      display: flex;
      align-items: flex-start;
      gap: 12px;
      padding: 12px 16px;
      border-radius: 8px;
      box-shadow: 0 4px 12px rgba(0,0,0,0.15);
      min-width: 300px;
      max-width: 450px;
      position: relative;
      overflow: hidden;
    }
    .toast-success { background: #fff; border-left: 4px solid #4caf50; }
    .toast-error { background: #fff; border-left: 4px solid #f44336; }
    .toast-warning { background: #fff; border-left: 4px solid #ff9800; }
    .toast-info { background: #fff; border-left: 4px solid #2196f3; }
    .toast-title { font-weight: bold; margin-bottom: 4px; }
    .toast-message { font-size: 14px; color: #555; }
    .toast-close {
      margin-left: auto;
      background: none;
      border: none;
      cursor: pointer;
      color: #999;
      padding: 0 4px;
    }
    .toast-progress {
      position: absolute;
      bottom: 0;
      left: 0;
      height: 3px;
      background: rgba(0,0,0,0.2);
      animation: progress linear forwards;
      animation-play-state: running;
    }
    @keyframes progress {
      from { width: 100%; }
      to { width: 0%; }
    }
  `],
  animations: [
    trigger('toastState', [
      state('visible', style({ opacity: 1, transform: 'translateX(0)' })),
      state('hidden', style({ opacity: 0, transform: 'translateX(100%)' })),
      transition('void => visible', [
        style({ opacity: 0, transform: 'translateX(100%)' }),
        animate('300ms ease-out'),
      ]),
      transition('visible => hidden', [
        animate('200ms ease-in', style({ opacity: 0, transform: 'translateX(100%)' })),
      ]),
    ]),
  ],
})
export class ToastComponent implements OnInit, OnDestroy {
  @Input() toast!: Toast;
  @Output() dismissed = new EventEmitter<string>();

  animationState: 'visible' | 'hidden' = 'visible';
  private timer: ReturnType<typeof setTimeout> | null = null;

  get icon(): string {
    const icons: Record<string, string> = {
      success: '✅',
      error: '❌',
      warning: '⚠️',
      info: 'ℹ️',
    };
    return icons[this.toast.type] || 'ℹ️';
  }

  constructor(private cdr: ChangeDetectorRef) {}

  ngOnInit(): void {
    if (this.toast.duration > 0) {
      this.timer = setTimeout(() => this.dismiss(), this.toast.duration);
    }
  }

  ngOnDestroy(): void {
    if (this.timer) clearTimeout(this.timer);
  }

  dismiss(): void {
    this.animationState = 'hidden';
  }

  onAnimationDone(event: any): void {
    if (event.toState === 'hidden') {
      this.dismissed.emit(this.toast.id);
    }
  }
}
```

### Toast Service

```typescript
// toast.service.ts
import {
  Injectable,
  ApplicationRef,
  ComponentRef,
  createComponent,
  EnvironmentInjector,
} from '@angular/core';
import { Toast, ToastType } from './toast.model';
import { ToastContainerComponent } from './toast-container.component';

@Injectable({ providedIn: 'root' })
export class ToastService {
  private containerRef: ComponentRef<ToastContainerComponent> | null = null;

  constructor(
    private appRef: ApplicationRef,
    private injector: EnvironmentInjector
  ) {}

  private getOrCreateContainer(): ToastContainerComponent {
    if (!this.containerRef) {
      // สร้าง Container ที่ body
      this.containerRef = createComponent(ToastContainerComponent, {
        environmentInjector: this.injector,
      });

      // Attach ไปยัง ApplicationRef เพื่อให้ Change Detection ทำงาน
      this.appRef.attachView(this.containerRef.hostView);

      // Append ไปยัง body
      document.body.appendChild(
        (this.containerRef.hostView as any).rootNodes[0]
      );
    }
    return this.containerRef.instance;
  }

  show(options: Partial<Toast> & { message: string }): void {
    const toast: Toast = {
      id: `toast-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`,
      type: 'info',
      duration: 5000,
      createdAt: new Date(),
      ...options,
    };

    this.getOrCreateContainer().addToast(toast);
  }

  success(message: string, title?: string, duration = 4000): void {
    this.show({ message, title, type: 'success', duration });
  }

  error(message: string, title?: string, duration = 0): void {
    this.show({ message, title: title || 'เกิดข้อผิดพลาด', type: 'error', duration });
  }

  warning(message: string, title?: string, duration = 5000): void {
    this.show({ message, title, type: 'warning', duration });
  }

  info(message: string, title?: string, duration = 4000): void {
    this.show({ message, title, type: 'info', duration });
  }

  clear(): void {
    this.containerRef?.instance.clearAll();
  }
}
```

### Toast Container Component

```typescript
// toast-container.component.ts
import {
  Component,
  ViewChild,
  ViewContainerRef,
  ComponentRef,
  OnDestroy,
} from '@angular/core';
import { Toast } from './toast.model';
import { ToastComponent } from './toast.component';

@Component({
  selector: 'app-toast-container',
  standalone: true,
  imports: [ToastComponent],
  template: `
    <div class="toast-container">
      <ng-container #toastHost></ng-container>
    </div>
  `,
  styles: [`
    .toast-container {
      position: fixed;
      top: 16px;
      right: 16px;
      z-index: 9999;
      display: flex;
      flex-direction: column;
      gap: 8px;
      pointer-events: none;
    }
    .toast-container > * {
      pointer-events: all;
    }
  `]
})
export class ToastContainerComponent implements OnDestroy {
  @ViewChild('toastHost', { read: ViewContainerRef })
  toastHost!: ViewContainerRef;

  private toastRefs = new Map<string, ComponentRef<ToastComponent>>();

  addToast(toast: Toast): void {
    const ref = this.toastHost.createComponent(ToastComponent);
    ref.setInput('toast', toast);

    ref.instance.dismissed.subscribe((id: string) => {
      this.removeToast(id);
    });

    this.toastRefs.set(toast.id, ref);
    ref.changeDetectorRef.detectChanges();
  }

  removeToast(id: string): void {
    const ref = this.toastRefs.get(id);
    if (ref) {
      ref.destroy();
      this.toastRefs.delete(id);
    }
  }

  clearAll(): void {
    this.toastRefs.forEach((ref) => ref.destroy());
    this.toastRefs.clear();
  }

  ngOnDestroy(): void {
    this.clearAll();
  }
}
```

### การใช้งาน Toast Service

```typescript
// any.component.ts
import { Component, inject } from '@angular/core';
import { ToastService } from '../toast/toast.service';

@Component({
  selector: 'app-demo',
  standalone: true,
  template: `
    <div class="demo">
      <h2>ทดสอบ Toast Notifications</h2>
      <button (click)="showSuccess()">Success</button>
      <button (click)="showError()">Error</button>
      <button (click)="showWarning()">Warning</button>
      <button (click)="showInfo()">Info</button>
      <button (click)="showCustom()">Custom</button>
      <button (click)="clearAll()">Clear All</button>
    </div>
  `
})
export class DemoComponent {
  private toast = inject(ToastService);

  showSuccess(): void {
    this.toast.success('บันทึกข้อมูลสำเร็จ', 'สำเร็จ');
  }

  showError(): void {
    this.toast.error('ไม่สามารถเชื่อมต่อเซิร์ฟเวอร์ได้', 'ข้อผิดพลาด');
  }

  showWarning(): void {
    this.toast.warning('session ของคุณใกล้หมดอายุ', 'คำเตือน');
  }

  showInfo(): void {
    this.toast.info('มีการอัปเดตระบบใหม่');
  }

  showCustom(): void {
    this.toast.show({
      message: 'Toast แบบกำหนดเอง ไม่ปิดอัตโนมัติ',
      title: 'Custom Toast',
      type: 'warning',
      duration: 0,  // ไม่ปิดเอง
    });
  }

  clearAll(): void {
    this.toast.clear();
  }
}
```

---

## Dynamic Modal System

```typescript
// modal.service.ts
import { Injectable, Type, createComponent, ApplicationRef, EnvironmentInjector } from '@angular/core';
import { Subject } from 'rxjs';

export interface ModalConfig<T = any> {
  component: Type<T>;
  data?: Partial<T>;
  closeable?: boolean;
}

export interface ModalResult<T = any> {
  confirmed: boolean;
  data?: T;
}

@Injectable({ providedIn: 'root' })
export class ModalService {
  private modalStack: any[] = [];

  constructor(
    private appRef: ApplicationRef,
    private injector: EnvironmentInjector
  ) {}

  open<T, R = any>(config: ModalConfig<T>): Promise<ModalResult<R>> {
    return new Promise((resolve) => {
      const overlayEl = document.createElement('div');
      overlayEl.className = 'modal-overlay';
      document.body.appendChild(overlayEl);
      document.body.style.overflow = 'hidden';

      const wrapperComponent = createComponent(ModalWrapperComponent, {
        environmentInjector: this.injector,
      });

      this.appRef.attachView(wrapperComponent.hostView);
      overlayEl.appendChild((wrapperComponent.hostView as any).rootNodes[0]);

      const contentRef = wrapperComponent.instance.viewContainer.createComponent(config.component);

      if (config.data) {
        Object.entries(config.data).forEach(([key, value]) => {
          contentRef.setInput(key, value);
        });
      }

      const cleanup = (result: ModalResult<R>) => {
        wrapperComponent.destroy();
        overlayEl.remove();
        document.body.style.overflow = '';
        resolve(result);
      };

      wrapperComponent.instance.closed.subscribe(cleanup);
    });
  }
}
```

---

## สรุป

| API | การใช้งาน |
|----|-----------|
| `ViewContainerRef.createComponent(Type)` | สร้าง Component ใน Container |
| `ComponentRef.setInput(key, value)` | กำหนด Input ให้ Dynamic Component |
| `ComponentRef.instance` | เข้าถึง Component Instance |
| `ComponentRef.destroy()` | ทำลาย Component |
| `createComponent(Type, {environmentInjector})` | สร้าง Component นอก Template |
| `ApplicationRef.attachView(view)` | Attach View ไปยัง App |

Dynamic Components เหมาะมากสำหรับ Toast, Modal, Tooltip และ UI Components ที่ต้องการสร้างแบบ Programmatic ใน Part ถัดไปจะเรียน Content Projection
