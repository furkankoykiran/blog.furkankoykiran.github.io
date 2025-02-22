---
title: "Python Loglama En İyi Uygulamaları - Production Logging Rehberi"
date: 2025-02-22 09:00:00 +0300
categories: [Python, Observability]
tags: [python, logging, elk, structured-logging, json, log-rotation, monitoring, debugging, production, best-practices]
image:
  path: /assets/img/posts/python-logging-flow-diagram.png
  alt: "Python Logging Akış Diagramı"
---

Production ortamında çalışan bir uygulamanın sağlığını, performansını ve hatalarını takip etmenin en kritik yolu **loglama**dır. İyi yapılandırılmış bir logging sistemi, debug süreçlerini hızlandırır, incident response'u iyileştirir ve sistem davranışları hakkında değerli içgörüler sağlar. Bu yazıda, Python'da **production-grade logging** sistemleri kurmayı, best practice'leri ve modern tooling'i detaylıca inceleyeceğiz.

## İçindekiler
1. Python Logging Temelleri
2. Log Levels ve Kullanım Senaryoları
3. Handlers ve Formatters
4. Structured Logging (JSON)
5. Log Rotation ve Arşivleme
6. Centralized Logging (ELK Stack)
7. Correlation IDs ve Distributed Tracing
8. Performance Considerations
9. Security ve Sensitive Data
10. Third-Party Logger Integration
11. Production Best Practices

## 1. Python Logging Temelleri

Python'un built-in `logging` modülü, esnek ve güçlü bir logging framework'ü sunar.

![Python Logging Flow](/assets/img/posts/python-logging-flow-diagram.png)
_Python Logging Akış Mimarisi_

### Logging Bileşenleri

**1. Logger**: Log record'ları oluşturur
**2. Handler**: Log record'ları nereye yazılacağını belirler (file, console, network)
**3. Formatter**: Log mesajlarının formatını belirler
**4. Filter**: Log record'larını filtreleyebilir

### Basic Setup

```python
import logging

# Basic config (development için basit)
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)

logger = logging.getLogger(__name__)

logger.debug("Debug mesajı")
logger.info("Info mesajı")
logger.warning("Warning mesajı")
logger.error("Error mesajı")
logger.critical("Critical mesajı")

# Exception logging
try:
    1 / 0
except Exception:
    logger.exception("Division by zero hatası!")
```

### Logger Hierarchy

```python
# Logger hierarchy (dot notation)
root_logger = logging.getLogger()                  # Root logger
app_logger = logging.getLogger('myapp')            # myapp logger
db_logger = logging.getLogger('myapp.database')    # myapp.database logger
api_logger = logging.getLogger('myapp.api')        # myapp.api logger

# Child logger'lar parent'ın konfigürasyonunu inherit eder
app_logger.setLevel(logging.INFO)
db_logger.setLevel(logging.DEBUG)  # DB için daha detaylı

# Logger'ları kullan
db_logger.debug("SELECT * FROM users")  # Sadece db_logger gösterir
api_logger.info("Request received")     # Hem api hem app logger'a gider
```

## 2. Log Levels ve Kullanım Senaryoları

![Log Levels Pyramid](/assets/img/posts/log-levels-pyramid.png)
_Log Seviyeleri Hiyerarşisi (En kritikten en detaylıya)_

### Log Level Tablosu

| Level | Numeric Value | Kullanım Senaryosu | Production'da? |
|-------|---------------|-------------------|----------------|
| **DEBUG** | 10 | Detaylı debug bilgileri, değişken değerleri | ❌ Genelde kapalı |
| **INFO** | 20 | Genel bilgi mesajları, iş akışı | ✅ Evet |
| **WARNING** | 30 | Potansiyel sorunlar, deprecated usage | ✅ Evet |
| **ERROR** | 40 | Hata durumları, exception'lar | ✅ Evet |
| **CRITICAL** | 50 | Kritik hatalar, sistem çökmesi | ✅ Evet |

### Log Level Best Practices

```python
import logging

logger = logging.getLogger(__name__)

# ✅ DEBUG: Development debugging
logger.debug(f"Processing user_id={user_id}, params={params}")
logger.debug(f"SQL Query: {query}")
logger.debug(f"API Response: {response.json()}")

# ✅ INFO: İş akışı ve önemli olaylar
logger.info(f"User {user_id} logged in from {ip_address}")
logger.info(f"Order {order_id} created successfully")
logger.info(f"Background job started: {job_name}")

# ✅ WARNING: Potansiyel sorunlar
logger.warning(f"API rate limit approaching: {current}/{limit}")
logger.warning(f"Deprecated function called: {func_name}")
logger.warning(f"Retry attempt {attempt}/{max_retries}")

# ✅ ERROR: İşlem başarısız
logger.error(f"Payment failed for order {order_id}: {error}")
logger.error(f"Database connection lost: {db_host}")
logger.error(f"Email delivery failed: {recipient}")

# ✅ CRITICAL: Sistem kritik durum
logger.critical(f"Database unreachable, service degraded")
logger.critical(f"Memory usage: {memory_percent}% (threshold: 90%)")
logger.critical(f"Disk space critical: {disk_free}MB remaining")

# ❌ YANLIŞ Kullanımlar
logger.info("Debug variable: x=5")           # DEBUG kullan
logger.error("User clicked button")          # INFO veya DEBUG kullan
logger.critical("Invalid password attempt")  # WARNING veya ERROR kullan
```

### Conditional Logging (Performance)

```python
# ❌ KÖTÜ: String formatlamı her zaman yapılır
logger.debug(f"User data: {expensive_operation()}")  # Daima çalışır!

# ✅ İYİ: isEnabledFor ile kontrol et
if logger.isEnabledFor(logging.DEBUG):
    logger.debug(f"User data: {expensive_operation()}")

# ✅ DAHA İYİ: Lazy evaluation
logger.debug("User data: %s", lambda: expensive_operation())

# ✅ EN İYİ: Decorator pattern
from functools import wraps

def log_execution(level=logging.DEBUG):
    """Log function execution with timing"""
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            if not logger.isEnabledFor(level):
                return func(*args, **kwargs)
            
            start = time.time()
            try:
                result = func(*args, **kwargs)
                duration = time.time() - start
                logger.log(level, f"{func.__name__} completed in {duration:.2f}s")
                return result
            except Exception as e:
                duration = time.time() - start
                logger.error(f"{func.__name__} failed after {duration:.2f}s: {e}")
                raise
        return wrapper
    return decorator

@log_execution(logging.INFO)
def process_data(data):
    # Processing logic
    return result
```

## 3. Handlers ve Formatters

### Common Handlers

```python
import logging
from logging.handlers import (
    RotatingFileHandler,
    TimedRotatingFileHandler,
    SysLogHandler,
    SMTPHandler,
    HTTPHandler,
)

logger = logging.getLogger('myapp')
logger.setLevel(logging.DEBUG)

# 1. Console Handler (stdout)
console_handler = logging.StreamHandler()
console_handler.setLevel(logging.INFO)
console_formatter = logging.Formatter(
    '%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)
console_handler.setFormatter(console_formatter)

# 2. File Handler (basic)
file_handler = logging.FileHandler('/var/log/myapp/app.log')
file_handler.setLevel(logging.DEBUG)

# 3. Rotating File Handler (size-based)
rotating_handler = RotatingFileHandler(
    '/var/log/myapp/app.log',
    maxBytes=10*1024*1024,  # 10MB
    backupCount=5,          # Keep 5 old files
)

# 4. Timed Rotating Handler (time-based)
timed_handler = TimedRotatingFileHandler(
    '/var/log/myapp/app.log',
    when='midnight',        # Rotate at midnight
    interval=1,             # Every day
    backupCount=30,         # Keep 30 days
)

# 5. Syslog Handler (Unix syslog)
syslog_handler = SysLogHandler(address='/dev/log')

# 6. SMTP Handler (email alerts)
smtp_handler = SMTPHandler(
    mailhost=('smtp.gmail.com', 587),
    fromaddr='alerts@myapp.com',
    toaddrs=['admin@myapp.com'],
    subject='Application Error',
    credentials=('user', 'password'),
    secure=(),
)
smtp_handler.setLevel(logging.ERROR)  # Sadece error'larda mail at

# 7. HTTP Handler (remote logging)
http_handler = HTTPHandler(
    'logging.example.com:9000',
    '/logs',
    method='POST',
)

# Handler'ları ekle
logger.addHandler(console_handler)
logger.addHandler(rotating_handler)
logger.addHandler(smtp_handler)
```

### Custom Handler Example

```python
import logging
import requests

class SlackHandler(logging.Handler):
    """Slack'e log gönderen custom handler"""
    
    def __init__(self, webhook_url: str, level=logging.ERROR):
        super().__init__(level)
        self.webhook_url = webhook_url
    
    def emit(self, record):
        """Log record'u Slack'e gönder"""
        try:
            log_entry = self.format(record)
            
            payload = {
                "text": f"🚨 *{record.levelname}*",
                "attachments": [{
                    "color": self._get_color(record.levelname),
                    "fields": [
                        {"title": "Message", "value": record.getMessage()},
                        {"title": "Logger", "value": record.name},
                        {"title": "File", "value": f"{record.filename}:{record.lineno}"},
                    ]
                }]
            }
            
            requests.post(self.webhook_url, json=payload, timeout=5)
        
        except Exception:
            self.handleError(record)
    
    def _get_color(self, level):
        """Level'a göre renk döndür"""
        colors = {
            'DEBUG': '#808080',
            'INFO': '#36a64f',
            'WARNING': '#ff9900',
            'ERROR': '#ff0000',
            'CRITICAL': '#990000',
        }
        return colors.get(level, '#808080')

# Kullanım
slack_handler = SlackHandler(webhook_url='https://hooks.slack.com/...')
logger.addHandler(slack_handler)

logger.error("Payment processing failed!")  # Slack'e gönderilir
```

### Advanced Formatters

```python
import logging
import json
from datetime import datetime

class ColoredFormatter(logging.Formatter):
    """Renkli console output"""
    
    COLORS = {
        'DEBUG': '\033[36m',    # Cyan
        'INFO': '\033[32m',     # Green
        'WARNING': '\033[33m',  # Yellow
        'ERROR': '\033[31m',    # Red
        'CRITICAL': '\033[35m', # Magenta
        'RESET': '\033[0m',     # Reset
    }
    
    def format(self, record):
        color = self.COLORS.get(record.levelname, self.COLORS['RESET'])
        record.levelname = f"{color}{record.levelname}{self.COLORS['RESET']}"
        return super().format(record)

class JSONFormatter(logging.Formatter):
    """JSON formatında log output"""
    
    def format(self, record):
        log_data = {
            'timestamp': datetime.utcnow().isoformat(),
            'level': record.levelname,
            'logger': record.name,
            'message': record.getMessage(),
            'module': record.module,
            'function': record.funcName,
            'line': record.lineno,
        }
        
        # Exception bilgisi varsa ekle
        if record.exc_info:
            log_data['exception'] = self.formatException(record.exc_info)
        
        # Extra fields
        if hasattr(record, 'user_id'):
            log_data['user_id'] = record.user_id
        
        if hasattr(record, 'request_id'):
            log_data['request_id'] = record.request_id
        
        return json.dumps(log_data)

# Kullanım
colored_handler = logging.StreamHandler()
colored_handler.setFormatter(ColoredFormatter(
    '%(asctime)s - %(name)s - %(levelname)s - %(message)s'
))

json_handler = logging.FileHandler('/var/log/myapp/app.json')
json_handler.setFormatter(JSONFormatter())

logger.addHandler(colored_handler)
logger.addHandler(json_handler)
```

## 4. Structured Logging (JSON)

Structured logging, log mesajlarını **machine-readable** formatta (JSON) kaydetmeyi sağlar. Bu sayede log'lar parse etmek, aramak ve analiz etmek çok kolay hale gelir.

![Structured Logging JSON](/assets/img/posts/structured-logging-json.png)
_Structured vs Unstructured Logging Karşılaştırması_

### python-json-logger Kullanımı

```bash
pip install python-json-logger
```

```python
from pythonjsonlogger import jsonlogger
import logging

# JSON formatter
logHandler = logging.StreamHandler()
formatter = jsonlogger.JsonFormatter(
    '%(timestamp)s %(level)s %(name)s %(message)s'
)
logHandler.setFormatter(formatter)

logger = logging.getLogger()
logger.addHandler(logHandler)
logger.setLevel(logging.INFO)

# Basit log
logger.info("User logged in")

# Output:
# {"timestamp": "2025-02-22T10:30:15.123456", "level": "INFO", 
#  "name": "root", "message": "User logged in"}

# Extra fields ile
logger.info("Payment processed", extra={
    'user_id': 12345,
    'amount': 99.99,
    'currency': 'USD',
    'transaction_id': 'txn_abc123',
})

# Output:
# {"timestamp": "...", "level": "INFO", "message": "Payment processed",
#  "user_id": 12345, "amount": 99.99, "currency": "USD", 
#  "transaction_id": "txn_abc123"}
```

### structlog ile Advanced Structured Logging

```bash
pip install structlog
```

```python
import structlog

# Structlog configuration
structlog.configure(
    processors=[
        structlog.stdlib.filter_by_level,
        structlog.stdlib.add_logger_name,
        structlog.stdlib.add_log_level,
        structlog.stdlib.PositionalArgumentsFormatter(),
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.StackInfoRenderer(),
        structlog.processors.format_exc_info,
        structlog.processors.UnicodeDecoder(),
        structlog.processors.JSONRenderer()
    ],
    context_class=dict,
    logger_factory=structlog.stdlib.LoggerFactory(),
    cache_logger_on_first_use=True,
)

logger = structlog.get_logger()

# Context binding
logger = logger.bind(user_id=12345, session_id='abc')

# Her log bu context'i içerecek
logger.info("user_action", action="login")
# {"event": "user_action", "action": "login", "user_id": 12345, 
#  "session_id": "abc", "timestamp": "..."}

logger.info("user_action", action="purchase", amount=99.99)
# {"event": "user_action", "action": "purchase", "amount": 99.99,
#  "user_id": 12345, "session_id": "abc", "timestamp": "..."}

# Context unbind
logger = logger.unbind("session_id")
```

### FastAPI ile Structured Logging

```python
from fastapi import FastAPI, Request
import structlog
import uuid

app = FastAPI()

# Structlog setup
structlog.configure(
    processors=[
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.JSONRenderer()
    ],
    logger_factory=structlog.PrintLoggerFactory(),
)

logger = structlog.get_logger()

@app.middleware("http")
async def logging_middleware(request: Request, call_next):
    """Her request için context oluştur"""
    request_id = str(uuid.uuid4())
    
    # Context bind
    log = logger.bind(
        request_id=request_id,
        method=request.method,
        path=request.url.path,
        client_ip=request.client.host,
    )
    
    # Request'i logla
    log.info("request_started")
    
    # Request'i işle
    import time
    start = time.time()
    
    response = await call_next(request)
    
    duration = time.time() - start
    
    # Response'u logla
    log.info(
        "request_completed",
        status_code=response.status_code,
        duration_ms=round(duration * 1000, 2),
    )
    
    # Request ID'yi header'a ekle
    response.headers["X-Request-ID"] = request_id
    
    return response

@app.get("/users/{user_id}")
async def get_user(user_id: int):
    """User endpoint with logging"""
    log = logger.bind(user_id=user_id)
    
    log.info("fetching_user")
    
    # Simulate DB query
    user = {"id": user_id, "name": "John"}
    
    log.info("user_fetched", user_name=user["name"])
    
    return user
```

## 5. Log Rotation ve Arşivleme

Production'da log dosyaları büyümeye devam eder. **Log rotation** ile log dosyaları belirli kriterlere göre rotate edilir ve eski log'lar arşivlenir.

### Size-Based Rotation

```python
from logging.handlers import RotatingFileHandler

# 10MB'a ulaşınca rotate et, 5 backup tut
handler = RotatingFileHandler(
    '/var/log/myapp/app.log',
    maxBytes=10*1024*1024,  # 10 MB
    backupCount=5,
)

# Dosyalar:
# app.log          (active)
# app.log.1        (en son rotate)
# app.log.2
# app.log.3
# app.log.4
# app.log.5        (en eski, sonraki rotate'te silinir)
```

### Time-Based Rotation

```python
from logging.handlers import TimedRotatingFileHandler

# Her gün gece yarısı rotate et
handler = TimedRotatingFileHandler(
    '/var/log/myapp/app.log',
    when='midnight',
    interval=1,
    backupCount=30,  # 30 gün sakla
)

# when parametreleri:
# 'S'  - Seconds
# 'M'  - Minutes
# 'H'  - Hours
# 'D'  - Days
# 'W0' - Monday (W1=Tuesday, ..., W6=Sunday)
# 'midnight' - Gece yarısı

# Dosya isimlendirme:
# app.log
# app.log.2025-02-22
# app.log.2025-02-21
# ...
```

### Compression ile Rotation

```python
import gzip
import shutil
from logging.handlers import TimedRotatingFileHandler

class CompressedRotatingFileHandler(TimedRotatingFileHandler):
    """Rotate ettikten sonra compress eden handler"""
    
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
    
    def rotator(self, source, dest):
        """Rotate işlemi sonrası çalışır"""
        # Compress
        with open(source, 'rb') as f_in:
            with gzip.open(f'{dest}.gz', 'wb') as f_out:
                shutil.copyfileobj(f_in, f_out)
        
        # Original dosyayı sil
        os.remove(source)

# Kullanım
handler = CompressedRotatingFileHandler(
    '/var/log/myapp/app.log',
    when='midnight',
    backupCount=90,  # 90 gün sakla (compressed)
)

# Dosyalar:
# app.log
# app.log.2025-02-22.gz
# app.log.2025-02-21.gz
```

### External Log Rotation (logrotate)

```bash
# /etc/logrotate.d/myapp
/var/log/myapp/*.log {
    daily                 # Günlük rotate
    rotate 30             # 30 gün sakla
    compress              # Compress et
    delaycompress         # Son rotate'i compress etme
    missingok             # Dosya yoksa hata verme
    notifempty            # Boş dosyayı rotate etme
    create 0644 www-data www-data  # Yeni dosya permissions
    sharedscripts
    postrotate
        # Application'a SIGHUP gönder (reload için)
        systemctl reload myapp
    endscript
}
```

## 6. Centralized Logging (ELK Stack)

Production ortamlarında, birden fazla server'dan gelen log'ları **merkezi bir yerde** toplamak kritik öneme sahiptir.

![ELK Stack Architecture](/assets/img/posts/elk-stack-architecture.png)
_ELK Stack (Elasticsearch + Logstash + Kibana) Mimarisi_

### ELK Stack Bileşenleri

**Elasticsearch**: Log'ları saklayan ve sorgulayan database  
**Logstash**: Log'ları parse eden ve transform eden pipeline  
**Kibana**: Log'ları visualize eden web UI  
**Filebeat**: Log dosyalarını Logstash'e gönderen agent

### Python → Filebeat → ELK

```python
# 1. JSON formatında log yaz
import logging
from pythonjsonlogger import jsonlogger

handler = logging.FileHandler('/var/log/myapp/app.json')
formatter = jsonlogger.JsonFormatter()
handler.setFormatter(formatter)

logger = logging.getLogger()
logger.addHandler(handler)
logger.setLevel(logging.INFO)

logger.info("Order created", extra={
    'order_id': 'order_123',
    'user_id': 456,
    'amount': 99.99,
})
```

```yaml
# 2. Filebeat configuration (filebeat.yml)
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /var/log/myapp/*.json
  json.keys_under_root: true
  json.add_error_key: true

output.logstash:
  hosts: ["logstash:5044"]
```

```ruby
# 3. Logstash pipeline (logstash.conf)
input {
  beats {
    port => 5044
  }
}

filter {
  # Parse timestamp
  date {
    match => ["timestamp", "ISO8601"]
  }
  
  # Add fields
  mutate {
    add_field => {
      "environment" => "production"
      "application" => "myapp"
    }
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "myapp-logs-%{+YYYY.MM.dd}"
  }
}
```

### Direct Elasticsearch Logging

```bash
pip install python-elasticsearch-logger
```

```python
from cmreslogging.handlers import CMRESHandler
import logging

# Elasticsearch handler
es_handler = CMRESHandler(
    hosts=[{'host': 'elasticsearch', 'port': 9200}],
    auth_type=CMRESHandler.AuthType.NO_AUTH,
    es_index_name="myapp-logs",
    es_doc_type="_doc",
)

logger = logging.getLogger()
logger.addHandler(es_handler)

logger.info("Direct to Elasticsearch", extra={
    'user_id': 123,
    'action': 'login',
})
```

### Grafana Loki Alternative

```bash
pip install python-logging-loki
```

```python
import logging_loki

handler = logging_loki.LokiHandler(
    url="http://loki:3100/loki/api/v1/push",
    tags={"application": "myapp", "environment": "production"},
    version="1",
)

logger = logging.getLogger()
logger.addHandler(handler)

logger.info("Log to Loki")
```

## 7. Correlation IDs ve Distributed Tracing

Microservice mimarilerinde, bir request birden fazla service'i geçer. **Correlation ID** ile tüm log'ları ilişkilendirip trace edebiliriz.

### Request ID Middleware

```python
from fastapi import FastAPI, Request
import uuid
import logging
from contextvars import ContextVar

app = FastAPI()

# Context variable for request ID
request_id_var: ContextVar[str] = ContextVar('request_id', default='')

class RequestIDFilter(logging.Filter):
    """Log'lara request_id ekleyen filter"""
    
    def filter(self, record):
        record.request_id = request_id_var.get('')
        return True

# Logger setup
logger = logging.getLogger()
logger.addFilter(RequestIDFilter())

formatter = logging.Formatter(
    '%(asctime)s [%(request_id)s] %(levelname)s %(message)s'
)

@app.middleware("http")
async def add_request_id(request: Request, call_next):
    """Her request'e unique ID ata"""
    # X-Request-ID header'dan al veya oluştur
    request_id = request.headers.get('X-Request-ID', str(uuid.uuid4()))
    
    # Context'e kaydet
    request_id_var.set(request_id)
    
    # Log
    logger.info(f"Request started: {request.method} {request.url.path}")
    
    # Process request
    response = await call_next(request)
    
    # Response'a ekle
    response.headers['X-Request-ID'] = request_id
    
    logger.info(f"Request completed: {response.status_code}")
    
    return response

@app.get("/users/{user_id}")
async def get_user(user_id: int):
    """Request ID otomatik log'lanır"""
    logger.info(f"Fetching user {user_id}")
    
    # Downstream service call
    # X-Request-ID header'ı forward et!
    headers = {'X-Request-ID': request_id_var.get()}
    # response = httpx.get(f'http://service2/users/{user_id}', headers=headers)
    
    return {"id": user_id}

# Log output:
# 2025-02-22 10:30:15 [a1b2c3d4] INFO Request started: GET /users/123
# 2025-02-22 10:30:15 [a1b2c3d4] INFO Fetching user 123
# 2025-02-22 10:30:16 [a1b2c3d4] INFO Request completed: 200
```

### OpenTelemetry Integration

```bash
pip install opentelemetry-api opentelemetry-sdk opentelemetry-instrumentation-logging
```

```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor, ConsoleSpanExporter
from opentelemetry.instrumentation.logging import LoggingInstrumentor
import logging

# Setup OpenTelemetry
trace.set_tracer_provider(TracerProvider())
trace.get_tracer_provider().add_span_processor(
    BatchSpanProcessor(ConsoleSpanExporter())
)

# Instrument logging
LoggingInstrumentor().instrument(set_logging_format=True)

tracer = trace.get_tracer(__name__)
logger = logging.getLogger(__name__)

@app.get("/process")
async def process_data():
    with tracer.start_as_current_span("process_data"):
        logger.info("Processing started")
        
        # Nested span
        with tracer.start_as_current_span("database_query"):
            logger.info("Querying database")
            # DB query
        
        with tracer.start_as_current_span("external_api_call"):
            logger.info("Calling external API")
            # API call
        
        logger.info("Processing completed")
    
    return {"status": "processed"}

# Log'lar otomatik trace_id ve span_id içerir!
```

## 8. Performance Considerations

### Async Logging (QueueHandler)

```python
import logging
from logging.handlers import QueueHandler, QueueListener
import queue

# Queue oluştur
log_queue = queue.Queue()

# QueueHandler (non-blocking)
queue_handler = QueueHandler(log_queue)

# Actual handler (blocking olabilir)
file_handler = logging.FileHandler('/var/log/myapp/app.log')
file_handler.setFormatter(logging.Formatter('%(asctime)s %(message)s'))

# QueueListener (arka planda çalışır)
listener = QueueListener(log_queue, file_handler, respect_handler_level=True)
listener.start()

# Logger'a QueueHandler ekle
logger = logging.getLogger()
logger.addHandler(queue_handler)
logger.setLevel(logging.INFO)

# Logging non-blocking!
logger.info("This is fast!")  # Anında döner

# Application shutdown'da
# listener.stop()
```

### Sampling (High-Throughput Scenarios)

```python
import logging
import random

class SamplingFilter(logging.Filter):
    """Log'ların sadece %10'unu geçir"""
    
    def __init__(self, sample_rate=0.1):
        super().__init__()
        self.sample_rate = sample_rate
    
    def filter(self, record):
        # ERROR ve üstü her zaman geçer
        if record.levelno >= logging.ERROR:
            return True
        
        # Diğerleri sample rate'e göre
        return random.random() < self.sample_rate

# Sadece %10 INFO/DEBUG log'u
handler = logging.StreamHandler()
handler.addFilter(SamplingFilter(sample_rate=0.1))
logger.addHandler(handler)
```

### Conditional Expensive Operations

```python
# ❌ KÖTÜ: Her zaman serialize edilir
logger.debug(f"Data: {json.dumps(large_dict)}")

# ✅ İYİ: Sadece DEBUG enabled ise
if logger.isEnabledFor(logging.DEBUG):
    logger.debug(f"Data: {json.dumps(large_dict)}")

# ✅ DAHA İYİ: Lazy evaluation
class LazyString:
    def __init__(self, func):
        self.func = func
    
    def __str__(self):
        return self.func()

logger.debug("Data: %s", LazyString(lambda: json.dumps(large_dict)))
```

## 9. Security ve Sensitive Data

### Sensitive Data Filtering

```python
import logging
import re

class SensitiveDataFilter(logging.Filter):
    """Sensitive data'yı mask eden filter"""
    
    PATTERNS = [
        (re.compile(r'\b\d{16}\b'), '****-****-****-****'),  # Credit card
        (re.compile(r'\b\d{3}-\d{2}-\d{4}\b'), '***-**-****'),  # SSN
        (re.compile(r'password["\']?\s*[:=]\s*["\']?([^"\'}\s]+)', re.I), r'password=***'),
        (re.compile(r'api[_-]?key["\']?\s*[:=]\s*["\']?([^"\'}\s]+)', re.I), r'api_key=***'),
        (re.compile(r'\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b'), 'email@***'),
    ]
    
    def filter(self, record):
        """Mask sensitive data in log message"""
        message = record.getMessage()
        
        for pattern, replacement in self.PATTERNS:
            message = pattern.sub(replacement, message)
        
        # Override message
        record.msg = message
        record.args = ()
        
        return True

# Kullanım
logger = logging.getLogger()
handler = logging.StreamHandler()
handler.addFilter(SensitiveDataFilter())
logger.addHandler(handler)

# Test
logger.info("User card: 4532123456789012")  # → "User card: ****-****-****-****"
logger.info("Password is: secret123")       # → "Password is: ***"
logger.info("API Key: sk-abc123xyz")        # → "API Key: ***"
```

### Secure Logging Best Practices

```python
# ✅ DO's
logger.info(f"Login attempt for user_id={user_id}")
logger.info(f"Order created", extra={'order_id': order_id, 'amount': amount})
logger.error(f"Payment failed for order {order_id}")

# ❌ DON'Ts
logger.info(f"Login: {username}/{password}")  # Password!
logger.debug(f"Credit card: {card_number}")   # PII!
logger.info(f"Session token: {token}")        # Credentials!
logger.error(f"SQL: {query}")                 # SQL injection risk!
```

## 10. Production Best Practices

### Complete Production Setup

```python
# production_logging.py
import logging
import logging.config
import os

LOGGING_CONFIG = {
    'version': 1,
    'disable_existing_loggers': False,
    
    'formatters': {
        'standard': {
            'format': '%(asctime)s [%(levelname)s] %(name)s: %(message)s'
        },
        'json': {
            '()': 'pythonjsonlogger.jsonlogger.JsonFormatter',
            'format': '%(timestamp)s %(level)s %(name)s %(message)s'
        },
    },
    
    'filters': {
        'request_id': {
            '()': 'myapp.logging.RequestIDFilter',
        },
        'sensitive_data': {
            '()': 'myapp.logging.SensitiveDataFilter',
        },
    },
    
    'handlers': {
        'console': {
            'class': 'logging.StreamHandler',
            'level': 'INFO',
            'formatter': 'standard',
            'stream': 'ext://sys.stdout',
        },
        'file': {
            'class': 'logging.handlers.TimedRotatingFileHandler',
            'level': 'DEBUG',
            'formatter': 'json',
            'filename': '/var/log/myapp/app.log',
            'when': 'midnight',
            'backupCount': 30,
            'filters': ['request_id', 'sensitive_data'],
        },
        'error_file': {
            'class': 'logging.handlers.RotatingFileHandler',
            'level': 'ERROR',
            'formatter': 'json',
            'filename': '/var/log/myapp/error.log',
            'maxBytes': 10485760,  # 10MB
            'backupCount': 10,
        },
        'slack': {
            'class': 'myapp.logging.SlackHandler',
            'level': 'CRITICAL',
            'webhook_url': os.getenv('SLACK_WEBHOOK_URL'),
        },
    },
    
    'loggers': {
        'myapp': {
            'level': 'DEBUG',
            'handlers': ['console', 'file', 'error_file'],
            'propagate': False,
        },
        'myapp.api': {
            'level': 'INFO',
        },
        'myapp.database': {
            'level': 'WARNING',
        },
        'uvicorn': {
            'level': 'INFO',
            'handlers': ['console'],
        },
        'sqlalchemy.engine': {
            'level': 'WARNING',  # SQL query'leri loglama
        },
    },
    
    'root': {
        'level': 'INFO',
        'handlers': ['console', 'file'],
    }
}

def setup_logging():
    """Setup production logging configuration"""
    logging.config.dictConfig(LOGGING_CONFIG)
    logger = logging.getLogger(__name__)
    logger.info("Logging configured successfully")

# Application startup
if __name__ == '__main__':
    setup_logging()
```

### Environment-Based Configuration

```python
import os

def get_log_level():
    """Get log level from environment"""
    env = os.getenv('ENVIRONMENT', 'development')
    
    levels = {
        'development': logging.DEBUG,
        'staging': logging.INFO,
        'production': logging.WARNING,
    }
    
    return levels.get(env, logging.INFO)

# Usage
logger.setLevel(get_log_level())
```

### Health Check Endpoint

```python
from fastapi import FastAPI
import logging

app = FastAPI()

@app.get("/health/logging")
async def logging_health():
    """Check if logging system is healthy"""
    logger = logging.getLogger()
    
    healthy = True
    issues = []
    
    # Check handlers
    if not logger.handlers:
        healthy = False
        issues.append("No handlers configured")
    
    # Check file handler writable
    for handler in logger.handlers:
        if isinstance(handler, logging.FileHandler):
            try:
                handler.stream.write("")  # Test write
            except Exception as e:
                healthy = False
                issues.append(f"File handler not writable: {e}")
    
    status = "healthy" if healthy else "unhealthy"
    
    return {
        "status": status,
        "handlers": len(logger.handlers),
        "issues": issues,
    }
```

## Sonuç

**Production-grade logging**, başarılı bir uygulamanın temel taşlarından biridir. İyi yapılandırılmış bir logging sistemi:

✅ **Debugging'i hızlandırır** (correlation IDs, structured logs)  
✅ **Incident response'u iyileştirir** (centralized logging, alerts)  
✅ **System health'i gösterir** (metrics, monitoring)  
✅ **Audit trail sağlar** (who did what when)  
✅ **Performance insight verir** (slow queries, bottlenecks)  

###核心 Best Practices

1. **Structured Logging kullan** (JSON formatı)
2. **Log levels'ı doğru kullan** (DEBUG/INFO/WARNING/ERROR/CRITICAL)
3. **Centralized logging** yap (ELK, Loki)
4. **Correlation IDs** ekle (distributed tracing)
5. **Sensitive data'yı maskele** (PII, credentials)
6. **Log rotation** uygula (disk dolmasın)
7. **Performance'a dikkat et** (async logging, sampling)
8. **Test et** (logging system health checks)

Modern cloud-native uygulamalarda, logging artık sadece "debug için console.log" değil - observability stack'inin kritik bir parçasıdır. Metrics, traces ve logs birlikte kullanıldığında, sistemlerinizi tam anlamıyla gözlemlenebilir hale getirirsiniz!

## Kaynaklar

- [Python Logging Documentation](https://docs.python.org/3/library/logging.html)
- [Structlog](https://www.structlog.org/)
- [ELK Stack](https://www.elastic.co/elastic-stack)
- [OpenTelemetry Python](https://opentelemetry.io/docs/instrumentation/python/)
- [Grafana Loki](https://grafana.com/oss/loki/)
- [12 Factor App: Logs](https://12factor.net/logs)
