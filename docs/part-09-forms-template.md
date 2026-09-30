# Part 09 — Template-driven Forms

## สารบัญ

1. [FormsModule](#formsmodule)
2. [ngModel Two-Way Binding](#ngmodel-two-way-binding)
3. [ngForm และ ngModelGroup](#ngform-และ-ngmodelgroup)
4. [Built-in Validators](#built-in-validators)
5. [Showing Validation Errors](#showing-validation-errors)
6. [Form State](#form-state)
7. [FormControl Reference](#formcontrol-reference)
8. [Submit Handling](#submit-handling)
9. [Workshop: User Registration Form](#workshop-user-registration-form)

---

## FormsModule

Template-driven Forms ใช้ Directive จาก `FormsModule` เพื่อจัดการ Form โดยตรงใน HTML Template

### ติดตั้ง FormsModule

```typescript
// src/app/app.module.ts
import { NgModule } from '@angular/core';
import { BrowserModule } from '@angular/platform-browser';
import { FormsModule } from '@angular/forms';  // ← Import ตรงนี้

import { AppComponent } from './app.component';

@NgModule({
  declarations: [AppComponent],
  imports: [
    BrowserModule,
    FormsModule  // ← เพิ่มใน imports
  ],
  bootstrap: [AppComponent]
})
export class AppModule { }
```

### ความแตกต่างระหว่าง Template-driven และ Reactive Forms

| Feature | Template-driven | Reactive |
|---------|----------------|---------|
| ตั้งค่าใน | HTML Template | TypeScript Class |
| Module | `FormsModule` | `ReactiveFormsModule` |
| Form Model | Angular สร้างให้ | เราสร้างเอง |
| Validation | HTML Attributes | Validators Functions |
| ความซับซ้อน | เรียบง่าย | ซับซ้อนกว่า แต่ยืดหยุ่น |
| ทดสอบ | ยากกว่า | ง่ายกว่า |
| ใช้เมื่อ | Form เรียบง่าย | Form ซับซ้อน, Dynamic |

---

## ngModel Two-Way Binding

`ngModel` เป็น Directive หลักของ Template-driven Forms ที่ทำ Two-way Data Binding ระหว่าง Input และ Property ของ Component

### One-way vs Two-way Binding

```html
<!-- One-way Binding (Property → View) -->
<input [value]="username">

<!-- One-way Binding (View → Property) -->
<input (input)="username = $event.target.value">

<!-- Two-way Binding (Banana in a Box Syntax) -->
<input [(ngModel)]="username">
<!-- เทียบเท่ากับ: -->
<input [ngModel]="username" (ngModelChange)="username = $event">
```

### การใช้ ngModel พื้นฐาน

```typescript
// src/app/pages/contact/contact.component.ts
import { Component } from '@angular/core';

@Component({
  selector: 'app-contact',
  templateUrl: './contact.component.html'
})
export class ContactComponent {
  name = '';
  email = '';
  subject = '';
  message = '';
  subscribeNewsletter = false;
  gender = '';
  favoriteCategory = 'Electronics';

  categories = ['Electronics', 'Clothing', 'Books', 'Sports', 'Food'];

  preview(): void {
    console.log({
      name: this.name,
      email: this.email,
      subject: this.subject,
      message: this.message,
      subscribeNewsletter: this.subscribeNewsletter,
      gender: this.gender,
      favoriteCategory: this.favoriteCategory
    });
  }
}
```

```html
<!-- contact.component.html -->
<form>
  <!-- Text Input -->
  <div class="form-group">
    <label for="name">ชื่อ:</label>
    <input
      type="text"
      id="name"
      [(ngModel)]="name"
      name="name"
      placeholder="กรอกชื่อของคุณ"
    >
    <p>ชื่อปัจจุบัน: {{ name }}</p>
  </div>

  <!-- Email Input -->
  <div class="form-group">
    <label for="email">อีเมล:</label>
    <input
      type="email"
      id="email"
      [(ngModel)]="email"
      name="email"
      placeholder="example@email.com"
    >
  </div>

  <!-- Textarea -->
  <div class="form-group">
    <label for="message">ข้อความ:</label>
    <textarea
      id="message"
      [(ngModel)]="message"
      name="message"
      rows="4"
      placeholder="กรอกข้อความ..."
    ></textarea>
    <small>{{ message.length }}/500 ตัวอักษร</small>
  </div>

  <!-- Checkbox -->
  <div class="form-group">
    <label>
      <input type="checkbox" [(ngModel)]="subscribeNewsletter" name="newsletter">
      รับข่าวสารทางอีเมล
    </label>
  </div>

  <!-- Radio Buttons -->
  <div class="form-group">
    <label>เพศ:</label>
    <label>
      <input type="radio" [(ngModel)]="gender" name="gender" value="male"> ชาย
    </label>
    <label>
      <input type="radio" [(ngModel)]="gender" name="gender" value="female"> หญิง
    </label>
    <label>
      <input type="radio" [(ngModel)]="gender" name="gender" value="other"> อื่น ๆ
    </label>
  </div>

  <!-- Select (Dropdown) -->
  <div class="form-group">
    <label for="category">หมวดหมู่ที่ชื่นชอบ:</label>
    <select id="category" [(ngModel)]="favoriteCategory" name="category">
      <option *ngFor="let cat of categories" [value]="cat">{{ cat }}</option>
    </select>
  </div>

  <button type="button" (click)="preview()">ดูตัวอย่าง</button>
</form>
```

### ngModel กับ Object

```typescript
// ใช้ Object แทน Property แยก ๆ
export interface ContactForm {
  name: string;
  email: string;
  message: string;
}

@Component({ selector: 'app-contact', template: `...` })
export class ContactComponent {
  formData: ContactForm = {
    name: '',
    email: '',
    message: ''
  };
}
```

```html
<input [(ngModel)]="formData.name" name="name">
<input [(ngModel)]="formData.email" name="email">
<textarea [(ngModel)]="formData.message" name="message"></textarea>
```

---

## ngForm และ ngModelGroup

### ngForm — เข้าถึง Form ทั้งหมด

```html
<!-- อ้างอิง Form ด้วย Template Reference Variable #myForm="ngForm" -->
<form #myForm="ngForm" (ngSubmit)="onSubmit(myForm)">

  <input [(ngModel)]="name" name="name" required>
  <input [(ngModel)]="email" name="email" email required>

  <button type="submit" [disabled]="myForm.invalid">ส่งข้อมูล</button>

  <!-- Debug: แสดงสถานะ Form -->
  <pre>Valid: {{ myForm.valid }}</pre>
  <pre>Value: {{ myForm.value | json }}</pre>
</form>
```

```typescript
import { NgForm } from '@angular/forms';

@Component({ selector: 'app-form', template: `...` })
export class FormComponent {
  name = '';
  email = '';

  onSubmit(form: NgForm): void {
    if (form.valid) {
      console.log('Form Value:', form.value);
      console.log('Form Valid:', form.valid);

      // Reset Form หลัง Submit
      form.reset();
    }
  }
}
```

### ngModelGroup — จัดกลุ่ม Input

```typescript
export interface Address {
  street: string;
  city: string;
  province: string;
  postalCode: string;
}

export interface UserFormData {
  firstName: string;
  lastName: string;
  email: string;
  address: Address;
}

@Component({ selector: 'app-user-form', template: `...` })
export class UserFormComponent {
  formData: UserFormData = {
    firstName: '',
    lastName: '',
    email: '',
    address: {
      street: '',
      city: '',
      province: '',
      postalCode: ''
    }
  };
}
```

```html
<form #userForm="ngForm" (ngSubmit)="onSubmit(userForm)">

  <!-- ข้อมูลส่วนตัว -->
  <div class="section">
    <h3>ข้อมูลส่วนตัว</h3>

    <input [(ngModel)]="formData.firstName" name="firstName" required>
    <input [(ngModel)]="formData.lastName" name="lastName" required>
    <input [(ngModel)]="formData.email" name="email" email required>
  </div>

  <!-- ที่อยู่ — จัดกลุ่มด้วย ngModelGroup -->
  <div ngModelGroup="address" #addressGroup="ngModelGroup" class="section">
    <h3>ที่อยู่</h3>

    <input [(ngModel)]="formData.address.street" name="street" required>
    <input [(ngModel)]="formData.address.city" name="city" required>
    <input [(ngModel)]="formData.address.province" name="province" required>
    <input [(ngModel)]="formData.address.postalCode"
           name="postalCode"
           pattern="[0-9]{5}"
           required>

    <div *ngIf="addressGroup.invalid && addressGroup.touched" class="error">
      กรุณากรอกที่อยู่ให้ครบถ้วน
    </div>
  </div>

  <button type="submit" [disabled]="userForm.invalid">บันทึก</button>

  <!-- แสดงค่าทั้งหมด -->
  <pre>{{ userForm.value | json }}</pre>

</form>
```

---

## Built-in Validators

Angular มี Validators ในตัวหลายตัวที่ใช้เป็น HTML Attribute

### ตาราง Built-in Validators

| Validator | HTML Attribute | คำอธิบาย |
|-----------|----------------|---------|
| Required | `required` | ต้องกรอก |
| Min Length | `minlength="n"` | ขั้นต่ำ n ตัวอักษร |
| Max Length | `maxlength="n"` | สูงสุด n ตัวอักษร |
| Pattern | `pattern="regex"` | ต้องตรงกับ Regex |
| Email | `email` | ต้องเป็น Email format |
| Min (number) | `min="n"` | ค่าต้องไม่น้อยกว่า n |
| Max (number) | `max="n"` | ค่าต้องไม่มากกว่า n |

### ตัวอย่างการใช้ทุก Validator

```html
<form #validForm="ngForm">

  <!-- required — ต้องกรอก -->
  <div class="form-group">
    <label>ชื่อผู้ใช้ *</label>
    <input
      type="text"
      [(ngModel)]="username"
      name="username"
      required
      #usernameCtrl="ngModel"
    >
    <div *ngIf="usernameCtrl.invalid && usernameCtrl.touched">
      <span *ngIf="usernameCtrl.errors?.['required']">กรุณากรอกชื่อผู้ใช้</span>
    </div>
  </div>

  <!-- minlength + maxlength -->
  <div class="form-group">
    <label>รหัสผ่าน (6-20 ตัวอักษร) *</label>
    <input
      type="password"
      [(ngModel)]="password"
      name="password"
      required
      minlength="6"
      maxlength="20"
      #passwordCtrl="ngModel"
    >
    <div *ngIf="passwordCtrl.invalid && passwordCtrl.touched">
      <span *ngIf="passwordCtrl.errors?.['required']">กรุณากรอกรหัสผ่าน</span>
      <span *ngIf="passwordCtrl.errors?.['minlength']">
        รหัสผ่านต้องมีอย่างน้อย
        {{ passwordCtrl.errors?.['minlength'].requiredLength }} ตัวอักษร
        (ปัจจุบัน: {{ passwordCtrl.errors?.['minlength'].actualLength }})
      </span>
      <span *ngIf="passwordCtrl.errors?.['maxlength']">
        รหัสผ่านต้องไม่เกิน {{ passwordCtrl.errors?.['maxlength'].requiredLength }} ตัวอักษร
      </span>
    </div>
  </div>

  <!-- email -->
  <div class="form-group">
    <label>อีเมล *</label>
    <input
      type="email"
      [(ngModel)]="email"
      name="email"
      required
      email
      #emailCtrl="ngModel"
    >
    <div *ngIf="emailCtrl.invalid && emailCtrl.touched">
      <span *ngIf="emailCtrl.errors?.['required']">กรุณากรอกอีเมล</span>
      <span *ngIf="emailCtrl.errors?.['email']">รูปแบบอีเมลไม่ถูกต้อง</span>
    </div>
  </div>

  <!-- pattern — เบอร์โทรไทย -->
  <div class="form-group">
    <label>เบอร์โทรศัพท์</label>
    <input
      type="tel"
      [(ngModel)]="phone"
      name="phone"
      pattern="^(0[689][0-9]{8})$"
      #phoneCtrl="ngModel"
    >
    <div *ngIf="phoneCtrl.invalid && phoneCtrl.touched">
      <span *ngIf="phoneCtrl.errors?.['pattern']">
        กรุณากรอกเบอร์โทรให้ถูกต้อง (เช่น 0812345678)
      </span>
    </div>
  </div>

  <!-- min + max สำหรับตัวเลข -->
  <div class="form-group">
    <label>อายุ (18-100 ปี) *</label>
    <input
      type="number"
      [(ngModel)]="age"
      name="age"
      required
      min="18"
      max="100"
      #ageCtrl="ngModel"
    >
    <div *ngIf="ageCtrl.invalid && ageCtrl.touched">
      <span *ngIf="ageCtrl.errors?.['required']">กรุณากรอกอายุ</span>
      <span *ngIf="ageCtrl.errors?.['min']">
        อายุต้องไม่น้อยกว่า {{ ageCtrl.errors?.['min'].min }} ปี
      </span>
      <span *ngIf="ageCtrl.errors?.['max']">
        อายุต้องไม่เกิน {{ ageCtrl.errors?.['max'].max }} ปี
      </span>
    </div>
  </div>

</form>
```

### Pattern สำหรับกรณีทั่วไป

```html
<!-- รหัสไปรษณีย์ไทย (5 หลัก) -->
<input pattern="[0-9]{5}">

<!-- เบอร์โทรไทย -->
<input pattern="^(0[689][0-9]{8})$">

<!-- เลขบัตรประชาชน (13 หลัก) -->
<input pattern="[0-9]{13}">

<!-- ตัวอักษรและตัวเลขเท่านั้น -->
<input pattern="[a-zA-Z0-9]+">

<!-- ชื่อภาษาไทย -->
<input pattern="[฀-๿\s]+">

<!-- URL -->
<input pattern="https?://.+">

<!-- รหัสผ่านแบบซับซ้อน (ตัวพิมพ์ใหญ่ + เล็ก + ตัวเลข อย่างน้อย 8 ตัว) -->
<input pattern="(?=.*[a-z])(?=.*[A-Z])(?=.*[0-9]).{8,}">
```

---

## Showing Validation Errors

### Pattern การแสดง Error

```html
<!-- รูปแบบมาตรฐาน -->
<div class="form-group">
  <label for="fieldName">Label *</label>
  <input
    type="text"
    id="fieldName"
    [(ngModel)]="fieldValue"
    name="fieldName"
    required
    #fieldCtrl="ngModel"
    [class.is-invalid]="fieldCtrl.invalid && fieldCtrl.touched"
    [class.is-valid]="fieldCtrl.valid && fieldCtrl.touched"
  >
  <!-- แสดง Error เฉพาะเมื่อ Invalid และ Touched -->
  <div class="invalid-feedback" *ngIf="fieldCtrl.invalid && fieldCtrl.touched">
    <span *ngIf="fieldCtrl.errors?.['required']">กรุณากรอกข้อมูล</span>
    <span *ngIf="fieldCtrl.errors?.['minlength']">ข้อมูลสั้นเกินไป</span>
    <span *ngIf="fieldCtrl.errors?.['maxlength']">ข้อมูลยาวเกินไป</span>
  </div>
  <div class="valid-feedback" *ngIf="fieldCtrl.valid && fieldCtrl.touched">
    ถูกต้อง ✓
  </div>
</div>
```

### แสดง Error แบบ Real-time (ขณะพิมพ์)

```html
<!-- แสดง Error ทันทีที่มีการเปลี่ยนแปลง -->
<input
  [(ngModel)]="email"
  name="email"
  email
  required
  #emailCtrl="ngModel"
  [class.is-invalid]="emailCtrl.invalid && emailCtrl.dirty"
>
<div *ngIf="emailCtrl.invalid && emailCtrl.dirty">
  <small class="text-danger" *ngIf="emailCtrl.errors?.['required']">
    กรุณากรอกอีเมล
  </small>
  <small class="text-danger" *ngIf="emailCtrl.errors?.['email']">
    รูปแบบอีเมลไม่ถูกต้อง
  </small>
</div>
```

### Error Message Component (Reusable)

```typescript
// src/app/components/validation-message/validation-message.component.ts
import { Component, Input } from '@angular/core';
import { AbstractControl } from '@angular/forms';

@Component({
  selector: 'app-validation-message',
  template: `
    <div class="validation-messages" *ngIf="control?.invalid && (control?.dirty || control?.touched)">
      <small class="error-msg" *ngIf="control?.errors?.['required']">
        {{ label || 'ช่องนี้' }} จำเป็นต้องกรอก
      </small>
      <small class="error-msg" *ngIf="control?.errors?.['email']">
        รูปแบบอีเมลไม่ถูกต้อง
      </small>
      <small class="error-msg" *ngIf="control?.errors?.['minlength']">
        ต้องมีอย่างน้อย {{ control?.errors?.['minlength'].requiredLength }} ตัวอักษร
      </small>
      <small class="error-msg" *ngIf="control?.errors?.['maxlength']">
        ต้องไม่เกิน {{ control?.errors?.['maxlength'].requiredLength }} ตัวอักษร
      </small>
      <small class="error-msg" *ngIf="control?.errors?.['pattern']">
        รูปแบบไม่ถูกต้อง
      </small>
      <small class="error-msg" *ngIf="control?.errors?.['min']">
        ค่าต้องไม่น้อยกว่า {{ control?.errors?.['min'].min }}
      </small>
      <small class="error-msg" *ngIf="control?.errors?.['max']">
        ค่าต้องไม่มากกว่า {{ control?.errors?.['max'].max }}
      </small>
      <small class="error-msg" *ngIf="control?.errors?.['custom']">
        {{ control?.errors?.['custom'] }}
      </small>
    </div>
  `,
  styles: [`.error-msg { color: #dc3545; display: block; font-size: 0.85rem; }`]
})
export class ValidationMessageComponent {
  @Input() control?: AbstractControl | null;
  @Input() label?: string;
}
```

```html
<!-- ใช้งาน Component -->
<input [(ngModel)]="email" name="email" email required #emailCtrl="ngModel">
<app-validation-message [control]="emailCtrl.control" label="อีเมล">
</app-validation-message>
```

---

## Form State

Angular ติดตาม State ของ Form และ FormControl แต่ละตัว

### State Properties

| Property | คำอธิบาย | CSS Class ที่ Angular เพิ่ม |
|----------|---------|--------------------------|
| `valid` | ข้อมูลถูกต้องตาม Validators ทั้งหมด | `ng-valid` |
| `invalid` | ข้อมูลไม่ถูกต้อง | `ng-invalid` |
| `pristine` | ยังไม่มีการแก้ไข | `ng-pristine` |
| `dirty` | มีการแก้ไขแล้ว | `ng-dirty` |
| `untouched` | ยังไม่เคย Focus/Blur | `ng-untouched` |
| `touched` | เคย Focus/Blur แล้ว | `ng-touched` |
| `pending` | กำลัง Validate แบบ Async | `ng-pending` |

### ตัวอย่างการใช้ State

```typescript
@Component({
  selector: 'app-state-demo',
  template: `
    <form #myForm="ngForm">
      <input
        [(ngModel)]="value"
        name="value"
        required
        minlength="3"
        #valueCtrl="ngModel"
      >

      <!-- แสดงสถานะ Form -->
      <div class="form-states">
        <p>Form Valid: <span [class]="myForm.valid ? 'green' : 'red'">{{ myForm.valid }}</span></p>
        <p>Form Dirty: {{ myForm.dirty }}</p>
        <p>Form Touched: {{ myForm.touched }}</p>
        <p>Form Pristine: {{ myForm.pristine }}</p>
      </div>

      <!-- แสดงสถานะ Control -->
      <div class="control-states">
        <p>Control Valid: {{ valueCtrl.valid }}</p>
        <p>Control Dirty: {{ valueCtrl.dirty }}</p>
        <p>Control Touched: {{ valueCtrl.touched }}</p>
        <p>Control Pristine: {{ valueCtrl.pristine }}</p>
        <p>Control Errors: {{ valueCtrl.errors | json }}</p>
      </div>
    </form>
  `
})
export class StateDemoComponent {
  value = '';
}
```

### Styling ตาม State

```css
/* styles.css หรือ component.css */

/* Untouched — ยังไม่ได้แตะ */
input.ng-untouched { border: 1px solid #ccc; }

/* Touched + Valid */
input.ng-touched.ng-valid { border: 1px solid #28a745; }

/* Touched + Invalid */
input.ng-touched.ng-invalid { border: 1px solid #dc3545; }

/* Dirty — มีการแก้ไข */
input.ng-dirty.ng-valid { background: #f0fff0; }
input.ng-dirty.ng-invalid { background: #fff0f0; }
```

### การใช้ State ในการแสดง/ซ่อน Error

```html
<!-- แสดง Error เมื่อ Touched (User click แล้ว click ออก) -->
<div *ngIf="ctrl.invalid && ctrl.touched">Error: touched</div>

<!-- แสดง Error เมื่อ Dirty (User เริ่มพิมพ์แล้ว) -->
<div *ngIf="ctrl.invalid && ctrl.dirty">Error: dirty</div>

<!-- แสดง Error เมื่อ Touched OR Dirty -->
<div *ngIf="ctrl.invalid && (ctrl.touched || ctrl.dirty)">Error: touched or dirty</div>

<!-- ซ่อน Submit button เมื่อ Form ยังไม่ Valid -->
<button [disabled]="myForm.invalid || myForm.pristine">ส่ง</button>

<!-- แสดงสัญลักษณ์ Valid -->
<span *ngIf="ctrl.valid && ctrl.dirty" class="valid-icon">✓</span>
```

---

## FormControl Reference

Template Reference Variable ให้เข้าถึง FormControl instance ของ Input ได้

### การสร้าง FormControl Reference

```html
<!-- ใช้ #variableName="ngModel" -->
<input
  [(ngModel)]="username"
  name="username"
  required
  minlength="3"
  #usernameCtrl="ngModel"
>

<!-- ตอนนี้ usernameCtrl คือ NgModel instance -->
<p>Value: {{ usernameCtrl.value }}</p>
<p>Valid: {{ usernameCtrl.valid }}</p>
<p>Errors: {{ usernameCtrl.errors | json }}</p>
```

### NgModel API

```typescript
// Properties ที่ใช้บ่อย
usernameCtrl.value        // ค่าปัจจุบัน
usernameCtrl.valid        // boolean
usernameCtrl.invalid      // boolean
usernameCtrl.pristine     // boolean (ยังไม่แก้ไข)
usernameCtrl.dirty        // boolean (แก้ไขแล้ว)
usernameCtrl.touched      // boolean (เคย blur)
usernameCtrl.untouched    // boolean
usernameCtrl.errors       // ValidationErrors | null
usernameCtrl.control      // FormControl instance
usernameCtrl.path         // string[]

// Methods
usernameCtrl.reset()           // รีเซ็ตค่า
usernameCtrl.setValue('new')   // ตั้งค่าใหม่
usernameCtrl.markAsTouched()   // ทำให้เป็น touched
usernameCtrl.markAsDirty()     // ทำให้เป็น dirty
```

### ใช้ FormControl ใน Component TypeScript

```typescript
import { Component, ViewChild } from '@angular/core';
import { NgForm, NgModel } from '@angular/forms';

@Component({
  selector: 'app-form',
  template: `
    <form #myForm="ngForm">
      <input [(ngModel)]="email" name="email" email required #emailCtrl="ngModel">
      <button (click)="checkEmail()">ตรวจสอบ</button>
    </form>
  `
})
export class FormComponent {
  email = '';

  @ViewChild('myForm') form!: NgForm;
  @ViewChild('emailCtrl') emailCtrl!: NgModel;

  checkEmail(): void {
    console.log('Email Valid:', this.emailCtrl.valid);
    console.log('Email Value:', this.emailCtrl.value);
    console.log('Form Value:', this.form.value);

    // บังคับให้แสดง Errors
    this.emailCtrl.control.markAsTouched();
  }

  resetForm(): void {
    this.form.reset();
    // หรือรีเซ็ตพร้อมค่าเริ่มต้น
    this.form.resetForm({ email: '' });
  }
}
```

---

## Submit Handling

### การ Submit Form

```html
<!-- วิธีที่ 1: ใช้ ngSubmit Event -->
<form #myForm="ngForm" (ngSubmit)="onSubmit(myForm)">
  <input [(ngModel)]="name" name="name" required>
  <button type="submit" [disabled]="myForm.invalid">ส่งข้อมูล</button>
</form>

<!-- วิธีที่ 2: ใช้ Click Event บน Button -->
<form #myForm="ngForm">
  <input [(ngModel)]="name" name="name" required>
  <button type="button" (click)="onSubmit(myForm)" [disabled]="myForm.invalid">
    ส่งข้อมูล
  </button>
</form>
```

### Component Submit Handler

```typescript
import { Component } from '@angular/core';
import { NgForm } from '@angular/forms';
import { UserService } from '../../services/user.service';

@Component({
  selector: 'app-submit-demo',
  templateUrl: './submit-demo.component.html'
})
export class SubmitDemoComponent {
  name = '';
  email = '';
  loading = false;
  successMessage = '';
  errorMessage = '';

  constructor(private userService: UserService) { }

  onSubmit(form: NgForm): void {
    // ตรวจสอบ Validity ก่อน
    if (form.invalid) {
      // Mark ทุก Field ว่า Touched เพื่อแสดง Error
      Object.keys(form.controls).forEach(key => {
        form.controls[key].markAsTouched();
      });
      return;
    }

    this.loading = true;
    this.successMessage = '';
    this.errorMessage = '';

    const data = form.value;

    this.userService.create(data).subscribe({
      next: (user) => {
        this.successMessage = `บันทึกข้อมูลของ ${user.name} สำเร็จ!`;
        this.loading = false;
        form.reset();  // รีเซ็ต Form หลัง Submit สำเร็จ
      },
      error: (err) => {
        this.errorMessage = 'เกิดข้อผิดพลาด: ' + err.message;
        this.loading = false;
      }
    });
  }
}
```

### Mark All as Touched (แสดง Error ทั้งหมด)

```typescript
// Utility Function สำหรับ Mark ทุก Control ว่า Touched
markAllAsTouched(form: NgForm): void {
  Object.values(form.controls).forEach(control => {
    control.markAsTouched();
  });
}
```

---

## Workshop: User Registration Form

Workshop นี้สร้าง Registration Form ที่สมบูรณ์พร้อม Validation ทุกประเภท

### Step 1: กำหนด Model

```typescript
// src/app/models/registration.model.ts

export interface RegistrationForm {
  firstName: string;
  lastName: string;
  username: string;
  email: string;
  password: string;
  confirmPassword: string;
  phone: string;
  birthDate: string;
  age: number | null;
  gender: string;
  address: {
    street: string;
    city: string;
    province: string;
    postalCode: string;
  };
  agreedToTerms: boolean;
  newsletter: boolean;
}

export interface RegisterResponse {
  success: boolean;
  message: string;
  userId?: number;
}
```

### Step 2: Registration Service

```typescript
// src/app/services/registration.service.ts
import { Injectable } from '@angular/core';
import { Observable, of, throwError } from 'rxjs';
import { delay } from 'rxjs/operators';
import { RegistrationForm, RegisterResponse } from '../models/registration.model';

@Injectable({ providedIn: 'root' })
export class RegistrationService {

  private registeredUsers: string[] = ['admin', 'user1', 'test'];
  private registeredEmails: string[] = ['admin@example.com', 'test@test.com'];

  // ตรวจสอบ Username ซ้ำ
  checkUsername(username: string): Observable<boolean> {
    const isTaken = this.registeredUsers.includes(username.toLowerCase());
    return of(isTaken).pipe(delay(500));
  }

  // ตรวจสอบ Email ซ้ำ
  checkEmail(email: string): Observable<boolean> {
    const isTaken = this.registeredEmails.includes(email.toLowerCase());
    return of(isTaken).pipe(delay(500));
  }

  // ลงทะเบียน
  register(data: RegistrationForm): Observable<RegisterResponse> {
    // จำลอง API Call
    if (this.registeredUsers.includes(data.username.toLowerCase())) {
      return throwError(() => new Error('ชื่อผู้ใช้นี้ถูกใช้งานแล้ว'));
    }

    this.registeredUsers.push(data.username.toLowerCase());
    this.registeredEmails.push(data.email.toLowerCase());

    return of({
      success: true,
      message: 'ลงทะเบียนสำเร็จ!',
      userId: Math.floor(Math.random() * 1000) + 100
    }).pipe(delay(1000));
  }
}
```

### Step 3: Registration Component

```typescript
// src/app/pages/register/register.component.ts
import { Component } from '@angular/core';
import { NgForm } from '@angular/forms';
import { Router } from '@angular/router';
import { RegistrationService } from '../../services/registration.service';
import { RegistrationForm } from '../../models/registration.model';

@Component({
  selector: 'app-register',
  templateUrl: './register.component.html',
  styleUrls: ['./register.component.css']
})
export class RegisterComponent {

  formData: RegistrationForm = {
    firstName: '',
    lastName: '',
    username: '',
    email: '',
    password: '',
    confirmPassword: '',
    phone: '',
    birthDate: '',
    age: null,
    gender: '',
    address: {
      street: '',
      city: '',
      province: '',
      postalCode: ''
    },
    agreedToTerms: false,
    newsletter: false
  };

  provinces: string[] = [
    'กรุงเทพมหานคร', 'เชียงใหม่', 'เชียงราย', 'ขอนแก่น',
    'นครราชสีมา', 'อุดรธานี', 'ภูเก็ต', 'สงขลา', 'ชลบุรี',
    'นนทบุรี', 'ปทุมธานี', 'สมุทรปราการ'
  ];

  loading = false;
  submitSuccess = false;
  submitError = '';
  registeredUserId?: number;

  showPassword = false;
  showConfirmPassword = false;

  constructor(
    private registrationService: RegistrationService,
    private router: Router
  ) { }

  // Toggle แสดง/ซ่อน Password
  togglePassword(): void {
    this.showPassword = !this.showPassword;
  }

  toggleConfirmPassword(): void {
    this.showConfirmPassword = !this.showConfirmPassword;
  }

  // ตรวจสอบว่า Password ตรงกัน
  isPasswordMatch(): boolean {
    return this.formData.password === this.formData.confirmPassword;
  }

  // ตรวจสอบ Strength ของ Password
  getPasswordStrength(): { level: string; color: string; score: number } {
    const pwd = this.formData.password;
    let score = 0;

    if (pwd.length >= 8) score++;
    if (pwd.length >= 12) score++;
    if (/[A-Z]/.test(pwd)) score++;
    if (/[a-z]/.test(pwd)) score++;
    if (/[0-9]/.test(pwd)) score++;
    if (/[^A-Za-z0-9]/.test(pwd)) score++;

    if (score <= 2) return { level: 'อ่อน', color: '#dc3545', score };
    if (score <= 4) return { level: 'ปานกลาง', color: '#ffc107', score };
    return { level: 'แข็งแกร่ง', color: '#28a745', score };
  }

  // Submit Form
  onSubmit(form: NgForm): void {
    // บังคับแสดง Errors ทั้งหมด
    if (form.invalid) {
      Object.keys(form.controls).forEach(key => {
        form.controls[key].markAsTouched();
      });
      return;
    }

    // ตรวจสอบ Password ตรงกัน
    if (!this.isPasswordMatch()) {
      return;
    }

    // ตรวจสอบยอมรับเงื่อนไข
    if (!this.formData.agreedToTerms) {
      alert('กรุณายอมรับข้อกำหนดและเงื่อนไขการใช้งาน');
      return;
    }

    this.loading = true;
    this.submitError = '';

    this.registrationService.register(this.formData).subscribe({
      next: (response) => {
        this.loading = false;
        this.submitSuccess = true;
        this.registeredUserId = response.userId;

        // นำทางไปหน้า Login หลัง 3 วินาที
        setTimeout(() => {
          this.router.navigate(['/login']);
        }, 3000);
      },
      error: (err) => {
        this.loading = false;
        this.submitError = err.message;
      }
    });
  }

  // รีเซ็ต Form
  resetForm(form: NgForm): void {
    form.reset();
    this.formData = {
      firstName: '', lastName: '', username: '', email: '',
      password: '', confirmPassword: '', phone: '',
      birthDate: '', age: null, gender: '',
      address: { street: '', city: '', province: '', postalCode: '' },
      agreedToTerms: false, newsletter: false
    };
    this.submitSuccess = false;
    this.submitError = '';
  }
}
```

### Step 4: Registration Template

```html
<!-- src/app/pages/register/register.component.html -->

<!-- แสดงเมื่อลงทะเบียนสำเร็จ -->
<div *ngIf="submitSuccess" class="success-page">
  <div class="success-icon">✅</div>
  <h2>ลงทะเบียนสำเร็จ!</h2>
  <p>รหัสผู้ใช้ของคุณ: <strong>#{{ registeredUserId }}</strong></p>
  <p>กำลังนำทางไปยังหน้าเข้าสู่ระบบ...</p>
  <a routerLink="/login" class="btn btn-primary">เข้าสู่ระบบเลย</a>
</div>

<!-- Form ลงทะเบียน -->
<div *ngIf="!submitSuccess" class="register-container">
  <div class="register-header">
    <h1>สมัครสมาชิก</h1>
    <p>กรอกข้อมูลเพื่อสร้างบัญชีใหม่</p>
  </div>

  <!-- Error Message -->
  <div *ngIf="submitError" class="alert alert-danger">
    <strong>เกิดข้อผิดพลาด:</strong> {{ submitError }}
  </div>

  <form
    #registerForm="ngForm"
    (ngSubmit)="onSubmit(registerForm)"
    novalidate
    class="register-form"
  >

    <!-- ===== ข้อมูลส่วนตัว ===== -->
    <fieldset class="form-section">
      <legend>ข้อมูลส่วนตัว</legend>

      <div class="form-row">
        <!-- ชื่อ -->
        <div class="form-group col-6">
          <label for="firstName">ชื่อ <span class="required">*</span></label>
          <input
            type="text"
            id="firstName"
            [(ngModel)]="formData.firstName"
            name="firstName"
            required
            minlength="2"
            maxlength="50"
            #firstNameCtrl="ngModel"
            [class.is-invalid]="firstNameCtrl.invalid && firstNameCtrl.touched"
            [class.is-valid]="firstNameCtrl.valid && firstNameCtrl.touched"
            placeholder="ชื่อจริง"
          >
          <div class="invalid-feedback" *ngIf="firstNameCtrl.invalid && firstNameCtrl.touched">
            <span *ngIf="firstNameCtrl.errors?.['required']">กรุณากรอกชื่อ</span>
            <span *ngIf="firstNameCtrl.errors?.['minlength']">ชื่อต้องมีอย่างน้อย 2 ตัวอักษร</span>
          </div>
        </div>

        <!-- นามสกุล -->
        <div class="form-group col-6">
          <label for="lastName">นามสกุล <span class="required">*</span></label>
          <input
            type="text"
            id="lastName"
            [(ngModel)]="formData.lastName"
            name="lastName"
            required
            minlength="2"
            maxlength="50"
            #lastNameCtrl="ngModel"
            [class.is-invalid]="lastNameCtrl.invalid && lastNameCtrl.touched"
            [class.is-valid]="lastNameCtrl.valid && lastNameCtrl.touched"
            placeholder="นามสกุล"
          >
          <div class="invalid-feedback" *ngIf="lastNameCtrl.invalid && lastNameCtrl.touched">
            <span *ngIf="lastNameCtrl.errors?.['required']">กรุณากรอกนามสกุล</span>
            <span *ngIf="lastNameCtrl.errors?.['minlength']">นามสกุลต้องมีอย่างน้อย 2 ตัวอักษร</span>
          </div>
        </div>
      </div>

      <!-- วันเกิด -->
      <div class="form-group">
        <label for="birthDate">วันเกิด</label>
        <input
          type="date"
          id="birthDate"
          [(ngModel)]="formData.birthDate"
          name="birthDate"
          [max]="getMaxDate()"
          #birthDateCtrl="ngModel"
        >
      </div>

      <!-- อายุ -->
      <div class="form-group">
        <label for="age">อายุ <span class="required">*</span></label>
        <input
          type="number"
          id="age"
          [(ngModel)]="formData.age"
          name="age"
          required
          min="18"
          max="100"
          #ageCtrl="ngModel"
          [class.is-invalid]="ageCtrl.invalid && ageCtrl.touched"
          [class.is-valid]="ageCtrl.valid && ageCtrl.touched"
          placeholder="อายุ (ปี)"
        >
        <div class="invalid-feedback" *ngIf="ageCtrl.invalid && ageCtrl.touched">
          <span *ngIf="ageCtrl.errors?.['required']">กรุณากรอกอายุ</span>
          <span *ngIf="ageCtrl.errors?.['min']">ต้องมีอายุอย่างน้อย 18 ปี</span>
          <span *ngIf="ageCtrl.errors?.['max']">กรุณากรอกอายุที่ถูกต้อง</span>
        </div>
      </div>

      <!-- เพศ -->
      <div class="form-group">
        <label>เพศ <span class="required">*</span></label>
        <div class="radio-group">
          <label class="radio-label">
            <input type="radio" [(ngModel)]="formData.gender" name="gender" value="male" required>
            ชาย
          </label>
          <label class="radio-label">
            <input type="radio" [(ngModel)]="formData.gender" name="gender" value="female">
            หญิง
          </label>
          <label class="radio-label">
            <input type="radio" [(ngModel)]="formData.gender" name="gender" value="other">
            ไม่ระบุ
          </label>
        </div>
      </div>

      <!-- เบอร์โทรศัพท์ -->
      <div class="form-group">
        <label for="phone">เบอร์โทรศัพท์</label>
        <input
          type="tel"
          id="phone"
          [(ngModel)]="formData.phone"
          name="phone"
          pattern="^(0[689][0-9]{8})$"
          #phoneCtrl="ngModel"
          [class.is-invalid]="phoneCtrl.invalid && phoneCtrl.touched"
          [class.is-valid]="phoneCtrl.valid && phoneCtrl.touched && formData.phone !== ''"
          placeholder="0812345678"
        >
        <div class="invalid-feedback" *ngIf="phoneCtrl.invalid && phoneCtrl.touched">
          <span *ngIf="phoneCtrl.errors?.['pattern']">
            รูปแบบเบอร์โทรไม่ถูกต้อง (เช่น 0812345678)
          </span>
        </div>
      </div>

    </fieldset>

    <!-- ===== บัญชีผู้ใช้ ===== -->
    <fieldset class="form-section">
      <legend>ข้อมูลบัญชี</legend>

      <!-- ชื่อผู้ใช้ -->
      <div class="form-group">
        <label for="username">ชื่อผู้ใช้ <span class="required">*</span></label>
        <input
          type="text"
          id="username"
          [(ngModel)]="formData.username"
          name="username"
          required
          minlength="4"
          maxlength="20"
          pattern="[a-zA-Z0-9_]+"
          #usernameCtrl="ngModel"
          [class.is-invalid]="usernameCtrl.invalid && usernameCtrl.touched"
          [class.is-valid]="usernameCtrl.valid && usernameCtrl.touched"
          placeholder="username (ตัวอักษร, ตัวเลข, _)"
        >
        <div class="invalid-feedback" *ngIf="usernameCtrl.invalid && usernameCtrl.touched">
          <span *ngIf="usernameCtrl.errors?.['required']">กรุณากรอกชื่อผู้ใช้</span>
          <span *ngIf="usernameCtrl.errors?.['minlength']">ต้องมีอย่างน้อย 4 ตัวอักษร</span>
          <span *ngIf="usernameCtrl.errors?.['maxlength']">ต้องไม่เกิน 20 ตัวอักษร</span>
          <span *ngIf="usernameCtrl.errors?.['pattern']">ใช้ได้เฉพาะตัวอักษร ตัวเลข และ _</span>
        </div>
        <small class="form-text">ชื่อผู้ใช้จะปรากฎบน Profile ของคุณ</small>
      </div>

      <!-- อีเมล -->
      <div class="form-group">
        <label for="email">อีเมล <span class="required">*</span></label>
        <input
          type="email"
          id="email"
          [(ngModel)]="formData.email"
          name="email"
          required
          email
          #emailCtrl="ngModel"
          [class.is-invalid]="emailCtrl.invalid && emailCtrl.touched"
          [class.is-valid]="emailCtrl.valid && emailCtrl.touched"
          placeholder="example@email.com"
        >
        <div class="invalid-feedback" *ngIf="emailCtrl.invalid && emailCtrl.touched">
          <span *ngIf="emailCtrl.errors?.['required']">กรุณากรอกอีเมล</span>
          <span *ngIf="emailCtrl.errors?.['email']">รูปแบบอีเมลไม่ถูกต้อง</span>
        </div>
      </div>

      <!-- รหัสผ่าน -->
      <div class="form-group">
        <label for="password">รหัสผ่าน <span class="required">*</span></label>
        <div class="input-group">
          <input
            [type]="showPassword ? 'text' : 'password'"
            id="password"
            [(ngModel)]="formData.password"
            name="password"
            required
            minlength="8"
            maxlength="50"
            pattern="(?=.*[a-z])(?=.*[A-Z])(?=.*[0-9]).{8,}"
            #passwordCtrl="ngModel"
            [class.is-invalid]="passwordCtrl.invalid && passwordCtrl.touched"
            [class.is-valid]="passwordCtrl.valid && passwordCtrl.touched"
            placeholder="อย่างน้อย 8 ตัว มีตัวพิมพ์ใหญ่ เล็ก และตัวเลข"
          >
          <button type="button" class="btn-toggle-pwd" (click)="togglePassword()">
            {{ showPassword ? '🙈' : '👁️' }}
          </button>
        </div>

        <!-- Password Strength Indicator -->
        <div *ngIf="formData.password" class="password-strength">
          <div class="strength-bar">
            <div
              class="strength-fill"
              [style.width.%]="(getPasswordStrength().score / 6) * 100"
              [style.background]="getPasswordStrength().color"
            ></div>
          </div>
          <small [style.color]="getPasswordStrength().color">
            ความแข็งแกร่ง: {{ getPasswordStrength().level }}
          </small>
        </div>

        <div class="invalid-feedback" *ngIf="passwordCtrl.invalid && passwordCtrl.touched">
          <span *ngIf="passwordCtrl.errors?.['required']">กรุณากรอกรหัสผ่าน</span>
          <span *ngIf="passwordCtrl.errors?.['minlength']">รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร</span>
          <span *ngIf="passwordCtrl.errors?.['pattern']">
            ต้องมีตัวพิมพ์ใหญ่ ตัวพิมพ์เล็ก และตัวเลข
          </span>
        </div>
      </div>

      <!-- ยืนยันรหัสผ่าน -->
      <div class="form-group">
        <label for="confirmPassword">ยืนยันรหัสผ่าน <span class="required">*</span></label>
        <div class="input-group">
          <input
            [type]="showConfirmPassword ? 'text' : 'password'"
            id="confirmPassword"
            [(ngModel)]="formData.confirmPassword"
            name="confirmPassword"
            required
            #confirmPasswordCtrl="ngModel"
            [class.is-invalid]="(confirmPasswordCtrl.touched || confirmPasswordCtrl.dirty) && !isPasswordMatch()"
            [class.is-valid]="confirmPasswordCtrl.touched && isPasswordMatch() && formData.confirmPassword !== ''"
            placeholder="กรอกรหัสผ่านอีกครั้ง"
          >
          <button type="button" class="btn-toggle-pwd" (click)="toggleConfirmPassword()">
            {{ showConfirmPassword ? '🙈' : '👁️' }}
          </button>
        </div>
        <div
          class="invalid-feedback"
          *ngIf="(confirmPasswordCtrl.touched || confirmPasswordCtrl.dirty) && !isPasswordMatch()"
        >
          รหัสผ่านไม่ตรงกัน
        </div>
      </div>

    </fieldset>

    <!-- ===== ที่อยู่ ===== -->
    <fieldset class="form-section" ngModelGroup="address" #addressGroup="ngModelGroup">
      <legend>ที่อยู่</legend>

      <!-- บ้านเลขที่/ถนน -->
      <div class="form-group">
        <label for="street">บ้านเลขที่ / ถนน <span class="required">*</span></label>
        <input
          type="text"
          id="street"
          [(ngModel)]="formData.address.street"
          name="street"
          required
          #streetCtrl="ngModel"
          [class.is-invalid]="streetCtrl.invalid && streetCtrl.touched"
          placeholder="เลขที่บ้าน ถนน ซอย"
        >
        <div class="invalid-feedback" *ngIf="streetCtrl.invalid && streetCtrl.touched">
          กรุณากรอกที่อยู่
        </div>
      </div>

      <div class="form-row">
        <!-- เมือง/อำเภอ -->
        <div class="form-group col-6">
          <label for="city">เมือง / อำเภอ <span class="required">*</span></label>
          <input
            type="text"
            id="city"
            [(ngModel)]="formData.address.city"
            name="city"
            required
            #cityCtrl="ngModel"
            [class.is-invalid]="cityCtrl.invalid && cityCtrl.touched"
            placeholder="อำเภอ/เขต"
          >
          <div class="invalid-feedback" *ngIf="cityCtrl.invalid && cityCtrl.touched">
            กรุณากรอกเมือง/อำเภอ
          </div>
        </div>

        <!-- จังหวัด -->
        <div class="form-group col-6">
          <label for="province">จังหวัด <span class="required">*</span></label>
          <select
            id="province"
            [(ngModel)]="formData.address.province"
            name="province"
            required
            #provinceCtrl="ngModel"
            [class.is-invalid]="provinceCtrl.invalid && provinceCtrl.touched"
          >
            <option value="">-- เลือกจังหวัด --</option>
            <option *ngFor="let p of provinces" [value]="p">{{ p }}</option>
          </select>
          <div class="invalid-feedback" *ngIf="provinceCtrl.invalid && provinceCtrl.touched">
            กรุณาเลือกจังหวัด
          </div>
        </div>
      </div>

      <!-- รหัสไปรษณีย์ -->
      <div class="form-group">
        <label for="postalCode">รหัสไปรษณีย์ <span class="required">*</span></label>
        <input
          type="text"
          id="postalCode"
          [(ngModel)]="formData.address.postalCode"
          name="postalCode"
          required
          pattern="[0-9]{5}"
          #postalCodeCtrl="ngModel"
          [class.is-invalid]="postalCodeCtrl.invalid && postalCodeCtrl.touched"
          [class.is-valid]="postalCodeCtrl.valid && postalCodeCtrl.touched"
          placeholder="10000"
          maxlength="5"
        >
        <div class="invalid-feedback" *ngIf="postalCodeCtrl.invalid && postalCodeCtrl.touched">
          <span *ngIf="postalCodeCtrl.errors?.['required']">กรุณากรอกรหัสไปรษณีย์</span>
          <span *ngIf="postalCodeCtrl.errors?.['pattern']">รหัสไปรษณีย์ต้องเป็นตัวเลข 5 หลัก</span>
        </div>
      </div>

    </fieldset>

    <!-- ===== ข้อตกลง ===== -->
    <fieldset class="form-section">
      <legend>ข้อตกลง</legend>

      <!-- Newsletter -->
      <div class="form-group">
        <label class="checkbox-label">
          <input
            type="checkbox"
            [(ngModel)]="formData.newsletter"
            name="newsletter"
          >
          <span>รับข่าวสารและโปรโมชันทางอีเมล</span>
        </label>
      </div>

      <!-- Terms -->
      <div class="form-group">
        <label class="checkbox-label required-checkbox">
          <input
            type="checkbox"
            [(ngModel)]="formData.agreedToTerms"
            name="agreedToTerms"
            required
            #termsCtrl="ngModel"
          >
          <span>
            ฉันยอมรับ
            <a href="/terms" target="_blank">ข้อกำหนดการใช้งาน</a>
            และ
            <a href="/privacy" target="_blank">นโยบายความเป็นส่วนตัว</a>
            <span class="required">*</span>
          </span>
        </label>
        <div class="invalid-feedback" *ngIf="termsCtrl.invalid && termsCtrl.touched">
          กรุณายอมรับข้อตกลงก่อนสมัครสมาชิก
        </div>
      </div>

    </fieldset>

    <!-- ===== ปุ่ม Submit ===== -->
    <div class="form-actions">
      <button
        type="button"
        class="btn btn-secondary"
        (click)="resetForm(registerForm)"
        [disabled]="loading"
      >
        ล้างข้อมูล
      </button>

      <button
        type="submit"
        class="btn btn-primary"
        [disabled]="loading || !formData.agreedToTerms"
      >
        <span *ngIf="loading">กำลังสมัครสมาชิก...</span>
        <span *ngIf="!loading">สมัครสมาชิก</span>
      </button>
    </div>

    <!-- Debug Section (ลบออกใน Production) -->
    <details class="debug-section">
      <summary>Debug Info</summary>
      <p>Form Valid: {{ registerForm.valid }}</p>
      <p>Form Dirty: {{ registerForm.dirty }}</p>
      <p>Form Touched: {{ registerForm.touched }}</p>
      <pre>Form Value: {{ registerForm.value | json }}</pre>
    </details>

  </form>
</div>
```

### Step 5: CSS Styles

```css
/* src/app/pages/register/register.component.css */

.register-container {
  max-width: 700px;
  margin: 2rem auto;
  padding: 0 1rem;
}

.register-header {
  text-align: center;
  margin-bottom: 2rem;
}

.register-header h1 {
  font-size: 2rem;
  color: #333;
}

.form-section {
  border: 1px solid #dee2e6;
  border-radius: 8px;
  padding: 1.5rem;
  margin-bottom: 1.5rem;
}

.form-section legend {
  font-weight: 600;
  color: #495057;
  padding: 0 0.5rem;
  font-size: 1.1rem;
}

.form-group {
  margin-bottom: 1rem;
}

.form-group label {
  display: block;
  font-weight: 500;
  margin-bottom: 0.4rem;
  color: #333;
}

.form-row {
  display: flex;
  gap: 1rem;
}

.col-6 {
  flex: 1;
}

input[type="text"],
input[type="email"],
input[type="password"],
input[type="number"],
input[type="date"],
input[type="tel"],
select,
textarea {
  width: 100%;
  padding: 0.5rem 0.75rem;
  font-size: 1rem;
  border: 1px solid #ced4da;
  border-radius: 4px;
  transition: border-color 0.2s;
  box-sizing: border-box;
}

input:focus,
select:focus,
textarea:focus {
  outline: none;
  border-color: #80bdff;
  box-shadow: 0 0 0 3px rgba(0, 123, 255, 0.25);
}

input.is-invalid,
select.is-invalid {
  border-color: #dc3545;
}

input.is-invalid:focus,
select.is-invalid:focus {
  box-shadow: 0 0 0 3px rgba(220, 53, 69, 0.25);
}

input.is-valid,
select.is-valid {
  border-color: #28a745;
}

.invalid-feedback {
  color: #dc3545;
  font-size: 0.85rem;
  margin-top: 0.25rem;
  display: block;
}

.form-text {
  color: #6c757d;
  font-size: 0.85rem;
  margin-top: 0.25rem;
}

.required {
  color: #dc3545;
}

/* Password */
.input-group {
  display: flex;
  gap: 0.5rem;
}

.input-group input {
  flex: 1;
}

.btn-toggle-pwd {
  background: none;
  border: 1px solid #ced4da;
  border-radius: 4px;
  padding: 0.5rem;
  cursor: pointer;
  font-size: 1rem;
}

/* Password Strength */
.password-strength {
  margin-top: 0.5rem;
}

.strength-bar {
  height: 6px;
  background: #e9ecef;
  border-radius: 3px;
  overflow: hidden;
  margin-bottom: 0.25rem;
}

.strength-fill {
  height: 100%;
  transition: width 0.3s, background 0.3s;
}

/* Radio & Checkbox */
.radio-group {
  display: flex;
  gap: 1.5rem;
  padding: 0.5rem 0;
}

.radio-label {
  display: flex;
  align-items: center;
  gap: 0.4rem;
  cursor: pointer;
  font-weight: normal;
}

.checkbox-label {
  display: flex;
  align-items: flex-start;
  gap: 0.5rem;
  cursor: pointer;
  font-weight: normal;
}

.checkbox-label input[type="checkbox"] {
  width: auto;
  margin-top: 0.2rem;
}

/* Buttons */
.form-actions {
  display: flex;
  gap: 1rem;
  justify-content: flex-end;
  margin-top: 1.5rem;
}

.btn {
  padding: 0.6rem 1.5rem;
  font-size: 1rem;
  border-radius: 4px;
  border: none;
  cursor: pointer;
  transition: background 0.2s, opacity 0.2s;
}

.btn:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.btn-primary {
  background: #007bff;
  color: white;
}

.btn-primary:hover:not(:disabled) {
  background: #0056b3;
}

.btn-secondary {
  background: #6c757d;
  color: white;
}

.btn-secondary:hover:not(:disabled) {
  background: #545b62;
}

/* Alert */
.alert {
  padding: 0.75rem 1rem;
  border-radius: 4px;
  margin-bottom: 1rem;
}

.alert-danger {
  background: #f8d7da;
  color: #721c24;
  border: 1px solid #f5c6cb;
}

/* Success Page */
.success-page {
  text-align: center;
  padding: 4rem 2rem;
}

.success-icon {
  font-size: 4rem;
  margin-bottom: 1rem;
}

/* Debug */
.debug-section {
  margin-top: 2rem;
  padding: 1rem;
  background: #f8f9fa;
  border-radius: 4px;
  font-size: 0.85rem;
}

.debug-section summary {
  cursor: pointer;
  color: #6c757d;
}

/* Responsive */
@media (max-width: 576px) {
  .form-row {
    flex-direction: column;
    gap: 0;
  }

  .form-actions {
    flex-direction: column-reverse;
  }

  .btn {
    width: 100%;
  }
}
```

### Step 6: เพิ่ม Method ที่ขาดใน Component

```typescript
// เพิ่มใน register.component.ts

// คำนวณวันที่สูงสุดสำหรับวันเกิด (ต้องอายุ 18+)
getMaxDate(): string {
  const today = new Date();
  const maxDate = new Date(
    today.getFullYear() - 18,
    today.getMonth(),
    today.getDate()
  );
  return maxDate.toISOString().split('T')[0];
}
```

### Step 7: ลงทะเบียน Module

```typescript
// src/app/app.module.ts
import { NgModule } from '@angular/core';
import { BrowserModule } from '@angular/platform-browser';
import { FormsModule } from '@angular/forms';
import { HttpClientModule } from '@angular/common/http';

import { AppRoutingModule } from './app-routing.module';
import { AppComponent } from './app.component';
import { RegisterComponent } from './pages/register/register.component';
import { ValidationMessageComponent } from './components/validation-message/validation-message.component';

@NgModule({
  declarations: [
    AppComponent,
    RegisterComponent,
    ValidationMessageComponent
  ],
  imports: [
    BrowserModule,
    FormsModule,
    HttpClientModule,
    AppRoutingModule
  ],
  bootstrap: [AppComponent]
})
export class AppModule { }
```

---

## สรุปทบทวน

### Cheat Sheet

```html
<!-- Two-way Binding -->
<input [(ngModel)]="value" name="fieldName">

<!-- Template Reference -->
<input #ctrl="ngModel">

<!-- Required -->
<input required>

<!-- MinLength / MaxLength -->
<input minlength="6" maxlength="50">

<!-- Email -->
<input type="email" email>

<!-- Pattern -->
<input pattern="[0-9]{5}">

<!-- Min / Max (number) -->
<input type="number" min="0" max="100">

<!-- Form Reference -->
<form #myForm="ngForm" (ngSubmit)="submit(myForm)">

<!-- Disabled Submit when Invalid -->
<button [disabled]="myForm.invalid">Submit</button>

<!-- Show Error -->
<div *ngIf="ctrl.invalid && ctrl.touched">
  <span *ngIf="ctrl.errors?.['required']">Required</span>
</div>

<!-- CSS State Classes -->
[class.is-invalid]="ctrl.invalid && ctrl.touched"
[class.is-valid]="ctrl.valid && ctrl.touched"

<!-- Group Fields -->
<div ngModelGroup="address" #grp="ngModelGroup">
```

### State Properties Summary

```typescript
// Form (NgForm)
form.valid       // ทุก Control ผ่าน Validator
form.invalid     // มี Control อย่างน้อย 1 ตัวที่ไม่ผ่าน
form.dirty       // มีการแก้ไขอย่างน้อย 1 Control
form.pristine    // ยังไม่มีการแก้ไขเลย
form.touched     // มีการ Focus แล้ว Blur อย่างน้อย 1 Control
form.untouched   // ยังไม่มี Control ที่ถูก Touch

// Control (NgModel)
ctrl.value       // ค่าปัจจุบัน
ctrl.errors      // { required: true, minlength: {...}, ... } | null
ctrl.valid
ctrl.invalid
ctrl.dirty
ctrl.pristine
ctrl.touched
ctrl.untouched
```

---

*เอกสารนี้เป็นส่วนหนึ่งของหลักสูตร Angular — Part 09*
