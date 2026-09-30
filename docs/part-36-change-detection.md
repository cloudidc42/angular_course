# Part 36: Change Detection ใน Angular

## บทนำ

Change Detection คือกระบวนการที่ Angular ใช้ตรวจสอบว่าข้อมูลใน Component มีการเปลี่ยนแปลงหรือไม่ และอัปเดต DOM ตามที่จำเป็น การเข้าใจ Change Detection อย่างลึกซึ้งจะช่วยให้เราเพิ่มประสิทธิภาพแอปพลิเคชันได้อย่างมีนัยสำคัญ

---

## 1. Default Change Detection Strategy

โดยค่าเริ่มต้น Angular จะตรวจสอบทุก Component ในต้นไม้ Component ทุกครั้งที่มี:
- Event เกิดขึ้น (click, keyup, etc.)
- HTTP Request เสร็จสิ้น
- Timer (setTimeout, setInterval) ทำงาน
- Promise resolve

```typescript
// app/components/counter/counter.component.ts
import { Component } from '@angular/core';

@Component({
  selector: 'app-counter',
  template: `
    <div class="counter">
      <h2>Counter: {{ count }}</h2>
      <button (click)="increment()">เพิ่ม</button>
      <button (click)="decrement()">ลด</button>
      <p>Render count: {{ renderCount }}</p>
    </div>
  `
})
export class CounterComponent {
  count = 0;
  renderCount = 0;

  increment() {
    this.count++;
  }

  decrement() {
    this.count--;
  }

  // ngDoCheck จะถูกเรียกทุกครั้งที่ Angular ทำ Change Detection
  ngDoCheck() {
    this.renderCount++;
  }
}
```

---

## 2. OnPush Strategy

OnPush Strategy บอก Angular ให้ตรวจสอบ Component เฉพาะเมื่อ:
1. Input reference เปลี่ยน (ไม่ใช่แค่ค่า)
2. Event เกิดขึ้นใน Component หรือ child ของมัน
3. Async pipe ได้รับค่าใหม่
4. เรียก `markForCheck()` หรือ `detectChanges()` ด้วยตนเอง

```typescript
// app/components/user-card/user-card.component.ts
import { Component, Input, ChangeDetectionStrategy, OnInit } from '@angular/core';

interface User {
  id: number;
  name: string;
  email: string;
  role: string;
}

@Component({
  selector: 'app-user-card',
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <div class="user-card">
      <h3>{{ user.name }}</h3>
      <p>Email: {{ user.email }}</p>
      <p>Role: {{ user.role }}</p>
    </div>
  `,
  styles: [`
    .user-card {
      border: 1px solid #ddd;
      padding: 16px;
      border-radius: 8px;
      margin: 8px;
    }
  `]
})
export class UserCardComponent implements OnInit {
  @Input() user!: User;

  ngOnInit() {
    console.log('UserCard initialized for:', this.user.name);
  }
}
```

```typescript
// app/components/user-list/user-list.component.ts
import { Component } from '@angular/core';

interface User {
  id: number;
  name: string;
  email: string;
  role: string;
}

@Component({
  selector: 'app-user-list',
  template: `
    <div>
      <h2>รายชื่อผู้ใช้</h2>
      <button (click)="addUser()">เพิ่มผู้ใช้</button>
      <button (click)="updateFirstUser()">อัปเดตผู้ใช้แรก (mutate)</button>
      <button (click)="replaceFirstUser()">แทนที่ผู้ใช้แรก (immutable)</button>

      <app-user-card
        *ngFor="let user of users"
        [user]="user">
      </app-user-card>
    </div>
  `
})
export class UserListComponent {
  users: User[] = [
    { id: 1, name: 'สมชาย', email: 'somchai@example.com', role: 'admin' },
    { id: 2, name: 'สมหญิง', email: 'somying@example.com', role: 'user' }
  ];

  addUser() {
    // สร้าง array ใหม่ (immutable) - OnPush จะทำงาน
    this.users = [
      ...this.users,
      {
        id: this.users.length + 1,
        name: `ผู้ใช้ ${this.users.length + 1}`,
        email: `user${this.users.length + 1}@example.com`,
        role: 'user'
      }
    ];
  }

  updateFirstUser() {
    // Mutate object โดยตรง - OnPush จะไม่อัปเดต!
    this.users[0].name = 'ชื่อใหม่ (จะไม่อัปเดต)';
  }

  replaceFirstUser() {
    // สร้าง object ใหม่ - OnPush จะทำงาน
    const newUsers = [...this.users];
    newUsers[0] = { ...newUsers[0], name: 'ชื่อใหม่ (จะอัปเดต)' };
    this.users = newUsers;
  }
}
```

---

## 3. ChangeDetectorRef

`ChangeDetectorRef` ให้เราควบคุม Change Detection ด้วยตนเอง

### 3.1 markForCheck()

```typescript
// app/components/realtime-data/realtime-data.component.ts
import {
  Component,
  ChangeDetectionStrategy,
  ChangeDetectorRef,
  OnInit,
  OnDestroy
} from '@angular/core';
import { Subject, interval } from 'rxjs';
import { takeUntil } from 'rxjs/operators';

@Component({
  selector: 'app-realtime-data',
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <div class="realtime">
      <h3>ข้อมูล Real-time</h3>
      <p>เวลาปัจจุบัน: {{ currentTime }}</p>
      <p>จำนวนอัปเดต: {{ updateCount }}</p>
    </div>
  `
})
export class RealtimeDataComponent implements OnInit, OnDestroy {
  currentTime = '';
  updateCount = 0;
  private destroy$ = new Subject<void>();

  constructor(private cdr: ChangeDetectorRef) {}

  ngOnInit() {
    // อัปเดตทุก 1 วินาที
    interval(1000)
      .pipe(takeUntil(this.destroy$))
      .subscribe(() => {
        this.currentTime = new Date().toLocaleTimeString('th-TH');
        this.updateCount++;
        // บอก Angular ว่า Component นี้ต้องการอัปเดต
        this.cdr.markForCheck();
      });
  }

  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

### 3.2 detectChanges()

```typescript
// app/components/manual-detect/manual-detect.component.ts
import {
  Component,
  ChangeDetectionStrategy,
  ChangeDetectorRef
} from '@angular/core';

@Component({
  selector: 'app-manual-detect',
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <div>
      <h3>Manual Change Detection</h3>
      <p>ข้อความ: {{ message }}</p>
      <button (click)="updateAsync()">อัปเดตแบบ Async</button>
    </div>
  `
})
export class ManualDetectComponent {
  message = 'ข้อความเริ่มต้น';

  constructor(private cdr: ChangeDetectorRef) {}

  updateAsync() {
    // จำลอง callback จาก third-party library
    setTimeout(() => {
      this.message = 'ข้อความจาก callback';
      // detectChanges() จะรัน Change Detection ทันทีสำหรับ Component นี้และลูก
      this.cdr.detectChanges();
    }, 1000);
  }
}
```

---

## 4. detach() และ reattach()

```typescript
// app/components/heavy-list/heavy-list.component.ts
import {
  Component,
  ChangeDetectorRef,
  OnInit,
  OnDestroy
} from '@angular/core';

interface Item {
  id: number;
  value: number;
  label: string;
}

@Component({
  selector: 'app-heavy-list',
  template: `
    <div>
      <h3>Heavy List (Detached Change Detection)</h3>
      <div class="controls">
        <button (click)="attach()">เปิด Change Detection</button>
        <button (click)="detach()">ปิด Change Detection</button>
        <button (click)="manualCheck()">ตรวจสอบแบบ Manual</button>
      </div>
      <p>Status: {{ isAttached ? 'กำลังตรวจสอบ' : 'หยุดตรวจสอบ' }}</p>
      <p>จำนวน items: {{ items.length }}</p>
      <ul>
        <li *ngFor="let item of items">
          {{ item.label }}: {{ item.value }}
        </li>
      </ul>
    </div>
  `
})
export class HeavyListComponent implements OnInit, OnDestroy {
  items: Item[] = [];
  isAttached = true;
  private intervalId?: ReturnType<typeof setInterval>;

  constructor(private cdr: ChangeDetectorRef) {}

  ngOnInit() {
    // จำลองข้อมูลที่เปลี่ยนบ่อย
    this.intervalId = setInterval(() => {
      this.items = Array.from({ length: 100 }, (_, i) => ({
        id: i,
        value: Math.random() * 100,
        label: `Item ${i}`
      }));
    }, 100);
  }

  attach() {
    this.cdr.reattach();
    this.isAttached = true;
  }

  detach() {
    // หยุด Change Detection สำหรับ Component นี้
    this.cdr.detach();
    this.isAttached = false;
  }

  manualCheck() {
    // ตรวจสอบเพียงครั้งเดียวแม้จะ detach
    this.cdr.detectChanges();
  }

  ngOnDestroy() {
    if (this.intervalId) {
      clearInterval(this.intervalId);
    }
  }
}
```

---

## 5. Zone.js และ NgZone

Zone.js คือ library ที่ Angular ใช้ติดตาม async operations

```typescript
// app/services/performance.service.ts
import { Injectable, NgZone } from '@angular/core';
import { Subject } from 'rxjs';

@Injectable({
  providedIn: 'root'
})
export class PerformanceService {
  private dataUpdated$ = new Subject<any[]>();
  dataUpdated = this.dataUpdated$.asObservable();

  constructor(private ngZone: NgZone) {}

  // รัน code นอก Zone เพื่อหลีกเลี่ยง Change Detection
  startHeavyComputation() {
    this.ngZone.runOutsideAngular(() => {
      // งานหนักที่ไม่ต้องการให้ Angular ตรวจสอบทุก tick
      const result = this.heavyComputation();

      // กลับมา Zone เมื่อต้องการอัปเดต UI
      this.ngZone.run(() => {
        this.dataUpdated$.next(result);
      });
    });
  }

  private heavyComputation(): any[] {
    const results = [];
    for (let i = 0; i < 10000; i++) {
      results.push({ id: i, value: Math.sqrt(i) });
    }
    return results;
  }
}
```

```typescript
// app/components/zone-example/zone-example.component.ts
import { Component, NgZone, ChangeDetectorRef } from '@angular/core';

@Component({
  selector: 'app-zone-example',
  template: `
    <div>
      <h3>NgZone Example</h3>
      <p>ผลลัพธ์: {{ result }}</p>
      <p>เวลาที่ใช้: {{ duration }}ms</p>
      <button (click)="runInsideZone()">รันใน Zone (ช้า)</button>
      <button (click)="runOutsideZone()">รันนอก Zone (เร็ว)</button>
    </div>
  `
})
export class ZoneExampleComponent {
  result = 0;
  duration = 0;

  constructor(
    private ngZone: NgZone,
    private cdr: ChangeDetectorRef
  ) {}

  runInsideZone() {
    const start = performance.now();
    let sum = 0;

    // Loop นี้จะ trigger Change Detection ทุก iteration!
    for (let i = 0; i < 100000; i++) {
      sum += i;
    }

    this.result = sum;
    this.duration = Math.round(performance.now() - start);
  }

  runOutsideZone() {
    const start = performance.now();

    this.ngZone.runOutsideAngular(() => {
      let sum = 0;
      for (let i = 0; i < 100000; i++) {
        sum += i;
      }

      // อัปเดต UI เพียงครั้งเดียวเมื่อเสร็จ
      this.ngZone.run(() => {
        this.result = sum;
        this.duration = Math.round(performance.now() - start);
      });
    });
  }
}
```

---

## 6. Signals (Angular 16+)

Angular 16 แนะนำ Signals ซึ่งเป็นระบบ reactivity ใหม่ที่ทำงานร่วมกับ Change Detection ได้ดีกว่า

```typescript
// app/components/signals-demo/signals-demo.component.ts
import { Component, signal, computed, effect } from '@angular/core';

@Component({
  selector: 'app-signals-demo',
  template: `
    <div>
      <h3>Signals Demo</h3>

      <div>
        <h4>Counter</h4>
        <p>Count: {{ count() }}</p>
        <p>Double: {{ double() }}</p>
        <p>Is Even: {{ isEven() }}</p>
        <button (click)="increment()">+</button>
        <button (click)="decrement()">-</button>
        <button (click)="reset()">Reset</button>
      </div>

      <div>
        <h4>ประวัติการเปลี่ยนแปลง</h4>
        <ul>
          <li *ngFor="let entry of history()">{{ entry }}</li>
        </ul>
      </div>
    </div>
  `
})
export class SignalsDemoComponent {
  // สร้าง signal
  count = signal(0);
  history = signal<string[]>([]);

  // computed signal - อัปเดตอัตโนมัติเมื่อ count เปลี่ยน
  double = computed(() => this.count() * 2);
  isEven = computed(() => this.count() % 2 === 0);

  constructor() {
    // effect จะทำงานทุกครั้งที่ signal ที่ใช้เปลี่ยน
    effect(() => {
      const current = this.count();
      this.history.update(h => [
        ...h,
        `เวลา ${new Date().toLocaleTimeString('th-TH')}: count = ${current}`
      ].slice(-5)); // เก็บแค่ 5 รายการล่าสุด
    });
  }

  increment() {
    this.count.update(v => v + 1);
  }

  decrement() {
    this.count.update(v => v - 1);
  }

  reset() {
    this.count.set(0);
  }
}
```

---

## 7. Best Practices สำหรับ Change Detection

### 7.1 ใช้ async pipe แทน manual subscription

```typescript
// ไม่ดี - ต้อง markForCheck ด้วยตนเอง
@Component({
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `<p>{{ data }}</p>`
})
export class BadComponent implements OnInit {
  data: any;

  constructor(
    private service: DataService,
    private cdr: ChangeDetectorRef
  ) {}

  ngOnInit() {
    this.service.getData().subscribe(data => {
      this.data = data;
      this.cdr.markForCheck(); // ต้องเรียกเอง!
    });
  }
}

// ดี - async pipe จัดการให้อัตโนมัติ
@Component({
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `<p>{{ data$ | async }}</p>`
})
export class GoodComponent {
  data$ = this.service.getData();

  constructor(private service: DataService) {}
}
```

### 7.2 Immutable data patterns

```typescript
// app/store/todo.store.ts
import { Injectable, signal, computed } from '@angular/core';

interface Todo {
  id: number;
  text: string;
  completed: boolean;
}

@Injectable({ providedIn: 'root' })
export class TodoStore {
  private todos = signal<Todo[]>([]);

  // computed values
  allTodos = computed(() => this.todos());
  completedTodos = computed(() => this.todos().filter(t => t.completed));
  pendingTodos = computed(() => this.todos().filter(t => !t.completed));
  totalCount = computed(() => this.todos().length);

  addTodo(text: string) {
    const newTodo: Todo = {
      id: Date.now(),
      text,
      completed: false
    };
    // Immutable update
    this.todos.update(todos => [...todos, newTodo]);
  }

  toggleTodo(id: number) {
    // Immutable update
    this.todos.update(todos =>
      todos.map(todo =>
        todo.id === id
          ? { ...todo, completed: !todo.completed }
          : todo
      )
    );
  }

  removeTodo(id: number) {
    this.todos.update(todos => todos.filter(todo => todo.id !== id));
  }
}
```

---

## 8. การ Debug Change Detection

```typescript
// app/directives/check-changes.directive.ts
import { Directive, DoCheck, Input } from '@angular/core';

@Directive({
  selector: '[appCheckChanges]'
})
export class CheckChangesDirective implements DoCheck {
  @Input('appCheckChanges') label = '';
  private checkCount = 0;

  ngDoCheck() {
    this.checkCount++;
    if (this.checkCount <= 10) {
      console.log(`[${this.label}] Change Detection ran ${this.checkCount} times`);
    }
  }
}
```

```typescript
// การใช้งาน
// <div appCheckChanges="MyComponent">...</div>
```

---

## 9. ตัวอย่างสมบูรณ์: Dashboard ที่มีประสิทธิภาพ

```typescript
// app/components/dashboard/dashboard.component.ts
import {
  Component,
  ChangeDetectionStrategy,
  signal,
  computed,
  OnInit,
  OnDestroy
} from '@angular/core';
import { interval, Subject } from 'rxjs';
import { takeUntil, map } from 'rxjs/operators';
import { toSignal } from '@angular/core/rxjs-interop';

interface Metric {
  label: string;
  value: number;
  unit: string;
  trend: 'up' | 'down' | 'stable';
}

@Component({
  selector: 'app-dashboard',
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <div class="dashboard">
      <h2>Dashboard</h2>

      <div class="metrics-grid">
        <div
          class="metric-card"
          *ngFor="let metric of metrics(); trackBy: trackMetric"
          [class.trend-up]="metric.trend === 'up'"
          [class.trend-down]="metric.trend === 'down'"
        >
          <h4>{{ metric.label }}</h4>
          <p class="value">{{ metric.value | number:'1.0-2' }} {{ metric.unit }}</p>
          <span class="trend">
            {{ metric.trend === 'up' ? '↑' : metric.trend === 'down' ? '↓' : '→' }}
          </span>
        </div>
      </div>

      <div class="summary">
        <p>ค่าเฉลี่ย CPU: {{ avgCPU() | number:'1.0-1' }}%</p>
        <p>อัปเดตล่าสุด: {{ lastUpdated() }}</p>
      </div>
    </div>
  `,
  styles: [`
    .dashboard { padding: 20px; }
    .metrics-grid { display: grid; grid-template-columns: repeat(3, 1fr); gap: 16px; }
    .metric-card { padding: 16px; border: 1px solid #ddd; border-radius: 8px; }
    .trend-up { border-color: green; }
    .trend-down { border-color: red; }
    .value { font-size: 24px; font-weight: bold; }
  `]
})
export class DashboardComponent implements OnInit, OnDestroy {
  metrics = signal<Metric[]>([
    { label: 'CPU Usage', value: 45, unit: '%', trend: 'stable' },
    { label: 'Memory', value: 2.4, unit: 'GB', trend: 'up' },
    { label: 'Disk I/O', value: 12, unit: 'MB/s', trend: 'down' },
    { label: 'Network', value: 100, unit: 'Mbps', trend: 'stable' },
    { label: 'Requests', value: 1250, unit: '/min', trend: 'up' },
    { label: 'Error Rate', value: 0.5, unit: '%', trend: 'down' }
  ]);

  lastUpdated = signal('ยังไม่ได้อัปเดต');

  avgCPU = computed(() => {
    const cpuMetric = this.metrics().find(m => m.label === 'CPU Usage');
    return cpuMetric?.value ?? 0;
  });

  private destroy$ = new Subject<void>();

  ngOnInit() {
    interval(2000)
      .pipe(takeUntil(this.destroy$))
      .subscribe(() => {
        this.updateMetrics();
      });
  }

  private updateMetrics() {
    this.metrics.update(metrics =>
      metrics.map(metric => ({
        ...metric,
        value: this.randomizeValue(metric.value),
        trend: this.getTrend()
      }))
    );
    this.lastUpdated.set(new Date().toLocaleTimeString('th-TH'));
  }

  private randomizeValue(current: number): number {
    const change = (Math.random() - 0.5) * current * 0.1;
    return Math.max(0, current + change);
  }

  private getTrend(): 'up' | 'down' | 'stable' {
    const rand = Math.random();
    if (rand < 0.33) return 'up';
    if (rand < 0.66) return 'down';
    return 'stable';
  }

  trackMetric(index: number, metric: Metric): string {
    return metric.label;
  }

  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

---

## สรุป

| เทคนิค | ใช้เมื่อ |
|--------|---------|
| Default | Component เล็ก ๆ ที่ไม่ซับซ้อน |
| OnPush | Component ที่มี Input แบบ immutable |
| markForCheck() | มี async operation นอก Angular zone |
| detectChanges() | ต้องการ update ทันที |
| detach/reattach | Component ที่อัปเดตบ่อยมาก |
| runOutsideAngular | งานหนักที่ไม่ต้องการ UI update |
| Signals | Angular 16+ สำหรับ reactive state |

Change Detection ที่มีประสิทธิภาพจะช่วยลด re-render ที่ไม่จำเป็นและทำให้แอปพลิเคชันทำงานเร็วขึ้นอย่างเห็นได้ชัด
