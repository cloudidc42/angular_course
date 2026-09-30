# Part 54: File Upload ใน Angular

## บทนำ

การอัปโหลดไฟล์เป็นฟีเจอร์ที่พบบ่อยในแอปพลิเคชัน บทนี้จะสอนการสร้าง File Upload component ที่มีคุณสมบัติครบถ้วน ตั้งแต่ drag & drop ไปจนถึง preview และ validation

## 1. File Upload Service

```typescript
// file-upload.service.ts
import { Injectable } from '@angular/core';
import { HttpClient, HttpEventType, HttpRequest } from '@angular/common/http';
import { Observable } from 'rxjs';
import { map, filter } from 'rxjs/operators';

export interface UploadProgress {
  progress: number;
  status: 'uploading' | 'complete' | 'error';
  url?: string;
}

export interface UploadResult {
  url: string;
  filename: string;
  size: number;
  mimeType: string;
}

@Injectable({ providedIn: 'root' })
export class FileUploadService {
  private readonly uploadUrl = '/api/upload';

  constructor(private http: HttpClient) {}

  uploadFile(file: File, folder = 'uploads'): Observable<UploadProgress> {
    const formData = new FormData();
    formData.append('file', file, file.name);
    formData.append('folder', folder);

    const req = new HttpRequest('POST', this.uploadUrl, formData, {
      reportProgress: true
    });

    return this.http.request(req).pipe(
      map(event => {
        switch (event.type) {
          case HttpEventType.UploadProgress:
            const progress = Math.round(
              (100 * event.loaded) / (event.total || 1)
            );
            return { progress, status: 'uploading' as const };

          case HttpEventType.Response:
            return {
              progress: 100,
              status: 'complete' as const,
              url: (event.body as UploadResult)?.url
            };

          default:
            return { progress: 0, status: 'uploading' as const };
        }
      }),
      filter(event => event.progress > 0)
    );
  }

  uploadMultipleFiles(files: File[]): Observable<UploadProgress[]> {
    const formData = new FormData();
    files.forEach((file, index) => {
      formData.append(`file_${index}`, file, file.name);
    });

    const req = new HttpRequest('POST', `${this.uploadUrl}/multiple`, formData, {
      reportProgress: true
    });

    return this.http.request<UploadResult[]>(req).pipe(
      map(event => {
        if (event.type === HttpEventType.UploadProgress) {
          const progress = Math.round(
            (100 * event.loaded) / (event.total || 1)
          );
          return files.map(() => ({ progress, status: 'uploading' as const }));
        }
        if (event.type === HttpEventType.Response) {
          return files.map((_, i) => ({
            progress: 100,
            status: 'complete' as const,
            url: event.body?.[i]?.url
          }));
        }
        return files.map(() => ({ progress: 0, status: 'uploading' as const }));
      })
    );
  }

  deleteFile(fileId: string): Observable<void> {
    return this.http.delete<void>(`${this.uploadUrl}/${fileId}`);
  }
}
```

## 2. File Validation Service

```typescript
// file-validation.service.ts
import { Injectable } from '@angular/core';

export interface FileValidationConfig {
  maxSize?: number; // bytes
  allowedTypes?: string[];
  allowedExtensions?: string[];
  maxFiles?: number;
}

export interface ValidationResult {
  valid: boolean;
  errors: string[];
}

@Injectable({ providedIn: 'root' })
export class FileValidationService {
  private readonly SIZE_UNITS = ['B', 'KB', 'MB', 'GB'];

  validate(files: File[], config: FileValidationConfig): ValidationResult {
    const errors: string[] = [];

    if (config.maxFiles && files.length > config.maxFiles) {
      errors.push(`อัปโหลดได้สูงสุด ${config.maxFiles} ไฟล์`);
    }

    files.forEach(file => {
      const fileErrors = this.validateFile(file, config);
      errors.push(...fileErrors);
    });

    return {
      valid: errors.length === 0,
      errors
    };
  }

  validateFile(file: File, config: FileValidationConfig): string[] {
    const errors: string[] = [];

    if (config.maxSize && file.size > config.maxSize) {
      const maxSizeStr = this.formatFileSize(config.maxSize);
      const fileSizeStr = this.formatFileSize(file.size);
      errors.push(`"${file.name}" มีขนาด ${fileSizeStr} เกินขีดจำกัด ${maxSizeStr}`);
    }

    if (config.allowedTypes && !config.allowedTypes.includes(file.type)) {
      errors.push(`"${file.name}" ไม่ใช่ประเภทไฟล์ที่รองรับ`);
    }

    if (config.allowedExtensions) {
      const ext = file.name.split('.').pop()?.toLowerCase() || '';
      if (!config.allowedExtensions.includes(ext)) {
        errors.push(`"${file.name}" นามสกุลไฟล์ไม่รองรับ (อนุญาต: ${config.allowedExtensions.join(', ')})`);
      }
    }

    return errors;
  }

  formatFileSize(bytes: number): string {
    let i = 0;
    while (bytes >= 1024 && i < this.SIZE_UNITS.length - 1) {
      bytes /= 1024;
      i++;
    }
    return `${bytes.toFixed(1)} ${this.SIZE_UNITS[i]}`;
  }

  isImage(file: File): boolean {
    return file.type.startsWith('image/');
  }

  isVideo(file: File): boolean {
    return file.type.startsWith('video/');
  }

  isPDF(file: File): boolean {
    return file.type === 'application/pdf';
  }

  generatePreview(file: File): Promise<string> {
    return new Promise((resolve, reject) => {
      if (!this.isImage(file)) {
        reject(new Error('ไม่ใช่ไฟล์รูปภาพ'));
        return;
      }

      const reader = new FileReader();
      reader.onload = (e) => resolve(e.target?.result as string);
      reader.onerror = () => reject(new Error('อ่านไฟล์ไม่สำเร็จ'));
      reader.readAsDataURL(file);
    });
  }
}
```

## 3. Drag & Drop File Upload Component

```typescript
// file-upload.component.ts
import {
  Component, Input, Output, EventEmitter,
  HostListener, ElementRef, ViewChild
} from '@angular/core';
import { CommonModule } from '@angular/common';
import { FileValidationService, FileValidationConfig } from './file-validation.service';
import { FileUploadService, UploadProgress } from './file-upload.service';

export interface UploadedFile {
  file: File;
  preview?: string;
  progress: number;
  status: 'pending' | 'uploading' | 'complete' | 'error';
  url?: string;
  error?: string;
}

@Component({
  selector: 'app-file-upload',
  standalone: true,
  imports: [CommonModule],
  template: `
    <div
      class="upload-zone"
      [class.dragging]="isDragging"
      [class.disabled]="disabled"
      (dragover)="onDragOver($event)"
      (dragleave)="onDragLeave($event)"
      (drop)="onDrop($event)"
      (click)="fileInput.click()"
    >
      <input
        #fileInput
        type="file"
        [multiple]="multiple"
        [accept]="acceptString"
        (change)="onFileSelected($event)"
        style="display: none"
      />

      <div *ngIf="!files.length" class="upload-placeholder">
        <div class="upload-icon">📁</div>
        <p class="upload-text">ลากไฟล์มาวางที่นี่ หรือคลิกเพื่อเลือกไฟล์</p>
        <p class="upload-hint">
          รองรับ: {{ acceptString || 'ทุกประเภท' }} |
          ขนาดสูงสุด: {{ formatSize(config.maxSize || 0) }}
        </p>
      </div>

      <div *ngIf="files.length" class="file-list" (click)="$event.stopPropagation()">
        <div *ngFor="let item of files; let i = index" class="file-item">
          <div class="file-preview" *ngIf="item.preview">
            <img [src]="item.preview" [alt]="item.file.name" />
          </div>
          <div class="file-icon" *ngIf="!item.preview">
            {{ getFileIcon(item.file) }}
          </div>

          <div class="file-info">
            <span class="file-name">{{ item.file.name }}</span>
            <span class="file-size">{{ formatSize(item.file.size) }}</span>
          </div>

          <div class="file-progress" *ngIf="item.status === 'uploading'">
            <div class="progress-bar">
              <div class="progress-fill" [style.width.%]="item.progress"></div>
            </div>
            <span>{{ item.progress }}%</span>
          </div>

          <div class="file-status">
            <span *ngIf="item.status === 'complete'" class="status-complete">✓</span>
            <span *ngIf="item.status === 'error'" class="status-error" [title]="item.error">✗</span>
          </div>

          <button
            class="remove-btn"
            (click)="removeFile(i)"
            [disabled]="item.status === 'uploading'"
          >×</button>
        </div>

        <div class="upload-actions" *ngIf="multiple">
          <button class="add-more-btn" (click)="fileInput.click()">+ เพิ่มไฟล์</button>
        </div>
      </div>
    </div>

    <div *ngIf="validationErrors.length" class="validation-errors">
      <div *ngFor="let error of validationErrors" class="error-item">⚠️ {{ error }}</div>
    </div>

    <button
      *ngIf="files.length && showUploadButton"
      class="upload-btn"
      (click)="uploadAll()"
      [disabled]="isUploading"
    >
      {{ isUploading ? 'กำลังอัปโหลด...' : 'อัปโหลดทั้งหมด' }}
    </button>
  `,
  styles: [`
    .upload-zone {
      border: 2px dashed #ccc;
      border-radius: 8px;
      padding: 2rem;
      text-align: center;
      cursor: pointer;
      transition: all 0.2s;
      min-height: 150px;
    }
    .upload-zone:hover, .dragging {
      border-color: #007bff;
      background: #f0f8ff;
    }
    .disabled { opacity: 0.5; cursor: not-allowed; }
    .upload-icon { font-size: 3rem; }
    .upload-text { font-size: 1.1rem; margin: 0.5rem 0; }
    .upload-hint { color: #6c757d; font-size: 0.85rem; }
    .file-list { text-align: left; }
    .file-item {
      display: flex;
      align-items: center;
      gap: 0.75rem;
      padding: 0.5rem;
      border: 1px solid #eee;
      border-radius: 4px;
      margin-bottom: 0.5rem;
    }
    .file-preview img { width: 48px; height: 48px; object-fit: cover; border-radius: 4px; }
    .file-icon { font-size: 2rem; }
    .file-info { flex: 1; overflow: hidden; }
    .file-name { display: block; font-weight: 500; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
    .file-size { font-size: 0.8rem; color: #6c757d; }
    .file-progress { width: 100px; }
    .progress-bar { height: 4px; background: #eee; border-radius: 2px; }
    .progress-fill { height: 100%; background: #007bff; border-radius: 2px; transition: width 0.2s; }
    .status-complete { color: #28a745; }
    .status-error { color: #dc3545; cursor: help; }
    .remove-btn {
      background: none;
      border: none;
      font-size: 1.2rem;
      cursor: pointer;
      color: #6c757d;
    }
    .validation-errors { margin-top: 0.5rem; }
    .error-item { color: #dc3545; font-size: 0.9rem; margin: 0.25rem 0; }
    .upload-btn {
      margin-top: 1rem;
      padding: 0.5rem 2rem;
      background: #007bff;
      color: white;
      border: none;
      border-radius: 4px;
      cursor: pointer;
      font-size: 1rem;
    }
    .upload-btn:disabled { opacity: 0.5; }
    .add-more-btn {
      background: none;
      border: 1px dashed #ccc;
      border-radius: 4px;
      padding: 0.25rem 0.75rem;
      cursor: pointer;
      color: #007bff;
    }
  `]
})
export class FileUploadComponent {
  @Input() multiple = false;
  @Input() showUploadButton = true;
  @Input() autoUpload = false;
  @Input() disabled = false;
  @Input() config: FileValidationConfig = {
    maxSize: 10 * 1024 * 1024, // 10MB
    allowedTypes: ['image/jpeg', 'image/png', 'image/gif', 'application/pdf'],
    maxFiles: 5
  };

  @Output() filesSelected = new EventEmitter<File[]>();
  @Output() uploadComplete = new EventEmitter<string[]>();
  @Output() uploadError = new EventEmitter<string>();

  files: UploadedFile[] = [];
  isDragging = false;
  isUploading = false;
  validationErrors: string[] = [];

  constructor(
    private validationService: FileValidationService,
    private uploadService: FileUploadService
  ) {}

  get acceptString(): string {
    return this.config.allowedTypes?.join(',') || '';
  }

  @HostListener('dragover', ['$event'])
  onDragOver(event: DragEvent): void {
    event.preventDefault();
    event.stopPropagation();
    this.isDragging = true;
  }

  @HostListener('dragleave', ['$event'])
  onDragLeave(event: DragEvent): void {
    event.preventDefault();
    event.stopPropagation();
    this.isDragging = false;
  }

  @HostListener('drop', ['$event'])
  onDrop(event: DragEvent): void {
    event.preventDefault();
    event.stopPropagation();
    this.isDragging = false;

    const files = Array.from(event.dataTransfer?.files || []);
    this.handleFiles(files);
  }

  onFileSelected(event: Event): void {
    const input = event.target as HTMLInputElement;
    if (input.files) {
      this.handleFiles(Array.from(input.files));
      input.value = ''; // Reset input
    }
  }

  private async handleFiles(files: File[]): Promise<void> {
    const validation = this.validationService.validate(files, this.config);
    this.validationErrors = validation.errors;

    if (!validation.valid) return;

    for (const file of files) {
      const uploadedFile: UploadedFile = {
        file,
        progress: 0,
        status: 'pending'
      };

      if (this.validationService.isImage(file)) {
        try {
          uploadedFile.preview = await this.validationService.generatePreview(file);
        } catch {}
      }

      this.files.push(uploadedFile);
    }

    this.filesSelected.emit(files);

    if (this.autoUpload) {
      this.uploadAll();
    }
  }

  removeFile(index: number): void {
    this.files.splice(index, 1);
  }

  uploadAll(): void {
    this.isUploading = true;
    const pendingFiles = this.files.filter(f => f.status === 'pending');
    const completedUrls: string[] = [];
    let completed = 0;

    pendingFiles.forEach(item => {
      item.status = 'uploading';
      
      this.uploadService.uploadFile(item.file).subscribe({
        next: (progress) => {
          item.progress = progress.progress;
          if (progress.status === 'complete') {
            item.status = 'complete';
            item.url = progress.url;
            if (progress.url) completedUrls.push(progress.url);
            
            completed++;
            if (completed === pendingFiles.length) {
              this.isUploading = false;
              this.uploadComplete.emit(completedUrls);
            }
          }
        },
        error: (err) => {
          item.status = 'error';
          item.error = err.message;
          completed++;
          this.uploadError.emit(err.message);
          
          if (completed === pendingFiles.length) {
            this.isUploading = false;
          }
        }
      });
    });
  }

  getFileIcon(file: File): string {
    if (this.validationService.isImage(file)) return '🖼️';
    if (this.validationService.isVideo(file)) return '🎥';
    if (this.validationService.isPDF(file)) return '📄';
    if (file.name.endsWith('.zip') || file.name.endsWith('.rar')) return '📦';
    return '📁';
  }

  formatSize(bytes: number): string {
    return this.validationService.formatFileSize(bytes);
  }
}
```

## 4. Image Preview Component

```typescript
// image-preview.component.ts
import { Component, Input, Output, EventEmitter } from '@angular/core';
import { CommonModule } from '@angular/common';

@Component({
  selector: 'app-image-preview',
  standalone: true,
  imports: [CommonModule],
  template: `
    <div class="preview-grid">
      <div
        *ngFor="let image of images; let i = index"
        class="preview-item"
      >
        <img [src]="image.url" [alt]="image.name" (click)="openLightbox(i)" />
        <div class="overlay">
          <button (click)="remove.emit(i)" class="remove-btn">🗑️</button>
        </div>
        <div class="image-name">{{ image.name }}</div>
      </div>
    </div>

    <!-- Lightbox -->
    <div *ngIf="lightboxIndex !== null" class="lightbox" (click)="closeLightbox()">
      <button class="lightbox-prev" (click)="$event.stopPropagation(); prevImage()">‹</button>
      <img
        [src]="images[lightboxIndex!].url"
        [alt]="images[lightboxIndex!].name"
        (click)="$event.stopPropagation()"
      />
      <button class="lightbox-next" (click)="$event.stopPropagation(); nextImage()">›</button>
      <button class="lightbox-close" (click)="closeLightbox()">×</button>
    </div>
  `,
  styles: [`
    .preview-grid {
      display: grid;
      grid-template-columns: repeat(auto-fill, minmax(150px, 1fr));
      gap: 1rem;
    }
    .preview-item {
      position: relative;
      aspect-ratio: 1;
      overflow: hidden;
      border-radius: 8px;
      cursor: pointer;
    }
    .preview-item img {
      width: 100%;
      height: 100%;
      object-fit: cover;
      transition: transform 0.2s;
    }
    .preview-item:hover img { transform: scale(1.05); }
    .overlay {
      position: absolute;
      top: 0; right: 0;
      padding: 0.5rem;
      opacity: 0;
      transition: opacity 0.2s;
    }
    .preview-item:hover .overlay { opacity: 1; }
    .remove-btn {
      background: rgba(0,0,0,0.5);
      border: none;
      border-radius: 50%;
      cursor: pointer;
      padding: 0.25rem;
    }
    .image-name {
      position: absolute;
      bottom: 0; left: 0; right: 0;
      background: rgba(0,0,0,0.5);
      color: white;
      padding: 0.25rem;
      font-size: 0.75rem;
      text-align: center;
      white-space: nowrap;
      overflow: hidden;
      text-overflow: ellipsis;
    }
    .lightbox {
      position: fixed;
      inset: 0;
      background: rgba(0,0,0,0.9);
      display: flex;
      align-items: center;
      justify-content: center;
      z-index: 9999;
    }
    .lightbox img { max-width: 90vw; max-height: 90vh; object-fit: contain; }
    .lightbox-prev, .lightbox-next {
      position: absolute;
      top: 50%;
      transform: translateY(-50%);
      background: rgba(255,255,255,0.1);
      border: none;
      color: white;
      font-size: 3rem;
      cursor: pointer;
      padding: 1rem;
    }
    .lightbox-prev { left: 1rem; }
    .lightbox-next { right: 1rem; }
    .lightbox-close {
      position: absolute;
      top: 1rem; right: 1rem;
      background: none;
      border: none;
      color: white;
      font-size: 2rem;
      cursor: pointer;
    }
  `]
})
export class ImagePreviewComponent {
  @Input() images: { url: string; name: string }[] = [];
  @Output() remove = new EventEmitter<number>();

  lightboxIndex: number | null = null;

  openLightbox(index: number): void {
    this.lightboxIndex = index;
  }

  closeLightbox(): void {
    this.lightboxIndex = null;
  }

  prevImage(): void {
    if (this.lightboxIndex !== null) {
      this.lightboxIndex = (this.lightboxIndex - 1 + this.images.length) % this.images.length;
    }
  }

  nextImage(): void {
    if (this.lightboxIndex !== null) {
      this.lightboxIndex = (this.lightboxIndex + 1) % this.images.length;
    }
  }
}
```

## สรุป

File Upload component ที่สมบูรณ์ควรมี:
- Drag & drop support
- File type validation
- Size validation
- Preview สำหรับรูปภาพ
- Progress indicator
- Multiple file upload
- Error handling
- Accessibility (keyboard navigation)
