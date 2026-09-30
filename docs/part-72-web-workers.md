# Part 72: Web Workers ใน Angular

## Web Workers คืออะไร

Web Workers ช่วยให้เราทำงานหนักๆ (heavy computation) ใน background thread แยกจาก main thread ป้องกัน UI freezing

### ปัญหาที่ Web Workers แก้ได้

- การคำนวณตัวเลขซับซ้อน
- Image/data processing
- JSON parsing ขนาดใหญ่
- Machine learning inference

---

## 1. สร้าง Web Worker ใน Angular

### ใช้ Angular CLI

```bash
ng generate web-worker app

# หรือระบุ path
ng generate web-worker app/workers/computation
```

### Worker File: computation.worker.ts

```typescript
// app/workers/computation.worker.ts

/// <reference lib="webworker" />

// ประเภทของ message
interface WorkerMessage {
  type: 'SORT' | 'FILTER' | 'CALCULATE' | 'PROCESS_IMAGE';
  payload: any;
  id: string;
}

interface WorkerResponse {
  type: 'SUCCESS' | 'ERROR' | 'PROGRESS';
  result?: any;
  error?: string;
  progress?: number;
  id: string;
}

addEventListener('message', ({ data }: MessageEvent<WorkerMessage>) => {
  const { type, payload, id } = data;

  try {
    let result: any;

    switch (type) {
      case 'SORT':
        result = handleSort(payload);
        break;
      case 'FILTER':
        result = handleFilter(payload);
        break;
      case 'CALCULATE':
        result = handleCalculate(payload);
        break;
      default:
        throw new Error(`Unknown message type: ${type}`);
    }

    const response: WorkerResponse = { type: 'SUCCESS', result, id };
    postMessage(response);
  } catch (error) {
    const response: WorkerResponse = {
      type: 'ERROR',
      error: (error as Error).message,
      id
    };
    postMessage(response);
  }
});

// เรียงข้อมูลขนาดใหญ่
function handleSort(data: { items: any[]; key: string; order: 'asc' | 'desc' }): any[] {
  const { items, key, order } = data;
  return [...items].sort((a, b) => {
    const valA = a[key];
    const valB = b[key];
    const dir = order === 'asc' ? 1 : -1;
    
    if (typeof valA === 'string') {
      return valA.localeCompare(valB) * dir;
    }
    return (valA - valB) * dir;
  });
}

// กรองข้อมูล
function handleFilter(data: { items: any[]; filters: Record<string, any> }): any[] {
  const { items, filters } = data;
  return items.filter(item => {
    return Object.entries(filters).every(([key, value]) => {
      if (value === null || value === undefined) return true;
      if (typeof value === 'string') {
        return String(item[key]).toLowerCase().includes(value.toLowerCase());
      }
      return item[key] === value;
    });
  });
}

// คำนวณ Fibonacci (ตัวอย่าง heavy computation)
function handleCalculate(data: { n: number }): number {
  return fibonacci(data.n);
}

function fibonacci(n: number): number {
  if (n <= 1) return n;
  
  let a = 0, b = 1;
  for (let i = 2; i <= n; i++) {
    const temp = a + b;
    a = b;
    b = temp;
  }
  return b;
}
```

---

## 2. Worker Service

```typescript
// app/services/worker.service.ts
import { Injectable, NgZone, OnDestroy } from '@angular/core';
import { Observable, Subject, fromEvent, BehaviorSubject } from 'rxjs';
import { filter, map, take, takeUntil } from 'rxjs/operators';

interface WorkerJob {
  id: string;
  type: string;
  payload: any;
  resolve: (value: any) => void;
  reject: (reason: any) => void;
}

@Injectable({ providedIn: 'root' })
export class WorkerService implements OnDestroy {
  private worker!: Worker;
  private jobQueue = new Map<string, WorkerJob>();
  private isReady$ = new BehaviorSubject(false);
  private destroy$ = new Subject<void>();
  private jobCounter = 0;

  constructor(private ngZone: NgZone) {
    this.initWorker();
  }

  private initWorker(): void {
    if (typeof Worker !== 'undefined') {
      this.worker = new Worker(
        new URL('../workers/computation.worker', import.meta.url),
        { type: 'module' }
      );

      this.worker.onmessage = ({ data }) => {
        this.ngZone.run(() => this.handleResponse(data));
      };

      this.worker.onerror = (error) => {
        console.error('Worker error:', error);
        this.ngZone.run(() => {
          // Reject pending jobs
          this.jobQueue.forEach(job => job.reject(error));
          this.jobQueue.clear();
        });
      };

      this.isReady$.next(true);
    } else {
      console.warn('Web Workers ไม่รองรับใน environment นี้');
    }
  }

  private handleResponse(response: any): void {
    const job = this.jobQueue.get(response.id);
    if (!job) return;

    this.jobQueue.delete(response.id);

    if (response.type === 'SUCCESS') {
      job.resolve(response.result);
    } else if (response.type === 'ERROR') {
      job.reject(new Error(response.error));
    }
  }

  // ส่งงานไปยัง Worker และรับผลลัพธ์เป็น Promise
  run<T>(type: string, payload: any): Promise<T> {
    return new Promise((resolve, reject) => {
      if (!this.worker) {
        // Fallback: ทำงานบน main thread
        reject(new Error('Worker ไม่พร้อมใช้งาน'));
        return;
      }

      const id = `job-${++this.jobCounter}-${Date.now()}`;
      this.jobQueue.set(id, { id, type, payload, resolve, reject });
      
      this.worker.postMessage({ type, payload, id });
    });
  }

  // ส่งงานและรับเป็น Observable
  runAsObservable<T>(type: string, payload: any): Observable<T> {
    return new Observable(observer => {
      this.run<T>(type, payload)
        .then(result => {
          observer.next(result);
          observer.complete();
        })
        .catch(error => observer.error(error));
    });
  }

  sortData<T>(items: T[], key: keyof T, order: 'asc' | 'desc' = 'asc'): Promise<T[]> {
    return this.run<T[]>('SORT', { items, key, order });
  }

  filterData<T>(items: T[], filters: Partial<T>): Promise<T[]> {
    return this.run<T[]>('FILTER', { items, filters });
  }

  calculateFibonacci(n: number): Promise<number> {
    return this.run<number>('CALCULATE', { n });
  }

  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
    this.worker?.terminate();
  }
}
```

---

## 3. ใช้งาน Worker ใน Component

```typescript
// app/components/data-processor/data-processor.component.ts
import { Component, OnInit } from '@angular/core';
import { WorkerService } from '../../services/worker.service';

interface Employee {
  id: number;
  name: string;
  department: string;
  salary: number;
  joinDate: string;
}

@Component({
  selector: 'app-data-processor',
  template: `
    <div class="processor">
      <h2>Data Processor - Web Worker Demo</h2>
      
      <div class="controls">
        <button (click)="generateData()" [disabled]="isProcessing">
          สร้างข้อมูล 100,000 รายการ
        </button>
        
        <button (click)="sortData()" [disabled]="isProcessing || !data.length">
          เรียงข้อมูล (Worker)
        </button>

        <button (click)="calculateFib()" [disabled]="isProcessing">
          คำนวณ Fibonacci(45) (Worker)
        </button>
      </div>

      <div class="filters">
        <input 
          [(ngModel)]="filterText" 
          placeholder="ค้นหาชื่อ..."
          (input)="onFilterChange()"
        >
        <select [(ngModel)]="selectedDept" (change)="onFilterChange()">
          <option value="">ทุกแผนก</option>
          <option *ngFor="let dept of departments" [value]="dept">{{ dept }}</option>
        </select>
      </div>

      <div class="stats">
        <div class="stat">
          <span>ข้อมูลทั้งหมด</span>
          <strong>{{ data.length | number }}</strong>
        </div>
        <div class="stat">
          <span>แสดงผล</span>
          <strong>{{ displayData.length | number }}</strong>
        </div>
        <div class="stat" *ngIf="processingTime">
          <span>เวลาประมวลผล</span>
          <strong>{{ processingTime }}ms</strong>
        </div>
      </div>

      <div *ngIf="fibResult" class="fib-result">
        Fibonacci(45) = {{ fibResult | number }}
      </div>

      <div *ngIf="isProcessing" class="loading">
        กำลังประมวลผล...
        <div class="spinner"></div>
      </div>

      <div class="table-wrapper">
        <table *ngIf="displayData.length > 0">
          <thead>
            <tr>
              <th>ID</th>
              <th (click)="sortBy('name')">ชื่อ</th>
              <th (click)="sortBy('department')">แผนก</th>
              <th (click)="sortBy('salary')">เงินเดือน</th>
            </tr>
          </thead>
          <tbody>
            <tr *ngFor="let emp of displayData | slice:0:50">
              <td>{{ emp.id }}</td>
              <td>{{ emp.name }}</td>
              <td>{{ emp.department }}</td>
              <td>{{ emp.salary | currency:'THB':'symbol':'1.0-0' }}</td>
            </tr>
          </tbody>
        </table>
        <p *ngIf="displayData.length > 50" class="note">
          แสดง 50 รายการแรกจาก {{ displayData.length | number }} รายการ
        </p>
      </div>
    </div>
  `
})
export class DataProcessorComponent implements OnInit {
  data: Employee[] = [];
  displayData: Employee[] = [];
  isProcessing = false;
  processingTime: number | null = null;
  fibResult: number | null = null;
  filterText = '';
  selectedDept = '';
  currentSortKey: keyof Employee = 'id';
  currentSortOrder: 'asc' | 'desc' = 'asc';

  departments = ['Engineering', 'Marketing', 'Sales', 'HR', 'Finance', 'Operations'];

  constructor(private workerService: WorkerService) {}

  ngOnInit(): void {}

  generateData(): void {
    const firstNames = ['สมชาย', 'สมหญิง', 'วิชัย', 'นิดา', 'ประยุทธ์', 'สุภา', 'กิตติ', 'พิมพ์'];
    const lastNames = ['ใจดี', 'มีสุข', 'รักงาน', 'ชอบเรียน', 'ทำดี', 'มุ่งมั่น'];
    
    this.data = Array.from({ length: 100000 }, (_, i) => ({
      id: i + 1,
      name: `${firstNames[Math.floor(Math.random() * firstNames.length)]} ${lastNames[Math.floor(Math.random() * lastNames.length)]}`,
      department: this.departments[Math.floor(Math.random() * this.departments.length)],
      salary: 20000 + Math.floor(Math.random() * 80000),
      joinDate: new Date(2020 + Math.floor(Math.random() * 5), Math.floor(Math.random() * 12), Math.floor(Math.random() * 28) + 1).toISOString()
    }));
    
    this.displayData = [...this.data];
    console.log('สร้างข้อมูลเสร็จสิ้น:', this.data.length, 'รายการ');
  }

  async sortData(): Promise<void> {
    if (!this.data.length) return;
    
    this.isProcessing = true;
    const start = performance.now();

    try {
      this.data = await this.workerService.sortData(this.data, 'salary', 'desc');
      this.displayData = [...this.data];
      this.processingTime = Math.round(performance.now() - start);
    } catch (error) {
      console.error('Sort error:', error);
    } finally {
      this.isProcessing = false;
    }
  }

  async sortBy(key: keyof Employee): Promise<void> {
    if (this.currentSortKey === key) {
      this.currentSortOrder = this.currentSortOrder === 'asc' ? 'desc' : 'asc';
    } else {
      this.currentSortKey = key;
      this.currentSortOrder = 'asc';
    }

    this.isProcessing = true;
    const start = performance.now();

    try {
      this.displayData = await this.workerService.sortData(this.displayData, key, this.currentSortOrder);
      this.processingTime = Math.round(performance.now() - start);
    } finally {
      this.isProcessing = false;
    }
  }

  async calculateFib(): Promise<void> {
    this.isProcessing = true;
    const start = performance.now();

    try {
      this.fibResult = await this.workerService.calculateFibonacci(45);
      this.processingTime = Math.round(performance.now() - start);
    } finally {
      this.isProcessing = false;
    }
  }

  async onFilterChange(): Promise<void> {
    const filters: Partial<Employee> = {};
    if (this.filterText) filters.name = this.filterText;
    if (this.selectedDept) filters.department = this.selectedDept as any;

    if (Object.keys(filters).length === 0) {
      this.displayData = [...this.data];
      return;
    }

    this.isProcessing = true;
    try {
      this.displayData = await this.workerService.filterData(this.data, filters);
    } finally {
      this.isProcessing = false;
    }
  }
}
```

---

## 4. Shared Array Buffer สำหรับ Performance สูงสุด

```typescript
// app/workers/shared-memory.worker.ts
/// <reference lib="webworker" />

addEventListener('message', ({ data }) => {
  const { sharedBuffer, length } = data;
  
  // ทำงานกับ SharedArrayBuffer
  const view = new Int32Array(sharedBuffer);
  
  // คำนวณผลรวม
  let sum = 0;
  for (let i = 0; i < length; i++) {
    sum += view[i];
  }
  
  postMessage({ sum });
});

// ใช้ใน component
// const sharedBuffer = new SharedArrayBuffer(Int32Array.BYTES_PER_ELEMENT * 1000);
// const view = new Int32Array(sharedBuffer);
// worker.postMessage({ sharedBuffer, length: 1000 });
```

---

## สรุป

| วิธี | เหมาะกับ |
|------|----------|
| Worker Service | งานทั่วไป |
| Comlink | API ซับซ้อน |
| SharedArrayBuffer | High performance |

### Best Practices

1. ใช้ Worker สำหรับงานที่ใช้เวลา > 50ms
2. Serialize/Deserialize ข้อมูลอย่างระมัดระวัง
3. จัดการ error handling
4. Terminate worker เมื่อไม่ใช้งาน
5. Test worker แยกจาก component
