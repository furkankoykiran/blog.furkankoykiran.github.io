---
title: "Python ile Hız Sınırlama ve API Throttling"
description: "API güvenliği için rate limiting algoritmaları. Fixed window, sliding window, token bucket, Redis distributed solutions ve FastAPI entegrasyonu."
date: "2025-01-18 09:00:00 +0300"
categories: [API, Performance]
tags: [python, rate-limiting, api, redis, throttling, fastapi, performance, security]
image:
  path: /assets/img/posts/rate-limiting-algorithms-comparison.png
  alt: "Rate Limiting Algorithms Comparison"
---

Modern API'lerde hız sınırlama (rate limiting), sistem kaynaklarının korunması, DoS saldırılarının önlenmesi ve API kullanımının adil paylaşımı için kritik öneme sahiptir. Bu yazıda, Python ile çeşitli rate limiting algoritmalarını, Redis tabanlı distributed çözümleri ve FastAPI entegrasyonunu detaylıca inceleyeceğiz.

## Rate Limiting Nedir?

Rate limiting, belirli bir zaman diliminde bir kullanıcının veya IP adresinin yapabileceği istek sayısını sınırlama işlemidir. Amaçlar:

- **Kaynak Koruması**: Server'ın aşırı yüklenmesini önleme
- **DoS/DDoS Koruması**: Kötü niyetli saldırıları engelleme
- **API Abuse Prevention**: API kötüye kullanımını önleme
- **Fair Usage**: Adil kaynak dağılımı
- **Cost Control**: Maliyetli işlemlerin kontrolü
- **SLA Enforcement**: Servis seviyesi anlaşmalarının uygulanması

### Temel Terminoloji

```python
# Rate Limit Kavramları
rate_limit = {
    "limit": 100,          # İzin verilen istek sayısı
    "window": 60,          # Zaman penceresi (saniye)
    "remaining": 95,       # Kalan istek hakkı
    "reset": 1704207600,   # Reset zamanı (Unix timestamp)
    "retry_after": 35      # Yeniden deneme için bekleme süresi
}

# HTTP Headers
headers = {
    "X-RateLimit-Limit": "100",
    "X-RateLimit-Remaining": "95",
    "X-RateLimit-Reset": "1704207600",
    "Retry-After": "35"
}
```

## Rate Limiting Algoritmaları

![Rate Limiting Algorithms](/assets/img/posts/rate-limiting-algorithms-comparison.png){: w="800" h="500" .shadow }
_Comparison of rate limiting algorithms_

### 1. Fixed Window Counter

En basit algoritma. Sabit zaman pencerelerinde istek sayısını sayar.

```python
from datetime import datetime, timedelta
from collections import defaultdict
import threading

class FixedWindowRateLimiter:
    """Sabit pencere rate limiter"""
    
    def __init__(self, limit: int, window_seconds: int):
        """
        Args:
            limit: Pencere başına izin verilen istek sayısı
            window_seconds: Pencere boyutu (saniye)
        """
        self.limit = limit
        self.window_seconds = window_seconds
        self.requests = defaultdict(lambda: {'count': 0, 'window_start': None})
        self.lock = threading.Lock()
    
    def allow_request(self, key: str) -> bool:
        """
        İsteğe izin verilip verilmediğini kontrol et
        
        Args:
            key: Kullanıcı tanımlayıcı (user_id, IP, API key)
        
        Returns:
            İstek izin verildi mi
        """
        with self.lock:
            now = datetime.utcnow()
            user_data = self.requests[key]
            
            # İlk istek veya yeni pencere başlangıcı
            if user_data['window_start'] is None:
                user_data['window_start'] = now
                user_data['count'] = 1
                return True
            
            # Pencere süresi dolmuş mu?
            window_end = user_data['window_start'] + timedelta(seconds=self.window_seconds)
            
            if now >= window_end:
                # Yeni pencere başlat
                user_data['window_start'] = now
                user_data['count'] = 1
                return True
            
            # Mevcut pencere içinde
            if user_data['count'] < self.limit:
                user_data['count'] += 1
                return True
            
            return False
    
    def get_info(self, key: str) -> dict:
        """Rate limit bilgilerini döndür"""
        with self.lock:
            user_data = self.requests[key]
            
            if user_data['window_start'] is None:
                return {
                    'limit': self.limit,
                    'remaining': self.limit,
                    'reset': None
                }
            
            window_end = user_data['window_start'] + timedelta(seconds=self.window_seconds)
            remaining = max(0, self.limit - user_data['count'])
            
            return {
                'limit': self.limit,
                'remaining': remaining,
                'reset': int(window_end.timestamp())
            }

# Kullanım
limiter = FixedWindowRateLimiter(limit=10, window_seconds=60)

# Simülasyon
for i in range(12):
    user_id = "user_123"
    allowed = limiter.allow_request(user_id)
    info = limiter.get_info(user_id)
    
    print(f"Request {i+1}: {'✅ Allowed' if allowed else '❌ Blocked'} - "
          f"Remaining: {info['remaining']}/{info['limit']}")
```

**Avantajları:**
- Basit implementasyon
- Düşük memory kullanımı
- O(1) time complexity

**Dezavantajları:**
- **Burst problem**: Pencere sınırında 2x request mümkün
- Pencere geçişlerinde ani yük artışı

### 2. Sliding Window Log

Her isteğin timestamp'ini saklar, dinamik pencere kullanır.

```python
from datetime import datetime, timedelta
import bisect

class SlidingWindowLogRateLimiter:
    """Sliding window log rate limiter"""
    
    def __init__(self, limit: int, window_seconds: int):
        self.limit = limit
        self.window_seconds = window_seconds
        self.requests = {}  # {key: [timestamp1, timestamp2, ...]}
        self.lock = threading.Lock()
    
    def allow_request(self, key: str) -> bool:
        """İsteğe izin ver ve timestamp kaydet"""
        with self.lock:
            now = datetime.utcnow()
            cutoff = now - timedelta(seconds=self.window_seconds)
            
            # Kullanıcının isteklerini al
            if key not in self.requests:
                self.requests[key] = []
            
            # Eski istekleri temizle
            self.requests[key] = [
                ts for ts in self.requests[key]
                if ts > cutoff
            ]
            
            # Limit kontrolü
            if len(self.requests[key]) < self.limit:
                self.requests[key].append(now)
                return True
            
            return False
    
    def get_info(self, key: str) -> dict:
        """Rate limit bilgileri"""
        with self.lock:
            now = datetime.utcnow()
            cutoff = now - timedelta(seconds=self.window_seconds)
            
            if key not in self.requests:
                return {
                    'limit': self.limit,
                    'remaining': self.limit,
                    'reset': int((now + timedelta(seconds=self.window_seconds)).timestamp())
                }
            
            # Güncel istekleri filtrele
            recent_requests = [ts for ts in self.requests[key] if ts > cutoff]
            remaining = max(0, self.limit - len(recent_requests))
            
            # En eski isteğin expire olma zamanı
            reset_time = recent_requests[0] + timedelta(seconds=self.window_seconds) if recent_requests else now
            
            return {
                'limit': self.limit,
                'remaining': remaining,
                'reset': int(reset_time.timestamp())
            }

# Kullanım
limiter = SlidingWindowLogRateLimiter(limit=5, window_seconds=10)

import time
for i in range(7):
    allowed = limiter.allow_request("user_456")
    info = limiter.get_info("user_456")
    print(f"Request {i+1}: {'✅' if allowed else '❌'} - Remaining: {info['remaining']}")
    time.sleep(1)
```

**Avantajları:**
- Tam doğruluk
- Burst problem yok
- Esnek zaman pencereleri

**Dezavantajları:**
- Yüksek memory kullanımı (her timestamp saklanır)
- O(n) cleanup işlemi

### 3. Sliding Window Counter

Fixed window ve sliding log'un hibrit versiyonu. Memory efficient.

```python
import math
from datetime import datetime, timedelta

class SlidingWindowCounterRateLimiter:
    """Sliding window counter (hybrid) rate limiter"""
    
    def __init__(self, limit: int, window_seconds: int):
        self.limit = limit
        self.window_seconds = window_seconds
        self.requests = {}  # {key: {'prev_count': 0, 'prev_window_start': ts, 'curr_count': 0, 'curr_window_start': ts}}
        self.lock = threading.Lock()
    
    def allow_request(self, key: str) -> bool:
        """Weighted count ile sliding window"""
        with self.lock:
            now = datetime.utcnow()
            
            if key not in self.requests:
                self.requests[key] = {
                    'prev_count': 0,
                    'prev_window_start': now - timedelta(seconds=self.window_seconds),
                    'curr_count': 0,
                    'curr_window_start': now
                }
            
            data = self.requests[key]
            
            # Yeni pencereye geçiş kontrolü
            elapsed = (now - data['curr_window_start']).total_seconds()
            if elapsed >= self.window_seconds:
                # Pencereyi kaydır
                data['prev_count'] = data['curr_count']
                data['prev_window_start'] = data['curr_window_start']
                data['curr_count'] = 0
                data['curr_window_start'] = now
                elapsed = 0
            
            # Weighted count hesapla
            # Önceki pencereden kalan kısmın ağırlığı + mevcut pencere
            overlap_percentage = 1 - (elapsed / self.window_seconds)
            weighted_count = (data['prev_count'] * overlap_percentage) + data['curr_count']
            
            if weighted_count < self.limit:
                data['curr_count'] += 1
                return True
            
            return False

# Kullanım
limiter = SlidingWindowCounterRateLimiter(limit=10, window_seconds=60)
```

**Avantajları:**
- Düşük memory (sadece 2 counter)
- Burst problem minimize
- O(1) complexity

**Dezavantajları:**
- Approximate count (tam doğru değil)
- Kenar durumlar

### 4. Token Bucket Algorithm

![Token Bucket Algorithm](/assets/img/posts/token-bucket-algorithm-diagram.png)
*Token bucket rate limiting mechanism*

Bucket'ta tokenlar birikir, her istek bir token harcar. Burst'lere izin verir.

```python
import time

class TokenBucketRateLimiter:
    """Token bucket rate limiter"""
    
    def __init__(self, capacity: int, refill_rate: float):
        """
        Args:
            capacity: Bucket kapasitesi (max token sayısı)
            refill_rate: Saniye başına eklenecek token sayısı
        """
        self.capacity = capacity
        self.refill_rate = refill_rate
        self.buckets = {}  # {key: {'tokens': float, 'last_refill': timestamp}}
        self.lock = threading.Lock()
    
    def _refill(self, bucket: dict) -> None:
        """Bucket'ı yeniden doldur"""
        now = time.time()
        time_passed = now - bucket['last_refill']
        
        # Geçen süreye göre token ekle
        tokens_to_add = time_passed * self.refill_rate
        bucket['tokens'] = min(self.capacity, bucket['tokens'] + tokens_to_add)
        bucket['last_refill'] = now
    
    def allow_request(self, key: str, tokens: int = 1) -> bool:
        """
        İstek için token tüket
        
        Args:
            key: Kullanıcı identifier
            tokens: Tüketilecek token sayısı (farklı endpoint'ler için)
        
        Returns:
            İstek kabul edildi mi
        """
        with self.lock:
            # Bucket oluştur
            if key not in self.buckets:
                self.buckets[key] = {
                    'tokens': self.capacity,
                    'last_refill': time.time()
                }
            
            bucket = self.buckets[key]
            self._refill(bucket)
            
            # Token var mı kontrol et
            if bucket['tokens'] >= tokens:
                bucket['tokens'] -= tokens
                return True
            
            return False
    
    def get_info(self, key: str) -> dict:
        """Rate limit bilgileri"""
        with self.lock:
            if key not in self.buckets:
                return {
                    'capacity': self.capacity,
                    'available_tokens': self.capacity,
                    'refill_rate': self.refill_rate
                }
            
            bucket = self.buckets[key]
            self._refill(bucket)
            
            return {
                'capacity': self.capacity,
                'available_tokens': int(bucket['tokens']),
                'refill_rate': self.refill_rate
            }

# Kullanım
limiter = TokenBucketRateLimiter(capacity=10, refill_rate=1.0)  # 1 token/second

# Simülasyon: Burst trafiği
print("=== Burst Test ===")
for i in range(15):
    allowed = limiter.allow_request("user_789")
    info = limiter.get_info("user_789")
    print(f"Request {i+1}: {'✅' if allowed else '❌'} - Tokens: {info['available_tokens']}/{info['capacity']}")

time.sleep(5)  # 5 saniye bekle (5 token yenilenir)

print("\n=== After waiting 5 seconds ===")
for i in range(7):
    allowed = limiter.allow_request("user_789")
    info = limiter.get_info("user_789")
    print(f"Request {i+1}: {'✅' if allowed else '❌'} - Tokens: {info['available_tokens']}/{info['capacity']}")
```

**Avantajları:**
- Burst trafiğine izin verir
- Düz işleyiş (smooth rate)
- Esnek (farklı endpoint'ler için farklı cost)

**Dezavantajları:**
- Floating point hesaplamalar
- Burst abuse riski

### 5. Leaky Bucket Algorithm

![Leaky Bucket Algorithm](/assets/img/posts/leaky-bucket-rate-limiting.png)
*Leaky bucket constant rate output*

Token bucket'ın tersi. İstekleri queue'da tutar, sabit hızda işler.

```python
from collections import deque
import threading
import time

class LeakyBucketRateLimiter:
    """Leaky bucket rate limiter with queue"""
    
    def __init__(self, capacity: int, leak_rate: float):
        """
        Args:
            capacity: Queue kapasitesi (max request sayısı)
            leak_rate: Saniye başına işlenecek request sayısı
        """
        self.capacity = capacity
        self.leak_rate = leak_rate
        self.buckets = {}  # {key: {'queue': deque, 'last_leak': timestamp}}
        self.lock = threading.Lock()
    
    def _leak(self, bucket: dict) -> None:
        """Queue'dan istekleri sızdır (işle)"""
        now = time.time()
        time_passed = now - bucket['last_leak']
        
        # Geçen sürede kaç request işlenebilir
        leaks = int(time_passed * self.leak_rate)
        
        if leaks > 0:
            # Queue'dan request çıkar
            for _ in range(min(leaks, len(bucket['queue']))):
                bucket['queue'].popleft()
            
            bucket['last_leak'] = now
    
    def allow_request(self, key: str) -> bool:
        """İsteği queue'ya ekle"""
        with self.lock:
            # Bucket oluştur
            if key not in self.buckets:
                self.buckets[key] = {
                    'queue': deque(),
                    'last_leak': time.time()
                }
            
            bucket = self.buckets[key]
            self._leak(bucket)
            
            # Queue dolu mu?
            if len(bucket['queue']) < self.capacity:
                bucket['queue'].append(time.time())
                return True
            
            return False
    
    def get_info(self, key: str) -> dict:
        """Rate limit bilgileri"""
        with self.lock:
            if key not in self.buckets:
                return {
                    'capacity': self.capacity,
                    'queued': 0,
                    'leak_rate': self.leak_rate
                }
            
            bucket = self.buckets[key]
            self._leak(bucket)
            
            return {
                'capacity': self.capacity,
                'queued': len(bucket['queue']),
                'leak_rate': self.leak_rate
            }

# Kullanım
limiter = LeakyBucketRateLimiter(capacity=5, leak_rate=1.0)

for i in range(8):
    allowed = limiter.allow_request("user_101")
    info = limiter.get_info("user_101")
    print(f"Request {i+1}: {'✅ Queued' if allowed else '❌ Queue Full'} - "
          f"Queue: {info['queued']}/{info['capacity']}")
```

**Avantajları:**
- Sabit çıkış hızı (smooth traffic)
- Downstream koruması

**Dezavantajları:**
- Latency artışı (queue wait)
- Memory overhead (queue)

## Redis ile Distributed Rate Limiting

![Redis Distributed Rate Limiter](/assets/img/posts/redis-distributed-rate-limiter.png)
*Distributed rate limiting architecture with Redis*

### Redis Token Bucket Implementation

```python
import redis
import time
import math

class RedisTokenBucketRateLimiter:
    """Redis tabanlı distributed token bucket"""
    
    def __init__(self, redis_client: redis.Redis, capacity: int, refill_rate: float):
        self.redis = redis_client
        self.capacity = capacity
        self.refill_rate = refill_rate
    
    def _get_key(self, identifier: str) -> str:
        """Redis key oluştur"""
        return f"rate_limit:token_bucket:{identifier}"
    
    def allow_request(self, identifier: str, tokens: int = 1) -> tuple[bool, dict]:
        """
        Lua script ile atomic operation
        
        Returns:
            (allowed, info_dict)
        """
        key = self._get_key(identifier)
        now = time.time()
        
        # Lua script (atomic işlem garantisi)
        script = """
        local key = KEYS[1]
        local capacity = tonumber(ARGV[1])
        local refill_rate = tonumber(ARGV[2])
        local tokens_requested = tonumber(ARGV[3])
        local now = tonumber(ARGV[4])
        
        -- Mevcut state'i al
        local bucket = redis.call('HMGET', key, 'tokens', 'last_refill')
        local tokens = tonumber(bucket[1])
        local last_refill = tonumber(bucket[2])
        
        -- İlk istek
        if not tokens then
            tokens = capacity
            last_refill = now
        end
        
        -- Refill hesapla
        local time_passed = now - last_refill
        local tokens_to_add = time_passed * refill_rate
        tokens = math.min(capacity, tokens + tokens_to_add)
        
        -- Token tüket
        local allowed = 0
        if tokens >= tokens_requested then
            tokens = tokens - tokens_requested
            allowed = 1
        end
        
        -- State güncelle
        redis.call('HMSET', key, 'tokens', tokens, 'last_refill', now)
        redis.call('EXPIRE', key, 3600)  -- 1 saat TTL
        
        return {allowed, tokens}
        """
        
        result = self.redis.eval(
            script,
            1,  # Number of keys
            key,
            self.capacity,
            self.refill_rate,
            tokens,
            now
        )
        
        allowed = bool(result[0])
        remaining_tokens = int(result[1])
        
        return allowed, {
            'capacity': self.capacity,
            'remaining': remaining_tokens,
            'refill_rate': self.refill_rate
        }

# Setup
r = redis.Redis(host='localhost', port=6379, decode_responses=True)
limiter = RedisTokenBucketRateLimiter(r, capacity=100, refill_rate=10.0)

# Kullanım
allowed, info = limiter.allow_request("user_abc", tokens=5)
print(f"Allowed: {allowed}, Remaining: {info['remaining']}")
```

### Redis Sliding Window Implementation

```python
class RedisSlidingWindowRateLimiter:
    """Redis sorted set ile sliding window"""
    
    def __init__(self, redis_client: redis.Redis, limit: int, window_seconds: int):
        self.redis = redis_client
        self.limit = limit
        self.window_seconds = window_seconds
    
    def _get_key(self, identifier: str) -> str:
        return f"rate_limit:sliding_window:{identifier}"
    
    def allow_request(self, identifier: str) -> tuple[bool, dict]:
        """
        Sorted set ile timestamp tracking
        """
        key = self._get_key(identifier)
        now = time.time()
        window_start = now - self.window_seconds
        
        # Lua script
        script = """
        local key = KEYS[1]
        local window_start = tonumber(ARGV[1])
        local now = tonumber(ARGV[2])
        local limit = tonumber(ARGV[3])
        
        -- Eski istekleri sil
        redis.call('ZREMRANGEBYSCORE', key, '-inf', window_start)
        
        -- Mevcut istek sayısı
        local count = redis.call('ZCARD', key)
        
        local allowed = 0
        if count < limit then
            -- Yeni istek ekle (score = timestamp, member = unique id)
            redis.call('ZADD', key, now, now .. ':' .. math.random())
            redis.call('EXPIRE', key, 3600)
            allowed = 1
            count = count + 1
        end
        
        return {allowed, limit - count}
        """
        
        result = self.redis.eval(
            script,
            1,
            key,
            window_start,
            now,
            self.limit
        )
        
        allowed = bool(result[0])
        remaining = int(result[1])
        
        return allowed, {
            'limit': self.limit,
            'remaining': remaining,
            'reset': int(now + self.window_seconds)
        }

# Kullanım
limiter = RedisSlidingWindowRateLimiter(r, limit=100, window_seconds=60)
allowed, info = limiter.allow_request("api_key_xyz")
```

## FastAPI Integration

### Decorator ile Rate Limiting

```python
from fastapi import FastAPI, HTTPException, Request, Response
from functools import wraps
import asyncio

app = FastAPI()

# Global limiter
limiter = RedisTokenBucketRateLimiter(r, capacity=100, refill_rate=1.0)

def rate_limit(tokens: int = 1):
    """Rate limiting decorator"""
    def decorator(func):
        @wraps(func)
        async def wrapper(*args, **kwargs):
            # Request objesini bul
            request = None
            for arg in args:
                if isinstance(arg, Request):
                    request = arg
                    break
            
            if not request:
                raise ValueError("Request object not found")
            
            # Identifier (IP veya user ID)
            identifier = request.client.host
            
            # Rate limit kontrolü
            allowed, info = limiter.allow_request(identifier, tokens)
            
            if not allowed:
                raise HTTPException(
                    status_code=429,
                    detail="Too Many Requests",
                    headers={
                        "X-RateLimit-Limit": str(info['capacity']),
                        "X-RateLimit-Remaining": "0",
                        "Retry-After": "60"
                    }
                )
            
            # Response'a header ekle
            response = await func(*args, **kwargs)
            
            if isinstance(response, Response):
                response.headers["X-RateLimit-Limit"] = str(info['capacity'])
                response.headers["X-RateLimit-Remaining"] = str(info['remaining'])
            
            return response
        
        return wrapper
    return decorator

# Kullanım
@app.get("/api/data")
@rate_limit(tokens=1)
async def get_data(request: Request):
    return {"message": "Data fetched successfully"}

@app.post("/api/expensive-operation")
@rate_limit(tokens=10)  # Pahalı işlem daha fazla token harcar
async def expensive_operation(request: Request):
    return {"message": "Operation completed"}
```

### Middleware ile Global Rate Limiting

```python
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request
from starlette.responses import JSONResponse

class RateLimitMiddleware(BaseHTTPMiddleware):
    """Global rate limiting middleware"""
    
    def __init__(self, app, limiter, exclude_paths: list = None):
        super().__init__(app)
        self.limiter = limiter
        self.exclude_paths = exclude_paths or []
    
    async def dispatch(self, request: Request, call_next):
        # Exclude edilen path'leri kontrol et
        if request.url.path in self.exclude_paths:
            return await call_next(request)
        
        # Rate limit kontrolü
        identifier = request.client.host
        allowed, info = self.limiter.allow_request(identifier)
        
        if not allowed:
            return JSONResponse(
                status_code=429,
                content={"detail": "Too Many Requests"},
                headers={
                    "X-RateLimit-Limit": str(info['capacity']),
                    "X-RateLimit-Remaining": "0",
                    "X-RateLimit-Reset": str(info.get('reset', '')),
                    "Retry-After": "60"
                }
            )
        
        # İsteği işle
        response = await call_next(request)
        
        # Response header'ları ekle
        response.headers["X-RateLimit-Limit"] = str(info['capacity'])
        response.headers["X-RateLimit-Remaining"] = str(info['remaining'])
        
        return response

# Middleware ekle
app.add_middleware(
    RateLimitMiddleware,
    limiter=limiter,
    exclude_paths=["/health", "/metrics"]
)
```

### Tiered Rate Limiting (Plan Bazlı)

> SaaS ürünlerinde farklı plan seviyelerine göre değişken rate limit uygulamak yaygındır. Free, Pro, Enterprise gibi tier'lara göre limitleri ayarlayın.
{: .prompt-info }

```python
from enum import Enum
from pydantic import BaseModel

class UserPlan(str, Enum):
    FREE = "free"
    BASIC = "basic"
    PREMIUM = "premium"
    ENTERPRISE = "enterprise"

# Plan limitleri
RATE_LIMITS = {
    UserPlan.FREE: {'capacity': 10, 'refill_rate': 0.1},      # 10/dakika
    UserPlan.BASIC: {'capacity': 100, 'refill_rate': 1.0},    # 60/dakika
    UserPlan.PREMIUM: {'capacity': 1000, 'refill_rate': 10.0}, # 600/dakika
    UserPlan.ENTERPRISE: {'capacity': 10000, 'refill_rate': 100.0} # 6000/dakika
}

class User(BaseModel):
    id: str
    plan: UserPlan

async def get_current_user(request: Request) -> User:
    """Auth token'dan kullanıcıyı al"""
    # Simulated
    return User(id="user123", plan=UserPlan.PREMIUM)

@app.get("/api/premium-data")
async def get_premium_data(request: Request, user: User = Depends(get_current_user)):
    """Plan bazlı rate limiting"""
    
    # Kullanıcı planına göre limiter
    limits = RATE_LIMITS[user.plan]
    limiter = RedisTokenBucketRateLimiter(
        r,
        capacity=limits['capacity'],
        refill_rate=limits['refill_rate']
    )
    
    allowed, info = limiter.allow_request(f"user:{user.id}")
    
    if not allowed:
        raise HTTPException(
            status_code=429,
            detail=f"Rate limit exceeded for {user.plan.value} plan",
            headers={
                "X-RateLimit-Limit": str(info['capacity']),
                "X-RateLimit-Remaining": "0",
                "X-RateLimit-Reset": str(info.get('reset', '')),
                "Retry-After": "60"
            }
        )
    
    return {
        "data": "Premium content",
        "plan": user.plan.value,
        "rate_limit": info
    }
```

## Best Practices

> Rate limit aşımında kullanıcıya açık bilgi verin. X-RateLimit-* header'ları ve Retry-After ile ne zaman yeniden deneyebileceğini bildirin.
{: .prompt-warning }

### 1. Informative Error Responses

```python
@app.exception_handler(HTTPException)
async def rate_limit_exception_handler(request: Request, exc: HTTPException):
    """429 hatası için özel response"""
    
    if exc.status_code == 429:
        return JSONResponse(
            status_code=429,
            content={
                "error": {
                    "code": "RATE_LIMIT_EXCEEDED",
                    "message": "You have exceeded your request rate limit",
                    "details": {
                        "limit": exc.headers.get("X-RateLimit-Limit"),
                        "reset_at": exc.headers.get("X-RateLimit-Reset"),
                        "retry_after": exc.headers.get("Retry-After"),
                        "docs": "https://api.example.com/docs/rate-limits"
                    }
                }
            },
            headers=dict(exc.headers)
        )
    
    return exc
```

### 2. Graceful Degradation

```python
async def get_data_with_fallback(identifier: str):
    """Rate limit aşıldığında cached data döndür"""
    
    allowed, info = limiter.allow_request(identifier)
    
    if allowed:
        # Fresh data
        data = await fetch_fresh_data()
        await cache.set(f"data:{identifier}", data, expire=300)
        return data, {"source": "fresh", "rate_limit": info}
    else:
        # Cached data (graceful degradation)
        cached_data = await cache.get(f"data:{identifier}")
        
        if cached_data:
            return cached_data, {
                "source": "cached",
                "message": "Rate limit exceeded, serving cached data",
                "rate_limit": info
            }
        
        # Cache de yok, hata döndür
        raise HTTPException(
            status_code=429,
            detail="Rate limit exceeded and no cached data available"
        )
```

### 3. Dynamic Rate Limiting

```python
async def get_dynamic_limit(user: User, endpoint: str) -> dict:
    """Kullanıcı davranışına göre dinamik limit"""
    
    # Son 24 saatteki abuse skorunu hesapla
    abuse_score = await calculate_abuse_score(user.id)
    
    # Base limit (plan bazlı)
    base_limits = RATE_LIMITS[user.plan]
    
    # Abuse varsa limiti düşür
    if abuse_score > 0.8:
        capacity = int(base_limits['capacity'] * 0.5)  # %50 azalt
        refill_rate = base_limits['refill_rate'] * 0.5
    elif abuse_score > 0.5:
        capacity = int(base_limits['capacity'] * 0.75)  # %25 azalt
        refill_rate = base_limits['refill_rate'] * 0.75
    else:
        capacity = base_limits['capacity']
        refill_rate = base_limits['refill_rate']
    
    # Endpoint-specific multipliers
    endpoint_multipliers = {
        "/api/search": 2.0,      # Search daha cömert
        "/api/export": 0.5,      # Export daha kısıtlı
        "/api/delete": 0.1       # Delete çok kısıtlı
    }
    
    multiplier = endpoint_multipliers.get(endpoint, 1.0)
    
    return {
        'capacity': int(capacity * multiplier),
        'refill_rate': refill_rate * multiplier
    }

async def calculate_abuse_score(user_id: str) -> float:
    """Abuse skoru hesapla (0.0-1.0)"""
    
    # Metrikler
    metrics = {
        'failed_requests_24h': await redis.get(f"metrics:{user_id}:failed_24h") or 0,
        'rate_limit_hits_24h': await redis.get(f"metrics:{user_id}:rate_limit_24h") or 0,
        'unusual_patterns': await detect_unusual_patterns(user_id)
    }
    
    # Skor hesapla
    score = (
        (int(metrics['failed_requests_24h']) / 1000) * 0.4 +
        (int(metrics['rate_limit_hits_24h']) / 100) * 0.4 +
        (1.0 if metrics['unusual_patterns'] else 0.0) * 0.2
    )
    
    return min(1.0, score)
```

### 4. Monitoring ve Alerting

```python
from prometheus_client import Counter, Histogram

# Metrics
rate_limit_requests = Counter(
    'rate_limit_requests_total',
    'Total rate limit checks',
    ['identifier', 'allowed']
)

rate_limit_latency = Histogram(
    'rate_limit_check_duration_seconds',
    'Rate limit check latency'
)

class MonitoredRateLimiter:
    """Metrics ile rate limiter"""
    
    def __init__(self, limiter):
        self.limiter = limiter
    
    def allow_request(self, identifier: str, tokens: int = 1):
        """Monitored rate limit check"""
        
        with rate_limit_latency.time():
            allowed, info = self.limiter.allow_request(identifier, tokens)
        
        # Metrics kaydet
        rate_limit_requests.labels(
            identifier=identifier,
            allowed=str(allowed)
        ).inc()
        
        # Alert koşulları
        if not allowed:
            # Log
            logger.warning(
                f"Rate limit exceeded",
                extra={
                    'identifier': identifier,
                    'tokens_requested': tokens,
                    'remaining': info.get('remaining', 0)
                }
            )
            
            # Çok fazla rate limit varsa alert
            await check_rate_limit_spike(identifier)
        
        return allowed, info

async def check_rate_limit_spike(identifier: str):
    """Rate limit spike detection"""
    
    key = f"rate_limit_spike:{identifier}"
    count = await redis.incr(key)
    await redis.expire(key, 60)  # 1 dakika pencere
    
    if count > 50:  # 1 dakikada 50+ rate limit
        await send_alert(
            f"Rate limit spike detected for {identifier}",
            severity="warning"
        )
```

## Sonuç

Python ile rate limiting, API güvenliği ve performansının temel taşlarından biridir. Bu yazıda öğrendiklerimiz:

- **Algoritmalar**: Fixed window, sliding window, token bucket, leaky bucket
- **Redis Integration**: Distributed rate limiting çözümleri
- **FastAPI**: Decorator, middleware ve tiered limiting
- **Best Practices**: Monitoring, dynamic limits, graceful degradation

### Önemli Noktalar

1. **Doğru Algoritma Seçimi**: İhtiyacınıza göre (burst vs smooth)
2. **Redis ile Scalability**: Distributed sistemlerde consistency
3. **Informative Responses**: Kullanıcıya net bilgi verin
4. **Monitoring**: Rate limit metrics'leri takip edin
5. **Graceful Degradation**: Limit aşımında alternatif sunun

### Algorithm Seçim Rehberi

| Algoritma | Kullanım Durumu | Avantaj | Dezavantaj |
|-----------|-----------------|---------|------------|
| **Fixed Window** | Basit use case'ler | En basit, düşük overhead | Burst problem |
| **Sliding Window Log** | Tam doğruluk gerekli | Kesin sayım | Yüksek memory |
| **Sliding Window Counter** | Balanced çözüm | Memory efficient | Approximate |
| **Token Bucket** | Burst izni gerekli | Esnek, burst friendly | Abuse riski |
| **Leaky Bucket** | Sabit çıkış hızı | Smooth traffic | Latency artışı |

### Kaynaklar

- [IETF RFC 6585 - HTTP 429 Too Many Requests](https://tools.ietf.org/html/rfc6585) - HTTP 429 standardı
- [Redis Rate Limiting Patterns](https://redis.io/docs/manual/patterns/rate-limiting/) - Redis resmi rate limiting patterns
- [Stripe API Rate Limits](https://stripe.com/docs/rate-limits) - Stripe'ın rate limiting yaklaşımı
- [FastAPI Rate Limiting](https://fastapi.tiangolo.com/) - FastAPI middleware örnekleri

Artık production-ready, scalable ve güvenli rate limiting sistemleri kurabilirsiniz!
