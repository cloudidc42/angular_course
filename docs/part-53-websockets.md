# Part 53: WebSocket ใน Angular

## บทนำ

WebSocket ช่วยให้แอปพลิเคชันสามารถสื่อสารแบบ real-time กับ server ได้ Angular รองรับ WebSocket ผ่าน RxJS `webSocket` operator และยังใช้ Socket.io ได้ด้วย

## 1. RxJS WebSocket

RxJS มี `webSocket` ฟังก์ชันที่ทำให้ WebSocket เป็น Observable

```bash
# ไม่ต้องติดตั้งเพิ่ม เพราะมากับ RxJS แล้ว
```

```typescript
// websocket.service.ts
import { Injectable, OnDestroy } from '@angular/core';
import { webSocket, WebSocketSubject } from 'rxjs/webSocket';
import { Observable, Subject, timer, EMPTY } from 'rxjs';
import { retryWhen, delayWhen, tap, switchAll, catchError } from 'rxjs/operators';
import { environment } from '../environments/environment';

export interface WebSocketMessage<T = any> {
  type: string;
  payload: T;
  timestamp?: number;
}

@Injectable({ providedIn: 'root' })
export class WebSocketService implements OnDestroy {
  private socket$: WebSocketSubject<WebSocketMessage> | null = null;
  private messagesSubject$ = new Subject<Observable<WebSocketMessage>>();
  private readonly WS_ENDPOINT = environment.wsUrl || 'ws://localhost:8080';
  private readonly RECONNECT_INTERVAL = 2000;

  messages$ = this.messagesSubject$.pipe(
    switchAll(),
    catchError(e => {
      console.error('WebSocket error:', e);
      return EMPTY;
    })
  );

  connect(): void {
    if (!this.socket$ || this.socket$.closed) {
      this.socket$ = this.createSocket();
      this.messagesSubject$.next(this.socket$.pipe(
        retryWhen(errors =>
          errors.pipe(
            tap(err => console.log('WebSocket error, reconnecting...', err)),
            delayWhen(() => timer(this.RECONNECT_INTERVAL))
          )
        )
      ));
    }
  }

  disconnect(): void {
    if (this.socket$) {
      this.socket$.complete();
      this.socket$ = null;
    }
  }

  sendMessage(message: WebSocketMessage): void {
    if (this.socket$) {
      this.socket$.next(message);
    }
  }

  private createSocket(): WebSocketSubject<WebSocketMessage> {
    return webSocket({
      url: this.WS_ENDPOINT,
      openObserver: {
        next: () => console.log('WebSocket connected')
      },
      closeObserver: {
        next: () => console.log('WebSocket disconnected')
      }
    });
  }

  ngOnDestroy(): void {
    this.disconnect();
  }
}
```

## 2. Chat Application ด้วย WebSocket

```typescript
// chat.service.ts
import { Injectable } from '@angular/core';
import { Observable, Subject } from 'rxjs';
import { filter, map } from 'rxjs/operators';
import { WebSocketService } from './websocket.service';

export interface ChatMessage {
  id: string;
  userId: string;
  username: string;
  text: string;
  timestamp: Date;
  room: string;
}

export interface UserJoinedEvent {
  userId: string;
  username: string;
  room: string;
}

@Injectable({ providedIn: 'root' })
export class ChatService {
  constructor(private wsService: WebSocketService) {}

  connect(): void {
    this.wsService.connect();
  }

  joinRoom(room: string, username: string): void {
    this.wsService.sendMessage({
      type: 'JOIN_ROOM',
      payload: { room, username }
    });
  }

  sendMessage(room: string, text: string): void {
    this.wsService.sendMessage({
      type: 'CHAT_MESSAGE',
      payload: { room, text, timestamp: Date.now() }
    });
  }

  getMessages(room: string): Observable<ChatMessage> {
    return this.wsService.messages$.pipe(
      filter(msg => msg.type === 'CHAT_MESSAGE' && msg.payload.room === room),
      map(msg => ({
        ...msg.payload,
        timestamp: new Date(msg.payload.timestamp)
      }))
    );
  }

  getUserJoined(): Observable<UserJoinedEvent> {
    return this.wsService.messages$.pipe(
      filter(msg => msg.type === 'USER_JOINED'),
      map(msg => msg.payload)
    );
  }

  getUserLeft(): Observable<UserJoinedEvent> {
    return this.wsService.messages$.pipe(
      filter(msg => msg.type === 'USER_LEFT'),
      map(msg => msg.payload)
    );
  }

  leaveRoom(room: string): void {
    this.wsService.sendMessage({
      type: 'LEAVE_ROOM',
      payload: { room }
    });
  }
}
```

```typescript
// chat-room.component.ts
import { Component, OnInit, OnDestroy, ViewChild, ElementRef } from '@angular/core';
import { CommonModule } from '@angular/common';
import { FormsModule } from '@angular/forms';
import { Subject } from 'rxjs';
import { takeUntil } from 'rxjs/operators';
import { ChatService, ChatMessage } from './chat.service';

@Component({
  selector: 'app-chat-room',
  standalone: true,
  imports: [CommonModule, FormsModule],
  template: `
    <div class="chat-container">
      <div class="chat-header">
        <h2>ห้องสนทนา: {{ roomName }}</h2>
        <span class="online-count">ออนไลน์: {{ onlineUsers.length }} คน</span>
      </div>

      <div class="messages" #messageContainer>
        <div
          *ngFor="let msg of messages"
          class="message"
          [class.own-message]="msg.userId === currentUserId"
        >
          <div class="message-header">
            <strong>{{ msg.username }}</strong>
            <span class="timestamp">{{ msg.timestamp | date:'HH:mm' }}</span>
          </div>
          <div class="message-body">{{ msg.text }}</div>
        </div>
      </div>

      <div class="input-area">
        <input
          [(ngModel)]="newMessage"
          (keyup.enter)="sendMessage()"
          placeholder="พิมพ์ข้อความ..."
          class="message-input"
        />
        <button (click)="sendMessage()" [disabled]="!newMessage.trim()">
          ส่ง
        </button>
      </div>
    </div>
  `,
  styles: [`
    .chat-container {
      display: flex;
      flex-direction: column;
      height: 600px;
      border: 1px solid #ddd;
      border-radius: 8px;
      overflow: hidden;
    }
    .chat-header {
      display: flex;
      justify-content: space-between;
      padding: 1rem;
      background: #007bff;
      color: white;
    }
    .messages {
      flex: 1;
      overflow-y: auto;
      padding: 1rem;
    }
    .message {
      margin-bottom: 1rem;
      padding: 0.5rem;
      border-radius: 8px;
      background: #f8f9fa;
    }
    .own-message {
      background: #dcf8c6;
      margin-left: 20%;
    }
    .message-header {
      display: flex;
      justify-content: space-between;
      font-size: 0.85rem;
    }
    .input-area {
      display: flex;
      padding: 1rem;
      border-top: 1px solid #ddd;
    }
    .message-input {
      flex: 1;
      padding: 0.5rem;
      border: 1px solid #ddd;
      border-radius: 4px;
      margin-right: 0.5rem;
    }
    button {
      padding: 0.5rem 1rem;
      background: #007bff;
      color: white;
      border: none;
      border-radius: 4px;
      cursor: pointer;
    }
    button:disabled { opacity: 0.5; }
  `]
})
export class ChatRoomComponent implements OnInit, OnDestroy {
  @ViewChild('messageContainer') messageContainer!: ElementRef;
  
  messages: ChatMessage[] = [];
  onlineUsers: string[] = [];
  newMessage = '';
  roomName = 'general';
  currentUserId = 'user123';

  private destroy$ = new Subject<void>();

  constructor(private chatService: ChatService) {}

  ngOnInit(): void {
    this.chatService.connect();
    this.chatService.joinRoom(this.roomName, 'ผู้ใช้');

    this.chatService.getMessages(this.roomName).pipe(
      takeUntil(this.destroy$)
    ).subscribe(msg => {
      this.messages.push(msg);
      this.scrollToBottom();
    });

    this.chatService.getUserJoined().pipe(
      takeUntil(this.destroy$)
    ).subscribe(event => {
      this.onlineUsers.push(event.userId);
    });
  }

  ngOnDestroy(): void {
    this.chatService.leaveRoom(this.roomName);
    this.destroy$.next();
    this.destroy$.complete();
  }

  sendMessage(): void {
    if (this.newMessage.trim()) {
      this.chatService.sendMessage(this.roomName, this.newMessage);
      this.newMessage = '';
    }
  }

  private scrollToBottom(): void {
    setTimeout(() => {
      if (this.messageContainer) {
        const el = this.messageContainer.nativeElement;
        el.scrollTop = el.scrollHeight;
      }
    }, 0);
  }
}
```

## 3. Socket.io Integration

```bash
npm install socket.io-client
npm install --save-dev @types/socket.io-client
```

```typescript
// socket.service.ts
import { Injectable, NgZone, OnDestroy } from '@angular/core';
import { io, Socket } from 'socket.io-client';
import { Observable, Subject, fromEvent } from 'rxjs';
import { takeUntil } from 'rxjs/operators';

@Injectable({ providedIn: 'root' })
export class SocketService implements OnDestroy {
  private socket: Socket | null = null;
  private destroy$ = new Subject<void>();
  private connected$ = new Subject<boolean>();

  isConnected$ = this.connected$.asObservable();

  constructor(private ngZone: NgZone) {}

  connect(url: string, options?: any): void {
    this.ngZone.runOutsideAngular(() => {
      this.socket = io(url, {
        transports: ['websocket'],
        autoConnect: true,
        ...options
      });

      this.socket.on('connect', () => {
        this.ngZone.run(() => {
          console.log('Socket.io connected:', this.socket?.id);
          this.connected$.next(true);
        });
      });

      this.socket.on('disconnect', (reason) => {
        this.ngZone.run(() => {
          console.log('Socket.io disconnected:', reason);
          this.connected$.next(false);
        });
      });

      this.socket.on('connect_error', (error) => {
        this.ngZone.run(() => {
          console.error('Socket.io connection error:', error);
        });
      });
    });
  }

  emit(event: string, data?: any): void {
    if (this.socket?.connected) {
      this.socket.emit(event, data);
    }
  }

  on<T>(event: string): Observable<T> {
    return new Observable<T>(observer => {
      this.socket?.on(event, (data: T) => {
        this.ngZone.run(() => observer.next(data));
      });

      return () => {
        this.socket?.off(event);
      };
    });
  }

  once<T>(event: string): Observable<T> {
    return new Observable<T>(observer => {
      this.socket?.once(event, (data: T) => {
        this.ngZone.run(() => {
          observer.next(data);
          observer.complete();
        });
      });
    });
  }

  disconnect(): void {
    if (this.socket) {
      this.socket.disconnect();
      this.socket = null;
    }
  }

  ngOnDestroy(): void {
    this.disconnect();
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

## 4. Real-time Dashboard

```typescript
// realtime-dashboard.component.ts
import { Component, OnInit, OnDestroy } from '@angular/core';
import { CommonModule } from '@angular/common';
import { Subject } from 'rxjs';
import { takeUntil } from 'rxjs/operators';
import { SocketService } from './socket.service';

interface DashboardStats {
  activeUsers: number;
  totalOrders: number;
  revenue: number;
  serverLoad: number;
}

interface Notification {
  id: string;
  type: 'info' | 'warning' | 'error';
  message: string;
  timestamp: Date;
}

@Component({
  selector: 'app-realtime-dashboard',
  standalone: true,
  imports: [CommonModule],
  template: `
    <div class="dashboard">
      <h1>Dashboard แบบ Real-time</h1>
      
      <div class="stats-grid">
        <div class="stat-card">
          <h3>ผู้ใช้งานออนไลน์</h3>
          <div class="stat-value">{{ stats.activeUsers | number }}</div>
        </div>
        <div class="stat-card">
          <h3>ออเดอร์วันนี้</h3>
          <div class="stat-value">{{ stats.totalOrders | number }}</div>
        </div>
        <div class="stat-card">
          <h3>รายได้</h3>
          <div class="stat-value">{{ stats.revenue | currency:'THB' }}</div>
        </div>
        <div class="stat-card" [class.warning]="stats.serverLoad > 80">
          <h3>โหลดเซิร์ฟเวอร์</h3>
          <div class="stat-value">{{ stats.serverLoad }}%</div>
        </div>
      </div>

      <div class="notifications">
        <h2>การแจ้งเตือน</h2>
        <div
          *ngFor="let notif of notifications"
          class="notification"
          [class]="'notification--' + notif.type"
        >
          <span>{{ notif.message }}</span>
          <small>{{ notif.timestamp | date:'HH:mm:ss' }}</small>
        </div>
      </div>
    </div>
  `,
  styles: [`
    .stats-grid {
      display: grid;
      grid-template-columns: repeat(4, 1fr);
      gap: 1rem;
      margin: 1rem 0;
    }
    .stat-card {
      padding: 1.5rem;
      background: white;
      border-radius: 8px;
      box-shadow: 0 2px 4px rgba(0,0,0,0.1);
      text-align: center;
    }
    .stat-value { font-size: 2rem; font-weight: bold; color: #007bff; }
    .warning .stat-value { color: #dc3545; }
    .notification {
      padding: 0.75rem;
      margin: 0.5rem 0;
      border-radius: 4px;
      display: flex;
      justify-content: space-between;
    }
    .notification--info { background: #cce5ff; }
    .notification--warning { background: #fff3cd; }
    .notification--error { background: #f8d7da; }
  `]
})
export class RealtimeDashboardComponent implements OnInit, OnDestroy {
  stats: DashboardStats = {
    activeUsers: 0,
    totalOrders: 0,
    revenue: 0,
    serverLoad: 0
  };
  
  notifications: Notification[] = [];
  private destroy$ = new Subject<void>();

  constructor(private socketService: SocketService) {}

  ngOnInit(): void {
    this.socketService.connect('http://localhost:3000');

    this.socketService.on<DashboardStats>('stats_update').pipe(
      takeUntil(this.destroy$)
    ).subscribe(stats => {
      this.stats = stats;
    });

    this.socketService.on<Notification>('notification').pipe(
      takeUntil(this.destroy$)
    ).subscribe(notif => {
      this.notifications.unshift({
        ...notif,
        timestamp: new Date()
      });
      
      // เก็บแค่ 20 รายการล่าสุด
      if (this.notifications.length > 20) {
        this.notifications = this.notifications.slice(0, 20);
      }
    });
  }

  ngOnDestroy(): void {
    this.socketService.disconnect();
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

## 5. WebSocket Connection Status

```typescript
// connection-status.component.ts
import { Component, OnInit, OnDestroy } from '@angular/core';
import { CommonModule } from '@angular/common';
import { Subject, fromEvent, merge, of } from 'rxjs';
import { map, startWith, takeUntil } from 'rxjs/operators';

@Component({
  selector: 'app-connection-status',
  standalone: true,
  imports: [CommonModule],
  template: `
    <div class="connection-status" [class.offline]="!isOnline">
      <span class="status-dot"></span>
      {{ isOnline ? 'เชื่อมต่อแล้ว' : 'ขาดการเชื่อมต่อ' }}
    </div>
  `,
  styles: [`
    .connection-status {
      display: flex;
      align-items: center;
      gap: 0.5rem;
      padding: 0.25rem 0.75rem;
      border-radius: 9999px;
      background: #28a745;
      color: white;
      font-size: 0.85rem;
    }
    .offline { background: #dc3545; }
    .status-dot {
      width: 8px;
      height: 8px;
      border-radius: 50%;
      background: white;
    }
  `]
})
export class ConnectionStatusComponent implements OnInit, OnDestroy {
  isOnline = navigator.onLine;
  private destroy$ = new Subject<void>();

  ngOnInit(): void {
    merge(
      fromEvent(window, 'online').pipe(map(() => true)),
      fromEvent(window, 'offline').pipe(map(() => false))
    ).pipe(
      startWith(navigator.onLine),
      takeUntil(this.destroy$)
    ).subscribe(isOnline => {
      this.isOnline = isOnline;
    });
  }

  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

## 6. Heartbeat / Ping-Pong

```typescript
// heartbeat.service.ts
import { Injectable, OnDestroy } from '@angular/core';
import { interval, Subject, Subscription } from 'rxjs';
import { takeUntil } from 'rxjs/operators';
import { WebSocketService } from './websocket.service';

@Injectable({ providedIn: 'root' })
export class HeartbeatService implements OnDestroy {
  private heartbeatSubscription: Subscription | null = null;
  private destroy$ = new Subject<void>();
  private readonly HEARTBEAT_INTERVAL = 30000; // 30 วินาที

  constructor(private wsService: WebSocketService) {}

  startHeartbeat(): void {
    this.stopHeartbeat();
    
    this.heartbeatSubscription = interval(this.HEARTBEAT_INTERVAL).pipe(
      takeUntil(this.destroy$)
    ).subscribe(() => {
      this.wsService.sendMessage({
        type: 'PING',
        payload: { timestamp: Date.now() }
      });
    });
  }

  stopHeartbeat(): void {
    if (this.heartbeatSubscription) {
      this.heartbeatSubscription.unsubscribe();
      this.heartbeatSubscription = null;
    }
  }

  ngOnDestroy(): void {
    this.stopHeartbeat();
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

## สรุป

| Feature | RxJS webSocket | Socket.io |
|---------|---------------|-----------|
| ความซับซ้อน | ต่ำ | ปานกลาง |
| Auto-reconnect | ต้องทำเอง | Built-in |
| Room/Namespace | ไม่มี | มี |
| Fallback | ไม่มี | HTTP long-polling |
| TypeScript | ดี | ดี |

WebSocket เหมาะกับ chat, live notifications, real-time data, และ collaborative features
