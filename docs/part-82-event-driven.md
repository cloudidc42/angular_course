# Part 82: Event-Driven Architecture ใน Angular

## Event-Driven คืออะไร

สถาปัตยกรรมที่ components สื่อสารกันผ่าน events แทนการเรียกหากันโดยตรง ทำให้ decoupled

---

## 1. Event Bus Service

```typescript
// core/event-bus/event-bus.service.ts
import { Injectable } from '@angular/core';
import { Subject, Observable, filter, map } from 'rxjs';

export interface AppEvent<T = any> {
  type: string;
  payload: T;
  timestamp: Date;
  source?: string;
  correlationId?: string;
}

export type EventHandler<T = any> = (event: AppEvent<T>) => void;

@Injectable({ providedIn: 'root' })
export class EventBusService {
  private eventSubject = new Subject<AppEvent>();
  private events$ = this.eventSubject.asObservable();

  // Publish event
  publish<T>(event: Omit<AppEvent<T>, 'timestamp'>): void {
    const fullEvent: AppEvent<T> = {
      ...event,
      timestamp: new Date(),
      correlationId: event.correlationId || this.generateId()
    };
    
    console.debug(`[EventBus] ${fullEvent.type}`, fullEvent.payload);
    this.eventSubject.next(fullEvent);
  }

  // Subscribe ทุก events
  on<T>(eventType: string): Observable<AppEvent<T>> {
    return this.events$.pipe(
      filter(event => event.type === eventType),
      map(event => event as AppEvent<T>)
    );
  }

  // Subscribe หลาย event types
  onMany<T>(...eventTypes: string[]): Observable<AppEvent<T>> {
    return this.events$.pipe(
      filter(event => eventTypes.includes(event.type)),
      map(event => event as AppEvent<T>)
    );
  }

  // Subscribe ทุก events (สำหรับ logging)
  onAll(): Observable<AppEvent> {
    return this.events$.asObservable();
  }

  private generateId(): string {
    return `${Date.now()}-${Math.random().toString(36).slice(2, 9)}`;
  }
}
```

---

## 2. Domain Events

```typescript
// core/event-bus/domain-events.ts

// ประกาศ event types ทั้งหมด
export const DomainEvents = {
  // User Events
  USER_LOGGED_IN: 'USER_LOGGED_IN',
  USER_LOGGED_OUT: 'USER_LOGGED_OUT',
  USER_PROFILE_UPDATED: 'USER_PROFILE_UPDATED',
  
  // Product Events
  PRODUCT_CREATED: 'PRODUCT_CREATED',
  PRODUCT_UPDATED: 'PRODUCT_UPDATED',
  PRODUCT_DELETED: 'PRODUCT_DELETED',
  PRODUCT_VIEWED: 'PRODUCT_VIEWED',
  
  // Cart Events
  ITEM_ADDED_TO_CART: 'ITEM_ADDED_TO_CART',
  ITEM_REMOVED_FROM_CART: 'ITEM_REMOVED_FROM_CART',
  CART_CLEARED: 'CART_CLEARED',
  
  // Order Events
  ORDER_PLACED: 'ORDER_PLACED',
  ORDER_CONFIRMED: 'ORDER_CONFIRMED',
  ORDER_SHIPPED: 'ORDER_SHIPPED',
  ORDER_DELIVERED: 'ORDER_DELIVERED',
  ORDER_CANCELLED: 'ORDER_CANCELLED',
  
  // Payment Events
  PAYMENT_INITIATED: 'PAYMENT_INITIATED',
  PAYMENT_SUCCESS: 'PAYMENT_SUCCESS',
  PAYMENT_FAILED: 'PAYMENT_FAILED',
  
  // Notification Events
  SHOW_TOAST: 'SHOW_TOAST',
  SHOW_MODAL: 'SHOW_MODAL'
} as const;

export type DomainEventType = typeof DomainEvents[keyof typeof DomainEvents];

// Payload types
export interface UserLoggedInPayload {
  userId: string;
  username: string;
  roles: string[];
}

export interface ItemAddedToCartPayload {
  productId: string;
  productName: string;
  quantity: number;
  price: number;
}

export interface OrderPlacedPayload {
  orderId: string;
  items: { productId: string; quantity: number; price: number }[];
  total: number;
  userId: string;
}

export interface ToastPayload {
  message: string;
  type: 'success' | 'error' | 'warning' | 'info';
  duration?: number;
}
```

---

## 3. Cart Service ที่ใช้ Event Bus

```typescript
// features/cart/cart.service.ts
import { Injectable } from '@angular/core';
import { BehaviorSubject } from 'rxjs';
import { EventBusService } from '../../core/event-bus/event-bus.service';
import { DomainEvents } from '../../core/event-bus/domain-events';

interface CartItem {
  productId: string;
  name: string;
  price: number;
  quantity: number;
  imageUrl?: string;
}

@Injectable({ providedIn: 'root' })
export class CartService {
  private items$ = new BehaviorSubject<CartItem[]>([]);
  
  cart$ = this.items$.asObservable();

  constructor(private eventBus: EventBusService) {}

  addItem(product: { id: string; name: string; price: number }, quantity = 1): void {
    const current = this.items$.value;
    const existingIndex = current.findIndex(i => i.productId === product.id);

    let updated: CartItem[];
    
    if (existingIndex >= 0) {
      updated = current.map((item, i) => 
        i === existingIndex 
          ? { ...item, quantity: item.quantity + quantity }
          : item
      );
    } else {
      updated = [...current, {
        productId: product.id,
        name: product.name,
        price: product.price,
        quantity
      }];
    }

    this.items$.next(updated);
    
    // Publish event - ส่วนอื่นของ app รับได้
    this.eventBus.publish({
      type: DomainEvents.ITEM_ADDED_TO_CART,
      payload: {
        productId: product.id,
        productName: product.name,
        quantity,
        price: product.price
      },
      source: 'CartService'
    });

    // แจ้ง toast
    this.eventBus.publish({
      type: DomainEvents.SHOW_TOAST,
      payload: {
        message: `เพิ่ม "${product.name}" ลงตะกร้าแล้ว`,
        type: 'success',
        duration: 3000
      }
    });
  }

  removeItem(productId: string): void {
    const item = this.items$.value.find(i => i.productId === productId);
    if (!item) return;

    this.items$.next(this.items$.value.filter(i => i.productId !== productId));
    
    this.eventBus.publish({
      type: DomainEvents.ITEM_REMOVED_FROM_CART,
      payload: { productId, productName: item.name }
    });
  }

  clear(): void {
    this.items$.next([]);
    this.eventBus.publish({
      type: DomainEvents.CART_CLEARED,
      payload: {}
    });
  }

  get total(): number {
    return this.items$.value.reduce((sum, item) => sum + (item.price * item.quantity), 0);
  }
}
```

---

## 4. Event Handlers (Subscribers)

```typescript
// core/event-handlers/analytics.handler.ts
import { Injectable, OnDestroy } from '@angular/core';
import { Subject } from 'rxjs';
import { takeUntil } from 'rxjs/operators';
import { EventBusService } from '../event-bus/event-bus.service';
import { DomainEvents } from '../event-bus/domain-events';

@Injectable({ providedIn: 'root' })
export class AnalyticsEventHandler implements OnDestroy {
  private destroy$ = new Subject<void>();

  constructor(
    private eventBus: EventBusService
  ) {
    this.setupHandlers();
  }

  private setupHandlers(): void {
    // Track product views
    this.eventBus.on(DomainEvents.PRODUCT_VIEWED)
      .pipe(takeUntil(this.destroy$))
      .subscribe(event => {
        this.track('product_view', {
          product_id: event.payload.productId,
          product_name: event.payload.name,
          category: event.payload.category
        });
      });

    // Track add to cart
    this.eventBus.on(DomainEvents.ITEM_ADDED_TO_CART)
      .pipe(takeUntil(this.destroy$))
      .subscribe(event => {
        this.track('add_to_cart', {
          product_id: event.payload.productId,
          value: event.payload.price * event.payload.quantity
        });
      });

    // Track purchases
    this.eventBus.on(DomainEvents.ORDER_PLACED)
      .pipe(takeUntil(this.destroy$))
      .subscribe(event => {
        this.track('purchase', {
          transaction_id: event.payload.orderId,
          value: event.payload.total,
          items: event.payload.items
        });
      });
  }

  private track(event: string, data: Record<string, any>): void {
    // Google Analytics 4
    if (typeof gtag !== 'undefined') {
      gtag('event', event, data);
    }
    console.log(`[Analytics] ${event}`, data);
  }

  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

### Notification Handler

```typescript
// core/event-handlers/notification.handler.ts
import { Injectable, OnDestroy } from '@angular/core';
import { Subject } from 'rxjs';
import { takeUntil } from 'rxjs/operators';
import { EventBusService } from '../event-bus/event-bus.service';
import { DomainEvents, ToastPayload } from '../event-bus/domain-events';

@Injectable({ providedIn: 'root' })
export class NotificationEventHandler implements OnDestroy {
  private destroy$ = new Subject<void>();

  constructor(
    private eventBus: EventBusService,
    private toastService: ToastService
  ) {
    this.setupHandlers();
  }

  private setupHandlers(): void {
    this.eventBus.on<ToastPayload>(DomainEvents.SHOW_TOAST)
      .pipe(takeUntil(this.destroy$))
      .subscribe(event => {
        this.toastService.show(event.payload);
      });

    // แจ้งเตือนเมื่อ order สำเร็จ
    this.eventBus.on(DomainEvents.ORDER_PLACED)
      .pipe(takeUntil(this.destroy$))
      .subscribe(event => {
        this.toastService.show({
          message: `สั่งซื้อสำเร็จ! หมายเลขคำสั่งซื้อ: ${event.payload.orderId}`,
          type: 'success',
          duration: 8000
        });
      });
      
    // แจ้งเตือน payment failed
    this.eventBus.on(DomainEvents.PAYMENT_FAILED)
      .pipe(takeUntil(this.destroy$))
      .subscribe(event => {
        this.toastService.show({
          message: `การชำระเงินล้มเหลว: ${event.payload.reason}`,
          type: 'error',
          duration: 10000
        });
      });
  }

  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

---

## 5. Toast Service

```typescript
// shared/services/toast.service.ts
import { Injectable } from '@angular/core';
import { BehaviorSubject } from 'rxjs';

interface Toast {
  id: string;
  message: string;
  type: 'success' | 'error' | 'warning' | 'info';
  duration: number;
}

@Injectable({ providedIn: 'root' })
export class ToastService {
  private toasts$ = new BehaviorSubject<Toast[]>([]);
  
  toasts = this.toasts$.asObservable();

  show(toast: Omit<Toast, 'id'>): void {
    const newToast: Toast = {
      ...toast,
      id: Date.now().toString(),
      duration: toast.duration || 5000
    };

    this.toasts$.next([...this.toasts$.value, newToast]);

    setTimeout(() => this.dismiss(newToast.id), newToast.duration);
  }

  dismiss(id: string): void {
    this.toasts$.next(this.toasts$.value.filter(t => t.id !== id));
  }
}
```

---

## 6. Event Logger (สำหรับ Debug)

```typescript
// core/event-bus/event-logger.service.ts
import { Injectable, OnDestroy } from '@angular/core';
import { Subject } from 'rxjs';
import { takeUntil } from 'rxjs/operators';
import { EventBusService } from './event-bus.service';

@Injectable({ providedIn: 'root' })
export class EventLoggerService implements OnDestroy {
  private logs: any[] = [];
  private destroy$ = new Subject<void>();
  private isEnabled = !environment.production;

  constructor(private eventBus: EventBusService) {
    if (this.isEnabled) {
      this.startLogging();
    }
  }

  private startLogging(): void {
    this.eventBus.onAll()
      .pipe(takeUntil(this.destroy$))
      .subscribe(event => {
        const log = {
          type: event.type,
          payload: event.payload,
          timestamp: event.timestamp,
          source: event.source
        };
        
        this.logs.push(log);
        
        console.groupCollapsed(`%c[EventBus] ${event.type}`, 'color: #2196f3; font-weight: bold');
        console.log('Payload:', event.payload);
        console.log('Time:', event.timestamp);
        if (event.source) console.log('Source:', event.source);
        console.groupEnd();
      });
  }

  getEventHistory(): any[] {
    return [...this.logs];
  }

  clearHistory(): void {
    this.logs = [];
  }

  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
  }
}

declare const environment: { production: boolean };
```

---

## สรุป

| Pattern | ข้อดี |
|---------|-------|
| Event Bus | Loose coupling, Easy to add handlers |
| Domain Events | Named events, Type safety |
| Event Sourcing | Audit trail, Time travel debugging |
| CQRS | Read/Write separation |

### เมื่อไหร่ควรใช้ Events

- สื่อสารระหว่าง sibling components
- Cross-module communication
- Logging และ Analytics
- Side effects ที่ decoupled
- Real-time notifications
