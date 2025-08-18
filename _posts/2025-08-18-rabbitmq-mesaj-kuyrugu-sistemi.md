---
title: "RabbitMQ ile Mesaj Kuyruğu Sistemleri: Producer, Consumer ve Routing"
date: 2025-08-18 09:00:00 +0300
categories: [Backend, Message-Queue]
tags: [rabbitmq, message-queue, amqp, microservices, async]
image:
  path: /assets/img/posts/rabbitmq-architecture-diagram.png
  alt: "RabbitMQ Mimarisi ve Bileşenleri"
---

Modern mikroservis mimarilerinde asenkron iletişim ve mesaj kuyruğu sistemleri kritik rol oynar. RabbitMQ, AMQP protokolü üzerine kurulu, güvenilir ve ölçeklenebilir bir mesaj broker'ıdır. Bu yazıda RabbitMQ'nun temellerinden ileri seviye kullanım senaryolarına kadar kapsamlı bir rehber sunacağız.

## RabbitMQ Nedir?

RabbitMQ, Erlang dilinde yazılmış açık kaynaklı bir message broker'dır. Producer ve consumer arasında mesaj iletimini yönetir, mesajların güvenli bir şekilde saklanmasını ve iletilmesini sağlar.

### Temel Kavramlar

```plaintext
# RabbitMQ Bileşenleri
Producer (Üretici)
    ↓ (Mesaj gönderir)
Exchange (Dağıtıcı)
    ↓ (Routing key ile yönlendirir)
Queue (Kuyruk)
    ↓ (Mesaj tutar)
Consumer (Tüketici)
```

- **Producer**: Mesaj gönderen uygulama
- **Consumer**: Mesaj alan uygulama
- **Queue**: Mesajların saklandığı buffer
- **Exchange**: Mesajları routing key'e göre kuyruklara yönlendiren component
- **Binding**: Exchange ile queue arasındaki bağlantı
- **Routing Key**: Mesajın hangi kuyruğa gideceğini belirleyen key
- **Virtual Host**: Logical izolasyon sağlayan namespace

### Kullanım Senaryoları

```bash
# Asenkron İşlemler
- Email/SMS gönderimi
- Image/video processing
- Report generation
- Batch jobs

# Mikroservis İletişimi
- Event-driven architecture
- Service decoupling
- Load distribution

# Veri Pipeline
- Log aggregation
- Stream processing
- ETL processes
```

## Kurulum ve Başlangıç

### Docker ile Hızlı Başlangıç

```bash
# RabbitMQ with Management UI
docker run -d \
  --name rabbitmq \
  -p 5672:5672 \
  -p 15672:15672 \
  -e RABBITMQ_DEFAULT_USER=admin \
  -e RABBITMQ_DEFAULT_PASS=admin123 \
  rabbitmq:3-management

# Management UI: http://localhost:15672
# Credentials: admin / admin123
```

### Python ile RabbitMQ (pika)

```bash
pip install pika
```

```python
import pika
import json

# Bağlantı oluşturma
credentials = pika.PlainCredentials('admin', 'admin123')
parameters = pika.ConnectionParameters(
    host='localhost',
    port=5672,
    credentials=credentials,
    virtual_host='/',
    heartbeat=600,
    blocked_connection_timeout=300
)

connection = pika.BlockingConnection(parameters)
channel = connection.channel()

# Queue declare
channel.queue_declare(
    queue='hello',
    durable=True,  # Restart sonrası kalıcı
    exclusive=False,
    auto_delete=False
)

print("RabbitMQ bağlantısı başarılı!")
```

## Basic Producer-Consumer

### Simple Producer

```python
import pika
import json

def send_message(queue_name: str, message: dict):
    """Temel mesaj gönderme"""
    # Bağlantı
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()
    
    # Queue declare
    channel.queue_declare(queue=queue_name, durable=True)
    
    # Mesaj gönder
    channel.basic_publish(
        exchange='',
        routing_key=queue_name,
        body=json.dumps(message),
        properties=pika.BasicProperties(
            delivery_mode=2,  # Persistent message
            content_type='application/json'
        )
    )
    
    print(f"Mesaj gönderildi: {message}")
    connection.close()

# Kullanım
send_message('tasks', {'task_id': 1, 'action': 'send_email'})
```

### Simple Consumer

```python
import pika
import json
import time

def process_message(ch, method, properties, body):
    """Callback function - mesaj işleme"""
    try:
        message = json.loads(body)
        print(f"Mesaj alındı: {message}")
        
        # İş mantığı (örnek: 2 saniye)
        time.sleep(2)
        print("İşlem tamamlandı")
        
        # ACK (acknowledge) - mesaj başarıyla işlendi
        ch.basic_ack(delivery_tag=method.delivery_tag)
        
    except Exception as e:
        print(f"Hata: {e}")
        # NACK - mesaj tekrar kuyruğa dönecek
        ch.basic_nack(delivery_tag=method.delivery_tag, requeue=True)

def start_consumer(queue_name: str):
    """Consumer başlat"""
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()
    
    # Queue declare (idempotent)
    channel.queue_declare(queue=queue_name, durable=True)
    
    # QoS (Quality of Service) - aynı anda max 1 mesaj
    channel.basic_qos(prefetch_count=1)
    
    # Consumer callback
    channel.basic_consume(
        queue=queue_name,
        on_message_callback=process_message,
        auto_ack=False  # Manuel ACK
    )
    
    print(f"Consumer başlatıldı. '{queue_name}' kuyruğu dinleniyor...")
    channel.start_consuming()

# Kullanım
start_consumer('tasks')
```

## Exchange Types

![RabbitMQ Exchange Routing](/assets/img/posts/rabbitmq-exchanges-routing.png)
_RabbitMQ exchange türleri: Direct, Fanout, Topic, Headers_

### Direct Exchange

```python
def setup_direct_exchange():
    """Direct exchange - routing key exact match"""
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()
    
    # Exchange declare
    channel.exchange_declare(
        exchange='logs_direct',
        exchange_type='direct',
        durable=True
    )
    
    # Queues
    severities = ['info', 'warning', 'error']
    for severity in severities:
        queue_name = f'log_{severity}'
        channel.queue_declare(queue=queue_name, durable=True)
        
        # Binding
        channel.queue_bind(
            exchange='logs_direct',
            queue=queue_name,
            routing_key=severity
        )
    
    connection.close()
    print("Direct exchange yapılandırıldı")

# Producer
def send_log(severity: str, message: str):
    """Log mesajı gönder"""
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()
    
    channel.basic_publish(
        exchange='logs_direct',
        routing_key=severity,  # info, warning, error
        body=message,
        properties=pika.BasicProperties(delivery_mode=2)
    )
    
    print(f"[{severity}] Log gönderildi: {message}")
    connection.close()

# Kullanım
send_log('error', 'Database connection failed')
send_log('warning', 'High memory usage detected')
```

### Fanout Exchange

```python
def setup_fanout_exchange():
    """Fanout exchange - tüm kuyruklara broadcast"""
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()
    
    # Exchange
    channel.exchange_declare(
        exchange='notifications',
        exchange_type='fanout',
        durable=True
    )
    
    # Multiple queues (email, sms, push)
    services = ['email_service', 'sms_service', 'push_service']
    for service in services:
        channel.queue_declare(queue=service, durable=True)
        channel.queue_bind(exchange='notifications', queue=service)
    
    connection.close()

def broadcast_notification(message: dict):
    """Tüm servislere bildirim gönder"""
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()
    
    channel.basic_publish(
        exchange='notifications',
        routing_key='',  # Fanout'ta önemsiz
        body=json.dumps(message),
        properties=pika.BasicProperties(delivery_mode=2)
    )
    
    print(f"Broadcast gönderildi: {message}")
    connection.close()

# Kullanım
broadcast_notification({
    'user_id': 123,
    'title': 'Yeni mesajınız var',
    'body': 'Lorem ipsum...'
})
```

### Topic Exchange

```python
def setup_topic_exchange():
    """Topic exchange - pattern matching routing"""
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()
    
    # Exchange
    channel.exchange_declare(
        exchange='logs_topic',
        exchange_type='topic',
        durable=True
    )
    
    # Bindings with patterns
    bindings = [
        ('all_logs', '#'),  # Tüm loglar
        ('error_logs', '*.error.*'),  # Her servisin error logları
        ('api_logs', 'api.*.*'),  # API'nin tüm logları
        ('critical_logs', '*.*.critical')  # Tüm critical loglar
    ]
    
    for queue_name, routing_pattern in bindings:
        channel.queue_declare(queue=queue_name, durable=True)
        channel.queue_bind(
            exchange='logs_topic',
            queue=queue_name,
            routing_key=routing_pattern
        )
    
    connection.close()

def send_topic_log(routing_key: str, message: str):
    """Topic routing ile log gönder
    routing_key format: <service>.<level>.<severity>
    Örnek: api.error.critical
    """
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()
    
    channel.basic_publish(
        exchange='logs_topic',
        routing_key=routing_key,
        body=message,
        properties=pika.BasicProperties(delivery_mode=2)
    )
    
    print(f"[{routing_key}] Log gönderildi")
    connection.close()

# Kullanım
send_topic_log('api.error.critical', 'API sunucusu yanıt vermiyor')
send_topic_log('database.warning.medium', 'Slow query detected')
```

## RabbitMQ Connection Manager

```python
import pika
import json
import logging
from typing import Callable, Optional, Dict, Any
from contextlib import contextmanager

class RabbitMQManager:
    """RabbitMQ connection ve channel yönetimi"""
    
    def __init__(
        self,
        host: str = 'localhost',
        port: int = 5672,
        username: str = 'guest',
        password: str = 'guest',
        virtual_host: str = '/'
    ):
        self.host = host
        self.port = port
        self.username = username
        self.password = password
        self.virtual_host = virtual_host
        self.logger = logging.getLogger(__name__)
        
        self.connection: Optional[pika.BlockingConnection] = None
        self.channel: Optional[pika.channel.Channel] = None
    
    def connect(self):
        """RabbitMQ'ya bağlan"""
        try:
            credentials = pika.PlainCredentials(self.username, self.password)
            parameters = pika.ConnectionParameters(
                host=self.host,
                port=self.port,
                virtual_host=self.virtual_host,
                credentials=credentials,
                heartbeat=600,
                blocked_connection_timeout=300
            )
            
            self.connection = pika.BlockingConnection(parameters)
            self.channel = self.connection.channel()
            self.logger.info("RabbitMQ bağlantısı başarılı")
            
        except Exception as e:
            self.logger.error(f"Bağlantı hatası: {e}")
            raise
    
    def close(self):
        """Bağlantıyı kapat"""
        if self.connection and not self.connection.is_closed:
            self.connection.close()
            self.logger.info("RabbitMQ bağlantısı kapatıldı")
    
    @contextmanager
    def get_channel(self):
        """Context manager ile channel kullanımı"""
        try:
            self.connect()
            yield self.channel
        finally:
            self.close()
    
    def declare_queue(
        self,
        queue_name: str,
        durable: bool = True,
        exclusive: bool = False,
        auto_delete: bool = False,
        arguments: Optional[Dict] = None
    ):
        """Queue declare"""
        if not self.channel:
            self.connect()
        
        self.channel.queue_declare(
            queue=queue_name,
            durable=durable,
            exclusive=exclusive,
            auto_delete=auto_delete,
            arguments=arguments or {}
        )
        self.logger.info(f"Queue declared: {queue_name}")
    
    def declare_exchange(
        self,
        exchange_name: str,
        exchange_type: str = 'direct',
        durable: bool = True
    ):
        """Exchange declare"""
        if not self.channel:
            self.connect()
        
        self.channel.exchange_declare(
            exchange=exchange_name,
            exchange_type=exchange_type,
            durable=durable
        )
        self.logger.info(f"Exchange declared: {exchange_name} ({exchange_type})")
    
    def bind_queue(
        self,
        queue_name: str,
        exchange_name: str,
        routing_key: str = ''
    ):
        """Queue-exchange binding"""
        if not self.channel:
            self.connect()
        
        self.channel.queue_bind(
            queue=queue_name,
            exchange=exchange_name,
            routing_key=routing_key
        )
        self.logger.info(f"Binding: {queue_name} <- {exchange_name} ({routing_key})")
    
    def publish(
        self,
        message: Dict[str, Any],
        exchange: str = '',
        routing_key: str = '',
        persistent: bool = True
    ):
        """Mesaj publish et"""
        if not self.channel:
            self.connect()
        
        properties = pika.BasicProperties(
            delivery_mode=2 if persistent else 1,
            content_type='application/json'
        )
        
        self.channel.basic_publish(
            exchange=exchange,
            routing_key=routing_key,
            body=json.dumps(message),
            properties=properties
        )
        self.logger.info(f"Mesaj gönderildi: {routing_key}")
    
    def consume(
        self,
        queue_name: str,
        callback: Callable,
        auto_ack: bool = False,
        prefetch_count: int = 1
    ):
        """Consumer başlat"""
        if not self.channel:
            self.connect()
        
        self.channel.basic_qos(prefetch_count=prefetch_count)
        self.channel.basic_consume(
            queue=queue_name,
            on_message_callback=callback,
            auto_ack=auto_ack
        )
        
        self.logger.info(f"Consumer başlatıldı: {queue_name}")
        self.channel.start_consuming()

# Kullanım
rabbitmq = RabbitMQManager(
    host='localhost',
    username='admin',
    password='admin123'
)

# Setup
rabbitmq.connect()
rabbitmq.declare_queue('tasks')
rabbitmq.publish({'task_id': 1}, routing_key='tasks')
rabbitmq.close()
```

## İleri Seviye Özellikler

![RabbitMQ Integration Patterns](/assets/img/posts/rabbitmq-integration-patterns.png)
_RabbitMQ entegrasyon pattern'leri ve kullanım senaryoları_

### Dead Letter Exchange (DLX)

```python
def setup_dlx():
    """Dead Letter Exchange - hatalı mesajlar için"""
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()
    
    # DLX exchange
    channel.exchange_declare(
        exchange='dlx_exchange',
        exchange_type='direct',
        durable=True
    )
    
    # DLX queue
    channel.queue_declare(queue='dlx_queue', durable=True)
    channel.queue_bind(
        exchange='dlx_exchange',
        queue='dlx_queue',
        routing_key='failed'
    )
    
    # Ana queue (DLX ile)
    channel.queue_declare(
        queue='main_queue',
        durable=True,
        arguments={
            'x-dead-letter-exchange': 'dlx_exchange',
            'x-dead-letter-routing-key': 'failed',
            'x-message-ttl': 60000,  # 60 saniye TTL
            'x-max-length': 1000  # Max queue size
        }
    )
    
    connection.close()
```

### Priority Queue

```python
def setup_priority_queue():
    """Priority queue - öncelikli mesajlar"""
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()
    
    # Priority queue declare
    channel.queue_declare(
        queue='priority_tasks',
        durable=True,
        arguments={'x-max-priority': 10}  # 0-10 arası priority
    )
    
    connection.close()

def send_priority_message(message: dict, priority: int):
    """Öncelikli mesaj gönder"""
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()
    
    channel.basic_publish(
        exchange='',
        routing_key='priority_tasks',
        body=json.dumps(message),
        properties=pika.BasicProperties(
            delivery_mode=2,
            priority=priority  # 0-10
        )
    )
    
    print(f"Priority {priority} mesaj gönderildi")
    connection.close()

# Kullanım
send_priority_message({'task': 'urgent'}, priority=10)  # En yüksek
send_priority_message({'task': 'normal'}, priority=5)
send_priority_message({'task': 'low'}, priority=1)
```

### Message TTL (Time To Live)

```python
def send_with_ttl(message: dict, ttl_ms: int):
    """TTL'li mesaj gönder"""
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()
    
    channel.queue_declare(queue='ttl_queue', durable=True)
    
    channel.basic_publish(
        exchange='',
        routing_key='ttl_queue',
        body=json.dumps(message),
        properties=pika.BasicProperties(
            delivery_mode=2,
            expiration=str(ttl_ms)  # Milisaniye cinsinden
        )
    )
    
    print(f"TTL {ttl_ms}ms mesaj gönderildi")
    connection.close()

# 10 saniye sonra expire olacak mesaj
send_with_ttl({'task': 'temporary'}, ttl_ms=10000)
```

### RPC Pattern (Request-Reply)

```python
import uuid

class RabbitMQRPC:
    """RPC client implementation"""
    
    def __init__(self):
        self.connection = pika.BlockingConnection(
            pika.ConnectionParameters('localhost')
        )
        self.channel = self.connection.channel()
        
        # Callback queue
        result = self.channel.queue_declare(queue='', exclusive=True)
        self.callback_queue = result.method.queue
        
        self.channel.basic_consume(
            queue=self.callback_queue,
            on_message_callback=self.on_response,
            auto_ack=True
        )
        
        self.response = None
        self.corr_id = None
    
    def on_response(self, ch, method, props, body):
        """RPC response handler"""
        if self.corr_id == props.correlation_id:
            self.response = body.decode()
    
    def call(self, n: int) -> str:
        """RPC çağrısı yap"""
        self.response = None
        self.corr_id = str(uuid.uuid4())
        
        self.channel.basic_publish(
            exchange='',
            routing_key='rpc_queue',
            properties=pika.BasicProperties(
                reply_to=self.callback_queue,
                correlation_id=self.corr_id,
            ),
            body=str(n)
        )
        
        # Response bekle
        while self.response is None:
            self.connection.process_data_events()
        
        return self.response

# RPC Server
def fibonacci(n):
    if n == 0:
        return 0
    elif n == 1:
        return 1
    else:
        return fibonacci(n-1) + fibonacci(n-2)

def on_rpc_request(ch, method, props, body):
    """RPC request handler"""
    n = int(body)
    print(f"Fibonacci({n}) hesaplanıyor...")
    
    response = fibonacci(n)
    
    ch.basic_publish(
        exchange='',
        routing_key=props.reply_to,
        properties=pika.BasicProperties(
            correlation_id=props.correlation_id
        ),
        body=str(response)
    )
    
    ch.basic_ack(delivery_tag=method.delivery_tag)

# Kullanım
rpc = RabbitMQRPC()
print("RPC istek gönderiliyor...")
response = rpc.call(10)
print(f"Response: {response}")
```

## Best Practices

### Connection Pooling

```python
from queue import Queue
import threading

class ConnectionPool:
    """RabbitMQ connection pool"""
    
    def __init__(self, size: int = 5):
        self.size = size
        self.pool = Queue(maxsize=size)
        self._initialize_pool()
    
    def _initialize_pool(self):
        """Pool'u doldur"""
        for _ in range(self.size):
            conn = pika.BlockingConnection(
                pika.ConnectionParameters('localhost')
            )
            self.pool.put(conn)
    
    def get_connection(self):
        """Pool'dan connection al"""
        return self.pool.get()
    
    def return_connection(self, connection):
        """Connection'ı pool'a geri ver"""
        self.pool.put(connection)
    
    def close_all(self):
        """Tüm connection'ları kapat"""
        while not self.pool.empty():
            conn = self.pool.get()
            conn.close()
```

### Retry Mechanism

```python
import time
from functools import wraps

def retry(max_attempts: int = 3, delay: int = 1):
    """Retry decorator"""
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(1, max_attempts + 1):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    if attempt == max_attempts:
                        raise
                    print(f"Attempt {attempt} failed: {e}. Retrying...")
                    time.sleep(delay * attempt)
        return wrapper
    return decorator

@retry(max_attempts=3, delay=2)
def publish_with_retry(message: dict):
    """Retry ile mesaj gönder"""
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()
    
    channel.basic_publish(
        exchange='',
        routing_key='tasks',
        body=json.dumps(message),
        properties=pika.BasicProperties(delivery_mode=2)
    )
    
    connection.close()
```

## Monitoring

```python
def get_queue_stats(queue_name: str) -> dict:
    """Queue istatistikleri"""
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()
    
    # Passive declare - queue'yu değiştirmeden bilgi al
    method = channel.queue_declare(queue=queue_name, passive=True)
    
    stats = {
        'queue': queue_name,
        'message_count': method.method.message_count,
        'consumer_count': method.method.consumer_count
    }
    
    connection.close()
    return stats

# Kullanım
stats = get_queue_stats('tasks')
print(f"Bekleyen mesaj: {stats['message_count']}")
print(f"Aktif consumer: {stats['consumer_count']}")
```

## Sonuç

RabbitMQ, mikroservis mimarilerinde asenkron iletişim ve event-driven sistemler için güçlü bir altyapı sağlar. Bu yazıda ele aldığımız konular:

1. **Temel Kavramlar**: Producer, consumer, exchange, queue
2. **Exchange Types**: Direct, fanout, topic, headers
3. **İleri Özellikler**: DLX, priority queue, TTL, RPC
4. **Best Practices**: Connection pooling, retry mechanism
5. **Monitoring**: Queue istatistikleri ve health checks

Production ortamında cluster kurulumu, HA (high availability), monitoring (Prometheus/Grafana) ve disaster recovery stratejileri mutlaka planlanmalıdır. RabbitMQ Management Plugin ile web tabanlı izleme ve yönetim de mümkündür.

## Kaynaklar

- [RabbitMQ Official Documentation](https://www.rabbitmq.com/documentation.html)
- [RabbitMQ Tutorials](https://www.rabbitmq.com/getstarted.html)
- [pika Documentation](https://pika.readthedocs.io/)
- [CloudAMQP RabbitMQ Guide](https://www.cloudamqp.com/blog/index.html)
