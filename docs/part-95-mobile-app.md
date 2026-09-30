# Part 95: Mobile App ด้วย Ionic Framework กับ Angular

## Ionic คืออะไร

Ionic เป็น framework สำหรับสร้าง hybrid mobile app ที่ใช้ Angular (หรือ React/Vue) เขียนครั้งเดียว รันได้บน iOS, Android และ Web

---

## 1. ติดตั้ง Ionic

```bash
npm install -g @ionic/cli

# สร้าง project ใหม่
ionic start my-app blank --type=angular

# หรือ เลือก template
ionic start my-app tabs --type=angular

# รัน dev server
ionic serve

# รัน บน Android
ionic cap run android

# รัน บน iOS
ionic cap run ios
```

---

## 2. Ionic Components หลัก

```typescript
// app.component.ts
import { Component } from '@angular/core';

@Component({
  selector: 'app-root',
  template: `
    <ion-app>
      <ion-router-outlet></ion-router-outlet>
    </ion-app>
  `
})
export class AppComponent {}
```

```typescript
// features/home/home.page.ts
import { Component, OnInit } from '@angular/core';
import { IonicModule, LoadingController, ToastController, AlertController, ActionSheetController } from '@ionic/angular';
import { ProductService } from '../../core/services/product.service';

interface Product {
  id: number;
  name: string;
  price: number;
  image: string;
  category: string;
  rating: number;
}

@Component({
  selector: 'app-home',
  template: `
    <ion-header>
      <ion-toolbar color="primary">
        <ion-title>ร้านค้า</ion-title>
        <ion-buttons slot="end">
          <ion-button (click)="openCart()">
            <ion-icon name="cart-outline"></ion-icon>
            <ion-badge color="danger" *ngIf="cartCount > 0">{{ cartCount }}</ion-badge>
          </ion-button>
        </ion-buttons>
      </ion-toolbar>
      <ion-toolbar>
        <ion-searchbar
          [(ngModel)]="searchTerm"
          (ionInput)="search($event)"
          placeholder="ค้นหาสินค้า..."
          animated
        ></ion-searchbar>
      </ion-toolbar>
    </ion-header>

    <ion-content>
      <!-- Refresher -->
      <ion-refresher slot="fixed" (ionRefresh)="refresh($event)">
        <ion-refresher-content></ion-refresher-content>
      </ion-refresher>

      <!-- Category Chips -->
      <div class="category-chips">
        <ion-chip
          *ngFor="let cat of categories"
          [color]="selectedCategory === cat ? 'primary' : 'medium'"
          (click)="filterByCategory(cat)"
        >
          {{ cat }}
        </ion-chip>
      </div>

      <!-- Product Grid -->
      <ion-grid>
        <ion-row>
          <ion-col size="6" *ngFor="let product of filteredProducts">
            <ion-card (click)="viewProduct(product)">
              <img [src]="product.image" [alt]="product.name">
              <ion-card-content>
                <h3>{{ product.name }}</h3>
                <p class="price">฿{{ product.price | number }}</p>
                <div class="rating">
                  <ion-icon 
                    name="star" 
                    color="warning"
                    *ngFor="let star of [1,2,3,4,5]; let i = index"
                    [name]="i < product.rating ? 'star' : 'star-outline'"
                  ></ion-icon>
                </div>
                <ion-button 
                  expand="block" 
                  size="small"
                  (click)="addToCart(product, $event)"
                >
                  เพิ่มลงตะกร้า
                </ion-button>
              </ion-card-content>
            </ion-card>
          </ion-col>
        </ion-row>
      </ion-grid>

      <!-- Infinite Scroll -->
      <ion-infinite-scroll (ionInfinite)="loadMore($event)">
        <ion-infinite-scroll-content
          loadingSpinner="bubbles"
          loadingText="กำลังโหลด..."
        ></ion-infinite-scroll-content>
      </ion-infinite-scroll>
    </ion-content>
  `,
  styles: [`
    .category-chips {
      padding: 8px 16px;
      display: flex;
      gap: 8px;
      overflow-x: auto;
    }
    ion-card img { width: 100%; height: 140px; object-fit: cover; }
    .price { color: #e91e63; font-weight: 700; font-size: 16px; }
    .rating { display: flex; margin-bottom: 8px; }
  `]
})
export class HomePage implements OnInit {
  products: Product[] = [];
  filteredProducts: Product[] = [];
  categories = ['ทั้งหมด', 'อิเล็กทรอนิกส์', 'เสื้อผ้า', 'อาหาร'];
  selectedCategory = 'ทั้งหมด';
  searchTerm = '';
  cartCount = 0;
  page = 1;

  constructor(
    private productService: ProductService,
    private loadingCtrl: LoadingController,
    private toastCtrl: ToastController,
    private alertCtrl: AlertController,
    private actionSheetCtrl: ActionSheetController
  ) {}

  async ngOnInit(): Promise<void> {
    await this.loadProducts();
  }

  async loadProducts(): Promise<void> {
    const loading = await this.loadingCtrl.create({
      message: 'กำลังโหลดสินค้า...',
      spinner: 'crescent'
    });
    await loading.present();

    try {
      this.products = await this.productService.getProducts().toPromise() as Product[];
      this.filteredProducts = this.products;
    } finally {
      await loading.dismiss();
    }
  }

  search(event: any): void {
    const term = event.detail.value?.toLowerCase() || '';
    this.filteredProducts = this.products.filter(p =>
      p.name.toLowerCase().includes(term)
    );
  }

  filterByCategory(category: string): void {
    this.selectedCategory = category;
    if (category === 'ทั้งหมด') {
      this.filteredProducts = this.products;
    } else {
      this.filteredProducts = this.products.filter(p => p.category === category);
    }
  }

  async addToCart(product: Product, event: Event): Promise<void> {
    event.stopPropagation();
    this.cartCount++;

    const toast = await this.toastCtrl.create({
      message: `เพิ่ม ${product.name} ลงตะกร้าแล้ว`,
      duration: 2000,
      position: 'bottom',
      color: 'success',
      buttons: [{ text: 'ดูตะกร้า', role: 'undo' }]
    });
    await toast.present();
  }

  async viewProduct(product: Product): Promise<void> {
    const actionSheet = await this.actionSheetCtrl.create({
      header: product.name,
      buttons: [
        {
          text: 'ดูรายละเอียด',
          icon: 'eye-outline',
          handler: () => { /* navigate to detail */ }
        },
        {
          text: 'เพิ่มลงตะกร้า',
          icon: 'cart-outline',
          handler: () => this.addToCart(product, new Event('click'))
        },
        {
          text: 'บันทึกในรายการโปรด',
          icon: 'heart-outline',
          handler: () => { /* add to wishlist */ }
        },
        {
          text: 'ยกเลิก',
          role: 'cancel'
        }
      ]
    });
    await actionSheet.present();
  }

  async refresh(event: any): Promise<void> {
    await this.loadProducts();
    event.target.complete();
  }

  async loadMore(event: any): Promise<void> {
    // โหลดหน้าถัดไป
    this.page++;
    const more = await this.productService.getProducts({ page: this.page }).toPromise() as Product[];
    
    if (more && more.length) {
      this.products = [...this.products, ...more];
      this.filteredProducts = [...this.filteredProducts, ...more];
      event.target.complete();
    } else {
      event.target.disabled = true;
    }
  }

  openCart(): void { /* navigate to cart */ }
}
```

---

## 3. Tab Navigation

```typescript
// app-routing.module.ts
const routes: Routes = [
  {
    path: 'tabs',
    component: TabsPage,
    children: [
      { path: 'home', loadChildren: () => import('./home/home.module').then(m => m.HomePageModule) },
      { path: 'search', loadChildren: () => import('./search/search.module').then(m => m.SearchPageModule) },
      { path: 'cart', loadChildren: () => import('./cart/cart.module').then(m => m.CartPageModule) },
      { path: 'profile', loadChildren: () => import('./profile/profile.module').then(m => m.ProfilePageModule) },
    ]
  },
  { path: '', redirectTo: 'tabs/home', pathMatch: 'full' }
];
```

```typescript
// tabs/tabs.page.ts
import { Component } from '@angular/core';

@Component({
  selector: 'app-tabs',
  template: `
    <ion-tabs>
      <ion-tab-bar slot="bottom">
        <ion-tab-button tab="home">
          <ion-icon name="home-outline"></ion-icon>
          <ion-label>หน้าแรก</ion-label>
        </ion-tab-button>
        <ion-tab-button tab="search">
          <ion-icon name="search-outline"></ion-icon>
          <ion-label>ค้นหา</ion-label>
        </ion-tab-button>
        <ion-tab-button tab="cart">
          <ion-icon name="cart-outline"></ion-icon>
          <ion-label>ตะกร้า</ion-label>
          <ion-badge color="danger">{{ cartCount }}</ion-badge>
        </ion-tab-button>
        <ion-tab-button tab="profile">
          <ion-icon name="person-outline"></ion-icon>
          <ion-label>บัญชี</ion-label>
        </ion-tab-button>
      </ion-tab-bar>
    </ion-tabs>
  `
})
export class TabsPage {
  cartCount = 0;
}
```

---

## 4. Capacitor Plugin - Camera

```bash
npm install @capacitor/camera
npx cap sync
```

```typescript
// features/profile/profile.page.ts
import { Component } from '@angular/core';
import { Camera, CameraResultType, CameraSource } from '@capacitor/camera';

@Component({
  selector: 'app-profile',
  template: `
    <ion-header>
      <ion-toolbar>
        <ion-title>โปรไฟล์</ion-title>
      </ion-toolbar>
    </ion-header>

    <ion-content>
      <div class="profile-header">
        <div class="avatar-container" (click)="changeAvatar()">
          <img [src]="avatarUrl || '/assets/default-avatar.png'" class="avatar">
          <div class="avatar-overlay">
            <ion-icon name="camera-outline"></ion-icon>
          </div>
        </div>
        <h2>{{ userName }}</h2>
        <p>{{ userEmail }}</p>
      </div>

      <ion-list>
        <ion-item>
          <ion-label>
            <h3>ชื่อ</h3>
            <p>{{ userName }}</p>
          </ion-label>
          <ion-button slot="end" fill="clear" (click)="editName()">แก้ไข</ion-button>
        </ion-item>
        <ion-item>
          <ion-label>
            <h3>อีเมล</h3>
            <p>{{ userEmail }}</p>
          </ion-label>
        </ion-item>
        <ion-item>
          <ion-label>
            <h3>เบอร์โทรศัพท์</h3>
            <p>{{ userPhone || 'ยังไม่ได้ระบุ' }}</p>
          </ion-label>
          <ion-button slot="end" fill="clear" (click)="editPhone()">แก้ไข</ion-button>
        </ion-item>
      </ion-list>
    </ion-content>
  `,
  styles: [`
    .profile-header { text-align: center; padding: 24px; }
    .avatar-container { position: relative; display: inline-block; }
    .avatar { width: 100px; height: 100px; border-radius: 50%; object-fit: cover; }
    .avatar-overlay {
      position: absolute; bottom: 0; right: 0;
      background: #333; border-radius: 50%;
      width: 28px; height: 28px;
      display: flex; align-items: center; justify-content: center;
      color: white; font-size: 14px;
    }
  `]
})
export class ProfilePage {
  avatarUrl = '';
  userName = 'ผู้ใช้ Angular';
  userEmail = 'user@example.com';
  userPhone = '';

  async changeAvatar(): Promise<void> {
    const image = await Camera.getPhoto({
      quality: 85,
      allowEditing: true,
      resultType: CameraResultType.DataUrl,
      source: CameraSource.Prompt,
      promptLabelHeader: 'เลือกรูปโปรไฟล์',
      promptLabelPhoto: 'จากคลังรูป',
      promptLabelPicture: 'ถ่ายรูป'
    });

    if (image.dataUrl) {
      this.avatarUrl = image.dataUrl;
      // อัพโหลดไปยัง server
    }
  }

  editName(): void { /* open modal */ }
  editPhone(): void { /* open modal */ }
}
```

---

## 5. Capacitor Plugins อื่น ๆ

```typescript
// core/device/device.service.ts
import { Injectable } from '@angular/core';
import { Haptics, ImpactStyle } from '@capacitor/haptics';
import { Network } from '@capacitor/network';
import { Geolocation } from '@capacitor/geolocation';
import { App } from '@capacitor/app';

@Injectable({ providedIn: 'root' })
export class DeviceService {
  // Haptic Feedback
  async vibrate(style: 'light' | 'medium' | 'heavy' = 'medium'): Promise<void> {
    const styleMap = {
      light: ImpactStyle.Light,
      medium: ImpactStyle.Medium,
      heavy: ImpactStyle.Heavy
    };
    await Haptics.impact({ style: styleMap[style] });
  }

  // ตรวจสอบ Network
  async getNetworkStatus() {
    return await Network.getStatus();
  }

  setupNetworkListener(callback: (connected: boolean) => void): void {
    Network.addListener('networkStatusChange', status => {
      callback(status.connected);
    });
  }

  // Geolocation
  async getCurrentPosition() {
    return await Geolocation.getCurrentPosition({
      enableHighAccuracy: true,
      timeout: 10000
    });
  }

  // App State
  setupAppStateListener(callback: (isActive: boolean) => void): void {
    App.addListener('appStateChange', state => {
      callback(state.isActive);
    });
  }
}
```

---

## สรุป

| Feature | Ionic/Capacitor |
|---------|-----------------|
| UI Components | ion-card, ion-list, ion-tabs |
| Navigation | ion-tabs, NavController |
| Device API | Camera, GPS, Haptics |
| Network | Network plugin |
| Build Target | iOS, Android, PWA |

### ข้อแตกต่าง Native vs Hybrid

| เรื่อง | Native | Hybrid (Ionic) |
|--------|--------|----------------|
| Performance | สูงสุด | ดี |
| Code sharing | 0% | ~90% |
| Dev speed | ช้า | เร็ว |
| Native API | ทั้งหมด | ผ่าน Capacitor |
| Team | 2 team | 1 team |
