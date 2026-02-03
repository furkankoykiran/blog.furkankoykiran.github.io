---
title: "Webhok: Olay Güdümlü Mimarinin Temeli ve Güvenlik"
description: "Webhook sadece bir HTTP POST değildir. İmza doğrulama, Idempotency ve Asenkron kuyruk yönetimi ile sağlam bir webhook altyapısı kurun."
date: "2025-01-25 11:30:00 +0300"
categories: [Backend, Architecture, Security]
tags: [webhook, event-driven, security, python, hmac]
image:
  path: /assets/img/posts/webhook-architecture-diagram.png
  alt: "Webhook Architecture and Flow"
---


Modern yazılım dünyası "Polling"den (Sürekli sorma) "Push"a (Bildirim) evrildi.
Stripe ile ödeme alırken "Ödeme oldu mu?" diye dakikada bir API'ye sormazsınız. Stripe size "Ödeme gerçekleşti!" diye haber verir.
İşte buna **Webhook** denir.

Ancak webhook işlemek, basit bir API endpoint'i yazmaktan çok daha karmaşıktır.
İşin içine **Güvenlik** (Kim gönderdi?), **Güvenilirlik** (Ya sunucum kapalıysa?) ve **Tutarlılık** (Aynı mesaj iki kere gelirse?) girdiğinde, basit bir controller fonksiyonu yetersiz kalır.

Bu yazıda, production-grade bir webhook alıcı (handler) tasarımını, "Replay Attack" önlemlerini ve ölçeklenebilir kuyruk mimarisini inceleyeceğiz.

## 1. Güvenlik: HMAC İmza Doğrulama

Endpoint'iniz public (herkese açık) olmak zorundadır. Peki ya ben `curl -X POST https://sizin-api.com/webhooks` ile sahte bir "Ödeme Başarılı" mesajı atarsam?
Sisteminizi kandırıp bedava ürün alabilirim.
Bunu engellemenin yolu **HMAC (Hash-based Message Authentication Code)** imzasıdır.

Webhook sağlayıcısı (Github, Stripe vb.), mesajı sizin bildiğiniz gizli bir anahtarla (Secret Key) hashler ve header'a koyar (`X-Hub-Signature`).

```python
import hmac
import hashlib
from fastapi import Request, HTTPException

SECRET_KEY = b"my_super_secret_webhook_key"

async def verify_signature(request: Request):
    payload = await request.body()
    signature = request.headers.get("X-Hub-Signature-256")
    if not signature:
         raise HTTPException(status_code=403, detail="No Signature")
    
    # 1. İmza Formatını Kontrol Et
    # Genelde format 'sha256=....' şeklindedir
    try:
        algo, hash_val = signature.split("=")
    except ValueError:
        raise HTTPException(status_code=400, detail="Invalid Signature Format")

    expected = hmac.new(SECRET_KEY, payload, hashlib.sha256).hexdigest()
    
    # 2. Timing Attack Önlemi (secrets.compare_digest)
    if not hmac.compare_digest(expected, hash_val):
        raise HTTPException(status_code=403, detail="Invalid Signature")
```
Bu kontrolü yapmadan ASLA webhook işlemeyin.

## 2. İleri Güvenlik: Replay Attack Önleme

Saldırgan, geçerli bir webhook isteğini (imzası doğru) yakalayıp (Man-in-the-middle), 1 saat sonra tekrar sunucunuza gönderirse ne olur?
İmza hala geçerlidir! Ama işlem tekrar eder.
Bunu önlemek için **Timestamp** kontrolü şarttır.
Stripe gibi sağlayıcılar header'a `Stripe-Signature-Timestamp` gibi bir bilgi ekler.

```python
import time

def verify_timestamp(headers):
    timestamp = int(headers.get("X-Timestamp"))
    now = int(time.time())
    
    # Eğer istek 5 dakika (300sn) öncesine aitse reddet
    if now - timestamp > 300:
        raise HTTPException(status_code=400, detail="Timestamp too old (Replay Attack?)")
```
Bu sayede eski paketlerin tekrar sisteme sokulmasını engellersiniz.

## 3. Asenkron Mimari: "Hemen Dön, Sonra İşle"

Webhook sağlayıcıları sabırsızdır. İsteği atar ve 2-3 saniye içinde "200 OK" bekler.
Eğer siz "Dur şu ödemeyi veritabanına yazayım, faturayı oluşturup mail atayım" derseniz, işlem uzar ve timeout olur.
Sağlayıcı "Hata oldu" sanıp aynı webhook'u tekrar gönderir (Retry). Kısır döngüye girersiniz.

**Altın Kural:** Webhook endpoint'inde ASLA iş yapmayın. Sadece kuyruğa atın.

```python
from pydantic import BaseModel, ValidationError

class PaymentEvent(BaseModel):
    id: str
    amount: int
    currency: str

@app.post("/webhook")
async def handle_webhook(request: Request, background_tasks: BackgroundTasks):
    await verify_signature(request)
    
    # Payload Validasyonu (Pydantic)
    try:
        body = await request.json()
        event = PaymentEvent(**body)
    except ValidationError as e:
        # 400 Bad Request dön ki sağlayıcı bir daha denemesin (Hatalı format)
        return JSONResponse(status_code=400, content={"error": str(e)})

    # Event'i kuyruğa at (Celery/Redis)
    background_tasks.add_task(process_payment_queue, event.dict())
    
    return {"status": "received"}
```

## 4. Idempotency ve Distributed Locking

Ağ hatası oldu, Stripe cevabınızı alamadı ve webhook'u tekrar gönderdi.
Fatura oluşturma kodunuz iki kere çalışırsa, müşteriye iki fatura gider.
Bunu önlemek için **Idempotency Key** kullanmalısınız.

```python
import redis

r = redis.Redis()

def process_payment_queue(event):
    event_id = event['id']
    lock_key = f"lock:event:{event_id}"
    processed_key = f"processed:event:{event_id}"
    
    # 1. Daha önce işlenmiş mi?
    if r.get(processed_key):
        return # Sessizce çık

    # 2. Şu an başka bir worker işliyor mu? (Race Condition)
    with r.lock(lock_key, timeout=10):
        # Transaction Başlat
        try:
            create_invoice(event)
            # İşleme bitince ID'yi işaretle (24 saat sakla)
            r.setex(processed_key, 86400, "1")
        except Exception:
            # Hata varsa, lock zaten kalkacak, tekrar denensin.
            raise
```
Idempotent sistemler, "En az bir kere" (At-least-once) teslimat garantisi olan dağıtık sistemlerin sigortasıdır.

## 5. Fan-Out Pattern (Olay Dağıtımı)

Uygulamanız büyüdükçe, tek bir webhook birden fazla servisi ilgilendirebilir.
"Kullanıcı Kayıt Oldu" eventi geldiğinde:
1.  CRM servisi (Hubspot) güncellenmeli.
2.  Email servisi "Hoşgeldin" demeli.
3.  Analytics servisi loglamalı.

Bunları tek bir fonksiyonda yapmak yerine, bir **Event Bus** (RabbitMQ Topic Exchange veya AWS SNS) kullanın.
Webhook Handler sadece eventi Bus'a bırakır (`UserCreated`).
Diğer mikroservisler bu eventi dinler ve bağımsız olarak çalışır.
Böylece Email servisi çökerse, CRM güncellemesi bundan etkilenmez.

![Event Driven Architecture](/assets/img/posts/rabbitmq-architecture-diagram.png)
*Event Bus ve Microservices etkileşimi.*

## 6. Test Etmek: Localhost Tünelleri

Geliştirme yaparken webhook'ları localhost'a yönlendirmek için **Ngrok** veya **Cloudflare Tunnel** kullanın.
`ngrok http 8000` komutu size `https://xy.ngrok-free.app` adresini verir.
Sağlayıcıya bu adresi girip canlı debug yapabilirsiniz.

Ayrıca **RequestBin** veya **Webhook.site** kullanarak, kod yazmadan önce gelen payload'un yapısını (Headerlar, Body) incelemek çok faydalıdır.

## Sonuç

Webhook entegrasyonu, modern API geliştirmenin omurgasıdır.
"Aldım, işledim" basitliğinde değildir.
Güvenlik (HMAC, Timestamp), Performans (Async Queue) ve Tutarlılık (Idempotency) saç ayaklarını sağlam kurmazsanız, gece yarısı telefonunuz "Mükerrer çekim yapılmış!" diye çalar.
Sisteminizi savunmacı (Defensive) programlama ile kurun, her gelen isteği potansiyel bir tehdit veya mükerrer (duplicate) olarak görün.

Olay güdümlü mimarinin (EDA) esnekliği harikadır, ama disiplin gerektirir.

