# Part 61: Clean Architecture ใน Angular

## บทนำ

Clean Architecture แบ่งแอปพลิเคชันออกเป็นชั้น (layers) ที่มีการพึ่งพากันชัดเจน โดย inner layers ไม่รู้จัก outer layers

```
┌─────────────────────────────────────────────────┐
│  Frameworks & Drivers (UI, DB, External APIs)   │  ← Outer Layer
│  ┌─────────────────────────────────────────┐    │
│  │  Interface Adapters (Controllers, Views) │    │
│  │  ┌─────────────────────────────────┐    │    │
│  │  │  Application Business Rules     │    │    │
│  │  │  (Use Cases / Services)         │    │    │
│  │  │  ┌───────────────────────┐      │    │    │
│  │  │  │  Enterprise Business  │      │    │    │
│  │  │  │  Rules (Entities)     │      │    │    │
│  │  │  └───────────────────────┘      │    │    │
│  │  └─────────────────────────────────┘    │    │
│  └─────────────────────────────────────────┘    │
└─────────────────────────────────────────────────┘
```

## 1. Domain Layer (Enterprise Business Rules)

```typescript
// domain/entities/user.entity.ts
export class UserEntity {
  private constructor(
    public readonly id: string,
    public readonly email: string,
    public readonly name: string,
    public readonly role: UserRole,
    public readonly active: boolean,
    public readonly createdAt: Date
  ) {}

  static create(
    id: string,
    email: string,
    name: string,
    role: UserRole = UserRole.USER
  ): UserEntity {
    if (!UserEntity.isValidEmail(email)) {
      throw new Error(`Invalid email: ${email}`);
    }
    if (!name || name.trim().length === 0) {
      throw new Error('Name cannot be empty');
    }

    return new UserEntity(id, email, name.trim(), role, true, new Date());
  }

  activate(): UserEntity {
    return new UserEntity(this.id, this.email, this.name, this.role, true, this.createdAt);
  }

  deactivate(): UserEntity {
    return new UserEntity(this.id, this.email, this.name, this.role, false, this.createdAt);
  }

  changeRole(newRole: UserRole): UserEntity {
    return new UserEntity(this.id, this.email, this.name, newRole, this.active, this.createdAt);
  }

  isAdmin(): boolean {
    return this.role === UserRole.ADMIN;
  }

  canAccess(permission: Permission): boolean {
    const rolePermissions: Record<UserRole, Permission[]> = {
      [UserRole.ADMIN]: Object.values(Permission),
      [UserRole.USER]: [Permission.READ_PRODUCTS, Permission.CREATE_ORDER],
      [UserRole.GUEST]: [Permission.READ_PRODUCTS]
    };
    return rolePermissions[this.role]?.includes(permission) ?? false;
  }

  static isValidEmail(email: string): boolean {
    return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);
  }
}

export enum UserRole {
  ADMIN = 'admin',
  USER = 'user',
  GUEST = 'guest'
}

export enum Permission {
  READ_PRODUCTS = 'read:products',
  CREATE_ORDER = 'create:order',
  MANAGE_USERS = 'manage:users',
  VIEW_REPORTS = 'view:reports'
}
```

```typescript
// domain/entities/order.entity.ts
export class OrderEntity {
  private constructor(
    public readonly id: string,
    public readonly userId: string,
    public readonly items: OrderItem[],
    public readonly status: OrderStatus,
    public readonly createdAt: Date,
    public readonly updatedAt: Date
  ) {}

  static create(userId: string, items: OrderItem[]): OrderEntity {
    if (items.length === 0) {
      throw new Error('Order must have at least one item');
    }

    const validItems = items.filter(item => item.quantity > 0 && item.price > 0);
    if (validItems.length !== items.length) {
      throw new Error('All items must have positive quantity and price');
    }

    return new OrderEntity(
      crypto.randomUUID(),
      userId,
      items,
      OrderStatus.PENDING,
      new Date(),
      new Date()
    );
  }

  get totalAmount(): number {
    return this.items.reduce((sum, item) => sum + (item.price * item.quantity), 0);
  }

  get itemCount(): number {
    return this.items.reduce((sum, item) => sum + item.quantity, 0);
  }

  confirm(): OrderEntity {
    if (this.status !== OrderStatus.PENDING) {
      throw new Error(`Cannot confirm order with status: ${this.status}`);
    }
    return this.withStatus(OrderStatus.CONFIRMED);
  }

  ship(): OrderEntity {
    if (this.status !== OrderStatus.CONFIRMED) {
      throw new Error(`Cannot ship order with status: ${this.status}`);
    }
    return this.withStatus(OrderStatus.SHIPPED);
  }

  deliver(): OrderEntity {
    if (this.status !== OrderStatus.SHIPPED) {
      throw new Error(`Cannot deliver order with status: ${this.status}`);
    }
    return this.withStatus(OrderStatus.DELIVERED);
  }

  cancel(): OrderEntity {
    if ([OrderStatus.SHIPPED, OrderStatus.DELIVERED].includes(this.status)) {
      throw new Error(`Cannot cancel order with status: ${this.status}`);
    }
    return this.withStatus(OrderStatus.CANCELLED);
  }

  private withStatus(status: OrderStatus): OrderEntity {
    return new OrderEntity(
      this.id, this.userId, this.items, status, this.createdAt, new Date()
    );
  }
}

export enum OrderStatus {
  PENDING = 'pending',
  CONFIRMED = 'confirmed',
  SHIPPED = 'shipped',
  DELIVERED = 'delivered',
  CANCELLED = 'cancelled'
}

export interface OrderItem {
  productId: string;
  productName: string;
  quantity: number;
  price: number;
}
```

## 2. Domain Repository Interfaces

```typescript
// domain/repositories/user.repository.ts
export abstract class UserRepository {
  abstract findById(id: string): Promise<UserEntity | null>;
  abstract findByEmail(email: string): Promise<UserEntity | null>;
  abstract findAll(page: number, limit: number): Promise<{ items: UserEntity[]; total: number }>;
  abstract save(user: UserEntity): Promise<UserEntity>;
  abstract delete(id: string): Promise<void>;
  abstract exists(email: string): Promise<boolean>;
}

// domain/repositories/order.repository.ts
export abstract class OrderRepository {
  abstract findById(id: string): Promise<OrderEntity | null>;
  abstract findByUserId(userId: string): Promise<OrderEntity[]>;
  abstract save(order: OrderEntity): Promise<OrderEntity>;
  abstract delete(id: string): Promise<void>;
}
```

## 3. Application Layer (Use Cases)

```typescript
// application/use-cases/create-order.use-case.ts
import { Injectable } from '@angular/core';

export interface CreateOrderInput {
  userId: string;
  items: Array<{
    productId: string;
    quantity: number;
  }>;
}

export interface CreateOrderOutput {
  orderId: string;
  totalAmount: number;
  status: string;
}

@Injectable({ providedIn: 'root' })
export class CreateOrderUseCase {
  constructor(
    private orderRepository: OrderRepository,
    private userRepository: UserRepository,
    private productService: ProductDomainService,
    private paymentService: PaymentService,
    private eventBus: DomainEventBus
  ) {}

  async execute(input: CreateOrderInput): Promise<CreateOrderOutput> {
    // 1. ตรวจสอบว่า user มีอยู่และ active
    const user = await this.userRepository.findById(input.userId);
    if (!user) {
      throw new Error(`User ${input.userId} not found`);
    }
    if (!user.active) {
      throw new Error('User account is not active');
    }
    if (!user.canAccess(Permission.CREATE_ORDER)) {
      throw new Error('User does not have permission to create orders');
    }

    // 2. ดึงข้อมูลสินค้าและตรวจสอบ stock
    const orderItems: OrderItem[] = [];
    for (const item of input.items) {
      const product = await this.productService.findById(item.productId);
      if (!product) {
        throw new Error(`Product ${item.productId} not found`);
      }
      if (product.stock < item.quantity) {
        throw new Error(`Insufficient stock for product ${product.name}`);
      }
      
      orderItems.push({
        productId: product.id,
        productName: product.name,
        quantity: item.quantity,
        price: product.price
      });
    }

    // 3. สร้าง Order entity
    const order = OrderEntity.create(input.userId, orderItems);

    // 4. บันทึก Order
    const savedOrder = await this.orderRepository.save(order);

    // 5. Publish domain event
    await this.eventBus.publish({
      type: 'OrderCreated',
      payload: { orderId: savedOrder.id, userId: input.userId, totalAmount: savedOrder.totalAmount }
    });

    return {
      orderId: savedOrder.id,
      totalAmount: savedOrder.totalAmount,
      status: savedOrder.status
    };
  }
}
```

```typescript
// application/use-cases/register-user.use-case.ts
export interface RegisterUserInput {
  email: string;
  name: string;
  password: string;
}

export interface RegisterUserOutput {
  userId: string;
  email: string;
  name: string;
}

@Injectable({ providedIn: 'root' })
export class RegisterUserUseCase {
  constructor(
    private userRepository: UserRepository,
    private passwordHasher: PasswordHasher,
    private emailService: EmailService
  ) {}

  async execute(input: RegisterUserInput): Promise<RegisterUserOutput> {
    // ตรวจสอบ email ซ้ำ
    const exists = await this.userRepository.exists(input.email);
    if (exists) {
      throw new Error(`Email ${input.email} is already registered`);
    }

    // สร้าง User entity (validate inside)
    const user = UserEntity.create(
      crypto.randomUUID(),
      input.email,
      input.name
    );

    // Hash password (infrastructure concern)
    const passwordHash = await this.passwordHasher.hash(input.password);

    // บันทึก user
    const savedUser = await this.userRepository.save(user);

    // ส่ง welcome email (async, ไม่รอ)
    this.emailService.sendWelcomeEmail(savedUser.email, savedUser.name).catch(
      err => console.error('Failed to send welcome email:', err)
    );

    return {
      userId: savedUser.id,
      email: savedUser.email,
      name: savedUser.name
    };
  }
}
```

## 4. Infrastructure Layer

```typescript
// infrastructure/repositories/http-user.repository.ts
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { lastValueFrom } from 'rxjs';

// DTO interfaces
interface UserDto {
  id: string;
  email: string;
  name: string;
  role: string;
  active: boolean;
  created_at: string;
}

@Injectable()
export class HttpUserRepository extends UserRepository {
  private readonly baseUrl = '/api/users';

  constructor(private http: HttpClient) {
    super();
  }

  async findById(id: string): Promise<UserEntity | null> {
    try {
      const dto = await lastValueFrom(this.http.get<UserDto>(`${this.baseUrl}/${id}`));
      return this.toEntity(dto);
    } catch (error: any) {
      if (error.status === 404) return null;
      throw error;
    }
  }

  async findByEmail(email: string): Promise<UserEntity | null> {
    try {
      const users = await lastValueFrom(
        this.http.get<UserDto[]>(`${this.baseUrl}?email=${encodeURIComponent(email)}`)
      );
      return users.length > 0 ? this.toEntity(users[0]) : null;
    } catch {
      return null;
    }
  }

  async findAll(page: number, limit: number): Promise<{ items: UserEntity[]; total: number }> {
    const response = await lastValueFrom(
      this.http.get<{ items: UserDto[]; total: number }>(
        `${this.baseUrl}?page=${page}&limit=${limit}`
      )
    );
    return {
      items: response.items.map(dto => this.toEntity(dto)),
      total: response.total
    };
  }

  async save(user: UserEntity): Promise<UserEntity> {
    const dto = this.toDto(user);
    const saved = await lastValueFrom(
      this.http.put<UserDto>(`${this.baseUrl}/${user.id}`, dto)
    );
    return this.toEntity(saved);
  }

  async delete(id: string): Promise<void> {
    await lastValueFrom(this.http.delete<void>(`${this.baseUrl}/${id}`));
  }

  async exists(email: string): Promise<boolean> {
    const user = await this.findByEmail(email);
    return user !== null;
  }

  // Mapping: Entity ↔ DTO
  private toEntity(dto: UserDto): UserEntity {
    return UserEntity.create(dto.id, dto.email, dto.name, dto.role as UserRole);
  }

  private toDto(entity: UserEntity): Partial<UserDto> {
    return {
      id: entity.id,
      email: entity.email,
      name: entity.name,
      role: entity.role,
      active: entity.active
    };
  }
}
```

## 5. Presentation Layer (Angular Components)

```typescript
// presentation/orders/create-order.component.ts
import { Component, OnInit } from '@angular/core';
import { FormBuilder, FormGroup, ReactiveFormsModule, Validators } from '@angular/forms';
import { CreateOrderUseCase } from '../../application/use-cases/create-order.use-case';
import { Router } from '@angular/router';
import { CommonModule } from '@angular/common';

@Component({
  selector: 'app-create-order',
  standalone: true,
  imports: [CommonModule, ReactiveFormsModule],
  template: `
    <div class="create-order">
      <h1>สร้างออเดอร์ใหม่</h1>
      
      <form [formGroup]="orderForm" (ngSubmit)="onSubmit()">
        <!-- form fields -->
        <button type="submit" [disabled]="orderForm.invalid || submitting">
          {{ submitting ? 'กำลังสร้าง...' : 'สร้างออเดอร์' }}
        </button>
      </form>

      <div *ngIf="error" class="error">{{ error }}</div>
    </div>
  `
})
export class CreateOrderComponent {
  orderForm: FormGroup;
  submitting = false;
  error: string | null = null;

  constructor(
    private fb: FormBuilder,
    private createOrderUseCase: CreateOrderUseCase, // Use Case ไม่ใช่ Service โดยตรง
    private router: Router
  ) {
    this.orderForm = this.fb.group({
      userId: ['', Validators.required]
    });
  }

  async onSubmit(): Promise<void> {
    if (this.orderForm.invalid) return;
    
    this.submitting = true;
    this.error = null;

    try {
      const result = await this.createOrderUseCase.execute({
        userId: this.orderForm.value.userId,
        items: []
      });
      
      this.router.navigate(['/orders', result.orderId]);
    } catch (err: any) {
      this.error = err.message;
    } finally {
      this.submitting = false;
    }
  }
}
```

## 6. Dependency Injection Configuration

```typescript
// app.config.ts
import { ApplicationConfig } from '@angular/core';
import { provideHttpClient } from '@angular/common/http';
import { provideRouter } from '@angular/router';

import { UserRepository } from './domain/repositories/user.repository';
import { OrderRepository } from './domain/repositories/order.repository';
import { HttpUserRepository } from './infrastructure/repositories/http-user.repository';
import { HttpOrderRepository } from './infrastructure/repositories/http-order.repository';

export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient(),
    provideRouter([]),
    // Bind abstractions to implementations
    { provide: UserRepository, useClass: HttpUserRepository },
    { provide: OrderRepository, useClass: HttpOrderRepository }
  ]
};
```

## สรุป Clean Architecture Layers

| Layer | ความรับผิดชอบ | ตัวอย่าง |
|-------|------------|---------|
| Domain | Business rules, Entities | UserEntity, OrderEntity |
| Application | Use Cases, Orchestration | CreateOrderUseCase |
| Infrastructure | DB, HTTP, External APIs | HttpUserRepository |
| Presentation | UI, User interaction | Angular Components |

**กฎสำคัญ**: Dependencies ไหลเข้าด้านใน (inward) เท่านั้น Domain ไม่รู้จัก Infrastructure
