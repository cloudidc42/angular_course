# Part 30 — Angular Signals

## Signals คืออะไร?

Angular Signals เป็นระบบ Reactivity ใหม่ที่เพิ่มมาใน Angular 16 และพัฒนาต่อเนื่องถึง Angular 17+ ช่วยให้การจัดการ State และการตอบสนองต่อการเปลี่ยนแปลงง่ายและมีประสิทธิภาพมากขึ้น

### ปัญหาที่ Signals แก้ไข

ก่อนหน้านี้ Angular ใช้ Zone.js ในการตรวจจับการเปลี่ยนแปลง (Change Detection) ซึ่งทำงานโดยการตรวจสอบ Component Tree ทั้งหมด Signals แก้ปัญหานี้โดย:

1. **Granular Reactivity** — รู้ว่า Value ไหนเปลี่ยน ไม่ต้องตรวจสอบทั้งหมด
2. **Better Performance** — ลด Change Detection ที่ไม่จำเป็น
3. **Simpler Code** — โค้ดอ่านง่ายกว่า RxJS ในกรณีที่ไม่ซับซ้อน
4. **No Zone.js Dependency** — รองรับการรัน Zoneless ในอนาคต

---

## signal() — สร้าง Writable Signal

```typescript
import { signal } from '@angular/core';

// สร้าง Signal
const count = signal(0);         // Signal<number>
const name = signal('สมชาย');    // Signal<string>
const items = signal<string[]>([]);  // Signal<string[]>

// อ่านค่า — เรียกเป็น Function
console.log(count());    // 0
console.log(name());     // 'สมชาย'

// เปลี่ยนค่าด้วย set()
count.set(5);
name.set('สมหญิง');

// เปลี่ยนค่าจากค่าเดิมด้วย update()
count.update((current) => current + 1);
items.update((list) => [...list, 'รายการใหม่']);

// mutate() สำหรับ Objects/Arrays (Angular 16 เท่านั้น ถูกเอาออกใน 17)
// ไม่แนะนำ — ใช้ update() แทน
```

---

## computed() — Derived Signal

Computed Signal คำนวณค่าจาก Signal อื่น และ Memoize ผลลัพธ์

```typescript
import { signal, computed } from '@angular/core';

const price = signal(100);
const quantity = signal(3);
const discount = signal(10);  // เปอร์เซ็นต์

// Computed Signal
const subtotal = computed(() => price() * quantity());
const discountAmount = computed(() => subtotal() * (discount() / 100));
const total = computed(() => subtotal() - discountAmount());

console.log(subtotal());      // 300
console.log(discountAmount()); // 30
console.log(total());         // 270

price.set(150);
console.log(subtotal());      // 450 — คำนวณใหม่อัตโนมัติ
console.log(total());         // 405 — คำนวณตาม chain
```

### Computed ที่ซับซ้อน

```typescript
const users = signal<User[]>([
  { id: 1, name: 'Alice', role: 'admin', isActive: true },
  { id: 2, name: 'Bob', role: 'viewer', isActive: false },
  { id: 3, name: 'Charlie', role: 'editor', isActive: true },
]);

const filterRole = signal<string | null>(null);
const searchTerm = signal('');

// Derived: กรองตาม Role และ Search
const filteredUsers = computed(() => {
  const term = searchTerm().toLowerCase();
  const role = filterRole();

  return users().filter((user) => {
    const matchRole = !role || user.role === role;
    const matchSearch = !term || user.name.toLowerCase().includes(term);
    return matchRole && matchSearch;
  });
});

// Derived: สถิติ
const stats = computed(() => {
  const all = users();
  return {
    total: all.length,
    active: all.filter((u) => u.isActive).length,
    admins: all.filter((u) => u.role === 'admin').length,
  };
});
```

---

## effect() — Side Effects

effect() ทำงานเมื่อ Signals ที่ใช้ภายในเปลี่ยนค่า

```typescript
import { signal, computed, effect } from '@angular/core';

const theme = signal<'light' | 'dark'>('light');
const fontSize = signal(16);

// Effect จะทำงานทุกครั้งที่ theme หรือ fontSize เปลี่ยน
effect(() => {
  document.body.setAttribute('data-theme', theme());
  document.documentElement.style.fontSize = `${fontSize()}px`;
  console.log(`Theme: ${theme()}, Font Size: ${fontSize()}px`);
});

theme.set('dark');  // Effect ทำงาน: "Theme: dark, Font Size: 16px"
fontSize.set(18);   // Effect ทำงาน: "Theme: dark, Font Size: 18px"
```

### Effect ใน Component

```typescript
import { Component, signal, effect } from '@angular/core';

@Component({
  selector: 'app-search',
  template: `
    <input (input)="onSearch($event)" placeholder="ค้นหา..." />
    <div *ngFor="let result of results()">{{ result }}</div>
  `
})
export class SearchComponent {
  searchTerm = signal('');
  results = signal<string[]>([]);
  isSearching = signal(false);

  constructor(private searchService: SearchService) {
    // Effect จะทำงานทุกครั้งที่ searchTerm เปลี่ยน
    effect(() => {
      const term = this.searchTerm();

      if (term.length < 2) {
        this.results.set([]);
        return;
      }

      this.isSearching.set(true);

      // ต้องระมัดระวัง: effect ไม่รองรับ async โดยตรง
      // ใช้ subscription หรือ toSignal แทน
      this.searchService.search(term).subscribe((data) => {
        this.results.set(data);
        this.isSearching.set(false);
      });
    });
  }

  onSearch(event: Event): void {
    this.searchTerm.set((event.target as HTMLInputElement).value);
  }
}
```

### การ Cleanup ใน Effect

```typescript
effect((onCleanup) => {
  const timer = setInterval(() => {
    console.log('Tick:', Date.now());
  }, 1000);

  // cleanup จะถูกเรียกก่อน Effect ทำงานครั้งถัดไป หรือเมื่อ Component ถูกทำลาย
  onCleanup(() => clearInterval(timer));
});
```

---

## Signal Inputs (Angular 17+)

Input ที่เป็น Signal ช่วยให้ใช้ Computed และ Effect กับ Input ได้

```typescript
import { Component, input, computed } from '@angular/core';

@Component({
  selector: 'app-product-card',
  standalone: true,
  template: `
    <div class="card">
      <h3>{{ product().name }}</h3>
      <div class="price">
        <span [class.sale]="isOnSale()">{{ product().price | currency:'THB' }}</span>
        <span *ngIf="isOnSale()" class="original">
          {{ product().originalPrice | currency:'THB' }}
        </span>
      </div>
      <div class="discount-badge" *ngIf="discountPercent() > 0">
        -{{ discountPercent() }}%
      </div>
    </div>
  `
})
export class ProductCardComponent {
  // input() สร้าง Signal Input
  product = input.required<Product>();

  // optional input พร้อม default value
  showBadge = input(true);

  // Computed จาก Input
  isOnSale = computed(() => this.product().isOnSale);

  discountPercent = computed(() => {
    const p = this.product();
    if (!p.isOnSale || p.originalPrice <= p.price) return 0;
    return Math.round(((p.originalPrice - p.price) / p.originalPrice) * 100);
  });
}
```

### Output Signals (Angular 17+)

```typescript
import { Component, output } from '@angular/core';

@Component({
  selector: 'app-quantity-picker',
  standalone: true,
  template: `
    <button (click)="decrease()">-</button>
    <span>{{ count() }}</span>
    <button (click)="increase()">+</button>
  `
})
export class QuantityPickerComponent {
  minValue = input(1);
  maxValue = input(99);
  initialValue = input(1);

  count = signal(1);

  // output() แทน EventEmitter
  quantityChange = output<number>();

  ngOnInit(): void {
    this.count.set(this.initialValue());
  }

  increase(): void {
    if (this.count() < this.maxValue()) {
      this.count.update((v) => v + 1);
      this.quantityChange.emit(this.count());
    }
  }

  decrease(): void {
    if (this.count() > this.minValue()) {
      this.count.update((v) => v - 1);
      this.quantityChange.emit(this.count());
    }
  }
}
```

---

## toSignal() และ toObservable()

เชื่อมต่อ Signals กับ RxJS

### toSignal() — Observable → Signal

```typescript
import { toSignal } from '@angular/core/rxjs-interop';
import { Component, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';

@Component({
  selector: 'app-products',
  standalone: true,
  template: `
    <div *ngIf="products()">
      <div *ngFor="let p of products()">{{ p.name }}</div>
    </div>
    <div *ngIf="!products()">กำลังโหลด...</div>
  `
})
export class ProductsComponent {
  private http = inject(HttpClient);

  // แปลง Observable เป็น Signal
  products = toSignal(
    this.http.get<Product[]>('/api/products'),
    { initialValue: null }  // ค่าเริ่มต้นก่อนที่ Observable จะ Emit
  );
}
```

### toObservable() — Signal → Observable

```typescript
import { toObservable } from '@angular/core/rxjs-interop';
import { Component, signal, inject } from '@angular/core';
import { switchMap } from 'rxjs/operators';

@Component({
  selector: 'app-product-detail',
  standalone: true,
  template: `
    <div *ngIf="product()">
      <h2>{{ product()?.name }}</h2>
    </div>
  `
})
export class ProductDetailComponent {
  private http = inject(HttpClient);

  selectedId = signal<number | null>(null);

  // แปลง Signal เป็น Observable แล้วทำ HTTP Request
  product = toSignal(
    toObservable(this.selectedId).pipe(
      switchMap((id) =>
        id ? this.http.get<Product>(`/api/products/${id}`) : of(null)
      )
    ),
    { initialValue: null }
  );

  selectProduct(id: number): void {
    this.selectedId.set(id);
  }
}
```

---

## Signals vs RxJS

| คุณสมบัติ | Signals | RxJS Observables |
|-----------|---------|-----------------|
| **Synchronous** | ✅ เสมอ | ❌ ขึ้นอยู่กับ Operator |
| **Lazy** | ❌ ทำงานทันที | ✅ ทำงานเมื่อ Subscribe |
| **Multiple Values** | ✅ เก็บค่าปัจจุบัน | ✅ Stream ของค่า |
| **Time-based Ops** | ❌ | ✅ debounceTime, delay |
| **HTTP Requests** | ❌ ต้อง Wrap | ✅ เหมาะมาก |
| **Simple State** | ✅ เหมาะมาก | ⚠️ ซับซ้อนกว่า |
| **Complex Async** | ⚠️ ใช้ร่วมกับ RxJS | ✅ เหมาะมาก |

### เมื่อใช้ Signals

- State ที่ไม่ซับซ้อน (ค่า Single Value)
- UI State (loading, error, selected item)
- Derived/Computed Values
- ใช้ร่วมกับ Input/Output ใน Angular 17+

### เมื่อใช้ RxJS

- HTTP Requests
- WebSocket, Server-Sent Events
- Operations ที่ต้องการ Time-based (debounce, delay, timer)
- Complex Async Flows ที่ต้องการ mergeMap, combineLatest

---

## Workshop: Reactive Counter ด้วย Signals

```typescript
// reactive-counter.component.ts
import { Component, signal, computed, effect } from '@angular/core';
import { CommonModule } from '@angular/common';
import { FormsModule } from '@angular/forms';

interface CounterHistory {
  value: number;
  action: string;
  timestamp: Date;
}

@Component({
  selector: 'app-reactive-counter',
  standalone: true,
  imports: [CommonModule, FormsModule],
  template: `
    <div class="counter">
      <h2>Reactive Counter</h2>

      <!-- Display -->
      <div
        class="display"
        [class.positive]="isPositive()"
        [class.negative]="isNegative()"
        [class.zero]="count() === 0"
      >
        {{ count() }}
      </div>

      <!-- Status -->
      <div class="status">
        สถานะ: {{ statusText() }}
        | ค่าสัมบูรณ์: {{ absoluteValue() }}
      </div>

      <!-- Controls -->
      <div class="controls">
        <button (click)="decrement()">-1</button>
        <button (click)="decrementBy(5)">-5</button>
        <button (click)="reset()" class="reset">Reset</button>
        <button (click)="incrementBy(5)">+5</button>
        <button (click)="increment()">+1</button>
      </div>

      <!-- Custom Amount -->
      <div class="custom">
        <label>ปรับค่าทีละ:</label>
        <input
          type="number"
          [(ngModel)]="customAmountStr"
          (ngModelChange)="updateCustomAmount($event)"
        />
        <button (click)="incrementBy(customAmount())">+{{ customAmount() }}</button>
        <button (click)="decrementBy(customAmount())">-{{ customAmount() }}</button>
      </div>

      <!-- Stats -->
      <div class="stats" *ngIf="sessionStats() as stats">
        <div>เพิ่มรวม: {{ stats.totalIncrement }}</div>
        <div>ลดรวม: {{ stats.totalDecrement }}</div>
        <div>จำนวน Actions: {{ stats.totalActions }}</div>
        <div>ค่าสูงสุด: {{ stats.maxValue }}</div>
        <div>ค่าต่ำสุด: {{ stats.minValue }}</div>
      </div>

      <!-- History -->
      <div class="history">
        <h4>ประวัติ ({{ history().length }} รายการ)</h4>
        <button (click)="clearHistory()" *ngIf="history().length">ล้างประวัติ</button>
        <ul>
          <li *ngFor="let h of recentHistory()">
            <span class="time">{{ h.timestamp | date:'HH:mm:ss' }}</span>
            <span class="action">{{ h.action }}</span>
            <span class="value" [class.positive]="h.value > 0" [class.negative]="h.value < 0">
              {{ h.value }}
            </span>
          </li>
        </ul>
      </div>
    </div>
  `,
  styles: [`
    .counter { max-width: 400px; margin: 0 auto; padding: 20px; }
    .display {
      font-size: 5rem;
      text-align: center;
      padding: 20px;
      border-radius: 8px;
      margin: 16px 0;
      transition: all 0.3s;
    }
    .display.positive { background: #e8f5e9; color: #2e7d32; }
    .display.negative { background: #ffebee; color: #c62828; }
    .display.zero { background: #f5f5f5; color: #333; }
    .controls, .custom { display: flex; gap: 8px; margin: 8px 0; flex-wrap: wrap; }
    button { padding: 8px 16px; cursor: pointer; border: 1px solid #ddd; border-radius: 4px; }
    button.reset { background: #ff9800; color: white; border: none; }
    .stats { display: grid; grid-template-columns: 1fr 1fr; gap: 8px; margin: 16px 0; }
    .stats div { padding: 8px; background: #f5f5f5; border-radius: 4px; }
    .history ul { max-height: 200px; overflow-y: auto; padding: 0; list-style: none; }
    .history li { display: flex; gap: 8px; padding: 4px; border-bottom: 1px solid #eee; }
    .value.positive { color: green; }
    .value.negative { color: red; }
  `]
})
export class ReactiveCounterComponent {
  // Core State
  count = signal(0);
  history = signal<CounterHistory[]>([]);
  customAmountStr = '10';
  customAmount = signal(10);

  // Computed
  isPositive = computed(() => this.count() > 0);
  isNegative = computed(() => this.count() < 0);
  absoluteValue = computed(() => Math.abs(this.count()));

  statusText = computed(() => {
    const c = this.count();
    if (c > 100) return '🔥 สูงมาก';
    if (c > 50) return '⬆️ สูง';
    if (c > 0) return '✅ บวก';
    if (c === 0) return '⭕ ศูนย์';
    if (c > -50) return '⬇️ ต่ำ';
    return '❄️ ต่ำมาก';
  });

  recentHistory = computed(() =>
    this.history().slice(-10).reverse()
  );

  sessionStats = computed(() => {
    const h = this.history();
    if (h.length === 0) return null;

    const values = h.map((e) => e.value);
    const increments = h.filter((e) => e.value > 0);
    const decrements = h.filter((e) => e.value < 0);

    return {
      totalIncrement: increments.reduce((s, e) => s + e.value, 0),
      totalDecrement: Math.abs(decrements.reduce((s, e) => s + e.value, 0)),
      totalActions: h.length,
      maxValue: Math.max(...values),
      minValue: Math.min(...values),
    };
  });

  // Effect — บันทึก State ลง LocalStorage
  private saveEffect = effect(() => {
    const state = { count: this.count(), history: this.history() };
    try {
      localStorage.setItem('counter-state', JSON.stringify(state));
    } catch {}
  });

  constructor() {
    // โหลด State จาก LocalStorage
    try {
      const saved = localStorage.getItem('counter-state');
      if (saved) {
        const { count, history } = JSON.parse(saved);
        this.count.set(count);
        this.history.set(history.map((h: any) => ({
          ...h,
          timestamp: new Date(h.timestamp),
        })));
      }
    } catch {}
  }

  private addHistory(action: string, value: number): void {
    this.history.update((h) => [
      ...h,
      { value, action, timestamp: new Date() },
    ]);
  }

  increment(): void {
    this.count.update((c) => c + 1);
    this.addHistory('increment', 1);
  }

  decrement(): void {
    this.count.update((c) => c - 1);
    this.addHistory('decrement', -1);
  }

  incrementBy(amount: number): void {
    this.count.update((c) => c + amount);
    this.addHistory(`+${amount}`, amount);
  }

  decrementBy(amount: number): void {
    this.count.update((c) => c - amount);
    this.addHistory(`-${amount}`, -amount);
  }

  reset(): void {
    const old = this.count();
    this.count.set(0);
    this.addHistory('reset', -old);
  }

  updateCustomAmount(value: string): void {
    const num = parseInt(value, 10);
    if (!isNaN(num) && num > 0) {
      this.customAmount.set(num);
    }
  }

  clearHistory(): void {
    this.history.set([]);
  }
}
```

---

## Advanced: Signal-based Service

```typescript
// theme.service.ts
import { Injectable, signal, computed, effect } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class ThemeService {
  private _theme = signal<'light' | 'dark'>('light');
  private _fontSize = signal(16);
  private _primaryColor = signal('#1976d2');

  // Public read-only
  readonly theme = this._theme.asReadonly();
  readonly fontSize = this._fontSize.asReadonly();
  readonly primaryColor = this._primaryColor.asReadonly();

  // Computed
  readonly isDarkMode = computed(() => this._theme() === 'dark');
  readonly fontSizeLabel = computed(() => {
    const size = this._fontSize();
    if (size <= 14) return 'เล็ก';
    if (size <= 16) return 'ปกติ';
    if (size <= 18) return 'ใหญ่';
    return 'ใหญ่มาก';
  });

  constructor() {
    // โหลดจาก LocalStorage
    try {
      const saved = localStorage.getItem('theme-settings');
      if (saved) {
        const settings = JSON.parse(saved);
        if (settings.theme) this._theme.set(settings.theme);
        if (settings.fontSize) this._fontSize.set(settings.fontSize);
        if (settings.primaryColor) this._primaryColor.set(settings.primaryColor);
      }
    } catch {}

    // Apply theme effect
    effect(() => {
      document.documentElement.setAttribute('data-theme', this._theme());
      document.documentElement.style.setProperty('--font-size-base', `${this._fontSize()}px`);
      document.documentElement.style.setProperty('--color-primary', this._primaryColor());

      // บันทึก Settings
      try {
        localStorage.setItem('theme-settings', JSON.stringify({
          theme: this._theme(),
          fontSize: this._fontSize(),
          primaryColor: this._primaryColor(),
        }));
      } catch {}
    });
  }

  toggleTheme(): void {
    this._theme.update((t) => (t === 'light' ? 'dark' : 'light'));
  }

  setFontSize(size: number): void {
    this._fontSize.set(Math.max(12, Math.min(24, size)));
  }

  setPrimaryColor(color: string): void {
    this._primaryColor.set(color);
  }
}
```

---

## สรุป

| API | การใช้งาน |
|----|-----------|
| `signal(value)` | สร้าง Writable Signal |
| `signal.set(value)` | กำหนดค่าใหม่ทั้งหมด |
| `signal.update(fn)` | อัปเดตจากค่าเดิม |
| `signal.asReadonly()` | ทำให้ Signal เป็น Read-only |
| `computed(() => ...)` | สร้าง Derived Signal |
| `effect(() => ...)` | สร้าง Side Effect |
| `input()` / `input.required()` | Signal Input (Angular 17+) |
| `output()` | Signal Output (Angular 17+) |
| `toSignal(obs$)` | Observable → Signal |
| `toObservable(sig)` | Signal → Observable |

Angular Signals เป็น Feature ที่จะเปลี่ยนแปลงวิธีการเขียน Angular ในอนาคต ใน Part ถัดไปจะเรียน Standalone Components ซึ่งทำงานร่วมกับ Signals ได้ดีมาก
