# Part 24 — Unit Testing พื้นฐาน

## บทนำ

Unit Testing ช่วยให้เราตรวจสอบว่าโค้ดทำงานถูกต้อง และป้องกัน regression เมื่อมีการแก้ไข Angular ใช้ Jasmine เป็น test framework เริ่มต้น และ Karma เป็น test runner

### ทำไมต้องเขียน Tests?
- ตรวจจับ bugs ได้เร็วก่อน production
- Documentation ที่มีชีวิต (tests บอกว่าโค้ดควรทำอะไร)
- Refactor ได้อย่างมั่นใจ
- CI/CD integration

---

## 1. Jasmine, Karma, Jest

### 1.1 Jasmine

Jasmine เป็น BDD (Behavior-Driven Development) test framework

```typescript
// โครงสร้างพื้นฐานของ Jasmine

describe('Calculator', () => {

  // setup ก่อน test ทุกตัว
  beforeEach(() => {
    // initialize
  });

  // cleanup หลัง test ทุกตัว
  afterEach(() => {
    // cleanup
  });

  // setup ครั้งเดียวก่อน tests ทั้งหมด
  beforeAll(() => {
    // one-time setup
  });

  // cleanup ครั้งเดียวหลัง tests ทั้งหมด
  afterAll(() => {
    // one-time cleanup
  });

  it('should add two numbers', () => {
    expect(1 + 1).toBe(2);
  });

  it('should subtract numbers', () => {
    expect(5 - 3).toBe(2);
  });

  // Skip test
  xit('should multiply', () => {
    // ไม่รัน test นี้
  });

  // Run only this test (ใช้ชั่วคราว อย่าลืมลบ!)
  // fit('should divide', () => { ... });
});
```

### 1.2 Jasmine Matchers

```typescript
// Equality
expect(value).toBe(2);                    // ===
expect(value).toEqual({ a: 1 });          // deep equal
expect(value).not.toBe(0);

// Truthiness
expect(value).toBeTruthy();               // truthy
expect(value).toBeFalsy();                // falsy
expect(value).toBeNull();
expect(value).toBeUndefined();
expect(value).toBeDefined();

// Numbers
expect(3.14).toBeCloseTo(3.14, 2);        // ใกล้เคียง (decimal places)
expect(10).toBeGreaterThan(5);
expect(10).toBeGreaterThanOrEqual(10);
expect(5).toBeLessThan(10);

// Strings
expect('Hello World').toContain('World');
expect('Hello').toMatch(/^H/);            // regex
expect('Hello').toMatch('He');

// Arrays
expect([1, 2, 3]).toContain(2);
expect([1, 2, 3]).toHaveSize(3);
expect([1, 2, 3]).toEqual(jasmine.arrayContaining([1, 3]));  // ไม่สนลำดับ

// Objects
expect(obj).toEqual(jasmine.objectContaining({ key: 'value' }));

// Functions / Exceptions
expect(() => { throw new Error('fail') }).toThrow();
expect(() => { throw new Error('fail') }).toThrowError('fail');

// Spies
expect(spy).toHaveBeenCalled();
expect(spy).toHaveBeenCalledTimes(2);
expect(spy).toHaveBeenCalledWith('arg1', 'arg2');
```

### 1.3 รัน Tests

```bash
# รัน tests ด้วย Karma (default)
ng test

# รัน tests แบบ headless (สำหรับ CI)
ng test --no-watch --browsers=ChromeHeadless

# รัน tests แบบ coverage
ng test --code-coverage

# ดู coverage report
open coverage/angular-app/index.html
```

---

## 2. TestBed Setup

### 2.1 TestBed คืออะไร

TestBed เป็น utility ของ Angular ที่สร้าง testing environment คล้ายกับ NgModule แต่ใช้สำหรับ testing

```typescript
import { TestBed } from '@angular/core/testing';

describe('MyService', () => {

  let service: MyService;

  beforeEach(() => {
    TestBed.configureTestingModule({
      // Declare components
      declarations: [],

      // Import modules
      imports: [],

      // Provide services
      providers: [
        MyService,
        // Override providers
        { provide: OtherService, useClass: MockOtherService },
        { provide: TOKEN, useValue: 'mock-value' }
      ]
    });

    // Get service instance
    service = TestBed.inject(MyService);
  });

  it('should be created', () => {
    expect(service).toBeTruthy();
  });
});
```

---

## 3. Testing Components

### 3.1 Component Testing พื้นฐาน

```typescript
// src/app/components/greeting/greeting.component.spec.ts
import { ComponentFixture, TestBed } from '@angular/core/testing';
import { By } from '@angular/platform-browser';
import { GreetingComponent } from './greeting.component';

describe('GreetingComponent', () => {
  let component: GreetingComponent;
  let fixture: ComponentFixture<GreetingComponent>;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [GreetingComponent]  // สำหรับ standalone component
    }).compileComponents();

    fixture = TestBed.createComponent(GreetingComponent);
    component = fixture.componentInstance;
    fixture.detectChanges();  // trigger initial change detection
  });

  it('should create the component', () => {
    expect(component).toBeTruthy();
  });

  it('should render greeting', () => {
    component.name = 'สมชาย';
    fixture.detectChanges();  // trigger change detection

    const compiled = fixture.nativeElement as HTMLElement;
    expect(compiled.querySelector('h1')?.textContent).toContain('สมชาย');
  });

  it('should use query by CSS selector', () => {
    component.name = 'Angular';
    fixture.detectChanges();

    // ใช้ By.css สำหรับ DebugElement
    const h1 = fixture.debugElement.query(By.css('h1'));
    expect(h1.nativeElement.textContent).toContain('Angular');
  });

  it('should handle button click', () => {
    const button = fixture.debugElement.query(By.css('button'));
    button.triggerEventHandler('click', null);
    fixture.detectChanges();

    expect(component.clickCount).toBe(1);
  });
});
```

### 3.2 Greeting Component ที่จะ test

```typescript
// greeting.component.ts
import { Component, Input } from '@angular/core';
import { CommonModule } from '@angular/common';

@Component({
  selector: 'app-greeting',
  standalone: true,
  imports: [CommonModule],
  template: `
    <h1>สวัสดี {{ name }}!</h1>
    <p>คุณคลิกแล้ว {{ clickCount }} ครั้ง</p>
    <button (click)="onClick()">คลิก</button>
    <div *ngIf="showMessage" class="message">Hello!</div>
  `
})
export class GreetingComponent {
  @Input() name = 'World';
  clickCount = 0;
  showMessage = false;

  onClick(): void {
    this.clickCount++;
    this.showMessage = true;
  }
}
```

### 3.3 Testing Component ที่มี Service

```typescript
// product-list.component.spec.ts
import { ComponentFixture, TestBed } from '@angular/core/testing';
import { of, throwError } from 'rxjs';
import { ProductListComponent } from './product-list.component';
import { ProductService } from '../../services/product.service';

// Mock Service
const mockProductService = {
  getProducts: jasmine.createSpy('getProducts').and.returnValue(
    of([
      { id: 1, name: 'Product A', price: 100 },
      { id: 2, name: 'Product B', price: 200 }
    ])
  )
};

describe('ProductListComponent', () => {
  let component: ProductListComponent;
  let fixture: ComponentFixture<ProductListComponent>;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [ProductListComponent],
      providers: [
        // Override ProductService ด้วย mock
        { provide: ProductService, useValue: mockProductService }
      ]
    }).compileComponents();

    fixture = TestBed.createComponent(ProductListComponent);
    component = fixture.componentInstance;
    fixture.detectChanges();
  });

  it('should load products on init', () => {
    expect(mockProductService.getProducts).toHaveBeenCalled();
    expect(component.products.length).toBe(2);
  });

  it('should display product names', () => {
    fixture.detectChanges();
    const productElements = fixture.nativeElement.querySelectorAll('.product-name');
    expect(productElements[0].textContent).toContain('Product A');
    expect(productElements[1].textContent).toContain('Product B');
  });

  it('should handle error', () => {
    mockProductService.getProducts.and.returnValue(
      throwError(() => new Error('API Error'))
    );

    component.ngOnInit();
    fixture.detectChanges();

    expect(component.error).toBe('ไม่สามารถโหลดสินค้าได้');
  });
});
```

---

## 4. Testing Services

### 4.1 Testing Simple Service

```typescript
// src/app/services/math.service.ts
import { Injectable } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class MathService {
  add(a: number, b: number): number {
    return a + b;
  }

  subtract(a: number, b: number): number {
    return a - b;
  }

  multiply(a: number, b: number): number {
    return a * b;
  }

  divide(a: number, b: number): number {
    if (b === 0) {
      throw new Error('Cannot divide by zero');
    }
    return a / b;
  }

  factorial(n: number): number {
    if (n < 0) throw new Error('Factorial is not defined for negative numbers');
    if (n === 0 || n === 1) return 1;
    return n * this.factorial(n - 1);
  }
}
```

```typescript
// src/app/services/math.service.spec.ts
import { TestBed } from '@angular/core/testing';
import { MathService } from './math.service';

describe('MathService', () => {
  let service: MathService;

  beforeEach(() => {
    TestBed.configureTestingModule({});
    service = TestBed.inject(MathService);
  });

  it('should be created', () => {
    expect(service).toBeTruthy();
  });

  describe('add()', () => {
    it('should add two positive numbers', () => {
      expect(service.add(2, 3)).toBe(5);
    });

    it('should add negative numbers', () => {
      expect(service.add(-2, -3)).toBe(-5);
    });

    it('should add zero', () => {
      expect(service.add(5, 0)).toBe(5);
    });
  });

  describe('divide()', () => {
    it('should divide two numbers', () => {
      expect(service.divide(10, 2)).toBe(5);
    });

    it('should throw error when dividing by zero', () => {
      expect(() => service.divide(10, 0)).toThrowError('Cannot divide by zero');
    });
  });

  describe('factorial()', () => {
    it('should return 1 for 0', () => {
      expect(service.factorial(0)).toBe(1);
    });

    it('should calculate factorial correctly', () => {
      expect(service.factorial(5)).toBe(120);
    });

    it('should throw for negative numbers', () => {
      expect(() => service.factorial(-1)).toThrow();
    });
  });
});
```

### 4.2 Testing HTTP Service

```typescript
// src/app/services/product.service.ts
import { Injectable } from '@angular/core';
import { HttpClient, HttpParams } from '@angular/common/http';
import { Observable } from 'rxjs';
import { map, catchError } from 'rxjs/operators';

export interface Product {
  id: number;
  name: string;
  price: number;
  category: string;
}

@Injectable({ providedIn: 'root' })
export class ProductService {
  private apiUrl = '/api/products';

  constructor(private http: HttpClient) {}

  getProducts(category?: string): Observable<Product[]> {
    let params = new HttpParams();
    if (category) {
      params = params.set('category', category);
    }
    return this.http.get<Product[]>(this.apiUrl, { params });
  }

  getProduct(id: number): Observable<Product> {
    return this.http.get<Product>(`${this.apiUrl}/${id}`);
  }

  createProduct(product: Omit<Product, 'id'>): Observable<Product> {
    return this.http.post<Product>(this.apiUrl, product);
  }

  updateProduct(id: number, product: Partial<Product>): Observable<Product> {
    return this.http.patch<Product>(`${this.apiUrl}/${id}`, product);
  }

  deleteProduct(id: number): Observable<void> {
    return this.http.delete<void>(`${this.apiUrl}/${id}`);
  }
}
```

```typescript
// src/app/services/product.service.spec.ts
import { TestBed } from '@angular/core/testing';
import {
  HttpClientTestingModule,
  HttpTestingController
} from '@angular/common/http/testing';
import { ProductService, Product } from './product.service';

describe('ProductService', () => {
  let service: ProductService;
  let httpMock: HttpTestingController;

  const mockProducts: Product[] = [
    { id: 1, name: 'Product A', price: 100, category: 'Electronics' },
    { id: 2, name: 'Product B', price: 200, category: 'Books' }
  ];

  beforeEach(() => {
    TestBed.configureTestingModule({
      imports: [HttpClientTestingModule],
      providers: [ProductService]
    });

    service = TestBed.inject(ProductService);
    httpMock = TestBed.inject(HttpTestingController);
  });

  afterEach(() => {
    // ตรวจสอบว่าไม่มี request ค้างอยู่
    httpMock.verify();
  });

  describe('getProducts()', () => {
    it('should return products', () => {
      service.getProducts().subscribe(products => {
        expect(products.length).toBe(2);
        expect(products[0].name).toBe('Product A');
      });

      const req = httpMock.expectOne('/api/products');
      expect(req.request.method).toBe('GET');
      req.flush(mockProducts);
    });

    it('should pass category as query param', () => {
      service.getProducts('Electronics').subscribe();

      const req = httpMock.expectOne(
        req => req.url === '/api/products' &&
               req.params.get('category') === 'Electronics'
      );
      req.flush([mockProducts[0]]);
    });

    it('should handle HTTP error', () => {
      service.getProducts().subscribe({
        error: (error) => {
          expect(error.status).toBe(500);
        }
      });

      const req = httpMock.expectOne('/api/products');
      req.flush('Server Error', { status: 500, statusText: 'Internal Server Error' });
    });
  });

  describe('createProduct()', () => {
    it('should send POST request', () => {
      const newProduct = { name: 'New Product', price: 300, category: 'Tech' };
      const createdProduct = { id: 3, ...newProduct };

      service.createProduct(newProduct).subscribe(product => {
        expect(product.id).toBe(3);
      });

      const req = httpMock.expectOne('/api/products');
      expect(req.request.method).toBe('POST');
      expect(req.request.body).toEqual(newProduct);
      req.flush(createdProduct);
    });
  });

  describe('deleteProduct()', () => {
    it('should send DELETE request', () => {
      service.deleteProduct(1).subscribe();

      const req = httpMock.expectOne('/api/products/1');
      expect(req.request.method).toBe('DELETE');
      req.flush(null);
    });
  });
});
```

---

## 5. Testing Pipes

```typescript
// src/app/pipes/currency-thai.pipe.ts
import { Pipe, PipeTransform } from '@angular/core';

@Pipe({
  name: 'currencyThai',
  standalone: true
})
export class CurrencyThaiPipe implements PipeTransform {
  transform(value: number, currencySymbol = '฿'): string {
    if (value === null || value === undefined) return '';
    return `${currencySymbol}${value.toLocaleString('th-TH', {
      minimumFractionDigits: 2,
      maximumFractionDigits: 2
    })}`;
  }
}
```

```typescript
// src/app/pipes/currency-thai.pipe.spec.ts
import { CurrencyThaiPipe } from './currency-thai.pipe';

describe('CurrencyThaiPipe', () => {
  let pipe: CurrencyThaiPipe;

  beforeEach(() => {
    pipe = new CurrencyThaiPipe();
  });

  it('should create an instance', () => {
    expect(pipe).toBeTruthy();
  });

  it('should format number with default symbol', () => {
    const result = pipe.transform(1000);
    expect(result).toContain('฿');
    expect(result).toContain('1,000.00');
  });

  it('should use custom symbol', () => {
    const result = pipe.transform(500, '$');
    expect(result).toContain('$');
  });

  it('should handle null', () => {
    expect(pipe.transform(null as unknown as number)).toBe('');
  });

  it('should handle zero', () => {
    const result = pipe.transform(0);
    expect(result).toContain('0.00');
  });
});
```

---

## 6. Mock Dependencies

### 6.1 Jasmine Spies

```typescript
// ประเภทของ Spies

// 1. Spy ที่ call through (เรียก original function ด้วย)
const spy = spyOn(service, 'method').and.callThrough();

// 2. Spy ที่ return value
const spy2 = spyOn(service, 'method').and.returnValue('mocked value');

// 3. Spy ที่ return Observable
const spy3 = spyOn(service, 'getData').and.returnValue(of(mockData));

// 4. Spy ที่ return Promise
const spy4 = spyOn(service, 'fetchData').and.returnValue(
  Promise.resolve(mockData)
);

// 5. Spy ที่ throw error
const spy5 = spyOn(service, 'riskyMethod').and.throwError('Error!');

// 6. Spy ที่ call fake function
const spy6 = spyOn(service, 'method').and.callFake((arg) => {
  return `Fake: ${arg}`;
});

// ตรวจสอบ spy calls
expect(spy).toHaveBeenCalled();
expect(spy).toHaveBeenCalledTimes(2);
expect(spy).toHaveBeenCalledWith('expected arg');
expect(spy.calls.count()).toBe(2);
expect(spy.calls.first().args).toEqual(['first arg']);
expect(spy.calls.mostRecent().returnValue).toBe('result');
```

### 6.2 Mock Services

```typescript
// สร้าง Mock Service แบบ class
class MockUserService {
  currentUser$ = of({ id: 1, name: 'Test User', role: 'admin' });

  getUser = jasmine.createSpy('getUser').and.returnValue(
    of({ id: 1, name: 'Test User', role: 'admin' })
  );

  login = jasmine.createSpy('login').and.returnValue(
    of({ token: 'mock-token' })
  );

  logout = jasmine.createSpy('logout');
}

// ใช้ใน TestBed
TestBed.configureTestingModule({
  providers: [
    { provide: UserService, useClass: MockUserService }
  ]
});

// หรือใช้ useValue กับ object literal
const mockUserService = {
  currentUser$: of({ id: 1, name: 'Test User' }),
  getUser: jasmine.createSpy('getUser').and.returnValue(of(mockUser)),
  login: jasmine.createSpy('login').and.returnValue(of({ token: 'mock' }))
};

TestBed.configureTestingModule({
  providers: [
    { provide: UserService, useValue: mockUserService }
  ]
});
```

---

## 7. Workshop: Testing Product Service

```typescript
// src/app/services/product-service-advanced.service.ts
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { BehaviorSubject, Observable } from 'rxjs';
import { tap, catchError, map } from 'rxjs/operators';

export interface Product {
  id: number;
  name: string;
  price: number;
  category: string;
  stock: number;
}

export interface ProductState {
  products: Product[];
  loading: boolean;
  error: string | null;
}

@Injectable({ providedIn: 'root' })
export class ProductServiceAdvanced {

  private stateSubject = new BehaviorSubject<ProductState>({
    products: [],
    loading: false,
    error: null
  });

  state$ = this.stateSubject.asObservable();
  products$ = this.state$.pipe(map(s => s.products));
  loading$ = this.state$.pipe(map(s => s.loading));

  constructor(private http: HttpClient) {}

  loadProducts(): Observable<Product[]> {
    this.updateState({ loading: true, error: null });

    return this.http.get<Product[]>('/api/products').pipe(
      tap(products => {
        this.updateState({ products, loading: false });
      }),
      catchError(error => {
        this.updateState({
          loading: false,
          error: 'Failed to load products'
        });
        throw error;
      })
    );
  }

  addProduct(product: Omit<Product, 'id'>): Observable<Product> {
    return this.http.post<Product>('/api/products', product).pipe(
      tap(newProduct => {
        const current = this.stateSubject.getValue();
        this.updateState({
          products: [...current.products, newProduct]
        });
      })
    );
  }

  private updateState(partial: Partial<ProductState>): void {
    this.stateSubject.next({
      ...this.stateSubject.getValue(),
      ...partial
    });
  }

  getProductById(id: number): Product | undefined {
    return this.stateSubject.getValue().products.find(p => p.id === id);
  }
}
```

```typescript
// src/app/services/product-service-advanced.service.spec.ts
import { TestBed, fakeAsync, tick } from '@angular/core/testing';
import {
  HttpClientTestingModule,
  HttpTestingController
} from '@angular/common/http/testing';
import { ProductServiceAdvanced, Product } from './product-service-advanced.service';

describe('ProductServiceAdvanced', () => {
  let service: ProductServiceAdvanced;
  let httpMock: HttpTestingController;

  const mockProducts: Product[] = [
    { id: 1, name: 'Laptop', price: 35000, category: 'Electronics', stock: 10 },
    { id: 2, name: 'Mouse',  price: 800,   category: 'Electronics', stock: 50 },
    { id: 3, name: 'Book',   price: 350,   category: 'Books',        stock: 100 }
  ];

  beforeEach(() => {
    TestBed.configureTestingModule({
      imports: [HttpClientTestingModule],
      providers: [ProductServiceAdvanced]
    });

    service = TestBed.inject(ProductServiceAdvanced);
    httpMock = TestBed.inject(HttpTestingController);
  });

  afterEach(() => {
    httpMock.verify();
  });

  // --- State Tests ---

  describe('Initial State', () => {
    it('should start with empty products', () => {
      service.state$.subscribe(state => {
        expect(state.products).toEqual([]);
        expect(state.loading).toBeFalse();
        expect(state.error).toBeNull();
      });
    });
  });

  // --- loadProducts Tests ---

  describe('loadProducts()', () => {
    it('should set loading to true during request', () => {
      let loadingState: boolean | undefined;

      service.loading$.subscribe(loading => loadingState = loading);
      service.loadProducts().subscribe();

      // ก่อน response — loading ควร true
      expect(loadingState).toBeTrue();

      httpMock.expectOne('/api/products').flush(mockProducts);

      // หลัง response — loading ควร false
      expect(loadingState).toBeFalse();
    });

    it('should update products state on success', () => {
      service.loadProducts().subscribe();
      httpMock.expectOne('/api/products').flush(mockProducts);

      service.products$.subscribe(products => {
        expect(products.length).toBe(3);
        expect(products[0].name).toBe('Laptop');
      });
    });

    it('should set error state on failure', () => {
      service.loadProducts().subscribe({
        error: () => {}
      });

      httpMock.expectOne('/api/products').flush(
        'Server Error',
        { status: 500, statusText: 'Internal Server Error' }
      );

      service.state$.subscribe(state => {
        expect(state.error).toBe('Failed to load products');
        expect(state.loading).toBeFalse();
      });
    });
  });

  // --- addProduct Tests ---

  describe('addProduct()', () => {
    it('should add product to state', fakeAsync(() => {
      const newProduct = {
        name: 'Keyboard',
        price: 2500,
        category: 'Electronics',
        stock: 25
      };
      const createdProduct = { id: 4, ...newProduct };

      // Load initial products first
      service.loadProducts().subscribe();
      httpMock.expectOne('/api/products').flush(mockProducts);
      tick();

      // Add new product
      service.addProduct(newProduct).subscribe();
      httpMock.expectOne('/api/products').flush(createdProduct);
      tick();

      service.products$.subscribe(products => {
        expect(products.length).toBe(4);
        expect(products[3].name).toBe('Keyboard');
      });
    }));
  });

  // --- getProductById Tests ---

  describe('getProductById()', () => {
    beforeEach(() => {
      service.loadProducts().subscribe();
      httpMock.expectOne('/api/products').flush(mockProducts);
    });

    it('should find existing product', () => {
      const product = service.getProductById(1);
      expect(product).toBeDefined();
      expect(product?.name).toBe('Laptop');
    });

    it('should return undefined for non-existing product', () => {
      const product = service.getProductById(999);
      expect(product).toBeUndefined();
    });
  });

  // --- Async Tests ---

  describe('Async operations', () => {
    it('should work with fakeAsync', fakeAsync(() => {
      let products: Product[] = [];

      service.loadProducts().subscribe();
      httpMock.expectOne('/api/products').flush(mockProducts);

      service.products$.subscribe(p => products = p);
      tick();

      expect(products.length).toBe(3);
    }));
  });
});
```

---

## สรุปบทที่ 24

### Testing Pyramid

```
         /\
        /  \
       /  E2E\     (น้อย — เร็ว/ช้าที่สุด)
      /--------\
     / Integration\  (ปานกลาง)
    /  Tests      \
   /----------------\
  /   Unit Tests     \  (มาก — เร็วที่สุด)
 /____________________\
```

### Best Practices

1. **Test Isolation** — แต่ละ test ควร independent กัน
2. **AAA Pattern** — Arrange, Act, Assert
3. **Mock Dependencies** — ใช้ mock แทน real services
4. **ตั้งชื่อ test ที่สื่อความหมาย** — `should [expected behavior] when [condition]`
5. **Test edge cases** — null, undefined, empty arrays, errors
6. **ใช้ `fakeAsync/tick`** — สำหรับ async operations ที่ใช้ timer
7. **ใช้ `HttpTestingController`** — สำหรับ HTTP requests
8. **Code Coverage ไม่ใช่เป้าหมาย** — quality > quantity

| Tool | ใช้สำหรับ |
|------|---------|
| `TestBed` | Setup testing environment |
| `ComponentFixture` | Test components |
| `HttpTestingController` | Mock HTTP requests |
| `fakeAsync/tick` | Fake timer-based async |
| `jasmine.createSpy` | Create spy functions |
| `spyOn` | Spy on existing methods |
| `By.css` | Query DOM elements |
