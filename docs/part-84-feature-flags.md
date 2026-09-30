# Part 84: Feature Flags ใน Angular

## Feature Flags คืออะไร

Feature Flags (Feature Toggles) ช่วยให้เราเปิด/ปิด features โดยไม่ต้อง deploy ใหม่ ใช้สำหรับ A/B Testing, Gradual Rollouts, Kill Switch

---

## 1. Feature Flag Service

```typescript
// core/feature-flags/feature-flag.service.ts
import { Injectable, Inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { BehaviorSubject, Observable } from 'rxjs';
import { map, tap } from 'rxjs/operators';

export interface FeatureFlag {
  key: string;
  enabled: boolean;
  rolloutPercentage?: number;  // 0-100
  allowedUsers?: string[];
  allowedRoles?: string[];
  expiresAt?: string;
  metadata?: Record<string, any>;
}

export interface FlagConfig {
  [key: string]: FeatureFlag;
}

@Injectable({ providedIn: 'root' })
export class FeatureFlagService {
  private flags$ = new BehaviorSubject<FlagConfig>({});
  private userId: string = '';
  private userRoles: string[] = [];

  constructor(private http: HttpClient) {}

  // โหลด flags จาก server
  initialize(userId: string, roles: string[]): Observable<FlagConfig> {
    this.userId = userId;
    this.userRoles = roles;

    return this.http.get<FlagConfig>('/api/feature-flags').pipe(
      tap(flags => this.flags$.next(flags))
    );
  }

  // ตรวจสอบว่า feature เปิดอยู่ไหม
  isEnabled(flagKey: string): boolean {
    const flag = this.flags$.value[flagKey];
    if (!flag) return false;
    if (!flag.enabled) return false;

    // ตรวจสอบ expiry
    if (flag.expiresAt && new Date(flag.expiresAt) < new Date()) return false;

    // ตรวจสอบ allowed users
    if (flag.allowedUsers?.length) {
      if (flag.allowedUsers.includes(this.userId)) return true;
      return false;
    }

    // ตรวจสอบ allowed roles
    if (flag.allowedRoles?.length) {
      if (this.userRoles.some(role => flag.allowedRoles!.includes(role))) return true;
      return false;
    }

    // Percentage rollout
    if (flag.rolloutPercentage !== undefined) {
      return this.isInRollout(flagKey, flag.rolloutPercentage);
    }

    return true;
  }

  // Observable version
  isEnabled$(flagKey: string): Observable<boolean> {
    return this.flags$.pipe(
      map(() => this.isEnabled(flagKey))
    );
  }

  // Consistent percentage rollout based on userId
  private isInRollout(flagKey: string, percentage: number): boolean {
    const hash = this.hashString(`${flagKey}-${this.userId}`);
    return (hash % 100) < percentage;
  }

  private hashString(str: string): number {
    let hash = 0;
    for (let i = 0; i < str.length; i++) {
      hash = ((hash << 5) - hash) + str.charCodeAt(i);
      hash = hash & hash;
    }
    return Math.abs(hash);
  }

  // Manual override (สำหรับ development)
  override(flagKey: string, enabled: boolean): void {
    const current = this.flags$.value;
    this.flags$.next({
      ...current,
      [flagKey]: { ...current[flagKey], key: flagKey, enabled }
    });
  }

  // Reset overrides
  resetOverrides(): void {
    this.initialize(this.userId, this.userRoles).subscribe();
  }

  getFlag(key: string): FeatureFlag | undefined {
    return this.flags$.value[key];
  }

  getAllFlags(): FlagConfig {
    return { ...this.flags$.value };
  }
}
```

---

## 2. Feature Flag Directive

```typescript
// core/feature-flags/feature-flag.directive.ts
import { 
  Directive, Input, TemplateRef, ViewContainerRef, 
  OnInit, OnDestroy 
} from '@angular/core';
import { Subject } from 'rxjs';
import { takeUntil } from 'rxjs/operators';
import { FeatureFlagService } from './feature-flag.service';

@Directive({ selector: '[featureFlag]' })
export class FeatureFlagDirective implements OnInit, OnDestroy {
  @Input('featureFlag') flagKey = '';
  @Input('featureFlagElse') elseTemplate?: TemplateRef<any>;
  
  private destroy$ = new Subject<void>();
  private hasView = false;

  constructor(
    private templateRef: TemplateRef<any>,
    private viewContainer: ViewContainerRef,
    private featureFlags: FeatureFlagService
  ) {}

  ngOnInit(): void {
    this.featureFlags.isEnabled$(this.flagKey)
      .pipe(takeUntil(this.destroy$))
      .subscribe(enabled => this.updateView(enabled));
  }

  private updateView(enabled: boolean): void {
    if (enabled && !this.hasView) {
      this.viewContainer.clear();
      this.viewContainer.createEmbeddedView(this.templateRef);
      this.hasView = true;
    } else if (!enabled) {
      this.viewContainer.clear();
      this.hasView = false;
      
      if (this.elseTemplate) {
        this.viewContainer.createEmbeddedView(this.elseTemplate);
      }
    }
  }

  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

```html
<!-- การใช้งาน -->
<ng-container *featureFlag="'new-checkout-flow'">
  <app-new-checkout></app-new-checkout>
</ng-container>

<!-- พร้อม else -->
<ng-template [featureFlag]="'dark-mode'" [featureFlagElse]="lightMode">
  <app-dark-theme></app-dark-theme>
</ng-template>

<ng-template #lightMode>
  <app-light-theme></app-light-theme>
</ng-template>
```

---

## 3. LaunchDarkly Integration

```typescript
// core/feature-flags/launch-darkly.service.ts
import { Injectable } from '@angular/core';
import * as LDClient from 'launchdarkly-js-client-sdk';
import { BehaviorSubject, Observable } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class LaunchDarklyService {
  private client!: LDClient.LDClient;
  private isReady$ = new BehaviorSubject<boolean>(false);

  async initialize(userId: string, userAttributes?: Record<string, any>): Promise<void> {
    const user: LDClient.LDUser = {
      key: userId,
      email: userAttributes?.['email'],
      name: userAttributes?.['name'],
      custom: userAttributes
    };

    this.client = LDClient.initialize(
      'YOUR_LAUNCHDARKLY_CLIENT_ID',
      user
    );

    await this.client.waitUntilReady();
    this.isReady$.next(true);
  }

  isEnabled(flagKey: string, defaultValue = false): boolean {
    if (!this.client) return defaultValue;
    return this.client.variation(flagKey, defaultValue) as boolean;
  }

  getVariation<T>(flagKey: string, defaultValue: T): T {
    if (!this.client) return defaultValue;
    return this.client.variation(flagKey, defaultValue) as T;
  }

  onChange(flagKey: string): Observable<boolean> {
    return new Observable(observer => {
      if (!this.client) {
        observer.next(false);
        return;
      }

      // Current value
      observer.next(this.isEnabled(flagKey));

      // Listen for changes
      this.client.on(`change:${flagKey}`, (value) => {
        observer.next(value as boolean);
      });
    });
  }

  async updateUser(userId: string, attributes?: Record<string, any>): Promise<void> {
    if (!this.client) return;
    await this.client.identify({ key: userId, ...attributes });
  }
}
```

---

## 4. Environment-based Flags

```typescript
// environments/feature-flags.ts
export const FeatureFlags = {
  // Stable features
  DARK_MODE: 'dark-mode',
  NEW_NAVIGATION: 'new-navigation',
  
  // Beta features
  AI_SEARCH: 'ai-search',
  VIDEO_CALL: 'video-call',
  
  // Experimental
  NEW_CHECKOUT: 'new-checkout-flow',
  CRYPTO_PAYMENT: 'crypto-payment'
} as const;

// environments/environment.ts
export const environment = {
  production: false,
  featureFlags: {
    [FeatureFlags.DARK_MODE]: true,
    [FeatureFlags.NEW_NAVIGATION]: true,
    [FeatureFlags.AI_SEARCH]: false,
    [FeatureFlags.VIDEO_CALL]: false,
    [FeatureFlags.NEW_CHECKOUT]: false,
    [FeatureFlags.CRYPTO_PAYMENT]: false
  }
};

// environments/environment.prod.ts
export const environment = {
  production: true,
  featureFlags: {
    [FeatureFlags.DARK_MODE]: true,
    [FeatureFlags.NEW_NAVIGATION]: true,
    [FeatureFlags.AI_SEARCH]: true,  // เปิด prod
    [FeatureFlags.VIDEO_CALL]: false,
    [FeatureFlags.NEW_CHECKOUT]: false,
    [FeatureFlags.CRYPTO_PAYMENT]: false
  }
};
```

---

## 5. Flag Management Dashboard

```typescript
// admin/feature-flag-dashboard/feature-flag-dashboard.component.ts
import { Component, OnInit } from '@angular/core';
import { FeatureFlagService, FeatureFlag } from '../../../core/feature-flags/feature-flag.service';

@Component({
  selector: 'app-flag-dashboard',
  template: `
    <div class="flag-dashboard">
      <h2>Feature Flags Management</h2>
      
      <div class="search-box">
        <input [(ngModel)]="search" placeholder="ค้นหา flag...">
      </div>

      <div class="flags-list">
        <div 
          *ngFor="let flag of filteredFlags" 
          class="flag-item"
          [class.enabled]="flag.enabled"
        >
          <div class="flag-info">
            <span class="flag-key">{{ flag.key }}</span>
            <div class="flag-details">
              <span *ngIf="flag.rolloutPercentage !== undefined">
                Rollout: {{ flag.rolloutPercentage }}%
              </span>
              <span *ngIf="flag.expiresAt">
                หมดอายุ: {{ flag.expiresAt | date }}
              </span>
            </div>
          </div>
          
          <div class="flag-controls">
            <label class="toggle">
              <input 
                type="checkbox" 
                [checked]="flag.enabled"
                (change)="toggleFlag(flag.key, $event)"
              >
              <span class="slider"></span>
            </label>
            <span class="status">{{ flag.enabled ? 'เปิด' : 'ปิด' }}</span>
          </div>
        </div>
      </div>

      <div class="actions">
        <button (click)="refreshFlags()" class="btn-secondary">รีเฟรช</button>
        <button (click)="resetAll()" class="btn-danger">Reset ทั้งหมด</button>
      </div>
    </div>
  `,
  styles: [`
    .flag-dashboard { padding: 24px; }
    .flags-list { display: flex; flex-direction: column; gap: 8px; margin: 16px 0; }
    .flag-item {
      display: flex;
      justify-content: space-between;
      align-items: center;
      padding: 12px 16px;
      background: white;
      border-radius: 8px;
      border-left: 4px solid #ddd;
      box-shadow: 0 1px 3px rgba(0,0,0,0.1);
    }
    .flag-item.enabled { border-left-color: #4caf50; }
    .flag-key { font-weight: bold; font-family: monospace; }
    .flag-details { font-size: 12px; color: #999; }
    .toggle { position: relative; display: inline-block; width: 44px; height: 24px; }
    .toggle input { opacity: 0; width: 0; height: 0; }
    .slider {
      position: absolute; cursor: pointer;
      inset: 0; background: #ccc; border-radius: 24px;
      transition: 0.4s;
    }
    .slider:before {
      content: ''; position: absolute;
      width: 16px; height: 16px; left: 4px; bottom: 4px;
      background: white; border-radius: 50%;
      transition: 0.4s;
    }
    input:checked + .slider { background: #4caf50; }
    input:checked + .slider:before { transform: translateX(20px); }
    .flag-controls { display: flex; align-items: center; gap: 8px; }
    .actions { display: flex; gap: 8px; justify-content: flex-end; }
    .btn-secondary { padding: 8px 16px; background: #eee; border: none; border-radius: 6px; cursor: pointer; }
    .btn-danger { padding: 8px 16px; background: #f44336; color: white; border: none; border-radius: 6px; cursor: pointer; }
  `]
})
export class FeatureFlagDashboardComponent implements OnInit {
  flags: FeatureFlag[] = [];
  search = '';

  constructor(private featureFlagService: FeatureFlagService) {}

  ngOnInit(): void {
    this.loadFlags();
  }

  loadFlags(): void {
    const allFlags = this.featureFlagService.getAllFlags();
    this.flags = Object.values(allFlags);
  }

  get filteredFlags(): FeatureFlag[] {
    if (!this.search) return this.flags;
    return this.flags.filter(f => 
      f.key.toLowerCase().includes(this.search.toLowerCase())
    );
  }

  toggleFlag(key: string, event: Event): void {
    const enabled = (event.target as HTMLInputElement).checked;
    this.featureFlagService.override(key, enabled);
    
    const flag = this.flags.find(f => f.key === key);
    if (flag) flag.enabled = enabled;
  }

  refreshFlags(): void {
    this.featureFlagService.resetOverrides();
    setTimeout(() => this.loadFlags(), 500);
  }

  resetAll(): void {
    if (confirm('ต้องการ reset flag ทั้งหมดหรือไม่?')) {
      this.featureFlagService.resetOverrides();
      setTimeout(() => this.loadFlags(), 500);
    }
  }
}
```

---

## สรุป

| เครื่องมือ | Use Case |
|-----------|----------|
| Environment flags | Simple on/off per environment |
| LaunchDarkly | Enterprise, Real-time updates |
| ConfigCat | Budget-friendly |
| Custom service | Full control |

### Best Practices

1. ใช้ constants สำหรับ flag keys
2. Document แต่ละ flag ว่าทำไมสร้าง
3. กำหนด expiry date
4. Clean up flags ที่ไม่ใช้แล้ว
5. Monitor flag evaluation ด้วย analytics
6. Test ทั้ง enabled และ disabled states
