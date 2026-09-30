# Part 67: CI/CD สำหรับ Angular

## บทนำ

CI/CD (Continuous Integration/Continuous Deployment) ช่วยให้การ deploy แอปพลิเคชัน Angular เป็นกระบวนการอัตโนมัติ ลดข้อผิดพลาด และเพิ่มความเร็ว

## 1. GitHub Actions

### Workflow พื้นฐาน

```yaml
# .github/workflows/ci.yml
name: CI Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

env:
  NODE_VERSION: '20'
  ANGULAR_VERSION: '17'

jobs:
  # Job 1: Code Quality
  quality:
    name: Code Quality Check
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run ESLint
        run: npm run lint

      - name: Check TypeScript
        run: npx tsc --noEmit

      - name: Check formatting (Prettier)
        run: npm run format:check

  # Job 2: Unit Tests
  unit-tests:
    name: Unit Tests
    runs-on: ubuntu-latest
    needs: quality
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - run: npm ci

      - name: Run unit tests with coverage
        run: npm run test:ci
        env:
          CI: true

      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v4
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
          files: ./coverage/lcov.info
          flags: unittests

      - name: Check coverage threshold
        run: |
          COVERAGE=$(cat coverage/coverage-summary.json | jq '.total.lines.pct')
          if (( $(echo "$COVERAGE < 80" | bc -l) )); then
            echo "Coverage $COVERAGE% is below 80% threshold"
            exit 1
          fi

  # Job 3: E2E Tests
  e2e-tests:
    name: E2E Tests
    runs-on: ubuntu-latest
    needs: quality
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - run: npm ci

      - name: Build app
        run: npm run build -- --configuration=test

      - name: Run Cypress E2E
        uses: cypress-io/github-action@v6
        with:
          start: npx serve -s dist/my-app -p 4200
          wait-on: 'http://localhost:4200'
          browser: chrome
          headless: true
        env:
          CYPRESS_BASE_URL: http://localhost:4200

      - uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: cypress-screenshots
          path: cypress/screenshots

  # Job 4: Build
  build:
    name: Build Application
    runs-on: ubuntu-latest
    needs: [unit-tests, e2e-tests]
    strategy:
      matrix:
        environment: [staging, production]
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - run: npm ci

      - name: Build for ${{ matrix.environment }}
        run: npm run build -- --configuration=${{ matrix.environment }}

      - name: Upload build artifacts
        uses: actions/upload-artifact@v4
        with:
          name: build-${{ matrix.environment }}
          path: dist/
          retention-days: 7
```

### Deployment Workflow

```yaml
# .github/workflows/deploy.yml
name: Deploy Pipeline

on:
  push:
    branches: [main]
  workflow_dispatch:
    inputs:
      environment:
        description: 'Deploy to'
        required: true
        default: 'staging'
        type: choice
        options: [staging, production]

jobs:
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    environment: staging
    if: github.ref == 'refs/heads/develop' || github.event.inputs.environment == 'staging'
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - run: npm ci

      - name: Build for staging
        run: npm run build -- --configuration=staging
        env:
          API_URL: ${{ vars.STAGING_API_URL }}

      - name: Deploy to AWS S3
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ap-southeast-1

      - run: |
          aws s3 sync dist/my-app/ s3://${{ secrets.STAGING_S3_BUCKET }} --delete
          aws cloudfront create-invalidation \
            --distribution-id ${{ secrets.STAGING_CF_DISTRIBUTION }} \
            --paths "/*"

      - name: Notify Slack
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "✅ Deployed to Staging: ${{ github.sha }}",
              "channel": "#deployments"
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}

  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    environment: production
    needs: deploy-staging
    if: github.ref == 'refs/heads/main' || github.event.inputs.environment == 'production'
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - run: npm ci

      - name: Build for production
        run: npm run build -- --configuration=production
        env:
          API_URL: ${{ vars.PRODUCTION_API_URL }}
          SENTRY_DSN: ${{ secrets.SENTRY_DSN }}

      - name: Configure AWS
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ap-southeast-1

      - name: Deploy to S3
        run: |
          aws s3 sync dist/my-app/ s3://${{ secrets.PROD_S3_BUCKET }} \
            --delete \
            --cache-control "max-age=31536000" \
            --exclude "index.html"
          
          # index.html ไม่ cache
          aws s3 cp dist/my-app/index.html s3://${{ secrets.PROD_S3_BUCKET }}/index.html \
            --cache-control "no-cache, no-store, must-revalidate"

      - name: Invalidate CloudFront
        run: |
          aws cloudfront create-invalidation \
            --distribution-id ${{ secrets.PROD_CF_DISTRIBUTION }} \
            --paths "/*"

      - name: Create GitHub Release
        uses: actions/create-release@v1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          tag_name: v${{ github.run_number }}
          release_name: Release ${{ github.run_number }}
          body: |
            ## Changes
            ${{ github.event.head_commit.message }}
```

## 2. GitLab CI

```yaml
# .gitlab-ci.yml
image: node:20-alpine

variables:
  npm_config_cache: "$CI_PROJECT_DIR/.npm"
  CYPRESS_CACHE_FOLDER: "$CI_PROJECT_DIR/cache/Cypress"

cache:
  key:
    files:
      - package-lock.json
  paths:
    - .npm
    - cache/Cypress
    - node_modules/

stages:
  - prepare
  - test
  - build
  - deploy

# Templates
.node_template: &node_template
  before_script:
    - npm ci --prefer-offline

install:
  stage: prepare
  <<: *node_template
  script:
    - echo "Dependencies installed"
  artifacts:
    paths:
      - node_modules/
    expire_in: 1 hour

lint:
  stage: test
  <<: *node_template
  script:
    - npm run lint
    - npm run format:check
  needs: [install]

unit-test:
  stage: test
  <<: *node_template
  script:
    - npm run test:ci
  coverage: '/Lines\s*:\s*(\d+\.?\d*)%/'
  artifacts:
    when: always
    reports:
      junit: junit.xml
      coverage_report:
        coverage_format: cobertura
        path: coverage/cobertura-coverage.xml
  needs: [install]

e2e-test:
  stage: test
  image: cypress/browsers:node20-chrome-latest
  <<: *node_template
  script:
    - npm run build:test
    - npx serve -s dist/my-app -p 4200 &
    - npx cypress run --browser chrome --headless
  artifacts:
    when: always
    paths:
      - cypress/videos/
      - cypress/screenshots/
    expire_in: 1 week
  needs: [install]

build:staging:
  stage: build
  <<: *node_template
  script:
    - npm run build -- --configuration=staging
  environment:
    name: staging
  artifacts:
    paths:
      - dist/
    expire_in: 1 day
  only:
    - develop
  needs: [lint, unit-test]

build:production:
  stage: build
  <<: *node_template
  script:
    - npm run build -- --configuration=production
  environment:
    name: production
  artifacts:
    paths:
      - dist/
    expire_in: 1 week
  only:
    - main
  needs: [lint, unit-test, e2e-test]

deploy:staging:
  stage: deploy
  image: amazon/aws-cli
  script:
    - aws s3 sync dist/my-app/ s3://$STAGING_S3_BUCKET --delete
    - aws cloudfront create-invalidation --distribution-id $STAGING_CF_ID --paths "/*"
  environment:
    name: staging
    url: https://staging.example.com
  only:
    - develop
  needs: [build:staging]

deploy:production:
  stage: deploy
  image: amazon/aws-cli
  script:
    - aws s3 sync dist/my-app/ s3://$PROD_S3_BUCKET --delete
    - aws cloudfront create-invalidation --distribution-id $PROD_CF_ID --paths "/*"
  environment:
    name: production
    url: https://example.com
  when: manual
  only:
    - main
  needs: [build:production]
```

## 3. Angular Scripts Configuration

```json
// package.json
{
  "scripts": {
    "start": "ng serve",
    "build": "ng build",
    "build:staging": "ng build --configuration=staging",
    "build:production": "ng build --configuration=production",
    "test": "ng test",
    "test:ci": "ng test --watch=false --browsers=ChromeHeadless --code-coverage",
    "lint": "ng lint",
    "e2e": "cypress open",
    "e2e:ci": "cypress run --headless",
    "format": "prettier --write \"src/**/*.{ts,html,scss}\"",
    "format:check": "prettier --check \"src/**/*.{ts,html,scss}\"",
    "analyze": "ng build --stats-json && npx webpack-bundle-analyzer dist/stats.json"
  }
}
```

## 4. Environment Configurations

```typescript
// src/environments/environment.ts
export const environment = {
  production: false,
  apiUrl: 'http://localhost:3000/api',
  wsUrl: 'ws://localhost:3000',
  sentryDsn: '',
  analyticsId: ''
};

// src/environments/environment.staging.ts
export const environment = {
  production: false,
  apiUrl: 'https://api-staging.example.com',
  wsUrl: 'wss://api-staging.example.com',
  sentryDsn: 'https://xxx@sentry.io/staging',
  analyticsId: 'UA-STAGING-1'
};

// src/environments/environment.prod.ts
export const environment = {
  production: true,
  apiUrl: 'https://api.example.com',
  wsUrl: 'wss://api.example.com',
  sentryDsn: 'https://xxx@sentry.io/production',
  analyticsId: 'UA-PROD-1'
};
```

## 5. Bundle Size Check

```yaml
# .github/workflows/bundle-check.yml
name: Bundle Size Check

on: [pull_request]

jobs:
  bundle-size:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm run build -- --stats-json

      - name: Check bundle size
        uses: houghtonap/bundlesize@v2
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          files: |
            [
              { "path": "dist/my-app/*.js", "maxSize": "500kb" },
              { "path": "dist/my-app/main.*.js", "maxSize": "200kb" },
              { "path": "dist/my-app/*.css", "maxSize": "50kb" }
            ]
```

## สรุป CI/CD Pipeline

```
Code Push
    ↓
Lint + Type Check
    ↓
Unit Tests + Coverage
    ↓
E2E Tests
    ↓
Build (Staging/Prod)
    ↓
Deploy Staging → Test → Deploy Production
```

| Stage | เครื่องมือ | เวลาเฉลี่ย |
|-------|---------|---------|
| Lint | ESLint + Prettier | 1-2 นาที |
| Unit Tests | Jest + Coverage | 3-5 นาที |
| E2E Tests | Cypress | 5-10 นาที |
| Build | Angular CLI | 3-5 นาที |
| Deploy | AWS/GCP/Azure | 2-3 นาที |

CI/CD ที่ดีช่วยให้ทีมสามารถ deploy ได้หลายครั้งต่อวันด้วยความมั่นใจสูง
