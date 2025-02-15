---
title: "Zarif Kapanış ve Sinyal İşleme - Graceful Shutdown Rehberi"
date: 2025-02-15 09:00:00 +0300
categories: [Python, DevOps]
tags: [python, signals, graceful-shutdown, sigterm, sigint, kubernetes, systemd, lifecycle, cleanup, production]
image:
  path: /assets/img/posts/graceful-shutdown-flow.png
  alt: "Graceful Shutdown İş Akışı Diagramı"
---

Production ortamında çalışan uygulamalar, kapatılırken veya yeniden başlatılırken **zarif bir şekilde** (gracefully) kapanmalıdır. Aksi takdirde, devam eden istekler yarıda kesilir, database bağlantıları düzgün kapatılmaz ve veri kaybı yaşanabilir. Bu yazıda, Python uygulamalarında **signal handling**, **graceful shutdown** ve **cleanup** stratejilerini derinlemesine inceleyeceğiz.

## İçindekiler
1. Graceful Shutdown Nedir?
2. Unix Signals ve Python
3. Signal Handler Implementation
4. FastAPI/Flask Graceful Shutdown
5. Database Connection Cleanup
6. In-Flight Request Handling
7. Kubernetes preStop Hooks
8. Systemd Integration
9. AsyncIO Cleanup
10. Testing Graceful Shutdown
11. Production Best Practices

## 1. Graceful Shutdown Nedir?

**Graceful shutdown**, bir uygulamanın kapatılma sinyali aldığında, mevcut işleri tamamlayıp kaynakları düzgün şekilde serbest bırakarak kapanmasıdır.

![Graceful Shutdown Flow](/assets/img/posts/graceful-shutdown-flow.png)
_Graceful Shutdown Süreci_

### Neden Önemli?

**❌ Ungraceful Shutdown Sorunları:**
```python
# Process kill edildiğinde:
- ❌ Devam eden HTTP request'ler yarıda kesilir
- ❌ Database transaction'ları rollback olmaz
- ❌ File descriptor'lar açık kalır
- ❌ Temp dosyalar silinmez
- ❌ Cache'ler flush edilmez
- ❌ Downstream service'lere haber verilmez
```

**✅ Graceful Shutdown Avantajları:**
```python
- ✅ Devam eden istekler tamamlanır
- ✅ Yeni istekler kabul edilmez (reject)
- ✅ Database bağlantıları cleanly kapatılır
- ✅ Resource'lar (file, socket) temizlenir
- ✅ State persistent storage'a yazılır
- ✅ Downstream service'lere drain notification
```

### Shutdown Phases

```
1. SIGTERM alındı
   ↓
2. Yeni istekleri reddet (health check fail)
   ↓
3. Mevcut istekleri tamamla (timeout ile)
   ↓
4. Background job'ları durdur
   ↓
5. Database/cache connection'ları kapat
   ↓
6. Temp dosyaları temizle
   ↓
7. Gracefully exit (exit code 0)
```

## 2. Unix Signals ve Python

Unix/Linux sistemlerde, process'ler arası iletişim **signal'ler** ile yapılır. Python, `signal` modülü ile bu signal'leri yakalayabilir.

![Linux Signals](/assets/img/posts/linux-signals-diagram.png)
_Linux Signal Türleri ve Davranışları_

### Temel Signal'ler

| Signal | Değer | Default Davranış | Yakalanabilir? | Kullanım |
|--------|-------|------------------|----------------|----------|
| **SIGTERM** | 15 | Process terminate | ✅ Evet | Graceful shutdown (önerilen) |
| **SIGINT** | 2 | Interrupt (Ctrl+C) | ✅ Evet | User interrupt |
| **SIGKILL** | 9 | Force kill | ❌ Hayır | Son çare (ungraceful) |
| **SIGHUP** | 1 | Hangup | ✅ Evet | Config reload |
| **SIGQUIT** | 3 | Quit with core dump | ✅ Evet | Debug amaçlı |
| **SIGUSR1/2** | 10/12 | User-defined | ✅ Evet | Custom logic |

### Signal Yakalama Basics

```python
import signal
import sys
import time

def signal_handler(signum, frame):
    """Signal handler fonksiyonu"""
    signal_name = signal.Signals(signum).name
    print(f"\n🛑 {signal_name} sinyali alındı!")
    print(f"Frame: {frame.f_code.co_filename}:{frame.f_lineno}")
    
    # Cleanup logic
    cleanup()
    
    # Exit
    sys.exit(0)

def cleanup():
    """Kapatma öncesi temizlik"""
    print("🧹 Cleanup yapılıyor...")
    # Close connections, flush caches, etc.
    time.sleep(1)
    print("✅ Cleanup tamamlandı")

# Signal handler'ları kaydet
signal.signal(signal.SIGTERM, signal_handler)
signal.signal(signal.SIGINT, signal_handler)

print("✨ Uygulama çalışıyor... (Ctrl+C veya SIGTERM ile durdur)")

# Main loop
try:
    while True:
        print("💻 Çalışıyor...")
        time.sleep(5)
except KeyboardInterrupt:
    print("\n⚠️ KeyboardInterrupt yakalandı")
    cleanup()
```

### Signal Davranışları

```python
# SIGTERM: Graceful shutdown (recommended)
# kill <pid>
# docker stop <container>  # 10 saniye sonra SIGKILL
# kubectl delete pod <pod>  # terminationGracePeriodSeconds

# SIGINT: User interrupt
# Ctrl+C
# kill -2 <pid>

# SIGKILL: Force kill (YAKALANAMIYOR!)
# kill -9 <pid>
# docker kill <container>

# SIGHUP: Reload configuration
# kill -1 <pid>
# systemctl reload service
```

## 3. Signal Handler Implementation

### Basic Signal Handler Class

```python
import signal
import sys
from typing import Callable, List

class GracefulShutdown:
    """Graceful shutdown manager"""
    
    def __init__(self):
        self.shutdown_handlers: List[Callable] = []
        self.is_shutting_down = False
        
        # Register signal handlers
        signal.signal(signal.SIGTERM, self._signal_handler)
        signal.signal(signal.SIGINT, self._signal_handler)
    
    def register(self, func: Callable):
        """Cleanup fonksiyonu kaydet"""
        self.shutdown_handlers.append(func)
        return func
    
    def _signal_handler(self, signum, frame):
        """Signal yakalandığında çalışır"""
        if self.is_shutting_down:
            print("⚠️ Zaten kapatılıyor, lütfen bekleyin...")
            return
        
        signal_name = signal.Signals(signum).name
        print(f"\n🛑 {signal_name} alındı, graceful shutdown başlıyor...")
        
        self.is_shutting_down = True
        self._shutdown()
    
    def _shutdown(self):
        """Tüm cleanup handler'ları çalıştır"""
        for handler in self.shutdown_handlers:
            try:
                print(f"🧹 {handler.__name__} çalıştırılıyor...")
                handler()
            except Exception as e:
                print(f"❌ {handler.__name__} hatası: {e}")
        
        print("✅ Graceful shutdown tamamlandı")
        sys.exit(0)

# Kullanım
shutdown_manager = GracefulShutdown()

@shutdown_manager.register
def close_database():
    """Database bağlantılarını kapat"""
    print("📦 Database bağlantıları kapatılıyor...")
    # db.close()

@shutdown_manager.register
def flush_cache():
    """Cache'i flush et"""
    print("💾 Cache flush ediliyor...")
    # cache.flush()

@shutdown_manager.register
def cleanup_temp_files():
    """Geçici dosyaları temizle"""
    print("🗑️ Temp dosyalar siliniyor...")
    # os.remove('/tmp/app_*')

# Main application
print("✨ Uygulama başladı")
while not shutdown_manager.is_shutting_down:
    time.sleep(1)
```

### Context Manager ile Cleanup

```python
from contextlib import contextmanager
import atexit

class ResourceManager:
    """Resource management with cleanup"""
    
    def __init__(self):
        self.resources = []
        
        # Register cleanup on exit
        atexit.register(self.cleanup_all)
        
        # Signal handlers
        signal.signal(signal.SIGTERM, self._signal_handler)
        signal.signal(signal.SIGINT, self._signal_handler)
    
    @contextmanager
    def acquire(self, resource_name: str):
        """Resource acquire with automatic cleanup"""
        resource = self._create_resource(resource_name)
        self.resources.append(resource)
        
        try:
            yield resource
        finally:
            self._release_resource(resource)
            self.resources.remove(resource)
    
    def _signal_handler(self, signum, frame):
        print(f"\n🛑 Signal {signum} alındı")
        self.cleanup_all()
        sys.exit(0)
    
    def cleanup_all(self):
        """Tüm resource'ları temizle"""
        print(f"🧹 {len(self.resources)} resource temizleniyor...")
        
        for resource in self.resources[:]:  # Copy to avoid modification during iteration
            try:
                self._release_resource(resource)
                self.resources.remove(resource)
            except Exception as e:
                print(f"❌ Resource cleanup error: {e}")
    
    def _create_resource(self, name):
        """Simulate resource creation"""
        print(f"✅ Resource created: {name}")
        return {'name': name, 'fd': open(f'/tmp/{name}', 'w')}
    
    def _release_resource(self, resource):
        """Simulate resource cleanup"""
        print(f"🗑️ Resource released: {resource['name']}")
        resource['fd'].close()

# Kullanım
manager = ResourceManager()

with manager.acquire('database'):
    with manager.acquire('cache'):
        print("💼 İş yapılıyor...")
        time.sleep(100)  # SIGTERM ile kesilir
```

## 4. FastAPI/Flask Graceful Shutdown

### FastAPI Graceful Shutdown

```python
from fastapi import FastAPI, Request
from contextlib import asynccontextmanager
import asyncio
import signal

# Global shutdown flag
is_shutting_down = False

@asynccontextmanager
async def lifespan(app: FastAPI):
    """Application lifespan manager"""
    
    # Startup
    print("🚀 FastAPI başlatılıyor...")
    
    # Database pool
    app.state.db = await create_db_pool()
    
    # Redis connection
    app.state.redis = await create_redis_pool()
    
    # Background tasks
    app.state.background_tasks = set()
    
    # Signal handlers
    setup_signal_handlers()
    
    yield
    
    # Shutdown
    print("🛑 FastAPI kapatılıyor...")
    
    global is_shutting_down
    is_shutting_down = True
    
    # Wait for ongoing requests (max 30s)
    await wait_for_requests(timeout=30)
    
    # Close connections
    await app.state.redis.close()
    await app.state.db.close()
    
    print("✅ FastAPI kapandı")

app = FastAPI(lifespan=lifespan)

# Health check endpoint (fail when shutting down)
@app.get("/health")
async def health_check():
    """Health check for load balancer"""
    if is_shutting_down:
        return {"status": "shutting_down"}, 503
    
    return {"status": "healthy"}

# Middleware to reject new requests during shutdown
@app.middleware("http")
async def shutdown_middleware(request: Request, call_next):
    """Reject requests during shutdown"""
    if is_shutting_down:
        return JSONResponse(
            status_code=503,
            content={"error": "Service shutting down"}
        )
    
    response = await call_next(request)
    return response

async def wait_for_requests(timeout: int = 30):
    """Wait for ongoing requests to complete"""
    print(f"⏳ Mevcut istekler bekleniyor (max {timeout}s)...")
    
    start = time.time()
    while time.time() - start < timeout:
        # Check if there are ongoing requests
        # Bu örnek basitleştirilmiş, gerçekte request counter tutulmalı
        await asyncio.sleep(0.5)
    
    print("✅ İstekler tamamlandı veya timeout oldu")

def setup_signal_handlers():
    """Setup graceful shutdown signal handlers"""
    def handler(signum, frame):
        print(f"\n🛑 Signal {signum} alındı")
        # Shutdown handled by lifespan
    
    signal.signal(signal.SIGTERM, handler)
    signal.signal(signal.SIGINT, handler)

# Example endpoint with cleanup
@app.post("/process")
async def process_data(data: dict):
    """Process data with cleanup"""
    if is_shutting_down:
        return {"error": "shutting_down"}, 503
    
    # Simulate work
    await asyncio.sleep(2)
    
    return {"status": "processed"}
```

### Flask Graceful Shutdown

```python
from flask import Flask, jsonify
import signal
import sys
from werkzeug.serving import make_server
import threading

app = Flask(__name__)
is_shutting_down = False
server = None

class ShutdownableServer:
    """Flask server with graceful shutdown"""
    
    def __init__(self, app, host='0.0.0.0', port=5000):
        self.server = make_server(host, port, app, threaded=True)
        self.ctx = app.app_context()
        self.ctx.push()
        
        # Signal handlers
        signal.signal(signal.SIGTERM, self._signal_handler)
        signal.signal(signal.SIGINT, self._signal_handler)
    
    def _signal_handler(self, signum, frame):
        global is_shutting_down
        print(f"\n🛑 Signal {signum} alındı, shutdown başlıyor...")
        is_shutting_down = True
        
        # Shutdown server
        threading.Thread(target=self.shutdown).start()
    
    def shutdown(self):
        """Graceful shutdown"""
        print("⏳ Mevcut istekler tamamlanıyor...")
        time.sleep(2)  # Wait for requests
        
        print("🧹 Cleanup yapılıyor...")
        # Close DB connections
        # Flush caches
        
        print("🛑 Server kapatılıyor...")
        self.server.shutdown()
        
        print("✅ Graceful shutdown tamamlandı")
        sys.exit(0)
    
    def run(self):
        """Run server"""
        print("🚀 Flask başlatıldı")
        self.server.serve_forever()

@app.before_request
def check_shutdown():
    """Reject requests during shutdown"""
    if is_shutting_down:
        return jsonify({"error": "shutting_down"}), 503

@app.route('/health')
def health():
    """Health check"""
    if is_shutting_down:
        return jsonify({"status": "shutting_down"}), 503
    return jsonify({"status": "healthy"})

if __name__ == '__main__':
    server = ShutdownableServer(app)
    server.run()
```

## 5. Database Connection Cleanup

### SQLAlchemy Connection Pool

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from contextlib import contextmanager
import signal
import sys

class DatabaseManager:
    """Database manager with graceful shutdown"""
    
    def __init__(self, db_url: str):
        self.engine = create_engine(
            db_url,
            pool_size=20,
            max_overflow=10,
            pool_pre_ping=True,  # Connection health check
        )
        self.SessionLocal = sessionmaker(bind=self.engine)
        self.is_shutting_down = False
        
        # Signal handlers
        signal.signal(signal.SIGTERM, self._signal_handler)
        signal.signal(signal.SIGINT, self._signal_handler)
    
    def _signal_handler(self, signum, frame):
        print("\n🛑 Database shutdown başlıyor...")
        self.shutdown()
        sys.exit(0)
    
    def shutdown(self):
        """Graceful database shutdown"""
        if self.is_shutting_down:
            return
        
        self.is_shutting_down = True
        
        print("⏳ Aktif database session'lar bekleniyor...")
        
        # Dispose connection pool
        self.engine.dispose()
        
        print("✅ Database connections kapatıldı")
    
    @contextmanager
    def session_scope(self):
        """Provide transactional scope with cleanup"""
        if self.is_shutting_down:
            raise Exception("Database shutting down")
        
        session = self.SessionLocal()
        try:
            yield session
            session.commit()
        except Exception:
            session.rollback()
            raise
        finally:
            session.close()

# Kullanım
db = DatabaseManager('postgresql://user:pass@localhost/db')

try:
    with db.session_scope() as session:
        # Database operations
        result = session.execute("SELECT * FROM users")
except Exception as e:
    print(f"Error: {e}")
```

### AsyncPG Connection Pool

```python
import asyncpg
import asyncio
import signal

class AsyncDatabaseManager:
    """Async database manager with graceful shutdown"""
    
    def __init__(self, dsn: str):
        self.dsn = dsn
        self.pool = None
        self.is_shutting_down = False
    
    async def connect(self):
        """Create connection pool"""
        self.pool = await asyncpg.create_pool(
            self.dsn,
            min_size=10,
            max_size=50,
            command_timeout=60,
        )
        print("✅ Database pool oluşturuldu")
    
    async def shutdown(self):
        """Graceful shutdown"""
        if self.is_shutting_down:
            return
        
        self.is_shutting_down = True
        print("🛑 Database pool kapatılıyor...")
        
        if self.pool:
            # Wait for ongoing queries (with timeout)
            await asyncio.wait_for(
                self.pool.close(),
                timeout=30.0
            )
        
        print("✅ Database pool kapatıldı")
    
    async def execute(self, query: str, *args):
        """Execute query with shutdown check"""
        if self.is_shutting_down:
            raise Exception("Database shutting down")
        
        async with self.pool.acquire() as conn:
            return await conn.fetch(query, *args)

# Kullanım
async def main():
    db = AsyncDatabaseManager('postgresql://user:pass@localhost/db')
    await db.connect()
    
    # Setup signal handler
    loop = asyncio.get_event_loop()
    
    def signal_handler():
        print("\n🛑 SIGTERM alındı")
        asyncio.create_task(db.shutdown())
        loop.stop()
    
    loop.add_signal_handler(signal.SIGTERM, signal_handler)
    loop.add_signal_handler(signal.SIGINT, signal_handler)
    
    try:
        # Application logic
        while not db.is_shutting_down:
            await asyncio.sleep(1)
    finally:
        await db.shutdown()

asyncio.run(main())
```

## 6. In-Flight Request Handling

### Request Counter Pattern

```python
import asyncio
from fastapi import FastAPI
from contextvars import ContextVar

# Context variable to track requests
request_count: ContextVar[int] = ContextVar('request_count', default=0)

class RequestTracker:
    """Track in-flight requests"""
    
    def __init__(self):
        self.active_requests = 0
        self.lock = asyncio.Lock()
    
    async def increment(self):
        """Increment active request counter"""
        async with self.lock:
            self.active_requests += 1
    
    async def decrement(self):
        """Decrement active request counter"""
        async with self.lock:
            self.active_requests -= 1
    
    async def wait_for_completion(self, timeout: float = 30.0):
        """Wait for all requests to complete"""
        start = asyncio.get_event_loop().time()
        
        while self.active_requests > 0:
            if asyncio.get_event_loop().time() - start > timeout:
                print(f"⚠️ Timeout: {self.active_requests} requests still active")
                break
            
            await asyncio.sleep(0.1)
        
        print(f"✅ All requests completed (or timeout)")

# Global tracker
tracker = RequestTracker()

app = FastAPI()

@app.middleware("http")
async def track_requests(request, call_next):
    """Track active requests"""
    await tracker.increment()
    
    try:
        response = await call_next(request)
        return response
    finally:
        await tracker.decrement()

@app.on_event("shutdown")
async def shutdown_event():
    """Wait for requests on shutdown"""
    print(f"🛑 Shutdown: {tracker.active_requests} active requests")
    await tracker.wait_for_completion(timeout=30.0)
```

## 7. Kubernetes preStop Hooks

![Kubernetes preStop Lifecycle](/assets/img/posts/kubernetes-prestop-lifecycle.png)
_Kubernetes Pod Termination Lifecycle_

### Pod Termination Sequence

```
1. kubectl delete pod
   ↓
2. Pod status: Terminating
   ↓
3. preStop hook çalıştırılır (paralel)
4. SIGTERM gönderilir (paralel)
   ↓
5. terminationGracePeriodSeconds bekle (default 30s)
   ↓
6. Hala canlıysa SIGKILL gönder
```

### Kubernetes Deployment with preStop

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fastapi-app
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: app
        image: myapp:latest
        ports:
        - containerPort: 8000
        
        # Liveness probe (pod sağlıklı mı?)
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 10
          periodSeconds: 5
        
        # Readiness probe (trafik alabilir mi?)
        readinessProbe:
          httpGet:
            path: /ready
            port: 8000
          initialDelaySeconds: 5
          periodSeconds: 3
        
        # preStop hook
        lifecycle:
          preStop:
            exec:
              command:
              - /bin/sh
              - -c
              - |
                # Health check'i fail et (trafik gelmesin)
                touch /tmp/shutting-down
                
                # Load balancer'ın bunu fark etmesi için bekle
                sleep 10
                
                # Graceful shutdown trigger
                kill -TERM 1
      
      # Graceful termination period
      terminationGracePeriodSeconds: 60
```

### Health Check Implementation

```python
from fastapi import FastAPI
from pathlib import Path

app = FastAPI()

SHUTDOWN_FILE = Path("/tmp/shutting-down")

@app.get("/health")
async def health():
    """Liveness probe - pod canlı mı?"""
    return {"status": "alive"}

@app.get("/ready")
async def readiness():
    """Readiness probe - trafik alabilir mi?"""
    if SHUTDOWN_FILE.exists():
        return {"status": "not_ready"}, 503
    
    # Check dependencies
    if not db.is_connected():
        return {"status": "not_ready", "reason": "db_down"}, 503
    
    return {"status": "ready"}

@app.on_event("shutdown")
async def shutdown():
    """Graceful shutdown"""
    # Create shutdown file
    SHUTDOWN_FILE.touch()
    
    print("⏳ Waiting for ongoing requests...")
    await asyncio.sleep(5)
    
    print("🧹 Cleanup...")
    # Close connections
```

### Pod Graceful Shutdown Diagram

![Pod Graceful Shutdown](/assets/img/posts/pod-graceful-shutdown-diagram.png)
_Kubernetes Pod Zarif Kapanış Süreci_

## 8. Systemd Integration

### Systemd Service File

```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My FastAPI Application
After=network.target redis.service postgresql.service
Wants=redis.service postgresql.service

[Service]
Type=notify
User=appuser
Group=appuser
WorkingDirectory=/opt/myapp

# Environment
Environment="PYTHONUNBUFFERED=1"
Environment="CONFIG_FILE=/etc/myapp/config.yaml"

# Start command
ExecStart=/opt/myapp/venv/bin/python -m uvicorn app.main:app --host 0.0.0.0 --port 8000

# Graceful reload (SIGHUP)
ExecReload=/bin/kill -HUP $MAINPID

# Graceful stop (SIGTERM)
KillMode=mixed
KillSignal=SIGTERM
TimeoutStopSec=60
SendSIGKILL=yes

# Restart policy
Restart=always
RestartSec=10

# Logging
StandardOutput=journal
StandardError=journal
SyslogIdentifier=myapp

[Install]
WantedBy=multi-user.target
```

### Systemd Notify Support

```python
import sys
import socket

def notify_systemd(message: str):
    """Send notification to systemd"""
    notify_socket = os.getenv('NOTIFY_SOCKET')
    
    if not notify_socket:
        return  # Not running under systemd
    
    sock = socket.socket(socket.AF_UNIX, socket.SOCK_DGRAM)
    
    try:
        sock.sendto(message.encode(), notify_socket)
    finally:
        sock.close()

# Application startup
notify_systemd("READY=1")
notify_systemd("STATUS=Application started")

# During shutdown
notify_systemd("STOPPING=1")
notify_systemd("STATUS=Shutting down gracefully")

# On reload
signal.signal(signal.SIGHUP, reload_config)

def reload_config(signum, frame):
    """Reload configuration on SIGHUP"""
    print("🔄 Configuration reloading...")
    load_config()
    notify_systemd("RELOADING=1")
    notify_systemd("STATUS=Configuration reloaded")
```

### Systemd Commands

```bash
# Service kontrol
sudo systemctl start myapp
sudo systemctl stop myapp          # SIGTERM gönderir
sudo systemctl reload myapp        # SIGHUP gönderir
sudo systemctl restart myapp

# Status kontrolü
sudo systemctl status myapp
sudo journalctl -u myapp -f        # Logs

# Enable on boot
sudo systemctl enable myapp
```

## 9. AsyncIO Cleanup

### AsyncIO Signal Handling

```python
import asyncio
import signal
from typing import Set

class AsyncApplication:
    """AsyncIO application with graceful shutdown"""
    
    def __init__(self):
        self.is_shutting_down = False
        self.background_tasks: Set[asyncio.Task] = set()
        self.cleanup_callbacks = []
    
    def add_cleanup(self, coro):
        """Register cleanup coroutine"""
        self.cleanup_callbacks.append(coro)
    
    async def start_background_task(self, coro):
        """Start background task with tracking"""
        task = asyncio.create_task(coro)
        self.background_tasks.add(task)
        task.add_done_callback(self.background_tasks.discard)
        return task
    
    async def shutdown(self):
        """Graceful shutdown"""
        if self.is_shutting_down:
            return
        
        self.is_shutting_down = True
        print("\n🛑 Shutdown başlıyor...")
        
        # Cancel background tasks
        print(f"🚫 {len(self.background_tasks)} background task iptal ediliyor...")
        for task in self.background_tasks:
            task.cancel()
        
        # Wait for tasks to finish
        await asyncio.gather(*self.background_tasks, return_exceptions=True)
        
        # Run cleanup callbacks
        print("🧹 Cleanup yapılıyor...")
        for callback in self.cleanup_callbacks:
            try:
                await callback()
            except Exception as e:
                print(f"❌ Cleanup error: {e}")
        
        print("✅ Shutdown tamamlandı")
    
    def setup_signal_handlers(self):
        """Setup signal handlers for graceful shutdown"""
        loop = asyncio.get_event_loop()
        
        for sig in (signal.SIGTERM, signal.SIGINT):
            loop.add_signal_handler(
                sig,
                lambda s=sig: asyncio.create_task(self._handle_signal(s))
            )
    
    async def _handle_signal(self, sig):
        """Handle shutdown signal"""
        print(f"\n🛑 Signal {sig} alındı")
        await self.shutdown()
        asyncio.get_event_loop().stop()

# Kullanım
async def worker_task(app: AsyncApplication):
    """Background worker"""
    try:
        while not app.is_shutting_down:
            print("💼 Worker çalışıyor...")
            await asyncio.sleep(2)
    except asyncio.CancelledError:
        print("⚠️ Worker cancelled")
        raise

async def cleanup_database():
    """Cleanup callback"""
    print("📦 Database cleanup...")
    await asyncio.sleep(0.5)

async def cleanup_cache():
    """Cleanup callback"""
    print("💾 Cache cleanup...")
    await asyncio.sleep(0.5)

async def main():
    app = AsyncApplication()
    app.setup_signal_handlers()
    
    # Register cleanup
    app.add_cleanup(cleanup_database)
    app.add_cleanup(cleanup_cache)
    
    # Start background tasks
    await app.start_background_task(worker_task(app))
    await app.start_background_task(worker_task(app))
    
    # Run until shutdown
    try:
        while not app.is_shutting_down:
            await asyncio.sleep(1)
    except KeyboardInterrupt:
        await app.shutdown()

if __name__ == '__main__':
    asyncio.run(main())
```

### TaskGroup with Cleanup (Python 3.11+)

```python
import asyncio

async def worker(name: str):
    """Worker task"""
    try:
        while True:
            print(f"Worker {name} running")
            await asyncio.sleep(1)
    except asyncio.CancelledError:
        print(f"Worker {name} cancelled, cleaning up...")
        # Cleanup logic
        await asyncio.sleep(0.5)
        print(f"Worker {name} cleanup done")
        raise

async def main():
    try:
        async with asyncio.TaskGroup() as tg:
            # All tasks will be cancelled together on shutdown
            tg.create_task(worker("A"))
            tg.create_task(worker("B"))
            tg.create_task(worker("C"))
    
    except* asyncio.CancelledError:
        print("All workers cancelled")

if __name__ == '__main__':
    try:
        asyncio.run(main())
    except KeyboardInterrupt:
        print("\n✅ Graceful shutdown")
```

## 10. Testing Graceful Shutdown

### Unit Test

```python
import asyncio
import signal
import pytest
from unittest.mock import MagicMock, patch

@pytest.mark.asyncio
async def test_graceful_shutdown():
    """Test graceful shutdown behavior"""
    app = AsyncApplication()
    
    # Mock cleanup
    cleanup_called = False
    
    async def mock_cleanup():
        nonlocal cleanup_called
        cleanup_called = True
    
    app.add_cleanup(mock_cleanup)
    
    # Trigger shutdown
    await app.shutdown()
    
    # Assertions
    assert app.is_shutting_down is True
    assert cleanup_called is True

def test_signal_handler():
    """Test signal handler registration"""
    shutdown_called = False
    
    def handler(signum, frame):
        nonlocal shutdown_called
        shutdown_called = True
    
    signal.signal(signal.SIGTERM, handler)
    
    # Send signal to self
    os.kill(os.getpid(), signal.SIGTERM)
    
    time.sleep(0.1)
    assert shutdown_called is True
```

### Integration Test

```python
import subprocess
import time
import requests

def test_graceful_shutdown_integration():
    """Test graceful shutdown in real process"""
    
    # Start application
    proc = subprocess.Popen(
        ['python', 'app.py'],
        stdout=subprocess.PIPE,
        stderr=subprocess.PIPE,
    )
    
    # Wait for startup
    time.sleep(2)
    
    # Verify app is running
    response = requests.get('http://localhost:8000/health')
    assert response.status_code == 200
    
    # Send SIGTERM
    proc.send_signal(signal.SIGTERM)
    
    # Wait for graceful shutdown
    try:
        proc.wait(timeout=10)
        assert proc.returncode == 0  # Clean exit
    except subprocess.TimeoutExpired:
        proc.kill()  # Force kill if hung
        pytest.fail("Graceful shutdown timeout")
```

## 11. Production Best Practices

### Checklist

```python
✅ Signal Handling
- [ ] SIGTERM handler implement edildi
- [ ] SIGINT handler implement edildi
- [ ] Timeout mechanism var (max 30-60s)
- [ ] Double SIGTERM için protection var

✅ Resource Cleanup
- [ ] Database connection'lar kapatılıyor
- [ ] Cache flush ediliyor
- [ ] File descriptor'lar kapatılıyor
- [ ] Temp dosyalar siliniyor
- [ ] Background task'lar cancel ediliyor

✅ Request Handling
- [ ] Yeni request'ler reject ediliyor
- [ ] Mevcut request'ler tamamlanıyor
- [ ] Health check fail oluyor (load balancer için)
- [ ] Request counter/tracking var

✅ Container/K8s
- [ ] preStop hook tanımlı
- [ ] terminationGracePeriodSeconds yeterli (30-60s)
- [ ] Readiness probe var
- [ ] Liveness probe var

✅ Monitoring
- [ ] Shutdown event'leri loglaniyor
- [ ] Metrics export ediliyor
- [ ] Alert sistemi var
- [ ] Graceful shutdown süresi ölçülüyor
```

### Production Template

```python
import asyncio
import signal
import logging
from typing import List, Callable
from pathlib import Path

logger = logging.getLogger(__name__)

class ProductionApp:
    """Production-ready application with graceful shutdown"""
    
    def __init__(self, shutdown_timeout: int = 30):
        self.shutdown_timeout = shutdown_timeout
        self.is_shutting_down = False
        self.shutdown_event = asyncio.Event()
        self.cleanup_handlers: List[Callable] = []
        
        # Shutdown marker for K8s
        self.shutdown_marker = Path("/tmp/shutting-down")
    
    def register_cleanup(self, handler: Callable):
        """Register cleanup handler"""
        self.cleanup_handlers.append(handler)
    
    def setup_signals(self):
        """Setup signal handlers"""
        loop = asyncio.get_event_loop()
        
        for sig in (signal.SIGTERM, signal.SIGINT):
            loop.add_signal_handler(
                sig,
                lambda s=sig: asyncio.create_task(self.shutdown(s))
            )
        
        logger.info("Signal handlers registered")
    
    async def shutdown(self, sig=None):
        """Graceful shutdown"""
        if self.is_shutting_down:
            logger.warning("Already shutting down, ignoring signal")
            return
        
        self.is_shutting_down = True
        logger.info(f"Shutdown initiated (signal={sig})")
        
        # Create shutdown marker
        self.shutdown_marker.touch()
        
        # Wait for inflight requests
        logger.info("Waiting for inflight requests...")
        try:
            await asyncio.wait_for(
                self.wait_for_requests(),
                timeout=self.shutdown_timeout
            )
        except asyncio.TimeoutError:
            logger.warning(f"Shutdown timeout after {self.shutdown_timeout}s")
        
        # Run cleanup handlers
        logger.info(f"Running {len(self.cleanup_handlers)} cleanup handlers")
        for handler in self.cleanup_handlers:
            try:
                if asyncio.iscoroutinefunction(handler):
                    await handler()
                else:
                    handler()
            except Exception as e:
                logger.error(f"Cleanup handler error: {e}")
        
        logger.info("Graceful shutdown complete")
        self.shutdown_event.set()
    
    async def wait_for_requests(self):
        """Wait for requests to complete"""
        # Implement your request tracking logic
        await asyncio.sleep(1)
    
    async def run(self):
        """Run application"""
        self.setup_signals()
        
        logger.info("Application started")
        
        # Your application logic here
        await self.shutdown_event.wait()
        
        logger.info("Application stopped")

# Usage
async def main():
    app = ProductionApp(shutdown_timeout=30)
    
    # Register cleanup
    @app.register_cleanup
    async def cleanup_db():
        logger.info("Closing database connections")
        # await db.close()
    
    @app.register_cleanup
    async def cleanup_cache():
        logger.info("Flushing cache")
        # await cache.flush()
    
    await app.run()

if __name__ == '__main__':
    logging.basicConfig(level=logging.INFO)
    asyncio.run(main())
```

## Sonuç

**Graceful shutdown**, production uygulamalarının olmazsa olmaz bir özelliğidir. Doğru implement edildiğinde:

✅ **Zero downtime deployments** sağlar  
✅ **Data loss** önler  
✅ **Resource leaks** engeller  
✅ **User experience** iyileştirir  
✅ **Debugging** kolaylaşır  

###핵심 Prensipler

1. **SIGTERM'i yakala**, SIGKILL'e güvenme
2. **Timeout uygula**, sonsuz beklemeler
3. **Health check'leri fail et**, yeni trafik gelmesin
4. **Mevcut işleri bitir**, yenilerini reddet
5. **Resource'ları temizle**, connection, file, cache
6. **Test et**, production'da sürprizle karşılaşma

Modern container ve orchestration platformlarında (Docker, Kubernetes), graceful shutdown doğru çalıştığında deploy süreçleri sorunsuz ilerler ve kullanıcı kesintisi yaşanmaz.

## Kaynaklar

- [Python Signal Documentation](https://docs.python.org/3/library/signal.html)
- [Kubernetes Pod Lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)
- [FastAPI Lifespan Events](https://fastapi.tiangolo.com/advanced/events/)
- [Systemd Service Files](https://www.freedesktop.org/software/systemd/man/systemd.service.html)
- [Graceful Shutdown Patterns](https://aws.amazon.com/blogs/containers/graceful-shutdowns-with-ecs/)
