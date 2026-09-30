# Part 90: White-Label App ใน Angular

## White-Label คืออะไร

White-label คือ product ที่ client นำไป rebrand เป็นของตัวเอง แตกต่างจาก Multi-tenant คือแต่ละ client จะ deploy เป็น app แยก

---

## 1. Brand Configuration System

```typescript
// core/brand/brand.config.ts

export interface BrandConfig {
  name: string;
  shortName: string;
  tagline: string;
  description: string;
  logo: {
    light: string;
    dark: string;
    icon: string;
    favicon: string;
  };
  colors: {
    primary: string;
    primaryDark: string;
    primaryLight: string;
    secondary: string;
    accent: string;
    success: string;
    warning: string;
    error: string;
    background: string;
    surface: string;
    text: string;
    textSecondary: string;
    border: string;
  };
  typography: {
    fontFamilyHeading: string;
    fontFamilyBody: string;
    fontUrl?: string;
  };
  spacing: {
    borderRadius: string;
    borderRadiusLg: string;
  };
  contact: {
    email: string;
    phone: string;
    website: string;
    address: string;
  };
  social: {
    facebook?: string;
    instagram?: string;
    twitter?: string;
    line?: string;
  };
  legal: {
    companyName: string;
    taxId?: string;
    privacyPolicyUrl: string;
    termsOfServiceUrl: string;
  };
  features: {
    [key: string]: boolean;
  };
}
```

---

## 2. Brand Files

### แบรนด์ A (เช่น FinTech)

```typescript
// brands/fintech/brand.config.ts
import { BrandConfig } from '../../core/brand/brand.config';

export const FINTECH_BRAND: BrandConfig = {
  name: 'FinanceMax',
  shortName: 'FM',
  tagline: 'จัดการการเงินอัจฉริยะ',
  description: 'แพลตฟอร์มบริหารการเงินสำหรับธุรกิจยุคใหม่',
  logo: {
    light: '/assets/brands/fintech/logo-light.svg',
    dark: '/assets/brands/fintech/logo-dark.svg',
    icon: '/assets/brands/fintech/icon.svg',
    favicon: '/assets/brands/fintech/favicon.ico'
  },
  colors: {
    primary: '#1a237e',
    primaryDark: '#121858',
    primaryLight: '#e8eaf6',
    secondary: '#26c6da',
    accent: '#ffd54f',
    success: '#4caf50',
    warning: '#ff9800',
    error: '#f44336',
    background: '#f5f7fa',
    surface: '#ffffff',
    text: '#1a1a2e',
    textSecondary: '#666680',
    border: '#e0e0ef'
  },
  typography: {
    fontFamilyHeading: "'IBM Plex Sans Thai', sans-serif",
    fontFamilyBody: "'IBM Plex Sans Thai', sans-serif",
    fontUrl: 'https://fonts.googleapis.com/css2?family=IBM+Plex+Sans+Thai:wght@300;400;500;600;700&display=swap'
  },
  spacing: {
    borderRadius: '4px',
    borderRadiusLg: '8px'
  },
  contact: {
    email: 'support@financemax.co.th',
    phone: '02-XXX-XXXX',
    website: 'https://financemax.co.th',
    address: '123 สีลม กรุงเทพมหานคร 10500'
  },
  social: {
    facebook: 'https://fb.com/financemax',
    line: '@financemax'
  },
  legal: {
    companyName: 'บริษัท ไฟแนนซ์แม็กซ์ จำกัด',
    taxId: '0105565012345',
    privacyPolicyUrl: '/privacy',
    termsOfServiceUrl: '/terms'
  },
  features: {
    multiCurrency: true,
    taxReport: true,
    inventory: false,
    ecommerce: false
  }
};
```

### แบรนด์ B (E-commerce)

```typescript
// brands/ecommerce/brand.config.ts
export const ECOMMERCE_BRAND: BrandConfig = {
  name: 'ShopPro',
  shortName: 'SP',
  tagline: 'ขายของง่าย กำไรงาม',
  description: 'ระบบจัดการร้านค้าออนไลน์ครบวงจร',
  logo: {
    light: '/assets/brands/ecommerce/logo-light.png',
    dark: '/assets/brands/ecommerce/logo-dark.png',
    icon: '/assets/brands/ecommerce/icon.png',
    favicon: '/assets/brands/ecommerce/favicon.ico'
  },
  colors: {
    primary: '#e91e63',
    primaryDark: '#c2185b',
    primaryLight: '#fce4ec',
    secondary: '#ff6f00',
    accent: '#00bcd4',
    success: '#4caf50',
    warning: '#ff9800',
    error: '#f44336',
    background: '#fff8f9',
    surface: '#ffffff',
    text: '#2d1b1b',
    textSecondary: '#78595e',
    border: '#ffd0d8'
  },
  typography: {
    fontFamilyHeading: "'Prompt', sans-serif",
    fontFamilyBody: "'Sarabun', sans-serif",
    fontUrl: 'https://fonts.googleapis.com/css2?family=Prompt:wght@400;600;700&family=Sarabun:wght@300;400;500&display=swap'
  },
  spacing: {
    borderRadius: '12px',
    borderRadiusLg: '20px'
  },
  contact: {
    email: 'hello@shoppro.co.th',
    phone: '065-XXX-XXXX',
    website: 'https://shoppro.co.th',
    address: '456 เยาวราช กรุงเทพมหานคร 10100'
  },
  social: {
    facebook: 'https://fb.com/shoppro',
    instagram: 'https://instagram.com/shoppro',
    line: '@shoppro'
  },
  legal: {
    companyName: 'บริษัท ช็อปโปร จำกัด',
    privacyPolicyUrl: '/privacy',
    termsOfServiceUrl: '/terms'
  },
  features: {
    multiCurrency: false,
    taxReport: false,
    inventory: true,
    ecommerce: true
  }
};
```

---

## 3. Brand Service

```typescript
// core/brand/brand.service.ts
import { Injectable } from '@angular/core';
import { BrandConfig } from './brand.config';
import { BRAND_CONFIG } from './brand.token';
import { Inject } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class BrandService {
  constructor(@Inject(BRAND_CONFIG) private config: BrandConfig) {
    this.applyBrand();
  }

  private applyBrand(): void {
    this.applyCSSVariables();
    this.loadFonts();
    this.setMetaTags();
    this.setFavicon();
  }

  private applyCSSVariables(): void {
    const root = document.documentElement;
    const { colors, typography, spacing } = this.config;

    const vars: Record<string, string> = {
      '--brand-primary': colors.primary,
      '--brand-primary-dark': colors.primaryDark,
      '--brand-primary-light': colors.primaryLight,
      '--brand-secondary': colors.secondary,
      '--brand-accent': colors.accent,
      '--brand-success': colors.success,
      '--brand-warning': colors.warning,
      '--brand-error': colors.error,
      '--brand-background': colors.background,
      '--brand-surface': colors.surface,
      '--brand-text': colors.text,
      '--brand-text-secondary': colors.textSecondary,
      '--brand-border': colors.border,
      '--brand-font-heading': typography.fontFamilyHeading,
      '--brand-font-body': typography.fontFamilyBody,
      '--brand-radius': spacing.borderRadius,
      '--brand-radius-lg': spacing.borderRadiusLg
    };

    Object.entries(vars).forEach(([prop, value]) => {
      root.style.setProperty(prop, value);
    });
  }

  private loadFonts(): void {
    if (!this.config.typography.fontUrl) return;

    const preload = document.createElement('link');
    preload.rel = 'preconnect';
    preload.href = 'https://fonts.googleapis.com';
    document.head.appendChild(preload);

    const link = document.createElement('link');
    link.rel = 'stylesheet';
    link.href = this.config.typography.fontUrl;
    document.head.appendChild(link);
  }

  private setMetaTags(): void {
    document.title = this.config.name;
    
    const setMeta = (name: string, content: string) => {
      let meta = document.querySelector(`meta[name="${name}"]`);
      if (!meta) {
        meta = document.createElement('meta');
        meta.setAttribute('name', name);
        document.head.appendChild(meta);
      }
      meta.setAttribute('content', content);
    };

    setMeta('description', this.config.description);
    setMeta('application-name', this.config.name);
    setMeta('theme-color', this.config.colors.primary);
    
    // OG tags
    const setOG = (property: string, content: string) => {
      let meta = document.querySelector(`meta[property="${property}"]`);
      if (!meta) {
        meta = document.createElement('meta');
        meta.setAttribute('property', property);
        document.head.appendChild(meta);
      }
      meta.setAttribute('content', content);
    };

    setOG('og:title', this.config.name);
    setOG('og:description', this.config.description);
    setOG('og:image', this.config.logo.light);
  }

  private setFavicon(): void {
    const favicon = document.getElementById('favicon') as HTMLLinkElement;
    if (favicon) {
      favicon.href = this.config.logo.favicon;
    }
  }

  get name(): string { return this.config.name; }
  get logo(): BrandConfig['logo'] { return this.config.logo; }
  get colors(): BrandConfig['colors'] { return this.config.colors; }
  get contact(): BrandConfig['contact'] { return this.config.contact; }
  get social(): BrandConfig['social'] { return this.config.social; }
  get legal(): BrandConfig['legal'] { return this.config.legal; }
  
  hasFeature(feature: string): boolean {
    return this.config.features[feature] ?? false;
  }
}
```

---

## 4. Brand Token

```typescript
// core/brand/brand.token.ts
import { InjectionToken } from '@angular/core';
import { BrandConfig } from './brand.config';

export const BRAND_CONFIG = new InjectionToken<BrandConfig>('BRAND_CONFIG');
```

---

## 5. Build Configuration

```typescript
// app.module.ts - Fintech Version
import { BRAND_CONFIG } from './core/brand/brand.token';
import { FINTECH_BRAND } from './brands/fintech/brand.config';

@NgModule({
  providers: [
    { provide: BRAND_CONFIG, useValue: FINTECH_BRAND }
  ]
})
export class AppModule {}
```

```json
// angular.json - ใช้ fileReplacements
{
  "configurations": {
    "fintech": {
      "fileReplacements": [
        {
          "replace": "src/environments/environment.ts",
          "with": "src/environments/environment.fintech.ts"
        },
        {
          "replace": "src/app/app.module.ts",
          "with": "src/app/app.module.fintech.ts"
        }
      ]
    },
    "ecommerce": {
      "fileReplacements": [
        {
          "replace": "src/environments/environment.ts",
          "with": "src/environments/environment.ecommerce.ts"
        },
        {
          "replace": "src/app/app.module.ts",
          "with": "src/app/app.module.ecommerce.ts"
        }
      ]
    }
  }
}
```

```bash
# Build สำหรับแต่ละ brand
ng build --configuration fintech
ng build --configuration ecommerce

# หรือ custom scripts
npm run build:fintech
npm run build:ecommerce
```

---

## 6. Footer Component ที่ใช้ Brand

```typescript
// shared/components/footer/footer.component.ts
import { Component } from '@angular/core';
import { BrandService } from '../../../core/brand/brand.service';

@Component({
  selector: 'app-footer',
  template: `
    <footer class="app-footer">
      <div class="footer-content">
        <div class="footer-brand">
          <img [src]="brand.logo.light" [alt]="brand.name" height="32">
          <p>{{ brandService.config?.tagline }}</p>
        </div>
        
        <div class="footer-links">
          <a [href]="brand.legal.privacyPolicyUrl">นโยบายความเป็นส่วนตัว</a>
          <a [href]="brand.legal.termsOfServiceUrl">เงื่อนไขการใช้บริการ</a>
          <a href="/contact">ติดต่อเรา</a>
        </div>

        <div class="footer-social">
          <a *ngIf="social.facebook" [href]="social.facebook" target="_blank" rel="noopener">Facebook</a>
          <a *ngIf="social.line" [href]="'https://line.me/R/ti/p/' + social.line" target="_blank" rel="noopener">Line</a>
          <a *ngIf="social.instagram" [href]="social.instagram" target="_blank" rel="noopener">Instagram</a>
        </div>
      </div>
      
      <div class="footer-bottom">
        <p>© {{ currentYear }} {{ brand.legal.companyName }} สงวนสิทธิ์ทุกประการ</p>
      </div>
    </footer>
  `
})
export class FooterComponent {
  brand = this.brandService.logo ? this.brandService : null;
  social = this.brandService.social;
  currentYear = new Date().getFullYear();

  constructor(public brandService: BrandService) {}
}
```

---

## สรุป

| แนวทาง | เมื่อไหร่ใช้ |
|--------|------------|
| fileReplacements | Build-time brand switching |
| InjectionToken | Runtime brand config |
| CSS Variables | Dynamic theming |
| Environment files | Per-brand API URLs |

### Checklist White-Label

- [ ] สร้าง brand config interface ที่ครบถ้วน
- [ ] CSS variables ครอบคลุมทุก UI ส่วน
- [ ] Assets แยกตาม brand
- [ ] Font loading per brand
- [ ] Build scripts สำหรับแต่ละ brand
- [ ] Test UI ด้วยทุก brand
