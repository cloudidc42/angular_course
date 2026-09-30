# Part 71: Accessibility ใน Angular (ARIA, Keyboard Navigation, Screen Reader)

## ทำไม Accessibility ถึงสำคัญ

Accessibility (a11y) คือการทำให้แอปพลิเคชันใช้งานได้สำหรับทุกคน รวมถึงผู้พิการทางสายตา การได้ยิน หรือการเคลื่อนไหว ในหลายประเทศมีกฎหมายบังคับให้เว็บไซต์ต้องผ่านมาตรฐาน WCAG 2.1

### มาตรฐาน WCAG 2.1

- **Level A**: ขั้นต่ำสุด
- **Level AA**: มาตรฐานทั่วไปที่ใช้กัน
- **Level AAA**: สูงสุด

---

## 1. ARIA Roles และ Attributes

### ARIA คืออะไร

ARIA (Accessible Rich Internet Applications) คือชุดของ attributes ที่เพิ่มความหมายให้กับ HTML elements

```typescript
// app/components/alert/alert.component.ts
import { Component, Input } from '@angular/core';

@Component({
  selector: 'app-alert',
  template: `
    <div 
      role="alert"
      [attr.aria-live]="live"
      [attr.aria-atomic]="true"
      [class]="'alert alert-' + type"
    >
      <span class="sr-only">{{ srPrefix }}</span>
      {{ message }}
    </div>
  `,
  styles: [`
    .sr-only {
      position: absolute;
      width: 1px;
      height: 1px;
      padding: 0;
      margin: -1px;
      overflow: hidden;
      clip: rect(0, 0, 0, 0);
      white-space: nowrap;
      border: 0;
    }
  `]
})
export class AlertComponent {
  @Input() message = '';
  @Input() type: 'success' | 'warning' | 'error' | 'info' = 'info';
  @Input() live: 'polite' | 'assertive' | 'off' = 'polite';

  get srPrefix(): string {
    const prefixes = {
      success: 'สำเร็จ:',
      warning: 'คำเตือน:',
      error: 'ข้อผิดพลาด:',
      info: 'ข้อมูล:'
    };
    return prefixes[this.type];
  }
}
```

### Dialog Component พร้อม ARIA

```typescript
// app/components/dialog/dialog.component.ts
import { Component, Input, Output, EventEmitter, ElementRef, AfterViewInit, OnDestroy } from '@angular/core';

@Component({
  selector: 'app-dialog',
  template: `
    <div 
      class="dialog-overlay"
      [attr.aria-hidden]="!isOpen"
      (click)="onOverlayClick($event)"
    >
      <div 
        role="dialog"
        [attr.aria-modal]="true"
        [attr.aria-labelledby]="titleId"
        [attr.aria-describedby]="descId"
        class="dialog-content"
        #dialogContent
      >
        <h2 [id]="titleId">{{ title }}</h2>
        <div [id]="descId">
          <ng-content></ng-content>
        </div>
        <div class="dialog-actions">
          <button 
            (click)="confirm.emit()"
            class="btn-primary"
          >
            ยืนยัน
          </button>
          <button 
            (click)="cancel.emit()"
            class="btn-secondary"
            #closeBtn
          >
            ยกเลิก
          </button>
        </div>
      </div>
    </div>
  `
})
export class DialogComponent implements AfterViewInit, OnDestroy {
  @Input() title = '';
  @Input() isOpen = false;
  @Output() confirm = new EventEmitter<void>();
  @Output() cancel = new EventEmitter<void>();

  titleId = `dialog-title-${Math.random().toString(36).slice(2)}`;
  descId = `dialog-desc-${Math.random().toString(36).slice(2)}`;

  private previousFocus: HTMLElement | null = null;

  ngAfterViewInit(): void {
    if (this.isOpen) {
      this.trapFocus();
    }
  }

  private trapFocus(): void {
    // บันทึก focus ก่อนหน้า
    this.previousFocus = document.activeElement as HTMLElement;
    
    // Focus ไปที่ dialog
    const dialog = document.querySelector('[role="dialog"]') as HTMLElement;
    if (dialog) {
      const focusable = dialog.querySelectorAll<HTMLElement>(
        'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
      );
      if (focusable.length > 0) {
        focusable[0].focus();
      }
    }
  }

  onOverlayClick(event: MouseEvent): void {
    if ((event.target as HTMLElement).classList.contains('dialog-overlay')) {
      this.cancel.emit();
    }
  }

  ngOnDestroy(): void {
    // คืน focus กลับ
    if (this.previousFocus) {
      this.previousFocus.focus();
    }
  }
}
```

---

## 2. Keyboard Navigation

### Focus Management Service

```typescript
// app/services/focus.service.ts
import { Injectable } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class FocusService {
  
  // เก็บ focus trap stack
  private trapStack: HTMLElement[] = [];

  trapFocus(container: HTMLElement): void {
    this.trapStack.push(container);
    this.setFocusTrap(container);
  }

  releaseFocus(): void {
    this.trapStack.pop();
    if (this.trapStack.length > 0) {
      this.setFocusTrap(this.trapStack[this.trapStack.length - 1]);
    }
  }

  private setFocusTrap(container: HTMLElement): void {
    const focusableElements = this.getFocusableElements(container);
    
    if (focusableElements.length === 0) return;

    const firstElement = focusableElements[0];
    const lastElement = focusableElements[focusableElements.length - 1];

    container.addEventListener('keydown', (e: KeyboardEvent) => {
      if (e.key !== 'Tab') return;

      if (e.shiftKey) {
        if (document.activeElement === firstElement) {
          e.preventDefault();
          lastElement.focus();
        }
      } else {
        if (document.activeElement === lastElement) {
          e.preventDefault();
          firstElement.focus();
        }
      }
    });

    firstElement.focus();
  }

  private getFocusableElements(container: HTMLElement): HTMLElement[] {
    const selector = [
      'a[href]',
      'button:not([disabled])',
      'input:not([disabled])',
      'select:not([disabled])',
      'textarea:not([disabled])',
      '[tabindex]:not([tabindex="-1"])',
      '[contenteditable="true"]'
    ].join(', ');

    return Array.from(container.querySelectorAll<HTMLElement>(selector))
      .filter(el => !el.closest('[hidden]') && !el.closest('[aria-hidden="true"]'));
  }

  moveFocus(direction: 'next' | 'prev', container?: HTMLElement): void {
    const root = container || document.body;
    const focusable = this.getFocusableElements(root);
    const currentIndex = focusable.indexOf(document.activeElement as HTMLElement);

    if (currentIndex === -1) {
      focusable[0]?.focus();
      return;
    }

    let nextIndex: number;
    if (direction === 'next') {
      nextIndex = (currentIndex + 1) % focusable.length;
    } else {
      nextIndex = (currentIndex - 1 + focusable.length) % focusable.length;
    }

    focusable[nextIndex]?.focus();
  }
}
```

### Keyboard Shortcut Directive

```typescript
// app/directives/keyboard-shortcut.directive.ts
import { Directive, Input, HostListener, OnInit, OnDestroy } from '@angular/core';

interface ShortcutConfig {
  key: string;
  ctrl?: boolean;
  alt?: boolean;
  shift?: boolean;
  action: () => void;
  description: string;
}

@Directive({
  selector: '[appKeyboardShortcut]'
})
export class KeyboardShortcutDirective implements OnInit, OnDestroy {
  @Input('appKeyboardShortcut') shortcuts: ShortcutConfig[] = [];

  @HostListener('document:keydown', ['$event'])
  handleKeydown(event: KeyboardEvent): void {
    for (const shortcut of this.shortcuts) {
      const keyMatch = event.key.toLowerCase() === shortcut.key.toLowerCase();
      const ctrlMatch = !shortcut.ctrl || event.ctrlKey;
      const altMatch = !shortcut.alt || event.altKey;
      const shiftMatch = !shortcut.shift || event.shiftKey;

      if (keyMatch && ctrlMatch && altMatch && shiftMatch) {
        event.preventDefault();
        shortcut.action();
        break;
      }
    }
  }

  ngOnInit(): void {
    // ลงทะเบียน shortcuts สำหรับ screen reader announcement
    console.log('Keyboard shortcuts registered:', 
      this.shortcuts.map(s => s.description));
  }

  ngOnDestroy(): void {
    // cleanup
  }
}
```

### Navigation Component พร้อม Skip Link

```typescript
// app/components/skip-link/skip-link.component.ts
import { Component } from '@angular/core';

@Component({
  selector: 'app-skip-link',
  template: `
    <a 
      href="#main-content" 
      class="skip-link"
      (click)="skipToMain($event)"
    >
      ข้ามไปยังเนื้อหาหลัก
    </a>
  `,
  styles: [`
    .skip-link {
      position: absolute;
      top: -40px;
      left: 0;
      background: #000;
      color: #fff;
      padding: 8px;
      z-index: 9999;
      transition: top 0.3s;
    }
    .skip-link:focus {
      top: 0;
    }
  `]
})
export class SkipLinkComponent {
  skipToMain(event: Event): void {
    event.preventDefault();
    const mainContent = document.getElementById('main-content');
    if (mainContent) {
      mainContent.setAttribute('tabindex', '-1');
      mainContent.focus();
      // ลบ tabindex หลัง focus
      mainContent.addEventListener('blur', () => {
        mainContent.removeAttribute('tabindex');
      }, { once: true });
    }
  }
}
```

---

## 3. Screen Reader Support

### Live Region Service

```typescript
// app/services/live-region.service.ts
import { Injectable } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class LiveRegionService {
  private politeRegion: HTMLElement;
  private assertiveRegion: HTMLElement;

  constructor() {
    this.politeRegion = this.createRegion('polite');
    this.assertiveRegion = this.createRegion('assertive');
  }

  private createRegion(type: 'polite' | 'assertive'): HTMLElement {
    const region = document.createElement('div');
    region.setAttribute('aria-live', type);
    region.setAttribute('aria-atomic', 'true');
    region.setAttribute('role', 'status');
    region.style.cssText = `
      position: absolute;
      width: 1px;
      height: 1px;
      padding: 0;
      overflow: hidden;
      clip: rect(0, 0, 0, 0);
      white-space: nowrap;
      border: 0;
    `;
    document.body.appendChild(region);
    return region;
  }

  // แจ้ง screen reader แบบ polite (รอก่อน)
  announce(message: string): void {
    this.politeRegion.textContent = '';
    setTimeout(() => {
      this.politeRegion.textContent = message;
    }, 100);
  }

  // แจ้ง screen reader ทันที (สำคัญมาก)
  announceAssertive(message: string): void {
    this.assertiveRegion.textContent = '';
    setTimeout(() => {
      this.assertiveRegion.textContent = message;
    }, 100);
  }
}
```

### Accessible Table Component

```typescript
// app/components/accessible-table/accessible-table.component.ts
import { Component, Input } from '@angular/core';

interface Column {
  key: string;
  label: string;
  sortable?: boolean;
}

@Component({
  selector: 'app-accessible-table',
  template: `
    <div role="region" [attr.aria-label]="tableLabel">
      <table 
        [attr.aria-rowcount]="data.length + 1"
        [attr.aria-colcount]="columns.length"
      >
        <caption>{{ caption }}</caption>
        <thead>
          <tr>
            <th 
              *ngFor="let col of columns; let i = index"
              scope="col"
              [attr.aria-sort]="getSortState(col.key)"
              [attr.aria-colindex]="i + 1"
            >
              <button 
                *ngIf="col.sortable"
                (click)="sort(col.key)"
                [attr.aria-label]="'เรียงตาม ' + col.label"
              >
                {{ col.label }}
                <span aria-hidden="true">{{ getSortIcon(col.key) }}</span>
              </button>
              <span *ngIf="!col.sortable">{{ col.label }}</span>
            </th>
          </tr>
        </thead>
        <tbody>
          <tr 
            *ngFor="let row of data; let i = index"
            [attr.aria-rowindex]="i + 2"
          >
            <td 
              *ngFor="let col of columns; let j = index"
              [attr.aria-colindex]="j + 1"
            >
              {{ row[col.key] }}
            </td>
          </tr>
        </tbody>
      </table>
      
      <div role="status" aria-live="polite" class="sr-only">
        {{ statusMessage }}
      </div>
    </div>
  `
})
export class AccessibleTableComponent {
  @Input() columns: Column[] = [];
  @Input() data: any[] = [];
  @Input() caption = '';
  @Input() tableLabel = '';

  sortKey = '';
  sortDir: 'asc' | 'desc' | null = null;
  statusMessage = '';

  getSortState(key: string): string {
    if (this.sortKey !== key) return 'none';
    return this.sortDir === 'asc' ? 'ascending' : 'descending';
  }

  getSortIcon(key: string): string {
    if (this.sortKey !== key) return '⇅';
    return this.sortDir === 'asc' ? '↑' : '↓';
  }

  sort(key: string): void {
    if (this.sortKey === key) {
      this.sortDir = this.sortDir === 'asc' ? 'desc' : 'asc';
    } else {
      this.sortKey = key;
      this.sortDir = 'asc';
    }

    this.data = [...this.data].sort((a, b) => {
      const dir = this.sortDir === 'asc' ? 1 : -1;
      return a[key] > b[key] ? dir : -dir;
    });

    const col = this.columns.find(c => c.key === key);
    this.statusMessage = `เรียงตาม ${col?.label} ${this.sortDir === 'asc' ? 'น้อยไปมาก' : 'มากไปน้อย'}`;
  }
}
```

---

## 4. Forms Accessibility

### Accessible Form Component

```typescript
// app/components/accessible-form/accessible-form.component.ts
import { Component, OnInit } from '@angular/core';
import { FormBuilder, FormGroup, Validators } from '@angular/forms';

@Component({
  selector: 'app-accessible-form',
  template: `
    <form 
      [formGroup]="form" 
      (ngSubmit)="onSubmit()"
      novalidate
      aria-label="แบบฟอร์มลงทะเบียน"
    >
      <div class="form-group">
        <label for="fullname" id="fullname-label">
          ชื่อ-นามสกุล
          <span aria-hidden="true" class="required">*</span>
          <span class="sr-only">จำเป็นต้องกรอก</span>
        </label>
        <input 
          id="fullname"
          type="text"
          formControlName="fullname"
          aria-labelledby="fullname-label"
          [attr.aria-describedby]="getDescribedBy('fullname')"
          [attr.aria-invalid]="isInvalid('fullname')"
          [attr.aria-required]="true"
          autocomplete="name"
        >
        <div 
          *ngIf="isInvalid('fullname')"
          [id]="'fullname-error'"
          role="alert"
          class="error-message"
        >
          {{ getError('fullname') }}
        </div>
        <div [id]="'fullname-hint'" class="hint">
          กรอกชื่อและนามสกุลจริง
        </div>
      </div>

      <div class="form-group">
        <label for="email" id="email-label">
          อีเมล
          <span aria-hidden="true" class="required">*</span>
        </label>
        <input 
          id="email"
          type="email"
          formControlName="email"
          [attr.aria-describedby]="getDescribedBy('email')"
          [attr.aria-invalid]="isInvalid('email')"
          aria-required="true"
          autocomplete="email"
        >
        <div 
          *ngIf="isInvalid('email')"
          [id]="'email-error'"
          role="alert"
          class="error-message"
        >
          {{ getError('email') }}
        </div>
      </div>

      <fieldset>
        <legend>เพศ</legend>
        <div class="radio-group">
          <input 
            type="radio" 
            id="male" 
            formControlName="gender" 
            value="male"
          >
          <label for="male">ชาย</label>
        </div>
        <div class="radio-group">
          <input 
            type="radio" 
            id="female" 
            formControlName="gender" 
            value="female"
          >
          <label for="female">หญิง</label>
        </div>
      </fieldset>

      <button 
        type="submit"
        [disabled]="form.invalid"
        [attr.aria-disabled]="form.invalid"
      >
        ลงทะเบียน
      </button>
    </form>
  `
})
export class AccessibleFormComponent implements OnInit {
  form!: FormGroup;

  constructor(private fb: FormBuilder) {}

  ngOnInit(): void {
    this.form = this.fb.group({
      fullname: ['', [Validators.required, Validators.minLength(3)]],
      email: ['', [Validators.required, Validators.email]],
      gender: ['', Validators.required]
    });
  }

  isInvalid(field: string): boolean {
    const control = this.form.get(field);
    return !!(control?.invalid && control?.touched);
  }

  getDescribedBy(field: string): string {
    const ids = [`${field}-hint`];
    if (this.isInvalid(field)) ids.push(`${field}-error`);
    return ids.join(' ');
  }

  getError(field: string): string {
    const control = this.form.get(field);
    if (!control?.errors) return '';
    
    if (control.errors['required']) return 'กรุณากรอกข้อมูลนี้';
    if (control.errors['email']) return 'รูปแบบอีเมลไม่ถูกต้อง';
    if (control.errors['minlength']) return `ต้องมีอย่างน้อย ${control.errors['minlength'].requiredLength} ตัวอักษร`;
    
    return 'ข้อมูลไม่ถูกต้อง';
  }

  onSubmit(): void {
    if (this.form.valid) {
      console.log('Form submitted:', this.form.value);
    }
  }
}
```

---

## 5. Color Contrast และ Visual Indicators

### Accessible Color Utility

```typescript
// app/utils/color-contrast.util.ts

export function getContrastRatio(color1: string, color2: string): number {
  const l1 = getRelativeLuminance(color1);
  const l2 = getRelativeLuminance(color2);
  
  const lighter = Math.max(l1, l2);
  const darker = Math.min(l1, l2);
  
  return (lighter + 0.05) / (darker + 0.05);
}

function getRelativeLuminance(color: string): number {
  const rgb = hexToRgb(color);
  if (!rgb) return 0;
  
  const [r, g, b] = [rgb.r / 255, rgb.g / 255, rgb.b / 255].map(c => {
    return c <= 0.03928 
      ? c / 12.92 
      : Math.pow((c + 0.055) / 1.055, 2.4);
  });
  
  return 0.2126 * r + 0.7152 * g + 0.0722 * b;
}

function hexToRgb(hex: string): { r: number; g: number; b: number } | null {
  const result = /^#?([a-f\d]{2})([a-f\d]{2})([a-f\d]{2})$/i.exec(hex);
  return result ? {
    r: parseInt(result[1], 16),
    g: parseInt(result[2], 16),
    b: parseInt(result[3], 16)
  } : null;
}

// WCAG AA ต้องการ 4.5:1 สำหรับ normal text, 3:1 สำหรับ large text
export function meetsWCAG(ratio: number, level: 'AA' | 'AAA' = 'AA', large = false): boolean {
  if (level === 'AAA') return large ? ratio >= 4.5 : ratio >= 7;
  return large ? ratio >= 3 : ratio >= 4.5;
}
```

---

## 6. Testing Accessibility

### Accessibility Test Utilities

```typescript
// app/testing/a11y-test.utils.ts
import { ComponentFixture } from '@angular/core/testing';

export function checkAriaAttributes(element: HTMLElement): string[] {
  const issues: string[] = [];
  
  // ตรวจสอบ images
  const images = element.querySelectorAll('img');
  images.forEach((img, i) => {
    if (!img.getAttribute('alt') && img.getAttribute('alt') !== '') {
      issues.push(`Image ${i + 1}: ขาด alt attribute`);
    }
  });
  
  // ตรวจสอบ buttons
  const buttons = element.querySelectorAll('button');
  buttons.forEach((btn, i) => {
    const hasText = btn.textContent?.trim();
    const hasAriaLabel = btn.getAttribute('aria-label');
    const hasAriaLabelledBy = btn.getAttribute('aria-labelledby');
    
    if (!hasText && !hasAriaLabel && !hasAriaLabelledBy) {
      issues.push(`Button ${i + 1}: ขาด accessible name`);
    }
  });
  
  // ตรวจสอบ form inputs
  const inputs = element.querySelectorAll('input, select, textarea');
  inputs.forEach((input, i) => {
    const id = input.getAttribute('id');
    const label = id ? element.querySelector(`label[for="${id}"]`) : null;
    const ariaLabel = input.getAttribute('aria-label');
    const ariaLabelledBy = input.getAttribute('aria-labelledby');
    
    if (!label && !ariaLabel && !ariaLabelledBy) {
      issues.push(`Input ${i + 1}: ขาด label`);
    }
  });
  
  return issues;
}

// ใช้กับ Jest/Jasmine
export function expectNoA11yIssues(fixture: ComponentFixture<any>): void {
  const issues = checkAriaAttributes(fixture.nativeElement);
  expect(issues).toEqual([]);
}
```

---

## สรุป

| หัวข้อ | สิ่งที่ต้องทำ |
|--------|--------------|
| ARIA Roles | เพิ่ม role, aria-label, aria-describedby |
| Keyboard | Tab order, Focus trap, Skip links |
| Screen Reader | Live regions, SR-only text |
| Forms | Label associations, Error messages |
| Visual | Color contrast 4.5:1, Focus indicators |

### Checklist WCAG 2.1 AA

- [ ] ทุก image มี alt text
- [ ] Color contrast อย่างน้อย 4.5:1
- [ ] สามารถใช้งานด้วย keyboard เท่านั้น
- [ ] Focus indicator มองเห็นได้ชัดเจน
- [ ] Form labels เชื่อมกับ inputs
- [ ] Error messages อ่านออกเสียงได้
- [ ] Skip navigation link
- [ ] Page title อธิบายเนื้อหา
- [ ] Language attribute บน html element
- [ ] Heading hierarchy ถูกต้อง (h1 > h2 > h3)
