# Part 99: Top 50 Angular Interview Questions และคำตอบ

## คำถามระดับ Basic (1-15)

### 1. Angular คืออะไร และต่างจาก AngularJS อย่างไร?

Angular (2+) คือ TypeScript-based framework สำหรับสร้าง SPA จาก Google
- Angular: TypeScript, Component-based, Hierarchical DI, RxJS
- AngularJS: JavaScript, MVC, $scope, Two-way binding แบบเก่า

### 2. Component คืออะไร?

Component คือส่วนประกอบหลักของ Angular app ประกอบด้วย:
- Template (HTML)
- Class (TypeScript)
- Metadata (@Component decorator)
- Styles (CSS/SCSS)

```typescript
@Component({
  selector: 'app-hello',
  template: `<h1>สวัสดี {{ name }}</h1>`,
  styles: [`h1 { color: blue; }`]
})
export class HelloComponent {
  name = 'Angular';
}
```

### 3. Module คืออะไร?

NgModule จัดกลุ่ม components, directives, pipes และ services ที่เกี่ยวข้องกัน

```typescript
@NgModule({
  declarations: [MyComponent],     // components/directives/pipes
  imports: [CommonModule],          // modules อื่น
  providers: [MyService],           // services
  exports: [MyComponent]            // เปิดให้ module อื่นใช้
})
export class MyModule {}
```

### 4. Decorators ที่ใช้บ่อยมีอะไรบ้าง?

| Decorator | ใช้สำหรับ |
|-----------|---------|
| @Component | สร้าง component |
| @Injectable | สร้าง service |
| @NgModule | สร้าง module |
| @Input | รับค่าจาก parent |
| @Output | ส่งค่าไป parent |
| @ViewChild | เข้าถึง child |
| @HostListener | ฟัง DOM events |
| @Pipe | สร้าง pipe |

### 5. Data Binding มีกี่แบบ?

```html
<!-- 1. Interpolation (one-way) -->
{{ title }}

<!-- 2. Property Binding (one-way) -->
[src]="imageUrl"

<!-- 3. Event Binding (one-way) -->
(click)="handleClick()"

<!-- 4. Two-way Binding -->
[(ngModel)]="username"
```

### 6. Directive คืออะไร? มีกี่ประเภท?

Directive คือ instruction ที่บอก Angular ว่าต้องทำอะไรกับ DOM element

- **Component Directive**: มี template (AppComponent)
- **Structural Directive**: เปลี่ยนโครงสร้าง DOM (*ngIf, *ngFor, *ngSwitch)
- **Attribute Directive**: เปลี่ยน appearance/behavior ([ngClass], [ngStyle])

### 7. Service และ Dependency Injection คืออะไร?

Service คือ class ที่มี business logic หรือ shared functionality
DI คือ pattern ที่ Angular inject dependencies ให้อัตโนมัติ

```typescript
@Injectable({ providedIn: 'root' }) // Singleton
export class UserService {
  getUsers(): Observable<User[]> {
    return this.http.get<User[]>('/api/users');
  }
}

@Component({...})
export class MyComponent {
  constructor(private userService: UserService) {} // Auto-injected
}
```

### 8. Lifecycle Hooks มีอะไรบ้าง?

```typescript
ngOnChanges()     // Input เปลี่ยน
ngOnInit()        // หลัง constructor, inputs พร้อม
ngDoCheck()       // ทุก change detection cycle
ngAfterContentInit()    // หลัง ng-content
ngAfterContentChecked() // หลัง content checked
ngAfterViewInit()       // หลัง view render
ngAfterViewChecked()    // หลัง view checked
ngOnDestroy()     // ก่อน component ถูก destroy
```

### 9. Pipe คืออะไร?

Pipe transform ค่าใน template โดยไม่เปลี่ยนค่าต้นฉบับ

```html
{{ price | currency:'THB' }}
{{ date | date:'dd/MM/yyyy' }}
{{ name | uppercase }}
{{ description | slice:0:100 }}
{{ value | async }}
```

### 10. Lazy Loading คืออะไร?

Lazy loading โหลด module เมื่อ navigate ไปยัง route นั้น ลด initial bundle size

```typescript
const routes: Routes = [
  {
    path: 'admin',
    loadChildren: () => import('./admin/admin.module').then(m => m.AdminModule)
  }
];
```

### 11. Router และ Navigation

```typescript
// Navigate ด้วย RouterLink
<a [routerLink]="['/products', product.id]">ดูสินค้า</a>

// Navigate ด้วย code
this.router.navigate(['/products', id]);
this.router.navigateByUrl('/products?page=2');
```

### 12. Forms มีกี่แบบ?

- **Template-driven**: ใช้ ngModel ใน template, ง่ายกว่า
- **Reactive**: ใช้ FormGroup/FormControl, ทดสอบได้ดีกว่า

### 13. HttpClient คืออะไร?

Angular's HTTP client สำหรับเรียก REST API

```typescript
this.http.get<Product[]>('/api/products')
  .pipe(catchError(error => throwError(() => error)))
  .subscribe(products => this.products = products);
```

### 14. ViewEncapsulation คืออะไร?

- **Emulated** (default): CSS scoped ด้วย attribute selectors
- **None**: CSS global (no encapsulation)
- **ShadowDom**: ใช้ native Shadow DOM

### 15. AOT vs JIT คืออะไร?

- **JIT** (Just-in-Time): compile ใน browser ขณะรัน (development)
- **AOT** (Ahead-of-Time): compile ก่อน deploy (production), เร็วกว่า, ขนาดเล็กกว่า

---

## คำถามระดับ Intermediate (16-35)

### 16. Change Detection คืออะไร?

Angular ตรวจสอบ component tree เมื่อ event เกิดขึ้น และ update DOM

```typescript
// OnPush - อัพเดทเฉพาะเมื่อ input เปลี่ยน
@Component({ changeDetection: ChangeDetectionStrategy.OnPush })
export class OptimizedComponent {
  @Input() data!: Data;
}
```

### 17. RxJS Observable vs Promise

| Observable | Promise |
|-----------|---------|
| Lazy | Eager |
| Multiple values | Single value |
| Cancellable | ไม่ cancellable |
| Operators | ไม่มี |

### 18. Subject ต่างจาก Observable อย่างไร?

Subject คือ Observable ที่ emit ได้เอง (both Observable and Observer)

```typescript
const subject = new Subject<string>();
subject.subscribe(v => console.log('A:', v));
subject.next('hello'); // emit
subject.subscribe(v => console.log('B:', v)); // subscribe หลัง emit จะไม่ได้ค่าเก่า

// BehaviorSubject: มีค่าเริ่มต้น, subscriber ใหม่ได้ค่าล่าสุด
const bs = new BehaviorSubject('initial');

// ReplaySubject: replay N ค่าล่าสุด
const rs = new ReplaySubject(3);
```

### 19. Guards มีกี่ประเภท?

```typescript
canActivate     // ก่อน navigate เข้า route
canActivateChild // ก่อน navigate เข้า child routes
canDeactivate   // ก่อน navigate ออกจาก route
canLoad         // ก่อน lazy load module
resolve         // โหลด data ก่อน activate route
```

### 20. Interceptor คืออะไร?

Interceptor ดักจับ HTTP requests/responses

```typescript
@Injectable()
export class AuthInterceptor implements HttpInterceptor {
  intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    const token = this.authService.getToken();
    const cloned = req.clone({
      headers: req.headers.set('Authorization', `Bearer ${token}`)
    });
    return next.handle(cloned);
  }
}
```

### 21. ControlValueAccessor คืออะไร?

Interface สำหรับสร้าง custom form control ที่ใช้กับ ngModel/formControl ได้

```typescript
@Component({ providers: [{ provide: NG_VALUE_ACCESSOR, useExisting: forwardRef(() => RatingComponent), multi: true }] })
export class RatingComponent implements ControlValueAccessor {
  value = 0;
  onChange = (val: number) => {};
  onTouched = () => {};
  
  writeValue(val: number): void { this.value = val; }
  registerOnChange(fn: any): void { this.onChange = fn; }
  registerOnTouched(fn: any): void { this.onTouched = fn; }
}
```

### 22. NgZone คืออะไร?

NgZone คือ service ที่ Angular ใช้ track async operations เพื่อ trigger change detection

```typescript
// รัน outside zone (ปรับปรุง performance)
this.ngZone.runOutsideAngular(() => {
  setInterval(() => this.drawCanvas(), 16);
});

// กลับเข้า zone เมื่อต้อง update UI
this.ngZone.run(() => {
  this.data = newData; // trigger CD
});
```

### 23. Content Projection คืออะไร?

ng-content ให้ parent ส่ง content เข้า component

```html
<!-- Card Component -->
<div class="card">
  <ng-content select="[card-header]"></ng-content>
  <ng-content></ng-content>
</div>

<!-- ใช้งาน -->
<app-card>
  <h2 card-header>หัวเรื่อง</h2>
  <p>เนื้อหา</p>
</app-card>
```

### 24. Async Pipe คืออะไร?

Async pipe subscribe Observable อัตโนมัติ และ unsubscribe เมื่อ component destroy

```html
<div *ngIf="user$ | async as user">{{ user.name }}</div>
<li *ngFor="let item of items$ | async">{{ item }}</li>
```

### 25. ViewChild และ ContentChild ต่างกันอย่างไร?

- **ViewChild**: เข้าถึง element ใน template ของ component เอง
- **ContentChild**: เข้าถึง element ที่ถูก project เข้ามาด้วย ng-content

### 26. InjectionToken คืออะไร?

InjectionToken สร้าง DI token สำหรับ inject ค่าที่ไม่ใช่ class

```typescript
const API_URL = new InjectionToken<string>('API_URL');

// Provide
providers: [{ provide: API_URL, useValue: 'https://api.example.com' }]

// Inject
constructor(@Inject(API_URL) private apiUrl: string) {}
```

### 27. forRoot() และ forChild() ต่างกันอย่างไร?

- **forRoot()**: ให้ใช้ใน AppModule เพื่อสร้าง singleton service
- **forChild()**: ให้ใช้ใน feature modules ที่ไม่สร้าง service ซ้ำ

### 28. Signals คืออะไร (Angular 16+)?

Signals เป็น reactive primitive ใหม่ของ Angular

```typescript
// Signal
const count = signal(0);
count.set(1);         // set
count.update(n => n + 1); // update
count();              // read

// Computed
const doubled = computed(() => count() * 2);

// Effect
effect(() => console.log('count changed:', count()));
```

### 29. Standalone Components (Angular 14+)?

Component ที่ไม่ต้องอยู่ใน NgModule

```typescript
@Component({
  standalone: true,
  imports: [CommonModule, RouterModule],
  selector: 'app-home',
  template: `<h1>Home</h1>`
})
export class HomeComponent {}
```

### 30. defer block (Angular 17+)?

```html
@defer (on viewport) {
  <app-heavy-component />
} @placeholder {
  <div>กำลังโหลด...</div>
} @loading (minimum 500ms) {
  <app-spinner />
} @error {
  <p>โหลดไม่ได้</p>
}
```

### 31. OnPush Change Detection ทำงานอย่างไร?

Component จะ check เฉพาะเมื่อ:
1. Input reference เปลี่ยน
2. Component เอง emit event
3. Observable ส่งค่า (ใช้ async pipe)
4. markForCheck() หรือ detectChanges() ถูกเรียก

### 32. Tree Shaking คืออะไร?

กระบวนการ remove code ที่ไม่ได้ใช้ออกจาก bundle โดย Webpack/Rollup
Angular ใช้ Pure annotations และ side-effect free code

### 33. Renderer2 vs ElementRef?

- **ElementRef**: เข้าถึง DOM โดยตรง (ควรหลีกเลี่ยง)
- **Renderer2**: เป็น abstraction layer ที่ปลอดภัยกว่า, รองรับ SSR

### 34. Server-Side Rendering (SSR)?

Angular Universal render app บน server เพื่อ SEO และ performance

```bash
ng add @angular/ssr
```

### 35. Testing: Unit vs Integration vs E2E?

- **Unit**: test service/component แยกกัน (Jest/Jasmine)
- **Integration**: test หลาย components ร่วมกัน
- **E2E**: test ทั้ง flow ใน browser จริง (Cypress/Playwright)

---

## คำถามระดับ Advanced (36-50)

### 36. State Management options?

- **NgRx**: Redux pattern, complex apps
- **Akita**: simpler, entity management
- **NgXs**: decorator-based
- **Signals**: built-in, Angular 16+
- **Services + BehaviorSubject**: simple apps

### 37. Memory Leaks ใน Angular?

สาเหตุ:
1. ไม่ unsubscribe Observables
2. Event listeners ที่ไม่ remove
3. setInterval ที่ไม่ clearInterval

แก้ไข: takeUntil, async pipe, ngOnDestroy

### 38. Zone.js คืออะไร?

Zone.js monkey-patches async APIs (setTimeout, Promise, etc.) เพื่อให้ Angular รู้ว่าต้อง run change detection เมื่อใด

### 39. การ optimize bundle size?

1. Lazy loading
2. Tree shaking
3. Code splitting
4. Remove unused imports
5. Use smaller libraries
6. Compression (gzip/brotli)
7. Angular budgets

### 40. Angular Universal SSR vs CSR vs SSG?

- **CSR**: render ใน browser
- **SSR**: render บน server ทุก request
- **SSG**: render ตอน build time (static)

### 41. Micro-Frontend Architecture?

แบ่ง app เป็นหลาย independent apps ที่ run รวมกัน
- Angular Elements (Web Components)
- Module Federation (Webpack 5)

### 42. Hydration คืออะไร (Angular 16+)?

กระบวนการ attach Angular event listeners เข้ากับ HTML ที่ render จาก SSR โดยไม่ต้อง re-render ใหม่

### 43. Feature Flags Pattern?

```typescript
@Injectable({ providedIn: 'root' })
export class FeatureService {
  isEnabled(feature: string): boolean {
    return this.config.features[feature] ?? false;
  }
}
```

### 44. CQRS Pattern ใน Frontend?

แยก Command (write) และ Query (read)

```typescript
class CommandBus {
  execute<T>(command: Command): Promise<T> {
    const handler = this.handlers.get(command.constructor.name);
    return handler.execute(command);
  }
}
```

### 45. Repository Pattern?

Abstract data access layer

```typescript
abstract class ProductRepository {
  abstract getAll(): Observable<Product[]>;
  abstract getById(id: number): Observable<Product>;
}

@Injectable({ providedIn: 'root' })
class HttpProductRepository extends ProductRepository {
  getAll() { return this.http.get<Product[]>('/api/products'); }
}
```

### 46. Strategy Pattern ใน Angular?

```typescript
interface SortStrategy {
  sort<T>(items: T[]): T[];
}

@Injectable({ providedIn: 'root' })
export class SortService {
  private strategy: SortStrategy = new DefaultSortStrategy();
  setStrategy(strategy: SortStrategy) { this.strategy = strategy; }
  sort<T>(items: T[]): T[] { return this.strategy.sort(items); }
}
```

### 47. Decorator Pattern?

TypeScript decorators เป็นตัวอย่างที่ดีของ Decorator pattern

```typescript
function Log(target: any, key: string, descriptor: PropertyDescriptor) {
  const original = descriptor.value;
  descriptor.value = function(...args: any[]) {
    console.log(`${key} called with`, args);
    return original.apply(this, args);
  };
  return descriptor;
}
```

### 48. Performance Profiling?

1. Chrome DevTools Performance tab
2. Angular DevTools (profiler)
3. Lighthouse audit
4. `ng build --stats-json` + webpack-bundle-analyzer

### 49. Security Best Practices?

1. ใช้ Angular DomSanitizer สำหรับ dynamic HTML
2. HTTPS เสมอ
3. CSRF tokens
4. ไม่เก็บ sensitive data ใน localStorage
5. Content Security Policy headers
6. Input validation

### 50. Angular vs React vs Vue?

| เรื่อง | Angular | React | Vue |
|-------|---------|-------|-----|
| Type | Full framework | UI library | Progressive framework |
| Language | TypeScript | JS/TS | JS/TS |
| Learning curve | สูง | กลาง | ต่ำ |
| Performance | ดี | ดีมาก | ดีมาก |
| เหมาะกับ | Enterprise | Large apps | Small-Medium |

---

## Tips สำหรับ Interview

1. **อธิบาย Why** ไม่ใช่แค่ What
2. **ยกตัวอย่าง** จาก project จริงที่ทำ
3. **พูดถึง Trade-offs** ของแต่ละ approach
4. **ถาม** ถ้าไม่แน่ใจ requirements
5. **Code live** ได้อย่างมั่นใจด้วยการฝึก
