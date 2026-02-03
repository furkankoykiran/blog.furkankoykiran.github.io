---
title: "Traefik v3: Docker Çağının Reverse Proxy Çözümü"
description: "Nginx config dosyaları arasında kaybolduysanız, Traefik'in dinamik dünyasına hoş geldiniz. Docker etiketleri ile otomatik keşif ve SSL yönetimi."
date: "2024-12-15 11:30:00 +0300"
categories: [DevOps, Network, Docker]
tags: [traefik, reverse-proxy, docker, ssl, load-balancing]
image:
  path: /assets/img/posts/traefik-architecture-diagram.png
  alt: "Traefik Dynamic Configuration Architecture"
---

Modern Docker altyapılarında Nginx kullanmak, statik yapısı nedeniyle "Blue/Green Deployment" gibi süreçlerde operasyonel yük yaratabilir. Konteynerler dinamikken, statik konfigürasyon dosyalarıyla uğraşmak hataya açıktır.

Traefik, "Cloud Native" bir Reverse Proxy'dir. En büyük özelliği **Dinamik Konfigürasyon**dur.
Konteyneri kaldırdığınız an Traefik onu fark eder, trafiği yönlendirir ve SSL sertifikasını otomatik alır.
Go ile yazılmıştır, tek bir binary'dir ve dependency gerektirmez.

![Traefik Architecture](/assets/img/posts/traefik-architecture-diagram.png)
*Provider (Docker) -> Traefik Core -> EntryPoint -> Router -> Middleware -> Service.*

## 1. Static vs Dynamic Config

Traefik'in çalışma mantığını anlamak için bu ikilimi çözmeniz gerekir. Yeni başlayanların en çok kafasını karıştıran nokta burasıdır.
-   **Static Config:** Traefik'in açılış (startup) ayarlarıdır. (Hangi portu dinleyecek? Log seviyesi ne olacak? Dashboard açık mı? Provider olarak Docker mı kullanacak?). Genelde `traefik.yml` dosyasında durur ve Traefik yeniden başlatılmadan değişmez.
-   **Dynamic Config:** Servislerinizin (Routing) ayarlarıdır. (Hangi domain nereye gidecek? Hangi middleware çalışacak? Yük dengeleme nasıl olacak?). Bu ayarlar Docker Label'larından veya harici bir dosyadan (`dynamic_conf.yml`) okunur ve **Hot Reload** olur.

```yaml
# traefik.yml (Static)
entryPoints:
  web:
    address: ":80"
    # HTTP'yi HTTPS'e zorla
    http:
      redirections:
        entryPoint:
          to: websecure
          scheme: https
  websecure:
    address: ":443"

api:
  dashboard: true # Güvenlik uyarısı: Production'da bunu şifreleyin!

providers:
  docker:
    endpoint: "unix:///var/run/docker.sock"
    exposedByDefault: false # Her konteyner otomatik açılmasın, ben seçeyim
```

## 2. Docker Labels: Büyü Burada Başlıyor

Artık config dosyası yok. Her şey `docker-compose.yml` içinde, servisin yanında duruyor.
Servisinizi tanımlarken ona etiketler (Labels) yapıştırıyorsunuz:

```yaml
services:
  whoami:
    image: traefik/whoami
    labels:
      - "traefik.enable=true" # Bu servisi Traefik görsün
      # Router Tanımı: Hangi domain?
      - "traefik.http.routers.whoami.rule=Host(`whoami.example.com`)"
      # Entrypoint: HTTPS portunu dinle
      - "traefik.http.routers.whoami.entrypoints=websecure"
      # SSL Resolver: Sertifikayı kim üretecek?
      - "traefik.http.routers.whoami.tls.certresolver=myresolver"
      # Service Portu: Konteynerin iç portu
      - "traefik.http.services.whoami.loadbalancer.server.port=80"
```

Bu etiketleri gören Traefik:
1.  Docker Socket üzerinden event'i alır: "Yeni bir konteyner başladı!"
2.  `whoami.example.com` domainini yönlendirme tablosuna ekler.
3.  Let's Encrypt'e gidip sertifika ister (DNS veya HTTP Challenge ile).
4.  Trafiği konteynerin 80 portuna yönlendirir.

## 3. Middlewares: Trafiği Manipüle Etmek

Traefik'te bir istek servise ulaşmadan önce "Middleware" zincirinden geçer.
Nginx'teki `auth_basic`, `gzip`, `limit_req` gibi modüllerin karşılığıdır.
Middleware'ler tekrar kullanılabilir parçalardır. Bir kere tanımla, on serviste kullan.

**Örnek: Dashboard Güvenliği (Basic Auth + IP Whitelist)**
Traefik Dashboard'u dünyaya açmak istiyorsanız, mutlaka şifrelemelisiniz.

```yaml
labels:
  # Middleware Tanımı: Basic Auth
  - "traefik.http.middlewares.auth.basicauth.users=admin:$$apr1$$..."
  # Middleware Tanımı: IP Whitelist (Sadece VPN IP'm)
  - "traefik.http.middlewares.vpn-only.ipwhitelist.sourcerange=10.0.0.0/24"
  
  # Zincirleme Kullanım (Chain)
  - "traefik.http.routers.dashboard.middlewares=auth,vpn-only"
```

Aynı şekilde, API'nızı korumak için Rate Limit ekleyebilirsiniz:

```yaml
labels:
  # Saniyede ortalama 100 istek, anlık patlama (burst) 50
  - "traefik.http.middlewares.ratelimit.ratelimit.average=100"
  - "traefik.http.middlewares.ratelimit.ratelimit.burst=50"
```
Bu sistem, uygulamanıza kod yazmadan (Decorator vs kullanmadan) altyapı seviyesinde koruma sağlar. Uygulamanız yükü hissetmez bile, Traefik kapıda karşılar.

![Middleware Chain](/assets/img/posts/reverse-proxy-routing-diagram.png)
*Entrypoint -> Router -> Middleware 1 -> Middleware 2 -> Service.*

## 4. Yük Dengeleme ve Canary Deployments

Traefik, varsayılan olarak "Round Robin" mantığıyla çalışır. Eğer `whoami` servisinden 3 tane replica (`deploy: replicas: 3`) açarsanız, trafiği eşit dağıtır.
Ama daha gelişmiş senaryolar da mümkündür.

**Weighted Round Robin (Canary Deployment):**
Trafiğin %90'ını v1 sürümüne, %10'unu v2 sürümüne göndermek ister misiniz?
```yaml
labels:
  - "traefik.http.services.app-v1.loadbalancer.weighted.weight=90"
  - "traefik.http.services.app-v2.loadbalancer.weighted.weight=10"
```
Bu özellik sayesinde yeni sürümü production'da küçük bir kitleye test edebilirsiniz.

**Sticky Sessions (Session Affinity):**
Eğer uygulamanız stateful ise (kullanıcı session'ı RAM'de tutuyorsa), kullanıcının hep aynı konteynere gitmesi gerekir.
```yaml
labels:
  - "traefik.http.services.app.loadbalancer.sticky.cookie=true"
  - "traefik.http.services.app.loadbalancer.sticky.cookie.name=srv_id"
```
Bu konfigürasyon ile Traefik kullanıcıya bir cookie atar ve sonraki isteklerde onu hep aynı "Backend Server"a yönlendirir.

## 5. SSL Yönetimi: ACME (Let's Encrypt)

Nginx'te `certbot` kurup cron job ayarlamakla uğraştığınız günleri unutun.
"Sertifika süresi dolmuş" maili görüp paniklediğiniz günleri de unutun.
Traefik içinde ACME client gömülü gelir.
Tek yapmanız gereken `traefik.yml` içinde bir "Certificate Resolver" tanımlamaktır.

```yaml
certificatesResolvers:
  myresolver:
    acme:
      email: your@email.com
      storage: acme.json # Sertifikaları burada saklar (Volume yapın!)
      # opsiyon 1: HTTP Challenge (80 portu açık olmalı)
      httpChallenge:
        entryPoint: web
      # opsiyon 2: DNS Challenge (Wildcard SSL için gerekli)
      # dnsChallenge:
      #   provider: cloudflare
```

Traefik, sertifikanın süresinin dolmasına 30 gün kala otomatik yeniler.
Diyelim ki 100 tane subdomaininiz var (`api.site.com`, `admin.site.com`, `blog.site.com`...).
Hepsine `tls.certresolver=myresolver` etiketini verin ve arkanıza yaslanın.
**Önemli:** `acme.json` dosyasını `volume` olarak saklamayı unutmayın. Konteyner ölürse sertifikalar gitmesin, yoksa Let's Encrypt sizi banlar (Rate Limit).

![Let's Encrypt Flow](/assets/img/posts/traefik-letsencrypt-ssl.png)
*HTTP Challenge veya DNS Challenge ile otomatik sertifika üretimi.*

## 6. Sık Karşılaşılan Sorunlar (Troubleshooting)

Traefik kurarken karşılaşacağınız muhtemel hatalar ve çözümleri:

**1. "404 Not Found" Hatası:**
Router'ınız doğru çalışmıyor veya kural eşleşmiyor olabilir.
-   **Çözüm:** Dashboard'u açın! Router'ın statüsü "Success" mi? Servis (Backend) IP'si görünüyor mu?
-   `Host` kuralını kontrol edin. `www` ile `non-www` fark edebilir.

**2. "Gateway Timeout" (504):**
Traefik servise ulaşamıyor.
-   **Çözüm:** Traefik ve uygulamanızın **AYNI** Docker network'ünde olduğundan emin olun.
-   `docker network inspect traefik-net` ile kontrol edin.

**3. "Internal Server Error" (SSL Hatası):**
Sertifika oluşturulamadı.
-   **Çözüm:** `traefik` loglarına bakın (`docker logs traefik`).
-   Let's Encrypt limitlerine takılmış olabilirsiniz. Test aşamasında `caServer: "https://acme-staging-v02.api.letsencrypt.org/directory"` kullanın.

## 7. Dashboard ve Gözlemlenebilirlik

"Hangi servis çalışıyor? Hangi router hata veriyor? Kim nereye yönleniyor?"
Traefik'in 8080 portunda çalışan harika bir Dashboard'u vardır.
Tüm altyapınızı, router'larınızı, servislerinizi görsel olarak size sunar. Debug yapmak için birebirdir.

Senior bir tavsiye: Dashboard'u asla `insecure: true` modunda prodüksiyonda açmayın.
Onu da bir router arkasına alın (`traefik.http.routers.api.service=api@internal`) ve güçlü bir şifre ile koruyun.
Ayrıca Traefik, Prometheus metriği de basar. Grafana ile bağlayıp "Saniyede kaç istek geliyor?", "Hata oranı ne?", "Latency (Gecikme) süresi ne kadar?" gibi metrikleri izleyebilirsiniz.
Prometheus ile Autoscaling (HPA) bile tetikleyebilirsiniz! "İstek sayısı 1000'i geçerse yeni pod aç" diyebilirsiniz.

![Traefik Dashboard](/assets/img/posts/traefik-docker-load-balancer.png)
*Router, Service ve Middleware durumlarını gösteren yönetim paneli.*

## Sonuç

Traefik, konteyner dünyasının İsviçre Çakısıdır. Kurulumu Nginx'e göre farklı hissettirebilir ancak kazandırdığı operasyonel hız paha biçilemez.
Altyapınız büyüdükçe, konfigürasyonunuz küçülür. Servisler kendi ayarlarını (Labels) taşır.
Nginx hala statik dosya sunumu için kraldır, ama dinamik yönlendirme (Service Discovery) için Traefik modern standardı belirler.
Eğer hala `nginx -s reload` yazıyorsanız, otomasyona geçme vaktiniz gelmiştir.
