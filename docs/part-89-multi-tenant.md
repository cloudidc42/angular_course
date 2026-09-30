# Part 89: Multi-Tenant Architecture ใน Angular

## Multi-Tenant คืออะไร

แอปเดียวที่รองรับหลาย tenants (องค์กร/ลูกค้า) แต่ละ tenant มี configuration, data, theme แยกกัน

---

## 1. Tenant Service

```typescript
// core/tenant/tenant.service.ts
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { BehaviorSubject, Observable } from 'rxjs';
import { tap } from 'rxjs/operators';

export interface TenantConfig {
  id: string;
  name: string;
  slug: string;
  domain: string;
  logoUrl: string;
  faviconUrl: string;
  theme: TenantTheme;
  features: string[];
  settings: Record<string, any>;
  plan: 'free' | 'starter' | 'pro' | 'enterprise';
  locale: string;
  timezone: string;
  currency: string;
}

export interface TenantTheme {
  primaryColor: string;
  secondaryColor: string;
  accentColor: string;
  backgroundColor: string;
  textColor: string;
  fontFamily: string;
  borderRadius: string;
  logoPosition: 'left' | 'center';
  darkMode: boolean;
}

@Injectable({ providedIn: 'root' })
export class TenantService {
  private currentTenant$ = new BehaviorSubject<TenantConfig | null>(null);
  
  tenant$ = this.currentTenant$.asObservable();

  constructor(private http: HttpClient) {}

  // โหลด tenant จาก subdomain หรือ path
  initialize(): Observable<TenantConfig> {
    const tenantSlug = this.detectTenantSlug();
    
    return this.http.get<TenantConfig>(`/api/tenants/${tenantSlug}`).pipe(
      tap(tenant => {
        this.currentTenant$.next(tenant);
        this.applyTheme(tenant.theme);
        this.setDocumentMeta(tenant);
      })
    );
  }

  private detectTenantSlug(): string {
    const hostname = window.location.hostname;
    
    // subdomain: tenant1.myapp.com → 'tenant1'
    const subdomainMatch = hostname.match(/^([^.]+)\./);
    if (subdomainMatch && subdomainMatch[1] !== 'www' && subdomainMatch[1] !== 'app') {
      return subdomainMatch[1];
    }

    // path: myapp.com/tenant/tenant1 → 'tenant1'
    const pathMatch = window.location.pathname.match(/^\/tenant\/([^/]+)/);
    if (pathMatch) return pathMatch[1];

    // query param: myapp.com?tenant=tenant1
    const urlParams = new URLSearchParams(window.location.search);
    const tenantParam = urlParams.get('tenant');
    if (tenantParam) return tenantParam;

    return 'default';
  }

  private applyTheme(theme: TenantTheme): void {
    const root = document.documentElement;
    
    root.style.setProperty('--primary-color', theme.primaryColor);
    root.style.setProperty('--secondary-color', theme.secondaryColor);
    root.style.setProperty('--accent-color', theme.accentColor);
    root.style.setProperty('--bg-color', theme.backgroundColor);
    root.style.setProperty('--text-color', theme.textColor);
    root.style.setProperty('--font-family', theme.fontFamily);
    root.style.setProperty('--border-radius', theme.borderRadius);
    
    if (theme.darkMode) {
      document.body.setAttribute('data-theme', 'dark');
    }
  }

  private setDocumentMeta(tenant: TenantConfig): void {
    document.title = tenant.name;
    
    const favicon = document.getElementById('favicon') as HTMLLinkElement;
    if (favicon && tenant.faviconUrl) {
      favicon.href = tenant.faviconUrl;
    }

    const metaDescription = document.querySelector('meta[name="description"]');
    if (metaDescription) {
      metaDescription.setAttribute('content', `${tenant.name} - Powered by MyApp`);
    }
  }

  get currentTenant(): TenantConfig | null {
    return this.currentTenant$.value;
  }

  hasFeature(feature: string): boolean {
    return this.currentTenant?.features.includes(feature) ?? false;
  }

  getSetting<T>(key: string, defaultValue?: T): T | undefined {
    return this.currentTenant?.settings[key] as T ?? defaultValue;
  }

  isPlan(plan: TenantConfig['plan']): boolean {
    const planOrder: TenantConfig['plan'][] = ['free', 'starter', 'pro', 'enterprise'];
    const currentIndex = planOrder.indexOf(this.currentTenant?.plan || 'free');
    const requiredIndex = planOrder.indexOf(plan);
    return currentIndex >= requiredIndex;
  }
}
```

---

## 2. Tenant Interceptor

```typescript
// core/tenant/tenant.interceptor.ts
import { Injectable } from '@angular/core';
import { HttpInterceptor, HttpRequest, HttpHandler } from '@angular/common/http';
import { TenantService } from './tenant.service';

@Injectable()
export class TenantInterceptor implements HttpInterceptor {
  constructor(private tenantService: TenantService) {}

  intercept(req: HttpRequest<any>, next: HttpHandler) {
    const tenant = this.tenantService.currentTenant;
    
    if (tenant && req.url.includes('/api/')) {
      const cloned = req.clone({
        headers: req.headers
          .set('X-Tenant-ID', tenant.id)
          .set('X-Tenant-Slug', tenant.slug)
      });
      return next.handle(cloned);
    }
    
    return next.handle(req);
  }
}
```

---

## 3. Tenant-Aware Component

```typescript
// shared/components/tenant-logo/tenant-logo.component.ts
import { Component, OnInit } from '@angular/core';
import { TenantService, TenantConfig } from '../../../core/tenant/tenant.service';
import { Observable } from 'rxjs';

@Component({
  selector: 'app-tenant-logo',
  template: `
    <ng-container *ngIf="tenant$ | async as tenant">
      <img 
        [src]="tenant.logoUrl" 
        [alt]="tenant.name + ' Logo'"
        class="tenant-logo"
        [class.center]="tenant.theme.logoPosition === 'center'"
      >
    </ng-container>
  `,
  styles: [`
    .tenant-logo { max-height: 48px; width: auto; }
    .center { display: block; margin: 0 auto; }
  `]
})
export class TenantLogoComponent {
  tenant$ = this.tenantService.tenant$;

  constructor(private tenantService: TenantService) {}
}
```

---

## 4. Feature Guard สำหรับ Tenant

```typescript
// core/tenant/tenant-feature.guard.ts
import { Injectable } from '@angular/core';
import { CanActivate, ActivatedRouteSnapshot, Router } from '@angular/router';
import { TenantService } from './tenant.service';

@Injectable({ providedIn: 'root' })
export class TenantFeatureGuard implements CanActivate {
  constructor(
    private tenantService: TenantService,
    private router: Router
  ) {}

  canActivate(route: ActivatedRouteSnapshot): boolean {
    const requiredFeature = route.data['feature'] as string;
    const requiredPlan = route.data['plan'] as TenantConfig['plan'];
    
    if (requiredFeature && !this.tenantService.hasFeature(requiredFeature)) {
      this.router.navigate(['/upgrade'], { 
        queryParams: { feature: requiredFeature }
      });
      return false;
    }

    if (requiredPlan && !this.tenantService.isPlan(requiredPlan)) {
      this.router.navigate(['/upgrade'], { 
        queryParams: { plan: requiredPlan }
      });
      return false;
    }

    return true;
  }
}

// Routing
// {
//   path: 'reports/advanced',
//   component: AdvancedReportsComponent,
//   canActivate: [TenantFeatureGuard],
//   data: { feature: 'advanced-reports', plan: 'pro' }
// }
```

---

## 5. Dynamic Theming

```typescript
// core/tenant/theme.service.ts
import { Injectable } from '@angular/core';
import { TenantTheme } from './tenant.service';

@Injectable({ providedIn: 'root' })
export class ThemeService {
  applyTheme(theme: TenantTheme): void {
    const root = document.documentElement;
    
    // CSS Custom Properties
    const properties: Record<string, string> = {
      '--color-primary': theme.primaryColor,
      '--color-primary-dark': this.darken(theme.primaryColor, 15),
      '--color-primary-light': this.lighten(theme.primaryColor, 80),
      '--color-secondary': theme.secondaryColor,
      '--color-accent': theme.accentColor,
      '--color-bg': theme.backgroundColor,
      '--color-text': theme.textColor,
      '--font-family': theme.fontFamily,
      '--border-radius': theme.borderRadius,
      '--border-radius-lg': `calc(${theme.borderRadius} * 2)`,
      '--border-radius-sm': `calc(${theme.borderRadius} / 2)`
    };

    Object.entries(properties).forEach(([prop, value]) => {
      root.style.setProperty(prop, value);
    });

    // Apply dark mode
    if (theme.darkMode) {
      document.body.classList.add('dark-theme');
    } else {
      document.body.classList.remove('dark-theme');
    }
  }

  private darken(hex: string, amount: number): string {
    return this.adjustColor(hex, -amount);
  }

  private lighten(hex: string, amount: number): string {
    return this.adjustColor(hex, amount);
  }

  private adjustColor(hex: string, amount: number): string {
    const num = parseInt(hex.replace('#', ''), 16);
    const r = Math.max(0, Math.min(255, (num >> 16) + amount));
    const g = Math.max(0, Math.min(255, ((num >> 8) & 0x00FF) + amount));
    const b = Math.max(0, Math.min(255, (num & 0x0000FF) + amount));
    return '#' + ((1 << 24) + (r << 16) + (g << 8) + b).toString(16).slice(1);
  }

  getContrastColor(hexColor: string): 'white' | 'black' {
    const r = parseInt(hexColor.slice(1, 3), 16);
    const g = parseInt(hexColor.slice(3, 5), 16);
    const b = parseInt(hexColor.slice(5, 7), 16);
    const luminance = (0.299 * r + 0.587 * g + 0.114 * b) / 255;
    return luminance > 0.5 ? 'black' : 'white';
  }
}
```

---

## 6. Tenant Settings Component

```typescript
// features/settings/tenant-settings.component.ts
import { Component, OnInit } from '@angular/core';
import { FormBuilder, FormGroup } from '@angular/forms';
import { TenantService } from '../../core/tenant/tenant.service';
import { ThemeService } from '../../core/tenant/theme.service';

@Component({
  selector: 'app-tenant-settings',
  template: `
    <div class="settings-page">
      <h2>ตั้งค่าองค์กร</h2>
      
      <form [formGroup]="settingsForm" (ngSubmit)="saveSettings()">
        <div class="section">
          <h3>Theme</h3>
          <div class="color-pickers">
            <label>
              สีหลัก
              <input type="color" formControlName="primaryColor" (change)="previewTheme()">
            </label>
            <label>
              สีรอง
              <input type="color" formControlName="secondaryColor" (change)="previewTheme()">
            </label>
          </div>
          
          <label>
            Font Family
            <select formControlName="fontFamily">
              <option value="'Sarabun', sans-serif">Sarabun (ไทย)</option>
              <option value="'Noto Sans Thai', sans-serif">Noto Sans Thai</option>
              <option value="'Inter', sans-serif">Inter (English)</option>
            </select>
          </label>

          <label class="toggle-label">
            Dark Mode
            <input type="checkbox" formControlName="darkMode" (change)="previewTheme()">
          </label>
        </div>

        <div class="preview-section">
          <h3>Preview</h3>
          <div class="preview-card" [style.background]="'var(--color-bg)'">
            <button class="preview-btn" [style.background]="settingsForm.get('primaryColor')?.value">
              ปุ่มหลัก
            </button>
            <p [style.color]="'var(--color-text)'">ตัวอักษรปกติ</p>
          </div>
        </div>

        <button type="submit" [disabled]="isSaving" class="btn-save">
          {{ isSaving ? 'กำลังบันทึก...' : 'บันทึกการตั้งค่า' }}
        </button>
      </form>
    </div>
  `
})
export class TenantSettingsComponent implements OnInit {
  settingsForm!: FormGroup;
  isSaving = false;

  constructor(
    private fb: FormBuilder,
    private tenantService: TenantService,
    private themeService: ThemeService
  ) {}

  ngOnInit(): void {
    const tenant = this.tenantService.currentTenant;
    
    this.settingsForm = this.fb.group({
      primaryColor: [tenant?.theme.primaryColor || '#2196f3'],
      secondaryColor: [tenant?.theme.secondaryColor || '#9c27b0'],
      fontFamily: [tenant?.theme.fontFamily || "'Sarabun', sans-serif"],
      darkMode: [tenant?.theme.darkMode || false]
    });
  }

  previewTheme(): void {
    const values = this.settingsForm.value;
    this.themeService.applyTheme({
      ...this.tenantService.currentTenant!.theme,
      primaryColor: values.primaryColor,
      secondaryColor: values.secondaryColor,
      fontFamily: values.fontFamily,
      darkMode: values.darkMode
    });
  }

  saveSettings(): void {
    this.isSaving = true;
    // API call to save...
    setTimeout(() => this.isSaving = false, 1000);
  }
}

interface TenantConfig {
  plan: 'free' | 'starter' | 'pro' | 'enterprise';
  theme: any;
  features: string[];
  settings: any;
  name: string;
  id: string;
  slug: string;
  domain: string;
  logoUrl: string;
  faviconUrl: string;
  locale: string;
  timezone: string;
  currency: string;
}
```

---

## สรุป

| Component | หน้าที่ |
|-----------|---------|
| TenantService | โหลด/เก็บ tenant config |
| TenantInterceptor | เพิ่ม tenant headers ทุก request |
| TenantFeatureGuard | ป้องกัน routes ตาม features |
| ThemeService | Apply CSS variables |

### Tenant Isolation

1. **Data**: ทุก API call ส่ง tenant ID
2. **Theme**: CSS variables ต่างกันต่อ tenant
3. **Features**: Enable/disable ตาม plan
4. **Routing**: บาง routes ต้องการ feature
5. **Storage**: Prefix localStorage ด้วย tenant ID
