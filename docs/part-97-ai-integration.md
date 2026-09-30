# Part 97: AI Integration ใน Angular (OpenAI, Claude API, Streaming)

## AI Integration Overview

เรียนรู้การผสานรวม AI APIs เข้ากับ Angular app ทั้ง OpenAI GPT และ Anthropic Claude พร้อม streaming responses

---

## 1. ติดตั้ง

```bash
npm install openai
npm install @anthropic-ai/sdk
```

---

## 2. AI Service (OpenAI + Claude)

```typescript
// core/ai/ai.service.ts
import { Injectable } from '@angular/core';
import { HttpClient, HttpHeaders } from '@angular/common/http';
import { Observable, Subject } from 'rxjs';

export interface ChatMessage {
  role: 'user' | 'assistant' | 'system';
  content: string;
}

export interface StreamChunk {
  text: string;
  done: boolean;
}

@Injectable({ providedIn: 'root' })
export class AIService {
  constructor(private http: HttpClient) {}

  // ========================================
  // OpenAI GPT
  // ========================================
  chatWithGPT(messages: ChatMessage[]): Observable<StreamChunk> {
    const subject = new Subject<StreamChunk>();

    this.http.post('/api/ai/openai/chat', { messages }, {
      observe: 'response',
      responseType: 'text'
    }).subscribe({
      next: (response) => {
        // Non-streaming response
        const text = response.body || '';
        subject.next({ text, done: false });
        subject.next({ text: '', done: true });
        subject.complete();
      },
      error: (err) => subject.error(err)
    });

    return subject.asObservable();
  }

  // Streaming ด้วย fetch API (ไม่ใช่ HttpClient)
  async streamWithGPT(
    messages: ChatMessage[],
    onChunk: (text: string) => void,
    onDone: () => void,
    onError: (error: Error) => void
  ): Promise<void> {
    const response = await fetch('/api/ai/openai/stream', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ messages })
    });

    if (!response.ok) {
      onError(new Error(`HTTP error: ${response.status}`));
      return;
    }

    const reader = response.body?.getReader();
    if (!reader) return;

    const decoder = new TextDecoder();
    let buffer = '';

    while (true) {
      const { done, value } = await reader.read();
      if (done) break;

      buffer += decoder.decode(value, { stream: true });
      const lines = buffer.split('\n');
      buffer = lines.pop() || '';

      for (const line of lines) {
        if (line.startsWith('data: ')) {
          const data = line.slice(6).trim();
          if (data === '[DONE]') {
            onDone();
            return;
          }
          try {
            const parsed = JSON.parse(data);
            const text = parsed.choices?.[0]?.delta?.content || '';
            if (text) onChunk(text);
          } catch {
            // skip malformed lines
          }
        }
      }
    }
    onDone();
  }

  // ========================================
  // Anthropic Claude
  // ========================================
  async streamWithClaude(
    messages: ChatMessage[],
    onChunk: (text: string) => void,
    onDone: () => void,
    onError: (error: Error) => void
  ): Promise<void> {
    const response = await fetch('/api/ai/claude/stream', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        model: 'claude-opus-4-5',
        max_tokens: 2048,
        messages: messages.filter(m => m.role !== 'system'),
        system: messages.find(m => m.role === 'system')?.content
      })
    });

    if (!response.ok) {
      onError(new Error(`HTTP error: ${response.status}`));
      return;
    }

    const reader = response.body?.getReader();
    if (!reader) return;

    const decoder = new TextDecoder();
    let buffer = '';

    while (true) {
      const { done, value } = await reader.read();
      if (done) break;

      buffer += decoder.decode(value, { stream: true });
      const lines = buffer.split('\n');
      buffer = lines.pop() || '';

      for (const line of lines) {
        if (line.startsWith('data: ')) {
          const data = line.slice(6).trim();
          try {
            const parsed = JSON.parse(data);
            if (parsed.type === 'content_block_delta') {
              onChunk(parsed.delta?.text || '');
            } else if (parsed.type === 'message_stop') {
              onDone();
              return;
            }
          } catch {
            // skip
          }
        }
      }
    }
    onDone();
  }
}
```

---

## 3. AI Chat Component

```typescript
// features/ai-chat/ai-chat.component.ts
import { Component, OnInit, OnDestroy, ViewChild, ElementRef, ChangeDetectorRef, NgZone } from '@angular/core';
import { FormControl, Validators } from '@angular/forms';
import { AIService, ChatMessage } from '../../core/ai/ai.service';

@Component({
  selector: 'app-ai-chat',
  template: `
    <div class="chat-container">
      <div class="chat-header">
        <div class="ai-avatar">🤖</div>
        <div class="header-info">
          <h3>AI Assistant</h3>
          <span class="status" [class.online]="!isStreaming">
            {{ isStreaming ? 'กำลังพิมพ์...' : 'พร้อมใช้งาน' }}
          </span>
        </div>
        <div class="header-actions">
          <button (click)="clearChat()" title="ล้างประวัติ">🗑️</button>
          <select [(ngModel)]="selectedModel" (ngModelChange)="onModelChange()">
            <option value="gpt-4">GPT-4</option>
            <option value="gpt-3.5-turbo">GPT-3.5</option>
            <option value="claude-opus-4-5">Claude Opus</option>
            <option value="claude-sonnet-4-5">Claude Sonnet</option>
          </select>
        </div>
      </div>

      <div class="messages-container" #messagesContainer>
        <!-- Welcome Message -->
        <div class="welcome-message" *ngIf="messages.length === 0">
          <div class="welcome-icon">💬</div>
          <h4>สวัสดี! ฉันคือ AI Assistant</h4>
          <p>ถามอะไรก็ได้ที่คุณอยากรู้</p>
          <div class="suggestions">
            <button 
              *ngFor="let s of suggestions"
              class="suggestion-chip"
              (click)="sendSuggestion(s)"
            >{{ s }}</button>
          </div>
        </div>

        <!-- Messages -->
        <div 
          class="message" 
          *ngFor="let msg of messages; let i = index"
          [class.user-message]="msg.role === 'user'"
          [class.assistant-message]="msg.role === 'assistant'"
        >
          <div class="message-avatar">
            {{ msg.role === 'user' ? '👤' : '🤖' }}
          </div>
          <div class="message-bubble">
            <div class="message-content" [innerHTML]="formatMessage(msg.content)"></div>
            <div class="message-time">{{ getMessageTime(i) }}</div>
          </div>
        </div>

        <!-- Streaming Message -->
        <div class="message assistant-message" *ngIf="isStreaming">
          <div class="message-avatar">🤖</div>
          <div class="message-bubble">
            <div class="message-content" [innerHTML]="formatMessage(streamingText)"></div>
            <span class="typing-cursor">▋</span>
          </div>
        </div>

        <!-- Error -->
        <div class="error-message" *ngIf="errorMessage">
          ⚠️ {{ errorMessage }}
          <button (click)="retryLastMessage()">ลองใหม่</button>
        </div>
      </div>

      <div class="input-area">
        <div class="system-prompt-toggle" *ngIf="showSystemPrompt">
          <textarea
            [(ngModel)]="systemPrompt"
            placeholder="System Prompt (optional)..."
            rows="2"
          ></textarea>
        </div>
        
        <div class="input-row">
          <button 
            class="btn-system"
            (click)="showSystemPrompt = !showSystemPrompt"
            title="System Prompt"
          >⚙️</button>
          
          <textarea
            [formControl]="messageInput"
            placeholder="พิมพ์ข้อความ... (Enter ส่ง, Shift+Enter ขึ้นบรรทัดใหม่)"
            rows="1"
            class="message-input"
            (keydown)="onKeyDown($event)"
            (input)="autoResize($event)"
          ></textarea>
          
          <button 
            class="btn-send"
            (click)="sendMessage()"
            [disabled]="!messageInput.value?.trim() || isStreaming"
          >
            <span *ngIf="!isStreaming">➤</span>
            <div *ngIf="isStreaming" class="stop-btn" (click)="stopStreaming()">⏹</div>
          </button>
        </div>
      </div>
    </div>
  `,
  styles: [`
    .chat-container {
      display: flex; flex-direction: column;
      height: 100vh; max-width: 800px; margin: 0 auto;
      background: #f5f5f5;
    }
    .chat-header {
      display: flex; align-items: center; gap: 12px;
      padding: 16px; background: #1a237e; color: white;
    }
    .ai-avatar { font-size: 28px; }
    .header-info h3 { margin: 0; font-size: 16px; }
    .status { font-size: 12px; opacity: 0.8; }
    .status.online::before { content: '● '; color: #4caf50; }
    .header-actions { margin-left: auto; display: flex; gap: 8px; align-items: center; }
    .header-actions select { padding: 4px 8px; border-radius: 4px; border: none; }
    .header-actions button { background: none; border: none; cursor: pointer; font-size: 18px; }

    .messages-container {
      flex: 1; overflow-y: auto; padding: 16px;
      display: flex; flex-direction: column; gap: 12px;
    }
    .welcome-message { text-align: center; padding: 40px 20px; color: #666; }
    .welcome-icon { font-size: 48px; margin-bottom: 16px; }
    .suggestions { display: flex; flex-wrap: wrap; gap: 8px; justify-content: center; margin-top: 16px; }
    .suggestion-chip {
      padding: 8px 16px; background: white; border: 1px solid #ddd;
      border-radius: 20px; cursor: pointer; font-size: 13px;
    }
    .suggestion-chip:hover { background: #e8eaf6; border-color: #3f51b5; }

    .message { display: flex; gap: 8px; }
    .message-avatar { font-size: 24px; flex-shrink: 0; }
    .user-message { flex-direction: row-reverse; }
    .message-bubble {
      max-width: 75%; padding: 12px 16px;
      border-radius: 16px; position: relative;
    }
    .user-message .message-bubble {
      background: #1a237e; color: white; border-bottom-right-radius: 4px;
    }
    .assistant-message .message-bubble {
      background: white; border-bottom-left-radius: 4px;
      box-shadow: 0 1px 2px rgba(0,0,0,0.1);
    }
    .message-content { white-space: pre-wrap; line-height: 1.5; font-size: 14px; }
    .message-time { font-size: 11px; opacity: 0.6; margin-top: 4px; text-align: right; }
    .typing-cursor { animation: blink 1s infinite; }
    @keyframes blink { 0%, 100% { opacity: 1; } 50% { opacity: 0; } }

    .error-message {
      padding: 12px; background: #ffebee; color: #c62828;
      border-radius: 8px; display: flex; align-items: center; gap: 8px;
    }
    .error-message button {
      padding: 4px 12px; background: #c62828; color: white;
      border: none; border-radius: 4px; cursor: pointer;
    }

    .input-area { padding: 16px; background: white; border-top: 1px solid #eee; }
    .system-prompt-toggle textarea {
      width: 100%; margin-bottom: 8px; padding: 8px;
      border: 1px solid #ddd; border-radius: 8px; resize: none;
      font-size: 13px; box-sizing: border-box;
    }
    .input-row { display: flex; gap: 8px; align-items: flex-end; }
    .btn-system {
      background: none; border: none; cursor: pointer; font-size: 20px;
      padding: 8px; opacity: 0.6;
    }
    .message-input {
      flex: 1; padding: 10px 14px; border: 1px solid #ddd;
      border-radius: 20px; resize: none; font-size: 14px;
      max-height: 120px; outline: none; transition: border-color 0.2s;
    }
    .message-input:focus { border-color: #1a237e; }
    .btn-send {
      width: 44px; height: 44px; border-radius: 50%;
      background: #1a237e; color: white; border: none;
      cursor: pointer; font-size: 18px; flex-shrink: 0;
    }
    .btn-send:disabled { opacity: 0.5; }
    .stop-btn { cursor: pointer; }
  `]
})
export class AIChatComponent implements OnInit {
  @ViewChild('messagesContainer') messagesContainer!: ElementRef;

  messages: (ChatMessage & { timestamp: Date })[] = [];
  messageInput = new FormControl('', Validators.required);
  systemPrompt = 'คุณเป็น AI Assistant ที่ช่วยเหลือผู้ใช้ภาษาไทย ตอบสั้นกระชับและเป็นประโยชน์';
  showSystemPrompt = false;
  isStreaming = false;
  streamingText = '';
  errorMessage = '';
  selectedModel = 'claude-sonnet-4-5';
  abortController: AbortController | null = null;

  suggestions = [
    'Angular คืออะไร?',
    'ช่วยเขียน TypeScript interface',
    'อธิบาย RxJS Observable',
    'วิธีใช้ Angular Signals'
  ];

  messageTimes: Date[] = [];

  constructor(
    private aiService: AIService,
    private cdr: ChangeDetectorRef,
    private ngZone: NgZone
  ) {}

  ngOnInit(): void {}

  async sendMessage(): Promise<void> {
    const text = this.messageInput.value?.trim();
    if (!text || this.isStreaming) return;

    this.messageInput.setValue('');
    this.errorMessage = '';

    const userMessage: ChatMessage & { timestamp: Date } = {
      role: 'user',
      content: text,
      timestamp: new Date()
    };
    this.messages.push(userMessage);
    this.messageTimes.push(new Date());

    await this.callAI(text);
  }

  private async callAI(userText: string): Promise<void> {
    this.isStreaming = true;
    this.streamingText = '';

    const chatMessages: ChatMessage[] = [
      { role: 'system', content: this.systemPrompt },
      ...this.messages.map(m => ({ role: m.role, content: m.content }))
    ];

    const isClaudeModel = this.selectedModel.startsWith('claude');

    try {
      const streamFn = isClaudeModel
        ? this.aiService.streamWithClaude.bind(this.aiService)
        : this.aiService.streamWithGPT.bind(this.aiService);

      await streamFn(
        chatMessages,
        (chunk: string) => {
          this.ngZone.run(() => {
            this.streamingText += chunk;
            this.cdr.detectChanges();
            this.scrollToBottom();
          });
        },
        () => {
          this.ngZone.run(() => {
            // เพิ่ม assistant message
            this.messages.push({
              role: 'assistant',
              content: this.streamingText,
              timestamp: new Date()
            });
            this.messageTimes.push(new Date());
            this.streamingText = '';
            this.isStreaming = false;
            this.cdr.detectChanges();
            this.scrollToBottom();
          });
        },
        (error: Error) => {
          this.ngZone.run(() => {
            this.errorMessage = error.message;
            this.isStreaming = false;
            this.cdr.detectChanges();
          });
        }
      );
    } catch (error: any) {
      this.ngZone.run(() => {
        this.errorMessage = error.message || 'เกิดข้อผิดพลาด';
        this.isStreaming = false;
      });
    }
  }

  stopStreaming(): void {
    this.abortController?.abort();
    this.isStreaming = false;
    if (this.streamingText) {
      this.messages.push({
        role: 'assistant',
        content: this.streamingText + ' [หยุด]',
        timestamp: new Date()
      });
      this.messageTimes.push(new Date());
      this.streamingText = '';
    }
  }

  sendSuggestion(text: string): void {
    this.messageInput.setValue(text);
    this.sendMessage();
  }

  clearChat(): void {
    this.messages = [];
    this.messageTimes = [];
    this.streamingText = '';
    this.errorMessage = '';
  }

  retryLastMessage(): void {
    const lastUser = [...this.messages].reverse().find(m => m.role === 'user');
    if (lastUser) {
      this.errorMessage = '';
      this.callAI(lastUser.content);
    }
  }

  onModelChange(): void {
    console.log('Model changed to:', this.selectedModel);
  }

  formatMessage(content: string): string {
    return content
      .replace(/`([^`]+)`/g, '<code>$1</code>')
      .replace(/\*\*([^*]+)\*\*/g, '<strong>$1</strong>');
  }

  getMessageTime(index: number): string {
    const time = this.messageTimes[index];
    if (!time) return '';
    return time.toLocaleTimeString('th-TH', { hour: '2-digit', minute: '2-digit' });
  }

  onKeyDown(event: KeyboardEvent): void {
    if (event.key === 'Enter' && !event.shiftKey) {
      event.preventDefault();
      this.sendMessage();
    }
  }

  autoResize(event: Event): void {
    const el = event.target as HTMLTextAreaElement;
    el.style.height = 'auto';
    el.style.height = Math.min(el.scrollHeight, 120) + 'px';
  }

  private scrollToBottom(): void {
    setTimeout(() => {
      const container = this.messagesContainer?.nativeElement;
      if (container) container.scrollTop = container.scrollHeight;
    }, 50);
  }
}
```

---

## 4. Node.js Backend (Proxy)

```javascript
// server/routes/ai.js
const express = require('express');
const router = express.Router();
const Anthropic = require('@anthropic-ai/sdk');
const OpenAI = require('openai');

const anthropic = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY });
const openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

// Claude Streaming
router.post('/claude/stream', async (req, res) => {
  const { messages, model, max_tokens, system } = req.body;
  
  res.setHeader('Content-Type', 'text/event-stream');
  res.setHeader('Cache-Control', 'no-cache');
  res.setHeader('Connection', 'keep-alive');

  try {
    const stream = anthropic.messages.stream({
      model: model || 'claude-sonnet-4-5',
      max_tokens: max_tokens || 2048,
      system,
      messages
    });

    for await (const event of stream) {
      res.write(`data: ${JSON.stringify(event)}\n\n`);
    }
    res.write('data: {"type":"message_stop"}\n\n');
    res.end();
  } catch (error) {
    res.write(`data: ${JSON.stringify({ type: 'error', error: error.message })}\n\n`);
    res.end();
  }
});

// OpenAI Streaming
router.post('/openai/stream', async (req, res) => {
  const { messages, model } = req.body;
  
  res.setHeader('Content-Type', 'text/event-stream');
  res.setHeader('Cache-Control', 'no-cache');
  res.setHeader('Connection', 'keep-alive');

  try {
    const stream = await openai.chat.completions.create({
      model: model || 'gpt-4',
      messages,
      stream: true
    });

    for await (const chunk of stream) {
      res.write(`data: ${JSON.stringify(chunk)}\n\n`);
    }
    res.write('data: [DONE]\n\n');
    res.end();
  } catch (error) {
    res.write(`data: ${JSON.stringify({ error: error.message })}\n\n`);
    res.end();
  }
});

module.exports = router;
```

---

## สรุป

| Provider | Model | เหมาะกับ |
|---------|-------|---------|
| OpenAI | GPT-4 | General purpose |
| OpenAI | GPT-3.5-turbo | เร็ว ถูก |
| Anthropic | Claude Opus | ซับซ้อน |
| Anthropic | Claude Sonnet | สมดุล |

### Best Practices

1. ไม่ใส่ API key ใน frontend
2. ใช้ backend proxy เสมอ
3. Rate limit ทั้ง client และ server
4. Sanitize output ก่อนแสดง
5. Handle streaming errors
6. ตั้ง max_tokens เหมาะสม
