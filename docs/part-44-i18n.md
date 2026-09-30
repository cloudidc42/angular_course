# Part 44: Internationalization (i18n) ใน Angular

## บทนำ

i18n (Internationalization) คือกระบวนการทำให้แอปพลิเคชันรองรับหลายภาษาและหลาย locale ในบทนี้เราจะเรียนรู้การ setup i18n, translations, locale settings และการใช้ pipes

---

## 1. Angular Built-in i18n

Angular มี built-in i18n support ผ่าน `@angular/localize`

```bash
# ติดตั้ง localize package
ng add @angular/localize
```

### 1.1 Mark Text สำหรับ Translation

```html
<!-- app/components/home/home.component.html -->

<!-- i18n attribute สำหรับ element content -->
<h1 i18n="home page title|The main title on the home page@@homeTitle">
  ยินดีต้อนรับ
</h1>

<!-- i18n สำหรับ attribute -->
<img src="logo.png" i18n-alt="@@logoAlt" alt="Logo ของแอปพลิเคชัน">

<!-- i18n พร้อม interpolation -->
<p i18n="@@welcomeMessage">
  สวัสดี, {{ userName }}!
</p>

<!-- plural translation -->
<p i18n="@@itemCount">{count, plural,
  =0 {ไม่มีสินค้า}
  =1 {สินค้า 1 ชิ้น}
  other {สินค้า {{ count }} ชิ้น}
}</p>

<!-- select translation -->
<p i18n="@@genderGreeting">{gender, select,
  male {คุณผู้ชาย}
  female {คุณผู้หญิง}
  other {คุณ}
}</p>
```

```bash
# Extract messages
ng extract-i18n --output-path src/locales
# สร้างไฟล์ messages.xlf
```

### 1.2 Translation Files

```xml
<!-- src/locales/messages.th.xlf -->
<?xml version="1.0" encoding="UTF-8" ?>
<xliff version="1.2" xmlns="urn:oasis:names:tc:xliff:document:1.2">
  <file source-language="en-US" datatype="plaintext" original="ng2.template" target-language="th">
    <body>
      <trans-unit id="homeTitle" datatype="html">
        <source>Welcome</source>
        <target>ยินดีต้อนรับ</target>
      </trans-unit>
      <trans-unit id="logoAlt" datatype="html">
        <source>Application Logo</source>
        <target>โลโก้ของแอปพลิเคชัน</target>
      </trans-unit>
      <trans-unit id="welcomeMessage" datatype="html">
        <source>Hello, <x id="INTERPOLATION" equiv-text="{{ userName }}"/>!</source>
        <target>สวัสดี, <x id="INTERPOLATION" equiv-text="{{ userName }}"/>!</target>
      </trans-unit>
    </body>
  </xliff>
</xliff>
```

---

## 2. Angular i18n ด้วย ngx-translate (ยืดหยุ่นกว่า)

```bash
npm install @ngx-translate/core @ngx-translate/http-loader
```

### 2.1 Setup Module

```typescript
// app/app.module.ts
import { NgModule } from '@angular/core';
import { HttpClientModule, HttpClient } from '@angular/common/http';
import { TranslateModule, TranslateLoader } from '@ngx-translate/core';
import { TranslateHttpLoader } from '@ngx-translate/http-loader';

// Factory function สำหรับโหลด translation files
export function createTranslateLoader(http: HttpClient) {
  return new TranslateHttpLoader(http, './assets/i18n/', '.json');
}

@NgModule({
  imports: [
    HttpClientModule,
    TranslateModule.forRoot({
      defaultLanguage: 'th',
      loader: {
        provide: TranslateLoader,
        useFactory: createTranslateLoader,
        deps: [HttpClient]
      }
    })
  ]
})
export class AppModule {}
```

### 2.2 Translation Files (JSON)

```json
// src/assets/i18n/th.json
{
  "COMMON": {
    "SAVE": "บันทึก",
    "CANCEL": "ยกเลิก",
    "DELETE": "ลบ",
    "EDIT": "แก้ไข",
    "SEARCH": "ค้นหา",
    "LOADING": "กำลังโหลด...",
    "NO_DATA": "ไม่พบข้อมูล",
    "CONFIRM": "ยืนยัน",
    "BACK": "กลับ",
    "NEXT": "ถัดไป",
    "PREVIOUS": "ก่อนหน้า"
  },
  "AUTH": {
    "LOGIN": "เข้าสู่ระบบ",
    "LOGOUT": "ออกจากระบบ",
    "REGISTER": "สมัครสมาชิก",
    "EMAIL": "อีเมล",
    "PASSWORD": "รหัสผ่าน",
    "FORGOT_PASSWORD": "ลืมรหัสผ่าน?",
    "REMEMBER_ME": "จดจำฉัน",
    "WELCOME_BACK": "ยินดีต้อนรับกลับ, {{name}}!",
    "LOGIN_SUCCESS": "เข้าสู่ระบบสำเร็จ",
    "LOGIN_FAILED": "อีเมลหรือรหัสผ่านไม่ถูกต้อง"
  },
  "PRODUCTS": {
    "TITLE": "สินค้า",
    "ADD": "เพิ่มสินค้า",
    "EDIT": "แก้ไขสินค้า",
    "DELETE_CONFIRM": "ต้องการลบสินค้า \"{{name}}\" หรือไม่?",
    "PRICE": "ราคา",
    "STOCK": "สต็อก",
    "CATEGORY": "หมวดหมู่",
    "ITEMS_COUNT": {
      "ZERO": "ไม่มีสินค้า",
      "ONE": "สินค้า 1 ชิ้น",
      "OTHER": "สินค้า {{count}} ชิ้น"
    }
  },
  "VALIDATION": {
    "REQUIRED": "ฟิลด์นี้จำเป็นต้องกรอก",
    "EMAIL": "รูปแบบอีเมลไม่ถูกต้อง",
    "MIN_LENGTH": "ต้องมีอย่างน้อย {{min}} ตัวอักษร",
    "MAX_LENGTH": "ต้องมีไม่เกิน {{max}} ตัวอักษร"
  }
}
```

```json
// src/assets/i18n/en.json
{
  "COMMON": {
    "SAVE": "Save",
    "CANCEL": "Cancel",
    "DELETE": "Delete",
    "EDIT": "Edit",
    "SEARCH": "Search",
    "LOADING": "Loading...",
    "NO_DATA": "No data found"
  },
  "AUTH": {
    "LOGIN": "Login",
    "LOGOUT": "Logout",
    "REGISTER": "Register",
    "EMAIL": "Email",
    "PASSWORD": "Password",
    "FORGOT_PASSWORD": "Forgot password?",
    "REMEMBER_ME": "Remember me",
    "WELCOME_BACK": "Welcome back, {{name}}!",
    "LOGIN_SUCCESS": "Login successful",
    "LOGIN_FAILED": "Invalid email or password"
  },
  "PRODUCTS": {
    "TITLE": "Products",
    "ADD": "Add Product",
    "EDIT": "Edit Product",
    "DELETE_CONFIRM": "Do you want to delete \"{{name}}\"?"
  }
}
```

---

## 3. Language Service

```typescript
// app/services/language.service.ts
import { Injectable, signal } from '@angular/core';
import { TranslateService } from '@ngx-translate/core';
import { Observable } from 'rxjs';

export interface Language {
  code: string;
  name: string;
  flag: string;
  dir: 'ltr' | 'rtl';
}

@Injectable({ providedIn: 'root' })
export class LanguageService {
  readonly supportedLanguages: Language[] = [
    { code: 'th', name: 'ภาษาไทย', flag: '🇹🇭', dir: 'ltr' },
    { code: 'en', name: 'English', flag: '🇺🇸', dir: 'ltr' },
    { code: 'zh', name: '中文', flag: '🇨🇳', dir: 'ltr' },
    { code: 'ar', name: 'العربية', flag: '🇸🇦', dir: 'rtl' }
  ];

  private readonly LANG_KEY = 'app-language';
  currentLang = signal<string>(this.getSavedLanguage());

  constructor(private translate: TranslateService) {
    const langs = this.supportedLanguages.map(l => l.code);
    this.translate.addLangs(langs);
    this.translate.setDefaultLang('th');
    this.setLanguage(this.currentLang());
  }

  setLanguage(langCode: string): void {
    const lang = this.supportedLanguages.find(l => l.code === langCode);
    if (!lang) return;

    this.translate.use(langCode).subscribe(() => {
      this.currentLang.set(langCode);
      localStorage.setItem(this.LANG_KEY, langCode);
      document.documentElement.lang = langCode;
      document.documentElement.dir = lang.dir;
    });
  }

  translate$(key: string, params?: any): Observable<string> {
    return this.translate.get(key, params);
  }

  instant(key: string, params?: any): string {
    return this.translate.instant(key, params);
  }

  getCurrentLanguage(): Language {
    return this.supportedLanguages.find(l => l.code === this.currentLang())
      || this.supportedLanguages[0];
  }

  private getSavedLanguage(): string {
    return localStorage.getItem(this.LANG_KEY)
      || navigator.language.split('-')[0]
      || 'th';
  }
}
```

---

## 4. การใช้งาน TranslatePipe ใน Templates

```typescript
// app/components/header/header.component.ts
import { Component } from '@angular/core';
import { TranslateModule } from '@ngx-translate/core';
import { LanguageService, Language } from '../../services/language.service';

@Component({
  selector: 'app-header',
  standalone: true,
  imports: [TranslateModule],
  template: `
    <header class="app-header">
      <!-- ใช้ translate pipe -->
      <h1>{{ 'APP.TITLE' | translate }}</h1>

      <!-- ใช้ translate directive -->
      <button [attr.aria-label]="'COMMON.SAVE' | translate">
        {{ 'COMMON.SAVE' | translate }}
      </button>

      <!-- Translation พร้อม parameters -->
      <p>{{ 'AUTH.WELCOME_BACK' | translate: { name: currentUser?.name } }}</p>

      <!-- Language Switcher -->
      <div class="lang-switcher">
        <button
          *ngFor="let lang of languageService.supportedLanguages"
          (click)="setLang(lang)"
          [class.active]="lang.code === languageService.currentLang()"
          [title]="lang.name"
        >
          {{ lang.flag }} {{ lang.name }}
        </button>
      </div>
    </header>
  `
})
export class HeaderComponent {
  currentUser = { name: 'สมชาย' };

  constructor(public languageService: LanguageService) {}

  setLang(lang: Language) {
    this.languageService.setLanguage(lang.code);
  }
}
```

---

## 5. Locale Pipes

### 5.1 Date Pipe ตาม Locale

```typescript
// app/app.module.ts
import { LOCALE_ID, NgModule } from '@angular/core';
import { registerLocaleData } from '@angular/common';
import localeTh from '@angular/common/locales/th';
import localeEn from '@angular/common/locales/en';

registerLocaleData(localeTh);
registerLocaleData(localeEn);

@NgModule({
  providers: [
    { provide: LOCALE_ID, useValue: 'th-TH' }
  ]
})
export class AppModule {}
```

```typescript
// app/components/locale-demo/locale-demo.component.ts
import { Component } from '@angular/core';

@Component({
  selector: 'app-locale-demo',
  template: `
    <div class="locale-demo">
      <h3>ตัวอย่าง Locale Pipes</h3>

      <table>
        <tr>
          <th>ประเภท</th>
          <th>ค่า</th>
          <th>ผลลัพธ์ (th-TH)</th>
          <th>ผลลัพธ์ (en-US)</th>
        </tr>

        <!-- Date Pipe -->
        <tr>
          <td>วันที่</td>
          <td>{{ today }}</td>
          <td>{{ today | date:'fullDate':'':'th-TH' }}</td>
          <td>{{ today | date:'fullDate':'':'en-US' }}</td>
        </tr>
        <tr>
          <td>วันที่สั้น</td>
          <td></td>
          <td>{{ today | date:'dd/MM/yyyy':'':'th-TH' }}</td>
          <td>{{ today | date:'MM/dd/yyyy':'':'en-US' }}</td>
        </tr>

        <!-- Number Pipe -->
        <tr>
          <td>ตัวเลข</td>
          <td>{{ bigNumber }}</td>
          <td>{{ bigNumber | number:'1.2-2':'th-TH' }}</td>
          <td>{{ bigNumber | number:'1.2-2':'en-US' }}</td>
        </tr>

        <!-- Currency Pipe -->
        <tr>
          <td>สกุลเงิน (THB)</td>
          <td>{{ price }}</td>
          <td>{{ price | currency:'THB':'symbol':'1.2-2':'th-TH' }}</td>
          <td>{{ price | currency:'THB':'code':'1.2-2':'en-US' }}</td>
        </tr>
        <tr>
          <td>สกุลเงิน (USD)</td>
          <td></td>
          <td>{{ price | currency:'USD':'symbol':'1.2-2':'th-TH' }}</td>
          <td>{{ price | currency:'USD':'symbol':'1.2-2':'en-US' }}</td>
        </tr>

        <!-- Percent Pipe -->
        <tr>
          <td>เปอร์เซ็นต์</td>
          <td>{{ percentage }}</td>
          <td>{{ percentage | percent:'1.1-2':'th-TH' }}</td>
          <td>{{ percentage | percent:'1.1-2':'en-US' }}</td>
        </tr>
      </table>
    </div>
  `,
  styles: [`
    table { width: 100%; border-collapse: collapse; }
    th, td { padding: 8px 12px; border: 1px solid #ddd; }
    th { background: #f5f5f5; }
  `]
})
export class LocaleDemoComponent {
  today = new Date();
  bigNumber = 1234567.89;
  price = 1500.50;
  percentage = 0.856;
}
```

---

## 6. Custom Translation Pipe

```typescript
// app/pipes/translate-enum.pipe.ts
import { Pipe, PipeTransform } from '@angular/core';
import { TranslateService } from '@ngx-translate/core';

// แปล enum values ให้เป็นข้อความภาษาไทย/อังกฤษ
@Pipe({
  name: 'translateEnum',
  standalone: true,
  pure: false
})
export class TranslateEnumPipe implements PipeTransform {
  constructor(private translate: TranslateService) {}

  transform(value: string, prefix: string): string {
    const key = `${prefix}.${value.toUpperCase()}`;
    return this.translate.instant(key);
  }
}
```

```json
// ใน translation file เพิ่ม enum translations
// th.json
{
  "STATUS": {
    "ACTIVE": "ใช้งาน",
    "INACTIVE": "ปิดใช้",
    "PENDING": "รอดำเนินการ",
    "DELETED": "ถูกลบ"
  },
  "PRIORITY": {
    "LOW": "ต่ำ",
    "MEDIUM": "กลาง",
    "HIGH": "สูง",
    "CRITICAL": "วิกฤต"
  }
}
```

```html
<!-- การใช้งาน -->
<span>{{ user.status | translateEnum:'STATUS' }}</span>
<span>{{ task.priority | translateEnum:'PRIORITY' }}</span>
```

---

## 7. RTL Support

```typescript
// app/services/direction.service.ts
import { Injectable, signal } from '@angular/core';
import { DOCUMENT } from '@angular/common';
import { inject } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class DirectionService {
  private document = inject(DOCUMENT);
  direction = signal<'ltr' | 'rtl'>('ltr');

  setDirection(dir: 'ltr' | 'rtl') {
    this.direction.set(dir);
    this.document.documentElement.dir = dir;
    this.document.body.dir = dir;
    this.document.body.classList.toggle('rtl', dir === 'rtl');
  }
}
```

```scss
// styles/rtl.scss
body.rtl {
  // สลับ margin/padding สำหรับ RTL
  .app-content {
    margin-left: 0;
    margin-right: 250px; // sidenav width
  }

  .text-left { text-align: right; }
  .text-right { text-align: left; }

  // Angular Material RTL support
  direction: rtl;
}
```

---

## สรุป

| แนวทาง | ข้อดี | ข้อเสีย |
|--------|------|--------|
| Angular Built-in i18n | Official, compile-time | Build หลายครั้ง, ไม่ flexible |
| ngx-translate | Dynamic, ยืดหยุ่น, runtime | ต้องติดตั้งเพิ่ม |

สำหรับแอปพลิเคชันที่ต้องการเปลี่ยนภาษาแบบ real-time แนะนำใช้ `ngx-translate` ส่วน Angular built-in i18n เหมาะกับแอปที่ build แยก bundle ต่อ locale
