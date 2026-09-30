# Part 64: E2E Testing ด้วย Cypress ใน Angular

## บทนำ

Cypress เป็นเครื่องมือ E2E testing ที่ทรงพลังและใช้งานง่าย มีความสามารถในการ test ทั้ง E2E และ Component testing

## 1. การติดตั้ง Cypress

```bash
npm install --save-dev cypress @cypress/angular

# เพิ่ม cypress ใน Angular project
ng add @cypress/schematic
```

```json
// cypress.config.ts
import { defineConfig } from 'cypress';

export default defineConfig({
  e2e: {
    baseUrl: 'http://localhost:4200',
    specPattern: 'cypress/e2e/**/*.cy.ts',
    supportFile: 'cypress/support/e2e.ts',
    viewportWidth: 1280,
    viewportHeight: 720,
    video: true,
    screenshotOnRunFailure: true,
    defaultCommandTimeout: 8000,
    requestTimeout: 10000
  },
  component: {
    devServer: {
      framework: 'angular',
      bundler: 'webpack'
    },
    specPattern: '**/*.cy.ts'
  }
});
```

## 2. Custom Commands

```typescript
// cypress/support/commands.ts
/// <reference types="cypress" />

declare global {
  namespace Cypress {
    interface Chainable {
      login(email: string, password: string): Chainable<void>;
      loginAsAdmin(): Chainable<void>;
      logout(): Chainable<void>;
      interceptApi(method: string, path: string, fixture: string): Chainable<void>;
      waitForPage(): Chainable<void>;
      checkAccessibility(): Chainable<void>;
    }
  }
}

Cypress.Commands.add('login', (email: string, password: string) => {
  cy.session([email, password], () => {
    cy.visit('/login');
    cy.get('[data-cy=email-input]').type(email);
    cy.get('[data-cy=password-input]').type(password);
    cy.get('[data-cy=submit-btn]').click();
    cy.url().should('not.include', '/login');
  });
});

Cypress.Commands.add('loginAsAdmin', () => {
  // ใช้ API เพื่อ login แทน UI (เร็วกว่า)
  cy.request({
    method: 'POST',
    url: '/api/auth/login',
    body: {
      email: Cypress.env('ADMIN_EMAIL'),
      password: Cypress.env('ADMIN_PASSWORD')
    }
  }).then(response => {
    window.localStorage.setItem('auth_token', JSON.stringify(response.body));
  });
});

Cypress.Commands.add('logout', () => {
  cy.clearLocalStorage('auth_token');
  cy.visit('/login');
});

Cypress.Commands.add('interceptApi', (method: string, path: string, fixture: string) => {
  cy.intercept(method, `**/api/${path}`, { fixture }).as(path.replace('/', '-'));
});

Cypress.Commands.add('waitForPage', () => {
  cy.get('.loading-overlay', { timeout: 10000 }).should('not.exist');
  cy.get('[data-cy=page-loaded]').should('exist');
});
```

## 3. Page Object Pattern

```typescript
// cypress/support/pages/login.page.ts
export class LoginPage {
  get emailInput() { return cy.get('[data-cy=email-input]'); }
  get passwordInput() { return cy.get('[data-cy=password-input]'); }
  get submitButton() { return cy.get('[data-cy=submit-btn]'); }
  get errorMessage() { return cy.get('[data-cy=error-message]'); }
  get forgotPasswordLink() { return cy.get('[data-cy=forgot-password]'); }
  get rememberMeCheckbox() { return cy.get('[data-cy=remember-me]'); }

  visit() {
    cy.visit('/login');
    return this;
  }

  fillEmail(email: string) {
    this.emailInput.clear().type(email);
    return this;
  }

  fillPassword(password: string) {
    this.passwordInput.clear().type(password);
    return this;
  }

  submit() {
    this.submitButton.click();
    return this;
  }

  login(email: string, password: string) {
    this.fillEmail(email);
    this.fillPassword(password);
    this.submit();
    return this;
  }

  assertErrorMessage(message: string) {
    this.errorMessage.should('be.visible').and('contain', message);
    return this;
  }

  assertRedirectedTo(url: string) {
    cy.url().should('include', url);
    return this;
  }
}

// cypress/support/pages/products.page.ts
export class ProductsPage {
  get searchInput() { return cy.get('[data-cy=product-search]'); }
  get productCards() { return cy.get('[data-cy=product-card]'); }
  get addProductButton() { return cy.get('[data-cy=add-product-btn]'); }
  get loadingSpinner() { return cy.get('[data-cy=loading]'); }
  get pagination() { return cy.get('[data-cy=pagination]'); }

  visit() {
    cy.visit('/products');
    return this;
  }

  search(query: string) {
    this.searchInput.clear().type(query);
    cy.wait('@products-search');
    return this;
  }

  getProductCard(productName: string) {
    return this.productCards.contains(productName).parents('[data-cy=product-card]');
  }

  deleteProduct(productName: string) {
    this.getProductCard(productName)
      .find('[data-cy=delete-btn]')
      .click();
    cy.get('[data-cy=confirm-dialog]').find('[data-cy=confirm-btn]').click();
    return this;
  }

  assertProductCount(count: number) {
    this.productCards.should('have.length', count);
    return this;
  }

  assertProductVisible(name: string) {
    this.productCards.contains(name).should('be.visible');
    return this;
  }

  assertProductNotVisible(name: string) {
    this.productCards.contains(name).should('not.exist');
    return this;
  }
}
```

## 4. E2E Tests

```typescript
// cypress/e2e/auth/login.cy.ts
import { LoginPage } from '../../support/pages/login.page';

describe('Authentication', () => {
  const loginPage = new LoginPage();

  beforeEach(() => {
    cy.visit('/');
  });

  context('Login Flow', () => {
    it('should redirect unauthenticated users to login', () => {
      cy.url().should('include', '/login');
    });

    it('should show validation errors on empty submit', () => {
      loginPage.visit().submit();
      loginPage.assertErrorMessage('กรุณากรอกอีเมล');
    });

    it('should show error for invalid email format', () => {
      loginPage
        .visit()
        .fillEmail('invalid-email')
        .submit();
      loginPage.assertErrorMessage('รูปแบบอีเมลไม่ถูกต้อง');
    });

    it('should show error for wrong credentials', () => {
      cy.intercept('POST', '/api/auth/login', {
        statusCode: 401,
        body: { message: 'อีเมลหรือรหัสผ่านไม่ถูกต้อง' }
      }).as('loginRequest');

      loginPage
        .visit()
        .login('wrong@example.com', 'wrongpassword');

      cy.wait('@loginRequest');
      loginPage.assertErrorMessage('อีเมลหรือรหัสผ่านไม่ถูกต้อง');
    });

    it('should login successfully and redirect to dashboard', () => {
      cy.intercept('POST', '/api/auth/login', {
        statusCode: 200,
        body: { token: 'fake-jwt-token', user: { id: '1', name: 'Test User' } }
      }).as('loginRequest');

      loginPage
        .visit()
        .login('admin@example.com', 'password123');

      cy.wait('@loginRequest');
      loginPage.assertRedirectedTo('/dashboard');
    });

    it('should logout successfully', () => {
      cy.loginAsAdmin();
      cy.visit('/dashboard');
      cy.get('[data-cy=logout-btn]').click();
      cy.url().should('include', '/login');
    });
  });
});
```

```typescript
// cypress/e2e/products/products-crud.cy.ts
import { ProductsPage } from '../../support/pages/products.page';

describe('Products CRUD', () => {
  const productsPage = new ProductsPage();

  beforeEach(() => {
    cy.loginAsAdmin();
    cy.intercept('GET', '/api/products*', { fixture: 'products.json' }).as('getProducts');
    productsPage.visit();
    cy.wait('@getProducts');
  });

  it('should display product list', () => {
    productsPage.assertProductCount(10);
    productsPage.assertProductVisible('iPhone 15');
  });

  it('should search products', () => {
    cy.intercept('GET', '/api/products*search=iPhone*', {
      fixture: 'products-search.json'
    }).as('products-search');

    productsPage.search('iPhone');
    productsPage.assertProductVisible('iPhone 15');
    productsPage.assertProductNotVisible('Samsung Galaxy');
  });

  it('should create a new product', () => {
    cy.intercept('POST', '/api/products', {
      statusCode: 201,
      body: { id: '999', name: 'New Product', price: 500 }
    }).as('createProduct');

    productsPage.addProductButton.click();
    cy.url().should('include', '/products/new');

    cy.get('[data-cy=product-name]').type('New Product');
    cy.get('[data-cy=product-price]').type('500');
    cy.get('[data-cy=product-stock]').type('100');
    cy.get('[data-cy=save-btn]').click();

    cy.wait('@createProduct');
    cy.url().should('match', /\/products\/\d+/);
    cy.get('[data-cy=success-message]').should('be.visible');
  });

  it('should delete a product with confirmation', () => {
    cy.intercept('DELETE', '/api/products/1', { statusCode: 204 }).as('deleteProduct');

    productsPage.deleteProduct('iPhone 15');

    cy.wait('@deleteProduct');
    productsPage.assertProductNotVisible('iPhone 15');
    cy.get('[data-cy=success-toast]').should('contain', 'ลบสินค้าเรียบร้อยแล้ว');
  });

  it('should paginate products', () => {
    cy.intercept('GET', '/api/products*page=2*', {
      fixture: 'products-page2.json'
    }).as('page2');

    cy.get('[data-cy=next-page-btn]').click();
    cy.wait('@page2');
    
    cy.get('[data-cy=current-page]').should('contain', '2');
  });
});
```

## 5. API Mocking

```typescript
// cypress/fixtures/products.json
// cypress/e2e/api-mocking.cy.ts

describe('API Mocking Strategies', () => {
  it('should use fixture file', () => {
    cy.intercept('GET', '/api/users', { fixture: 'users.json' }).as('getUsers');
    cy.visit('/users');
    cy.wait('@getUsers');
  });

  it('should mock with dynamic response', () => {
    cy.intercept('POST', '/api/orders', (req) => {
      // ตรวจสอบ request body
      expect(req.body).to.have.property('userId');
      
      req.reply({
        statusCode: 201,
        body: {
          id: 'new-order-id',
          ...req.body,
          createdAt: new Date().toISOString()
        }
      });
    }).as('createOrder');
  });

  it('should simulate network error', () => {
    cy.intercept('GET', '/api/products', {
      forceNetworkError: true
    }).as('networkError');
    
    cy.visit('/products');
    cy.wait('@networkError');
    cy.get('[data-cy=error-message]').should('be.visible');
  });

  it('should simulate slow response', () => {
    cy.intercept('GET', '/api/products', (req) => {
      req.reply({
        delay: 3000, // 3 วินาที
        fixture: 'products.json'
      });
    });
    
    cy.visit('/products');
    cy.get('[data-cy=loading-spinner]').should('be.visible');
    cy.get('[data-cy=product-card]', { timeout: 10000 }).should('be.visible');
  });
});
```

## 6. Component Testing

```typescript
// src/app/button/button.component.cy.ts
import { ButtonComponent } from './button.component';

describe('ButtonComponent (Cypress Component Testing)', () => {
  it('renders primary button', () => {
    cy.mount(ButtonComponent, {
      componentProperties: {
        variant: 'primary',
        disabled: false
      }
    });

    cy.get('button').should('have.class', 'btn-primary');
    cy.get('button').should('not.be.disabled');
  });

  it('emits click event', () => {
    const onClickSpy = cy.spy().as('onClickSpy');
    
    cy.mount(ButtonComponent, {
      componentProperties: {
        clicked: { emit: onClickSpy } as any
      }
    });

    cy.get('button').click();
    cy.get('@onClickSpy').should('have.been.called');
  });

  it('shows loading state', () => {
    cy.mount(ButtonComponent, {
      componentProperties: { loading: true }
    });
    
    cy.get('.spinner').should('be.visible');
    cy.get('button').should('be.disabled');
  });

  it('is accessible', () => {
    cy.mount(ButtonComponent, {
      componentProperties: {
        disabled: false
      },
      template: '<my-company-button>คลิกที่นี่</my-company-button>'
    });
    
    cy.get('button').should('have.attr', 'type', 'button');
    // ตรวจสอบ accessibility
    cy.checkA11y();
  });
});
```

## 7. CI/CD Integration

```yaml
# .github/workflows/cypress.yml
name: Cypress E2E Tests

on: [push, pull_request]

jobs:
  cypress:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Build Angular app
        run: npm run build
      
      - name: Run Cypress E2E tests
        uses: cypress-io/github-action@v6
        with:
          start: npm run start:ci
          wait-on: 'http://localhost:4200'
          wait-on-timeout: 60
          browser: chrome
          record: true
        env:
          CYPRESS_RECORD_KEY: ${{ secrets.CYPRESS_RECORD_KEY }}
          ADMIN_EMAIL: ${{ secrets.ADMIN_EMAIL }}
          ADMIN_PASSWORD: ${{ secrets.ADMIN_PASSWORD }}
      
      - name: Upload screenshots on failure
        uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: cypress-screenshots
          path: cypress/screenshots
      
      - name: Upload videos
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: cypress-videos
          path: cypress/videos
```

## สรุป Cypress Best Practices

| หัวข้อ | แนวทาง |
|-------|-------|
| Selectors | ใช้ `data-cy` attribute |
| Authentication | ใช้ cy.session() หรือ API login |
| API Mocking | cy.intercept() สำหรับ isolation |
| Page Objects | แยก locators ออกจาก tests |
| Test Data | ใช้ Fixtures และ Factory functions |
| CI/CD | ใช้ Cypress Dashboard สำหรับ recording |

Cypress ช่วยให้ E2E tests รวดเร็ว เชื่อถือได้ และ debug ง่าย
