# Part 39: Authentication System ใน Angular

## บทนำ

ระบบ Authentication เป็นส่วนสำคัญในแอปพลิเคชันส่วนใหญ่ ในบทนี้เราจะสร้างระบบ Auth ที่สมบูรณ์ประกอบด้วย Auth Service, Login/Logout flow, Protected Routes และ Token Storage

---

## 1. Auth Models

```typescript
// app/models/auth.models.ts
export interface LoginRequest {
  email: string;
  password: string;
  rememberMe?: boolean;
}

export interface RegisterRequest {
  name: string;
  email: string;
  password: string;
}

export interface AuthResponse {
  accessToken: string;
  refreshToken: string;
  user: User;
  expiresIn: number;
}

export interface User {
  id: string;
  name: string;
  email: string;
  roles: string[];
  avatar?: string;
  createdAt: string;
}

export interface AuthState {
  user: User | null;
  isAuthenticated: boolean;
  isLoading: boolean;
  error: string | null;
}
```

---

## 2. Token Service

```typescript
// app/services/token.service.ts
import { Injectable } from '@angular/core';

const ACCESS_TOKEN_KEY = 'access_token';
const REFRESH_TOKEN_KEY = 'refresh_token';
const USER_KEY = 'current_user';

@Injectable({ providedIn: 'root' })
export class TokenService {

  // เก็บ Access Token
  setAccessToken(token: string, rememberMe = false): void {
    if (rememberMe) {
      localStorage.setItem(ACCESS_TOKEN_KEY, token);
    } else {
      sessionStorage.setItem(ACCESS_TOKEN_KEY, token);
    }
  }

  getAccessToken(): string | null {
    return (
      localStorage.getItem(ACCESS_TOKEN_KEY) ||
      sessionStorage.getItem(ACCESS_TOKEN_KEY)
    );
  }

  // เก็บ Refresh Token (ใน localStorage เพื่อ persist)
  setRefreshToken(token: string): void {
    localStorage.setItem(REFRESH_TOKEN_KEY, token);
  }

  getRefreshToken(): string | null {
    return localStorage.getItem(REFRESH_TOKEN_KEY);
  }

  // เก็บข้อมูลผู้ใช้
  setUser(user: any): void {
    localStorage.setItem(USER_KEY, JSON.stringify(user));
  }

  getUser(): any | null {
    const userData = localStorage.getItem(USER_KEY);
    try {
      return userData ? JSON.parse(userData) : null;
    } catch {
      return null;
    }
  }

  // ล้างข้อมูลทั้งหมด
  clearAll(): void {
    localStorage.removeItem(ACCESS_TOKEN_KEY);
    localStorage.removeItem(REFRESH_TOKEN_KEY);
    localStorage.removeItem(USER_KEY);
    sessionStorage.removeItem(ACCESS_TOKEN_KEY);
  }

  // ตรวจสอบว่า token หมดอายุหรือยัง
  isTokenExpired(token: string): boolean {
    try {
      const payload = JSON.parse(atob(token.split('.')[1]));
      const expiry = payload.exp * 1000;
      return Date.now() > expiry;
    } catch {
      return true;
    }
  }

  hasValidToken(): boolean {
    const token = this.getAccessToken();
    return !!token && !this.isTokenExpired(token);
  }
}
```

---

## 3. Auth Service

```typescript
// app/services/auth.service.ts
import { Injectable, signal, computed } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Router } from '@angular/router';
import { Observable, throwError, BehaviorSubject } from 'rxjs';
import { tap, catchError, map } from 'rxjs/operators';
import {
  LoginRequest,
  RegisterRequest,
  AuthResponse,
  User,
  AuthState
} from '../models/auth.models';
import { TokenService } from './token.service';

@Injectable({ providedIn: 'root' })
export class AuthService {
  private readonly API_URL = '/api/auth';

  // State management ด้วย Signals
  private authState = signal<AuthState>({
    user: null,
    isAuthenticated: false,
    isLoading: false,
    error: null
  });

  // Public computed signals
  currentUser = computed(() => this.authState().user);
  isAuthenticated = computed(() => this.authState().isAuthenticated);
  isLoading = computed(() => this.authState().isLoading);
  authError = computed(() => this.authState().error);

  // BehaviorSubject สำหรับ backward compatibility
  private userSubject = new BehaviorSubject<User | null>(null);
  user$ = this.userSubject.asObservable();

  constructor(
    private http: HttpClient,
    private router: Router,
    private tokenService: TokenService
  ) {
    this.initializeFromStorage();
  }

  private initializeFromStorage(): void {
    if (this.tokenService.hasValidToken()) {
      const user = this.tokenService.getUser();
      if (user) {
        this.setAuthenticated(user);
      }
    }
  }

  login(credentials: LoginRequest): Observable<AuthResponse> {
    this.authState.update(s => ({ ...s, isLoading: true, error: null }));

    return this.http.post<AuthResponse>(`${this.API_URL}/login`, credentials).pipe(
      tap(response => {
        this.tokenService.setAccessToken(response.accessToken, credentials.rememberMe);
        this.tokenService.setRefreshToken(response.refreshToken);
        this.tokenService.setUser(response.user);
        this.setAuthenticated(response.user);
      }),
      catchError(error => {
        const message = error.error?.message || 'เกิดข้อผิดพลาด กรุณาลองใหม่';
        this.authState.update(s => ({
          ...s,
          isLoading: false,
          error: message
        }));
        return throwError(() => error);
      })
    );
  }

  register(data: RegisterRequest): Observable<AuthResponse> {
    this.authState.update(s => ({ ...s, isLoading: true, error: null }));

    return this.http.post<AuthResponse>(`${this.API_URL}/register`, data).pipe(
      tap(response => {
        this.tokenService.setAccessToken(response.accessToken);
        this.tokenService.setRefreshToken(response.refreshToken);
        this.tokenService.setUser(response.user);
        this.setAuthenticated(response.user);
      }),
      catchError(error => {
        const message = error.error?.message || 'ไม่สามารถสมัครสมาชิกได้';
        this.authState.update(s => ({
          ...s,
          isLoading: false,
          error: message
        }));
        return throwError(() => error);
      })
    );
  }

  logout(redirect = true): void {
    // เรียก API เพื่อ invalidate token ที่ server
    const token = this.tokenService.getRefreshToken();
    if (token) {
      this.http.post(`${this.API_URL}/logout`, { refreshToken: token })
        .subscribe({ error: () => {} }); // ไม่ต้องรอผล
    }

    this.tokenService.clearAll();
    this.authState.set({
      user: null,
      isAuthenticated: false,
      isLoading: false,
      error: null
    });
    this.userSubject.next(null);

    if (redirect) {
      this.router.navigate(['/login']);
    }
  }

  hasRole(role: string): boolean {
    return this.currentUser()?.roles.includes(role) ?? false;
  }

  hasAnyRole(roles: string[]): boolean {
    return roles.some(role => this.hasRole(role));
  }

  private setAuthenticated(user: User): void {
    this.authState.update(s => ({
      ...s,
      user,
      isAuthenticated: true,
      isLoading: false,
      error: null
    }));
    this.userSubject.next(user);
  }
}
```

---

## 4. Login Component

```typescript
// app/components/login/login.component.ts
import { Component, OnInit } from '@angular/core';
import { FormBuilder, FormGroup, Validators } from '@angular/forms';
import { Router, ActivatedRoute } from '@angular/router';
import { AuthService } from '../../services/auth.service';

@Component({
  selector: 'app-login',
  template: `
    <div class="login-container">
      <div class="login-card">
        <h1>เข้าสู่ระบบ</h1>

        <div *ngIf="errorMessage" class="alert alert-danger">
          {{ errorMessage }}
        </div>

        <form [formGroup]="loginForm" (ngSubmit)="onSubmit()">
          <div class="form-group">
            <label for="email">อีเมล</label>
            <input
              id="email"
              type="email"
              formControlName="email"
              class="form-control"
              [class.is-invalid]="email?.invalid && email?.touched"
              placeholder="example@email.com"
            >
            <div class="invalid-feedback" *ngIf="email?.invalid && email?.touched">
              <span *ngIf="email?.errors?.['required']">กรุณาใส่อีเมล</span>
              <span *ngIf="email?.errors?.['email']">รูปแบบอีเมลไม่ถูกต้อง</span>
            </div>
          </div>

          <div class="form-group">
            <label for="password">รหัสผ่าน</label>
            <div class="password-wrapper">
              <input
                id="password"
                [type]="showPassword ? 'text' : 'password'"
                formControlName="password"
                class="form-control"
                [class.is-invalid]="password?.invalid && password?.touched"
              >
              <button
                type="button"
                class="toggle-password"
                (click)="showPassword = !showPassword"
              >
                {{ showPassword ? '🙈' : '👁️' }}
              </button>
            </div>
            <div class="invalid-feedback" *ngIf="password?.invalid && password?.touched">
              กรุณาใส่รหัสผ่าน
            </div>
          </div>

          <div class="form-check">
            <input
              id="rememberMe"
              type="checkbox"
              formControlName="rememberMe"
              class="form-check-input"
            >
            <label for="rememberMe" class="form-check-label">จดจำฉัน</label>
          </div>

          <button
            type="submit"
            class="btn btn-primary btn-block"
            [disabled]="loginForm.invalid || isLoading"
          >
            <span *ngIf="isLoading">กำลังเข้าสู่ระบบ...</span>
            <span *ngIf="!isLoading">เข้าสู่ระบบ</span>
          </button>
        </form>

        <div class="links">
          <a routerLink="/forgot-password">ลืมรหัสผ่าน?</a>
          <a routerLink="/register">สมัครสมาชิกใหม่</a>
        </div>
      </div>
    </div>
  `,
  styles: [`
    .login-container {
      display: flex;
      justify-content: center;
      align-items: center;
      min-height: 100vh;
      background: #f5f5f5;
    }
    .login-card {
      width: 400px;
      padding: 32px;
      background: white;
      border-radius: 8px;
      box-shadow: 0 2px 10px rgba(0,0,0,0.1);
    }
    .form-group { margin-bottom: 16px; }
    .form-control { width: 100%; padding: 8px 12px; border: 1px solid #ddd; border-radius: 4px; }
    .is-invalid { border-color: #dc3545; }
    .invalid-feedback { color: #dc3545; font-size: 12px; }
    .password-wrapper { position: relative; }
    .toggle-password { position: absolute; right: 8px; top: 50%; transform: translateY(-50%); background: none; border: none; cursor: pointer; }
    .btn { padding: 10px 16px; border: none; border-radius: 4px; cursor: pointer; width: 100%; }
    .btn-primary { background: #007bff; color: white; }
    .btn-primary:disabled { opacity: 0.6; }
    .links { margin-top: 16px; display: flex; justify-content: space-between; }
    .alert { padding: 12px; border-radius: 4px; margin-bottom: 16px; }
    .alert-danger { background: #f8d7da; color: #842029; }
  `]
})
export class LoginComponent implements OnInit {
  loginForm!: FormGroup;
  showPassword = false;
  isLoading = false;
  errorMessage = '';
  private returnUrl = '/dashboard';

  constructor(
    private fb: FormBuilder,
    private authService: AuthService,
    private router: Router,
    private route: ActivatedRoute
  ) {}

  ngOnInit() {
    this.loginForm = this.fb.group({
      email: ['', [Validators.required, Validators.email]],
      password: ['', Validators.required],
      rememberMe: [false]
    });

    this.returnUrl = this.route.snapshot.queryParams['returnUrl'] || '/dashboard';

    // ถ้าล็อกอินแล้ว redirect ไปหน้าหลัก
    if (this.authService.isAuthenticated()) {
      this.router.navigate([this.returnUrl]);
    }
  }

  get email() { return this.loginForm.get('email'); }
  get password() { return this.loginForm.get('password'); }

  onSubmit() {
    if (this.loginForm.invalid) {
      this.loginForm.markAllAsTouched();
      return;
    }

    this.isLoading = true;
    this.errorMessage = '';

    this.authService.login(this.loginForm.value).subscribe({
      next: () => {
        this.router.navigate([this.returnUrl]);
      },
      error: (err) => {
        this.isLoading = false;
        this.errorMessage = err.error?.message || 'อีเมลหรือรหัสผ่านไม่ถูกต้อง';
      }
    });
  }
}
```

---

## 5. Auth Guards

```typescript
// app/guards/auth.guard.ts
import { inject } from '@angular/core';
import { CanActivateFn, Router } from '@angular/router';
import { AuthService } from '../services/auth.service';

export const authGuard: CanActivateFn = (route, state) => {
  const authService = inject(AuthService);
  const router = inject(Router);

  if (authService.isAuthenticated()) {
    return true;
  }

  // เก็บ URL ที่พยายามเข้าถึงไว้เพื่อ redirect หลังล็อกอิน
  return router.createUrlTree(['/login'], {
    queryParams: { returnUrl: state.url }
  });
};

// Guard สำหรับ page ที่ไม่ควรเข้าถึงหลังล็อกอิน (เช่น login, register)
export const guestGuard: CanActivateFn = (route, state) => {
  const authService = inject(AuthService);
  const router = inject(Router);

  if (!authService.isAuthenticated()) {
    return true;
  }

  return router.createUrlTree(['/dashboard']);
};
```

```typescript
// app/app-routing.module.ts
import { NgModule } from '@angular/core';
import { RouterModule, Routes } from '@angular/router';
import { authGuard, guestGuard } from './guards/auth.guard';

const routes: Routes = [
  {
    path: 'login',
    loadComponent: () => import('./components/login/login.component').then(m => m.LoginComponent),
    canActivate: [guestGuard]
  },
  {
    path: 'register',
    loadComponent: () => import('./components/register/register.component').then(m => m.RegisterComponent),
    canActivate: [guestGuard]
  },
  {
    path: 'dashboard',
    loadComponent: () => import('./components/dashboard/dashboard.component').then(m => m.DashboardComponent),
    canActivate: [authGuard]
  },
  {
    path: 'profile',
    loadComponent: () => import('./components/profile/profile.component').then(m => m.ProfileComponent),
    canActivate: [authGuard]
  },
  { path: '', redirectTo: '/dashboard', pathMatch: 'full' },
  { path: '**', redirectTo: '/dashboard' }
];

@NgModule({
  imports: [RouterModule.forRoot(routes)],
  exports: [RouterModule]
})
export class AppRoutingModule {}
```

---

## 6. Navbar Component พร้อม Auth State

```typescript
// app/components/navbar/navbar.component.ts
import { Component } from '@angular/core';
import { Router } from '@angular/router';
import { AuthService } from '../../services/auth.service';

@Component({
  selector: 'app-navbar',
  template: `
    <nav class="navbar">
      <div class="navbar-brand">
        <a routerLink="/">MyApp</a>
      </div>

      <div class="navbar-menu">
        <ng-container *ngIf="authService.isAuthenticated(); else guestMenu">
          <span class="user-info">
            สวัสดี, {{ authService.currentUser()?.name }}
          </span>
          <a routerLink="/profile">โปรไฟล์</a>
          <a routerLink="/dashboard">Dashboard</a>
          <button class="btn-logout" (click)="logout()">ออกจากระบบ</button>
        </ng-container>

        <ng-template #guestMenu>
          <a routerLink="/login">เข้าสู่ระบบ</a>
          <a routerLink="/register">สมัครสมาชิก</a>
        </ng-template>
      </div>
    </nav>
  `,
  styles: [`
    .navbar { display: flex; justify-content: space-between; align-items: center; padding: 16px 24px; background: #333; color: white; }
    .navbar a { color: white; text-decoration: none; margin: 0 8px; }
    .btn-logout { background: #dc3545; color: white; border: none; padding: 6px 12px; border-radius: 4px; cursor: pointer; }
  `]
})
export class NavbarComponent {
  constructor(
    public authService: AuthService,
    private router: Router
  ) {}

  logout() {
    this.authService.logout();
  }
}
```

---

## สรุป

ระบบ Authentication ที่สมบูรณ์ประกอบด้วย:
1. **TokenService** - จัดการ token storage อย่างปลอดภัย
2. **AuthService** - จัดการ state และ API calls
3. **Login Component** - UI สำหรับเข้าสู่ระบบ
4. **Auth Guards** - ป้องกัน routes ที่ต้องการ authentication
5. **Navbar** - แสดงสถานะการล็อกอินใน UI
