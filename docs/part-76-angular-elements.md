# Part 76: Angular Elements - สร้าง Web Components

## Angular Elements คืออะไร

Angular Elements แปลง Angular components เป็น Custom Elements (Web Components) ที่ใช้งานได้ใน HTML ธรรมดา หรือ framework อื่น

---

## 1. ติดตั้งและตั้งค่า

```bash
ng add @angular/elements

# หรือ manual
npm install @angular/elements
npm install @webcomponents/custom-elements  # polyfill
```

### app.module.ts

```typescript
import { NgModule, Injector } from '@angular/core';
import { BrowserModule } from '@angular/platform-browser';
import { createCustomElement } from '@angular/elements';
import { RatingComponent } from './rating/rating.component';
import { AlertBannerComponent } from './alert-banner/alert-banner.component';

@NgModule({
  declarations: [RatingComponent, AlertBannerComponent],
  imports: [BrowserModule],
  // ไม่ต้องมี bootstrap เมื่อใช้ Elements
})
export class AppModule {
  constructor(private injector: Injector) {}

  ngDoBootstrap(): void {
    // แปลง components เป็น custom elements
    const RatingElement = createCustomElement(RatingComponent, {
      injector: this.injector
    });
    customElements.define('app-rating', RatingElement);

    const AlertElement = createCustomElement(AlertBannerComponent, {
      injector: this.injector
    });
    customElements.define('app-alert-banner', AlertElement);
  }
}
```

---

## 2. Rating Component

```typescript
// app/rating/rating.component.ts
import { 
  Component, Input, Output, EventEmitter, 
  ChangeDetectionStrategy, OnChanges, SimpleChanges 
} from '@angular/core';

@Component({
  selector: 'app-rating',
  template: `
    <div 
      class="rating-container"
      role="group"
      [attr.aria-label]="'คะแนน: ' + currentRating + ' จาก ' + maxStars"
    >
      <button
        *ngFor="let star of stars; let i = index"
        class="star"
        [class.filled]="i < currentRating"
        [class.hover]="i < hoverRating"
        [attr.aria-label]="'ให้ ' + (i + 1) + ' ดาว'"
        [attr.aria-pressed]="i < currentRating"
        (click)="setRating(i + 1)"
        (mouseenter)="setHover(i + 1)"
        (mouseleave)="clearHover()"
        (keydown.enter)="setRating(i + 1)"
        (keydown.space)="setRating(i + 1)"
        type="button"
      >
        <svg viewBox="0 0 24 24" width="24" height="24">
          <path d="M12 17.27L18.18 21l-1.64-7.03L22 9.24l-7.19-.61L12 2 9.19 8.63 2 9.24l5.46 4.73L5.82 21z"/>
        </svg>
      </button>
      
      <span class="rating-value" *ngIf="showValue">
        {{ currentRating }}/{{ maxStars }}
      </span>
    </div>
  `,
  styles: [`
    :host {
      display: inline-block;
      --star-color: #ffd700;
      --star-empty: #ddd;
    }
    .rating-container {
      display: inline-flex;
      align-items: center;
      gap: 4px;
    }
    .star {
      background: none;
      border: none;
      cursor: pointer;
      padding: 2px;
      color: var(--star-empty);
      transition: color 0.2s, transform 0.1s;
    }
    .star:hover { transform: scale(1.2); }
    .star.filled { color: var(--star-color); }
    .star.hover { color: var(--star-color); opacity: 0.8; }
    .star svg { fill: currentColor; }
    .rating-value {
      font-size: 14px;
      color: #666;
      margin-left: 8px;
    }
  `],
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class RatingComponent implements OnChanges {
  @Input() value = 0;
  @Input() maxStars = 5;
  @Input() readonly = false;
  @Input() showValue = true;
  @Input() color = '#ffd700';
  
  @Output() ratingChange = new EventEmitter<number>();

  currentRating = 0;
  hoverRating = 0;
  stars: number[] = [];

  ngOnChanges(changes: SimpleChanges): void {
    if (changes['maxStars']) {
      this.stars = Array.from({ length: this.maxStars }, (_, i) => i);
    }
    if (changes['value']) {
      this.currentRating = this.value;
    }
  }

  setRating(rating: number): void {
    if (this.readonly) return;
    // Toggle: คลิกดาวเดิม = ยกเลิก
    this.currentRating = this.currentRating === rating ? 0 : rating;
    this.ratingChange.emit(this.currentRating);
    
    // Dispatch custom event สำหรับ non-Angular environments
    const event = new CustomEvent('rating-changed', {
      detail: { value: this.currentRating },
      bubbles: true
    });
    // element.dispatchEvent(event);
  }

  setHover(rating: number): void {
    if (!this.readonly) this.hoverRating = rating;
  }

  clearHover(): void {
    this.hoverRating = 0;
  }
}
```

---

## 3. Alert Banner Component

```typescript
// app/alert-banner/alert-banner.component.ts
import { 
  Component, Input, Output, EventEmitter, 
  ChangeDetectionStrategy, OnInit, OnDestroy 
} from '@angular/core';

export type AlertType = 'info' | 'success' | 'warning' | 'error';

@Component({
  selector: 'app-alert-banner',
  template: `
    <div 
      *ngIf="visible"
      class="alert-banner"
      [class]="'alert-' + type"
      role="alert"
      [attr.aria-live]="type === 'error' ? 'assertive' : 'polite'"
    >
      <span class="alert-icon">{{ getIcon() }}</span>
      <div class="alert-content">
        <strong *ngIf="title" class="alert-title">{{ title }}</strong>
        <p class="alert-message">{{ message }}</p>
      </div>
      
      <div class="alert-actions" *ngIf="actionLabel">
        <button class="btn-action" (click)="onAction()">
          {{ actionLabel }}
        </button>
      </div>
      
      <button 
        *ngIf="dismissible"
        class="btn-dismiss"
        (click)="dismiss()"
        aria-label="ปิดแจ้งเตือน"
      >
        ✕
      </button>
    </div>
  `,
  styles: [`
    :host {
      display: block;
      --info-color: #2196f3;
      --success-color: #4caf50;
      --warning-color: #ff9800;
      --error-color: #f44336;
    }
    .alert-banner {
      display: flex;
      align-items: flex-start;
      gap: 12px;
      padding: 12px 16px;
      border-radius: 8px;
      border-left: 4px solid;
    }
    .alert-info { 
      background: #e3f2fd; 
      border-color: var(--info-color); 
      color: #1565c0;
    }
    .alert-success { 
      background: #e8f5e9; 
      border-color: var(--success-color); 
      color: #2e7d32;
    }
    .alert-warning { 
      background: #fff3e0; 
      border-color: var(--warning-color); 
      color: #e65100;
    }
    .alert-error { 
      background: #ffebee; 
      border-color: var(--error-color); 
      color: #b71c1c;
    }
    .alert-icon { font-size: 20px; }
    .alert-content { flex: 1; }
    .alert-title { display: block; margin-bottom: 4px; }
    .alert-message { margin: 0; font-size: 14px; }
    .btn-dismiss {
      background: none;
      border: none;
      cursor: pointer;
      opacity: 0.6;
      font-size: 16px;
    }
    .btn-dismiss:hover { opacity: 1; }
    .btn-action {
      background: none;
      border: 1px solid currentColor;
      padding: 4px 12px;
      border-radius: 4px;
      cursor: pointer;
      font-size: 13px;
    }
    .btn-action:hover { background: rgba(0,0,0,0.05); }
  `],
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class AlertBannerComponent implements OnInit, OnDestroy {
  @Input() type: AlertType = 'info';
  @Input() title = '';
  @Input() message = '';
  @Input() dismissible = true;
  @Input() actionLabel = '';
  @Input() autoDismiss = 0; // milliseconds, 0 = no auto dismiss
  
  @Output() dismissed = new EventEmitter<void>();
  @Output() actionClicked = new EventEmitter<void>();

  visible = true;
  private timer: any;

  ngOnInit(): void {
    if (this.autoDismiss > 0) {
      this.timer = setTimeout(() => this.dismiss(), this.autoDismiss);
    }
  }

  getIcon(): string {
    const icons: Record<AlertType, string> = {
      info: 'ℹ️',
      success: '✅',
      warning: '⚠️',
      error: '❌'
    };
    return icons[this.type];
  }

  dismiss(): void {
    this.visible = false;
    this.dismissed.emit();
  }

  onAction(): void {
    this.actionClicked.emit();
  }

  ngOnDestroy(): void {
    if (this.timer) clearTimeout(this.timer);
  }
}
```

---

## 4. Shadow DOM

```typescript
// app/shadow-component/shadow.component.ts
import { Component, ViewEncapsulation } from '@angular/core';

@Component({
  selector: 'app-shadow-widget',
  template: `
    <div class="widget">
      <h3>Shadow DOM Widget</h3>
      <p>Styles ที่นี่ไม่รั่วออกไปข้างนอก</p>
      <slot></slot>  <!-- สำหรับ content projection ใน Web Components -->
    </div>
  `,
  styles: [`
    /* Styles เหล่านี้ scoped ใน Shadow DOM */
    :host {
      display: block;
      font-family: 'Segoe UI', sans-serif;
    }
    .widget {
      padding: 20px;
      border: 2px solid #2196f3;
      border-radius: 12px;
      background: white;
    }
    h3 { color: #2196f3; margin: 0 0 8px; }
    p { color: #666; font-size: 14px; }
  `],
  encapsulation: ViewEncapsulation.ShadowDom  // ← Shadow DOM!
})
export class ShadowWidgetComponent {}
```

---

## 5. Build สำหรับ Distribution

```typescript
// app.module.ts - สำหรับ build แบบ single JS file
import { NgModule, Injector, APP_INITIALIZER } from '@angular/core';
import { BrowserModule } from '@angular/platform-browser';
import { createCustomElement } from '@angular/elements';
import { RatingComponent } from './rating/rating.component';
import { AlertBannerComponent } from './alert-banner/alert-banner.component';

@NgModule({
  declarations: [RatingComponent, AlertBannerComponent],
  imports: [BrowserModule],
  entryComponents: [RatingComponent, AlertBannerComponent]
})
export class AppModule {
  constructor(private injector: Injector) {
    const RatingElement = createCustomElement(RatingComponent, { injector });
    customElements.define('app-rating', RatingElement);
    
    const AlertElement = createCustomElement(AlertBannerComponent, { injector });
    customElements.define('app-alert-banner', AlertElement);
  }

  ngDoBootstrap(): void {}
}
```

### Build Script

```json
// package.json scripts
{
  "scripts": {
    "build:elements": "ng build --configuration production --output-hashing none",
    "bundle:elements": "cat dist/my-elements/runtime.js dist/my-elements/polyfills.js dist/my-elements/main.js > elements.bundle.js"
  }
}
```

---

## 6. ใช้งานใน HTML ธรรมดา

```html
<!DOCTYPE html>
<html>
<head>
  <title>Angular Elements Demo</title>
</head>
<body>
  <!-- ใช้งานเหมือน HTML element ปกติ -->
  <app-rating value="3" max-stars="5" show-value="true"></app-rating>
  
  <app-alert-banner 
    type="success"
    title="บันทึกสำเร็จ"
    message="ข้อมูลถูกบันทึกแล้ว"
    dismissible="true"
    auto-dismiss="5000"
  ></app-alert-banner>

  <script src="elements.bundle.js"></script>
  
  <script>
    // รับ event จาก Angular Elements
    const rating = document.querySelector('app-rating');
    rating.addEventListener('ratingChange', (e) => {
      console.log('คะแนน:', e.detail);
    });

    // เปลี่ยน attribute
    rating.setAttribute('value', '4');
  </script>
</body>
</html>
```

### ใช้ใน React

```jsx
// React component ที่ใช้ Angular Elements
import React, { useRef, useEffect } from 'react';

function ProductRating({ initialValue, onChange }) {
  const ratingRef = useRef(null);

  useEffect(() => {
    const element = ratingRef.current;
    const handleChange = (e) => onChange(e.detail.value);
    
    element.addEventListener('ratingChange', handleChange);
    return () => element.removeEventListener('ratingChange', handleChange);
  }, [onChange]);

  return (
    <app-rating 
      ref={ratingRef}
      value={initialValue}
      max-stars="5"
    />
  );
}
```

---

## สรุป

| Feature | รายละเอียด |
|---------|-----------|
| createCustomElement | แปลง Angular component |
| customElements.define | ลงทะเบียน custom element |
| Shadow DOM | Style isolation |
| Slots | Content projection |
| Attributes | @Input ↔ HTML attributes |
| Events | @Output ↔ CustomEvent |

### Use Cases

- Design system ที่ใช้ข้าม framework
- Micro-frontends
- WordPress/CMS widgets
- Email template components
