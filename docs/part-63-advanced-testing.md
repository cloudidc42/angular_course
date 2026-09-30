# Part 63: Advanced Testing ใน Angular

## บทนำ

การทดสอบที่ดีช่วยให้แอปพลิเคชันมีคุณภาพสูงและลดข้อบกพร่อง บทนี้จะครอบคลุม Component Harnesses, Integration Testing และ Angular Testing Library

## 1. Component Harnesses

Component Harness ช่วยให้ test ไม่ขึ้นอยู่กับ DOM structure โดยตรง

```typescript
// button/button.component.harness.ts
import { ComponentHarness } from '@angular/cdk/testing';

export class ButtonHarness extends ComponentHarness {
  static hostSelector = 'app-button, my-company-button';

  async click(): Promise<void> {
    const button = await this.locatorFor('button')();
    await button.click();
  }

  async getText(): Promise<string> {
    const button = await this.locatorFor('button')();
    return (await button.text()).trim();
  }

  async isDisabled(): Promise<boolean> {
    const button = await this.locatorFor('button')();
    return button.getProperty<boolean>('disabled');
  }

  async isLoading(): Promise<boolean> {
    const spinner = await this.locatorForOptional('.spinner')();
    return spinner !== null;
  }

  async getVariant(): Promise<string> {
    const button = await this.locatorFor('button')();
    const className = await button.getAttribute('class') || '';
    const match = className.match(/btn-(\w+)/);
    return match ? match[1] : 'primary';
  }
}
```

### ใช้ Harness ใน Tests

```typescript
// button.component.spec.ts
import { TestbedHarnessEnvironment } from '@angular/cdk/testing/testbed';
import { TestBed, ComponentFixture } from '@angular/core/testing';
import { ButtonHarness } from './button.component.harness';
import { ButtonComponent } from './button.component';

describe('ButtonComponent (Harness)', () => {
  let fixture: ComponentFixture<ButtonComponent>;
  let harness: ButtonHarness;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [ButtonComponent]
    }).compileComponents();

    fixture = TestBed.createComponent(ButtonComponent);
    const loader = TestbedHarnessEnvironment.loader(fixture);
    harness = await loader.getHarness(ButtonHarness);
  });

  it('should display button text', async () => {
    fixture.componentInstance.variant = 'primary';
    fixture.detectChanges();
    
    // ไม่ต้องรู้ DOM structure
    const text = await harness.getText();
    expect(text).toBeDefined();
  });

  it('should be disabled when disabled input is true', async () => {
    fixture.componentInstance.disabled = true;
    fixture.detectChanges();
    
    const isDisabled = await harness.isDisabled();
    expect(isDisabled).toBe(true);
  });

  it('should show loading spinner when loading', async () => {
    fixture.componentInstance.loading = true;
    fixture.detectChanges();
    
    const isLoading = await harness.isLoading();
    expect(isLoading).toBe(true);
  });
});
```

## 2. Custom Harness สำหรับ Form

```typescript
// login-form.harness.ts
import { ComponentHarness, HarnessPredicate } from '@angular/cdk/testing';

export class LoginFormHarness extends ComponentHarness {
  static hostSelector = 'app-login-form';

  async setEmail(email: string): Promise<void> {
    const input = await this.locatorFor('input[formControlName="email"]')();
    await input.clear();
    await input.sendKeys(email);
  }

  async setPassword(password: string): Promise<void> {
    const input = await this.locatorFor('input[formControlName="password"]')();
    await input.clear();
    await input.sendKeys(password);
  }

  async submit(): Promise<void> {
    const button = await this.locatorFor('button[type="submit"]')();
    await button.click();
  }

  async getEmailError(): Promise<string | null> {
    const error = await this.locatorForOptional('.email-error')();
    if (!error) return null;
    return error.text();
  }

  async getPasswordError(): Promise<string | null> {
    const error = await this.locatorForOptional('.password-error')();
    if (!error) return null;
    return error.text();
  }

  async isSubmitDisabled(): Promise<boolean> {
    const button = await this.locatorFor('button[type="submit"]')();
    return button.getProperty<boolean>('disabled');
  }

  async getServerError(): Promise<string | null> {
    const error = await this.locatorForOptional('.server-error')();
    if (!error) return null;
    return error.text();
  }

  async fillAndSubmit(email: string, password: string): Promise<void> {
    await this.setEmail(email);
    await this.setPassword(password);
    await this.submit();
  }
}
```

## 3. Integration Testing

```typescript
// products-list.integration.spec.ts
import { TestBed, fakeAsync, tick } from '@angular/core/testing';
import { HttpClientTestingModule, HttpTestingController } from '@angular/common/http/testing';
import { RouterTestingModule } from '@angular/router/testing';
import { ProductsListComponent } from './products-list.component';
import { ProductsService } from '../services/products.service';

describe('ProductsListComponent (Integration)', () => {
  let httpMock: HttpTestingController;
  let fixture: any;

  const mockProducts = [
    { id: '1', name: 'สินค้า A', price: 100, stock: 10, images: ['img1.jpg'] },
    { id: '2', name: 'สินค้า B', price: 200, stock: 0, images: ['img2.jpg'] }
  ];

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [
        ProductsListComponent,
        HttpClientTestingModule,
        RouterTestingModule
      ]
    }).compileComponents();

    httpMock = TestBed.inject(HttpTestingController);
    fixture = TestBed.createComponent(ProductsListComponent);
  });

  afterEach(() => {
    httpMock.verify();
  });

  it('should load products on init', fakeAsync(() => {
    fixture.detectChanges();

    const req = httpMock.expectOne('/api/products?page=1&limit=20');
    expect(req.request.method).toBe('GET');
    
    req.flush({
      items: mockProducts,
      total: 2
    });
    
    tick();
    fixture.detectChanges();

    const compiled = fixture.nativeElement;
    const cards = compiled.querySelectorAll('.product-card');
    expect(cards.length).toBe(2);
  }));

  it('should search products when user types', fakeAsync(() => {
    fixture.detectChanges();

    // Flush initial request
    httpMock.expectOne('/api/products?page=1&limit=20').flush({ items: [], total: 0 });

    // User types in search box
    const searchInput = fixture.nativeElement.querySelector('.search-input');
    searchInput.value = 'สินค้า A';
    searchInput.dispatchEvent(new Event('input'));

    tick(300); // debounce time
    fixture.detectChanges();

    const searchReq = httpMock.expectOne(req => req.url.includes('/api/products') && req.params.has('search'));
    expect(searchReq.request.params.get('search')).toBe('สินค้า A');
    
    searchReq.flush({ items: [mockProducts[0]], total: 1 });
    tick();
    fixture.detectChanges();

    const cards = fixture.nativeElement.querySelectorAll('.product-card');
    expect(cards.length).toBe(1);
  }));

  it('should show out-of-stock badge for products with 0 stock', fakeAsync(() => {
    fixture.detectChanges();
    
    httpMock.expectOne('/api/products?page=1&limit=20').flush({
      items: mockProducts,
      total: 2
    });
    
    tick();
    fixture.detectChanges();

    const outOfStockBadges = fixture.nativeElement.querySelectorAll('.out-of-stock');
    expect(outOfStockBadges.length).toBe(1);
  }));

  it('should handle API error gracefully', fakeAsync(() => {
    fixture.detectChanges();
    
    httpMock.expectOne('/api/products?page=1&limit=20').error(
      new ErrorEvent('Network error', { message: 'Connection failed' })
    );
    
    tick();
    fixture.detectChanges();

    const errorMessage = fixture.nativeElement.querySelector('.error-message');
    expect(errorMessage).toBeTruthy();
  }));
});
```

## 4. Angular Testing Library

```bash
npm install --save-dev @testing-library/angular @testing-library/jest-dom @testing-library/user-event
```

```typescript
// login.component.spec.ts (Testing Library style)
import { render, screen, fireEvent, waitFor } from '@testing-library/angular';
import userEvent from '@testing-library/user-event';
import { LoginComponent } from './login.component';
import { AuthService } from '../services/auth.service';
import { of, throwError } from 'rxjs';

describe('LoginComponent (Testing Library)', () => {
  const mockAuthService = {
    login: jest.fn()
  };

  beforeEach(() => {
    mockAuthService.login.mockClear();
  });

  async function renderLogin() {
    return render(LoginComponent, {
      providers: [
        { provide: AuthService, useValue: mockAuthService }
      ]
    });
  }

  it('should render login form', async () => {
    await renderLogin();
    
    expect(screen.getByLabelText('อีเมล')).toBeInTheDocument();
    expect(screen.getByLabelText('รหัสผ่าน')).toBeInTheDocument();
    expect(screen.getByRole('button', { name: 'เข้าสู่ระบบ' })).toBeInTheDocument();
  });

  it('should show validation errors on empty submit', async () => {
    await renderLogin();
    
    const submitButton = screen.getByRole('button', { name: 'เข้าสู่ระบบ' });
    await userEvent.click(submitButton);
    
    expect(await screen.findByText('กรุณากรอกอีเมล')).toBeInTheDocument();
    expect(await screen.findByText('กรุณากรอกรหัสผ่าน')).toBeInTheDocument();
  });

  it('should call auth service on valid form submit', async () => {
    mockAuthService.login.mockReturnValue(of({ token: 'abc123' }));
    await renderLogin();
    
    await userEvent.type(screen.getByLabelText('อีเมล'), 'test@example.com');
    await userEvent.type(screen.getByLabelText('รหัสผ่าน'), 'Password123!');
    await userEvent.click(screen.getByRole('button', { name: 'เข้าสู่ระบบ' }));
    
    await waitFor(() => {
      expect(mockAuthService.login).toHaveBeenCalledWith({
        email: 'test@example.com',
        password: 'Password123!'
      });
    });
  });

  it('should show error message on login failure', async () => {
    mockAuthService.login.mockReturnValue(
      throwError(() => ({ message: 'อีเมลหรือรหัสผ่านไม่ถูกต้อง' }))
    );
    await renderLogin();
    
    await userEvent.type(screen.getByLabelText('อีเมล'), 'wrong@example.com');
    await userEvent.type(screen.getByLabelText('รหัสผ่าน'), 'wrongpass');
    await userEvent.click(screen.getByRole('button', { name: 'เข้าสู่ระบบ' }));
    
    expect(await screen.findByText('อีเมลหรือรหัสผ่านไม่ถูกต้อง')).toBeInTheDocument();
  });

  it('should disable submit button while loading', async () => {
    const { promise, resolve } = createDeferred();
    mockAuthService.login.mockReturnValue(from(promise));
    await renderLogin();
    
    await userEvent.type(screen.getByLabelText('อีเมล'), 'test@example.com');
    await userEvent.type(screen.getByLabelText('รหัสผ่าน'), 'Password123!');
    await userEvent.click(screen.getByRole('button', { name: 'เข้าสู่ระบบ' }));
    
    expect(screen.getByRole('button', { name: 'กำลังเข้าสู่ระบบ...' })).toBeDisabled();
    
    resolve({ token: 'abc' });
  });
});

function createDeferred() {
  let resolve: (value: any) => void;
  const promise = new Promise(r => { resolve = r; });
  return { promise, resolve: resolve! };
}
```

## 5. Service Testing

```typescript
// products.service.spec.ts
import { TestBed } from '@angular/core/testing';
import { HttpClientTestingModule, HttpTestingController } from '@angular/common/http/testing';
import { ProductsService } from './products.service';

describe('ProductsService', () => {
  let service: ProductsService;
  let httpMock: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({
      imports: [HttpClientTestingModule],
      providers: [ProductsService]
    });
    
    service = TestBed.inject(ProductsService);
    httpMock = TestBed.inject(HttpTestingController);
  });

  afterEach(() => httpMock.verify());

  describe('getProducts', () => {
    it('should return paginated products', () => {
      const mockResponse = {
        items: [{ id: '1', name: 'Test', price: 100 }],
        total: 1
      };

      service.getProducts({ page: 1, limit: 10 }).subscribe(result => {
        expect(result.items.length).toBe(1);
        expect(result.total).toBe(1);
      });

      const req = httpMock.expectOne(r => r.url === '/api/products');
      expect(req.request.params.get('page')).toBe('1');
      expect(req.request.params.get('limit')).toBe('10');
      req.flush(mockResponse);
    });

    it('should handle search filter', () => {
      service.getProducts({ search: 'test', page: 1, limit: 10 }).subscribe();
      
      const req = httpMock.expectOne(r => r.params.has('search'));
      expect(req.request.params.get('search')).toBe('test');
      req.flush({ items: [], total: 0 });
    });
  });

  describe('createProduct', () => {
    it('should POST to /api/products', () => {
      const newProduct = { name: 'New Product', price: 150, stock: 20 };
      const createdProduct = { id: '123', ...newProduct };

      service.createProduct(newProduct as any).subscribe(result => {
        expect(result.id).toBe('123');
        expect(result.name).toBe('New Product');
      });

      const req = httpMock.expectOne('/api/products');
      expect(req.request.method).toBe('POST');
      expect(req.request.body).toEqual(newProduct);
      req.flush(createdProduct);
    });
  });
});
```

## 6. Testing NgRx Store

```typescript
// products.effects.spec.ts
import { TestBed } from '@angular/core/testing';
import { provideMockActions } from '@ngrx/effects/testing';
import { provideMockStore } from '@ngrx/store/testing';
import { Observable, of } from 'rxjs';
import { ProductsEffects } from './products.effects';
import { ProductsService } from '../services/products.service';
import { loadProducts, loadProductsSuccess, loadProductsFailure } from './products.actions';

describe('ProductsEffects', () => {
  let effects: ProductsEffects;
  let actions$: Observable<any>;
  let productsService: jest.Mocked<ProductsService>;

  beforeEach(() => {
    const mockService = {
      getProducts: jest.fn()
    };

    TestBed.configureTestingModule({
      providers: [
        ProductsEffects,
        provideMockActions(() => actions$),
        provideMockStore(),
        { provide: ProductsService, useValue: mockService }
      ]
    });

    effects = TestBed.inject(ProductsEffects);
    productsService = TestBed.inject(ProductsService) as jest.Mocked<ProductsService>;
  });

  it('should dispatch loadProductsSuccess on success', (done) => {
    const mockProducts = { items: [{ id: '1', name: 'Product' }], total: 1 };
    productsService.getProducts.mockReturnValue(of(mockProducts as any));

    actions$ = of(loadProducts({ page: 1, limit: 20 }));

    effects.loadProducts$.subscribe(action => {
      expect(action).toEqual(loadProductsSuccess({ products: mockProducts.items as any, total: mockProducts.total }));
      done();
    });
  });
});
```

## สรุป Testing Strategies

| เทคนิค | เหมาะกับ | เครื่องมือ |
|--------|---------|---------|
| Unit Tests | Services, Utils | Jest |
| Component Tests (Harness) | Complex UI components | CDK Testing |
| Integration Tests | Feature flows | TestBed + HttpMock |
| Testing Library | User-centric tests | @testing-library/angular |
| E2E Tests | Critical user flows | Cypress |

การทดสอบที่ดีควรครอบคลุม happy path, edge cases, และ error scenarios
