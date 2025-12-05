---
title: "GitHub Actions ile CI/CD Pipeline: Otomatik Test ve Deployment"
description: "GitHub Actions ile CI/CD pipeline oluşturma rehberi. Otomatik test, Docker build, multi-environment deployment ve güvenlik best practices."
date: 2025-03-22 10:00:00 +0300
categories: [DevOps, CI/CD]
tags: [github-actions, ci-cd, automation, testing, deployment, docker, workflows, yaml, devops, continuous-integration]
image:
  path: /assets/img/posts/github-actions-cicd-pipeline-diagram.png
  alt: "GitHub Actions CI/CD Pipeline Architecture"
---

## Giriş

GitHub Actions, GitHub'ın yerleşik CI/CD (Continuous Integration/Continuous Deployment) platformudur. Kod değişikliklerini otomatik olarak test etme, build alma ve production'a deploy etme süreçlerini tamamen otomatikleştirir.

Bu rehberde, GitHub Actions ile sıfırdan production-ready bir CI/CD pipeline'ı nasıl oluşturacağınızı öğreneceksiniz. Test otomasyonu, Docker build, multi-environment deployment ve best practice'leri detaylıca ele alacağız.

## GitHub Actions Nedir?

GitHub Actions, event-driven bir otomasyon platformudur. Repository'nizdeki belirli olaylar (push, pull request, release) tetiklendiğinde önceden tanımlanmış iş akışlarını (workflows) çalıştırır.

### Temel Kavramlar

- **Workflow**: Otomatik süreç tanımı (YAML dosyası)
- **Job**: Workflow içindeki bağımsız görev grupları
- **Step**: Job içindeki tek bir komut veya action
- **Action**: Yeniden kullanılabilir otomasyon bileşeni
- **Runner**: Workflow'ları çalıştıran sanal makine
- **Event**: Workflow'ı tetikleyen olay (push, pull_request, schedule, vb.)

### Workflow Dosya Yapısı

```yaml
# .github/workflows/ci.yml
name: CI Pipeline

# Tetikleyiciler
on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

# İş tanımları
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run tests
        run: npm test

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - name: Build application
        run: npm run build
```

## İlk Workflow Oluşturma

### Basit CI Workflow

```yaml
# .github/workflows/hello-world.yml
name: Hello World

on:
  push:
    branches: [ main ]
  workflow_dispatch:  # Manuel tetikleme

jobs:
  greet:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Say hello
        run: echo "Hello, GitHub Actions!"
      
      - name: Show environment
        run: |
          echo "Repository: ${{ github.repository }}"
          echo "Branch: ${{ github.ref }}"
          echo "Actor: ${{ github.actor }}"
```
{: file=".github/workflows/hello-world.yml" }

> `workflow_dispatch` event'i ile GitHub UI'dan manuel olarak workflow tetikleyebilirsiniz.
{: .prompt-tip }

### Python Projesi için CI

```yaml
# .github/workflows/python-ci.yml
name: Python CI

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ['3.9', '3.10', '3.11', '3.12']
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Set up Python ${{ matrix.python-version }}
        uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
      
      - name: Cache pip packages
        uses: actions/cache@v4
        with:
          path: ~/.cache/pip
          key: ${{ runner.os }}-pip-${{ hashFiles('**/requirements.txt') }}
          restore-keys: |
            ${{ runner.os }}-pip-
      
      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt
          pip install pytest pytest-cov
      
      - name: Run tests
        run: |
          pytest --cov=. --cov-report=xml
      
      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          file: ./coverage.xml
          fail_ci_if_error: true
```
{: file=".github/workflows/python-ci.yml" }

![CI/CD Deployment Automation Flow](/assets/img/posts/cicd-deployment-automation-flow.png){: w="700" h="400" .shadow }
_CI/CD deployment otomasyonu akış diyagramı_

## Node.js/JavaScript CI/CD

### Complete Node.js Workflow

```yaml
# .github/workflows/node-ci-cd.yml
name: Node.js CI/CD

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

env:
  NODE_VERSION: '20.x'

jobs:
  # Lint ve format kontrolü
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run ESLint
        run: npm run lint
      
      - name: Check formatting
        run: npm run format:check
  
  # Test
  test:
    runs-on: ubuntu-latest
    needs: lint
    
    strategy:
      matrix:
        node-version: [18.x, 20.x, 22.x]
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run tests
        run: npm test -- --coverage
      
      - name: Upload coverage
        if: matrix.node-version == '20.x'
        uses: codecov/codecov-action@v4
        with:
          files: ./coverage/coverage-final.json
  
  # Build
  build:
    runs-on: ubuntu-latest
    needs: test
    if: github.ref == 'refs/heads/main'
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Build application
        run: npm run build
      
      - name: Upload build artifact
        uses: actions/upload-artifact@v4
        with:
          name: dist
          path: dist/
          retention-days: 7
  
  # Deploy
  deploy:
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/main'
    
    steps:
      - name: Download build artifact
        uses: actions/download-artifact@v4
        with:
          name: dist
          path: dist/
      
      - name: Deploy to production
        run: |
          echo "Deploying to production..."
          # Deploy komutları buraya
```

## Docker Build ve Push

### Docker Image Build

```yaml
# .github/workflows/docker-build.yml
name: Docker Build & Push

on:
  push:
    branches: [ main ]
    tags:
      - 'v*'
  pull_request:
    branches: [ main ]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Set up QEMU
        uses: docker/setup-qemu-action@v3
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=semver,pattern={% raw %}{{version}}{% endraw %}
            type=semver,pattern={% raw %}{{major}}.{{minor}}{% endraw %}
            type=sha,prefix={% raw %}{{branch}}{% endraw %}-
      
      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          platforms: linux/amd64,linux/arm64
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

### Multi-Stage Docker Build

```dockerfile
# Dockerfile
# Build stage
FROM node:20-alpine AS builder

WORKDIR /app

COPY package*.json ./
RUN npm ci --only=production

COPY . .
RUN npm run build

# Production stage
FROM node:20-alpine

WORKDIR /app

COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY package*.json ./

USER node

EXPOSE 3000

CMD ["node", "dist/index.js"]
```

![GitHub Actions Workflow YAML Configuration](/assets/img/posts/github-actions-workflow-yaml.png){: w="700" h="400" .shadow }
_GitHub Actions workflow YAML konfigürasyon örneği_

## Matrix Testing

### Cross-Platform Testing

```yaml
# .github/workflows/matrix-test.yml
name: Matrix Testing

on: [push, pull_request]

jobs:
  test:
    runs-on: ${{ matrix.os }}
    
    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        python-version: ['3.9', '3.10', '3.11', '3.12']
        exclude:
          # Windows'da Python 3.9 test etme
          - os: windows-latest
            python-version: '3.9'
        include:
          # Özel konfigürasyon ekle
          - os: ubuntu-latest
            python-version: '3.12'
            experimental: true
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python ${{ matrix.python-version }}
        uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
      
      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt
      
      - name: Run tests
        run: pytest
        continue-on-error: ${{ matrix.experimental || false }}
```

### Database Matrix

```yaml
# .github/workflows/db-matrix.yml
name: Database Tests

on: [push, pull_request]

jobs:
  test-db:
    runs-on: ubuntu-latest
    
    strategy:
      matrix:
        database:
          - postgres:14
          - postgres:15
          - postgres:16
          - mysql:8.0
          - mysql:8.2
    
    services:
      db:
        image: ${{ matrix.database }}
        env:
          POSTGRES_PASSWORD: postgres
          MYSQL_ROOT_PASSWORD: mysql
        options: >-
          --health-cmd "pg_isready || mysqladmin ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
          - 3306:3306
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Run database tests
        run: |
          echo "Testing with ${{ matrix.database }}"
          # Test komutları
```
{: file=".github/workflows/matrix-testing.yml" }

## Secrets Yönetimi

> API key'leri, şifreleri ve token'ları asla GitHub repository'nizde saklamayın! GitHub Secrets kullanarak güvenli şekilde yönetin.
{: .prompt-danger }

### Repository Secrets

```yaml
# .github/workflows/deploy-with-secrets.yml
name: Deploy with Secrets

on:
  push:
    branches: [ main ]

jobs:
  deploy:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1
      
      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
      
      - name: Deploy to server
        env:
          SSH_PRIVATE_KEY: ${{ secrets.SSH_PRIVATE_KEY }}
          SERVER_HOST: ${{ secrets.SERVER_HOST }}
        run: |
          mkdir -p ~/.ssh
          echo "$SSH_PRIVATE_KEY" > ~/.ssh/id_rsa
          chmod 600 ~/.ssh/id_rsa
          ssh -o StrictHostKeyChecking=no user@$SERVER_HOST 'bash -s' < deploy.sh
```

### Environment Secrets

```yaml
# .github/workflows/multi-env-deploy.yml
name: Multi-Environment Deploy

on:
  push:
    branches: [ main, staging, develop ]

jobs:
  deploy-staging:
    if: github.ref == 'refs/heads/staging'
    runs-on: ubuntu-latest
    environment:
      name: staging
      url: https://staging.example.com
    
    steps:
      - name: Deploy to staging
        env:
          API_KEY: ${{ secrets.STAGING_API_KEY }}
          DATABASE_URL: ${{ secrets.STAGING_DATABASE_URL }}
        run: |
          echo "Deploying to staging..."
  
  deploy-production:
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://example.com
    
    steps:
      - name: Deploy to production
        env:
          API_KEY: ${{ secrets.PRODUCTION_API_KEY }}
          DATABASE_URL: ${{ secrets.PRODUCTION_DATABASE_URL }}
        run: |
          echo "Deploying to production..."
```

![CI/CD Pipeline Stages Diagram](/assets/img/posts/cicd-pipeline-stages-diagram.png){: w="700" h="400" .shadow }
_CI/CD pipeline aşamaları detaylı diyagram_

## Caching Stratejileri

### Dependency Caching

```yaml
# .github/workflows/caching.yml
name: Caching Demo

on: [push, pull_request]

jobs:
  python-cache:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-python@v5
        with:
          python-version: '3.11'
          cache: 'pip'  # Otomatik pip cache
      
      - name: Install dependencies
        run: pip install -r requirements.txt
  
  node-cache:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'  # Otomatik npm cache
      
      - run: npm ci
  
  custom-cache:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Cache build outputs
        uses: actions/cache@v4
        with:
          path: |
            ~/.cache
            build/
            dist/
          key: ${{ runner.os }}-build-${{ hashFiles('**/package-lock.json') }}
          restore-keys: |
            ${{ runner.os }}-build-
            ${{ runner.os }}-
      
      - name: Build project
        run: npm run build
```

### Docker Layer Caching

```yaml
# .github/workflows/docker-cache.yml
name: Docker Layer Cache

on: push

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Build with cache
        uses: docker/build-push-action@v5
        with:
          context: .
          push: false
          cache-from: type=gha
          cache-to: type=gha,mode=max
          tags: myapp:latest
```

## Artifacts (Yapılar)

### Build Artifacts

```yaml
# .github/workflows/artifacts.yml
name: Build Artifacts

on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Build application
        run: |
          mkdir -p dist
          echo "Build output" > dist/app.js
          echo "Build artifacts" > dist/assets.tar.gz
      
      - name: Upload build artifacts
        uses: actions/upload-artifact@v4
        with:
          name: build-artifacts
          path: |
            dist/**
            !dist/**/*.map
          retention-days: 30
          if-no-files-found: error
  
  test:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Download build artifacts
        uses: actions/download-artifact@v4
        with:
          name: build-artifacts
          path: dist/
      
      - name: Run tests
        run: |
          ls -la dist/
          echo "Testing build artifacts..."
  
  deploy:
    needs: [build, test]
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - name: Download artifacts
        uses: actions/download-artifact@v4
        with:
          name: build-artifacts
      
      - name: Deploy
        run: echo "Deploying artifacts..."
```

## Conditional Execution

### Path Filters

```yaml
# .github/workflows/conditional.yml
name: Conditional Execution

on:
  push:
    paths:
      - 'src/**'
      - 'tests/**'
      - 'package.json'

jobs:
  # Sadece frontend değişirse
  frontend:
    if: contains(github.event.head_commit.message, '[frontend]')
    runs-on: ubuntu-latest
    steps:
      - run: echo "Building frontend..."
  
  # Sadece backend değişirse
  backend:
    if: contains(github.event.head_commit.message, '[backend]')
    runs-on: ubuntu-latest
    steps:
      - run: echo "Building backend..."
  
  # Path-specific trigger
  api-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: dorny/paths-filter@v3
        id: changes
        with:
          filters: |
            api:
              - 'api/**'
              - 'tests/api/**'
            frontend:
              - 'frontend/**'
      
      - name: Run API tests
        if: steps.changes.outputs.api == 'true'
        run: npm run test:api
      
      - name: Run frontend tests
        if: steps.changes.outputs.frontend == 'true'
        run: npm run test:frontend
```

### Branch-Specific Actions

```yaml
# .github/workflows/branch-specific.yml
name: Branch-Specific Workflow

on:
  push:
    branches:
      - main
      - develop
      - 'feature/**'
      - 'release/**'

jobs:
  develop-only:
    if: github.ref == 'refs/heads/develop'
    runs-on: ubuntu-latest
    steps:
      - run: echo "Running on develop branch"
  
  main-only:
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - run: echo "Running on main branch"
  
  feature-branches:
    if: startsWith(github.ref, 'refs/heads/feature/')
    runs-on: ubuntu-latest
    steps:
      - run: echo "Running on feature branch"
```

## Reusable Workflows

### Workflow Template

```yaml
# .github/workflows/reusable-deploy.yml
name: Reusable Deploy

on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string
      version:
        required: false
        type: string
        default: 'latest'
    secrets:
      deploy-token:
        required: true
    outputs:
      deployment-url:
        description: "Deployment URL"
        value: ${{ jobs.deploy.outputs.url }}

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    outputs:
      url: ${{ steps.deploy.outputs.url }}
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy application
        id: deploy
        env:
          TOKEN: ${{ secrets.deploy-token }}
          VERSION: ${{ inputs.version }}
        run: |
          echo "Deploying version $VERSION to ${{ inputs.environment }}"
          echo "url=https://${{ inputs.environment }}.example.com" >> $GITHUB_OUTPUT
```

### Calling Reusable Workflow

```yaml
# .github/workflows/main-deploy.yml
name: Main Deploy Pipeline

on:
  push:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm test
  
  deploy-staging:
    needs: test
    uses: ./.github/workflows/reusable-deploy.yml
    with:
      environment: staging
      version: ${{ github.sha }}
    secrets:
      deploy-token: ${{ secrets.STAGING_DEPLOY_TOKEN }}
  
  deploy-production:
    needs: deploy-staging
    uses: ./.github/workflows/reusable-deploy.yml
    with:
      environment: production
      version: ${{ github.sha }}
    secrets:
      deploy-token: ${{ secrets.PROD_DEPLOY_TOKEN }}
  
  notify:
    needs: deploy-production
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deployed to ${{ needs.deploy-production.outputs.deployment-url }}"
```

## Custom Actions

### JavaScript Action

```javascript
// .github/actions/hello/index.js
const core = require('@actions/core');
const github = require('@actions/github');

try {
  const nameToGreet = core.getInput('who-to-greet');
  console.log(`Hello ${nameToGreet}!`);
  
  const time = new Date().toTimeString();
  core.setOutput('time', time);
  
  const payload = JSON.stringify(github.context.payload, undefined, 2);
  console.log(`Event payload: ${payload}`);
} catch (error) {
  core.setFailed(error.message);
}
```

```yaml
# .github/actions/hello/action.yml
name: 'Hello Action'
description: 'Greet someone'
inputs:
  who-to-greet:
    description: 'Who to greet'
    required: true
    default: 'World'
outputs:
  time:
    description: 'The time we greeted'
runs:
  using: 'node20'
  main: 'index.js'
```

### Composite Action

```yaml
# .github/actions/setup-app/action.yml
name: 'Setup Application'
description: 'Setup Node.js and install dependencies'

inputs:
  node-version:
    description: 'Node.js version'
    required: false
    default: '20'

runs:
  using: 'composite'
  steps:
    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: ${{ inputs.node-version }}
        cache: 'npm'
    
    - name: Install dependencies
      shell: bash
      run: npm ci
    
    - name: Print versions
      shell: bash
      run: |
        node --version
        npm --version
```

### Using Custom Action

```yaml
# .github/workflows/use-custom-action.yml
name: Use Custom Actions

on: push

jobs:
  greet:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Say hello
        uses: ./.github/actions/hello
        with:
          who-to-greet: 'GitHub Actions'
      
      - name: Setup app
        uses: ./.github/actions/setup-app
        with:
          node-version: '20'
      
      - name: Build
        run: npm run build
```

## Scheduled Workflows

### Cron Jobs

```yaml
# .github/workflows/scheduled.yml
name: Scheduled Tasks

on:
  schedule:
    # Her gün saat 02:00'de (UTC)
    - cron: '0 2 * * *'
    # Her Pazartesi saat 09:00'da
    - cron: '0 9 * * 1'
    # Her 6 saatte bir
    - cron: '0 */6 * * *'
  workflow_dispatch:  # Manuel tetikleme de ekle

jobs:
  daily-backup:
    runs-on: ubuntu-latest
    steps:
      - name: Backup database
        run: |
          echo "Running daily backup at $(date)"
          # Backup komutları
  
  weekly-cleanup:
    runs-on: ubuntu-latest
    if: github.event.schedule == '0 9 * * 1'
    steps:
      - name: Clean old artifacts
        run: |
          echo "Weekly cleanup at $(date)"
          # Cleanup komutları
  
  health-check:
    runs-on: ubuntu-latest
    steps:
      - name: Check service health
        run: |
          curl -f https://api.example.com/health || exit 1
```

## Deployment Strategies

### Blue-Green Deployment

```yaml
# .github/workflows/blue-green-deploy.yml
name: Blue-Green Deployment

on:
  push:
    branches: [ main ]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Build new version (green)
        run: docker build -t myapp:green .
      
      - name: Deploy to green environment
        run: |
          docker stop myapp-green || true
          docker rm myapp-green || true
          docker run -d --name myapp-green -p 8081:80 myapp:green
      
      - name: Health check green
        run: |
          sleep 10
          curl -f http://localhost:8081/health || exit 1
      
      - name: Switch traffic (blue -> green)
        run: |
          # Load balancer'da trafiği green'e yönlendir
          echo "Switching traffic to green..."
          # nginx/haproxy reconfigure
      
      - name: Monitor green
        run: |
          sleep 30
          # Metrikler ve loglar kontrol et
      
      - name: Cleanup old blue
        run: |
          docker stop myapp-blue || true
          docker rm myapp-blue || true
          docker tag myapp:green myapp:blue
```

### Canary Deployment

```yaml
# .github/workflows/canary-deploy.yml
name: Canary Deployment

on:
  push:
    branches: [ main ]

jobs:
  canary-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy canary (10% traffic)
        run: |
          kubectl set image deployment/myapp myapp=myapp:${{ github.sha }}
          kubectl scale deployment/myapp-canary --replicas=1
          kubectl scale deployment/myapp-stable --replicas=9
      
      - name: Monitor canary
        run: |
          sleep 300  # 5 dakika izle
          ERROR_RATE=$(kubectl logs deployment/myapp-canary | grep ERROR | wc -l)
          if [ $ERROR_RATE -gt 10 ]; then
            echo "Canary has high error rate, rolling back..."
            exit 1
          fi
      
      - name: Increase canary traffic (50%)
        run: |
          kubectl scale deployment/myapp-canary --replicas=5
          kubectl scale deployment/myapp-stable --replicas=5
      
      - name: Monitor again
        run: sleep 300
      
      - name: Full rollout (100%)
        run: |
          kubectl set image deployment/myapp-stable myapp=myapp:${{ github.sha }}
          kubectl scale deployment/myapp-stable --replicas=10
          kubectl scale deployment/myapp-canary --replicas=0
```

## Monitoring ve Notifications

### Slack Notifications

```yaml
# .github/workflows/slack-notify.yml
name: Slack Notifications

on:
  push:
    branches: [ main ]
  pull_request:
    types: [opened, reopened]

jobs:
  notify:
    runs-on: ubuntu-latest
    steps:
      - name: Notify start
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "Deployment started for ${{ github.repository }}",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "*Deployment Started*\nRepository: ${{ github.repository }}\nBranch: ${{ github.ref }}\nActor: ${{ github.actor }}"
                  }
                }
              ]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
      
      - name: Deploy
        run: echo "Deploying..."
      
      - name: Notify success
        if: success()
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "Deployment successful!",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "*Deployment Successful*\nRepository: ${{ github.repository }}\nCommit: ${{ github.sha }}"
                  }
                }
              ]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
      
      - name: Notify failure
        if: failure()
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "❌ Deployment failed!",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "*❌ Deployment Failed*\nRepository: ${{ github.repository }}\nWorkflow: ${{ github.workflow }}\nRun: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
                  }
                }
              ]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

### Email Notifications

```yaml
# .github/workflows/email-notify.yml
name: Email Notifications

on:
  workflow_run:
    workflows: ["CI Pipeline"]
    types: [completed]

jobs:
  notify:
    runs-on: ubuntu-latest
    steps:
      - name: Send email on failure
        if: ${{ github.event.workflow_run.conclusion == 'failure' }}
        uses: dawidd6/action-send-mail@v3
        with:
          server_address: smtp.gmail.com
          server_port: 465
          username: ${{ secrets.EMAIL_USERNAME }}
          password: ${{ secrets.EMAIL_PASSWORD }}
          subject: "CI Pipeline Failed: ${{ github.repository }}"
          body: |
            Workflow failed for ${{ github.repository }}
            
            Branch: ${{ github.ref }}
            Commit: ${{ github.sha }}
            Author: ${{ github.actor }}
            
            View logs: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}
          to: devops@example.com
          from: ci-notifications@example.com
```

## Best Practices

### Security Best Practices

```yaml
# .github/workflows/security.yml
name: Security Best Practices

on: [push, pull_request]

jobs:
  security-checks:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      security-events: write
    
    steps:
      - uses: actions/checkout@v4
      
      # 1. Dependency vulnerability scanning
      - name: Run Snyk to check for vulnerabilities
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
      
      # 2. Code scanning (SAST)
      - name: Initialize CodeQL
        uses: github/codeql-action/init@v3
        with:
          languages: javascript, python
      
      - name: Perform CodeQL Analysis
        uses: github/codeql-action/analyze@v3
      
      # 3. Secret scanning
      - name: Gitleaks scan
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      
      # 4. Container scanning
      - name: Build Docker image
        run: docker build -t myapp:test .
      
      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: 'myapp:test'
          format: 'sarif'
          output: 'trivy-results.sarif'
      
      - name: Upload Trivy results to GitHub Security
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: 'trivy-results.sarif'
```

### Performance Optimization

```yaml
# .github/workflows/optimized.yml
name: Optimized Workflow

on: push

jobs:
  parallel-jobs:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        task: [lint, test, build]
    
    steps:
      - uses: actions/checkout@v4
      
      # Paralel çalışma için görevleri ayır
      - name: Run ${{ matrix.task }}
        run: npm run ${{ matrix.task }}
  
  conditional-steps:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      # Cache kullan
      - uses: actions/cache@v4
        with:
          path: ~/.npm
          key: ${{ runner.os }}-npm-${{ hashFiles('**/package-lock.json') }}
      
      # Değişen dosyalara göre çalıştır
      - uses: dorny/paths-filter@v3
        id: changes
        with:
          filters: |
            src:
              - 'src/**'
            tests:
              - 'tests/**'
      
      - name: Build (only if src changed)
        if: steps.changes.outputs.src == 'true'
        run: npm run build
      
      - name: Test (only if tests changed)
        if: steps.changes.outputs.tests == 'true'
        run: npm test
```

### Error Handling

```yaml
# .github/workflows/error-handling.yml
name: Error Handling

on: push

jobs:
  resilient-job:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      # Timeout ekle
      - name: Run tests with timeout
        timeout-minutes: 10
        run: npm test
      
      # Retry mekanizması
      - name: Deploy with retry
        uses: nick-invision/retry@v2
        with:
          timeout_minutes: 5
          max_attempts: 3
          retry_wait_seconds: 30
          command: npm run deploy
      
      # Continue on error
      - name: Optional linting
        continue-on-error: true
        run: npm run lint
      
      # Conditional execution
      - name: Cleanup on failure
        if: failure()
        run: |
          echo "Workflow failed, cleaning up..."
          docker system prune -af
```

## Sonuç

GitHub Actions ile CI/CD pipeline'larınızı tamamen otomatikleştirebilir, kod kalitesini artırabilir ve deployment süreçlerinizi hızlandırabilirsiniz.

### Key Takeaways

- **Automation**: Manuel süreçleri ortadan kaldırın
- **Testing**: Her commit'te otomatik test çalıştırın
- **Security**: Güvenlik taramaları entegre edin
- **Monitoring**: Pipeline'ları sürekli izleyin
- **Optimization**: Cache ve paralel execution kullanın

### Kaynaklar

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [GitHub Actions Marketplace](https://github.com/marketplace?type=actions)
- [Awesome Actions](https://github.com/sdras/awesome-actions)
- [GitHub Actions Best Practices](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions)

CI/CD pipeline'ınızı kurarken güvenlik, performans ve maintainability'yi öncelik olarak görün!
