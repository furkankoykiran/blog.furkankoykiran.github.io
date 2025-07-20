---
title: "Nginx ile Yük Dengeleme ve Önbellekleme: Performans Optimizasyonu"
date: 2025-07-20 11:00:00 +0300
categories: [Infrastructure, Performance]
tags: [nginx, load-balancing, caching, reverse-proxy, performance]
image:
  path: /assets/img/posts/nginx-reverse-proxy-architecture.png
  alt: "Nginx Reverse Proxy Mimarisi"
---

Modern web uygulamalarında performans ve yüksek erişilebilirlik kritik öneme sahiptir. Nginx, hafif yapısı, yüksek performansı ve güçlü özellikleriyle web sunucusu, reverse proxy ve load balancer olarak tercih edilen bir çözümdür. Bu yazıda Nginx'in temel özelliklerini, yük dengeleme stratejilerini ve önbellekleme tekniklerini detaylı olarak inceleyeceğiz.

## Nginx Nedir?

Nginx (engine-x olarak okunur), Igor Sysoev tarafından 2004 yılında geliştirilen açık kaynaklı bir web sunucusu ve reverse proxy yazılımıdır. Event-driven, asenkron mimarisi sayesinde binlerce eşzamanlı bağlantıyı minimum kaynak kullanımıyla yönetebilir.

### Nginx vs Apache

```bash
# Apache (Process-based)
# Her istek için yeni bir process/thread
# Yüksek bellek kullanımı
# C10K problemi (10.000 eşzamanlı bağlantı)

# Nginx (Event-driven)
# Asenkron, non-blocking I/O
# Düşük bellek kullanımı
# 50.000+ eşzamanlı bağlantı
```

### Nginx Kurulumu

```bash
# Ubuntu/Debian
sudo apt update
sudo apt install nginx

# CentOS/RHEL
sudo yum install nginx

# macOS
brew install nginx

# Servisi başlatma
sudo systemctl start nginx
sudo systemctl enable nginx

# Durum kontrolü
sudo systemctl status nginx
sudo nginx -t  # Config test
```

## Temel Nginx Yapılandırması

### Ana Yapılandırma Dosyası

```nginx
# /etc/nginx/nginx.conf

user www-data;
worker_processes auto;  # CPU core sayısı kadar worker
pid /run/nginx.pid;

events {
    worker_connections 1024;  # Her worker'ın max bağlantı sayısı
    use epoll;  # Linux için optimize edilmiş
}

http {
    ##
    # Temel Ayarlar
    ##
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;
    
    # MIME types
    include /etc/nginx/mime.types;
    default_type application/octet-stream;
    
    ##
    # Logging
    ##
    access_log /var/log/nginx/access.log;
    error_log /var/log/nginx/error.log;
    
    ##
    # Gzip Compression
    ##
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;
    
    ##
    # Virtual Host Configs
    ##
    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
}
```

### Basit Web Sunucusu

```nginx
# /etc/nginx/sites-available/mysite
server {
    listen 80;
    listen [::]:80;
    
    server_name example.com www.example.com;
    
    root /var/www/html;
    index index.html index.htm;
    
    location / {
        try_files $uri $uri/ =404;
    }
    
    # Static files caching
    location ~* \.(jpg|jpeg|png|gif|ico|css|js)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
}
```

```bash
# Site'ı aktifleştirme
sudo ln -s /etc/nginx/sites-available/mysite /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

## Reverse Proxy Yapılandırması

Nginx'i backend uygulama sunucularının önüne koyarak güvenlik ve performans artırılır.

```nginx
# /etc/nginx/sites-available/app-proxy
server {
    listen 80;
    server_name api.example.com;
    
    # Reverse proxy settings
    location / {
        proxy_pass http://localhost:8080;
        
        # Headers
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # Timeouts
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
        
        # Buffering
        proxy_buffering on;
        proxy_buffer_size 4k;
        proxy_buffers 8 4k;
        proxy_busy_buffers_size 8k;
    }
    
    # Websocket support
    location /ws {
        proxy_pass http://localhost:8080;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
    
    # Health check endpoint
    location /health {
        access_log off;
        return 200 "healthy\n";
        add_header Content-Type text/plain;
    }
}
```

## Yük Dengeleme (Load Balancing)

![Nginx Load Balancing](/assets/img/posts/nginx-load-balancing-diagram.png)
_Nginx load balancing algoritmaları_

### Upstream Tanımlama

```nginx
# /etc/nginx/conf.d/upstream.conf

# Round Robin (varsayılan)
upstream backend_roundrobin {
    server 10.0.0.1:8080;
    server 10.0.0.2:8080;
    server 10.0.0.3:8080;
}

# Least Connections
upstream backend_leastconn {
    least_conn;
    server 10.0.0.1:8080;
    server 10.0.0.2:8080;
    server 10.0.0.3:8080;
}

# IP Hash (session persistence)
upstream backend_iphash {
    ip_hash;
    server 10.0.0.1:8080;
    server 10.0.0.2:8080;
    server 10.0.0.3:8080;
}

# Weighted Load Balancing
upstream backend_weighted {
    server 10.0.0.1:8080 weight=3;  # 3x daha fazla trafik
    server 10.0.0.2:8080 weight=2;
    server 10.0.0.3:8080 weight=1;
}

# Health checks ve backup
upstream backend_advanced {
    server 10.0.0.1:8080 max_fails=3 fail_timeout=30s;
    server 10.0.0.2:8080 max_fails=3 fail_timeout=30s;
    server 10.0.0.3:8080 backup;  # Sadece diğerleri fail olursa
}
```

### Load Balancer Yapılandırması

![Nginx Load Balancer Architecture](/assets/img/posts/nginx-load-balancer-architecture.png)
_Gelişmiş load balancing mimarisi_

```nginx
server {
    listen 80;
    server_name app.example.com;
    
    location / {
        proxy_pass http://backend_roundrobin;
        
        # Sticky sessions (Nginx Plus)
        # sticky cookie srv_id expires=1h domain=.example.com path=/;
        
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        
        # Connection pooling
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        
        # Retry logic
        proxy_next_upstream error timeout invalid_header http_500 http_502 http_503;
        proxy_next_upstream_tries 3;
        proxy_next_upstream_timeout 10s;
    }
}
```

## Önbellekleme (Caching)

### Proxy Cache Yapılandırması

```nginx
# /etc/nginx/nginx.conf içinde (http bloğu)

# Cache zone tanımlama
proxy_cache_path /var/cache/nginx/proxy
    levels=1:2
    keys_zone=app_cache:10m
    max_size=1g
    inactive=60m
    use_temp_path=off;

server {
    listen 80;
    server_name app.example.com;
    
    # Cache için upstream
    location / {
        proxy_pass http://backend;
        
        # Cache settings
        proxy_cache app_cache;
        proxy_cache_valid 200 60m;
        proxy_cache_valid 404 10m;
        proxy_cache_use_stale error timeout updating http_500 http_502 http_503 http_504;
        proxy_cache_background_update on;
        proxy_cache_lock on;
        
        # Cache headers
        add_header X-Cache-Status $upstream_cache_status;
        
        # Cache key
        proxy_cache_key "$scheme$request_method$host$request_uri";
        
        # Bypass cache için
        proxy_cache_bypass $http_pragma $http_authorization;
        proxy_no_cache $http_pragma $http_authorization;
    }
    
    # Specific paths
    location ~* \.(jpg|jpeg|png|gif|ico|css|js)$ {
        proxy_pass http://backend;
        proxy_cache app_cache;
        proxy_cache_valid 200 7d;
        expires 7d;
        add_header Cache-Control "public, immutable";
    }
    
    # Cache purge (Nginx Plus)
    # location ~ /purge(/.*) {
    #     proxy_cache_purge app_cache "$scheme$request_method$host$1";
    # }
}
```

### FastCGI Cache (PHP)

```nginx
# FastCGI cache zone
fastcgi_cache_path /var/cache/nginx/fastcgi
    levels=1:2
    keys_zone=php_cache:10m
    max_size=1g
    inactive=60m;

server {
    listen 80;
    server_name php-app.example.com;
    root /var/www/html;
    
    location ~ \.php$ {
        fastcgi_pass unix:/var/run/php/php8.1-fpm.sock;
        fastcgi_index index.php;
        include fastcgi_params;
        
        # FastCGI cache
        fastcgi_cache php_cache;
        fastcgi_cache_valid 200 60m;
        fastcgi_cache_use_stale error timeout updating invalid_header http_500;
        fastcgi_cache_bypass $http_pragma;
        fastcgi_no_cache $http_pragma;
        
        # Cache key
        fastcgi_cache_key "$scheme$request_method$host$request_uri";
        
        add_header X-FastCGI-Cache $upstream_cache_status;
    }
}
```

## SSL/TLS Yapılandırması

```nginx
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name example.com www.example.com;
    
    # SSL certificates
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    
    # SSL protocols ve ciphers
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;
    ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384';
    
    # SSL session
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;
    ssl_session_tickets off;
    
    # OCSP stapling
    ssl_stapling on;
    ssl_stapling_verify on;
    ssl_trusted_certificate /etc/letsencrypt/live/example.com/chain.pem;
    
    # Security headers
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    
    location / {
        proxy_pass http://backend;
        # ... proxy settings
    }
}

# HTTP to HTTPS redirect
server {
    listen 80;
    listen [::]:80;
    server_name example.com www.example.com;
    return 301 https://$server_name$request_uri;
}
```

## Rate Limiting

```nginx
# Rate limit zones
http {
    # IP bazlı rate limiting
    limit_req_zone $binary_remote_addr zone=ip_limit:10m rate=10r/s;
    
    # API key bazlı
    limit_req_zone $http_x_api_key zone=api_limit:10m rate=100r/s;
    
    server {
        listen 80;
        server_name api.example.com;
        
        location /api/ {
            # Rate limiting uygula
            limit_req zone=ip_limit burst=20 nodelay;
            limit_req zone=api_limit burst=50;
            
            # Rate limit aşıldığında custom response
            limit_req_status 429;
            
            proxy_pass http://backend;
        }
    }
}
```

## Güvenlik Yapılandırması

```nginx
server {
    listen 80;
    server_name secure-app.example.com;
    
    # DDoS protection
    client_body_timeout 10s;
    client_header_timeout 10s;
    send_timeout 10s;
    
    # Buffer overflow protection
    client_body_buffer_size 1K;
    client_header_buffer_size 1k;
    client_max_body_size 10M;
    large_client_header_buffers 2 1k;
    
    # Hide Nginx version
    server_tokens off;
    
    # Clickjacking protection
    add_header X-Frame-Options "SAMEORIGIN" always;
    
    # XSS protection
    add_header X-XSS-Protection "1; mode=block" always;
    add_header X-Content-Type-Options "nosniff" always;
    
    # CSP
    add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline';" always;
    
    # Block bad bots
    if ($http_user_agent ~* (bot|spider|crawler|scanner)) {
        return 403;
    }
    
    # Block specific IPs
    deny 192.168.1.100;
    allow all;
    
    location / {
        proxy_pass http://backend;
    }
}
```

## Performans Optimizasyonu

```nginx
http {
    # Worker processes
    worker_processes auto;
    worker_rlimit_nofile 65535;
    
    events {
        worker_connections 4096;
        use epoll;
        multi_accept on;
    }
    
    # TCP optimization
    tcp_nopush on;
    tcp_nodelay on;
    sendfile on;
    
    # Keepalive
    keepalive_timeout 65;
    keepalive_requests 100;
    
    # Gzip compression
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_min_length 1000;
    gzip_types
        text/plain
        text/css
        text/xml
        text/javascript
        application/json
        application/javascript
        application/xml+rss
        application/rss+xml
        font/truetype
        font/opentype
        application/vnd.ms-fontobject
        image/svg+xml;
    
    # Open file cache
    open_file_cache max=10000 inactive=20s;
    open_file_cache_valid 30s;
    open_file_cache_min_uses 2;
    open_file_cache_errors on;
    
    # Buffer sizes
    client_body_buffer_size 128k;
    client_max_body_size 50M;
    client_header_buffer_size 1k;
    large_client_header_buffers 4 16k;
    output_buffers 1 32k;
    postpone_output 1460;
}
```

## Monitoring ve Logging

```nginx
# Access log formatting
http {
    log_format main '$remote_addr - $remote_user [$time_local] '
                    '"$request" $status $body_bytes_sent '
                    '"$http_referer" "$http_user_agent" '
                    '$request_time $upstream_response_time';
    
    log_format json escape=json '{'
        '"time":"$time_iso8601",'
        '"remote_addr":"$remote_addr",'
        '"request":"$request",'
        '"status":$status,'
        '"body_bytes_sent":$body_bytes_sent,'
        '"request_time":$request_time,'
        '"upstream_response_time":"$upstream_response_time"'
    '}';
    
    access_log /var/log/nginx/access.log main;
    error_log /var/log/nginx/error.log warn;
    
    # Conditional logging
    map $status $loggable {
        ~^[23] 0;  # 2xx ve 3xx loglanmaz
        default 1;
    }
    
    server {
        access_log /var/log/nginx/access.log main if=$loggable;
    }
}

# Nginx status endpoint
server {
    listen 8080;
    server_name localhost;
    
    location /nginx_status {
        stub_status on;
        access_log off;
        allow 127.0.0.1;
        deny all;
    }
}
```

```bash
# Status kontrolü
curl http://localhost:8080/nginx_status
# Active connections: 2
# server accepts handled requests
#  45 45 123
# Reading: 0 Writing: 1 Waiting: 1
```

## Docker ile Nginx

```dockerfile
# Dockerfile
FROM nginx:1.25-alpine

# Custom config
COPY nginx.conf /etc/nginx/nginx.conf
COPY conf.d/ /etc/nginx/conf.d/

# Static files
COPY public/ /usr/share/nginx/html/

EXPOSE 80 443

CMD ["nginx", "-g", "daemon off;"]
```

```yaml
# docker-compose.yml
version: '3.8'

services:
  nginx:
    image: nginx:1.25-alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./conf.d:/etc/nginx/conf.d:ro
      - ./certs:/etc/nginx/certs:ro
      - nginx_cache:/var/cache/nginx
    depends_on:
      - backend
    restart: unless-stopped
  
  backend:
    image: myapp:latest
    expose:
      - "8080"
    restart: unless-stopped

volumes:
  nginx_cache:
```

## Sonuç

Nginx, modern web altyapısının vazgeçilmez bir bileşenidir. Bu yazıda ele aldığımız konular:

1. **Reverse Proxy**: Backend uygulamalarının önünde güvenlik katmanı
2. **Load Balancing**: Yük dağıtımı ve yüksek erişilebilirlik
3. **Caching**: Proxy cache ve FastCGI cache ile performans artışı
4. **SSL/TLS**: Güvenli iletişim ve modern TLS yapılandırması
5. **Rate Limiting**: DDoS koruması ve kaynak yönetimi
6. **Security**: Güvenlik header'ları ve koruma mekanizmaları

Production ortamında Nginx kullanırken monitoring, log analizi ve düzenli güvenlik güncellemeleri kritik öneme sahiptir. Prometheus exporter, ELK stack entegrasyonu ve otomatik sertifika yenileme (Let's Encrypt) gibi araçlarla eksiksiz bir altyapı oluşturabilirsiniz.

## Kaynaklar

- [Nginx Documentation](https://nginx.org/en/docs/)
- [Nginx Admin Guide](https://docs.nginx.com/nginx/admin-guide/)
- [DigitalOcean Nginx Tutorials](https://www.digitalocean.com/community/tags/nginx)
- [Mozilla SSL Configuration Generator](https://ssl-config.mozilla.org/)
