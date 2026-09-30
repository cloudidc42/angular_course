# Part 66: GraphQL ใน Angular ด้วย Apollo Client

## บทนำ

GraphQL เป็น query language สำหรับ API ที่ช่วยให้ client ขอข้อมูลเฉพาะที่ต้องการได้ Apollo Client เป็น library ยอดนิยมสำหรับใช้ GraphQL ใน Angular

## 1. การติดตั้ง

```bash
npm install apollo-angular @apollo/client graphql
```

## 2. ตั้งค่า Apollo Client

```typescript
// src/app/graphql.module.ts
import { NgModule } from '@angular/core';
import { ApolloModule, APOLLO_OPTIONS } from 'apollo-angular';
import { ApolloClientOptions, InMemoryCache, ApolloLink, split } from '@apollo/client/core';
import { HttpLink } from 'apollo-angular/http';
import { GraphQLWsLink } from '@apollo/client/link/subscriptions';
import { createClient } from 'graphql-ws';
import { getMainDefinition } from '@apollo/client/utilities';

export function createApollo(httpLink: HttpLink): ApolloClientOptions<any> {
  // HTTP Link สำหรับ queries และ mutations
  const http = httpLink.create({
    uri: 'http://localhost:4000/graphql'
  });

  // WebSocket Link สำหรับ subscriptions
  const ws = new GraphQLWsLink(
    createClient({ url: 'ws://localhost:4000/graphql' })
  );

  // Split: ใช้ ws สำหรับ subscription, http สำหรับอื่น
  const link = split(
    ({ query }) => {
      const def = getMainDefinition(query);
      return def.kind === 'OperationDefinition' && def.operation === 'subscription';
    },
    ws,
    http
  );

  // Auth Middleware
  const authLink = new ApolloLink((operation, forward) => {
    const token = localStorage.getItem('auth_token');
    operation.setContext({
      headers: {
        authorization: token ? `Bearer ${JSON.parse(token).accessToken}` : ''
      }
    });
    return forward(operation);
  });

  return {
    link: authLink.concat(link),
    cache: new InMemoryCache({
      typePolicies: {
        Query: {
          fields: {
            products: {
              // Pagination merge policy
              keyArgs: ['category', 'search'],
              merge(existing = { items: [] }, incoming) {
                return {
                  ...incoming,
                  items: [...(existing.items || []), ...(incoming.items || [])]
                };
              }
            }
          }
        }
      }
    }),
    defaultOptions: {
      watchQuery: {
        fetchPolicy: 'cache-and-network',
        errorPolicy: 'all'
      }
    }
  };
}

@NgModule({
  imports: [ApolloModule],
  providers: [
    {
      provide: APOLLO_OPTIONS,
      useFactory: createApollo,
      deps: [HttpLink]
    }
  ]
})
export class GraphQLModule {}
```

## 3. GraphQL Operations

```typescript
// graphql/queries/products.queries.ts
import { gql } from 'apollo-angular';

export const GET_PRODUCTS = gql`
  query GetProducts($page: Int!, $limit: Int!, $search: String, $category: String) {
    products(page: $page, limit: $limit, search: $search, category: $category) {
      items {
        id
        name
        description
        price
        stock
        category {
          id
          name
        }
        images
        rating
        reviewCount
      }
      total
      hasNextPage
    }
  }
`;

export const GET_PRODUCT = gql`
  query GetProduct($id: ID!) {
    product(id: $id) {
      id
      name
      description
      price
      stock
      category {
        id
        name
        slug
      }
      images
      rating
      reviewCount
      reviews(limit: 10) {
        id
        rating
        comment
        user {
          id
          name
          avatar
        }
        createdAt
      }
    }
  }
`;

// graphql/mutations/products.mutations.ts
export const CREATE_PRODUCT = gql`
  mutation CreateProduct($input: CreateProductInput!) {
    createProduct(input: $input) {
      id
      name
      price
      stock
      createdAt
    }
  }
`;

export const UPDATE_PRODUCT = gql`
  mutation UpdateProduct($id: ID!, $input: UpdateProductInput!) {
    updateProduct(id: $id, input: $input) {
      id
      name
      price
      stock
      updatedAt
    }
  }
`;

export const DELETE_PRODUCT = gql`
  mutation DeleteProduct($id: ID!) {
    deleteProduct(id: $id) {
      success
      message
    }
  }
`;

// graphql/subscriptions/products.subscriptions.ts
export const PRODUCT_UPDATED = gql`
  subscription OnProductUpdated($categoryId: String) {
    productUpdated(categoryId: $categoryId) {
      id
      name
      price
      stock
    }
  }
`;

export const ORDER_STATUS_CHANGED = gql`
  subscription OnOrderStatusChanged($userId: ID!) {
    orderStatusChanged(userId: $userId) {
      orderId
      status
      updatedAt
    }
  }
`;
```

## 4. Products Service ด้วย Apollo

```typescript
// services/products-gql.service.ts
import { Injectable } from '@angular/core';
import { Apollo } from 'apollo-angular';
import { Observable } from 'rxjs';
import { map, tap } from 'rxjs/operators';
import {
  GET_PRODUCTS,
  GET_PRODUCT,
  CREATE_PRODUCT,
  UPDATE_PRODUCT,
  DELETE_PRODUCT
} from '../graphql/queries/products.queries';

export interface Product {
  id: string;
  name: string;
  description: string;
  price: number;
  stock: number;
  category: { id: string; name: string };
  images: string[];
  rating: number;
  reviewCount: number;
}

export interface ProductsResponse {
  items: Product[];
  total: number;
  hasNextPage: boolean;
}

export interface CreateProductInput {
  name: string;
  description: string;
  price: number;
  stock: number;
  categoryId: string;
  images: string[];
}

@Injectable({ providedIn: 'root' })
export class ProductsGqlService {
  constructor(private apollo: Apollo) {}

  getProducts(options: {
    page: number;
    limit: number;
    search?: string;
    category?: string;
  }): Observable<ProductsResponse> {
    return this.apollo.watchQuery<{ products: ProductsResponse }>({
      query: GET_PRODUCTS,
      variables: options
    }).valueChanges.pipe(
      map(result => result.data.products)
    );
  }

  getProduct(id: string): Observable<Product> {
    return this.apollo.watchQuery<{ product: Product }>({
      query: GET_PRODUCT,
      variables: { id }
    }).valueChanges.pipe(
      map(result => result.data.product)
    );
  }

  createProduct(input: CreateProductInput): Observable<Product> {
    return this.apollo.mutate<{ createProduct: Product }>({
      mutation: CREATE_PRODUCT,
      variables: { input },
      // อัปเดต cache หลังจาก create
      update(cache, { data }) {
        const newProduct = data?.createProduct;
        if (!newProduct) return;

        cache.modify({
          fields: {
            products(existingProducts = { items: [], total: 0 }) {
              return {
                ...existingProducts,
                items: [newProduct, ...existingProducts.items],
                total: existingProducts.total + 1
              };
            }
          }
        });
      }
    }).pipe(
      map(result => result.data!.createProduct)
    );
  }

  updateProduct(id: string, input: Partial<CreateProductInput>): Observable<Product> {
    return this.apollo.mutate<{ updateProduct: Product }>({
      mutation: UPDATE_PRODUCT,
      variables: { id, input }
    }).pipe(
      map(result => result.data!.updateProduct)
    );
  }

  deleteProduct(id: string): Observable<{ success: boolean }> {
    return this.apollo.mutate<{ deleteProduct: { success: boolean } }>({
      mutation: DELETE_PRODUCT,
      variables: { id },
      // ลบออกจาก cache
      update(cache) {
        cache.modify({
          fields: {
            products(existingProducts = { items: [] }) {
              return {
                ...existingProducts,
                items: existingProducts.items.filter(
                  (p: { __ref: string }) => !p.__ref?.includes(id)
                )
              };
            }
          }
        });
      }
    }).pipe(
      map(result => result.data!.deleteProduct)
    );
  }
}
```

## 5. Component ใช้ Apollo

```typescript
// products/products-list.component.ts
import { Component, OnInit, OnDestroy } from '@angular/core';
import { CommonModule } from '@angular/common';
import { FormsModule } from '@angular/forms';
import { Apollo, QueryRef } from 'apollo-angular';
import { Subject } from 'rxjs';
import { takeUntil, debounceTime, distinctUntilChanged } from 'rxjs/operators';
import { ProductsGqlService, Product, ProductsResponse } from '../services/products-gql.service';
import { PRODUCT_UPDATED } from '../graphql/subscriptions/products.subscriptions';

@Component({
  selector: 'app-products-list',
  standalone: true,
  imports: [CommonModule, FormsModule],
  template: `
    <div class="products-page">
      <input
        [(ngModel)]="searchQuery"
        (ngModelChange)="onSearch($event)"
        placeholder="ค้นหาสินค้า..."
        class="search-input"
      />

      <div *ngIf="loading" class="loading">กำลังโหลด...</div>
      <div *ngIf="error" class="error">{{ error }}</div>

      <div class="products-grid" *ngIf="!loading">
        <div *ngFor="let product of products" class="product-card">
          <img [src]="product.images[0]" [alt]="product.name" />
          <h3>{{ product.name }}</h3>
          <p class="price">฿{{ product.price | number }}</p>
          <p class="rating">⭐ {{ product.rating }} ({{ product.reviewCount }})</p>
          <p class="stock" [class.low]="product.stock < 10">
            คงเหลือ: {{ product.stock }}
          </p>
          <button (click)="deleteProduct(product.id)">ลบ</button>
        </div>
      </div>

      <div *ngIf="hasNextPage" class="load-more">
        <button (click)="loadMore()" [disabled]="loading">โหลดเพิ่มเติม</button>
      </div>
    </div>
  `
})
export class ProductsListComponent implements OnInit, OnDestroy {
  products: Product[] = [];
  loading = false;
  error: string | null = null;
  hasNextPage = false;
  searchQuery = '';
  currentPage = 1;

  private destroy$ = new Subject<void>();
  private searchSubject = new Subject<string>();

  constructor(
    private productsService: ProductsGqlService,
    private apollo: Apollo
  ) {}

  ngOnInit(): void {
    this.loadProducts();
    this.setupSearch();
    this.subscribeToUpdates();
  }

  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
  }

  private loadProducts(page = 1): void {
    this.loading = true;
    this.productsService.getProducts({
      page,
      limit: 12,
      search: this.searchQuery || undefined
    }).pipe(
      takeUntil(this.destroy$)
    ).subscribe({
      next: (response) => {
        if (page === 1) {
          this.products = response.items;
        } else {
          this.products = [...this.products, ...response.items];
        }
        this.hasNextPage = response.hasNextPage;
        this.currentPage = page;
        this.loading = false;
      },
      error: (err) => {
        this.error = err.message;
        this.loading = false;
      }
    });
  }

  private setupSearch(): void {
    this.searchSubject.pipe(
      debounceTime(300),
      distinctUntilChanged(),
      takeUntil(this.destroy$)
    ).subscribe(() => {
      this.loadProducts(1);
    });
  }

  private subscribeToUpdates(): void {
    this.apollo.subscribe<{ productUpdated: Product }>({
      query: PRODUCT_UPDATED
    }).pipe(
      takeUntil(this.destroy$)
    ).subscribe({
      next: (result) => {
        const updated = result.data?.productUpdated;
        if (updated) {
          const index = this.products.findIndex(p => p.id === updated.id);
          if (index >= 0) {
            this.products = [
              ...this.products.slice(0, index),
              updated,
              ...this.products.slice(index + 1)
            ];
          }
        }
      }
    });
  }

  onSearch(query: string): void {
    this.searchSubject.next(query);
  }

  loadMore(): void {
    this.loadProducts(this.currentPage + 1);
  }

  deleteProduct(id: string): void {
    if (!confirm('ต้องการลบสินค้านี้ใช่หรือไม่?')) return;
    
    this.productsService.deleteProduct(id).subscribe({
      next: () => {
        this.products = this.products.filter(p => p.id !== id);
      },
      error: (err) => {
        this.error = `ลบไม่สำเร็จ: ${err.message}`;
      }
    });
  }
}
```

## 6. Fragments

```typescript
// graphql/fragments/product.fragment.ts
import { gql } from 'apollo-angular';

export const PRODUCT_FIELDS = gql`
  fragment ProductFields on Product {
    id
    name
    price
    stock
    images
    rating
  }
`;

// ใช้งานใน queries
export const GET_FEATURED_PRODUCTS = gql`
  ${PRODUCT_FIELDS}
  query GetFeaturedProducts {
    featuredProducts {
      ...ProductFields
      isFeatured
      discountPercent
    }
  }
`;
```

## 7. Error Handling

```typescript
// services/apollo-error-handler.service.ts
import { Injectable } from '@angular/core';
import { ApolloError } from '@apollo/client/core';
import { GraphQLError } from 'graphql';

@Injectable({ providedIn: 'root' })
export class ApolloErrorHandlerService {
  handle(error: ApolloError): string {
    // Network error
    if (error.networkError) {
      return 'ไม่สามารถเชื่อมต่อกับเซิร์ฟเวอร์ได้';
    }

    // GraphQL errors
    if (error.graphQLErrors?.length) {
      return error.graphQLErrors
        .map(e => this.formatGraphQLError(e))
        .join(', ');
    }

    return error.message;
  }

  private formatGraphQLError(error: GraphQLError): string {
    const code = error.extensions?.['code'] as string;
    
    switch (code) {
      case 'UNAUTHENTICATED':
        return 'กรุณาเข้าสู่ระบบ';
      case 'FORBIDDEN':
        return 'ไม่มีสิทธิ์เข้าถึง';
      case 'NOT_FOUND':
        return 'ไม่พบข้อมูลที่ต้องการ';
      case 'BAD_USER_INPUT':
        return error.message;
      default:
        return error.message || 'เกิดข้อผิดพลาด';
    }
  }
}
```

## สรุป

| Feature | REST API | GraphQL |
|---------|---------|---------|
| Over-fetching | มีปัญหา | ไม่มีปัญหา |
| Under-fetching | ต้อง multiple requests | Query เดียวพอ |
| Schema | ไม่ชัดเจน | Strongly typed |
| Real-time | ต้องใช้ Polling/WS แยก | Subscriptions built-in |
| Caching | Browser cache | Apollo InMemoryCache |

GraphQL เหมาะสำหรับแอปที่ต้องการ flexibility ในการ fetch data และมีหลาย client types
