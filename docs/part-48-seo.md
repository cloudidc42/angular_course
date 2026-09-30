# Part 48: SEO ใน Angular

## บทนำ

Search Engine Optimization (SEO) สำคัญมากสำหรับแอปพลิเคชัน web ที่ต้องการให้ search engines ค้นพบ ในบทนี้เราจะเรียนรู้การจัดการ Meta tags, Title service, Open Graph, Twitter Cards และ Schema.org structured data

---

## 1. Title Service

```typescript
// app/services/seo.service.ts
import { Injectable, inject } from '@angular/core';
import { Title, Meta } from '@angular/platform-browser';
import { Router, NavigationEnd, ActivatedRoute } from '@angular/router';
import { filter, map, mergeMap } from 'rxjs/operators';

export interface SeoConfig {
  title?: string;
  description?: string;
  keywords?: string[];
  image?: string;
  url?: string;
  type?: 'website' | 'article' | 'product';
  noIndex?: boolean;
  author?: string;
  publishedAt?: string;
  modifiedAt?: string;
}

@Injectable({ providedIn: 'root' })
export class SeoService {
  private readonly siteName = 'MyShop';
  private readonly siteUrl = 'https://myshop.example.com';
  private readonly defaultImage = '/assets/images/og-default.jpg';
  private readonly defaultDescription = 'ร้านค้าออนไลน์ที่ดีที่สุดในไทย สินค้าหลากหลาย ราคาดี จัดส่งทั่วประเทศ';

  constructor(
    private titleService: Title,
    private meta: Meta,
    private router: Router,
    private activatedRoute: ActivatedRoute
  ) {}

  // อัปเดต SEO metadata
  update(config: SeoConfig = {}) {
    const {
      title = '',
      description = this.defaultDescription,
      keywords = [],
      image = this.defaultImage,
      url = this.router.url,
      type = 'website',
      noIndex = false,
      author,
      publishedAt,
      modifiedAt
    } = config;

    const fullTitle = title ? `${title} | ${this.siteName}` : this.siteName;
    const fullUrl = `${this.siteUrl}${url}`;
    const fullImage = image.startsWith('http') ? image : `${this.siteUrl}${image}`;

    // Title
    this.titleService.setTitle(fullTitle);

    // Basic Meta
    this.setMeta('description', description);
    if (keywords.length > 0) {
      this.setMeta('keywords', keywords.join(', '));
    }
    if (author) this.setMeta('author', author);

    // Robots
    this.setMeta('robots', noIndex ? 'noindex, nofollow' : 'index, follow');
    this.setMeta('googlebot', noIndex ? 'noindex' : 'index, follow');

    // Canonical URL
    this.setLink('canonical', fullUrl);

    // Open Graph
    this.setOgMeta('og:title', fullTitle);
    this.setOgMeta('og:description', description);
    this.setOgMeta('og:image', fullImage);
    this.setOgMeta('og:url', fullUrl);
    this.setOgMeta('og:type', type);
    this.setOgMeta('og:site_name', this.siteName);
    this.setOgMeta('og:locale', 'th_TH');

    // Open Graph Image details
    this.setOgMeta('og:image:width', '1200');
    this.setOgMeta('og:image:height', '630');
    this.setOgMeta('og:image:alt', title || this.siteName);

    // Twitter Card
    this.setMeta('twitter:card', 'summary_large_image');
    this.setMeta('twitter:site', '@myshop_th');
    this.setMeta('twitter:title', fullTitle);
    this.setMeta('twitter:description', description);
    this.setMeta('twitter:image', fullImage);

    // Article-specific
    if (type === 'article') {
      if (author) this.setOgMeta('article:author', author);
      if (publishedAt) this.setOgMeta('article:published_time', publishedAt);
      if (modifiedAt) this.setOgMeta('article:modified_time', modifiedAt);
    }
  }

  private setMeta(name: string, content: string) {
    if (this.meta.getTag(`name='${name}'`)) {
      this.meta.updateTag({ name, content });
    } else {
      this.meta.addTag({ name, content });
    }
  }

  private setOgMeta(property: string, content: string) {
    if (this.meta.getTag(`property='${property}'`)) {
      this.meta.updateTag({ property, content });
    } else {
      this.meta.addTag({ property, content });
    }
  }

  private setLink(rel: string, href: string) {
    // ใช้ Renderer2 หรือ document inject เพื่อจัดการ link elements
    const existingLink = document.querySelector(`link[rel="${rel}"]`);
    if (existingLink) {
      existingLink.setAttribute('href', href);
    } else {
      const link = document.createElement('link');
      link.setAttribute('rel', rel);
      link.setAttribute('href', href);
      document.head.appendChild(link);
    }
  }

  // Auto-update จาก route data
  initRouteListener() {
    this.router.events
      .pipe(
        filter(event => event instanceof NavigationEnd),
        map(() => this.activatedRoute),
        map(route => {
          while (route.firstChild) route = route.firstChild;
          return route;
        }),
        mergeMap(route => route.data)
      )
      .subscribe(data => {
        if (data['seo']) {
          this.update(data['seo']);
        }
      });
  }
}
```

---

## 2. Schema.org Structured Data

```typescript
// app/services/structured-data.service.ts
import { Injectable, Renderer2, Inject } from '@angular/core';
import { DOCUMENT } from '@angular/common';

@Injectable({ providedIn: 'root' })
export class StructuredDataService {
  private scripts = new Map<string, HTMLScriptElement>();

  constructor(
    @Inject(DOCUMENT) private document: Document
  ) {}

  // Organization Schema
  addOrganization() {
    const schema = {
      '@context': 'https://schema.org',
      '@type': 'Organization',
      'name': 'MyShop',
      'url': 'https://myshop.example.com',
      'logo': 'https://myshop.example.com/assets/logo.png',
      'contactPoint': {
        '@type': 'ContactPoint',
        'telephone': '+66-2-123-4567',
        'contactType': 'customer service',
        'availableLanguage': ['Thai', 'English']
      },
      'sameAs': [
        'https://facebook.com/myshop',
        'https://twitter.com/myshop',
        'https://instagram.com/myshop'
      ]
    };

    this.addSchema('organization', schema);
  }

  // Product Schema
  addProduct(product: {
    name: string;
    description: string;
    image: string;
    price: number;
    currency: string;
    availability: 'InStock' | 'OutOfStock' | 'PreOrder';
    brand?: string;
    sku?: string;
    rating?: { value: number; count: number };
  }) {
    const schema: any = {
      '@context': 'https://schema.org',
      '@type': 'Product',
      'name': product.name,
      'description': product.description,
      'image': product.image,
      'offers': {
        '@type': 'Offer',
        'price': product.price,
        'priceCurrency': product.currency,
        'availability': `https://schema.org/${product.availability}`
      }
    };

    if (product.brand) {
      schema.brand = { '@type': 'Brand', 'name': product.brand };
    }

    if (product.sku) {
      schema.sku = product.sku;
    }

    if (product.rating) {
      schema.aggregateRating = {
        '@type': 'AggregateRating',
        'ratingValue': product.rating.value,
        'reviewCount': product.rating.count
      };
    }

    this.addSchema('product', schema);
  }

  // Article Schema
  addArticle(article: {
    title: string;
    description: string;
    image: string;
    author: string;
    publishedAt: string;
    modifiedAt?: string;
    url: string;
  }) {
    const schema = {
      '@context': 'https://schema.org',
      '@type': 'Article',
      'headline': article.title,
      'description': article.description,
      'image': article.image,
      'author': {
        '@type': 'Person',
        'name': article.author
      },
      'publisher': {
        '@type': 'Organization',
        'name': 'MyShop',
        'logo': {
          '@type': 'ImageObject',
          'url': 'https://myshop.example.com/assets/logo.png'
        }
      },
      'datePublished': article.publishedAt,
      'dateModified': article.modifiedAt || article.publishedAt,
      'mainEntityOfPage': {
        '@type': 'WebPage',
        '@id': article.url
      }
    };

    this.addSchema('article', schema);
  }

  // Breadcrumb Schema
  addBreadcrumb(items: { name: string; url: string }[]) {
    const schema = {
      '@context': 'https://schema.org',
      '@type': 'BreadcrumbList',
      'itemListElement': items.map((item, index) => ({
        '@type': 'ListItem',
        'position': index + 1,
        'name': item.name,
        'item': item.url
      }))
    };

    this.addSchema('breadcrumb', schema);
  }

  // FAQ Schema
  addFaq(faqs: { question: string; answer: string }[]) {
    const schema = {
      '@context': 'https://schema.org',
      '@type': 'FAQPage',
      'mainEntity': faqs.map(faq => ({
        '@type': 'Question',
        'name': faq.question,
        'acceptedAnswer': {
          '@type': 'Answer',
          'text': faq.answer
        }
      }))
    };

    this.addSchema('faq', schema);
  }

  private addSchema(id: string, schema: any) {
    this.removeSchema(id);

    const script = this.document.createElement('script');
    script.type = 'application/ld+json';
    script.text = JSON.stringify(schema);
    script.id = `schema-${id}`;

    this.document.head.appendChild(script);
    this.scripts.set(id, script);
  }

  removeSchema(id: string) {
    const existing = this.document.getElementById(`schema-${id}`);
    if (existing) {
      existing.remove();
    }
    this.scripts.delete(id);
  }

  removeAll() {
    this.scripts.forEach((_, id) => this.removeSchema(id));
  }
}
```

---

## 3. SEO ใน Route Configuration

```typescript
// app/app.routes.ts
import { Routes } from '@angular/router';
import { SeoConfig } from './services/seo.service';

export const routes: Routes = [
  {
    path: '',
    loadComponent: () => import('./pages/home/home.component').then(m => m.HomeComponent),
    data: {
      seo: {
        title: 'ร้านค้าออนไลน์',
        description: 'ช้อปสินค้าหลากหลาย ราคาดี จัดส่งทั่วประเทศ',
        keywords: ['ร้านค้าออนไลน์', 'ช้อปปิ้ง', 'สินค้าไทย'],
        type: 'website'
      } as SeoConfig
    }
  },
  {
    path: 'products',
    loadComponent: () => import('./pages/products/products.component').then(m => m.ProductsComponent),
    data: {
      seo: {
        title: 'สินค้าทั้งหมด',
        description: 'เลือกซื้อสินค้าหลากหลายหมวดหมู่',
        keywords: ['สินค้า', 'ซื้อสินค้า', 'ราคาดี']
      } as SeoConfig
    }
  }
];
```

---

## 4. Product Detail Page ที่ SEO-friendly

```typescript
// app/pages/product-detail/product-detail.component.ts
import { Component, OnInit, OnDestroy } from '@angular/core';
import { ActivatedRoute } from '@angular/router';
import { SeoService } from '../../services/seo.service';
import { StructuredDataService } from '../../services/structured-data.service';
import { ProductService } from '../../services/product.service';
import { Subject } from 'rxjs';
import { takeUntil, switchMap } from 'rxjs/operators';

@Component({
  selector: 'app-product-detail',
  template: `
    <div *ngIf="product" class="product-detail">
      <!-- Breadcrumb -->
      <nav aria-label="breadcrumb">
        <ol class="breadcrumb">
          <li><a routerLink="/">หน้าหลัก</a></li>
          <li><a routerLink="/products">สินค้า</a></li>
          <li aria-current="page">{{ product.name }}</li>
        </ol>
      </nav>

      <div class="product-main">
        <div class="product-images">
          <img
            [src]="product.images[activeImage]"
            [alt]="product.name"
            width="600"
            height="600"
            priority
          >
          <div class="thumbnails">
            <img
              *ngFor="let img of product.images; let i = index"
              [src]="img"
              [alt]="product.name + ' - รูปที่ ' + (i + 1)"
              (click)="activeImage = i"
              [class.active]="activeImage === i"
              width="80"
              height="80"
            >
          </div>
        </div>

        <div class="product-info">
          <h1>{{ product.name }}</h1>
          <div class="rating">
            <span>★ {{ product.rating }}</span>
            <span>({{ product.reviewCount }} รีวิว)</span>
          </div>
          <p class="price">฿{{ product.price | number:'1.2-2' }}</p>
          <p class="description">{{ product.description }}</p>

          <button (click)="addToCart()">เพิ่มลงตะกร้า</button>
        </div>
      </div>
    </div>
  `
})
export class ProductDetailComponent implements OnInit, OnDestroy {
  product: any = null;
  activeImage = 0;
  private destroy$ = new Subject<void>();

  constructor(
    private route: ActivatedRoute,
    private seoService: SeoService,
    private structuredData: StructuredDataService,
    private productService: ProductService
  ) {}

  ngOnInit() {
    this.route.params
      .pipe(
        switchMap(params => this.productService.getById(params['id'])),
        takeUntil(this.destroy$)
      )
      .subscribe(product => {
        this.product = product;
        this.updateSeo(product);
      });
  }

  private updateSeo(product: any) {
    // อัปเดต Meta tags
    this.seoService.update({
      title: product.name,
      description: product.description.slice(0, 160),
      keywords: [product.name, product.category, 'ซื้อ', 'ราคา'],
      image: product.images[0],
      type: 'product'
    });

    // เพิ่ม Structured Data
    this.structuredData.addProduct({
      name: product.name,
      description: product.description,
      image: product.images[0],
      price: product.price,
      currency: 'THB',
      availability: product.stock > 0 ? 'InStock' : 'OutOfStock',
      brand: product.brand,
      sku: product.sku,
      rating: { value: product.rating, count: product.reviewCount }
    });

    // เพิ่ม Breadcrumb
    this.structuredData.addBreadcrumb([
      { name: 'หน้าหลัก', url: 'https://myshop.example.com' },
      { name: 'สินค้า', url: 'https://myshop.example.com/products' },
      { name: product.name, url: `https://myshop.example.com/products/${product.id}` }
    ]);
  }

  addToCart() {
    console.log('Add to cart:', this.product.id);
  }

  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
    this.structuredData.removeAll();
  }
}
```

---

## 5. Sitemap Generation

```typescript
// scripts/generate-sitemap.ts
import { writeFileSync } from 'fs';

interface SitemapEntry {
  url: string;
  lastmod?: string;
  changefreq?: 'always' | 'hourly' | 'daily' | 'weekly' | 'monthly' | 'yearly' | 'never';
  priority?: number;
}

function generateSitemap(entries: SitemapEntry[]): string {
  const urls = entries.map(entry => `
  <url>
    <loc>${entry.url}</loc>
    ${entry.lastmod ? `<lastmod>${entry.lastmod}</lastmod>` : ''}
    ${entry.changefreq ? `<changefreq>${entry.changefreq}</changefreq>` : ''}
    ${entry.priority !== undefined ? `<priority>${entry.priority}</priority>` : ''}
  </url>`).join('');

  return `<?xml version="1.0" encoding="UTF-8"?>
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
${urls}
</urlset>`;
}

const baseUrl = 'https://myshop.example.com';
const today = new Date().toISOString().split('T')[0];

const staticPages: SitemapEntry[] = [
  { url: `${baseUrl}/`, lastmod: today, changefreq: 'daily', priority: 1.0 },
  { url: `${baseUrl}/products`, lastmod: today, changefreq: 'daily', priority: 0.9 },
  { url: `${baseUrl}/about`, lastmod: today, changefreq: 'monthly', priority: 0.5 },
  { url: `${baseUrl}/contact`, lastmod: today, changefreq: 'monthly', priority: 0.5 }
];

const sitemap = generateSitemap(staticPages);
writeFileSync('dist/browser/sitemap.xml', sitemap);
console.log('Sitemap generated!');
```

---

## สรุป

SEO ที่สมบูรณ์ใน Angular ต้องมี:

| องค์ประกอบ | ความสำคัญ |
|-----------|----------|
| Title Tags | สูงมาก |
| Meta Description | สูง |
| Open Graph / Twitter Cards | กลาง (สำหรับ social sharing) |
| Schema.org | กลาง (rich snippets) |
| Canonical URL | สูง (ป้องกัน duplicate content) |
| Sitemap | กลาง |
| SSR/Prerender | สูงมาก (สำหรับ SPA) |

Angular SSR จำเป็นมากสำหรับ SEO เพราะ search engines อาจไม่ execute JavaScript
