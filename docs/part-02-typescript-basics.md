# Part 02 — TypeScript พื้นฐานสำหรับ Angular

## เนื้อหาในบทนี้
1. TypeScript คืออะไรและทำไมต้องใช้
2. Types พื้นฐาน
3. Interfaces
4. Classes และ OOP
5. Generics
6. Enums
7. Decorators
8. Type Utilities
9. Type Guards
10. Workshop: TypeScript ใน Angular Context

---

## 1. TypeScript คืออะไรและทำไมต้องใช้

**TypeScript** คือ Superset ของ JavaScript ที่เพิ่ม Static Typing เข้ามา พัฒนาโดย Microsoft

```
JavaScript → TypeScript
    +
Static Types + Compile-time checks + Better IDE support
```

### ประโยชน์ของ TypeScript ใน Angular

```typescript
// JavaScript — ไม่รู้ว่า user.name จะ undefined หรือเปล่า
function greet(user) {
  return 'Hello ' + user.name;
}

// TypeScript — รู้ทันทีว่า user ต้องมี property name
interface User {
  name: string;
  email: string;
}

function greet(user: User): string {
  return `Hello ${user.name}`;
}

greet({ name: 'สมชาย' });        // Error! ขาด email
greet({ name: 'สมชาย', email: 'somchai@gmail.com' }); // OK
```

---

## 2. Types พื้นฐาน

### Primitive Types

```typescript
// Boolean
let isActive: boolean = true;
let isLoading: boolean = false;

// Number
let age: number = 25;
let price: number = 99.99;
let hex: number = 0xf00d;

// String
let firstName: string = 'สมชาย';
let greeting: string = `สวัสดี ${firstName}`;

// Null และ Undefined
let nothing: null = null;
let notAssigned: undefined = undefined;

// Any — หลีกเลี่ยงการใช้!
let anything: any = 'hello';
anything = 42;       // OK แต่ไม่แนะนำ

// Unknown — ดีกว่า any
let unknownValue: unknown = 'hello';
// ต้อง type check ก่อนใช้
if (typeof unknownValue === 'string') {
  console.log(unknownValue.toUpperCase()); // OK
}

// Never — function ที่ไม่มีวัน return
function throwError(message: string): never {
  throw new Error(message);
}

// Void — function ที่ไม่ return ค่า
function logMessage(msg: string): void {
  console.log(msg);
}
```

### Array Types

```typescript
// Array แบบที่ 1
let numbers: number[] = [1, 2, 3, 4, 5];
let names: string[] = ['Alice', 'Bob', 'สมชาย'];

// Array แบบที่ 2 (Generic)
let scores: Array<number> = [95, 87, 92];
let users: Array<string> = ['admin', 'user'];

// Readonly Array
const readonlyNums: ReadonlyArray<number> = [1, 2, 3];
// readonlyNums.push(4);  // Error!

// Tuple — Array ที่รู้จำนวนและ type ของแต่ละตำแหน่ง
let point: [number, number] = [10, 20];
let person: [string, number, boolean] = ['Alice', 25, true];

// Named Tuple (TypeScript 4.0+)
type UserTuple = [name: string, age: number, isActive: boolean];
const user: UserTuple = ['สมชาย', 30, true];
```

### Union Types

```typescript
// Union — ค่าสามารถเป็น type ใดก็ได้ใน union
let id: string | number;
id = 'abc-123';  // OK
id = 42;         // OK
id = true;       // Error!

// Union กับ function
function formatId(id: string | number): string {
  if (typeof id === 'string') {
    return id.toUpperCase();
  }
  return id.toString();
}

// Discriminated Union — Pattern ที่นิยมใช้ใน Angular
type ApiResponse<T> =
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; error: string };

// ใช้งาน
function handleResponse<T>(response: ApiResponse<T>) {
  switch (response.status) {
    case 'loading':
      return 'กำลังโหลด...';
    case 'success':
      return response.data;    // TypeScript รู้ว่ามี data
    case 'error':
      return response.error;   // TypeScript รู้ว่ามี error
  }
}
```

### Intersection Types

```typescript
type Timestamped = {
  createdAt: Date;
  updatedAt: Date;
};

type Named = {
  firstName: string;
  lastName: string;
};

// รวม types เข้าด้วยกัน
type NamedTimestamped = Named & Timestamped;

const record: NamedTimestamped = {
  firstName: 'สมชาย',
  lastName: 'ใจดี',
  createdAt: new Date(),
  updatedAt: new Date()
};
```

### Literal Types

```typescript
// String Literal
type Direction = 'north' | 'south' | 'east' | 'west';
let dir: Direction = 'north';  // OK
// dir = 'up';  // Error!

// Number Literal
type DiceValue = 1 | 2 | 3 | 4 | 5 | 6;
function rollDice(): DiceValue {
  return Math.floor(Math.random() * 6 + 1) as DiceValue;
}

// Boolean Literal (ไม่ค่อยใช้แต่มี)
type IsActive = true | false;

// Template Literal Types (TypeScript 4.1+)
type EventName = `on${Capitalize<string>}`;
type Greeting = `Hello, ${string}!`;
```

---

## 3. Interfaces

### การสร้าง Interface

```typescript
// Interface พื้นฐาน
interface User {
  id: number;
  username: string;
  email: string;
  password: string;
}

// Interface กับ Optional Properties (?)
interface UserProfile {
  id: number;
  username: string;
  firstName?: string;    // optional
  lastName?: string;     // optional
  avatar?: string;       // optional
  bio?: string;          // optional
}

// Interface กับ Readonly Properties
interface Config {
  readonly apiUrl: string;
  readonly appName: string;
  maxRetries: number;  // เปลี่ยนได้
}

const config: Config = {
  apiUrl: 'https://api.example.com',
  appName: 'My App',
  maxRetries: 3
};
// config.apiUrl = 'other';  // Error! readonly
config.maxRetries = 5;       // OK

// Interface กับ Methods
interface Animal {
  name: string;
  speak(): string;
  move(distance: number): void;
}

// Implement Interface
class Dog implements Animal {
  constructor(public name: string) {}
  
  speak(): string {
    return `${this.name} says: Woof!`;
  }
  
  move(distance: number): void {
    console.log(`${this.name} moved ${distance}m`);
  }
}
```

### Interface ที่ใช้บ่อยใน Angular

```typescript
// Model สำหรับ API Response
interface ApiResponse<T> {
  data: T;
  message: string;
  statusCode: number;
  success: boolean;
}

interface PaginatedResponse<T> {
  data: T[];
  total: number;
  page: number;
  limit: number;
  totalPages: number;
}

// User Models
interface User {
  id: number;
  username: string;
  email: string;
  role: UserRole;
  isActive: boolean;
  createdAt: string;
  updatedAt: string;
}

interface CreateUserDto {
  username: string;
  email: string;
  password: string;
  role?: UserRole;
}

interface UpdateUserDto {
  username?: string;
  email?: string;
  firstName?: string;
  lastName?: string;
}

// Navigation
interface NavItem {
  label: string;
  route: string;
  icon?: string;
  children?: NavItem[];
  roles?: string[];
}

// Form
interface LoginForm {
  email: string;
  password: string;
  rememberMe: boolean;
}
```

### Interface Extending

```typescript
interface BaseEntity {
  id: number;
  createdAt: Date;
  updatedAt: Date;
}

interface Product extends BaseEntity {
  name: string;
  price: number;
  category: string;
  stock: number;
}

interface DigitalProduct extends Product {
  downloadUrl: string;
  fileSize: number;
  format: string;
}

// Extend หลาย interfaces
interface Admin extends User, Auditable {
  permissions: string[];
}

interface Auditable {
  createdBy: string;
  updatedBy: string;
}
```

---

## 4. Classes และ OOP

### Class พื้นฐาน

```typescript
class Person {
  // Properties
  private _id: number;
  protected name: string;
  public email: string;
  readonly createdAt: Date;

  // Constructor
  constructor(id: number, name: string, email: string) {
    this._id = id;
    this.name = name;
    this.email = email;
    this.createdAt = new Date();
  }

  // Getter
  get id(): number {
    return this._id;
  }

  // Setter
  set id(value: number) {
    if (value > 0) {
      this._id = value;
    }
  }

  // Method
  greet(): string {
    return `สวัสดี ฉันชื่อ ${this.name}`;
  }

  toString(): string {
    return `Person(${this._id}, ${this.name})`;
  }
}

const person = new Person(1, 'สมชาย', 'somchai@gmail.com');
console.log(person.greet());  // สวัสดี ฉันชื่อ สมชาย
```

### Constructor Shorthand

```typescript
// วิธีย่อ — สร้าง property และ assign ในบรรทัดเดียว
class User {
  constructor(
    public readonly id: number,
    public username: string,
    public email: string,
    private password: string,
    protected role: string = 'user'  // default value
  ) {}
  
  getInfo(): string {
    return `${this.username} (${this.email}) - ${this.role}`;
  }
}
```

### Inheritance

```typescript
class Employee extends Person {
  department: string;
  salary: number;

  constructor(
    id: number,
    name: string,
    email: string,
    department: string,
    salary: number
  ) {
    super(id, name, email);  // เรียก constructor ของ parent
    this.department = department;
    this.salary = salary;
  }

  // Override method
  override greet(): string {
    return `${super.greet()} ฉันทำงานที่ ${this.department}`;
  }
  
  getAnnualSalary(): number {
    return this.salary * 12;
  }
}

const emp = new Employee(1, 'สมชาย', 'somchai@co.th', 'IT', 50000);
console.log(emp.greet());
// สวัสดี ฉันชื่อ สมชาย ฉันทำงานที่ IT
```

### Abstract Classes

```typescript
abstract class Shape {
  abstract name: string;
  
  abstract area(): number;
  abstract perimeter(): number;
  
  // Concrete method (ใช้ร่วมกันได้)
  describe(): string {
    return `${this.name}: พื้นที่=${this.area()}, เส้นรอบวง=${this.perimeter()}`;
  }
}

class Circle extends Shape {
  name = 'วงกลม';
  
  constructor(private radius: number) {
    super();
  }
  
  area(): number {
    return Math.PI * this.radius ** 2;
  }
  
  perimeter(): number {
    return 2 * Math.PI * this.radius;
  }
}

class Rectangle extends Shape {
  name = 'สี่เหลี่ยม';
  
  constructor(private width: number, private height: number) {
    super();
  }
  
  area(): number {
    return this.width * this.height;
  }
  
  perimeter(): number {
    return 2 * (this.width + this.height);
  }
}

const circle = new Circle(5);
const rect = new Rectangle(4, 6);

console.log(circle.describe());  // วงกลม: พื้นที่=78.54, เส้นรอบวง=31.42
console.log(rect.describe());    // สี่เหลี่ยม: พื้นที่=24, เส้นรอบวง=20
```

### Static Members

```typescript
class Counter {
  private static instance: Counter;
  private static count: number = 0;
  
  private constructor() {}  // Singleton Pattern
  
  static getInstance(): Counter {
    if (!Counter.instance) {
      Counter.instance = new Counter();
    }
    return Counter.instance;
  }
  
  increment(): void {
    Counter.count++;
  }
  
  getCount(): number {
    return Counter.count;
  }
  
  static reset(): void {
    Counter.count = 0;
  }
}

const counter1 = Counter.getInstance();
const counter2 = Counter.getInstance();
console.log(counter1 === counter2);  // true (same instance)
```

---

## 5. Generics

### Generic Functions

```typescript
// ไม่ใช้ Generics — ต้องเขียนซ้ำ
function getFirstNumber(arr: number[]): number {
  return arr[0];
}

function getFirstString(arr: string[]): string {
  return arr[0];
}

// ใช้ Generics — เขียนครั้งเดียว
function getFirst<T>(arr: T[]): T {
  return arr[0];
}

console.log(getFirst([1, 2, 3]));        // 1
console.log(getFirst(['a', 'b', 'c']));  // 'a'
console.log(getFirst([{id: 1}, {id: 2}]));  // {id: 1}

// Generic function หลาย type parameters
function pair<K, V>(key: K, value: V): [K, V] {
  return [key, value];
}

const p1 = pair('name', 'สมชาย');
const p2 = pair(1, true);
```

### Generic Interfaces และ Classes

```typescript
// Generic Interface
interface Repository<T> {
  findById(id: number): Promise<T>;
  findAll(): Promise<T[]>;
  create(entity: T): Promise<T>;
  update(id: number, entity: Partial<T>): Promise<T>;
  delete(id: number): Promise<void>;
}

// Generic Class
class GenericRepository<T extends { id: number }> implements Repository<T> {
  private items: T[] = [];
  
  async findById(id: number): Promise<T> {
    const item = this.items.find(i => i.id === id);
    if (!item) throw new Error(`Item ${id} not found`);
    return item;
  }
  
  async findAll(): Promise<T[]> {
    return [...this.items];
  }
  
  async create(entity: T): Promise<T> {
    this.items.push(entity);
    return entity;
  }
  
  async update(id: number, data: Partial<T>): Promise<T> {
    const index = this.items.findIndex(i => i.id === id);
    if (index === -1) throw new Error(`Item ${id} not found`);
    this.items[index] = { ...this.items[index], ...data };
    return this.items[index];
  }
  
  async delete(id: number): Promise<void> {
    this.items = this.items.filter(i => i.id !== id);
  }
}

// ใช้งาน
interface Product {
  id: number;
  name: string;
  price: number;
}

const productRepo = new GenericRepository<Product>();
await productRepo.create({ id: 1, name: 'สินค้า A', price: 100 });
```

### Generic Constraints

```typescript
// ต้องมี property ที่กำหนด
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

const user = { id: 1, name: 'สมชาย', email: 'test@test.com' };
console.log(getProperty(user, 'name'));   // 'สมชาย'
console.log(getProperty(user, 'id'));     // 1
// getProperty(user, 'age');  // Error! 'age' ไม่มีใน user

// Generic กับ extends interface
interface HasLength {
  length: number;
}

function logLength<T extends HasLength>(item: T): void {
  console.log(`Length: ${item.length}`);
}

logLength('hello');           // Length: 5
logLength([1, 2, 3]);         // Length: 3
logLength({ length: 10 });    // Length: 10
// logLength(42);             // Error! number ไม่มี .length
```

---

## 6. Enums

### Numeric Enums

```typescript
enum Direction {
  North,    // 0
  South,    // 1
  East,     // 2
  West      // 3
}

enum HttpStatus {
  OK = 200,
  Created = 201,
  BadRequest = 400,
  Unauthorized = 401,
  Forbidden = 403,
  NotFound = 404,
  InternalServerError = 500
}

console.log(HttpStatus.OK);           // 200
console.log(HttpStatus[200]);         // 'OK' (reverse mapping)
```

### String Enums (แนะนำให้ใช้ใน Angular)

```typescript
enum UserRole {
  Admin = 'ADMIN',
  User = 'USER',
  Moderator = 'MODERATOR',
  Guest = 'GUEST'
}

enum OrderStatus {
  Pending = 'PENDING',
  Processing = 'PROCESSING',
  Shipped = 'SHIPPED',
  Delivered = 'DELIVERED',
  Cancelled = 'CANCELLED',
  Refunded = 'REFUNDED'
}

// ใช้ใน Angular Service
class OrderService {
  updateStatus(orderId: number, status: OrderStatus): void {
    console.log(`Order ${orderId}: ${status}`);
  }
}

const service = new OrderService();
service.updateStatus(1, OrderStatus.Shipped);
// service.updateStatus(1, 'SHIPPED');  // Error! ต้องใช้ enum
```

### Const Enums (performance ดีกว่า)

```typescript
const enum Color {
  Red = 'RED',
  Green = 'GREEN',
  Blue = 'BLUE'
}

// Compile ไปเป็น string โดยตรง ไม่สร้าง object
const color: Color = Color.Red;
// Compiled: const color = 'RED';
```

---

## 7. Decorators

Decorator เป็น core ของ Angular ใช้สำหรับ metadata

```typescript
// Class Decorator
function Component(options: { selector: string; template: string }) {
  return function(target: Function) {
    Reflect.defineMetadata('options', options, target);
  };
}

// Property Decorator
function Input() {
  return function(target: any, propertyKey: string) {
    // กำหนดว่า property นี้รับค่าจาก parent
  };
}

// Method Decorator
function Log(target: any, key: string, descriptor: PropertyDescriptor) {
  const original = descriptor.value;
  descriptor.value = function(...args: any[]) {
    console.log(`Calling ${key} with`, args);
    const result = original.apply(this, args);
    console.log(`${key} returned`, result);
    return result;
  };
  return descriptor;
}

// Parameter Decorator
function Inject(token: string) {
  return function(target: any, key: string, index: number) {
    // ระบุว่าจะ inject อะไรใน parameter นี้
  };
}

// Angular decorators ที่ใช้จริง:
// @Component, @Directive, @Pipe, @Injectable, @NgModule
// @Input, @Output, @ViewChild, @ContentChild
// @HostListener, @HostBinding
```

---

## 8. Type Utilities

TypeScript มี built-in utility types ที่มีประโยชน์มาก

```typescript
interface User {
  id: number;
  username: string;
  email: string;
  password: string;
  role: string;
  isActive: boolean;
  createdAt: Date;
}

// Partial<T> — ทุก property เป็น optional
type UpdateUserDto = Partial<User>;
// ใช้ได้: { email?: string; username?: string; ... }

// Required<T> — ทุก property เป็น required
type RequiredUser = Required<User>;
// ทุก property ต้องมีค่า

// Readonly<T> — ทุก property เป็น readonly
type ImmutableUser = Readonly<User>;
// ไม่สามารถแก้ไข property ได้

// Pick<T, K> — เลือก properties บางตัว
type UserCredentials = Pick<User, 'email' | 'password'>;
// { email: string; password: string; }

type UserPublicInfo = Pick<User, 'id' | 'username' | 'email'>;

// Omit<T, K> — ลบ properties บางตัวออก
type UserWithoutPassword = Omit<User, 'password'>;
type CreateUserDto = Omit<User, 'id' | 'createdAt'>;

// Record<K, T> — สร้าง object type
type UserMap = Record<number, User>;
type StatusMap = Record<string, boolean>;

const permissions: Record<string, boolean> = {
  canRead: true,
  canWrite: false,
  canDelete: false
};

// Exclude<T, U> — ลบ types ออกจาก union
type StringOrNumber = string | number | boolean;
type StringOrNumberOnly = Exclude<StringOrNumber, boolean>;
// string | number

// Extract<T, U> — เอาเฉพาะ types ที่ตรงกัน
type OnlyStrings = Extract<StringOrNumber, string>;
// string

// NonNullable<T> — ลบ null และ undefined
type MaybeString = string | null | undefined;
type DefiniteString = NonNullable<MaybeString>;
// string

// ReturnType<T> — ดึง return type ของ function
function getUser(): User {
  return {} as User;
}
type UserReturn = ReturnType<typeof getUser>;  // User

// Parameters<T> — ดึง parameter types ของ function
function createUser(name: string, age: number, active: boolean): User {
  return {} as User;
}
type CreateParams = Parameters<typeof createUser>;
// [name: string, age: number, active: boolean]

// InstanceType<T> — ดึง instance type ของ class constructor
class UserService {
  getUser() { return {} as User; }
}
type UserServiceInstance = InstanceType<typeof UserService>;
```

### Custom Utility Types

```typescript
// DeepPartial — Partial แบบ recursive
type DeepPartial<T> = {
  [K in keyof T]?: T[K] extends object ? DeepPartial<T[K]> : T[K];
};

// DeepReadonly
type DeepReadonly<T> = {
  readonly [K in keyof T]: T[K] extends object ? DeepReadonly<T[K]> : T[K];
};

// Nullable
type Nullable<T> = T | null;

// Optional
type Optional<T> = T | undefined;

// MaybeArray
type MaybeArray<T> = T | T[];
```

---

## 9. Type Guards

```typescript
// typeof guard
function formatValue(value: string | number): string {
  if (typeof value === 'string') {
    return value.toUpperCase();    // TypeScript รู้ว่า value เป็น string
  }
  return value.toFixed(2);         // TypeScript รู้ว่า value เป็น number
}

// instanceof guard
class HttpError extends Error {
  constructor(public statusCode: number, message: string) {
    super(message);
  }
}

class ValidationError extends Error {
  constructor(public field: string, message: string) {
    super(message);
  }
}

function handleError(error: Error): string {
  if (error instanceof HttpError) {
    return `HTTP ${error.statusCode}: ${error.message}`;
  }
  if (error instanceof ValidationError) {
    return `Validation on ${error.field}: ${error.message}`;
  }
  return error.message;
}

// Custom Type Guard
interface Cat {
  type: 'cat';
  meow(): void;
}

interface Dog {
  type: 'dog';
  bark(): void;
}

type Pet = Cat | Dog;

// Type predicate
function isCat(pet: Pet): pet is Cat {
  return pet.type === 'cat';
}

function makeSound(pet: Pet): void {
  if (isCat(pet)) {
    pet.meow();   // TypeScript รู้ว่าเป็น Cat
  } else {
    pet.bark();   // TypeScript รู้ว่าเป็น Dog
  }
}

// in operator guard
interface Admin {
  permissions: string[];
  ban(userId: number): void;
}

interface RegularUser {
  profile: { name: string };
}

function describeUser(user: Admin | RegularUser): string {
  if ('permissions' in user) {
    return `Admin with ${user.permissions.length} permissions`;
  }
  return `User: ${user.profile.name}`;
}
```

---

## 10. Workshop: TypeScript ใน Angular Context

### สร้าง Product Management System

```typescript
// ===============================================
// models/product.model.ts
// ===============================================

export enum ProductCategory {
  Electronics = 'ELECTRONICS',
  Clothing = 'CLOTHING',
  Food = 'FOOD',
  Books = 'BOOKS',
  Sports = 'SPORTS'
}

export enum ProductStatus {
  Active = 'ACTIVE',
  Inactive = 'INACTIVE',
  OutOfStock = 'OUT_OF_STOCK',
  Discontinued = 'DISCONTINUED'
}

export interface BaseEntity {
  id: number;
  createdAt: Date;
  updatedAt: Date;
}

export interface Product extends BaseEntity {
  name: string;
  description: string;
  price: number;
  discountPrice?: number;
  category: ProductCategory;
  status: ProductStatus;
  stock: number;
  images: string[];
  tags: string[];
  rating: number;
  reviewCount: number;
}

export type CreateProductDto = Omit<Product, 'id' | 'createdAt' | 'updatedAt' | 'rating' | 'reviewCount'>;
export type UpdateProductDto = Partial<CreateProductDto>;
export type ProductListItem = Pick<Product, 'id' | 'name' | 'price' | 'discountPrice' | 'category' | 'status' | 'stock' | 'rating'>;

export interface ProductFilter {
  search?: string;
  category?: ProductCategory;
  status?: ProductStatus;
  minPrice?: number;
  maxPrice?: number;
  inStock?: boolean;
  minRating?: number;
  tags?: string[];
}

export interface ProductSort {
  field: keyof ProductListItem;
  direction: 'asc' | 'desc';
}

export interface ProductPagination {
  page: number;
  limit: number;
}

export interface ProductQuery extends ProductFilter, ProductPagination {
  sort?: ProductSort;
}

export type ApiResponse<T> = {
  success: true;
  data: T;
  message: string;
} | {
  success: false;
  error: string;
  statusCode: number;
};

export interface PaginatedProducts {
  items: ProductListItem[];
  total: number;
  page: number;
  limit: number;
  totalPages: number;
}
```

```typescript
// ===============================================
// services/product.service.ts
// ===============================================

import { Injectable } from '@angular/core';
import { HttpClient, HttpParams } from '@angular/common/http';
import { Observable, throwError } from 'rxjs';
import { catchError, map } from 'rxjs/operators';
import {
  Product,
  CreateProductDto,
  UpdateProductDto,
  ProductQuery,
  ApiResponse,
  PaginatedProducts
} from '../models/product.model';

@Injectable({
  providedIn: 'root'
})
export class ProductService {
  private readonly apiUrl = 'https://api.example.com/products';

  constructor(private http: HttpClient) {}

  getProducts(query: ProductQuery): Observable<PaginatedProducts> {
    let params = new HttpParams()
      .set('page', query.page.toString())
      .set('limit', query.limit.toString());

    if (query.search) params = params.set('search', query.search);
    if (query.category) params = params.set('category', query.category);
    if (query.status) params = params.set('status', query.status);
    if (query.minPrice !== undefined) params = params.set('minPrice', query.minPrice.toString());
    if (query.maxPrice !== undefined) params = params.set('maxPrice', query.maxPrice.toString());

    return this.http
      .get<ApiResponse<PaginatedProducts>>(this.apiUrl, { params })
      .pipe(
        map(response => {
          if (!response.success) throw new Error(response.error);
          return response.data;
        }),
        catchError(this.handleError)
      );
  }

  getProductById(id: number): Observable<Product> {
    return this.http
      .get<ApiResponse<Product>>(`${this.apiUrl}/${id}`)
      .pipe(
        map(response => {
          if (!response.success) throw new Error(response.error);
          return response.data;
        }),
        catchError(this.handleError)
      );
  }

  createProduct(dto: CreateProductDto): Observable<Product> {
    return this.http
      .post<ApiResponse<Product>>(this.apiUrl, dto)
      .pipe(
        map(response => {
          if (!response.success) throw new Error(response.error);
          return response.data;
        }),
        catchError(this.handleError)
      );
  }

  updateProduct(id: number, dto: UpdateProductDto): Observable<Product> {
    return this.http
      .patch<ApiResponse<Product>>(`${this.apiUrl}/${id}`, dto)
      .pipe(
        map(response => {
          if (!response.success) throw new Error(response.error);
          return response.data;
        }),
        catchError(this.handleError)
      );
  }

  deleteProduct(id: number): Observable<void> {
    return this.http
      .delete<ApiResponse<void>>(`${this.apiUrl}/${id}`)
      .pipe(
        map(response => {
          if (!response.success) throw new Error(response.error);
        }),
        catchError(this.handleError)
      );
  }

  private handleError(error: any): Observable<never> {
    console.error('API Error:', error);
    return throwError(() => new Error(error.message || 'เกิดข้อผิดพลาด'));
  }
}
```

```typescript
// ===============================================
// components/product-list/product-list.component.ts
// ===============================================

import { Component, OnInit, OnDestroy } from '@angular/core';
import { CommonModule } from '@angular/common';
import { FormsModule } from '@angular/forms';
import { Subject } from 'rxjs';
import { takeUntil, debounceTime, distinctUntilChanged } from 'rxjs/operators';
import {
  ProductListItem,
  ProductQuery,
  ProductCategory,
  ProductStatus,
  PaginatedProducts
} from '../../models/product.model';
import { ProductService } from '../../services/product.service';

@Component({
  selector: 'app-product-list',
  standalone: true,
  imports: [CommonModule, FormsModule],
  template: `
    <div class="product-list">
      <h2>รายการสินค้า</h2>
      
      <!-- Filter Bar -->
      <div class="filters">
        <input
          [(ngModel)]="query.search"
          (ngModelChange)="onSearchChange()"
          placeholder="ค้นหาสินค้า..."
          class="search-input"
        />
        
        <select [(ngModel)]="query.category" (ngModelChange)="loadProducts()">
          <option value="">ทุกหมวดหมู่</option>
          <option *ngFor="let cat of categories" [value]="cat.value">
            {{ cat.label }}
          </option>
        </select>
      </div>

      <!-- Loading State -->
      <div *ngIf="isLoading" class="loading">กำลังโหลด...</div>

      <!-- Error State -->
      <div *ngIf="error" class="error">{{ error }}</div>

      <!-- Products Grid -->
      <div *ngIf="!isLoading && !error" class="products-grid">
        <div *ngFor="let product of products" class="product-card">
          <h3>{{ product.name }}</h3>
          <p class="price">
            <span *ngIf="product.discountPrice; else regularPrice">
              <del>{{ product.price | currency:'THB' }}</del>
              {{ product.discountPrice | currency:'THB' }}
            </span>
            <ng-template #regularPrice>
              {{ product.price | currency:'THB' }}
            </ng-template>
          </p>
          <span class="badge" [class]="'badge-' + product.status.toLowerCase()">
            {{ product.status }}
          </span>
          <div class="rating">⭐ {{ product.rating }}/5</div>
        </div>
      </div>

      <!-- Pagination -->
      <div *ngIf="pagination" class="pagination">
        <button
          [disabled]="query.page <= 1"
          (click)="changePage(query.page - 1)"
        >ก่อนหน้า</button>
        
        <span>หน้า {{ query.page }} / {{ pagination.totalPages }}</span>
        
        <button
          [disabled]="query.page >= pagination.totalPages"
          (click)="changePage(query.page + 1)"
        >ถัดไป</button>
      </div>
    </div>
  `
})
export class ProductListComponent implements OnInit, OnDestroy {
  products: ProductListItem[] = [];
  pagination: PaginatedProducts | null = null;
  isLoading = false;
  error: string | null = null;
  
  query: ProductQuery = {
    page: 1,
    limit: 12
  };
  
  categories = [
    { value: ProductCategory.Electronics, label: 'อิเล็กทรอนิกส์' },
    { value: ProductCategory.Clothing, label: 'เสื้อผ้า' },
    { value: ProductCategory.Food, label: 'อาหาร' },
    { value: ProductCategory.Books, label: 'หนังสือ' },
    { value: ProductCategory.Sports, label: 'กีฬา' }
  ];
  
  private destroy$ = new Subject<void>();
  private searchSubject = new Subject<string>();
  
  constructor(private productService: ProductService) {}
  
  ngOnInit(): void {
    // Debounce search
    this.searchSubject.pipe(
      debounceTime(300),
      distinctUntilChanged(),
      takeUntil(this.destroy$)
    ).subscribe(() => {
      this.query.page = 1;
      this.loadProducts();
    });
    
    this.loadProducts();
  }
  
  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
  }
  
  loadProducts(): void {
    this.isLoading = true;
    this.error = null;
    
    this.productService.getProducts(this.query).pipe(
      takeUntil(this.destroy$)
    ).subscribe({
      next: (result) => {
        this.products = result.items;
        this.pagination = result;
        this.isLoading = false;
      },
      error: (err: Error) => {
        this.error = err.message;
        this.isLoading = false;
      }
    });
  }
  
  onSearchChange(): void {
    this.searchSubject.next(this.query.search ?? '');
  }
  
  changePage(page: number): void {
    this.query = { ...this.query, page };
    this.loadProducts();
  }
}
```

---

## สรุปบทที่ 2

| หัวข้อ | สิ่งที่ได้เรียนรู้ |
|--------|------------------|
| Types | boolean, number, string, any, unknown, never |
| Arrays & Tuples | T[], Array\<T\>, [T, U] |
| Union & Intersection | T \| U, T & U |
| Literal Types | 'value' | 'other' |
| Interfaces | การกำหนด contract ของ object |
| Classes | OOP, inheritance, abstract, static |
| Generics | เขียนโค้ดที่ reusable |
| Enums | ค่าคงที่ที่มีชื่อ |
| Decorators | Metadata สำหรับ Angular |
| Utility Types | Partial, Readonly, Pick, Omit |
| Type Guards | typeof, instanceof, custom |

---

## แบบฝึกหัด

1. **ง่าย**: สร้าง interface สำหรับ Blog Post (title, content, author, tags, publishedAt)
2. **ปานกลาง**: สร้าง Generic `Stack<T>` class ที่มี push, pop, peek, isEmpty
3. **ท้าทาย**: สร้าง type-safe Event Emitter ด้วย Generics

---

## บทถัดไป

[Part 03 — สร้าง Component แรก →](part-03-first-component.md)
