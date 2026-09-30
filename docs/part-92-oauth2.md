# Part 92: OAuth 2.0 และ PKCE Flow ใน Angular

## OAuth 2.0 คืออะไร

OAuth 2.0 เป็น authorization framework ที่ให้ third-party apps เข้าถึง resources แทน user โดยไม่ต้องแชร์ password

---

## 1. OAuth 2.0 PKCE Flow

```
User → Angular App → Authorization Server (เช่น Google)
                           ↓
                    Authorization Code
                           ↓
Angular App ← Access Token + Refresh Token
```

### สร้าง Code Verifier และ Challenge

```typescript
// core/auth/pkce.utils.ts

export async function generateCodeVerifier(): Promise<string> {
  const array = new Uint8Array(32);
  crypto.getRandomValues(array);
  return base64UrlEncode(array);
}

export async function generateCodeChallenge(verifier: string): Promise<string> {
  const encoder = new TextEncoder();
  const data = encoder.encode(verifier);
  const digest = await crypto.subtle.digest('SHA-256', data);
  return base64UrlEncode(new Uint8Array(digest));
}

function base64UrlEncode(buffer: Uint8Array): string {
  return btoa(String.fromCharCode(...buffer))
    .replace(/\+/g, '-')
    .replace(/\//g, '_')
    .replace(/=/g, '');
}

export function generateState(): string {
  const array = new Uint8Array(16);
  crypto.getRandomValues(array);
  return base64UrlEncode(array);
}
```

---

## 2. OAuth Service

```typescript
// core/auth/oauth.service.ts
import { Injectable } from '@angular/core';
import { HttpClient, HttpParams } from '@angular/common/http';
import { Router } from '@angular/router';
import { BehaviorSubject, Observable, timer } from 'rxjs';
import { switchMap, tap } from 'rxjs/operators';
import { generateCodeVerifier, generateCodeChallenge, generateState } from './pkce.utils';

export interface TokenSet {
  accessToken: string;
  refreshToken?: string;
  idToken?: string;
  expiresIn: number;
  tokenType: string;
  scope: string;
  expiresAt: number;
}

export interface OAuthConfig {
  clientId: string;
  authorizationUrl: string;
  tokenUrl: string;
  redirectUri: string;
  scope: string;
  logoutUrl?: string;
  jwksUri?: string;
}

@Injectable({ providedIn: 'root' })
export class OAuthService {
  private tokenSet$ = new BehaviorSubject<TokenSet | null>(null);
  private refreshTimer: any;

  constructor(
    private http: HttpClient,
    private router: Router
  ) {
    this.loadTokenFromStorage();
  }

  // เริ่ม Authorization Code + PKCE flow
  async startLoginFlow(config: OAuthConfig, additionalParams?: Record<string, string>): Promise<void> {
    const codeVerifier = await generateCodeVerifier();
    const codeChallenge = await generateCodeChallenge(codeVerifier);
    const state = generateState();

    // บันทึกไว้ใน sessionStorage (ชั่วคราว)
    sessionStorage.setItem('oauth_code_verifier', codeVerifier);
    sessionStorage.setItem('oauth_state', state);
    sessionStorage.setItem('oauth_config', JSON.stringify(config));

    // สร้าง authorization URL
    const params = new URLSearchParams({
      response_type: 'code',
      client_id: config.clientId,
      redirect_uri: config.redirectUri,
      scope: config.scope,
      state,
      code_challenge: codeChallenge,
      code_challenge_method: 'S256',
      ...additionalParams
    });

    // Redirect ไป Authorization Server
    window.location.href = `${config.authorizationUrl}?${params}`;
  }

  // Handle callback จาก Authorization Server
  async handleCallback(code: string, state: string): Promise<boolean> {
    const storedState = sessionStorage.getItem('oauth_state');
    const codeVerifier = sessionStorage.getItem('oauth_code_verifier');
    const configStr = sessionStorage.getItem('oauth_config');

    // Validate state (ป้องกัน CSRF)
    if (state !== storedState) {
      console.error('State mismatch - possible CSRF attack');
      return false;
    }

    if (!codeVerifier || !configStr) {
      console.error('Missing PKCE data');
      return false;
    }

    const config: OAuthConfig = JSON.parse(configStr);

    // แลก code เป็น token
    try {
      const tokens = await this.exchangeCode(code, codeVerifier, config);
      this.setTokens(tokens);
      
      // Clean up
      sessionStorage.removeItem('oauth_code_verifier');
      sessionStorage.removeItem('oauth_state');
      sessionStorage.removeItem('oauth_config');
      
      return true;
    } catch (error) {
      console.error('Token exchange failed:', error);
      return false;
    }
  }

  private async exchangeCode(
    code: string,
    codeVerifier: string,
    config: OAuthConfig
  ): Promise<TokenSet> {
    const body = new URLSearchParams({
      grant_type: 'authorization_code',
      code,
      redirect_uri: config.redirectUri,
      client_id: config.clientId,
      code_verifier: codeVerifier
    });

    const response = await this.http.post<any>(
      config.tokenUrl,
      body.toString(),
      { headers: { 'Content-Type': 'application/x-www-form-urlencoded' } }
    ).toPromise();

    return {
      accessToken: response.access_token,
      refreshToken: response.refresh_token,
      idToken: response.id_token,
      expiresIn: response.expires_in,
      tokenType: response.token_type,
      scope: response.scope,
      expiresAt: Date.now() + (response.expires_in * 1000)
    };
  }

  // Refresh token
  async refreshTokens(config: OAuthConfig): Promise<boolean> {
    const current = this.tokenSet$.value;
    if (!current?.refreshToken) return false;

    try {
      const body = new URLSearchParams({
        grant_type: 'refresh_token',
        refresh_token: current.refreshToken,
        client_id: config.clientId
      });

      const response = await this.http.post<any>(
        config.tokenUrl,
        body.toString(),
        { headers: { 'Content-Type': 'application/x-www-form-urlencoded' } }
      ).toPromise();

      const newTokens: TokenSet = {
        accessToken: response.access_token,
        refreshToken: response.refresh_token || current.refreshToken,
        idToken: response.id_token,
        expiresIn: response.expires_in,
        tokenType: response.token_type,
        scope: response.scope,
        expiresAt: Date.now() + (response.expires_in * 1000)
      };

      this.setTokens(newTokens);
      return true;
    } catch {
      this.clearTokens();
      return false;
    }
  }

  private setTokens(tokens: TokenSet): void {
    this.tokenSet$.next(tokens);
    
    // บันทึกไว้ใน memory (ไม่ใช้ localStorage สำหรับ security)
    // หรือใช้ secure cookie ที่ server side

    // ตั้ง auto refresh
    this.scheduleTokenRefresh(tokens);
  }

  private scheduleTokenRefresh(tokens: TokenSet): void {
    if (this.refreshTimer) clearTimeout(this.refreshTimer);
    
    const timeUntilExpiry = tokens.expiresAt - Date.now();
    const refreshAt = timeUntilExpiry - (60 * 1000); // Refresh 1 นาทีก่อนหมดอายุ
    
    if (refreshAt > 0) {
      this.refreshTimer = setTimeout(() => {
        // this.refreshTokens(config); // ต้องการ config
      }, refreshAt);
    }
  }

  getAccessToken(): string | null {
    const tokens = this.tokenSet$.value;
    if (!tokens) return null;
    if (Date.now() >= tokens.expiresAt) return null;
    return tokens.accessToken;
  }

  get isAuthenticated(): boolean {
    const token = this.getAccessToken();
    return !!token;
  }

  clearTokens(): void {
    this.tokenSet$.next(null);
    if (this.refreshTimer) clearTimeout(this.refreshTimer);
  }

  private loadTokenFromStorage(): void {
    // Optional: โหลดจาก memory หรือ secure cookie
  }
}
```

---

## 3. Callback Handler Component

```typescript
// features/auth/callback/callback.component.ts
import { Component, OnInit } from '@angular/core';
import { ActivatedRoute, Router } from '@angular/router';
import { OAuthService } from '../../../core/auth/oauth.service';

@Component({
  selector: 'app-callback',
  template: `
    <div class="callback-container">
      <div *ngIf="isProcessing" class="processing">
        <div class="spinner-large"></div>
        <p>กำลังตรวจสอบข้อมูล...</p>
      </div>
      
      <div *ngIf="error" class="error-state">
        <div class="error-icon">⚠️</div>
        <h3>เกิดข้อผิดพลาด</h3>
        <p>{{ error }}</p>
        <button (click)="retryLogin()">ลองใหม่</button>
      </div>
    </div>
  `,
  styles: [`
    .callback-container { 
      min-height: 100vh; 
      display: flex; 
      align-items: center; 
      justify-content: center; 
    }
    .processing { text-align: center; }
    .spinner-large {
      width: 60px; height: 60px;
      border: 4px solid #eee;
      border-top-color: #2196f3;
      border-radius: 50%;
      animation: spin 1s linear infinite;
      margin: 0 auto 16px;
    }
    @keyframes spin { to { transform: rotate(360deg); } }
    .error-state { text-align: center; }
    .error-icon { font-size: 48px; margin-bottom: 16px; }
    button { padding: 10px 24px; background: #2196f3; color: white; border: none; border-radius: 6px; cursor: pointer; margin-top: 16px; }
  `]
})
export class CallbackComponent implements OnInit {
  isProcessing = true;
  error = '';

  constructor(
    private route: ActivatedRoute,
    private router: Router,
    private oauthService: OAuthService
  ) {}

  async ngOnInit(): Promise<void> {
    const code = this.route.snapshot.queryParams['code'];
    const state = this.route.snapshot.queryParams['state'];
    const errorParam = this.route.snapshot.queryParams['error'];

    if (errorParam) {
      this.isProcessing = false;
      this.error = this.route.snapshot.queryParams['error_description'] || 'เกิดข้อผิดพลาดในการเข้าสู่ระบบ';
      return;
    }

    if (!code || !state) {
      this.isProcessing = false;
      this.error = 'ข้อมูลการเข้าสู่ระบบไม่สมบูรณ์';
      return;
    }

    const success = await this.oauthService.handleCallback(code, state);

    if (success) {
      // Navigate ไปหน้าที่ต้องการ
      const returnUrl = sessionStorage.getItem('oauth_return_url') || '/dashboard';
      sessionStorage.removeItem('oauth_return_url');
      this.router.navigateByUrl(returnUrl);
    } else {
      this.isProcessing = false;
      this.error = 'ไม่สามารถตรวจสอบข้อมูลได้ กรุณาลองใหม่';
    }
  }

  retryLogin(): void {
    this.router.navigate(['/login']);
  }
}
```

---

## 4. JWT Decode Service

```typescript
// core/auth/jwt.service.ts
import { Injectable } from '@angular/core';

export interface JWTPayload {
  sub: string;
  email?: string;
  name?: string;
  roles?: string[];
  permissions?: string[];
  exp: number;
  iat: number;
  iss: string;
  aud: string | string[];
}

@Injectable({ providedIn: 'root' })
export class JWTService {
  decode(token: string): JWTPayload | null {
    try {
      const parts = token.split('.');
      if (parts.length !== 3) return null;
      
      const payload = JSON.parse(atob(
        parts[1].replace(/-/g, '+').replace(/_/g, '/')
      ));
      
      return payload as JWTPayload;
    } catch {
      return null;
    }
  }

  isExpired(token: string): boolean {
    const payload = this.decode(token);
    if (!payload) return true;
    return Date.now() >= payload.exp * 1000;
  }

  getExpiry(token: string): Date | null {
    const payload = this.decode(token);
    if (!payload) return null;
    return new Date(payload.exp * 1000);
  }

  getClaims(token: string): JWTPayload | null {
    return this.decode(token);
  }

  getUserId(token: string): string | null {
    return this.decode(token)?.sub ?? null;
  }

  getRoles(token: string): string[] {
    return this.decode(token)?.roles ?? [];
  }
}
```

---

## สรุป

| ขั้นตอน | รายละเอียด |
|---------|-----------|
| 1. Code Verifier | สร้าง random string 32 bytes |
| 2. Code Challenge | SHA-256 hash ของ verifier |
| 3. Authorization Request | ส่ง challenge ไปด้วย |
| 4. Auth Code | ได้รับจาก server |
| 5. Token Exchange | ส่ง code + verifier |
| 6. Access Token | ใช้เรียก API |
| 7. Refresh | ก่อนหมดอายุ |

### Security Best Practices

- ใช้ PKCE เสมอสำหรับ SPA
- ไม่เก็บ token ใน localStorage
- Validate state parameter
- Validate token signature
- ใช้ short-lived access tokens (15 นาที)
- Rotate refresh tokens
