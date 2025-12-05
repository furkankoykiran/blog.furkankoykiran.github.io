---
title: "Webhook Handler ile Olay Güdümlü Mimari"
description: "FastAPI ile production-ready webhook handler sistemi. Event-driven mimari, signature verification, retry mechanism, idempotency ve async processing."
date: "2025-01-25 09:00:00 +0300"
categories: [Python, Backend Development]
tags: [webhooks, event-driven, fastapi, async, integration, api]
image:
  path: /assets/img/posts/webhook-architecture-diagram.png
  alt: "Webhook Architecture Diagram"
---

Modern uygulama mimarisinde, sistemler arası gerçek zamanlı iletişim kritik bir gereksinim haline geldi. Webhooklar, bu gereksinimi karşılamak için event-driven (olay güdümlü) mimari deseninde kullanılan güçlü bir araçtır. Bu yazıda, Python ve FastAPI kullanarak production-ready bir webhook handler sistemi geliştireceğiz.

## Webhook Nedir?

Webhook, bir uygulamada gerçekleşen olayları diğer uygulamalara HTTP POST istekleriyle bildiren "ters API" mekanizmasıdır. Geleneksel API'larda istemci sürekli veri sorgulaması yaparken (polling), webhooklarda sunucu bir olay olduğunda proaktif olarak bildirim gönderir.

### Webhook vs Polling

**Polling Yaklaşımı:**
```python
import time
import requests

# Geleneksel polling - Verimsiz!
while True:
    response = requests.get("https://api.example.com/orders/status")
    if response.json()["status"] == "completed":
        process_order(response.json())
        break
    time.sleep(10)  # 10 saniye bekle ve tekrar dene
```

**Webhook Yaklaşımı:**
```python
from fastapi import FastAPI, Request

app = FastAPI()

# Event-driven - Verimli!
@app.post("/webhooks/order-completed")
async def handle_order_completion(request: Request):
    """Sipariş tamamlandığında otomatik çağrılır"""
    payload = await request.json()
    await process_order(payload)
    return {"status": "received"}
```

Webhook yaklaşımı gereksiz API çağrılarını ortadan kaldırarak bandwidth tasarrufu sağlar ve gerçek zamanlı tepki verme imkanı sunar.

![FastAPI Webhook Event-Driven Architecture](/assets/img/posts/fastapi-webhook-event-driven.png){: w="800" h="500" .shadow }
_FastAPI ile event-driven webhook sistemi_

## FastAPI ile Webhook Endpoint Oluşturma

### Temel Webhook Handler

```python
from fastapi import FastAPI, Request, HTTPException, BackgroundTasks
from pydantic import BaseModel, Field, validator
from typing import Optional, Dict, Any
from datetime import datetime
import hmac
import hashlib
import json

app = FastAPI(title="Webhook Handler Service")

class WebhookPayload(BaseModel):
    """Webhook payload validasyonu"""
    event_type: str = Field(..., description="Olay tipi")
    event_id: str = Field(..., description="Benzersiz olay ID'si")
    timestamp: datetime = Field(..., description="Olay zamanı")
    data: Dict[str, Any] = Field(..., description="Olay verisi")
    
    @validator('event_type')
    def validate_event_type(cls, v):
        """Desteklenen event tiplerini kontrol et"""
        allowed_events = [
            'order.created',
            'order.completed',
            'payment.received',
            'shipment.dispatched'
        ]
        if v not in allowed_events:
            raise ValueError(f"Unsupported event type: {v}")
        return v

@app.post("/webhooks/receive")
async def receive_webhook(
    request: Request,
    background_tasks: BackgroundTasks
):
    """
    Webhook alma endpoint'i
    
    Bu endpoint:
    1. Webhook imzasını doğrular
    2. Payload'ı parse eder ve validate eder
    3. İşlemi background task olarak zamanlar
    4. Hızlıca 200 OK döner
    """
    # 1. İmza doğrulama
    signature = request.headers.get("X-Webhook-Signature")
    if not signature:
        raise HTTPException(status_code=401, detail="Missing signature")
    
    body = await request.body()
    if not verify_signature(body, signature):
        raise HTTPException(status_code=401, detail="Invalid signature")
    
    # 2. Payload parsing
    try:
        payload_dict = json.loads(body)
        payload = WebhookPayload(**payload_dict)
    except Exception as e:
        raise HTTPException(status_code=400, detail=f"Invalid payload: {str(e)}")
    
    # 3. Background processing
    background_tasks.add_task(process_webhook, payload)
    
    # 4. Hızlı response (webhook gönderen taraf timeout'a düşmesin)
    return {
        "status": "accepted",
        "event_id": payload.event_id,
        "received_at": datetime.utcnow().isoformat()
    }

def verify_signature(body: bytes, signature: str) -> bool:
    """
    HMAC-SHA256 ile webhook imzasını doğrula
    
    Güvenlik için kritik! Webhook'un gerçekten beklenen
    kaynaktan geldiğini garanti eder.
    """
    secret = "your-webhook-secret-key"  # Ortam değişkeninden al
    expected_signature = hmac.new(
        secret.encode(),
        body,
        hashlib.sha256
    ).hexdigest()
    
    # Timing attack'lara karşı güvenli karşılaştırma
    return hmac.compare_digest(expected_signature, signature)

async def process_webhook(payload: WebhookPayload):
    """
    Webhook işleme logic'i
    
    Bu fonksiyon background task olarak çalışır,
    böylece webhook endpoint hızlı response dönebilir.
    """
    print(f"Processing webhook: {payload.event_type}")
    
    # Event tipine göre işlem yap
    if payload.event_type == "order.created":
        await handle_order_created(payload.data)
    elif payload.event_type == "payment.received":
        await handle_payment_received(payload.data)
    # ... diğer event tipleri
```

### Payload Validation

Webhook payloadları dış kaynaklardan geldiği için sıkı validation kritiktir:

```python
from pydantic import BaseModel, Field, validator, root_validator
from typing import Literal, Union
from decimal import Decimal

class OrderCreatedPayload(BaseModel):
    """order.created eventi için özel payload modeli"""
    order_id: str = Field(..., regex=r'^ORD-\d{8}$')
    customer_email: str = Field(..., regex=r'^[\w\.-]+@[\w\.-]+\.\w+$')
    total_amount: Decimal = Field(..., gt=0, max_digits=10, decimal_places=2)
    currency: Literal["USD", "EUR", "TRY"] = "USD"
    items: list[dict] = Field(..., min_items=1)
    
    @validator('items')
    def validate_items(cls, items):
        """Her item'ın gerekli alanları içerdiğini doğrula"""
        required_fields = {'product_id', 'quantity', 'price'}
        for item in items:
            if not all(field in item for field in required_fields):
                raise ValueError(f"Item missing required fields: {required_fields}")
            if item['quantity'] <= 0:
                raise ValueError("Item quantity must be positive")
        return items
    
    @root_validator
    def validate_total(cls, values):
        """Total amount ile item fiyatlarının toplamını karşılaştır"""
        items = values.get('items', [])
        calculated_total = sum(
            Decimal(str(item['price'])) * item['quantity'] 
            for item in items
        )
        
        if abs(calculated_total - values['total_amount']) > Decimal('0.01'):
            raise ValueError("Total amount mismatch")
        
        return values

# Event-specific handler
async def handle_order_created(data: dict):
    """Sipariş oluşturma eventi işle"""
    try:
        order = OrderCreatedPayload(**data)
        
        # İş logic'i
        print(f"New order: {order.order_id}")
        await send_confirmation_email(order.customer_email)
        await update_inventory(order.items)
        await notify_warehouse(order)
        
    except Exception as e:
        # Hata loglama ve alerting
        print(f"Error processing order.created: {e}")
        await send_alert(f"Webhook processing failed: {e}")
```

![Webhook Retry Mechanism](/assets/img/posts/webhook-retry-mechanism.png){: w="800" h="500" .shadow }
_Webhook retry ve error handling mekanizması_

## Retry Logic ve Hata Yönetimi

Webhook işleme sırasında hatalar oluşabilir. Production sistemlerinde robust retry mekanizması şarttır:

```python
import asyncio
from functools import wraps
from typing import Callable
import aioredis

class RetryConfig:
    """Retry stratejisi konfigürasyonu"""
    MAX_RETRIES = 5
    INITIAL_DELAY = 1  # saniye
    MAX_DELAY = 300    # 5 dakika
    BACKOFF_FACTOR = 2  # Exponential backoff
    
def exponential_backoff_retry(
    max_retries: int = RetryConfig.MAX_RETRIES,
    initial_delay: int = RetryConfig.INITIAL_DELAY,
    max_delay: int = RetryConfig.MAX_DELAY,
    backoff_factor: int = RetryConfig.BACKOFF_FACTOR
):
    """
    Exponential backoff retry decorator
    
    Retry delay'leri: 1s, 2s, 4s, 8s, 16s...
    Her başarısız denemeden sonra bekleme süresi katlanarak artar.
    """
    def decorator(func: Callable):
        @wraps(func)
        async def wrapper(*args, **kwargs):
            last_exception = None
            
            for attempt in range(max_retries):
                try:
                    return await func(*args, **kwargs)
                except Exception as e:
                    last_exception = e
                    
                    if attempt == max_retries - 1:
                        # Son deneme de başarısız
                        raise
                    
                    # Delay hesapla (exponential backoff)
                    delay = min(
                        initial_delay * (backoff_factor ** attempt),
                        max_delay
                    )
                    
                    print(f"Attempt {attempt + 1} failed: {e}")
                    print(f"Retrying in {delay} seconds...")
                    
                    await asyncio.sleep(delay)
            
            raise last_exception
        
        return wrapper
    return decorator

@exponential_backoff_retry(max_retries=5)
async def process_webhook_with_retry(payload: WebhookPayload):
    """
    Retry mekanizmasıyla webhook işleme
    
    Geçici hatalar (network, rate limit) için otomatik retry yapar.
    """
    # Harici API çağrısı (başarısız olabilir)
    async with aiohttp.ClientSession() as session:
        async with session.post(
            "https://api.partner.com/notify",
            json=payload.dict(),
            timeout=aiohttp.ClientTimeout(total=10)
        ) as response:
            response.raise_for_status()
            return await response.json()

# Dead Letter Queue (DLQ) için
class DeadLetterQueue:
    """
    Tekrar tekrar başarısız olan webhook'ları sakla
    
    DLQ, manual investigation veya delayed retry için kullanılır.
    """
    def __init__(self, redis_url: str):
        self.redis = aioredis.from_url(redis_url)
    
    async def add(self, payload: WebhookPayload, error: str):
        """Başarısız webhook'u DLQ'ya ekle"""
        dlq_item = {
            "payload": payload.dict(),
            "error": error,
            "failed_at": datetime.utcnow().isoformat(),
            "retry_count": payload.dict().get("retry_count", 0)
        }
        
        # Redis sorted set'e ekle (timestamp'e göre sıralı)
        await self.redis.zadd(
            "webhook:dlq",
            {json.dumps(dlq_item): datetime.utcnow().timestamp()}
        )
        
        # Alert gönder
        await send_dlq_alert(payload.event_id, error)
    
    async def get_failed_webhooks(self, limit: int = 100):
        """DLQ'daki başarısız webhook'ları getir"""
        items = await self.redis.zrange("webhook:dlq", 0, limit - 1)
        return [json.loads(item) for item in items]
    
    async def retry_dlq_item(self, item_json: str):
        """DLQ'daki bir item'ı tekrar dene"""
        item = json.loads(item_json)
        payload = WebhookPayload(**item["payload"])
        
        try:
            await process_webhook_with_retry(payload)
            # Başarılı olursa DLQ'dan çıkar
            await self.redis.zrem("webhook:dlq", item_json)
        except Exception as e:
            # Hala başarısız, retry_count'u artır
            item["retry_count"] = item.get("retry_count", 0) + 1
            await self.redis.zadd(
                "webhook:dlq",
                {json.dumps(item): datetime.utcnow().timestamp()}
            )

# Kullanım
dlq = DeadLetterQueue("redis://localhost:6379")

@app.post("/webhooks/receive")
async def receive_webhook_with_dlq(
    request: Request,
    background_tasks: BackgroundTasks
):
    """DLQ entegrasyonlu webhook handler"""
    # ... signature validation ve parsing ...
    
    try:
        await process_webhook_with_retry(payload)
    except Exception as e:
        # Tüm retry'lar başarısız, DLQ'ya gönder
        await dlq.add(payload, str(e))
    
    return {"status": "accepted", "event_id": payload.event_id}

# DLQ monitoring endpoint
@app.get("/admin/dlq")
async def get_dlq_status():
    """DLQ durumunu getir (admin endpoint)"""
    failed_items = await dlq.get_failed_webhooks()
    return {
        "count": len(failed_items),
        "items": failed_items
    }

@app.post("/admin/dlq/{item_index}/retry")
async def retry_dlq_item(item_index: int):
    """DLQ'daki belirli bir item'ı tekrar dene"""
    items = await dlq.get_failed_webhooks()
    if item_index >= len(items):
        raise HTTPException(status_code=404, detail="Item not found")
    
    item = items[item_index]
    await dlq.retry_dlq_item(json.dumps(item))
    return {"status": "retry_scheduled"}
```

## Async Queue ile Yüksek Throughput

Yüksek trafikli webhook sistemlerinde async queue kullanımı performans için kritiktir:

```python
import asyncio
from asyncio import Queue
from typing import List
import signal

class WebhookProcessor:
    """
    Async queue tabanlı webhook işlemci
    
    Multiple worker'larla paralel işleme sağlar.
    """
    def __init__(self, num_workers: int = 10, queue_size: int = 1000):
        self.queue: Queue[WebhookPayload] = Queue(maxsize=queue_size)
        self.num_workers = num_workers
        self.workers: List[asyncio.Task] = []
        self.running = False
    
    async def start(self):
        """Worker'ları başlat"""
        self.running = True
        self.workers = [
            asyncio.create_task(self._worker(i))
            for i in range(self.num_workers)
        ]
        print(f"Started {self.num_workers} webhook workers")
    
    async def stop(self):
        """Graceful shutdown"""
        print("Stopping webhook processor...")
        self.running = False
        
        # Queue'daki kalan item'ları işle
        await self.queue.join()
        
        # Worker'ları durdur
        for worker in self.workers:
            worker.cancel()
        
        await asyncio.gather(*self.workers, return_exceptions=True)
        print("Webhook processor stopped")
    
    async def enqueue(self, payload: WebhookPayload) -> bool:
        """
        Webhook'u queue'ya ekle
        
        Returns:
            True if enqueued, False if queue is full
        """
        try:
            # Non-blocking put with timeout
            await asyncio.wait_for(
                self.queue.put(payload),
                timeout=1.0
            )
            return True
        except asyncio.TimeoutError:
            # Queue full, webhook rejected
            print(f"Queue full, rejecting webhook: {payload.event_id}")
            return False
    
    async def _worker(self, worker_id: int):
        """
        Worker task - queue'dan webhook alıp işler
        """
        print(f"Worker {worker_id} started")
        
        while self.running:
            try:
                # Queue'dan al (timeout ile)
                payload = await asyncio.wait_for(
                    self.queue.get(),
                    timeout=1.0
                )
                
                try:
                    # Webhook'u işle
                    await self._process_with_timeout(payload, timeout=30)
                except Exception as e:
                    print(f"Worker {worker_id} error: {e}")
                    # DLQ'ya gönder
                    await dlq.add(payload, str(e))
                finally:
                    # Queue task'ını işaretle
                    self.queue.task_done()
                    
            except asyncio.TimeoutError:
                # Queue'da item yok, devam et
                continue
            except asyncio.CancelledError:
                print(f"Worker {worker_id} cancelled")
                break
        
        print(f"Worker {worker_id} stopped")
    
    async def _process_with_timeout(
        self,
        payload: WebhookPayload,
        timeout: int = 30
    ):
        """
        Webhook'u timeout ile işle
        
        Uzun süren işlemlerin sistem'i bloklamasını önler.
        """
        try:
            await asyncio.wait_for(
                process_webhook_with_retry(payload),
                timeout=timeout
            )
        except asyncio.TimeoutError:
            raise Exception(f"Webhook processing timeout: {payload.event_id}")
    
    async def get_stats(self) -> dict:
        """Queue ve worker istatistikleri"""
        return {
            "queue_size": self.queue.qsize(),
            "queue_maxsize": self.queue.maxsize,
            "num_workers": self.num_workers,
            "active_workers": sum(1 for w in self.workers if not w.done())
        }

# Global processor instance
processor = WebhookProcessor(num_workers=10, queue_size=1000)

@app.on_event("startup")
async def startup_event():
    """FastAPI startup - worker'ları başlat"""
    await processor.start()
    
    # Graceful shutdown için signal handler
    loop = asyncio.get_event_loop()
    for sig in (signal.SIGTERM, signal.SIGINT):
        loop.add_signal_handler(
            sig,
            lambda: asyncio.create_task(shutdown())
        )

@app.on_event("shutdown")
async def shutdown_event():
    """FastAPI shutdown - worker'ları durdur"""
    await processor.stop()

async def shutdown():
    """Graceful shutdown handler"""
    print("Received shutdown signal")
    await processor.stop()

@app.post("/webhooks/receive")
async def receive_webhook_queued(request: Request):
    """
    Queue-based webhook receiver
    
    Webhook'u queue'ya ekler ve hızlıca döner.
    Gerçek işlem background worker'larda yapılır.
    """
    # Validation...
    signature = request.headers.get("X-Webhook-Signature")
    if not signature:
        raise HTTPException(status_code=401, detail="Missing signature")
    
    body = await request.body()
    if not verify_signature(body, signature):
        raise HTTPException(status_code=401, detail="Invalid signature")
    
    payload = WebhookPayload(**json.loads(body))
    
    # Queue'ya ekle
    enqueued = await processor.enqueue(payload)
    
    if not enqueued:
        # Queue full, rate limit response
        raise HTTPException(
            status_code=429,
            detail="Queue full, please retry later",
            headers={"Retry-After": "60"}
        )
    
    return {
        "status": "queued",
        "event_id": payload.event_id,
        "queue_position": processor.queue.qsize()
    }

@app.get("/health/processor")
async def processor_health():
    """Processor health check endpoint"""
    stats = await processor.get_stats()
    
    # Queue %80 doluysa warning
    queue_usage = stats["queue_size"] / stats["queue_maxsize"]
    status = "healthy"
    if queue_usage > 0.8:
        status = "warning"
    if queue_usage >= 1.0:
        status = "critical"
    
    return {
        "status": status,
        "stats": stats,
        "queue_usage_percent": round(queue_usage * 100, 2)
    }
```

![Webhook Idempotency Pattern](/assets/img/posts/webhook-idempotency-pattern.png){: w="700" h="400" .shadow }
_Idempotency pattern ile duplicate webhook handling_

## Idempotency - Aynı Webhook'u Tekrar İşlememe

Webhook provider'lar bazen aynı eventi birden fazla gönderebilir (network retry, bug, vb.). Sisteminiz idempotent olmalıdır:

```python
import aioredis
from datetime import timedelta
import hashlib

class IdempotencyManager:
    """
    Webhook idempotency yönetimi
    
    Aynı event_id'nin birden fazla işlenmesini önler.
    """
    def __init__(self, redis_url: str):
        self.redis = aioredis.from_url(redis_url)
        self.ttl = timedelta(days=7)  # 7 gün cache
    
    def _generate_key(self, event_id: str) -> str:
        """Idempotency key oluştur"""
        return f"webhook:idempotency:{event_id}"
    
    async def is_processed(self, event_id: str) -> bool:
        """Event daha önce işlendi mi?"""
        key = self._generate_key(event_id)
        exists = await self.redis.exists(key)
        return bool(exists)
    
    async def mark_processed(
        self,
        event_id: str,
        result: dict = None
    ):
        """Event'i işlenmiş olarak işaretle"""
        key = self._generate_key(event_id)
        value = {
            "processed_at": datetime.utcnow().isoformat(),
            "result": result or {}
        }
        
        # TTL ile sakla (7 gün sonra otomatik silinir)
        await self.redis.setex(
            key,
            self.ttl,
            json.dumps(value)
        )
    
    async def get_result(self, event_id: str) -> dict:
        """Daha önce işlenen event'in sonucunu getir"""
        key = self._generate_key(event_id)
        data = await self.redis.get(key)
        
        if data:
            return json.loads(data)
        return None
    
    async def cleanup_old_entries(self):
        """
        TTL expired olmayan eski entry'leri temizle
        
        Background task olarak periyodik çalıştırılır.
        """
        # Redis'te TTL otomatik çalışıyor,
        # bu method ek temizlik için kullanılabilir
        cursor = 0
        pattern = "webhook:idempotency:*"
        
        while True:
            cursor, keys = await self.redis.scan(
                cursor,
                match=pattern,
                count=100
            )
            
            for key in keys:
                ttl = await self.redis.ttl(key)
                if ttl < 0:  # TTL yoksa
                    await self.redis.delete(key)
            
            if cursor == 0:
                break

# Global idempotency manager
idempotency = IdempotencyManager("redis://localhost:6379")

@app.post("/webhooks/receive")
async def receive_webhook_idempotent(request: Request):
    """
    Idempotent webhook receiver
    
    Aynı event_id'yi birden fazla işlemez.
    """
    # Validation...
    body = await request.body()
    payload = WebhookPayload(**json.loads(body))
    
    # Idempotency check
    if await idempotency.is_processed(payload.event_id):
        # Daha önce işlenmiş, cached result dön
        cached_result = await idempotency.get_result(payload.event_id)
        return {
            "status": "already_processed",
            "event_id": payload.event_id,
            "processed_at": cached_result["processed_at"],
            "result": cached_result["result"]
        }
    
    # İlk defa işleniyor
    try:
        result = await process_webhook_with_retry(payload)
        
        # Başarıyla işlendi, mark et
        await idempotency.mark_processed(payload.event_id, result)
        
        return {
            "status": "processed",
            "event_id": payload.event_id,
            "result": result
        }
    except Exception as e:
        # Hata durumunda mark etme!
        # Retry'da tekrar deneyebilsin
        raise

# Content-based idempotency (event_id yoksa)
def generate_content_hash(payload: dict) -> str:
    """
    Payload içeriğinden hash üret
    
    event_id olmayan durumlarda kullanılır.
    """
    # Timestamp hariç tüm field'ları hashle
    payload_copy = payload.copy()
    payload_copy.pop('timestamp', None)
    
    content = json.dumps(payload_copy, sort_keys=True)
    return hashlib.sha256(content.encode()).hexdigest()

@app.post("/webhooks/receive-content-hash")
async def receive_webhook_content_hash(request: Request):
    """
    Content-based idempotency
    
    event_id olmayan webhook'lar için.
    """
    body = await request.body()
    payload_dict = json.loads(body)
    
    # Content hash'i idempotency key olarak kullan
    content_hash = generate_content_hash(payload_dict)
    
    if await idempotency.is_processed(content_hash):
        cached_result = await idempotency.get_result(content_hash)
        return {
            "status": "duplicate_detected",
            "content_hash": content_hash,
            "processed_at": cached_result["processed_at"]
        }
    
    # Process...
    payload = WebhookPayload(**payload_dict)
    result = await process_webhook_with_retry(payload)
    await idempotency.mark_processed(content_hash, result)
    
    return {"status": "processed", "content_hash": content_hash}
```

## Rate Limiting ve Backpressure

Webhook endpoint'lerinizi DDoS ve abuse'den korumak için rate limiting şarttır:

```python
from slowapi import Limiter, _rate_limit_exceeded_handler
from slowapi.util import get_remote_address
from slowapi.errors import RateLimitExceeded
from starlette.requests import Request

# Rate limiter
limiter = Limiter(key_func=get_remote_address)
app.state.limiter = limiter
app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)

@app.post("/webhooks/receive")
@limiter.limit("100/minute")  # Dakikada 100 webhook
async def receive_webhook_rate_limited(request: Request):
    """Rate-limited webhook endpoint"""
    # ... webhook processing ...
    pass

# IP-based rate limiting
class IPRateLimiter:
    """
    Redis-based IP rate limiter
    
    Distributed rate limiting için.
    """
    def __init__(self, redis_url: str):
        self.redis = aioredis.from_url(redis_url)
    
    async def is_allowed(
        self,
        ip: str,
        limit: int = 100,
        window: int = 60
    ) -> bool:
        """
        IP'nin rate limit'e takılıp takılmadığını kontrol et
        
        Args:
            ip: Client IP
            limit: Max request count
            window: Time window (seconds)
        """
        key = f"rate_limit:{ip}"
        
        # Current count
        current = await self.redis.get(key)
        
        if current is None:
            # İlk request
            await self.redis.setex(key, window, 1)
            return True
        
        current = int(current)
        
        if current >= limit:
            # Limit aşıldı
            return False
        
        # Counter'ı artır
        await self.redis.incr(key)
        return True
    
    async def get_remaining(self, ip: str, limit: int = 100) -> int:
        """Kalan request quota'sını getir"""
        key = f"rate_limit:{ip}"
        current = await self.redis.get(key)
        
        if current is None:
            return limit
        
        return max(0, limit - int(current))

rate_limiter = IPRateLimiter("redis://localhost:6379")

@app.post("/webhooks/receive")
async def receive_webhook_ip_limited(request: Request):
    """IP-based rate limiting"""
    client_ip = request.client.host
    
    # Rate limit check
    if not await rate_limiter.is_allowed(client_ip, limit=100, window=60):
        remaining = await rate_limiter.get_remaining(client_ip, limit=100)
        raise HTTPException(
            status_code=429,
            detail="Rate limit exceeded",
            headers={
                "X-RateLimit-Limit": "100",
                "X-RateLimit-Remaining": str(remaining),
                "X-RateLimit-Reset": str(60),
                "Retry-After": "60"
            }
        )
    
    # Process webhook...
    return {"status": "accepted"}

# Backpressure management
@app.post("/webhooks/receive")
async def receive_webhook_with_backpressure(request: Request):
    """
    Backpressure management
    
    System overload durumunda 503 döner.
    """
    # Queue capacity check
    stats = await processor.get_stats()
    queue_usage = stats["queue_size"] / stats["queue_maxsize"]
    
    if queue_usage > 0.95:  # Queue %95 dolu
        raise HTTPException(
            status_code=503,
            detail="Service temporarily overloaded",
            headers={"Retry-After": "120"}
        )
    
    # Process...
    return {"status": "accepted"}
```

## Webhook Gönderme (Outbound Webhooks)

Sisteminiz başka servislere webhook gönderecekse:

```python
import aiohttp
from typing import List, Optional
from enum import Enum

class WebhookSubscription(BaseModel):
    """Webhook subscription modeli"""
    id: str
    url: str
    secret: str
    events: List[str]  # Subscribe edilen event'ler
    active: bool = True
    retry_config: Optional[dict] = None

class WebhookDeliveryStatus(str, Enum):
    """Delivery status enum"""
    PENDING = "pending"
    DELIVERED = "delivered"
    FAILED = "failed"
    RETRYING = "retrying"

class OutboundWebhookSender:
    """
    Webhook gönderme servisi
    
    Retry, rate limiting ve delivery tracking ile.
    """
    def __init__(self):
        self.session: Optional[aiohttp.ClientSession] = None
    
    async def start(self):
        """HTTP session başlat"""
        self.session = aiohttp.ClientSession(
            timeout=aiohttp.ClientTimeout(total=10),
            connector=aiohttp.TCPConnector(limit=100)
        )
    
    async def stop(self):
        """HTTP session kapat"""
        if self.session:
            await self.session.close()
    
    def _generate_signature(self, body: bytes, secret: str) -> str:
        """Webhook signature oluştur"""
        return hmac.new(
            secret.encode(),
            body,
            hashlib.sha256
        ).hexdigest()
    
    async def send_webhook(
        self,
        subscription: WebhookSubscription,
        event_type: str,
        payload: dict,
        max_retries: int = 3
    ) -> dict:
        """
        Tek bir subscription'a webhook gönder
        
        Returns:
            Delivery result
        """
        if not subscription.active:
            return {
                "status": WebhookDeliveryStatus.FAILED,
                "reason": "Subscription inactive"
            }
        
        if event_type not in subscription.events:
            return {
                "status": WebhookDeliveryStatus.FAILED,
                "reason": f"Not subscribed to {event_type}"
            }
        
        # Payload hazırla
        webhook_payload = {
            "event_type": event_type,
            "event_id": str(uuid.uuid4()),
            "timestamp": datetime.utcnow().isoformat(),
            "data": payload
        }
        
        body = json.dumps(webhook_payload).encode()
        signature = self._generate_signature(body, subscription.secret)
        
        # Headers
        headers = {
            "Content-Type": "application/json",
            "X-Webhook-Signature": signature,
            "X-Webhook-Event": event_type,
            "User-Agent": "YourApp-Webhook/1.0"
        }
        
        # Retry loop
        for attempt in range(max_retries):
            try:
                async with self.session.post(
                    subscription.url,
                    data=body,
                    headers=headers
                ) as response:
                    # 2xx = success
                    if 200 <= response.status < 300:
                        return {
                            "status": WebhookDeliveryStatus.DELIVERED,
                            "status_code": response.status,
                            "attempt": attempt + 1
                        }
                    
                    # 4xx = client error, don't retry
                    if 400 <= response.status < 500:
                        return {
                            "status": WebhookDeliveryStatus.FAILED,
                            "status_code": response.status,
                            "reason": f"Client error: {response.status}",
                            "attempt": attempt + 1
                        }
                    
                    # 5xx = server error, retry
                    if attempt < max_retries - 1:
                        await asyncio.sleep(2 ** attempt)  # Exponential backoff
                        continue
                    
                    return {
                        "status": WebhookDeliveryStatus.FAILED,
                        "status_code": response.status,
                        "reason": f"Server error after {max_retries} attempts",
                        "attempt": attempt + 1
                    }
                    
            except asyncio.TimeoutError:
                if attempt < max_retries - 1:
                    await asyncio.sleep(2 ** attempt)
                    continue
                
                return {
                    "status": WebhookDeliveryStatus.FAILED,
                    "reason": "Timeout",
                    "attempt": attempt + 1
                }
            
            except Exception as e:
                if attempt < max_retries - 1:
                    await asyncio.sleep(2 ** attempt)
                    continue
                
                return {
                    "status": WebhookDeliveryStatus.FAILED,
                    "reason": str(e),
                    "attempt": attempt + 1
                }
        
        return {
            "status": WebhookDeliveryStatus.FAILED,
            "reason": "Max retries exceeded"
        }
    
    async def broadcast_webhook(
        self,
        event_type: str,
        payload: dict,
        subscriptions: List[WebhookSubscription]
    ) -> List[dict]:
        """
        Tüm subscriber'lara webhook broadcast et
        
        Parallel gönderim yapar.
        """
        tasks = [
            self.send_webhook(sub, event_type, payload)
            for sub in subscriptions
            if sub.active and event_type in sub.events
        ]
        
        results = await asyncio.gather(*tasks, return_exceptions=True)
        
        return [
            {
                "subscription_id": sub.id,
                "result": result if not isinstance(result, Exception) else {
                    "status": WebhookDeliveryStatus.FAILED,
                    "reason": str(result)
                }
            }
            for sub, result in zip(subscriptions, results)
        ]

# Webhook sender instance
webhook_sender = OutboundWebhookSender()

@app.on_event("startup")
async def startup_webhook_sender():
    await webhook_sender.start()

@app.on_event("shutdown")
async def shutdown_webhook_sender():
    await webhook_sender.stop()

# Event trigger endpoint
@app.post("/events/trigger")
async def trigger_event(
    event_type: str,
    payload: dict,
    background_tasks: BackgroundTasks
):
    """
    Event trigger eder ve webhook'ları gönderir
    
    Örnek: Sipariş tamamlandığında webhook gönder
    """
    # Subscriptions'ları getir (DB'den)
    subscriptions = await get_active_subscriptions(event_type)
    
    # Background'da webhook'ları gönder
    background_tasks.add_task(
        webhook_sender.broadcast_webhook,
        event_type,
        payload,
        subscriptions
    )
    
    return {
        "status": "event_triggered",
        "event_type": event_type,
        "subscriber_count": len(subscriptions)
    }

# Subscription management endpoints
@app.post("/webhooks/subscriptions")
async def create_subscription(subscription: WebhookSubscription):
    """Yeni webhook subscription oluştur"""
    # DB'ye kaydet
    await save_subscription(subscription)
    return {"status": "created", "subscription_id": subscription.id}

@app.delete("/webhooks/subscriptions/{subscription_id}")
async def delete_subscription(subscription_id: str):
    """Webhook subscription'ı sil"""
    await remove_subscription(subscription_id)
    return {"status": "deleted"}

@app.post("/webhooks/subscriptions/{subscription_id}/test")
async def test_subscription(subscription_id: str):
    """
    Subscription'ı test et
    
    Test webhook'u gönderir.
    """
    subscription = await get_subscription(subscription_id)
    
    result = await webhook_sender.send_webhook(
        subscription,
        "test.event",
        {"message": "This is a test webhook"}
    )
    
    return result
```

## Monitoring ve Observability

Production webhook sistemlerinde monitoring kritiktir:

```python
from prometheus_client import Counter, Histogram, Gauge
import time

# Prometheus metrics
webhook_received_total = Counter(
    'webhook_received_total',
    'Total webhooks received',
    ['event_type', 'status']
)

webhook_processing_duration = Histogram(
    'webhook_processing_duration_seconds',
    'Webhook processing duration',
    ['event_type']
)

webhook_queue_size = Gauge(
    'webhook_queue_size',
    'Current webhook queue size'
)

webhook_dlq_size = Gauge(
    'webhook_dlq_size',
    'Dead letter queue size'
)

@app.post("/webhooks/receive")
async def receive_webhook_monitored(request: Request):
    """Monitoring entegrasyonlu webhook handler"""
    start_time = time.time()
    
    try:
        # Parse payload
        body = await request.body()
        payload = WebhookPayload(**json.loads(body))
        
        # Process
        await processor.enqueue(payload)
        
        # Metrics
        webhook_received_total.labels(
            event_type=payload.event_type,
            status='success'
        ).inc()
        
        processing_time = time.time() - start_time
        webhook_processing_duration.labels(
            event_type=payload.event_type
        ).observe(processing_time)
        
        # Queue metrics
        stats = await processor.get_stats()
        webhook_queue_size.set(stats["queue_size"])
        
        return {"status": "accepted"}
        
    except Exception as e:
        webhook_received_total.labels(
            event_type='unknown',
            status='error'
        ).inc()
        raise

# Metrics endpoint
from prometheus_client import generate_latest

@app.get("/metrics")
async def metrics():
    """Prometheus metrics endpoint"""
    return Response(
        generate_latest(),
        media_type="text/plain"
    )

# Health check
@app.get("/health")
async def health_check():
    """Comprehensive health check"""
    try:
        # Redis check
        await idempotency.redis.ping()
        redis_healthy = True
    except:
        redis_healthy = False
    
    # Queue check
    stats = await processor.get_stats()
    queue_healthy = stats["queue_size"] < stats["queue_maxsize"] * 0.9
    
    # DLQ check
    dlq_items = await dlq.get_failed_webhooks(limit=1)
    dlq_count = len(dlq_items)
    
    overall_healthy = redis_healthy and queue_healthy
    
    return {
        "status": "healthy" if overall_healthy else "unhealthy",
        "components": {
            "redis": "ok" if redis_healthy else "error",
            "queue": "ok" if queue_healthy else "warning",
            "dlq_count": dlq_count
        },
        "stats": stats
    }

# Structured logging
import structlog

logger = structlog.get_logger()

@app.post("/webhooks/receive")
async def receive_webhook_logged(request: Request):
    """Structured logging ile webhook handler"""
    body = await request.body()
    payload = WebhookPayload(**json.loads(body))
    
    logger.info(
        "webhook_received",
        event_type=payload.event_type,
        event_id=payload.event_id,
        client_ip=request.client.host
    )
    
    try:
        await processor.enqueue(payload)
        
        logger.info(
            "webhook_queued",
            event_id=payload.event_id,
            queue_size=processor.queue.qsize()
        )
        
    except Exception as e:
        logger.error(
            "webhook_error",
            event_id=payload.event_id,
            error=str(e),
            exc_info=True
        )
        raise
    
    return {"status": "accepted"}
```

## Best Practices ve Güvenlik

### Güvenlik Checklist

```python
# 1. HTTPS Zorunlu
@app.middleware("http")
async def enforce_https(request: Request, call_next):
    """Production'da HTTPS zorunlu tut"""
    if not request.url.scheme == "https" and not request.url.hostname == "localhost":
        return Response("HTTPS required", status_code=403)
    return await call_next(request)

# 2. Signature Verification (Zorunlu!)
# Her webhook'ta signature doğrula

# 3. IP Whitelist
ALLOWED_IPS = ["192.168.1.100", "10.0.0.50"]  # Partner IP'leri

@app.middleware("http")
async def ip_whitelist(request: Request, call_next):
    """Sadece bilinen IP'lerden webhook kabul et"""
    client_ip = request.client.host
    
    if request.url.path.startswith("/webhooks/"):
        if client_ip not in ALLOWED_IPS:
            logger.warning(
                "webhook_blocked",
                ip=client_ip,
                path=request.url.path
            )
            return Response("Forbidden", status_code=403)
    
    return await call_next(request)

# 4. Request Size Limit
from fastapi import FastAPI
from starlette.middleware.base import BaseHTTPMiddleware

class RequestSizeLimitMiddleware(BaseHTTPMiddleware):
    """Request body size'ı sınırla"""
    def __init__(self, app, max_size: int = 1_000_000):  # 1MB
        super().__init__(app)
        self.max_size = max_size
    
    async def dispatch(self, request: Request, call_next):
        content_length = request.headers.get("content-length")
        
        if content_length and int(content_length) > self.max_size:
            return Response(
                "Request body too large",
                status_code=413
            )
        
        return await call_next(request)

app.add_middleware(RequestSizeLimitMiddleware, max_size=1_000_000)

# 5. Timeout Protection
# Her webhook işlemi için timeout belirle (30s)

# 6. Secrets Management
import os

WEBHOOK_SECRET = os.environ.get("WEBHOOK_SECRET")
if not WEBHOOK_SECRET:
    raise RuntimeError("WEBHOOK_SECRET environment variable not set")

# 7. Audit Logging
async def audit_log(event_id: str, action: str, details: dict):
    """Webhook işlemlerini audit log'a kaydet"""
    logger.info(
        "webhook_audit",
        event_id=event_id,
        action=action,
        details=details,
        timestamp=datetime.utcnow().isoformat()
    )
```

## Örnek: Tam Production Setup

```python
# main.py
from fastapi import FastAPI, Request, HTTPException, BackgroundTasks
from contextlib import asynccontextmanager
import signal

@asynccontextmanager
async def lifespan(app: FastAPI):
    """Application lifespan management"""
    # Startup
    print("Starting webhook service...")
    await processor.start()
    await webhook_sender.start()
    
    # Setup graceful shutdown
    loop = asyncio.get_event_loop()
    for sig in (signal.SIGTERM, signal.SIGINT):
        loop.add_signal_handler(
            sig,
            lambda: asyncio.create_task(graceful_shutdown())
        )
    
    yield
    
    # Shutdown
    print("Shutting down webhook service...")
    await processor.stop()
    await webhook_sender.stop()

app = FastAPI(
    title="Production Webhook Service",
    version="1.0.0",
    lifespan=lifespan
)

# Middleware
app.add_middleware(RequestSizeLimitMiddleware, max_size=1_000_000)

@app.post("/webhooks/{provider}")
async def receive_webhook_multi_provider(
    provider: str,
    request: Request,
    background_tasks: BackgroundTasks
):
    """
    Multi-provider webhook endpoint
    
    /webhooks/stripe
    /webhooks/shopify
    /webhooks/github
    """
    # Provider-specific validation
    if provider == "stripe":
        signature = request.headers.get("stripe-signature")
        # Stripe-specific validation
    elif provider == "shopify":
        signature = request.headers.get("x-shopify-hmac-sha256")
        # Shopify-specific validation
    else:
        raise HTTPException(status_code=404, detail="Unknown provider")
    
    # Common processing
    body = await request.body()
    payload = WebhookPayload(**json.loads(body))
    
    # Idempotency check
    if await idempotency.is_processed(payload.event_id):
        return {"status": "duplicate"}
    
    # Queue for processing
    await processor.enqueue(payload)
    await idempotency.mark_processed(payload.event_id)
    
    return {"status": "accepted", "event_id": payload.event_id}

async def graceful_shutdown():
    """Graceful shutdown handler"""
    print("Graceful shutdown initiated...")
    
    # Yeni webhook'ları reddet
    app.state.accepting_webhooks = False
    
    # Queue'daki işleri bitir
    await processor.stop()
    
    # Bağlantıları kapat
    await webhook_sender.stop()
    
    print("Graceful shutdown complete")

if __name__ == "__main__":
    import uvicorn
    
    uvicorn.run(
        "main:app",
        host="0.0.0.0",
        port=8000,
        workers=4,
        log_level="info",
        access_log=True
    )
```

## Sonuç

Production-ready bir webhook sistemi geliştirmek için:

1. **Signature Verification**: Her webhook'u kriptografik olarak doğrula
2. **Idempotency**: Aynı eventi birden fazla işleme
3. **Retry Logic**: Exponential backoff ile akıllı retry
4. **Queue Management**: Async queue ile yüksek throughput
5. **DLQ**: Başarısız webhook'ları sakla ve yönet
6. **Rate Limiting**: Abuse'den korun
7. **Monitoring**: Prometheus metrics ve structured logging
8. **Security**: HTTPS, IP whitelist, size limit
9. **Graceful Shutdown**: Hiçbir webhook kaybetme

Webhook'lar event-driven mimaride kritik rol oynar. Doğru implement edildiğinde, sistemler arası gerçek zamanlı, güvenilir ve ölçeklenebilir iletişim sağlar.

**Kaynaklar:**
- [Webhook.site](https://webhook.site/) - Webhook testing
- [Svix Webhook Gateway](https://www.svix.com/)
- [FastAPI Documentation](https://fastapi.tiangolo.com/)
- [Idempotency Patterns](https://stripe.com/docs/api/idempotent_requests)
