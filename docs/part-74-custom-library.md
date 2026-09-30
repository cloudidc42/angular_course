# Part 74: สร้าง Angular Component Library ด้วย ng-packagr

## ทำไมต้องสร้าง Library

- แชร์ components ระหว่าง projects
- Publish ขึ้น npm
- สร้าง Design System ให้ทีม

---

## 1. สร้าง Library Project

```bash
# สร้าง Angular workspace
ng new my-workspace --create-application=false
cd my-workspace

# สร้าง library
ng generate library my-ui-lib

# สร้าง demo app
ng generate application demo-app
```

### โครงสร้างไฟล์

```
my-workspace/
├── projects/
│   ├── my-ui-lib/
│   │   ├── src/
│   │   │   ├── lib/
│   │   │   │   ├── button/
│   │   │   │   ├── input/
│   │   │   │   └── modal/
│   │   │   ├── public-api.ts    ← export ทุกอย่างที่นี่
│   │   │   └── index.ts
│   │   ├── ng-package.json
│   │   └── tsconfig.lib.json
│   └── demo-app/
├── angular.json
└── package.json
```

---

## 2. สร้าง Button Component

```typescript
// projects/my-ui-lib/src/lib/button/button.component.ts
import { 
  Component, Input, Output, EventEmitter, 
  HostListener, ChangeDetectionStrategy 
} from '@angular/core';

export type ButtonVariant = 'primary' | 'secondary' | 'danger' | 'ghost' | 'link';
export type ButtonSize = 'sm' | 'md' | 'lg';

@Component({
  selector: 'ui-button',
  template: `
    <button
      [type]="type"
      [disabled]="disabled || loading"
      [attr.aria-disabled]="disabled || loading"
      [attr.aria-busy]="loading"
      [class]="computedClasses"
      (click)="handleClick($event)"
    >
      <span *ngIf="loading" class="spinner" aria-hidden="true"></span>
      <span *ngIf="iconLeft && !loading" class="icon icon-left" aria-hidden="true">
        {{ iconLeft }}
      </span>
      <span class="btn-text">
        <ng-content></ng-content>
      </span>
      <span *ngIf="iconRight" class="icon icon-right" aria-hidden="true">
        {{ iconRight }}
      </span>
    </button>
  `,
  styles: [`
    :host { display: inline-block; }
    
    button {
      display: inline-flex;
      align-items: center;
      gap: 8px;
      font-family: inherit;
      font-weight: 500;
      border-radius: 6px;
      cursor: pointer;
      transition: all 0.2s;
      border: 2px solid transparent;
      outline: none;
    }
    
    button:focus-visible {
      outline: 2px solid #2196f3;
      outline-offset: 2px;
    }
    
    /* Sizes */
    .btn-sm { padding: 6px 12px; font-size: 13px; }
    .btn-md { padding: 8px 16px; font-size: 14px; }
    .btn-lg { padding: 12px 24px; font-size: 16px; }
    
    /* Variants */
    .btn-primary { background: #2196f3; color: white; }
    .btn-primary:hover:not(:disabled) { background: #1976d2; }
    
    .btn-secondary { background: transparent; border-color: #2196f3; color: #2196f3; }
    .btn-secondary:hover:not(:disabled) { background: #e3f2fd; }
    
    .btn-danger { background: #f44336; color: white; }
    .btn-danger:hover:not(:disabled) { background: #d32f2f; }
    
    .btn-ghost { background: transparent; color: #666; }
    .btn-ghost:hover:not(:disabled) { background: #f5f5f5; }
    
    .btn-link { background: transparent; color: #2196f3; padding-left: 0; padding-right: 0; }
    .btn-link:hover:not(:disabled) { text-decoration: underline; }
    
    /* States */
    button:disabled { opacity: 0.5; cursor: not-allowed; }
    .btn-loading { position: relative; }
    
    /* Full width */
    .btn-full { width: 100%; justify-content: center; }
    
    /* Spinner */
    .spinner {
      width: 14px;
      height: 14px;
      border: 2px solid rgba(255,255,255,0.3);
      border-top-color: white;
      border-radius: 50%;
      animation: spin 0.8s linear infinite;
    }
    .btn-secondary .spinner,
    .btn-ghost .spinner { border-color: rgba(0,0,0,0.2); border-top-color: #666; }
    
    @keyframes spin { to { transform: rotate(360deg); } }
  `],
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class ButtonComponent {
  @Input() variant: ButtonVariant = 'primary';
  @Input() size: ButtonSize = 'md';
  @Input() type: 'button' | 'submit' | 'reset' = 'button';
  @Input() disabled = false;
  @Input() loading = false;
  @Input() fullWidth = false;
  @Input() iconLeft = '';
  @Input() iconRight = '';
  
  @Output() clicked = new EventEmitter<MouseEvent>();

  get computedClasses(): string {
    return [
      `btn-${this.variant}`,
      `btn-${this.size}`,
      this.loading ? 'btn-loading' : '',
      this.fullWidth ? 'btn-full' : ''
    ].filter(Boolean).join(' ');
  }

  handleClick(event: MouseEvent): void {
    if (!this.disabled && !this.loading) {
      this.clicked.emit(event);
    }
  }
}
```

---

## 3. Input Component

```typescript
// projects/my-ui-lib/src/lib/input/input.component.ts
import { 
  Component, Input, Output, EventEmitter, 
  forwardRef, ChangeDetectionStrategy 
} from '@angular/core';
import { ControlValueAccessor, NG_VALUE_ACCESSOR } from '@angular/forms';

@Component({
  selector: 'ui-input',
  template: `
    <div class="input-wrapper" [class.has-error]="hasError" [class.disabled]="disabled">
      <label *ngIf="label" [for]="inputId" class="input-label">
        {{ label }}
        <span *ngIf="required" class="required" aria-hidden="true">*</span>
      </label>
      
      <div class="input-container">
        <span *ngIf="prefixIcon" class="input-icon prefix">{{ prefixIcon }}</span>
        <input
          [id]="inputId"
          [type]="type"
          [placeholder]="placeholder"
          [disabled]="disabled"
          [readonly]="readonly"
          [value]="value"
          [attr.aria-required]="required"
          [attr.aria-invalid]="hasError"
          [attr.aria-describedby]="describedBy"
          (input)="onInput($event)"
          (blur)="onBlur()"
          (focus)="onFocus()"
          class="input-field"
          [class.has-prefix]="prefixIcon"
          [class.has-suffix]="suffixIcon || clearable"
        >
        <span *ngIf="suffixIcon" class="input-icon suffix">{{ suffixIcon }}</span>
        <button 
          *ngIf="clearable && value" 
          class="clear-btn" 
          type="button"
          (click)="clear()"
          aria-label="ล้างข้อความ"
        >✕</button>
      </div>
      
      <div class="input-footer">
        <span 
          *ngIf="errorMessage && hasError"
          [id]="errorId"
          class="error-message" 
          role="alert"
        >
          {{ errorMessage }}
        </span>
        <span *ngIf="hint && !hasError" [id]="hintId" class="hint">{{ hint }}</span>
        <span *ngIf="maxLength" class="char-count" [class.over]="value.length > maxLength">
          {{ value.length }}/{{ maxLength }}
        </span>
      </div>
    </div>
  `,
  styles: [`
    .input-wrapper { display: flex; flex-direction: column; gap: 4px; }
    .input-label { font-size: 14px; font-weight: 500; color: #333; }
    .required { color: #f44336; margin-left: 2px; }
    .input-container { position: relative; display: flex; align-items: center; }
    .input-field {
      width: 100%;
      padding: 8px 12px;
      border: 1px solid #ddd;
      border-radius: 6px;
      font-size: 14px;
      transition: border-color 0.2s;
      outline: none;
      box-sizing: border-box;
    }
    .input-field:focus { border-color: #2196f3; box-shadow: 0 0 0 3px rgba(33,150,243,0.1); }
    .has-error .input-field { border-color: #f44336; }
    .has-error .input-field:focus { box-shadow: 0 0 0 3px rgba(244,67,54,0.1); }
    .disabled .input-field { background: #f5f5f5; cursor: not-allowed; }
    .input-icon { position: absolute; color: #999; font-size: 16px; }
    .prefix { left: 10px; }
    .suffix { right: 10px; }
    .has-prefix { padding-left: 34px; }
    .has-suffix { padding-right: 34px; }
    .clear-btn {
      position: absolute; right: 8px;
      background: none; border: none; cursor: pointer;
      color: #999; font-size: 14px;
      padding: 2px; border-radius: 50%;
    }
    .clear-btn:hover { background: #f5f5f5; color: #333; }
    .error-message { font-size: 12px; color: #f44336; }
    .hint { font-size: 12px; color: #999; }
    .char-count { font-size: 12px; color: #999; margin-left: auto; }
    .char-count.over { color: #f44336; }
    .input-footer { display: flex; gap: 8px; }
  `],
  providers: [
    {
      provide: NG_VALUE_ACCESSOR,
      useExisting: forwardRef(() => InputComponent),
      multi: true
    }
  ],
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class InputComponent implements ControlValueAccessor {
  @Input() label = '';
  @Input() placeholder = '';
  @Input() type = 'text';
  @Input() hint = '';
  @Input() errorMessage = '';
  @Input() hasError = false;
  @Input() required = false;
  @Input() disabled = false;
  @Input() readonly = false;
  @Input() clearable = false;
  @Input() prefixIcon = '';
  @Input() suffixIcon = '';
  @Input() maxLength?: number;
  
  @Output() valueChange = new EventEmitter<string>();
  @Output() focused = new EventEmitter<void>();
  @Output() blurred = new EventEmitter<void>();

  value = '';
  inputId = `ui-input-${Math.random().toString(36).slice(2)}`;
  errorId = `${this.inputId}-error`;
  hintId = `${this.inputId}-hint`;

  get describedBy(): string {
    const ids: string[] = [];
    if (this.hint) ids.push(this.hintId);
    if (this.hasError && this.errorMessage) ids.push(this.errorId);
    return ids.join(' ') || undefined as any;
  }

  // ControlValueAccessor
  private onChange: (value: string) => void = () => {};
  private onTouched: () => void = () => {};

  writeValue(value: string): void {
    this.value = value ?? '';
  }

  registerOnChange(fn: (value: string) => void): void {
    this.onChange = fn;
  }

  registerOnTouched(fn: () => void): void {
    this.onTouched = fn;
  }

  setDisabledState(isDisabled: boolean): void {
    this.disabled = isDisabled;
  }

  onInput(event: Event): void {
    this.value = (event.target as HTMLInputElement).value;
    this.onChange(this.value);
    this.valueChange.emit(this.value);
  }

  onBlur(): void {
    this.onTouched();
    this.blurred.emit();
  }

  onFocus(): void {
    this.focused.emit();
  }

  clear(): void {
    this.value = '';
    this.onChange('');
    this.valueChange.emit('');
  }
}
```

---

## 4. Public API และ Module

```typescript
// projects/my-ui-lib/src/lib/my-ui-lib.module.ts
import { NgModule } from '@angular/core';
import { CommonModule } from '@angular/common';
import { ButtonComponent } from './button/button.component';
import { InputComponent } from './input/input.component';

@NgModule({
  declarations: [ButtonComponent, InputComponent],
  imports: [CommonModule],
  exports: [ButtonComponent, InputComponent]
})
export class MyUiLibModule {}

// projects/my-ui-lib/src/public-api.ts
export * from './lib/my-ui-lib.module';
export * from './lib/button/button.component';
export * from './lib/input/input.component';
```

---

## 5. Build และ Publish

```bash
# Build library
ng build my-ui-lib

# ผลลัพธ์อยู่ใน dist/my-ui-lib/

# Test local
cd dist/my-ui-lib
npm pack

# Publish npm
npm publish

# Semantic versioning
npm version patch   # 1.0.0 → 1.0.1
npm version minor   # 1.0.0 → 1.1.0
npm version major   # 1.0.0 → 2.0.0
```

### ng-package.json

```json
{
  "$schema": "../../node_modules/ng-packagr/ng-package.schema.json",
  "lib": {
    "entryFile": "src/public-api.ts"
  },
  "assets": ["./styles/**/*.scss"],
  "deleteDestPath": false
}
```

---

## 6. ใช้ Library ใน Project อื่น

```typescript
// app.module.ts (project อื่น)
import { MyUiLibModule } from 'my-ui-lib';

@NgModule({
  imports: [MyUiLibModule]
})
export class AppModule {}
```

```html
<!-- ใน template -->
<ui-button variant="primary" (clicked)="save()">
  บันทึกข้อมูล
</ui-button>

<ui-input 
  label="ชื่อผู้ใช้"
  placeholder="กรอกชื่อผู้ใช้"
  [clearable]="true"
  [(ngModel)]="username"
></ui-input>
```

---

## สรุป

| ขั้นตอน | คำสั่ง |
|---------|--------|
| สร้าง library | `ng generate library` |
| Build | `ng build my-lib` |
| Test local | `npm link` |
| Publish | `npm publish` |

### Checklist ก่อน Publish

- [ ] เพิ่ม `package.json` metadata (description, keywords, repository)
- [ ] เขียน README.md
- [ ] เพิ่ม CHANGELOG.md
- [ ] Test ครบทุก component
- [ ] ตั้งค่า `peerDependencies` ให้ถูกต้อง
- [ ] Build แบบ production mode
