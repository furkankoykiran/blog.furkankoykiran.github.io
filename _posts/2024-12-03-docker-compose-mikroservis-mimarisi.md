---
title: "Docker ve Docker Compose ile Mikroservis Mimarisi Geliştirme"
description: "Docker Compose ile tam mikroservis mimarisi. Container orchestration, service discovery, load balancing ve production deployment."
date: 2024-12-03 09:00:00 +0300
categories: [DevOps, Infrastructure]
tags: [docker, docker-compose, microservices, container, deployment, orchestration]
image:
  path: /assets/img/posts/docker-architecture-diagram.png
  alt: "Docker Mimarisi ve Bileşenleri"
---

Modern yazılım geliştirmede konteynerleştirme, uygulamaların taşınabilir, ölçeklenebilir ve tutarlı bir şekilde çalışmasını sağlayan temel bir teknoloji haline gelmiştir. Docker, konteyner teknolojisinin en popüler implementasyonudur ve Docker Compose, çoklu konteyner uygulamaların yönetimini kolaylaştırır. Bu yazıda Docker temellerinden mikroservis mimarisi oluşturmaya kadar detaylı bir rehber sunacağız.

## Docker Nedir ve Neden Kullanmalıyız?

Docker, uygulamaları ve bağımlılıklarını izole edilmiş konteynerlerde çalıştırmamızı sağlayan bir platform'dur. Sanal makinelere göre daha hafif ve hızlıdır çünkü işletim sistemi çekirdeğini paylaşır.

### Docker'ın Avantajları

- **Tutarlılık**: "Benim makinemde çalışıyordu" problemini çözer
- **Taşınabilirlik**: Herhangi bir ortamda aynı şekilde çalışır
- **İzolasyon**: Her konteyner izole bir ortamda çalışır
- **Hız**: Saniyeler içinde başlatılıp durdurulabilir
- **Kaynak Verimliliği**: VM'lere göre çok daha az kaynak tüketir
- **Ölçeklenebilirlik**: Kolayca yatay ölçeklendirme yapılabilir

## Docker Kurulumu

Ubuntu/Debian için Docker kurulumu:

```bash
# Sistem güncellemesi
sudo apt update
sudo apt upgrade -y

# Gerekli paketleri kur
sudo apt install -y apt-transport-https ca-certificates curl software-properties-common

# Docker GPG anahtarını ekle
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# Docker repository'yi ekle
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Docker'ı kur
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Docker'ı başlat ve otomatik başlatmayı etkinleştir
sudo systemctl start docker
sudo systemctl enable docker

# Kullanıcıyı docker grubuna ekle (sudo olmadan çalıştırmak için)
sudo usermod -aG docker $USER

# Değişikliklerin geçerli olması için oturumu yenile
newgrp docker

# Docker kurulumunu test et
docker --version
docker compose version
```

## Docker Temel Kavramlar

### Image (İmaj)

Docker image, uygulamayı çalıştırmak için gereken tüm bileşenleri içeren read-only şablondur:

```bash
# Docker Hub'dan image çekme
docker pull nginx:latest
docker pull python:3.11-slim
docker pull postgres:15-alpine

# Mevcut image'ları listeleme
docker images

# Image silme
docker rmi nginx:latest

# Image detaylarını inceleme
docker inspect nginx:latest
```

### Container (Konteyner)

Container, bir image'in çalışan instance'ıdır:

```bash
# Konteyner çalıştırma
docker run nginx

# Detached mode (arkaplanda) çalıştırma
docker run -d --name my-nginx nginx

# Port mapping ile çalıştırma
docker run -d -p 8080:80 --name web-server nginx

# Environment variable ile çalıştırma
docker run -d -e POSTGRES_PASSWORD=secret postgres:15

# Çalışan konteynerleri listeleme
docker ps

# Tüm konteynerleri listeleme (durdurulmuş olanlar dahil)
docker ps -a

# Konteyner durdurma
docker stop my-nginx

# Konteyner başlatma
docker start my-nginx

# Konteyner yeniden başlatma
docker restart my-nginx

# Konteyner silme
docker rm my-nginx

# Çalışan konteyneri zorla silme
docker rm -f my-nginx

# Tüm durdurulmuş konteynerleri silme
docker container prune
```

### Dockerfile Oluşturma

Dockerfile, custom image oluşturmak için kullanılan talimatlar dosyasıdır:

```dockerfile
# Base image seçimi
FROM python:3.11-slim

# Metadata
LABEL maintainer="dev@example.com"
LABEL version="1.0"
LABEL description="FastAPI microservice"

# Çalışma dizini oluştur
WORKDIR /app

# Sistem bağımlılıklarını kur
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
    gcc \
    && rm -rf /var/lib/apt/lists/*

# Python bağımlılıklarını kopyala
COPY requirements.txt .

# Bağımlılıkları kur
RUN pip install --no-cache-dir -r requirements.txt

# Uygulama kodunu kopyala
COPY . .

# Gerekli dizinleri oluştur
RUN mkdir -p /app/logs /app/data

# Non-root kullanıcı oluştur (security best practice)
RUN useradd -m -u 1000 appuser && \
    chown -R appuser:appuser /app

# Non-root kullanıcıya geç
USER appuser

# Port açıklaması (dokümantasyon amaçlı)
EXPOSE 8000

# Health check tanımlama
HEALTHCHECK --interval=30s --timeout=3s --start-period=40s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

# Başlangıç komutu
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```
{: file="Dockerfile" }

Image build etme:

```bash
# Image oluşturma
docker build -t myapp:1.0 .

# Build argument ile oluşturma
docker build --build-arg VERSION=1.0 -t myapp:1.0 .

# No-cache ile oluşturma (tüm katmanları yeniden oluştur)
docker build --no-cache -t myapp:1.0 .

# Multi-stage build örneği
```

### Multi-Stage Build

Production için optimize edilmiş image oluşturma:

```dockerfile
# Build stage
FROM python:3.11 AS builder

WORKDIR /app

# Bağımlılıkları kur
COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

# Production stage
FROM python:3.11-slim

WORKDIR /app

# Build stage'den sadece gerekli dosyaları kopyala
COPY --from=builder /root/.local /root/.local
COPY . .

# PATH'i güncelle
ENV PATH=/root/.local/bin:$PATH

USER 1000

EXPOSE 8000

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```
{: file="Dockerfile.multistage" }

Bu yaklaşım, final image boyutunu önemli ölçüde küçültür.

![Docker Compose Microservices](/assets/img/posts/docker-compose-microservices.png)
_Docker Compose ile Mikroservis Mimarisi_

## Docker Compose ile Çoklu Konteyner Yönetimi

Docker Compose, birden fazla konteyneri tek bir YAML dosyası ile yönetmemizi sağlar.

### Basit Docker Compose Örneği

```yaml
version: '3.8'

services:
  # Web servisi
  web:
    build: ./web
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql://user:pass@db:5432/mydb
      - REDIS_URL=redis://redis:6379
    depends_on:
      - db
      - redis
    volumes:
      - ./web:/app
      - web-logs:/app/logs
    networks:
      - app-network
    restart: unless-stopped

  # Veritabanı servisi
  db:
    image: postgres:15-alpine
    environment:
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=pass
      - POSTGRES_DB=mydb
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql:ro
    networks:
      - app-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Redis servisi
  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes
    volumes:
      - redis-data:/data
    networks:
      - app-network
    restart: unless-stopped

# Volume tanımlamaları
volumes:
  postgres-data:
  redis-data:
  web-logs:

# Network tanımlamaları
networks:
  app-network:
    driver: bridge
```
{: file="docker-compose.yml" }

### Docker Compose Komutları

```bash
# Servisleri başlat (detached mode)
docker compose up -d

# Servisleri başlat ve logları takip et
docker compose up

# Belirli bir servisi başlat
docker compose up -d web

# Servisleri durdur
docker compose stop

# Servisleri durdur ve kaldır
docker compose down

# Servisleri, volume'ları ve orphan container'ları da kaldır
docker compose down -v --remove-orphans

# Servisleri yeniden oluştur ve başlat
docker compose up -d --build

# Çalışan servislerin durumunu görüntüle
docker compose ps

# Servis loglarını görüntüle
docker compose logs

# Belirli bir servisin loglarını takip et
docker compose logs -f web

# Servisi yeniden başlat
docker compose restart web

# Servise komut çalıştır
docker compose exec web python manage.py migrate

# Yeni bir shell başlat
docker compose exec web /bin/bash

# Çalışan konteynerlere ait kaynakları görüntüle
docker compose top

# Servisleri ölçeklendir
docker compose up -d --scale web=3
```

![Docker Networking](/assets/img/posts/docker-container-networking.png)
_Docker Container Networking ve Bridge Architecture_

## Docker Networking

Docker, konteynerler arası iletişim için güçlü network özellikleri sunar.

### Network Türleri

```bash
# Bridge network (default)
docker network create my-bridge-network

# Host network (konteynerdoğrudan host network kullanır)
docker run --network host nginx

# None network (network yok)
docker run --network none alpine

# Custom bridge network oluşturma
docker network create --driver bridge \
  --subnet=172.18.0.0/16 \
  --gateway=172.18.0.1 \
  custom-network

# Network'leri listeleme
docker network ls

# Network detaylarını görüntüleme
docker network inspect my-bridge-network

# Konteyneri network'e bağlama
docker network connect my-bridge-network my-container

# Konteyneri network'ten ayırma
docker network disconnect my-bridge-network my-container

# Kullanılmayan network'leri silme
docker network prune
```

### Docker Compose ile Network Yönetimi

```yaml
version: '3.8'

services:
  frontend:
    image: nginx
    networks:
      - frontend-net
      - backend-net
    ports:
      - "80:80"

  api:
    build: ./api
    networks:
      - backend-net
      - database-net
    environment:
      - DB_HOST=database

  database:
    image: postgres:15
    networks:
      - database-net
    volumes:
      - db-data:/var/lib/postgresql/data

networks:
  # Frontend network (public)
  frontend-net:
    driver: bridge

  # Backend network (internal communication)
  backend-net:
    driver: bridge
    internal: false

  # Database network (isolated)
  database-net:
    driver: bridge
    internal: true

volumes:
  db-data:
```

![Docker Volumes](/assets/img/posts/docker-volumes-storage.png)
_Docker Volumes ve Persistent Storage_

## Docker Volumes ile Veri Kalıcılığı

Konteynerler ephemeral (geçici) olduğu için kalıcı veri depolamak için volume kullanırız.

### Volume Türleri

```bash
# Named volume oluşturma
docker volume create my-data

# Volume'ları listeleme
docker volume ls

# Volume detaylarını görüntüleme
docker volume inspect my-data

# Volume kullanarak konteyner başlatma
docker run -d -v my-data:/data nginx

# Bind mount kullanma (host dizinini mount etme)
docker run -d -v /host/path:/container/path nginx

# Read-only bind mount
docker run -d -v /host/path:/container/path:ro nginx

# tmpfs mount (RAM'de geçici depolama)
docker run -d --tmpfs /tmp:rw,size=1g nginx

# Volume silme
docker volume rm my-data

# Kullanılmayan volume'ları silme
docker volume prune
```

### Docker Compose ile Volume Yönetimi

```yaml
version: '3.8'

services:
  app:
    build: .
    volumes:
      # Named volume
      - app-data:/app/data
      
      # Bind mount (kod geliştirme için)
      - ./src:/app/src:ro
      
      # Anonymous volume
      - /app/node_modules
      
      # Config file mount
      - ./config.yml:/app/config.yml:ro

  database:
    image: postgres:15
    volumes:
      # Database data
      - db-data:/var/lib/postgresql/data
      
      # Init scripts
      - ./init-scripts:/docker-entrypoint-initdb.d:ro
      
      # Backup directory
      - ./backups:/backups

volumes:
  app-data:
    driver: local
    driver_opts:
      type: none
      device: /mnt/app-data
      o: bind

  db-data:
    driver: local
```

## Mikroservis Mimarisi Örneği

Gerçek dünya senaryosu için kapsamlı bir mikroservis projesi oluşturalım.

### Proje Yapısı

```
microservices-app/
├── docker-compose.yml
├── docker-compose.prod.yml
├── .env
├── nginx/
│   ├── Dockerfile
│   └── nginx.conf
├── api-gateway/
│   ├── Dockerfile
│   ├── requirements.txt
│   └── main.py
├── user-service/
│   ├── Dockerfile
│   ├── requirements.txt
│   └── main.py
├── product-service/
│   ├── Dockerfile
│   ├── requirements.txt
│   └── main.py
└── order-service/
    ├── Dockerfile
    ├── requirements.txt
    └── main.py
```

### Ana Docker Compose Dosyası

```yaml
# docker-compose.yml
version: '3.8'

x-common-variables: &common-variables
  REDIS_URL: redis://redis:6379
  RABBITMQ_URL: amqp://guest:guest@rabbitmq:5672

services:
  # Nginx Reverse Proxy
  nginx:
    build: ./nginx
    ports:
      - "80:80"
      - "443:443"
    depends_on:
      - api-gateway
    networks:
      - frontend
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
      - ./ssl:/etc/nginx/ssl:ro
      - nginx-logs:/var/log/nginx
    restart: unless-stopped

  # API Gateway
  api-gateway:
    build:
      context: ./api-gateway
      args:
        - VERSION=${VERSION:-latest}
    environment:
      <<: *common-variables
      USER_SERVICE_URL: http://user-service:8001
      PRODUCT_SERVICE_URL: http://product-service:8002
      ORDER_SERVICE_URL: http://order-service:8003
    networks:
      - frontend
      - backend
    depends_on:
      - user-service
      - product-service
      - order-service
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    restart: unless-stopped

  # User Service
  user-service:
    build: ./user-service
    environment:
      <<: *common-variables
      DATABASE_URL: postgresql://user:${POSTGRES_PASSWORD}@user-db:5432/users
      JWT_SECRET: ${JWT_SECRET}
    networks:
      - backend
      - user-db-net
    depends_on:
      user-db:
        condition: service_healthy
      redis:
        condition: service_started
    deploy:
      replicas: 2
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
    restart: unless-stopped

  user-db:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: users
    volumes:
      - user-db-data:/var/lib/postgresql/data
      - ./user-service/init.sql:/docker-entrypoint-initdb.d/init.sql:ro
    networks:
      - user-db-net
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped

  # Product Service
  product-service:
    build: ./product-service
    environment:
      <<: *common-variables
      DATABASE_URL: postgresql://user:${POSTGRES_PASSWORD}@product-db:5432/products
      ELASTICSEARCH_URL: http://elasticsearch:9200
    networks:
      - backend
      - product-db-net
      - search-net
    depends_on:
      product-db:
        condition: service_healthy
      elasticsearch:
        condition: service_healthy
    restart: unless-stopped

  product-db:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: products
    volumes:
      - product-db-data:/var/lib/postgresql/data
    networks:
      - product-db-net
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped

  # Order Service
  order-service:
    build: ./order-service
    environment:
      <<: *common-variables
      DATABASE_URL: postgresql://user:${POSTGRES_PASSWORD}@order-db:5432/orders
      PAYMENT_API_KEY: ${PAYMENT_API_KEY}
    networks:
      - backend
      - order-db-net
    depends_on:
      order-db:
        condition: service_healthy
      rabbitmq:
        condition: service_healthy
    restart: unless-stopped

  order-db:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: orders
    volumes:
      - order-db-data:/var/lib/postgresql/data
    networks:
      - order-db-net
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped

  # Redis (Caching)
  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes --requirepass ${REDIS_PASSWORD}
    volumes:
      - redis-data:/data
    networks:
      - backend
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 3
    restart: unless-stopped

  # RabbitMQ (Message Queue)
  rabbitmq:
    image: rabbitmq:3-management-alpine
    environment:
      RABBITMQ_DEFAULT_USER: ${RABBITMQ_USER:-guest}
      RABBITMQ_DEFAULT_PASS: ${RABBITMQ_PASSWORD:-guest}
    ports:
      - "15672:15672"  # Management UI
    volumes:
      - rabbitmq-data:/var/lib/rabbitmq
    networks:
      - backend
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "ping"]
      interval: 30s
      timeout: 10s
      retries: 3
    restart: unless-stopped

  # Elasticsearch (Search)
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    volumes:
      - elasticsearch-data:/usr/share/elasticsearch/data
    networks:
      - search-net
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:9200/_cluster/health || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 3
    restart: unless-stopped

# Volume Tanımlamaları
volumes:
  user-db-data:
  product-db-data:
  order-db-data:
  redis-data:
  rabbitmq-data:
  elasticsearch-data:
  nginx-logs:

# Network Tanımlamaları
networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: false
  user-db-net:
    driver: bridge
    internal: true
  product-db-net:
    driver: bridge
    internal: true
  order-db-net:
    driver: bridge
    internal: true
  search-net:
    driver: bridge
```

### Environment Variables

```bash
# .env
VERSION=1.0.0

# Database
POSTGRES_PASSWORD=your_secure_password_here

# Redis
REDIS_PASSWORD=your_redis_password

# RabbitMQ
RABBITMQ_USER=admin
RABBITMQ_PASSWORD=your_rabbitmq_password

# JWT
JWT_SECRET=your_jwt_secret_key_min_32_chars

# Payment API
PAYMENT_API_KEY=your_payment_api_key
```

### Nginx Configuration

```nginx
# nginx/nginx.conf
upstream api_gateway {
    least_conn;
    server api-gateway:8000 max_fails=3 fail_timeout=30s;
}

server {
    listen 80;
    server_name api.example.com;

    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;

    # Logging
    access_log /var/log/nginx/access.log combined;
    error_log /var/log/nginx/error.log warn;

    # Rate limiting
    limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;
    limit_req zone=api_limit burst=20 nodelay;

    # API Gateway
    location /api/ {
        proxy_pass http://api_gateway/;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Timeouts
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;

        # Buffering
        proxy_buffering off;
        proxy_request_buffering off;
    }

    # Health check endpoint
    location /health {
        access_log off;
        return 200 "healthy\n";
        add_header Content-Type text/plain;
    }
}
```

## Production Best Practices

### 1. Multi-Environment Setup

```yaml
# docker-compose.prod.yml
version: '3.8'

services:
  api-gateway:
    deploy:
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  # Production-specific overrides
  nginx:
    volumes:
      - /etc/letsencrypt:/etc/letsencrypt:ro
```

Çalıştırma:

```bash
# Development
docker compose up -d

# Production
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

### 2. Backup ve Restore

```bash
#!/bin/bash
# backup.sh

# Database backup
docker compose exec -T user-db pg_dump -U user users > backups/users_$(date +%Y%m%d_%H%M%S).sql

# Volume backup
docker run --rm \
  -v microservices-app_user-db-data:/data \
  -v $(pwd)/backups:/backup \
  alpine tar czf /backup/user-db-volume_$(date +%Y%m%d_%H%M%S).tar.gz /data
```

```bash
#!/bin/bash
# restore.sh

# Database restore
docker compose exec -T user-db psql -U user users < backups/users_20241203_120000.sql

# Volume restore
docker run --rm \
  -v microservices-app_user-db-data:/data \
  -v $(pwd)/backups:/backup \
  alpine sh -c "cd /data && tar xzf /backup/user-db-volume_20241203_120000.tar.gz --strip 1"
```

### 3. Monitoring ve Logging

```yaml
# docker-compose.monitoring.yml
version: '3.8'

services:
  # Prometheus (Metrics)
  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus-data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
    ports:
      - "9090:9090"
    networks:
      - monitoring

  # Grafana (Visualization)
  grafana:
    image: grafana/grafana:latest
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_PASSWORD}
    volumes:
      - grafana-data:/var/lib/grafana
      - ./grafana/dashboards:/etc/grafana/provisioning/dashboards:ro
    ports:
      - "3000:3000"
    networks:
      - monitoring
    depends_on:
      - prometheus

  # Loki (Logging)
  loki:
    image: grafana/loki:latest
    ports:
      - "3100:3100"
    volumes:
      - loki-data:/loki
    networks:
      - monitoring

volumes:
  prometheus-data:
  grafana-data:
  loki-data:

networks:
  monitoring:
    driver: bridge
```

### 4. CI/CD Pipeline

```yaml
# .github/workflows/deploy.yml
name: Deploy to Production

on:
  push:
    branches: [ main ]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2

      - name: Login to Docker Hub
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Build and push
        uses: docker/build-push-action@v4
        with:
          context: ./api-gateway
          push: true
          tags: username/api-gateway:${{ github.sha }},username/api-gateway:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Deploy to server
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            cd /opt/microservices-app
            docker compose pull
            docker compose up -d --remove-orphans
            docker system prune -f
```

## Sonuç

Docker ve Docker Compose, modern uygulama geliştirme ve deployment süreçlerinde vazgeçilmez araçlar haline gelmiştir. Mikroservis mimarisinde, her servisin izole edilmesi, bağımsız ölçeklendirilmesi ve yönetilmesi için ideal bir platform sunarlar.

Bu yazıda öğrendikleriniz:
- Docker temel kavramları ve kullanımı
- Dockerfile ile custom image oluşturma
- Docker Compose ile çoklu konteyner yönetimi
- Network ve volume yönetimi
- Production-ready mikroservis mimarisi
- Monitoring, backup ve CI/CD entegrasyonu

### Önerilen Kaynaklar

- [Docker Resmi Dokümantasyonu](https://docs.docker.com/)
- [Docker Compose Specification](https://docs.docker.com/compose/compose-file/)
- [Docker Best Practices](https://docs.docker.com/develop/dev-best-practices/)
- [Docker Security](https://docs.docker.com/engine/security/)
- [Kubernetes (Next Step)](https://kubernetes.io/)

Bir sonraki yazımızda, Redis ile önbellekleme ve oturum yönetimini inceleyeceğiz. Takipte kalın!
