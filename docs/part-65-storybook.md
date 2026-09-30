# Part 65: Storybook สำหรับ Angular

## บทนำ

Storybook เป็นเครื่องมือพัฒนา UI components แบบ isolated ช่วยให้สามารถสร้าง document และทดสอบ components ได้โดยไม่ต้องรันแอปพลิเคชันทั้งหมด

## 1. การติดตั้ง

```bash
npx storybook@latest init --type angular

# หรือสำหรับ project ที่มีอยู่
ng add @storybook/angular
```

### โครงสร้างไฟล์

```
.storybook/
├── main.ts          # Storybook configuration
├── preview.ts       # Global decorators, parameters
└── manager.ts       # UI customization

src/
├── stories/
│   ├── Introduction.stories.mdx  # Documentation
│   └── ...
├── app/
│   ├── button/
│   │   ├── button.component.ts
│   │   ├── button.component.spec.ts
│   │   └── button.stories.ts    # Stories อยู่ใกล้ component
```

## 2. ตั้งค่า Storybook

```typescript
// .storybook/main.ts
import type { StorybookConfig } from '@storybook/angular';

const config: StorybookConfig = {
  stories: ['../src/**/*.stories.@(js|jsx|ts|tsx|mdx)'],
  addons: [
    '@storybook/addon-essentials',      // Controls, Actions, Docs, Viewport, Backgrounds
    '@storybook/addon-interactions',     // Interaction testing
    '@storybook/addon-a11y',            // Accessibility checking
    '@storybook/addon-storysource',     // Show story source code
  ],
  framework: {
    name: '@storybook/angular',
    options: {}
  },
  docs: {
    autodocs: 'tag'
  }
};

export default config;
```

```typescript
// .storybook/preview.ts
import type { Preview } from '@storybook/angular';
import { applicationConfig } from '@storybook/angular';
import { importProvidersFrom } from '@angular/core';
import { BrowserAnimationsModule } from '@angular/platform-browser/animations';
import { HttpClientModule } from '@angular/common/http';

const preview: Preview = {
  parameters: {
    actions: { argTypesRegex: '^on[A-Z].*' },
    controls: {
      matchers: {
        color: /(background|color)$/i,
        date: /Date$/i,
      }
    },
    backgrounds: {
      default: 'light',
      values: [
        { name: 'light', value: '#ffffff' },
        { name: 'dark', value: '#1a1a2e' },
        { name: 'gray', value: '#f8f9fa' }
      ]
    },
    viewport: {
      viewports: {
        mobile: { name: 'Mobile', styles: { width: '375px', height: '812px' } },
        tablet: { name: 'Tablet', styles: { width: '768px', height: '1024px' } },
        desktop: { name: 'Desktop', styles: { width: '1280px', height: '900px' } }
      }
    }
  },
  decorators: [
    applicationConfig({
      providers: [
        importProvidersFrom(BrowserAnimationsModule),
        importProvidersFrom(HttpClientModule)
      ]
    })
  ]
};

export default preview;
```

## 3. Writing Stories

### Button Component Stories

```typescript
// src/app/shared/ui/button/button.stories.ts
import type { Meta, StoryObj } from '@storybook/angular';
import { ButtonComponent } from './button.component';

const meta: Meta<ButtonComponent> = {
  title: 'Shared/UI/Button',
  component: ButtonComponent,
  tags: ['autodocs'],
  argTypes: {
    variant: {
      control: 'select',
      options: ['primary', 'secondary', 'danger', 'outline'],
      description: 'ประเภทของปุ่ม',
      table: {
        defaultValue: { summary: 'primary' }
      }
    },
    size: {
      control: 'select',
      options: ['sm', 'md', 'lg'],
      description: 'ขนาดของปุ่ม'
    },
    disabled: {
      control: 'boolean',
      description: 'ปิดใช้งานปุ่ม'
    },
    loading: {
      control: 'boolean',
      description: 'แสดงสถานะกำลังโหลด'
    },
    clicked: { action: 'clicked' }
  },
  args: {
    variant: 'primary',
    size: 'md',
    disabled: false,
    loading: false
  }
};

export default meta;
type Story = StoryObj<ButtonComponent>;

// Basic stories
export const Primary: Story = {
  args: { variant: 'primary' },
  render: (args) => ({
    props: args,
    template: '<my-company-button [variant]="variant" [size]="size" [disabled]="disabled" [loading]="loading" (clicked)="clicked($event)">คลิกที่นี่</my-company-button>'
  })
};

export const Secondary: Story = {
  args: { variant: 'secondary' },
  render: (args) => ({
    props: args,
    template: '<my-company-button [variant]="variant">ยกเลิก</my-company-button>'
  })
};

export const Danger: Story = {
  args: { variant: 'danger' },
  render: (args) => ({
    props: args,
    template: '<my-company-button [variant]="variant">ลบ</my-company-button>'
  })
};

export const Loading: Story = {
  args: { loading: true },
  render: (args) => ({
    props: args,
    template: '<my-company-button [loading]="loading">กำลังบันทึก...</my-company-button>'
  })
};

export const Disabled: Story = {
  args: { disabled: true },
  render: (args) => ({
    props: args,
    template: '<my-company-button [disabled]="disabled">ปุ่มถูกปิดใช้งาน</my-company-button>'
  })
};

// Size variants
export const AllSizes: Story = {
  render: () => ({
    template: `
      <div style="display: flex; gap: 1rem; align-items: center;">
        <my-company-button size="sm">Small</my-company-button>
        <my-company-button size="md">Medium</my-company-button>
        <my-company-button size="lg">Large</my-company-button>
      </div>
    `
  })
};

// All variants
export const AllVariants: Story = {
  render: () => ({
    template: `
      <div style="display: flex; gap: 1rem; flex-wrap: wrap;">
        <my-company-button variant="primary">Primary</my-company-button>
        <my-company-button variant="secondary">Secondary</my-company-button>
        <my-company-button variant="danger">Danger</my-company-button>
        <my-company-button variant="outline">Outline</my-company-button>
      </div>
    `
  })
};
```

### Form Component Stories

```typescript
// src/app/shared/ui/form/input/input.stories.ts
import type { Meta, StoryObj } from '@storybook/angular';
import { FormsModule, ReactiveFormsModule, FormControl, Validators } from '@angular/forms';
import { InputComponent } from './input.component';

const meta: Meta<InputComponent> = {
  title: 'Shared/UI/Form/Input',
  component: InputComponent,
  tags: ['autodocs'],
  decorators: [
    moduleMetadata({
      imports: [FormsModule, ReactiveFormsModule]
    })
  ],
  argTypes: {
    type: {
      control: 'select',
      options: ['text', 'email', 'password', 'number', 'tel']
    },
    label: { control: 'text' },
    placeholder: { control: 'text' },
    disabled: { control: 'boolean' },
    required: { control: 'boolean' }
  }
};

export default meta;
type Story = StoryObj<InputComponent>;

export const Default: Story = {
  args: {
    label: 'ชื่อ',
    placeholder: 'กรอกชื่อของคุณ'
  }
};

export const WithError: Story = {
  render: () => ({
    template: `
      <app-input
        label="อีเมล"
        type="email"
        [formControl]="emailControl"
        [showErrors]="true"
      ></app-input>
    `,
    props: {
      emailControl: new FormControl('invalid-email', [Validators.email, Validators.required])
    }
  })
};

export const Password: Story = {
  args: {
    type: 'password',
    label: 'รหัสผ่าน',
    placeholder: 'กรอกรหัสผ่านของคุณ'
  }
};
```

## 4. Stories with Actions และ Controls

```typescript
// src/app/products/product-card/product-card.stories.ts
import type { Meta, StoryObj } from '@storybook/angular';
import { moduleMetadata } from '@storybook/angular';
import { RouterTestingModule } from '@angular/router/testing';
import { ProductCardComponent } from './product-card.component';
import { Product } from '../models/product.model';

const mockProduct: Product = {
  id: '1',
  name: 'iPhone 15 Pro',
  description: 'สมาร์ทโฟนรุ่นล่าสุดจาก Apple',
  price: 45900,
  originalPrice: 49900,
  stock: 15,
  rating: 4.5,
  reviewCount: 128,
  images: ['https://via.placeholder.com/300'],
  category: 'สมาร์ทโฟน',
  tags: ['apple', 'iphone', 'flagship'],
  isFeatured: true
};

const meta: Meta<ProductCardComponent> = {
  title: 'Products/ProductCard',
  component: ProductCardComponent,
  tags: ['autodocs'],
  decorators: [
    moduleMetadata({
      imports: [RouterTestingModule]
    })
  ],
  argTypes: {
    addToCart: { action: 'addToCart' },
    addToWishlist: { action: 'addToWishlist' }
  }
};

export default meta;
type Story = StoryObj<ProductCardComponent>;

export const Default: Story = {
  args: { product: mockProduct }
};

export const OutOfStock: Story = {
  args: {
    product: { ...mockProduct, stock: 0 }
  }
};

export const WithDiscount: Story = {
  args: {
    product: {
      ...mockProduct,
      price: 35900,
      originalPrice: 45900
    }
  }
};

export const LowStock: Story = {
  args: {
    product: { ...mockProduct, stock: 3 }
  }
};

export const Grid: Story = {
  render: () => ({
    template: `
      <div style="display: grid; grid-template-columns: repeat(3, 1fr); gap: 1rem; padding: 1rem;">
        <app-product-card [product]="product"></app-product-card>
        <app-product-card [product]="outOfStock"></app-product-card>
        <app-product-card [product]="withDiscount"></app-product-card>
      </div>
    `,
    props: {
      product: mockProduct,
      outOfStock: { ...mockProduct, stock: 0 },
      withDiscount: { ...mockProduct, price: 35900 }
    }
  })
};
```

## 5. Interaction Testing

```typescript
// src/app/login/login.stories.ts
import type { Meta, StoryObj } from '@storybook/angular';
import { within, userEvent, expect } from '@storybook/test';
import { moduleMetadata } from '@storybook/angular';
import { ReactiveFormsModule } from '@angular/forms';
import { LoginComponent } from './login.component';

const meta: Meta<LoginComponent> = {
  title: 'Auth/LoginForm',
  component: LoginComponent,
  decorators: [
    moduleMetadata({
      imports: [ReactiveFormsModule]
    })
  ]
};

export default meta;
type Story = StoryObj<LoginComponent>;

export const EmptyForm: Story = {};

export const FilledForm: Story = {
  play: async ({ canvasElement }) => {
    const canvas = within(canvasElement);
    
    // กรอกข้อมูล
    await userEvent.type(canvas.getByLabelText('อีเมล'), 'test@example.com');
    await userEvent.type(canvas.getByLabelText('รหัสผ่าน'), 'Password123!');
    
    // ตรวจสอบว่า submit button ถูก enable
    const submitButton = canvas.getByRole('button', { name: 'เข้าสู่ระบบ' });
    await expect(submitButton).not.toBeDisabled();
  }
};

export const ValidationErrors: Story = {
  play: async ({ canvasElement }) => {
    const canvas = within(canvasElement);
    
    // คลิก submit โดยไม่กรอกข้อมูล
    await userEvent.click(canvas.getByRole('button', { name: 'เข้าสู่ระบบ' }));
    
    // ตรวจสอบ error messages
    await expect(canvas.getByText('กรุณากรอกอีเมล')).toBeInTheDocument();
    await expect(canvas.getByText('กรุณากรอกรหัสผ่าน')).toBeInTheDocument();
  }
};
```

## 6. Accessibility Testing

```typescript
// src/app/modal/modal.stories.ts
import type { Meta, StoryObj } from '@storybook/angular';
import { ModalComponent } from './modal.component';

const meta: Meta<ModalComponent> = {
  title: 'Shared/UI/Modal',
  component: ModalComponent,
  parameters: {
    a11y: {
      // ตั้งค่า accessibility rules
      config: {
        rules: [
          { id: 'color-contrast', enabled: true },
          { id: 'keyboard-navigation', enabled: true }
        ]
      }
    }
  }
};

export default meta;

export const Accessible: StoryObj<ModalComponent> = {
  args: {
    title: 'ยืนยันการลบ',
    open: true
  },
  render: (args) => ({
    props: args,
    template: `
      <app-modal [title]="title" [open]="open">
        <p>คุณต้องการลบรายการนี้ใช่หรือไม่?</p>
        <footer>
          <button aria-label="ยืนยันการลบ">ยืนยัน</button>
          <button aria-label="ยกเลิก">ยกเลิก</button>
        </footer>
      </app-modal>
    `
  })
};
```

## 7. MDX Documentation

```mdx
<!-- src/stories/Introduction.stories.mdx -->
import { Meta } from '@storybook/blocks';

<Meta title="Introduction" />

# Component Library

ยินดีต้อนรับสู่ Component Library ของเรา

## Getting Started

```bash
npm run storybook
```

## Design Tokens

| Token | Value | Usage |
|-------|-------|-------|
| `--color-primary` | `#007bff` | Primary actions |
| `--color-danger` | `#dc3545` | Destructive actions |
| `--spacing-sm` | `0.5rem` | Small gaps |
```

## สรุป Storybook Best Practices

| หัวข้อ | แนวทาง |
|-------|-------|
| Stories | สร้าง story สำหรับทุก state ของ component |
| Controls | ใช้ argTypes เพื่อ document props |
| Actions | ใช้ action สำหรับ event callbacks |
| Docs | ใช้ autodocs tag |
| Testing | ใช้ play function สำหรับ interaction tests |
| Accessibility | ติดตั้ง @storybook/addon-a11y |

Storybook ช่วยให้ทีมสามารถพัฒนา, document และ test UI components ได้อย่างมีประสิทธิภาพ
