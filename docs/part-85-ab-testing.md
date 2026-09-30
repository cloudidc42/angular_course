# Part 85: A/B Testing ใน Angular

## A/B Testing คืออะไร

A/B Testing คือการทดสอบ 2 version ของ feature พร้อมกัน เพื่อดูว่าแบบไหน performance ดีกว่า

---

## 1. A/B Testing Service

```typescript
// core/ab-testing/ab-test.service.ts
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable, BehaviorSubject } from 'rxjs';
import { tap } from 'rxjs/operators';

export interface Experiment {
  id: string;
  name: string;
  variants: Variant[];
  status: 'draft' | 'running' | 'completed';
  startDate: string;
  endDate?: string;
  goal: string;
  trafficAllocation: number;  // % of users in experiment
}

export interface Variant {
  id: string;
  name: string;
  weight: number;  // 0-100, must sum to 100
  config?: Record<string, any>;
}

export interface UserAssignment {
  experimentId: string;
  variantId: string;
  userId: string;
  assignedAt: Date;
}

@Injectable({ providedIn: 'root' })
export class ABTestService {
  private experiments: Map<string, Experiment> = new Map();
  private assignments: Map<string, UserAssignment> = new Map();
  private userId = '';

  constructor(private http: HttpClient) {}

  initialize(userId: string): Observable<Experiment[]> {
    this.userId = userId;
    
    return this.http.get<Experiment[]>('/api/experiments').pipe(
      tap(experiments => {
        experiments.forEach(exp => {
          this.experiments.set(exp.id, exp);
        });
        
        // โหลด assignment จาก localStorage
        this.loadAssignments();
      })
    );
  }

  // รับ variant สำหรับ experiment
  getVariant(experimentId: string): Variant | null {
    const experiment = this.experiments.get(experimentId);
    if (!experiment || experiment.status !== 'running') return null;

    // ตรวจสอบว่า user อยู่ใน experiment ไหม
    if (!this.isUserInExperiment(experimentId, experiment.trafficAllocation)) {
      return null;
    }

    // ดึง assignment ที่มีอยู่แล้ว (sticky assignment)
    const existingAssignment = this.assignments.get(experimentId);
    if (existingAssignment) {
      return experiment.variants.find(v => v.id === existingAssignment.variantId) || null;
    }

    // สร้าง assignment ใหม่
    const variant = this.assignVariant(experiment);
    
    if (variant) {
      const assignment: UserAssignment = {
        experimentId,
        variantId: variant.id,
        userId: this.userId,
        assignedAt: new Date()
      };
      
      this.assignments.set(experimentId, assignment);
      this.saveAssignments();
      
      // Track assignment
      this.trackExposure(experimentId, variant.id);
    }

    return variant;
  }

  // ตรวจสอบว่า user อยู่ใน variant ไหน
  isInVariant(experimentId: string, variantId: string): boolean {
    const variant = this.getVariant(experimentId);
    return variant?.id === variantId;
  }

  // Track conversion event
  trackConversion(experimentId: string, goalName: string, value?: number): void {
    const assignment = this.assignments.get(experimentId);
    if (!assignment) return;

    const event = {
      experimentId,
      variantId: assignment.variantId,
      userId: this.userId,
      goal: goalName,
      value,
      timestamp: new Date()
    };

    // ส่ง event ไป analytics
    this.http.post('/api/experiments/conversion', event).subscribe();
    
    // GA4
    if (typeof gtag !== 'undefined') {
      gtag('event', 'experiment_conversion', {
        experiment_id: experimentId,
        variant_id: assignment.variantId,
        goal_name: goalName,
        value
      });
    }
  }

  private isUserInExperiment(experimentId: string, trafficAllocation: number): boolean {
    const hash = this.hashString(`${experimentId}-${this.userId}`);
    return (hash % 100) < trafficAllocation;
  }

  private assignVariant(experiment: Experiment): Variant | null {
    if (!experiment.variants.length) return null;
    
    const hash = this.hashString(`${experiment.id}-${this.userId}-variant`);
    const position = hash % 100;
    
    let accumulated = 0;
    for (const variant of experiment.variants) {
      accumulated += variant.weight;
      if (position < accumulated) return variant;
    }
    
    return experiment.variants[experiment.variants.length - 1];
  }

  private trackExposure(experimentId: string, variantId: string): void {
    this.http.post('/api/experiments/exposure', {
      experimentId,
      variantId,
      userId: this.userId,
      timestamp: new Date()
    }).subscribe();

    if (typeof gtag !== 'undefined') {
      gtag('event', 'experiment_impression', {
        experiment_id: experimentId,
        variant_id: variantId
      });
    }
  }

  private hashString(str: string): number {
    let hash = 0;
    for (let i = 0; i < str.length; i++) {
      hash = ((hash << 5) - hash) + str.charCodeAt(i);
      hash = hash & hash;
    }
    return Math.abs(hash);
  }

  private saveAssignments(): void {
    try {
      const data: Record<string, UserAssignment> = {};
      this.assignments.forEach((v, k) => { data[k] = v; });
      localStorage.setItem('ab_assignments', JSON.stringify(data));
    } catch {}
  }

  private loadAssignments(): void {
    try {
      const stored = localStorage.getItem('ab_assignments');
      if (stored) {
        const data = JSON.parse(stored) as Record<string, UserAssignment>;
        Object.entries(data).forEach(([k, v]) => {
          this.assignments.set(k, v);
        });
      }
    } catch {}
  }
}

declare const gtag: Function;
```

---

## 2. A/B Test Directive

```typescript
// core/ab-testing/ab-test.directive.ts
import { Directive, Input, TemplateRef, ViewContainerRef, OnInit } from '@angular/core';
import { ABTestService } from './ab-test.service';

@Directive({ selector: '[abTest]' })
export class ABTestDirective implements OnInit {
  @Input('abTest') experimentId = '';
  @Input('abTestVariant') variantId = '';
  @Input('abTestElse') elseTemplate?: TemplateRef<any>;

  constructor(
    private template: TemplateRef<any>,
    private viewContainer: ViewContainerRef,
    private abTestService: ABTestService
  ) {}

  ngOnInit(): void {
    const inVariant = this.abTestService.isInVariant(this.experimentId, this.variantId);
    
    if (inVariant) {
      this.viewContainer.createEmbeddedView(this.template);
    } else if (this.elseTemplate) {
      this.viewContainer.createEmbeddedView(this.elseTemplate);
    }
  }
}
```

---

## 3. ตัวอย่างการทดสอบ CTA Button

```typescript
// features/home/hero/hero.component.ts
import { Component, OnInit } from '@angular/core';
import { ABTestService, Variant } from '../../../core/ab-testing/ab-test.service';

const HERO_CTA_EXPERIMENT = 'hero-cta-experiment';

@Component({
  selector: 'app-hero',
  template: `
    <section class="hero">
      <h1>{{ heroConfig.headline }}</h1>
      <p>{{ heroConfig.subheadline }}</p>
      
      <!-- A/B Test on CTA Button -->
      <button 
        class="cta-button"
        [class]="heroConfig.buttonClass"
        [style.background]="heroConfig.buttonColor"
        (click)="onCTAClick()"
      >
        {{ heroConfig.buttonText }}
      </button>
      
      <!-- Test บน layout ทั้งหมด -->
      <ng-container [ngSwitch]="currentVariantId">
        <app-hero-variant-a *ngSwitchCase="'variant-a'"></app-hero-variant-a>
        <app-hero-variant-b *ngSwitchCase="'variant-b'"></app-hero-variant-b>
        <app-hero-control *ngSwitchDefault></app-hero-control>
      </ng-container>
    </section>
  `
})
export class HeroComponent implements OnInit {
  currentVariantId = 'control';
  heroConfig = this.getControlConfig();

  constructor(private abTestService: ABTestService) {}

  ngOnInit(): void {
    const variant = this.abTestService.getVariant(HERO_CTA_EXPERIMENT);
    
    if (variant) {
      this.currentVariantId = variant.id;
      this.heroConfig = this.getVariantConfig(variant);
    }
  }

  onCTAClick(): void {
    this.abTestService.trackConversion(HERO_CTA_EXPERIMENT, 'cta_click');
    // Navigate to signup...
  }

  private getControlConfig() {
    return {
      headline: 'ยินดีต้อนรับสู่แพลตฟอร์มของเรา',
      subheadline: 'เริ่มต้นใช้งานฟรีวันนี้',
      buttonText: 'ลงทะเบียนฟรี',
      buttonClass: 'btn-primary',
      buttonColor: '#2196f3'
    };
  }

  private getVariantConfig(variant: Variant) {
    const configs: Record<string, any> = {
      'variant-a': {
        headline: 'เพิ่มประสิทธิภาพธุรกิจของคุณ',
        subheadline: 'ประหยัดเวลา 5 ชั่วโมงต่อสัปดาห์',
        buttonText: 'เริ่มทดลองใช้ 14 วัน',
        buttonClass: 'btn-success',
        buttonColor: '#4caf50'
      },
      'variant-b': {
        headline: 'สมาร์ทเวิร์ก เพื่อทีมยุคใหม่',
        subheadline: 'ใช้งานได้ทันที ไม่ต้องติดตั้ง',
        buttonText: '🚀 เริ่มต้นเลย',
        buttonClass: 'btn-orange',
        buttonColor: '#ff5722'
      }
    };

    return configs[variant.id] || this.getControlConfig();
  }
}
```

---

## 4. Results Dashboard

```typescript
// admin/ab-test-dashboard/ab-results.component.ts
import { Component, OnInit } from '@angular/core';
import { HttpClient } from '@angular/common/http';

interface ExperimentResult {
  experimentId: string;
  experimentName: string;
  variants: {
    id: string;
    name: string;
    exposures: number;
    conversions: number;
    conversionRate: number;
    significance?: number;
    lift?: number;
    isWinner?: boolean;
  }[];
  status: string;
  startDate: string;
}

@Component({
  selector: 'app-ab-results',
  template: `
    <div class="results-dashboard">
      <h2>ผลลัพธ์ A/B Testing</h2>
      
      <div *ngFor="let exp of experiments" class="experiment-card">
        <div class="exp-header">
          <h3>{{ exp.experimentName }}</h3>
          <span class="status" [class]="exp.status">{{ exp.status }}</span>
        </div>
        
        <table class="results-table">
          <thead>
            <tr>
              <th>Variant</th>
              <th>Exposures</th>
              <th>Conversions</th>
              <th>Rate</th>
              <th>Lift</th>
              <th>Significance</th>
            </tr>
          </thead>
          <tbody>
            <tr 
              *ngFor="let v of exp.variants"
              [class.winner]="v.isWinner"
            >
              <td>
                {{ v.name }}
                <span *ngIf="v.isWinner" class="winner-badge">🏆 Winner</span>
              </td>
              <td>{{ v.exposures | number }}</td>
              <td>{{ v.conversions | number }}</td>
              <td>{{ v.conversionRate | percent:'1.2-2' }}</td>
              <td [class.positive]="v.lift && v.lift > 0" [class.negative]="v.lift && v.lift < 0">
                {{ v.lift ? (v.lift | number:'+1.1-1') + '%' : '-' }}
              </td>
              <td>
                <span [class]="getSignificanceClass(v.significance)">
                  {{ v.significance ? v.significance + '%' : 'N/A' }}
                </span>
              </td>
            </tr>
          </tbody>
        </table>
        
        <!-- Recommendation -->
        <div class="recommendation" *ngIf="getRecommendation(exp) as rec">
          <strong>คำแนะนำ:</strong> {{ rec }}
        </div>
      </div>
    </div>
  `,
  styles: [`
    .experiment-card { background: white; border-radius: 8px; padding: 20px; margin-bottom: 24px; box-shadow: 0 2px 8px rgba(0,0,0,0.1); }
    .exp-header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 16px; }
    .status { padding: 4px 12px; border-radius: 12px; font-size: 13px; }
    .status.running { background: #e8f5e9; color: #2e7d32; }
    .status.completed { background: #e3f2fd; color: #1565c0; }
    .results-table { width: 100%; border-collapse: collapse; }
    .results-table th, .results-table td { padding: 10px; text-align: left; border-bottom: 1px solid #eee; }
    .results-table th { background: #f5f5f5; font-weight: 600; }
    .winner { background: #f9fff9; }
    .winner-badge { background: #ffd700; padding: 2px 8px; border-radius: 4px; font-size: 12px; margin-left: 8px; }
    .positive { color: #2e7d32; font-weight: bold; }
    .negative { color: #c62828; font-weight: bold; }
    .sig-high { color: #2e7d32; font-weight: bold; }
    .sig-medium { color: #f57f17; }
    .sig-low { color: #999; }
    .recommendation { margin-top: 16px; padding: 12px; background: #e3f2fd; border-radius: 6px; font-size: 14px; }
  `]
})
export class ABResultsComponent implements OnInit {
  experiments: ExperimentResult[] = [];

  constructor(private http: HttpClient) {}

  ngOnInit(): void {
    this.http.get<ExperimentResult[]>('/api/experiments/results')
      .subscribe(results => this.experiments = results);
  }

  getSignificanceClass(significance?: number): string {
    if (!significance) return 'sig-low';
    if (significance >= 95) return 'sig-high';
    if (significance >= 80) return 'sig-medium';
    return 'sig-low';
  }

  getRecommendation(exp: ExperimentResult): string {
    const winner = exp.variants.find(v => v.isWinner);
    if (!winner) {
      const maxConvRate = Math.max(...exp.variants.map(v => v.conversionRate));
      const leading = exp.variants.find(v => v.conversionRate === maxConvRate);
      
      if (!leading?.significance || leading.significance < 95) {
        return 'ยังต้องการข้อมูลเพิ่มเติมเพื่อสรุปผล';
      }
    }
    
    return winner 
      ? `Deploy "${winner.name}" เนื่องจาก conversion rate สูงกว่า ${winner.lift?.toFixed(1)}%`
      : 'ทดลองต่อไปจนกว่าจะได้ statistical significance 95%+';
  }
}
```

---

## สรุป

| ขั้นตอน | รายละเอียด |
|---------|-----------|
| Define Experiment | กำหนด variants และ weights |
| Assign Users | Hash-based, sticky assignment |
| Track Exposure | บันทึกว่า user เห็น variant ไหน |
| Track Conversion | บันทึก goal completions |
| Analyze Results | Statistical significance |
| Ship Winner | Deploy winning variant |

### Statistical Significance

ต้องการ:
- **ตัวอย่าง**: อย่างน้อย 1,000+ per variant
- **Significance**: 95% confidence level
- **Duration**: อย่างน้อย 2 สัปดาห์ เพื่อ avoid day-of-week effects
