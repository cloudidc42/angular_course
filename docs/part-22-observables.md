# Part 22 — Observables และ Subjects

## บทนำ

ในบทนี้เราจะเรียนรู้ Subjects ประเภทต่างๆ ความแตกต่างระหว่าง Hot/Cold Observables และ Higher-order Observables ที่ใช้บ่อยใน Angular

---

## 1. Subject และประเภทต่างๆ

### 1.1 Subject พื้นฐาน

Subject เป็นทั้ง Observable และ Observer ในเวลาเดียวกัน ใช้สำหรับ multicasting (ส่งค่าให้ subscribers หลายคนพร้อมกัน)

```typescript
import { Subject } from 'rxjs';

const subject = new Subject<number>();

// Subscribe ก่อน
subject.subscribe(val => console.log('Observer A:', val));
subject.subscribe(val => console.log('Observer B:', val));

// Emit ค่า
subject.next(1);  // Observer A: 1, Observer B: 1
subject.next(2);  // Observer A: 2, Observer B: 2

// Subscribe หลัง emit — ไม่ได้รับค่าก่อนหน้า
subject.subscribe(val => console.log('Observer C:', val));
subject.next(3);  // Observer A: 3, Observer B: 3, Observer C: 3

// Complete
subject.complete();

// Error
// subject.error(new Error('Something went wrong'));
```

```typescript
// การใช้ Subject ใน Angular Service
@Injectable({ providedIn: 'root' })
export class EventBusService {
  private eventSubject = new Subject<AppEvent>();

  // expose เป็น Observable (ไม่ให้ access .next() จากภายนอก)
  events$ = this.eventSubject.asObservable();

  emit(event: AppEvent): void {
    this.eventSubject.next(event);
  }
}
```

### 1.2 BehaviorSubject

BehaviorSubject เก็บค่าปัจจุบัน (current value) และส่งค่านั้นให้ subscriber ใหม่ทันที

```typescript
import { BehaviorSubject } from 'rxjs';

// ต้องกำหนด initial value
const behaviorSubject = new BehaviorSubject<number>(0);

behaviorSubject.subscribe(val => console.log('A:', val));
// A: 0  (ได้รับค่าเริ่มต้นทันที)

behaviorSubject.next(1);
// A: 1

behaviorSubject.next(2);
// A: 2

// Subscribe ใหม่ — ได้รับค่าล่าสุด (2) ทันที
behaviorSubject.subscribe(val => console.log('B:', val));
// B: 2

behaviorSubject.next(3);
// A: 3, B: 3

// อ่านค่าปัจจุบัน (synchronous)
console.log('Current value:', behaviorSubject.getValue());
// Current value: 3
```

```typescript
// การใช้ BehaviorSubject สำหรับ State Management
@Injectable({ providedIn: 'root' })
export class UserService {
  // State
  private userSubject = new BehaviorSubject<User | null>(null);
  private loadingSubject = new BehaviorSubject<boolean>(false);

  // Public observables (read-only)
  user$ = this.userSubject.asObservable();
  loading$ = this.loadingSubject.asObservable();
  isLoggedIn$ = this.user$.pipe(map(user => user !== null));

  login(credentials: LoginCredentials): Observable<User> {
    this.loadingSubject.next(true);

    return this.http.post<User>('/api/login', credentials).pipe(
      tap(user => {
        this.userSubject.next(user);
        this.loadingSubject.next(false);
      }),
      catchError(error => {
        this.loadingSubject.next(false);
        return throwError(() => error);
      })
    );
  }

  logout(): void {
    this.userSubject.next(null);
  }

  getCurrentUser(): User | null {
    return this.userSubject.getValue();
  }
}
```

### 1.3 ReplaySubject

ReplaySubject เก็บ buffer ของ n ค่าล่าสุดและส่งให้ subscriber ใหม่

```typescript
import { ReplaySubject } from 'rxjs';

// เก็บ 3 ค่าล่าสุด
const replaySubject = new ReplaySubject<number>(3);

replaySubject.next(1);
replaySubject.next(2);
replaySubject.next(3);
replaySubject.next(4);
replaySubject.next(5);

// Subscribe ใหม่ — ได้รับ 3 ค่าล่าสุด (3, 4, 5) ทันที
replaySubject.subscribe(val => console.log('B:', val));
// B: 3, B: 4, B: 5

// กำหนด time window ด้วย (เก็บค่าที่ emit ภายใน 2 วินาที)
const timedReplay = new ReplaySubject<string>(100, 2000);
```

```typescript
// การใช้ ReplaySubject สำหรับ chat messages
@Injectable({ providedIn: 'root' })
export class ChatService {
  // เก็บ 50 ข้อความล่าสุด
  private messagesSubject = new ReplaySubject<ChatMessage>(50);
  messages$ = this.messagesSubject.asObservable();

  sendMessage(content: string): void {
    const message: ChatMessage = {
      id: Date.now(),
      content,
      timestamp: new Date(),
      sender: 'user'
    };
    this.messagesSubject.next(message);
  }
}
```

### 1.4 AsyncSubject

AsyncSubject emit เฉพาะค่าสุดท้ายและเฉพาะเมื่อ complete

```typescript
import { AsyncSubject } from 'rxjs';

const asyncSubject = new AsyncSubject<number>();

asyncSubject.subscribe(val => console.log('A:', val));

asyncSubject.next(1);  // ไม่ emit
asyncSubject.next(2);  // ไม่ emit
asyncSubject.next(3);  // ไม่ emit

// Subscribe ใหม่ก่อน complete
asyncSubject.subscribe(val => console.log('B:', val));

asyncSubject.next(4);  // ยังไม่ emit
asyncSubject.complete();
// A: 4
// B: 4  (ได้รับค่าสุดท้ายทันที เพราะ complete แล้ว)

// Subscribe ใหม่หลัง complete — ก็ยังได้รับค่าสุดท้าย
asyncSubject.subscribe(val => console.log('C:', val));
// C: 4
```

---

## 2. Hot vs Cold Observables

### 2.1 Cold Observables

Cold Observable เริ่ม execution ใหม่ทุกครั้งที่มี subscriber ใหม่ (like a movie on demand)

```typescript
import { Observable, interval } from 'rxjs';
import { take } from 'rxjs/operators';

// Cold Observable — แต่ละ subscriber ได้ stream ของตัวเอง
const cold$ = new Observable<number>(subscriber => {
  console.log('Observable started for new subscriber');
  let count = 0;
  const timer = setInterval(() => subscriber.next(count++), 1000);
  return () => clearInterval(timer);
});

// Subscriber 1 เริ่ม execution ใหม่
cold$.pipe(take(3)).subscribe(val => console.log('S1:', val));
// Observable started for new subscriber
// S1: 0, S1: 1, S1: 2

// รอ 2 วินาที
setTimeout(() => {
  // Subscriber 2 เริ่ม execution ใหม่ (ไม่เกี่ยวกับ S1)
  cold$.pipe(take(3)).subscribe(val => console.log('S2:', val));
  // Observable started for new subscriber
  // S2: 0, S2: 1, S2: 2 (เริ่มจาก 0 ใหม่)
}, 2000);
```

### 2.2 Hot Observables

Hot Observable มี shared execution — subscribers ทุกคนรับค่าเดียวกัน ณ เวลาเดียวกัน (like live TV)

```typescript
import { Subject, fromEvent } from 'rxjs';

// Subject เป็น Hot Observable
const hot$ = new Subject<number>();

// Subscriber 1
hot$.subscribe(val => console.log('S1:', val));

hot$.next(1);   // S1: 1
hot$.next(2);   // S1: 2

// Subscriber 2 สมัครทีหลัง
hot$.subscribe(val => console.log('S2:', val));

hot$.next(3);   // S1: 3, S2: 3 (ได้รับพร้อมกัน)
hot$.next(4);   // S1: 4, S2: 4

// fromEvent เป็น Hot Observable
const clicks$ = fromEvent(document, 'click');
// clicks$ จะ emit เมื่อ user คลิก ไม่ใช่ตอน subscribe
```

### 2.3 แปลง Cold เป็น Hot ด้วย share/publish

```typescript
import { interval, share, publish, refCount, shareReplay } from 'rxjs';

// share() = publish() + refCount()
const shared$ = interval(1000).pipe(
  share()
);

// Subscriber 1 และ 2 ใช้ execution เดียวกัน
shared$.pipe(take(5)).subscribe(val => console.log('S1:', val));

setTimeout(() => {
  shared$.pipe(take(3)).subscribe(val => console.log('S2:', val));
}, 2000);
// S2 เริ่มรับจากค่าที่ 2 ต่อไป ไม่ใช่จาก 0 ใหม่

// shareReplay — เก็บ buffer สำหรับ subscriber ใหม่
const sharedReplay$ = this.http.get<User[]>('/api/users').pipe(
  shareReplay(1)  // เก็บ response ล่าสุด 1 ค่า
);

// Subscribe หลายครั้ง — HTTP request จะถูกเรียกแค่ครั้งเดียว
sharedReplay$.subscribe(users => this.usersList = users);
sharedReplay$.subscribe(users => this.userCount = users.length);
```

---

## 3. Multicasting

```typescript
import { Subject, interval } from 'rxjs';
import { multicast, refCount, publish } from 'rxjs/operators';

// Multicast ด้วย Subject
const source$ = interval(1000);
const multicasted$ = source$.pipe(
  multicast(() => new Subject<number>()),
  refCount()
);

// ทุก subscriber ใช้ Subject เดียวกัน
multicasted$.subscribe(val => console.log('A:', val));
multicasted$.subscribe(val => console.log('B:', val));
```

---

## 4. Higher-order Observables

Higher-order Observable คือ Observable ที่ emit Observables อื่น

### 4.1 switchMap — ยกเลิก inner Observable เก่าเมื่อมีค่าใหม่

```typescript
import { switchMap } from 'rxjs/operators';

// การใช้งานหลัก: search, navigation
this.searchControl.valueChanges.pipe(
  debounceTime(300),
  distinctUntilChanged(),
  switchMap(term => {
    if (!term) return of([]);
    // ถ้ามี term ใหม่ก่อน request เก่าเสร็จ — ยกเลิก request เก่า
    return this.searchService.search(term);
  })
).subscribe(results => this.results = results);

// ตัวอย่าง Route params
this.route.params.pipe(
  switchMap(params => this.productService.getProduct(params['id']))
  // ถ้าเปลี่ยน route ก่อน load เสร็จ — ยกเลิก request เก่า
).subscribe(product => this.product = product);
```

### 4.2 mergeMap (flatMap) — รัน inner Observables แบบขนาน

```typescript
import { mergeMap } from 'rxjs/operators';

// ใช้เมื่อต้องการ parallel requests
const userIds = [1, 2, 3, 4, 5];

from(userIds).pipe(
  mergeMap(id => this.userService.getUser(id))
  // ทุก request รันพร้อมกัน
).subscribe(user => this.users.push(user));

// กำหนด concurrency
from(userIds).pipe(
  mergeMap(id => this.userService.getUser(id), 2)  // รันได้แค่ 2 request พร้อมกัน
).subscribe(user => this.users.push(user));

// ใช้กับ file uploads
from(files).pipe(
  mergeMap(file => this.uploadService.upload(file))
).subscribe(result => console.log('Uploaded:', result.url));
```

### 4.3 concatMap — รัน inner Observables ทีละตัวตามลำดับ

```typescript
import { concatMap } from 'rxjs/operators';

// ใช้เมื่อลำดับสำคัญ (เช่น save operations)
const saveOperations = [
  { type: 'save', data: { id: 1, name: 'Updated Name' } },
  { type: 'save', data: { id: 2, name: 'Another Update' } },
  { type: 'save', data: { id: 3, name: 'Third Update' } }
];

from(saveOperations).pipe(
  concatMap(operation => this.apiService.save(operation.data))
  // รอให้ save แรกเสร็จก่อน ค่อยทำ save ต่อไป
).subscribe(result => console.log('Saved:', result));

// ใช้กับ animation queue
fromEvent(button, 'click').pipe(
  concatMap(() => this.animationService.runAnimation())
  // รอ animation ก่อนหน้าจบก่อน ค่อยเริ่ม animation ใหม่
).subscribe();
```

### 4.4 exhaustMap — ignore inner Observable ถ้า current ยังไม่เสร็จ

```typescript
import { exhaustMap } from 'rxjs/operators';

// ใช้เมื่อต้องการ prevent duplicate actions (login, form submit)
fromEvent(loginButton, 'click').pipe(
  exhaustMap(() => this.authService.login(credentials))
  // ถ้ากำลัง login อยู่ คลิกอีกก็ถูก ignore
).subscribe(user => this.handleLogin(user));

// ป้องกัน double submit
fromEvent(submitButton, 'click').pipe(
  exhaustMap(() => this.formService.submit(formData))
).subscribe(result => this.handleSuccess(result));
```

### 4.5 เปรียบเทียบ Higher-order Operators

```
Input:  --A--------B--------C-->
         |         |         |
         Inner:   Inner:   Inner:
         a1--a2   b1--b2   c1--c2

switchMap:   --a1--a2--b1--b2--c1--c2-->
             (ยกเลิก A เมื่อ B มา)
             ผลจริง: --a1--(ยกเลิก)--b1--b2--(ยกเลิก)--c1--c2-->

mergeMap:    --a1--a2--b1--b2--c1--c2-->  (อาจ interleave กัน)
             (รันทุกอันพร้อมกัน)

concatMap:   --a1--a2--b1--b2--c1--c2-->
             (รอ A จบก่อน ค่อยเริ่ม B)

exhaustMap:  --a1--a2-----------c1--c2-->
             (ignore B เพราะ A ยังไม่เสร็จ)
```

---

## 5. Combination Operators

### 5.1 combineLatest — รวม latest values จากทุก Observables

```typescript
import { combineLatest, BehaviorSubject } from 'rxjs';

const price$ = new BehaviorSubject<number>(100);
const quantity$ = new BehaviorSubject<number>(1);
const discount$ = new BehaviorSubject<number>(0);

// emit ทุกครั้งที่ค่าใดค่าหนึ่งเปลี่ยน
const total$ = combineLatest([price$, quantity$, discount$]).pipe(
  map(([price, quantity, discount]) => {
    const subtotal = price * quantity;
    return subtotal - (subtotal * discount / 100);
  })
);

total$.subscribe(total => console.log('Total:', total));
// Total: 100

price$.next(150);
// Total: 150

quantity$.next(3);
// Total: 450

discount$.next(10);
// Total: 405
```

### 5.2 forkJoin — รอให้ทุก Observables complete แล้วรวมผล

```typescript
import { forkJoin } from 'rxjs';

// เหมาะสำหรับ parallel API calls ที่ต้องรอทุกอันเสร็จ
forkJoin({
  users: this.http.get<User[]>('/api/users'),
  products: this.http.get<Product[]>('/api/products'),
  categories: this.http.get<Category[]>('/api/categories')
}).subscribe(({ users, products, categories }) => {
  console.log('Users:', users.length);
  console.log('Products:', products.length);
  console.log('Categories:', categories.length);
});

// หรือแบบ array
forkJoin([
  this.http.get<User>(`/api/users/${id}`),
  this.http.get<Order[]>(`/api/users/${id}/orders`)
]).subscribe(([user, orders]) => {
  this.user = user;
  this.orders = orders;
});
```

### 5.3 zip — จับคู่ values ตาม index

```typescript
import { zip, of } from 'rxjs';

const names$ = of('สมชาย', 'สมหญิง', 'สมศักดิ์');
const ages$ = of(25, 30, 35);
const cities$ = of('กรุงเทพ', 'เชียงใหม่', 'ขอนแก่น');

zip(names$, ages$, cities$).subscribe(([name, age, city]) => {
  console.log(`${name}, ${age} ปี, ${city}`);
});
// สมชาย, 25 ปี, กรุงเทพ
// สมหญิง, 30 ปี, เชียงใหม่
// สมศักดิ์, 35 ปี, ขอนแก่น
```

---

## 6. Workshop: Real-time Data Stream

```typescript
// src/app/components/realtime-dashboard/realtime-dashboard.component.ts
import { Component, OnInit, OnDestroy } from '@angular/core';
import { CommonModule } from '@angular/common';
import {
  Subject,
  BehaviorSubject,
  interval,
  combineLatest
} from 'rxjs';
import {
  map,
  scan,
  shareReplay,
  takeUntil,
  withLatestFrom,
  startWith
} from 'rxjs/operators';

interface StockPrice {
  symbol: string;
  price: number;
  change: number;
  changePercent: number;
}

interface DashboardState {
  stocks: StockPrice[];
  lastUpdate: Date;
  totalValue: number;
}

@Component({
  selector: 'app-realtime-dashboard',
  standalone: true,
  imports: [CommonModule],
  template: `
    <div class="dashboard">
      <div class="dashboard-header">
        <h2>Real-time Stock Dashboard</h2>
        <div class="controls">
          <button (click)="toggleStream()">
            {{ isStreaming ? '⏸ หยุด' : '▶ เริ่ม' }}
          </button>
          <span class="update-time">
            อัปเดตล่าสุด: {{ (state$ | async)?.lastUpdate | date:'HH:mm:ss' }}
          </span>
        </div>
      </div>

      <ng-container *ngIf="state$ | async as state">
        <div class="stocks-grid">
          <div
            *ngFor="let stock of state.stocks"
            class="stock-card"
            [class.positive]="stock.change > 0"
            [class.negative]="stock.change < 0"
          >
            <div class="stock-symbol">{{ stock.symbol }}</div>
            <div class="stock-price">{{ stock.price | number:'1.2-2' }}</div>
            <div class="stock-change">
              {{ stock.change > 0 ? '+' : '' }}{{ stock.change | number:'1.2-2' }}
              ({{ stock.changePercent | number:'1.2-2' }}%)
            </div>
          </div>
        </div>

        <div class="summary">
          <strong>มูลค่ารวม: {{ state.totalValue | number:'1.2-2' }} บาท</strong>
        </div>
      </ng-container>
    </div>
  `,
  styles: [`
    .dashboard { padding: 20px; }
    .dashboard-header {
      display: flex;
      justify-content: space-between;
      align-items: center;
      margin-bottom: 20px;
    }
    .controls { display: flex; align-items: center; gap: 12px; }
    .update-time { color: #666; font-size: 14px; }
    .stocks-grid {
      display: grid;
      grid-template-columns: repeat(auto-fill, minmax(150px, 1fr));
      gap: 12px;
      margin-bottom: 20px;
    }
    .stock-card {
      padding: 16px;
      border-radius: 8px;
      background: #f8f9fa;
      text-align: center;
    }
    .stock-card.positive { background: #d4edda; }
    .stock-card.negative { background: #f8d7da; }
    .stock-symbol { font-weight: bold; font-size: 18px; }
    .stock-price { font-size: 24px; margin: 4px 0; }
    .stock-change { font-size: 14px; color: #666; }
    .positive .stock-change { color: #28a745; }
    .negative .stock-change { color: #dc3545; }
  `]
})
export class RealtimeDashboardComponent implements OnInit, OnDestroy {

  private destroy$ = new Subject<void>();
  private streamActive$ = new BehaviorSubject<boolean>(true);
  isStreaming = true;

  // Initial stock data
  private initialStocks: StockPrice[] = [
    { symbol: 'ADVANC', price: 218.00, change: 0, changePercent: 0 },
    { symbol: 'AOT',    price: 65.75,  change: 0, changePercent: 0 },
    { symbol: 'CPALL',  price: 57.50,  change: 0, changePercent: 0 },
    { symbol: 'KBANK',  price: 135.00, change: 0, changePercent: 0 },
    { symbol: 'PTT',    price: 31.75,  change: 0, changePercent: 0 },
    { symbol: 'SCB',    price: 105.50, change: 0, changePercent: 0 }
  ];

  state$!: ReturnType<typeof this.createStateStream>;

  ngOnInit(): void {
    this.state$ = this.createStateStream();
  }

  private createStateStream() {
    // ราคาเริ่มต้น
    const initialPrices = this.initialStocks.reduce((acc, stock) => {
      acc[stock.symbol] = stock.price;
      return acc;
    }, {} as Record<string, number>);

    // Stream ของราคา
    const priceUpdates$ = interval(1500).pipe(
      withLatestFrom(this.streamActive$),
      map(([_, isActive]) => {
        if (!isActive) return null;
        // สุ่มเลือก stock และเปลี่ยนราคา
        const idx = Math.floor(Math.random() * this.initialStocks.length);
        const symbol = this.initialStocks[idx].symbol;
        const change = (Math.random() - 0.5) * 4;
        return { symbol, change };
      }),
      // accumulate state
      scan((prices: Record<string, number>, update) => {
        if (!update) return prices;
        return {
          ...prices,
          [update.symbol]: Math.max(1, prices[update.symbol] + update.change)
        };
      }, initialPrices),
      startWith(initialPrices),
      shareReplay(1)
    );

    return priceUpdates$.pipe(
      map(prices => {
        const stocks: StockPrice[] = this.initialStocks.map(s => {
          const currentPrice = prices[s.symbol];
          const change = currentPrice - s.price;
          return {
            symbol: s.symbol,
            price: currentPrice,
            change,
            changePercent: (change / s.price) * 100
          };
        });

        return {
          stocks,
          lastUpdate: new Date(),
          totalValue: stocks.reduce((sum, s) => sum + s.price * 100, 0)
        } as DashboardState;
      }),
      takeUntil(this.destroy$)
    );
  }

  toggleStream(): void {
    this.isStreaming = !this.isStreaming;
    this.streamActive$.next(this.isStreaming);
  }

  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

---

## สรุปบทที่ 22

| Subject Type | เก็บค่า | Replay ให้ subscriber ใหม่ |
|-------------|---------|--------------------------|
| Subject | ไม่เก็บ | ไม่ |
| BehaviorSubject | ค่าล่าสุด 1 ค่า | ใช่ (1 ค่า) |
| ReplaySubject | n ค่าล่าสุด | ใช่ (n ค่า) |
| AsyncSubject | ค่าสุดท้าย | ใช่ (เมื่อ complete) |

| Operator | พฤติกรรม | ใช้เมื่อ |
|----------|---------|---------|
| switchMap | ยกเลิก inner เก่า | Search, Navigation |
| mergeMap | รันขนาน | Parallel requests |
| concatMap | รัน sequential | Ordered operations |
| exhaustMap | Ignore ถ้ายังไม่เสร็จ | Prevent double-click |
| combineLatest | latest จากทุกตัว | Form calculations |
| forkJoin | รอทุกตัว complete | Parallel API calls |
| zip | จับคู่ตาม index | Paired streams |
