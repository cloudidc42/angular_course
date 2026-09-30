# Part 07 — Services และ Dependency Injection

## สารบัญ

1. [Service คืออะไร](#service-คืออะไร)
2. [สร้าง Service ด้วย @Injectable](#สร้าง-service-ด้วย-injectable)
3. [providedIn: 'root' vs Module vs Component](#providedin-root-vs-module-vs-component)
4. [Hierarchical Injection](#hierarchical-injection)
5. [การใช้ Service ใน Component](#การใช้-service-ใน-component)
6. [สร้าง Data Service (CRUD)](#สร้าง-data-service-crud)
7. [Service Pattern: Singleton, Factory](#service-pattern-singleton-factory)
8. [Workshop: Shopping Cart Service](#workshop-shopping-cart-service)

---

## Service คืออะไร

**Service** ใน Angular คือ Class ที่มีหน้าที่รับผิดชอบงานเฉพาะด้าน เช่น การดึงข้อมูลจาก API, การจัดการ State, การ Log, หรือการแชร์ข้อมูลระหว่าง Component

### ทำไมต้องใช้ Service?

หากไม่มี Service เราจะเขียน Logic ทั้งหมดไว้ใน Component ซึ่งทำให้:
- Component มีขนาดใหญ่และยากต่อการดูแล
- ไม่สามารถแชร์ Logic ระหว่าง Component ได้
- ทดสอบยาก (Tight Coupling)

ด้วย Service เราสามารถ:
- แยก Business Logic ออกจาก UI Logic
- แชร์ข้อมูลระหว่าง Component หลาย ๆ ตัว
- ทำ Unit Test ได้ง่ายขึ้น
- นำกลับมาใช้ใหม่ได้ (Reusable)

### หลักการ Single Responsibility

```
Component  →  แสดงผล UI และจัดการ User Interaction
Service    →  Business Logic, Data Management, Shared State
```

---

## สร้าง Service ด้วย @Injectable

### โครงสร้างพื้นฐานของ Service

```typescript
// src/app/services/greeting.service.ts

import { Injectable } from '@angular/core';

@Injectable({
  providedIn: 'root'  // บอก Angular ว่า Service นี้ใช้ได้ทั้ง App
})
export class GreetingService {

  private greeting = 'สวัสดี';

  greet(name: string): string {
    return `${this.greeting}, ${name}!`;
  }

  setGreeting(newGreeting: string): void {
    this.greeting = newGreeting;
  }
}
```

### สร้าง Service ด้วย Angular CLI

```bash
# สร้าง Service ใหม่
ng generate service services/greeting
# หรือย่อ
ng g s services/greeting
```

Angular CLI จะสร้าง 2 ไฟล์:
- `greeting.service.ts` — ตัว Service
- `greeting.service.spec.ts` — ไฟล์ทดสอบ

### @Injectable Decorator

`@Injectable()` บอกให้ Angular รู้ว่า Class นี้สามารถถูก Inject ได้ และสามารถรับ Dependency จาก DI System

```typescript
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';

@Injectable({
  providedIn: 'root'
})
export class UserService {

  // Inject HttpClient เข้ามาใน Service
  constructor(private http: HttpClient) { }

  getUsers() {
    return this.http.get('/api/users');
  }
}
```

---

## providedIn: 'root' vs Module vs Component

### 1. providedIn: 'root' (แนะนำสำหรับส่วนใหญ่)

```typescript
@Injectable({
  providedIn: 'root'  // Singleton — มีแค่ instance เดียวทั้ง App
})
export class AuthService {
  private isLoggedIn = false;

  login(username: string, password: string): boolean {
    // Logic การ login
    this.isLoggedIn = true;
    return this.isLoggedIn;
  }

  logout(): void {
    this.isLoggedIn = false;
  }

  checkLogin(): boolean {
    return this.isLoggedIn;
  }
}
```

**ข้อดี:**
- Angular ทำ Tree-shaking ให้อัตโนมัติ (ถ้าไม่ใช้ก็ไม่รวมใน Bundle)
- ง่ายต่อการใช้งาน
- Singleton ทั่วทั้ง Application

### 2. providedIn: Module

```typescript
// src/app/admin/admin.module.ts
import { NgModule } from '@angular/core';
import { AdminService } from './admin.service';

@NgModule({
  providers: [AdminService]  // วิธีที่ 1: ใส่ใน providers ของ Module
})
export class AdminModule { }
```

```typescript
// admin.service.ts
@Injectable({
  providedIn: AdminModule  // วิธีที่ 2: ระบุ Module โดยตรง
})
export class AdminService {
  getAdminData() {
    return { role: 'admin', data: [] };
  }
}
```

**เมื่อใช้:** เมื่อต้องการให้ Service ใช้งานได้เฉพาะใน Module นั้น ๆ

### 3. providedIn: Component (Component-scoped)

```typescript
// product-list.component.ts
import { Component } from '@angular/core';
import { ProductFilterService } from './product-filter.service';

@Component({
  selector: 'app-product-list',
  templateUrl: './product-list.component.html',
  providers: [ProductFilterService]  // สร้าง instance ใหม่สำหรับ Component นี้
})
export class ProductListComponent {
  constructor(private filterService: ProductFilterService) { }
}
```

**เมื่อใช้:** เมื่อต้องการ Instance ที่แยกกันของแต่ละ Component (ไม่แชร์ State)

### ตารางเปรียบเทียบ

| ระดับ | Scope | Instance | ใช้เมื่อ |
|-------|-------|----------|---------|
| `root` | ทั้ง App | 1 ตัว | Service ทั่วไป, Singleton |
| Module | ใน Module | 1 ตัวต่อ Module | Feature-specific Service |
| Component | ใน Component | 1 ตัวต่อ Component | State ที่แยกกัน |

---

## Hierarchical Injection

Angular มีระบบ DI แบบ Hierarchical (ลำดับชั้น) ซึ่งหมายความว่า Injector ลูกสามารถใช้ Service จาก Injector พ่อได้

```
Root Injector (AppModule)
  └── Feature Module Injector
        └── Component Injector
              └── Child Component Injector
```

### ตัวอย่าง Hierarchical Injection

```typescript
// theme.service.ts
@Injectable()
export class ThemeService {
  private theme = 'light';

  getTheme(): string {
    return this.theme;
  }

  setTheme(theme: string): void {
    this.theme = theme;
  }
}
```

```typescript
// app.component.ts — Root level
@Component({
  selector: 'app-root',
  template: `<app-parent></app-parent>`,
  providers: [ThemeService]  // Instance 1 สำหรับ app-root และลูกหลาน
})
export class AppComponent { }
```

```typescript
// parent.component.ts
@Component({
  selector: 'app-parent',
  template: `
    <p>Theme: {{ theme }}</p>
    <app-child></app-child>
  `
  // ไม่มี providers → ใช้ Instance จาก AppComponent
})
export class ParentComponent {
  theme: string;

  constructor(private themeService: ThemeService) {
    this.theme = themeService.getTheme();
  }
}
```

```typescript
// child.component.ts
@Component({
  selector: 'app-child',
  template: `<p>Child Theme: {{ theme }}</p>`,
  providers: [ThemeService]  // Instance 2 ใหม่แยกจาก Instance 1
})
export class ChildComponent {
  theme: string;

  constructor(private themeService: ThemeService) {
    this.theme = themeService.getTheme();
    // ใช้ Instance ของตัวเองแยกจาก Parent
  }
}
```

### Injection Tokens

เมื่อต้องการ Inject ค่าที่ไม่ใช่ Class:

```typescript
// tokens.ts
import { InjectionToken } from '@angular/core';

export interface AppConfig {
  apiUrl: string;
  maxItems: number;
  debugMode: boolean;
}

export const APP_CONFIG = new InjectionToken<AppConfig>('app.config');
```

```typescript
// app.module.ts
import { APP_CONFIG } from './tokens';

@NgModule({
  providers: [
    {
      provide: APP_CONFIG,
      useValue: {
        apiUrl: 'https://api.example.com',
        maxItems: 100,
        debugMode: false
      }
    }
  ]
})
export class AppModule { }
```

```typescript
// some.service.ts
import { Inject, Injectable } from '@angular/core';
import { APP_CONFIG, AppConfig } from './tokens';

@Injectable({ providedIn: 'root' })
export class SomeService {
  constructor(@Inject(APP_CONFIG) private config: AppConfig) {
    console.log('API URL:', config.apiUrl);
  }
}
```

---

## การใช้ Service ใน Component

### วิธีที่ 1: Constructor Injection (แนะนำ)

```typescript
// product.component.ts
import { Component, OnInit } from '@angular/core';
import { ProductService } from '../services/product.service';
import { Product } from '../models/product.model';

@Component({
  selector: 'app-product',
  template: `
    <div *ngFor="let product of products">
      <h3>{{ product.name }}</h3>
      <p>ราคา: {{ product.price | currency:'THB' }}</p>
    </div>
  `
})
export class ProductComponent implements OnInit {
  products: Product[] = [];

  // Angular Inject ProductService ให้อัตโนมัติ
  constructor(private productService: ProductService) { }

  ngOnInit(): void {
    this.products = this.productService.getAllProducts();
  }
}
```

### วิธีที่ 2: inject() Function (Angular 14+)

```typescript
import { Component, OnInit, inject } from '@angular/core';
import { ProductService } from '../services/product.service';

@Component({
  selector: 'app-product',
  template: `...`
})
export class ProductComponent implements OnInit {
  // ใช้ inject() แทน Constructor
  private productService = inject(ProductService);

  products = this.productService.getAllProducts();

  ngOnInit(): void {
    // ใช้งานได้เลย
  }
}
```

### การใช้ Service ร่วมกัน (Service Communication)

```typescript
// notification.service.ts
import { Injectable } from '@angular/core';
import { Subject, Observable } from 'rxjs';

export interface Notification {
  message: string;
  type: 'success' | 'error' | 'warning' | 'info';
}

@Injectable({ providedIn: 'root' })
export class NotificationService {
  private notificationSubject = new Subject<Notification>();

  // Observable ที่ Component อื่นสามารถ Subscribe ได้
  notifications$: Observable<Notification> = this.notificationSubject.asObservable();

  show(message: string, type: 'success' | 'error' | 'warning' | 'info' = 'info'): void {
    this.notificationSubject.next({ message, type });
  }

  success(message: string): void {
    this.show(message, 'success');
  }

  error(message: string): void {
    this.show(message, 'error');
  }
}
```

```typescript
// product-form.component.ts
@Component({ selector: 'app-product-form', template: `...` })
export class ProductFormComponent {
  constructor(
    private productService: ProductService,
    private notificationService: NotificationService
  ) { }

  saveProduct(product: Product): void {
    this.productService.create(product).subscribe({
      next: () => this.notificationService.success('บันทึกสำเร็จ!'),
      error: () => this.notificationService.error('เกิดข้อผิดพลาด')
    });
  }
}
```

```typescript
// notification.component.ts
@Component({
  selector: 'app-notification',
  template: `
    <div *ngIf="currentNotification" 
         [class]="'alert alert-' + currentNotification.type">
      {{ currentNotification.message }}
    </div>
  `
})
export class NotificationComponent implements OnInit, OnDestroy {
  currentNotification: Notification | null = null;
  private subscription?: Subscription;

  constructor(private notificationService: NotificationService) { }

  ngOnInit(): void {
    this.subscription = this.notificationService.notifications$.subscribe(
      notification => {
        this.currentNotification = notification;
        setTimeout(() => this.currentNotification = null, 3000);
      }
    );
  }

  ngOnDestroy(): void {
    this.subscription?.unsubscribe();
  }
}
```

---

## สร้าง Data Service (CRUD)

### Model Interface

```typescript
// src/app/models/product.model.ts
export interface Product {
  id: number;
  name: string;
  description: string;
  price: number;
  category: string;
  stock: number;
  imageUrl?: string;
  createdAt: Date;
  updatedAt: Date;
}

export type CreateProductDto = Omit<Product, 'id' | 'createdAt' | 'updatedAt'>;
export type UpdateProductDto = Partial<CreateProductDto>;
```

### Product Service แบบ In-Memory (Mock Data)

```typescript
// src/app/services/product.service.ts
import { Injectable } from '@angular/core';
import { Observable, of, throwError } from 'rxjs';
import { delay } from 'rxjs/operators';
import { Product, CreateProductDto, UpdateProductDto } from '../models/product.model';

@Injectable({ providedIn: 'root' })
export class ProductService {

  private products: Product[] = [
    {
      id: 1,
      name: 'MacBook Pro',
      description: 'โน้ตบุ๊กประสิทธิภาพสูง',
      price: 59900,
      category: 'Electronics',
      stock: 10,
      imageUrl: 'assets/macbook.jpg',
      createdAt: new Date('2024-01-01'),
      updatedAt: new Date('2024-01-01')
    },
    {
      id: 2,
      name: 'iPhone 15',
      description: 'สมาร์ทโฟน Apple รุ่นล่าสุด',
      price: 32900,
      category: 'Electronics',
      stock: 25,
      imageUrl: 'assets/iphone.jpg',
      createdAt: new Date('2024-01-15'),
      updatedAt: new Date('2024-01-15')
    },
    {
      id: 3,
      name: 'iPad Air',
      description: 'แท็บเล็ตน้ำหนักเบาประสิทธิภาพสูง',
      price: 21900,
      category: 'Electronics',
      stock: 15,
      imageUrl: 'assets/ipad.jpg',
      createdAt: new Date('2024-02-01'),
      updatedAt: new Date('2024-02-01')
    }
  ];

  private nextId = 4;

  // READ ALL — ดึงสินค้าทั้งหมด
  getAll(): Observable<Product[]> {
    return of([...this.products]).pipe(delay(300)); // จำลอง API delay
  }

  // READ ONE — ดึงสินค้าตาม ID
  getById(id: number): Observable<Product> {
    const product = this.products.find(p => p.id === id);
    if (!product) {
      return throwError(() => new Error(`ไม่พบสินค้า ID: ${id}`));
    }
    return of({ ...product }).pipe(delay(200));
  }

  // READ BY CATEGORY — กรองตามหมวดหมู่
  getByCategory(category: string): Observable<Product[]> {
    const filtered = this.products.filter(p =>
      p.category.toLowerCase() === category.toLowerCase()
    );
    return of([...filtered]).pipe(delay(200));
  }

  // SEARCH — ค้นหาสินค้า
  search(keyword: string): Observable<Product[]> {
    const lower = keyword.toLowerCase();
    const results = this.products.filter(p =>
      p.name.toLowerCase().includes(lower) ||
      p.description.toLowerCase().includes(lower)
    );
    return of([...results]).pipe(delay(200));
  }

  // CREATE — เพิ่มสินค้าใหม่
  create(dto: CreateProductDto): Observable<Product> {
    const newProduct: Product = {
      ...dto,
      id: this.nextId++,
      createdAt: new Date(),
      updatedAt: new Date()
    };
    this.products.push(newProduct);
    return of({ ...newProduct }).pipe(delay(300));
  }

  // UPDATE — แก้ไขสินค้า
  update(id: number, dto: UpdateProductDto): Observable<Product> {
    const index = this.products.findIndex(p => p.id === id);
    if (index === -1) {
      return throwError(() => new Error(`ไม่พบสินค้า ID: ${id}`));
    }

    const updated: Product = {
      ...this.products[index],
      ...dto,
      id, // ป้องกันการเปลี่ยน ID
      updatedAt: new Date()
    };

    this.products[index] = updated;
    return of({ ...updated }).pipe(delay(300));
  }

  // DELETE — ลบสินค้า
  delete(id: number): Observable<void> {
    const index = this.products.findIndex(p => p.id === id);
    if (index === -1) {
      return throwError(() => new Error(`ไม่พบสินค้า ID: ${id}`));
    }
    this.products.splice(index, 1);
    return of(undefined).pipe(delay(200));
  }

  // UTILITY — นับจำนวนสินค้า
  count(): Observable<number> {
    return of(this.products.length);
  }

  // UTILITY — ดึงหมวดหมู่ทั้งหมด
  getCategories(): Observable<string[]> {
    const categories = [...new Set(this.products.map(p => p.category))];
    return of(categories);
  }
}
```

### Product Service แบบเรียก HTTP API จริง

```typescript
// src/app/services/product-api.service.ts
import { Injectable } from '@angular/core';
import { HttpClient, HttpParams, HttpErrorResponse } from '@angular/common/http';
import { Observable, throwError } from 'rxjs';
import { catchError, map, retry } from 'rxjs/operators';
import { Product, CreateProductDto, UpdateProductDto } from '../models/product.model';

interface ApiResponse<T> {
  success: boolean;
  data: T;
  message?: string;
  total?: number;
}

@Injectable({ providedIn: 'root' })
export class ProductApiService {
  private readonly apiUrl = 'https://api.example.com/products';

  constructor(private http: HttpClient) { }

  getAll(page = 1, limit = 10): Observable<Product[]> {
    const params = new HttpParams()
      .set('page', page.toString())
      .set('limit', limit.toString());

    return this.http
      .get<ApiResponse<Product[]>>(this.apiUrl, { params })
      .pipe(
        map(response => response.data),
        retry(3),
        catchError(this.handleError)
      );
  }

  getById(id: number): Observable<Product> {
    return this.http
      .get<ApiResponse<Product>>(`${this.apiUrl}/${id}`)
      .pipe(
        map(response => response.data),
        catchError(this.handleError)
      );
  }

  create(dto: CreateProductDto): Observable<Product> {
    return this.http
      .post<ApiResponse<Product>>(this.apiUrl, dto)
      .pipe(
        map(response => response.data),
        catchError(this.handleError)
      );
  }

  update(id: number, dto: UpdateProductDto): Observable<Product> {
    return this.http
      .put<ApiResponse<Product>>(`${this.apiUrl}/${id}`, dto)
      .pipe(
        map(response => response.data),
        catchError(this.handleError)
      );
  }

  delete(id: number): Observable<void> {
    return this.http
      .delete<void>(`${this.apiUrl}/${id}`)
      .pipe(catchError(this.handleError));
  }

  private handleError(error: HttpErrorResponse): Observable<never> {
    let errorMessage = 'เกิดข้อผิดพลาดที่ไม่ทราบสาเหตุ';

    if (error.error instanceof ErrorEvent) {
      // Client-side error
      errorMessage = `ข้อผิดพลาดฝั่ง Client: ${error.error.message}`;
    } else {
      // Server-side error
      switch (error.status) {
        case 400: errorMessage = 'ข้อมูลไม่ถูกต้อง'; break;
        case 401: errorMessage = 'ไม่ได้รับอนุญาต กรุณาเข้าสู่ระบบ'; break;
        case 403: errorMessage = 'ไม่มีสิทธิ์เข้าถึงข้อมูลนี้'; break;
        case 404: errorMessage = 'ไม่พบข้อมูลที่ต้องการ'; break;
        case 500: errorMessage = 'ข้อผิดพลาดจาก Server'; break;
        default: errorMessage = `Error ${error.status}: ${error.message}`;
      }
    }

    console.error('API Error:', errorMessage, error);
    return throwError(() => new Error(errorMessage));
  }
}
```

---

## Service Pattern: Singleton, Factory

### Singleton Pattern

Service ที่ provide ใน `root` จะเป็น Singleton โดยอัตโนมัติ

```typescript
// src/app/services/app-state.service.ts
import { Injectable } from '@angular/core';
import { BehaviorSubject, Observable } from 'rxjs';

export interface AppState {
  user: User | null;
  language: 'th' | 'en';
  theme: 'light' | 'dark';
  isLoading: boolean;
}

interface User {
  id: number;
  name: string;
  email: string;
  role: string;
}

@Injectable({ providedIn: 'root' })  // Singleton ทั่วทั้ง App
export class AppStateService {

  private state = new BehaviorSubject<AppState>({
    user: null,
    language: 'th',
    theme: 'light',
    isLoading: false
  });

  // ให้ Component Subscribe เพื่อรับการอัปเดต
  state$: Observable<AppState> = this.state.asObservable();

  // Getter สำหรับดูค่าปัจจุบัน
  get currentState(): AppState {
    return this.state.getValue();
  }

  // อัปเดตบางส่วนของ State
  updateState(partial: Partial<AppState>): void {
    this.state.next({
      ...this.currentState,
      ...partial
    });
  }

  setUser(user: User | null): void {
    this.updateState({ user });
  }

  setTheme(theme: 'light' | 'dark'): void {
    this.updateState({ theme });
    document.body.setAttribute('data-theme', theme);
  }

  setLoading(isLoading: boolean): void {
    this.updateState({ isLoading });
  }
}
```

### Factory Pattern

ใช้เมื่อต้องการสร้าง Service แตกต่างกันตามเงื่อนไข:

```typescript
// src/app/services/logger.service.ts
export abstract class LoggerService {
  abstract log(message: string): void;
  abstract warn(message: string): void;
  abstract error(message: string, error?: Error): void;
}

// Console Logger สำหรับ Development
export class ConsoleLoggerService extends LoggerService {
  log(message: string): void {
    console.log(`[INFO] ${new Date().toISOString()}: ${message}`);
  }

  warn(message: string): void {
    console.warn(`[WARN] ${new Date().toISOString()}: ${message}`);
  }

  error(message: string, error?: Error): void {
    console.error(`[ERROR] ${new Date().toISOString()}: ${message}`, error);
  }
}

// Remote Logger สำหรับ Production
export class RemoteLoggerService extends LoggerService {
  constructor(private http: HttpClient, private apiUrl: string) {
    super();
  }

  log(message: string): void {
    this.sendLog('info', message);
  }

  warn(message: string): void {
    this.sendLog('warn', message);
  }

  error(message: string, error?: Error): void {
    this.sendLog('error', message, error);
  }

  private sendLog(level: string, message: string, error?: Error): void {
    const payload = {
      level,
      message,
      error: error?.message,
      timestamp: new Date().toISOString(),
      userAgent: navigator.userAgent
    };
    this.http.post(`${this.apiUrl}/logs`, payload).subscribe();
  }
}
```

```typescript
// app.module.ts
import { environment } from '../environments/environment';

@NgModule({
  providers: [
    {
      provide: LoggerService,
      useFactory: (http: HttpClient) => {
        if (environment.production) {
          return new RemoteLoggerService(http, environment.logApiUrl);
        }
        return new ConsoleLoggerService();
      },
      deps: [HttpClient]  // Dependencies ที่ Factory ต้องการ
    }
  ]
})
export class AppModule { }
```

```typescript
// any.component.ts
@Component({ selector: 'app-any', template: `...` })
export class AnyComponent {
  constructor(private logger: LoggerService) {
    // ได้ ConsoleLoggerService ใน Dev
    // ได้ RemoteLoggerService ใน Prod
    this.logger.log('Component initialized');
  }
}
```

### useExisting — Alias Pattern

```typescript
// บางครั้งต้องการให้ Token หนึ่งชี้ไปยัง Service เดียวกัน
@NgModule({
  providers: [
    ProductService,
    { provide: 'PRODUCT_SERVICE', useExisting: ProductService }
  ]
})
export class AppModule { }
```

---

## Workshop: Shopping Cart Service

ในส่วนนี้เราจะสร้าง Shopping Cart Service ที่สมบูรณ์ พร้อม Component สำหรับแสดงผล

### Step 1: สร้าง Model

```typescript
// src/app/models/cart.model.ts

export interface CartItem {
  id: string;        // UUID
  productId: number;
  name: string;
  price: number;
  quantity: number;
  imageUrl?: string;
  total: number;     // price * quantity
}

export interface Cart {
  items: CartItem[];
  subtotal: number;      // ราคารวมก่อน VAT
  vat: number;           // VAT 7%
  total: number;         // ราคารวมทั้งหมด
  itemCount: number;     // จำนวนสินค้าทั้งหมด
  updatedAt: Date;
}
```

### Step 2: สร้าง Cart Service

```typescript
// src/app/services/cart.service.ts
import { Injectable } from '@angular/core';
import { BehaviorSubject, Observable } from 'rxjs';
import { map } from 'rxjs/operators';
import { Cart, CartItem } from '../models/cart.model';
import { Product } from '../models/product.model';

@Injectable({ providedIn: 'root' })
export class CartService {

  private readonly VAT_RATE = 0.07;
  private readonly STORAGE_KEY = 'shopping_cart';

  private cartSubject = new BehaviorSubject<Cart>(this.loadFromStorage());

  // Observable สาธารณะ
  cart$: Observable<Cart> = this.cartSubject.asObservable();

  // Computed Observables
  itemCount$: Observable<number> = this.cart$.pipe(
    map(cart => cart.itemCount)
  );

  total$: Observable<number> = this.cart$.pipe(
    map(cart => cart.total)
  );

  // โหลด Cart จาก LocalStorage
  private loadFromStorage(): Cart {
    try {
      const stored = localStorage.getItem(this.STORAGE_KEY);
      if (stored) {
        const cart = JSON.parse(stored) as Cart;
        cart.updatedAt = new Date(cart.updatedAt);
        return cart;
      }
    } catch (e) {
      console.error('ไม่สามารถโหลด Cart จาก Storage:', e);
    }
    return this.createEmptyCart();
  }

  // บันทึก Cart ลง LocalStorage
  private saveToStorage(cart: Cart): void {
    try {
      localStorage.setItem(this.STORAGE_KEY, JSON.stringify(cart));
    } catch (e) {
      console.error('ไม่สามารถบันทึก Cart:', e);
    }
  }

  // สร้าง Cart ว่างเปล่า
  private createEmptyCart(): Cart {
    return {
      items: [],
      subtotal: 0,
      vat: 0,
      total: 0,
      itemCount: 0,
      updatedAt: new Date()
    };
  }

  // คำนวณ Cart ใหม่
  private recalculate(items: CartItem[]): Cart {
    const subtotal = items.reduce((sum, item) => sum + item.total, 0);
    const vat = subtotal * this.VAT_RATE;
    const total = subtotal + vat;
    const itemCount = items.reduce((sum, item) => sum + item.quantity, 0);

    return {
      items: [...items],
      subtotal: Math.round(subtotal * 100) / 100,
      vat: Math.round(vat * 100) / 100,
      total: Math.round(total * 100) / 100,
      itemCount,
      updatedAt: new Date()
    };
  }

  // อัปเดต State
  private updateCart(items: CartItem[]): void {
    const newCart = this.recalculate(items);
    this.cartSubject.next(newCart);
    this.saveToStorage(newCart);
  }

  // เพิ่มสินค้าลงตะกร้า
  addItem(product: Product, quantity = 1): void {
    const currentItems = [...this.cartSubject.getValue().items];
    const existingIndex = currentItems.findIndex(
      item => item.productId === product.id
    );

    if (existingIndex >= 0) {
      // สินค้ามีอยู่แล้ว — เพิ่ม Quantity
      const existing = currentItems[existingIndex];
      currentItems[existingIndex] = {
        ...existing,
        quantity: existing.quantity + quantity,
        total: (existing.quantity + quantity) * existing.price
      };
    } else {
      // สินค้าใหม่ — เพิ่ม Item
      const newItem: CartItem = {
        id: `${product.id}-${Date.now()}`,
        productId: product.id,
        name: product.name,
        price: product.price,
        quantity,
        imageUrl: product.imageUrl,
        total: product.price * quantity
      };
      currentItems.push(newItem);
    }

    this.updateCart(currentItems);
  }

  // ลบสินค้าออกจากตะกร้า
  removeItem(itemId: string): void {
    const currentItems = this.cartSubject.getValue().items;
    const newItems = currentItems.filter(item => item.id !== itemId);
    this.updateCart(newItems);
  }

  // อัปเดต Quantity
  updateQuantity(itemId: string, quantity: number): void {
    if (quantity <= 0) {
      this.removeItem(itemId);
      return;
    }

    const currentItems = [...this.cartSubject.getValue().items];
    const index = currentItems.findIndex(item => item.id === itemId);

    if (index >= 0) {
      currentItems[index] = {
        ...currentItems[index],
        quantity,
        total: currentItems[index].price * quantity
      };
      this.updateCart(currentItems);
    }
  }

  // ล้างตะกร้าทั้งหมด
  clearCart(): void {
    this.updateCart([]);
  }

  // ดูสินค้าในตะกร้าตาม ProductId
  getItem(productId: number): CartItem | undefined {
    return this.cartSubject.getValue().items.find(
      item => item.productId === productId
    );
  }

  // ตรวจสอบว่าสินค้าอยู่ในตะกร้าหรือไม่
  isInCart(productId: number): boolean {
    return !!this.getItem(productId);
  }

  // Getter สำหรับ Snapshot ปัจจุบัน
  get currentCart(): Cart {
    return this.cartSubject.getValue();
  }
}
```

### Step 3: Cart Component

```typescript
// src/app/components/cart/cart.component.ts
import { Component, OnInit } from '@angular/core';
import { Observable } from 'rxjs';
import { CartService } from '../../services/cart.service';
import { Cart, CartItem } from '../../models/cart.model';

@Component({
  selector: 'app-cart',
  templateUrl: './cart.component.html',
  styleUrls: ['./cart.component.css']
})
export class CartComponent implements OnInit {
  cart$!: Observable<Cart>;

  constructor(private cartService: CartService) { }

  ngOnInit(): void {
    this.cart$ = this.cartService.cart$;
  }

  removeItem(itemId: string): void {
    this.cartService.removeItem(itemId);
  }

  updateQuantity(itemId: string, event: Event): void {
    const input = event.target as HTMLInputElement;
    const quantity = parseInt(input.value, 10);
    if (!isNaN(quantity)) {
      this.cartService.updateQuantity(itemId, quantity);
    }
  }

  clearCart(): void {
    if (confirm('ต้องการล้างตะกร้าสินค้าทั้งหมด?')) {
      this.cartService.clearCart();
    }
  }

  checkout(): void {
    // นำไปสู่หน้า Checkout
    console.log('ไปหน้า Checkout', this.cartService.currentCart);
  }

  trackByItemId(index: number, item: CartItem): string {
    return item.id;
  }
}
```

```html
<!-- src/app/components/cart/cart.component.html -->
<div class="cart-container" *ngIf="cart$ | async as cart">

  <div class="cart-header">
    <h2>ตะกร้าสินค้า</h2>
    <span class="badge">{{ cart.itemCount }} ชิ้น</span>
  </div>

  <!-- กรณีตะกร้าว่าง -->
  <div class="cart-empty" *ngIf="cart.items.length === 0">
    <p>ตะกร้าสินค้าว่างเปล่า</p>
    <a routerLink="/products" class="btn btn-primary">เลือกซื้อสินค้า</a>
  </div>

  <!-- รายการสินค้า -->
  <div class="cart-items" *ngIf="cart.items.length > 0">
    <div class="cart-item"
         *ngFor="let item of cart.items; trackBy: trackByItemId">
      <img [src]="item.imageUrl || 'assets/default-product.png'"
           [alt]="item.name"
           class="item-image">

      <div class="item-details">
        <h4>{{ item.name }}</h4>
        <p class="item-price">{{ item.price | currency:'THB':'symbol':'1.2-2' }}</p>
      </div>

      <div class="item-quantity">
        <button (click)="updateQuantity(item.id, $event)"
                [value]="item.quantity - 1"
                class="btn-qty">-</button>
        <input type="number"
               [value]="item.quantity"
               min="1"
               max="99"
               (change)="updateQuantity(item.id, $event)"
               class="qty-input">
        <button (click)="updateQuantity(item.id, $event)"
                [value]="item.quantity + 1"
                class="btn-qty">+</button>
      </div>

      <div class="item-total">
        {{ item.total | currency:'THB':'symbol':'1.2-2' }}
      </div>

      <button (click)="removeItem(item.id)"
              class="btn-remove"
              title="ลบสินค้า">
        ✕
      </button>
    </div>
  </div>

  <!-- สรุปราคา -->
  <div class="cart-summary" *ngIf="cart.items.length > 0">
    <div class="summary-row">
      <span>ราคารวม:</span>
      <span>{{ cart.subtotal | currency:'THB':'symbol':'1.2-2' }}</span>
    </div>
    <div class="summary-row">
      <span>VAT (7%):</span>
      <span>{{ cart.vat | currency:'THB':'symbol':'1.2-2' }}</span>
    </div>
    <div class="summary-row total">
      <span>รวมทั้งสิ้น:</span>
      <span>{{ cart.total | currency:'THB':'symbol':'1.2-2' }}</span>
    </div>

    <div class="cart-actions">
      <button (click)="clearCart()" class="btn btn-secondary">
        ล้างตะกร้า
      </button>
      <button (click)="checkout()" class="btn btn-primary">
        ชำระเงิน
      </button>
    </div>
  </div>

</div>
```

### Step 4: Cart Badge Component (แสดงจำนวนใน Header)

```typescript
// src/app/components/cart-badge/cart-badge.component.ts
import { Component } from '@angular/core';
import { Observable } from 'rxjs';
import { CartService } from '../../services/cart.service';

@Component({
  selector: 'app-cart-badge',
  template: `
    <a routerLink="/cart" class="cart-icon">
      🛒
      <span class="badge" *ngIf="(itemCount$ | async) as count">
        {{ count }}
      </span>
    </a>
  `,
  styles: [`
    .cart-icon { position: relative; text-decoration: none; font-size: 1.5rem; }
    .badge {
      position: absolute; top: -8px; right: -8px;
      background: red; color: white;
      border-radius: 50%; padding: 2px 6px;
      font-size: 0.75rem; font-weight: bold;
    }
  `]
})
export class CartBadgeComponent {
  itemCount$: Observable<number>;

  constructor(private cartService: CartService) {
    this.itemCount$ = this.cartService.itemCount$;
  }
}
```

### Step 5: Add to Cart Button Component

```typescript
// src/app/components/add-to-cart/add-to-cart.component.ts
import { Component, Input } from '@angular/core';
import { CartService } from '../../services/cart.service';
import { Product } from '../../models/product.model';

@Component({
  selector: 'app-add-to-cart',
  template: `
    <div class="add-to-cart">
      <div *ngIf="!isInCart; else alreadyInCart">
        <input type="number" [(ngModel)]="quantity" min="1" max="99" class="qty-input">
        <button (click)="addToCart()" class="btn btn-primary" [disabled]="adding">
          {{ adding ? 'กำลังเพิ่ม...' : 'เพิ่มลงตะกร้า' }}
        </button>
      </div>

      <ng-template #alreadyInCart>
        <span class="in-cart-badge">✓ อยู่ในตะกร้าแล้ว</span>
        <a routerLink="/cart" class="btn btn-secondary btn-sm">ดูตะกร้า</a>
      </ng-template>
    </div>
  `
})
export class AddToCartComponent {
  @Input() product!: Product;

  quantity = 1;
  adding = false;

  constructor(private cartService: CartService) { }

  get isInCart(): boolean {
    return this.cartService.isInCart(this.product.id);
  }

  addToCart(): void {
    this.adding = true;
    this.cartService.addItem(this.product, this.quantity);

    // แสดง Feedback สั้น ๆ
    setTimeout(() => {
      this.adding = false;
    }, 500);
  }
}
```

### Step 6: ลงทะเบียน Module

```typescript
// src/app/app.module.ts
import { NgModule } from '@angular/core';
import { BrowserModule } from '@angular/platform-browser';
import { FormsModule } from '@angular/forms';
import { HttpClientModule } from '@angular/common/http';

import { CartComponent } from './components/cart/cart.component';
import { CartBadgeComponent } from './components/cart-badge/cart-badge.component';
import { AddToCartComponent } from './components/add-to-cart/add-to-cart.component';

@NgModule({
  declarations: [
    CartComponent,
    CartBadgeComponent,
    AddToCartComponent
  ],
  imports: [
    BrowserModule,
    FormsModule,
    HttpClientModule
  ]
})
export class AppModule { }
```

---

## สรุป

| Concept | คำอธิบาย |
|---------|---------|
| `@Injectable()` | Decorator ที่ทำให้ Class สามารถ Inject ได้ |
| `providedIn: 'root'` | Singleton สำหรับทั้ง App |
| `providedIn: Module` | Scoped ใน Feature Module |
| `providers: [...]` | Component-level Instance แยกกัน |
| Constructor Injection | วิธีมาตรฐานในการรับ Service |
| `inject()` function | วิธีใหม่ใน Angular 14+ |
| BehaviorSubject | เก็บ State ที่ Component สามารถ Subscribe ได้ |
| Factory Provider | สร้าง Service ต่างกันตามเงื่อนไข |

---

*เอกสารนี้เป็นส่วนหนึ่งของหลักสูตร Angular — Part 07*
