# Part 91: Single Sign-On (SSO) ด้วย Auth0 และ Keycloak

## SSO คืออะไร

Single Sign-On ให้ user login ครั้งเดียว แล้วเข้าถึงได้หลายแอป Auth0 และ Keycloak เป็น Identity Providers ที่นิยม

---

## 1. Auth0 Integration

### ติดตั้ง

```bash
npm install @auth0/auth0-angular
```

### app.module.ts

```typescript
import { NgModule } from '@angular/core';
import { AuthModule } from '@auth0/auth0-angular';
import { environment } from '../environments/environment';

@NgModule({
  imports: [
    AuthModule.forRoot({
      domain: environment.auth0.domain,
      clientId: environment.auth0.clientId,
      authorizationParams: {
        redirect_uri: window.location.origin,
        audience: environment.auth0.audience,
        scope: 'openid profile email'
      },
      // Automatically attach token ไปทุก API request
      httpInterceptor: {
        allowedList: [
          `${environment.apiUrl}/api/*`
        ]
      }
    })
  ]
})
export class AppModule {}
```

### environment.ts

```typescript
export const environment = {
  production: false,
  apiUrl: 'https://api.myapp.com',
  auth0: {
    domain: 'myapp.auth0.com',
    clientId: 'YOUR_AUTH0_CLIENT_ID',
    audience: 'https://api.myapp.com'
  }
};
```

---

## 2. Auth Service Wrapper

```typescript
// core/auth/auth.service.ts
import { Injectable } from '@angular/core';
import { AuthService as Auth0Service, User } from '@auth0/auth0-angular';
import { Observable, combineLatest, BehaviorSubject } from 'rxjs';
import { map, switchMap, tap } from 'rxjs/operators';
import { HttpClient } from '@angular/common/http';

export interface AppUser {
  id: string;
  email: string;
  name: string;
  picture?: string;
  roles: string[];
  permissions: string[];
  metadata?: Record<string, any>;
}

@Injectable({ providedIn: 'root' })
export class AuthService {
  private appUser$ = new BehaviorSubject<AppUser | null>(null);

  constructor(
    private auth0: Auth0Service,
    private http: HttpClient
  ) {
    // โหลด app user เมื่อ auth0 user พร้อม
    this.auth0.user$.pipe(
      switchMap(user => user ? this.loadAppUser(user) : [null])
    ).subscribe(user => this.appUser$.next(user));
  }

  get user$(): Observable<AppUser | null> {
    return this.appUser$.asObservable();
  }

  get isAuthenticated$(): Observable<boolean> {
    return this.auth0.isAuthenticated$;
  }

  get isLoading$(): Observable<boolean> {
    return this.auth0.isLoading$;
  }

  login(returnUrl?: string): void {
    this.auth0.loginWithRedirect({
      appState: { target: returnUrl || '/' }
    });
  }

  loginWithPopup(): Observable<void> {
    return this.auth0.loginWithPopup();
  }

  logout(): void {
    this.auth0.logout({
      logoutParams: {
        returnTo: window.location.origin
      }
    });
    this.appUser$.next(null);
  }

  getToken(): Observable<string> {
    return this.auth0.getAccessTokenSilently();
  }

  hasRole(role: string): Observable<boolean> {
    return this.user$.pipe(
      map(user => user?.roles.includes(role) ?? false)
    );
  }

  hasPermission(permission: string): Observable<boolean> {
    return this.user$.pipe(
      map(user => user?.permissions.includes(permission) ?? false)
    );
  }

  private loadAppUser(auth0User: User): Observable<AppUser> {
    return this.http.get<AppUser>(`/api/users/profile`).pipe(
      tap(profile => {
        // Merge Auth0 user data กับ app user data
      })
    );
  }
}
```

---

## 3. Auth Guard

```typescript
// core/auth/auth.guard.ts
import { Injectable } from '@angular/core';
import { CanActivate, ActivatedRouteSnapshot, Router } from '@angular/router';
import { Observable } from 'rxjs';
import { map, switchMap } from 'rxjs/operators';
import { AuthService } from './auth.service';

@Injectable({ providedIn: 'root' })
export class AuthGuard implements CanActivate {
  constructor(private authService: AuthService, private router: Router) {}

  canActivate(route: ActivatedRouteSnapshot): Observable<boolean> {
    const requiredRoles = route.data['roles'] as string[] || [];
    const requiredPermissions = route.data['permissions'] as string[] || [];

    return this.authService.isAuthenticated$.pipe(
      switchMap(isAuthenticated => {
        if (!isAuthenticated) {
          this.authService.login(route.url.join('/'));
          return [false];
        }

        return this.authService.user$.pipe(
          map(user => {
            if (!user) return false;

            // ตรวจสอบ roles
            if (requiredRoles.length > 0) {
              const hasRole = requiredRoles.some(role => user.roles.includes(role));
              if (!hasRole) {
                this.router.navigate(['/unauthorized']);
                return false;
              }
            }

            // ตรวจสอบ permissions
            if (requiredPermissions.length > 0) {
              const hasPermission = requiredPermissions.every(p => user.permissions.includes(p));
              if (!hasPermission) {
                this.router.navigate(['/unauthorized']);
                return false;
              }
            }

            return true;
          })
        );
      })
    );
  }
}
```

---

## 4. Login Component

```typescript
// features/auth/login/login.component.ts
import { Component, OnInit } from '@angular/core';
import { ActivatedRoute, Router } from '@angular/router';
import { AuthService } from '../../../core/auth/auth.service';

@Component({
  selector: 'app-login',
  template: `
    <div class="login-container">
      <div class="login-card">
        <img [src]="logoUrl" alt="Logo" class="logo">
        <h2>เข้าสู่ระบบ</h2>
        <p class="subtitle">ใช้บัญชีองค์กรของคุณ</p>
        
        <div *ngIf="isLoading$ | async" class="loading-state">
          <div class="spinner"></div>
          <p>กำลังตรวจสอบสถานะการเข้าสู่ระบบ...</p>
        </div>
        
        <div *ngIf="!(isLoading$ | async)">
          <button 
            class="btn-login"
            (click)="loginWithRedirect()"
          >
            <img src="/assets/icons/google.svg" alt="" width="20">
            เข้าสู่ระบบด้วย Google
          </button>
          
          <button 
            class="btn-login btn-microsoft"
            (click)="loginWithMicrosoft()"
          >
            <img src="/assets/icons/microsoft.svg" alt="" width="20">
            เข้าสู่ระบบด้วย Microsoft
          </button>

          <div class="divider"><span>หรือ</span></div>

          <button 
            class="btn-login btn-email"
            (click)="loginWithEmail()"
          >
            เข้าสู่ระบบด้วยอีเมล
          </button>
        </div>
        
        <p class="terms">
          การเข้าสู่ระบบหมายความว่าคุณยอมรับ
          <a href="/terms">เงื่อนไขการใช้บริการ</a>
        </p>
      </div>
    </div>
  `,
  styles: [`
    .login-container {
      min-height: 100vh;
      display: flex;
      align-items: center;
      justify-content: center;
      background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
    }
    .login-card {
      background: white;
      padding: 48px 40px;
      border-radius: 16px;
      width: 380px;
      text-align: center;
      box-shadow: 0 20px 60px rgba(0,0,0,0.2);
    }
    .logo { height: 48px; margin-bottom: 24px; }
    h2 { font-size: 24px; font-weight: 700; margin: 0 0 8px; color: #333; }
    .subtitle { color: #666; font-size: 14px; margin-bottom: 32px; }
    .btn-login {
      width: 100%;
      padding: 12px;
      border: 2px solid #ddd;
      border-radius: 8px;
      background: white;
      cursor: pointer;
      display: flex;
      align-items: center;
      justify-content: center;
      gap: 12px;
      font-size: 15px;
      font-weight: 500;
      margin-bottom: 12px;
      transition: all 0.2s;
    }
    .btn-login:hover { border-color: #667eea; background: #f8f8ff; }
    .btn-microsoft { background: #0078d4; color: white; border-color: #0078d4; }
    .btn-microsoft:hover { background: #006abc; }
    .btn-email { color: #667eea; border-color: #667eea; }
    .divider { text-align: center; margin: 16px 0; color: #999; font-size: 13px; position: relative; }
    .divider:before, .divider:after {
      content: '';
      position: absolute;
      top: 50%;
      width: 40%;
      height: 1px;
      background: #eee;
    }
    .divider:before { left: 0; }
    .divider:after { right: 0; }
    .terms { font-size: 12px; color: #999; margin-top: 24px; }
    .terms a { color: #667eea; }
    .loading-state { padding: 24px 0; }
    .spinner {
      width: 40px; height: 40px;
      border: 3px solid #eee;
      border-top-color: #667eea;
      border-radius: 50%;
      animation: spin 0.8s linear infinite;
      margin: 0 auto 12px;
    }
    @keyframes spin { to { transform: rotate(360deg); } }
  `]
})
export class LoginComponent implements OnInit {
  isLoading$ = this.authService.isLoading$;
  logoUrl = '/assets/logo.svg';

  constructor(
    private authService: AuthService,
    private router: Router,
    private route: ActivatedRoute
  ) {}

  ngOnInit(): void {
    // Redirect ถ้า authenticated แล้ว
    this.authService.isAuthenticated$.subscribe(isAuth => {
      if (isAuth) {
        const returnUrl = this.route.snapshot.queryParams['returnUrl'] || '/dashboard';
        this.router.navigateByUrl(returnUrl);
      }
    });
  }

  loginWithRedirect(): void {
    this.authService.login();
  }

  loginWithMicrosoft(): void {
    // Auth0 connection สำหรับ Microsoft
    // this.auth0.loginWithRedirect({ connection: 'windowslive' });
  }

  loginWithEmail(): void {
    this.authService.login();
  }
}
```

---

## 5. Keycloak Integration

```bash
npm install keycloak-angular keycloak-js
```

```typescript
// keycloak.init.ts
import { KeycloakService } from 'keycloak-angular';

export function initializeKeycloak(keycloak: KeycloakService) {
  return () =>
    keycloak.init({
      config: {
        url: 'http://keycloak.myapp.com/auth',
        realm: 'my-realm',
        clientId: 'angular-app'
      },
      initOptions: {
        onLoad: 'check-sso',
        silentCheckSsoRedirectUri: window.location.origin + '/assets/silent-check-sso.html'
      },
      enableBearerInterceptor: true,
      bearerPrefix: 'Bearer',
      bearerExcludedUrls: ['/assets', '/clients/public']
    });
}

// app.module.ts
import { KeycloakAngularModule, KeycloakService } from 'keycloak-angular';

@NgModule({
  imports: [KeycloakAngularModule],
  providers: [
    {
      provide: APP_INITIALIZER,
      useFactory: initializeKeycloak,
      multi: true,
      deps: [KeycloakService]
    }
  ]
})
export class AppModule {}
```

---

## สรุป

| Feature | Auth0 | Keycloak |
|---------|-------|----------|
| Hosting | Cloud (managed) | Self-hosted |
| Price | Free tier + paid | Open source |
| Setup | ง่าย | ซับซ้อนกว่า |
| Customization | จำกัด | ยืดหยุ่นมาก |
| Enterprise | ดี | ดีมาก |
| Social Login | พร้อมใช้ | ต้องตั้งค่าเอง |

### Best Practices

1. ใช้ PKCE flow สำหรับ SPA
2. ไม่เก็บ token ใน localStorage
3. Refresh token ก่อนหมดอายุ
4. Handle logout ทุก tab
5. Secure sensitive routes ด้วย guard
