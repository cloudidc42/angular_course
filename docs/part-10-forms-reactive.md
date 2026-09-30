# Part 10 — Reactive Forms ใน Angular

## บทนำ

Reactive Forms เป็นวิธีการสร้างฟอร์มใน Angular ที่ให้การควบคุมฝั่ง Component class อย่างเต็มที่ ต่างจาก Template-driven Forms ที่พึ่งพา directive ใน template เป็นหลัก Reactive Forms ใช้ Observable streams และ immutable data model ทำให้ง่ายต่อการทดสอบ จัดการค่าแบบ dynamic และรองรับ validation ที่ซับซ้อน

### ข้อดีของ Reactive Forms
- ควบคุม state ของฟอร์มได้อย่างสมบูรณ์จาก TypeScript
- ทดสอบง่ายกว่า Template-driven Forms
- รองรับ validation ซับซ้อนและ async validation
- ใช้ RxJS Observables สำหรับติดตาม valueChanges และ statusChanges
- เหมาะกับฟอร์มที่มีโครงสร้างซับซ้อน เช่น FormArray

---

## 10.1 การ Setup ReactiveFormsModule

ก่อนใช้งาน Reactive Forms ต้อง import `ReactiveFormsModule` ใน module หรือ component

### วิธีที่ 1: Import ใน AppModule (Angular 14 และก่อนหน้า)

```typescript
// app.module.ts
import { NgModule } from '@angular/core';
import { BrowserModule } from '@angular/platform-browser';
import { ReactiveFormsModule } from '@angular/forms';

import { AppComponent } from './app.component';

@NgModule({
  declarations: [AppComponent],
  imports: [
    BrowserModule,
    ReactiveFormsModule  // เพิ่มตรงนี้
  ],
  bootstrap: [AppComponent]
})
export class AppModule {}
```

### วิธีที่ 2: Import ใน Standalone Component (Angular 15+)

```typescript
// my-form.component.ts
import { Component } from '@angular/core';
import { ReactiveFormsModule } from '@angular/forms';

@Component({
  selector: 'app-my-form',
  standalone: true,
  imports: [ReactiveFormsModule],  // เพิ่มตรงนี้
  templateUrl: './my-form.component.html'
})
export class MyFormComponent {}
```

---

## 10.2 FormControl — หน่วยพื้นฐานของฟอร์ม

`FormControl` คือ building block ที่เล็กที่สุดของ Reactive Forms ใช้ติดตามค่าและ validation state ของ input เดี่ยว

```typescript
// form-control-example.component.ts
import { Component, OnInit } from '@angular/core';
import { FormControl, Validators } from '@angular/forms';

@Component({
  selector: 'app-form-control-example',
  standalone: true,
  imports: [ReactiveFormsModule],
  template: `
    <div class="form-group">
      <label for="email">อีเมล</label>
      <input
        id="email"
        type="email"
        [formControl]="emailControl"
        class="form-control"
        [class.is-invalid]="emailControl.invalid && emailControl.touched"
      />

      <!-- แสดง error messages -->
      <div *ngIf="emailControl.invalid && emailControl.touched" class="invalid-feedback">
        <span *ngIf="emailControl.errors?.['required']">กรุณากรอกอีเมล</span>
        <span *ngIf="emailControl.errors?.['email']">รูปแบบอีเมลไม่ถูกต้อง</span>
      </div>
    </div>

    <div class="debug-info">
      <p>ค่าปัจจุบัน: {{ emailControl.value }}</p>
      <p>สถานะ: {{ emailControl.status }}</p>
      <p>touched: {{ emailControl.touched }}</p>
      <p>dirty: {{ emailControl.dirty }}</p>
    </div>

    <button (click)="resetControl()">Reset</button>
  `
})
export class FormControlExampleComponent implements OnInit {
  // สร้าง FormControl พร้อม initial value และ validators
  emailControl = new FormControl('', [
    Validators.required,
    Validators.email
  ]);

  ngOnInit(): void {
    // ติดตามการเปลี่ยนแปลงค่า
    this.emailControl.valueChanges.subscribe(value => {
      console.log('Email changed:', value);
    });

    // ติดตาม status changes
    this.emailControl.statusChanges.subscribe(status => {
      console.log('Status:', status); // VALID, INVALID, PENDING, DISABLED
    });
  }

  resetControl(): void {
    this.emailControl.reset('');
  }
}
```

### Properties สำคัญของ FormControl

| Property | ประเภท | คำอธิบาย |
|----------|--------|----------|
| `value` | any | ค่าปัจจุบันของ control |
| `status` | string | 'VALID', 'INVALID', 'PENDING', 'DISABLED' |
| `valid` | boolean | true เมื่อ validation ผ่าน |
| `invalid` | boolean | true เมื่อ validation ไม่ผ่าน |
| `pristine` | boolean | true เมื่อยังไม่มีการแก้ไข |
| `dirty` | boolean | true เมื่อมีการแก้ไขแล้ว |
| `touched` | boolean | true เมื่อ blur แล้ว |
| `untouched` | boolean | true เมื่อยังไม่ได้ blur |
| `errors` | object | null หรือ object ของ errors |

---

## 10.3 FormGroup — จัดกลุ่ม FormControl

`FormGroup` ใช้รวม FormControl หลายตัวเข้าด้วยกัน เป็นโครงสร้างหลักของฟอร์ม

```typescript
// form-group-example.component.ts
import { Component, OnInit } from '@angular/core';
import { FormGroup, FormControl, Validators } from '@angular/forms';
import { ReactiveFormsModule } from '@angular/forms';
import { CommonModule } from '@angular/common';

@Component({
  selector: 'app-form-group-example',
  standalone: true,
  imports: [ReactiveFormsModule, CommonModule],
  template: `
    <form [formGroup]="profileForm" (ngSubmit)="onSubmit()">
      <div class="form-group">
        <label>ชื่อจริง</label>
        <input type="text" formControlName="firstName" class="form-control">
        <div *ngIf="profileForm.get('firstName')?.invalid && profileForm.get('firstName')?.touched">
          <small class="text-danger">กรุณากรอกชื่อจริง</small>
        </div>
      </div>

      <div class="form-group">
        <label>นามสกุล</label>
        <input type="text" formControlName="lastName" class="form-control">
      </div>

      <!-- Nested FormGroup -->
      <div formGroupName="address" class="border p-3">
        <h5>ที่อยู่</h5>
        <div class="form-group">
          <label>บ้านเลขที่</label>
          <input type="text" formControlName="street" class="form-control">
        </div>
        <div class="form-group">
          <label>เมือง</label>
          <input type="text" formControlName="city" class="form-control">
        </div>
        <div class="form-group">
          <label>รหัสไปรษณีย์</label>
          <input type="text" formControlName="zipCode" class="form-control">
        </div>
      </div>

      <button type="submit" [disabled]="profileForm.invalid">บันทึก</button>
      <button type="button" (click)="resetForm()">ล้างฟอร์ม</button>
    </form>

    <!-- แสดงค่าทั้งหมด (debug) -->
    <pre>{{ profileForm.value | json }}</pre>
  `
})
export class FormGroupExampleComponent implements OnInit {
  profileForm = new FormGroup({
    firstName: new FormControl('', [Validators.required, Validators.minLength(2)]),
    lastName: new FormControl('', Validators.required),
    address: new FormGroup({
      street: new FormControl(''),
      city: new FormControl('', Validators.required),
      zipCode: new FormControl('', [Validators.pattern(/^\d{5}$/)])
    })
  });

  ngOnInit(): void {
    // ติดตามการเปลี่ยนแปลงทั้ง form
    this.profileForm.valueChanges.subscribe(values => {
      console.log('Form values changed:', values);
    });
  }

  onSubmit(): void {
    if (this.profileForm.valid) {
      console.log('Form submitted:', this.profileForm.value);
      // ส่งข้อมูลไป API
    } else {
      // mark ทุก field ว่า touched เพื่อแสดง errors
      this.profileForm.markAllAsTouched();
    }
  }

  resetForm(): void {
    this.profileForm.reset();
  }

  // เข้าถึง control ผ่าน getter
  get firstName() {
    return this.profileForm.get('firstName');
  }

  get city() {
    return this.profileForm.get('address.city');
  }
}
```

---

## 10.4 FormBuilder — สร้างฟอร์มได้ง่ายขึ้น

`FormBuilder` เป็น service ที่ช่วยลด boilerplate code ในการสร้าง FormGroup, FormControl, และ FormArray

```typescript
// form-builder-example.component.ts
import { Component } from '@angular/core';
import { FormBuilder, FormGroup, Validators } from '@angular/forms';
import { ReactiveFormsModule } from '@angular/forms';
import { CommonModule } from '@angular/common';

@Component({
  selector: 'app-form-builder-example',
  standalone: true,
  imports: [ReactiveFormsModule, CommonModule],
  template: `
    <form [formGroup]="loginForm" (ngSubmit)="onLogin()">
      <div class="mb-3">
        <label class="form-label">อีเมล</label>
        <input type="email" class="form-control" formControlName="email">
        <div *ngIf="email?.invalid && email?.touched" class="text-danger">
          <small *ngIf="email?.errors?.['required']">กรุณากรอกอีเมล</small>
          <small *ngIf="email?.errors?.['email']">รูปแบบอีเมลไม่ถูกต้อง</small>
        </div>
      </div>

      <div class="mb-3">
        <label class="form-label">รหัสผ่าน</label>
        <input type="password" class="form-control" formControlName="password">
        <div *ngIf="password?.invalid && password?.touched" class="text-danger">
          <small *ngIf="password?.errors?.['required']">กรุณากรอกรหัสผ่าน</small>
          <small *ngIf="password?.errors?.['minlength']">
            รหัสผ่านต้องมีอย่างน้อย {{ password?.errors?.['minlength']?.requiredLength }} ตัวอักษร
          </small>
        </div>
      </div>

      <div class="mb-3 form-check">
        <input type="checkbox" class="form-check-input" formControlName="rememberMe">
        <label class="form-check-label">จดจำการเข้าสู่ระบบ</label>
      </div>

      <button type="submit" class="btn btn-primary" [disabled]="loginForm.invalid">
        เข้าสู่ระบบ
      </button>
    </form>
  `
})
export class FormBuilderExampleComponent {
  // inject FormBuilder
  loginForm: FormGroup;

  constructor(private fb: FormBuilder) {
    // fb.group() แทน new FormGroup({})
    this.loginForm = this.fb.group({
      email: ['', [Validators.required, Validators.email]],
      // syntax: [initialValue, syncValidators, asyncValidators]
      password: ['', [Validators.required, Validators.minLength(8)]],
      rememberMe: [false]
    });
  }

  get email() { return this.loginForm.get('email'); }
  get password() { return this.loginForm.get('password'); }

  onLogin(): void {
    if (this.loginForm.valid) {
      const { email, password, rememberMe } = this.loginForm.value;
      console.log('Login:', { email, password, rememberMe });
    }
  }
}
```

---

## 10.5 Validators — การ Validate ข้อมูล

Angular มี built-in validators หลายตัว และยังสามารถสร้าง custom validator เองได้

### Built-in Validators

```typescript
import { Validators } from '@angular/forms';

// ตัวอย่างการใช้ Validators ต่างๆ
const form = this.fb.group({
  name: ['', [
    Validators.required,           // ต้องมีค่า
    Validators.minLength(2),       // ต้องยาวอย่างน้อย 2 ตัว
    Validators.maxLength(50)       // ต้องยาวไม่เกิน 50 ตัว
  ]],
  age: ['', [
    Validators.required,
    Validators.min(18),            // ต้องมากกว่าหรือเท่ากับ 18
    Validators.max(120)            // ต้องน้อยกว่าหรือเท่ากับ 120
  ]],
  email: ['', [
    Validators.required,
    Validators.email               // ต้องเป็นรูปแบบ email
  ]],
  phone: ['', [
    Validators.required,
    Validators.pattern(/^[0-9]{10}$/)  // ต้องตรงกับ pattern
  ]],
  website: ['', Validators.nullValidator]  // ผ่านเสมอ (ไม่ validate)
});
```

---

## 10.6 Custom Validators — สร้าง Validator เอง

### Sync Custom Validator

```typescript
// validators/custom-validators.ts
import { AbstractControl, ValidationErrors, ValidatorFn } from '@angular/forms';

// Validator ตรวจสอบว่าไม่มีช่องว่าง
export function noWhitespaceValidator(): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null => {
    const value = control.value || '';
    const hasWhitespace = value.trim().length === 0 && value.length > 0;
    return hasWhitespace ? { whitespace: true } : null;
  };
}

// Validator ตรวจสอบเบอร์โทรศัพท์ไทย
export function thaiPhoneValidator(): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null => {
    const value = control.value || '';
    // เบอร์ไทยขึ้นต้นด้วย 0 ตามด้วยตัวเลข 9 ตัว
    const thaiPhonePattern = /^0[0-9]{9}$/;
    const isValid = thaiPhonePattern.test(value);
    return !isValid && value.length > 0 ? { thaiPhone: true } : null;
  };
}

// Validator เปรียบเทียบสองฟิลด์ (เช่น password confirm)
export function passwordMatchValidator(passwordKey: string, confirmKey: string): ValidatorFn {
  return (group: AbstractControl): ValidationErrors | null => {
    const password = group.get(passwordKey)?.value;
    const confirm = group.get(confirmKey)?.value;

    if (password && confirm && password !== confirm) {
      // set error บน confirm control
      group.get(confirmKey)?.setErrors({ passwordMismatch: true });
      return { passwordMismatch: true };
    }

    // clear error ถ้า match
    const confirmControl = group.get(confirmKey);
    if (confirmControl?.errors?.['passwordMismatch']) {
      const errors = { ...confirmControl.errors };
      delete errors['passwordMismatch'];
      confirmControl.setErrors(Object.keys(errors).length ? errors : null);
    }

    return null;
  };
}

// Validator ตรวจสอบวันที่ (ต้องไม่เป็นอดีต)
export function futureDateValidator(): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null => {
    const value = control.value;
    if (!value) return null;

    const selectedDate = new Date(value);
    const today = new Date();
    today.setHours(0, 0, 0, 0);

    return selectedDate < today ? { pastDate: true } : null;
  };
}
```

### การใช้งาน Custom Validators

```typescript
// register-form.component.ts
import { Component } from '@angular/core';
import { FormBuilder, Validators } from '@angular/forms';
import { ReactiveFormsModule } from '@angular/forms';
import { CommonModule } from '@angular/common';
import {
  noWhitespaceValidator,
  thaiPhoneValidator,
  passwordMatchValidator
} from './validators/custom-validators';

@Component({
  selector: 'app-register-form',
  standalone: true,
  imports: [ReactiveFormsModule, CommonModule],
  template: `
    <form [formGroup]="registerForm" (ngSubmit)="onRegister()">
      <div class="mb-3">
        <label>ชื่อผู้ใช้</label>
        <input type="text" class="form-control" formControlName="username">
        <div *ngIf="username?.invalid && username?.touched" class="text-danger">
          <small *ngIf="username?.errors?.['required']">กรุณากรอกชื่อผู้ใช้</small>
          <small *ngIf="username?.errors?.['whitespace']">ชื่อผู้ใช้ต้องไม่เป็นช่องว่าง</small>
        </div>
      </div>

      <div class="mb-3">
        <label>เบอร์โทร</label>
        <input type="tel" class="form-control" formControlName="phone">
        <div *ngIf="phone?.invalid && phone?.touched" class="text-danger">
          <small *ngIf="phone?.errors?.['thaiPhone']">รูปแบบเบอร์โทรไม่ถูกต้อง (ต้องเริ่มด้วย 0 และมี 10 หลัก)</small>
        </div>
      </div>

      <div class="mb-3">
        <label>รหัสผ่าน</label>
        <input type="password" class="form-control" formControlName="password">
      </div>

      <div class="mb-3">
        <label>ยืนยันรหัสผ่าน</label>
        <input type="password" class="form-control" formControlName="confirmPassword">
        <div *ngIf="confirmPassword?.errors?.['passwordMismatch']" class="text-danger">
          <small>รหัสผ่านไม่ตรงกัน</small>
        </div>
      </div>

      <button type="submit" class="btn btn-primary">สมัครสมาชิก</button>
    </form>
  `
})
export class RegisterFormComponent {
  registerForm = this.fb.group({
    username: ['', [Validators.required, noWhitespaceValidator()]],
    phone: ['', [Validators.required, thaiPhoneValidator()]],
    password: ['', [Validators.required, Validators.minLength(8)]],
    confirmPassword: ['', Validators.required]
  }, {
    validators: [passwordMatchValidator('password', 'confirmPassword')]
  });

  constructor(private fb: FormBuilder) {}

  get username() { return this.registerForm.get('username'); }
  get phone() { return this.registerForm.get('phone'); }
  get confirmPassword() { return this.registerForm.get('confirmPassword'); }

  onRegister(): void {
    if (this.registerForm.valid) {
      console.log('Register:', this.registerForm.value);
    } else {
      this.registerForm.markAllAsTouched();
    }
  }
}
```

---

## 10.7 Async Validators — Validate แบบ Asynchronous

Async Validators ใช้สำหรับการตรวจสอบที่ต้องเรียก API เช่น ตรวจสอบว่า username ถูกใช้แล้วหรือยัง

```typescript
// validators/async-validators.ts
import { AbstractControl, AsyncValidatorFn, ValidationErrors } from '@angular/forms';
import { Observable, of } from 'rxjs';
import { map, catchError, debounceTime, distinctUntilChanged, switchMap, first } from 'rxjs/operators';
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';

@Injectable({ providedIn: 'root' })
export class AsyncValidatorsService {
  constructor(private http: HttpClient) {}

  // ตรวจสอบว่า username ถูกใช้แล้ว
  usernameAvailable(): AsyncValidatorFn {
    return (control: AbstractControl): Observable<ValidationErrors | null> => {
      if (!control.value) {
        return of(null);
      }

      return of(control.value).pipe(
        debounceTime(400),           // รอ 400ms ก่อน call API
        distinctUntilChanged(),      // เรียกใหม่เฉพาะค่าเปลี่ยน
        switchMap(username =>
          this.http.get<{ available: boolean }>(`/api/check-username?username=${username}`)
        ),
        map(response => response.available ? null : { usernameTaken: true }),
        catchError(() => of(null)),  // ถ้า API error ให้ผ่าน
        first()                      // complete observable หลัง emit ครั้งแรก
      );
    };
  }

  // ตรวจสอบอีเมล
  emailAvailable(): AsyncValidatorFn {
    return (control: AbstractControl): Observable<ValidationErrors | null> => {
      if (!control.value) return of(null);

      // จำลอง API call
      return of(control.value).pipe(
        debounceTime(400),
        switchMap(email =>
          this.http.get<{ exists: boolean }>(`/api/check-email?email=${email}`)
        ),
        map(response => response.exists ? { emailTaken: true } : null),
        catchError(() => of(null)),
        first()
      );
    };
  }
}
```

```typescript
// async-validation.component.ts
import { Component } from '@angular/core';
import { FormBuilder, Validators } from '@angular/forms';
import { AsyncValidatorsService } from './validators/async-validators';
import { ReactiveFormsModule } from '@angular/forms';
import { CommonModule } from '@angular/common';

@Component({
  selector: 'app-async-validation',
  standalone: true,
  imports: [ReactiveFormsModule, CommonModule],
  template: `
    <form [formGroup]="form">
      <div class="mb-3">
        <label>ชื่อผู้ใช้</label>
        <input type="text" class="form-control" formControlName="username"
          [class.is-valid]="username?.valid && username?.dirty"
          [class.is-invalid]="username?.invalid && username?.dirty">

        <!-- แสดงสถานะ loading -->
        <div *ngIf="username?.pending" class="text-muted">
          <small>กำลังตรวจสอบ...</small>
        </div>

        <div *ngIf="username?.valid && username?.dirty" class="valid-feedback">
          ชื่อผู้ใช้นี้ใช้งานได้
        </div>

        <div *ngIf="username?.invalid && username?.dirty" class="invalid-feedback">
          <small *ngIf="username?.errors?.['required']">กรุณากรอกชื่อผู้ใช้</small>
          <small *ngIf="username?.errors?.['minlength']">ต้องมีอย่างน้อย 3 ตัวอักษร</small>
          <small *ngIf="username?.errors?.['usernameTaken']">ชื่อผู้ใช้นี้ถูกใช้แล้ว</small>
        </div>
      </div>
    </form>
  `
})
export class AsyncValidationComponent {
  form = this.fb.group({
    username: [
      '',
      [Validators.required, Validators.minLength(3)],  // sync validators
      [this.asyncValidators.usernameAvailable()]         // async validators (array ที่ 3)
    ]
  });

  constructor(
    private fb: FormBuilder,
    private asyncValidators: AsyncValidatorsService
  ) {}

  get username() { return this.form.get('username'); }
}
```

---

## 10.8 valueChanges และ statusChanges Observables

```typescript
// observable-example.component.ts
import { Component, OnInit, OnDestroy } from '@angular/core';
import { FormBuilder, FormGroup } from '@angular/forms';
import { Subscription } from 'rxjs';
import { debounceTime, distinctUntilChanged, filter } from 'rxjs/operators';

@Component({
  selector: 'app-observable-example',
  standalone: true,
  imports: [ReactiveFormsModule, CommonModule],
  template: `
    <form [formGroup]="searchForm">
      <input type="text" formControlName="query" placeholder="ค้นหา..." class="form-control">
    </form>

    <div *ngIf="isSearching">กำลังค้นหา...</div>
    <ul>
      <li *ngFor="let result of searchResults">{{ result }}</li>
    </ul>
  `
})
export class ObservableExampleComponent implements OnInit, OnDestroy {
  searchForm: FormGroup;
  searchResults: string[] = [];
  isSearching = false;
  private subscriptions = new Subscription();

  constructor(private fb: FormBuilder) {
    this.searchForm = this.fb.group({
      query: ['']
    });
  }

  ngOnInit(): void {
    // ติดตาม valueChanges พร้อม debounce
    const valueChangeSub = this.searchForm.get('query')!.valueChanges.pipe(
      debounceTime(300),         // รอ 300ms หลังพิมพ์หยุด
      distinctUntilChanged(),    // ไม่ส่งถ้าค่าเหมือนเดิม
      filter(value => value.length >= 2)  // ค้นหาเมื่อพิมพ์ >= 2 ตัว
    ).subscribe(query => {
      this.performSearch(query);
    });

    this.subscriptions.add(valueChangeSub);

    // ติดตาม statusChanges ของทั้ง form
    const statusSub = this.searchForm.statusChanges.subscribe(status => {
      console.log('Form status:', status);
    });

    this.subscriptions.add(statusSub);
  }

  ngOnDestroy(): void {
    // unsubscribe เมื่อ component ถูก destroy
    this.subscriptions.unsubscribe();
  }

  private performSearch(query: string): void {
    this.isSearching = true;
    // จำลองการค้นหา
    setTimeout(() => {
      this.searchResults = [`ผลลัพธ์ 1 สำหรับ "${query}"`, `ผลลัพธ์ 2 สำหรับ "${query}"`];
      this.isSearching = false;
    }, 500);
  }
}
```

---

## 10.9 FormArray — จัดการ List ของ Items

`FormArray` ใช้สำหรับฟอร์มที่มี items แบบ dynamic เช่น รายการสินค้า, ทักษะ, หรือที่อยู่หลายรายการ

```typescript
// form-array-example.component.ts
import { Component } from '@angular/core';
import { FormBuilder, FormGroup, FormArray, Validators } from '@angular/forms';
import { ReactiveFormsModule } from '@angular/forms';
import { CommonModule } from '@angular/common';

@Component({
  selector: 'app-form-array-example',
  standalone: true,
  imports: [ReactiveFormsModule, CommonModule],
  template: `
    <form [formGroup]="resumeForm" (ngSubmit)="onSubmit()">
      <h3>ข้อมูลส่วนตัว</h3>
      <div class="mb-3">
        <label>ชื่อ-นามสกุล</label>
        <input type="text" class="form-control" formControlName="fullName">
      </div>

      <h3>ทักษะ</h3>
      <div formArrayName="skills">
        <div *ngFor="let skill of skills.controls; let i = index" class="d-flex gap-2 mb-2">
          <input
            type="text"
            class="form-control"
            [formControlName]="i"
            placeholder="ทักษะ {{ i + 1 }}"
          >
          <button type="button" class="btn btn-danger btn-sm" (click)="removeSkill(i)">
            ลบ
          </button>
        </div>
      </div>
      <button type="button" class="btn btn-secondary btn-sm mb-3" (click)="addSkill()">
        + เพิ่มทักษะ
      </button>

      <h3>ประสบการณ์การทำงาน</h3>
      <div formArrayName="experiences">
        <div *ngFor="let exp of experiences.controls; let i = index"
             [formGroupName]="i" class="card mb-3">
          <div class="card-body">
            <div class="mb-2">
              <label>บริษัท</label>
              <input type="text" class="form-control" formControlName="company">
            </div>
            <div class="mb-2">
              <label>ตำแหน่ง</label>
              <input type="text" class="form-control" formControlName="position">
            </div>
            <div class="row">
              <div class="col">
                <label>ปีที่เริ่ม</label>
                <input type="number" class="form-control" formControlName="startYear">
              </div>
              <div class="col">
                <label>ปีที่สิ้นสุด</label>
                <input type="number" class="form-control" formControlName="endYear">
              </div>
            </div>
            <button type="button" class="btn btn-danger btn-sm mt-2"
                    (click)="removeExperience(i)">
              ลบ
            </button>
          </div>
        </div>
      </div>
      <button type="button" class="btn btn-secondary btn-sm mb-3"
              (click)="addExperience()">
        + เพิ่มประสบการณ์
      </button>

      <button type="submit" class="btn btn-primary d-block">บันทึก Resume</button>
    </form>

    <pre class="mt-3">{{ resumeForm.value | json }}</pre>
  `
})
export class FormArrayExampleComponent {
  resumeForm: FormGroup;

  constructor(private fb: FormBuilder) {
    this.resumeForm = this.fb.group({
      fullName: ['', Validators.required],
      skills: this.fb.array([
        this.fb.control('Angular'),
        this.fb.control('TypeScript')
      ]),
      experiences: this.fb.array([
        this.createExperienceGroup()
      ])
    });
  }

  // Getter สำหรับ FormArray
  get skills(): FormArray {
    return this.resumeForm.get('skills') as FormArray;
  }

  get experiences(): FormArray {
    return this.resumeForm.get('experiences') as FormArray;
  }

  // สร้าง FormGroup สำหรับ experience
  private createExperienceGroup(): FormGroup {
    return this.fb.group({
      company: ['', Validators.required],
      position: ['', Validators.required],
      startYear: ['', [Validators.required, Validators.min(1900)]],
      endYear: ['']
    });
  }

  addSkill(): void {
    this.skills.push(this.fb.control('', Validators.required));
  }

  removeSkill(index: number): void {
    this.skills.removeAt(index);
  }

  addExperience(): void {
    this.experiences.push(this.createExperienceGroup());
  }

  removeExperience(index: number): void {
    this.experiences.removeAt(index);
  }

  onSubmit(): void {
    if (this.resumeForm.valid) {
      console.log('Resume data:', this.resumeForm.value);
    } else {
      this.resumeForm.markAllAsTouched();
    }
  }
}
```

---

## 10.10 Dynamic Forms — สร้างฟอร์มแบบ Dynamic

```typescript
// dynamic-form.component.ts
import { Component, OnInit } from '@angular/core';
import { FormBuilder, FormGroup, Validators } from '@angular/forms';
import { ReactiveFormsModule } from '@angular/forms';
import { CommonModule } from '@angular/common';

// Interface กำหนดโครงสร้าง field
interface FormField {
  key: string;
  label: string;
  type: 'text' | 'email' | 'number' | 'select' | 'textarea' | 'checkbox';
  required?: boolean;
  options?: { value: string; label: string }[];
  placeholder?: string;
}

@Component({
  selector: 'app-dynamic-form',
  standalone: true,
  imports: [ReactiveFormsModule, CommonModule],
  template: `
    <form [formGroup]="dynamicForm" (ngSubmit)="onSubmit()">
      <div *ngFor="let field of formFields" class="mb-3">
        <label class="form-label">
          {{ field.label }}
          <span *ngIf="field.required" class="text-danger">*</span>
        </label>

        <!-- Text / Email / Number -->
        <input
          *ngIf="['text', 'email', 'number'].includes(field.type)"
          [type]="field.type"
          class="form-control"
          [formControlName]="field.key"
          [placeholder]="field.placeholder || ''"
        >

        <!-- Textarea -->
        <textarea
          *ngIf="field.type === 'textarea'"
          class="form-control"
          [formControlName]="field.key"
          rows="3"
        ></textarea>

        <!-- Select -->
        <select
          *ngIf="field.type === 'select'"
          class="form-select"
          [formControlName]="field.key"
        >
          <option value="">-- เลือก --</option>
          <option *ngFor="let opt of field.options" [value]="opt.value">
            {{ opt.label }}
          </option>
        </select>

        <!-- Checkbox -->
        <div *ngIf="field.type === 'checkbox'" class="form-check">
          <input type="checkbox" class="form-check-input" [formControlName]="field.key">
        </div>

        <!-- Error messages -->
        <div *ngIf="dynamicForm.get(field.key)?.invalid && dynamicForm.get(field.key)?.touched"
             class="text-danger">
          <small>{{ field.label }} จำเป็นต้องกรอก</small>
        </div>
      </div>

      <button type="submit" class="btn btn-primary">ส่งข้อมูล</button>
    </form>

    <div *ngIf="submittedData" class="mt-3">
      <h5>ข้อมูลที่ส่ง:</h5>
      <pre>{{ submittedData | json }}</pre>
    </div>
  `
})
export class DynamicFormComponent implements OnInit {
  // กำหนด fields แบบ dynamic (อาจมาจาก API)
  formFields: FormField[] = [
    { key: 'name', label: 'ชื่อ', type: 'text', required: true, placeholder: 'กรอกชื่อ' },
    { key: 'email', label: 'อีเมล', type: 'email', required: true },
    { key: 'age', label: 'อายุ', type: 'number', required: false },
    {
      key: 'department', label: 'แผนก', type: 'select', required: true,
      options: [
        { value: 'dev', label: 'Development' },
        { value: 'design', label: 'Design' },
        { value: 'marketing', label: 'Marketing' }
      ]
    },
    { key: 'bio', label: 'ประวัติย่อ', type: 'textarea' },
    { key: 'agree', label: 'ยอมรับเงื่อนไข', type: 'checkbox', required: true }
  ];

  dynamicForm!: FormGroup;
  submittedData: any = null;

  constructor(private fb: FormBuilder) {}

  ngOnInit(): void {
    // สร้าง form controls จาก field definitions
    const controls: { [key: string]: any } = {};

    this.formFields.forEach(field => {
      const validators = [];
      if (field.required) {
        validators.push(Validators.required);
      }
      if (field.type === 'email') {
        validators.push(Validators.email);
      }
      controls[field.key] = ['', validators];
    });

    this.dynamicForm = this.fb.group(controls);
  }

  onSubmit(): void {
    if (this.dynamicForm.valid) {
      this.submittedData = this.dynamicForm.value;
    } else {
      this.dynamicForm.markAllAsTouched();
    }
  }
}
```

---

## 10.11 Workshop: Product Create/Edit Form

Workshop นี้จะสร้างฟอร์มครบครันสำหรับสร้างและแก้ไขสินค้า รวมถึงการจัดการรูปภาพและ tags

### สร้าง Product Model

```typescript
// models/product.model.ts
export interface Product {
  id?: number;
  name: string;
  description: string;
  price: number;
  category: string;
  stock: number;
  images: string[];
  tags: string[];
  isActive: boolean;
  specifications: ProductSpec[];
}

export interface ProductSpec {
  key: string;
  value: string;
}
```

### สร้าง Product Form Component

```typescript
// product-form/product-form.component.ts
import { Component, Input, Output, EventEmitter, OnInit, OnChanges } from '@angular/core';
import { FormBuilder, FormGroup, FormArray, Validators, AbstractControl } from '@angular/forms';
import { ReactiveFormsModule } from '@angular/forms';
import { CommonModule } from '@angular/common';
import { Product, ProductSpec } from '../models/product.model';

@Component({
  selector: 'app-product-form',
  standalone: true,
  imports: [ReactiveFormsModule, CommonModule],
  template: `
    <div class="product-form-container">
      <h2>{{ isEditMode ? 'แก้ไขสินค้า' : 'เพิ่มสินค้าใหม่' }}</h2>

      <form [formGroup]="productForm" (ngSubmit)="onSubmit()" class="needs-validation">

        <!-- ข้อมูลพื้นฐาน -->
        <div class="card mb-3">
          <div class="card-header">ข้อมูลพื้นฐาน</div>
          <div class="card-body">

            <div class="mb-3">
              <label class="form-label">ชื่อสินค้า <span class="text-danger">*</span></label>
              <input type="text" class="form-control"
                     formControlName="name"
                     [class.is-invalid]="name?.invalid && name?.touched">
              <div class="invalid-feedback">
                <span *ngIf="name?.errors?.['required']">กรุณากรอกชื่อสินค้า</span>
                <span *ngIf="name?.errors?.['minlength']">ชื่อต้องมีอย่างน้อย 3 ตัวอักษร</span>
                <span *ngIf="name?.errors?.['maxlength']">ชื่อต้องไม่เกิน 100 ตัวอักษร</span>
              </div>
            </div>

            <div class="mb-3">
              <label class="form-label">รายละเอียด</label>
              <textarea class="form-control" formControlName="description" rows="4"></textarea>
            </div>

            <div class="row">
              <div class="col-md-4 mb-3">
                <label class="form-label">ราคา (บาท) <span class="text-danger">*</span></label>
                <input type="number" class="form-control"
                       formControlName="price"
                       [class.is-invalid]="price?.invalid && price?.touched">
                <div class="invalid-feedback">
                  <span *ngIf="price?.errors?.['required']">กรุณากรอกราคา</span>
                  <span *ngIf="price?.errors?.['min']">ราคาต้องมากกว่า 0</span>
                </div>
              </div>

              <div class="col-md-4 mb-3">
                <label class="form-label">หมวดหมู่ <span class="text-danger">*</span></label>
                <select class="form-select"
                        formControlName="category"
                        [class.is-invalid]="category?.invalid && category?.touched">
                  <option value="">-- เลือกหมวดหมู่ --</option>
                  <option value="electronics">อิเล็กทรอนิกส์</option>
                  <option value="clothing">เสื้อผ้า</option>
                  <option value="food">อาหาร</option>
                  <option value="sports">กีฬา</option>
                </select>
                <div class="invalid-feedback">กรุณาเลือกหมวดหมู่</div>
              </div>

              <div class="col-md-4 mb-3">
                <label class="form-label">จำนวนสต็อก</label>
                <input type="number" class="form-control" formControlName="stock">
              </div>
            </div>

            <div class="mb-3 form-check">
              <input type="checkbox" class="form-check-input" formControlName="isActive" id="isActive">
              <label class="form-check-label" for="isActive">เปิดใช้งาน</label>
            </div>
          </div>
        </div>

        <!-- รูปภาพ -->
        <div class="card mb-3">
          <div class="card-header">รูปภาพสินค้า</div>
          <div class="card-body">
            <div formArrayName="images">
              <div *ngFor="let img of images.controls; let i = index"
                   class="d-flex gap-2 mb-2">
                <input type="url" class="form-control"
                       [formControlName]="i"
                       placeholder="URL รูปภาพ">
                <button type="button" class="btn btn-outline-danger btn-sm"
                        (click)="removeImage(i)">ลบ</button>
              </div>
            </div>
            <button type="button" class="btn btn-outline-primary btn-sm"
                    (click)="addImage()">
              + เพิ่มรูปภาพ
            </button>
          </div>
        </div>

        <!-- Tags -->
        <div class="card mb-3">
          <div class="card-header">Tags</div>
          <div class="card-body">
            <div formArrayName="tags">
              <div class="d-flex flex-wrap gap-2 mb-2">
                <div *ngFor="let tag of tags.controls; let i = index"
                     class="badge bg-primary d-flex align-items-center gap-1">
                  <input type="text"
                         style="background:transparent; border:none; color:white; width:80px"
                         [formControlName]="i">
                  <button type="button"
                          style="background:none; border:none; color:white; cursor:pointer"
                          (click)="removeTag(i)">×</button>
                </div>
              </div>
            </div>
            <button type="button" class="btn btn-outline-secondary btn-sm"
                    (click)="addTag()">
              + เพิ่ม Tag
            </button>
          </div>
        </div>

        <!-- Specifications -->
        <div class="card mb-3">
          <div class="card-header">ข้อมูลเฉพาะสินค้า</div>
          <div class="card-body">
            <div formArrayName="specifications">
              <div *ngFor="let spec of specifications.controls; let i = index"
                   [formGroupName]="i" class="row mb-2">
                <div class="col">
                  <input type="text" class="form-control" formControlName="key"
                         placeholder="คุณสมบัติ (เช่น สี)">
                </div>
                <div class="col">
                  <input type="text" class="form-control" formControlName="value"
                         placeholder="ค่า (เช่น แดง)">
                </div>
                <div class="col-auto">
                  <button type="button" class="btn btn-outline-danger btn-sm"
                          (click)="removeSpec(i)">ลบ</button>
                </div>
              </div>
            </div>
            <button type="button" class="btn btn-outline-secondary btn-sm"
                    (click)="addSpec()">
              + เพิ่มข้อมูล
            </button>
          </div>
        </div>

        <!-- Buttons -->
        <div class="d-flex gap-2">
          <button type="submit" class="btn btn-primary">
            {{ isEditMode ? 'บันทึกการแก้ไข' : 'เพิ่มสินค้า' }}
          </button>
          <button type="button" class="btn btn-secondary" (click)="onCancel()">
            ยกเลิก
          </button>
          <button type="button" class="btn btn-outline-danger" (click)="resetForm()">
            ล้างข้อมูล
          </button>
        </div>

        <!-- Form Status Debug -->
        <div class="mt-3 text-muted small">
          <span>Status: {{ productForm.status }}</span> |
          <span>Valid: {{ productForm.valid }}</span> |
          <span>Dirty: {{ productForm.dirty }}</span>
        </div>
      </form>
    </div>
  `
})
export class ProductFormComponent implements OnInit, OnChanges {
  @Input() product: Product | null = null;
  @Input() isEditMode = false;
  @Output() save = new EventEmitter<Product>();
  @Output() cancel = new EventEmitter<void>();

  productForm!: FormGroup;

  constructor(private fb: FormBuilder) {}

  ngOnInit(): void {
    this.buildForm();
    if (this.product) {
      this.patchForm(this.product);
    }
  }

  ngOnChanges(): void {
    if (this.productForm && this.product) {
      this.patchForm(this.product);
    }
  }

  private buildForm(): void {
    this.productForm = this.fb.group({
      name: ['', [
        Validators.required,
        Validators.minLength(3),
        Validators.maxLength(100)
      ]],
      description: [''],
      price: [0, [Validators.required, Validators.min(0.01)]],
      category: ['', Validators.required],
      stock: [0, [Validators.min(0)]],
      isActive: [true],
      images: this.fb.array([]),
      tags: this.fb.array([]),
      specifications: this.fb.array([])
    });
  }

  private patchForm(product: Product): void {
    // Clear arrays ก่อน
    this.images.clear();
    this.tags.clear();
    this.specifications.clear();

    // Patch basic fields
    this.productForm.patchValue({
      name: product.name,
      description: product.description,
      price: product.price,
      category: product.category,
      stock: product.stock,
      isActive: product.isActive
    });

    // Patch arrays
    product.images.forEach(img => this.images.push(this.fb.control(img)));
    product.tags.forEach(tag => this.tags.push(this.fb.control(tag)));
    product.specifications.forEach(spec => this.specifications.push(
      this.fb.group({ key: [spec.key], value: [spec.value] })
    ));
  }

  // Getters
  get name(): AbstractControl { return this.productForm.get('name')!; }
  get price(): AbstractControl { return this.productForm.get('price')!; }
  get category(): AbstractControl { return this.productForm.get('category')!; }
  get images(): FormArray { return this.productForm.get('images') as FormArray; }
  get tags(): FormArray { return this.productForm.get('tags') as FormArray; }
  get specifications(): FormArray { return this.productForm.get('specifications') as FormArray; }

  // Image methods
  addImage(): void { this.images.push(this.fb.control('')); }
  removeImage(i: number): void { this.images.removeAt(i); }

  // Tag methods
  addTag(): void { this.tags.push(this.fb.control('')); }
  removeTag(i: number): void { this.tags.removeAt(i); }

  // Spec methods
  addSpec(): void {
    this.specifications.push(this.fb.group({ key: [''], value: [''] }));
  }
  removeSpec(i: number): void { this.specifications.removeAt(i); }

  onSubmit(): void {
    if (this.productForm.valid) {
      this.save.emit(this.productForm.value as Product);
    } else {
      this.productForm.markAllAsTouched();
    }
  }

  onCancel(): void {
    this.cancel.emit();
  }

  resetForm(): void {
    this.productForm.reset({ isActive: true, price: 0, stock: 0 });
    this.images.clear();
    this.tags.clear();
    this.specifications.clear();
  }
}
```

### การใช้งาน Product Form ใน Parent Component

```typescript
// products/products.component.ts
import { Component } from '@angular/core';
import { CommonModule } from '@angular/common';
import { ProductFormComponent } from './product-form/product-form.component';
import { Product } from './models/product.model';

@Component({
  selector: 'app-products',
  standalone: true,
  imports: [CommonModule, ProductFormComponent],
  template: `
    <div class="container mt-4">
      <div *ngIf="!showForm">
        <div class="d-flex justify-content-between mb-3">
          <h1>สินค้าทั้งหมด</h1>
          <button class="btn btn-primary" (click)="showAddForm()">+ เพิ่มสินค้า</button>
        </div>

        <div class="row">
          <div *ngFor="let p of products" class="col-md-4 mb-3">
            <div class="card">
              <div class="card-body">
                <h5>{{ p.name }}</h5>
                <p class="text-muted">฿{{ p.price | number }}</p>
                <button class="btn btn-sm btn-outline-primary me-2"
                        (click)="editProduct(p)">แก้ไข</button>
              </div>
            </div>
          </div>
        </div>
      </div>

      <div *ngIf="showForm">
        <app-product-form
          [product]="selectedProduct"
          [isEditMode]="isEditMode"
          (save)="onSave($event)"
          (cancel)="onCancel()"
        ></app-product-form>
      </div>
    </div>
  `
})
export class ProductsComponent {
  showForm = false;
  isEditMode = false;
  selectedProduct: Product | null = null;

  products: Product[] = [
    {
      id: 1,
      name: 'iPhone 15 Pro',
      description: 'สมาร์ทโฟนรุ่นใหม่จาก Apple',
      price: 49900,
      category: 'electronics',
      stock: 50,
      images: ['https://example.com/iphone.jpg'],
      tags: ['apple', 'smartphone'],
      isActive: true,
      specifications: [{ key: 'สี', value: 'Titanium Black' }]
    }
  ];

  showAddForm(): void {
    this.selectedProduct = null;
    this.isEditMode = false;
    this.showForm = true;
  }

  editProduct(product: Product): void {
    this.selectedProduct = product;
    this.isEditMode = true;
    this.showForm = true;
  }

  onSave(product: Product): void {
    if (this.isEditMode && this.selectedProduct?.id) {
      const index = this.products.findIndex(p => p.id === this.selectedProduct!.id);
      this.products[index] = { ...product, id: this.selectedProduct.id };
    } else {
      this.products.push({ ...product, id: Date.now() });
    }
    this.showForm = false;
  }

  onCancel(): void {
    this.showForm = false;
  }
}
```

---

## สรุป Part 10

ใน Part นี้เราได้เรียนรู้:

- **ReactiveFormsModule** — การ setup และ import
- **FormControl** — หน่วยพื้นฐานที่ติดตามค่าและ validation state
- **FormGroup** — การรวม controls เป็นกลุ่ม รองรับ nested groups
- **FormBuilder** — shorthand สำหรับสร้างฟอร์มได้ง่ายขึ้น
- **Validators** — built-in validators ต่างๆ
- **Custom Validators** — สร้าง validator ตามความต้องการ
- **Async Validators** — validate ผ่าน API
- **valueChanges/statusChanges** — Observable สำหรับติดตามการเปลี่ยนแปลง
- **FormArray** — จัดการ list ของ items แบบ dynamic
- **Dynamic Forms** — สร้างฟอร์มจาก configuration
- **Workshop** — Product Create/Edit Form ครบวงจร

### สิ่งที่ควรฝึกเพิ่มเติม
1. สร้าง multi-step form wizard
2. การ save draft ลง localStorage ด้วย valueChanges
3. การ optimize performance ด้วย `updateOn: 'blur'` หรือ `updateOn: 'submit'`
4. การ implement form reset confirmation
