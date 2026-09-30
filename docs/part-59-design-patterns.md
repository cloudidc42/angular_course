# Part 59: Design Patterns ใน Angular

## บทนำ

Design Patterns คือแนวทางแก้ปัญหาที่พิสูจน์แล้วในการพัฒนาซอฟต์แวร์ Angular ใช้ patterns หลายอย่างอยู่แล้ว และเราสามารถนำ patterns อื่นมาใช้เพิ่มได้

## 1. Repository Pattern

Repository Pattern แยก logic การเข้าถึงข้อมูลออกจาก business logic

```typescript
// interfaces/repository.interface.ts
export interface IRepository<T, ID = string> {
  findAll(filters?: Partial<T>): Promise<T[]>;
  findById(id: ID): Promise<T | null>;
  create(entity: Omit<T, 'id' | 'createdAt' | 'updatedAt'>): Promise<T>;
  update(id: ID, changes: Partial<T>): Promise<T>;
  delete(id: ID): Promise<void>;
  count(filters?: Partial<T>): Promise<number>;
}
```

```typescript
// models/user.model.ts
export interface User {
  id: string;
  email: string;
  name: string;
  role: 'admin' | 'user' | 'guest';
  active: boolean;
  createdAt: Date;
  updatedAt: Date;
}
```

```typescript
// repositories/user.repository.ts
import { Injectable } from '@angular/core';
import { HttpClient, HttpParams } from '@angular/common/http';
import { Observable, from } from 'rxjs';
import { map } from 'rxjs/operators';
import { IRepository } from '../interfaces/repository.interface';
import { User } from '../models/user.model';

export abstract class UserRepository implements IRepository<User> {
  abstract findAll(filters?: Partial<User>): Promise<User[]>;
  abstract findById(id: string): Promise<User | null>;
  abstract findByEmail(email: string): Promise<User | null>;
  abstract create(user: Omit<User, 'id' | 'createdAt' | 'updatedAt'>): Promise<User>;
  abstract update(id: string, changes: Partial<User>): Promise<User>;
  abstract delete(id: string): Promise<void>;
  abstract count(filters?: Partial<User>): Promise<number>;
}

@Injectable()
export class HttpUserRepository extends UserRepository {
  private readonly baseUrl = '/api/users';

  constructor(private http: HttpClient) {
    super();
  }

  async findAll(filters?: Partial<User>): Promise<User[]> {
    let params = new HttpParams();
    if (filters) {
      Object.entries(filters).forEach(([key, value]) => {
        if (value !== undefined) {
          params = params.set(key, String(value));
        }
      });
    }
    return this.http.get<User[]>(this.baseUrl, { params }).toPromise() as Promise<User[]>;
  }

  async findById(id: string): Promise<User | null> {
    return this.http.get<User>(`${this.baseUrl}/${id}`).toPromise() || null;
  }

  async findByEmail(email: string): Promise<User | null> {
    const users = await this.findAll({ email } as Partial<User>);
    return users[0] || null;
  }

  async create(user: Omit<User, 'id' | 'createdAt' | 'updatedAt'>): Promise<User> {
    return this.http.post<User>(this.baseUrl, user).toPromise() as Promise<User>;
  }

  async update(id: string, changes: Partial<User>): Promise<User> {
    return this.http.patch<User>(`${this.baseUrl}/${id}`, changes).toPromise() as Promise<User>;
  }

  async delete(id: string): Promise<void> {
    await this.http.delete(`${this.baseUrl}/${id}`).toPromise();
  }

  async count(filters?: Partial<User>): Promise<number> {
    const users = await this.findAll(filters);
    return users.length;
  }
}

// Mock Repository สำหรับ Testing
@Injectable()
export class MockUserRepository extends UserRepository {
  private users: User[] = [
    {
      id: '1',
      email: 'admin@example.com',
      name: 'Admin User',
      role: 'admin',
      active: true,
      createdAt: new Date(),
      updatedAt: new Date()
    }
  ];

  async findAll(filters?: Partial<User>): Promise<User[]> {
    if (!filters) return this.users;
    return this.users.filter(u =>
      Object.entries(filters).every(([key, val]) => (u as any)[key] === val)
    );
  }

  async findById(id: string): Promise<User | null> {
    return this.users.find(u => u.id === id) || null;
  }

  async findByEmail(email: string): Promise<User | null> {
    return this.users.find(u => u.email === email) || null;
  }

  async create(data: Omit<User, 'id' | 'createdAt' | 'updatedAt'>): Promise<User> {
    const user: User = {
      ...data,
      id: Math.random().toString(36).substr(2, 9),
      createdAt: new Date(),
      updatedAt: new Date()
    };
    this.users.push(user);
    return user;
  }

  async update(id: string, changes: Partial<User>): Promise<User> {
    const index = this.users.findIndex(u => u.id === id);
    if (index === -1) throw new Error('User not found');
    this.users[index] = { ...this.users[index], ...changes, updatedAt: new Date() };
    return this.users[index];
  }

  async delete(id: string): Promise<void> {
    this.users = this.users.filter(u => u.id !== id);
  }

  async count(): Promise<number> {
    return this.users.length;
  }
}
```

### ใช้งานผ่าน DI

```typescript
// app.config.ts
import { UserRepository, HttpUserRepository } from './repositories/user.repository';

export const appConfig = {
  providers: [
    { provide: UserRepository, useClass: HttpUserRepository }
  ]
};
```

## 2. Observer Pattern (EventEmitter/Subject)

```typescript
// event-store.service.ts
import { Injectable } from '@angular/core';
import { Subject, Observable, filter } from 'rxjs';

export interface AppEvent<T = any> {
  type: string;
  payload: T;
  timestamp: Date;
  source?: string;
}

@Injectable({ providedIn: 'root' })
export class EventStoreService {
  private events$ = new Subject<AppEvent>();

  // Publish event
  dispatch<T>(type: string, payload: T, source?: string): void {
    this.events$.next({
      type,
      payload,
      timestamp: new Date(),
      source
    });
  }

  // Subscribe to specific event type
  on<T>(type: string): Observable<AppEvent<T>> {
    return this.events$.pipe(
      filter(event => event.type === type)
    ) as Observable<AppEvent<T>>;
  }

  // Subscribe to multiple event types
  onAny(...types: string[]): Observable<AppEvent> {
    return this.events$.pipe(
      filter(event => types.includes(event.type))
    );
  }
}

// Event Types Constants
export const USER_EVENTS = {
  LOGGED_IN: 'USER_LOGGED_IN',
  LOGGED_OUT: 'USER_LOGGED_OUT',
  UPDATED: 'USER_UPDATED',
  DELETED: 'USER_DELETED'
} as const;

export const PRODUCT_EVENTS = {
  CREATED: 'PRODUCT_CREATED',
  UPDATED: 'PRODUCT_UPDATED',
  DELETED: 'PRODUCT_DELETED',
  VIEWED: 'PRODUCT_VIEWED'
} as const;
```

## 3. Factory Pattern

```typescript
// validators/validator.factory.ts
import { AbstractControl, ValidatorFn, Validators } from '@angular/forms';

export interface FieldConfig {
  type: 'text' | 'email' | 'phone' | 'url' | 'password' | 'custom';
  required?: boolean;
  minLength?: number;
  maxLength?: number;
  pattern?: RegExp;
  custom?: ValidatorFn;
}

export class ValidatorFactory {
  static create(config: FieldConfig): ValidatorFn[] {
    const validators: ValidatorFn[] = [];

    if (config.required) {
      validators.push(Validators.required);
    }

    if (config.minLength !== undefined) {
      validators.push(Validators.minLength(config.minLength));
    }

    if (config.maxLength !== undefined) {
      validators.push(Validators.maxLength(config.maxLength));
    }

    switch (config.type) {
      case 'email':
        validators.push(Validators.email);
        break;
      case 'phone':
        validators.push(Validators.pattern(/^[0-9]{10}$/));
        break;
      case 'url':
        validators.push(Validators.pattern(/^https?:\/\/.+/));
        break;
      case 'password':
        validators.push(
          Validators.minLength(8),
          ValidatorFactory.strongPassword()
        );
        break;
      case 'custom':
        if (config.custom) {
          validators.push(config.custom);
        }
        break;
    }

    if (config.pattern) {
      validators.push(Validators.pattern(config.pattern));
    }

    return validators;
  }

  static strongPassword(): ValidatorFn {
    return (control: AbstractControl) => {
      const value = control.value as string;
      if (!value) return null;

      const hasUpperCase = /[A-Z]/.test(value);
      const hasLowerCase = /[a-z]/.test(value);
      const hasNumber = /\d/.test(value);
      const hasSpecial = /[@$!%*?&]/.test(value);

      const valid = hasUpperCase && hasLowerCase && hasNumber && hasSpecial;
      return valid ? null : {
        weakPassword: {
          missing: {
            uppercase: !hasUpperCase,
            lowercase: !hasLowerCase,
            number: !hasNumber,
            special: !hasSpecial
          }
        }
      };
    };
  }
}
```

## 4. Strategy Pattern

```typescript
// export/export-strategy.interface.ts
export interface ExportStrategy {
  export(data: any[], filename: string): void;
  mimeType: string;
  extension: string;
}
```

```typescript
// export/csv-export.strategy.ts
import { ExportStrategy } from './export-strategy.interface';

export class CsvExportStrategy implements ExportStrategy {
  mimeType = 'text/csv';
  extension = 'csv';

  export(data: any[], filename: string): void {
    if (!data.length) return;

    const headers = Object.keys(data[0]);
    const csvContent = [
      headers.join(','),
      ...data.map(row =>
        headers.map(header => {
          const value = row[header];
          return typeof value === 'string' ? `"${value.replace(/"/g, '""')}"` : value;
        }).join(',')
      )
    ].join('\n');

    this.download(csvContent, `${filename}.csv`, this.mimeType);
  }

  private download(content: string, filename: string, mimeType: string): void {
    const blob = new Blob(['﻿' + content], { type: `${mimeType};charset=utf-8` });
    const url = URL.createObjectURL(blob);
    const link = document.createElement('a');
    link.href = url;
    link.download = filename;
    link.click();
    URL.revokeObjectURL(url);
  }
}
```

```typescript
// export/json-export.strategy.ts
import { ExportStrategy } from './export-strategy.interface';

export class JsonExportStrategy implements ExportStrategy {
  mimeType = 'application/json';
  extension = 'json';

  export(data: any[], filename: string): void {
    const content = JSON.stringify(data, null, 2);
    const blob = new Blob([content], { type: this.mimeType });
    const url = URL.createObjectURL(blob);
    const link = document.createElement('a');
    link.href = url;
    link.download = `${filename}.json`;
    link.click();
    URL.revokeObjectURL(url);
  }
}
```

```typescript
// export/export.service.ts
import { Injectable } from '@angular/core';
import { ExportStrategy } from './export-strategy.interface';
import { CsvExportStrategy } from './csv-export.strategy';
import { JsonExportStrategy } from './json-export.strategy';

type ExportFormat = 'csv' | 'json' | 'excel';

@Injectable({ providedIn: 'root' })
export class ExportService {
  private strategies = new Map<ExportFormat, ExportStrategy>([
    ['csv', new CsvExportStrategy()],
    ['json', new JsonExportStrategy()]
  ]);

  export(data: any[], filename: string, format: ExportFormat): void {
    const strategy = this.strategies.get(format);
    if (!strategy) {
      throw new Error(`Export format "${format}" not supported`);
    }
    strategy.export(data, filename);
  }

  registerStrategy(format: ExportFormat, strategy: ExportStrategy): void {
    this.strategies.set(format, strategy);
  }

  getSupportedFormats(): ExportFormat[] {
    return Array.from(this.strategies.keys());
  }
}
```

### ใช้งาน Strategy Pattern

```typescript
// data-table.component.ts
import { Component } from '@angular/core';
import { CommonModule } from '@angular/common';
import { ExportService } from './export/export.service';

@Component({
  selector: 'app-data-table',
  standalone: true,
  imports: [CommonModule],
  template: `
    <div class="table-actions">
      <button
        *ngFor="let format of exportFormats"
        (click)="exportData(format)"
      >
        Export {{ format.toUpperCase() }}
      </button>
    </div>
    <table>
      <thead>
        <tr>
          <th *ngFor="let col of columns">{{ col }}</th>
        </tr>
      </thead>
      <tbody>
        <tr *ngFor="let row of data">
          <td *ngFor="let col of columns">{{ row[col] }}</td>
        </tr>
      </tbody>
    </table>
  `
})
export class DataTableComponent {
  data = [
    { ชื่อ: 'สมชาย', อายุ: 30, เมือง: 'กรุงเทพ' },
    { ชื่อ: 'สมหญิง', อายุ: 25, เมือง: 'เชียงใหม่' }
  ];
  columns = ['ชื่อ', 'อายุ', 'เมือง'];
  exportFormats: Array<'csv' | 'json'> = ['csv', 'json'];

  constructor(private exportService: ExportService) {}

  exportData(format: 'csv' | 'json'): void {
    this.exportService.export(this.data, 'รายชื่อ', format);
  }
}
```

## 5. Decorator Pattern

```typescript
// decorators/log.decorator.ts
export function Log(target: any, propertyKey: string, descriptor: PropertyDescriptor) {
  const originalMethod = descriptor.value;

  descriptor.value = function(...args: any[]) {
    console.log(`[${target.constructor.name}] ${propertyKey} called with:`, args);
    const result = originalMethod.apply(this, args);
    
    if (result instanceof Promise) {
      return result.then(res => {
        console.log(`[${target.constructor.name}] ${propertyKey} returned:`, res);
        return res;
      });
    }
    
    console.log(`[${target.constructor.name}] ${propertyKey} returned:`, result);
    return result;
  };

  return descriptor;
}

export function Memoize(target: any, propertyKey: string, descriptor: PropertyDescriptor) {
  const cache = new Map<string, any>();
  const originalMethod = descriptor.value;

  descriptor.value = function(...args: any[]) {
    const key = JSON.stringify(args);
    
    if (cache.has(key)) {
      console.log(`[Memoize] Cache HIT for ${propertyKey}`);
      return cache.get(key);
    }

    const result = originalMethod.apply(this, args);
    cache.set(key, result);
    return result;
  };

  return descriptor;
}

// การใช้งาน
class CalculationService {
  @Log
  @Memoize
  calculateDiscount(price: number, percent: number): number {
    return price * (1 - percent / 100);
  }
}
```

## สรุป Design Patterns

| Pattern | ประโยชน์ | ตัวอย่างใน Angular |
|---------|---------|-----------------|
| Repository | แยก data access | Service + HTTP |
| Observer | React ต่อ events | EventEmitter, Subject |
| Factory | สร้าง objects อย่างยืดหยุ่น | ValidatorFactory |
| Strategy | เปลี่ยน algorithm ได้ | Export formats |
| Decorator | เพิ่ม behavior โดยไม่แก้ code | @Log, @Cache |
| Singleton | instance เดียว | Injectable root |
| Facade | Single interface | Service layer |

Design Patterns ช่วยให้โค้ดมีโครงสร้างที่ดี อ่านง่าย และแก้ไขได้ง่ายในอนาคต
