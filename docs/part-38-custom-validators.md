# Part 38: Custom Validators ใน Angular

## บทนำ

Angular มี built-in validators เช่น `required`, `email`, `minLength` แต่บ่อยครั้งเราต้องการ validation logic ที่ซับซ้อนกว่านั้น บทนี้จะครอบคลุม Sync Validators, Async Validators, Validator Factories และ Composed Validators

---

## 1. Sync Custom Validators

### 1.1 Validator แบบง่าย

```typescript
// app/validators/custom.validators.ts
import { AbstractControl, ValidationErrors, ValidatorFn } from '@angular/forms';

// ตรวจสอบเบอร์โทรศัพท์ไทย
export function thaiPhoneValidator(control: AbstractControl): ValidationErrors | null {
  if (!control.value) return null; // ถ้าว่างปล่อยให้ required validator จัดการ

  const phoneRegex = /^(\+66|0)[689]\d{8}$/;
  const cleaned = control.value.replace(/[\s-]/g, '');

  return phoneRegex.test(cleaned)
    ? null
    : { thaiPhone: { value: control.value, message: 'เบอร์โทรศัพท์ไม่ถูกต้อง' } };
}

// ตรวจสอบเลขบัตรประชาชนไทย
export function thaiIdValidator(control: AbstractControl): ValidationErrors | null {
  if (!control.value) return null;

  const id = control.value.replace(/\D/g, '');
  if (id.length !== 13) {
    return { thaiId: { message: 'เลขบัตรประชาชนต้องมี 13 หลัก' } };
  }

  // Algorithm ตรวจสอบ checksum
  let sum = 0;
  for (let i = 0; i < 12; i++) {
    sum += parseInt(id[i]) * (13 - i);
  }

  const checkDigit = (11 - (sum % 11)) % 10;
  if (checkDigit !== parseInt(id[12])) {
    return { thaiId: { message: 'เลขบัตรประชาชนไม่ถูกต้อง' } };
  }

  return null;
}

// ตรวจสอบรหัสผ่านที่แข็งแกร่ง
export function strongPasswordValidator(control: AbstractControl): ValidationErrors | null {
  if (!control.value) return null;

  const password = control.value as string;
  const errors: string[] = [];

  if (password.length < 8) errors.push('ต้องมีอย่างน้อย 8 ตัวอักษร');
  if (!/[A-Z]/.test(password)) errors.push('ต้องมีตัวพิมพ์ใหญ่');
  if (!/[a-z]/.test(password)) errors.push('ต้องมีตัวพิมพ์เล็ก');
  if (!/\d/.test(password)) errors.push('ต้องมีตัวเลข');
  if (!/[!@#$%^&*(),.?":{}|<>]/.test(password)) errors.push('ต้องมีอักขระพิเศษ');

  return errors.length > 0
    ? { strongPassword: { requirements: errors } }
    : null;
}

// ตรวจสอบ URL
export function urlValidator(control: AbstractControl): ValidationErrors | null {
  if (!control.value) return null;

  try {
    new URL(control.value);
    return null;
  } catch {
    return { invalidUrl: { value: control.value } };
  }
}

// ตรวจสอบช่วงตัวเลข
export function numberRangeValidator(min: number, max: number): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null => {
    if (!control.value) return null;

    const value = parseFloat(control.value);
    if (isNaN(value)) return { notANumber: true };
    if (value < min) return { belowMin: { min, actual: value } };
    if (value > max) return { aboveMax: { max, actual: value } };

    return null;
  };
}
```

### 1.2 Validator Factories

```typescript
// app/validators/validator-factories.ts
import { AbstractControl, ValidationErrors, ValidatorFn } from '@angular/forms';

// Factory สำหรับ forbidden words
export function forbiddenWordsValidator(forbiddenWords: string[]): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null => {
    if (!control.value) return null;

    const value = control.value.toLowerCase();
    const found = forbiddenWords.find(word => value.includes(word.toLowerCase()));

    return found
      ? { forbiddenWord: { word: found } }
      : null;
  };
}

// Factory สำหรับ file type
export function fileTypeValidator(allowedTypes: string[]): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null => {
    const file = control.value as File;
    if (!file) return null;

    const fileExtension = file.name.split('.').pop()?.toLowerCase() || '';
    const allowed = allowedTypes.map(t => t.toLowerCase());

    return allowed.includes(fileExtension)
      ? null
      : { invalidFileType: { allowed: allowedTypes, actual: fileExtension } };
  };
}

// Factory สำหรับ max file size
export function maxFileSizeValidator(maxSizeInMB: number): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null => {
    const file = control.value as File;
    if (!file) return null;

    const maxBytes = maxSizeInMB * 1024 * 1024;
    if (file.size > maxBytes) {
      return {
        fileTooLarge: {
          maxSize: `${maxSizeInMB}MB`,
          actualSize: `${(file.size / 1024 / 1024).toFixed(2)}MB`
        }
      };
    }

    return null;
  };
}

// Factory สำหรับตรวจสอบว่าไม่ตรงกับค่าที่กำหนด
export function notEqualToValidator(forbiddenValue: any): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null => {
    return control.value === forbiddenValue
      ? { notEqualTo: { forbiddenValue } }
      : null;
  };
}

// Factory สำหรับ conditional required
export function conditionalRequiredValidator(
  conditionFn: (form: AbstractControl) => boolean
): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null => {
    const form = control.parent;
    if (!form) return null;

    if (conditionFn(form) && !control.value) {
      return { conditionalRequired: true };
    }

    return null;
  };
}
```

---

## 2. Async Validators

```typescript
// app/validators/async.validators.ts
import {
  AbstractControl,
  ValidationErrors,
  AsyncValidatorFn
} from '@angular/forms';
import { Observable, of, timer } from 'rxjs';
import { map, switchMap, catchError, first } from 'rxjs/operators';
import { inject } from '@angular/core';
import { UserService } from '../services/user.service';

// ตรวจสอบว่า username ถูกใช้ไปแล้ว (ด้วย debounce)
export function usernameAvailableValidator(userService: UserService): AsyncValidatorFn {
  return (control: AbstractControl): Observable<ValidationErrors | null> => {
    if (!control.value) return of(null);

    // debounce 500ms เพื่อลด API calls
    return timer(500).pipe(
      switchMap(() => userService.checkUsernameAvailable(control.value)),
      map(isAvailable => isAvailable ? null : { usernameTaken: true }),
      catchError(() => of(null)), // ถ้า API error ให้ผ่าน
      first() // complete observable
    );
  };
}

// ตรวจสอบ email
export function emailAvailableValidator(userService: UserService): AsyncValidatorFn {
  return (control: AbstractControl): Observable<ValidationErrors | null> => {
    if (!control.value) return of(null);

    return timer(300).pipe(
      switchMap(() => userService.checkEmailAvailable(control.value)),
      map(result => {
        if (!result.available) {
          return { emailTaken: { suggestions: result.suggestions } };
        }
        return null;
      }),
      catchError(() => of(null)),
      first()
    );
  };
}
```

```typescript
// app/services/user.service.ts
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable, of } from 'rxjs';
import { delay } from 'rxjs/operators';

@Injectable({ providedIn: 'root' })
export class UserService {
  private takenUsernames = ['admin', 'user', 'test', 'guest'];
  private takenEmails = ['admin@example.com', 'test@example.com'];

  constructor(private http: HttpClient) {}

  checkUsernameAvailable(username: string): Observable<boolean> {
    // จำลอง API call
    const isTaken = this.takenUsernames.includes(username.toLowerCase());
    return of(!isTaken).pipe(delay(200));
  }

  checkEmailAvailable(email: string): Observable<{ available: boolean; suggestions?: string[] }> {
    const isTaken = this.takenEmails.includes(email.toLowerCase());

    if (isTaken) {
      const base = email.split('@')[0];
      return of({
        available: false,
        suggestions: [`${base}1@example.com`, `${base}_2@example.com`]
      }).pipe(delay(300));
    }

    return of({ available: true }).pipe(delay(300));
  }
}
```

---

## 3. ใช้ Validators ใน Component

```typescript
// app/components/signup/signup.component.ts
import { Component, OnInit } from '@angular/core';
import {
  FormBuilder,
  FormGroup,
  Validators,
  AbstractControl
} from '@angular/forms';
import { UserService } from '../../services/user.service';
import {
  thaiPhoneValidator,
  strongPasswordValidator,
  numberRangeValidator
} from '../../validators/custom.validators';
import { forbiddenWordsValidator } from '../../validators/validator-factories';
import {
  usernameAvailableValidator,
  emailAvailableValidator
} from '../../validators/async.validators';
import { matchFieldsValidator } from '../../validators/cross-field.validators';

@Component({
  selector: 'app-signup',
  template: `
    <form [formGroup]="signupForm" (ngSubmit)="onSubmit()">
      <h2>สมัครสมาชิก</h2>

      <!-- Username field -->
      <div class="form-group">
        <label>Username:</label>
        <input formControlName="username">
        <div class="status-indicator">
          <span *ngIf="username?.pending" class="checking">กำลังตรวจสอบ...</span>
          <span *ngIf="username?.valid && !username?.pending" class="valid">✓ ใช้ได้</span>
        </div>
        <div class="errors" *ngIf="username?.invalid && username?.touched">
          <small *ngIf="username?.errors?.['required']">กรุณาใส่ username</small>
          <small *ngIf="username?.errors?.['minlength']">ต้องมีอย่างน้อย 4 ตัวอักษร</small>
          <small *ngIf="username?.errors?.['forbiddenWord']">
            ไม่สามารถใช้คำว่า "{{ username?.errors?.['forbiddenWord']?.word }}" ได้
          </small>
          <small *ngIf="username?.errors?.['usernameTaken']">Username นี้ถูกใช้แล้ว</small>
        </div>
      </div>

      <!-- Email field -->
      <div class="form-group">
        <label>Email:</label>
        <input type="email" formControlName="email">
        <div *ngIf="email?.errors?.['emailTaken']" class="suggestion">
          <p>Email นี้ถูกใช้แล้ว แนะนำ:</p>
          <button
            *ngFor="let s of email?.errors?.['emailTaken']?.suggestions"
            type="button"
            (click)="useEmailSuggestion(s)"
          >{{ s }}</button>
        </div>
      </div>

      <!-- Phone field -->
      <div class="form-group">
        <label>เบอร์โทรศัพท์:</label>
        <input formControlName="phone" placeholder="0812345678">
        <div *ngIf="phone?.invalid && phone?.touched">
          <small *ngIf="phone?.errors?.['thaiPhone']">
            {{ phone?.errors?.['thaiPhone']?.message }}
          </small>
        </div>
      </div>

      <!-- Age field -->
      <div class="form-group">
        <label>อายุ:</label>
        <input type="number" formControlName="age">
        <div *ngIf="age?.invalid && age?.touched">
          <small *ngIf="age?.errors?.['belowMin']">ต้องมีอายุอย่างน้อย 18 ปี</small>
          <small *ngIf="age?.errors?.['aboveMax']">อายุต้องไม่เกิน 120 ปี</small>
        </div>
      </div>

      <!-- Password fields -->
      <div formGroupName="passwords">
        <div class="form-group">
          <label>รหัสผ่าน:</label>
          <input type="password" formControlName="password">
          <div *ngIf="password?.invalid && password?.touched">
            <ul *ngIf="password?.errors?.['strongPassword']">
              <li *ngFor="let req of password?.errors?.['strongPassword']?.requirements">
                {{ req }}
              </li>
            </ul>
          </div>
        </div>

        <div class="form-group">
          <label>ยืนยันรหัสผ่าน:</label>
          <input type="password" formControlName="confirmPassword">
          <small *ngIf="passwordsGroup?.errors?.['fieldsMismatch']">
            รหัสผ่านไม่ตรงกัน
          </small>
        </div>
      </div>

      <button type="submit" [disabled]="signupForm.invalid || signupForm.pending">
        <span *ngIf="signupForm.pending">กำลังตรวจสอบ...</span>
        <span *ngIf="!signupForm.pending">สมัครสมาชิก</span>
      </button>
    </form>
  `
})
export class SignupComponent implements OnInit {
  signupForm!: FormGroup;

  constructor(
    private fb: FormBuilder,
    private userService: UserService
  ) {}

  ngOnInit() {
    this.signupForm = this.fb.group({
      username: [
        '',
        {
          validators: [
            Validators.required,
            Validators.minLength(4),
            forbiddenWordsValidator(['admin', 'root', 'superuser'])
          ],
          asyncValidators: [usernameAvailableValidator(this.userService)],
          updateOn: 'blur'
        }
      ],
      email: [
        '',
        {
          validators: [Validators.required, Validators.email],
          asyncValidators: [emailAvailableValidator(this.userService)],
          updateOn: 'blur'
        }
      ],
      phone: ['', thaiPhoneValidator],
      age: ['', [Validators.required, numberRangeValidator(18, 120)]],
      passwords: this.fb.group(
        {
          password: ['', [Validators.required, strongPasswordValidator]],
          confirmPassword: ['', Validators.required]
        },
        { validators: matchFieldsValidator('password', 'confirmPassword') }
      )
    });
  }

  get username() { return this.signupForm.get('username'); }
  get email() { return this.signupForm.get('email'); }
  get phone() { return this.signupForm.get('phone'); }
  get age() { return this.signupForm.get('age'); }
  get password() { return this.signupForm.get('passwords.password'); }
  get passwordsGroup() { return this.signupForm.get('passwords'); }

  useEmailSuggestion(email: string) {
    this.signupForm.patchValue({ email });
  }

  onSubmit() {
    if (this.signupForm.valid) {
      console.log('Signup:', this.signupForm.value);
    } else {
      this.signupForm.markAllAsTouched();
    }
  }
}
```

---

## 4. Composed Validators

```typescript
// app/validators/composed.validators.ts
import {
  AbstractControl,
  ValidationErrors,
  ValidatorFn,
  Validators
} from '@angular/forms';
import { thaiPhoneValidator, strongPasswordValidator } from './custom.validators';

// รวม validators หลายตัวเข้าด้วยกัน
export function composeValidators(...validators: ValidatorFn[]): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null => {
    const errors: ValidationErrors = {};
    let hasError = false;

    for (const validator of validators) {
      const result = validator(control);
      if (result) {
        Object.assign(errors, result);
        hasError = true;
      }
    }

    return hasError ? errors : null;
  };
}

// Validators สำเร็จรูปสำหรับกรณีที่ใช้บ่อย
export const ProfileValidators = {
  name: composeValidators(
    Validators.required,
    Validators.minLength(2),
    Validators.maxLength(100)
  ),

  password: composeValidators(
    Validators.required,
    Validators.minLength(8),
    strongPasswordValidator
  ),

  thaiPhone: composeValidators(
    Validators.required,
    thaiPhoneValidator
  )
};

// Conditional validator - ทำงานเฉพาะเมื่อเงื่อนไขเป็นจริง
export function when(
  condition: (control: AbstractControl) => boolean,
  validator: ValidatorFn
): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null => {
    return condition(control) ? validator(control) : null;
  };
}
```

---

## 5. Error Messages Service

```typescript
// app/services/form-error.service.ts
import { Injectable } from '@angular/core';
import { AbstractControl } from '@angular/forms';

type ErrorMessageMap = Record<string, (error: any) => string>;

@Injectable({ providedIn: 'root' })
export class FormErrorService {
  private errorMessages: ErrorMessageMap = {
    required: () => 'ฟิลด์นี้จำเป็นต้องกรอก',
    email: () => 'รูปแบบ email ไม่ถูกต้อง',
    minlength: (err) => `ต้องมีอย่างน้อย ${err.requiredLength} ตัวอักษร (มี ${err.actualLength})`,
    maxlength: (err) => `ต้องมีไม่เกิน ${err.requiredLength} ตัวอักษร`,
    min: (err) => `ค่าต้องไม่น้อยกว่า ${err.min}`,
    max: (err) => `ค่าต้องไม่มากกว่า ${err.max}`,
    pattern: () => 'รูปแบบไม่ถูกต้อง',
    thaiPhone: (err) => err.message || 'เบอร์โทรศัพท์ไม่ถูกต้อง',
    thaiId: (err) => err.message || 'เลขบัตรประชาชนไม่ถูกต้อง',
    strongPassword: (err) => err.requirements?.join(', ') || 'รหัสผ่านไม่แข็งแกร่งพอ',
    usernameTaken: () => 'Username นี้ถูกใช้แล้ว',
    emailTaken: () => 'Email นี้ถูกใช้แล้ว',
    fieldsMismatch: () => 'ค่าไม่ตรงกัน',
    invalidDateRange: () => 'ช่วงวันที่ไม่ถูกต้อง',
    forbiddenWord: (err) => `ไม่สามารถใช้คำว่า "${err.word}" ได้`
  };

  getErrors(control: AbstractControl): string[] {
    if (!control.errors) return [];

    return Object.entries(control.errors)
      .map(([key, value]) => {
        const messageFn = this.errorMessages[key];
        return messageFn ? messageFn(value) : `Validation error: ${key}`;
      });
  }

  getFirstError(control: AbstractControl): string {
    const errors = this.getErrors(control);
    return errors[0] || '';
  }

  registerErrorMessage(key: string, messageFn: (error: any) => string) {
    this.errorMessages[key] = messageFn;
  }
}
```

```typescript
// app/components/field-error/field-error.component.ts
import { Component, Input } from '@angular/core';
import { AbstractControl } from '@angular/forms';
import { FormErrorService } from '../../services/form-error.service';

@Component({
  selector: 'app-field-error',
  template: `
    <div *ngIf="shouldShow" class="field-errors">
      <small *ngFor="let error of errors" class="error-message">
        {{ error }}
      </small>
    </div>
  `,
  styles: [`
    .field-errors { margin-top: 4px; }
    .error-message { display: block; color: #dc3545; font-size: 12px; }
  `]
})
export class FieldErrorComponent {
  @Input() control?: AbstractControl | null;
  @Input() showOn: 'touched' | 'dirty' | 'always' = 'touched';

  constructor(private errorService: FormErrorService) {}

  get shouldShow(): boolean {
    if (!this.control?.invalid) return false;

    switch (this.showOn) {
      case 'touched': return !!this.control.touched;
      case 'dirty': return !!this.control.dirty;
      case 'always': return true;
      default: return false;
    }
  }

  get errors(): string[] {
    if (!this.control) return [];
    return this.errorService.getErrors(this.control);
  }
}
```

---

## สรุป

| ประเภท | ใช้เมื่อ |
|-------|---------|
| Sync Validator | ตรวจสอบข้อมูลทันที ไม่ต้องรอ API |
| Async Validator | ตรวจสอบกับ server (username, email) |
| Validator Factory | ต้องการ parameter สำหรับ validation |
| Composed Validator | รวมหลาย validators เข้าด้วยกัน |
| Cross-field Validator | ตรวจสอบความสัมพันธ์ระหว่างหลายฟิลด์ |

Custom Validators ช่วยให้ validation logic เป็น reusable และ testable โดยแยกออกจาก Component
