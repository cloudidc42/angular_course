# Part 93: Payment Integration (Stripe, Omise) ใน Angular

## Payment Security Principles

- **Never** เก็บ card number ใน client
- ใช้ tokenization (Stripe/Omise ทำให้)
- HTTPS เท่านั้น
- PCI DSS compliance ผ่าน payment provider

---

## 1. Stripe Integration

### ติดตั้ง

```bash
npm install @stripe/stripe-js
```

### Payment Service

```typescript
// core/payment/stripe.service.ts
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { loadStripe, Stripe, StripeElements, PaymentIntent } from '@stripe/stripe-js';
import { environment } from '../../environments/environment';

export interface PaymentResult {
  success: boolean;
  paymentIntentId?: string;
  error?: string;
}

@Injectable({ providedIn: 'root' })
export class StripeService {
  private stripe: Stripe | null = null;
  private elements: StripeElements | null = null;

  constructor(private http: HttpClient) {
    this.initialize();
  }

  private async initialize(): Promise<void> {
    this.stripe = await loadStripe(environment.stripePublicKey);
  }

  async createPaymentIntent(amount: number, currency = 'thb'): Promise<string> {
    const response = await this.http.post<{ clientSecret: string }>('/api/payments/create-intent', {
      amount: amount * 100, // Stripe ใช้ satang/cents
      currency
    }).toPromise();
    
    return response!.clientSecret;
  }

  async mountCardElement(container: HTMLElement): Promise<void> {
    if (!this.stripe) throw new Error('Stripe ไม่พร้อมใช้งาน');

    this.elements = this.stripe.elements({
      appearance: {
        theme: 'stripe',
        variables: {
          colorPrimary: '#2196f3',
          fontFamily: "'Sarabun', sans-serif",
          borderRadius: '8px'
        }
      }
    });

    const cardElement = this.elements.create('card', {
      style: {
        base: {
          fontSize: '16px',
          color: '#333',
          fontFamily: "'Sarabun', sans-serif",
          '::placeholder': { color: '#aab7c4' }
        },
        invalid: { color: '#f44336' }
      },
      hidePostalCode: true
    });

    cardElement.mount(container);

    cardElement.on('change', (event) => {
      if (event.error) {
        console.error('Card error:', event.error.message);
      }
    });
  }

  async processPayment(clientSecret: string, billingDetails: {
    name: string;
    email: string;
    phone?: string;
  }): Promise<PaymentResult> {
    if (!this.stripe || !this.elements) {
      return { success: false, error: 'Stripe ไม่พร้อมใช้งาน' };
    }

    const cardElement = this.elements.getElement('card');
    if (!cardElement) {
      return { success: false, error: 'ไม่พบ card element' };
    }

    const { paymentIntent, error } = await this.stripe.confirmCardPayment(clientSecret, {
      payment_method: {
        card: cardElement,
        billing_details: {
          name: billingDetails.name,
          email: billingDetails.email,
          phone: billingDetails.phone
        }
      }
    });

    if (error) {
      return { success: false, error: error.message };
    }

    return {
      success: paymentIntent?.status === 'succeeded',
      paymentIntentId: paymentIntent?.id
    };
  }

  async createPaymentMethod(billingDetails: {
    name: string;
    email: string;
  }): Promise<string | null> {
    if (!this.stripe || !this.elements) return null;
    
    const cardElement = this.elements.getElement('card');
    if (!cardElement) return null;

    const { paymentMethod, error } = await this.stripe.createPaymentMethod({
      type: 'card',
      card: cardElement,
      billing_details: billingDetails
    });

    if (error) {
      console.error('Payment method error:', error);
      return null;
    }

    return paymentMethod?.id || null;
  }
}
```

---

## 2. Payment Form Component

```typescript
// features/checkout/payment-form/payment-form.component.ts
import { Component, OnInit, AfterViewInit, ElementRef, ViewChild, Output, EventEmitter } from '@angular/core';
import { FormBuilder, FormGroup, Validators } from '@angular/forms';
import { StripeService, PaymentResult } from '../../../core/payment/stripe.service';

interface OrderSummary {
  items: { name: string; price: number; quantity: number }[];
  subtotal: number;
  tax: number;
  total: number;
}

@Component({
  selector: 'app-payment-form',
  template: `
    <div class="payment-form">
      <h3>ชำระเงิน</h3>
      
      <!-- Order Summary -->
      <div class="order-summary">
        <h4>สรุปคำสั่งซื้อ</h4>
        <div class="summary-row" *ngFor="let item of order.items">
          <span>{{ item.name }} x{{ item.quantity }}</span>
          <span>฿{{ item.price * item.quantity | number:'1.2-2' }}</span>
        </div>
        <div class="summary-divider"></div>
        <div class="summary-row">
          <span>ราคาสินค้า</span>
          <span>฿{{ order.subtotal | number:'1.2-2' }}</span>
        </div>
        <div class="summary-row">
          <span>ภาษี (7%)</span>
          <span>฿{{ order.tax | number:'1.2-2' }}</span>
        </div>
        <div class="summary-row total">
          <span>รวมทั้งหมด</span>
          <strong>฿{{ order.total | number:'1.2-2' }}</strong>
        </div>
      </div>

      <!-- Billing Form -->
      <form [formGroup]="billingForm" class="billing-section">
        <h4>ข้อมูลผู้ชำระ</h4>
        <div class="form-row">
          <label>
            ชื่อ-นามสกุล
            <input formControlName="name" placeholder="ชื่อบนบัตร">
          </label>
        </div>
        <div class="form-row">
          <label>
            อีเมล
            <input formControlName="email" type="email" placeholder="email@example.com">
          </label>
        </div>
        <div class="form-row">
          <label>
            เบอร์โทรศัพท์ (ไม่บังคับ)
            <input formControlName="phone" placeholder="0xx-xxx-xxxx">
          </label>
        </div>
      </form>

      <!-- Stripe Card Element -->
      <div class="card-section">
        <h4>ข้อมูลบัตร</h4>
        <div class="card-icons">
          <img src="/assets/icons/visa.svg" alt="Visa" height="24">
          <img src="/assets/icons/mastercard.svg" alt="Mastercard" height="24">
          <img src="/assets/icons/amex.svg" alt="Amex" height="24">
          <img src="/assets/icons/jcb.svg" alt="JCB" height="24">
        </div>
        
        <div 
          #cardContainer 
          class="card-element-container"
          [class.error]="cardError"
        ></div>
        
        <p class="card-error" *ngIf="cardError">{{ cardError }}</p>
        
        <p class="security-note">
          🔒 ข้อมูลบัตรของคุณถูกเข้ารหัสด้วย SSL 256-bit
        </p>
      </div>

      <!-- Submit Button -->
      <button 
        class="btn-pay"
        (click)="processPayment()"
        [disabled]="isProcessing || billingForm.invalid"
      >
        <span *ngIf="!isProcessing">
          ชำระเงิน ฿{{ order.total | number:'1.2-2' }}
        </span>
        <span *ngIf="isProcessing" class="loading-text">
          <div class="spinner-btn"></div>
          กำลังดำเนินการ...
        </span>
      </button>

      <!-- Success State -->
      <div class="success-state" *ngIf="paymentSuccess">
        <div class="success-icon">✅</div>
        <h3>ชำระเงินสำเร็จ!</h3>
        <p>หมายเลขธุรกรรม: {{ paymentId }}</p>
        <p>อีเมลยืนยันจะถูกส่งไปยัง {{ billingForm.get('email')?.value }}</p>
      </div>
    </div>
  `,
  styles: [`
    .payment-form { max-width: 480px; margin: 0 auto; padding: 24px; }
    .order-summary { background: #f9f9f9; border-radius: 8px; padding: 16px; margin-bottom: 24px; }
    .order-summary h4, .billing-section h4, .card-section h4 { margin: 0 0 12px; font-size: 16px; color: #333; }
    .summary-row { display: flex; justify-content: space-between; padding: 4px 0; font-size: 14px; }
    .summary-divider { border-top: 1px solid #ddd; margin: 8px 0; }
    .summary-row.total { font-size: 16px; font-weight: bold; color: #333; }
    .billing-section { margin-bottom: 24px; }
    .form-row { margin-bottom: 16px; }
    .form-row label { display: block; font-size: 14px; color: #666; margin-bottom: 4px; }
    .form-row input { 
      width: 100%; padding: 10px 12px; 
      border: 1px solid #ddd; border-radius: 6px;
      font-size: 15px; box-sizing: border-box;
    }
    .form-row input:focus { border-color: #2196f3; outline: none; }
    .card-section { margin-bottom: 24px; }
    .card-icons { display: flex; gap: 8px; margin-bottom: 12px; }
    .card-element-container {
      padding: 12px;
      border: 1px solid #ddd;
      border-radius: 6px;
      background: white;
    }
    .card-element-container.error { border-color: #f44336; }
    .card-error { color: #f44336; font-size: 13px; margin-top: 4px; }
    .security-note { font-size: 12px; color: #999; margin-top: 8px; }
    .btn-pay {
      width: 100%;
      padding: 14px;
      background: #2196f3;
      color: white;
      border: none;
      border-radius: 8px;
      font-size: 16px;
      font-weight: 600;
      cursor: pointer;
      transition: background 0.2s;
    }
    .btn-pay:hover:not(:disabled) { background: #1976d2; }
    .btn-pay:disabled { opacity: 0.6; cursor: not-allowed; }
    .loading-text { display: flex; align-items: center; justify-content: center; gap: 8px; }
    .spinner-btn {
      width: 18px; height: 18px;
      border: 2px solid rgba(255,255,255,0.3);
      border-top-color: white;
      border-radius: 50%;
      animation: spin 0.8s linear infinite;
    }
    @keyframes spin { to { transform: rotate(360deg); } }
    .success-state { text-align: center; padding: 32px 0; }
    .success-icon { font-size: 48px; margin-bottom: 16px; }
  `]
})
export class PaymentFormComponent implements OnInit, AfterViewInit {
  @ViewChild('cardContainer') cardContainer!: ElementRef;
  @Output() paymentComplete = new EventEmitter<{ success: boolean; paymentId?: string }>();

  order: OrderSummary = {
    items: [{ name: 'สินค้าตัวอย่าง', price: 990, quantity: 1 }],
    subtotal: 990,
    tax: 69.3,
    total: 1059.3
  };

  billingForm: FormGroup;
  isProcessing = false;
  paymentSuccess = false;
  paymentId = '';
  cardError = '';

  constructor(
    private fb: FormBuilder,
    private stripeService: StripeService
  ) {
    this.billingForm = this.fb.group({
      name: ['', [Validators.required, Validators.minLength(3)]],
      email: ['', [Validators.required, Validators.email]],
      phone: ['']
    });
  }

  ngOnInit(): void {}

  async ngAfterViewInit(): Promise<void> {
    await this.stripeService.mountCardElement(this.cardContainer.nativeElement);
  }

  async processPayment(): Promise<void> {
    if (this.billingForm.invalid) return;
    
    this.isProcessing = true;
    this.cardError = '';

    try {
      // สร้าง Payment Intent
      const clientSecret = await this.stripeService.createPaymentIntent(this.order.total);

      // ยืนยันการชำระเงิน
      const result = await this.stripeService.processPayment(clientSecret, {
        name: this.billingForm.get('name')?.value,
        email: this.billingForm.get('email')?.value,
        phone: this.billingForm.get('phone')?.value
      });

      if (result.success) {
        this.paymentSuccess = true;
        this.paymentId = result.paymentIntentId || '';
        this.paymentComplete.emit({ success: true, paymentId: this.paymentId });
      } else {
        this.cardError = result.error || 'การชำระเงินล้มเหลว กรุณาลองใหม่';
      }
    } catch (error: any) {
      this.cardError = 'เกิดข้อผิดพลาด: ' + error.message;
    } finally {
      this.isProcessing = false;
    }
  }
}
```

---

## 3. Omise (ไทย) Integration

```typescript
// core/payment/omise.service.ts
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';

declare const OmiseCard: any;

@Injectable({ providedIn: 'root' })
export class OmiseService {
  constructor(private http: HttpClient) {
    this.loadScript();
  }

  private loadScript(): void {
    const script = document.createElement('script');
    script.src = 'https://cdn.omise.co/card.js';
    script.dataset['key'] = 'pkey_YOUR_PUBLIC_KEY';
    document.head.appendChild(script);
  }

  // สร้าง Token สำหรับ card
  createToken(cardData: {
    name: string;
    number: string;
    expiration_month: string;
    expiration_year: string;
    security_code: string;
  }): Promise<string> {
    return new Promise((resolve, reject) => {
      OmiseCard.createToken('card', cardData, (statusCode: number, response: any) => {
        if (statusCode === 200) {
          resolve(response.id);
        } else {
          reject(new Error(response.message || 'การสร้าง token ล้มเหลว'));
        }
      });
    });
  }

  // สร้าง Charge
  async charge(token: string, amount: number, description: string): Promise<any> {
    return this.http.post('/api/payments/omise/charge', {
      token,
      amount: amount * 100,  // Omise ใช้ satang
      currency: 'THB',
      description
    }).toPromise();
  }
}
```

---

## สรุป

| Payment Provider | เหมาะกับ |
|----------------|---------|
| Stripe | International, ง่ายต่อ integrate |
| Omise | ไทย, รองรับ PromptPay |
| 2C2P | Enterprise Thailand |
| GB Prime Pay | Thailand, หลาย channels |

### Security Checklist

- [ ] ใช้ HTTPS เท่านั้น
- [ ] ไม่ log card data
- [ ] Tokenization ด้วย payment provider
- [ ] Validate amount ทั้ง client และ server
- [ ] CSRF protection
- [ ] Webhook signature validation
- [ ] Store transaction IDs เท่านั้น
