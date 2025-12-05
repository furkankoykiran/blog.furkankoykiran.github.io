---
title: "Redis ile Önbellekleme ve Oturum Yönetimi: Performansı Maksimuma Çıkarın"
description: "Redis ile yüksek performanslı önbellekleme ve oturum yönetimi. Caching stratejileri, pub/sub, data structures ve production best practices."
date: 2024-12-10 09:00:00 +0300
categories: [Backend, Cache]
tags: [redis, cache, session, performance, scalability, memory]
image:
  path: /assets/img/posts/redis-architecture-diagram.png
  alt: "Redis Mimarisi ve In-Memory Database"
---

Redis (REmote DIctionary Server), yüksek performanslı, in-memory key-value veri deposudur. Mikroservis mimarilerinde önbellekleme, oturum yönetimi, message queue ve real-time analytics gibi birçok kritik görevde kullanılır. Bu yazıda Redis'in temellerinden production-ready kullanımına kadar detaylı bir rehber sunacağız.

## Redis Neden Bu Kadar Popüler?

Redis'in popülaritesinin arkasında birçok teknik avantaj bulunur:

### Performans

- **In-Memory Storage**: Tüm veriler RAM'de tutulur, disk I/O yok
- **Single-Threaded**: Lock mekanizmasına gerek yok, atomic operasyonlar
- **10-100x Hız Artışı**: Database sorgularına göre mikrosaniye mertebesinde yanıt süreleri
- **Sub-millisecond Latency**: Ortalama < 1ms okuma/yazma süresi

### Veri Yapıları

Redis sadece string değil, birçok gelişmiş veri yapısı sunar:
- **Strings**: Basit key-value çiftleri
- **Hashes**: Object benzeri yapılar
- **Lists**: FIFO/LIFO kuyruklar
- **Sets**: Benzersiz değer koleksiyonları
- **Sorted Sets**: Sıralı kümeler (leaderboard için ideal)
- **Bitmaps**: Bit-level operasyonlar
- **HyperLogLogs**: Kardinalite tahmini
- **Streams**: Log benzeri veri yapısı

### Kullanım Alanları

- **Caching**: Database query sonuçlarını önbellekleme
- **Session Store**: Kullanıcı oturumlarını merkezi yönetim
- **Rate Limiting**: API rate limiter implementasyonu
- **Leaderboards**: Gaming skorlar, ranking sistemleri
- **Real-time Analytics**: Sayaçlar, metrikler
- **Pub/Sub**: Message broker, event streaming
- **Geospatial**: Lokasyon bazlı sorgular

## Redis Kurulumu

### Ubuntu/Debian için Kurulum

```bash
# Redis resmi repository'yi ekle
curl -fsSL https://packages.redis.io/gpg | sudo gpg --dearmor -o /usr/share/keyrings/redis-archive-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/redis-archive-keyring.gpg] https://packages.redis.io/deb $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/redis.list

# Redis'i kur
sudo apt update
sudo apt install redis-server redis-tools

# Redis'i başlat
sudo systemctl start redis-server
sudo systemctl enable redis-server

# Kurulumu test et
redis-cli ping
# Yanıt: PONG
```

### Docker ile Kurulum

```bash
# Redis container çalıştır
docker run -d \
  --name redis \
  -p 6379:6379 \
  -v redis-data:/data \
  redis:7-alpine redis-server --appendonly yes

# Persistent data ile
docker run -d \
  --name redis-persistent \
  -p 6379:6379 \
  -v /my/redis/data:/data \
  -v /my/redis/conf:/usr/local/etc/redis \
  redis:7-alpine redis-server /usr/local/etc/redis/redis.conf

# Redis CLI'ya bağlan
docker exec -it redis redis-cli
```

### Docker Compose ile Production Setup

```yaml
version: '3.8'

services:
  redis:
    image: redis:7-alpine
    container_name: redis-server
    command: redis-server --requirepass ${REDIS_PASSWORD} --maxmemory 512mb --maxmemory-policy allkeys-lru
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
      - ./redis.conf:/usr/local/etc/redis/redis.conf
    environment:
      - REDIS_REPLICATION_MODE=master
    networks:
      - app-network
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 3
    restart: unless-stopped

  redis-commander:
    image: rediscommander/redis-commander:latest
    container_name: redis-ui
    environment:
      - REDIS_HOSTS=local:redis:6379:0:${REDIS_PASSWORD}
    ports:
      - "8081:8081"
    depends_on:
      - redis
    networks:
      - app-network

volumes:
  redis-data:

networks:
  app-network:
    driver: bridge
```
{: file="docker-compose.yml" }

## Redis Temel Komutlar

### String Operasyonları

```bash
# SET ve GET
SET user:1000:name "John Doe"
GET user:1000:name

# EX ile expiration (saniye)
SET session:abc123 "user_data" EX 3600

# PX ile expiration (milisaniye)
SET token:xyz789 "auth_token" PX 300000

# NX: Sadece key yoksa set et
SET product:100:stock 50 NX

# XX: Sadece key varsa update et
SET product:100:stock 45 XX

# GETSET: Get edip yeni değer set et
GETSET counter:page_views 0

# MSET ve MGET (Multiple)
MSET user:1:name "Alice" user:1:age "30" user:1:city "Istanbul"
MGET user:1:name user:1:age user:1:city

# INCR ve DECR (Atomic counter)
INCR page:views
INCRBY page:views 10
DECR product:stock
DECRBY product:stock 5

# APPEND
SET msg "Hello"
APPEND msg " World"
GET msg  # "Hello World"

# STRLEN
STRLEN msg  # 11

# Key expire yönetimi
EXPIRE user:session 3600
TTL user:session
PERSIST user:session  # Expiration'ı kaldır
```

![Redis Caching Pattern](/assets/img/posts/redis-caching-pattern.png)
_Redis ile Caching Stratejileri_

### Hash Operasyonları

Hash'ler, object-like yapılar oluşturmak için idealdir:

```bash
# HSET ve HGET
HSET user:1000 name "John Doe" email "john@example.com" age 30
HGET user:1000 name

# HMSET (Multiple field set - deprecated, HSET kullan)
HSET user:1000 name "John" email "john@ex.com" age 30

# HMGET (Multiple field get)
HMGET user:1000 name email age

# HGETALL (Tüm field'ları getir)
HGETALL user:1000

# HEXISTS (Field var mı kontrol et)
HEXISTS user:1000 name

# HDEL (Field sil)
HDEL user:1000 age

# HINCRBY (Numeric field artır)
HINCRBY user:1000 login_count 1

# HKEYS ve HVALS
HKEYS user:1000  # Tüm field isimlerini getir
HVALS user:1000  # Tüm değerleri getir

# HLEN (Field sayısı)
HLEN user:1000
```

### List Operasyonları

List'ler FIFO/LIFO queue implementasyonu için kullanılır:

```bash
# LPUSH ve RPUSH (Başa/Sona ekle)
LPUSH queue:tasks "task1"
RPUSH queue:tasks "task2" "task3"

# LPOP ve RPOP (Baştan/Sondan çıkar)
LPOP queue:tasks
RPOP queue:tasks

# LRANGE (Range query)
LRANGE queue:tasks 0 -1  # Tüm elemanları getir
LRANGE queue:tasks 0 10  # İlk 10 elemanı getir

# LLEN (List uzunluğu)
LLEN queue:tasks

# LINDEX (Index'e göre eleman)
LINDEX queue:tasks 0

# LSET (Index'e set et)
LSET queue:tasks 0 "updated_task"

# LTRIM (Listeyi trim et)
LTRIM queue:tasks 0 99  # İlk 100 elemanı tut

# BLPOP ve BRPOP (Blocking pop - queue için ideal)
BLPOP queue:tasks 30  # 30 saniye bekle
```

### Set Operasyonları

Set'ler benzersiz değerler içindir:

```bash
# SADD (Set'e ekle)
SADD tags:post:100 "python" "redis" "cache"

# SMEMBERS (Tüm elemanları getir)
SMEMBERS tags:post:100

# SISMEMBER (Eleman var mı kontrol)
SISMEMBER tags:post:100 "python"

# SREM (Eleman sil)
SREM tags:post:100 "cache"

# SCARD (Set boyutu)
SCARD tags:post:100

# Set operasyonları
SADD set1 "a" "b" "c"
SADD set2 "b" "c" "d"

SINTER set1 set2  # Kesişim: ["b", "c"]
SUNION set1 set2  # Birleşim: ["a", "b", "c", "d"]
SDIFF set1 set2   # Fark: ["a"]

# SPOP (Random eleman çıkar)
SPOP tags:post:100

# SRANDMEMBER (Random eleman getir, silme)
SRANDMEMBER tags:post:100 2
```

### Sorted Set Operasyonları

Sorted set'ler score ile sıralanmış kümelerdir (leaderboard için mükemmel):

```bash
# ZADD (Score ile ekle)
ZADD leaderboard:game1 1000 "player1"
ZADD leaderboard:game1 1500 "player2" 1200 "player3"

# ZRANGE (Score'a göre sıralı getir)
ZRANGE leaderboard:game1 0 -1 WITHSCORES
ZRANGE leaderboard:game1 0 9 WITHSCORES  # Top 10

# ZREVRANGE (Ters sıralı - en yüksekten başla)
ZREVRANGE leaderboard:game1 0 9 WITHSCORES

# ZSCORE (Belirli member'ın score'u)
ZSCORE leaderboard:game1 "player1"

# ZINCRBY (Score artır)
ZINCRBY leaderboard:game1 50 "player1"

# ZRANK ve ZREVRANK (Sıralama)
ZRANK leaderboard:game1 "player1"      # 0'dan başlayan rank
ZREVRANK leaderboard:game1 "player1"   # Ters sıralı rank

# ZCOUNT (Score aralığındaki eleman sayısı)
ZCOUNT leaderboard:game1 1000 2000

# ZRANGEBYSCORE (Score aralığına göre getir)
ZRANGEBYSCORE leaderboard:game1 1000 1500 WITHSCORES

# ZREM (Member sil)
ZREM leaderboard:game1 "player1"

# ZREMRANGEBYRANK (Rank aralığını sil)
ZREMRANGEBYRANK leaderboard:game1 10 -1  # 11. sıradan sonrakileri sil
```

![Redis Data Structures](/assets/img/posts/redis-data-structures.png)
_Redis Veri Yapıları: String, Hash, List, Set, Sorted Set_

## Python ile Redis Kullanımı

### Kurulum ve Bağlantı

```bash
pip install redis redis-py-cluster
```
{: file="bash" }

```python
import redis
from redis import Redis, ConnectionPool
from typing import Optional, Dict, List
import json

# Basit bağlantı
r = redis.Redis(
    host='localhost',
    port=6379,
    db=0,
    decode_responses=True  # String olarak decode et
)

# Connection pool ile (production için önerilen)
pool = ConnectionPool(
    host='localhost',
    port=6379,
    db=0,
    max_connections=50,
    decode_responses=True
)
r = Redis(connection_pool=pool)

# Authentication ile
r = redis.Redis(
    host='localhost',
    port=6379,
    password='your_password',
    db=0,
    decode_responses=True
)

# Redis URL ile
r = redis.from_url('redis://:password@localhost:6379/0')

# Bağlantıyı test et
try:
    r.ping()
    print("Redis bağlantısı başarılı!")
except redis.ConnectionError:
    print("Redis'e bağlanılamıyor!")
```
{: file="redis_connection.py" }

### Cache Wrapper Sınıfı

```python
from functools import wraps
import hashlib
import pickle
from typing import Any, Callable

class RedisCache:
    """Redis cache wrapper sınıfı"""
    
    def __init__(self, redis_client: Redis, default_ttl: int = 3600):
        self.redis = redis_client
        self.default_ttl = default_ttl
    
    def _make_key(self, prefix: str, *args, **kwargs) -> str:
        """Cache key oluştur"""
        key_parts = [prefix] + [str(arg) for arg in args]
        if kwargs:
            key_parts.append(str(sorted(kwargs.items())))
        key_string = ":".join(key_parts)
        return f"cache:{hashlib.md5(key_string.encode()).hexdigest()}"
    
    def get(self, key: str) -> Optional[Any]:
        """Cache'den veri al"""
        data = self.redis.get(key)
        if data:
            return pickle.loads(data)
        return None
    
    def set(self, key: str, value: Any, ttl: Optional[int] = None) -> bool:
        """Cache'e veri kaydet"""
        ttl = ttl or self.default_ttl
        serialized = pickle.dumps(value)
        return self.redis.setex(key, ttl, serialized)
    
    def delete(self, key: str) -> bool:
        """Cache'den sil"""
        return bool(self.redis.delete(key))
    
    def exists(self, key: str) -> bool:
        """Key var mı kontrol et"""
        return bool(self.redis.exists(key))
    
    def clear_pattern(self, pattern: str) -> int:
        """Pattern'e uyan tüm key'leri sil"""
        keys = self.redis.keys(pattern)
        if keys:
            return self.redis.delete(*keys)
        return 0
    
    def cache_function(self, ttl: Optional[int] = None, prefix: str = "func"):
        """Decorator: Function sonucunu cache'le"""
        def decorator(func: Callable) -> Callable:
            @wraps(func)
            def wrapper(*args, **kwargs):
                # Cache key oluştur
                cache_key = self._make_key(
                    f"{prefix}:{func.__name__}",
                    *args,
                    **kwargs
                )
                
                # Cache'den dene
                cached_result = self.get(cache_key)
                if cached_result is not None:
                    print(f"Cache HIT: {cache_key}")
                    return cached_result
                
                # Cache MISS: Fonksiyonu çalıştır
                print(f"Cache MISS: {cache_key}")
                result = func(*args, **kwargs)
                
                # Sonucu cache'le
                self.set(cache_key, result, ttl or self.default_ttl)
                
                return result
            
            return wrapper
        return decorator


# Kullanım örneği
cache = RedisCache(r, default_ttl=300)

@cache.cache_function(ttl=600, prefix="user")
def get_user_from_db(user_id: int) -> Dict:
    """Simulated database query"""
    import time
    time.sleep(2)  # Yavaş DB sorgusu simülasyonu
    return {
        "id": user_id,
        "name": "John Doe",
        "email": "john@example.com"
    }

# İlk çağrı: DB'den gelir (2 saniye)
user = get_user_from_db(1)

# İkinci çağrı: Cache'den gelir (milisaniyeler)
user = get_user_from_db(1)
```
{: file="cache_wrapper.py" }

### FastAPI ile Cache Entegrasyonu

```python
from fastapi import FastAPI, Depends
from pydantic import BaseModel
import redis.asyncio as aioredis
from typing import Optional
import json

app = FastAPI()

# Async Redis client
redis_client: Optional[aioredis.Redis] = None

@app.on_event("startup")
async def startup():
    global redis_client
    redis_client = await aioredis.from_url(
        "redis://localhost:6379",
        encoding="utf-8",
        decode_responses=True
    )

@app.on_event("shutdown")
async def shutdown():
    if redis_client:
        await redis_client.close()

class Product(BaseModel):
    id: int
    name: str
    price: float
    stock: int

async def get_redis() -> aioredis.Redis:
    """Redis dependency"""
    return redis_client

@app.get("/products/{product_id}")
async def get_product(
    product_id: int,
    redis: aioredis.Redis = Depends(get_redis)
):
    """Product getir - Cache stratejisi ile"""
    cache_key = f"product:{product_id}"
    
    # Cache'den dene
    cached = await redis.get(cache_key)
    if cached:
        return {"source": "cache", "data": json.loads(cached)}
    
    # DB'den çek (simulated)
    product = Product(
        id=product_id,
        name=f"Product {product_id}",
        price=99.99,
        stock=100
    )
    
    # Cache'e kaydet (1 saat)
    await redis.setex(
        cache_key,
        3600,
        product.model_dump_json()
    )
    
    return {"source": "database", "data": product}

@app.put("/products/{product_id}")
async def update_product(
    product_id: int,
    product: Product,
    redis: aioredis.Redis = Depends(get_redis)
):
    """Product güncelle ve cache'i invalidate et"""
    # DB'yi güncelle (simulated)
    
    # Cache'i invalidate et
    await redis.delete(f"product:{product_id}")
    
    # İlgili cache pattern'lerini temizle
    keys = await redis.keys(f"product:list:*")
    if keys:
        await redis.delete(*keys)
    
    return {"message": "Product updated", "invalidated_cache": len(keys) + 1}

@app.post("/cache/flush")
async def flush_cache(redis: aioredis.Redis = Depends(get_redis)):
    """Tüm cache'i temizle (dikkatli kullan!)"""
    await redis.flushdb()
    return {"message": "Cache flushed"}
```
{: file="fastapi_cache.py" }

![Redis Session Management](/assets/img/posts/redis-session-management.png)
_Redis ile Mikroservislerde Session Yönetimi_

## Session Management

### Flask ile Session Store

```python
from flask import Flask, session
from flask_session import Session
import redis

app = Flask(__name__)

# Session configuration
app.config['SECRET_KEY'] = 'your-secret-key-here'
app.config['SESSION_TYPE'] = 'redis'
app.config['SESSION_PERMANENT'] = False
app.config['SESSION_USE_SIGNER'] = True
app.config['SESSION_KEY_PREFIX'] = 'session:'
app.config['SESSION_REDIS'] = redis.Redis(
    host='localhost',
    port=6379,
    db=0
)

Session(app)

@app.route('/login', methods=['POST'])
def login():
    # User authentication
    user_id = 12345
    
    # Session'a kaydet
    session['user_id'] = user_id
    session['username'] = 'johndoe'
    session['roles'] = ['user', 'admin']
    
    return {'message': 'Logged in successfully'}

@app.route('/profile')
def profile():
    # Session'dan oku
    user_id = session.get('user_id')
    if not user_id:
        return {'error': 'Not authenticated'}, 401
    
    return {
        'user_id': user_id,
        'username': session.get('username'),
        'roles': session.get('roles')
    }

@app.route('/logout', methods=['POST'])
def logout():
    session.clear()
    return {'message': 'Logged out successfully'}
```
{: file="flask_session.py" }

### Custom Session Manager

```python
import uuid
import json
from datetime import timedelta
from typing import Optional, Dict, Any

class SessionManager:
    """Redis-based session manager"""
    
    def __init__(self, redis_client: Redis, prefix: str = "session"):
        self.redis = redis_client
        self.prefix = prefix
        self.default_ttl = timedelta(hours=24)
    
    def create_session(self, user_id: int, data: Dict[str, Any]) -> str:
        """Yeni session oluştur"""
        session_id = str(uuid.uuid4())
        session_key = f"{self.prefix}:{session_id}"
        
        session_data = {
            'user_id': user_id,
            'created_at': datetime.utcnow().isoformat(),
            **data
        }
        
        self.redis.setex(
            session_key,
            self.default_ttl,
            json.dumps(session_data)
        )
        
        return session_id
    
    def get_session(self, session_id: str) -> Optional[Dict]:
        """Session verisini getir"""
        session_key = f"{self.prefix}:{session_id}"
        data = self.redis.get(session_key)
        
        if data:
            # TTL'i yenile (sliding expiration)
            self.redis.expire(session_key, self.default_ttl)
            return json.loads(data)
        
        return None
    
    def update_session(self, session_id: str, data: Dict[str, Any]) -> bool:
        """Session'ı güncelle"""
        session_key = f"{self.prefix}:{session_id}"
        current_data = self.get_session(session_id)
        
        if not current_data:
            return False
        
        current_data.update(data)
        current_data['updated_at'] = datetime.utcnow().isoformat()
        
        self.redis.setex(
            session_key,
            self.default_ttl,
            json.dumps(current_data)
        )
        
        return True
    
    def delete_session(self, session_id: str) -> bool:
        """Session'ı sil (logout)"""
        session_key = f"{self.prefix}:{session_id}"
        return bool(self.redis.delete(session_key))
    
    def get_user_sessions(self, user_id: int) -> List[str]:
        """User'ın tüm session'larını getir"""
        pattern = f"{self.prefix}:*"
        sessions = []
        
        for key in self.redis.scan_iter(match=pattern, count=100):
            data = self.redis.get(key)
            if data:
                session_data = json.loads(data)
                if session_data.get('user_id') == user_id:
                    sessions.append(key.split(':')[-1])
        
        return sessions
    
    def invalidate_user_sessions(self, user_id: int) -> int:
        """User'ın tüm session'larını sil"""
        sessions = self.get_user_sessions(user_id)
        count = 0
        
        for session_id in sessions:
            if self.delete_session(session_id):
                count += 1
        
        return count
```
{: file="session_manager.py" }

## Rate Limiting

### Token Bucket Algorithm

```python
import time
from typing import Tuple

class RateLimiter:
    """Redis-based rate limiter (Token Bucket)"""
    
    def __init__(self, redis_client: Redis):
        self.redis = redis_client
    
    def is_allowed(
        self,
        identifier: str,
        max_requests: int = 100,
        window_seconds: int = 60
    ) -> Tuple[bool, Dict]:
        """Rate limit kontrolü"""
        key = f"ratelimit:{identifier}"
        current_time = int(time.time())
        
        # Lua script (atomic operation)
        lua_script = """
        local key = KEYS[1]
        local max_requests = tonumber(ARGV[1])
        local window = tonumber(ARGV[2])
        local current_time = tonumber(ARGV[3])
        
        local count = redis.call('GET', key)
        
        if not count then
            redis.call('SETEX', key, window, 1)
            return {1, max_requests - 1, window}
        end
        
        count = tonumber(count)
        
        if count < max_requests then
            redis.call('INCR', key)
            local ttl = redis.call('TTL', key)
            return {1, max_requests - count - 1, ttl}
        end
        
        local ttl = redis.call('TTL', key)
        return {0, 0, ttl}
        """
        
        allowed, remaining, reset = self.redis.eval(
            lua_script,
            1,
            key,
            max_requests,
            window_seconds,
            current_time
        )
        
        return bool(allowed), {
            'limit': max_requests,
            'remaining': int(remaining),
            'reset': int(reset)
        }

# FastAPI middleware örneği
from fastapi import Request, HTTPException
from starlette.middleware.base import BaseHTTPMiddleware

class RateLimitMiddleware(BaseHTTPMiddleware):
    def __init__(self, app, redis_client: Redis):
        super().__init__(app)
        self.limiter = RateLimiter(redis_client)
    
    async def dispatch(self, request: Request, call_next):
        # Client IP'ye göre rate limit
        client_ip = request.client.host
        
        allowed, info = self.limiter.is_allowed(
            identifier=client_ip,
            max_requests=100,
            window_seconds=60
        )
        
        if not allowed:
            raise HTTPException(
                status_code=429,
                detail=f"Rate limit exceeded. Try again in {info['reset']} seconds",
                headers={
                    'X-RateLimit-Limit': str(info['limit']),
                    'X-RateLimit-Remaining': '0',
                    'X-RateLimit-Reset': str(info['reset'])
                }
            )
        
        response = await call_next(request)
        
        # Rate limit headers ekle
        response.headers['X-RateLimit-Limit'] = str(info['limit'])
        response.headers['X-RateLimit-Remaining'] = str(info['remaining'])
        response.headers['X-RateLimit-Reset'] = str(info['reset'])
        
        return response

# Kullanım
app.add_middleware(RateLimitMiddleware, redis_client=r)
```
{: file="rate_limiter.py" }

## Production Best Practices

### 1. Redis Configuration (redis.conf)

```conf
# Memory management
maxmemory 2gb
maxmemory-policy allkeys-lru

# Persistence (RDB + AOF)
save 900 1
save 300 10
save 60 10000
appendonly yes
appendfsync everysec

# Security
requirepass your_strong_password_here
rename-command FLUSHDB ""
rename-command FLUSHALL ""
rename-command CONFIG "CONFIG_abc123"

# Performance
tcp-backlog 511
timeout 300
tcp-keepalive 60
databases 16

# Logging
loglevel notice
logfile /var/log/redis/redis-server.log
```
{: file="redis.conf" }

### 2. Monitoring

```python
def get_redis_info(redis_client: Redis) -> Dict:
    """Redis metrics"""
    info = redis_client.info()
    
    return {
        'version': info['redis_version'],
        'uptime_days': info['uptime_in_days'],
        'connected_clients': info['connected_clients'],
        'used_memory': f"{info['used_memory_human']}",
        'used_memory_peak': f"{info['used_memory_peak_human']}",
        'total_commands_processed': info['total_commands_processed'],
        'instantaneous_ops_per_sec': info['instantaneous_ops_per_sec'],
        'keyspace_hits': info['keyspace_hits'],
        'keyspace_misses': info['keyspace_misses'],
        'hit_rate': round(
            info['keyspace_hits'] / 
            (info['keyspace_hits'] + info['keyspace_misses']) * 100, 
            2
        ) if (info['keyspace_hits'] + info['keyspace_misses']) > 0 else 0
    }
```
{: file="redis_monitoring.py" }

### 3. Connection Pooling

```python
# Connection pool best practices
pool = redis.ConnectionPool(
    host='localhost',
    port=6379,
    password='your_password',
    db=0,
    max_connections=50,
    socket_timeout=5,
    socket_connect_timeout=5,
    socket_keepalive=True,
    health_check_interval=30,
    decode_responses=True
)

r = redis.Redis(connection_pool=pool)
```
{: file="connection_pool.py" }

### 4. Error Handling

```python
from redis.exceptions import (
    ConnectionError,
    TimeoutError,
    RedisError
)

def safe_redis_operation(func):
    """Redis operation wrapper with error handling"""
    @wraps(func)
    def wrapper(*args, **kwargs):
        try:
            return func(*args, **kwargs)
        except ConnectionError:
            logger.error("Redis connection error")
            # Fallback logic
            return None
        except TimeoutError:
            logger.error("Redis timeout")
            return None
        except RedisError as e:
            logger.error(f"Redis error: {e}")
            return None
    return wrapper

@safe_redis_operation
def get_from_cache(key: str):
    return r.get(key)
```
{: file="error_handling.py" }

## Sonuç

Redis, modern web uygulamalarında vazgeçilmez bir bileşen haline gelmiştir. Önbellekleme ve oturum yönetimi gibi temel kullanımların ötesinde, rate limiting, leaderboard, pub/sub ve real-time analytics gibi birçok advanced kullanım senaryosu sunar.

Bu yazıda öğrendikleriniz:
- Redis kurulumu ve temel komutlar
- Tüm veri yapıları (String, Hash, List, Set, Sorted Set)
- Python ile Redis entegrasyonu
- Cache stratejileri ve function decorator
- Session management implementasyonu
- Rate limiting algoritmaları
- Production best practices

### Önerilen Kaynaklar

- [Redis Resmi Dokümantasyonu](https://redis.io/docs/)
- [Redis Commands](https://redis.io/commands/)
- [redis-py Documentation](https://redis-py.readthedocs.io/)
- [Redis University](https://university.redis.io/)
- [Redis Design Patterns](https://redis.io/docs/manual/patterns/)

Bir sonraki yazımızda Traefik ile reverse proxy ve SSL yönetimini inceleyeceğiz. Takipte kalın!
