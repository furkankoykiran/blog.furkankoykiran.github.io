---
title: "Docker ile Python Uygulaması Containerization"
description: "Docker ile Python uygulaması containerization rehberi. Multi-stage builds, Docker Compose, Kubernetes deployment ve production best practices."
date: "2024-10-14 09:30:00 +0300"
categories: [DevOps, Python]
tags: [docker, python, containerization, devops, deployment, microservices, kubernetes]
image:
  path: /assets/img/posts/docker-python-containerization-architecture.png
  alt: "Docker ile Python Uygulaması Containerization Mimarisi"
---

## Giriş: "Works on my Machine" Devrinin Sonu

Yazılım geliştirme süreçlerinde karşılaşılan en klasik sorunlardan biri, uygulamanın bir geliştiricinin bilgisayarında sorunsuz çalışırken sunucuda veya bir başka geliştiricinin ortamında patlamasıdır. Python dünyasında bu durum; farklı `pip` versiyonları, çakışan `virtualenv` yapıları veya eksik sistem kütüphaneleri nedeniyle daha da karmaşık bir hal alabilir. 

Bir senior mühendis olarak, Docker'ı sadece bir "paketleme aracı" değil, uygulamanın yaşam döngüsünü standardize eden bir "kontrat" olarak görüyorum. Docker sayesinde, kodumuzun çalışacağı işletim sistemi çekirdeğinden, içindeki çevre değişkenlerine kadar her şeyi kod olarak tanımlayabiliyoruz. 

Bu mimariyi [Python ile Otomatik Yatırım Sistemleri]({% post_url 2024-09-12-building-automated-trading-systems-python %}) geliştirirken nasıl kullandığımızı görebilirsiniz. Bu yazıda, Python projelerinde sıradan bir Docker kurulumunun ötesine geçip, production ortamlarında güvenle koşan yapılar kurmayı inceleyeceğiz.

![Docker ve Python Mimarisi](/assets/img/posts/docker-python-containerization-architecture.png)

## Temel Dockerfile Prensipleri ve Senior Dokunuşları

Çoğu tutorial'da `FROM python:3.9` ile başlayan ve tüm dosyaları tek seferde kopyalayan Dockerfile'lar görürsünüz. Ancak profesyonel bir projede bu yaklaşım hem güvenlik hem de hız açısından sınıfta kalır.

**Neden `slim` imajları tercih etmeliyiz?**
`python:3.x` imajları genellikle Debian tabanlıdır ve içinde yüzlerce gereksiz kütüphane barındırır. Bu da imaj boyutunun 1GB'a yaklaşmasına neden olur. `slim` varyasyonları ise sadece Python'ın çalışması için gereken minimum araçları içerir. 

```dockerfile
# Kötü Uygulama Örneği
FROM python:3.11
COPY . /app
RUN pip install -r requirements.txt
CMD ["python", "app.py"]

# İyi (Senior) Uygulama Örneği
FROM python:3.11-slim
WORKDIR /app
# Bağımlılıkları önbelleğe almak için önce kopyalıyoruz
COPY requirements.txt .
RUN pip install --no-cache-dir --upgrade pip && \
    pip install --no-cache-dir -r requirements.txt
# Uygulama kodu en son kopyalanmalı
COPY . .
# Güvenlik prensibi: Root olmayan kullanıcı
RUN useradd -m appuser && chown -R appuser /app
USER appuser
# Sinyal yönetimi için ENTRYPOINT ve CMD ayrımı
ENTRYPOINT ["gunicorn"]
CMD ["--bind", "0.0.0.0:5000", "--workers", "4", "app:app"]
```

## Multi-Stage Builds: İmaj Boyutu ve Güvenlik

Python uygulamalarında özellikle `gcc` veya `libpq-dev` gibi derleme araçlarına ihtiyaç duyduğumuzda imaj boyutu şişer. Multi-stage build tekniği ile bu araçları "builder" aşamasında kullanıp, final imajına sadece derlenmiş çıktıları taşıyabiliriz.

![Multi-Stage Build Katmanları](/assets/img/posts/docker-multi-stage-build-layers.png)

### Örnek Bir Multi-Stage Yapısı:

```dockerfile
# Faz 1: Builder
FROM python:3.11-slim as builder
RUN apt-get update && apt-get install -y gcc libpq-dev
RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Faz 2: Runtime
FROM python:3.11-slim
RUN apt-get update && apt-get install -y libpq5 && rm -rf /var/lib/apt/lists/*
COPY --from=builder /opt/venv /opt/venv
WORKDIR /app
COPY . .
ENV PATH="/opt/venv/bin:$PATH"
USER 1000
CMD ["python", "main.py"]
```

Bu yöntemle 800MB'lık bir imajı 150MB seviyelerine indirebilirsiniz. Bu sadece disk alanından tasarruf sağlamaz, aynı zamanda CI/CD süreçlerindeki push/pull hızını artırır.

## Docker Compose ile Mikroservis Orkestrasyonu

Modern uygulamalar nadiren tek başına çalışır. Yanında bir PostgreSQL veritabanı, Redis veya bir worker bulunur. Tüm bunları yerel bilgisayarınızda kurup konfigüre etmek yerine Docker Compose kullanmak büyük bir konfor sağlar.

![Mikroservis Mimarisi](/assets/img/posts/docker-compose-microservices-architecture.png)

### Gelişmiş Docker Compose Konfigürasyonu:

```yaml
version: '3.8'
services:
  web:
    build: .
    ports: ["5000:5000"]
    environment:
      - ENVIRONMENT=production
      - DATABASE_URL=postgres://user:pass@db:5432/mydb
    depends_on:
      db:
        condition: service_healthy
  db:
    image: postgres:15-alpine
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
```

## Production Ortamında Kritik Başarı Faktörleri

Uygulamanızı production'a çıkarken şu kontrol listesini mutlaka gözden geçirmelisiniz:

1.  **Gereksiz Ayrıcalıklardan Kaçının:** Konteyner içindeki işlemler asla `root` kullanıcısı ile çalışmamalıdır. Bu, sızma girişimlerinde saldırganın yetkilerini kısıtlayan en temel güvenlik önlemidir.
2.  **Sinyal Yönetimi ve Graceful Shutdown:** Python scriptlerini CMD ile çalıştırmak yerine sinyalleri yakalayan bir process manager kullanın. Bu, veritabanı bağlantılarının ve açık dosya soketlerinin düzgünce kapanmasını sağlar.
3.  **Kaynak Sınırları:** Konteynerlerin kontrolsüzce CPU ve RAM tüketmesini engellemek için kaynak sınırları (limits) tanımlayın. Aksi halde tek bir hafıza sızıntısı tüm ana makineyi (host) kilitleyebilir.
4.  **Logging:** Logları dosya yerine `stdout` akışına yönlendirin. Docker bu akışları yakalayıp merkezi log toplama sistemlerine (ELK Stack gibi) aktarmak üzere son derece verimli bir şekilde tasarlanmıştır.

![Deployment İş Akışı](/assets/img/posts/docker-deployment-workflow-lifecycle.png)

## Healthcheck Mekanizmaları Neden Önemli?

Bir konteynerin sadece "çalışıyor" olması (running status), uygulamanın sağlıklı olduğu anlamına gelmez. Uygulama içeride kilitlenmiş veya veritabanına bağlanamıyor olabilir.

**Dockerfile İçinde Healthcheck Kullanımı:**
```dockerfile
HEALTHCHECK --interval=30s --timeout=3s \
  CMD curl -f http://localhost:5000/health || exit 1
```
Bu komut sayesinde, Docker veya Kubernetes uygulamanın gerçekten cevap verip vermediğini bilir. Eğer uygulama "unhealthy" duruma düşerse, orkestrasyon araçları konteyneri otomatik olarak yeniden başlatabilir veya trafik göndermeyi bırakabilir.

## İsimlendirme ve Tagleme Stratejileri

Profesyonel projelerde imaj tagleme rastgele yapılmamalıdır. Genellikle `v1.2.3` gibi semantik versiyonlama veya `git commmit hash` kullanımı tercih edilir.

**Mühendislik Notu:** `latest` tag'ini asla production'da kullanmayın. `latest` her zaman en güncel olanı değil, en son push edileni ifade eder ve bu da production ortamında hangi kodun çalıştığını takip etmenizi imkansız kılıp rollback süreçlerini riske atar. Her zaman tahmin edilebilir ve izlenebilir bir versiyon numarası sistemi kurun.

## Teknik Sözlük (Glossary)

- **Build Context:** Docker build komutu çalıştırıldığında Docker daemon'a gönderilen dosya ve klasörler kümesi. Standardı `.dockerignore` ile filtrelemektir.
- **Layering:** Docker imajlarının üst üste binen dosya sistemi değişikliklerinden oluşmasıdır. Her komut (RUN, COPY) yeni bir katman yaratır.
- **Image Manifest:** İmajın içindeki katmanların ve konfigürasyonun JSON formatındaki resmi tanımı ve hash değeridir.
- **Container Runtime:** Konteynerlerin düşük seviyede (kernel seviyesi) çalışmasını sağlayan yazılımdır (runc veya containerd gibi).
- **Orchestration:** Konteynerlerin oto-ölçekleme, ağ yönetimi ve dağıtımını sağlayan sistemlerin genel adıdır (Kubernetes gibi).

## CI/CD Entegrasyonu ve Otomatik Build Süreçleri

Profesyonel bir yazılım yaşam döngüsünde (SDLC), Docker imajları manuel olarak build edilmemelidir. GitHub Actions veya GitLab CI gibi araçlarla bu süreç otomatize edilmelidir.

**Örnek CI Akışı:**
```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Build and Push
        run: |
          docker build -t my-app:${{ github.sha }} .
          docker push my-registry/my-app:${{ github.sha }}
```
Bu akış, her commit için geri dönülebilir (rollback-able) bir artifact oluşturmanızı sağlayarak hata anında müdahale kapasitenizi artırır.

## Sonuç: Konteyner Bir Seçenek Değil, Standarttır

Docker kullanmak sadece bir devops görevi değil, yazılım kalitesini artıran bir mühendislik disiplinidir. Python uygulamalarınızı konteynerize ederken gösterdiğiniz özen, production ortamında yaşayacağınız uykusuz geceleri minimize edecektir. 

Unutmayın ki en iyi Docker imajı; en küçük, en güvenli ve en hızlı ayağa kalkan imajdır. Bu prensipleri projelerinize uyguladığınızda, ekibinizin deployment süreçlerine olan güveninin nasıl arttığını bizzat gözlemleyeceksiniz. Geleceğin bulut bilişim dünyasında konteynerleşmiş bir yapıya sahip olmak, sizi her türlü altyapı değişikliğine karşı esnek kılacaktır.

## Son Mühendislik Notları ve Pro-Tip

- **Güvenlik:** `.dockerignore` dosyanızı oluşturmayı ve içine `.env` ve `.git` dizinlerini eklemeyi asla unutmayın. Bu sızıntıları önleyen en temel kuraldır.
- **Performans:** Uygulamanızın başlama süresini (boot time) optimize edin. Konteyner dünyasında hız, sistemin yanıt verme kabiliyeti ve güvenlik için kritiktir.
- **Disiplin:** Her zaman Dockerfile'ınızı projenin kalbi olarak görün ve mutlaka kod review süreçlerine dahil edin.
- **Pro-Tip:** Imaj build ederken `--build-arg` kullanarak build zamanında değişkenler geçebilir ve imajınızın metaverisine (label) versiyon bilgilerini ekleyebilirsiniz.

Kendi projelerimde Docker'ı merkeze koyduğumdan beri, bağımlılık çakışmalarıyla uğraşmak yerine yeni özellikler geliştirmeye çok daha fazla vakit ayırabiliyorum. Bu dönüşüm, her modern yazılımcının geçmesi gereken bir olgunlaşma evresidir. Terminal başındaki her mühendis için Docker, cebindeki en keskin ve en güvenilir isviçre çakısıdır.
