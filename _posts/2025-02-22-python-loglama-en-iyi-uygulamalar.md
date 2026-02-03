---
title: "Python Loglama: 'print' Kullanmayı Bırakın"
description: "Production ortamında debug yapmak samanlıkta iğne aramaya benzemesin. Structured Logging, ELK Stack ve Correlation ID ile profesyonel izleme."
date: "2025-02-22 13:00:00 +0300"
categories: [Backend, DevOps, Observability]
tags: [logging, python, elk-stack, structlog, observability]
image:
  path: /assets/img/posts/elk-stack-architecture.png
  alt: "ELK Stack Logging Architecture"
---


Bir yazılımcının "Senior" oluşu, kod yazma hızından değil, yazdığı kodun **gözlemlenebilirliğinden (Observability)** anlaşılır.
Junior geliştirici `print("Hata burda 1")` yazar. Senior geliştirici ise, gece 3'te sistemi ayağa kaldıracak structured logları kurgular.

Localhost'ta her şey kolaydır. Ama 50 mikroservisin koştuğu, saniyede 10.000 isteğin aktığı bir Kubernetes cluster'ında, kaybolan bir isteği bulmak samanlıkta iğne aramaktan zordur.
Eğer loglarınız sadece metin (text) tabanlıysa, geçmiş olsun. `grep` ile boğuşarak saatlerinizi harcarsınız.

Bu yazıda, Python'un standart `logging` modülünü bir kenara bırakıp, modern dünyanın standardı olan **Structured Logging** ve **Structlog** kütüphanesine derinlemesine dalacağız.

## 1. Neden `logging` Modülü Yetmez?

Python'un `logging` modülü 20 yıl önce tasarlandı. O zamanlar loglar dosyalara (syslog) yazılırdı ve insanlar okurdu.
Bugün logları makineler (Elasticsearch, Datadog, CloudWatch) okuyor.

Klasik log:
`2025-02-22 13:00:00 - ERROR - User 123 failed to purchase item 456 using coupon SAVE10.`

Bu logu makinenin anlaması için Regex (Regular Expression) yazmanız gerekir. Ve log formatı değişirse, Regex patlar.

Structured (JSON) log:
```json
{
  "timestamp": "2025-02-22T13:00:00Z",
  "level": "error",
  "event": "purchase_failed",
  "user_id": 123,
  "item_id": 456,
  "coupon": "SAVE10",
  "error_code": "INSUFFICIENT_FUNDS"
}
```
Şimdi Kibana'da veya CloudWatch Logs Insights'ta şu sorguyu atabilirsiniz:
`event="purchase_failed" AND coupon="SAVE10"` -> **"SAVE10 kuponuyla hata alan herkesi getir."**
İşte güç budur.

## 2. Structlog: Python İçin Modern Loglama

Standart `logging` modülü ile JSON çıktısı almak mümkündür ama acı vericidir. **Structlog**, bu iş için doğmuştur.

**Kurulum:**
`pip install structlog`

**Temel Konfigürasyon:**
Sadece JSON basmak yetmez, loglara otomatik olarak timestamp, log level ve hata stacktrace'i de eklenmelidir.

```python
import structlog
import logging
import sys

def configure_logging():
    structlog.configure(
        processors=[
            structlog.contextvars.merge_contextvars, # Asyncio context desteği
            structlog.processors.add_log_level,
            structlog.processors.TimeStamper(fmt="iso"),
            structlog.processors.StackInfoRenderer(),
            structlog.processors.format_exc_info, # Exception traceback'i güzelleştir
            structlog.processors.JSONRenderer() # Çıktıyı JSON yap
        ],
        logger_factory=structlog.PrintLoggerFactory(),
        cache_logger_on_first_use=True,
    )

log = structlog.get_logger()
```

## 3. Contextual Logging (Bağlamsal Loglama)

Bir API isteği geldiğinde, o isteğe ait `request_id`, `user_id`, `ip_address` gibi bilgileri her log satırına tek tek eklemek hamallıktır.
Structlog'un `bind` özelliği ile bu bilgileri bir kere bağlarsınız, o context (bağlam) boyunca tüm loglarda otomatik görünür.

```python
# Middleware veya Decorator içinde
def handle_request(request):
    # Bu context'teki tüm loglara request_id ekle
    structlog.contextvars.bind_contextvars(request_id=str(uuid.uuid4()))
    
    process_user(request.user)
    
    # context temizliği (gerekirse)
    structlog.contextvars.clear_contextvars()

def process_user(user):
    log = structlog.get_logger()
    # Çıktıda otomatik olarak request_id olacak!
    log.info("user_processed", user_id=user.id) 
```

Bu özellik, asenkron (async/await) kodlarda da `contextvars` sayesinde sorunsuz çalışır. İsteğin yaşam döngüsü boyunca log bütünlüğü sağlanır.

## 4. Hassas Veri Gizleme (PII Masking)

Loglar debug için harikadır ama içinde kullanıcının şifresi, kredi kartı numarası veya email adresi varsa, KVKK/GDPR cezası yersiniz.
Structlog işlemcileri (processors) ile log yazılmadan hemen önce hassas verileri maskeleyebilirsiniz.

```python
def mask_sensitive_data(logger, method_name, event_dict):
    if "password" in event_dict:
        event_dict["password"] = "***MASKED***"
    if "email" in event_dict:
        # Email: f***@gmail.com formatına çevir
        email = event_dict["email"]
        if "@" in email:
            user, domain = email.split("@")
            event_dict["email"] = f"{user[0]}***@{domain}"
    return event_dict

# Konfigürasyona ekleyin
processors=[
    ...,
    mask_sensitive_data,
    structlog.processors.JSONRenderer()
]
```
Artık developer yanlışlıkla `log.info("signup", password=user_input)` yazsa bile, diskteki logda şifre görünmez. Güvenlik, süreçlere gömülmelidir.

## 5. Distributed Tracing ve Correlation ID

Mikroservis dünyasında, Frontend -> API Gateway -> Auth Service -> Payment Service zincirinde, hatanın nerede olduğunu bulmak için **Correlation ID** (veya Trace ID) şarttır.
OpenTelemetry gibi standartlar var ama basitçe şunu yapmalısınız:
1.  API Gateway (veya Nginx), gelen isteğe `X-Request-ID` header'ı ekler.
2.  Python uygulamanız bu header'ı okur ve structlog context'ine `trace_id` olarak ekler.
3.  Başka bir servise HTTP isteği atarken, bu `trace_id`'yi header olarak iletir.

Böylece Kibana'da tek bir ID ile tüm sistemdeki akışı haritalayabilirsiniz.

## 6. Performans: Loglama Sistemi Yavaşlatmamalı

JSON serialize etmek CPU maliyetidir. Dosyaya yazmak (Disk I/O) yavaştır.
High-performance sistemlerde loglama "Blocking" olmamalıdır.
*   **Console Output:** Docker kullanıyorsanız logları `stdout`'a basın. Bırakın dosya yazma işini Docker Daemon veya Fluentbit yapsın. Uygulamanız diskle uğraşmasın.
*   **Async Logging:** Eğer çok yoğun log basıyorsanız, logları bir kuyruğa (queue) atıp, ayrı bir thread'de yazmayı değerlendirin. Ancak Python'un GIL'i yüzünden bu her zaman performans artışı sağlamaz. Genellikle `orjson` gibi hızlı JSON kütüphanelerini structlog ile entegre etmek en iyi sonucu verir.

```python
import orjson

structlog.configure(
    processors=[
        ...,
        structlog.processors.JSONRenderer(serializer=orjson.dumps)
    ],
    ...
)
```

## 7. ELK Stack ile Entegrasyon İpuçları

Loglarınızı Elasticsearch'e gönderirken dikkat etmeniz gerekenler:
*   **Mapping:** Elasticsearch'te sayısal alanların (örn: `latency_ms`) sayı (integer/float) olarak, metinlerin keyword olarak indexlendiğinden emin olun. Yoksa `latency > 500` sorgusu atamazsınız.
*   **Index Lifecycle (ILM):** Loglar çok hızlı büyür. 30 günden eski logları otomatik silen veya Cold Storage'a (S3) taşıyan politikalarınız olsun. Yoksa disk dolar ve Elastic çöker.

## Sonuç

Loglama, "olsa iyi olur" değil, "olmazsa olmaz" bir özelliktir.
İyi bir structlog konfigürasyonu, PII masking ve trace ID implementasyonu, sizi production yangınlarında kahraman yapar.
Gecenin bir yarısı uyandırıldığınızda, sorunu 5 dakikada bulmakla, sabaha kadar log okumak arasındaki fark budur.
Şimdi `print`lerinizi silin ve o logger'ı doğru yapılandırın.

