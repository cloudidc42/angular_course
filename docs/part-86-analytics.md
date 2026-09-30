# Part 86: Analytics ใน Angular (Google Analytics 4, Mixpanel)

## Analytics คืออะไร

การติดตามพฤติกรรมผู้ใช้เพื่อทำความเข้าใจว่าใช้งานอย่างไร และปรับปรุง UX

---

## 1. Analytics Service

```typescript
// core/analytics/analytics.service.ts
import { Injectable } from '@angular/core';
import { Router, NavigationEnd } from '@angular/router';
import { filter } from 'rxjs/operators';

export interface TrackEventOptions {
  category?: string;
  label?: string;
  value?: number;
  nonInteraction?: boolean;
  [key: string]: any;
}

@Injectable({ providedIn: 'root' })
export class AnalyticsService {
  private userId: string | null = null;
  private sessionId: string;
  private providers: AnalyticsProvider[] = [];

  constructor(private router: Router) {
    this.sessionId = this.generateSessionId();
    this.setupPageTracking();
  }

  // ลงทะเบียน providers
  addProvider(provider: AnalyticsProvider): void {
    this.providers.push(provider);
  }

  // Track page view
  trackPageView(url?: string): void {
    const page = url || this.router.url;
    const title = document.title;

    this.providers.forEach(p => p.trackPageView(page, title));
  }

  // Track custom event
  trackEvent(eventName: string, options?: TrackEventOptions): void {
    const event = {
      ...options,
      sessionId: this.sessionId,
      timestamp: new Date().toISOString(),
      url: this.router.url
    };

    this.providers.forEach(p => p.trackEvent(eventName, event));
  }

  // Track e-commerce
  trackPurchase(transaction: {
    id: string;
    total: number;
    currency?: string;
    items: { id: string; name: string; price: number; quantity: number }[];
  }): void {
    this.providers.forEach(p => p.trackPurchase(transaction));
  }

  // Set user
  setUser(userId: string, traits?: Record<string, any>): void {
    this.userId = userId;
    this.providers.forEach(p => p.setUser(userId, traits));
  }

  // Clear user (logout)
  clearUser(): void {
    this.userId = null;
    this.providers.forEach(p => p.clearUser());
  }

  private setupPageTracking(): void {
    this.router.events.pipe(
      filter(event => event instanceof NavigationEnd)
    ).subscribe((event: any) => {
      this.trackPageView(event.urlAfterRedirects);
    });
  }

  private generateSessionId(): string {
    return `sess-${Date.now()}-${Math.random().toString(36).slice(2, 9)}`;
  }
}

// Abstract provider
export abstract class AnalyticsProvider {
  abstract trackPageView(url: string, title: string): void;
  abstract trackEvent(event: string, data: any): void;
  abstract trackPurchase(transaction: any): void;
  abstract setUser(userId: string, traits?: any): void;
  abstract clearUser(): void;
}
```

---

## 2. Google Analytics 4 Provider

```typescript
// core/analytics/providers/ga4.provider.ts
import { Injectable } from '@angular/core';
import { AnalyticsProvider } from '../analytics.service';

declare global {
  interface Window {
    dataLayer: any[];
    gtag: (...args: any[]) => void;
  }
}

@Injectable()
export class GA4Provider extends AnalyticsProvider {
  private measurementId: string;

  constructor(measurementId: string) {
    super();
    this.measurementId = measurementId;
    this.loadScript();
  }

  private loadScript(): void {
    // Load gtag.js
    const script = document.createElement('script');
    script.src = `https://www.googletagmanager.com/gtag/js?id=${this.measurementId}`;
    script.async = true;
    document.head.appendChild(script);

    // Initialize
    window.dataLayer = window.dataLayer || [];
    window.gtag = function() {
      window.dataLayer.push(arguments);
    };
    
    window.gtag('js', new Date());
    window.gtag('config', this.measurementId, {
      send_page_view: false  // จะ track เอง
    });
  }

  trackPageView(url: string, title: string): void {
    window.gtag('event', 'page_view', {
      page_location: window.location.origin + url,
      page_title: title,
      page_path: url
    });
  }

  trackEvent(eventName: string, data: any): void {
    const { category, label, value, ...params } = data;
    
    window.gtag('event', eventName, {
      event_category: category,
      event_label: label,
      value: value,
      ...params
    });
  }

  trackPurchase(transaction: any): void {
    window.gtag('event', 'purchase', {
      transaction_id: transaction.id,
      value: transaction.total,
      currency: transaction.currency || 'THB',
      items: transaction.items.map((item: any) => ({
        item_id: item.id,
        item_name: item.name,
        price: item.price,
        quantity: item.quantity
      }))
    });
  }

  // Enhanced Ecommerce
  trackAddToCart(item: any): void {
    window.gtag('event', 'add_to_cart', {
      currency: 'THB',
      value: item.price * item.quantity,
      items: [{
        item_id: item.id,
        item_name: item.name,
        price: item.price,
        quantity: item.quantity,
        item_category: item.category
      }]
    });
  }

  trackBeginCheckout(cart: any): void {
    window.gtag('event', 'begin_checkout', {
      currency: 'THB',
      value: cart.total,
      items: cart.items.map((item: any) => ({
        item_id: item.id,
        item_name: item.name,
        price: item.price,
        quantity: item.quantity
      }))
    });
  }

  setUser(userId: string, traits?: any): void {
    window.gtag('set', 'user_id', userId);
    window.gtag('set', 'user_properties', traits);
  }

  clearUser(): void {
    window.gtag('set', 'user_id', null);
  }
}
```

---

## 3. Mixpanel Provider

```typescript
// core/analytics/providers/mixpanel.provider.ts
import { Injectable } from '@angular/core';
import { AnalyticsProvider } from '../analytics.service';

declare const mixpanel: any;

@Injectable()
export class MixpanelProvider extends AnalyticsProvider {
  constructor(private token: string) {
    super();
    this.loadScript();
  }

  private loadScript(): void {
    // Mixpanel stub
    (function(c, a) {
      if ((window as any).mixpanel) return;
      const b = (window as any).mixpanel = [];
      const d = ['init', 'track', 'identify', 'people', 'set_config'];
      d.forEach(f => {
        b[f] = function() {
          b.push([f].concat(Array.from(arguments)));
        };
      });
      const e = c.createElement('script');
      e.type = 'text/javascript';
      e.async = true;
      e.src = 'https://cdn.mxpnl.com/libs/mixpanel-2-latest.min.js';
      const g = c.getElementsByTagName('script')[0];
      g.parentNode?.insertBefore(e, g);
    })(document, window);

    mixpanel.init(this.token);
  }

  trackPageView(url: string, title: string): void {
    mixpanel.track('Page View', {
      url,
      title,
      referrer: document.referrer
    });
  }

  trackEvent(eventName: string, data: any): void {
    mixpanel.track(eventName, data);
  }

  trackPurchase(transaction: any): void {
    mixpanel.track('Purchase Completed', {
      transaction_id: transaction.id,
      total: transaction.total,
      currency: transaction.currency || 'THB',
      items_count: transaction.items.length
    });

    // Revenue tracking
    mixpanel.people.track_charge(transaction.total);
  }

  setUser(userId: string, traits?: any): void {
    mixpanel.identify(userId);
    if (traits) {
      mixpanel.people.set({
        $name: traits.name,
        $email: traits.email,
        ...traits
      });
    }
  }

  clearUser(): void {
    mixpanel.reset();
  }
}
```

---

## 4. Custom Event Tracking Directive

```typescript
// core/analytics/directives/track-click.directive.ts
import { Directive, Input, HostListener } from '@angular/core';
import { AnalyticsService } from '../analytics.service';

@Directive({ selector: '[trackClick]' })
export class TrackClickDirective {
  @Input('trackClick') eventName = '';
  @Input() trackData: Record<string, any> = {};
  @Input() trackCategory = '';

  constructor(private analytics: AnalyticsService) {}

  @HostListener('click', ['$event'])
  onClick(event: MouseEvent): void {
    if (!this.eventName) return;
    
    this.analytics.trackEvent(this.eventName, {
      category: this.trackCategory,
      element: (event.target as HTMLElement).tagName,
      ...this.trackData
    });
  }
}

// ใช้งาน
// <button 
//   trackClick="add_to_cart"
//   trackCategory="product_page"
//   [trackData]="{ product_id: product.id }"
// >
//   เพิ่มในตะกร้า
// </button>
```

---

## 5. Funnel Tracking

```typescript
// core/analytics/funnel-tracker.service.ts
import { Injectable } from '@angular/core';
import { AnalyticsService } from './analytics.service';

export interface FunnelStep {
  name: string;
  completed: boolean;
  completedAt?: Date;
  data?: Record<string, any>;
}

@Injectable({ providedIn: 'root' })
export class FunnelTrackerService {
  private funnels = new Map<string, FunnelStep[]>();

  constructor(private analytics: AnalyticsService) {}

  initFunnel(funnelName: string, steps: string[]): void {
    this.funnels.set(funnelName, steps.map(name => ({
      name,
      completed: false
    })));
  }

  completeStep(funnelName: string, stepName: string, data?: Record<string, any>): void {
    const funnel = this.funnels.get(funnelName);
    if (!funnel) return;

    const step = funnel.find(s => s.name === stepName);
    if (!step) return;

    step.completed = true;
    step.completedAt = new Date();
    step.data = data;

    this.analytics.trackEvent('funnel_step_completed', {
      funnel_name: funnelName,
      step_name: stepName,
      step_index: funnel.indexOf(step),
      ...data
    });

    // ตรวจสอบว่า funnel complete ไหม
    if (funnel.every(s => s.completed)) {
      this.analytics.trackEvent('funnel_completed', {
        funnel_name: funnelName,
        total_steps: funnel.length
      });
    }
  }

  abandonFunnel(funnelName: string, reason?: string): void {
    const funnel = this.funnels.get(funnelName);
    if (!funnel) return;

    const completedSteps = funnel.filter(s => s.completed).length;
    const lastStep = funnel[completedSteps - 1]?.name;

    this.analytics.trackEvent('funnel_abandoned', {
      funnel_name: funnelName,
      completed_steps: completedSteps,
      total_steps: funnel.length,
      last_completed_step: lastStep,
      reason
    });

    this.funnels.delete(funnelName);
  }
}

// ใช้งานใน checkout
// ngOnInit(): void {
//   this.funnel.initFunnel('checkout', ['cart', 'address', 'payment', 'confirmation']);
//   this.funnel.completeStep('checkout', 'cart');
// }
```

---

## 6. Analytics Dashboard Component

```typescript
// admin/analytics/analytics-dashboard.component.ts
import { Component, OnInit } from '@angular/core';
import { HttpClient } from '@angular/common/http';

interface MetricCard {
  label: string;
  value: number;
  change: number;
  format: 'number' | 'currency' | 'percent';
}

@Component({
  selector: 'app-analytics-dashboard',
  template: `
    <div class="analytics-dashboard">
      <h2>Analytics Overview</h2>
      
      <!-- Metric Cards -->
      <div class="metrics-grid">
        <div *ngFor="let metric of metrics" class="metric-card">
          <div class="metric-label">{{ metric.label }}</div>
          <div class="metric-value">{{ formatMetric(metric) }}</div>
          <div class="metric-change" [class.positive]="metric.change > 0" [class.negative]="metric.change < 0">
            {{ metric.change > 0 ? '+' : '' }}{{ metric.change | number:'1.1-1' }}%
            <span>vs ปีที่แล้ว</span>
          </div>
        </div>
      </div>

      <!-- Top Events -->
      <div class="events-section">
        <h3>Top Events (7 วันที่ผ่านมา)</h3>
        <table>
          <thead>
            <tr>
              <th>Event</th>
              <th>Count</th>
              <th>Unique Users</th>
              <th>Trend</th>
            </tr>
          </thead>
          <tbody>
            <tr *ngFor="let event of topEvents">
              <td>{{ event.name }}</td>
              <td>{{ event.count | number }}</td>
              <td>{{ event.uniqueUsers | number }}</td>
              <td>
                <span [class]="event.trend > 0 ? 'up' : 'down'">
                  {{ event.trend > 0 ? '↑' : '↓' }} {{ event.trend | number:'1.1-1' }}%
                </span>
              </td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>
  `,
  styles: [`
    .analytics-dashboard { padding: 24px; }
    .metrics-grid { display: grid; grid-template-columns: repeat(4, 1fr); gap: 16px; margin: 24px 0; }
    .metric-card { background: white; padding: 20px; border-radius: 12px; box-shadow: 0 2px 8px rgba(0,0,0,0.08); }
    .metric-label { font-size: 13px; color: #666; margin-bottom: 8px; }
    .metric-value { font-size: 28px; font-weight: bold; color: #333; }
    .metric-change { font-size: 13px; margin-top: 8px; }
    .metric-change.positive { color: #2e7d32; }
    .metric-change.negative { color: #c62828; }
    .metric-change span { color: #999; }
    table { width: 100%; border-collapse: collapse; }
    th, td { padding: 10px; text-align: left; border-bottom: 1px solid #eee; }
    th { background: #f5f5f5; font-weight: 600; }
    .up { color: #2e7d32; }
    .down { color: #c62828; }
  `]
})
export class AnalyticsDashboardComponent implements OnInit {
  metrics: MetricCard[] = [];
  topEvents: any[] = [];

  constructor(private http: HttpClient) {}

  ngOnInit(): void {
    this.loadMetrics();
    this.loadTopEvents();
  }

  loadMetrics(): void {
    // Mock data - replace with actual API
    this.metrics = [
      { label: 'ผู้ใช้ทั้งหมด', value: 45823, change: 12.5, format: 'number' },
      { label: 'รายได้', value: 1250000, change: 8.3, format: 'currency' },
      { label: 'Conversion Rate', value: 3.2, change: 0.5, format: 'percent' },
      { label: 'ค่าเฉลี่ยต่อ Session', value: 185, change: -2.1, format: 'currency' }
    ];
  }

  loadTopEvents(): void {
    this.topEvents = [
      { name: 'page_view', count: 125043, uniqueUsers: 45823, trend: 5.2 },
      { name: 'add_to_cart', count: 8523, uniqueUsers: 6234, trend: 12.1 },
      { name: 'begin_checkout', count: 3421, uniqueUsers: 3100, trend: 8.5 },
      { name: 'purchase', count: 1456, uniqueUsers: 1456, trend: 3.2 },
      { name: 'product_view', count: 45231, uniqueUsers: 32100, trend: -1.5 }
    ];
  }

  formatMetric(metric: MetricCard): string {
    if (metric.format === 'currency') {
      return new Intl.NumberFormat('th-TH', { style: 'currency', currency: 'THB', maximumFractionDigits: 0 }).format(metric.value);
    }
    if (metric.format === 'percent') {
      return metric.value.toFixed(1) + '%';
    }
    return new Intl.NumberFormat('th-TH').format(metric.value);
  }
}
```

---

## สรุป

| เครื่องมือ | เหมาะกับ |
|-----------|---------|
| Google Analytics 4 | General analytics, free |
| Mixpanel | User behavior, funnels |
| Amplitude | Product analytics |
| Hotjar | Heatmaps, recordings |

### Events ที่ควร Track

1. Page views
2. Button clicks (CTA)
3. Form submissions
4. Search queries
5. Add to cart
6. Purchase/Conversion
7. Error events
8. Feature usage
