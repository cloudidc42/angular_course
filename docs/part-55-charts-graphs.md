# Part 55: Charts & Graphs ใน Angular

## บทนำ

การแสดงผลข้อมูลด้วยกราฟเป็นสิ่งสำคัญในแอปพลิเคชันธุรกิจ บทนี้จะสอนการใช้ Chart.js และ ApexCharts ใน Angular

## 1. Chart.js Integration

```bash
npm install chart.js ng2-charts
```

### ตั้งค่าใน Module

```typescript
// app.config.ts (Standalone)
import { ApplicationConfig } from '@angular/core';
import { provideCharts, withDefaultRegisterables } from 'ng2-charts';

export const appConfig: ApplicationConfig = {
  providers: [
    provideCharts(withDefaultRegisterables())
  ]
};
```

### Line Chart Component

```typescript
// line-chart.component.ts
import { Component, Input, OnChanges, SimpleChanges } from '@angular/core';
import { BaseChartDirective } from 'ng2-charts';
import { ChartConfiguration, ChartData, ChartType } from 'chart.js';

@Component({
  selector: 'app-line-chart',
  standalone: true,
  imports: [BaseChartDirective],
  template: `
    <div class="chart-container" [style.height.px]="height">
      <canvas baseChart
        [data]="chartData"
        [options]="chartOptions"
        [type]="chartType"
      ></canvas>
    </div>
  `,
  styles: [`
    .chart-container { position: relative; width: 100%; }
  `]
})
export class LineChartComponent implements OnChanges {
  @Input() labels: string[] = [];
  @Input() datasets: { label: string; data: number[]; color?: string }[] = [];
  @Input() title = '';
  @Input() height = 300;

  chartType: ChartType = 'line';

  chartData: ChartData = {
    labels: [],
    datasets: []
  };

  chartOptions: ChartConfiguration['options'] = {
    responsive: true,
    maintainAspectRatio: false,
    plugins: {
      legend: { position: 'top' },
      title: {
        display: true,
        text: this.title
      }
    },
    scales: {
      y: {
        beginAtZero: true,
        grid: { color: 'rgba(0,0,0,0.05)' }
      },
      x: {
        grid: { color: 'rgba(0,0,0,0.05)' }
      }
    }
  };

  ngOnChanges(changes: SimpleChanges): void {
    if (changes['labels'] || changes['datasets']) {
      this.updateChart();
    }
    if (changes['title']) {
      this.chartOptions = {
        ...this.chartOptions,
        plugins: {
          ...this.chartOptions?.plugins,
          title: { display: !!this.title, text: this.title }
        }
      };
    }
  }

  private updateChart(): void {
    this.chartData = {
      labels: this.labels,
      datasets: this.datasets.map((ds, i) => ({
        label: ds.label,
        data: ds.data,
        borderColor: ds.color || this.getDefaultColor(i),
        backgroundColor: this.hexToRgba(ds.color || this.getDefaultColor(i), 0.1),
        tension: 0.4,
        fill: true,
        pointRadius: 4,
        pointHoverRadius: 6
      }))
    };
  }

  private getDefaultColor(index: number): string {
    const colors = ['#007bff', '#28a745', '#dc3545', '#ffc107', '#17a2b8'];
    return colors[index % colors.length];
  }

  private hexToRgba(hex: string, alpha: number): string {
    const result = /^#?([a-f\d]{2})([a-f\d]{2})([a-f\d]{2})$/i.exec(hex);
    if (!result) return `rgba(0,0,0,${alpha})`;
    return `rgba(${parseInt(result[1], 16)}, ${parseInt(result[2], 16)}, ${parseInt(result[3], 16)}, ${alpha})`;
  }
}
```

### Bar Chart Component

```typescript
// bar-chart.component.ts
import { Component, Input, OnChanges, SimpleChanges } from '@angular/core';
import { BaseChartDirective } from 'ng2-charts';
import { ChartConfiguration, ChartData, ChartType } from 'chart.js';

@Component({
  selector: 'app-bar-chart',
  standalone: true,
  imports: [BaseChartDirective],
  template: `
    <div class="chart-wrapper" [style.height.px]="height">
      <canvas baseChart
        [data]="chartData"
        [options]="chartOptions"
        [type]="chartType"
      ></canvas>
    </div>
  `,
  styles: [`.chart-wrapper { position: relative; width: 100%; }`]
})
export class BarChartComponent implements OnChanges {
  @Input() labels: string[] = [];
  @Input() data: number[] = [];
  @Input() label = 'ข้อมูล';
  @Input() color = '#007bff';
  @Input() horizontal = false;
  @Input() height = 300;

  chartType: ChartType = 'bar';

  chartData: ChartData<'bar'> = {
    labels: [],
    datasets: []
  };

  chartOptions: ChartConfiguration<'bar'>['options'] = {
    responsive: true,
    maintainAspectRatio: false,
    indexAxis: 'x',
    plugins: {
      legend: { display: false },
      tooltip: {
        callbacks: {
          label: (context) => `${context.dataset.label}: ${context.parsed.y?.toLocaleString('th-TH')}`
        }
      }
    }
  };

  ngOnChanges(changes: SimpleChanges): void {
    if (changes['labels'] || changes['data'] || changes['color']) {
      this.updateChart();
    }
    if (changes['horizontal']) {
      this.chartOptions = {
        ...this.chartOptions,
        indexAxis: this.horizontal ? 'y' : 'x'
      };
    }
  }

  private updateChart(): void {
    this.chartData = {
      labels: this.labels,
      datasets: [{
        label: this.label,
        data: this.data,
        backgroundColor: this.data.map((_, i) => this.getBarColor(i)),
        borderRadius: 4,
        borderWidth: 0
      }]
    };
  }

  private getBarColor(index: number): string {
    const colors = [
      '#007bff', '#28a745', '#dc3545', '#ffc107',
      '#17a2b8', '#6f42c1', '#fd7e14', '#20c997'
    ];
    return colors[index % colors.length];
  }
}
```

### Pie/Doughnut Chart

```typescript
// pie-chart.component.ts
import { Component, Input, OnChanges, SimpleChanges } from '@angular/core';
import { BaseChartDirective } from 'ng2-charts';
import { ChartData, ChartType, ChartOptions } from 'chart.js';

@Component({
  selector: 'app-pie-chart',
  standalone: true,
  imports: [BaseChartDirective],
  template: `
    <div class="pie-container" [style.height.px]="size">
      <canvas baseChart
        [data]="chartData"
        [options]="chartOptions"
        [type]="chartType"
      ></canvas>
    </div>
  `,
  styles: [`.pie-container { position: relative; width: 100%; }`]
})
export class PieChartComponent implements OnChanges {
  @Input() labels: string[] = [];
  @Input() data: number[] = [];
  @Input() doughnut = false;
  @Input() size = 300;
  @Input() showLegend = true;

  get chartType(): ChartType {
    return this.doughnut ? 'doughnut' : 'pie';
  }

  chartData: ChartData<'pie' | 'doughnut'> = {
    labels: [],
    datasets: []
  };

  chartOptions: ChartOptions<'pie' | 'doughnut'> = {
    responsive: true,
    maintainAspectRatio: false,
    plugins: {
      legend: {
        position: 'right',
        display: this.showLegend
      },
      tooltip: {
        callbacks: {
          label: (context) => {
            const total = (context.dataset.data as number[]).reduce((a, b) => a + b, 0);
            const percentage = ((context.parsed / total) * 100).toFixed(1);
            return `${context.label}: ${context.parsed.toLocaleString('th-TH')} (${percentage}%)`;
          }
        }
      }
    }
  };

  ngOnChanges(changes: SimpleChanges): void {
    if (changes['labels'] || changes['data']) {
      this.chartData = {
        labels: this.labels,
        datasets: [{
          data: this.data,
          backgroundColor: this.generateColors(this.data.length),
          hoverOffset: 10
        }]
      };
    }
  }

  private generateColors(count: number): string[] {
    const baseColors = [
      '#FF6384', '#36A2EB', '#FFCE56', '#4BC0C0',
      '#9966FF', '#FF9F40', '#FF6384', '#C9CBCF'
    ];
    return Array.from({ length: count }, (_, i) => baseColors[i % baseColors.length]);
  }
}
```

## 2. ApexCharts Integration

```bash
npm install apexcharts ng-apexcharts
```

```typescript
// apex-sales-chart.component.ts
import { Component, OnInit } from '@angular/core';
import {
  NgApexchartsModule,
  ApexChart,
  ApexAxisChartSeries,
  ApexXAxis,
  ApexDataLabels,
  ApexStroke,
  ApexYAxis,
  ApexTitleSubtitle,
  ApexTooltip,
  ApexFill,
  ApexLegend
} from 'ng-apexcharts';

export type ChartOptions = {
  series: ApexAxisChartSeries;
  chart: ApexChart;
  xaxis: ApexXAxis;
  yaxis: ApexYAxis;
  stroke: ApexStroke;
  dataLabels: ApexDataLabels;
  title: ApexTitleSubtitle;
  tooltip: ApexTooltip;
  fill: ApexFill;
  legend: ApexLegend;
};

@Component({
  selector: 'app-apex-sales-chart',
  standalone: true,
  imports: [NgApexchartsModule],
  template: `
    <apx-chart
      [series]="chartOptions.series"
      [chart]="chartOptions.chart"
      [xaxis]="chartOptions.xaxis"
      [yaxis]="chartOptions.yaxis"
      [stroke]="chartOptions.stroke"
      [dataLabels]="chartOptions.dataLabels"
      [title]="chartOptions.title"
      [tooltip]="chartOptions.tooltip"
      [fill]="chartOptions.fill"
      [legend]="chartOptions.legend"
    ></apx-chart>
  `
})
export class ApexSalesChartComponent implements OnInit {
  chartOptions!: ChartOptions;

  ngOnInit(): void {
    this.chartOptions = {
      series: [
        {
          name: 'ยอดขายปีนี้',
          data: [45000, 52000, 38000, 74000, 56000, 68000,
                 72000, 84000, 91000, 67000, 85000, 95000]
        },
        {
          name: 'ยอดขายปีที่แล้ว',
          data: [35000, 41000, 36000, 60000, 44000, 52000,
                 63000, 71000, 78000, 55000, 70000, 82000]
        }
      ],
      chart: {
        height: 400,
        type: 'area',
        toolbar: { show: true },
        zoom: { enabled: true }
      },
      xaxis: {
        categories: ['ม.ค.', 'ก.พ.', 'มี.ค.', 'เม.ย.', 'พ.ค.', 'มิ.ย.',
                     'ก.ค.', 'ส.ค.', 'ก.ย.', 'ต.ค.', 'พ.ย.', 'ธ.ค.']
      },
      yaxis: {
        labels: {
          formatter: (value) => `฿${(value / 1000).toFixed(0)}K`
        }
      },
      stroke: { curve: 'smooth', width: 2 },
      dataLabels: { enabled: false },
      title: {
        text: 'ยอดขายรายเดือน',
        align: 'left',
        style: { fontSize: '18px', fontWeight: 'bold' }
      },
      tooltip: {
        y: {
          formatter: (value) => `฿${value.toLocaleString('th-TH')}`
        }
      },
      fill: {
        type: 'gradient',
        gradient: {
          shadeIntensity: 1,
          opacityFrom: 0.7,
          opacityTo: 0.1
        }
      },
      legend: { position: 'top' }
    };
  }
}
```

## 3. Dashboard Component

```typescript
// analytics-dashboard.component.ts
import { Component, OnInit } from '@angular/core';
import { CommonModule } from '@angular/common';
import { LineChartComponent } from './line-chart.component';
import { BarChartComponent } from './bar-chart.component';
import { PieChartComponent } from './pie-chart.component';

interface KPI {
  title: string;
  value: string;
  change: number;
  icon: string;
}

@Component({
  selector: 'app-analytics-dashboard',
  standalone: true,
  imports: [CommonModule, LineChartComponent, BarChartComponent, PieChartComponent],
  template: `
    <div class="dashboard">
      <h1>Analytics Dashboard</h1>

      <!-- KPI Cards -->
      <div class="kpi-grid">
        <div *ngFor="let kpi of kpis" class="kpi-card">
          <div class="kpi-icon">{{ kpi.icon }}</div>
          <div class="kpi-content">
            <div class="kpi-title">{{ kpi.title }}</div>
            <div class="kpi-value">{{ kpi.value }}</div>
            <div class="kpi-change" [class.positive]="kpi.change > 0" [class.negative]="kpi.change < 0">
              {{ kpi.change > 0 ? '+' : '' }}{{ kpi.change }}%
            </div>
          </div>
        </div>
      </div>

      <!-- Charts Row -->
      <div class="charts-row">
        <div class="chart-card large">
          <h3>ยอดขายรายวัน (7 วันล่าสุด)</h3>
          <app-line-chart
            [labels]="dailyLabels"
            [datasets]="dailyDatasets"
            [height]="280"
          ></app-line-chart>
        </div>

        <div class="chart-card">
          <h3>สัดส่วนตามหมวดหมู่</h3>
          <app-pie-chart
            [labels]="categoryLabels"
            [data]="categoryData"
            [doughnut]="true"
            [size]="280"
          ></app-pie-chart>
        </div>
      </div>

      <!-- Bottom Charts -->
      <div class="charts-row">
        <div class="chart-card">
          <h3>Top 10 สินค้าขายดี</h3>
          <app-bar-chart
            [labels]="topProductLabels"
            [data]="topProductData"
            [horizontal]="true"
            [height]="320"
          ></app-bar-chart>
        </div>
      </div>
    </div>
  `,
  styles: [`
    .dashboard { padding: 1rem; }
    .kpi-grid {
      display: grid;
      grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
      gap: 1rem;
      margin-bottom: 1.5rem;
    }
    .kpi-card {
      display: flex;
      align-items: center;
      gap: 1rem;
      padding: 1.5rem;
      background: white;
      border-radius: 12px;
      box-shadow: 0 2px 8px rgba(0,0,0,0.08);
    }
    .kpi-icon { font-size: 2.5rem; }
    .kpi-title { color: #6c757d; font-size: 0.85rem; }
    .kpi-value { font-size: 1.8rem; font-weight: bold; }
    .kpi-change { font-size: 0.9rem; font-weight: 500; }
    .positive { color: #28a745; }
    .negative { color: #dc3545; }
    .charts-row {
      display: grid;
      grid-template-columns: 2fr 1fr;
      gap: 1rem;
      margin-bottom: 1rem;
    }
    .chart-card {
      background: white;
      padding: 1.5rem;
      border-radius: 12px;
      box-shadow: 0 2px 8px rgba(0,0,0,0.08);
    }
    .chart-card h3 { margin: 0 0 1rem; color: #333; }
    .large { grid-column: span 1; }
  `]
})
export class AnalyticsDashboardComponent implements OnInit {
  kpis: KPI[] = [
    { title: 'ยอดขายรวม', value: '฿1.2M', change: 12.5, icon: '💰' },
    { title: 'ออเดอร์ใหม่', value: '348', change: 8.2, icon: '📦' },
    { title: 'ลูกค้าใหม่', value: '89', change: -3.1, icon: '👥' },
    { title: 'Conversion Rate', value: '3.2%', change: 0.5, icon: '📈' }
  ];

  dailyLabels = ['จ', 'อ', 'พ', 'พฤ', 'ศ', 'ส', 'อา'];
  dailyDatasets = [
    {
      label: 'ยอดขาย',
      data: [85000, 92000, 78000, 95000, 88000, 110000, 75000],
      color: '#007bff'
    },
    {
      label: 'เป้าหมาย',
      data: [80000, 80000, 80000, 80000, 80000, 80000, 80000],
      color: '#dc3545'
    }
  ];

  categoryLabels = ['อิเล็กทรอนิกส์', 'เสื้อผ้า', 'อาหาร', 'หนังสือ', 'อื่นๆ'];
  categoryData = [35, 25, 20, 12, 8];

  topProductLabels = [
    'iPhone 15', 'Samsung S24', 'AirPods Pro',
    'iPad Air', 'MacBook Air', 'Galaxy Tab',
    'Sony WH-1000XM5', 'Apple Watch', 'Xiaomi 14', 'OnePlus 12'
  ];
  topProductData = [450, 380, 320, 280, 260, 240, 210, 190, 170, 150];

  ngOnInit(): void {}
}
```

## 4. Real-time Chart Update

```typescript
// realtime-chart.component.ts
import { Component, OnInit, OnDestroy } from '@angular/core';
import { BaseChartDirective } from 'ng2-charts';
import { ChartData, ChartOptions } from 'chart.js';
import { interval, Subject } from 'rxjs';
import { takeUntil } from 'rxjs/operators';

@Component({
  selector: 'app-realtime-chart',
  standalone: true,
  imports: [BaseChartDirective],
  template: `
    <div style="height: 300px; position: relative;">
      <canvas baseChart
        [data]="chartData"
        [options]="chartOptions"
        type="line"
      ></canvas>
    </div>
    <div style="margin-top: 1rem;">
      <button (click)="toggleUpdate()">
        {{ isRunning ? 'หยุด' : 'เริ่ม' }}อัปเดต
      </button>
      <span style="margin-left: 1rem; color: #6c757d;">
        ค่าปัจจุบัน: {{ currentValue.toFixed(2) }}
      </span>
    </div>
  `
})
export class RealtimeChartComponent implements OnInit, OnDestroy {
  private readonly MAX_POINTS = 20;
  private destroy$ = new Subject<void>();
  isRunning = false;
  currentValue = 50;

  chartData: ChartData<'line'> = {
    labels: [],
    datasets: [{
      label: 'Real-time Data',
      data: [],
      borderColor: '#007bff',
      backgroundColor: 'rgba(0,123,255,0.1)',
      tension: 0.4,
      fill: true,
      pointRadius: 2
    }]
  };

  chartOptions: ChartOptions<'line'> = {
    responsive: true,
    maintainAspectRatio: false,
    animation: { duration: 0 },
    scales: {
      y: { min: 0, max: 100 }
    },
    plugins: { legend: { display: false } }
  };

  ngOnInit(): void {
    // เริ่มต้นด้วยข้อมูล
    for (let i = 0; i < this.MAX_POINTS; i++) {
      this.addDataPoint();
    }
    this.startUpdate();
  }

  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
  }

  toggleUpdate(): void {
    if (this.isRunning) {
      this.destroy$.next();
      this.isRunning = false;
    } else {
      this.startUpdate();
    }
  }

  private startUpdate(): void {
    this.isRunning = true;
    interval(1000).pipe(
      takeUntil(this.destroy$)
    ).subscribe(() => {
      this.addDataPoint();
      this.trimData();
    });
  }

  private addDataPoint(): void {
    // สร้างข้อมูล random ที่ smooth
    const change = (Math.random() - 0.5) * 10;
    this.currentValue = Math.max(0, Math.min(100, this.currentValue + change));

    const now = new Date().toLocaleTimeString('th-TH');
    this.chartData.labels?.push(now);
    (this.chartData.datasets[0].data as number[]).push(this.currentValue);
  }

  private trimData(): void {
    if ((this.chartData.labels?.length || 0) > this.MAX_POINTS) {
      this.chartData.labels?.shift();
      (this.chartData.datasets[0].data as number[]).shift();
    }
  }
}
```

## สรุป

| Library | ข้อดี | ข้อเสีย |
|---------|-------|---------|
| Chart.js (ng2-charts) | เบา, คุ้นเคย, ฟรี | Animation จำกัด |
| ApexCharts | สวย, Interactive สูง | ขนาดใหญ่กว่า |
| D3.js | ยืดหยุ่นสูงสุด | เรียนรู้ยาก |
| ngx-charts | Angular-native | Chart types น้อย |

เลือก Chart.js สำหรับกราฟพื้นฐาน และ ApexCharts สำหรับ Dashboard ที่ต้องการความสวยงาม
