# Part 19 — Custom Directives ใน Angular

## บทนำ

Directive คือคลาสที่ใช้เพิ่มพฤติกรรมให้กับ DOM element ใน Angular มี Directive อยู่ 3 ประเภทหลัก:

1. **Component Directive** — Directive ที่มี Template (Component นั่นเอง)
2. **Attribute Directive** — เปลี่ยน appearance หรือ behavior ของ element
3. **Structural Directive** — เปลี่ยนโครงสร้าง DOM (เพิ่ม/ลบ element)

---

## 1. Attribute Directive สร้างเอง

### 1.1 พื้นฐาน Attribute Directive

Attribute Directive ใช้เพื่อเปลี่ยน appearance หรือ behavior ของ element โดยไม่เปลี่ยนโครงสร้าง DOM

```bash
# สร้าง directive ด้วย Angular CLI
ng generate directive directives/highlight
# หรือย่อ
ng g d directives/highlight
```

```typescript
// src/app/directives/highlight.directive.ts
import {
  Directive,
  ElementRef,
  OnInit,
  Input,
  HostListener,
  Renderer2
} from '@angular/core';

@Directive({
  selector: '[appHighlight]',
  standalone: true
})
export class HighlightDirective implements OnInit {

  @Input() appHighlight = 'yellow';        // สีที่ต้องการ highlight
  @Input() defaultColor = 'transparent';   // สีเริ่มต้น

  constructor(
    private el: ElementRef,
    private renderer: Renderer2
  ) {}

  ngOnInit(): void {
    // ตั้งค่าเริ่มต้น
    this.renderer.setStyle(
      this.el.nativeElement,
      'backgroundColor',
      this.defaultColor
    );
  }

  @HostListener('mouseenter')
  onMouseEnter(): void {
    this.renderer.setStyle(
      this.el.nativeElement,
      'backgroundColor',
      this.appHighlight
    );
    this.renderer.setStyle(
      this.el.nativeElement,
      'transition',
      'background-color 0.3s ease'
    );
  }

  @HostListener('mouseleave')
  onMouseLeave(): void {
    this.renderer.setStyle(
      this.el.nativeElement,
      'backgroundColor',
      this.defaultColor
    );
  }
}
```

### 1.2 การใช้งาน Highlight Directive

```html
<!-- app.component.html -->

<!-- ใช้แบบพื้นฐาน - highlight สีเหลือง -->
<p appHighlight>วางเมาส์ที่นี่เพื่อ highlight</p>

<!-- กำหนดสีเอง -->
<p [appHighlight]="'lightblue'">Highlight สีฟ้า</p>

<!-- กำหนดทั้งสี highlight และสีเริ่มต้น -->
<p
  [appHighlight]="'#ff6b6b'"
  [defaultColor]="'#f8f9fa'"
>
  Highlight สีแดง
</p>

<!-- ใช้กับ div -->
<div
  [appHighlight]="selectedColor"
  class="card p-3"
>
  <h3>Card with Highlight</h3>
  <p>เนื้อหาในการ์ด</p>
</div>
```

### 1.3 Directive แบบมีความซับซ้อน

```typescript
// src/app/directives/tooltip.directive.ts
import {
  Directive,
  ElementRef,
  Input,
  HostListener,
  OnDestroy,
  Renderer2
} from '@angular/core';

@Directive({
  selector: '[appTooltip]',
  standalone: true
})
export class TooltipDirective implements OnDestroy {

  @Input() appTooltip = '';            // ข้อความ tooltip
  @Input() tooltipPosition: 'top' | 'bottom' | 'left' | 'right' = 'top';
  @Input() tooltipDelay = 200;         // delay ก่อนแสดง (milliseconds)

  private tooltipElement: HTMLElement | null = null;
  private showTimeout: ReturnType<typeof setTimeout> | null = null;

  constructor(
    private el: ElementRef,
    private renderer: Renderer2
  ) {}

  @HostListener('mouseenter')
  onMouseEnter(): void {
    this.showTimeout = setTimeout(() => {
      this.showTooltip();
    }, this.tooltipDelay);
  }

  @HostListener('mouseleave')
  onMouseLeave(): void {
    if (this.showTimeout) {
      clearTimeout(this.showTimeout);
    }
    this.hideTooltip();
  }

  private showTooltip(): void {
    // สร้าง tooltip element
    this.tooltipElement = this.renderer.createElement('div');
    const text = this.renderer.createText(this.appTooltip);

    this.renderer.appendChild(this.tooltipElement, text);
    this.renderer.appendChild(document.body, this.tooltipElement);

    // กำหนด style
    this.renderer.setStyle(this.tooltipElement, 'position', 'fixed');
    this.renderer.setStyle(this.tooltipElement, 'background', 'rgba(0,0,0,0.8)');
    this.renderer.setStyle(this.tooltipElement, 'color', 'white');
    this.renderer.setStyle(this.tooltipElement, 'padding', '6px 12px');
    this.renderer.setStyle(this.tooltipElement, 'borderRadius', '4px');
    this.renderer.setStyle(this.tooltipElement, 'fontSize', '14px');
    this.renderer.setStyle(this.tooltipElement, 'zIndex', '9999');
    this.renderer.setStyle(this.tooltipElement, 'pointerEvents', 'none');

    // คำนวณตำแหน่ง
    this.setPosition();

    // Animation
    this.renderer.setStyle(this.tooltipElement, 'opacity', '0');
    this.renderer.setStyle(this.tooltipElement, 'transition', 'opacity 0.2s ease');

    setTimeout(() => {
      if (this.tooltipElement) {
        this.renderer.setStyle(this.tooltipElement, 'opacity', '1');
      }
    }, 10);
  }

  private setPosition(): void {
    if (!this.tooltipElement) return;

    const hostRect = this.el.nativeElement.getBoundingClientRect();
    const tooltipRect = this.tooltipElement.getBoundingClientRect();
    const offset = 8;

    let top = 0;
    let left = 0;

    switch (this.tooltipPosition) {
      case 'top':
        top = hostRect.top - tooltipRect.height - offset;
        left = hostRect.left + (hostRect.width - tooltipRect.width) / 2;
        break;
      case 'bottom':
        top = hostRect.bottom + offset;
        left = hostRect.left + (hostRect.width - tooltipRect.width) / 2;
        break;
      case 'left':
        top = hostRect.top + (hostRect.height - tooltipRect.height) / 2;
        left = hostRect.left - tooltipRect.width - offset;
        break;
      case 'right':
        top = hostRect.top + (hostRect.height - tooltipRect.height) / 2;
        left = hostRect.right + offset;
        break;
    }

    this.renderer.setStyle(this.tooltipElement, 'top', `${top}px`);
    this.renderer.setStyle(this.tooltipElement, 'left', `${left}px`);
  }

  private hideTooltip(): void {
    if (this.tooltipElement) {
      this.renderer.removeChild(document.body, this.tooltipElement);
      this.tooltipElement = null;
    }
  }

  ngOnDestroy(): void {
    this.hideTooltip();
    if (this.showTimeout) {
      clearTimeout(this.showTimeout);
    }
  }
}
```

---

## 2. Structural Directive สร้างเอง

### 2.1 ทำความเข้าใจ Structural Directive

Structural Directive ใช้เพื่อเปลี่ยนโครงสร้าง DOM โดยการเพิ่มหรือลบ element เช่น `*ngIf`, `*ngFor`, `*ngSwitch`

สัญลักษณ์ `*` เป็น syntactic sugar สำหรับ `<ng-template>`

```html
<!-- เขียนแบบสั้น -->
<div *ngIf="isVisible">เนื้อหา</div>

<!-- เขียนแบบเต็ม (Angular แปลงให้) -->
<ng-template [ngIf]="isVisible">
  <div>เนื้อหา</div>
</ng-template>
```

### 2.2 สร้าง Custom Structural Directive

```typescript
// src/app/directives/unless.directive.ts
import {
  Directive,
  Input,
  TemplateRef,
  ViewContainerRef
} from '@angular/core';

@Directive({
  selector: '[appUnless]',
  standalone: true
})
export class UnlessDirective {

  private hasView = false;

  constructor(
    private templateRef: TemplateRef<unknown>,
    private viewContainer: ViewContainerRef
  ) {}

  // appUnless เป็น opposite ของ ngIf (แสดงเมื่อ condition เป็น false)
  @Input() set appUnless(condition: boolean) {
    if (!condition && !this.hasView) {
      // เพิ่ม template เข้า DOM
      this.viewContainer.createEmbeddedView(this.templateRef);
      this.hasView = true;
    } else if (condition && this.hasView) {
      // ลบ template ออกจาก DOM
      this.viewContainer.clear();
      this.hasView = false;
    }
  }
}
```

### 2.3 Structural Directive แบบซับซ้อน - Repeat Directive

```typescript
// src/app/directives/repeat.directive.ts
import {
  Directive,
  Input,
  TemplateRef,
  ViewContainerRef,
  OnChanges,
  SimpleChanges
} from '@angular/core';

interface RepeatContext {
  $implicit: number;   // index ปัจจุบัน (เข้าถึงด้วย let i)
  index: number;
  first: boolean;
  last: boolean;
  even: boolean;
  odd: boolean;
}

@Directive({
  selector: '[appRepeat]',
  standalone: true
})
export class RepeatDirective implements OnChanges {

  @Input() appRepeat = 0;

  constructor(
    private templateRef: TemplateRef<RepeatContext>,
    private viewContainer: ViewContainerRef
  ) {}

  ngOnChanges(changes: SimpleChanges): void {
    if (changes['appRepeat']) {
      this.updateView();
    }
  }

  private updateView(): void {
    this.viewContainer.clear();

    for (let i = 0; i < this.appRepeat; i++) {
      this.viewContainer.createEmbeddedView(this.templateRef, {
        $implicit: i,
        index: i,
        first: i === 0,
        last: i === this.appRepeat - 1,
        even: i % 2 === 0,
        odd: i % 2 !== 0
      });
    }
  }

  // Static method สำหรับ Type Checking
  static ngTemplateContextGuard(
    _directive: RepeatDirective,
    ctx: unknown
  ): ctx is RepeatContext {
    return true;
  }
}
```

```html
<!-- การใช้งาน Repeat Directive -->
<div *appRepeat="5; let i; let isFirst = first; let isLast = last">
  <span *ngIf="isFirst">[First] </span>
  Item {{ i + 1 }}
  <span *ngIf="isLast"> [Last]</span>
</div>

<!-- แสดงผล: [First] Item 1, Item 2, Item 3, Item 4, Item 5 [Last] -->
```

---

## 3. HostListener และ HostBinding

### 3.1 HostListener

`@HostListener` ใช้สำหรับฟัง DOM events บน host element

```typescript
// src/app/directives/click-tracker.directive.ts
import {
  Directive,
  HostListener,
  ElementRef,
  Output,
  EventEmitter
} from '@angular/core';

@Directive({
  selector: '[appClickTracker]',
  standalone: true
})
export class ClickTrackerDirective {

  @Output() clickPosition = new EventEmitter<{ x: number; y: number }>();

  constructor(private el: ElementRef) {}

  // ฟัง click event บน host element
  @HostListener('click', ['$event'])
  onClick(event: MouseEvent): void {
    const rect = this.el.nativeElement.getBoundingClientRect();
    const relativeX = event.clientX - rect.left;
    const relativeY = event.clientY - rect.top;

    this.clickPosition.emit({ x: relativeX, y: relativeY });

    console.log(`คลิกที่ตำแหน่ง: (${relativeX}, ${relativeY})`);
  }

  // ฟัง keyboard event บน document
  @HostListener('document:keydown.escape')
  onEscapeKey(): void {
    console.log('กด Escape key');
  }

  // ฟัง window scroll event
  @HostListener('window:scroll', ['$event'])
  onScroll(event: Event): void {
    console.log('Scroll position:', window.scrollY);
  }

  // ฟัง event บน element ตัวเอง
  @HostListener('mouseenter')
  @HostListener('focus')
  onEnterOrFocus(): void {
    console.log('Element ได้รับ focus หรือ hover');
  }
}
```

### 3.2 HostBinding

`@HostBinding` ใช้สำหรับ bind property หรือ attribute ให้กับ host element

```typescript
// src/app/directives/active-link.directive.ts
import {
  Directive,
  HostBinding,
  HostListener,
  Input,
  OnInit
} from '@angular/core';

@Directive({
  selector: '[appActiveLink]',
  standalone: true
})
export class ActiveLinkDirective implements OnInit {

  @Input() activePath = '';

  // Bind CSS class กับ host element
  @HostBinding('class.active') isActive = false;
  @HostBinding('class.disabled') isDisabled = false;

  // Bind attribute กับ host element
  @HostBinding('attr.aria-current') ariaCurrent: string | undefined;

  // Bind style กับ host element
  @HostBinding('style.fontWeight') fontWeight = 'normal';
  @HostBinding('style.color') color = 'inherit';

  // Bind property กับ host element
  @HostBinding('tabIndex') tabIndex = 0;

  ngOnInit(): void {
    // ตรวจสอบ URL ปัจจุบัน
    this.checkActive();
  }

  @HostListener('click')
  onClick(): void {
    // navigate ไปยัง activePath
    console.log('Navigate to:', this.activePath);
  }

  private checkActive(): void {
    const currentUrl = window.location.pathname;
    this.isActive = currentUrl === this.activePath;

    if (this.isActive) {
      this.ariaCurrent = 'page';
      this.fontWeight = 'bold';
      this.color = '#007bff';
    }
  }
}
```

### 3.3 การรวม HostListener และ HostBinding

```typescript
// src/app/directives/ripple.directive.ts
import {
  Directive,
  HostBinding,
  HostListener,
  ElementRef,
  Renderer2,
  Input
} from '@angular/core';

@Directive({
  selector: '[appRipple]',
  standalone: true
})
export class RippleDirective {

  @Input() rippleColor = 'rgba(255, 255, 255, 0.3)';
  @Input() rippleDuration = 600; // milliseconds

  @HostBinding('style.position') position = 'relative';
  @HostBinding('style.overflow') overflow = 'hidden';
  @HostBinding('style.cursor') cursor = 'pointer';

  constructor(
    private el: ElementRef,
    private renderer: Renderer2
  ) {}

  @HostListener('click', ['$event'])
  onClick(event: MouseEvent): void {
    this.createRipple(event);
  }

  private createRipple(event: MouseEvent): void {
    const button = this.el.nativeElement as HTMLElement;
    const rect = button.getBoundingClientRect();

    const x = event.clientX - rect.left;
    const y = event.clientY - rect.top;

    const diameter = Math.max(button.clientWidth, button.clientHeight);
    const radius = diameter / 2;

    // สร้าง ripple element
    const ripple = this.renderer.createElement('span');

    this.renderer.setStyle(ripple, 'position', 'absolute');
    this.renderer.setStyle(ripple, 'width', `${diameter}px`);
    this.renderer.setStyle(ripple, 'height', `${diameter}px`);
    this.renderer.setStyle(ripple, 'left', `${x - radius}px`);
    this.renderer.setStyle(ripple, 'top', `${y - radius}px`);
    this.renderer.setStyle(ripple, 'background', this.rippleColor);
    this.renderer.setStyle(ripple, 'borderRadius', '50%');
    this.renderer.setStyle(ripple, 'transform', 'scale(0)');
    this.renderer.setStyle(
      ripple,
      'animation',
      `ripple ${this.rippleDuration}ms linear`
    );
    this.renderer.setStyle(ripple, 'pointerEvents', 'none');

    // เพิ่ม CSS Animation
    const style = this.renderer.createElement('style');
    const styleText = this.renderer.createText(`
      @keyframes ripple {
        to {
          transform: scale(4);
          opacity: 0;
        }
      }
    `);
    this.renderer.appendChild(style, styleText);
    this.renderer.appendChild(document.head, style);

    this.renderer.appendChild(button, ripple);

    setTimeout(() => {
      this.renderer.removeChild(button, ripple);
    }, this.rippleDuration);
  }
}
```

---

## 4. Directive Composition API (Angular 15+)

Angular 15 แนะนำ Directive Composition API ที่ให้เราสามารถ compose directives เข้าด้วยกันได้

### 4.1 hostDirectives

```typescript
// src/app/directives/clickable.directive.ts
import { Directive, HostBinding, HostListener } from '@angular/core';

@Directive({
  selector: '[appClickable]',
  standalone: true
})
export class ClickableDirective {
  @HostBinding('style.cursor') cursor = 'pointer';
  @HostBinding('attr.role') role = 'button';
  @HostBinding('attr.tabIndex') tabIndex = '0';

  @HostListener('keydown.enter', ['$event'])
  @HostListener('keydown.space', ['$event'])
  onKeyDown(event: KeyboardEvent): void {
    event.preventDefault();
    (event.target as HTMLElement).click();
  }
}
```

```typescript
// src/app/directives/focusable.directive.ts
import { Directive, HostBinding, HostListener } from '@angular/core';

@Directive({
  selector: '[appFocusable]',
  standalone: true
})
export class FocusableDirective {
  @HostBinding('class.focused') isFocused = false;

  @HostListener('focus')
  onFocus(): void {
    this.isFocused = true;
  }

  @HostListener('blur')
  onBlur(): void {
    this.isFocused = false;
  }
}
```

```typescript
// src/app/directives/button.directive.ts - ใช้ Directive Composition API
import {
  Directive,
  HostBinding,
  Input
} from '@angular/core';
import { ClickableDirective } from './clickable.directive';
import { FocusableDirective } from './focusable.directive';
import { RippleDirective } from './ripple.directive';

@Directive({
  selector: '[appButton]',
  standalone: true,
  // รวม directives หลายๆ ตัวเข้าด้วยกัน
  hostDirectives: [
    ClickableDirective,
    FocusableDirective,
    {
      directive: RippleDirective,
      // expose inputs ของ RippleDirective ออกมา
      inputs: ['rippleColor', 'rippleDuration']
    }
  ]
})
export class ButtonDirective {

  @Input() variant: 'primary' | 'secondary' | 'danger' = 'primary';

  @HostBinding('class')
  get classes(): string {
    return `btn btn-${this.variant}`;
  }
}
```

```html
<!-- การใช้งาน Directive Composition -->
<button
  appButton
  variant="primary"
  rippleColor="rgba(255,255,255,0.4)"
>
  คลิกฉัน
</button>

<!-- ได้ผลลัพธ์: มีทั้ง clickable, focusable, ripple effect และ button styles -->
```

### 4.2 การ expose inputs และ outputs

```typescript
// src/app/directives/form-field.directive.ts
import { Directive, HostBinding, Input, Output, EventEmitter } from '@angular/core';
import { ValidationDirective } from './validation.directive';

@Directive({
  selector: '[appFormField]',
  standalone: true,
  hostDirectives: [
    {
      directive: ValidationDirective,
      // expose inputs ออกมาให้ใช้งานจากภายนอก
      inputs: ['required', 'minLength', 'maxLength', 'pattern'],
      // expose outputs ออกมาให้ใช้งานจากภายนอก
      outputs: ['validationError']
    }
  ]
})
export class FormFieldDirective {
  @HostBinding('class.form-field') isFormField = true;
}
```

---

## 5. Workshop: Tooltip, Highlight, Permission Directives

### 5.1 Permission Directive

```typescript
// src/app/directives/permission.directive.ts
import {
  Directive,
  Input,
  TemplateRef,
  ViewContainerRef,
  OnInit,
  OnDestroy
} from '@angular/core';
import { Subscription } from 'rxjs';
import { AuthService } from '../services/auth.service';

@Directive({
  selector: '[appPermission]',
  standalone: true
})
export class PermissionDirective implements OnInit, OnDestroy {

  @Input() appPermission: string | string[] = [];
  @Input() appPermissionElse?: TemplateRef<unknown>;

  private subscription?: Subscription;
  private hasView = false;

  constructor(
    private templateRef: TemplateRef<unknown>,
    private viewContainer: ViewContainerRef,
    private authService: AuthService
  ) {}

  ngOnInit(): void {
    // ฟัง permission changes
    this.subscription = this.authService.currentUser$.subscribe(user => {
      this.updateView(user?.permissions ?? []);
    });
  }

  private updateView(userPermissions: string[]): void {
    const requiredPermissions = Array.isArray(this.appPermission)
      ? this.appPermission
      : [this.appPermission];

    // ตรวจสอบว่า user มี permission ที่ต้องการหรือไม่
    const hasPermission = requiredPermissions.every(permission =>
      userPermissions.includes(permission)
    );

    this.viewContainer.clear();

    if (hasPermission) {
      // แสดง content หลัก
      this.viewContainer.createEmbeddedView(this.templateRef);
    } else if (this.appPermissionElse) {
      // แสดง else template ถ้ามี
      this.viewContainer.createEmbeddedView(this.appPermissionElse);
    }
  }

  ngOnDestroy(): void {
    this.subscription?.unsubscribe();
  }
}
```

```typescript
// src/app/services/auth.service.ts
import { Injectable } from '@angular/core';
import { BehaviorSubject } from 'rxjs';

interface User {
  id: number;
  name: string;
  permissions: string[];
}

@Injectable({ providedIn: 'root' })
export class AuthService {

  private currentUserSubject = new BehaviorSubject<User | null>(null);
  currentUser$ = this.currentUserSubject.asObservable();

  setUser(user: User): void {
    this.currentUserSubject.next(user);
  }

  hasPermission(permission: string): boolean {
    const user = this.currentUserSubject.value;
    return user?.permissions.includes(permission) ?? false;
  }
}
```

```html
<!-- การใช้งาน Permission Directive -->

<!-- แสดงเฉพาะผู้ที่มี permission 'admin' -->
<div *appPermission="'admin'">
  <button>ลบข้อมูล</button>
</div>

<!-- แสดงเฉพาะผู้ที่มี permission หลายอย่าง -->
<div *appPermission="['admin', 'write']">
  <button>แก้ไขข้อมูล</button>
</div>

<!-- มี else template -->
<div *appPermission="'admin'; else noPermission">
  <button>Admin Action</button>
</div>
<ng-template #noPermission>
  <p>คุณไม่มีสิทธิ์เข้าถึงส่วนนี้</p>
</ng-template>
```

### 5.2 สรุปการประกาศใน NgModule หรือ imports

```typescript
// app.component.ts (Standalone)
import { Component } from '@angular/core';
import { HighlightDirective } from './directives/highlight.directive';
import { TooltipDirective } from './directives/tooltip.directive';
import { PermissionDirective } from './directives/permission.directive';
import { RepeatDirective } from './directives/repeat.directive';
import { ButtonDirective } from './directives/button.directive';

@Component({
  selector: 'app-root',
  standalone: true,
  imports: [
    HighlightDirective,
    TooltipDirective,
    PermissionDirective,
    RepeatDirective,
    ButtonDirective
  ],
  template: `
    <!-- Highlight Directive -->
    <p [appHighlight]="'#fff3cd'" [defaultColor]="'white'">
      Hover เพื่อ highlight
    </p>

    <!-- Tooltip Directive -->
    <button
      [appTooltip]="'คลิกเพื่อบันทึก'"
      tooltipPosition="top"
    >
      บันทึก
    </button>

    <!-- Permission Directive -->
    <div *appPermission="'admin'">Admin Panel</div>

    <!-- Repeat Directive -->
    <div *appRepeat="3; let i">Row {{ i + 1 }}</div>

    <!-- Button Directive (Composition) -->
    <div appButton variant="primary">Click Me</div>
  `
})
export class AppComponent {}
```

---

## สรุปบทที่ 19

| แนวคิด | คำอธิบาย |
|--------|----------|
| Attribute Directive | เปลี่ยน appearance/behavior ไม่เปลี่ยนโครงสร้าง DOM |
| Structural Directive | เปลี่ยนโครงสร้าง DOM (เพิ่ม/ลบ element) |
| @HostListener | ฟัง DOM events บน host element |
| @HostBinding | Bind property/attribute/style/class กับ host element |
| TemplateRef | Reference ไปยัง `<ng-template>` สำหรับ Structural Directive |
| ViewContainerRef | ใช้สร้างและจัดการ embedded views |
| Directive Composition | รวม directives หลายตัวเข้าด้วยกัน (Angular 15+) |
| Renderer2 | จัดการ DOM อย่างปลอดภัย (รองรับ SSR) |

### Best Practices

1. **ใช้ Renderer2 แทน ElementRef.nativeElement โดยตรง** — รองรับ Server-Side Rendering
2. **Cleanup ใน ngOnDestroy** — ยกเลิก subscriptions, ลบ DOM elements
3. **ตั้งชื่อ selector ให้ชัดเจน** — ใช้ prefix เช่น `app` เพื่อป้องกัน name collision
4. **ใช้ Input setter** — สำหรับ reactive behavior ใน Structural Directives
5. **ทดสอบ directives แยกต่างหาก** — เขียน unit tests ครอบคลุม
