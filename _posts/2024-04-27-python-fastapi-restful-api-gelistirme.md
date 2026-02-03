---
title: "Python FastAPI ile RESTful API Geliştirme: Modern ve Yüksek Performanslı Mimariler"
description: "FastAPI kullanarak modern, hızlı ve güvenli RESTful API'ler geliştirme rehberi. Async mimari, Pydantic validasyon ve performans optimizasyonu."
date: "2024-04-27"
categories: [Backend, Python]
tags: [fastapi, rest-api, python, async, backend, microservices, pydantic]
image:
  path: "/assets/img/posts/fastapi-microservices-architecture.png"
  alt: "FastAPI Microservices Mimarisi"
---

## Giriş: Modern Web Dünyasında Hız ve Verimlilik

Python ekosisteminde uzun süre Django'nun "her şey dahil" yapısı ve Flask'ın "minimalist" yaklaşımı hüküm sürdü. Ancak modern mikroservis mimarileri ve yüksek trafikli sistemler, daha hızlı, asenkron ve tip güvenliği sağlayan bir yapıya ihtiyaç duyuyordu. İşte FastAPI bu noktada, Starlette ve Pydantic'in gücünü birleştirerek sahneye çıktı. Bir senior mühendis için FastAPI sadece bir framework değil; asenkron programlamanın avantajlarını üretim ortamına güvenle taşıyan bir orkestrasyon aracıdır.

Bu yazıda, FastAPI'nin temel mimari avantajlarını, asenkron endpoint yönetimini ve bir API'yi "production-ready" hale getiren kritik bileşenleri inceleyeceğiz.

![API Tasarım Prensipleri](/assets/img/posts/api-jwt-authentication-flow.jpg)

## Core Mimari: Neden FastAPI Bu Kadar Hızlı?

FastAPI'nin performansının arkasında iki dev isim yatar:
1.  **Starlette:** Asenkron bir web framework'ü (ASGI).
2.  **Pydantic:** Veri validasyonu ve modelleme kütüphanesi.

### Asenkron Yapı (async/await)
Geleneksel web framework'lerinde bir istek geldiğinde, I/O işlemleri (veritabanı sorgusu, dosya okuma vb.) bitene kadar o "worker" meşgul edilir. FastAPI'de ise `async` anahtar kelimesiyle bu bekleme süreleri boşa çıkmaz; sistem diğer istekleri karşılamaya devam eder.

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/status")
async def get_status():
    # Asenkron bir işlem simülasyonu
    return {"status": "online", "message": "Yüksek performanslı API aktif."}
```

## Pydantic Gücü: Veri Validasyonu Artık Yük Değil

Pydantic, Python'ın "Type Hinting" (Tip Belirtme) özelliğini kullanarak veriyi hem doğrular hem de istediğiniz formata (JSON, Dict vb.) dönüştürür. Hatalı veri daha kontrolcüye (controller) ulaşmadan 422 hatasıyla geri döner.

![FastAPI Bağımlılık Entegrasyonu](/assets/img/posts/fastapi-architecture-diagram.png)

## Dependency Injection (Bağımlılık Enjeksiyonu)

FastAPI'yi rakiplerinden ayıran en güçlü özelliklerinden biri yerleşik `Depends` sistemidir. Veritabanı bağlantısı, kullanıcı yetkilendirme veya logging gibi işlemleri her fonksiyonda tekrar yazmak yerine merkezi bir yerden yönetebilirsiniz.

![Web Uygulama Mimarisi](/assets/img/posts/fastapi-microservices-architecture.png)

## Veritabanı Entegrasyonu: SQLAlchemy ve Async Verimliliği

Bir API'nin darboğaz (bottleneck) noktası genellikle veritabanıdır. FastAPI ile SQLAlchemy kullanırken, özellikle büyük verilerde asenkron sürücüleri (örn: `asyncpg` veya `aiosqlite`) tercih etmek hayati önem taşır.

**Senior Analizi:** Senkron bir veritabanı sürücüsü kullanıyorsanız, endpoint'inizi `async def` yerine sadece `def` ile tanımlamalısınız. Aksi takdirde FastAPI'nin "thread pool" yönetimi verimsiz çalışabilir.

```python
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import sessionmaker

DATABASE_URL = "postgresql+asyncpg://user:pass@localhost/dbname"
engine = create_async_engine(DATABASE_URL)
AsyncSessionLocal = sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)

async def get_db():
    async with AsyncSessionLocal() as session:
        yield session
```

## Güvenlik ve JWT: Yetkilendirme Stratejileri

RESTful API'lerde stateless (durumsuz) yapı esastır. FastAPI, OAuth2 standartlarını JWT (JSON Web Token) ile sarmallayarak basit ve güvenli bir kimlik doğrulama katmanı sunar.

### JWT Yaşam Döngüsü:
1. Kullanıcı kullanıcı adı/şifre ile giriş yapar.
2. Sunucu, gizli bir anahtar (SECRET_KEY) ile imzalanmış bir Token üretir.
3. Kullanıcı sonraki her istekte bu token'ı `Authorization: Bearer <token>` başlığıyla gönderir.

**Güvenlik Uyarısı:** JWT token'ları şifreli değildir, sadece imzalıdır. Token içerisine asla şifre veya kredi kartı gibi hassas veriler koymamalısınız.

![Veritabanı Tasarım Prensipleri](/assets/img/posts/database-relationships-one-to-many.png)

## Katmanlı Mimari (Layered Architecture)

Kodun sürdürülebilirliği için Controller (Endpoint), Service (İş Mantığı) ve Repository (Veri Erişimi) katmanlarını ayırmak senior seviyesinde bir zorunluluktur. Büyük projelerde her şeyi `main.py` içine yazmak, kodun "spagetti"ye dönüşmesine neden olur.

![FastAPI Worker Yapısı](/assets/img/posts/fastapi-microservices-architecture.png)

## Middleware: API Trafiğini Yönetmek

Middleware katmanı, her istek (request) ve her yanıt (response) arasında çalışan özelleştirilebilir bir yapıdır. CORS ayarlarından, request heradlarını loglamaya kadar pek çok işlem burada yapılır.

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_methods=["*"],
    allow_headers=["*"],
)
```

## Background Tasks: Uzun Süreli İşlemleri Arka Plana Atmak

Bazen bir endpoint içerisinde ağır bir işlem yapmanız gerekebilir (Örn: E-posta göndermek veya bir dosyayı işlemek). Kullanıcının bu işlemin bitmesini beklemesi kötü bir deneyimdir. FastAPI'nin `BackgroundTasks` sınıfı ile bu işleri arka plana atabilirsiniz.

```python
from fastapi import BackgroundTasks

def process_file_in_bg(filename: str):
    # Ağır dosya işleme mantığı
    pass

@app.post("/process/{filename}")
async def start_process(filename: str, tasks: BackgroundTasks):
    tasks.add_task(process_file_in_bg, filename)
    return {"message": "İşlem arka planda başlatıldı."}
```

## Test Odaklı Geliştirme (TDD): Pytest ile Kalite Güvencesi

Profesyonel bir API, her zaman testleri ile birlikte anılır. FastAPI, `TestClient` sayesinde Starlette'in gücünü kullanarak API endpoint'lerinizi saniyeler içinde test etmenizi sağlar.

**Senior İpucu:** Testlerde gerçek veritabanı yerine, RAM üzerinde çalışan bir `SQLite` veritabanı kullanarak test hızınızı artırabilirsiniz.

![Cloud Altyapı Mimarisi](/assets/img/posts/blockchain-relayer-architecture.png)

## Deployment: Docker ve Gunicorn/Uvicorn Kombinasyonu

FastAPI tek başına bir sunucu değildir; onu çalıştıracak bir ASGI sunucusuna (Uvicorn) ihtiyacı vardır. Production ortamında ise Uvicorn'u yöneten bir process manager (Gunicorn) kullanmak, uygulamanızın crash durumlarında otomatik ayağa kalkmasını sağlar.

## Dockerization: Konteynırlarda API Gücü

Modern deployment stratejilerinin vazgeçilmezi olan Docker, API'nizin her ortamda (local, staging, production) aynı şekilde çalışmasını garanti eder.

```dockerfile
FROM python:3.9-slim

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .
CMD ["gunicorn", "-w", "4", "-k", "uvicorn.workers.UvicornWorker", "main:app", "--bind", "0.0.0.0:8000"]
```

## Teknik Sözlük (Glossary)

- **ASGI (Asynchronous Server Gateway Interface):** Asenkron Python web sunucuları için standart arayüz.
- **Pydantic:** Python tip belirtimlerini kullanarak veri doğrulama yapan kütüphane.
- **Dependency Injection:** Bir sınıfın veya fonksiyonun ihtiyaç duyduğu nesnelerin dışarıdan sağlanması.
- **CORS (Cross-Origin Resource Sharing):** Farklı kökenlerden gelen kaynak paylaşımı güvenliği.
- **JWT (JSON Web Token):** Taraflar arasında güvenli veri iletimi sağlayan kompakt bir format.
- **ORM (Object-Relational Mapping):** Nesne tabanlı kodlar ile ilişkisel veritabanları arasındaki köprü.
- **Uvicorn:** Yıldırım hızında bir ASGI sunucusu.
- **Gunicorn:** Python WSGI/ASGI uygulamaları için popüler bir HTTP sunucusu yönetimi.

## Sonuç: FastAPI ile Geleceğe Hazır Olun

FastAPI, sadece hızıyla değil, sunduğu geliştirici deneyimi ve otomatik dokümantasyon gibi özellikleriyle modern web geliştirme dünyasında standart haline gelmiştir. Bir senior mühendis olarak, framework'ün sunduğu asenkron yetenekleri doğru kullanmak ve mimariyi katmanlı bir yapıda kurgulamak, projenizin başarısını belirleyecektir.

Geliştirdiğiniz bu güçlü API altyapısını blockchain dünyasıyla birleştirmek isterseniz [Web3.py ile Ethereum Etkileşimi](/web3-development/python/2024/05/30/web3py-ethereum-blockchain-etkilesimi/) rehberimize, mikroservis mimarileri hakkında daha fazla teknik detay için ise [Polygon Network Mimarisi Analizi](/blockchain/infrastructure/2024/06/05/polygon-network-architecture-deep-dive/) yazımıza göz atabilirsiniz.

## İleri Okuma ve Kaynaklar
- [FastAPI Resmi Dokümantasyonu](https://fastapi.tiangolo.com/)
- [Pydantic Dokümantasyonu](https://docs.pydantic.dev/)
- [SQLAlchemy Async Guide](https://docs.sqlalchemy.org/en/20/orm/extensions/asyncio.html)
- [Test-Driven Development with Python](https://www.obeythetestinggoat.com/)
