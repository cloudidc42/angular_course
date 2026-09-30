# Part 62: Domain-Driven Design (DDD) ใน Angular

## บทนำ

Domain-Driven Design (DDD) เป็นแนวทางการออกแบบซอฟต์แวร์ที่มุ่งเน้นที่ Domain Logic โดยใช้ภาษาเดียวกัน (Ubiquitous Language) กับ business stakeholders

## 1. Value Objects

Value Objects คือ objects ที่ถูก identify ด้วยค่า ไม่ใช่ identity

```typescript
// domain/value-objects/email.value-object.ts
export class Email {
  private readonly _value: string;

  private constructor(value: string) {
    this._value = value.toLowerCase().trim();
  }

  static create(value: string): Email {
    if (!value || !value.trim()) {
      throw new Error('Email cannot be empty');
    }
    
    const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    if (!emailRegex.test(value.trim())) {
      throw new Error(`Invalid email format: ${value}`);
    }

    return new Email(value);
  }

  get value(): string {
    return this._value;
  }

  getDomain(): string {
    return this._value.split('@')[1];
  }

  equals(other: Email): boolean {
    return this._value === other._value;
  }

  toString(): string {
    return this._value;
  }
}
```

```typescript
// domain/value-objects/money.value-object.ts
export class Money {
  private constructor(
    private readonly _amount: number,
    private readonly _currency: string
  ) {
    if (_amount < 0) {
      throw new Error('Amount cannot be negative');
    }
    if (!_currency || _currency.length !== 3) {
      throw new Error('Currency must be a 3-letter code (e.g., THB, USD)');
    }
  }

  static of(amount: number, currency = 'THB'): Money {
    return new Money(amount, currency.toUpperCase());
  }

  static zero(currency = 'THB'): Money {
    return new Money(0, currency);
  }

  get amount(): number {
    return this._amount;
  }

  get currency(): string {
    return this._currency;
  }

  add(other: Money): Money {
    this.assertSameCurrency(other);
    return new Money(this._amount + other._amount, this._currency);
  }

  subtract(other: Money): Money {
    this.assertSameCurrency(other);
    const result = this._amount - other._amount;
    if (result < 0) throw new Error('Cannot subtract: result would be negative');
    return new Money(result, this._currency);
  }

  multiply(factor: number): Money {
    if (factor < 0) throw new Error('Factor cannot be negative');
    return new Money(Math.round(this._amount * factor * 100) / 100, this._currency);
  }

  isGreaterThan(other: Money): boolean {
    this.assertSameCurrency(other);
    return this._amount > other._amount;
  }

  isZero(): boolean {
    return this._amount === 0;
  }

  equals(other: Money): boolean {
    return this._amount === other._amount && this._currency === other._currency;
  }

  format(): string {
    return new Intl.NumberFormat('th-TH', {
      style: 'currency',
      currency: this._currency
    }).format(this._amount);
  }

  private assertSameCurrency(other: Money): void {
    if (this._currency !== other._currency) {
      throw new Error(`Currency mismatch: ${this._currency} vs ${other._currency}`);
    }
  }
}
```

```typescript
// domain/value-objects/address.value-object.ts
export class Address {
  private constructor(
    public readonly street: string,
    public readonly city: string,
    public readonly province: string,
    public readonly postalCode: string,
    public readonly country: string
  ) {}

  static create(
    street: string,
    city: string,
    province: string,
    postalCode: string,
    country = 'TH'
  ): Address {
    if (!street?.trim()) throw new Error('Street is required');
    if (!city?.trim()) throw new Error('City is required');
    if (!province?.trim()) throw new Error('Province is required');
    if (!/^\d{5}$/.test(postalCode)) throw new Error('Postal code must be 5 digits');

    return new Address(street.trim(), city.trim(), province.trim(), postalCode, country);
  }

  equals(other: Address): boolean {
    return (
      this.street === other.street &&
      this.city === other.city &&
      this.province === other.province &&
      this.postalCode === other.postalCode &&
      this.country === other.country
    );
  }

  toFormattedString(): string {
    return `${this.street}, ${this.city}, ${this.province} ${this.postalCode}`;
  }
}
```

## 2. Aggregates

Aggregate คือ cluster ของ entities ที่มี root entity หลัก

```typescript
// domain/aggregates/shopping-cart.aggregate.ts
export class ShoppingCartAggregate {
  private _items: CartItem[] = [];
  private _events: DomainEvent[] = [];

  private constructor(
    public readonly id: string,
    public readonly userId: string,
    items: CartItem[]
  ) {
    this._items = [...items];
  }

  static create(userId: string): ShoppingCartAggregate {
    if (!userId?.trim()) throw new Error('User ID is required');
    
    const cart = new ShoppingCartAggregate(
      crypto.randomUUID(),
      userId,
      []
    );
    
    cart.addEvent({
      type: 'CartCreated',
      payload: { cartId: cart.id, userId },
      occurredAt: new Date()
    });
    
    return cart;
  }

  static reconstitute(
    id: string,
    userId: string,
    items: CartItem[]
  ): ShoppingCartAggregate {
    return new ShoppingCartAggregate(id, userId, items);
  }

  get items(): ReadonlyArray<CartItem> {
    return [...this._items];
  }

  get totalAmount(): Money {
    return this._items.reduce(
      (total, item) => total.add(item.subtotal),
      Money.zero()
    );
  }

  get totalItems(): number {
    return this._items.reduce((sum, item) => sum + item.quantity, 0);
  }

  get isEmpty(): boolean {
    return this._items.length === 0;
  }

  addItem(productId: string, productName: string, price: Money, quantity: number): void {
    if (quantity <= 0) throw new Error('Quantity must be positive');

    const existingIndex = this._items.findIndex(i => i.productId === productId);

    if (existingIndex >= 0) {
      const existing = this._items[existingIndex];
      this._items[existingIndex] = existing.increaseQuantity(quantity);
    } else {
      this._items.push(CartItem.create(productId, productName, price, quantity));
    }

    this.addEvent({
      type: 'CartItemAdded',
      payload: { cartId: this.id, productId, quantity },
      occurredAt: new Date()
    });
  }

  removeItem(productId: string): void {
    const index = this._items.findIndex(i => i.productId === productId);
    if (index === -1) throw new Error(`Product ${productId} not in cart`);

    this._items.splice(index, 1);

    this.addEvent({
      type: 'CartItemRemoved',
      payload: { cartId: this.id, productId },
      occurredAt: new Date()
    });
  }

  updateQuantity(productId: string, quantity: number): void {
    if (quantity <= 0) {
      this.removeItem(productId);
      return;
    }

    const index = this._items.findIndex(i => i.productId === productId);
    if (index === -1) throw new Error(`Product ${productId} not in cart`);

    this._items[index] = this._items[index].withQuantity(quantity);
  }

  clear(): void {
    this._items = [];
    this.addEvent({
      type: 'CartCleared',
      payload: { cartId: this.id },
      occurredAt: new Date()
    });
  }

  checkout(): CheckoutResult {
    if (this.isEmpty) throw new Error('Cannot checkout an empty cart');
    
    this.addEvent({
      type: 'CartCheckedOut',
      payload: {
        cartId: this.id,
        userId: this.userId,
        totalAmount: this.totalAmount.amount,
        itemCount: this.totalItems
      },
      occurredAt: new Date()
    });

    return {
      items: this.items.map(item => ({
        productId: item.productId,
        quantity: item.quantity,
        price: item.price.amount
      })),
      totalAmount: this.totalAmount
    };
  }

  getDomainEvents(): DomainEvent[] {
    return [...this._events];
  }

  clearDomainEvents(): void {
    this._events = [];
  }

  private addEvent(event: DomainEvent): void {
    this._events.push(event);
  }
}

// Cart Item Value Object (part of aggregate)
class CartItem {
  private constructor(
    public readonly productId: string,
    public readonly productName: string,
    public readonly price: Money,
    public readonly quantity: number
  ) {}

  static create(
    productId: string,
    productName: string,
    price: Money,
    quantity: number
  ): CartItem {
    return new CartItem(productId, productName, price, quantity);
  }

  get subtotal(): Money {
    return this.price.multiply(this.quantity);
  }

  increaseQuantity(amount: number): CartItem {
    return new CartItem(this.productId, this.productName, this.price, this.quantity + amount);
  }

  withQuantity(quantity: number): CartItem {
    return new CartItem(this.productId, this.productName, this.price, quantity);
  }
}

// Interfaces
interface DomainEvent {
  type: string;
  payload: Record<string, any>;
  occurredAt: Date;
}

interface CheckoutResult {
  items: Array<{ productId: string; quantity: number; price: number }>;
  totalAmount: Money;
}
```

## 3. Domain Events

```typescript
// domain/events/domain-event.ts
export interface DomainEvent<T = any> {
  eventId: string;
  type: string;
  aggregateId: string;
  aggregateType: string;
  payload: T;
  occurredAt: Date;
  version: number;
}

// domain/events/order-events.ts
export interface OrderCreatedEvent {
  orderId: string;
  userId: string;
  items: Array<{ productId: string; quantity: number; price: number }>;
  totalAmount: number;
}

export interface OrderStatusChangedEvent {
  orderId: string;
  previousStatus: string;
  newStatus: string;
  changedAt: Date;
}

export interface OrderCancelledEvent {
  orderId: string;
  userId: string;
  reason?: string;
}
```

```typescript
// domain/event-bus/domain-event-bus.ts
import { Injectable } from '@angular/core';
import { Subject } from 'rxjs';
import { filter, map } from 'rxjs/operators';

type EventHandler<T = any> = (event: DomainEvent<T>) => Promise<void> | void;

@Injectable({ providedIn: 'root' })
export class DomainEventBus {
  private eventSubject = new Subject<DomainEvent>();

  publish(event: DomainEvent): void {
    this.eventSubject.next(event);
    console.log(`[DomainEvent] ${event.type}:`, event.payload);
  }

  publishAll(events: DomainEvent[]): void {
    events.forEach(event => this.publish(event));
  }

  subscribe<T>(
    eventType: string,
    handler: EventHandler<T>
  ): { unsubscribe: () => void } {
    const subscription = this.eventSubject.pipe(
      filter(event => event.type === eventType),
      map(event => event as DomainEvent<T>)
    ).subscribe(event => {
      try {
        handler(event);
      } catch (err) {
        console.error(`Error handling event ${eventType}:`, err);
      }
    });

    return { unsubscribe: () => subscription.unsubscribe() };
  }
}
```

## 4. Domain Services

```typescript
// domain/services/discount.domain-service.ts
import { Injectable } from '@angular/core';

export interface DiscountRule {
  type: 'percentage' | 'fixed' | 'buy_n_get_m';
  value: number;
  minAmount?: number;
  minItems?: number;
}

@Injectable({ providedIn: 'root' })
export class DiscountDomainService {
  calculateDiscount(
    totalAmount: Money,
    items: CartItem[],
    rules: DiscountRule[]
  ): Money {
    let discount = Money.zero();

    for (const rule of rules) {
      const ruleDiscount = this.applyRule(totalAmount, items, rule);
      discount = discount.add(ruleDiscount);
    }

    // ส่วนลดต้องไม่เกินราคารวม
    if (discount.isGreaterThan(totalAmount)) {
      return totalAmount;
    }

    return discount;
  }

  private applyRule(
    totalAmount: Money,
    items: CartItem[],
    rule: DiscountRule
  ): Money {
    switch (rule.type) {
      case 'percentage':
        if (rule.minAmount && totalAmount.amount < rule.minAmount) {
          return Money.zero();
        }
        return totalAmount.multiply(rule.value / 100);

      case 'fixed':
        if (rule.minAmount && totalAmount.amount < rule.minAmount) {
          return Money.zero();
        }
        return Money.of(rule.value);

      case 'buy_n_get_m':
        // ตัวอย่าง: ซื้อ 2 แถม 1
        // คืนราคาสินค้าถูกสุดฟรี
        const itemCount = items.reduce((sum, item) => sum + item.quantity, 0);
        if (rule.minItems && itemCount < rule.minItems) {
          return Money.zero();
        }
        const sortedByPrice = [...items].sort((a, b) => a.price.amount - b.price.amount);
        return sortedByPrice[0]?.price || Money.zero();

      default:
        return Money.zero();
    }
  }
}
```

## 5. Bounded Contexts

```
┌─────────────────────┐  ┌─────────────────────┐
│   Catalog Context   │  │   Order Context      │
│                     │  │                      │
│  Product            │  │  Order               │
│  Category           │  │  OrderItem           │
│  ProductVariant     │  │  Shipping            │
│                     │  │                      │
└─────────────────────┘  └─────────────────────┘
         ↑                          ↑
         │  Context Map             │
         └──────────────────────────┘
              (Product → OrderItem mapping)
```

```typescript
// context-maps/product-to-order.mapper.ts
// แปลง Product จาก Catalog Context ไปเป็น OrderItem ใน Order Context

export class ProductToOrderItemMapper {
  static map(product: CatalogProduct, quantity: number): OrderItem {
    return {
      productId: product.id,
      productName: product.name,
      price: product.currentPrice.amount,
      quantity,
      // Order context ไม่จำเป็นต้องรู้รายละเอียดทั้งหมดของ Product
    };
  }
}
```

## สรุป DDD Concepts

| Concept | ความหมาย | ตัวอย่าง |
|---------|---------|---------|
| Value Object | Object ที่ identify ด้วยค่า | Money, Email, Address |
| Entity | Object ที่มี identity | User, Order |
| Aggregate | Cluster ของ entities | ShoppingCart + CartItems |
| Domain Event | สิ่งที่เกิดขึ้นใน domain | OrderCreated |
| Repository | Abstract data access | UserRepository |
| Domain Service | Logic ที่ไม่ fit ใน entity | DiscountService |
| Bounded Context | ขอบเขตของ domain | Catalog, Order, Payment |

DDD ช่วยให้โค้ดสะท้อน business logic ได้ชัดเจน และง่ายต่อการสื่อสารกับ stakeholders
