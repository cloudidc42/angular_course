# Part 98: Angular Best Practices และ Code Quality

## หลักการสำคัญ

Code ที่ดีคือ code ที่อ่านง่าย บำรุงรักษาง่าย และทดสอบได้

---

## 1. Component Design

### DO - สิ่งที่ควรทำ

```typescript
// ✅ Smart Component - จัดการ state และ logic
@Component({
  selector: 'app-product-list',
  template: `
    <app-product-grid
      [products]="products$ | async"
      [isLoading]="isLoading$ | async"
      (addToCart)="addToCart($event)"
    ></app-product-grid>
  `
})
export class ProductListComponent {
  products$ = this.store.select(selectProducts);
  isLoading$ = this.store.select(selectLoading);

  constructor(private store: Store) {}

  addToCart(product: Product): void {
    this.store.dispatch(addToCart({ product }));
  }
}

// ✅ Dumb Component - รับ input, ส่ง output เท่านั้น
@Component({
  selector: 'app-product-grid',
  template: `
    <div class="grid">
      <app-product-card
        *ngFor="let product of products; trackBy: trackById"
        [product]="product"
        (addToCart)="addToCart.emit($event)"
      ></app-product-card>
    </div>
    <app-loading *ngIf="isLoading"></app-loading>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class ProductGridComponent {
  @Input() products: Product[] | null = [];
  @Input() isLoading: boolean | null = false;
  @Output() addToCart = new EventEmitter<Product>();

  trackById = (i: number, p: Product) => p.id;
}
```

### DON'T - สิ่งที่ไม่ควรทำ

```typescript
// ❌ Component ที่ทำทุกอย่าง (God Component)
@Component({ selector: 'app-dashboard' })
export class DashboardComponent implements OnInit {
  products: Product[] = [];
  orders: Order[] = [];
  users: User[] = [];
  analytics: Analytics = {};
  // ... logic ทั้งหมดอยู่ที่นี่
  // component นี้ยาว 1000+ บรรทัด
}

// ❌ Mutate input โดยตรง
@Component({ selector: 'app-item' })
export class ItemComponent {
  @Input() item!: Item;
  
  update(): void {
    this.item.name = 'new name'; // ❌ mutate input!
  }
}

// ✅ Emit event แทน
@Component({ selector: 'app-item' })
export class ItemComponent {
  @Input() item!: Item;
  @Output() updated = new EventEmitter<Item>();
  
  update(): void {
    this.updated.emit({ ...this.item, name: 'new name' }); // ✅
  }
}
```

---

## 2. RxJS Best Practices

```typescript
// ✅ Unsubscribe อย่างถูกวิธี
@Component({ selector: 'app-data' })
export class DataComponent implements OnInit, OnDestroy {
  private destroy$ = new Subject<void>();

  ngOnInit(): void {
    this.dataService.getData()
      .pipe(takeUntil(this.destroy$))
      .subscribe(data => this.data = data);
  }

  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
  }
}

// ✅ ยิ่งดี - ใช้ async pipe
@Component({
  template: `
    <div *ngFor="let item of data$ | async">{{ item.name }}</div>
  `
})
export class DataComponent {
  data$ = this.dataService.getData();
  constructor(private dataService: DataService) {}
}

// ✅ Error handling ใน Observable
this.apiService.getProducts()
  .pipe(
    catchError(error => {
      this.errorService.handle(error);
      return of([]);  // fallback value
    }),
    finalize(() => this.isLoading = false)
  )
  .subscribe(products => this.products = products);

// ❌ อย่าทำ nested subscriptions
this.service.getUser().subscribe(user => {
  this.service.getOrders(user.id).subscribe(orders => { // ❌
    // ...
  });
});

// ✅ ใช้ switchMap แทน
this.service.getUser().pipe(
  switchMap(user => this.service.getOrders(user.id))
).subscribe(orders => this.orders = orders);
```

---

## 3. TypeScript Best Practices

```typescript
// ✅ ใช้ Interface แทน any
interface Product {
  id: number;
  name: string;
  price: number;
  category: ProductCategory;
}

// ✅ Readonly สำหรับ immutable objects
interface Config {
  readonly apiUrl: string;
  readonly timeout: number;
}

// ✅ Union Types
type Status = 'pending' | 'active' | 'inactive' | 'deleted';

// ✅ Generic Types
function getById<T extends { id: number }>(items: T[], id: number): T | undefined {
  return items.find(item => item.id === id);
}

// ✅ Type Guards
function isProduct(obj: unknown): obj is Product {
  return typeof obj === 'object' && obj !== null && 'id' in obj && 'name' in obj;
}

// ❌ ไม่ใช้ any
function processData(data: any): any { return data; }

// ✅ Generic แทน any
function processData<T>(data: T): T { return data; }

// ✅ Optional Chaining และ Nullish Coalescing
const name = user?.profile?.displayName ?? 'ผู้ใช้ไม่ระบุชื่อ';
const count = items?.length ?? 0;
```

---

## 4. Service Design

```typescript
// ✅ Single Responsibility
@Injectable({ providedIn: 'root' })
export class ProductApiService {
  // เฉพาะ HTTP calls
  getProducts(): Observable<Product[]> {
    return this.http.get<Product[]>('/api/products');
  }
}

@Injectable({ providedIn: 'root' })
export class ProductStateService {
  // เฉพาะ state management
  private products$ = new BehaviorSubject<Product[]>([]);
  products = this.products$.asObservable();
  setProducts(products: Product[]): void {
    this.products$.next(products);
  }
}

// ✅ Error handling ใน service
@Injectable({ providedIn: 'root' })
export class ProductService {
  getProducts(): Observable<Product[]> {
    return this.http.get<Product[]>('/api/products').pipe(
      retry(2),
      catchError(this.handleError)
    );
  }

  private handleError(error: HttpErrorResponse): Observable<never> {
    let message = 'เกิดข้อผิดพลาด';
    if (error.status === 404) message = 'ไม่พบข้อมูล';
    if (error.status === 403) message = 'ไม่มีสิทธิ์เข้าถึง';
    return throwError(() => new Error(message));
  }
}
```

---

## 5. Performance Best Practices

```typescript
// ✅ trackBy ใน ngFor
@Component({
  template: `
    <div *ngFor="let item of items; trackBy: trackByFn">
      {{ item.name }}
    </div>
  `
})
export class ListComponent {
  trackByFn(index: number, item: Item): number {
    return item.id; // ใช้ unique identifier
  }
}

// ✅ OnPush Change Detection
@Component({
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `{{ data?.name }}`
})
export class PureComponent {
  @Input() data: DataItem | null = null;
}

// ✅ Lazy Loading
const routes: Routes = [
  {
    path: 'admin',
    loadChildren: () => import('./admin/admin.module').then(m => m.AdminModule),
    canActivate: [AdminGuard]
  }
];

// ✅ Pure Pipes
@Pipe({ name: 'formatPrice', pure: true })
export class FormatPricePipe implements PipeTransform {
  transform(value: number, currency = 'THB'): string {
    return `฿${value.toLocaleString()}`;
  }
}

// ❌ Impure function ใน template
// <div>{{ getExpensiveValue() }}</div>  ❌ เรียกทุก CD cycle

// ✅ คำนวณใน component
get expensiveValue(): string {
  return this.computeOnce(); // เรียกเมื่อ input เปลี่ยนเท่านั้น
}
```

---

## 6. Testing Best Practices

```typescript
// ✅ Unit Test ที่ดี
describe('ProductService', () => {
  let service: ProductService;
  let httpMock: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({
      imports: [HttpClientTestingModule],
      providers: [ProductService]
    });
    service = TestBed.inject(ProductService);
    httpMock = TestBed.inject(HttpTestingController);
  });

  afterEach(() => httpMock.verify());

  it('should fetch products', (done) => {
    const mockProducts: Product[] = [
      { id: 1, name: 'สินค้า A', price: 100, category: 'A' }
    ];

    service.getProducts().subscribe(products => {
      expect(products).toEqual(mockProducts);
      expect(products.length).toBe(1);
      done();
    });

    const req = httpMock.expectOne('/api/products');
    expect(req.request.method).toBe('GET');
    req.flush(mockProducts);
  });

  it('should handle errors', (done) => {
    service.getProducts().subscribe({
      error: (error) => {
        expect(error.message).toBe('ไม่พบข้อมูล');
        done();
      }
    });

    httpMock.expectOne('/api/products').flush(
      { message: 'Not Found' },
      { status: 404, statusText: 'Not Found' }
    );
  });
});

// ✅ Component Test
describe('ProductCardComponent', () => {
  let component: ProductCardComponent;
  let fixture: ComponentFixture<ProductCardComponent>;

  beforeEach(() => {
    TestBed.configureTestingModule({
      declarations: [ProductCardComponent],
      schemas: [NO_ERRORS_SCHEMA]
    });
    fixture = TestBed.createComponent(ProductCardComponent);
    component = fixture.componentInstance;
    component.product = { id: 1, name: 'Test', price: 100, category: 'X' };
    fixture.detectChanges();
  });

  it('should display product name', () => {
    const el = fixture.debugElement.query(By.css('.product-name'));
    expect(el.nativeElement.textContent).toContain('Test');
  });

  it('should emit addToCart when button clicked', () => {
    spyOn(component.addToCart, 'emit');
    const btn = fixture.debugElement.query(By.css('.btn-add'));
    btn.triggerEventHandler('click', null);
    expect(component.addToCart.emit).toHaveBeenCalledWith(component.product);
  });
});
```

---

## 7. Code Review Checklist

```markdown
## Pre-commit Checklist

### Performance
- [ ] ngFor ใช้ trackBy
- [ ] OnPush สำหรับ presentational components
- [ ] Lazy load routes
- [ ] Unsubscribe จาก Observables

### Code Quality
- [ ] ไม่มี any type
- [ ] ไม่มี console.log ใน production code
- [ ] Error handling ครบถ้วน
- [ ] Function มี single responsibility

### Security
- [ ] ไม่มี sensitive data ใน localStorage
- [ ] API keys ไม่อยู่ใน frontend
- [ ] Input validation
- [ ] XSS protection (ใช้ innerText แทน innerHTML)

### Testing
- [ ] Unit tests ผ่าน
- [ ] Test coverage เพียงพอ
- [ ] No broken tests
```

---

## 8. Folder Structure

```
src/
├── app/
│   ├── core/              # Singleton services, guards, interceptors
│   │   ├── auth/
│   │   ├── http/
│   │   └── store/
│   ├── shared/            # Reusable components, pipes, directives
│   │   ├── components/
│   │   ├── pipes/
│   │   └── directives/
│   ├── features/          # Feature modules (lazy loaded)
│   │   ├── products/
│   │   ├── cart/
│   │   └── checkout/
│   └── app-routing.module.ts
└── environments/
```

---

## สรุป Do/Don't

| เรื่อง | DO ✅ | DON'T ❌ |
|-------|-------|----------|
| Types | Interface, Generic | any |
| Component | Single responsibility | God component |
| Observable | async pipe, takeUntil | Manual subscribe |
| Performance | OnPush, trackBy | Default CD |
| Testing | Unit + Integration | No tests |
| Error | Handle ทุก case | Ignore errors |
