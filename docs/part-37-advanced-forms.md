# Part 37: Advanced Forms ใน Angular

## บทนำ

Angular Reactive Forms มีความสามารถขั้นสูงที่ช่วยจัดการ Form ที่ซับซ้อนได้อย่างมีประสิทธิภาพ ในบทนี้เราจะเรียนรู้เกี่ยวกับ Dynamic Forms, FormArray ที่ซับซ้อน และ Cross-field Validation

---

## 1. Dynamic Forms

Dynamic Forms คือ Form ที่โครงสร้างเปลี่ยนแปลงได้ตาม runtime

```typescript
// app/models/form-field.model.ts
export interface FormFieldConfig {
  key: string;
  label: string;
  type: 'text' | 'email' | 'number' | 'select' | 'checkbox' | 'textarea' | 'date';
  required?: boolean;
  placeholder?: string;
  options?: { value: string; label: string }[];
  defaultValue?: any;
  validators?: ValidatorConfig[];
  row?: number;
  col?: number;
  colSpan?: number;
}

export interface ValidatorConfig {
  type: 'required' | 'minLength' | 'maxLength' | 'min' | 'max' | 'pattern' | 'email';
  value?: any;
  message: string;
}
```

```typescript
// app/services/form-builder.service.ts
import { Injectable } from '@angular/core';
import { FormGroup, FormControl, Validators, ValidatorFn, AbstractControl } from '@angular/forms';
import { FormFieldConfig, ValidatorConfig } from '../models/form-field.model';

@Injectable({ providedIn: 'root' })
export class DynamicFormBuilderService {

  buildForm(fields: FormFieldConfig[]): FormGroup {
    const group: Record<string, FormControl> = {};

    fields.forEach(field => {
      const validators = this.buildValidators(field.validators || []);
      group[field.key] = new FormControl(
        field.defaultValue ?? null,
        validators
      );
    });

    return new FormGroup(group);
  }

  private buildValidators(configs: ValidatorConfig[]): ValidatorFn[] {
    return configs.map(config => {
      switch (config.type) {
        case 'required': return Validators.required;
        case 'email': return Validators.email;
        case 'minLength': return Validators.minLength(config.value);
        case 'maxLength': return Validators.maxLength(config.value);
        case 'min': return Validators.min(config.value);
        case 'max': return Validators.max(config.value);
        case 'pattern': return Validators.pattern(config.value);
        default: return Validators.nullValidator;
      }
    });
  }

  getErrorMessage(control: AbstractControl, validators: ValidatorConfig[]): string {
    for (const config of validators) {
      if (control.hasError(config.type)) {
        return config.message;
      }
    }
    return '';
  }
}
```

```typescript
// app/components/dynamic-form/dynamic-form.component.ts
import { Component, Input, Output, EventEmitter, OnInit } from '@angular/core';
import { FormGroup } from '@angular/forms';
import { FormFieldConfig } from '../../models/form-field.model';
import { DynamicFormBuilderService } from '../../services/form-builder.service';

@Component({
  selector: 'app-dynamic-form',
  template: `
    <form [formGroup]="form" (ngSubmit)="onSubmit()">
      <div class="form-grid">
        <ng-container *ngFor="let field of fields">
          <div
            class="form-field"
            [style.grid-column]="'span ' + (field.colSpan || 1)"
          >
            <label [for]="field.key">
              {{ field.label }}
              <span *ngIf="field.required" class="required">*</span>
            </label>

            <!-- Text, Email, Number, Date -->
            <input
              *ngIf="['text', 'email', 'number', 'date'].includes(field.type)"
              [type]="field.type"
              [id]="field.key"
              [formControlName]="field.key"
              [placeholder]="field.placeholder || ''"
              class="form-control"
              [class.is-invalid]="isInvalid(field.key)"
            >

            <!-- Textarea -->
            <textarea
              *ngIf="field.type === 'textarea'"
              [id]="field.key"
              [formControlName]="field.key"
              [placeholder]="field.placeholder || ''"
              class="form-control"
              [class.is-invalid]="isInvalid(field.key)"
              rows="4"
            ></textarea>

            <!-- Select -->
            <select
              *ngIf="field.type === 'select'"
              [id]="field.key"
              [formControlName]="field.key"
              class="form-control"
              [class.is-invalid]="isInvalid(field.key)"
            >
              <option value="">-- เลือก --</option>
              <option
                *ngFor="let option of field.options"
                [value]="option.value"
              >{{ option.label }}</option>
            </select>

            <!-- Checkbox -->
            <div *ngIf="field.type === 'checkbox'" class="checkbox-wrapper">
              <input
                type="checkbox"
                [id]="field.key"
                [formControlName]="field.key"
              >
              <label [for]="field.key">{{ field.placeholder }}</label>
            </div>

            <!-- Error message -->
            <div *ngIf="isInvalid(field.key)" class="error-message">
              {{ getError(field) }}
            </div>
          </div>
        </ng-container>
      </div>

      <div class="form-actions">
        <button type="button" (click)="onReset()">ล้างข้อมูล</button>
        <button type="submit" [disabled]="form.invalid">บันทึก</button>
      </div>
    </form>
  `,
  styles: [`
    .form-grid { display: grid; grid-template-columns: repeat(2, 1fr); gap: 16px; }
    .form-field { display: flex; flex-direction: column; gap: 4px; }
    .form-control { padding: 8px; border: 1px solid #ddd; border-radius: 4px; }
    .is-invalid { border-color: red; }
    .error-message { color: red; font-size: 12px; }
    .required { color: red; }
    .form-actions { margin-top: 16px; display: flex; gap: 8px; justify-content: flex-end; }
  `]
})
export class DynamicFormComponent implements OnInit {
  @Input() fields: FormFieldConfig[] = [];
  @Output() formSubmit = new EventEmitter<any>();

  form!: FormGroup;

  constructor(private formBuilder: DynamicFormBuilderService) {}

  ngOnInit() {
    this.form = this.formBuilder.buildForm(this.fields);
  }

  isInvalid(key: string): boolean {
    const control = this.form.get(key);
    return !!(control?.invalid && control?.touched);
  }

  getError(field: FormFieldConfig): string {
    const control = this.form.get(field.key);
    if (!control) return '';
    return this.formBuilder.getErrorMessage(control, field.validators || []);
  }

  onSubmit() {
    if (this.form.valid) {
      this.formSubmit.emit(this.form.value);
    } else {
      this.form.markAllAsTouched();
    }
  }

  onReset() {
    this.form.reset();
  }
}
```

---

## 2. FormArray ที่ซับซ้อน

### 2.1 FormArray พื้นฐาน

```typescript
// app/components/order-form/order-form.component.ts
import { Component, OnInit } from '@angular/core';
import {
  FormBuilder,
  FormGroup,
  FormArray,
  Validators,
  AbstractControl
} from '@angular/forms';

interface OrderItem {
  productId: string;
  name: string;
  quantity: number;
  price: number;
}

@Component({
  selector: 'app-order-form',
  template: `
    <form [formGroup]="orderForm" (ngSubmit)="onSubmit()">
      <h2>สร้างคำสั่งซื้อ</h2>

      <!-- ข้อมูลลูกค้า -->
      <div formGroupName="customer">
        <h3>ข้อมูลลูกค้า</h3>
        <div class="form-row">
          <label>ชื่อ:</label>
          <input formControlName="name" placeholder="ชื่อลูกค้า">
          <div *ngIf="customerName?.invalid && customerName?.touched" class="error">
            กรุณาใส่ชื่อลูกค้า
          </div>
        </div>
        <div class="form-row">
          <label>ที่อยู่:</label>
          <textarea formControlName="address" placeholder="ที่อยู่จัดส่ง"></textarea>
        </div>
      </div>

      <!-- รายการสินค้า -->
      <div>
        <h3>รายการสินค้า</h3>
        <div formArrayName="items">
          <div
            *ngFor="let item of itemsArray.controls; let i = index"
            [formGroupName]="i"
            class="item-row"
          >
            <select formControlName="productId" (change)="onProductChange(i)">
              <option value="">-- เลือกสินค้า --</option>
              <option *ngFor="let p of products" [value]="p.id">{{ p.name }}</option>
            </select>

            <input
              type="number"
              formControlName="quantity"
              placeholder="จำนวน"
              min="1"
            >

            <input
              type="number"
              formControlName="price"
              placeholder="ราคา"
              [readonly]="true"
            >

            <span class="subtotal">฿{{ getSubtotal(i) | number:'1.2-2' }}</span>

            <button type="button" (click)="removeItem(i)">ลบ</button>
          </div>
        </div>

        <button type="button" (click)="addItem()">+ เพิ่มสินค้า</button>
      </div>

      <!-- สรุป -->
      <div class="summary">
        <p>ราคารวม: ฿{{ totalPrice | number:'1.2-2' }}</p>
        <p>ภาษี (7%): ฿{{ taxAmount | number:'1.2-2' }}</p>
        <p><strong>ยอดรวมทั้งหมด: ฿{{ grandTotal | number:'1.2-2' }}</strong></p>
      </div>

      <button type="submit" [disabled]="orderForm.invalid">สั่งซื้อ</button>
    </form>
  `
})
export class OrderFormComponent implements OnInit {
  orderForm!: FormGroup;

  products = [
    { id: 'P001', name: 'สินค้า A', price: 150 },
    { id: 'P002', name: 'สินค้า B', price: 250 },
    { id: 'P003', name: 'สินค้า C', price: 350 }
  ];

  constructor(private fb: FormBuilder) {}

  ngOnInit() {
    this.orderForm = this.fb.group({
      customer: this.fb.group({
        name: ['', [Validators.required, Validators.minLength(2)]],
        address: ['', Validators.required]
      }),
      items: this.fb.array([this.createItem()])
    });
  }

  get itemsArray(): FormArray {
    return this.orderForm.get('items') as FormArray;
  }

  get customerName(): AbstractControl | null {
    return this.orderForm.get('customer.name');
  }

  createItem(): FormGroup {
    return this.fb.group({
      productId: ['', Validators.required],
      name: [''],
      quantity: [1, [Validators.required, Validators.min(1)]],
      price: [0]
    });
  }

  addItem() {
    this.itemsArray.push(this.createItem());
  }

  removeItem(index: number) {
    if (this.itemsArray.length > 1) {
      this.itemsArray.removeAt(index);
    }
  }

  onProductChange(index: number) {
    const item = this.itemsArray.at(index);
    const productId = item.get('productId')?.value;
    const product = this.products.find(p => p.id === productId);

    if (product) {
      item.patchValue({
        name: product.name,
        price: product.price
      });
    }
  }

  getSubtotal(index: number): number {
    const item = this.itemsArray.at(index);
    const qty = item.get('quantity')?.value || 0;
    const price = item.get('price')?.value || 0;
    return qty * price;
  }

  get totalPrice(): number {
    return this.itemsArray.controls.reduce((sum, _, i) => sum + this.getSubtotal(i), 0);
  }

  get taxAmount(): number {
    return this.totalPrice * 0.07;
  }

  get grandTotal(): number {
    return this.totalPrice + this.taxAmount;
  }

  onSubmit() {
    if (this.orderForm.valid) {
      console.log('Order:', this.orderForm.value);
    } else {
      this.orderForm.markAllAsTouched();
    }
  }
}
```

---

## 3. Cross-field Validation

```typescript
// app/validators/cross-field.validators.ts
import { AbstractControl, ValidationErrors, ValidatorFn, FormGroup } from '@angular/forms';

// Validator สำหรับตรวจสอบว่าสองฟิลด์มีค่าตรงกัน
export function matchFieldsValidator(field1: string, field2: string): ValidatorFn {
  return (group: AbstractControl): ValidationErrors | null => {
    const val1 = group.get(field1)?.value;
    const val2 = group.get(field2)?.value;

    if (val1 !== val2) {
      // เพิ่ม error ให้ field2
      group.get(field2)?.setErrors({ fieldsMismatch: true });
      return { fieldsMismatch: { field1, field2 } };
    }

    // ล้าง error ถ้าตรงกัน
    const currentErrors = group.get(field2)?.errors;
    if (currentErrors) {
      const { fieldsMismatch, ...rest } = currentErrors;
      group.get(field2)?.setErrors(Object.keys(rest).length ? rest : null);
    }

    return null;
  };
}

// Validator สำหรับตรวจสอบช่วงวันที่
export function dateRangeValidator(startField: string, endField: string): ValidatorFn {
  return (group: AbstractControl): ValidationErrors | null => {
    const start = group.get(startField)?.value;
    const end = group.get(endField)?.value;

    if (start && end && new Date(start) > new Date(end)) {
      return { invalidDateRange: { start, end } };
    }

    return null;
  };
}

// Validator สำหรับตรวจสอบ budget
export function budgetValidator(spentField: string, budgetField: string): ValidatorFn {
  return (group: AbstractControl): ValidationErrors | null => {
    const spent = parseFloat(group.get(spentField)?.value) || 0;
    const budget = parseFloat(group.get(budgetField)?.value) || 0;

    if (budget > 0 && spent > budget) {
      return {
        overBudget: {
          spent,
          budget,
          excess: spent - budget
        }
      };
    }

    return null;
  };
}

// Validator ตรวจสอบอย่างน้อยหนึ่ง checkbox
export function atLeastOneCheckedValidator(minRequired = 1): ValidatorFn {
  return (group: AbstractControl): ValidationErrors | null => {
    const controls = (group as FormGroup).controls;
    const checkedCount = Object.values(controls)
      .filter(ctrl => ctrl.value === true).length;

    if (checkedCount < minRequired) {
      return { atLeastOneRequired: { required: minRequired, actual: checkedCount } };
    }

    return null;
  };
}
```

```typescript
// app/components/registration-form/registration-form.component.ts
import { Component, OnInit } from '@angular/core';
import { FormBuilder, FormGroup, Validators } from '@angular/forms';
import {
  matchFieldsValidator,
  dateRangeValidator
} from '../../validators/cross-field.validators';

@Component({
  selector: 'app-registration-form',
  template: `
    <form [formGroup]="registrationForm" (ngSubmit)="onSubmit()">
      <h2>ลงทะเบียน</h2>

      <!-- ข้อมูลพื้นฐาน -->
      <div class="section">
        <h3>ข้อมูลบัญชี</h3>

        <div class="form-group">
          <label>Username:</label>
          <input formControlName="username" placeholder="username">
          <div *ngIf="f['username'].invalid && f['username'].touched">
            <small *ngIf="f['username'].errors?.['required']">กรุณาใส่ username</small>
            <small *ngIf="f['username'].errors?.['minlength']">ต้องมีอย่างน้อย 4 ตัวอักษร</small>
          </div>
        </div>

        <div class="form-group">
          <label>Email:</label>
          <input type="email" formControlName="email" placeholder="email">
          <div *ngIf="f['email'].invalid && f['email'].touched">
            <small *ngIf="f['email'].errors?.['required']">กรุณาใส่ email</small>
            <small *ngIf="f['email'].errors?.['email']">รูปแบบ email ไม่ถูกต้อง</small>
          </div>
        </div>
      </div>

      <!-- รหัสผ่าน -->
      <div class="section" formGroupName="passwords">
        <h3>รหัสผ่าน</h3>

        <div class="form-group">
          <label>รหัสผ่าน:</label>
          <input type="password" formControlName="password" placeholder="รหัสผ่าน">
        </div>

        <div class="form-group">
          <label>ยืนยันรหัสผ่าน:</label>
          <input type="password" formControlName="confirmPassword" placeholder="ยืนยันรหัสผ่าน">
          <div *ngIf="passwordsGroup?.errors?.['fieldsMismatch']">
            <small class="error">รหัสผ่านไม่ตรงกัน</small>
          </div>
        </div>
      </div>

      <!-- ช่วงเวลาสมัคร -->
      <div class="section" formGroupName="subscription">
        <h3>ระยะเวลาสมาชิก</h3>

        <div class="form-group">
          <label>วันที่เริ่ม:</label>
          <input type="date" formControlName="startDate">
        </div>

        <div class="form-group">
          <label>วันที่สิ้นสุด:</label>
          <input type="date" formControlName="endDate">
          <div *ngIf="subscriptionGroup?.errors?.['invalidDateRange']">
            <small class="error">วันที่สิ้นสุดต้องมาหลังวันที่เริ่ม</small>
          </div>
        </div>
      </div>

      <button type="submit" [disabled]="registrationForm.invalid">ลงทะเบียน</button>
    </form>
  `
})
export class RegistrationFormComponent implements OnInit {
  registrationForm!: FormGroup;

  constructor(private fb: FormBuilder) {}

  ngOnInit() {
    this.registrationForm = this.fb.group({
      username: ['', [Validators.required, Validators.minLength(4)]],
      email: ['', [Validators.required, Validators.email]],
      passwords: this.fb.group(
        {
          password: ['', [Validators.required, Validators.minLength(8)]],
          confirmPassword: ['', Validators.required]
        },
        { validators: matchFieldsValidator('password', 'confirmPassword') }
      ),
      subscription: this.fb.group(
        {
          startDate: ['', Validators.required],
          endDate: ['', Validators.required]
        },
        { validators: dateRangeValidator('startDate', 'endDate') }
      )
    });
  }

  get f() {
    return this.registrationForm.controls;
  }

  get passwordsGroup() {
    return this.registrationForm.get('passwords');
  }

  get subscriptionGroup() {
    return this.registrationForm.get('subscription');
  }

  onSubmit() {
    if (this.registrationForm.valid) {
      console.log('Form value:', this.registrationForm.value);
    } else {
      this.registrationForm.markAllAsTouched();
    }
  }
}
```

---

## 4. Nested FormArrays

```typescript
// app/components/survey-form/survey-form.component.ts
import { Component, OnInit } from '@angular/core';
import { FormBuilder, FormGroup, FormArray, Validators } from '@angular/forms';

@Component({
  selector: 'app-survey-form',
  template: `
    <form [formGroup]="surveyForm" (ngSubmit)="onSubmit()">
      <h2>แบบสำรวจ</h2>

      <div formArrayName="sections">
        <div
          *ngFor="let section of sectionsArray.controls; let si = index"
          [formGroupName]="si"
          class="section"
        >
          <h3>หมวดหมู่ที่ {{ si + 1 }}</h3>
          <input formControlName="title" placeholder="ชื่อหมวดหมู่">
          <button type="button" (click)="removeSection(si)">ลบหมวดหมู่</button>

          <!-- คำถามใน section -->
          <div formArrayName="questions">
            <div
              *ngFor="let q of getQuestions(si).controls; let qi = index"
              [formGroupName]="qi"
              class="question"
            >
              <input formControlName="text" placeholder="คำถาม">
              <select formControlName="type">
                <option value="text">ข้อความ</option>
                <option value="rating">คะแนน (1-5)</option>
                <option value="yesno">ใช่/ไม่ใช่</option>
              </select>
              <button type="button" (click)="removeQuestion(si, qi)">ลบคำถาม</button>
            </div>
          </div>

          <button type="button" (click)="addQuestion(si)">+ เพิ่มคำถาม</button>
        </div>
      </div>

      <button type="button" (click)="addSection()">+ เพิ่มหมวดหมู่</button>
      <button type="submit">บันทึกแบบสำรวจ</button>
    </form>
  `
})
export class SurveyFormComponent implements OnInit {
  surveyForm!: FormGroup;

  constructor(private fb: FormBuilder) {}

  ngOnInit() {
    this.surveyForm = this.fb.group({
      title: ['', Validators.required],
      sections: this.fb.array([this.createSection()])
    });
  }

  get sectionsArray(): FormArray {
    return this.surveyForm.get('sections') as FormArray;
  }

  getQuestions(sectionIndex: number): FormArray {
    return this.sectionsArray.at(sectionIndex).get('questions') as FormArray;
  }

  createSection(): FormGroup {
    return this.fb.group({
      title: ['', Validators.required],
      questions: this.fb.array([this.createQuestion()])
    });
  }

  createQuestion(): FormGroup {
    return this.fb.group({
      text: ['', Validators.required],
      type: ['text', Validators.required]
    });
  }

  addSection() {
    this.sectionsArray.push(this.createSection());
  }

  removeSection(index: number) {
    this.sectionsArray.removeAt(index);
  }

  addQuestion(sectionIndex: number) {
    this.getQuestions(sectionIndex).push(this.createQuestion());
  }

  removeQuestion(sectionIndex: number, questionIndex: number) {
    this.getQuestions(sectionIndex).removeAt(questionIndex);
  }

  onSubmit() {
    if (this.surveyForm.valid) {
      console.log('Survey:', JSON.stringify(this.surveyForm.value, null, 2));
    }
  }
}
```

---

## 5. Form Value Changes และ Form State

```typescript
// app/components/smart-form/smart-form.component.ts
import { Component, OnInit, OnDestroy } from '@angular/core';
import { FormBuilder, FormGroup, Validators } from '@angular/forms';
import { Subject } from 'rxjs';
import { takeUntil, debounceTime, distinctUntilChanged } from 'rxjs/operators';

@Component({
  selector: 'app-smart-form',
  template: `
    <form [formGroup]="smartForm" (ngSubmit)="onSubmit()">
      <h2>Smart Form (Auto-save)</h2>

      <div class="save-status">
        <span *ngIf="saveStatus === 'saving'">กำลังบันทึก...</span>
        <span *ngIf="saveStatus === 'saved'">บันทึกแล้ว ✓</span>
        <span *ngIf="saveStatus === 'error'" class="error">เกิดข้อผิดพลาด!</span>
      </div>

      <div class="form-group">
        <label>ชื่อ:</label>
        <input formControlName="name">
      </div>

      <div class="form-group">
        <label>ประเภท:</label>
        <select formControlName="type">
          <option value="individual">บุคคล</option>
          <option value="company">บริษัท</option>
        </select>
      </div>

      <!-- แสดงเฉพาะเมื่อเลือก company -->
      <div *ngIf="showCompanyFields" class="form-group">
        <label>ชื่อบริษัท:</label>
        <input formControlName="companyName">
        <label>เลขทะเบียน:</label>
        <input formControlName="taxId">
      </div>

      <div class="form-state">
        <p>Form Valid: {{ smartForm.valid }}</p>
        <p>Form Dirty: {{ smartForm.dirty }}</p>
        <p>Form Touched: {{ smartForm.touched }}</p>
        <p>Form Pristine: {{ smartForm.pristine }}</p>
      </div>
    </form>
  `
})
export class SmartFormComponent implements OnInit, OnDestroy {
  smartForm!: FormGroup;
  saveStatus: 'idle' | 'saving' | 'saved' | 'error' = 'idle';
  showCompanyFields = false;
  private destroy$ = new Subject<void>();

  constructor(private fb: FormBuilder) {}

  ngOnInit() {
    this.smartForm = this.fb.group({
      name: ['', Validators.required],
      type: ['individual'],
      companyName: [''],
      taxId: ['']
    });

    // ติดตามการเปลี่ยนแปลงของ type field
    this.smartForm.get('type')!.valueChanges
      .pipe(takeUntil(this.destroy$))
      .subscribe(type => {
        this.showCompanyFields = type === 'company';
        this.updateCompanyValidators(type);
      });

    // Auto-save เมื่อข้อมูลเปลี่ยน
    this.smartForm.valueChanges
      .pipe(
        debounceTime(1000),
        distinctUntilChanged(),
        takeUntil(this.destroy$)
      )
      .subscribe(value => {
        if (this.smartForm.valid && this.smartForm.dirty) {
          this.autoSave(value);
        }
      });
  }

  private updateCompanyValidators(type: string) {
    const companyName = this.smartForm.get('companyName');
    const taxId = this.smartForm.get('taxId');

    if (type === 'company') {
      companyName?.setValidators([Validators.required]);
      taxId?.setValidators([Validators.required, Validators.pattern(/^\d{13}$/)]);
    } else {
      companyName?.clearValidators();
      taxId?.clearValidators();
    }

    companyName?.updateValueAndValidity();
    taxId?.updateValueAndValidity();
  }

  private autoSave(value: any) {
    this.saveStatus = 'saving';
    // จำลองการบันทึก
    setTimeout(() => {
      try {
        localStorage.setItem('form_draft', JSON.stringify(value));
        this.saveStatus = 'saved';
        setTimeout(() => this.saveStatus = 'idle', 2000);
      } catch {
        this.saveStatus = 'error';
      }
    }, 500);
  }

  onSubmit() {
    if (this.smartForm.valid) {
      console.log('Submit:', this.smartForm.value);
    }
  }

  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

---

## สรุป

- **Dynamic Forms** ช่วยสร้าง Form จาก configuration แทนการเขียน HTML ตรง ๆ
- **FormArray** ใช้สำหรับรายการที่จำนวนไม่แน่นอน
- **Nested FormArray** ใช้สำหรับโครงสร้างที่ซ้อนกันหลายชั้น
- **Cross-field Validation** ใช้ validator ที่ group level แทน field level
- **valueChanges** ใช้ติดตามการเปลี่ยนแปลงเพื่อทำ auto-save หรือ conditional logic
