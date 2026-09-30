# Part 06 — Pipes และการจัดการข้อมูล

## สารบัญ

1. [Pipes คืออะไร](#pipes-intro)
2. [Built-in Pipes: date](#date-pipe)
3. [Built-in Pipes: currency, number, decimal, percent](#number-pipes)
4. [Built-in Pipes: uppercase, lowercase, titlecase](#case-pipes)
5. [Built-in Pipes: json, keyvalue, async](#utility-pipes)
6. [Built-in Pipes: slice](#slice-pipe)
7. [การสร้าง Custom Pipe](#custom-pipe)
8. [Pure vs Impure Pipe](#pure-vs-impure)
9. [Pipe Chaining และ Parameterization](#pipe-chaining)
10. [Workshop: Data Formatting Pipe Collection](#workshop)

---

## 1. Pipes คืออะไร {#pipes-intro}

Pipe คือ function ที่รับค่าเข้ามา แปลงค่า และส่งค่าที่แปลงแล้วออกไป โดยใช้ใน template ด้วย operator `|`

```
value | pipeName:param1:param2
```

### รูปแบบการใช้งาน

```typescript
// ตัวอย่างพื้นฐาน
{{ 'hello world' | uppercase }}               // HELLO WORLD
{{ 1234567.89 | number:'1.2-2' }}             // 1,234,567.89
{{ today | date:'dd/MM/yyyy' }}               // 25/12/2024
{{ price | currency:'THB':'symbol':'1.0-0' }} // ฿35,000
{{ data | json }}                             // {"key": "value"}
{{ longText | slice:0:100 }}                  // ตัวอักษร 100 ตัวแรก
```

### Pipe ใน TypeScript

```typescript
// pipes-demo.component.ts
import { Component } from '@angular/core';
import {
  DatePipe,
  CurrencyPipe,
  DecimalPipe,
  PercentPipe,
  UpperCasePipe,
  LowerCasePipe,
  TitleCasePipe,
  JsonPipe,
  SlicePipe,
  KeyValuePipe,
  AsyncPipe
} from '@angular/common';

@Component({
  selector: 'app-pipes-demo',
  standalone: true,
  imports: [
    DatePipe, CurrencyPipe, DecimalPipe, PercentPipe,
    UpperCasePipe, LowerCasePipe, TitleCasePipe,
    JsonPipe, SlicePipe, KeyValuePipe, AsyncPipe
  ],
  template: `<div>สาธิต Pipes</div>`
})
export class PipesDemoComponent {}
```

### การใช้ Pipe ใน TypeScript code (ไม่ใช่แค่ template)

```typescript
// การใช้ pipe ใน component/service
import { DatePipe, CurrencyPipe } from '@angular/common';
import { Component, inject } from '@angular/core';

@Component({...})
export class MyComponent {
  // วิธีที่ 1: inject ผ่าน constructor
  private datePipe = inject(DatePipe);
  private currencyPipe = inject(CurrencyPipe);

  formatDateForApi(date: Date): string {
    return this.datePipe.transform(date, 'yyyy-MM-dd') ?? '';
  }

  formatPrice(price: number): string {
    return this.currencyPipe.transform(price, 'THB', 'symbol', '1.0-0') ?? '0';
  }
}

// ต้อง provide ใน providers ด้วย
// providers: [DatePipe, CurrencyPipe]
```

---

## 2. Built-in Pipes: date {#date-pipe}

`DatePipe` ใช้สำหรับจัดรูปแบบวันที่และเวลา

### รูปแบบที่ใช้ได้

```typescript
// date-pipe.component.ts
import { Component } from '@angular/core';
import { DatePipe } from '@angular/common';

@Component({
  selector: 'app-date-pipe',
  standalone: true,
  imports: [DatePipe],
  template: `
    <div>
      <h3>Date Pipe ตัวอย่าง</h3>
      
      <!-- รูปแบบมาตรฐาน -->
      <p>short: {{ today | date:'short' }}</p>
      <!-- 12/25/24, 10:30 AM -->
      
      <p>medium: {{ today | date:'medium' }}</p>
      <!-- Dec 25, 2024, 10:30:00 AM -->
      
      <p>long: {{ today | date:'long' }}</p>
      <!-- December 25, 2024 at 10:30:00 AM GMT+7 -->
      
      <p>full: {{ today | date:'full' }}</p>
      <!-- Wednesday, December 25, 2024 at 10:30:00 AM Indochina Time -->
      
      <p>shortDate: {{ today | date:'shortDate' }}</p>
      <!-- 12/25/24 -->
      
      <p>mediumDate: {{ today | date:'mediumDate' }}</p>
      <!-- Dec 25, 2024 -->
      
      <p>longDate: {{ today | date:'longDate' }}</p>
      <!-- December 25, 2024 -->
      
      <p>fullDate: {{ today | date:'fullDate' }}</p>
      <!-- Wednesday, December 25, 2024 -->
      
      <p>shortTime: {{ today | date:'shortTime' }}</p>
      <!-- 10:30 AM -->
      
      <p>mediumTime: {{ today | date:'mediumTime' }}</p>
      <!-- 10:30:00 AM -->
      
      <!-- รูปแบบที่กำหนดเอง -->
      <p>dd/MM/yyyy: {{ today | date:'dd/MM/yyyy' }}</p>
      <!-- 25/12/2024 -->
      
      <p>วันที่ไทย: {{ today | date:'d MMMM yyyy' }}</p>
      <!-- 25 December 2024 -->
      
      <p>รูปแบบสั้น: {{ today | date:'d/M/yy' }}</p>
      <!-- 25/12/24 -->
      
      <p>พร้อมเวลา: {{ today | date:'dd/MM/yyyy HH:mm:ss' }}</p>
      <!-- 25/12/2024 10:30:45 -->
      
      <p>เวลา 12 ชั่วโมง: {{ today | date:'hh:mm:ss a' }}</p>
      <!-- 10:30:45 AM -->
      
      <p>เวลา 24 ชั่วโมง: {{ today | date:'HH:mm' }}</p>
      <!-- 10:30 -->
      
      <p>วันในสัปดาห์: {{ today | date:'EEEE' }}</p>
      <!-- Wednesday -->
      
      <p>วันย่อ: {{ today | date:'EEE' }}</p>
      <!-- Wed -->
      
      <p>เดือน: {{ today | date:'MMMM' }}</p>
      <!-- December -->
      
      <p>เดือนย่อ: {{ today | date:'MMM' }}</p>
      <!-- Dec -->
      
      <p>ปี: {{ today | date:'yyyy' }}</p>
      <!-- 2024 -->
      
      <p>ปี 2 หลัก: {{ today | date:'yy' }}</p>
      <!-- 24 -->
      
      <!-- Locale -->
      <p>Thai locale: {{ today | date:'fullDate':'':'th' }}</p>
      
      <!-- Timezone -->
      <p>UTC: {{ today | date:'dd/MM/yyyy HH:mm':'UTC' }}</p>
      <p>Bangkok: {{ today | date:'dd/MM/yyyy HH:mm':'+0700' }}</p>
      
      <!-- null/undefined handling -->
      <p>Null date: {{ nullDate | date:'dd/MM/yyyy' }}</p>
      <!-- ไม่แสดงอะไร (returns null) -->
      
      <!-- ใช้กับ Unix timestamp -->
      <p>Unix timestamp: {{ unixTimestamp | date:'dd/MM/yyyy' }}</p>
      
      <!-- ใช้กับ ISO string -->
      <p>ISO string: {{ isoString | date:'dd/MM/yyyy HH:mm' }}</p>
    </div>
  `
})
export class DatePipeComponent {
  today = new Date();
  nullDate: Date | null = null;
  unixTimestamp = 1703487045000; // milliseconds
  isoString = '2024-12-25T10:30:00.000Z';
}
```

### Date Format Symbols

| Symbol | ความหมาย | ตัวอย่าง |
|--------|----------|----------|
| `d` | วันที่ (1-31) | 5, 25 |
| `dd` | วันที่ (01-31) | 05, 25 |
| `M` | เดือน (1-12) | 1, 12 |
| `MM` | เดือน (01-12) | 01, 12 |
| `MMM` | เดือนย่อ | Jan, Dec |
| `MMMM` | เดือนเต็ม | January, December |
| `y` | ปี | 2024 |
| `yy` | ปี 2 หลัก | 24 |
| `yyyy` | ปี 4 หลัก | 2024 |
| `H` | ชั่วโมง 0-23 | 0, 23 |
| `HH` | ชั่วโมง 00-23 | 00, 23 |
| `h` | ชั่วโมง 1-12 | 1, 12 |
| `hh` | ชั่วโมง 01-12 | 01, 12 |
| `m` | นาที 0-59 | 0, 59 |
| `mm` | นาที 00-59 | 00, 59 |
| `s` | วินาที | 0, 59 |
| `ss` | วินาที 00-59 | 00, 59 |
| `a` | AM/PM | AM, PM |
| `E`/`EE`/`EEE` | วันย่อ | Mon, Tue |
| `EEEE` | วันเต็ม | Monday |

### DatePipe ใน Service

```typescript
// date.service.ts
import { Injectable } from '@angular/core';
import { DatePipe } from '@angular/common';

@Injectable({
  providedIn: 'root'
})
export class DateService {
  constructor(private datePipe: DatePipe) {}

  formatForDisplay(date: Date | string | null): string {
    return this.datePipe.transform(date, 'dd/MM/yyyy') ?? '-';
  }

  formatDateTime(date: Date | string | null): string {
    return this.datePipe.transform(date, 'dd/MM/yyyy HH:mm') ?? '-';
  }

  formatForApi(date: Date): string {
    return this.datePipe.transform(date, 'yyyy-MM-dd') ?? '';
  }

  getRelativeTime(date: Date): string {
    const now = new Date();
    const diffMs = now.getTime() - date.getTime();
    const diffMins = Math.floor(diffMs / 60000);
    const diffHours = Math.floor(diffMins / 60);
    const diffDays = Math.floor(diffHours / 24);

    if (diffMins < 1) return 'เพิ่งตอนนี้';
    if (diffMins < 60) return `${diffMins} นาทีที่แล้ว`;
    if (diffHours < 24) return `${diffHours} ชั่วโมงที่แล้ว`;
    if (diffDays < 7) return `${diffDays} วันที่แล้ว`;
    return this.datePipe.transform(date, 'dd/MM/yyyy') ?? '';
  }
}
```

---

## 3. Built-in Pipes: currency, number, decimal, percent {#number-pipes}

### CurrencyPipe

```typescript
// currency-pipe.component.ts
import { Component } from '@angular/core';
import { CurrencyPipe, DecimalPipe, PercentPipe } from '@angular/common';

@Component({
  selector: 'app-number-pipes',
  standalone: true,
  imports: [CurrencyPipe, DecimalPipe, PercentPipe],
  template: `
    <div>
      <h3>Currency Pipe</h3>
      
      <!-- รูปแบบ: {{ value | currency:code:display:digitsInfo:locale }} -->
      
      <!-- บาทไทย -->
      <p>{{ price | currency:'THB' }}</p>
      <!-- THB35,000.00 -->
      
      <p>{{ price | currency:'THB':'symbol' }}</p>
      <!-- ฿35,000.00 -->
      
      <p>{{ price | currency:'THB':'symbol':'1.0-0' }}</p>
      <!-- ฿35,000 -->
      
      <p>{{ price | currency:'THB':'code' }}</p>
      <!-- THB35,000.00 -->
      
      <p>{{ price | currency:'THB':'symbol-narrow' }}</p>
      <!-- ฿35,000.00 -->
      
      <!-- ดอลลาร์สหรัฐ -->
      <p>{{ usdPrice | currency:'USD':'symbol':'1.2-2' }}</p>
      <!-- $1,234.56 -->
      
      <!-- ยูโร -->
      <p>{{ euroPrice | currency:'EUR':'symbol':'1.2-2' }}</p>
      <!-- €1,234.56 -->
      
      <!-- เยนญี่ปุ่น -->
      <p>{{ yenPrice | currency:'JPY':'symbol':'1.0-0' }}</p>
      <!-- ¥150,000 -->
      
      <!-- Custom: ไม่แสดง symbol -->
      <p>{{ price | currency:'THB':'':'1.0-0' }}</p>
      <!-- 35,000 -->
      
      <hr>
      
      <h3>Decimal (Number) Pipe</h3>
      
      <!-- รูปแบบ: {{ value | number:digitsInfo:locale }} -->
      <!-- digitsInfo: minInteger.minFraction-maxFraction -->
      
      <p>{{ bigNumber | number }}</p>
      <!-- 1,234,567.89 (default: 1.0-3) -->
      
      <p>{{ bigNumber | number:'1.2-2' }}</p>
      <!-- 1,234,567.89 -->
      
      <p>{{ bigNumber | number:'1.0-0' }}</p>
      <!-- 1,234,568 (ปัดเศษ) -->
      
      <p>{{ smallNumber | number:'1.3-5' }}</p>
      <!-- 0.123 (3-5 decimal places) -->
      
      <p>{{ integer | number:'4.0-0' }}</p>
      <!-- 0,042 (padded with leading zeros) -->
      
      <p>{{ bigNumber | number:'':'th' }}</p>
      <!-- ด้วย locale ไทย -->
      
      <hr>
      
      <h3>Percent Pipe</h3>
      
      <!-- รูปแบบ: {{ value | percent:digitsInfo:locale }} -->
      <!-- value คือ 0-1 (0.75 = 75%) -->
      
      <p>{{ 0.75 | percent }}</p>
      <!-- 75% -->
      
      <p>{{ 0.1234 | percent:'1.1-2' }}</p>
      <!-- 12.34% -->
      
      <p>{{ 1.5 | percent }}</p>
      <!-- 150% -->
      
      <p>{{ 0.0056 | percent:'1.2-2' }}</p>
      <!-- 0.56% -->
      
      <!-- ใช้กับ progress bar -->
      <div class="progress-bar">
        <div
          class="progress-fill"
          [style.width]="(completionRate * 100) + '%'">
        </div>
        <span>{{ completionRate | percent:'1.1-1' }}</span>
      </div>
    </div>
  `
})
export class NumberPipesComponent {
  price = 35000;
  usdPrice = 1234.56;
  euroPrice = 1234.56;
  yenPrice = 150000;
  bigNumber = 1234567.89;
  smallNumber = 0.12345;
  integer = 42;
  completionRate = 0.657;
}
```

### DigitsInfo Format

```
{minIntegerDigits}.{minFractionDigits}-{maxFractionDigits}

ตัวอย่าง:
'1.0-0' = อย่างน้อย 1 หลักก่อนทศนิยม, ไม่มีทศนิยม
'1.2-2' = อย่างน้อย 1 หลักก่อนทศนิยม, 2 หลักทศนิยม
'4.0-0' = อย่างน้อย 4 หลักก่อนทศนิยม (pad ด้วย 0)
'1.0-3' = อย่างน้อย 1 หลักก่อนทศนิยม, 0-3 หลักทศนิยม
```

---

## 4. Built-in Pipes: uppercase, lowercase, titlecase {#case-pipes}

```typescript
// case-pipes.component.ts
import { Component } from '@angular/core';
import { UpperCasePipe, LowerCasePipe, TitleCasePipe } from '@angular/common';

@Component({
  selector: 'app-case-pipes',
  standalone: true,
  imports: [UpperCasePipe, LowerCasePipe, TitleCasePipe],
  template: `
    <div>
      <!-- UpperCase: แปลงทุกตัวอักษรเป็นพิมพ์ใหญ่ -->
      <p>ต้นฉบับ: "{{ text }}"</p>
      <p>UpperCase: "{{ text | uppercase }}"</p>
      <!-- HELLO ANGULAR WORLD -->
      
      <!-- LowerCase: แปลงทุกตัวอักษรเป็นพิมพ์เล็ก -->
      <p>LowerCase: "{{ text | lowercase }}"</p>
      <!-- hello angular world -->
      
      <!-- TitleCase: แปลงตัวแรกของแต่ละคำเป็นพิมพ์ใหญ่ -->
      <p>TitleCase: "{{ text | titlecase }}"</p>
      <!-- Hello Angular World -->
      
      <!-- ตัวอย่างการใช้งานจริง -->
      <div class="examples">
        <!-- Header ที่ต้องการ uppercase -->
        <h2>{{ pageTitle | uppercase }}</h2>
        
        <!-- Tag/Badge ที่ต้องการ uppercase -->
        <span class="badge">{{ status | uppercase }}</span>
        
        <!-- ชื่อที่อาจพิมพ์ผิด case -->
        <p>{{ userInput | titlecase }}</p>
        
        <!-- URL ที่ต้องการ lowercase -->
        <a [href]="'https://example.com/' + (urlSlug | lowercase)">
          {{ urlSlug | lowercase }}
        </a>
        
        <!-- Email validation -->
        <p>Email: {{ email | lowercase }}</p>
        
        <!-- Product name display -->
        <p>{{ productName | titlecase }}</p>
      </div>
      
      <!-- ใช้ใน *ngFor -->
      <ul>
        <li *ngFor="let name of names">
          {{ name | titlecase }}
        </li>
      </ul>
    </div>
  `
})
export class CasePipesComponent {
  text = 'hello angular World';
  pageTitle = 'product management';
  status = 'active';
  userInput = 'john DOE smith';
  urlSlug = 'My-Article-Title';
  email = 'USER@EXAMPLE.COM';
  productName = 'macbook pro m3 max';
  names = ['alice JONES', 'BOB smith', 'charlie BROWN'];
}
```

---

## 5. Built-in Pipes: json, keyvalue, async {#utility-pipes}

### JsonPipe

```typescript
// json-pipe.component.ts
import { Component } from '@angular/core';
import { JsonPipe } from '@angular/common';

@Component({
  selector: 'app-json-pipe',
  standalone: true,
  imports: [JsonPipe],
  template: `
    <div>
      <!-- json pipe: แปลง object เป็น JSON string (สำหรับ debug) -->
      
      <!-- แบบง่าย -->
      <pre>{{ simpleObj | json }}</pre>
      
      <!-- Object ซับซ้อน -->
      <pre>{{ complexObj | json }}</pre>
      
      <!-- Array -->
      <pre>{{ dataArray | json }}</pre>
      
      <!-- ใช้ debug ขณะพัฒนา -->
      <details>
        <summary>Debug: Form Data</summary>
        <pre>{{ formData | json }}</pre>
      </details>
      
      <!-- ใช้กับ API response -->
      <div class="api-response">
        <h4>API Response:</h4>
        <pre class="code">{{ apiResponse | json }}</pre>
      </div>
    </div>
  `,
  styles: [`
    pre { background: #f4f4f4; padding: 12px; border-radius: 6px; overflow: auto; }
    .code { background: #1e293b; color: #e2e8f0; }
  `]
})
export class JsonPipeComponent {
  simpleObj = { name: 'Angular', version: 17 };

  complexObj = {
    user: {
      id: 1,
      name: 'สมชาย ใจดี',
      roles: ['admin', 'editor'],
      settings: {
        theme: 'dark',
        language: 'th',
        notifications: true
      }
    },
    timestamp: new Date().toISOString()
  };

  dataArray = [
    { id: 1, name: 'Item 1', active: true },
    { id: 2, name: 'Item 2', active: false },
    { id: 3, name: 'Item 3', active: true }
  ];

  formData = {
    username: 'john_doe',
    email: 'john@example.com',
    isAdmin: false,
    createdAt: new Date().toISOString()
  };

  apiResponse = {
    status: 'success',
    data: {
      products: [
        { id: 1, name: 'Product A', price: 100 }
      ],
      pagination: {
        page: 1,
        total: 50,
        perPage: 10
      }
    }
  };
}
```

### KeyValuePipe

```typescript
// keyvalue-pipe.component.ts
import { Component } from '@angular/core';
import { KeyValuePipe, NgFor } from '@angular/common';

@Component({
  selector: 'app-keyvalue-pipe',
  standalone: true,
  imports: [KeyValuePipe, NgFor],
  template: `
    <div>
      <!-- keyvalue pipe: แปลง object/Map เป็น [{key, value}] array -->
      
      <!-- Object ธรรมดา -->
      <table>
        <tr *ngFor="let item of productDetails | keyvalue">
          <td><strong>{{ item.key }}</strong></td>
          <td>{{ item.value }}</td>
        </tr>
      </table>
      
      <!-- keyvalue กับ sorting -->
      <div *ngFor="let entry of userProfile | keyvalue:compareByKey">
        {{ entry.key }}: {{ entry.value }}
      </div>
      
      <!-- keyvalue กับ Map object -->
      <div *ngFor="let entry of configMap | keyvalue">
        <span>{{ entry.key }}:</span>
        <span>{{ entry.value }}</span>
      </div>
      
      <!-- Nested keyvalue -->
      <div *ngFor="let section of appConfig | keyvalue">
        <h4>{{ section.key }}</h4>
        <div *ngFor="let item of toObject(section.value) | keyvalue">
          &nbsp;&nbsp;{{ item.key }}: {{ item.value }}
        </div>
      </div>
      
      <!-- keyvalue กับ Record type -->
      <div *ngFor="let item of statusMap | keyvalue">
        <span [class]="'badge badge-' + item.value">
          {{ item.key }}: {{ item.value }}
        </span>
      </div>
    </div>
  `
})
export class KeyValuePipeComponent {
  productDetails = {
    brand: 'Apple',
    model: 'iPhone 15 Pro',
    color: 'Black Titanium',
    storage: '256GB',
    ram: '8GB',
    battery: '3274 mAh',
    display: '6.1 นิ้ว OLED',
    camera: '48MP Main'
  };

  userProfile: Record<string, string> = {
    firstName: 'สมชาย',
    lastName: 'ใจดี',
    email: 'somchai@example.com',
    phone: '0812345678',
    role: 'Admin'
  };

  configMap = new Map([
    ['apiUrl', 'https://api.example.com'],
    ['timeout', '30000'],
    ['retries', '3'],
    ['debug', 'false']
  ]);

  appConfig = {
    database: {
      host: 'localhost',
      port: '5432',
      name: 'myapp'
    },
    server: {
      port: '3000',
      host: '0.0.0.0'
    }
  };

  statusMap: Record<string, string> = {
    order_1: 'pending',
    order_2: 'processing',
    order_3: 'delivered'
  };

  compareByKey(a: { key: string }, b: { key: string }): number {
    return a.key.localeCompare(b.key);
  }

  toObject(value: any): Record<string, any> {
    return value as Record<string, any>;
  }
}
```

### AsyncPipe

```typescript
// async-pipe.component.ts
import { Component, OnInit } from '@angular/core';
import { AsyncPipe, NgIf, NgFor, JsonPipe } from '@angular/common';
import { Observable, of, from, timer, interval, Subject } from 'rxjs';
import { map, take, shareReplay, startWith, catchError } from 'rxjs/operators';
import { HttpClient } from '@angular/common/http';

interface User {
  id: number;
  name: string;
  email: string;
}

@Component({
  selector: 'app-async-pipe',
  standalone: true,
  imports: [AsyncPipe, NgIf, NgFor, JsonPipe],
  template: `
    <div>
      <!-- async pipe กับ Observable -->
      <p>Observable value: {{ value$ | async }}</p>
      
      <!-- async pipe กับ Promise -->
      <p>Promise value: {{ promiseValue$ | async }}</p>
      
      <!-- async กับ null check -->
      <p *ngIf="nullableData$ | async as data">
        {{ data }}
      </p>
      
      <!-- async กับ loading state ด้วย ngIf -->
      <ng-container *ngIf="users$ | async as users; else loadingTpl">
        <ul>
          <li *ngFor="let user of users">
            {{ user.name }} - {{ user.email }}
          </li>
        </ul>
        <p>จำนวนผู้ใช้: {{ users.length }}</p>
      </ng-container>
      <ng-template #loadingTpl>
        <p>กำลังโหลดผู้ใช้...</p>
      </ng-template>
      
      <!-- async กับ new @if syntax -->
      @if (users$ | async; as users) {
        <ul>
          @for (user of users; track user.id) {
            <li>{{ user.name }}</li>
          }
        </ul>
      } @else {
        <p>กำลังโหลด...</p>
      }
      
      <!-- Timer ด้วย async -->
      <p>นับวินาที: {{ timer$ | async }}</p>
      
      <!-- Countdown -->
      <p>Countdown: {{ countdown$ | async }}</p>
      
      <!-- async กับ error handling -->
      <ng-container *ngIf="dataWithError$ | async as data">
        {{ data }}
      </ng-container>
      
      <!-- Multiple async subscriptions (ไม่แนะนำ - subscribe หลายครั้ง) -->
      <!-- ควรใช้ shareReplay() ร่วมกับ async แทน -->
      <p>Shared data 1: {{ sharedData$ | async }}</p>
      <p>Shared data 2: {{ sharedData$ | async }}</p>
      <!-- เนื่องจาก sharedData$ ใช้ shareReplay(1) จะ subscribe เพียงครั้งเดียว -->
      
      <!-- async กับ Subject -->
      <p>Subject value: {{ subject$ | async }}</p>
      <button (click)="emitValue()">Emit Value</button>
      
      <!-- Real-world: Search results -->
      <div>
        <input (keyup)="onSearch($event)" placeholder="ค้นหา...">
        <ul *ngIf="searchResults$ | async as results">
          <li *ngFor="let item of results">{{ item }}</li>
        </ul>
      </div>
    </div>
  `
})
export class AsyncPipeComponent implements OnInit {
  // Observable ธรรมดา
  value$ = of('Hello Async!');

  // Promise
  promiseValue$ = Promise.resolve('Hello Promise!');

  // Nullable
  nullableData$ = of<string | null>('มีข้อมูล');

  // Users Observable (จำลอง HTTP call)
  users$!: Observable<User[]>;

  // Timer
  timer$ = interval(1000).pipe(take(60));

  // Countdown
  countdown$ = timer(10000, 1000).pipe(
    map(tick => 10 - tick),
    take(11)
  );

  // Error handling
  dataWithError$ = of('data').pipe(
    map(data => {
      if (Math.random() > 0.5) throw new Error('Random error');
      return data;
    }),
    catchError(err => of(`Error: ${err.message}`))
  );

  // Shared Observable (ป้องกัน multiple subscriptions)
  sharedData$ = of(Math.random()).pipe(
    shareReplay(1)
  );

  // Subject
  private subjectSource = new Subject<string>();
  subject$ = this.subjectSource.asObservable().pipe(
    startWith('รอรับค่า...')
  );

  // Search
  searchResults$!: Observable<string[]>;
  private searchTerm = new Subject<string>();

  private counter = 0;

  ngOnInit(): void {
    // จำลอง HTTP request
    this.users$ = of([
      { id: 1, name: 'สมชาย ใจดี', email: 'somchai@example.com' },
      { id: 2, name: 'สมหญิง รักดี', email: 'somying@example.com' }
    ]);

    this.searchResults$ = this.searchTerm.pipe(
      map(term => ['Result A', 'Result B', 'Result C'].filter(
        r => r.toLowerCase().includes(term.toLowerCase())
      ))
    );
  }

  emitValue(): void {
    this.counter++;
    this.subjectSource.next(`Value ${this.counter} at ${new Date().toLocaleTimeString()}`);
  }

  onSearch(event: Event): void {
    const input = event.target as HTMLInputElement;
    this.searchTerm.next(input.value);
  }
}
```

---

## 6. Built-in Pipes: slice {#slice-pipe}

```typescript
// slice-pipe.component.ts
import { Component } from '@angular/core';
import { SlicePipe, NgFor } from '@angular/common';

@Component({
  selector: 'app-slice-pipe',
  standalone: true,
  imports: [SlicePipe, NgFor],
  template: `
    <div>
      <!-- slice pipe สำหรับ string -->
      <p>ต้นฉบับ: "{{ longText }}"</p>
      <p>slice:0:50: "{{ longText | slice:0:50 }}..."</p>
      <p>slice:10: "{{ longText | slice:10 }}"</p>
      <p>slice:-20: "{{ longText | slice:-20 }}"</p>
      
      <!-- slice pipe สำหรับ Array -->
      <h4>Array ต้นฉบับ ({{ items.length }} items):</h4>
      <p>{{ items | json }}</p>
      
      <h4>slice:0:3:</h4>
      <ul>
        <li *ngFor="let item of items | slice:0:3">{{ item }}</li>
      </ul>
      
      <h4>slice:2:5:</h4>
      <ul>
        <li *ngFor="let item of items | slice:2:5">{{ item }}</li>
      </ul>
      
      <h4>slice:-3 (3 ตัวท้าย):</h4>
      <ul>
        <li *ngFor="let item of items | slice:-3">{{ item }}</li>
      </ul>
      
      <!-- Pagination ด้วย slice -->
      <div class="pagination-demo">
        <h4>Pagination Demo</h4>
        <p>หน้า {{ currentPage }}/{{ totalPages }}</p>
        
        <ul>
          <li *ngFor="let item of products | slice:pageStart:pageEnd">
            {{ item.name }} - {{ item.price | number:'1.0-0' }} บาท
          </li>
        </ul>
        
        <div class="pagination-controls">
          <button [disabled]="currentPage === 1" (click)="prevPage()">← ก่อนหน้า</button>
          <span>{{ currentPage }}</span>
          <button [disabled]="currentPage === totalPages" (click)="nextPage()">ถัดไป →</button>
        </div>
      </div>
      
      <!-- "See More" / "Read More" ด้วย slice -->
      <div class="read-more">
        <p>{{ description | slice:0:(isExpanded ? description.length : 100) }}</p>
        <span *ngIf="description.length > 100">
          <button (click)="isExpanded = !isExpanded">
            {{ isExpanded ? 'ย่อ' : '...อ่านต่อ' }}
          </button>
        </span>
      </div>
      
      <!-- Tag Cloud (แสดงเฉพาะบางส่วน) -->
      <div class="tags">
        <span *ngFor="let tag of tags | slice:0:(showAllTags ? tags.length : 5)" class="tag">
          {{ tag }}
        </span>
        <button *ngIf="tags.length > 5" (click)="showAllTags = !showAllTags">
          {{ showAllTags ? 'แสดงน้อยลง' : '+' + (tags.length - 5) + ' เพิ่มเติม' }}
        </button>
      </div>
    </div>
  `,
  styles: [`
    .tag { padding: 4px 10px; background: #e3f2fd; border-radius: 12px; margin: 2px; display: inline-block; font-size: 13px; }
    .pagination-controls { display: flex; gap: 12px; align-items: center; margin-top: 12px; }
    .pagination-controls button { padding: 6px 12px; border: 1px solid #ddd; background: white; cursor: pointer; border-radius: 6px; }
    .pagination-controls button:disabled { opacity: 0.5; cursor: not-allowed; }
  `]
})
export class SlicePipeComponent {
  longText = 'Angular เป็น framework สำหรับสร้าง web application ที่ทรงพลัง พัฒนาโดย Google และ community ขนาดใหญ่';
  items = ['Angular', 'React', 'Vue', 'Svelte', 'Next.js', 'Nuxt.js', 'Remix', 'Astro'];
  isExpanded = false;
  showAllTags = false;

  description = 'Angular คือ platform และ framework สำหรับสร้าง single-page client applications โดยใช้ HTML และ TypeScript Angular ถูกพัฒนาโดย Google และ community ขนาดใหญ่ โดยมีเป้าหมายหลักคือการทำให้การพัฒนา web application เป็นเรื่องที่ง่ายและมีประสิทธิภาพ';

  tags = ['Angular', 'TypeScript', 'JavaScript', 'HTML', 'CSS', 'RxJS', 'NgRx', 'Material', 'Testing', 'PWA', 'SSR'];

  currentPage = 1;
  itemsPerPage = 3;

  products = [
    { name: 'iPhone 15', price: 35000 },
    { name: 'Samsung S24', price: 28000 },
    { name: 'MacBook Pro', price: 75000 },
    { name: 'Dell XPS', price: 55000 },
    { name: 'iPad Pro', price: 38000 },
    { name: 'AirPods Pro', price: 9000 },
    { name: 'Galaxy Watch', price: 12000 },
    { name: 'Apple Watch', price: 14000 }
  ];

  get pageStart(): number {
    return (this.currentPage - 1) * this.itemsPerPage;
  }

  get pageEnd(): number {
    return this.pageStart + this.itemsPerPage;
  }

  get totalPages(): number {
    return Math.ceil(this.products.length / this.itemsPerPage);
  }

  prevPage(): void {
    if (this.currentPage > 1) this.currentPage--;
  }

  nextPage(): void {
    if (this.currentPage < this.totalPages) this.currentPage++;
  }
}
```

---

## 7. การสร้าง Custom Pipe {#custom-pipe}

### Custom Pipe พื้นฐาน

```typescript
// truncate.pipe.ts
import { Pipe, PipeTransform } from '@angular/core';

@Pipe({
  name: 'truncate',
  standalone: true
})
export class TruncatePipe implements PipeTransform {
  transform(value: string, limit = 100, ellipsis = '...'): string {
    if (!value) return '';
    if (value.length <= limit) return value;
    return value.substring(0, limit).trim() + ellipsis;
  }
}

// การใช้งาน:
// {{ longText | truncate }}
// {{ longText | truncate:50 }}
// {{ longText | truncate:50:'...' }}
// {{ longText | truncate:50:' [อ่านต่อ]' }}
```

```typescript
// thai-date.pipe.ts - แปลงวันที่เป็นภาษาไทย
import { Pipe, PipeTransform } from '@angular/core';

@Pipe({
  name: 'thaiDate',
  standalone: true
})
export class ThaiDatePipe implements PipeTransform {
  private thaiMonths = [
    '', 'มกราคม', 'กุมภาพันธ์', 'มีนาคม', 'เมษายน', 'พฤษภาคม', 'มิถุนายน',
    'กรกฎาคม', 'สิงหาคม', 'กันยายน', 'ตุลาคม', 'พฤศจิกายน', 'ธันวาคม'
  ];

  private thaiMonthsShort = [
    '', 'ม.ค.', 'ก.พ.', 'มี.ค.', 'เม.ย.', 'พ.ค.', 'มิ.ย.',
    'ก.ค.', 'ส.ค.', 'ก.ย.', 'ต.ค.', 'พ.ย.', 'ธ.ค.'
  ];

  private thaiDays = ['อาทิตย์', 'จันทร์', 'อังคาร', 'พุธ', 'พฤหัสบดี', 'ศุกร์', 'เสาร์'];

  transform(value: Date | string | null, format = 'long', buddhistYear = false): string {
    if (!value) return '-';

    const date = new Date(value);
    if (isNaN(date.getTime())) return 'วันที่ไม่ถูกต้อง';

    const day = date.getDate();
    const month = date.getMonth() + 1;
    const year = buddhistYear ? date.getFullYear() + 543 : date.getFullYear();
    const dayName = this.thaiDays[date.getDay()];

    switch (format) {
      case 'short':
        return `${day} ${this.thaiMonthsShort[month]} ${year}`;
      case 'long':
        return `${day} ${this.thaiMonths[month]} ${year}`;
      case 'full':
        return `วัน${dayName}ที่ ${day} ${this.thaiMonths[month]} พ.ศ. ${year + 543}`;
      case 'buddhist':
        return `${day} ${this.thaiMonths[month]} ${year + 543}`;
      case 'day-month':
        return `${day} ${this.thaiMonths[month]}`;
      case 'month-year':
        return `${this.thaiMonths[month]} ${year}`;
      case 'relative':
        return this.getRelativeTime(date);
      default:
        return `${day}/${month}/${year}`;
    }
  }

  private getRelativeTime(date: Date): string {
    const now = new Date();
    const diffMs = now.getTime() - date.getTime();
    const diffSecs = Math.floor(diffMs / 1000);
    const diffMins = Math.floor(diffSecs / 60);
    const diffHours = Math.floor(diffMins / 60);
    const diffDays = Math.floor(diffHours / 24);
    const diffWeeks = Math.floor(diffDays / 7);
    const diffMonths = Math.floor(diffDays / 30);
    const diffYears = Math.floor(diffDays / 365);

    if (diffSecs < 60) return 'เมื่อสักครู่';
    if (diffMins < 60) return `${diffMins} นาทีที่แล้ว`;
    if (diffHours < 24) return `${diffHours} ชั่วโมงที่แล้ว`;
    if (diffDays < 7) return `${diffDays} วันที่แล้ว`;
    if (diffWeeks < 4) return `${diffWeeks} สัปดาห์ที่แล้ว`;
    if (diffMonths < 12) return `${diffMonths} เดือนที่แล้ว`;
    return `${diffYears} ปีที่แล้ว`;
  }
}

// การใช้งาน:
// {{ date | thaiDate }}                  → 25 ธันวาคม 2024
// {{ date | thaiDate:'short' }}          → 25 ธ.ค. 2024
// {{ date | thaiDate:'full' }}           → วันพุธที่ 25 ธันวาคม พ.ศ. 2567
// {{ date | thaiDate:'buddhist' }}       → 25 ธันวาคม 2567
// {{ date | thaiDate:'relative' }}       → 3 ชั่วโมงที่แล้ว
```

```typescript
// number-thai.pipe.ts - แปลงตัวเลขเป็นภาษาไทย
import { Pipe, PipeTransform } from '@angular/core';

@Pipe({
  name: 'numberThai',
  standalone: true
})
export class NumberThaiPipe implements PipeTransform {
  private units = ['', 'หนึ่ง', 'สอง', 'สาม', 'สี่', 'ห้า', 'หก', 'เจ็ด', 'แปด', 'เก้า'];
  private places = ['', 'สิบ', 'ร้อย', 'พัน', 'หมื่น', 'แสน', 'ล้าน'];

  transform(value: number | null, type: 'text' | 'currency' = 'currency'): string {
    if (value === null || value === undefined) return '-';
    if (value === 0) return type === 'currency' ? 'ศูนย์บาทถ้วน' : 'ศูนย์';

    const isNegative = value < 0;
    const absValue = Math.abs(value);

    if (type === 'currency') {
      const baht = Math.floor(absValue);
      const satang = Math.round((absValue - baht) * 100);
      let result = this.numberToWords(baht) + 'บาท';
      if (satang > 0) {
        result += this.numberToWords(satang) + 'สตางค์';
      } else {
        result += 'ถ้วน';
      }
      return (isNegative ? 'ลบ' : '') + result;
    }

    return (isNegative ? 'ลบ' : '') + this.numberToWords(absValue);
  }

  private numberToWords(num: number): string {
    if (num === 0) return 'ศูนย์';
    if (num < 0) return 'ลบ' + this.numberToWords(-num);

    const millions = Math.floor(num / 1000000);
    const remainder = num % 1000000;

    if (millions > 0) {
      return this.numberToWords(millions) + 'ล้าน' +
        (remainder > 0 ? this.numberToWords(remainder) : '');
    }

    let result = '';
    const digits = num.toString().padStart(6, '0').split('').map(Number);
    const placeValues = ['แสน', 'หมื่น', 'พัน', 'ร้อย', 'สิบ', ''];

    for (let i = 0; i < 6; i++) {
      if (digits[i] === 0) continue;
      if (i === 4) {
        // หลักสิบ
        if (digits[i] === 1) {
          result += 'สิบ';
        } else if (digits[i] === 2) {
          result += 'ยี่สิบ';
        } else {
          result += this.units[digits[i]] + 'สิบ';
        }
      } else if (i === 5 && digits[4] !== 0 && digits[i] === 1) {
        // หลักหน่วย เมื่อมีหลักสิบแล้ว ให้ใช้ 'เอ็ด' แทน 'หนึ่ง'
        result += 'เอ็ด';
      } else {
        result += this.units[digits[i]] + placeValues[i];
      }
    }

    return result;
  }
}

// การใช้งาน:
// {{ 1234 | numberThai:'text' }}        → หนึ่งพันสองร้อยสามสิบสี่
// {{ 1234.50 | numberThai:'currency' }} → หนึ่งพันสองร้อยสามสิบสี่บาทห้าสิบสตางค์
// {{ 1000000 | numberThai:'currency' }} → หนึ่งล้านบาทถ้วน
```

```typescript
// phone-format.pipe.ts - จัดรูปแบบเบอร์โทรศัพท์
import { Pipe, PipeTransform } from '@angular/core';

@Pipe({
  name: 'phoneFormat',
  standalone: true
})
export class PhoneFormatPipe implements PipeTransform {
  transform(value: string | null, format: 'display' | 'link' = 'display'): string {
    if (!value) return '';

    // ลบตัวอักษรที่ไม่ใช่ตัวเลข
    const cleaned = value.replace(/\D/g, '');

    if (format === 'link') {
      return `tel:${cleaned}`;
    }

    // จัดรูปแบบ: 0812345678 → 081-234-5678
    if (cleaned.length === 10) {
      return `${cleaned.slice(0, 3)}-${cleaned.slice(3, 6)}-${cleaned.slice(6)}`;
    }

    // จัดรูปแบบ: 021234567 → 02-123-4567
    if (cleaned.length === 9) {
      return `${cleaned.slice(0, 2)}-${cleaned.slice(2, 5)}-${cleaned.slice(5)}`;
    }

    return value; // คืนค่าเดิมถ้าไม่รู้จักรูปแบบ
  }
}

// การใช้งาน:
// {{ '0812345678' | phoneFormat }}           → 081-234-5678
// {{ '0812345678' | phoneFormat:'link' }}    → tel:0812345678
```

```typescript
// file-size.pipe.ts - แปลงขนาดไฟล์
import { Pipe, PipeTransform } from '@angular/core';

@Pipe({
  name: 'fileSize',
  standalone: true
})
export class FileSizePipe implements PipeTransform {
  transform(bytes: number | null, precision = 2): string {
    if (bytes === null || bytes === undefined) return '-';
    if (bytes === 0) return '0 Bytes';

    const units = ['Bytes', 'KB', 'MB', 'GB', 'TB', 'PB'];
    const index = Math.floor(Math.log(bytes) / Math.log(1024));

    if (index === 0) return `${bytes} Bytes`;

    const value = bytes / Math.pow(1024, index);
    return `${value.toFixed(precision)} ${units[index]}`;
  }
}

// การใช้งาน:
// {{ 1024 | fileSize }}            → 1.00 KB
// {{ 1048576 | fileSize }}         → 1.00 MB
// {{ 1073741824 | fileSize }}      → 1.00 GB
// {{ 1536 | fileSize:0 }}          → 2 KB
// {{ 2500000 | fileSize:1 }}       → 2.4 MB
```

---

## 8. Pure vs Impure Pipe {#pure-vs-impure}

### Pure Pipe (default)

```typescript
// pure-pipe.component.ts
import { Pipe, PipeTransform } from '@angular/core';

// Pure Pipe: Angular เรียก transform() เฉพาะเมื่อ input เปลี่ยนเป็น primitive value ใหม่
// หรือ object/array reference ใหม่
@Pipe({
  name: 'sortBy',
  standalone: true,
  pure: true // default
})
export class SortByPipe implements PipeTransform {
  transform(array: any[], field: string, order: 'asc' | 'desc' = 'asc'): any[] {
    if (!Array.isArray(array)) return array;

    console.log('SortByPipe: transform called'); // เรียกแค่ครั้งเดียวถ้า input ไม่เปลี่ยน

    return [...array].sort((a, b) => {
      const aVal = a[field];
      const bVal = b[field];
      const comparison = String(aVal).localeCompare(String(bVal));
      return order === 'asc' ? comparison : -comparison;
    });
  }
}

// ข้อควรระวัง Pure Pipe:
// ถ้า push เข้า array โดยไม่สร้าง array ใหม่ Pipe จะไม่ถูกเรียกอีก!
// ต้องทำ: this.items = [...this.items, newItem]; // สร้าง reference ใหม่
// ไม่ใช่: this.items.push(newItem); // ไม่ trigger pipe
```

### Impure Pipe

```typescript
// impure-pipe.ts
import { Pipe, PipeTransform } from '@angular/core';

// Impure Pipe: Angular เรียก transform() ทุก change detection cycle
// ใช้เมื่อ input เป็น object/array ที่ mutate ได้
@Pipe({
  name: 'filterBy',
  standalone: true,
  pure: false // IMPURE!
})
export class FilterByPipe implements PipeTransform {
  transform(array: any[], property: string, value: any): any[] {
    if (!Array.isArray(array)) return array;
    if (!value) return array;

    // เรียกทุก change detection cycle - ระวังเรื่อง performance!
    return array.filter(item => {
      const itemValue = item[property];
      if (typeof itemValue === 'string') {
        return itemValue.toLowerCase().includes(String(value).toLowerCase());
      }
      return itemValue === value;
    });
  }
}

// ข้อควรระวัง Impure Pipe:
// เรียกบ่อยมาก (ทุก change detection) → อาจส่งผลต่อ performance
// แนะนำให้ใช้ pure pipe ร่วมกับการ clone array แทน

// ตัวอย่างการใช้งาน:
// <li *ngFor="let item of items | filterBy:'name':searchTerm">
```

### เปรียบเทียบ Pure vs Impure

```typescript
// pure-vs-impure.component.ts
import { Component } from '@angular/core';
import { NgFor } from '@angular/common';
import { FormsModule } from '@angular/forms';

@Component({
  selector: 'app-pure-vs-impure',
  standalone: true,
  imports: [NgFor, FormsModule, SortByPipe, FilterByPipe],
  template: `
    <div>
      <h3>Pure Pipe (SortBy)</h3>
      <!-- จะ re-sort เมื่อ items reference เปลี่ยน หรือ sortField/sortOrder เปลี่ยน -->
      <button (click)="addItem()">เพิ่มรายการ (ต้องสร้าง array ใหม่)</button>
      <ul>
        <li *ngFor="let item of items | sortBy:sortField:sortOrder">
          {{ item.name }} ({{ item.price | number:'1.0-0' }})
        </li>
      </ul>
      
      <h3>Impure Pipe (FilterBy)</h3>
      <!-- เรียกทุก change detection ไม่ว่า array จะเปลี่ยนหรือไม่ -->
      <input [(ngModel)]="searchTerm" placeholder="ค้นหา...">
      <ul>
        <li *ngFor="let item of items | filterBy:'name':searchTerm">
          {{ item.name }}
        </li>
      </ul>
    </div>
  `
})
export class PureVsImpureComponent {
  sortField = 'name';
  sortOrder: 'asc' | 'desc' = 'asc';
  searchTerm = '';

  items = [
    { name: 'Angular', price: 0 },
    { name: 'React', price: 0 },
    { name: 'Vue', price: 0 }
  ];

  addItem(): void {
    // สร้าง array ใหม่เพื่อ trigger pure pipe
    this.items = [...this.items, { name: `Item ${this.items.length + 1}`, price: Math.random() * 1000 }];
  }
}
```

---

## 9. Pipe Chaining และ Parameterization {#pipe-chaining}

```typescript
// pipe-chaining.component.ts
import { Component } from '@angular/core';
import { DatePipe, CurrencyPipe, UpperCasePipe, SlicePipe, TitleCasePipe } from '@angular/common';

@Component({
  selector: 'app-pipe-chaining',
  standalone: true,
  imports: [DatePipe, CurrencyPipe, UpperCasePipe, SlicePipe, TitleCasePipe],
  template: `
    <div>
      <!-- Pipe Chaining: ใช้หลาย pipe ต่อกัน -->
      
      <!-- ตัวอักษรพิมพ์ใหญ่ + ตัด string -->
      <p>{{ longTitle | uppercase | slice:0:30 }}</p>
      
      <!-- titlecase + slice -->
      <p>{{ productName | titlecase | slice:0:20 }}</p>
      
      <!-- date + uppercase (ไม่ค่อยมีประโยชน์แต่ทำได้) -->
      <p>{{ today | date:'MMMM yyyy' | uppercase }}</p>
      
      <!-- Multiple parameters -->
      <!-- currency: code:display:digits:locale -->
      <p>{{ price | currency:'THB':'symbol':'1.0-0':'th-TH' }}</p>
      
      <!-- date: format:timezone:locale -->
      <p>{{ today | date:'dd MMMM yyyy':'UTC+7':'th-TH' }}</p>
      
      <!-- slice: start:end -->
      <p>{{ items | slice:0:5 | json }}</p>
      
      <!-- Custom pipe chaining -->
      <p>{{ description | truncate:100 | titlecase }}</p>
      
      <!-- Pipe กับ conditional (ternary) -->
      <p>{{ (isActive ? price : 0) | currency:'THB':'symbol':'1.0-0' }}</p>
      
      <!-- Pipe กับ method return value -->
      <p>{{ getTotal() | currency:'THB':'symbol':'1.0-0' }}</p>
      
      <!-- Pipe ใน attribute binding -->
      <img [title]="today | date:'dd/MM/yyyy'" src="image.jpg" alt="">
      
      <!-- Pipe ใน ngIf expression -->
      <p *ngIf="(date | date:'yyyy') === '2024'">
        ปี 2024
      </p>
      
      <!-- Pipe กับ interpolation ใน event binding (ไม่แนะนำ) -->
      <!-- ใช้ใน template แสดงค่า เท่านั้น -->
    </div>
  `
})
export class PipeChainingComponent {
  longTitle = 'this is a very long title that needs to be truncated';
  productName = 'macbook pro m3 max 14 inch';
  today = new Date();
  price = 35000;
  isActive = true;
  description = 'angular framework for building web applications';
  items = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10];

  getTotal(): number {
    return 35000 * 1.07;
  }
}
```

---

## 10. Workshop: Data Formatting Pipe Collection {#workshop}

Workshop สร้าง collection ของ Custom Pipes สำหรับ e-commerce

### โครงสร้างโปรเจค

```
src/app/shared/pipes/
├── index.ts                    (export ทุก pipes)
├── truncate.pipe.ts
├── thai-date.pipe.ts
├── phone-format.pipe.ts
├── file-size.pipe.ts
├── sort-by.pipe.ts
├── filter-by.pipe.ts
├── highlight.pipe.ts
├── safe-html.pipe.ts
├── time-ago.pipe.ts
├── number-abbrev.pipe.ts
└── thai-baht.pipe.ts
```

### Highlight Pipe (เน้นข้อความที่ค้นหา)

```typescript
// highlight.pipe.ts
import { Pipe, PipeTransform } from '@angular/core';
import { DomSanitizer, SafeHtml } from '@angular/platform-browser';

@Pipe({
  name: 'highlight',
  standalone: true
})
export class HighlightPipe implements PipeTransform {
  constructor(private sanitizer: DomSanitizer) {}

  transform(value: string | null, search: string): SafeHtml {
    if (!value) return '';
    if (!search) return value;

    const escapedSearch = search.replace(/[.*+?^${}()|[\]\\]/g, '\\$&');
    const regex = new RegExp(escapedSearch, 'gi');

    const highlighted = value.replace(
      regex,
      match => `<mark class="highlight">${match}</mark>`
    );

    return this.sanitizer.bypassSecurityTrustHtml(highlighted);
  }
}

// การใช้งาน:
// <p [innerHTML]="text | highlight:searchTerm"></p>
// CSS: mark.highlight { background: #fff176; padding: 2px; border-radius: 2px; }
```

### SafeHtml Pipe

```typescript
// safe-html.pipe.ts
import { Pipe, PipeTransform } from '@angular/core';
import { DomSanitizer, SafeHtml } from '@angular/platform-browser';

@Pipe({
  name: 'safeHtml',
  standalone: true
})
export class SafeHtmlPipe implements PipeTransform {
  constructor(private sanitizer: DomSanitizer) {}

  transform(value: string | null): SafeHtml {
    if (!value) return '';
    return this.sanitizer.bypassSecurityTrustHtml(value);
  }
}

// การใช้งาน (ใช้ระวัง - อย่าใช้กับ user input โดยตรง):
// <div [innerHTML]="trustedHtmlContent | safeHtml"></div>
```

### Time Ago Pipe (เวลาสัมพัทธ์)

```typescript
// time-ago.pipe.ts
import { Pipe, PipeTransform, NgZone, ChangeDetectorRef, OnDestroy } from '@angular/core';

@Pipe({
  name: 'timeAgo',
  standalone: true,
  pure: false // ต้องเป็น impure เพราะต้อง update ตามเวลา
})
export class TimeAgoPipe implements PipeTransform, OnDestroy {
  private timer: ReturnType<typeof setInterval> | null = null;

  constructor(
    private changeDetectorRef: ChangeDetectorRef,
    private ngZone: NgZone
  ) {}

  transform(value: Date | string | null): string {
    if (!value) return '';

    this.removeTimer();

    const date = new Date(value);
    const now = new Date();
    const seconds = Math.round(Math.abs((now.getTime() - date.getTime()) / 1000));
    const isFuture = now.getTime() < date.getTime();

    let timeoutSeconds = 60;

    if (seconds <= 45) {
      timeoutSeconds = 45 - seconds;
      return 'เมื่อสักครู่';
    } else if (seconds <= 90) {
      return isFuture ? 'ใน 1 นาที' : '1 นาทีที่แล้ว';
    } else if (seconds <= 2700) {
      const minutes = Math.round(seconds / 60);
      timeoutSeconds = 30;
      return isFuture ? `ใน ${minutes} นาที` : `${minutes} นาทีที่แล้ว`;
    } else if (seconds <= 5400) {
      return isFuture ? 'ใน 1 ชั่วโมง' : '1 ชั่วโมงที่แล้ว';
    } else if (seconds <= 86400) {
      const hours = Math.round(seconds / 3600);
      timeoutSeconds = 3600;
      return isFuture ? `ใน ${hours} ชั่วโมง` : `${hours} ชั่วโมงที่แล้ว`;
    } else if (seconds <= 172800) {
      return isFuture ? 'พรุ่งนี้' : 'เมื่อวาน';
    } else if (seconds <= 2592000) {
      const days = Math.round(seconds / 86400);
      timeoutSeconds = 86400;
      return isFuture ? `ใน ${days} วัน` : `${days} วันที่แล้ว`;
    } else if (seconds <= 31536000) {
      const months = Math.round(seconds / 2592000);
      timeoutSeconds = 86400;
      return isFuture ? `ใน ${months} เดือน` : `${months} เดือนที่แล้ว`;
    } else {
      const years = Math.round(seconds / 31536000);
      timeoutSeconds = 86400;
      return isFuture ? `ใน ${years} ปี` : `${years} ปีที่แล้ว`;
    }

    this.ngZone.runOutsideAngular(() => {
      if (typeof window !== 'undefined') {
        this.timer = setInterval(() => {
          this.ngZone.run(() => this.changeDetectorRef.markForCheck());
        }, timeoutSeconds * 1000);
      }
    });

    return 'เมื่อสักครู่';
  }

  private removeTimer(): void {
    if (this.timer) {
      clearInterval(this.timer);
      this.timer = null;
    }
  }

  ngOnDestroy(): void {
    this.removeTimer();
  }
}

// การใช้งาน:
// {{ post.createdAt | timeAgo }}     → 3 ชั่วโมงที่แล้ว
// {{ event.startDate | timeAgo }}    → ใน 2 วัน
```

### Number Abbreviation Pipe

```typescript
// number-abbrev.pipe.ts
import { Pipe, PipeTransform } from '@angular/core';

@Pipe({
  name: 'numberAbbrev',
  standalone: true
})
export class NumberAbbrevPipe implements PipeTransform {
  transform(value: number | null, precision = 1, locale = 'th'): string {
    if (value === null || value === undefined) return '-';

    const abs = Math.abs(value);
    const sign = value < 0 ? '-' : '';

    const thaiLabels = {
      thousand: 'พัน',
      million: 'ล้าน',
      billion: 'พันล้าน',
      trillion: 'ล้านล้าน'
    };

    const engLabels = {
      thousand: 'K',
      million: 'M',
      billion: 'B',
      trillion: 'T'
    };

    const labels = locale === 'th' ? thaiLabels : engLabels;

    if (abs >= 1_000_000_000_000) {
      return sign + (abs / 1_000_000_000_000).toFixed(precision) + labels.trillion;
    } else if (abs >= 1_000_000_000) {
      return sign + (abs / 1_000_000_000).toFixed(precision) + labels.billion;
    } else if (abs >= 1_000_000) {
      return sign + (abs / 1_000_000).toFixed(precision) + labels.million;
    } else if (abs >= 1_000) {
      return sign + (abs / 1_000).toFixed(precision) + labels.thousand;
    }

    return sign + abs.toString();
  }
}

// การใช้งาน:
// {{ 1500 | numberAbbrev }}              → 1.5พัน
// {{ 2500000 | numberAbbrev }}           → 2.5ล้าน
// {{ 1200000000 | numberAbbrev:'en' }}   → 1.2B
// {{ 45000 | numberAbbrev:0 }}           → 45พัน
```

### Thai Baht to Words Pipe

```typescript
// thai-baht.pipe.ts
import { Pipe, PipeTransform } from '@angular/core';

@Pipe({
  name: 'thaiBaht',
  standalone: true
})
export class ThaiBahtPipe implements PipeTransform {
  private readonly ones = ['', 'หนึ่ง', 'สอง', 'สาม', 'สี่', 'ห้า', 'หก', 'เจ็ด', 'แปด', 'เก้า'];
  private readonly tens = ['', 'สิบ', 'ยี่สิบ', 'สามสิบ', 'สี่สิบ', 'ห้าสิบ', 'หกสิบ', 'เจ็ดสิบ', 'แปดสิบ', 'เก้าสิบ'];

  transform(value: number | null): string {
    if (value === null || value === undefined) return '-';
    if (isNaN(value)) return 'ค่าไม่ถูกต้อง';
    if (value === 0) return 'ศูนย์บาทถ้วน';

    const isNegative = value < 0;
    const abs = Math.abs(value);
    const baht = Math.floor(abs);
    const satang = Math.round((abs - baht) * 100);

    const bahtText = this.convertToText(baht);
    const satangText = satang > 0 ? this.convertToText(satang) + 'สตางค์' : 'ถ้วน';

    return (isNegative ? 'ลบ' : '') + bahtText + 'บาท' + satangText;
  }

  private convertToText(num: number): string {
    if (num === 0) return '';

    let result = '';
    const millions = Math.floor(num / 1_000_000);
    const thousands = Math.floor((num % 1_000_000) / 1_000);
    const hundreds = Math.floor((num % 1_000) / 100);
    const ten = Math.floor((num % 100) / 10);
    const one = num % 10;

    if (millions > 0) result += this.convertToText(millions) + 'ล้าน';
    if (thousands > 0) result += this.convertToText(thousands) + 'พัน';
    if (hundreds > 0) result += this.ones[hundreds] + 'ร้อย';

    if (ten > 0) {
      if (ten === 1) {
        result += 'สิบ';
      } else {
        result += this.tens[ten];
      }
    }

    if (one > 0) {
      if (one === 1 && ten > 0) {
        result += 'เอ็ด';
      } else {
        result += this.ones[one];
      }
    }

    return result;
  }
}

// การใช้งาน:
// {{ 1234 | thaiBaht }}        → หนึ่งพันสองร้อยสามสิบสี่บาทถ้วน
// {{ 99.50 | thaiBaht }}       → เก้าสิบเก้าบาทห้าสิบสตางค์
// {{ 1000000 | thaiBaht }}     → หนึ่งล้านบาทถ้วน
```

### Pipe Showcase Component

```typescript
// pipe-showcase.component.ts
import { Component } from '@angular/core';
import { NgFor, NgIf, DatePipe, CurrencyPipe, DecimalPipe, PercentPipe, UpperCasePipe, TitleCasePipe, SlicePipe, AsyncPipe, JsonPipe } from '@angular/common';
import { FormsModule } from '@angular/forms';
import { TruncatePipe } from './pipes/truncate.pipe';
import { ThaiDatePipe } from './pipes/thai-date.pipe';
import { PhoneFormatPipe } from './pipes/phone-format.pipe';
import { FileSizePipe } from './pipes/file-size.pipe';
import { TimeAgoPipe } from './pipes/time-ago.pipe';
import { NumberAbbrevPipe } from './pipes/number-abbrev.pipe';
import { ThaiBahtPipe } from './pipes/thai-baht.pipe';
import { HighlightPipe } from './pipes/highlight.pipe';

@Component({
  selector: 'app-pipe-showcase',
  standalone: true,
  imports: [
    NgFor, NgIf, DatePipe, CurrencyPipe, DecimalPipe, PercentPipe,
    UpperCasePipe, TitleCasePipe, SlicePipe, AsyncPipe, JsonPipe, FormsModule,
    TruncatePipe, ThaiDatePipe, PhoneFormatPipe, FileSizePipe,
    TimeAgoPipe, NumberAbbrevPipe, ThaiBahtPipe, HighlightPipe
  ],
  template: `
    <div class="pipe-showcase">
      <h1>Pipe Showcase</h1>
      
      <!-- Search สำหรับ Highlight Demo -->
      <div class="demo-section">
        <h2>Highlight Pipe</h2>
        <input [(ngModel)]="searchTerm" placeholder="พิมพ์เพื่อค้นหา...">
        <div class="result">
          <p [innerHTML]="sampleText | highlight:searchTerm"></p>
        </div>
      </div>
      
      <!-- Date Pipes -->
      <div class="demo-section">
        <h2>Date Pipes</h2>
        <table>
          <tr *ngFor="let fmt of dateFormats">
            <td class="label">{{ fmt.label }}</td>
            <td class="value">{{ sampleDate | date:fmt.format }}</td>
          </tr>
        </table>
        
        <h3>Thai Date Pipe</h3>
        <table>
          <tr *ngFor="let fmt of thaiDateFormats">
            <td class="label">{{ fmt.label }}</td>
            <td class="value">{{ sampleDate | thaiDate:fmt.format }}</td>
          </tr>
        </table>
      </div>
      
      <!-- Number Pipes -->
      <div class="demo-section">
        <h2>Number Pipes</h2>
        <table>
          <tr *ngFor="let item of numberExamples">
            <td class="label">{{ item.label }}</td>
            <td class="value code">{{ item.expression }}</td>
            <td class="value result">{{ item.value }}</td>
          </tr>
        </table>
      </div>
      
      <!-- Currency Demo -->
      <div class="demo-section">
        <h2>Currency & Thai Baht</h2>
        <div class="currency-demo">
          <input
            type="number"
            [(ngModel)]="demoAmount"
            placeholder="กรอกจำนวนเงิน">
          
          <table>
            <tr>
              <td>currency (฿):</td>
              <td>{{ demoAmount | currency:'THB':'symbol':'1.0-0' }}</td>
            </tr>
            <tr>
              <td>number abbrev (ไทย):</td>
              <td>{{ demoAmount | numberAbbrev:1:'th' }}</td>
            </tr>
            <tr>
              <td>number abbrev (EN):</td>
              <td>{{ demoAmount | numberAbbrev:1:'en' }}</td>
            </tr>
            <tr>
              <td>Thai Baht (ตัวอักษร):</td>
              <td>{{ demoAmount | thaiBaht }}</td>
            </tr>
          </table>
        </div>
      </div>
      
      <!-- Text Pipes -->
      <div class="demo-section">
        <h2>Text Pipes</h2>
        <div class="text-demo">
          <textarea [(ngModel)]="demoText" rows="3"></textarea>
          <p><strong>Truncate (50):</strong> {{ demoText | truncate:50 }}</p>
          <p><strong>Uppercase:</strong> {{ demoText | uppercase }}</p>
          <p><strong>TitleCase:</strong> {{ demoText | titlecase }}</p>
          <p><strong>Slice 0-20:</strong> {{ demoText | slice:0:20 }}</p>
        </div>
      </div>
      
      <!-- Phone Format -->
      <div class="demo-section">
        <h2>Phone Format Pipe</h2>
        <div *ngFor="let phone of samplePhones">
          <p>
            <strong>{{ phone }}</strong> →
            <a [href]="phone | phoneFormat:'link'">{{ phone | phoneFormat }}</a>
          </p>
        </div>
      </div>
      
      <!-- File Size -->
      <div class="demo-section">
        <h2>File Size Pipe</h2>
        <table>
          <tr *ngFor="let file of sampleFiles">
            <td>{{ file.name }}</td>
            <td>{{ file.size | fileSize }}</td>
            <td>{{ file.size | fileSize:0 }}</td>
          </tr>
        </table>
      </div>
      
      <!-- Time Ago -->
      <div class="demo-section">
        <h2>Time Ago Pipe</h2>
        <table>
          <tr *ngFor="let event of timeEvents">
            <td>{{ event.label }}</td>
            <td>{{ event.date | timeAgo }}</td>
            <td>{{ event.date | date:'dd/MM/yyyy HH:mm' }}</td>
          </tr>
        </table>
      </div>
      
      <!-- JSON Debug -->
      <div class="demo-section">
        <h2>JSON Pipe (Debug)</h2>
        <details>
          <summary>คลิกเพื่อดู raw data</summary>
          <pre>{{ debugData | json }}</pre>
        </details>
      </div>
    </div>
  `,
  styles: [`
    .pipe-showcase { max-width: 900px; margin: 0 auto; padding: 24px; font-family: sans-serif; }
    h1 { color: #1e293b; border-bottom: 3px solid #3b82f6; padding-bottom: 12px; }
    h2 { color: #334155; font-size: 18px; margin: 24px 0 12px; }
    h3 { color: #475569; font-size: 15px; margin: 16px 0 8px; }
    .demo-section {
      background: white; border-radius: 12px; padding: 20px;
      margin-bottom: 20px; box-shadow: 0 2px 8px rgba(0,0,0,0.06);
    }
    table { width: 100%; border-collapse: collapse; }
    td { padding: 8px 12px; border-bottom: 1px solid #f1f5f9; font-size: 14px; }
    td.label { color: #6b7280; width: 35%; }
    td.value { font-family: monospace; }
    td.code { color: #6b7280; font-size: 13px; }
    td.result { color: #1e293b; font-weight: 500; }
    input, textarea {
      width: 100%; padding: 8px 12px; border: 1px solid #e2e8f0;
      border-radius: 8px; font-size: 14px; margin-bottom: 12px; box-sizing: border-box;
    }
    .result { 
      padding: 12px; background: #f8fafc; border-radius: 8px; 
      border: 1px solid #e2e8f0; margin-top: 8px;
    }
    pre { background: #1e293b; color: #e2e8f0; padding: 16px; border-radius: 8px; overflow: auto; font-size: 13px; }
    mark.highlight { background: #fef08a; padding: 2px 0; border-radius: 2px; }
    
    @media (max-width: 600px) {
      table td { display: block; border-bottom: none; }
      table tr { border-bottom: 1px solid #f1f5f9; }
    }
  `]
})
export class PipeShowcaseComponent {
  searchTerm = 'Angular';
  demoAmount = 1500000;
  demoText = 'angular is a platform and framework for building single-page client applications';

  sampleText = 'Angular เป็น framework สำหรับการสร้าง web application ที่ทรงพลัง Angular มาพร้อมกับ features มากมายสำหรับการพัฒนา';

  sampleDate = new Date('2024-12-25T10:30:45');

  dateFormats = [
    { label: 'short', format: 'short' },
    { label: 'medium', format: 'medium' },
    { label: 'dd/MM/yyyy', format: 'dd/MM/yyyy' },
    { label: 'dd/MM/yyyy HH:mm', format: 'dd/MM/yyyy HH:mm' },
    { label: 'EEEE d MMMM yyyy', format: 'EEEE d MMMM yyyy' },
    { label: 'hh:mm:ss a', format: 'hh:mm:ss a' }
  ];

  thaiDateFormats = [
    { label: 'long (default)', format: 'long' },
    { label: 'short', format: 'short' },
    { label: 'full', format: 'full' },
    { label: 'buddhist', format: 'buddhist' },
    { label: 'relative', format: 'relative' }
  ];

  numberExamples = [
    { label: 'number (default)', expression: '1234567.89 | number', value: '1,234,567.89' },
    { label: 'number:1.2-2', expression: '1234567.89 | number:\'1.2-2\'', value: '1,234,567.89' },
    { label: 'number:1.0-0', expression: '1234567 | number:\'1.0-0\'', value: '1,234,567' },
    { label: 'currency THB', expression: '35000 | currency:\'THB\':\'symbol\':\'1.0-0\'', value: '฿35,000' },
    { label: 'percent', expression: '0.756 | percent:\'1.1-1\'', value: '75.6%' },
    { label: 'abbrev (ไทย)', expression: '1500000 | numberAbbrev', value: '1.5ล้าน' },
    { label: 'abbrev (EN)', expression: '1500000 | numberAbbrev:1:\'en\'', value: '1.5M' }
  ];

  samplePhones = ['0812345678', '021234567', '0891234567'];

  sampleFiles = [
    { name: 'report.pdf', size: 2457600 },
    { name: 'photo.jpg', size: 8388608 },
    { name: 'video.mp4', size: 524288000 },
    { name: 'notes.txt', size: 1024 },
    { name: 'database.sql', size: 1073741824 }
  ];

  timeEvents = [
    { label: 'เมื่อ 30 วินาที', date: new Date(Date.now() - 30000) },
    { label: 'เมื่อ 15 นาที', date: new Date(Date.now() - 15 * 60 * 1000) },
    { label: 'เมื่อ 3 ชั่วโมง', date: new Date(Date.now() - 3 * 60 * 60 * 1000) },
    { label: 'เมื่อวาน', date: new Date(Date.now() - 25 * 60 * 60 * 1000) },
    { label: 'เมื่อ 5 วัน', date: new Date(Date.now() - 5 * 24 * 60 * 60 * 1000) },
    { label: 'เมื่อ 2 เดือน', date: new Date(Date.now() - 60 * 24 * 60 * 60 * 1000) }
  ];

  debugData = {
    pipes: ['date', 'currency', 'number', 'truncate', 'thaiDate', 'highlight'],
    stats: {
      builtIn: 12,
      custom: 8,
      total: 20
    },
    lastUpdate: new Date().toISOString()
  };
}
```

### Pipe Index - Export ทุก Pipes

```typescript
// pipes/index.ts
export { TruncatePipe } from './truncate.pipe';
export { ThaiDatePipe } from './thai-date.pipe';
export { PhoneFormatPipe } from './phone-format.pipe';
export { FileSizePipe } from './file-size.pipe';
export { SortByPipe } from './sort-by.pipe';
export { FilterByPipe } from './filter-by.pipe';
export { HighlightPipe } from './highlight.pipe';
export { SafeHtmlPipe } from './safe-html.pipe';
export { TimeAgoPipe } from './time-ago.pipe';
export { NumberAbbrevPipe } from './number-abbrev.pipe';
export { ThaiBahtPipe } from './thai-baht.pipe';
```

---

## สรุป Part 06

### Built-in Pipes ทั้งหมด

| Pipe | Import จาก | ตัวอย่าง |
|------|------------|----------|
| `date` | `@angular/common` | `{{ date \| date:'dd/MM/yyyy' }}` |
| `currency` | `@angular/common` | `{{ price \| currency:'THB':'symbol' }}` |
| `number` | `@angular/common` | `{{ num \| number:'1.2-2' }}` |
| `percent` | `@angular/common` | `{{ 0.75 \| percent }}` |
| `uppercase` | `@angular/common` | `{{ text \| uppercase }}` |
| `lowercase` | `@angular/common` | `{{ text \| lowercase }}` |
| `titlecase` | `@angular/common` | `{{ text \| titlecase }}` |
| `json` | `@angular/common` | `{{ obj \| json }}` |
| `keyvalue` | `@angular/common` | `*ngFor="let e of obj \| keyvalue"` |
| `async` | `@angular/common` | `{{ obs$ \| async }}` |
| `slice` | `@angular/common` | `{{ arr \| slice:0:5 }}` |

### Checklist สร้าง Custom Pipe

- [ ] สร้างไฟล์ `name.pipe.ts`
- [ ] เพิ่ม decorator `@Pipe({ name: '...', standalone: true })`
- [ ] implement `PipeTransform` interface
- [ ] เขียน `transform()` method
- [ ] กำหนด `pure: false` ถ้าต้องการ impure pipe
- [ ] Export จาก `index.ts`
- [ ] Import ใน component ที่ใช้งาน
- [ ] เขียน unit test สำหรับ pipe

### Best Practices

1. **ใช้ Pure Pipe เมื่อทำได้** เพราะ Angular จะ cache ผลลัพธ์
2. **Impure Pipe ระวังเรื่อง performance** เรียกทุก change detection
3. **ไม่ควรทำ side effects ใน pipe** (เช่น HTTP call, DOM manipulation)
4. **ตั้งชื่อ Pipe ให้ชัดเจน** เช่น `thaiDate`, `fileSize`, `highlight`
5. **ใช้ Pipe chaining อย่างระมัดระวัง** อ่านจากซ้ายไปขวา
6. **เพิ่ม null/undefined check** ในทุก pipe เสมอ
7. **สร้าง unit tests** สำหรับทุก custom pipe
8. **ใช้ DatePipe สำหรับ Date** ไม่ใช่ `toLocaleDateString()`
9. **ใช้ CurrencyPipe กับ i18n** สำหรับแสดงเงินหลายสกุล
10. **Async Pipe จัดการ unsubscribe ให้อัตโนมัติ** ดีกว่า subscribe เอง

---

*ก่อนหน้า: [Part 05 — Built-in Directives](part-05-directives.md)*  
*ต่อไป: Part 07 — Services และ Dependency Injection (เร็วๆ นี้)*
