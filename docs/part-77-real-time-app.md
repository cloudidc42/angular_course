# Part 77: Real-Time App ด้วย Socket.io

## Real-Time คืออะไร

Real-time applications ส่ง/รับข้อมูลแบบ instant โดยไม่ต้อง refresh เช่น Live Chat, Notifications, Live Dashboard

---

## 1. ติดตั้ง Socket.io

```bash
# Client side
npm install socket.io-client

# Server side (Node.js)
npm install socket.io express cors
npm install -D @types/socket.io-client
```

---

## 2. Socket Service

```typescript
// app/services/socket.service.ts
import { Injectable, NgZone, OnDestroy } from '@angular/core';
import { Observable, Subject, BehaviorSubject, fromEvent } from 'rxjs';
import { map, filter, takeUntil } from 'rxjs/operators';
import { io, Socket } from 'socket.io-client';
import { environment } from '../../environments/environment';

export interface SocketMessage {
  event: string;
  data: any;
}

@Injectable({ providedIn: 'root' })
export class SocketService implements OnDestroy {
  private socket!: Socket;
  private destroy$ = new Subject<void>();
  
  public connected$ = new BehaviorSubject<boolean>(false);
  public reconnecting$ = new BehaviorSubject<boolean>(false);

  constructor(private ngZone: NgZone) {}

  connect(userId?: string): void {
    this.socket = io(environment.socketUrl, {
      transports: ['websocket', 'polling'],
      auth: {
        userId,
        token: localStorage.getItem('auth_token')
      },
      reconnection: true,
      reconnectionAttempts: 5,
      reconnectionDelay: 1000
    });

    this.setupListeners();
  }

  private setupListeners(): void {
    this.socket.on('connect', () => {
      this.ngZone.run(() => {
        this.connected$.next(true);
        this.reconnecting$.next(false);
        console.log('Socket connected:', this.socket.id);
      });
    });

    this.socket.on('disconnect', (reason) => {
      this.ngZone.run(() => {
        this.connected$.next(false);
        console.log('Socket disconnected:', reason);
      });
    });

    this.socket.on('reconnecting', (attempt) => {
      this.ngZone.run(() => {
        this.reconnecting$.next(true);
        console.log('Reconnecting attempt:', attempt);
      });
    });

    this.socket.on('connect_error', (error) => {
      console.error('Connection error:', error);
    });
  }

  // ส่ง event
  emit(event: string, data?: any): void {
    if (!this.socket?.connected) {
      console.warn('Socket not connected');
      return;
    }
    this.socket.emit(event, data);
  }

  // ส่งและรอ response (acknowledgement)
  emitWithAck<T>(event: string, data?: any): Promise<T> {
    return new Promise((resolve, reject) => {
      const timeout = setTimeout(() => {
        reject(new Error('Socket timeout'));
      }, 5000);

      this.socket.emit(event, data, (response: T) => {
        clearTimeout(timeout);
        resolve(response);
      });
    });
  }

  // รับ event เป็น Observable
  on<T>(event: string): Observable<T> {
    return new Observable(observer => {
      const handler = (data: T) => {
        this.ngZone.run(() => observer.next(data));
      };
      
      this.socket.on(event, handler);
      
      return () => {
        this.socket.off(event, handler);
      };
    }).pipe(takeUntil(this.destroy$));
  }

  // รับ event ครั้งเดียว
  once<T>(event: string): Promise<T> {
    return new Promise(resolve => {
      this.socket.once(event, (data: T) => {
        this.ngZone.run(() => resolve(data));
      });
    });
  }

  joinRoom(room: string): void {
    this.emit('join-room', { room });
  }

  leaveRoom(room: string): void {
    this.emit('leave-room', { room });
  }

  disconnect(): void {
    this.socket?.disconnect();
  }

  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
    this.disconnect();
  }
}
```

---

## 3. Live Chat Component

```typescript
// app/components/live-chat/live-chat.component.ts
import { Component, OnInit, OnDestroy, ViewChild, ElementRef, ChangeDetectorRef } from '@angular/core';
import { FormControl, Validators } from '@angular/forms';
import { Subject } from 'rxjs';
import { takeUntil } from 'rxjs/operators';
import { SocketService } from '../../services/socket.service';

interface ChatMessage {
  id: string;
  userId: string;
  username: string;
  content: string;
  timestamp: Date;
  type: 'text' | 'system' | 'image';
  status?: 'sending' | 'sent' | 'error';
  avatar?: string;
}

interface TypingUser {
  userId: string;
  username: string;
}

@Component({
  selector: 'app-live-chat',
  template: `
    <div class="chat-app">
      <!-- Header -->
      <div class="chat-header">
        <div class="room-info">
          <h3>ห้องแชท: {{ currentRoom }}</h3>
          <span class="online-count">
            <span class="dot online"></span>
            {{ onlineUsers.length }} คนออนไลน์
          </span>
        </div>
        
        <div class="connection-status" [class]="connectionStatus">
          {{ connectionStatusText }}
        </div>
      </div>

      <!-- Online Users Sidebar -->
      <div class="chat-layout">
        <div class="users-sidebar">
          <h4>ผู้ใช้ออนไลน์ ({{ onlineUsers.length }})</h4>
          <div 
            *ngFor="let user of onlineUsers" 
            class="user-item"
            [class.typing]="isTyping(user.id)"
          >
            <div class="user-avatar">{{ user.username[0] }}</div>
            <div class="user-info">
              <span class="username">{{ user.username }}</span>
              <span class="typing-indicator" *ngIf="isTyping(user.id)">
                กำลังพิมพ์...
              </span>
            </div>
          </div>
        </div>

        <!-- Messages Area -->
        <div class="messages-area" #messagesContainer>
          <div class="messages-list">
            <ng-container *ngFor="let msg of messages; trackBy: trackById">
              <!-- System Message -->
              <div *ngIf="msg.type === 'system'" class="system-message">
                {{ msg.content }}
              </div>
              
              <!-- Chat Message -->
              <div 
                *ngIf="msg.type !== 'system'"
                class="message-bubble"
                [class.own]="msg.userId === currentUserId"
                [class.sending]="msg.status === 'sending'"
                [class.error]="msg.status === 'error'"
              >
                <img 
                  *ngIf="msg.userId !== currentUserId"
                  class="message-avatar"
                  [src]="msg.avatar || 'default-avatar.png'"
                  [alt]="msg.username"
                >
                <div class="message-content">
                  <span 
                    class="sender-name"
                    *ngIf="msg.userId !== currentUserId"
                  >
                    {{ msg.username }}
                  </span>
                  <p class="message-text">{{ msg.content }}</p>
                  <div class="message-meta">
                    <span class="message-time">
                      {{ msg.timestamp | date:'HH:mm' }}
                    </span>
                    <span class="message-status" *ngIf="msg.userId === currentUserId">
                      {{ getStatusIcon(msg.status) }}
                    </span>
                  </div>
                </div>
              </div>
            </ng-container>
          </div>

          <!-- Typing Indicator -->
          <div class="typing-area" *ngIf="typingUsers.length > 0">
            <div class="typing-dots">
              <span></span><span></span><span></span>
            </div>
            <span class="typing-text">
              {{ getTypingText() }}
            </span>
          </div>
        </div>
      </div>

      <!-- Input Area -->
      <div class="chat-input-area">
        <div class="input-row">
          <button class="btn-emoji" (click)="toggleEmojiPicker()">😊</button>
          <input
            [formControl]="messageInput"
            placeholder="พิมพ์ข้อความ..."
            (keyup.enter)="sendMessage()"
            (input)="onTyping()"
            class="message-input"
          >
          <button 
            class="btn-send"
            (click)="sendMessage()"
            [disabled]="!messageInput.value.trim() || !isConnected"
          >
            ส่ง
          </button>
        </div>
      </div>
    </div>
  `,
  styles: [`
    .chat-app {
      height: 100vh;
      display: flex;
      flex-direction: column;
      max-width: 900px;
      margin: 0 auto;
      border: 1px solid #ddd;
      border-radius: 12px;
      overflow: hidden;
    }
    .chat-header {
      display: flex;
      justify-content: space-between;
      align-items: center;
      padding: 12px 16px;
      background: #2196f3;
      color: white;
    }
    .online-count { font-size: 13px; opacity: 0.9; }
    .dot { width: 8px; height: 8px; border-radius: 50%; display: inline-block; }
    .dot.online { background: #4caf50; }
    .connection-status {
      padding: 4px 12px;
      border-radius: 12px;
      font-size: 12px;
      background: rgba(255,255,255,0.2);
    }
    .chat-layout { display: flex; flex: 1; overflow: hidden; }
    .users-sidebar {
      width: 200px;
      border-right: 1px solid #eee;
      padding: 12px;
      overflow-y: auto;
    }
    .users-sidebar h4 { margin: 0 0 12px; font-size: 14px; color: #666; }
    .user-item {
      display: flex;
      align-items: center;
      gap: 8px;
      padding: 6px;
      border-radius: 6px;
      margin-bottom: 4px;
    }
    .user-item.typing { background: #f5f5f5; }
    .user-avatar {
      width: 32px; height: 32px;
      background: #2196f3; color: white;
      border-radius: 50%;
      display: flex; align-items: center; justify-content: center;
      font-size: 14px; font-weight: bold;
    }
    .typing-indicator { font-size: 11px; color: #999; }
    .messages-area { flex: 1; display: flex; flex-direction: column; overflow: hidden; }
    .messages-list { flex: 1; overflow-y: auto; padding: 16px; }
    .message-bubble {
      display: flex;
      gap: 8px;
      margin-bottom: 12px;
      align-items: flex-end;
    }
    .message-bubble.own { flex-direction: row-reverse; }
    .message-avatar { width: 32px; height: 32px; border-radius: 50%; }
    .message-content { max-width: 70%; }
    .sender-name { font-size: 12px; color: #999; margin-bottom: 4px; display: block; }
    .message-text {
      padding: 8px 12px;
      background: white;
      border-radius: 12px;
      border-bottom-left-radius: 4px;
      box-shadow: 0 1px 2px rgba(0,0,0,0.1);
      margin: 0;
      font-size: 14px;
    }
    .own .message-text { background: #2196f3; color: white; border-radius: 12px; border-bottom-right-radius: 4px; }
    .message-meta { display: flex; gap: 4px; font-size: 11px; color: #999; margin-top: 2px; }
    .own .message-meta { justify-content: flex-end; }
    .system-message {
      text-align: center;
      font-size: 12px;
      color: #999;
      padding: 8px;
      background: #f5f5f5;
      border-radius: 12px;
      margin: 8px 0;
    }
    .typing-area { display: flex; align-items: center; gap: 8px; padding: 8px 16px; }
    .typing-dots span {
      width: 6px; height: 6px;
      background: #999; border-radius: 50%;
      display: inline-block;
      animation: bounce 1.4s infinite;
    }
    .typing-dots span:nth-child(2) { animation-delay: 0.2s; }
    .typing-dots span:nth-child(3) { animation-delay: 0.4s; }
    @keyframes bounce { 0%, 60%, 100% { transform: translateY(0); } 30% { transform: translateY(-8px); } }
    .typing-text { font-size: 12px; color: #999; }
    .chat-input-area {
      border-top: 1px solid #eee;
      padding: 12px 16px;
    }
    .input-row { display: flex; gap: 8px; align-items: center; }
    .message-input {
      flex: 1;
      padding: 10px 14px;
      border: 1px solid #ddd;
      border-radius: 24px;
      font-size: 14px;
      outline: none;
    }
    .message-input:focus { border-color: #2196f3; }
    .btn-send {
      padding: 10px 20px;
      background: #2196f3;
      color: white;
      border: none;
      border-radius: 24px;
      cursor: pointer;
      font-size: 14px;
    }
    .btn-send:disabled { opacity: 0.5; cursor: not-allowed; }
    .btn-emoji { background: none; border: none; font-size: 20px; cursor: pointer; }
    .message-bubble.sending { opacity: 0.7; }
    .message-bubble.error .message-text { border: 1px solid #f44336; }
  `]
})
export class LiveChatComponent implements OnInit, OnDestroy {
  @ViewChild('messagesContainer') messagesContainer!: ElementRef;
  
  messages: ChatMessage[] = [];
  onlineUsers: { id: string; username: string }[] = [];
  typingUsers: TypingUser[] = [];
  messageInput = new FormControl('', Validators.required);
  
  currentUserId = 'user-' + Math.random().toString(36).slice(2, 8);
  currentUsername = 'ผู้ใช้ ' + Math.floor(Math.random() * 1000);
  currentRoom = 'general';
  isConnected = false;
  
  private destroy$ = new Subject<void>();
  private typingTimer: any;

  constructor(
    private socketService: SocketService,
    private cdr: ChangeDetectorRef
  ) {}

  ngOnInit(): void {
    this.socketService.connect(this.currentUserId);
    
    // รับ connection status
    this.socketService.connected$
      .pipe(takeUntil(this.destroy$))
      .subscribe(connected => {
        this.isConnected = connected;
        if (connected) {
          this.joinRoom();
        }
        this.cdr.markForCheck();
      });

    // รับข้อความ
    this.socketService.on<ChatMessage>('new-message')
      .pipe(takeUntil(this.destroy$))
      .subscribe(msg => {
        this.messages.push({ ...msg, timestamp: new Date(msg.timestamp) });
        this.cdr.markForCheck();
        this.scrollToBottom();
      });

    // รับ online users
    this.socketService.on<typeof this.onlineUsers>('online-users')
      .pipe(takeUntil(this.destroy$))
      .subscribe(users => {
        this.onlineUsers = users;
        this.cdr.markForCheck();
      });

    // รับ typing indicators
    this.socketService.on<TypingUser>('user-typing')
      .pipe(takeUntil(this.destroy$))
      .subscribe(user => {
        if (!this.typingUsers.find(u => u.userId === user.userId)) {
          this.typingUsers.push(user);
        }
        this.cdr.markForCheck();
      });

    this.socketService.on<{ userId: string }>('user-stopped-typing')
      .pipe(takeUntil(this.destroy$))
      .subscribe(({ userId }) => {
        this.typingUsers = this.typingUsers.filter(u => u.userId !== userId);
        this.cdr.markForCheck();
      });

    // รับ system messages
    this.socketService.on<{ content: string }>('system-message')
      .pipe(takeUntil(this.destroy$))
      .subscribe(({ content }) => {
        this.messages.push({
          id: Date.now().toString(),
          userId: 'system',
          username: 'System',
          content,
          timestamp: new Date(),
          type: 'system'
        });
        this.cdr.markForCheck();
        this.scrollToBottom();
      });
  }

  private joinRoom(): void {
    this.socketService.emit('join-room', {
      room: this.currentRoom,
      userId: this.currentUserId,
      username: this.currentUsername
    });
  }

  async sendMessage(): Promise<void> {
    const content = this.messageInput.value?.trim();
    if (!content || !this.isConnected) return;

    const message: ChatMessage = {
      id: Date.now().toString(),
      userId: this.currentUserId,
      username: this.currentUsername,
      content,
      timestamp: new Date(),
      type: 'text',
      status: 'sending'
    };

    this.messages.push(message);
    this.messageInput.reset();
    this.scrollToBottom();

    try {
      await this.socketService.emitWithAck('send-message', {
        room: this.currentRoom,
        content,
        userId: this.currentUserId,
        username: this.currentUsername
      });
      message.status = 'sent';
    } catch {
      message.status = 'error';
    }
    
    this.cdr.markForCheck();
  }

  onTyping(): void {
    this.socketService.emit('typing', {
      room: this.currentRoom,
      userId: this.currentUserId,
      username: this.currentUsername
    });

    clearTimeout(this.typingTimer);
    this.typingTimer = setTimeout(() => {
      this.socketService.emit('stop-typing', {
        room: this.currentRoom,
        userId: this.currentUserId
      });
    }, 1500);
  }

  isTyping(userId: string): boolean {
    return this.typingUsers.some(u => u.userId === userId);
  }

  getTypingText(): string {
    if (this.typingUsers.length === 1) {
      return `${this.typingUsers[0].username} กำลังพิมพ์...`;
    }
    if (this.typingUsers.length === 2) {
      return `${this.typingUsers[0].username} และ ${this.typingUsers[1].username} กำลังพิมพ์...`;
    }
    return `${this.typingUsers.length} คนกำลังพิมพ์...`;
  }

  getStatusIcon(status?: string): string {
    const icons: Record<string, string> = {
      sending: '⏳',
      sent: '✓',
      error: '⚠️'
    };
    return status ? icons[status] || '' : '';
  }

  get connectionStatus(): string {
    return this.isConnected ? 'connected' : 'disconnected';
  }

  get connectionStatusText(): string {
    return this.isConnected ? '🟢 เชื่อมต่อแล้ว' : '🔴 ไม่ได้เชื่อมต่อ';
  }

  private scrollToBottom(): void {
    setTimeout(() => {
      const container = this.messagesContainer?.nativeElement;
      if (container) {
        container.scrollTop = container.scrollHeight;
      }
    }, 50);
  }

  trackById(index: number, msg: ChatMessage): string {
    return msg.id;
  }

  toggleEmojiPicker(): void {
    console.log('Emoji picker - TODO: implement');
  }

  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
    clearTimeout(this.typingTimer);
    this.socketService.leaveRoom(this.currentRoom);
  }
}
```

---

## 4. Node.js Server

```javascript
// server/index.js
const express = require('express');
const { createServer } = require('http');
const { Server } = require('socket.io');
const cors = require('cors');

const app = express();
app.use(cors());

const httpServer = createServer(app);
const io = new Server(httpServer, {
  cors: {
    origin: 'http://localhost:4200',
    methods: ['GET', 'POST']
  }
});

// เก็บ state
const rooms = new Map(); // roomName -> Set of users
const userMap = new Map(); // socketId -> user info

io.on('connection', (socket) => {
  console.log('Client connected:', socket.id);

  socket.on('join-room', ({ room, userId, username }) => {
    socket.join(room);
    userMap.set(socket.id, { userId, username, room });
    
    if (!rooms.has(room)) rooms.set(room, new Set());
    rooms.get(room).add({ id: socket.id, userId, username });
    
    // แจ้งห้องว่ามีคนเข้า
    socket.to(room).emit('system-message', {
      content: `${username} เข้าร่วมห้อง`
    });
    
    // ส่ง online users
    io.to(room).emit('online-users', 
      Array.from(rooms.get(room)));
  });

  socket.on('send-message', (data, callback) => {
    const message = {
      id: Date.now().toString(),
      ...data,
      timestamp: new Date()
    };
    
    io.to(data.room).emit('new-message', message);
    callback({ success: true, id: message.id });
  });

  socket.on('typing', ({ room, userId, username }) => {
    socket.to(room).emit('user-typing', { userId, username });
  });

  socket.on('stop-typing', ({ room, userId }) => {
    socket.to(room).emit('user-stopped-typing', { userId });
  });

  socket.on('disconnect', () => {
    const user = userMap.get(socket.id);
    if (user) {
      const { room, username } = user;
      
      if (rooms.has(room)) {
        rooms.get(room).delete(
          Array.from(rooms.get(room)).find(u => u.id === socket.id)
        );
        io.to(room).emit('online-users', Array.from(rooms.get(room)));
        io.to(room).emit('system-message', {
          content: `${username} ออกจากห้อง`
        });
      }
      
      userMap.delete(socket.id);
    }
  });
});

httpServer.listen(3000, () => {
  console.log('Server running on port 3000');
});
```

---

## สรุป

| Feature | Implementation |
|---------|---------------|
| Connection | `socket.io-client` |
| Messages | `emit` + `on` |
| Typing | `emit('typing')` ทุกครั้งที่พิมพ์ |
| Rooms | `socket.join(room)` |
| Acknowledgement | callback ใน `emit` |
| Reconnection | auto reconnect settings |
