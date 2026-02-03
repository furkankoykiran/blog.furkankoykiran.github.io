---
layout: post
title: "RabbitMQ ile Production-Ready Mesajlaşma Mimarisi: 'Hello World'ün Ötesi"
date: 2025-08-18 10:30:00 +0300
categories: [Backend, System Design]
description: "RabbitMQ'da 'Prefetch Count' optimizasyonu, Dead Letter Exchange (DLX) kurgusu ve veri kaybını önleyen reliability pattern'leri üzerine kıdemli mühendis tecrübeleri."
image: assets/img/posts/rabbitmq-architecture-diagram.png
---

Birçok yazılımcı için RabbitMQ, "mesajı at, öbür taraf alsın" kadar basit görünür. Ancak production ortamında binlerce mesaj akarken RabbitMQ'nun RAM'i şiştiğinde, consumer'lar kilitlendiğinde veya kritik bir ödeme mesajı "kaybolduğunda", işin rengi değişir.

Bugün, RabbitMQ'yu sadece bir mesaj kuyruğu olarak değil, sisteminizin omurgası olarak nasıl kurgulamanız gerektiğini konuşacağız.

## 1. Sessiz Performans Katili: Prefetch Count

RabbitMQ varsayılan ayarlarında, kuyruktaki mesajları consumer'lara "olabildiğince hızlı" itmeye çalışır (Push Model). Eğer `prefetch_count` ayarını yapmazsanız, RabbitMQ binlerce mesajı tek bir consumer'ın RAM'ine yığabilir.

Sonuç? Consumer OOM (Out of Memory) olur, diğer consumer'lar boş boş beklerken biri boğulur.

**Doğru Strateji:**
*   **Adil Dağıtım (Fair Dispatch):** `prefetch_count=1`. Bu, "Ben elimdeki işi bitirip ACK (onay) yollamadan bana yeni iş verme" demektir. CPU-intensive işler için idealdir.
*   **Batch İşlemler:** Eğer mesajlarınız çok küçükse ve çok hızlı işleniyorsa (örneğin log parsing), `prefetch_count=50` gibi değerler network trafiğini (round-trip) azaltır ve throughput'u artırır.
*   **Global Flag:** Python (Pika) veya Go kütüphanelerinde `prefetch_count` verirken `global=True` derseniz, bu limit o channel'daki *tüm* consumer'lar için toplam limit olur. `global=False` derseniz (önerilen), her consumer için ayrı limit olur.

## 2. Reliability Trinity: Veri Kaybına Son

"Mesaj kayboldu" cümlesi, bir backend mühendisinin kabusudur. RabbitMQ'da %100 reliability (güvenilirlik) için şu 3 sacayağını kurmalısınız:

1.  **Queue Durability:** Kuyruğu oluştururken `durable=True` yapın. RabbitMQ restart olsa bile kuyruk tanımı silinmez.
2.  **Message Persistence:** Mesajı yayınlarken `delivery_mode=2` (Persistent) olarak işaretleyin. Bu, mesajın diske yazılmasını sağlar. (Performansı biraz düşürür ama veriyi korur).
3.  **Publisher Confirms:** En çok atlanan madde budur. Mesajı `channel.basic_publish()` ile attınız, peki RabbitMQ bunu aldı mı? Network koptuysa ne olacak? `confirm_delivery` modunu açarak, RabbitMQ'dan "aldım" onayı (Ack) beklemeden işlemi başarılı saymayın.

**Anti-Pattern:** Asla `auto_ack=True` kullanmayın! Bu, "Mesaj consumer'a ulaştığı an silinsin" demektir. İşlenirken hata alırsanız mesaj buhar olur. Her zaman iş bittikten sonra manuel `ch.basic_ack()` gönderin.

## 3. Emniyet Sübabı: Dead Letter Exchange (DLX)

Kodunuzda bug var ve bir mesaj consumer'ı sürekli crash ettiriyor. RabbitMQ mesajı tekrar kuyruğa koyuyor, consumer tekrar alıyor ve tekrar çöküyor. Bu "Poison Message" döngüsü tüm sistemi kilitler.

Çözüm: **Dead Letter Exchange.**

1.  Bir `dlx_exchange` ve buna bağlı `dlx_queue` oluşturun.
2.  Ana kuyruğunuzu tanımlarken `x-dead-letter-exchange: "dlx_exchange"` argümanını verin.
3.  Consumer, mesajı işleyemediğinde `basic_nack(requeue=False)` desin.

Bu durumda RabbitMQ, mesajı silmek yerine DLX'e yönlendirir. Siz de sabah kahvenizi içerken DLX kuyruğuna bakıp, "Bu mesaj neden hata vermiş?" diye inceleyebilir (Root Cause Analysis) ve bug'ı düzelttikten sonra tekrar işleme alabilirsiniz.

## 4. Exchange Tipleri: Doğru Aracı Seçmek

Her şeye `Direct Exchange` kullanıp geçmeyin. Mimarinizi esnetin:

*   **Direct:** Routing key tam eşleşmeli. (Log seviyesi: `error`, `info`)
*   **Fanout:** Routing key önemsizdir. Mesaj, o exchange'e bağlı *tüm* kuyruklara gider. "Yeni kullanıcı kaydoldu" eventi için harikadır; hem "Hoşgeldin E-postası" servisi, hem "Analytics" servisi, hem de "Cache Warmer" servisi aynı mesajı alır.
*   **Topic:** En esnek olanıdır. `europe.stock.usd` veya `asia.weather.rain` gibi pattern'ler (`#` ve `*` wildcard) kullanır. Örneğin `*.stock.#` diyerek tüm borsa verilerini dinleyebilirsiniz.
*   **Headers:** Routing key yerine header attributelarına bakar. Çok nadir kullanılır ama kompleks routing kuralları için (örneğin `format=pdf` ve `priority=high` olanlar) hayat kurtarır.

![RabbitMQ Exchange Types](assets/img/posts/rabbitmq-exchanges.png)

## 5. Sahadan Notlar: Troubleshooting

RabbitMQ ile savaşırken edindiğim bazı tecrübeler:

**Senaryo 1: Kuyruk Doluyor Ama Tüketim Yok**
*   **Kontrol:** Consumer'lar "Stuck" (takılı) kalmış olabilir mi? Eğer `prefetch_count` yüksekse ve işleminiz thread-safe değilse (örneğin Python'da global connection kullanımı), heartbeat thread'i durabilir ve RabbitMQ bağlantıyı kesebilir.
*   **Çözüm:** Consumer kodunuzda exception handling olduğundan ve `finally` bloğunda mutlaka ACK/NACK gönderildiğinden emin olun.

**Senaryo 2: RAM Kullanımı Tavan Yaptı**
*   **Kontrol:** "Unacknowledged" mesaj sayısı yüksek mi?
*   **Sebep:** Consumer'lar mesajları alıyor ama ACK göndermiyor. RabbitMQ bu mesajları RAM'de tutmak zorundadır çünkü her an "geri gelmesi" gerekebilir.

**Senaryo 3: Cluster Split-Brain**
*   RabbitMQ cluster kuracaksanız, Network Partition Handling stratejinizi (`pause_minority` vs `autoheal`) mutlaka belirleyin. Yanlış konfigürasyonda cluster ikiye bölünür ve veri tutarsızlığı oluşur. (Bkz: `[Kubernetes Container Orkestrasyon](/devops/kubernetes-container-orkestrasyon)`)

## 6. Monitoring: Görmediğini Yönetemezsin

Prometheus ve Grafana ikilisi burada da en iyi dostunuz. RabbitMQ Management Plugin güzeldir ama geçmişe dönük veri tutmaz.
(Bkz: `[Prometheus ve Grafana ile İzleme](/devops/prometheus-grafana-ile-izleme)`)

Takip etmeniz gereken kritik metrikler:
*   `rabbitmq_queue_messages_ready`: Bekleyen iş sayısı. (Queue Depth).
*   `rabbitmq_queue_messages_unacked`: İşlemdeki iş sayısı.
*   `rabbitmq_channel_consumer_count`: Consumer sayısı beklediğinizden azsa, bir servis çökmüş olabilir.

## 7. Production Checklist

Canlıya almadan önce bu listeyi tikleyin:

*   [ ] **User Yönetimi:** Default `guest/guest` kullanıcısını sildiniz mi?
*   [ ] **Vhost İzolasyonu:** Farklı projeleri aynı RabbitMQ'da tutuyorsanız, Virtual Host (vhost) ile izole ettiniz mi?
*   [ ] **File Descriptor Limiti:** İşletim seviyesinde (uygulama sunucusunda) `ulimit -n` değerini artırdınız mı? RabbitMQ soket açmayı sever.
*   [ ] **Queue Limitleri:** `x-max-length` veya `x-message-ttl` koydunu mu? Diski dolduracak bir consumer hatasına karşı sigortanız olsun.
*   [ ] **Connection Pooling:** Uygulamanız her mesaj için yeni connection mı açıyor? (Yapmayın!) Connection'ı açık tutup (Long-lived), channel oluşturun.


## 8. Quorum Queues: Yeni Nesil HA

Eskiden "Mirrored Queues" kullanırdık ama artık **Quorum Queues** standardı geldi. Raft konsensüs algoritmasını kullanır ve veri bütünlüğü konusunda çok daha hassastır.

Eğer finansal bir işlem yapıyorsanız veya veri kaybı lüksünüz sıfırsa, "Classic Queue" yerine mutlaka "Quorum Queue" kullanın. Diski biraz daha fazla kullanır ama içiniz rahat eder.

```yaml
# Quorum Queue Tanımlama
x-queue-type: quorum
```

## 9. Mimari Desenler: Hangi Çekiç Nereye?

RabbitMQ sadece bir boru değildir. Farklı desenlerle farklı problemleri çözersiniz:

1.  **Work Queues (Göre Dağıtımı):**
    *   *Senaryo:* PDF oluşturma veya Resim resize işlemleri.
    *   *Yapı:* Tek bir kuyruğu dinleyen 5 worker. Herbiri sırayla iş alır (`Generic Task`).
2.  **Publish/Subscribe (Fanout):**
    *   *Senaryo:* Bir kullanıcı kayıt oldu.
    *   *Yapı:* Exchange'e mesaj atılır, o exchange hem "Email Service" kuyruğuna hem "SMS Service" kuyruğuna kopyalar. Servisler birbirini bilmez (Decoupling).
3.  **RPC (Remote Procedure Call):**
    *   *Senaryo:* Frontend, Backend'den bir hesaplama yapıp *cevabını* bekliyor.
    *   *Yapı:* Request mesajının içine `reply_to` (cevap kuyruğu) ve `correlation_id` (işlem ID) eklenir. Worker işi yapar ve cevabı o özel kuyruğa geri yazar. HTTP gibi davranır ama asenkrondur.

## 10. Kütüphane Seçimi: Python Tarafı

Python dünyasında standart `pika` kütüphanesidir. Ancak `pika` senkron çalışır ve blocking I/O yapar. Eğer FastAPI veya AsyncIO tabanlı modern bir altyapınız varsa, **aio-pika** veya **arq** (Redis tabanlı ama alternatif) kullanmanızı öneririm. (Bkz: `[Python AsyncIO ve Paralel İşlem](/backend/python-asyncio-paralel-islem)`)

Senkron `pika` kullanıyorsanız, connection kopmalarını yönetmek ("Reconnection Strategy") sizin sorumluluğunuzdadır ve inanın bana, production network'ü her zaman kopar.

**Anti-Pattern: Huge Payloads**
RabbitMQ bir veritabanı değildir. Mesajın içine 10MB'lık JSON veya Base64 resim koymayın. Mesajın içine sadece `image_path` veya `s3_url` koyun. RabbitMQ küçük mesajlarla (1KB - 10KB) şov yapar, büyük mesajlarla sürünür.


## 11. Karşılaştırma: RabbitMQ vs Kafka

Mülakatların vazgeçilmez sorusudur: "Neden Kafka yerine RabbitMQ seçtin?" veya tam tersi. İşte cevabınız:

| Özellik | RabbitMQ | Apache Kafka |
| :--- | :--- | :--- |
| **Model** | Push (Consumer'a iter) | Pull (Consumer çeker) |
| **Routing** | Çok Güçlü (Topic, Header, Fanout) | Zayıf (Partition bazlı) |
| **Mesaj Saklama** | RAM ağırlıklı, işlenince silinir | Disk ağırlıklı, günlerce saklanır |
| **Throughput** | 40k - 100k msg/sec | 1M+ msg/sec |
| **Kullanım Alanı** | Kompleks routing, Transactional işler | Big Data, Log Aggregation, Event Streaming |

Eğer "Log topluyorum" diyorsanız Kafka; "Sipariş işliyorum, fatura kesiyorum" diyorsanız RabbitMQ (Smart Broker, Dumb Consumer) daha doğru tercihtir. (Bkz: `[Apache Kafka Event Streaming](/devops/apache-kafka-event-streaming)`)

## 12. Kod Örneği: Robust Consumer (Python)

Production'da kapanan bağlantıyı (connection lost) otomatik yöneten basit bir yapı:

```python
import pika
import time

def connect():
    while True:
        try:
            connection = pika.BlockingConnection(
                pika.ConnectionParameters('localhost', heartbeat=600)
            )
            channel = connection.channel()
            # Prefetch Count: 1 (Adil Dağıtım)
            channel.basic_qos(prefetch_count=1) 
            return channel
        except pika.exceptions.AMQPConnectionError:
            print("Bağlantı koptu, 5sn sonra tekrar deneniyor...")
            time.sleep(5)

def callback(ch, method, properties, body):
    try:
        print(f"İşleniyor: {body}")
        # İş mantığı burada...
        ch.basic_ack(delivery_tag=method.delivery_tag)
    except Exception as e:
        print(f"Hata: {e}")
        # Hata durumunda NACK ve DLX'e yönlendirme
        ch.basic_nack(delivery_tag=method.delivery_tag, requeue=False)

# Main Loop
channel = connect()
channel.basic_consume(queue='task_queue', on_message_callback=callback)

try:
    channel.start_consuming()
except KeyboardInterrupt:
    channel.stop_consuming()
```

## Özetle

RabbitMQ, "kur ve unut" türü bir yazılım değildir. Doğru konfigüre edildiğinde, sisteminizin en güvenilir parçası olur. Yanlış konfigüre edildiğinde ise, sistemin darboğazı (bottleneck) haline gelir.

Dead Letter Exchange kullanın, manuel ACK'dan korkmayın ve consumer'larınızı her zaman "fail" edebilecek şekilde tasarlayın. Unutmayın, dağıtık sistemlerde hata bir istisna değil, bir kuraldır.

---
**Meraklısına Not:** Sisteminizde binlerce mikroservis varsa ve Prometheus yetersiz kalıyorsa, "Federation" yapısını veya batch joblar için "Pushgateway" konusunu (Bkz: `[Celery ve ARQ ile Asenkron Görev Kuyruğu](/backend/celery-arq-asenkron-gorev-kuyrugu)`) araştırmanızı öneririm.
