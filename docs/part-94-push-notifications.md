# Part 94: Push Notifications ด้วย Web Push API และ FCM

## Push Notifications คืออะไร

Push Notifications ให้เราส่งข้อความถึง user แม้ไม่ได้เปิดแอปอยู่ ใช้ Web Push API ร่วมกับ Service Worker

---

## 1. Angular PWA Setup

```bash
ng add @angular/pwa
```

---

## 2. Push Notification Service

```typescript
// core/push/push-notification.service.ts
import { Injectable } from '@angular/core';
import { SwPush } from '@angular/service-worker';
import { HttpClient } from '@angular/common/http';
import { BehaviorSubject, Observable } from 'rxjs';

export interface NotificationPermission {
  granted: boolean;
  subscription?: PushSubscription;
}

@Injectable({ providedIn: 'root' })
export class PushNotificationService {
  private permission$ = new BehaviorSubject<NotificationPermission>({
    granted: false
  });

  // VAPID Public Key - รับจาก server
  private readonly VAPID_PUBLIC_KEY = 'YOUR_VAPID_PUBLIC_KEY';

  constructor(
    private swPush: SwPush,
    private http: HttpClient
  ) {
    this.checkExistingSubscription();
    this.setupMessageListener();
  }

  private async checkExistingSubscription(): Promise<void> {
    if (!this.swPush.isEnabled) return;

    const subscription = await this.swPush.subscription.toPromise();
    if (subscription) {
      this.permission$.next({ granted: true, subscription });
    }
  }

  // ขอ permission และ subscribe
  async requestPermission(): Promise<NotificationPermission> {
    if (!('Notification' in window)) {
      return { granted: false };
    }

    // ขอ permission จาก browser
    const permission = await Notification.requestPermission();
    
    if (permission !== 'granted') {
      return { granted: false };
    }

    try {
      // Subscribe ไปยัง push server
      const subscription = await this.swPush.requestSubscription({
        serverPublicKey: this.VAPID_PUBLIC_KEY
      });

      // ส่ง subscription ไปบันทึกที่ server
      await this.saveSubscription(subscription);

      const result: NotificationPermission = { granted: true, subscription };
      this.permission$.next(result);
      return result;
    } catch (error) {
      console.error('Subscribe failed:', error);
      return { granted: false };
    }
  }

  private async saveSubscription(subscription: PushSubscription): Promise<void> {
    const userId = localStorage.getItem('userId');
    
    await this.http.post('/api/push/subscribe', {
      subscription: subscription.toJSON(),
      userId,
      deviceInfo: {
        userAgent: navigator.userAgent,
        platform: navigator.platform
      }
    }).toPromise();
  }

  async unsubscribe(): Promise<void> {
    const subscription = await this.swPush.subscription.toPromise();
    if (!subscription) return;

    await this.swPush.unsubscribe();
    
    // แจ้ง server ด้วย
    await this.http.post('/api/push/unsubscribe', {
      endpoint: subscription.endpoint
    }).toPromise();

    this.permission$.next({ granted: false });
  }

  private setupMessageListener(): void {
    this.swPush.messages.subscribe((message: any) => {
      console.log('Push message received:', message);
      // Handle foreground messages
    });

    this.swPush.notificationClicks.subscribe(({ action, notification }) => {
      console.log('Notification clicked:', action, notification);
      // Handle notification clicks
      if (notification.data?.url) {
        window.open(notification.data.url, '_blank');
      }
    });
  }

  // ส่ง notification ไปยัง specific user (ผ่าน server)
  async sendToUser(userId: string, payload: NotificationPayload): Promise<void> {
    await this.http.post('/api/push/send', {
      userId,
      payload
    }).toPromise();
  }

  // ส่ง notification ไปทุกคน (broadcast)
  async broadcast(payload: NotificationPayload): Promise<void> {
    await this.http.post('/api/push/broadcast', payload).toPromise();
  }

  get permissionStatus$(): Observable<NotificationPermission> {
    return this.permission$.asObservable();
  }

  get isSupported(): boolean {
    return this.swPush.isEnabled && 'Notification' in window;
  }

  get currentPermission(): string {
    return Notification.permission;
  }
}

export interface NotificationPayload {
  title: string;
  body: string;
  icon?: string;
  badge?: string;
  image?: string;
  data?: {
    url?: string;
    [key: string]: any;
  };
  actions?: { action: string; title: string; icon?: string }[];
  requireInteraction?: boolean;
  silent?: boolean;
  tag?: string;
}
```

---

## 3. Notification Permission Component

```typescript
// features/notifications/permission-prompt/permission-prompt.component.ts
import { Component, OnInit, Output, EventEmitter } from '@angular/core';
import { PushNotificationService } from '../../../core/push/push-notification.service';

@Component({
  selector: 'app-permission-prompt',
  template: `
    <div class="permission-prompt" *ngIf="shouldShow">
      <div class="prompt-icon">🔔</div>
      <div class="prompt-content">
        <h4>รับการแจ้งเตือน</h4>
        <p>อนุญาตให้แอปส่งการแจ้งเตือนสำคัญ เช่น สถานะคำสั่งซื้อ โปรโมชัน</p>
      </div>
      <div class="prompt-actions">
        <button class="btn-allow" (click)="allow()">อนุญาต</button>
        <button class="btn-dismiss" (click)="dismiss()">ไม่ต้องการ</button>
      </div>
    </div>
  `,
  styles: [`
    .permission-prompt {
      position: fixed;
      bottom: 80px;
      right: 16px;
      background: white;
      border-radius: 12px;
      padding: 16px;
      display: flex;
      gap: 12px;
      align-items: flex-start;
      box-shadow: 0 8px 32px rgba(0,0,0,0.15);
      z-index: 1000;
      max-width: 360px;
      border: 1px solid #eee;
    }
    .prompt-icon { font-size: 28px; }
    .prompt-content h4 { margin: 0 0 4px; font-size: 15px; }
    .prompt-content p { margin: 0; font-size: 13px; color: #666; }
    .prompt-actions { display: flex; gap: 8px; margin-top: 8px; }
    .btn-allow {
      padding: 6px 16px;
      background: #2196f3;
      color: white;
      border: none;
      border-radius: 6px;
      cursor: pointer;
      font-size: 14px;
    }
    .btn-dismiss {
      padding: 6px 16px;
      background: transparent;
      border: 1px solid #ddd;
      border-radius: 6px;
      cursor: pointer;
      font-size: 14px;
      color: #666;
    }
  `]
})
export class PermissionPromptComponent implements OnInit {
  @Output() allowed = new EventEmitter<void>();
  @Output() dismissed = new EventEmitter<void>();

  shouldShow = false;

  constructor(private pushService: PushNotificationService) {}

  ngOnInit(): void {
    // แสดงถ้า support และยังไม่ได้ตัดสินใจ
    if (this.pushService.isSupported && 
        this.pushService.currentPermission === 'default' &&
        !localStorage.getItem('push_dismissed')) {
      
      // รอ 5 วินาทีก่อนแสดง
      setTimeout(() => { this.shouldShow = true; }, 5000);
    }
  }

  async allow(): Promise<void> {
    this.shouldShow = false;
    const result = await this.pushService.requestPermission();
    
    if (result.granted) {
      this.allowed.emit();
    }
  }

  dismiss(): void {
    this.shouldShow = false;
    localStorage.setItem('push_dismissed', 'true');
    this.dismissed.emit();
  }
}
```

---

## 4. Firebase Cloud Messaging (FCM)

```bash
npm install firebase
```

```typescript
// core/push/fcm.service.ts
import { Injectable } from '@angular/core';
import { initializeApp } from 'firebase/app';
import { getMessaging, getToken, onMessage, Messaging } from 'firebase/messaging';
import { HttpClient } from '@angular/common/http';
import { Subject } from 'rxjs';

const firebaseConfig = {
  apiKey: "YOUR_API_KEY",
  authDomain: "your-app.firebaseapp.com",
  projectId: "your-app",
  messagingSenderId: "YOUR_SENDER_ID",
  appId: "YOUR_APP_ID"
};

@Injectable({ providedIn: 'root' })
export class FCMService {
  private messaging: Messaging;
  private messageReceived$ = new Subject<any>();

  messages$ = this.messageReceived$.asObservable();

  constructor(private http: HttpClient) {
    const app = initializeApp(firebaseConfig);
    this.messaging = getMessaging(app);
    this.setupForegroundListener();
  }

  async requestPermissionAndGetToken(): Promise<string | null> {
    try {
      const permission = await Notification.requestPermission();
      if (permission !== 'granted') return null;

      const token = await getToken(this.messaging, {
        vapidKey: 'YOUR_FCM_VAPID_KEY'
      });

      if (token) {
        await this.registerToken(token);
        return token;
      }
      return null;
    } catch (error) {
      console.error('FCM token error:', error);
      return null;
    }
  }

  private async registerToken(token: string): Promise<void> {
    await this.http.post('/api/push/fcm-token', { token }).toPromise();
  }

  private setupForegroundListener(): void {
    onMessage(this.messaging, (payload) => {
      console.log('FCM Message received:', payload);
      this.messageReceived$.next(payload);
      
      // แสดง notification เมื่อแอปเปิดอยู่
      if (payload.notification) {
        this.showLocalNotification(payload.notification);
      }
    });
  }

  private showLocalNotification(notification: { title?: string; body?: string }): void {
    if (Notification.permission === 'granted') {
      new Notification(notification.title || 'แจ้งเตือน', {
        body: notification.body,
        icon: '/assets/icons/icon-192x192.png'
      });
    }
  }
}
```

---

## 5. Service Worker สำหรับ Push

```javascript
// src/firebase-messaging-sw.js
importScripts('https://www.gstatic.com/firebasejs/9.0.0/firebase-app-compat.js');
importScripts('https://www.gstatic.com/firebasejs/9.0.0/firebase-messaging-compat.js');

firebase.initializeApp({
  apiKey: 'YOUR_API_KEY',
  projectId: 'your-app',
  messagingSenderId: 'YOUR_SENDER_ID',
  appId: 'YOUR_APP_ID'
});

const messaging = firebase.messaging();

// Handle background messages
messaging.onBackgroundMessage((payload) => {
  console.log('Background message:', payload);
  
  const { title, body, icon } = payload.notification || {};
  
  self.registration.showNotification(title || 'แจ้งเตือน', {
    body,
    icon: icon || '/assets/icons/icon-192x192.png',
    badge: '/assets/icons/badge-72x72.png',
    data: payload.data,
    actions: [
      { action: 'open', title: 'เปิดแอป' },
      { action: 'dismiss', title: 'ปิด' }
    ]
  });
});

// Handle notification click
self.addEventListener('notificationclick', (event) => {
  event.notification.close();
  
  if (event.action === 'open' || !event.action) {
    const url = event.notification.data?.url || '/';
    event.waitUntil(
      clients.openWindow(url)
    );
  }
});
```

---

## สรุป

| วิธีการ | รายละเอียด |
|--------|-----------|
| Web Push API | Standard, ใช้ Service Worker |
| Firebase FCM | Google, ใช้ง่าย |
| OneSignal | Third-party, มี dashboard |
| Push7 | เน้นไทย |

### Notification Best Practices

1. ขอ permission หลัง user engage กับแอปก่อน
2. อธิบายว่าจะส่งอะไร ก่อนขอ permission
3. ส่งเฉพาะที่ user ต้องการ
4. Allow unsubscribe ง่าย
5. Personalize notifications
6. ไม่ส่งมากเกินไป (max 1-3 ต่อวัน)
