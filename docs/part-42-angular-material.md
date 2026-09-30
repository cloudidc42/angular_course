# Part 42: Angular Material

## บทนำ

Angular Material เป็น UI component library สำหรับ Angular ที่ใช้ Material Design จาก Google ในบทนี้เราจะ setup Angular Material, ปรับแต่ง Theme และใช้งาน components ที่พบบ่อย

---

## 1. การติดตั้ง Angular Material

```bash
# ติดตั้งผ่าน Angular CLI (แนะนำ)
ng add @angular/material

# หรือติดตั้งแบบ manual
npm install @angular/material @angular/cdk @angular/animations
```

เมื่อรัน `ng add` จะถามคำถามต่อไปนี้:
- Choose a prebuilt theme: `Indigo/Pink` (หรือ theme อื่น)
- Set up global typography styles: `Yes`
- Include and enable animations: `Yes`

---

## 2. Custom Theme

```scss
// src/styles/material-theme.scss
@use '@angular/material' as mat;

// กำหนด typography
$app-typography: mat.define-typography-config(
  $font-family: 'Sarabun, Noto Sans Thai, sans-serif',
  $headline-1: mat.define-typography-level(96px, 96px, 300),
  $headline-2: mat.define-typography-level(60px, 60px, 300),
  $body-1: mat.define-typography-level(16px, 24px, 400),
  $body-2: mat.define-typography-level(14px, 20px, 400),
);

// สร้าง primary palette
$app-primary: mat.define-palette(mat.$indigo-palette, 600);
$app-accent: mat.define-palette(mat.$pink-palette, A200, A100, A400);
$app-warn: mat.define-palette(mat.$red-palette);

// Light theme
$app-light-theme: mat.define-light-theme((
  color: (
    primary: $app-primary,
    accent: $app-accent,
    warn: $app-warn,
  ),
  typography: $app-typography,
  density: 0,
));

// Dark theme
$app-dark-theme: mat.define-dark-theme((
  color: (
    primary: $app-primary,
    accent: $app-accent,
    warn: $app-warn,
  ),
));

// Apply themes
@include mat.all-component-themes($app-light-theme);

.dark-theme {
  @include mat.all-component-colors($app-dark-theme);
}
```

```scss
// src/styles.scss
@use './styles/material-theme' as theme;

// Google Fonts สำหรับภาษาไทย
@import url('https://fonts.googleapis.com/css2?family=Sarabun:wght@300;400;500;600;700&display=swap');

// Material Icons
@import url('https://fonts.googleapis.com/icon?family=Material+Icons');

body {
  margin: 0;
  font-family: 'Sarabun', sans-serif;
}
```

---

## 3. Theme Toggle Service

```typescript
// app/services/theme.service.ts
import { Injectable, signal, effect } from '@angular/core';
import { DOCUMENT } from '@angular/common';
import { inject } from '@angular/core';

type Theme = 'light' | 'dark';

@Injectable({ providedIn: 'root' })
export class ThemeService {
  private document = inject(DOCUMENT);
  private readonly THEME_KEY = 'app-theme';

  theme = signal<Theme>(this.getSavedTheme());

  constructor() {
    effect(() => {
      this.applyTheme(this.theme());
    });
  }

  toggleTheme() {
    this.theme.update(t => t === 'light' ? 'dark' : 'light');
  }

  setTheme(theme: Theme) {
    this.theme.set(theme);
  }

  private getSavedTheme(): Theme {
    const saved = localStorage.getItem(this.THEME_KEY) as Theme;
    if (saved) return saved;

    return window.matchMedia('(prefers-color-scheme: dark)').matches ? 'dark' : 'light';
  }

  private applyTheme(theme: Theme) {
    const body = this.document.body;
    if (theme === 'dark') {
      body.classList.add('dark-theme');
    } else {
      body.classList.remove('dark-theme');
    }
    localStorage.setItem(this.THEME_KEY, theme);
  }
}
```

---

## 4. Shared Material Module

```typescript
// app/shared/material.module.ts
import { NgModule } from '@angular/core';
import { MatButtonModule } from '@angular/material/button';
import { MatCardModule } from '@angular/material/card';
import { MatInputModule } from '@angular/material/input';
import { MatFormFieldModule } from '@angular/material/form-field';
import { MatSelectModule } from '@angular/material/select';
import { MatTableModule } from '@angular/material/table';
import { MatPaginatorModule } from '@angular/material/paginator';
import { MatSortModule } from '@angular/material/sort';
import { MatDialogModule } from '@angular/material/dialog';
import { MatSnackBarModule } from '@angular/material/snack-bar';
import { MatToolbarModule } from '@angular/material/toolbar';
import { MatSidenavModule } from '@angular/material/sidenav';
import { MatListModule } from '@angular/material/list';
import { MatIconModule } from '@angular/material/icon';
import { MatMenuModule } from '@angular/material/menu';
import { MatProgressSpinnerModule } from '@angular/material/progress-spinner';
import { MatProgressBarModule } from '@angular/material/progress-bar';
import { MatChipsModule } from '@angular/material/chips';
import { MatBadgeModule } from '@angular/material/badge';
import { MatTooltipModule } from '@angular/material/tooltip';
import { MatCheckboxModule } from '@angular/material/checkbox';
import { MatRadioModule } from '@angular/material/radio';
import { MatDatepickerModule } from '@angular/material/datepicker';
import { MatNativeDateModule } from '@angular/material/core';
import { MatAutocompleteModule } from '@angular/material/autocomplete';
import { MatStepperModule } from '@angular/material/stepper';
import { MatTabsModule } from '@angular/material/tabs';
import { MatExpansionModule } from '@angular/material/expansion';
import { MatSlideToggleModule } from '@angular/material/slide-toggle';

const MATERIAL_MODULES = [
  MatButtonModule, MatCardModule, MatInputModule, MatFormFieldModule,
  MatSelectModule, MatTableModule, MatPaginatorModule, MatSortModule,
  MatDialogModule, MatSnackBarModule, MatToolbarModule, MatSidenavModule,
  MatListModule, MatIconModule, MatMenuModule, MatProgressSpinnerModule,
  MatProgressBarModule, MatChipsModule, MatBadgeModule, MatTooltipModule,
  MatCheckboxModule, MatRadioModule, MatDatepickerModule, MatNativeDateModule,
  MatAutocompleteModule, MatStepperModule, MatTabsModule, MatExpansionModule,
  MatSlideToggleModule
];

@NgModule({
  imports: MATERIAL_MODULES,
  exports: MATERIAL_MODULES
})
export class MaterialModule {}
```

---

## 5. App Shell ด้วย Material Sidenav

```typescript
// app/components/app-shell/app-shell.component.ts
import { Component } from '@angular/core';
import { BreakpointObserver, Breakpoints } from '@angular/cdk/layout';
import { map } from 'rxjs/operators';
import { ThemeService } from '../../services/theme.service';
import { AuthService } from '../../services/auth.service';

@Component({
  selector: 'app-shell',
  template: `
    <mat-sidenav-container class="sidenav-container">

      <!-- Sidenav -->
      <mat-sidenav
        #sidenav
        [mode]="(isHandset$ | async) ? 'over' : 'side'"
        [opened]="!(isHandset$ | async)"
        class="app-sidenav"
      >
        <!-- Logo -->
        <div class="sidenav-header">
          <h2>MyApp</h2>
        </div>

        <!-- Navigation -->
        <mat-nav-list>
          <a mat-list-item routerLink="/dashboard" routerLinkActive="active">
            <mat-icon>dashboard</mat-icon>
            <span>Dashboard</span>
          </a>
          <a mat-list-item routerLink="/products" routerLinkActive="active">
            <mat-icon>inventory_2</mat-icon>
            <span>สินค้า</span>
          </a>
          <a mat-list-item routerLink="/orders" routerLinkActive="active">
            <mat-icon>receipt_long</mat-icon>
            <span>คำสั่งซื้อ</span>
          </a>
          <a mat-list-item routerLink="/reports" routerLinkActive="active">
            <mat-icon>bar_chart</mat-icon>
            <span>รายงาน</span>
          </a>

          <mat-divider></mat-divider>

          <a mat-list-item routerLink="/settings" routerLinkActive="active">
            <mat-icon>settings</mat-icon>
            <span>การตั้งค่า</span>
          </a>
        </mat-nav-list>
      </mat-sidenav>

      <!-- Main Content -->
      <mat-sidenav-content>
        <!-- Toolbar -->
        <mat-toolbar color="primary">
          <button mat-icon-button (click)="sidenav.toggle()" *ngIf="isHandset$ | async">
            <mat-icon>menu</mat-icon>
          </button>

          <span class="toolbar-spacer"></span>

          <!-- Theme Toggle -->
          <mat-slide-toggle
            [checked]="themeService.theme() === 'dark'"
            (change)="themeService.toggleTheme()"
            matTooltip="สลับ Dark Mode"
          >
            <mat-icon>dark_mode</mat-icon>
          </mat-slide-toggle>

          <!-- Notifications -->
          <button mat-icon-button [matBadge]="3" matBadgeColor="warn">
            <mat-icon>notifications</mat-icon>
          </button>

          <!-- User Menu -->
          <button mat-button [matMenuTriggerFor]="userMenu">
            <mat-icon>account_circle</mat-icon>
            {{ authService.currentUser()?.name }}
          </button>

          <mat-menu #userMenu>
            <button mat-menu-item routerLink="/profile">
              <mat-icon>person</mat-icon> โปรไฟล์
            </button>
            <button mat-menu-item (click)="authService.logout()">
              <mat-icon>logout</mat-icon> ออกจากระบบ
            </button>
          </mat-menu>
        </mat-toolbar>

        <!-- Page Content -->
        <div class="content-wrapper">
          <router-outlet></router-outlet>
        </div>
      </mat-sidenav-content>
    </mat-sidenav-container>
  `,
  styles: [`
    .sidenav-container { height: 100vh; }
    .app-sidenav { width: 250px; }
    .sidenav-header { padding: 16px; background: var(--mat-primary); color: white; }
    mat-nav-list a.active { background: rgba(0,0,0,0.1); }
    .toolbar-spacer { flex: 1; }
    .content-wrapper { padding: 24px; }
  `]
})
export class AppShellComponent {
  isHandset$ = this.breakpointObserver.observe(Breakpoints.Handset).pipe(
    map(result => result.matches)
  );

  constructor(
    private breakpointObserver: BreakpointObserver,
    public themeService: ThemeService,
    public authService: AuthService
  ) {}
}
```

---

## 6. Data Table Component

```typescript
// app/components/data-table/data-table.component.ts
import {
  Component, Input, ViewChild, OnInit, AfterViewInit
} from '@angular/core';
import { MatTableDataSource } from '@angular/material/table';
import { MatPaginator } from '@angular/material/paginator';
import { MatSort } from '@angular/material/sort';

@Component({
  selector: 'app-data-table',
  template: `
    <mat-card>
      <mat-card-header>
        <mat-card-title>{{ title }}</mat-card-title>
        <mat-card-subtitle>{{ subtitle }}</mat-card-subtitle>
      </mat-card-header>

      <mat-card-content>
        <!-- Search -->
        <mat-form-field appearance="outline" class="search-field">
          <mat-label>ค้นหา</mat-label>
          <mat-icon matPrefix>search</mat-icon>
          <input matInput (keyup)="applyFilter($event)" placeholder="พิมพ์เพื่อค้นหา...">
        </mat-form-field>

        <!-- Table -->
        <div class="table-container mat-elevation-z2">
          <table mat-table [dataSource]="dataSource" matSort>
            <ng-container *ngFor="let col of columns" [matColumnDef]="col.key">
              <th mat-header-cell *matHeaderCellDef mat-sort-header>{{ col.label }}</th>
              <td mat-cell *matCellDef="let row">
                <ng-container [ngSwitch]="col.type">
                  <span *ngSwitchCase="'currency'">{{ row[col.key] | currency:'THB':'symbol':'1.2-2' }}</span>
                  <span *ngSwitchCase="'date'">{{ row[col.key] | date:'dd/MM/yyyy' }}</span>
                  <mat-chip *ngSwitchCase="'status'" [color]="row[col.key] === 'active' ? 'primary' : 'warn'" selected>
                    {{ row[col.key] }}
                  </mat-chip>
                  <span *ngSwitchDefault>{{ row[col.key] }}</span>
                </ng-container>
              </td>
            </ng-container>

            <!-- Actions column -->
            <ng-container matColumnDef="actions">
              <th mat-header-cell *matHeaderCellDef>จัดการ</th>
              <td mat-cell *matCellDef="let row">
                <button mat-icon-button color="primary" (click)="edit.emit(row)" matTooltip="แก้ไข">
                  <mat-icon>edit</mat-icon>
                </button>
                <button mat-icon-button color="warn" (click)="delete.emit(row)" matTooltip="ลบ">
                  <mat-icon>delete</mat-icon>
                </button>
              </td>
            </ng-container>

            <tr mat-header-row *matHeaderRowDef="displayedColumns; sticky: true"></tr>
            <tr mat-row *matRowDef="let row; columns: displayedColumns;"></tr>

            <!-- No data row -->
            <tr class="mat-row" *matNoDataRow>
              <td class="mat-cell" [colSpan]="displayedColumns.length" style="text-align: center; padding: 20px;">
                ไม่พบข้อมูล
              </td>
            </tr>
          </table>
        </div>

        <mat-paginator
          [pageSizeOptions]="[10, 25, 50]"
          showFirstLastButtons
        ></mat-paginator>
      </mat-card-content>
    </mat-card>
  `
})
export class DataTableComponent<T> implements AfterViewInit {
  @Input() title = '';
  @Input() subtitle = '';
  @Input() columns: { key: string; label: string; type?: string }[] = [];
  @Input() set data(value: T[]) {
    this.dataSource.data = value;
  }

  @ViewChild(MatPaginator) paginator!: MatPaginator;
  @ViewChild(MatSort) sort!: MatSort;

  dataSource = new MatTableDataSource<T>([]);

  get displayedColumns(): string[] {
    return [...this.columns.map(c => c.key), 'actions'];
  }

  ngAfterViewInit() {
    this.dataSource.paginator = this.paginator;
    this.dataSource.sort = this.sort;
  }

  applyFilter(event: Event) {
    const filterValue = (event.target as HTMLInputElement).value;
    this.dataSource.filter = filterValue.trim().toLowerCase();

    if (this.dataSource.paginator) {
      this.dataSource.paginator.firstPage();
    }
  }
}
```

---

## 7. Dialog Service

```typescript
// app/services/dialog.service.ts
import { Injectable } from '@angular/core';
import { MatDialog } from '@angular/material/dialog';
import { MatSnackBar } from '@angular/material/snack-bar';
import { Observable } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class DialogService {
  constructor(
    private dialog: MatDialog,
    private snackBar: MatSnackBar
  ) {}

  openDialog<T>(
    component: any,
    data?: any,
    options?: { width?: string; disableClose?: boolean }
  ): Observable<T> {
    const dialogRef = this.dialog.open(component, {
      width: options?.width || '500px',
      disableClose: options?.disableClose || false,
      data
    });

    return dialogRef.afterClosed();
  }

  showSuccess(message: string) {
    this.snackBar.open(message, 'ปิด', {
      duration: 3000,
      panelClass: ['snack-success']
    });
  }

  showError(message: string) {
    this.snackBar.open(message, 'ปิด', {
      duration: 5000,
      panelClass: ['snack-error']
    });
  }

  showInfo(message: string) {
    this.snackBar.open(message, 'ตกลง', {
      duration: 4000
    });
  }
}
```

---

## สรุป

Angular Material มี components ที่ครอบคลุม:
- **Layout**: Toolbar, Sidenav, Card, Grid
- **Form**: Input, Select, Datepicker, Autocomplete, Stepper
- **Data Display**: Table, List, Chip, Badge
- **Feedback**: Dialog, Snackbar, Progress, Spinner
- **Navigation**: Menu, Tabs, Expansion Panel

การ Custom Theme ทำให้แอปพลิเคชันมีหน้าตาเป็นเอกลักษณ์ของแบรนด์
