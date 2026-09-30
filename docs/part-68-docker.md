# Part 68: Docker สำหรับ Angular

## บทนำ

Docker ช่วยให้แอปพลิเคชัน Angular สามารถ deploy ได้ทุกที่อย่างสม่ำเสมอ บทนี้จะสอนการสร้าง Dockerfile, nginx configuration และ multi-stage build

## 1. Dockerfile พื้นฐาน

```dockerfile
# Dockerfile
# Stage 1: Build Angular Application
FROM node:20-alpine AS builder

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm ci --only=production=false

# Copy source code
COPY . .

# Build Angular app for production
ARG CONFIGURATION=production
RUN npm run build -- --configuration=$CONFIGURATION

# Stage 2: Serve with nginx
FROM nginx:alpine AS production

# Remove default nginx config
RUN rm /etc/nginx/conf.d/default.conf

# Copy custom nginx config
COPY nginx/nginx.conf /etc/nginx/nginx.conf
COPY nginx/default.conf /etc/nginx/conf.d/default.conf

# Copy built app from builder stage
COPY --from=builder /app/dist/my-angular-app /usr/share/nginx/html

# Add non-root user
RUN adduser -D -g '' nginxuser && \
    chown -R nginxuser:nginxuser /usr/share/nginx/html && \
    chown -R nginxuser:nginxuser /var/cache/nginx

# Expose port
EXPOSE 80

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost/health || exit 1

# Run as non-root
USER nginxuser

CMD ["nginx", "-g", "daemon off;"]
```

## 2. Multi-Stage Build แบบ Advanced

```dockerfile
# Dockerfile.advanced
# ========================================
# Stage 1: Dependencies
# ========================================
FROM node:20-alpine AS deps

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install ALL dependencies (including devDependencies for build)
RUN npm ci --frozen-lockfile

# ========================================
# Stage 2: Builder
# ========================================
FROM node:20-alpine AS builder

WORKDIR /app

# Copy dependencies from deps stage
COPY --from=deps /app/node_modules ./node_modules
COPY . .

# Build arguments
ARG APP_NAME=my-angular-app
ARG CONFIGURATION=production
ARG BUILD_VERSION=latest
ARG API_URL=https://api.example.com

# Set environment for build
ENV NODE_ENV=production

# Build the application
RUN npm run build -- \
    --configuration=$CONFIGURATION \
    --output-path=dist/$APP_NAME

# ========================================
# Stage 3: Production Runtime
# ========================================
FROM nginx:1.25-alpine AS production

# Install tools for security hardening
RUN apk add --no-cache tzdata && \
    cp /usr/share/zoneinfo/Asia/Bangkok /etc/localtime && \
    echo "Asia/Bangkok" > /etc/timezone

# Create non-root user
RUN addgroup -g 1001 -S angular && \
    adduser -u 1001 -S angular -G angular

# Remove default nginx files
RUN rm -rf /usr/share/nginx/html/* && \
    rm /etc/nginx/conf.d/default.conf

# Copy nginx configurations
COPY --chown=angular:angular nginx/nginx.conf /etc/nginx/nginx.conf
COPY --chown=angular:angular nginx/default.conf /etc/nginx/conf.d/default.conf
COPY --chown=angular:angular nginx/security-headers.conf /etc/nginx/snippets/security-headers.conf

# Copy Angular build artifacts
ARG APP_NAME=my-angular-app
COPY --from=builder --chown=angular:angular /app/dist/$APP_NAME /usr/share/nginx/html

# Fix permissions
RUN chmod -R 755 /usr/share/nginx/html && \
    chown -R angular:angular /var/cache/nginx && \
    chown -R angular:angular /var/log/nginx && \
    chown -R angular:angular /etc/nginx/conf.d && \
    touch /var/run/nginx.pid && \
    chown -R angular:angular /var/run/nginx.pid

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD wget -q --spider http://localhost:8080/health || exit 1

USER angular

CMD ["nginx", "-g", "daemon off;"]
```

## 3. Nginx Configuration

```nginx
# nginx/nginx.conf
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
  worker_connections 1024;
  multi_accept on;
  use epoll;
}

http {
  include /etc/nginx/mime.types;
  default_type application/octet-stream;

  # Logging
  log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                  '$status $body_bytes_sent "$http_referer" '
                  '"$http_user_agent" "$http_x_forwarded_for"';
  access_log /var/log/nginx/access.log main;

  # Performance
  sendfile on;
  tcp_nopush on;
  tcp_nodelay on;
  keepalive_timeout 65;

  # Compression
  gzip on;
  gzip_vary on;
  gzip_proxied any;
  gzip_comp_level 6;
  gzip_min_length 256;
  gzip_types
    application/atom+xml
    application/javascript
    application/json
    application/rss+xml
    application/vnd.ms-fontobject
    application/x-font-ttf
    application/x-web-app-manifest+json
    application/xhtml+xml
    application/xml
    font/opentype
    image/svg+xml
    image/x-icon
    text/css
    text/plain
    text/x-component;

  include /etc/nginx/conf.d/*.conf;
}
```

```nginx
# nginx/default.conf
server {
  listen 8080;
  listen [::]:8080;
  server_name _;
  root /usr/share/nginx/html;
  index index.html;

  # Security headers
  include /etc/nginx/snippets/security-headers.conf;

  # Health check endpoint
  location /health {
    access_log off;
    return 200 "healthy\n";
    add_header Content-Type text/plain;
  }

  # Cache static assets aggressively
  location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
    expires 1y;
    add_header Cache-Control "public, immutable";
    access_log off;
  }

  # index.html - never cache
  location = /index.html {
    add_header Cache-Control "no-cache, no-store, must-revalidate";
    add_header Pragma "no-cache";
    add_header Expires "0";
  }

  # Angular routing: ส่งทุก path ไปที่ index.html
  location / {
    try_files $uri $uri/ /index.html;
  }

  # API Proxy (optional)
  location /api/ {
    proxy_pass http://backend:3000/;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection 'upgrade';
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_cache_bypass $http_upgrade;
    proxy_read_timeout 300s;
    proxy_connect_timeout 75s;
  }

  # Deny access to hidden files
  location ~ /\. {
    deny all;
    access_log off;
    log_not_found off;
  }
}
```

```nginx
# nginx/security-headers.conf
add_header X-Frame-Options "SAMEORIGIN" always;
add_header X-Content-Type-Options "nosniff" always;
add_header X-XSS-Protection "1; mode=block" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
add_header Permissions-Policy "camera=(), microphone=(), geolocation=()" always;
add_header Content-Security-Policy "
  default-src 'self';
  script-src 'self' 'unsafe-inline' 'unsafe-eval' https://cdn.example.com;
  style-src 'self' 'unsafe-inline' https://fonts.googleapis.com;
  font-src 'self' https://fonts.gstatic.com;
  img-src 'self' data: https:;
  connect-src 'self' https://api.example.com wss://api.example.com;
  frame-ancestors 'none';
" always;
```

## 4. Docker Compose

```yaml
# docker-compose.yml
version: '3.9'

services:
  # Angular Frontend
  frontend:
    build:
      context: .
      dockerfile: Dockerfile
      args:
        CONFIGURATION: production
        API_URL: ${API_URL:-http://localhost:3000}
    image: my-angular-app:${VERSION:-latest}
    ports:
      - "80:8080"
    environment:
      - TZ=Asia/Bangkok
    depends_on:
      backend:
        condition: service_healthy
    restart: unless-stopped
    networks:
      - app-network
    healthcheck:
      test: ["CMD", "wget", "-q", "--spider", "http://localhost:8080/health"]
      interval: 30s
      timeout: 5s
      retries: 3

  # Backend API
  backend:
    image: node:20-alpine
    working_dir: /app
    command: node server.js
    environment:
      - NODE_ENV=production
      - PORT=3000
      - DATABASE_URL=${DATABASE_URL}
      - JWT_SECRET=${JWT_SECRET}
    ports:
      - "3000:3000"
    networks:
      - app-network
    healthcheck:
      test: ["CMD", "wget", "-q", "--spider", "http://localhost:3000/health"]
      interval: 30s
      timeout: 5s
      retries: 3

  # Development override
  # docker-compose -f docker-compose.yml -f docker-compose.dev.yml up

networks:
  app-network:
    driver: bridge

volumes:
  nginx-logs:
```

```yaml
# docker-compose.dev.yml
version: '3.9'

services:
  frontend:
    build:
      target: builder
    command: npm start
    ports:
      - "4200:4200"
    volumes:
      - .:/app
      - /app/node_modules
    environment:
      - NODE_ENV=development
```

## 5. Scripts สำหรับ Docker

```bash
#!/bin/bash
# scripts/docker-build.sh

set -e

APP_NAME="my-angular-app"
VERSION=$(git describe --tags --always --dirty 2>/dev/null || echo "latest")
REGISTRY="gcr.io/my-project"

echo "Building $APP_NAME:$VERSION..."

# Build image
docker build \
  --tag "$APP_NAME:$VERSION" \
  --tag "$APP_NAME:latest" \
  --build-arg CONFIGURATION=production \
  --build-arg BUILD_VERSION="$VERSION" \
  --file Dockerfile \
  .

echo "Build complete: $APP_NAME:$VERSION"

# Tag for registry
if [ "${PUSH:-false}" = "true" ]; then
  docker tag "$APP_NAME:$VERSION" "$REGISTRY/$APP_NAME:$VERSION"
  docker tag "$APP_NAME:latest" "$REGISTRY/$APP_NAME:latest"
  docker push "$REGISTRY/$APP_NAME:$VERSION"
  docker push "$REGISTRY/$APP_NAME:latest"
  echo "Pushed to registry"
fi
```

## 6. .dockerignore

```
# .dockerignore
node_modules
dist
.git
.gitignore
*.md
*.log
.env*
.angular
coverage
cypress/videos
cypress/screenshots
.storybook
*.stories.ts
*.spec.ts
jest.config.*
karma.conf.js
```

## 7. Runtime Environment Variables

```typescript
// src/app/app.initializer.ts
import { APP_INITIALIZER } from '@angular/core';

export interface RuntimeConfig {
  apiUrl: string;
  wsUrl: string;
  featureFlags: Record<string, boolean>;
}

export function loadRuntimeConfig(): () => Promise<RuntimeConfig> {
  return () => fetch('/assets/config/runtime-config.json')
    .then(res => res.json())
    .then(config => {
      (window as any).__APP_CONFIG = config;
      return config;
    })
    .catch(() => ({
      apiUrl: '/api',
      wsUrl: 'ws://localhost:3000',
      featureFlags: {}
    }));
}

export const runtimeConfigProvider = {
  provide: APP_INITIALIZER,
  useFactory: loadRuntimeConfig,
  multi: true
};
```

```typescript
// src/app/config.service.ts
import { Injectable } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class ConfigService {
  get apiUrl(): string {
    return (window as any).__APP_CONFIG?.apiUrl || '/api';
  }

  get featureFlags(): Record<string, boolean> {
    return (window as any).__APP_CONFIG?.featureFlags || {};
  }

  isFeatureEnabled(flag: string): boolean {
    return this.featureFlags[flag] ?? false;
  }
}
```

```nginx
# nginx script สำหรับ inject runtime config
# nginx/entrypoint.sh
#!/bin/sh

# สร้าง runtime config จาก environment variables
cat > /usr/share/nginx/html/assets/config/runtime-config.json << EOF
{
  "apiUrl": "${API_URL:-/api}",
  "wsUrl": "${WS_URL:-ws://localhost:3000}",
  "featureFlags": {
    "newUI": ${FEATURE_NEW_UI:-false},
    "darkMode": ${FEATURE_DARK_MODE:-true}
  }
}
EOF

echo "Runtime config generated"
exec nginx -g "daemon off;"
```

## สรุป Docker Best Practices

| หัวข้อ | แนวทาง |
|-------|-------|
| Image Size | ใช้ multi-stage build, alpine base |
| Security | Non-root user, read-only filesystem |
| Caching | แยก dependencies layer |
| Nginx | Gzip, cache headers, security headers |
| Config | Runtime config ผ่าน environment variables |
| Health | HEALTHCHECK directive |

Docker ช่วยให้ deploy Angular app ได้อย่างสม่ำเสมอในทุก environment
