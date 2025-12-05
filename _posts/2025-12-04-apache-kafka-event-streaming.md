---
title: "Apache Kafka ile Event Streaming: Gerçek Zamanlı Veri Pipeline'ları"
description: "Apache Kafka ile gerçek zamanlı event streaming. Topic, partition, producer/consumer patterns, Python kafka-python entegrasyonu, fault tolerance ve distributed systems best practices."
date: "2025-12-04 14:00:00 +0300"
categories: [Data Engineering, Streaming]
tags: [kafka, event-streaming, python, kafka-python, real-time, data-pipeline, distributed-systems, messaging]
image:
  path: /assets/img/posts/kafka-architecture-producer-consumer.png
  alt: "Apache Kafka Architecture - Producer Consumer Model"
---

Apache Kafka, yüksek throughput, düşük latency ve fault-tolerant özellikleriyle gerçek zamanlı veri akışı (event streaming) için endüstri standardı haline gelmiş bir distributed streaming platformudur. Bu yazıda, Kafka'nın temellerinden başlayarak Python ile pratik implementasyonlara kadar detaylı bir rehber sunacağız.

## Apache Kafka Nedir?

Kafka, LinkedIn tarafından geliştirilip Apache Software Foundation'a bağışlanan, publish-subscribe (yayın-abone) modeline dayanan bir mesajlaşma sistemidir. Geleneksel message broker'lardan farklı olarak:

- **Yüksek throughput**: Saniyede milyonlarca mesaj işleyebilir
- **Düşük latency**: Milisaniye seviyesinde gecikme
- **Fault-tolerant**: Veri kaybı olmadan node hatalarına dayanıklı
- **Scalable**: Yatay olarak ölçeklenebilir
- **Persistent**: Mesajlar disk'te saklanır

![Kafka Architecture](/assets/img/posts/kafka-architecture-producer-consumer.png){: w="800" h="500" .shadow }
_Apache Kafka mimarisi - Producer, Broker, Consumer modeli_

> Kafka, geleneksel message queue'lardan farklı olarak mesajları siler değil, belirli bir süre saklar (retention policy). Aynı mesajlar birden fazla consumer tarafından okunabilir.
{: .prompt-tip }

### Kafka Temel Kavramları

**1. Topic (Konu)**
```bash
# Topic, mesajların kategorize edildiği kanaldır
# Örnek topic'ler:
- user-registrations
- order-events
- payment-transactions
- sensor-data
```

**2. Partition (Bölüm)**
```python
# Topic'ler partition'lara bölünür (paralel işlem için)
# Partition sayısı throughput'u belirler

# 3 partition'lu topic
user-events
├── partition-0
├── partition-1
└── partition-2

# Mesajlar key'e göre partition'lara dağıtılır
# Aynı key'e sahip mesajlar her zaman aynı partition'a gider
```

**3. Producer (Üretici)**
```python
# Mesaj gönderen uygulamalar
# Topic'e mesaj yazar
# Partition seçimini yapabilir veya Kafka'ya bırakabilir
```

**4. Consumer (Tüketici)**
```python
# Mesaj okuyan uygulamalar
# Consumer Group içinde çalışırlar
# Her partition bir consumer tarafından okunur
```

**5. Broker**
```bash
# Kafka sunucuları
# Mesajları saklar ve serve eder
# Cluster içinde replicate eder
```

**6. Offset**
```python
# Her partition'daki mesajın benzersiz ID'si
# Consumer'lar offset'i takip ederek okumaya devam eder
# 0'dan başlar, sıralı artar
```

## Kafka Kurulumu ve Konfigürasyonu

### Docker ile Hızlı Kurulum

```yaml
version: '3.8'

services:
  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
    ports:
      - "2181:2181"
    volumes:
      - zookeeper-data:/var/lib/zookeeper/data
      - zookeeper-logs:/var/lib/zookeeper/log

  kafka:
    image: confluentinc/cp-kafka:7.5.0
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
      - "9093:9093"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092,PLAINTEXT_INTERNAL://kafka:9093
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_INTERNAL:PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: "true"
      KAFKA_LOG_RETENTION_HOURS: 168
      KAFKA_LOG_SEGMENT_BYTES: 1073741824
    volumes:
      - kafka-data:/var/lib/kafka/data

  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    depends_on:
      - kafka
    ports:
      - "8080:8080"
    environment:
      KAFKA_CLUSTERS_0_NAME: local
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:9093
      KAFKA_CLUSTERS_0_ZOOKEEPER: zookeeper:2181

volumes:
  zookeeper-data:
  zookeeper-logs:
  kafka-data:
```

```bash
# Kafka'yı başlat
docker-compose up -d

# Kafka UI'a eriş: http://localhost:8080

# Kafka durumunu kontrol et
docker-compose ps

# Log'ları görüntüle
docker-compose logs -f kafka
```

### Python Kafka Client Kurulumu

```bash
# kafka-python kütüphanesi
pip install kafka-python

# Alternatif: confluent-kafka (daha performanslı)
pip install confluent-kafka

# Avro serialization için
pip install avro-python3
pip install confluent-kafka[avro]
```

## Kafka Producer: Mesaj Gönderme

### Basit Producer

```python
from kafka import KafkaProducer
import json
import time
from datetime import datetime

# Producer oluştur
producer = KafkaProducer(
    bootstrap_servers=['localhost:9092'],
    value_serializer=lambda v: json.dumps(v).encode('utf-8'),
    key_serializer=lambda k: k.encode('utf-8') if k else None,
    # Performans ve güvenilirlik ayarları
    acks='all',  # Tüm replica'lardan onay bekle
    retries=3,   # Hata durumunda yeniden deneme
    max_in_flight_requests_per_connection=1,  # Sıralama garantisi
    compression_type='gzip',  # Veri sıkıştırma
)

# Mesaj gönder
def send_message(topic, key, value):
    """
    Topic'e mesaj gönder
    
    Args:
        topic: Topic adı
        key: Mesaj key'i (partition seçimi için)
        value: Mesaj içeriği (dict)
    """
    try:
        # Asenkron gönderim
        future = producer.send(
            topic=topic,
            key=key,
            value=value,
            partition=None,  # Key'e göre otomatik seçim
        )
        
        # Metadata al (blocking)
        record_metadata = future.get(timeout=10)
        
        print(f"Message sent to {record_metadata.topic}")
        print(f"Partition: {record_metadata.partition}")
        print(f"Offset: {record_metadata.offset}")
        
        return record_metadata
        
    except Exception as e:
        print(f"Error sending message: {e}")
        raise

# Örnek kullanım
user_event = {
    'user_id': 12345,
    'action': 'login',
    'timestamp': datetime.now().isoformat(),
    'ip_address': '192.168.1.1',
    'device': 'mobile'
}

send_message(
    topic='user-events',
    key='user-12345',  # Aynı user her zaman aynı partition'a
    value=user_event
)

# Producer'ı kapat
producer.close()
```

![Kafka Python Implementation](/assets/img/posts/kafka-python-implementation.png)

### Advanced Producer: Batch ve Callback

```python
from kafka import KafkaProducer
from kafka.errors import KafkaError
import json
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class AdvancedProducer:
    def __init__(self, bootstrap_servers=['localhost:9092']):
        self.producer = KafkaProducer(
            bootstrap_servers=bootstrap_servers,
            value_serializer=lambda v: json.dumps(v).encode('utf-8'),
            key_serializer=lambda k: k.encode('utf-8') if k else None,
            
            # Batch ayarları (throughput için)
            batch_size=16384,  # 16KB batch
            linger_ms=10,      # 10ms bekle, batch'i doldur
            buffer_memory=33554432,  # 32MB buffer
            
            # Güvenilirlik
            acks='all',
            retries=5,
            retry_backoff_ms=100,
            
            # Performans
            compression_type='snappy',
            max_in_flight_requests_per_connection=5,
        )
        
        self.success_count = 0
        self.error_count = 0
    
    def on_success(self, record_metadata):
        """Başarılı gönderim callback'i"""
        self.success_count += 1
        logger.info(
            f"Message delivered to {record_metadata.topic} "
            f"[{record_metadata.partition}] at offset {record_metadata.offset}"
        )
    
    def on_error(self, exc):
        """Hatalı gönderim callback'i"""
        self.error_count += 1
        logger.error(f"Message delivery failed: {exc}")
    
    def send_async(self, topic, key, value):
        """Asenkron mesaj gönderimi"""
        self.producer.send(topic, key=key, value=value) \
            .add_callback(self.on_success) \
            .add_errback(self.on_error)
    
    def send_batch(self, topic, messages):
        """
        Toplu mesaj gönderimi
        
        Args:
            topic: Topic adı
            messages: [(key, value), ...] listesi
        """
        for key, value in messages:
            self.send_async(topic, key, value)
        
        # Buffer'daki tüm mesajları gönder
        self.producer.flush()
        
        logger.info(f"Batch sent: {self.success_count} success, {self.error_count} errors")
    
    def close(self):
        """Producer'ı kapat"""
        self.producer.flush()  # Bekleyen mesajları gönder
        self.producer.close()

# Kullanım
producer = AdvancedProducer()

# Batch mesaj gönderimi
events = [
    ('user-1', {'action': 'login', 'timestamp': '2025-12-04T10:00:00'}),
    ('user-2', {'action': 'purchase', 'amount': 99.99}),
    ('user-3', {'action': 'logout', 'session_duration': 3600}),
]

producer.send_batch('user-events', events)
producer.close()
```

## Kafka Consumer: Mesaj Okuma

### Basit Consumer

```python
from kafka import KafkaConsumer
import json

# Consumer oluştur
consumer = KafkaConsumer(
    'user-events',  # Topic adı
    bootstrap_servers=['localhost:9092'],
    value_deserializer=lambda m: json.loads(m.decode('utf-8')),
    key_deserializer=lambda k: k.decode('utf-8') if k else None,
    
    # Consumer group
    group_id='user-events-group',
    
    # Offset yönetimi
    auto_offset_reset='earliest',  # 'earliest' veya 'latest'
    enable_auto_commit=True,       # Otomatik offset commit
    auto_commit_interval_ms=5000,  # 5 saniyede bir commit
    
    # Performans
    max_poll_records=500,          # Her poll'da maksimum mesaj
    fetch_min_bytes=1,             # Minimum fetch boyutu
    fetch_max_wait_ms=500,         # Maksimum bekleme
)

print("Listening for messages...")

try:
    for message in consumer:
        print(f"\n--- New Message ---")
        print(f"Topic: {message.topic}")
        print(f"Partition: {message.partition}")
        print(f"Offset: {message.offset}")
        print(f"Key: {message.key}")
        print(f"Value: {message.value}")
        print(f"Timestamp: {message.timestamp}")
        
        # İş mantığı
        process_event(message.value)
        
except KeyboardInterrupt:
    print("\nShutting down consumer...")
finally:
    consumer.close()

def process_event(event):
    """Event işleme mantığı"""
    action = event.get('action')
    
    if action == 'login':
        print(f"User {event['user_id']} logged in")
    elif action == 'purchase':
        print(f"Purchase: ${event['amount']}")
    elif action == 'logout':
        print(f"Session ended: {event['session_duration']}s")
```

### Advanced Consumer: Manuel Commit ve Error Handling

```python
from kafka import KafkaConsumer, TopicPartition
from kafka.errors import KafkaError
import json
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class AdvancedConsumer:
    def __init__(self, topics, group_id, bootstrap_servers=['localhost:9092']):
        self.consumer = KafkaConsumer(
            *topics,
            bootstrap_servers=bootstrap_servers,
            value_deserializer=lambda m: json.loads(m.decode('utf-8')),
            key_deserializer=lambda k: k.decode('utf-8') if k else None,
            group_id=group_id,
            
            # Manuel commit
            enable_auto_commit=False,
            
            # Performans ayarları
            max_poll_records=100,
            max_poll_interval_ms=300000,  # 5 dakika
            session_timeout_ms=10000,     # 10 saniye
            heartbeat_interval_ms=3000,   # 3 saniye
        )
        
        self.processed_count = 0
        self.error_count = 0
    
    def process_message(self, message):
        """
        Mesajı işle
        
        Returns:
            bool: Başarılı ise True
        """
        try:
            logger.info(f"Processing message from offset {message.offset}")
            
            # İş mantığı
            event = message.value
            action = event.get('action')
            
            if action == 'login':
                self.handle_login(event)
            elif action == 'purchase':
                self.handle_purchase(event)
            else:
                logger.warning(f"Unknown action: {action}")
            
            self.processed_count += 1
            return True
            
        except Exception as e:
            logger.error(f"Error processing message: {e}")
            self.error_count += 1
            return False
    
    def handle_login(self, event):
        """Login event handler"""
        logger.info(f"User {event['user_id']} logged in from {event.get('ip_address')}")
        # Database'e yaz, cache güncelle, vs.
    
    def handle_purchase(self, event):
        """Purchase event handler"""
        logger.info(f"Purchase: ${event['amount']} by user {event['user_id']}")
        # Payment processing, inventory update, vs.
    
    def consume(self, max_messages=None):
        """
        Mesajları consume et
        
        Args:
            max_messages: Maksimum mesaj sayısı (None = sınırsız)
        """
        messages_consumed = 0
        
        try:
            while True:
                # Poll messages
                message_batch = self.consumer.poll(
                    timeout_ms=1000,
                    max_records=100
                )
                
                if not message_batch:
                    continue
                
                # Her partition için
                for topic_partition, messages in message_batch.items():
                    logger.info(
                        f"Received {len(messages)} messages from "
                        f"{topic_partition.topic}[{topic_partition.partition}]"
                    )
                    
                    # Mesajları işle
                    for message in messages:
                        success = self.process_message(message)
                        
                        if success:
                            # Başarılı ise offset'i commit et
                            self.consumer.commit()
                            messages_consumed += 1
                        else:
                            # Hatalı mesajı dead letter queue'ya gönder
                            self.send_to_dlq(message)
                            # Yine de commit et (sonsuz loop'tan kaçınmak için)
                            self.consumer.commit()
                
                # Maksimum mesaj sayısına ulaşıldı mı?
                if max_messages and messages_consumed >= max_messages:
                    break
        
        except KeyboardInterrupt:
            logger.info("Consumer interrupted by user")
        
        finally:
            self.close()
    
    def send_to_dlq(self, message):
        """Dead Letter Queue'ya mesaj gönder"""
        logger.warning(f"Sending message to DLQ: offset {message.offset}")
        # DLQ topic'ine gönder
        # producer.send('user-events-dlq', message.value)
    
    def seek_to_beginning(self):
        """Tüm partition'ları başa sar"""
        self.consumer.seek_to_beginning()
    
    def seek_to_offset(self, topic, partition, offset):
        """Belirli offset'e git"""
        tp = TopicPartition(topic, partition)
        self.consumer.seek(tp, offset)
    
    def get_current_offsets(self):
        """Mevcut offset'leri al"""
        return {
            tp: self.consumer.position(tp)
            for tp in self.consumer.assignment()
        }
    
    def close(self):
        """Consumer'ı kapat"""
        logger.info(f"Processed: {self.processed_count}, Errors: {self.error_count}")
        self.consumer.close()

# Kullanım
consumer = AdvancedConsumer(
    topics=['user-events', 'order-events'],
    group_id='analytics-group'
)

consumer.consume()
```

## Consumer Groups ve Partition Rebalancing

```python
from kafka import KafkaConsumer
import threading
import time

class ConsumerGroupExample:
    """
    Consumer Group örneği
    Aynı group_id'ye sahip consumer'lar partition'ları paylaşır
    """
    
    @staticmethod
    def create_consumer(consumer_id, group_id):
        """Consumer oluştur ve mesajları consume et"""
        consumer = KafkaConsumer(
            'user-events',
            bootstrap_servers=['localhost:9092'],
            group_id=group_id,
            value_deserializer=lambda m: json.loads(m.decode('utf-8')),
            auto_offset_reset='earliest',
        )
        
        print(f"Consumer {consumer_id} started")
        
        try:
            for message in consumer:
                print(
                    f"[Consumer {consumer_id}] "
                    f"Partition: {message.partition}, "
                    f"Offset: {message.offset}"
                )
                time.sleep(0.1)  # Simulated processing
        finally:
            consumer.close()
    
    @staticmethod
    def run_consumer_group():
        """
        3 consumer'lı bir group çalıştır
        
        Eğer topic 3 partition'a sahipse:
        - Her consumer 1 partition alır
        - Paralel işlem olur
        
        Eğer topic 6 partition'a sahipse:
        - Her consumer 2 partition alır
        """
        threads = []
        group_id = 'demo-group'
        
        for i in range(3):
            thread = threading.Thread(
                target=ConsumerGroupExample.create_consumer,
                args=(i, group_id)
            )
            thread.start()
            threads.append(thread)
        
        # Wait for all consumers
        for thread in threads:
            thread.join()

# Çalıştır
# ConsumerGroupExample.run_consumer_group()
```

![Kafka Event Streaming Pipeline](/assets/img/posts/kafka-event-streaming-pipeline.png)

## Gerçek Dünya Kullanım Senaryoları

### 1. Real-time Analytics Pipeline

```python
from kafka import KafkaConsumer, KafkaProducer
import json
from datetime import datetime
from collections import defaultdict
import time

class RealTimeAnalytics:
    """
    Gerçek zamanlı analitik pipeline
    User event'lerini consume eder, aggregate eder ve sonuçları yazar
    """
    
    def __init__(self):
        self.consumer = KafkaConsumer(
            'user-events',
            bootstrap_servers=['localhost:9092'],
            group_id='analytics-pipeline',
            value_deserializer=lambda m: json.loads(m.decode('utf-8')),
        )
        
        self.producer = KafkaProducer(
            bootstrap_servers=['localhost:9092'],
            value_serializer=lambda v: json.dumps(v).encode('utf-8'),
        )
        
        # Metrics storage (5 dakikalık window)
        self.window_size = 300  # 5 minutes
        self.metrics = defaultdict(lambda: {
            'login_count': 0,
            'purchase_count': 0,
            'total_revenue': 0.0,
            'unique_users': set(),
            'start_time': time.time()
        })
    
    def process_event(self, event):
        """Event'i işle ve metrics'i güncelle"""
        current_window = int(time.time() / self.window_size)
        window_metrics = self.metrics[current_window]
        
        action = event.get('action')
        user_id = event.get('user_id')
        
        # Unique user sayısı
        window_metrics['unique_users'].add(user_id)
        
        # Action'a göre metrics
        if action == 'login':
            window_metrics['login_count'] += 1
        elif action == 'purchase':
            window_metrics['purchase_count'] += 1
            window_metrics['total_revenue'] += event.get('amount', 0)
    
    def publish_metrics(self, window_id, metrics):
        """Aggregated metrics'i yayınla"""
        result = {
            'window_id': window_id,
            'timestamp': datetime.now().isoformat(),
            'login_count': metrics['login_count'],
            'purchase_count': metrics['purchase_count'],
            'total_revenue': metrics['total_revenue'],
            'unique_users': len(metrics['unique_users']),
            'duration': self.window_size
        }
        
        self.producer.send('analytics-results', value=result)
        print(f"Published metrics: {result}")
    
    def run(self):
        """Analytics pipeline'ı çalıştır"""
        last_window = None
        
        try:
            for message in self.consumer:
                event = message.value
                self.process_event(event)
                
                # Window değişti mi kontrol et
                current_window = int(time.time() / self.window_size)
                
                if last_window and current_window != last_window:
                    # Önceki window'u publish et
                    self.publish_metrics(last_window, self.metrics[last_window])
                    # Eski window'u sil (memory tasarrufu)
                    del self.metrics[last_window]
                
                last_window = current_window
        
        finally:
            self.consumer.close()
            self.producer.close()

# Çalıştır
# analytics = RealTimeAnalytics()
# analytics.run()
```

### 2. Event-Driven Microservices

```python
from kafka import KafkaConsumer, KafkaProducer
import json
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class OrderService:
    """
    Order microservice
    Order event'lerini consume eder ve diğer servislere event yayınlar
    """
    
    def __init__(self):
        self.consumer = KafkaConsumer(
            'order-created',
            bootstrap_servers=['localhost:9092'],
            group_id='order-service',
            value_deserializer=lambda m: json.loads(m.decode('utf-8')),
        )
        
        self.producer = KafkaProducer(
            bootstrap_servers=['localhost:9092'],
            value_serializer=lambda v: json.dumps(v).encode('utf-8'),
        )
    
    def process_order(self, order):
        """
        Sipariş işleme workflow'u
        
        1. Inventory check
        2. Payment processing
        3. Shipping arrangement
        4. Notification
        """
        order_id = order['order_id']
        
        try:
            # 1. Inventory Service'e event gönder
            inventory_event = {
                'order_id': order_id,
                'items': order['items'],
                'timestamp': datetime.now().isoformat()
            }
            self.producer.send('inventory-check-requested', value=inventory_event)
            logger.info(f"Inventory check requested for order {order_id}")
            
            # 2. Payment Service'e event gönder
            payment_event = {
                'order_id': order_id,
                'amount': order['total_amount'],
                'payment_method': order['payment_method']
            }
            self.producer.send('payment-requested', value=payment_event)
            logger.info(f"Payment requested for order {order_id}")
            
            # 3. Order status güncelle
            status_event = {
                'order_id': order_id,
                'status': 'processing',
                'timestamp': datetime.now().isoformat()
            }
            self.producer.send('order-status-updated', value=status_event)
            
        except Exception as e:
            logger.error(f"Error processing order {order_id}: {e}")
            # Failure event'i gönder
            self.producer.send('order-failed', value={
                'order_id': order_id,
                'error': str(e)
            })
    
    def run(self):
        """Service'i çalıştır"""
        logger.info("Order Service started")
        
        try:
            for message in self.consumer:
                order = message.value
                self.process_order(order)
        finally:
            self.consumer.close()
            self.producer.close()

class PaymentService:
    """Payment microservice"""
    
    def __init__(self):
        self.consumer = KafkaConsumer(
            'payment-requested',
            bootstrap_servers=['localhost:9092'],
            group_id='payment-service',
            value_deserializer=lambda m: json.loads(m.decode('utf-8')),
        )
        
        self.producer = KafkaProducer(
            bootstrap_servers=['localhost:9092'],
            value_serializer=lambda v: json.dumps(v).encode('utf-8'),
        )
    
    def process_payment(self, payment_request):
        """Ödeme işlemi"""
        order_id = payment_request['order_id']
        amount = payment_request['amount']
        
        try:
            # Payment gateway ile iletişim
            success = self.charge_payment(amount, payment_request['payment_method'])
            
            if success:
                # Success event
                self.producer.send('payment-completed', value={
                    'order_id': order_id,
                    'amount': amount,
                    'status': 'success'
                })
                logger.info(f"Payment completed for order {order_id}")
            else:
                # Failure event
                self.producer.send('payment-failed', value={
                    'order_id': order_id,
                    'reason': 'insufficient_funds'
                })
                logger.warning(f"Payment failed for order {order_id}")
        
        except Exception as e:
            logger.error(f"Payment error for order {order_id}: {e}")
            self.producer.send('payment-failed', value={
                'order_id': order_id,
                'reason': str(e)
            })
    
    def charge_payment(self, amount, method):
        """Simulated payment processing"""
        import random
        return random.random() > 0.1  # 90% success rate
    
    def run(self):
        """Service'i çalıştır"""
        logger.info("Payment Service started")
        
        try:
            for message in self.consumer:
                payment_request = message.value
                self.process_payment(payment_request)
        finally:
            self.consumer.close()
            self.producer.close()
```

### 3. Log Aggregation

```python
from kafka import KafkaConsumer
import json
import logging
from elasticsearch import Elasticsearch
from datetime import datetime

class LogAggregator:
    """
    Log aggregation service
    Farklı microservice'lerden gelen log'ları toplar ve Elasticsearch'e yazar
    """
    
    def __init__(self):
        self.consumer = KafkaConsumer(
            'application-logs',
            bootstrap_servers=['localhost:9092'],
            group_id='log-aggregator',
            value_deserializer=lambda m: json.loads(m.decode('utf-8')),
        )
        
        # Elasticsearch client
        self.es = Elasticsearch(['http://localhost:9200'])
        self.index_name = 'application-logs'
    
    def process_log(self, log_entry):
        """Log entry'sini Elasticsearch'e yaz"""
        try:
            # Log entry'yi zenginleştir
            enriched_log = {
                **log_entry,
                'ingestion_time': datetime.now().isoformat(),
                'timestamp': log_entry.get('timestamp', datetime.now().isoformat())
            }
            
            # Elasticsearch'e index et
            self.es.index(
                index=self.index_name,
                body=enriched_log
            )
            
            # Critical log'lar için alert
            if log_entry.get('level') == 'ERROR':
                self.send_alert(log_entry)
        
        except Exception as e:
            logging.error(f"Error processing log: {e}")
    
    def send_alert(self, log_entry):
        """Critical log için alert gönder"""
        # Slack, email, PagerDuty, vs.
        print(f"ALERT: {log_entry['service']} - {log_entry['message']}")
    
    def run(self):
        """Aggregator'ı çalıştır"""
        print("Log Aggregator started")
        
        try:
            for message in self.consumer:
                log_entry = message.value
                self.process_log(log_entry)
        finally:
            self.consumer.close()

# Log producer (her microservice'ten)
def send_log(service_name, level, message):
    """Log mesajı gönder"""
    producer = KafkaProducer(
        bootstrap_servers=['localhost:9092'],
        value_serializer=lambda v: json.dumps(v).encode('utf-8'),
    )
    
    log_entry = {
        'service': service_name,
        'level': level,
        'message': message,
        'timestamp': datetime.now().isoformat(),
        'host': 'server-1'
    }
    
    producer.send('application-logs', value=log_entry)
    producer.close()
```

## Kafka Monitoring ve Best Practices

### Monitoring Metrics

```python
from kafka import KafkaAdminClient
from kafka.admin import NewTopic

class KafkaMonitor:
    """Kafka cluster monitoring"""
    
    def __init__(self):
        self.admin_client = KafkaAdminClient(
            bootstrap_servers=['localhost:9092']
        )
    
    def get_cluster_info(self):
        """Cluster bilgilerini al"""
        cluster = self.admin_client._client.cluster
        
        print("Kafka Cluster Info:")
        print(f"Brokers: {len(cluster.brokers())}")
        for broker in cluster.brokers():
            print(f"  - Broker {broker.nodeId}: {broker.host}:{broker.port}")
        
        print(f"\nTopics: {len(cluster.topics())}")
        for topic in cluster.topics():
            partitions = cluster.partitions_for_topic(topic)
            print(f"  - {topic}: {len(partitions)} partitions")
    
    def create_topic(self, name, partitions=3, replication_factor=1):
        """Yeni topic oluştur"""
        topic = NewTopic(
            name=name,
            num_partitions=partitions,
            replication_factor=replication_factor
        )
        
        try:
            self.admin_client.create_topics([topic])
            print(f"Topic '{name}' created successfully")
        except Exception as e:
            print(f"Error creating topic: {e}")
    
    def delete_topic(self, name):
        """Topic sil"""
        try:
            self.admin_client.delete_topics([name])
            print(f"Topic '{name}' deleted successfully")
        except Exception as e:
            print(f"Error deleting topic: {e}")

# Kullanım
monitor = KafkaMonitor()
monitor.get_cluster_info()
```

### Best Practices

```python
"""
Kafka Best Practices
"""

# 1. Message Key Seçimi
# ✅ İyi: Partition balance için iyi key
producer.send('user-events', key=str(user_id), value=event)

# ❌ Kötü: Aynı key her zaman (partition imbalance)
producer.send('user-events', key='fixed-key', value=event)

# 2. Batch Processing
# ✅ İyi: Batch'lerle işle
messages = consumer.poll(timeout_ms=1000, max_records=100)
for topic_partition, records in messages.items():
    process_batch(records)  # Toplu işlem
    consumer.commit()

# ❌ Kötü: Her mesajı ayrı commit
for message in consumer:
    process(message)
    consumer.commit()  # Her mesajda commit (yavaş)

# 3. Error Handling
# ✅ İyi: Retry ve DLQ
try:
    process_message(message)
    consumer.commit()
except Exception as e:
    if is_retriable(e):
        retry_later(message)
    else:
        send_to_dlq(message)
    consumer.commit()  # İlerle

# 4. Idempotent Processing
# ✅ İyi: Idempotent tasarım
def process_order(order_id):
    if already_processed(order_id):
        return  # Skip duplicate
    
    # Process...
    mark_as_processed(order_id)

# 5. Monitoring
# Kritik metrics:
# - Consumer lag (topic offset - consumer offset)
# - Throughput (messages/sec)
# - Error rate
# - Partition distribution
```

## Sonuç

Apache Kafka, modern veri mühendisliği ve event-driven architecture'ların temel taşıdır. Bu yazıda ele aldığımız konular:

- **Kafka temel kavramları**: Topic, partition, offset, consumer group
- **Python implementation**: kafka-python ile producer/consumer
- **Advanced patterns**: Batch processing, error handling, DLQ
- **Real-world use cases**: Analytics pipeline, microservices, log aggregation
- **Best practices**: Performance, reliability, monitoring

**Kafka Kullanım Senaryoları:**
- Real-time analytics ve metrics
- Event-driven microservices
- Log aggregation ve monitoring
- Stream processing (Kafka Streams, Flink)
- CDC (Change Data Capture)
- Message queue ve pub/sub

**Dikkat Edilmesi Gerekenler:**
- Consumer lag'i düzenli monitor edin
- Partition sayısını throughput'a göre ayarlayın
- Replication factor ile fault-tolerance sağlayın
- Idempotent processing tasarlayın
- Dead letter queue stratejisi kullanın

Kafka, yüksek throughput, düşük latency ve güvenilirlik gerektiren tüm uygulamalarda tercih edilen bir çözümdür.
