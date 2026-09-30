# Part 79: Performance Profiling ใน Angular

## ทำไมต้อง Profile

การ profile ช่วยหาจุดที่ทำให้แอปช้า เพื่อ optimize อย่างถูกจุด

---

## 1. Angular DevTools

### ติดตั้ง

ติดตั้ง **Angular DevTools** extension จาก Chrome Web Store

### วิธีใช้

1. เปิด Chrome DevTools (F12)
2. คลิก tab "Angular"
3. ดู Component Tree
4. เปิด Profiler → Record → Interact → Stop

### อ่านผล Profiler

```
Component Tree Profiling:
- AppComponent: 2ms
  - HeaderComponent: 0.5ms
  - ProductListComponent: 45ms  ← ช้า!
    - ProductCardComponent x100: 40ms
```

---

## 2. Change Detection Profiling

```typescript
// app/services/change-detection-tracker.service.ts
import { Injectable, NgZone, ChangeDetectorRef, ApplicationRef } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class CDTrackerService {
  private checkCount = 0;
  private startTime = 0;

  constructor(private appRef: ApplicationRef) {}

  startTracking(): void {
    this.checkCount = 0;
    this.startTime = performance.now();
    
    // Override tick method
    const originalTick = this.appRef.tick.bind(this.appRef);
    this.appRef.tick = () => {
      this.checkCount++;
      const tickStart = performance.now();
      originalTick();
      const tickTime = performance.now() - tickStart;
      
      if (tickTime > 16) {
        console.warn(`⚠️ Slow tick: ${tickTime.toFixed(2)}ms`);
      }
    };
  }

  getStats(): { checks: number; duration: number; rate: number } {
    const duration = performance.now() - this.startTime;
    return {
      checks: this.checkCount,
      duration: Math.round(duration),
      rate: Math.round(this.checkCount / (duration / 1000))
    };
  }
}
```

---

## 3. Component Performance Decorator

```typescript
// app/decorators/performance.decorator.ts

export function TrackPerformance(label?: string) {
  return function(target: any, propertyKey: string, descriptor: PropertyDescriptor) {
    const originalMethod = descriptor.value;

    descriptor.value = function(...args: any[]) {
      const name = label || `${target.constructor.name}.${propertyKey}`;
      performance.mark(`${name}-start`);
      
      const result = originalMethod.apply(this, args);
      
      if (result instanceof Promise) {
        return result.finally(() => {
          performance.mark(`${name}-end`);
          performance.measure(name, `${name}-start`, `${name}-end`);
          const entries = performance.getEntriesByName(name);
          const duration = entries[entries.length - 1]?.duration || 0;
          
          if (duration > 100) {
            console.warn(`⚠️ Slow method: ${name} took ${duration.toFixed(2)}ms`);
          }
        });
      }
      
      performance.mark(`${name}-end`);
      performance.measure(name, `${name}-start`, `${name}-end`);
      
      const entries = performance.getEntriesByName(name);
      const duration = entries[entries.length - 1]?.duration || 0;
      if (duration > 50) {
        console.warn(`⚠️ Slow method: ${name} took ${duration.toFixed(2)}ms`);
      }
      
      return result;
    };

    return descriptor;
  };
}

// ใช้งาน
// @Component(...)
// export class MyComponent {
//   @TrackPerformance('heavy-calculation')
//   processData(data: any[]): any[] { ... }
// }
```

---

## 4. Lighthouse Audit Automation

```typescript
// scripts/lighthouse-audit.ts
import * as lighthouse from 'lighthouse';
import * as chromeLauncher from 'chrome-launcher';
import * as fs from 'fs';

interface LighthouseResult {
  categories: {
    performance: { score: number };
    accessibility: { score: number };
    'best-practices': { score: number };
    seo: { score: number };
  };
  audits: Record<string, {
    id: string;
    title: string;
    score: number | null;
    numericValue?: number;
    displayValue?: string;
  }>;
}

async function runAudit(url: string): Promise<void> {
  const chrome = await chromeLauncher.launch({ chromeFlags: ['--headless'] });
  
  const options = {
    logLevel: 'info' as const,
    output: 'html' as const,
    onlyCategories: ['performance', 'accessibility', 'best-practices', 'seo'],
    port: chrome.port,
    throttlingMethod: 'simulate' as const,
    formFactor: 'desktop' as const
  };

  const runnerResult = await lighthouse(url, options);
  
  if (runnerResult) {
    const { lhr, report } = runnerResult;
    
    // บันทึก HTML report
    fs.writeFileSync('lighthouse-report.html', report as string);
    
    // สรุปผล
    console.log('\n📊 Lighthouse Results:');
    console.log('-------------------');
    
    const scores = lhr.categories;
    const performance = Math.round((scores.performance?.score || 0) * 100);
    const accessibility = Math.round((scores.accessibility?.score || 0) * 100);
    const bestPractices = Math.round((scores['best-practices']?.score || 0) * 100);
    const seo = Math.round((scores.seo?.score || 0) * 100);
    
    console.log(`Performance:    ${getScoreEmoji(performance)} ${performance}`);
    console.log(`Accessibility:  ${getScoreEmoji(accessibility)} ${accessibility}`);
    console.log(`Best Practices: ${getScoreEmoji(bestPractices)} ${bestPractices}`);
    console.log(`SEO:            ${getScoreEmoji(seo)} ${seo}`);
    
    // Core Web Vitals
    console.log('\n📈 Core Web Vitals:');
    console.log(`LCP: ${lhr.audits['largest-contentful-paint']?.displayValue}`);
    console.log(`FID: ${lhr.audits['max-potential-fid']?.displayValue}`);
    console.log(`CLS: ${lhr.audits['cumulative-layout-shift']?.displayValue}`);
    console.log(`FCP: ${lhr.audits['first-contentful-paint']?.displayValue}`);
    console.log(`TTI: ${lhr.audits['interactive']?.displayValue}`);
    
    // Check thresholds
    const failed = [];
    if (performance < 90) failed.push('Performance');
    if (accessibility < 90) failed.push('Accessibility');
    
    if (failed.length > 0) {
      console.error(`\n❌ Failed thresholds: ${failed.join(', ')}`);
      process.exit(1);
    } else {
      console.log('\n✅ All thresholds passed!');
    }
  }

  await chrome.kill();
}

function getScoreEmoji(score: number): string {
  if (score >= 90) return '🟢';
  if (score >= 50) return '🟡';
  return '🔴';
}

runAudit('http://localhost:4200');
```

---

## 5. Bundle Analyzer

```bash
# ติดตั้ง
npm install -D webpack-bundle-analyzer

# Build พร้อม stats
ng build --stats-json

# วิเคราะห์
npx webpack-bundle-analyzer dist/my-app/stats.json
```

### Custom Bundle Analysis Script

```typescript
// scripts/analyze-bundle.ts
import * as fs from 'fs';
import * as path from 'path';

interface BundleStats {
  assets: {
    name: string;
    size: number;
  }[];
  modules: {
    name: string;
    size: number;
    modules?: any[];
  }[];
}

function analyzeBundleSize(statsPath: string): void {
  const stats: BundleStats = JSON.parse(
    fs.readFileSync(statsPath, 'utf-8')
  );

  console.log('\n📦 Bundle Analysis Report');
  console.log('=========================\n');

  // Assets
  console.log('📁 Assets:');
  const assets = stats.assets
    .filter(a => a.name.endsWith('.js') || a.name.endsWith('.css'))
    .sort((a, b) => b.size - a.size);
  
  let totalSize = 0;
  assets.forEach(asset => {
    const size = asset.size;
    totalSize += size;
    const sizeKB = (size / 1024).toFixed(1);
    const emoji = size > 500000 ? '🔴' : size > 200000 ? '🟡' : '🟢';
    console.log(`  ${emoji} ${asset.name}: ${sizeKB}KB`);
  });
  
  console.log(`\n  Total: ${(totalSize / 1024 / 1024).toFixed(2)}MB`);
  
  // Large modules
  console.log('\n📦 Largest modules:');
  const modules = stats.modules
    ?.sort((a, b) => b.size - a.size)
    .slice(0, 10) || [];
  
  modules.forEach(mod => {
    const sizeKB = (mod.size / 1024).toFixed(1);
    const name = mod.name.replace('./node_modules/', '');
    console.log(`  ${name}: ${sizeKB}KB`);
  });
}

analyzeBundleSize('dist/my-app/stats.json');
```

---

## 6. Performance Monitoring Component

```typescript
// app/components/perf-monitor/perf-monitor.component.ts
import { Component, OnInit, OnDestroy } from '@angular/core';
import { interval, Subscription } from 'rxjs';

interface PerfMetric {
  name: string;
  value: number;
  unit: string;
  status: 'good' | 'warning' | 'poor';
}

@Component({
  selector: 'app-perf-monitor',
  template: `
    <div class="perf-monitor" *ngIf="isVisible">
      <div class="perf-header">
        <span>⚡ Performance</span>
        <button (click)="toggle()">×</button>
      </div>
      
      <div class="metrics">
        <div 
          *ngFor="let metric of metrics"
          class="metric"
          [class]="metric.status"
        >
          <span class="metric-name">{{ metric.name }}</span>
          <span class="metric-value">
            {{ metric.value | number:'1.0-1' }}{{ metric.unit }}
          </span>
        </div>
      </div>
    </div>
    
    <button class="perf-toggle" (click)="toggle()" *ngIf="!isVisible">⚡</button>
  `,
  styles: [`
    .perf-monitor {
      position: fixed; bottom: 16px; left: 16px;
      background: rgba(0,0,0,0.85); color: white;
      border-radius: 8px; padding: 12px;
      font-family: monospace; font-size: 12px;
      z-index: 9999; min-width: 180px;
    }
    .perf-header { display: flex; justify-content: space-between; margin-bottom: 8px; font-weight: bold; }
    .perf-header button { background: none; border: none; color: white; cursor: pointer; }
    .metric { display: flex; justify-content: space-between; padding: 2px 0; }
    .metric.good .metric-value { color: #4caf50; }
    .metric.warning .metric-value { color: #ff9800; }
    .metric.poor .metric-value { color: #f44336; }
    .perf-toggle {
      position: fixed; bottom: 16px; left: 16px;
      background: rgba(0,0,0,0.7); color: white;
      border: none; border-radius: 50%;
      width: 40px; height: 40px;
      cursor: pointer; font-size: 18px;
      z-index: 9999;
    }
  `]
})
export class PerfMonitorComponent implements OnInit, OnDestroy {
  metrics: PerfMetric[] = [];
  isVisible = true;
  private subscription!: Subscription;
  private frameCount = 0;
  private lastFrameTime = performance.now();
  private rafId!: number;

  ngOnInit(): void {
    this.startFPSCounter();
    this.subscription = interval(1000).subscribe(() => {
      this.updateMetrics();
    });
  }

  private startFPSCounter(): void {
    const countFrame = () => {
      this.frameCount++;
      this.rafId = requestAnimationFrame(countFrame);
    };
    this.rafId = requestAnimationFrame(countFrame);
  }

  private updateMetrics(): void {
    const now = performance.now();
    const elapsed = now - this.lastFrameTime;
    const fps = Math.round(this.frameCount / (elapsed / 1000));
    
    this.frameCount = 0;
    this.lastFrameTime = now;

    const memory = (performance as any).memory;
    const usedMB = memory ? memory.usedJSHeapSize / 1024 / 1024 : 0;
    const totalMB = memory ? memory.totalJSHeapSize / 1024 / 1024 : 0;

    this.metrics = [
      {
        name: 'FPS',
        value: fps,
        unit: '',
        status: fps >= 55 ? 'good' : fps >= 30 ? 'warning' : 'poor'
      },
      {
        name: 'Memory',
        value: usedMB,
        unit: 'MB',
        status: usedMB < 50 ? 'good' : usedMB < 100 ? 'warning' : 'poor'
      },
      {
        name: 'Heap Total',
        value: totalMB,
        unit: 'MB',
        status: 'good'
      }
    ];
  }

  toggle(): void {
    this.isVisible = !this.isVisible;
  }

  ngOnDestroy(): void {
    this.subscription?.unsubscribe();
    cancelAnimationFrame(this.rafId);
  }
}
```

---

## สรุป

| Tool | วัตถุประสงค์ |
|------|-------------|
| Angular DevTools | Component tree + Change detection |
| Lighthouse | Core Web Vitals |
| Bundle Analyzer | ขนาด bundle |
| Performance API | Custom measurements |
| Chrome DevTools | Memory, Network, FPS |

### Core Web Vitals Targets

- **LCP** (Largest Contentful Paint): < 2.5s
- **FID** (First Input Delay): < 100ms
- **CLS** (Cumulative Layout Shift): < 0.1
- **FCP** (First Contentful Paint): < 1.8s
- **TTI** (Time to Interactive): < 3.8s
