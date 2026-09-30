# Part 45: Progressive Web App (PWA) ใน Angular

## บทนำ

Progressive Web App (PWA) คือแอปพลิเคชัน web ที่มีความสามารถเหมือน native app ได้แก่ ใช้งาน offline ได้, ติดตั้งบน device ได้ และมี push notifications ในบทนี้เราจะ setup PWA, Web Manifest และทำให้แอป installable

---

## 1. การติดตั้ง PWA

```bash
# เพิ่ม PWA support ด้วย Angular CLI
ng add @angular/pwa

# คำสั่งนี้จะ:
# - ติดตั้ง @angular/service-worker
# - สร้าง manifest.webmanifest
# - สร้าง ngsw-config.json
# - อัปเดต angular.json และ app.module.ts
```

---

## 2. Web App Manifest

```json
// src/manifest.webmanifest
{
  "name": "ร้านค้าออนไลน์ - MyShop",
  "short_name": "MyShop",
  "description": "ระบบจัดการร้านค้าออนไลน์ครบวงจร",
  "theme_color": "#1976d2",
  "background_color": "#ffffff",
  "display": "standalone",
  "scope": "/",
  "start_url": "/",
  "orientation": "portrait-primary",
  "lang": "th",
  "dir": "ltr",
  "categories": ["shopping", "business"],
  "icons": [
    {
      "src": "assets/icons/icon-72x72.png",
      "sizes": "72x72",
      "type": "image/png",
      "purpose": "maskable any"
    },
    {
      "src": "assets/icons/icon-96x96.png",
      "sizes": "96x96",
      "type": "image/png",
      "purpose": "maskable any"
    },
    {
      "src": "assets/icons/icon-128x128.png",
      "sizes": "128x128",
      "type": "image/png",
      "purpose": "maskable any"
    },
    {
      "src": "assets/icons/icon-144x144.png",
      "sizes": "144x144",
      "type": "image/png",
      "purpose": "maskable any"
    },
    {
      "src": "assets/icons/icon-152x152.png",
      "sizes": "152x152",
      "type": "image/png",
      "purpose": "maskable any"
    },
    {
      "src": "assets/icons/icon-192x192.png",
      "sizes": "192x192",
      "type": "image/png",
      "purpose": "maskable any"
    },
    {
      "src": "assets/icons/icon-384x384.png",
      "sizes": "384x384",
      "type": "image/png",
      "purpose": "maskable any"
    },
    {
      "src": "assets/icons/icon-512x512.png",
      "sizes": "512x512",
      "type": "image/png",
      "purpose": "maskable any"
    }
  ],
  "screenshots": [
    {
      "src": "assets/screenshots/desktop.png",
      "sizes": "1280x800",
      "type": "image/png",
      "form_factor": "wide",
      "label": "หน้า Dashboard บน Desktop"
    },
    {
      "src": "assets/screenshots/mobile.png",
      "sizes": "390x844",
      "type": "image/png",
      "form_factor": "narrow",
      "label": "หน้า Dashboard บน Mobile"
    }
  ],
  "shortcuts": [
    {
      "name": "เพิ่มสินค้า",
      "short_name": "เพิ่มสินค้า",
      "description": "เพิ่มสินค้าใหม่",
      "url": "/products/new",
      "icons": [{ "src": "assets/icons/add-product.png", "sizes": "96x96" }]
    },
    {
      "name": "คำสั่งซื้อ",
      "short_name": "คำสั่งซื้อ",
      "description": "ดูคำสั่งซื้อล่าสุด",
      "url": "/orders",
      "icons": [{ "src": "assets/icons/orders.png", "sizes": "96x96" }]
    }
  ]
}
```

---

## 3. Index.html Meta Tags

```html
<!-- src/index.html -->
<!DOCTYPE html>
<html lang="th">
<head>
  <meta charset="utf-8">
  <title>MyShop - ร้านค้าออนไลน์</title>
  <base href="/">
  <meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">

  <!-- PWA Meta Tags -->
  <meta name="theme-color" content="#1976d2">
  <meta name="mobile-web-app-capable" content="yes">
  <meta name="apple-mobile-web-app-capable" content="yes">
  <meta name="apple-mobile-web-app-status-bar-style" content="default">
  <meta name="apple-mobile-web-app-title" content="MyShop">
  <meta name="msapplication-TileColor" content="#1976d2">
  <meta name="msapplication-TileImage" content="assets/icons/icon-144x144.png">

  <!-- Manifest -->
  <link rel="manifest" href="manifest.webmanifest">

  <!-- Apple Touch Icons -->
  <link rel="apple-touch-icon" href="assets/icons/icon-152x152.png">
  <link rel="apple-touch-icon" sizes="180x180" href="assets/icons/icon-192x192.png">

  <!-- Favicon -->
  <link rel="icon" type="image/x-icon" href="favicon.ico">
  <link rel="icon" type="image/png" sizes="32x32" href="assets/icons/favicon-32x32.png">

  <!-- Splash Screens for iOS -->
  <link rel="apple-touch-startup-image"
    media="(device-width: 390px) and (device-height: 844px)"
    href="assets/splash/splash-390x844.png">
</head>
<body>
  <app-root></app-root>
</body>
</html>
```

---

## 4. PWA Install Service

```typescript
// app/services/pwa-install.service.ts
import { Injectable, signal } from '@angular/core';
import { Platform } from '@angular/cdk/platform';

@Injectable({ providedIn: 'root' })
export class PwaInstallService {
  private deferredPrompt: any = null;

  // Signals สำหรับ state
  canInstall = signal(false);
  isInstalled = signal(false);
  isInstalling = signal(false);
  platform = signal<'ios' | 'android' | 'desktop' | 'unknown'>('unknown');

  constructor(private cdkPlatform: Platform) {
    this.detectPlatform();
    this.listenForInstallPrompt();
    this.checkIfInstalled();
  }

  private detectPlatform() {
    if (this.cdkPlatform.IOS) {
      this.platform.set('ios');
    } else if (this.cdkPlatform.ANDROID) {
      this.platform.set('android');
    } else if (!this.cdkPlatform.isBrowser) {
      this.platform.set('unknown');
    } else {
      this.platform.set('desktop');
    }
  }

  private listenForInstallPrompt() {
    window.addEventListener('beforeinstallprompt', (event) => {
      event.preventDefault();
      this.deferredPrompt = event;
      this.canInstall.set(true);
    });

    window.addEventListener('appinstalled', () => {
      this.isInstalled.set(true);
      this.canInstall.set(false);
      this.deferredPrompt = null;
      console.log('PWA installed successfully');
    });
  }

  private checkIfInstalled(): void {
    // ตรวจสอบว่า run ใน standalone mode (ติดตั้งแล้ว)
    const isStandalone =
      window.matchMedia('(display-mode: standalone)').matches ||
      (window.navigator as any).standalone === true;

    this.isInstalled.set(isStandalone);
  }

  async install(): Promise<boolean> {
    if (!this.deferredPrompt) return false;

    this.isInstalling.set(true);

    try {
      await this.deferredPrompt.prompt();
      const choice = await this.deferredPrompt.userChoice;

      if (choice.outcome === 'accepted') {
        this.deferredPrompt = null;
        this.canInstall.set(false);
        return true;
      }
      return false;
    } catch (error) {
      console.error('Install failed:', error);
      return false;
    } finally {
      this.isInstalling.set(false);
    }
  }

  // คำแนะนำสำหรับ iOS (ไม่มี beforeinstallprompt)
  getIosInstallInstructions(): string[] {
    return [
      '1. แตะปุ่ม Share ที่ toolbar ด้านล่าง',
      '2. เลื่อนลงแล้วเลือก "Add to Home Screen"',
      '3. แตะ "Add" เพื่อติดตั้ง'
    ];
  }
}
```

---

## 5. Install Prompt Component

```typescript
// app/components/install-prompt/install-prompt.component.ts
import { Component, OnInit, signal } from '@angular/core';
import { PwaInstallService } from '../../services/pwa-install.service';

@Component({
  selector: 'app-install-prompt',
  template: `
    <!-- Android/Desktop Install Banner -->
    <div
      *ngIf="pwaService.canInstall() && !dismissed()"
      class="install-banner"
      role="banner"
    >
      <img src="assets/icons/icon-72x72.png" alt="App Icon" class="app-icon">
      <div class="install-info">
        <strong>ติดตั้ง MyShop</strong>
        <span>ใช้งานออฟไลน์ได้ เร็วกว่าการใช้ Browser</span>
      </div>
      <div class="install-actions">
        <button class="btn-install" (click)="install()" [disabled]="pwaService.isInstalling()">
          {{ pwaService.isInstalling() ? 'กำลังติดตั้ง...' : 'ติดตั้ง' }}
        </button>
        <button class="btn-dismiss" (click)="dismiss()">✕</button>
      </div>
    </div>

    <!-- iOS Instructions -->
    <div
      *ngIf="showIosInstructions()"
      class="ios-instructions"
    >
      <h4>ติดตั้งแอปบน iPhone/iPad</h4>
      <ol>
        <li *ngFor="let step of pwaService.getIosInstallInstructions()">{{ step }}</li>
      </ol>
      <button (click)="showIosInstructions.set(false)">ปิด</button>
    </div>

    <!-- Already installed -->
    <div *ngIf="pwaService.isInstalled()" class="installed-badge">
      ✓ ติดตั้งแล้ว
    </div>
  `,
  styles: [`
    .install-banner {
      position: fixed;
      bottom: 0;
      left: 0;
      right: 0;
      background: white;
      padding: 12px 16px;
      display: flex;
      align-items: center;
      gap: 12px;
      box-shadow: 0 -2px 10px rgba(0,0,0,0.1);
      z-index: 1000;
    }
    .app-icon { width: 48px; height: 48px; border-radius: 8px; }
    .install-info { flex: 1; display: flex; flex-direction: column; }
    .install-actions { display: flex; gap: 8px; align-items: center; }
    .btn-install { background: #1976d2; color: white; border: none; padding: 8px 16px; border-radius: 4px; cursor: pointer; }
    .btn-dismiss { background: none; border: none; font-size: 18px; cursor: pointer; color: #666; }
    .ios-instructions { background: white; padding: 20px; border-radius: 8px; box-shadow: 0 2px 10px rgba(0,0,0,0.1); }
    .installed-badge { position: fixed; bottom: 16px; right: 16px; background: #4caf50; color: white; padding: 6px 12px; border-radius: 4px; font-size: 12px; }
  `]
})
export class InstallPromptComponent {
  dismissed = signal(false);
  showIosInstructions = signal(false);

  constructor(public pwaService: PwaInstallService) {}

  async install() {
    const installed = await this.pwaService.install();
    if (!installed) {
      // ผู้ใช้ปฏิเสธ
      this.dismissed.set(true);
    }
  }

  dismiss() {
    this.dismissed.set(true);
    // จำว่า dismiss แล้ว 24 ชั่วโมง
    sessionStorage.setItem('pwa_dismissed', Date.now().toString());
  }
}
```

---

## 6. Offline Detection Component

```typescript
// app/components/offline-indicator/offline-indicator.component.ts
import { Component, OnInit, OnDestroy, signal } from '@angular/core';
import { Subject, fromEvent, merge } from 'rxjs';
import { takeUntil, map } from 'rxjs/operators';

@Component({
  selector: 'app-offline-indicator',
  template: `
    <div
      *ngIf="!isOnline()"
      class="offline-banner"
      role="alert"
    >
      <mat-icon>wifi_off</mat-icon>
      <span>ไม่มีการเชื่อมต่ออินเทอร์เน็ต - แสดงข้อมูลจาก Cache</span>
    </div>

    <div
      *ngIf="showReconnected()"
      class="reconnected-toast"
    >
      <mat-icon>wifi</mat-icon>
      <span>เชื่อมต่ออินเทอร์เน็ตแล้ว</span>
    </div>
  `,
  styles: [`
    .offline-banner {
      position: fixed;
      top: 0;
      left: 0;
      right: 0;
      background: #f44336;
      color: white;
      padding: 8px 16px;
      display: flex;
      align-items: center;
      gap: 8px;
      z-index: 9999;
    }
    .reconnected-toast {
      position: fixed;
      bottom: 16px;
      left: 50%;
      transform: translateX(-50%);
      background: #4caf50;
      color: white;
      padding: 12px 24px;
      border-radius: 4px;
      display: flex;
      align-items: center;
      gap: 8px;
      z-index: 9999;
    }
  `]
})
export class OfflineIndicatorComponent implements OnInit, OnDestroy {
  isOnline = signal(navigator.onLine);
  showReconnected = signal(false);

  private destroy$ = new Subject<void>();
  private reconnectedTimer?: ReturnType<typeof setTimeout>;

  ngOnInit() {
    merge(
      fromEvent(window, 'online').pipe(map(() => true)),
      fromEvent(window, 'offline').pipe(map(() => false))
    )
    .pipe(takeUntil(this.destroy$))
    .subscribe(online => {
      const wasOffline = !this.isOnline();
      this.isOnline.set(online);

      if (online && wasOffline) {
        this.showReconnected.set(true);
        clearTimeout(this.reconnectedTimer);
        this.reconnectedTimer = setTimeout(() => {
          this.showReconnected.set(false);
        }, 3000);
      }
    });
  }

  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
    clearTimeout(this.reconnectedTimer);
  }
}
```

---

## 7. Update Notification

```typescript
// app/services/app-update.service.ts
import { Injectable } from '@angular/core';
import { SwUpdate, VersionReadyEvent } from '@angular/service-worker';
import { filter } from 'rxjs/operators';
import { MatSnackBar } from '@angular/material/snack-bar';

@Injectable({ providedIn: 'root' })
export class AppUpdateService {
  constructor(
    private swUpdate: SwUpdate,
    private snackBar: MatSnackBar
  ) {
    if (this.swUpdate.isEnabled) {
      this.listenForUpdates();
      this.checkForUpdates();
    }
  }

  private listenForUpdates() {
    this.swUpdate.versionUpdates
      .pipe(filter((evt): evt is VersionReadyEvent => evt.type === 'VERSION_READY'))
      .subscribe(() => {
        const snackRef = this.snackBar.open(
          'มีเวอร์ชันใหม่! กรุณาอัปเดตเพื่อประสบการณ์ที่ดีขึ้น',
          'อัปเดตเดี๋ยวนี้',
          { duration: 0 }
        );

        snackRef.onAction().subscribe(() => {
          window.location.reload();
        });
      });
  }

  private checkForUpdates() {
    // ตรวจสอบทุก 6 ชั่วโมง
    setInterval(() => {
      this.swUpdate.checkForUpdate();
    }, 6 * 60 * 60 * 1000);
  }

  async forceUpdate() {
    await this.swUpdate.activateUpdate();
    window.location.reload();
  }
}
```

---

## สรุป

PWA ต้องการองค์ประกอบดังนี้:

| ส่วนประกอบ | หน้าที่ |
|-----------|--------|
| Web Manifest | ข้อมูลแอป, icons, display mode |
| Service Worker | Cache, offline support, push notifications |
| HTTPS | บังคับสำหรับ Service Worker |
| Install Prompt | แนะนำให้ผู้ใช้ติดตั้ง |
| Offline Detection | แจ้งสถานะ connection |
| Update Notification | แจ้ง version ใหม่ |

การสร้าง PWA ที่ดีต้องให้ผู้ใช้รู้สึกว่าใช้งานได้ดีทั้ง online และ offline
