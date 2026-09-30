# Part 21 — RxJS พื้นฐาน

## บทนำ

RxJS (Reactive Extensions for JavaScript) คือ library สำหรับ reactive programming โดยใช้ Observables เพื่อจัดการกับ asynchronous data streams และ event-based programs

### ทำไมต้องใช้ RxJS?
- จัดการ async operations ที่ซับซ้อนได้ง่าย
- รองรับ cancellation (ยกเลิก request ได้)
- Composable — รวม streams เข้าด้วยกันได้
- Angular ใช้ RxJS เป็นส่วนหลัก (HttpClient, Router, Forms)

---

## 1. Observable, Observer, Subscription

### 1.1 Observable

Observable คือ "stream of data" ที่ emit values ออกมาตามเวลา

```typescript
import { Observable } from 'rxjs';

// สร้าง Observable แบบง่าย
const myObservable = new Observable<number>(subscriber => {
  // subscriber.next() — emit ค่า
  subscriber.next(1);
  subscriber.next(2);
  subscriber.next(3);

  // subscriber.error() — emit error
  // subscriber.error(new Error('Something went wrong'));

  // subscriber.complete() — บอกว่า stream จบแล้ว
  subscriber.complete();
});
```

### 1.2 Observer

Observer คือ object ที่มี methods สำหรับรับค่าจาก Observable

```typescript
import { Observer } from 'rxjs';

const myObserver: Observer<number> = {
  next: (value) => console.log('รับค่า:', value),
  error: (err) => console.error('เกิด error:', err),
  complete: () => console.log('Stream จบแล้ว')
};
```

### 1.3 Subscription

Subscription คือ object ที่แทนการ subscribe Observable เราสามารถ unsubscribe เพื่อหยุดรับข้อมูลได้

```typescript
import { Observable } from 'rxjs';

const myObservable = new Observable<number>(subscriber => {
  let count = 0;
  const interval = setInterval(() => {
    subscriber.next(count++);
  }, 1000);

  // Teardown logic — จะรันเมื่อ unsubscribe
  return () => {
    clearInterval(interval);
    console.log('Cleanup!');
  };
});

// Subscribe
const subscription = myObservable.subscribe({
  next: value => console.log(value),
  error: err => console.error(err),
  complete: () => console.log('Done')
});

// ยกเลิกหลังจาก 5 วินาที
setTimeout(() => {
  subscription.unsubscribe();
  console.log('Unsubscribed!');
}, 5000);
```

### 1.4 การจัดการ Multiple Subscriptions

```typescript
import { Subscription } from 'rxjs';

// ใช้ Subscription เพื่อรวม subscriptions
const allSubscriptions = new Subscription();

allSubscriptions.add(
  observable1.subscribe(val => console.log('Stream 1:', val))
);
allSubscriptions.add(
  observable2.subscribe(val => console.log('Stream 2:', val))
);

// unsubscribe ทั้งหมดพร้อมกัน
allSubscriptions.unsubscribe();
```

---

## 2. Creation Operators

### 2.1 `of` — สร้าง Observable จากค่าที่กำหนด

```typescript
import { of } from 'rxjs';

// emit ค่าทีละตัวแล้ว complete
const numbers$ = of(1, 2, 3, 4, 5);
numbers$.subscribe(val => console.log(val));
// Output: 1, 2, 3, 4, 5, complete

// emit objects
const user$ = of({ id: 1, name: 'สมชาย' }, { id: 2, name: 'สมหญิง' });
user$.subscribe(user => console.log(user));

// มีประโยชน์ใน services
getUser(): Observable<User> {
  // ถ้า cache มีข้อมูล ส่งกลับจาก cache
  if (this.cachedUser) {
    return of(this.cachedUser);
  }
  return this.http.get<User>('/api/user');
}
```

### 2.2 `from` — สร้าง Observable จาก iterable หรือ Promise

```typescript
import { from } from 'rxjs';

// จาก Array
const fromArray$ = from([1, 2, 3, 4, 5]);
fromArray$.subscribe(val => console.log(val));
// emit ทีละตัว: 1, 2, 3, 4, 5

// จาก Promise
const promise = fetch('/api/data').then(res => res.json());
const fromPromise$ = from(promise);
fromPromise$.subscribe(data => console.log(data));

// จาก String (iterate แต่ละตัวอักษร)
const fromString$ = from('Hello');
fromString$.subscribe(char => console.log(char));
// H, e, l, l, o

// จาก Map
const map = new Map([['key1', 'val1'], ['key2', 'val2']]);
const fromMap$ = from(map);
fromMap$.subscribe(entry => console.log(entry));
// ['key1', 'val1'], ['key2', 'val2']

// จาก Generator
function* fibonacci() {
  let a = 0, b = 1;
  while (true) {
    yield a;
    [a, b] = [b, a + b];
  }
}
const fibonacci$ = from(fibonacci());
```

### 2.3 `interval` — emit ค่าตามเวลาที่กำหนด

```typescript
import { interval } from 'rxjs';
import { take } from 'rxjs/operators';

// emit 0, 1, 2, 3, ... ทุก 1 วินาที
const everySecond$ = interval(1000);
const sub = everySecond$.subscribe(val => console.log(val));

// ใช้ take เพื่อจำกัดจำนวน
interval(1000)
  .pipe(take(5))
  .subscribe(val => console.log(val));
// 0, 1, 2, 3, 4, complete

// ใช้สร้าง countdown timer
const countdown$ = interval(1000).pipe(take(10));
countdown$.subscribe({
  next: val => console.log(`${9 - val} วินาที`),
  complete: () => console.log('หมดเวลา!')
});
```

### 2.4 `timer` — emit ค่าหลังจาก delay

```typescript
import { timer } from 'rxjs';

// emit หลัง 3 วินาที แล้ว complete
const delayed$ = timer(3000);
delayed$.subscribe(val => console.log('emit:', val)); // emit: 0

// emit หลัง 1 วินาที แล้ว emit ทุก 500ms
const delayedInterval$ = timer(1000, 500);
delayedInterval$
  .pipe(take(5))
  .subscribe(val => console.log(val));
// (รอ 1 วินาที) 0, 1, 2, 3, 4
```

### 2.5 `fromEvent` — สร้าง Observable จาก DOM events

```typescript
import { fromEvent } from 'rxjs';
import { map, debounceTime } from 'rxjs/operators';

// ฟัง click events
const click$ = fromEvent<MouseEvent>(document, 'click');
click$.subscribe(event => {
  console.log(`คลิกที่ (${event.clientX}, ${event.clientY})`);
});

// ฟัง input events
const inputEl = document.querySelector('input')!;
const input$ = fromEvent<InputEvent>(inputEl, 'input');

input$.pipe(
  debounceTime(300),
  map(event => (event.target as HTMLInputElement).value)
).subscribe(value => console.log('Search:', value));

// ฟัง keyboard events
const keydown$ = fromEvent<KeyboardEvent>(document, 'keydown');
keydown$.pipe(
  map(event => event.key)
).subscribe(key => console.log('Key pressed:', key));

// ฟัง scroll events
const scroll$ = fromEvent(window, 'scroll');
scroll$.subscribe(() => {
  console.log('Scroll Y:', window.scrollY);
});
```

### 2.6 `ajax` — HTTP requests

```typescript
import { ajax } from 'rxjs/ajax';
import { map, catchError } from 'rxjs/operators';
import { of } from 'rxjs';

// GET request
const users$ = ajax.getJSON<User[]>('https://api.example.com/users');
users$.subscribe(users => console.log(users));

// POST request
const createUser$ = ajax({
  url: 'https://api.example.com/users',
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: { name: 'สมชาย', email: 'somchai@example.com' }
});

createUser$.pipe(
  map(response => response.response),
  catchError(error => {
    console.error('Error:', error);
    return of(null);
  })
).subscribe(user => console.log('สร้าง user:', user));
```

---

## 3. Pipeable Operators

### 3.1 `map` — แปลงค่า

```typescript
import { of } from 'rxjs';
import { map } from 'rxjs/operators';

// แปลงตัวเลข
of(1, 2, 3, 4, 5)
  .pipe(map(x => x * 2))
  .subscribe(val => console.log(val));
// 2, 4, 6, 8, 10

// แปลง object
const users$ = of(
  { id: 1, firstName: 'สมชาย', lastName: 'ใจดี' },
  { id: 2, firstName: 'สมหญิง', lastName: 'รักดี' }
);

users$.pipe(
  map(user => ({
    ...user,
    fullName: `${user.firstName} ${user.lastName}`
  }))
).subscribe(user => console.log(user.fullName));

// map กับ HTTP response
this.http.get<ApiResponse<User[]>>('/api/users').pipe(
  map(response => response.data)   // ดึงเฉพาะ data
).subscribe(users => this.users = users);
```

### 3.2 `filter` — กรองค่า

```typescript
import { from } from 'rxjs';
import { filter } from 'rxjs/operators';

// กรองเฉพาะเลขคู่
from([1, 2, 3, 4, 5, 6, 7, 8, 9, 10])
  .pipe(filter(n => n % 2 === 0))
  .subscribe(val => console.log(val));
// 2, 4, 6, 8, 10

// กรอง objects
interface Product {
  id: number;
  name: string;
  price: number;
  inStock: boolean;
}

const products$ = from<Product[]>([
  { id: 1, name: 'สินค้า A', price: 100, inStock: true },
  { id: 2, name: 'สินค้า B', price: 200, inStock: false },
  { id: 3, name: 'สินค้า C', price: 300, inStock: true }
]);

products$.pipe(
  filter(product => product.inStock),
  filter(product => product.price > 150)
).subscribe(product => console.log(product.name));
// สินค้า C
```

### 3.3 `take` — รับค่าตามจำนวนที่กำหนด

```typescript
import { interval } from 'rxjs';
import { take, takeLast, takeUntil, takeWhile } from 'rxjs/operators';
import { Subject } from 'rxjs';

// take(n) — รับแค่ n ค่าแรก
interval(1000).pipe(take(5)).subscribe(val => console.log(val));
// 0, 1, 2, 3, 4, complete

// takeLast(n) — รับ n ค่าสุดท้าย (ต้อง complete ก่อน)
from([1, 2, 3, 4, 5]).pipe(takeLast(2)).subscribe(val => console.log(val));
// 4, 5

// takeWhile — รับค่าจนกว่า condition จะ false
from([1, 2, 3, 4, 5, 6]).pipe(
  takeWhile(val => val < 4)
).subscribe(val => console.log(val));
// 1, 2, 3

// takeUntil — รับค่าจนกว่า notifier Observable จะ emit
const stop$ = new Subject<void>();
interval(1000).pipe(
  takeUntil(stop$)
).subscribe(val => console.log(val));

setTimeout(() => stop$.next(), 3500); // หยุดหลัง 3.5 วินาที
// 0, 1, 2, 3
```

### 3.4 `tap` — side effects โดยไม่เปลี่ยนค่า

```typescript
import { of } from 'rxjs';
import { tap, map } from 'rxjs/operators';

of(1, 2, 3).pipe(
  tap(val => console.log('Before map:', val)),    // side effect
  map(val => val * 10),
  tap(val => console.log('After map:', val))      // side effect
).subscribe(val => console.log('Final:', val));

// ใช้ tap สำหรับ logging
this.http.get<User[]>('/api/users').pipe(
  tap(users => console.log(`โหลด ${users.length} users`)),
  tap(users => this.analytics.track('users_loaded', { count: users.length }))
).subscribe(users => this.users = users);
```

### 3.5 `catchError` — จัดการ errors

```typescript
import { catchError, throwError } from 'rxjs';
import { HttpErrorResponse } from '@angular/common/http';

this.http.get<User[]>('/api/users').pipe(
  catchError((error: HttpErrorResponse) => {
    // Log error
    console.error('HTTP Error:', error.status, error.message);

    // Return fallback value
    if (error.status === 404) {
      return of([]);  // ส่ง empty array แทน
    }

    if (error.status === 401) {
      this.router.navigate(['/login']);
      return of([]);
    }

    // Re-throw error ที่จัดการไม่ได้
    return throwError(() => error);
  })
).subscribe(users => this.users = users);
```

---

## 4. Marble Diagrams

Marble Diagram เป็นวิธีแสดง Observable streams แบบ visual

```
ตัวอย่าง Marble Diagram notation:
------1------2------3------|>  source$
--map(x => x * 2)
------2------4------6------|>  result$

สัญลักษณ์:
-  = เวลาผ่านไป (1 tick)
1  = emit ค่า 1
|  = complete
X  = error
>  = ongoing (ยังไม่ complete)
```

```
filter(x => x > 2):
--1--2--3--4--5--|>  source$
-----------3--4--5--|>  filtered$

take(3):
--1--2--3--4--5--|>  source$
--1--2--3|        result$   (complete หลัง 3 ค่า)

map(x => x * 10):
--1--2--3--|>     source$
--10-20-30-|>     result$
```

---

## 5. Workshop: Live Search

```typescript
// src/app/components/live-search/live-search.component.ts
import { Component, OnInit, OnDestroy } from '@angular/core';
import { CommonModule } from '@angular/common';
import { ReactiveFormsModule, FormControl } from '@angular/forms';
import { HttpClient } from '@angular/common/http';
import {
  Subject,
  Observable,
  of,
  combineLatest
} from 'rxjs';
import {
  debounceTime,
  distinctUntilChanged,
  switchMap,
  catchError,
  tap,
  startWith,
  map,
  takeUntil
} from 'rxjs/operators';

interface SearchResult {
  id: number;
  title: string;
  description: string;
  category: string;
}

@Component({
  selector: 'app-live-search',
  standalone: true,
  imports: [CommonModule, ReactiveFormsModule],
  template: `
    <div class="search-container">
      <div class="search-box">
        <input
          [formControl]="searchControl"
          type="text"
          placeholder="ค้นหา..."
          class="form-control"
        />
        <span *ngIf="isLoading" class="loading-indicator">⏳</span>
        <button
          *ngIf="searchControl.value"
          (click)="clearSearch()"
          class="clear-btn"
        >✕</button>
      </div>

      <!-- Stats -->
      <div *ngIf="searchTerm" class="search-stats">
        <small>
          ผลการค้นหา "{{ searchTerm }}":
          {{ (results$ | async)?.length || 0 }} รายการ
        </small>
      </div>

      <!-- Results -->
      <div class="results-container">
        <ng-container *ngIf="results$ | async as results">
          <div *ngIf="results.length === 0 && searchTerm && !isLoading" class="no-results">
            ไม่พบผลการค้นหาสำหรับ "{{ searchTerm }}"
          </div>

          <div
            *ngFor="let item of results"
            class="result-item"
          >
            <div class="result-header">
              <h4>{{ item.title }}</h4>
              <span class="badge">{{ item.category }}</span>
            </div>
            <p>{{ item.description }}</p>
          </div>
        </ng-container>

        <!-- Loading State -->
        <div *ngIf="isLoading" class="loading-state">
          <div *ngFor="let i of [1,2,3]" class="skeleton-item">
            <div class="skeleton-title"></div>
            <div class="skeleton-text"></div>
          </div>
        </div>
      </div>

      <!-- Error State -->
      <div *ngIf="errorMessage" class="error-state">
        {{ errorMessage }}
      </div>
    </div>
  `,
  styles: [`
    .search-container { max-width: 600px; margin: 0 auto; }
    .search-box {
      position: relative;
      display: flex;
      align-items: center;
      margin-bottom: 8px;
    }
    .form-control { padding-right: 60px; }
    .loading-indicator {
      position: absolute;
      right: 36px;
    }
    .clear-btn {
      position: absolute;
      right: 8px;
      background: none;
      border: none;
      cursor: pointer;
      color: #999;
    }
    .result-item {
      padding: 12px;
      border: 1px solid #eee;
      border-radius: 4px;
      margin-bottom: 8px;
    }
    .result-header {
      display: flex;
      justify-content: space-between;
      align-items: center;
    }
    .badge {
      background: #007bff;
      color: white;
      padding: 2px 8px;
      border-radius: 12px;
      font-size: 12px;
    }
    .skeleton-item { margin-bottom: 12px; }
    .skeleton-title, .skeleton-text {
      background: linear-gradient(90deg, #f0f0f0 25%, #e0e0e0 50%, #f0f0f0 75%);
      background-size: 200% 100%;
      animation: shimmer 1.5s infinite;
      border-radius: 4px;
    }
    .skeleton-title { height: 20px; width: 60%; margin-bottom: 8px; }
    .skeleton-text { height: 14px; width: 90%; }
    @keyframes shimmer {
      0% { background-position: 200% 0; }
      100% { background-position: -200% 0; }
    }
  `]
})
export class LiveSearchComponent implements OnInit, OnDestroy {

  searchControl = new FormControl('');
  results$!: Observable<SearchResult[]>;
  isLoading = false;
  errorMessage = '';
  searchTerm = '';

  private destroy$ = new Subject<void>();

  // Mock data สำหรับ demo
  private mockData: SearchResult[] = [
    { id: 1, title: 'Angular Components', description: 'เรียนรู้การสร้าง Components ใน Angular', category: 'Framework' },
    { id: 2, title: 'RxJS Observables', description: 'ทำความเข้าใจ Observables และ Subjects', category: 'RxJS' },
    { id: 3, title: 'TypeScript Basics', description: 'พื้นฐาน TypeScript สำหรับ Angular', category: 'TypeScript' },
    { id: 4, title: 'Angular Services', description: 'การสร้างและใช้งาน Services', category: 'Framework' },
    { id: 5, title: 'HTTP Client', description: 'การเรียก API ด้วย HttpClient', category: 'HTTP' },
    { id: 6, title: 'Angular Router', description: 'การจัดการ Routes ใน Angular', category: 'Routing' },
    { id: 7, title: 'Reactive Forms', description: 'Reactive Forms และ Validation', category: 'Forms' },
    { id: 8, title: 'Angular Animations', description: 'การสร้าง Animations ใน Angular', category: 'UI' }
  ];

  ngOnInit(): void {
    this.results$ = this.searchControl.valueChanges.pipe(
      startWith(''),                          // เริ่มต้นด้วย empty string
      debounceTime(300),                      // รอ 300ms หลัง user หยุดพิมพ์
      distinctUntilChanged(),                 // ไม่ search ถ้าค่าเหมือนเดิม
      tap(term => {
        this.searchTerm = term || '';
        this.isLoading = !!term;
        this.errorMessage = '';
      }),
      switchMap(term => {
        if (!term || term.trim().length < 2) {
          this.isLoading = false;
          return of([]);
        }

        return this.searchItems(term).pipe(
          tap(() => this.isLoading = false),
          catchError(error => {
            console.error('Search error:', error);
            this.isLoading = false;
            this.errorMessage = 'เกิดข้อผิดพลาดในการค้นหา';
            return of([]);
          })
        );
      }),
      takeUntil(this.destroy$)
    );
  }

  private searchItems(term: string): Observable<SearchResult[]> {
    // Mock API call พร้อม delay
    return new Observable(subscriber => {
      const timeout = setTimeout(() => {
        const results = this.mockData.filter(item =>
          item.title.toLowerCase().includes(term.toLowerCase()) ||
          item.description.toLowerCase().includes(term.toLowerCase()) ||
          item.category.toLowerCase().includes(term.toLowerCase())
        );
        subscriber.next(results);
        subscriber.complete();
      }, 500);

      return () => clearTimeout(timeout);
    });
  }

  clearSearch(): void {
    this.searchControl.setValue('');
    this.searchTerm = '';
  }

  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

---

## สรุปบทที่ 21

| แนวคิด | คำอธิบาย |
|--------|----------|
| Observable | Stream ของ data ที่ emit values ตามเวลา |
| Observer | Object ที่รับค่าจาก Observable |
| Subscription | แทนการ subscribe สามารถ unsubscribe ได้ |
| `of` | สร้าง Observable จากค่าที่กำหนด |
| `from` | สร้าง Observable จาก iterable/Promise |
| `interval` | emit ค่าตามเวลาที่กำหนด |
| `timer` | emit ค่าหลัง delay |
| `fromEvent` | สร้าง Observable จาก DOM events |
| `map` | แปลงค่าแต่ละตัว |
| `filter` | กรองค่าตาม condition |
| `take` | จำกัดจำนวน emissions |
| `tap` | side effects |
| `catchError` | จัดการ errors |
| `debounceTime` | รอจน user หยุด input |
| `distinctUntilChanged` | ไม่ emit ถ้าค่าเหมือนเดิม |
| `switchMap` | เปลี่ยน inner Observable |

### ข้อควรระวัง

1. **อย่าลืม unsubscribe** — ใช้ `takeUntil(destroy$)` หรือ `async pipe`
2. **ใช้ `switchMap` กับ search** — ยกเลิก request เก่าเมื่อมี request ใหม่
3. **ระวัง memory leaks** — เมื่อ subscribe ใน component ต้องจัดการ lifecycle
