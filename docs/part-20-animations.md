# Part 20 — Angular Animations

## บทนำ

Angular มี Animation API ที่ทรงพลังซึ่ง build บน Web Animations API ทำให้เราสามารถสร้าง animations ที่ซับซ้อนได้โดยใช้ TypeScript/JavaScript

### ข้อดีของ Angular Animations
- Cross-browser compatible
- ทำงานได้ดีกับ Angular's change detection
- รองรับ Server-Side Rendering
- Declarative API ที่อ่านง่าย

---

## 1. ติดตั้ง BrowserAnimationsModule

### 1.1 Standalone Application (Angular 14+)

```typescript
// src/main.ts
import { bootstrapApplication } from '@angular/platform-browser';
import { provideAnimations } from '@angular/platform-browser/animations';
import { AppComponent } from './app/app.component';

bootstrapApplication(AppComponent, {
  providers: [
    provideAnimations()
    // หรือ provideNoopAnimations() สำหรับ testing/disable animations
  ]
});
```

### 1.2 NgModule Application

```typescript
// src/app/app.module.ts
import { NgModule } from '@angular/core';
import { BrowserModule } from '@angular/platform-browser';
import { BrowserAnimationsModule } from '@angular/platform-browser/animations';

@NgModule({
  imports: [
    BrowserModule,
    BrowserAnimationsModule
  ]
})
export class AppModule {}
```

---

## 2. Building Blocks: trigger, state, style, transition, animate

### 2.1 trigger

`trigger` เป็นจุดเริ่มต้นของ animation ทุกตัว

```typescript
import {
  trigger,
  state,
  style,
  transition,
  animate
} from '@angular/animations';

// syntax: trigger(name, [definitions])
const myAnimation = trigger('myAnimation', [
  // state, transition definitions ไปที่นี่
]);
```

### 2.2 state

`state` กำหนด style สำหรับแต่ละ state

```typescript
import { trigger, state, style } from '@angular/animations';

trigger('expandCollapse', [
  state('collapsed', style({
    height: '0',
    overflow: 'hidden',
    opacity: 0
  })),
  state('expanded', style({
    height: '*',  // * หมายถึงขนาดธรรมชาติของ element
    overflow: 'visible',
    opacity: 1
  }))
])
```

### 2.3 transition

`transition` กำหนดว่าจะ animate อย่างไรเมื่อเปลี่ยน state

```typescript
import { trigger, state, style, transition, animate } from '@angular/animations';

trigger('expandCollapse', [
  state('collapsed', style({ height: '0', opacity: 0 })),
  state('expanded', style({ height: '*', opacity: 1 })),

  // transition จาก collapsed ไป expanded
  transition('collapsed => expanded', [
    animate('300ms ease-in')
  ]),

  // transition จาก expanded ไป collapsed
  transition('expanded => collapsed', [
    animate('200ms ease-out')
  ]),

  // ทั้งสองทิศทาง (shorthand)
  transition('collapsed <=> expanded', [
    animate('300ms ease-in-out')
  ])
])
```

### 2.4 animate

`animate` กำหนด timing และ easing ของ animation

```typescript
// syntax: animate(timings, styles?)

animate('300ms')              // 300 milliseconds, default easing
animate('0.3s')               // 0.3 seconds
animate('300ms ease-in')      // 300ms พร้อม ease-in
animate('300ms 100ms')        // 300ms duration, 100ms delay
animate('300ms 100ms ease-out')  // duration + delay + easing

// Timing functions
animate('300ms linear')
animate('300ms ease')
animate('300ms ease-in')
animate('300ms ease-out')
animate('300ms ease-in-out')
animate('300ms cubic-bezier(0.25, 0.8, 0.25, 1)')
```

### 2.5 ตัวอย่างสมบูรณ์ — Expand/Collapse Animation

```typescript
// src/app/components/expandable/expandable.component.ts
import { Component, Input } from '@angular/core';
import {
  trigger,
  state,
  style,
  transition,
  animate
} from '@angular/animations';

@Component({
  selector: 'app-expandable',
  standalone: true,
  animations: [
    trigger('expandCollapse', [
      state('collapsed', style({
        height: '0px',
        overflow: 'hidden',
        opacity: 0,
        paddingTop: '0',
        paddingBottom: '0'
      })),
      state('expanded', style({
        height: '*',
        overflow: 'visible',
        opacity: 1
      })),
      transition('collapsed <=> expanded', [
        animate('300ms cubic-bezier(0.4, 0.0, 0.2, 1)')
      ])
    ])
  ],
  template: `
    <div class="expandable-wrapper">
      <div class="header" (click)="toggle()">
        <h3>{{ title }}</h3>
        <span class="icon">{{ isExpanded ? '▲' : '▼' }}</span>
      </div>

      <div [@expandCollapse]="isExpanded ? 'expanded' : 'collapsed'">
        <div class="content">
          <ng-content></ng-content>
        </div>
      </div>
    </div>
  `,
  styles: [`
    .expandable-wrapper { border: 1px solid #ddd; border-radius: 4px; }
    .header {
      display: flex;
      justify-content: space-between;
      padding: 12px 16px;
      cursor: pointer;
      background: #f8f9fa;
    }
    .content { padding: 16px; }
  `]
})
export class ExpandableComponent {
  @Input() title = '';
  isExpanded = false;

  toggle(): void {
    this.isExpanded = !this.isExpanded;
  }
}
```

---

## 3. Advanced Animation Functions

### 3.1 keyframes — กำหนด intermediate steps

```typescript
import { keyframes } from '@angular/animations';

trigger('bounce', [
  transition(':enter', [
    animate('600ms', keyframes([
      style({ opacity: 0, transform: 'translateY(-100%)', offset: 0 }),
      style({ opacity: 1, transform: 'translateY(20px)', offset: 0.6 }),
      style({ transform: 'translateY(-10px)', offset: 0.75 }),
      style({ transform: 'translateY(5px)', offset: 0.9 }),
      style({ transform: 'translateY(0)', offset: 1.0 })
    ]))
  ])
])
```

### 3.2 group — animations แบบขนาน

```typescript
import { group } from '@angular/animations';

trigger('cardFlip', [
  state('front', style({ transform: 'rotateY(0)' })),
  state('back', style({ transform: 'rotateY(180deg)' })),

  transition('front => back', [
    group([
      // ทั้งสองทำงานพร้อมกัน
      animate('400ms', style({ transform: 'rotateY(180deg)' })),
      animate('600ms', style({ backgroundColor: '#3f51b5' }))
    ])
  ])
])
```

### 3.3 sequence — animations แบบต่อเนื่อง

```typescript
import { sequence } from '@angular/animations';

trigger('slideAndFade', [
  transition(':enter', [
    sequence([
      // ทำงานต่อเนื่อง
      animate('200ms', style({ transform: 'translateX(-20px)' })),
      animate('400ms', style({
        transform: 'translateX(0)',
        opacity: 1
      }))
    ])
  ])
])
```

### 3.4 stagger — animate รายการแบบ cascade

```typescript
import { query, stagger } from '@angular/animations';

trigger('listAnimation', [
  transition('* => *', [
    query(':enter', [
      style({ opacity: 0, transform: 'translateY(-20px)' }),
      stagger(100, [  // delay 100ms ระหว่างแต่ละ item
        animate('300ms ease-out', style({
          opacity: 1,
          transform: 'translateY(0)'
        }))
      ])
    ], { optional: true })
  ])
])
```

### 3.5 query — animate child elements

```typescript
import { query, animateChild } from '@angular/animations';

trigger('pageAnimation', [
  transition(':enter', [
    query('.hero', [
      style({ opacity: 0, transform: 'scale(0.8)' }),
      animate('300ms', style({ opacity: 1, transform: 'scale(1)' }))
    ]),
    query('.content', [
      style({ opacity: 0 }),
      animate('200ms 100ms', style({ opacity: 1 }))
    ]),
    // animate children ด้วย
    query('@childAnimation', animateChild(), { optional: true })
  ])
])
```

---

## 4. Special Transitions

### 4.1 :enter และ :leave

```typescript
trigger('fadeInOut', [
  // :enter === void => *
  transition(':enter', [
    style({ opacity: 0, transform: 'translateY(-20px)' }),
    animate('300ms ease-out', style({ opacity: 1, transform: 'translateY(0)' }))
  ]),
  // :leave === * => void
  transition(':leave', [
    animate('200ms ease-in', style({ opacity: 0, transform: 'translateY(-20px)' }))
  ])
])
```

### 4.2 :increment และ :decrement

```typescript
trigger('slideCounter', [
  transition(':increment', [
    style({ transform: 'translateY(-100%)', opacity: 0 }),
    animate('300ms ease-out', style({ transform: 'translateY(0)', opacity: 1 }))
  ]),
  transition(':decrement', [
    style({ transform: 'translateY(100%)', opacity: 0 }),
    animate('300ms ease-out', style({ transform: 'translateY(0)', opacity: 1 }))
  ])
])
```

---

## 5. Route Transition Animations

### 5.1 สร้าง Route Animations

```typescript
// src/app/animations/route-animations.ts
import {
  trigger,
  transition,
  style,
  query,
  animateChild,
  group,
  animate
} from '@angular/animations';

export const slideInAnimation = trigger('routeAnimations', [
  // Slide in จากขวาเมื่อ navigate ไปข้างหน้า
  transition('HomePage => AboutPage', [
    style({ position: 'relative' }),
    query(':enter, :leave', [
      style({
        position: 'absolute',
        top: 0,
        left: 0,
        width: '100%'
      })
    ]),
    query(':enter', [
      style({ left: '100%' })
    ]),
    query(':leave', animateChild()),
    group([
      query(':leave', [
        animate('300ms ease-out', style({ left: '-100%' }))
      ]),
      query(':enter', [
        animate('300ms ease-out', style({ left: '0%' }))
      ])
    ])
  ]),

  // Slide in จากซ้ายเมื่อ navigate ย้อนกลับ
  transition('AboutPage => HomePage', [
    style({ position: 'relative' }),
    query(':enter, :leave', [
      style({ position: 'absolute', top: 0, left: 0, width: '100%' })
    ]),
    query(':enter', [style({ left: '-100%' })]),
    group([
      query(':leave', [
        animate('300ms ease-out', style({ left: '100%' }))
      ]),
      query(':enter', [
        animate('300ms ease-out', style({ left: '0%' }))
      ])
    ])
  ]),

  // Fade สำหรับ routes อื่นๆ
  transition('* <=> *', [
    query(':enter, :leave', [
      style({ position: 'absolute', top: 0, left: 0, width: '100%' })
    ], { optional: true }),
    query(':enter', [style({ opacity: 0 })], { optional: true }),
    group([
      query(':leave', [
        animate('200ms', style({ opacity: 0 }))
      ], { optional: true }),
      query(':enter', [
        animate('300ms 100ms', style({ opacity: 1 }))
      ], { optional: true })
    ])
  ])
]);
```

### 5.2 ตั้งค่า Route Animation

```typescript
// src/app/app.routes.ts
import { Routes } from '@angular/router';

export const routes: Routes = [
  {
    path: 'home',
    component: HomeComponent,
    data: { animation: 'HomePage' }  // ชื่อสำหรับ animation
  },
  {
    path: 'about',
    component: AboutComponent,
    data: { animation: 'AboutPage' }
  },
  {
    path: 'products',
    component: ProductsComponent,
    data: { animation: 'ProductsPage' }
  }
];
```

```typescript
// src/app/app.component.ts
import { Component } from '@angular/core';
import { RouterOutlet } from '@angular/router';
import { slideInAnimation } from './animations/route-animations';

@Component({
  selector: 'app-root',
  standalone: true,
  imports: [RouterOutlet],
  animations: [slideInAnimation],
  template: `
    <div class="page-container">
      <!-- ดึงข้อมูล animation data จาก route -->
      <div [@routeAnimations]="prepareRoute(outlet)" class="route-container">
        <router-outlet #outlet="outlet"></router-outlet>
      </div>
    </div>
  `,
  styles: [`
    .page-container { overflow: hidden; }
    .route-container { position: relative; }
  `]
})
export class AppComponent {
  prepareRoute(outlet: RouterOutlet): string {
    return outlet?.activatedRouteData?.['animation'] ?? '';
  }
}
```

---

## 6. Animation Callbacks

```typescript
@Component({
  selector: 'app-notification',
  animations: [
    trigger('notification', [
      transition(':enter', [
        style({ opacity: 0, transform: 'translateX(100%)' }),
        animate('300ms ease-out', style({ opacity: 1, transform: 'translateX(0)' }))
      ]),
      transition(':leave', [
        animate('200ms ease-in', style({ opacity: 0, transform: 'translateX(100%)' }))
      ])
    ])
  ],
  template: `
    <div
      [@notification]="state"
      (@notification.start)="onAnimationStart($event)"
      (@notification.done)="onAnimationDone($event)"
    >
      {{ message }}
    </div>
  `
})
export class NotificationComponent {
  state = 'visible';
  message = '';

  onAnimationStart(event: AnimationEvent): void {
    console.log('Animation เริ่มต้น:', event.triggerName, event.toState);
  }

  onAnimationDone(event: AnimationEvent): void {
    console.log('Animation จบ:', event.triggerName, event.toState);
    // ลบ notification หลัง animation จบ
    if (event.toState === 'void') {
      // cleanup
    }
  }
}
```

---

## 7. Workshop: Card Animations

```typescript
// src/app/components/animated-card/animated-card.component.ts
import { Component, Input, OnInit } from '@angular/core';
import { CommonModule } from '@angular/common';
import {
  trigger,
  state,
  style,
  transition,
  animate,
  keyframes,
  query,
  stagger
} from '@angular/animations';

interface CardItem {
  id: number;
  title: string;
  description: string;
  image: string;
  liked: boolean;
}

@Component({
  selector: 'app-animated-card',
  standalone: true,
  imports: [CommonModule],
  animations: [
    // Animation สำหรับ card เข้า/ออก
    trigger('cardEnterLeave', [
      transition(':enter', [
        style({
          opacity: 0,
          transform: 'scale(0.8) translateY(20px)'
        }),
        animate('400ms cubic-bezier(0.35, 0, 0.25, 1)', style({
          opacity: 1,
          transform: 'scale(1) translateY(0)'
        }))
      ]),
      transition(':leave', [
        animate('300ms ease-in', style({
          opacity: 0,
          transform: 'scale(0.8) translateY(-20px)'
        }))
      ])
    ]),

    // Animation สำหรับ list ของ cards
    trigger('cardListAnimation', [
      transition('* => *', [
        query(':enter', [
          style({ opacity: 0, transform: 'translateY(30px)' }),
          stagger(80, [
            animate('400ms ease-out', style({
              opacity: 1,
              transform: 'translateY(0)'
            }))
          ])
        ], { optional: true }),
        query(':leave', [
          stagger(50, [
            animate('250ms ease-in', style({
              opacity: 0,
              transform: 'scale(0.9)'
            }))
          ])
        ], { optional: true })
      ])
    ]),

    // Animation สำหรับ like button
    trigger('heartBeat', [
      state('unliked', style({ transform: 'scale(1)' })),
      state('liked', style({ transform: 'scale(1)', color: 'red' })),
      transition('unliked => liked', [
        animate('600ms', keyframes([
          style({ transform: 'scale(1)', offset: 0 }),
          style({ transform: 'scale(1.4)', offset: 0.3 }),
          style({ transform: 'scale(0.9)', offset: 0.5 }),
          style({ transform: 'scale(1.2)', offset: 0.7 }),
          style({ transform: 'scale(1)', color: 'red', offset: 1.0 })
        ]))
      ]),
      transition('liked => unliked', [
        animate('300ms ease-out', style({ color: 'gray', transform: 'scale(1)' }))
      ])
    ]),

    // Animation สำหรับ card hover
    trigger('cardHover', [
      state('normal', style({
        boxShadow: '0 2px 8px rgba(0,0,0,0.1)',
        transform: 'translateY(0)'
      })),
      state('hovered', style({
        boxShadow: '0 8px 24px rgba(0,0,0,0.2)',
        transform: 'translateY(-4px)'
      })),
      transition('normal <=> hovered', [
        animate('200ms ease-in-out')
      ])
    ]),

    // Animation สำหรับ skeleton loading
    trigger('skeleton', [
      state('loading', style({ opacity: 0.5 })),
      state('loaded', style({ opacity: 1 })),
      transition('loading => loaded', [
        animate('500ms ease-in')
      ])
    ])
  ],
  template: `
    <div class="cards-container">
      <!-- Controls -->
      <div class="controls mb-4">
        <button (click)="addCard()" class="btn btn-primary me-2">
          เพิ่มการ์ด
        </button>
        <button (click)="removeLastCard()" class="btn btn-danger me-2">
          ลบการ์ด
        </button>
        <button (click)="shuffleCards()" class="btn btn-secondary">
          สับเปลี่ยน
        </button>
      </div>

      <!-- Card Grid with List Animation -->
      <div class="card-grid" [@cardListAnimation]="cards.length">
        <div
          *ngFor="let card of cards; trackBy: trackById"
          [@cardEnterLeave]
          [@cardHover]="card['hovered'] ? 'hovered' : 'normal'"
          (mouseenter)="card['hovered'] = true"
          (mouseleave)="card['hovered'] = false"
          class="card"
        >
          <!-- Card Image -->
          <div class="card-img-container">
            <img [src]="card.image" [alt]="card.title" loading="lazy">
          </div>

          <!-- Card Body -->
          <div class="card-body">
            <h5 class="card-title">{{ card.title }}</h5>
            <p class="card-text">{{ card.description }}</p>

            <!-- Action Row -->
            <div class="card-actions">
              <!-- Like Button with heartbeat animation -->
              <button
                class="btn-like"
                [@heartBeat]="card.liked ? 'liked' : 'unliked'"
                (click)="toggleLike(card)"
              >
                ❤
              </button>

              <button
                class="btn btn-outline-primary btn-sm"
                (click)="removeCard(card.id)"
              >
                ลบ
              </button>
            </div>
          </div>
        </div>
      </div>

      <!-- Empty State -->
      <div *ngIf="cards.length === 0" [@cardEnterLeave] class="empty-state">
        <p>ไม่มีการ์ด กด "เพิ่มการ์ด" เพื่อเริ่มต้น</p>
      </div>
    </div>
  `,
  styles: [`
    .card-grid {
      display: grid;
      grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
      gap: 20px;
    }
    .card {
      border-radius: 8px;
      overflow: hidden;
      background: white;
      border: 1px solid #eee;
    }
    .card-img-container img {
      width: 100%;
      height: 180px;
      object-fit: cover;
    }
    .card-body { padding: 16px; }
    .card-actions {
      display: flex;
      justify-content: space-between;
      align-items: center;
      margin-top: 12px;
    }
    .btn-like {
      background: none;
      border: none;
      font-size: 24px;
      cursor: pointer;
      color: gray;
    }
    .empty-state {
      text-align: center;
      padding: 60px;
      color: #999;
    }
  `]
})
export class AnimatedCardComponent {

  cards: CardItem[] = [
    {
      id: 1,
      title: 'Angular Animation',
      description: 'เรียนรู้การสร้าง animation ใน Angular',
      image: 'https://picsum.photos/seed/1/400/300',
      liked: false
    },
    {
      id: 2,
      title: 'RxJS Basics',
      description: 'พื้นฐาน RxJS และ Observables',
      image: 'https://picsum.photos/seed/2/400/300',
      liked: true
    }
  ];

  private nextId = 3;

  addCard(): void {
    const id = this.nextId++;
    this.cards.push({
      id,
      title: `Card ${id}`,
      description: `เนื้อหาของการ์ดที่ ${id}`,
      image: `https://picsum.photos/seed/${id}/400/300`,
      liked: false
    });
  }

  removeCard(id: number): void {
    this.cards = this.cards.filter(card => card.id !== id);
  }

  removeLastCard(): void {
    if (this.cards.length > 0) {
      this.cards = this.cards.slice(0, -1);
    }
  }

  shuffleCards(): void {
    this.cards = [...this.cards].sort(() => Math.random() - 0.5);
  }

  toggleLike(card: CardItem): void {
    card.liked = !card.liked;
  }

  trackById(_: number, card: CardItem): number {
    return card.id;
  }
}
```

---

## สรุปบทที่ 20

| ฟังก์ชัน | หน้าที่ |
|---------|---------|
| `trigger` | จุดเริ่มต้นของ animation ทุกตัว |
| `state` | กำหนด style สำหรับแต่ละ state |
| `style` | กำหนด CSS styles |
| `transition` | กำหนดการ animate ระหว่าง states |
| `animate` | กำหนด timing และ easing |
| `keyframes` | กำหนด intermediate animation steps |
| `group` | รัน animations แบบขนาน |
| `sequence` | รัน animations แบบต่อเนื่อง |
| `stagger` | animate รายการแบบ cascade |
| `query` | animate child elements |

### Best Practices

1. **ใช้ `provideAnimations()` ใน providers** — แทน BrowserAnimationsModule
2. **ระวัง animations ที่ซับซ้อนเกิน** — อาจกระทบ performance
3. **ใช้ `optional: true` กับ query** — ป้องกัน error เมื่อไม่พบ element
4. **ใช้ trackBy กับ ngFor** — ช่วยให้ Angular identify items ได้ถูกต้อง
5. **ทดสอบบน mobile** — animations อาจช้ากว่าบน desktop
