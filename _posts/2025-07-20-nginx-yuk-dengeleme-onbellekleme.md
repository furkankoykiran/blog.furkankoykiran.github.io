---
layout: post
title: "Nginx Konfigürasyon Sanatı: Load Balancing ve DDoS Koruması"
date: 2025-07-20 11:45:00 +0300
categories: [DevOps, System Design]
description: "Nginx worker optimizasyonu, Rate Limiting ile DDoS koruması ve Production-Ready Caching stratejileri üzerine kıdemli DevOps notları."
image: assets/img/posts/nginx-load-balancing-diagram.png
---

Nginx, modern web mimarisinin İsviçre Çakısı'dır. İster basit bir statik site sunucusu, ister Kubernetes önünde bir Ingress Controller, isterseniz de API Gateway olarak kullanın; config dosyasının derinliklerine inmeden gerçek gücünü göremezsiniz.

Genellikle `apt install nginx` deyip varsayılan ayarlarla bırakıyoruz. Ancak production trafiği altında o varsayılan ayarlar, darboğazın (bottleneck) ta kendisi olur.

## 1. İşlemciyle Dans: Worker Tuning

Nginx, "Event-Driven" mimariye sahiptir. Apache gibi her bağlantı için thread açmaz.

*   `worker_processes auto;`: Eski alışkanlıklarla buraya sayı yazmayın. `auto` diyerek Nginx'in CPU çekirdek sayısı kadar worker açmasını sağlayın.
*   `worker_connections 1024;`: Varsayılan değer çok düşüktür. Bir worker aynı anda kaç bağlantıyı tutsun? High-traffic sitelerde bunu `65535`'e kadar çekiyoruz. (Tabii `ulimit -n` işletim sistemi limitini de artırmak şartıyla).

```nginx
# /etc/nginx/nginx.conf
worker_processes auto;
events {
    worker_connections 20000;
    use epoll; # Linux için en performanslı event model
    multi_accept on; # Worker aynı anda tüm yeni bağlantıları kabul etsin
}
```

## 2. DDoS Kalkanı: Rate Limiting

Pahalı WAF (Web Application Firewall) çözümlerinden önce, Nginx'in kendi silahlarını kullanın. "Rate Limiting", servisinizi brute-force ve DDoS saldırılarından koruyan en etkili yöntemdir.

Mekanizma "Leaky Bucket" (Delik Kova) algoritmasıyla çalışır.

```nginx
http {
    # 10MB'lik hafıza alanı, saniyede 10 istek limiti (her IP için)
    limit_req_zone $binary_remote_addr zone=mylimit:10m rate=10r/s;

    server {
        location /login {
            # Limiti aşanları reddet, burst ile ani sıçramalara (5 istek) izin ver
            limit_req zone=mylimit burst=5 nodelay;
            
            # Limite takılanlara 503 yerine 429 Too Many Requests dön
            limit_req_status 429; 
        }
    }
}
```
**Pro Tip:** `$binary_remote_addr` kullanın. `$remote_addr` string olarak IP tutar (7-15 byte), binary hali ise her zaman 4 byte'tır. Milyonlarca IP için bu fark RAM'de devasa yer tutar.

## 3. Load Balancing Stratejileri

Round Robin (sırayla dağıt) varsayılan yöntemdir ama her zaman en iyisi değildir.

*   **Least Connections (`least_conn`):** En az aktif bağlantısı olan sunucuya gönderir. İşlem sürelerinin değişken olduğu (kimi request 10ms, kimi 5s sürüyorsa) durumlarda en adil dağıtımdır.
*   **IP Hash (`ip_hash`):** Aynı IP'den gelen hep aynı sunucuya gider. "Sticky Session" ihtiyacınız varsa (ve Redis kullanmıyorsanız) hayat kurtarır.

```nginx
upstream backend_cluster {
    least_conn; # Strateji seçimi
    server backend1.example.com;
    server backend2.example.com max_fails=3 fail_timeout=30s;
    server backend3.example.com backup; # Diğerleri ölürse devreye girer
}
```

## 4. Mikro-Caching: 1 Saniyenin Gücü

Dinamik bir API'niz var ve veritabanı sorguları yoruyor. İçeriğin 1 saniye eski olması sorun değilse, **Micro-Caching** uygulayın.

Sadece 1 saniyelik cache bile, saniyede 10.000 istek gelen bir serviste, Backend'e giden yükü neredeyse sıfıra indirir.

```nginx
proxy_cache_path /tmp/nginx_cache levels=1:2 keys_zone=my_cache:10m max_size=1g inactive=60m;

server {
    location /api/news {
        proxy_cache my_cache;
        proxy_cache_valid 200 1s; # Sadece 1 saniye cache!
        proxy_cache_use_stale error timeout updating; # Backend ölüyse eski veriyi ver (Resilience)
    }
}
```

`proxy_cache_use_stale` direktifi hayati önem taşır. Backend 500 hatası veriyorsa veya timeout'a düştüyse, Nginx "Elimde 1 dakika öncesinin verisi var, hata göstermektense bunu veririm" der. Kullanıcı hatayı hissetmez.

## 5. Güvenlik Hardening (Sıkılaştırma)

Nginx versiyonunuzu dünyaya ilan etmeyin.
`server_tokens off;` komutu, Response Header'daki `Server: nginx/1.18.0` bilgisini `Server: nginx` olarak gizler.

Ayrıca **Slowloris** saldırılarına karşı timeout ayarlarını kısın:
```nginx
client_body_timeout 10s;
client_header_timeout 10s;
```
Saldırganlar bağlantıyı açıp çok yavaş veri göndererek connection pool'unuzu doldurmaya çalışır. Onlara 60 saniye bekleme lüksü tanımayın.

## 6. Sahadan Notlar: Troubleshooting

**Hata: "502 Bad Gateway"**
*   **Anlamı:** Nginx, Backend'e (örneğin Node.js veya Python Gunicorn) bağlanamadı.
*   **Çözüm:** Backend servisinizin portunu ve ayakta olup olmadığını kontrol edin. Unix Socket kullanıyorsanız dosya izinlerine bakın.

**Hata: "504 Gateway Timeout"**
*   **Anlamı:** Backend'e bağlandı ama cevap belirtilen sürede gelmedi.
*   **Çözüm:** Veritabanı sorgunuz yavaşlamıştır. Nginx'te `proxy_read_timeout` artırılabilir ama asıl çözüm kodu optimize etmektir.

**Hata: "413 Request Entity Too Large"**
*   **Anlamı:** Kullanıcı büyük bir dosya (resim/video) yüklüyor.
*   **Çözüm:** `client_max_body_size 10M;` ile limiti artırın (Default 1MB'dır).

## 7. HTTP/2 ve Keepalive

HTTP/2, tek bir TCP bağlantısı üzerinden paralel istek (Multiplexing) atılmasını sağlar. SSL kullanıyorsanız (ki zorundasınız), `listen 443 ssl http2;` diyerek performansı %30 artırabilirsiniz.

Keepalive ise TCP Handshake maliyetini düşürür. Upstream bloğunda mutlaka kullanın:
```nginx
upstream backend {
    server 127.0.0.1:8080;
    keepalive 32; # 32 adet idle connection'ı açık tut
}
```

## 8. Production Checklist

*   [ ] **Gzip Açık mı?** `gzip on;` ve `gzip_types` ile JSON/CSS/JS sıkıştırılıyor mu?
*   [ ] **SSL Tuning Yapıldı mı?** `ssl_protocols TLSv1.2 TLSv1.3;` haricini kapattınız mı?
*   [ ] **Access Log Formatı:** JSON formatına çevirdiniz mi? (ELK stack ile parse etmek için şart).
*   [ ] **Config Test:** Reload yapmadan önce `nginx -t` komutunu alışkanlık haline getirdiniz mi?


## 9. Security Headers: Görünmez Kalkan

Sadece SSL kurmak yetmez. Browser'a "Bu siteyle nasıl konuşacağını" dikte etmelisiniz.

```nginx
# Clickjacking koruması: Sitenizin iframe içinde açılmasını engeller
add_header X-Frame-Options "SAMEORIGIN" always;

# MIME-sniffing koruması
add_header X-Content-Type-Options "nosniff" always;

# XSS Koruması
add_header X-XSS-Protection "1; mode=block" always;

# HSTS (HTTP Strict Transport Security): 
# Browser'a der ki: "Beni 1 yıl boyunca (31536000s) sadece HTTPS ile hatırla"
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
```

**Dikkat:** `add_header` direktifi, eğer alt bloklarda (örneğin `location`) başka bir `add_header` varsa, üst bloktakileri (mirası) **ezer ve siler**. Bu yüzden headerları en üst ana bloğa koymak veya her blokta tekrar etmek gerekir.

## 10. Buffer Tuning: RAM vs Disk

Nginx varsayılan olarak küçük bufferlar kullanır. Eğer birisi size 50KB'lık bir JSON POST ederse ve bufferınız 16KB ise, Nginx bu veriyi diske yazar. Disk I/O, RAM'den 1000 kat yavaştır.

```nginx
http {
    # Header boyutunu artır (Cookie vs için)
    client_header_buffer_size 1k;
    large_client_header_buffers 4 8k;

    # Body buffer: Form postları için. 
    # Bu değerin altındaki istekler RAM'de, üstündekiler diske (temp file) yazılır.
    client_body_buffer_size 128k; 
}
```

## 11. Gzip vs Brotli

Gzip standarttır, Brotli ise gelecektir (Google tarafından geliştirilmiştir). Metin tabanlı dosyalarda (HTML, CSS, JS) Gzip'e göre %20 daha iyi sıkıştırma sağlar.

Eğer Nginx'i derleme şansınız varsa (`nginx -V` ile bakın, modül var mı?), Brotli'yi mutlaka açın. Yoksa Gzip'i en verimli seviyede (`gzip_comp_level 5;`) tutun. 9 yapmak CPU'yu yakar, 1 yapmak bandwidth'i harcar. 5 altın orandır.

## Özetle

Nginx, sadece bir "web sunucusu" değil, mimarinizin kapı bekçisidir. Yükü dağıtır, saldırıyı savuşturur ve hataları gizler. Onu varsayılan ayarlarla kullanmak, Ferrari'ye tüp taktırmak gibidir. Kaputun altına girin ve motorun hakkını verin.

Bir sonraki kesintinizde loglara bakarken, bu ayarların ne kadar değerli olduğunu hatırlayacaksınız.

