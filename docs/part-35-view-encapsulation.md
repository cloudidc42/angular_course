# Part 35 — View Encapsulation

## View Encapsulation คืออะไร?

View Encapsulation คือกลไกที่ Angular ใช้จำกัดขอบเขตของ CSS Styles ใน Component ป้องกัน Styles ของ Component หนึ่งรั่วไหลไปกระทบ Component อื่น

Angular มี 3 โหมด:
1. **Emulated** (ค่าเริ่มต้น) — จำลอง Shadow DOM ด้วย Attribute
2. **None** — ไม่มีการกำกัด Styles เป็น Global
3. **ShadowDom** — ใช้ Browser's Native Shadow DOM

---

## 1. Emulated (ค่าเริ่มต้น)

```typescript
import { Component, ViewEncapsulation } from '@angular/core';

@Component({
  selector: 'app-hello',
  template: '<p class="text">สวัสดี</p>',
  styles: ['p { color: red; }'],
  encapsulation: ViewEncapsulation.Emulated,  // ค่าเริ่มต้น ไม่ต้องระบุก็ได้
})
export class HelloComponent {}
```

Angular จะเพิ่ม Attribute พิเศษ `_nghost-xxx` และ `_ngcontent-xxx`:

```html
<!-- HTML ที่ render จริง -->
<app-hello _nghost-c1>
  <p _ngcontent-c1 class="text">สวัสดี</p>
</app-hello>
```

```css
/* CSS ที่ Angular สร้าง */
p[_ngcontent-c1] { color: red; }  /* จำกัดเฉพาะ Component นี้ */
```

---

## 2. None

```typescript
@Component({
  selector: 'app-global',
  template: '<p class="text">สวัสดี</p>',
  styles: ['p { color: blue; }'],
  encapsulation: ViewEncapsulation.None,  // Styles กลายเป็น Global
})
export class GlobalComponent {}
```

**ข้อดี:** ง่าย ไม่มีข้อจำกัด
**ข้อเสีย:** Styles รั่วไปยัง Components อื่น ใช้ระวัง!

---

## 3. ShadowDom

```typescript
@Component({
  selector: 'app-isolated',
  template: '<p class="text">สวัสดี</p>',
  styles: ['p { color: green; }'],
  encapsulation: ViewEncapsulation.ShadowDom,
})
export class IsolatedComponent {}
```

ใช้ Browser's Native Shadow DOM ทำให้ Component แยกออกจาก Document อย่างสมบูรณ์

---

## :host Selector

`:host` เลือก Host Element ของ Component เอง

```typescript
@Component({
  selector: 'app-card',
  standalone: true,
  template: `
    <div class="content">เนื้อหา</div>
  `,
  styles: [`
    /* เลือก app-card element */
    :host {
      display: block;
      border: 1px solid #ddd;
      border-radius: 8px;
      padding: 16px;
    }

    /* เมื่อ Component มี class 'active' */
    :host(.active) {
      border-color: #1976d2;
      background: #e3f2fd;
    }

    /* เมื่อ Component ถูก hover */
    :host(:hover) {
      box-shadow: 0 4px 8px rgba(0,0,0,0.1);
    }

    /* เมื่อ disabled attribute ถูกตั้ง */
    :host([disabled]) {
      opacity: 0.5;
      pointer-events: none;
    }
  `]
})
export class CardComponent {}
```

---

## :host-context Selector

`:host-context` เลือก Host ตาม Context ของ Parent

```typescript
@Component({
  selector: 'app-button',
  standalone: true,
  template: `<button>{{ label }}</button>`,
  styles: [`
    /* ปกติ */
    button {
      background: white;
      color: #333;
      padding: 8px 16px;
      border: 1px solid #ddd;
    }

    /* เมื่ออยู่ใน dark theme */
    :host-context(.dark-theme) button {
      background: #333;
      color: white;
      border-color: #555;
    }

    /* เมื่ออยู่ใน sidebar */
    :host-context(.sidebar) button {
      width: 100%;
    }

    /* เมื่ออยู่ใน form */
    :host-context(form) button {
      margin-top: 8px;
    }
  `]
})
export class ButtonComponent {
  @Input() label = 'Click';
}
```

---

## Global Styles vs Component Styles

### Global Styles (styles.scss)

```scss
/* styles.scss — Global */
/* ใช้สำหรับ Reset, Typography, Layout ที่ใช้ทั่วทั้งแอป */

/* CSS Variables (Design Tokens) */
:root {
  --color-primary: #1976d2;
  --color-secondary: #9c27b0;
  --color-success: #4caf50;
  --color-error: #f44336;
  --color-warning: #ff9800;

  --font-size-base: 16px;
  --font-size-sm: 14px;
  --font-size-lg: 18px;
  --font-size-xl: 24px;

  --spacing-xs: 4px;
  --spacing-sm: 8px;
  --spacing-md: 16px;
  --spacing-lg: 24px;
  --spacing-xl: 32px;

  --border-radius: 8px;
  --shadow-sm: 0 2px 4px rgba(0,0,0,0.08);
  --shadow-md: 0 4px 12px rgba(0,0,0,0.12);
  --shadow-lg: 0 8px 24px rgba(0,0,0,0.16);
}

/* Dark Mode */
[data-theme='dark'] {
  --color-background: #121212;
  --color-surface: #1e1e1e;
  --color-text: #ffffff;
  --color-text-secondary: #aaaaaa;
  --color-border: #333333;
}

/* Reset */
*, *::before, *::after {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}

body {
  font-family: 'Sarabun', sans-serif;
  font-size: var(--font-size-base);
  color: var(--color-text, #333);
  background: var(--color-background, #fff);
}

/* Utility Classes */
.sr-only {
  position: absolute;
  width: 1px;
  height: 1px;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
}
```

### Component Styles — ใช้ Variables ได้

```scss
/* button.component.scss */
/* CSS Variables จาก Global ใช้ได้ใน Component Styles */

:host {
  display: inline-flex;
}

button {
  padding: var(--spacing-sm) var(--spacing-md);
  border-radius: var(--border-radius);
  font-size: var(--font-size-base);
  cursor: pointer;
  border: none;
  transition: all 0.2s;

  &.primary {
    background: var(--color-primary);
    color: white;

    &:hover {
      filter: brightness(1.1);
    }
  }

  &.secondary {
    background: transparent;
    color: var(--color-primary);
    border: 1px solid var(--color-primary);
  }

  &:disabled {
    opacity: 0.6;
    cursor: not-allowed;
  }
}
```

---

## Workshop: Themed Component

### Theme Service

```typescript
// theme.service.ts
import { Injectable, signal, effect } from '@angular/core';

export type Theme = 'light' | 'dark' | 'system';
export type ColorScheme = 'blue' | 'green' | 'purple' | 'orange';

export interface ThemeConfig {
  theme: Theme;
  colorScheme: ColorScheme;
  fontSize: 'small' | 'medium' | 'large';
  borderRadius: 'none' | 'small' | 'medium' | 'large';
}

const COLOR_SCHEMES: Record<ColorScheme, { primary: string; secondary: string }> = {
  blue: { primary: '#1976d2', secondary: '#9c27b0' },
  green: { primary: '#388e3c', secondary: '#0288d1' },
  purple: { primary: '#7b1fa2', secondary: '#f57c00' },
  orange: { primary: '#f57c00', secondary: '#1976d2' },
};

const FONT_SIZES: Record<string, string> = {
  small: '14px',
  medium: '16px',
  large: '18px',
};

const BORDER_RADII: Record<string, string> = {
  none: '0',
  small: '4px',
  medium: '8px',
  large: '16px',
};

@Injectable({ providedIn: 'root' })
export class ThemeService {
  config = signal<ThemeConfig>({
    theme: 'light',
    colorScheme: 'blue',
    fontSize: 'medium',
    borderRadius: 'medium',
  });

  constructor() {
    // โหลด Config จาก LocalStorage
    try {
      const saved = localStorage.getItem('theme-config');
      if (saved) {
        this.config.set(JSON.parse(saved));
      }
    } catch {}

    // Apply Theme Effect
    effect(() => {
      const cfg = this.config();
      const root = document.documentElement;

      // Theme
      const prefersDark = window.matchMedia('(prefers-color-scheme: dark)').matches;
      const isDark = cfg.theme === 'dark' || (cfg.theme === 'system' && prefersDark);
      root.setAttribute('data-theme', isDark ? 'dark' : 'light');

      // Color Scheme
      const colors = COLOR_SCHEMES[cfg.colorScheme];
      root.style.setProperty('--color-primary', colors.primary);
      root.style.setProperty('--color-secondary', colors.secondary);

      // Font Size
      root.style.setProperty('--font-size-base', FONT_SIZES[cfg.fontSize]);

      // Border Radius
      root.style.setProperty('--border-radius', BORDER_RADII[cfg.borderRadius]);

      // บันทึก
      try {
        localStorage.setItem('theme-config', JSON.stringify(cfg));
      } catch {}
    });
  }

  updateConfig(partial: Partial<ThemeConfig>): void {
    this.config.update((c) => ({ ...c, ...partial }));
  }

  setTheme(theme: Theme): void {
    this.updateConfig({ theme });
  }

  setColorScheme(colorScheme: ColorScheme): void {
    this.updateConfig({ colorScheme });
  }

  setFontSize(fontSize: ThemeConfig['fontSize']): void {
    this.updateConfig({ fontSize });
  }
}
```

### Theme Settings Component

```typescript
// theme-settings.component.ts
import { Component, inject } from '@angular/core';
import { CommonModule } from '@angular/common';
import { FormsModule } from '@angular/forms';
import { ThemeService, ThemeConfig } from '../services/theme.service';

@Component({
  selector: 'app-theme-settings',
  standalone: true,
  imports: [CommonModule, FormsModule],
  template: `
    <div class="settings-panel">
      <h3>การตั้งค่าธีม</h3>

      <!-- Theme Mode -->
      <div class="setting-group">
        <label>โหมดสี</label>
        <div class="radio-group">
          <label *ngFor="let opt of themeOptions">
            <input
              type="radio"
              [value]="opt.value"
              [(ngModel)]="selectedTheme"
              (ngModelChange)="onThemeChange($event)"
            />
            {{ opt.label }}
          </label>
        </div>
      </div>

      <!-- Color Scheme -->
      <div class="setting-group">
        <label>ชุดสี</label>
        <div class="color-swatches">
          <button
            *ngFor="let scheme of colorSchemes"
            class="swatch"
            [style.background]="scheme.primary"
            [class.active]="config().colorScheme === scheme.value"
            (click)="onColorSchemeChange(scheme.value)"
            [title]="scheme.label"
          ></button>
        </div>
      </div>

      <!-- Font Size -->
      <div class="setting-group">
        <label>ขนาดตัวอักษร</label>
        <div class="slider-group">
          <button (click)="decreaseFontSize()">A-</button>
          <span>{{ fontSizeLabel }}</span>
          <button (click)="increaseFontSize()">A+</button>
        </div>
      </div>

      <!-- Border Radius -->
      <div class="setting-group">
        <label>ความโค้งมน</label>
        <select [(ngModel)]="selectedRadius" (ngModelChange)="onRadiusChange($event)">
          <option value="none">ไม่โค้ง</option>
          <option value="small">โค้งเล็กน้อย</option>
          <option value="medium">โค้งปานกลาง</option>
          <option value="large">โค้งมาก</option>
        </select>
      </div>

      <!-- Preview -->
      <div class="preview">
        <h4>ตัวอย่าง</h4>
        <button class="btn btn-primary">ปุ่มหลัก</button>
        <button class="btn btn-secondary">ปุ่มรอง</button>
        <div class="card-preview">
          <strong>การ์ดตัวอย่าง</strong>
          <p>นี่คือเนื้อหาในการ์ด</p>
        </div>
      </div>
    </div>
  `,
  styles: [`
    :host {
      display: block;
    }

    .settings-panel {
      padding: var(--spacing-md);
      background: var(--color-surface, #fff);
      border: 1px solid var(--color-border, #e0e0e0);
      border-radius: var(--border-radius);
    }

    .setting-group {
      margin-bottom: var(--spacing-md);
    }

    .setting-group label {
      display: block;
      font-weight: 600;
      margin-bottom: var(--spacing-xs);
      color: var(--color-text, #333);
    }

    .radio-group { display: flex; gap: var(--spacing-md); }
    .radio-group label { font-weight: normal; display: flex; align-items: center; gap: 4px; }

    .color-swatches { display: flex; gap: 8px; }
    .swatch {
      width: 32px;
      height: 32px;
      border-radius: 50%;
      border: 3px solid transparent;
      cursor: pointer;
      transition: transform 0.2s;
    }
    .swatch.active { border-color: var(--color-text, #333); transform: scale(1.2); }
    .swatch:hover { transform: scale(1.1); }

    .slider-group { display: flex; align-items: center; gap: var(--spacing-sm); }
    .slider-group button { padding: 4px 12px; }

    .preview { margin-top: var(--spacing-md); padding-top: var(--spacing-md); border-top: 1px solid var(--color-border, #e0e0e0); }
    .preview h4 { margin-bottom: var(--spacing-sm); }

    .btn {
      padding: var(--spacing-sm) var(--spacing-md);
      border-radius: var(--border-radius);
      border: none;
      cursor: pointer;
      margin-right: var(--spacing-sm);
      font-size: var(--font-size-base);
    }
    .btn-primary { background: var(--color-primary); color: white; }
    .btn-secondary { background: transparent; color: var(--color-primary); border: 1px solid var(--color-primary); }

    .card-preview {
      margin-top: var(--spacing-sm);
      padding: var(--spacing-sm) var(--spacing-md);
      border: 1px solid var(--color-border, #e0e0e0);
      border-radius: var(--border-radius);
      background: var(--color-surface, #fff);
    }

    /* Dark Mode Styles */
    :host-context([data-theme='dark']) .settings-panel {
      background: #1e1e1e;
      color: white;
    }
  `]
})
export class ThemeSettingsComponent {
  themeService = inject(ThemeService);
  config = this.themeService.config;

  selectedTheme = this.config().theme;
  selectedRadius = this.config().borderRadius;

  themeOptions = [
    { value: 'light', label: 'สว่าง' },
    { value: 'dark', label: 'มืด' },
    { value: 'system', label: 'ตามระบบ' },
  ];

  colorSchemes = [
    { value: 'blue', label: 'น้ำเงิน', primary: '#1976d2' },
    { value: 'green', label: 'เขียว', primary: '#388e3c' },
    { value: 'purple', label: 'ม่วง', primary: '#7b1fa2' },
    { value: 'orange', label: 'ส้ม', primary: '#f57c00' },
  ];

  fontSizes = ['small', 'medium', 'large'];

  get fontSizeLabel(): string {
    const labels: Record<string, string> = {
      small: 'เล็ก (14px)',
      medium: 'กลาง (16px)',
      large: 'ใหญ่ (18px)',
    };
    return labels[this.config().fontSize] || '';
  }

  onThemeChange(theme: string): void {
    this.themeService.setTheme(theme as any);
  }

  onColorSchemeChange(colorScheme: string): void {
    this.themeService.setColorScheme(colorScheme as any);
  }

  onRadiusChange(radius: string): void {
    this.themeService.updateConfig({ borderRadius: radius as any });
  }

  increaseFontSize(): void {
    const sizes = this.fontSizes;
    const idx = sizes.indexOf(this.config().fontSize);
    if (idx < sizes.length - 1) {
      this.themeService.setFontSize(sizes[idx + 1] as any);
    }
  }

  decreaseFontSize(): void {
    const sizes = this.fontSizes;
    const idx = sizes.indexOf(this.config().fontSize);
    if (idx > 0) {
      this.themeService.setFontSize(sizes[idx - 1] as any);
    }
  }
}
```

---

## เปรียบเทียบ View Encapsulation Modes

```typescript
// ทดสอบผลกระทบ Styles

// Component A — Emulated
@Component({
  selector: 'app-a',
  template: '<p class="text">Component A</p>',
  styles: ['.text { color: red; }'],
  encapsulation: ViewEncapsulation.Emulated,
})
export class ComponentA {}

// Component B — None
@Component({
  selector: 'app-b',
  template: '<p class="text">Component B</p>',
  styles: ['.text { color: blue; font-weight: bold; }'],
  encapsulation: ViewEncapsulation.None,
})
export class ComponentB {}
// Component B จะทำให้ .text ทั่วทั้งแอปเป็น bold!

// Component C — ShadowDom
@Component({
  selector: 'app-c',
  template: '<p class="text">Component C</p>',
  styles: ['.text { color: green; }'],
  encapsulation: ViewEncapsulation.ShadowDom,
})
export class ComponentC {}
// Component C แยกสมบูรณ์ ไม่ได้รับผลกระทบจาก Global Styles
```

---

## Best Practices

### 1. ใช้ CSS Custom Properties สำหรับ Theming

```scss
/* ใน styles.scss */
:root {
  --btn-bg: #1976d2;
  --btn-color: white;
}

/* ใน button.component.scss */
button {
  background: var(--btn-bg);  /* ใช้ Variable แทน Hardcode */
  color: var(--btn-color);
}
```

### 2. หลีกเลี่ยง ViewEncapsulation.None

```typescript
// ❌ หลีกเลี่ยง
@Component({
  encapsulation: ViewEncapsulation.None,
  styles: ['.important { font-weight: bold; }'],  // รั่วไปทั้งแอป!
})

// ✅ แนะนำ — ใช้ Emulated
@Component({
  styles: ['.important { font-weight: bold; }'],  // จำกัดใน Component
})
```

### 3. ใช้ ::ng-deep สำหรับ Override Library Styles (ใช้ระวัง)

```scss
/* ❌ ไม่แนะนำ — deprecated และอาจถูกเอาออกในอนาคต */
::ng-deep .mat-button { color: red; }

/* ✅ แนะนำ — ใช้ Global Styles แทน */
/* ใน styles.scss */
.mat-button { color: red; }

/* หรือใช้ CSS Custom Properties ที่ Material รองรับ */
:root { --mdc-text-button-label-text-color: red; }
```

---

## สรุป

| Mode | การ Scope Styles | ข้อดี | ข้อเสีย |
|------|-----------------|-------|---------|
| **Emulated** | Attribute-based | ทำงานทุก Browser | ยังมีบาง Edge Cases |
| **None** | ไม่มี (Global) | ง่าย | Styles รั่วทั่วแอป |
| **ShadowDom** | Browser Native | แยกสมบูรณ์ | Support บาง Browser |

| Selector | การใช้งาน |
|----------|-----------|
| `:host` | Host Element ของ Component |
| `:host(.class)` | Host เมื่อมี CSS Class |
| `:host-context(.parent)` | Host เมื่อ Parent มี Class |
| `::ng-deep` | Override Nested Styles (deprecated) |

View Encapsulation เป็นพื้นฐานสำคัญในการสร้าง Reusable Components ที่ Styles ไม่กระทบกัน การเข้าใจความแตกต่างระหว่าง Global Styles และ Component Styles จะช่วยให้สามารถออกแบบระบบ Design Token และ Theming ที่ดีได้

---

## สรุปภาพรวม Part 26-35

เราได้เรียนรู้หัวข้อขั้นสูงสำหรับ Angular ทั้งหมด 10 Parts:

| Part | หัวข้อ | สิ่งที่เรียน |
|------|--------|-------------|
| 26 | NgRx Store | Actions, Reducers, Selectors, Store |
| 27 | NgRx Effects | createEffect, Side Effects, Error Handling |
| 28 | NgRx Selectors | Memoization, Derived State, Parameterized Selectors |
| 29 | NgRx Entity | EntityState, EntityAdapter, CRUD Operations |
| 30 | Angular Signals | signal(), computed(), effect(), Signal Inputs |
| 31 | Standalone Components | bootstrapApplication, provideRouter, Migration |
| 32 | Advanced Routing | Resolver, Preloading, Router Events, Guards |
| 33 | Dynamic Components | ViewContainerRef, createComponent, Toast System |
| 34 | Content Projection | ng-content, Multi-slot, ngTemplateOutlet |
| 35 | View Encapsulation | Emulated, None, ShadowDom, :host, Theming |
