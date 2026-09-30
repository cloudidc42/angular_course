# Part 18: Custom Pipes

## บทนำ

Pipes ใน Angular คือ Functions ที่ใช้ใน Template เพื่อแปลงข้อมูลก่อนแสดงผล เช่น การฟอร์แมต วันที่ ตัวเลข สกุลเงิน หรือข้อความ สามารถสร้าง Custom Pipes ของตัวเองได้เพื่อใช้งานซ้ำในทั้งแอปพลิเคชัน

---

## 1. Pure vs Impure Pipe

### Pure Pipe (ค่าเริ่มต้น)

Pure Pipe จะ execute เฉพาะเมื่อ input เปลี่ยนแปลง (reference change หรือ primitive value change) Angular ตรวจสอบแค่ reference ไม่ใช่ deep comparison

```typescript
@Pipe({
  name: 'thaiCurrency',
  pure: true,     // default คือ true
})
export class ThaiCurrencyPipe implements PipeTransform {
  transform(value: number, currency = 'THB'): string {
    if (value == null) return '';
    return new Intl.NumberFormat('th-TH', {
      style: 'currency',
      currency,
    }).format(value);
  }
}
```

```
Pure Pipe — เมื่อไหร่ที่ execute:
- Input เป็น number: 1000 → 2000 (เปลี่ยน) ✓ execute
- Input เป็น string: 'foo' → 'bar' (เปลี่ยน) ✓ execute
- Input เป็น Array: push item (reference เดิม) ✗ ไม่ execute
- Input เป็น Object: เปลี่ยน property (reference เดิม) ✗ ไม่ execute
```

### Impure Pipe

Impure Pipe จะ execute ทุก Change Detection cycle ทำให้ช้ากว่าแต่รับรู้การเปลี่ยนแปลงภายใน Object/Array

```typescript
@Pipe({
  name: 'filterByCategory',
  pure: false,    // Impure: execute ทุก change detection
})
export class FilterByCategoryPipe implements PipeTransform {
  transform(products: Product[], category: string): Product[] {
    if (!products || !category) return products;
    return products.filter(p => p.category === category);
  }
}
```

```html
<!-- Impure Pipe รับรู้เมื่อ push ลง array -->
<div *ngFor="let product of products | filterByCategory:selectedCategory">
  {{ product.name }}
</div>
```

### เปรียบเทียบ Pure vs Impure

| | Pure | Impure |
|--|------|--------|
| **Execute เมื่อ** | Input reference เปลี่ยน | ทุก Change Detection |
| **Performance** | ดีกว่า | ช้ากว่า |
| **ใช้กับ** | Primitive, immutable objects | Array/Object ที่เปลี่ยน in-place |
| **แนะนำ** | ใช้เป็นหลัก | ใช้เมื่อจำเป็น |

---

## 2. สร้าง Custom Pipe

### โครงสร้างพื้นฐาน

```typescript
import { Pipe, PipeTransform } from '@angular/core';

@Pipe({
  name: 'pipeName',   // ชื่อที่ใช้ใน template: {{ value | pipeName }}
  pure: true,         // optional, default: true
})
export class PipeNamePipe implements PipeTransform {
  transform(value: any, ...args: any[]): any {
    // logic การแปลงข้อมูล
    return transformedValue;
  }
}
```

### สร้างด้วย Angular CLI

```bash
ng generate pipe shared/pipes/thai-currency
# หรือย่อ
ng g pipe shared/pipes/thai-currency
```

### Truncate Pipe — ตัดข้อความยาว

```typescript
// shared/pipes/truncate.pipe.ts
import { Pipe, PipeTransform } from '@angular/core';

@Pipe({
  name: 'truncate',
  standalone: true,
})
export class TruncatePipe implements PipeTransform {
  /**
   * ตัดข้อความที่ยาวเกินกำหนด
   * @param value ข้อความต้นฉบับ
   * @param limit จำนวนตัวอักษรสูงสุด (default: 100)
   * @param ellipsis สัญลักษณ์ที่ต่อท้าย (default: '...')
   */
  transform(
    value: string | null | undefined,
    limit = 100,
    ellipsis = '...'
  ): string {
    if (!value) return '';
    if (value.length <= limit) return value;
    return value.substring(0, limit - ellipsis.length) + ellipsis;
  }
}
```

```html
<!-- ใช้งาน -->
<p>{{ product.description | truncate }}</p>
<p>{{ product.description | truncate:50 }}</p>
<p>{{ product.description | truncate:50:'…' }}</p>
```

### SafeHtml Pipe — แสดง HTML ที่ปลอดภัย

```typescript
// shared/pipes/safe-html.pipe.ts
import { Pipe, PipeTransform } from '@angular/core';
import { DomSanitizer, SafeHtml } from '@angular/platform-browser';

@Pipe({
  name: 'safeHtml',
  standalone: true,
})
export class SafeHtmlPipe implements PipeTransform {
  constructor(private sanitizer: DomSanitizer) {}

  transform(value: string | null | undefined): SafeHtml {
    if (!value) return '';
    return this.sanitizer.bypassSecurityTrustHtml(value);
  }
}
```

```html
<!-- ใช้งาน -->
<div [innerHTML]="product.htmlDescription | safeHtml"></div>
```

### RelativeTime Pipe — แสดงเวลาแบบ relative

```typescript
// shared/pipes/relative-time.pipe.ts
import { Pipe, PipeTransform } from '@angular/core';

@Pipe({
  name: 'relativeTime',
  standalone: true,
  pure: false, // Impure เพราะเวลาเปลี่ยนตลอด
})
export class RelativeTimePipe implements PipeTransform {
  transform(value: Date | string | number | null): string {
    if (!value) return '';

    const date = new Date(value);
    const now = new Date();
    const diffMs = now.getTime() - date.getTime();

    const seconds = Math.floor(diffMs / 1000);
    const minutes = Math.floor(seconds / 60);
    const hours = Math.floor(minutes / 60);
    const days = Math.floor(hours / 24);
    const months = Math.floor(days / 30);
    const years = Math.floor(days / 365);

    if (seconds < 60) return 'เมื่อกี้';
    if (minutes < 60) return `${minutes} นาทีที่แล้ว`;
    if (hours < 24) return `${hours} ชั่วโมงที่แล้ว`;
    if (days < 30) return `${days} วันที่แล้ว`;
    if (months < 12) return `${months} เดือนที่แล้ว`;
    return `${years} ปีที่แล้ว`;
  }
}
```

```html
<!-- ใช้งาน -->
<span>{{ order.createdAt | relativeTime }}</span>
<!-- ผลลัพธ์: "3 นาทีที่แล้ว" -->
```

---

## 3. Pipe กับ Async Data

### Async Pipe

```typescript
// Async Pipe คือ built-in pipe ที่ subscribe Observable/Promise อัตโนมัติ
// และ unsubscribe เมื่อ Component ถูก destroy

@Component({
  template: `
    <!-- แบบ basic -->
    <div *ngIf="products$ | async as products">
      <app-product-card
        *ngFor="let product of products"
        [product]="product"
      ></app-product-card>
    </div>

    <!-- รับมือ loading/error -->
    <ng-container *ngIf="{ data: products$ | async, error: error$ | async } as vm">
      <div *ngIf="vm.error" class="error">{{ vm.error }}</div>
      <div *ngIf="vm.data" class="products">
        <app-product-card
          *ngFor="let product of vm.data"
          [product]="product"
        ></app-product-card>
      </div>
    </ng-container>
  `,
})
export class ProductListComponent {
  products$ = this.productService.getAll();
  error$ = new Subject<string>();

  constructor(private productService: ProductService) {}
}
```

### Custom Async-aware Pipe

```typescript
// shared/pipes/format-price.pipe.ts
import { Pipe, PipeTransform } from '@angular/core';

@Pipe({
  name: 'formatPrice',
  standalone: true,
})
export class FormatPricePipe implements PipeTransform {
  transform(
    price: number | null | undefined,
    options?: {
      currency?: string;
      showDecimal?: boolean;
      showCurrencySymbol?: boolean;
    }
  ): string {
    if (price == null) return 'ไม่ระบุราคา';

    const {
      currency = 'THB',
      showDecimal = false,
      showCurrencySymbol = true,
    } = options || {};

    const formatted = new Intl.NumberFormat('th-TH', {
      style: showCurrencySymbol ? 'currency' : 'decimal',
      currency,
      minimumFractionDigits: showDecimal ? 2 : 0,
      maximumFractionDigits: showDecimal ? 2 : 0,
    }).format(price);

    return formatted;
  }
}
```

```html
<!-- ใช้กับ Async Pipe -->
<div *ngIf="product$ | async as product">
  <span>{{ product.price | formatPrice }}</span>
  <span>{{ product.price | formatPrice:{ showDecimal: true } }}</span>
  <span>{{ product.price | formatPrice:{ currency: 'USD', showCurrencySymbol: true } }}</span>
</div>
```

---

## 4. Pipe Chaining

### การเชื่อม Pipe หลายตัว

```html
<!-- เชื่อม Pipe หลายตัวด้วย | -->
{{ value | pipe1 | pipe2 | pipe3 }}

<!-- ตัวอย่างจริง -->
{{ product.name | titlecase | truncate:50 }}
{{ product.description | truncate:200 | safeHtml }}
{{ user.createdAt | date:'dd/MM/yyyy' | thaiDate }}
{{ price | currency:'THB' | truncate:20 }}
```

### Pipe Chaining ที่ซับซ้อน

```typescript
// shared/pipes/highlight.pipe.ts
import { Pipe, PipeTransform } from '@angular/core';
import { DomSanitizer, SafeHtml } from '@angular/platform-browser';

@Pipe({
  name: 'highlight',
  standalone: true,
})
export class HighlightPipe implements PipeTransform {
  constructor(private sanitizer: DomSanitizer) {}

  transform(text: string, searchTerm: string): SafeHtml {
    if (!searchTerm || !text) return text;

    const regex = new RegExp(`(${this.escapeRegex(searchTerm)})`, 'gi');
    const highlighted = text.replace(
      regex,
      '<mark class="highlight">$1</mark>'
    );

    return this.sanitizer.bypassSecurityTrustHtml(highlighted);
  }

  private escapeRegex(text: string): string {
    return text.replace(/[.*+?^${}()|[\]\\]/g, '\\$&');
  }
}
```

```html
<!-- เชื่อม truncate + highlight -->
<p [innerHTML]="product.description | truncate:200 | highlight:searchTerm"></p>
```

---

## 5. Workshop: Thai Number, Date, Currency Pipes

### Thai Number Pipe

```typescript
// shared/pipes/thai-number.pipe.ts
import { Pipe, PipeTransform } from '@angular/core';

@Pipe({
  name: 'thaiNumber',
  standalone: true,
})
export class ThaiNumberPipe implements PipeTransform {
  private readonly THAI_DIGITS = ['๐', '๑', '๒', '๓', '๔', '๕', '๖', '๗', '๘', '๙'];

  /**
   * แปลงตัวเลขเป็นเลขไทยหรือจัดรูปแบบ
   * @param value ตัวเลข
   * @param format 'thai' = เลขไทย, 'arabic' = เลขอารบิก (default), 'short' = ย่อ (พัน ล้าน)
   * @param decimals จำนวนทศนิยม
   */
  transform(
    value: number | null | undefined,
    format: 'thai' | 'arabic' | 'short' = 'arabic',
    decimals = 0
  ): string {
    if (value == null || isNaN(value)) return '';

    switch (format) {
      case 'thai':
        return this.toThaiDigits(value, decimals);
      case 'short':
        return this.toShortForm(value);
      default:
        return this.toArabic(value, decimals);
    }
  }

  private toThaiDigits(value: number, decimals: number): string {
    const formatted = value.toFixed(decimals);
    return formatted.split('').map(char => {
      const digit = parseInt(char);
      return isNaN(digit) ? char : this.THAI_DIGITS[digit];
    }).join('');
  }

  private toArabic(value: number, decimals: number): string {
    return new Intl.NumberFormat('th-TH', {
      minimumFractionDigits: decimals,
      maximumFractionDigits: decimals,
    }).format(value);
  }

  private toShortForm(value: number): string {
    const absValue = Math.abs(value);
    const sign = value < 0 ? '-' : '';

    if (absValue >= 1_000_000_000) {
      return `${sign}${(absValue / 1_000_000_000).toFixed(1)}พันล้าน`;
    }
    if (absValue >= 1_000_000) {
      return `${sign}${(absValue / 1_000_000).toFixed(1)}ล้าน`;
    }
    if (absValue >= 1_000) {
      return `${sign}${(absValue / 1_000).toFixed(1)}พัน`;
    }
    return `${sign}${absValue}`;
  }
}
```

```html
<!-- ตัวอย่างการใช้งาน -->
<span>{{ 1234567 | thaiNumber }}</span>
<!-- ผลลัพธ์: 1,234,567 -->

<span>{{ 1234567 | thaiNumber:'thai' }}</span>
<!-- ผลลัพธ์: ๑,๒๓๔,๕๖๗ -->

<span>{{ 1234567 | thaiNumber:'short' }}</span>
<!-- ผลลัพธ์: 1.2ล้าน -->

<span>{{ 1500000 | thaiNumber:'short' }}</span>
<!-- ผลลัพธ์: 1.5ล้าน -->

<span>{{ 42000 | thaiNumber:'short' }}</span>
<!-- ผลลัพธ์: 42.0พัน -->
```

### Thai Date Pipe

```typescript
// shared/pipes/thai-date.pipe.ts
import { Pipe, PipeTransform } from '@angular/core';

type ThaiDateFormat =
  | 'short'       // 1/1/2567
  | 'medium'      // 1 ม.ค. 2567
  | 'long'        // 1 มกราคม 2567
  | 'full'        // วันจันทร์ที่ 1 มกราคม พ.ศ. 2567
  | 'time'        // 08:30 น.
  | 'datetime'    // 1 ม.ค. 2567 08:30 น.
  | 'relative';   // 3 วันที่แล้ว

@Pipe({
  name: 'thaiDate',
  standalone: true,
})
export class ThaiDatePipe implements PipeTransform {
  private readonly THAI_MONTHS_SHORT = [
    'ม.ค.', 'ก.พ.', 'มี.ค.', 'เม.ย.', 'พ.ค.', 'มิ.ย.',
    'ก.ค.', 'ส.ค.', 'ก.ย.', 'ต.ค.', 'พ.ย.', 'ธ.ค.',
  ];

  private readonly THAI_MONTHS_LONG = [
    'มกราคม', 'กุมภาพันธ์', 'มีนาคม', 'เมษายน', 'พฤษภาคม', 'มิถุนายน',
    'กรกฎาคม', 'สิงหาคม', 'กันยายน', 'ตุลาคม', 'พฤศจิกายน', 'ธันวาคม',
  ];

  private readonly THAI_DAYS = [
    'อาทิตย์', 'จันทร์', 'อังคาร', 'พุธ', 'พฤหัสบดี', 'ศุกร์', 'เสาร์',
  ];

  transform(
    value: Date | string | number | null | undefined,
    format: ThaiDateFormat = 'medium'
  ): string {
    if (!value) return '';

    const date = new Date(value);
    if (isNaN(date.getTime())) return 'วันที่ไม่ถูกต้อง';

    const buddhistYear = date.getFullYear() + 543;
    const day = date.getDate();
    const month = date.getMonth();
    const dayOfWeek = date.getDay();
    const hours = date.getHours().toString().padStart(2, '0');
    const minutes = date.getMinutes().toString().padStart(2, '0');

    switch (format) {
      case 'short':
        return `${day}/${month + 1}/${buddhistYear}`;

      case 'medium':
        return `${day} ${this.THAI_MONTHS_SHORT[month]} ${buddhistYear}`;

      case 'long':
        return `${day} ${this.THAI_MONTHS_LONG[month]} ${buddhistYear}`;

      case 'full':
        return `วัน${this.THAI_DAYS[dayOfWeek]}ที่ ${day} ${this.THAI_MONTHS_LONG[month]} พ.ศ. ${buddhistYear}`;

      case 'time':
        return `${hours}:${minutes} น.`;

      case 'datetime':
        return `${day} ${this.THAI_MONTHS_SHORT[month]} ${buddhistYear} ${hours}:${minutes} น.`;

      case 'relative':
        return this.getRelativeTime(date);

      default:
        return `${day} ${this.THAI_MONTHS_SHORT[month]} ${buddhistYear}`;
    }
  }

  private getRelativeTime(date: Date): string {
    const now = new Date();
    const diffMs = now.getTime() - date.getTime();
    const diffSeconds = Math.floor(diffMs / 1000);
    const diffMinutes = Math.floor(diffSeconds / 60);
    const diffHours = Math.floor(diffMinutes / 60);
    const diffDays = Math.floor(diffHours / 24);
    const diffMonths = Math.floor(diffDays / 30);
    const diffYears = Math.floor(diffDays / 365);

    if (diffSeconds < 60) return 'เมื่อกี้';
    if (diffMinutes < 60) return `${diffMinutes} นาทีที่แล้ว`;
    if (diffHours < 24) return `${diffHours} ชั่วโมงที่แล้ว`;
    if (diffDays < 30) return `${diffDays} วันที่แล้ว`;
    if (diffMonths < 12) return `${diffMonths} เดือนที่แล้ว`;
    return `${diffYears} ปีที่แล้ว`;
  }
}
```

```html
<!-- ตัวอย่างการใช้งาน -->
<span>{{ '2024-01-15' | thaiDate }}</span>
<!-- ผลลัพธ์: 15 ม.ค. 2567 -->

<span>{{ '2024-01-15' | thaiDate:'short' }}</span>
<!-- ผลลัพธ์: 15/1/2567 -->

<span>{{ '2024-01-15' | thaiDate:'long' }}</span>
<!-- ผลลัพธ์: 15 มกราคม 2567 -->

<span>{{ '2024-01-15T14:30:00' | thaiDate:'full' }}</span>
<!-- ผลลัพธ์: วันจันทร์ที่ 15 มกราคม พ.ศ. 2567 -->

<span>{{ '2024-01-15T14:30:00' | thaiDate:'datetime' }}</span>
<!-- ผลลัพธ์: 15 ม.ค. 2567 14:30 น. -->

<span>{{ lastUpdated | thaiDate:'relative' }}</span>
<!-- ผลลัพธ์: 3 วันที่แล้ว -->
```

### Thai Currency Pipe

```typescript
// shared/pipes/thai-currency.pipe.ts
import { Pipe, PipeTransform } from '@angular/core';

export interface ThaiCurrencyOptions {
  currency?: 'THB' | 'USD' | 'EUR' | 'JPY' | 'CNY';
  showSymbol?: boolean;    // แสดงสัญลักษณ์สกุลเงิน
  showCode?: boolean;      // แสดงรหัสสกุลเงิน (THB)
  showDecimal?: boolean;   // แสดงทศนิยม
  compact?: boolean;       // แสดงแบบย่อ (1.5M)
}

@Pipe({
  name: 'thaiCurrency',
  standalone: true,
})
export class ThaiCurrencyPipe implements PipeTransform {
  private readonly CURRENCY_SYMBOLS: Record<string, string> = {
    THB: '฿',
    USD: '$',
    EUR: '€',
    JPY: '¥',
    CNY: '¥',
  };

  transform(
    value: number | null | undefined,
    options: ThaiCurrencyOptions = {}
  ): string {
    if (value == null || isNaN(value)) return '-';

    const {
      currency = 'THB',
      showSymbol = true,
      showCode = false,
      showDecimal = false,
      compact = false,
    } = options;

    let formatted: string;

    if (compact) {
      formatted = this.formatCompact(value, currency);
    } else {
      formatted = new Intl.NumberFormat('th-TH', {
        style: showSymbol ? 'currency' : 'decimal',
        currency: showSymbol ? currency : undefined,
        minimumFractionDigits: showDecimal ? 2 : 0,
        maximumFractionDigits: showDecimal ? 2 : 0,
      }).format(value);
    }

    if (showCode && !showSymbol) {
      formatted = `${formatted} ${currency}`;
    }

    return formatted;
  }

  private formatCompact(value: number, currency: string): string {
    const symbol = this.CURRENCY_SYMBOLS[currency] || currency;
    const absValue = Math.abs(value);
    const sign = value < 0 ? '-' : '';

    if (absValue >= 1_000_000_000) {
      return `${sign}${symbol}${(absValue / 1_000_000_000).toFixed(1)}B`;
    }
    if (absValue >= 1_000_000) {
      return `${sign}${symbol}${(absValue / 1_000_000).toFixed(1)}M`;
    }
    if (absValue >= 1_000) {
      return `${sign}${symbol}${(absValue / 1_000).toFixed(1)}K`;
    }
    return `${sign}${symbol}${absValue.toFixed(0)}`;
  }
}
```

```html
<!-- ตัวอย่างการใช้งาน -->
<span>{{ 1500 | thaiCurrency }}</span>
<!-- ผลลัพธ์: ฿1,500 -->

<span>{{ 1500.50 | thaiCurrency:{ showDecimal: true } }}</span>
<!-- ผลลัพธ์: ฿1,500.50 -->

<span>{{ 1500 | thaiCurrency:{ currency: 'USD' } }}</span>
<!-- ผลลัพธ์: $1,500 -->

<span>{{ 1500000 | thaiCurrency:{ compact: true } }}</span>
<!-- ผลลัพธ์: ฿1.5M -->

<span>{{ 1500000 | thaiCurrency:{ showSymbol: false, showCode: true } }}</span>
<!-- ผลลัพธ์: 1,500,000 THB -->
```

### Discount Pipe

```typescript
// shared/pipes/discount.pipe.ts
import { Pipe, PipeTransform } from '@angular/core';

@Pipe({
  name: 'discount',
  standalone: true,
})
export class DiscountPipe implements PipeTransform {
  /**
   * คำนวณราคาหลังลด
   * @param originalPrice ราคาเต็ม
   * @param discountPercent เปอร์เซ็นต์ส่วนลด (0-100)
   * @param mode 'price' = ราคาหลังลด, 'saved' = จำนวนที่ประหยัด, 'percent' = แสดง %
   */
  transform(
    originalPrice: number | null | undefined,
    discountPercent: number = 0,
    mode: 'price' | 'saved' | 'percent' = 'price'
  ): number | string {
    if (originalPrice == null) return 0;
    if (discountPercent <= 0) return originalPrice;
    if (discountPercent >= 100) return 0;

    const discount = originalPrice * (discountPercent / 100);
    const finalPrice = originalPrice - discount;

    switch (mode) {
      case 'price':
        return Math.round(finalPrice);
      case 'saved':
        return Math.round(discount);
      case 'percent':
        return `-${discountPercent}%`;
      default:
        return Math.round(finalPrice);
    }
  }
}
```

```html
<!-- ตัวอย่างการใช้งาน -->
<div class="product-price">
  <!-- ราคาเต็ม -->
  <span class="original-price">{{ product.price | thaiCurrency }}</span>

  <!-- ป้ายส่วนลด -->
  <span class="badge-discount">{{ product.discountPercent | discount:0:'percent' }}</span>

  <!-- ราคาหลังลด -->
  <span class="final-price">
    {{ product.price | discount:product.discountPercent | thaiCurrency }}
  </span>

  <!-- จำนวนที่ประหยัด -->
  <span class="saved-text">
    ประหยัด {{ product.price | discount:product.discountPercent:'saved' | thaiCurrency }}
  </span>
</div>
```

### ลงทะเบียน Pipes ใน SharedModule

```typescript
// shared/shared.module.ts
import { NgModule } from '@angular/core';
import { CommonModule } from '@angular/common';

// Pipes
import { ThaiNumberPipe } from './pipes/thai-number.pipe';
import { ThaiDatePipe } from './pipes/thai-date.pipe';
import { ThaiCurrencyPipe } from './pipes/thai-currency.pipe';
import { DiscountPipe } from './pipes/discount.pipe';
import { TruncatePipe } from './pipes/truncate.pipe';
import { SafeHtmlPipe } from './pipes/safe-html.pipe';
import { RelativeTimePipe } from './pipes/relative-time.pipe';
import { HighlightPipe } from './pipes/highlight.pipe';
import { FormatPricePipe } from './pipes/format-price.pipe';

const PIPES = [
  ThaiNumberPipe,
  ThaiDatePipe,
  ThaiCurrencyPipe,
  DiscountPipe,
  TruncatePipe,
  SafeHtmlPipe,
  RelativeTimePipe,
  HighlightPipe,
  FormatPricePipe,
];

@NgModule({
  imports: [CommonModule, ...PIPES],  // Standalone Pipes import ผ่าน imports
  exports: [...PIPES],
})
export class SharedModule {}
```

### ตัวอย่าง Component ใช้งานครบ

```typescript
// features/products/product-card/product-card.component.ts
import { Component, Input } from '@angular/core';
import { CommonModule } from '@angular/common';
import { RouterModule } from '@angular/router';
import { ThaiCurrencyPipe } from '../../../shared/pipes/thai-currency.pipe';
import { ThaiDatePipe } from '../../../shared/pipes/thai-date.pipe';
import { ThaiNumberPipe } from '../../../shared/pipes/thai-number.pipe';
import { DiscountPipe } from '../../../shared/pipes/discount.pipe';
import { TruncatePipe } from '../../../shared/pipes/truncate.pipe';
import { RelativeTimePipe } from '../../../shared/pipes/relative-time.pipe';

export interface Product {
  id: string;
  name: string;
  description: string;
  price: number;
  discountPercent: number;
  imageUrl: string;
  rating: number;
  reviewCount: number;
  category: string;
  stock: number;
  createdAt: Date;
}

@Component({
  selector: 'app-product-card',
  standalone: true,
  imports: [
    CommonModule,
    RouterModule,
    ThaiCurrencyPipe,
    ThaiDatePipe,
    ThaiNumberPipe,
    DiscountPipe,
    TruncatePipe,
    RelativeTimePipe,
  ],
  template: `
    <div class="product-card" [routerLink]="['/products', product.id]">
      <!-- รูปภาพ -->
      <div class="image-wrapper">
        <img [src]="product.imageUrl" [alt]="product.name">
        <span
          class="discount-badge"
          *ngIf="product.discountPercent > 0"
        >
          {{ product.discountPercent | discount:0:'percent' }}
        </span>
        <span
          class="out-of-stock"
          *ngIf="product.stock === 0"
        >
          สินค้าหมด
        </span>
      </div>

      <!-- ข้อมูลสินค้า -->
      <div class="card-body">
        <h3 class="product-name">{{ product.name | truncate:60 }}</h3>
        <p class="product-description">{{ product.description | truncate:100 }}</p>

        <!-- ราคา -->
        <div class="price-section">
          <span
            class="original-price"
            *ngIf="product.discountPercent > 0"
          >
            {{ product.price | thaiCurrency }}
          </span>
          <span class="final-price">
            {{ product.price | discount:product.discountPercent | thaiCurrency }}
          </span>
        </div>

        <!-- Rating และ Reviews -->
        <div class="rating-section">
          <div class="stars">
            <span *ngFor="let star of getStars(product.rating)">{{ star }}</span>
          </div>
          <span class="review-count">
            ({{ product.reviewCount | thaiNumber }})
          </span>
        </div>

        <!-- Stock -->
        <div class="stock-info" *ngIf="product.stock > 0 && product.stock < 10">
          <span class="low-stock">
            เหลือเพียง {{ product.stock | thaiNumber:'thai' }} ชิ้น!
          </span>
        </div>

        <!-- เพิ่มเมื่อ -->
        <div class="meta">
          <small class="date-added">เพิ่มเมื่อ {{ product.createdAt | thaiDate:'relative' }}</small>
        </div>
      </div>
    </div>
  `,
  styleUrls: ['./product-card.component.scss'],
})
export class ProductCardComponent {
  @Input({ required: true }) product!: Product;

  getStars(rating: number): string[] {
    const stars = [];
    for (let i = 1; i <= 5; i++) {
      if (rating >= i) stars.push('⭐');
      else if (rating >= i - 0.5) stars.push('✨');
      else stars.push('☆');
    }
    return stars;
  }
}
```

### Unit Test สำหรับ Custom Pipes

```typescript
// shared/pipes/thai-currency.pipe.spec.ts
import { ThaiCurrencyPipe } from './thai-currency.pipe';

describe('ThaiCurrencyPipe', () => {
  let pipe: ThaiCurrencyPipe;

  beforeEach(() => {
    pipe = new ThaiCurrencyPipe();
  });

  it('should create an instance', () => {
    expect(pipe).toBeTruthy();
  });

  it('should format THB by default', () => {
    const result = pipe.transform(1500);
    expect(result).toBe('฿1,500');
  });

  it('should show decimals when requested', () => {
    const result = pipe.transform(1500.5, { showDecimal: true });
    expect(result).toBe('฿1,500.50');
  });

  it('should format USD', () => {
    const result = pipe.transform(1500, { currency: 'USD' });
    expect(result).toBe('$1,500');
  });

  it('should return "-" for null', () => {
    const result = pipe.transform(null);
    expect(result).toBe('-');
  });

  it('should format compact millions', () => {
    const result = pipe.transform(1500000, { compact: true });
    expect(result).toBe('฿1.5M');
  });

  it('should format compact thousands', () => {
    const result = pipe.transform(42500, { compact: true });
    expect(result).toBe('฿42.5K');
  });
});
```

```typescript
// shared/pipes/thai-date.pipe.spec.ts
import { ThaiDatePipe } from './thai-date.pipe';

describe('ThaiDatePipe', () => {
  let pipe: ThaiDatePipe;

  beforeEach(() => {
    pipe = new ThaiDatePipe();
  });

  const testDate = new Date('2024-01-15T14:30:00');

  it('should format medium by default', () => {
    const result = pipe.transform(testDate);
    expect(result).toBe('15 ม.ค. 2567');
  });

  it('should format short', () => {
    const result = pipe.transform(testDate, 'short');
    expect(result).toBe('15/1/2567');
  });

  it('should format long', () => {
    const result = pipe.transform(testDate, 'long');
    expect(result).toBe('15 มกราคม 2567');
  });

  it('should format time', () => {
    const result = pipe.transform(testDate, 'time');
    expect(result).toBe('14:30 น.');
  });

  it('should return empty string for null', () => {
    expect(pipe.transform(null)).toBe('');
  });
});
```

---

## สรุป

| Pipe | หน้าที่ |
|------|---------|
| **ThaiNumberPipe** | แปลงตัวเลขเป็นเลขไทย หรือรูปแบบย่อ |
| **ThaiDatePipe** | แสดงวันที่เป็น พ.ศ. ภาษาไทย |
| **ThaiCurrencyPipe** | แสดงสกุลเงินไทย และสกุลเงินต่างประเทศ |
| **DiscountPipe** | คำนวณราคาหลังส่วนลด |
| **TruncatePipe** | ตัดข้อความยาว |
| **SafeHtmlPipe** | แสดง HTML อย่างปลอดภัย |
| **RelativeTimePipe** | แสดงเวลาแบบ relative |
| **HighlightPipe** | Highlight keyword ในข้อความ |

### Best Practices

1. **ใช้ Pure Pipe** เป็นหลัก เพราะ performance ดีกว่า
2. **ใช้ Impure Pipe** เฉพาะเมื่อต้องการรับรู้การเปลี่ยนแปลงภายใน Array/Object
3. **Standalone Pipe** ใน Angular 14+ ง่ายต่อการ reuse
4. **รองรับ null/undefined** เสมอใน transform method
5. **Unit Test Pipes** ง่ายมากเพราะเป็น Pure Function
6. **ไม่ใส่ HTTP Call** ใน Pipe ใช้ Service แทน แล้วใช้ async pipe
