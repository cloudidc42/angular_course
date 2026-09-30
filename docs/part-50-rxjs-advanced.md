# Part 50: RxJS Advanced Operators และ Patterns

## บทนำ

RxJS เป็น library สำหรับ Reactive Programming ด้วย Observables ในบทนี้เราจะเรียนรู้ advanced operators, custom operators, error handling strategies และ patterns ที่ใช้บ่อยในการพัฒนา Angular

---

## 1. Higher-Order Observable Operators

### 1.1 switchMap - ยกเลิก request เก่าเมื่อมี request ใหม่

```typescript
// app/components/search/search.component.ts
import { Component, OnInit } from '@angular/core';
import { FormControl } from '@angular/forms';
import { HttpClient } from '@angular/common/http';
import { Observable, EMPTY } from 'rxjs';
import {
  debounceTime,
  distinctUntilChanged,
  switchMap,
  catchError,
  startWith,
  tap,
  map
} from 'rxjs/operators';

interface SearchResult {
  id: number;
  name: string;
  description: string;
}

@Component({
  selector: 'app-search',
  template: `
    <div class="search-container">
      <input
        [formControl]="searchControl"
        placeholder="ค้นหาสินค้า..."
        class="search-input"
      >

      <div *ngIf="isSearching" class="loading">กำลังค้นหา...</div>

      <ul class="results" *ngIf="results$ | async as results">
        <li *ngFor="let result of results">
          <strong>{{ result.name }}</strong>
          <p>{{ result.description }}</p>
        </li>
        <li *ngIf="results.length === 0 && !isSearching">
          ไม่พบผลการค้นหา
        </li>
      </ul>
    </div>
  `
})
export class SearchComponent implements OnInit {
  searchControl = new FormControl('');
  results$!: Observable<SearchResult[]>;
  isSearching = false;

  constructor(private http: HttpClient) {}

  ngOnInit() {
    this.results$ = this.searchControl.valueChanges.pipe(
      startWith(''),
      debounceTime(300),          // รอ 300ms หลังพิมพ์
      distinctUntilChanged(),      // ไม่ค้นหาถ้าข้อความเหมือนเดิม
      tap(() => this.isSearching = true),
      switchMap(term => {          // ยกเลิก request เก่าถ้ามี term ใหม่
        if (!term || term.length < 2) {
          this.isSearching = false;
          return EMPTY;
        }
        return this.http.get<SearchResult[]>(`/api/search?q=${term}`).pipe(
          catchError(() => {
            this.isSearching = false;
            return [];
          })
        );
      }),
      tap(() => this.isSearching = false)
    );
  }
}
```

### 1.2 mergeMap - รัน requests พร้อมกัน

```typescript
// รัน requests หลายอัน concurrently
import { from, mergeMap, toArray } from 'rxjs';
import { HttpClient } from '@angular/common/http';

function loadMultipleProducts(ids: number[], http: HttpClient) {
  return from(ids).pipe(
    mergeMap(id =>
      http.get(`/api/products/${id}`),
      3 // max 3 concurrent requests
    ),
    toArray() // รวมผลลัพธ์เป็น array
  );
}
```

### 1.3 concatMap - รัน requests ทีละอัน ตามลำดับ

```typescript
// อัปเดต items ตามลำดับ (ไม่ให้เกิด race condition)
import { from, concatMap } from 'rxjs';

function updateItemsSequentially(items: any[], http: HttpClient) {
  return from(items).pipe(
    concatMap(item =>
      http.put(`/api/items/${item.id}`, item)
    )
  );
}
```

### 1.4 exhaustMap - ไม่รับ request ใหม่จนกว่าจะเสร็จ

```typescript
// สำหรับ form submit - ป้องกัน double submit
import { fromEvent, exhaustMap } from 'rxjs';

// ใน component
const submitBtn = document.querySelector('#submit') as HTMLElement;

fromEvent(submitBtn, 'click').pipe(
  exhaustMap(() =>
    this.http.post('/api/save', this.form.value)
    // request ใหม่จะถูกละเว้นจนกว่า request นี้จะเสร็จ
  )
).subscribe();
```

---

## 2. Custom RxJS Operators

```typescript
// app/operators/custom.operators.ts
import {
  Observable,
  OperatorFunction,
  pipe,
  throwError,
  timer,
  Subject
} from 'rxjs';
import {
  map,
  tap,
  catchError,
  retry,
  retryWhen,
  mergeMap,
  take,
  finalize,
  scan,
  shareReplay,
  distinctUntilChanged,
  debounceTime
} from 'rxjs/operators';

// Operator: Log ทุก emission
export function debug<T>(label: string): OperatorFunction<T, T> {
  return (source: Observable<T>) => source.pipe(
    tap({
      next: value => console.log(`[${label}] Next:`, value),
      error: err => console.error(`[${label}] Error:`, err),
      complete: () => console.log(`[${label}] Complete`)
    })
  );
}

// Operator: Retry with exponential backoff
export function retryWithBackoff<T>(
  maxRetries = 3,
  initialDelay = 1000,
  scaleFactor = 2
): OperatorFunction<T, T> {
  return (source: Observable<T>) => source.pipe(
    retryWhen(errors =>
      errors.pipe(
        scan((retryCount, error) => {
          if (retryCount >= maxRetries) {
            throw error;
          }
          return retryCount + 1;
        }, 0),
        mergeMap(retryCount => {
          const delay = initialDelay * Math.pow(scaleFactor, retryCount - 1);
          console.log(`Retrying in ${delay}ms (attempt ${retryCount}/${maxRetries})`);
          return timer(delay);
        })
      )
    )
  );
}

// Operator: Loading state
export function withLoading<T>(
  loadingFn: (loading: boolean) => void
): OperatorFunction<T, T> {
  return (source: Observable<T>) => {
    loadingFn(true);
    return source.pipe(
      finalize(() => loadingFn(false))
    );
  };
}

// Operator: Cache ผลลัพธ์
export function cache<T>(ttlMs: number): OperatorFunction<T, T> {
  let cachedValue: T | undefined;
  let cacheTime = 0;

  return (source: Observable<T>) => new Observable<T>(observer => {
    const now = Date.now();
    if (cachedValue !== undefined && now - cacheTime < ttlMs) {
      observer.next(cachedValue);
      observer.complete();
      return;
    }

    return source.subscribe({
      next: value => {
        cachedValue = value;
        cacheTime = Date.now();
        observer.next(value);
      },
      error: err => observer.error(err),
      complete: () => observer.complete()
    });
  });
}

// Operator: Distinct changes (deep comparison)
export function distinctDeep<T>(): OperatorFunction<T, T> {
  return distinctUntilChanged((a, b) => JSON.stringify(a) === JSON.stringify(b));
}

// Operator: Map พร้อม error handling
export function safeMap<T, R>(
  mapFn: (value: T) => R,
  errorValue: R
): OperatorFunction<T, R> {
  return map(value => {
    try {
      return mapFn(value);
    } catch {
      return errorValue;
    }
  });
}

// Operator: บันทึก performance
export function measureTime<T>(label: string): OperatorFunction<T, T> {
  let startTime: number;
  return (source: Observable<T>) => source.pipe(
    tap({
      subscribe: () => startTime = performance.now(),
      finalize: () => {
        const elapsed = performance.now() - startTime;
        console.log(`[${label}] Completed in ${elapsed.toFixed(2)}ms`);
      }
    })
  );
}
```

---

## 3. Error Handling Strategies

```typescript
// app/services/api-with-error-handling.service.ts
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable, throwError, of, EMPTY } from 'rxjs';
import {
  catchError,
  retry,
  timeout,
  tap
} from 'rxjs/operators';
import { MatSnackBar } from '@angular/material/snack-bar';
import { Router } from '@angular/router';
import { retryWithBackoff } from '../operators/custom.operators';

@Injectable({ providedIn: 'root' })
export class ApiService {
  constructor(
    private http: HttpClient,
    private snackBar: MatSnackBar,
    private router: Router
  ) {}

  get<T>(url: string, options?: {
    retries?: number;
    timeoutMs?: number;
    defaultValue?: T;
    silentError?: boolean;
  }): Observable<T> {
    const {
      retries = 3,
      timeoutMs = 10000,
      defaultValue,
      silentError = false
    } = options || {};

    return this.http.get<T>(url).pipe(
      timeout(timeoutMs),
      retryWithBackoff(retries, 1000),
      catchError(error => this.handleError(error, defaultValue, silentError))
    );
  }

  private handleError<T>(
    error: any,
    defaultValue?: T,
    silent = false
  ): Observable<T> {
    console.error('API Error:', error);

    const message = this.getErrorMessage(error);

    if (!silent) {
      this.snackBar.open(message, 'ปิด', { duration: 5000 });
    }

    switch (error.status) {
      case 401:
        this.router.navigate(['/login']);
        return EMPTY;

      case 403:
        this.router.navigate(['/forbidden']);
        return EMPTY;

      case 404:
        return defaultValue !== undefined ? of(defaultValue) : throwError(() => error);

      case 0:
        return defaultValue !== undefined ? of(defaultValue) : throwError(() => error);

      default:
        return defaultValue !== undefined ? of(defaultValue) : throwError(() => error);
    }
  }

  private getErrorMessage(error: any): string {
    if (error.status === 0) return 'ไม่สามารถเชื่อมต่อ server ได้';
    if (error.status === 401) return 'กรุณาเข้าสู่ระบบใหม่';
    if (error.status === 403) return 'ไม่มีสิทธิ์เข้าถึง';
    if (error.status === 404) return 'ไม่พบข้อมูล';
    if (error.status >= 500) return 'เกิดข้อผิดพลาดที่ server';
    return error.error?.message || 'เกิดข้อผิดพลาด กรุณาลองใหม่';
  }
}
```

---

## 4. State Management ด้วย RxJS

```typescript
// app/store/cart.store.ts
import { Injectable, signal, computed } from '@angular/core';
import { BehaviorSubject, Observable } from 'rxjs';
import { distinctUntilChanged, map } from 'rxjs/operators';

interface CartItem {
  productId: number;
  name: string;
  price: number;
  quantity: number;
}

interface CartState {
  items: CartItem[];
  isLoading: boolean;
  error: string | null;
}

const initialState: CartState = {
  items: [],
  isLoading: false,
  error: null
};

@Injectable({ providedIn: 'root' })
export class CartStore {
  private state$ = new BehaviorSubject<CartState>(initialState);

  // Selectors
  items$ = this.state$.pipe(
    map(s => s.items),
    distinctUntilChanged()
  );

  totalItems$ = this.items$.pipe(
    map(items => items.reduce((sum, item) => sum + item.quantity, 0))
  );

  totalPrice$ = this.items$.pipe(
    map(items => items.reduce((sum, item) => sum + item.price * item.quantity, 0))
  );

  // Actions
  addItem(item: Omit<CartItem, 'quantity'>) {
    const current = this.state$.value;
    const existing = current.items.find(i => i.productId === item.productId);

    if (existing) {
      this.updateItem(item.productId, existing.quantity + 1);
    } else {
      this.setState({
        items: [...current.items, { ...item, quantity: 1 }]
      });
    }
  }

  removeItem(productId: number) {
    const items = this.state$.value.items.filter(i => i.productId !== productId);
    this.setState({ items });
  }

  updateItem(productId: number, quantity: number) {
    if (quantity <= 0) {
      this.removeItem(productId);
      return;
    }

    const items = this.state$.value.items.map(item =>
      item.productId === productId ? { ...item, quantity } : item
    );
    this.setState({ items });
  }

  clearCart() {
    this.setState({ items: [] });
  }

  private setState(partial: Partial<CartState>) {
    this.state$.next({ ...this.state$.value, ...partial });
  }
}
```

---

## 5. Advanced Patterns

### 5.1 Poll Pattern

```typescript
// app/services/polling.service.ts
import { Injectable } from '@angular/core';
import { Observable, timer, Subject } from 'rxjs';
import { switchMap, takeUntil, share, tap } from 'rxjs/operators';
import { HttpClient } from '@angular/common/http';

@Injectable({ providedIn: 'root' })
export class PollingService {
  constructor(private http: HttpClient) {}

  // Poll API ทุก N milliseconds
  poll<T>(
    url: string,
    intervalMs: number,
    stop$: Observable<any>
  ): Observable<T> {
    return timer(0, intervalMs).pipe(
      switchMap(() => this.http.get<T>(url)),
      takeUntil(stop$),
      share() // share กับ multiple subscribers
    );
  }
}
```

```typescript
// การใช้งาน
@Component({
  template: `
    <div>
      <h3>Order Status</h3>
      <p>{{ (orderStatus$ | async)?.status }}</p>
      <button (click)="stopPolling()">หยุด</button>
    </div>
  `
})
export class OrderStatusComponent implements OnDestroy {
  private stopPolling$ = new Subject<void>();

  orderStatus$ = this.pollingService.poll<any>(
    '/api/orders/123/status',
    5000,
    this.stopPolling$
  );

  constructor(private pollingService: PollingService) {}

  stopPolling() {
    this.stopPolling$.next();
    this.stopPolling$.complete();
  }

  ngOnDestroy() {
    this.stopPolling$.next();
    this.stopPolling$.complete();
  }
}
```

### 5.2 WebSocket Observable

```typescript
// app/services/websocket.service.ts
import { Injectable } from '@angular/core';
import { Observable, Subject, EMPTY, timer } from 'rxjs';
import { webSocket, WebSocketSubject } from 'rxjs/webSocket';
import {
  retryWhen,
  delayWhen,
  tap,
  catchError,
  switchMap
} from 'rxjs/operators';

@Injectable({ providedIn: 'root' })
export class WebSocketService {
  private socket$?: WebSocketSubject<any>;
  private reconnectAttempts = 0;
  private maxReconnectAttempts = 5;

  connect(url: string): Observable<any> {
    if (!this.socket$ || this.socket$.closed) {
      this.socket$ = webSocket({
        url,
        openObserver: {
          next: () => {
            console.log('WebSocket connected');
            this.reconnectAttempts = 0;
          }
        },
        closeObserver: {
          next: () => console.log('WebSocket disconnected')
        }
      });
    }

    return this.socket$.pipe(
      retryWhen(errors =>
        errors.pipe(
          tap(() => {
            this.reconnectAttempts++;
            console.log(`Reconnecting (${this.reconnectAttempts}/${this.maxReconnectAttempts})`);
          }),
          delayWhen(() => timer(Math.min(1000 * this.reconnectAttempts, 30000))),
          switchMap(() => {
            if (this.reconnectAttempts > this.maxReconnectAttempts) {
              throw new Error('Max reconnect attempts reached');
            }
            return EMPTY;
          })
        )
      )
    );
  }

  send(message: any) {
    this.socket$?.next(message);
  }

  disconnect() {
    this.socket$?.complete();
  }
}
```

---

## 6. RxJS Testing

```typescript
// app/services/data.service.spec.ts
import { TestBed } from '@angular/core/testing';
import { TestScheduler } from 'rxjs/testing';
import { cold, hot } from 'jasmine-marbles';
import { DataService } from './data.service';

describe('DataService', () => {
  let scheduler: TestScheduler;

  beforeEach(() => {
    scheduler = new TestScheduler((actual, expected) => {
      expect(actual).toEqual(expected);
    });
  });

  it('should debounce search', () => {
    scheduler.run(({ cold, expectObservable }) => {
      // a = search "a", b = search "ab", c = search "abc"
      const input =  cold('--a-b--c--|', { a: 'a', b: 'ab', c: 'abc' });
      const expected =    '------c---|';

      const result = input.pipe(debounceTime(50, scheduler));
      expectObservable(result).toBe(expected, { c: 'abc' });
    });
  });

  it('should retry on error', () => {
    scheduler.run(({ cold, expectObservable }) => {
      const source = cold('#', {}, new Error('Network error'));
      const expected = '---#';  // retry 3 times then fail

      const result = source.pipe(
        retry(3),
      );
      expectObservable(result).toBe(expected, {}, new Error('Network error'));
    });
  });
});
```

---

## สรุป

| Operator | ใช้เมื่อ |
|---------|---------|
| switchMap | ค้นหา, navigation (ยกเลิกของเก่า) |
| mergeMap | parallel requests |
| concatMap | sequential, ordered |
| exhaustMap | form submit (ป้องกัน double) |
| retryWhen | retry พร้อม backoff |
| shareReplay | share multicasted + replay |
| combineLatest | รอทั้งหมด แล้ว combine |
| forkJoin | รันพร้อมกัน รอทุกอันเสร็จ |
| zip | pair emissions |

RxJS ที่ดีควร:
1. **Unsubscribe** เสมอ (ใช้ `takeUntil` หรือ `async pipe`)
2. **Handle errors** อย่างเหมาะสม
3. **Avoid nested subscriptions** - ใช้ higher-order operators แทน
4. **Prefer declarative** style ด้วย composition
