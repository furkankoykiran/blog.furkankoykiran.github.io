---
title: "Nginx: Sadece Load Balancer Değil, Web'in İsviçre Çakısı"
description: "Apache'nin hantallığından sıkıldınız mı? Nginx ile statik dosya sunumu, Gzip sıkıştırma, güvenlik headerları ve performans tuning."
date: "2025-03-08 10:00:00 +0300"
categories: [DevOps, Web Server, Linux]
tags: [nginx, web-server, performance, security, caching]
image:
  path: /assets/img/posts/nginx-reverse-proxy-architecture.png
  alt: "Nginx Event-Driven Architecture"
---

İnternetin %30'undan fazlası Nginx üzerinde çalışıyor.
Çoğumuz onu "Reverse Proxy" (uygulamanın önündeki kapı) olarak tanısak da, özünde o dünyanın en hızlı Web Sunucularından (Web Server) biridir.
React/Vue uygulamanızın `dist` klasörünü sunmak, resim/CSS dosyalarını servis etmek için Python veya Node.js kullanmak CPU israfıdır.
Bu iş Nginx'in uzmanlık alanıdır.
Bu yazıda, Nginx'i "sadece yönlendirici" olmaktan çıkarıp, sitenizi hızlandıran ve koruyan bir kaleye dönüştüreceğiz.

![Nginx Architecture](/assets/img/posts/nginx-reverse-proxy-architecture.png)
*Master Process ve Worker Process'lerin asenkron çalışma yapısı.*

## 1. Statik Dosya Sunumu ve Önbellekleme

Bir tarayıcı sitenize girdiğinde logo.png, style.css gibi dosyaları ister.
Nginx `sendfile on;` direktifi ile bu dosyaları kernel seviyesinde (zero-copy) ağ kartına kopyalar. CPU'ya neredeyse hiç uğramaz.

```nginx
server {
    listen 80;
    server_name example.com;
    root /var/www/html;

    # Statik dosyalar için Browser Cache
    location ~* \.(jpg|jpeg|png|gif|ico|css|js)$ {
        expires 30d; # 30 gün boyunca bir daha sorma
        add_header Cache-Control "public, no-transform";
    }
}
```
`expires` ayarı sayesinde tarayıcı dosyayı diskine kaydeder ve sunucunuza bir daha istek atmaz. Siteniz "çak diye" açılır.

## 2. Load Balancing ve Upstream Modülü

Nginx'i sadece tek bir sunucu önünde kullanıyorsanız potansiyelini harcıyorsunuz. Trafiği birden fazla backend sunucusuna dağıtarak (Load Balancing) sisteminizi ölçekleyebilirsiniz.

```nginx
upstream backend_servers {
    least_conn; # En az bağlantısı olan sunucuya git (Adil dağıtım)
    server 10.0.0.1:8000 weight=3; # Bu sunucuya 3 kat daha fazla yük ver
    server 10.0.0.2:8000;
    server 10.0.0.3:8000 backup; # Diğerleri çökerse devreye gir
}

server {
    location / {
        proxy_pass http://backend_servers;
    }
}
```
**Algoritma Seçimi:** Varsayılan `Round Robin`'dir. Ancak uzun süren işlemleriniz varsa `least_conn` daha stabil çalışır. IP adresini sabitlemek isterseniz (Session Sticky) `ip_hash` kullanın.

## 3. Micro-Caching: Dinamik İçeriği Statikleştirin

Veritabanından gelen bir sorgu 200ms sürüyor. Saniyede 100 istek gelirse veritabanı çöker.
Peki bu veri her saniye değişiyor mu? Hayır. O zaman Nginx'e "bunu 1 saniye hafızanda tut" derseniz ne olur?

```nginx
# Cache yolunu tanımla: /var/cache/nginx altında, 10MB keys_zone, 60dk pasiflik süresi
proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=my_micro_cache:10m max_size=1g inactive=60m;

server {
    location /api/news {
        proxy_cache my_micro_cache;
        proxy_cache_valid 200 1s; # Sadece 1 saniye cache!
        proxy_cache_use_stale error timeout updating; # Backend çökerse eski veriyi göster
        proxy_pass http://backend_servers;
    }
}
```
Bu **"1 saniyelik cache"**, ani trafik patlamalarında (Slashdot Effect) backend sunucunuzu ipten alır. Saniyede 10.000 istek gelse bile backend'e sadece 1 istek gider.


## 2. Gzip ve Brotli Sıkıştırma

Metin dosyalarını (HTML, CSS, JSON) sıkıştırarak göndermek, bant genişliğinden %70 tasarruf sağlar.
Nginx'te Gzip standarttır. Ama daha iyisi var: **Brotli** (Google'ın algoritması).

```nginx
# nginx.conf
gzip on;
gzip_types text/plain text/css application/json application/javascript;
gzip_min_length 1000; # 1KB'dan küçükse uğraşma
```
Bu ayar, 100KB'lık JavaScript dosyanızı 30KB'a düşürür. Mobil kullanıcılar size dua eder.

## 3. Güvenlik Headerları (Harden Nginx)

Varsayılan ayarlarla Nginx güvenli değildir. `Server: nginx/1.18.0` gibi versiyon bilgilerini kapatmalısınız (`server_tokens off;`).
Daha da önemlisi, modern güvenlik headerlarını eklemelisiniz.

```nginx
add_header X-Frame-Options "SAMEORIGIN"; # Clickjacking koruması
add_header X-XSS-Protection "1; mode=block"; # XSS filtresi
add_header X-Content-Type-Options "nosniff";
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always; # Sadece HTTPS
```
Bu satırlar, sitenizi SecurityHeaders.com testinden "A+" almanızı sağlayacak temel taşlardır.

![HTTP Request Lifecycle](/assets/img/posts/nginx-virtual-host-structure.png)
*İstemci -> Nginx -> Uygulama Sunucusu arasındaki istek yaşam döngüsü.*

## 4. Performans Tuning: Worker Processes

Nginx kaç işlemci kullanmalı?
Genelde `auto` ayarı iyidir ama yüksek trafikli sitelerde ince ayar gerekir.
`worker_processes auto;`: CPU çekirdek sayısı kadar worker açar.
`worker_connections 1024;`: Her worker aynı anda kaç bağlantı yönetebilir?
Toplam kapasite = Worker * Connections.
Dosya açıklama limitini (File Descriptor Limit) işletim sisteminde de artırmayı unutmayın (`ulimit -n`).

## 5. Rate Limiting: Saldırıları Yavaşlatın

Önceki yazımda (API Throttling) uygulama seviyesinde limit koymuştuk.
Nginx ile bunu en kapıda yapabiliriz.

```nginx
# IP başına saniyede 10 istek limiti tanımla (10MB hafıza)
limit_req_zone $binary_remote_addr zone=mylimit:10m rate=10r/s;

server {
    location /login {
        # Limiti uygula, anlık 5 isteğe kadar izin ver (Burst)
        limit_req zone=mylimit burst=5 nodelay;
    }
}
```
Brute-force saldırıları için harika bir ilk savunma hattıdır.

![Security Headers](/assets/img/posts/nginx-ssl-tls-certificate-flow.png)
*HSTS, CSP ve diğer güvenlik başlıklarının browser üzerindeki etkisi.*

## 7. Observability: Logları Maymunun Okuyabileceği Hale Getirin

Varsayılan Nginx logları (`access.log`) düz metindir ve parse etmesi zordur (`grep` ile boğuşursunuz).
Modern dünyada loglar **JSON** olmalıdır ki ELK Stack (Elasticsearch), Graylog veya Datadog gibi sistemler bunları direk yutabilsin.

```nginx
log_format json_analytics escape=json
  '{'
    '"time_local": "$time_local",'
    '"remote_addr": "$remote_addr",'
    '"request_uri": "$request_uri",'
    '"status": "$status",'
    '"request_time": "$request_time",' # Backend kaç saniyede cevap döndü? (En kritiği!)
    '"upstream_response_time": "$upstream_response_time",'
    '"user_agent": "$http_user_agent"'
  '}';

access_log /var/log/nginx/access_json.log json_analytics;
```
Bu sayede Kibana'da **"Response time > 2s olan istekler"** diye sorgu atıp, darboğazı milisaniyeler içinde bulabilirsiniz.

## 8. Hardening: Buffer Overflow Koruması

Bir saldırgan HTTP header'ına 10MB veri koyup gönderirse ne olur? Sunucu hafızasını şişirir (Buffer Overflow).
Nginx'in varsayılan limitlerini sıkılaştırarak bu kapıyı kapatın:

```nginx
client_body_buffer_size 10K;
client_header_buffer_size 1k;
client_max_body_size 8m; # Dosya yükleme limiti
large_client_header_buffers 2 1k;

# Slowloris saldırıları için timeoutları kısın
client_body_timeout 12;
client_header_timeout 12;
keepalive_timeout 15;
send_timeout 10;
```
Bu ayarlar, "ağır kanlı" saldırganların bağlantılarını erkenden keser.


## Sonuç

Nginx, Apache gibi "her istek için bir process" açmaz. Asenkron (Event-Driven) mimarisi sayesinde azıcık RAM ile on binlerce bağlantıyı yönetebilir.
Konfigürasyon dili (Config Syntax) biraz korkutucu gelebilir. Süslü parantezler, noktalı virgüller... Bir hata yaparsanız servis başlamaz.
Tavsiyem: `nginx -t` komutunu parmak hafızanıza kazıyın. Konfigürasyonu değiştirdikten sonra test etmeden asla reload yapmayın.
Doğru yapılandırılmış bir Nginx, uygulamanızın performansını ve güvenliğini arttıran en ucuz yatırımdır.
Onu sadece "Pass-Through" bir proxy olarak kullanmak, elindeki Ferrari ile markete gitmeye benzer. Yeteneklerini keşfedin.
