# Part 60: SOLID Principles ใน Angular

## บทนำ

SOLID เป็นหลักการ 5 ข้อที่ช่วยให้โค้ดมีคุณภาพสูง ง่ายต่อการบำรุงรักษา และขยายได้

## 1. Single Responsibility Principle (SRP)

**หลักการ**: แต่ละ class/service ควรมีความรับผิดชอบเพียงอย่างเดียว

### ❌ ไม่ดี

```typescript
// user.service.ts - ทำหลายอย่างเกินไป
@Injectable({ providedIn: 'root' })
export class UserService {
  constructor(private http: HttpClient) {}

  // HTTP operations
  getUsers(): Observable<User[]> {
    return this.http.get<User[]>('/api/users');
  }

  // Authentication
  login(email: string, password: string): Observable<{ token: string }> {
    return this.http.post<{ token: string }>('/api/login', { email, password });
  }

  // Validation
  validateEmail(email: string): boolean {
    return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);
  }

  // Formatting
  formatUserName(user: User): string {
    return `${user.firstName} ${user.lastName}`;
  }

  // Local Storage
  saveToStorage(key: string, data: any): void {
    localStorage.setItem(key, JSON.stringify(data));
  }

  // Email notification
  sendWelcomeEmail(user: User): Observable<void> {
    return this.http.post<void>('/api/emails/welcome', { userId: user.id });
  }
}
```

### ✅ ดี

```typescript
// users-api.service.ts - เฉพาะ HTTP operations
@Injectable({ providedIn: 'root' })
export class UsersApiService {
  private readonly baseUrl = '/api/users';
  
  constructor(private http: HttpClient) {}

  getAll(): Observable<User[]> {
    return this.http.get<User[]>(this.baseUrl);
  }

  getById(id: string): Observable<User> {
    return this.http.get<User>(`${this.baseUrl}/${id}`);
  }

  create(user: CreateUserDto): Observable<User> {
    return this.http.post<User>(this.baseUrl, user);
  }

  update(id: string, changes: UpdateUserDto): Observable<User> {
    return this.http.patch<User>(`${this.baseUrl}/${id}`, changes);
  }

  delete(id: string): Observable<void> {
    return this.http.delete<void>(`${this.baseUrl}/${id}`);
  }
}

// auth.service.ts - เฉพาะ authentication
@Injectable({ providedIn: 'root' })
export class AuthService {
  constructor(private http: HttpClient) {}

  login(credentials: LoginDto): Observable<AuthResponse> {
    return this.http.post<AuthResponse>('/api/auth/login', credentials);
  }

  logout(): Observable<void> {
    return this.http.post<void>('/api/auth/logout', {});
  }

  refreshToken(): Observable<AuthResponse> {
    return this.http.post<AuthResponse>('/api/auth/refresh', {});
  }
}

// user-validator.service.ts - เฉพาะ validation
@Injectable({ providedIn: 'root' })
export class UserValidatorService {
  validateEmail(email: string): boolean {
    return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);
  }

  validatePhone(phone: string): boolean {
    return /^[0-9]{10}$/.test(phone);
  }

  validatePasswordStrength(password: string): {
    score: number;
    feedback: string[];
  } {
    const feedback: string[] = [];
    let score = 0;

    if (password.length >= 8) score++;
    else feedback.push('ต้องมีอย่างน้อย 8 ตัวอักษร');

    if (/[A-Z]/.test(password)) score++;
    else feedback.push('ต้องมีตัวพิมพ์ใหญ่');

    if (/[a-z]/.test(password)) score++;
    else feedback.push('ต้องมีตัวพิมพ์เล็ก');

    if (/\d/.test(password)) score++;
    else feedback.push('ต้องมีตัวเลข');

    if (/[@$!%*?&]/.test(password)) score++;
    else feedback.push('ต้องมีอักขระพิเศษ (@$!%*?&)');

    return { score, feedback };
  }
}

// user-formatter.service.ts - เฉพาะ formatting
@Injectable({ providedIn: 'root' })
export class UserFormatterService {
  formatFullName(user: User): string {
    return [user.firstName, user.lastName].filter(Boolean).join(' ');
  }

  formatAvatar(user: User): string {
    return user.avatar || `https://ui-avatars.com/api/?name=${encodeURIComponent(this.formatFullName(user))}`;
  }

  formatRole(role: string): string {
    const roles: Record<string, string> = {
      admin: 'ผู้ดูแลระบบ',
      user: 'ผู้ใช้งาน',
      guest: 'ผู้เยี่ยมชม'
    };
    return roles[role] || role;
  }
}
```

## 2. Open/Closed Principle (OCP)

**หลักการ**: เปิดรับการขยาย (extension) แต่ปิดต่อการแก้ไข (modification)

```typescript
// notification/notification.interface.ts
export interface NotificationChannel {
  send(message: NotificationMessage): Promise<void>;
  getName(): string;
}

export interface NotificationMessage {
  to: string;
  subject: string;
  body: string;
  template?: string;
  data?: Record<string, any>;
}
```

```typescript
// notification/email.channel.ts
@Injectable()
export class EmailNotificationChannel implements NotificationChannel {
  constructor(private http: HttpClient) {}
  
  getName(): string { return 'email'; }

  async send(message: NotificationMessage): Promise<void> {
    await this.http.post('/api/notifications/email', message).toPromise();
  }
}

// notification/sms.channel.ts
@Injectable()
export class SmsNotificationChannel implements NotificationChannel {
  constructor(private http: HttpClient) {}
  
  getName(): string { return 'sms'; }

  async send(message: NotificationMessage): Promise<void> {
    await this.http.post('/api/notifications/sms', message).toPromise();
  }
}

// notification/push.channel.ts (เพิ่มใหม่โดยไม่แก้ service)
@Injectable()
export class PushNotificationChannel implements NotificationChannel {
  constructor(private http: HttpClient) {}
  
  getName(): string { return 'push'; }

  async send(message: NotificationMessage): Promise<void> {
    await this.http.post('/api/notifications/push', message).toPromise();
  }
}

// notification/notification.service.ts - ไม่ต้องแก้ไขเมื่อเพิ่ม channel ใหม่
@Injectable({ providedIn: 'root' })
export class NotificationService {
  private channels = new Map<string, NotificationChannel>();

  register(channel: NotificationChannel): void {
    this.channels.set(channel.getName(), channel);
  }

  async send(channelName: string, message: NotificationMessage): Promise<void> {
    const channel = this.channels.get(channelName);
    if (!channel) {
      throw new Error(`Notification channel "${channelName}" not found`);
    }
    await channel.send(message);
  }

  async sendToAll(message: NotificationMessage): Promise<void> {
    const promises = Array.from(this.channels.values()).map(ch => ch.send(message));
    await Promise.allSettled(promises);
  }
}
```

## 3. Liskov Substitution Principle (LSP)

**หลักการ**: Subclass ต้องสามารถแทนที่ Superclass ได้โดยไม่ทำให้พฤติกรรมเปลี่ยน

```typescript
// shapes/shape.abstract.ts
export abstract class Shape {
  abstract getArea(): number;
  abstract getPerimeter(): number;
  abstract toString(): string;

  // Common behavior ที่ทุก shape ต้องทำได้
  describe(): string {
    return `${this.toString()}: พื้นที่ = ${this.getArea().toFixed(2)}, เส้นรอบวง = ${this.getPerimeter().toFixed(2)}`;
  }
}

// shapes/circle.ts
export class Circle extends Shape {
  constructor(private radius: number) {
    super();
  }

  getArea(): number {
    return Math.PI * this.radius ** 2;
  }

  getPerimeter(): number {
    return 2 * Math.PI * this.radius;
  }

  toString(): string {
    return `วงกลม (r=${this.radius})`;
  }
}

// shapes/rectangle.ts
export class Rectangle extends Shape {
  constructor(
    protected width: number,
    protected height: number
  ) {
    super();
  }

  getArea(): number {
    return this.width * this.height;
  }

  getPerimeter(): number {
    return 2 * (this.width + this.height);
  }

  toString(): string {
    return `สี่เหลี่ยม (${this.width}x${this.height})`;
  }
}

// shapes/square.ts (LSP: Square IS-A Rectangle ที่ถูกต้อง)
export class Square extends Rectangle {
  constructor(side: number) {
    super(side, side);
  }

  toString(): string {
    return `จัตุรัส (${this.width})`;
  }
}

// การใช้งาน - ใช้ Shape แทนกันได้
@Component({
  selector: 'app-shapes',
  standalone: true,
  imports: [CommonModule],
  template: `
    <div *ngFor="let shape of shapes">
      <p>{{ shape.describe() }}</p>
    </div>
  `
})
export class ShapesComponent {
  // ใช้ Shape[] แทนที่จะระบุ type เฉพาะ
  shapes: Shape[] = [
    new Circle(5),
    new Rectangle(4, 6),
    new Square(3)
  ];
}
```

## 4. Interface Segregation Principle (ISP)

**หลักการ**: Interface ควรเล็กและเฉพาะเจาะจง ไม่ควรบังคับให้ implement method ที่ไม่ต้องการ

```typescript
// ❌ ไม่ดี - Interface ใหญ่เกินไป
interface IFullDataService {
  getData(): Observable<any[]>;
  createItem(data: any): Observable<any>;
  updateItem(id: string, data: any): Observable<any>;
  deleteItem(id: string): Observable<void>;
  exportToCsv(): void;
  exportToExcel(): void;
  print(): void;
  sendEmail(to: string): Observable<void>;
  generateReport(): Observable<Blob>;
}
```

```typescript
// ✅ ดี - แบ่ง Interface ตามความรับผิดชอบ

// interfaces/readable.interface.ts
export interface IReadable<T> {
  getAll(): Observable<T[]>;
  getById(id: string): Observable<T>;
}

// interfaces/writable.interface.ts
export interface IWritable<T, CreateDto = Partial<T>, UpdateDto = Partial<T>> {
  create(data: CreateDto): Observable<T>;
  update(id: string, data: UpdateDto): Observable<T>;
  delete(id: string): Observable<void>;
}

// interfaces/exportable.interface.ts
export interface IExportable {
  exportToCsv(filename: string): void;
  exportToJson(filename: string): void;
}

// interfaces/printable.interface.ts
export interface IPrintable {
  print(): void;
  generatePdf(): Observable<Blob>;
}

// services/products.service.ts - implements only what it needs
@Injectable({ providedIn: 'root' })
export class ProductsService implements IReadable<Product>, IWritable<Product>, IExportable {
  constructor(
    private http: HttpClient,
    private exportService: ExportService
  ) {}

  getAll(): Observable<Product[]> {
    return this.http.get<Product[]>('/api/products');
  }

  getById(id: string): Observable<Product> {
    return this.http.get<Product>(`/api/products/${id}`);
  }

  create(data: Omit<Product, 'id'>): Observable<Product> {
    return this.http.post<Product>('/api/products', data);
  }

  update(id: string, data: Partial<Product>): Observable<Product> {
    return this.http.patch<Product>(`/api/products/${id}`, data);
  }

  delete(id: string): Observable<void> {
    return this.http.delete<void>(`/api/products/${id}`);
  }

  exportToCsv(filename: string): void {
    this.getAll().subscribe(products => {
      this.exportService.export(products, filename, 'csv');
    });
  }

  exportToJson(filename: string): void {
    this.getAll().subscribe(products => {
      this.exportService.export(products, filename, 'json');
    });
  }
}

// services/reports.service.ts - implements different interfaces
@Injectable({ providedIn: 'root' })
export class ReportsService implements IReadable<Report>, IPrintable {
  constructor(private http: HttpClient) {}

  getAll(): Observable<Report[]> {
    return this.http.get<Report[]>('/api/reports');
  }

  getById(id: string): Observable<Report> {
    return this.http.get<Report>(`/api/reports/${id}`);
  }

  print(): void {
    window.print();
  }

  generatePdf(): Observable<Blob> {
    return this.http.get('/api/reports/export/pdf', { responseType: 'blob' });
  }
}
```

## 5. Dependency Inversion Principle (DIP)

**หลักการ**: High-level module ไม่ควรขึ้นต่อ Low-level module ทั้งคู่ควรขึ้นต่อ Abstraction

```typescript
// abstractions/logger.interface.ts
export abstract class Logger {
  abstract log(message: string, context?: any): void;
  abstract warn(message: string, context?: any): void;
  abstract error(message: string, error?: Error): void;
}

// loggers/console.logger.ts
@Injectable()
export class ConsoleLogger extends Logger {
  log(message: string, context?: any): void {
    console.log(`[INFO] ${message}`, context || '');
  }

  warn(message: string, context?: any): void {
    console.warn(`[WARN] ${message}`, context || '');
  }

  error(message: string, error?: Error): void {
    console.error(`[ERROR] ${message}`, error);
  }
}

// loggers/remote.logger.ts
@Injectable()
export class RemoteLogger extends Logger {
  constructor(private http: HttpClient) {
    super();
  }

  log(message: string, context?: any): void {
    this.sendLog('info', message, context);
  }

  warn(message: string, context?: any): void {
    this.sendLog('warn', message, context);
  }

  error(message: string, error?: Error): void {
    this.sendLog('error', message, { error: error?.message, stack: error?.stack });
  }

  private sendLog(level: string, message: string, context?: any): void {
    this.http.post('/api/logs', { level, message, context, timestamp: new Date() })
      .subscribe({ error: () => console.error('Failed to send log') });
  }
}

// services/order.service.ts - ขึ้นต่อ Abstract Logger ไม่ใช่ Concrete
@Injectable({ providedIn: 'root' })
export class OrderService {
  // ขึ้นต่อ abstract Logger ไม่ใช่ ConsoleLogger หรือ RemoteLogger
  constructor(
    private http: HttpClient,
    private logger: Logger // DIP!
  ) {}

  createOrder(order: CreateOrderDto): Observable<Order> {
    this.logger.log('Creating order', { order });
    
    return this.http.post<Order>('/api/orders', order).pipe(
      tap(created => {
        this.logger.log('Order created successfully', { id: created.id });
      })
    );
  }

  cancelOrder(orderId: string): Observable<void> {
    this.logger.warn('Cancelling order', { orderId });
    return this.http.delete<void>(`/api/orders/${orderId}`);
  }
}

// app.config.ts - เลือก implementation ที่ DI level
export const appConfig: ApplicationConfig = {
  providers: [
    {
      provide: Logger,
      useClass: environment.production ? RemoteLogger : ConsoleLogger
    }
  ]
};
```

## สรุป SOLID ใน Angular

| หลักการ | คำอธิบาย | ตัวอย่างใน Angular |
|--------|---------|-----------------|
| SRP | 1 class = 1 ความรับผิดชอบ | แยก service ตาม concern |
| OCP | ขยายได้ ไม่ต้องแก้ | Strategy/Plugin pattern |
| LSP | Subclass แทน Superclass ได้ | Inheritance ที่ถูกต้อง |
| ISP | Interface เล็กเฉพาะเจาะจง | แบ่ง interface ตาม capability |
| DIP | ขึ้นต่อ abstraction ไม่ใช่ concrete | Abstract class + DI |

SOLID ช่วยให้โค้ด Angular ง่ายต่อการ test, maintain และขยายในอนาคต
