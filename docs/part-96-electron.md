# Part 96: Desktop App ด้วย Electron + Angular

## Electron คืออะไร

Electron ใช้ Chromium + Node.js สร้าง desktop app ด้วย web tech ตัวอย่างแอปที่ใช้: VS Code, Slack, Discord

---

## 1. Setup

```bash
# สร้าง Angular project
ng new my-desktop-app

# ติดตั้ง Electron
npm install --save-dev electron electron-builder

# ติดตั้ง helper
npm install ngx-electron
```

```json
// package.json
{
  "main": "electron/main.js",
  "scripts": {
    "build": "ng build",
    "electron": "ng build && electron .",
    "electron:dev": "concurrently \"ng serve\" \"wait-on http://localhost:4200 && electron . --dev\"",
    "dist": "ng build --prod && electron-builder",
    "dist:win": "ng build --prod && electron-builder --win",
    "dist:mac": "ng build --prod && electron-builder --mac",
    "dist:linux": "ng build --prod && electron-builder --linux"
  },
  "build": {
    "appId": "com.myapp.desktop",
    "productName": "MyApp",
    "directories": {
      "output": "dist/electron-packages"
    },
    "files": ["dist/my-desktop-app/**/*", "electron/**/*"],
    "win": { "target": "nsis", "icon": "assets/icon.ico" },
    "mac": { "target": "dmg", "icon": "assets/icon.icns" },
    "linux": { "target": "AppImage", "icon": "assets/icon.png" }
  }
}
```

---

## 2. Electron Main Process

```javascript
// electron/main.js
const { app, BrowserWindow, ipcMain, dialog, shell, Menu, Tray } = require('electron');
const path = require('path');
const isDev = process.argv.includes('--dev');

let mainWindow;
let tray;

function createWindow() {
  mainWindow = new BrowserWindow({
    width: 1280,
    height: 800,
    minWidth: 800,
    minHeight: 600,
    webPreferences: {
      nodeIntegration: false,
      contextIsolation: true,
      preload: path.join(__dirname, 'preload.js')
    },
    titleBarStyle: 'hiddenInset', // macOS style
    icon: path.join(__dirname, '../assets/icon.png')
  });

  // โหลด Angular app
  if (isDev) {
    mainWindow.loadURL('http://localhost:4200');
    mainWindow.webContents.openDevTools();
  } else {
    mainWindow.loadFile(path.join(__dirname, '../dist/my-desktop-app/index.html'));
  }

  // Handle window closed
  mainWindow.on('closed', () => { mainWindow = null; });

  // Prevent navigation to external URLs
  mainWindow.webContents.on('will-navigate', (event, url) => {
    if (!url.startsWith('http://localhost') && !url.startsWith('file://')) {
      event.preventDefault();
      shell.openExternal(url); // เปิด browser แทน
    }
  });
}

// =====================
// System Tray
// =====================
function createTray() {
  tray = new Tray(path.join(__dirname, '../assets/tray-icon.png'));
  
  const contextMenu = Menu.buildFromTemplate([
    { label: 'เปิดแอป', click: () => mainWindow?.show() },
    { type: 'separator' },
    { label: 'ออกจากระบบ', click: () => app.quit() }
  ]);
  
  tray.setContextMenu(contextMenu);
  tray.setToolTip('MyApp');
  tray.on('click', () => mainWindow?.show());
}

// =====================
// IPC Handlers
// =====================

// Open file dialog
ipcMain.handle('open-file-dialog', async (event, options) => {
  const result = await dialog.showOpenDialog(mainWindow, {
    properties: ['openFile'],
    filters: options?.filters || [{ name: 'All Files', extensions: ['*'] }]
  });
  return result;
});

// Save file dialog
ipcMain.handle('save-file-dialog', async (event, options) => {
  const result = await dialog.showSaveDialog(mainWindow, {
    defaultPath: options?.defaultPath,
    filters: options?.filters || [{ name: 'All Files', extensions: ['*'] }]
  });
  return result;
});

// Read file
ipcMain.handle('read-file', async (event, filePath) => {
  const fs = require('fs').promises;
  return await fs.readFile(filePath, 'utf-8');
});

// Write file
ipcMain.handle('write-file', async (event, filePath, content) => {
  const fs = require('fs').promises;
  await fs.writeFile(filePath, content, 'utf-8');
  return { success: true };
});

// Get app version
ipcMain.handle('get-app-version', () => app.getVersion());

// Show notification (native)
ipcMain.handle('show-notification', (event, { title, body }) => {
  const { Notification } = require('electron');
  if (Notification.isSupported()) {
    new Notification({ title, body }).show();
  }
});

// =====================
// App Events
// =====================
app.whenReady().then(() => {
  createWindow();
  createTray();
  
  app.on('activate', () => {
    if (BrowserWindow.getAllWindows().length === 0) createWindow();
  });
});

app.on('window-all-closed', () => {
  if (process.platform !== 'darwin') app.quit();
});
```

---

## 3. Preload Script (Security Bridge)

```javascript
// electron/preload.js
const { contextBridge, ipcRenderer } = require('electron');

// เปิด API ที่ปลอดภัยให้ Angular ใช้
contextBridge.exposeInMainWorld('electronAPI', {
  // File System
  openFileDialog: (options) => ipcRenderer.invoke('open-file-dialog', options),
  saveFileDialog: (options) => ipcRenderer.invoke('save-file-dialog', options),
  readFile: (path) => ipcRenderer.invoke('read-file', path),
  writeFile: (path, content) => ipcRenderer.invoke('write-file', path, content),
  
  // App
  getVersion: () => ipcRenderer.invoke('get-app-version'),
  showNotification: (options) => ipcRenderer.invoke('show-notification', options),
  
  // Events from main process
  onMenuAction: (callback) => ipcRenderer.on('menu-action', (_, action) => callback(action)),
  removeMenuActionListener: () => ipcRenderer.removeAllListeners('menu-action')
});
```

---

## 4. Angular Electron Service

```typescript
// core/electron/electron.service.ts
import { Injectable } from '@angular/core';

// Type declaration สำหรับ window.electronAPI
declare global {
  interface Window {
    electronAPI?: {
      openFileDialog: (options?: FileDialogOptions) => Promise<FileDialogResult>;
      saveFileDialog: (options?: FileDialogOptions) => Promise<SaveDialogResult>;
      readFile: (path: string) => Promise<string>;
      writeFile: (path: string, content: string) => Promise<{ success: boolean }>;
      getVersion: () => Promise<string>;
      showNotification: (options: { title: string; body: string }) => Promise<void>;
      onMenuAction: (callback: (action: string) => void) => void;
      removeMenuActionListener: () => void;
    };
  }
}

export interface FileDialogOptions {
  filters?: { name: string; extensions: string[] }[];
  defaultPath?: string;
}

export interface FileDialogResult {
  canceled: boolean;
  filePaths: string[];
}

export interface SaveDialogResult {
  canceled: boolean;
  filePath?: string;
}

@Injectable({ providedIn: 'root' })
export class ElectronService {
  get isElectron(): boolean {
    return typeof window !== 'undefined' && !!window.electronAPI;
  }

  async openFileDialog(options?: FileDialogOptions): Promise<FileDialogResult> {
    if (!this.isElectron) throw new Error('ไม่ได้รันบน Electron');
    return window.electronAPI!.openFileDialog(options);
  }

  async saveFileDialog(options?: FileDialogOptions): Promise<SaveDialogResult> {
    if (!this.isElectron) throw new Error('ไม่ได้รันบน Electron');
    return window.electronAPI!.saveFileDialog(options);
  }

  async readFile(path: string): Promise<string> {
    if (!this.isElectron) throw new Error('ไม่ได้รันบน Electron');
    return window.electronAPI!.readFile(path);
  }

  async writeFile(path: string, content: string): Promise<void> {
    if (!this.isElectron) throw new Error('ไม่ได้รันบน Electron');
    await window.electronAPI!.writeFile(path, content);
  }

  async getVersion(): Promise<string> {
    if (!this.isElectron) return '1.0.0-web';
    return window.electronAPI!.getVersion();
  }

  async showNotification(title: string, body: string): Promise<void> {
    if (!this.isElectron) {
      // Fallback to Web Notification
      if (Notification.permission === 'granted') {
        new Notification(title, { body });
      }
      return;
    }
    await window.electronAPI!.showNotification({ title, body });
  }
}
```

---

## 5. Text Editor Component

```typescript
// features/editor/editor.component.ts
import { Component, OnInit, HostListener } from '@angular/core';
import { ElectronService } from '../../core/electron/electron.service';

@Component({
  selector: 'app-editor',
  template: `
    <div class="editor-container">
      <div class="toolbar">
        <span class="title-bar">
          {{ filename || 'ไม่มีชื่อ' }}
          <span class="modified-indicator" *ngIf="isModified">●</span>
        </span>
        <div class="toolbar-actions">
          <button (click)="newFile()">ใหม่</button>
          <button (click)="openFile()">เปิด</button>
          <button (click)="saveFile()" [disabled]="!isModified">บันทึก</button>
          <button (click)="saveFileAs()">บันทึกเป็น</button>
        </div>
      </div>

      <textarea
        class="editor"
        [(ngModel)]="content"
        (ngModelChange)="onContentChange()"
        placeholder="เริ่มพิมพ์ที่นี่..."
        spellcheck="false"
      ></textarea>

      <div class="status-bar">
        <span>{{ lineCount }} บรรทัด | {{ charCount }} ตัวอักษร</span>
        <span>{{ appVersion }}</span>
      </div>
    </div>
  `,
  styles: [`
    .editor-container { display: flex; flex-direction: column; height: 100vh; }
    .toolbar {
      display: flex; align-items: center; justify-content: space-between;
      padding: 8px 16px; background: #2d2d2d; color: white;
    }
    .title-bar { font-size: 14px; }
    .modified-indicator { color: #f0ad4e; margin-left: 4px; }
    .toolbar-actions { display: flex; gap: 8px; }
    .toolbar-actions button {
      padding: 4px 12px; background: #444; color: white;
      border: none; border-radius: 4px; cursor: pointer;
    }
    .toolbar-actions button:hover:not(:disabled) { background: #555; }
    .toolbar-actions button:disabled { opacity: 0.5; }
    .editor {
      flex: 1; padding: 16px; font-family: 'Courier New', monospace;
      font-size: 14px; line-height: 1.6; border: none; resize: none;
      background: #1e1e1e; color: #d4d4d4; outline: none;
    }
    .status-bar {
      display: flex; justify-content: space-between; padding: 4px 16px;
      background: #007acc; color: white; font-size: 12px;
    }
  `]
})
export class EditorComponent implements OnInit {
  content = '';
  filename = '';
  currentFilePath = '';
  isModified = false;
  appVersion = '';

  get lineCount(): number {
    return this.content.split('\n').length;
  }

  get charCount(): number {
    return this.content.length;
  }

  constructor(private electronService: ElectronService) {}

  async ngOnInit(): Promise<void> {
    this.appVersion = `v${await this.electronService.getVersion()}`;
  }

  @HostListener('document:keydown', ['$event'])
  handleKeyboardShortcuts(event: KeyboardEvent): void {
    if (event.ctrlKey || event.metaKey) {
      switch (event.key) {
        case 'n': event.preventDefault(); this.newFile(); break;
        case 'o': event.preventDefault(); this.openFile(); break;
        case 's': event.preventDefault();
          if (event.shiftKey) this.saveFileAs();
          else this.saveFile();
          break;
      }
    }
  }

  newFile(): void {
    if (this.isModified) {
      if (!confirm('มีการเปลี่ยนแปลงที่ยังไม่ได้บันทึก ต้องการเปิดไฟล์ใหม่?')) return;
    }
    this.content = '';
    this.filename = '';
    this.currentFilePath = '';
    this.isModified = false;
  }

  async openFile(): Promise<void> {
    const result = await this.electronService.openFileDialog({
      filters: [
        { name: 'ไฟล์ข้อความ', extensions: ['txt', 'md', 'json', 'ts'] },
        { name: 'ทั้งหมด', extensions: ['*'] }
      ]
    });

    if (!result.canceled && result.filePaths.length > 0) {
      const filePath = result.filePaths[0];
      this.content = await this.electronService.readFile(filePath);
      this.currentFilePath = filePath;
      this.filename = filePath.split('/').pop() || filePath.split('\\').pop() || '';
      this.isModified = false;
    }
  }

  async saveFile(): Promise<void> {
    if (this.currentFilePath) {
      await this.electronService.writeFile(this.currentFilePath, this.content);
      this.isModified = false;
      await this.electronService.showNotification('บันทึกสำเร็จ', `บันทึก ${this.filename} เรียบร้อย`);
    } else {
      await this.saveFileAs();
    }
  }

  async saveFileAs(): Promise<void> {
    const result = await this.electronService.saveFileDialog({
      defaultPath: this.filename || 'ไม่มีชื่อ.txt',
      filters: [{ name: 'ไฟล์ข้อความ', extensions: ['txt', 'md'] }]
    });

    if (!result.canceled && result.filePath) {
      await this.electronService.writeFile(result.filePath, this.content);
      this.currentFilePath = result.filePath;
      this.filename = result.filePath.split('/').pop() || '';
      this.isModified = false;
    }
  }

  onContentChange(): void {
    this.isModified = true;
  }
}
```

---

## สรุป

| เรื่อง | รายละเอียด |
|-------|-----------|
| Main Process | Node.js, เข้าถึง OS |
| Renderer | Angular app |
| IPC | สื่อสารระหว่าง main และ renderer |
| Preload | Security bridge |
| contextBridge | API ที่ปลอดภัย |
| electron-builder | Package และ distribute |

### Best Practices

1. ใช้ `contextIsolation: true` เสมอ
2. ไม่เปิด `nodeIntegration` ใน renderer
3. Validate ทุก IPC input
4. ใช้ contextBridge สำหรับ API
5. Handle deep links สำหรับ OAuth
