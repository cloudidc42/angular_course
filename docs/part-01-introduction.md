# Part 01 — บทนำ Angular และการติดตั้ง

## เนื้อหาในบทนี้
1. Angular คืออะไร?
2. ประวัติและวิวัฒนาการ
3. Angular vs React vs Vue
4. สถาปัตยกรรมโดยรวม
5. การติดตั้ง Environment
6. สร้างโปรเจ็กต์แรก
7. โครงสร้างโปรเจ็กต์
8. Angular CLI คำสั่งสำคัญ
9. Dev Tools ที่จำเป็น
10. Workshop: Hello World App

---

## 1. Angular คืออะไร?

**Angular** คือ Framework สำหรับพัฒนา Web Application แบบ Single-Page Application (SPA) ที่พัฒนาโดย Google เปิดตัวครั้งแรกในปี 2016 (Angular 2) และมีการพัฒนาต่อเนื่องมาจนถึงปัจจุบัน

### คุณสมบัติหลักของ Angular

```
Angular Framework
├── Component-Based Architecture
│   ├── Template (HTML)
│   ├── Class (TypeScript)
│   └── Styles (CSS/SCSS)
├── Two-Way Data Binding
├── Dependency Injection System
├── Reactive Programming (RxJS)
├── TypeScript First
├── Built-in Testing Support
└── Angular CLI (Command Line Interface)
```

### ทำไมต้องใช้ Angular?

| คุณสมบัติ | ประโยชน์ |
|-----------|---------|
| **TypeScript** | ตรวจจับข้อผิดพลาดตั้งแต่ compile time |
| **Opinionated** | มีแนวทางชัดเจน ทีมทำงานได้ง่าย |
| **Full Framework** | มีทุกอย่างในตัว ไม่ต้องเลือก library เพิ่ม |
| **Enterprise Ready** | เหมาะกับโปรเจ็กต์ขนาดใหญ่ |
| **Google Support** | มีทีม Google ดูแล stable มาก |
| **Large Community** | มีชุมชนขนาดใหญ่ หาความช่วยเหลือง่าย |

---

## 2. ประวัติและวิวัฒนาการ

```
Timeline ของ Angular
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
2010  AngularJS (Angular 1.x) — JavaScript, MVC
      │   ปัญหา: Performance, ไม่ scale ได้ดี
      │
2016  Angular 2 — TypeScript, Component-based
      │   เขียนใหม่ทั้งหมด ไม่ backward compatible
      │
2017  Angular 4 (ข้าม v3)
2017  Angular 5 — Build Optimizer
2018  Angular 6 — Angular Elements, CDK
2018  Angular 7 — Virtual Scrolling, Drag&Drop
2019  Angular 8 — Ivy (preview), Differential Loading
2020  Angular 9 — Ivy (default), Smaller bundles
2020  Angular 10 — TypeScript 3.9
2020  Angular 11 — Hot Module Replacement
2021  Angular 12 — Webpack 5, Strict Mode default
2021  Angular 13 — View Engine removed
2022  Angular 14 — Standalone Components (preview)
2022  Angular 15 — Standalone API stable
2023  Angular 16 — Signals (preview), SSR improvements
2023  Angular 17 — New syntax (@if, @for), Signals stable
2024  Angular 18 — Zoneless (experimental), improved SSR
2025  Angular 19+ — Full Signals, Zoneless stable
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## 3. Angular vs React vs Vue

| เกณฑ์ | Angular | React | Vue |
|-------|---------|-------|-----|
| **ประเภท** | Full Framework | Library | Progressive Framework |
| **ภาษา** | TypeScript | JSX/JS | JS/TS |
| **Learning Curve** | สูง | กลาง | ต่ำ |
| **ขนาด Bundle** | ใหญ่กว่า | กลาง | เล็กที่สุด |
| **Performance** | ดี | ดีมาก | ดีมาก |
| **Enterprise** | ดีมาก | ดี | ดี |
| **Job Market** | สูง | สูงมาก | กลาง |
| **Testing** | Built-in | ต้องเพิ่มเอง | ต้องเพิ่มเอง |
| **State Management** | NgRx/Signals | Redux/Zustand | Vuex/Pinia |
| **SSR** | Angular Universal | Next.js | Nuxt.js |

### เลือก Angular เมื่อ:
- โปรเจ็กต์ขนาดใหญ่ระดับ Enterprise
- ทีมต้องการโครงสร้างที่ชัดเจน
- ต้องการ TypeScript จากต้น
- มีทีม Backend ที่ใช้ Java/C# (คล้ายกัน)

---

## 4. สถาปัตยกรรมโดยรวม

```
┌─────────────────────────────────────────────────┐
│                  Angular Application              │
│                                                   │
│  ┌──────────────────────────────────────────┐    │
│  │              NgModule / Standalone        │    │
│  │  ┌──────────┐  ┌──────────┐  ┌────────┐ │    │
│  │  │Component │  │Component │  │Service │ │    │
│  │  │ Template │  │ Template │  │        │ │    │
│  │  │  Class   │  │  Class   │  │  DI    │ │    │
│  │  └──────────┘  └──────────┘  └────────┘ │    │
│  └──────────────────────────────────────────┘    │
│                                                   │
│  ┌───────────┐  ┌──────────┐  ┌─────────────┐   │
│  │  Router   │  │  Forms   │  │  HttpClient │   │
│  └───────────┘  └──────────┘  └─────────────┘   │
│                                                   │
│  ┌──────────────────────────────────────────┐    │
│  │              RxJS / Signals               │    │
│  └──────────────────────────────────────────┘    │
└─────────────────────────────────────────────────┘
```

### องค์ประกอบหลักของ Angular

**1. Modules (NgModule)**
```typescript
@NgModule({
  declarations: [AppComponent],
  imports: [BrowserModule, RouterModule],
  providers: [UserService],
  bootstrap: [AppComponent]
})
export class AppModule {}
```

**2. Components**
```typescript
@Component({
  selector: 'app-root',
  template: `<h1>Hello {{title}}</h1>`,
  styles: [`h1 { color: blue; }`]
})
export class AppComponent {
  title = 'My App';
}
```

**3. Services**
```typescript
@Injectable({
  providedIn: 'root'
})
export class DataService {
  getData() { return []; }
}
```

**4. Directives**
```html
<div *ngIf="show">แสดงเมื่อ show = true</div>
<li *ngFor="let item of items">{{ item }}</li>
```

**5. Pipes**
```html
{{ date | date:'dd/MM/yyyy' }}
{{ price | currency:'THB' }}
```

---

## 5. การติดตั้ง Environment

### ขั้นตอนที่ 1: ติดตั้ง Node.js

**Windows/Mac:**
ดาวน์โหลดจาก https://nodejs.org (เลือก LTS version)

**Linux (Ubuntu/Debian):**
```bash
# ติดตั้ง nvm (Node Version Manager)
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash

# รีโหลด shell
source ~/.bashrc

# ติดตั้ง Node.js LTS
nvm install --lts
nvm use --lts

# ตรวจสอบ version
node --version  # ควรได้ v18.x หรือ v20.x
npm --version   # ควรได้ v9.x หรือสูงกว่า
```

**macOS ด้วย Homebrew:**
```bash
brew install node
```

### ขั้นตอนที่ 2: ติดตั้ง Angular CLI

```bash
# ติดตั้ง Angular CLI แบบ Global
npm install -g @angular/cli

# ตรวจสอบ version
ng version

# Output ที่ควรเห็น:
#      _                      _                 ____ _     ___
#     / \   _ __   __ _ _   _| | __ _ _ __     / ___| |   |_ _|
#    / △ \ | '_ \ / _` | | | | |/ _` | '__|   | |   | |    | |
#   / ___ \| | | | (_| | |_| | | (_| | |      | |___| |___ | |
#  /_/   \_\_| |_|\__, |\__,_|_|\__,_|_|       \____|_____|___|
#                 |___/
#
# Angular CLI: 18.x.x
# Node: 20.x.x
# Package Manager: npm 10.x.x
# OS: linux x64
```

### ขั้นตอนที่ 3: ติดตั้ง VS Code Extensions

```
Extensions ที่จำเป็น:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
1. Angular Language Service     — autocomplete, type checking
2. ESLint                       — code linting
3. Prettier                     — code formatting
4. Angular Snippets             — code snippets
5. GitLens                      — git history
6. Thunder Client               — API testing
7. Material Icon Theme          — file icons
8. One Dark Pro                 — theme (แนะนำ)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

**ติดตั้งผ่าน command line:**
```bash
code --install-extension Angular.ng-template
code --install-extension dbaeumer.vscode-eslint
code --install-extension esbenp.prettier-vscode
code --install-extension johnpapa.Angular2
code --install-extension eamodio.gitlens
code --install-extension rangav.vscode-thunder-client
code --install-extension PKief.material-icon-theme
```

---

## 6. สร้างโปรเจ็กต์แรก

### สร้างโปรเจ็กต์ใหม่

```bash
# รูปแบบ: ng new <ชื่อโปรเจ็กต์>
ng new my-first-app

# ระบบจะถามคำถาม:
# ? Would you like to add Angular routing? (y/N)  → y
# ? Which stylesheet format would you like to use?
#   ❯ CSS
#     SCSS   (แนะนำสำหรับโปรเจ็กต์จริง)
#     Sass
#     Less

# สร้างด้วย options ทันที (ไม่ต้องตอบคำถาม):
ng new my-first-app --routing --style=scss --standalone

# เข้าไปในโฟลเดอร์
cd my-first-app

# เปิด VS Code
code .

# รัน Development Server
ng serve

# เปิดใน browser: http://localhost:4200
```

### รันด้วย options เพิ่มเติม

```bash
# รันบน port อื่น
ng serve --port 4300

# เปิด browser อัตโนมัติ
ng serve --open
ng serve -o

# รันพร้อม host อื่น (สำหรับ network access)
ng serve --host 0.0.0.0

# รันในโหมด production
ng serve --configuration=production
```

---

## 7. โครงสร้างโปรเจ็กต์

```
my-first-app/
│
├── src/                          ← โค้ดหลักของแอป
│   ├── app/                      ← Angular Application
│   │   ├── app.component.ts      ← Root Component (TypeScript)
│   │   ├── app.component.html    ← Template (HTML)
│   │   ├── app.component.scss    ← Styles (SCSS)
│   │   ├── app.component.spec.ts ← Tests
│   │   ├── app.module.ts         ← Root Module (ถ้าไม่ใช้ Standalone)
│   │   └── app.routes.ts         ← Routing (ถ้าใช้ Standalone)
│   │
│   ├── assets/                   ← รูปภาพ, ไฟล์ static
│   ├── environments/             ← Config ตาม environment
│   │   ├── environment.ts        ← Development config
│   │   └── environment.prod.ts   ← Production config
│   │
│   ├── index.html                ← HTML หลัก
│   ├── main.ts                   ← Entry point
│   └── styles.scss               ← Global styles
│
├── angular.json                  ← Angular CLI config
├── package.json                  ← Dependencies
├── tsconfig.json                 ← TypeScript config
├── tsconfig.app.json             ← TypeScript config for app
├── tsconfig.spec.json            ← TypeScript config for tests
├── .editorconfig                 ← Editor settings
├── .gitignore
└── README.md
```

### อธิบายไฟล์สำคัญ

**`src/main.ts`** — Entry point ของแอป:
```typescript
import { bootstrapApplication } from '@angular/platform-browser';
import { appConfig } from './app/app.config';
import { AppComponent } from './app/app.component';

bootstrapApplication(AppComponent, appConfig)
  .catch((err) => console.error(err));
```

**`src/app/app.component.ts`** — Root Component:
```typescript
import { Component } from '@angular/core';
import { RouterOutlet } from '@angular/router';

@Component({
  selector: 'app-root',
  standalone: true,
  imports: [RouterOutlet],
  templateUrl: './app.component.html',
  styleUrl: './app.component.scss'
})
export class AppComponent {
  title = 'my-first-app';
}
```

**`src/app/app.component.html`** — Template:
```html
<router-outlet />
```

**`angular.json`** — Angular CLI Configuration (บางส่วน):
```json
{
  "$schema": "./node_modules/@angular/cli/lib/config/schema.json",
  "version": 1,
  "newProjectRoot": "projects",
  "projects": {
    "my-first-app": {
      "projectType": "application",
      "architect": {
        "build": {
          "builder": "@angular-devkit/build-angular:application",
          "options": {
            "outputPath": "dist/my-first-app",
            "index": "src/index.html",
            "browser": "src/main.ts",
            "polyfills": ["zone.js"],
            "tsConfig": "tsconfig.app.json",
            "assets": ["src/favicon.ico", "src/assets"],
            "styles": ["src/styles.scss"],
            "scripts": []
          }
        }
      }
    }
  }
}
```

**`src/environments/environment.ts`**:
```typescript
export const environment = {
  production: false,
  apiUrl: 'http://localhost:3000/api',
  appVersion: '1.0.0'
};
```

**`src/environments/environment.prod.ts`**:
```typescript
export const environment = {
  production: true,
  apiUrl: 'https://api.myapp.com/api',
  appVersion: '1.0.0'
};
```

---

## 8. Angular CLI คำสั่งสำคัญ

### คำสั่งสร้าง (Generate)

```bash
# สร้าง Component
ng generate component my-component
ng g c my-component                  # ย่อ
ng g c components/header             # ในโฟลเดอร์ย่อย
ng g c components/header --inline-style --inline-template  # inline

# สร้าง Service
ng generate service my-service
ng g s services/user                 # ในโฟลเดอร์ย่อย

# สร้าง Module
ng generate module my-module
ng g m features/products             # Feature Module

# สร้าง Module พร้อม Routing
ng g m features/products --routing

# สร้าง Directive
ng generate directive highlight
ng g d directives/highlight

# สร้าง Pipe
ng generate pipe filter-name
ng g p pipes/truncate

# สร้าง Interface
ng generate interface models/user
ng g i models/product

# สร้าง Enum
ng generate enum models/status
ng g enum models/user-role

# สร้าง Class
ng generate class models/user
ng g cl models/base-model

# สร้าง Guard
ng generate guard auth
ng g g guards/auth

# สร้าง Interceptor
ng generate interceptor auth
ng g interceptor interceptors/auth

# สร้าง Resolver
ng generate resolver user-detail
ng g r resolvers/user-detail
```

### คำสั่ง Build

```bash
# Build สำหรับ Development
ng build

# Build สำหรับ Production
ng build --configuration=production
ng build --prod  # ย่อ (deprecated ใน Angular 12+)

# Build แบบ Watch (auto rebuild)
ng build --watch

# Build และดู Bundle Size
ng build --stats-json
npx webpack-bundle-analyzer dist/my-app/stats.json
```

### คำสั่ง Test

```bash
# รัน Unit Tests
ng test

# รัน Tests แบบ Single run (สำหรับ CI)
ng test --watch=false
ng test --no-watch

# รัน Tests พร้อม Coverage Report
ng test --code-coverage

# รัน E2E Tests
ng e2e
```

### คำสั่งอื่นๆ

```bash
# ตรวจสอบ code style
ng lint

# อัปเดต Angular
ng update
ng update @angular/core @angular/cli

# ดู version ของทุกอย่าง
ng version

# ดู help
ng help
ng generate --help
```

---

## 9. Dev Tools ที่จำเป็น

### Angular DevTools (Chrome Extension)

ติดตั้งจาก Chrome Web Store: "Angular DevTools"

```
Angular DevTools ทำอะไรได้บ้าง:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✓ ดู Component Tree
✓ Inspect Component properties
✓ ดู Change Detection cycles
✓ Profiling performance
✓ Debug Injector hierarchy
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### VS Code Settings สำหรับ Angular

สร้างไฟล์ `.vscode/settings.json`:
```json
{
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": true
  },
  "typescript.preferences.importModuleSpecifier": "relative",
  "angular.enable-strict-mode-prompt": false,
  "emmet.includeLanguages": {
    "typescript": "html"
  }
}
```

สร้างไฟล์ `.vscode/extensions.json`:
```json
{
  "recommendations": [
    "Angular.ng-template",
    "dbaeumer.vscode-eslint",
    "esbenp.prettier-vscode",
    "johnpapa.Angular2",
    "eamodio.gitlens",
    "PKief.material-icon-theme"
  ]
}
```

### Prettier Configuration

สร้างไฟล์ `.prettierrc`:
```json
{
  "singleQuote": true,
  "trailingComma": "es5",
  "printWidth": 100,
  "tabWidth": 2,
  "semi": true,
  "bracketSpacing": true,
  "arrowParens": "always"
}
```

### ESLint Configuration

สร้างไฟล์ `eslint.config.js` (Angular 18+):
```javascript
// @ts-check
const eslint = require("@eslint/js");
const tseslint = require("typescript-eslint");
const angular = require("angular-eslint");

module.exports = tseslint.config(
  {
    files: ["**/*.ts"],
    extends: [
      eslint.configs.recommended,
      ...tseslint.configs.recommended,
      ...tseslint.configs.stylistic,
      ...angular.configs.tsRecommended,
    ],
    processor: angular.processInlineTemplates,
    rules: {
      "@angular-eslint/directive-selector": [
        "error",
        {
          type: "attribute",
          prefix: "app",
          style: "camelCase",
        },
      ],
      "@angular-eslint/component-selector": [
        "error",
        {
          type: "element",
          prefix: "app",
          style: "kebab-case",
        },
      ],
    },
  },
  {
    files: ["**/*.html"],
    extends: [
      ...angular.configs.templateRecommended,
      ...angular.configs.templateAccessibility,
    ],
    rules: {},
  }
);
```

---

## 10. Workshop: Hello World App

มาสร้างแอปพลิเคชัน Hello World ที่สมบูรณ์กัน!

### ขั้นตอน 1: สร้างโปรเจ็กต์

```bash
ng new hello-world-app --routing --style=scss --standalone
cd hello-world-app
```

### ขั้นตอน 2: แก้ไข App Component

**`src/app/app.component.ts`:**
```typescript
import { Component, signal } from '@angular/core';
import { CommonModule } from '@angular/common';
import { RouterOutlet } from '@angular/router';

@Component({
  selector: 'app-root',
  standalone: true,
  imports: [CommonModule, RouterOutlet],
  templateUrl: './app.component.html',
  styleUrl: './app.component.scss'
})
export class AppComponent {
  // Property binding
  title = 'Hello World App';
  currentDate = new Date();
  
  // Signal (Angular 16+)
  counter = signal(0);
  name = signal('Angular');
  
  // Method
  increment() {
    this.counter.update(v => v + 1);
  }
  
  decrement() {
    this.counter.update(v => v - 1);
  }
  
  reset() {
    this.counter.set(0);
  }
  
  get greeting(): string {
    return `สวัสดี, ${this.name()}!`;
  }
}
```

**`src/app/app.component.html`:**
```html
<div class="app-container">
  <!-- Header -->
  <header class="header">
    <h1>{{ title }}</h1>
    <p class="date">{{ currentDate | date:'EEEE, d MMMM yyyy' }}</p>
  </header>

  <!-- Main Content -->
  <main class="main">
    <!-- Greeting Section -->
    <section class="card">
      <h2>{{ greeting }}</h2>
      <p>ยินดีต้อนรับสู่การเรียน Angular!</p>
    </section>

    <!-- Counter Section -->
    <section class="card">
      <h2>Counter: {{ counter() }}</h2>
      <div class="button-group">
        <button class="btn btn-danger" (click)="decrement()">-</button>
        <button class="btn btn-secondary" (click)="reset()">Reset</button>
        <button class="btn btn-primary" (click)="increment()">+</button>
      </div>
    </section>

    <!-- Name Input Section -->
    <section class="card">
      <h2>ทดสอบ Two-Way Binding</h2>
      <input 
        type="text" 
        class="input"
        [value]="name()"
        (input)="name.set($any($event.target).value)"
        placeholder="พิมพ์ชื่อของคุณ"
      />
      <p class="preview">ผลลัพธ์: {{ greeting }}</p>
    </section>

    <!-- Angular Features Info -->
    <section class="card features">
      <h2>Features ที่ใช้ในตัวอย่างนี้</h2>
      <ul>
        <li>✅ Component</li>
        <li>✅ Interpolation ({{ }})</li>
        <li>✅ Property Binding ([ ])</li>
        <li>✅ Event Binding (( ))</li>
        <li>✅ Pipes (date)</li>
        <li>✅ Signals (signal())</li>
        <li>✅ Methods</li>
        <li>✅ Computed Properties (getter)</li>
      </ul>
    </section>
  </main>

  <footer class="footer">
    <p>Angular Course — Part 01</p>
  </footer>
</div>
```

**`src/app/app.component.scss`:**
```scss
// Variables
$primary: #1976d2;
$secondary: #424242;
$danger: #e53935;
$success: #43a047;
$text: #212121;
$bg: #f5f5f5;
$card-bg: #ffffff;
$border: #e0e0e0;

// Reset & Base
* {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}

body {
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
  background-color: $bg;
  color: $text;
}

// App Container
.app-container {
  min-height: 100vh;
  display: flex;
  flex-direction: column;
}

// Header
.header {
  background: $primary;
  color: white;
  padding: 2rem;
  text-align: center;
  box-shadow: 0 2px 8px rgba(0,0,0,0.2);

  h1 {
    font-size: 2rem;
    margin-bottom: 0.5rem;
  }

  .date {
    opacity: 0.8;
    font-size: 0.9rem;
  }
}

// Main Content
.main {
  flex: 1;
  max-width: 800px;
  margin: 0 auto;
  padding: 2rem;
  width: 100%;
  display: flex;
  flex-direction: column;
  gap: 1.5rem;
}

// Card
.card {
  background: $card-bg;
  border-radius: 8px;
  padding: 1.5rem;
  box-shadow: 0 2px 4px rgba(0,0,0,0.08);
  border: 1px solid $border;

  h2 {
    font-size: 1.25rem;
    margin-bottom: 1rem;
    color: $primary;
  }

  p {
    line-height: 1.6;
    color: #555;
  }
}

// Buttons
.button-group {
  display: flex;
  gap: 0.75rem;
  margin-top: 0.5rem;
}

.btn {
  padding: 0.5rem 1.5rem;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 1rem;
  font-weight: 500;
  transition: all 0.2s ease;

  &:hover {
    opacity: 0.85;
    transform: translateY(-1px);
  }

  &:active {
    transform: translateY(0);
  }

  &.btn-primary {
    background: $primary;
    color: white;
  }

  &.btn-secondary {
    background: $secondary;
    color: white;
  }

  &.btn-danger {
    background: $danger;
    color: white;
  }
}

// Input
.input {
  width: 100%;
  padding: 0.75rem;
  border: 2px solid $border;
  border-radius: 4px;
  font-size: 1rem;
  transition: border-color 0.2s;

  &:focus {
    outline: none;
    border-color: $primary;
  }
}

.preview {
  margin-top: 0.75rem;
  padding: 0.75rem;
  background: #e3f2fd;
  border-radius: 4px;
  color: $primary !important;
  font-weight: 500;
}

// Features list
.features {
  ul {
    list-style: none;
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
    gap: 0.5rem;

    li {
      padding: 0.5rem;
      background: #f8f9fa;
      border-radius: 4px;
      font-size: 0.9rem;
    }
  }
}

// Footer
.footer {
  background: $secondary;
  color: white;
  text-align: center;
  padding: 1rem;
  font-size: 0.85rem;
  opacity: 0.9;
}

// Responsive
@media (max-width: 600px) {
  .header h1 {
    font-size: 1.5rem;
  }

  .main {
    padding: 1rem;
  }

  .button-group {
    flex-wrap: wrap;
  }
}
```

**`src/styles.scss`** (Global Styles):
```scss
/* Global Styles */
html, body {
  height: 100%;
  margin: 0;
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
}

/* CSS Custom Properties (Variables) */
:root {
  --primary-color: #1976d2;
  --secondary-color: #424242;
  --accent-color: #ff4081;
  --background: #f5f5f5;
  --surface: #ffffff;
  --text-primary: #212121;
  --text-secondary: #757575;
}
```

### ขั้นตอน 3: รันแอป

```bash
ng serve --open
```

เปิด browser ที่ http://localhost:4200 แล้วจะเห็นแอปที่มี:
- Header แสดงชื่อแอปและวันที่
- Counter ที่กด + และ - ได้
- Input ที่เปลี่ยน greeting ได้ทันที
- รายการ features ที่ใช้

### ขั้นตอน 4: Build Production

```bash
ng build --configuration=production

# ไฟล์ output อยู่ใน dist/hello-world-app/browser/
ls dist/hello-world-app/browser/
```

---

## สรุปบทที่ 1

ในบทนี้เราได้เรียนรู้:

| หัวข้อ | สิ่งที่ได้เรียนรู้ |
|--------|------------------|
| Angular คืออะไร | Framework ของ Google สำหรับ SPA |
| ประวัติ | AngularJS → Angular 2 → Angular 18+ |
| เปรียบเทียบ | Angular vs React vs Vue |
| สถาปัตยกรรม | Component, Module, Service, Directive, Pipe |
| การติดตั้ง | Node.js, Angular CLI, VS Code |
| Angular CLI | ng new, ng serve, ng build, ng generate |
| โครงสร้างโปรเจ็กต์ | src/, angular.json, tsconfig.json |
| Workshop | Hello World App ด้วย Signals |

---

## แบบฝึกหัด

1. **ง่าย**: สร้างโปรเจ็กต์ใหม่ชื่อ `my-portfolio` และเพิ่มข้อมูลส่วนตัวของคุณใน Component
2. **ปานกลาง**: เพิ่ม Dark Mode Toggle ใน Hello World App
3. **ท้าทาย**: สร้าง Todo List อย่างง่าย (add/remove/toggle) ใน 1 Component

---

## บทถัดไป

[Part 02 — TypeScript พื้นฐานสำหรับ Angular →](part-02-typescript-basics.md)

TypeScript เป็นพื้นฐานสำคัญของ Angular เราจะเรียนรู้:
- Types, Interfaces, Enums
- Classes และ OOP
- Generics
- Decorators
- และอื่นๆ ที่จำเป็นสำหรับ Angular
