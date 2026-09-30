# Part 13 — Lifecycle Hooks

## เนื้อหาในบทนี้
1. Lifecycle Hooks คืออะไร?
2. ลำดับการทำงานของ Lifecycle
3. ngOnChanges
4. ngOnInit
5. ngDoCheck
6. ngAfterContentInit & ngAfterContentChecked
7. ngAfterViewInit & ngAfterViewChecked
8. ngOnDestroy
9. Lifecycle ใน Standalone Components
10. Workshop: Timer Component

---

## 1. Lifecycle Hooks คืออะไร?

Angular ให้เราสามารถ "hook" เข้าไปในช่วงต่างๆ ของชีวิต Component ได้ ตั้งแต่ถูกสร้างจนถูกทำลาย

```
Lifecycle ของ Component
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  constructor()          ← สร้าง instance
        │
  ngOnChanges()          ← Input properties เปลี่ยน (ครั้งแรก)
        │
  ngOnInit()             ← Initialize เสร็จ (ครั้งเดียว)
        │
  ngDoCheck()            ← การตรวจสอบเพิ่มเติม (ทุก cycle)
        │
  ngAfterContentInit()   ← ng-content ถูก project เข้ามา (ครั้งเดียว)
        │
  ngAfterContentChecked()← หลังตรวจสอบ content (ทุก cycle)
        │
  ngAfterViewInit()      ← View และ child views init เสร็จ (ครั้งเดียว)
        │
  ngAfterViewChecked()   ← หลังตรวจสอบ view (ทุก cycle)
        │
     [Input changes]
        │
  ngOnChanges()          ← ทุกครั้งที่ Input เปลี่ยน
  ngDoCheck()            ← ทุก change detection cycle
  ngAfterContentChecked()
  ngAfterViewChecked()
        │
  ngOnDestroy()          ← ก่อน Component ถูกทำลาย
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## 2. Interfaces สำหรับ Lifecycle Hooks

```typescript
import {
  OnChanges, OnInit, DoCheck,
  AfterContentInit, AfterContentChecked,
  AfterViewInit, AfterViewChecked,
  OnDestroy,
  SimpleChanges
} from '@angular/core';

@Component({...})
export class ExampleComponent implements
  OnChanges, OnInit, DoCheck,
  AfterContentInit, AfterContentChecked,
  AfterViewInit, AfterViewChecked,
  OnDestroy {

  ngOnChanges(changes: SimpleChanges): void {}
  ngOnInit(): void {}
  ngDoCheck(): void {}
  ngAfterContentInit(): void {}
  ngAfterContentChecked(): void {}
  ngAfterViewInit(): void {}
  ngAfterViewChecked(): void {}
  ngOnDestroy(): void {}
}
```

---

## 3. ngOnChanges

เรียกเมื่อ **@Input property เปลี่ยนค่า** — รับ `SimpleChanges` เป็น argument

```typescript
import { Component, Input, OnChanges, SimpleChanges } from '@angular/core';

@Component({
  selector: 'app-product-detail',
  standalone: true,
  template: `
    <div>
      <h2>{{ product?.name }}</h2>
      <p>ราคา: {{ product?.price }}</p>
      <p>อัปเดต: {{ updateCount }} ครั้ง</p>
    </div>
  `
})
export class ProductDetailComponent implements OnChanges {
  @Input() product: { name: string; price: number } | null = null;
  @Input() currency = 'THB';
  
  updateCount = 0;
  
  ngOnChanges(changes: SimpleChanges): void {
    console.log('Changes:', changes);
    
    // ดูว่า property ไหนเปลี่ยน
    if (changes['product']) {
      const change = changes['product'];
      console.log('Previous:', change.previousValue);
      console.log('Current:', change.currentValue);
      console.log('First change:', change.firstChange);
      
      if (!change.firstChange) {
        // ไม่ใช่ครั้งแรก
        this.updateCount++;
        console.log('Product updated!');
      }
    }
    
    if (changes['currency']) {
      console.log('Currency changed to:', changes['currency'].currentValue);
    }
  }
}
```

### SimpleChanges API

```typescript
// SimpleChanges คือ object ที่มี key เป็นชื่อ @Input
// และ value เป็น SimpleChange object
interface SimpleChanges {
  [propName: string]: SimpleChange;
}

interface SimpleChange {
  previousValue: any;   // ค่าเก่า
  currentValue: any;    // ค่าใหม่
  firstChange: boolean; // ครั้งแรกหรือเปล่า?
  isFirstChange(): boolean; // method ที่เรียกได้
}
```

---

## 4. ngOnInit

เรียก **ครั้งเดียว** หลังจาก Component ถูก initialize และ Input properties ถูก set แล้ว

```typescript
import { Component, Input, OnInit } from '@angular/core';
import { HttpClient } from '@angular/common/http';

@Component({
  selector: 'app-user-profile',
  standalone: true,
  template: `
    <div *ngIf="user">
      <h2>{{ user.name }}</h2>
      <p>{{ user.email }}</p>
    </div>
    <div *ngIf="isLoading">Loading...</div>
    <div *ngIf="error">{{ error }}</div>
  `
})
export class UserProfileComponent implements OnInit {
  @Input() userId!: number;
  
  user: any = null;
  isLoading = false;
  error: string | null = null;
  
  constructor(private http: HttpClient) {
    // ❌ ไม่ควรเรียก API ที่นี่
    // constructor ควรใช้สำหรับ Dependency Injection เท่านั้น
  }
  
  ngOnInit(): void {
    // ✅ เรียก API ที่นี่
    this.loadUser();
    
    // ✅ Subscribe observables ที่นี่
    // ✅ Initialize state ที่ต้องใช้ Input
    // ✅ ทำ complex initialization
  }
  
  private loadUser(): void {
    this.isLoading = true;
    this.http
      .get(`/api/users/${this.userId}`)
      .subscribe({
        next: (user) => {
          this.user = user;
          this.isLoading = false;
        },
        error: (err) => {
          this.error = 'ไม่สามารถโหลดข้อมูลได้';
          this.isLoading = false;
        }
      });
  }
}
```

### Constructor vs ngOnInit

```typescript
@Component({...})
export class MyComponent implements OnInit {
  @Input() config!: Config;
  
  constructor(
    private userService: UserService,
    private router: Router
  ) {
    // ✅ ใช้ constructor สำหรับ:
    // - Dependency Injection เท่านั้น
    // - Initialize ค่าคงที่ที่ไม่ขึ้นกับ Input
    
    // ❌ ไม่ควร:
    // - เรียก this.config (ยังไม่ถูก set)
    // - เรียก API
    // - Subscribe observables
  }
  
  ngOnInit(): void {
    // ✅ ใช้ ngOnInit สำหรับ:
    // - เรียก API
    // - ใช้ this.config (ถูก set แล้ว)
    // - Subscribe observables
    // - Complex initialization
    console.log('Config:', this.config); // ✅ มีค่าแล้ว
  }
}
```

---

## 5. ngDoCheck

เรียก **ทุก change detection cycle** — ใช้ระวัง มันถูกเรียกบ่อยมาก!

```typescript
import { Component, DoCheck, Input, KeyValueDiffers, KeyValueDiffer } from '@angular/core';

@Component({
  selector: 'app-object-monitor',
  standalone: true,
  template: `
    <div>
      <h3>Object Monitor</h3>
      <p>การเปลี่ยนแปลง: {{ changeLog.length }} รายการ</p>
      <ul>
        <li *ngFor="let log of changeLog">{{ log }}</li>
      </ul>
    </div>
  `
})
export class ObjectMonitorComponent implements DoCheck {
  @Input() data: { [key: string]: any } = {};
  
  private differ: KeyValueDiffer<string, any>;
  changeLog: string[] = [];
  
  constructor(private differs: KeyValueDiffers) {
    this.differ = this.differs.find({}).create();
  }
  
  ngDoCheck(): void {
    // ตรวจสอบการเปลี่ยนแปลงใน object
    const changes = this.differ.diff(this.data);
    
    if (changes) {
      changes.forEachChangedItem(item => {
        this.changeLog.push(
          `เปลี่ยน "${item.key}": ${item.previousValue} → ${item.currentValue}`
        );
      });
      
      changes.forEachAddedItem(item => {
        this.changeLog.push(`เพิ่ม "${item.key}": ${item.currentValue}`);
      });
      
      changes.forEachRemovedItem(item => {
        this.changeLog.push(`ลบ "${item.key}"`);
      });
    }
  }
}
```

---

## 6. AfterContent Hooks

เกี่ยวข้องกับ **Content Projection** (`ng-content`)

```typescript
import {
  Component, ContentChild, ContentChildren,
  AfterContentInit, AfterContentChecked,
  QueryList, ElementRef
} from '@angular/core';

// Child component ที่จะถูก project
@Component({
  selector: 'app-tab',
  standalone: true,
  template: `<ng-content></ng-content>`
})
export class TabComponent {
  @Input() title = '';
  @Input() active = false;
}

// Parent component ที่ใช้ ng-content
@Component({
  selector: 'app-tabs',
  standalone: true,
  imports: [CommonModule],
  template: `
    <div class="tabs">
      <div class="tab-headers">
        <button
          *ngFor="let tab of tabs"
          [class.active]="tab.active"
          (click)="selectTab(tab)"
        >
          {{ tab.title }}
        </button>
      </div>
      <div class="tab-content">
        <ng-content></ng-content>
      </div>
    </div>
  `
})
export class TabsComponent implements AfterContentInit, AfterContentChecked {
  @ContentChildren(TabComponent) tabs!: QueryList<TabComponent>;
  
  ngAfterContentInit(): void {
    // ✅ ใช้ได้แล้ว! tabs ถูก project เข้ามาแล้ว
    console.log('Tabs count:', this.tabs.length);
    
    // Set tab แรกเป็น active
    if (this.tabs.length > 0) {
      this.tabs.first.active = true;
    }
    
    // Subscribe การเปลี่ยนแปลง
    this.tabs.changes.subscribe(() => {
      console.log('Tabs changed!');
    });
  }
  
  ngAfterContentChecked(): void {
    // เรียกทุกครั้งที่ content ถูก check
    // ⚠️ ระวัง performance! ไม่ควรทำงานหนักที่นี่
  }
  
  selectTab(selectedTab: TabComponent): void {
    this.tabs.forEach(tab => tab.active = false);
    selectedTab.active = true;
  }
}
```

---

## 7. AfterView Hooks

เกี่ยวข้องกับ **View** ของ Component เอง

```typescript
import {
  Component, ViewChild, ViewChildren,
  AfterViewInit, AfterViewChecked,
  QueryList, ElementRef
} from '@angular/core';

@Component({
  selector: 'app-chart',
  standalone: true,
  template: `
    <div>
      <canvas #chartCanvas width="400" height="300"></canvas>
      <input *ngFor="let item of items; let i = index"
             #inputRef
             [value]="item"
             (change)="updateItem(i, $event)" />
    </div>
  `
})
export class ChartComponent implements AfterViewInit, AfterViewChecked {
  @ViewChild('chartCanvas') canvas!: ElementRef<HTMLCanvasElement>;
  @ViewChildren('inputRef') inputs!: QueryList<ElementRef<HTMLInputElement>>;
  
  items = ['Item 1', 'Item 2', 'Item 3'];
  private ctx!: CanvasRenderingContext2D;
  
  ngAfterViewInit(): void {
    // ✅ DOM elements พร้อมใช้แล้ว
    this.ctx = this.canvas.nativeElement.getContext('2d')!;
    this.drawChart();
    
    console.log('Inputs count:', this.inputs.length);
    
    // Focus input แรก
    this.inputs.first?.nativeElement.focus();
  }
  
  ngAfterViewChecked(): void {
    // ⚠️ เรียกบ่อยมาก! ระวัง infinite loop
    // ไม่ควรแก้ไข property ที่อยู่ใน template ที่นี่
  }
  
  private drawChart(): void {
    this.ctx.clearRect(0, 0, 400, 300);
    this.ctx.fillStyle = '#1976d2';
    this.ctx.fillRect(50, 50, 300, 200);
    this.ctx.fillStyle = 'white';
    this.ctx.font = '20px Arial';
    this.ctx.fillText('Chart Placeholder', 110, 160);
  }
  
  updateItem(index: number, event: Event): void {
    this.items[index] = (event.target as HTMLInputElement).value;
  }
}
```

---

## 8. ngOnDestroy

เรียก **ก่อน Component ถูกทำลาย** — สำคัญมากสำหรับการ cleanup

```typescript
import { Component, OnInit, OnDestroy } from '@angular/core';
import { Subject, interval, Subscription } from 'rxjs';
import { takeUntil } from 'rxjs/operators';

@Component({
  selector: 'app-timer',
  standalone: true,
  template: `
    <div>
      <h3>เวลาที่ผ่านไป: {{ elapsed }} วินาที</h3>
      <p>ข้อความ: {{ latestMessage }}</p>
    </div>
  `
})
export class TimerComponent implements OnInit, OnDestroy {
  elapsed = 0;
  latestMessage = '';
  
  // Pattern 1: Subject สำหรับ takeUntil (แนะนำ)
  private destroy$ = new Subject<void>();
  
  // Pattern 2: Subscription array
  private subscriptions: Subscription[] = [];
  
  ngOnInit(): void {
    // Pattern 1: ใช้ takeUntil
    interval(1000)
      .pipe(takeUntil(this.destroy$))
      .subscribe(() => {
        this.elapsed++;
      });
      
    // Pattern 2: เก็บ subscription ไว้ unsubscribe ทีหลัง
    const messageSub = interval(3000).subscribe(() => {
      this.latestMessage = `อัปเดตที่ ${new Date().toLocaleTimeString()}`;
    });
    this.subscriptions.push(messageSub);
    
    // Event listeners
    document.addEventListener('keydown', this.onKeyDown);
    
    // setInterval
    // ❌ จะ leak ถ้าไม่ clearInterval
    // this.intervalId = setInterval(() => { ... }, 1000);
  }
  
  ngOnDestroy(): void {
    // Pattern 1: emit เพื่อ complete ทุก observable ที่ใช้ takeUntil
    this.destroy$.next();
    this.destroy$.complete();
    
    // Pattern 2: unsubscribe ทั้งหมด
    this.subscriptions.forEach(sub => sub.unsubscribe());
    
    // Remove event listeners
    document.removeEventListener('keydown', this.onKeyDown);
    
    // Clear timers
    // clearInterval(this.intervalId);
    // clearTimeout(this.timeoutId);
    
    console.log('TimerComponent destroyed - cleanup done');
  }
  
  private onKeyDown = (event: KeyboardEvent) => {
    console.log('Key:', event.key);
  };
}
```

### Memory Leak — ปัญหาที่พบบ่อย

```typescript
// ❌ BAD — Memory Leak!
@Component({...})
export class LeakyComponent implements OnInit {
  data: any[] = [];
  
  ngOnInit(): void {
    // Subscribe แต่ไม่เคย unsubscribe
    interval(1000).subscribe(() => {
      this.data.push(new Date());  // ยังทำงานแม้ component ถูก destroy
    });
    
    // Event listener ที่ไม่ถูก remove
    document.addEventListener('scroll', () => {
      console.log('Scrolled');
    });
  }
}

// ✅ GOOD — ไม่มี Memory Leak
@Component({...})
export class CleanComponent implements OnInit, OnDestroy {
  data: any[] = [];
  private destroy$ = new Subject<void>();
  
  ngOnInit(): void {
    interval(1000)
      .pipe(takeUntil(this.destroy$))
      .subscribe(() => {
        this.data.push(new Date());
      });
      
    document.addEventListener('scroll', this.onScroll);
  }
  
  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
    document.removeEventListener('scroll', this.onScroll);
  }
  
  private onScroll = () => {
    console.log('Scrolled');
  };
}
```

---

## 9. Lifecycle ใน Standalone Components (Angular 14+)

ไม่มีความแตกต่าง — ใช้ lifecycle hooks เหมือนกันทุกประการ

```typescript
import { Component, OnInit, OnDestroy, signal } from '@angular/core';
import { CommonModule } from '@angular/common';
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';  // Angular 16+

@Component({
  selector: 'app-modern',
  standalone: true,
  imports: [CommonModule],
  template: `<p>Count: {{ count() }}</p>`
})
export class ModernComponent implements OnInit {
  count = signal(0);
  
  constructor() {
    // Angular 16+ — takeUntilDestroyed ใช้ใน constructor
    interval(1000)
      .pipe(takeUntilDestroyed())  // ไม่ต้องสร้าง destroy$ เอง!
      .subscribe(() => {
        this.count.update(c => c + 1);
      });
  }
  
  ngOnInit(): void {
    console.log('Initialized');
  }
}
```

---

## 10. Workshop: Timer Component

```typescript
// timer.component.ts
import {
  Component, Input, Output, EventEmitter,
  OnInit, OnDestroy, OnChanges, SimpleChanges,
  signal, computed
} from '@angular/core';
import { CommonModule } from '@angular/common';
import { Subject, interval } from 'rxjs';
import { takeUntil, map } from 'rxjs/operators';

export type TimerMode = 'countdown' | 'stopwatch';
export type TimerStatus = 'idle' | 'running' | 'paused' | 'finished';

@Component({
  selector: 'app-timer',
  standalone: true,
  imports: [CommonModule],
  template: `
    <div class="timer" [class]="'timer-' + status()">
      
      <!-- Display -->
      <div class="time-display">
        <span class="hours" *ngIf="showHours">{{ formattedTime().hours }}:</span>
        <span class="minutes">{{ formattedTime().minutes }}:</span>
        <span class="seconds">{{ formattedTime().seconds }}</span>
      </div>
      
      <!-- Progress Bar (สำหรับ countdown) -->
      <div class="progress-bar" *ngIf="mode === 'countdown'">
        <div class="progress" [style.width.%]="progressPercent()"></div>
      </div>
      
      <!-- Status -->
      <p class="status-text">{{ statusText() }}</p>
      
      <!-- Controls -->
      <div class="controls">
        <button
          *ngIf="status() === 'idle' || status() === 'paused'"
          (click)="start()"
          class="btn btn-start"
        >
          {{ status() === 'paused' ? '▶ ต่อ' : '▶ เริ่ม' }}
        </button>
        
        <button
          *ngIf="status() === 'running'"
          (click)="pause()"
          class="btn btn-pause"
        >
          ⏸ หยุดชั่วคราว
        </button>
        
        <button
          *ngIf="status() !== 'idle'"
          (click)="reset()"
          class="btn btn-reset"
        >
          ↺ รีเซ็ต
        </button>
      </div>
      
      <!-- Lap Times (สำหรับ stopwatch) -->
      <div class="laps" *ngIf="mode === 'stopwatch' && laps.length > 0">
        <h4>Lap Times</h4>
        <div class="lap-list">
          <div
            *ngFor="let lap of laps; let i = index"
            class="lap-item"
            [class.fastest]="i === fastestLapIndex"
            [class.slowest]="i === slowestLapIndex && laps.length > 2"
          >
            <span>Lap {{ i + 1 }}</span>
            <span>{{ formatSeconds(lap) }}</span>
          </div>
        </div>
        <button (click)="addLap()" [disabled]="status() !== 'running'" class="btn btn-lap">
          + Lap
        </button>
      </div>
    </div>
  `,
  styles: [`
    .timer {
      background: #1a1a2e;
      color: white;
      padding: 2rem;
      border-radius: 16px;
      text-align: center;
      max-width: 320px;
      margin: 0 auto;
      font-family: monospace;
    }
    
    .time-display {
      font-size: 3.5rem;
      font-weight: 700;
      letter-spacing: 2px;
      margin-bottom: 1rem;
      color: #00d4ff;
    }
    
    .timer-finished .time-display { color: #ff4757; animation: blink 1s infinite; }
    
    @keyframes blink {
      0%, 100% { opacity: 1; }
      50% { opacity: 0.3; }
    }
    
    .progress-bar {
      height: 6px;
      background: rgba(255,255,255,0.2);
      border-radius: 3px;
      margin-bottom: 1rem;
      overflow: hidden;
    }
    
    .progress {
      height: 100%;
      background: linear-gradient(90deg, #00d4ff, #7b2ff7);
      transition: width 1s linear;
    }
    
    .status-text {
      color: rgba(255,255,255,0.6);
      font-size: 0.875rem;
      margin-bottom: 1.5rem;
    }
    
    .controls {
      display: flex;
      justify-content: center;
      gap: 0.75rem;
    }
    
    .btn {
      padding: 0.5rem 1.25rem;
      border: none;
      border-radius: 8px;
      cursor: pointer;
      font-size: 0.875rem;
      font-weight: 600;
      transition: all 0.2s;
      
      &:hover { opacity: 0.85; transform: scale(1.05); }
    }
    
    .btn-start { background: #00d4ff; color: #1a1a2e; }
    .btn-pause { background: #ffa502; color: #1a1a2e; }
    .btn-reset { background: rgba(255,255,255,0.2); color: white; }
    .btn-lap { background: #7b2ff7; color: white; margin-top: 0.5rem; }
    
    .laps {
      margin-top: 1.5rem;
      text-align: left;
      
      h4 { margin-bottom: 0.5rem; color: rgba(255,255,255,0.7); }
    }
    
    .lap-list { max-height: 200px; overflow-y: auto; }
    
    .lap-item {
      display: flex;
      justify-content: space-between;
      padding: 0.4rem 0.5rem;
      border-radius: 4px;
      font-size: 0.875rem;
      
      &.fastest { color: #2ed573; }
      &.slowest { color: #ff4757; }
    }
  `]
})
export class TimerComponent implements OnInit, OnDestroy, OnChanges {
  @Input() mode: TimerMode = 'stopwatch';
  @Input() initialSeconds = 60;  // สำหรับ countdown
  
  @Output() finished = new EventEmitter<void>();
  @Output() tick = new EventEmitter<number>();
  
  private destroy$ = new Subject<void>();
  private timerSub?: any;
  
  status = signal<TimerStatus>('idle');
  currentSeconds = signal(0);
  laps: number[] = [];
  private lastLapSeconds = 0;
  
  get showHours(): boolean {
    return this.currentSeconds() >= 3600;
  }
  
  formattedTime = computed(() => {
    const total = this.currentSeconds();
    const hours = Math.floor(total / 3600);
    const minutes = Math.floor((total % 3600) / 60);
    const seconds = total % 60;
    
    return {
      hours: String(hours).padStart(2, '0'),
      minutes: String(minutes).padStart(2, '0'),
      seconds: String(seconds).padStart(2, '0')
    };
  });
  
  progressPercent = computed(() => {
    if (this.mode !== 'countdown') return 0;
    return (this.currentSeconds() / this.initialSeconds) * 100;
  });
  
  statusText = computed(() => {
    switch (this.status()) {
      case 'idle': return 'พร้อมเริ่ม';
      case 'running': return this.mode === 'countdown' ? 'กำลังนับถอยหลัง...' : 'กำลังจับเวลา...';
      case 'paused': return 'หยุดชั่วคราว';
      case 'finished': return this.mode === 'countdown' ? 'หมดเวลา!' : 'เสร็จสิ้น';
    }
  });
  
  get fastestLapIndex(): number {
    if (this.laps.length < 2) return -1;
    return this.laps.indexOf(Math.min(...this.laps));
  }
  
  get slowestLapIndex(): number {
    if (this.laps.length < 2) return -1;
    return this.laps.indexOf(Math.max(...this.laps));
  }
  
  ngOnChanges(changes: SimpleChanges): void {
    if (changes['initialSeconds'] && !changes['initialSeconds'].firstChange) {
      this.reset();
    }
  }
  
  ngOnInit(): void {
    this.reset();
  }
  
  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
  }
  
  start(): void {
    if (this.status() === 'finished') return;
    
    this.status.set('running');
    
    this.timerSub = interval(1000)
      .pipe(takeUntil(this.destroy$))
      .subscribe(() => {
        if (this.mode === 'stopwatch') {
          this.currentSeconds.update(s => s + 1);
          this.tick.emit(this.currentSeconds());
        } else {
          if (this.currentSeconds() > 0) {
            this.currentSeconds.update(s => s - 1);
            this.tick.emit(this.currentSeconds());
            
            if (this.currentSeconds() === 0) {
              this.status.set('finished');
              this.finished.emit();
              this.timerSub?.unsubscribe();
            }
          }
        }
      });
  }
  
  pause(): void {
    this.status.set('paused');
    this.timerSub?.unsubscribe();
  }
  
  reset(): void {
    this.timerSub?.unsubscribe();
    this.status.set('idle');
    this.currentSeconds.set(
      this.mode === 'countdown' ? this.initialSeconds : 0
    );
    this.laps = [];
    this.lastLapSeconds = 0;
  }
  
  addLap(): void {
    if (this.status() !== 'running' || this.mode !== 'stopwatch') return;
    
    const lapTime = this.currentSeconds() - this.lastLapSeconds;
    this.laps.push(lapTime);
    this.lastLapSeconds = this.currentSeconds();
  }
  
  formatSeconds(seconds: number): string {
    const m = Math.floor(seconds / 60);
    const s = seconds % 60;
    return `${String(m).padStart(2, '0')}:${String(s).padStart(2, '0')}`;
  }
}
```

---

## สรุปบทที่ 13

| Hook | เรียกเมื่อ | ใช้สำหรับ |
|------|-----------|---------|
| ngOnChanges | Input เปลี่ยน | React ต่อ Input ที่เปลี่ยน |
| ngOnInit | Init ครั้งแรก | เรียก API, subscribe |
| ngDoCheck | ทุก CD cycle | Custom change detection |
| ngAfterContentInit | Content project แล้ว | ใช้ @ContentChild |
| ngAfterContentChecked | Content checked | ตรวจสอบ content |
| ngAfterViewInit | View init แล้ว | ใช้ @ViewChild, DOM |
| ngAfterViewChecked | View checked | ระวัง performance |
| ngOnDestroy | ก่อน destroy | Cleanup, unsubscribe |

---

## แบบฝึกหัด

1. สร้าง Component ที่แสดง log การเรียก lifecycle hooks ทุกตัว
2. สร้าง CountdownTimer ที่รับ `seconds` และ emit `finished`
3. สร้าง Component ที่ track จำนวนครั้งที่ render ด้วย `ngDoCheck`

---

[Part 14 — Angular Modules →](part-14-modules.md)
