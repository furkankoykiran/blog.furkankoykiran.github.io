---
layout: post
title: "Production-Grade Elasticsearch: 'Yellow Status'tan Kurtuluş Reçetesi"
date: 2025-08-05 09:00:00 +0300
categories: [Backend, System Design]
description: "Elasticsearch'te 'Dynamic Mapping' tuzağı, Split-Brain riski, Zero-Downtime Reindexing stratejileri ve performans optimizasyonu üzerine bir kıdemli mühendis manifestosu."
image: assets/img/posts/elasticsearch-components-diagram.jpg
---

MongoDB veya PostgreSQL yanında "arama motoru" olarak konumlandırdığımız Elasticsearch, aslında bakımı en nazlı distributed sistemlerden biridir. İlk kurulumda şahane çalışır, veri 100GB'ı aştığında veya node sayısı 3'ü geçtiğinde ise gerçek yüzünü (ve o korkunç `Red Status`'ı) gösterir.

Bu yazıda, Elasticsearch'ü bir "oyuncak" değil, "kritik altyapı" olarak yönetmenin inceliklerine bakacağız.

## 1. İlk Günah: Dynamic Mapping

Elasticsearch varsayılan olarak "şemesız" (schemaless) gibi davranır. Bir JSON atarsınız, o da "Hımm, bu bir sayıya benziyor" der ve field tipini `long` yapar. Buna **Dynamic Mapping** denir ve production için **tehlikelidir**.

**Senaryo:**
1.  İlk logonuz geldi: `{"status_code": 200}` -> ES bunu `long` yaptı.
2.  İkinci logunuz geldi: `{"status_code": "404 Not Found"}` -> ES "Hata! Ben bunu long bekliyordum, sen string yolladın" der ve logu reddeder (Mapper Parsing Exception).

**Best Practice:**
Production indekslerinizde `dynamic: strict` kullanın. Veri modelinizi (mapping) siz yönetin, ES'nin tahmin etmesine izin vermeyin.

```json
PUT /my-index
{
  "mappings": {
    "dynamic": "strict",
    "properties": {
      "user_id": { "type": "keyword" }, // keyword: exact match için
      "bio": { "type": "text" },        // text: full-text search için
      "created_at": { "type": "date" }
    }
  }
}
```

## 2. Cluster Sağlığı: Split-Brain Kâbusu

Cluster'ınızda ağ kopması (network partition) oldu ve 5 node'un 2'si diğer 3'ünden koptu. O 2 node, "Galiba master öldü, yeni master biziz" derse ne olur?

Aynı anda iki farklı master (Master-Slave değil, Master-Master!). Veri tutarsızlığı başlar, verileriniz bozulur. Buna **Split-Brain** denir.

**Çözüm:**
*   **Minimum Master Nodes:** (ES 7 öncesi) `discovery.zen.minimum_master_nodes` değerini `(master_eligible_nodes / 2) + 1` olarak ayarlayın. 3 master varsa değer 2 olmalı.
*   **Adil Sayı:** Master-eligible node sayınız her zaman TEK sayı (3, 5, 7) olsun. Çift sayıda oybirliği (quorum) sağlanamaz.

## 3. Zero-Downtime Reindexing: Alias Pattern

Veri tabanında bir tablonun şemasını değiştirmek (ALTER TABLE) zordur. Elasticsearch'te mapping değiştirmek (bir field'ı `text`'ten `keyword`e çevirmek) **imkansızdır**. İndeksi silip baştan oluşturmanız gerekir.

Peki production'da, canlı trafik akarken bunu nasıl yapacaksınız? Cevap: **Aliasing**.

Uyguamanız asla fiziksel indeks ismine (`logs-2025-08`) yazmamalı, her zaman bir takma isme (`logs-write-alias`) yazmalıdır.

**Flow:**
1.  Yeni indeks oluştur (`logs-v2`).
2.  Eski veriyi reindex et (`POST _reindex`).
3.  Alias'ı atomik bir işlemle eski indeksten alıp yeniye tak.

```json
POST /_aliases
{
  "actions": [
    { "remove": { "index": "logs-v1", "alias": "logs-current" } },
    { "add":    { "index": "logs-v2", "alias": "logs-current" } }
  ]
}
```
Bu işlem milisaniyeler sürer ve uygulama (kullanıcı) hiçbir kesinti hissetmez.

## 4. Sharding Matematiği

"Kaç shard açmalıyım?" sorusunun tek bir cevabı yoktur ama altın kurallar vardır:

1.  **Shard Boyutu:** Bir shard'ın boyutu 10GB ile 50GB arasında olmalıdır.
    *   Çok küçük shard (100MB) -> "Too many open files" hatası, yüksek heap kullanımı.
    *   Çok büyük shard (1TB) -> Rebalance süresi saatler sürer, recovery imkansızlaşır.
2.  **Replica Sayısı:** En az 1 olmalı. Tek node'unuz varsa replica 0 olmalı (aksi halde cluster Yellow kalır).

![Shard Allocation Diagram](assets/img/posts/elasticsearch-shards.jpg)

## 5. Performans Tuzağı: Deep Pagination

Kullanıcı "Sayfa 10,000"e gitmek istedi.
`from: 100000, size: 10`.

Elasticsearch bunu yapmak için her shard'dan ilk 100,010 kaydı çeker, RAM'de sıralar ve en üstteki 10 taneyi verir. Bu işlem CPU ve RAM'i öldürür.

**Çözüm:**
*   10,000'den sonrası için `search_after` API'sini kullanın. Bu, "cursor" mantığıyla çalışır ve bir önceki sayfanın son elemanından sonrasını getirir. Stateless'tır ve çok hızlıdır.

## 6. Sahadan Notlar: Troubleshooting

**Durum: Cluster Yellow**
*   **Anlamı:** Tüm primary shard'lar atandı ama bazı replica shard'lar atanamadı.
*   **Neden:** Genellikle disk doludur (High Watermark) veya tek node'lu clusterda replica=1 ayarlanmıştır (Replica'nın duracağı başka node yok).
*   **Debug:** `GET _cluster/allocation/explain` komutu size nedenini İngilizce olarak söyler.

**Durum: Circuit Breaker Exception**
*   **Anlamı:** Bir sorgu o kadar büyük heap alanı istedi ki, ES "OOM olmaktansa bu sorguyu reddederim" dedi.
*   **Neden:** Genellikle `fielddata=true` yapılmış text alanlarında aggregation yapmaya çalışmak veya çok büyük bucket size kullanımı.

## 7. Production Checklist

*   [ ] **Swap Kapalı mı?** `bootstrap.memory_lock: true` ayarlı mı? Swap, Java GC performansını öldürür.
*   [ ] **Heap Size:** RAM'in yarısı (ama max 31GB) olarak ayarlandı mı? (Compressed Oops sınırı).
*   [ ] **Wildcard Sorguları:** Başında `*` olan sorgulara (`*kiran`) izin veriyor musunuz? (Vermeyin, tüm indexi tarar).
*   [ ] **Snapshot/Restore:** S3 veya MinIO'ya düzenli snapshot alınıyor mu? Cluster çökerse tek kurtarıcınız budur. (Bkz: `[MinIO ile Object Storage Sistemi](/devops/minio-ile-object-storage-sistemi)`)


## 8. Arka Plandaki Sihir: Inverted Index ve Analyzers

Elasticsearch'ü klasik veritabanlarından ayıran ana özellik "Inverted Index" yapısıdır. Bir kitap indeksi gibi çalışır.

Siz "Kitap" kelimesini arattığınızda, ES tüm satırları (scan) gezmez. Doğrudan "Kitap" kelimesinin geçtiği döküman ID'lerini indeks tablosundan bulur. Ancak bunun çalışması için **Analyzer** süreci kritiktir.

**Senaryo:** `Standard Analyzer` kullanıyorsunuz.
Girdi: "İstanbul'da hava çok güzel!"
Token'lar: `[istanbul'da, hava, çok, güzel]`

Eğer kullanıcı "istanbul" (küçük harf ve kesme işareti olmadan) ararsa bulamayabilir. Bu yüzden doğru **Tokenizer** ve **Filter** (Lowercase, ASCII Folding) seçimi hayatidir.

```json
// Custom Analyzer Örneği (Türkçe Karakter Dostu)
"analysis": {
  "analyzer": {
    "my_turkish_analyzer": {
      "tokenizer": "standard",
      "filter": ["lowercase", "asciifolding"]
    }
  }
}
```

## 9. Node Rolleri: Her Node Eşit Değildir

Küçük clusterlarda her node "her işi" yapar. Ancak trafik arttıkça rolleri ayırmalısınız (Separation of Concerns):

1.  **Master-Eligible Nodes:** Sadece cluster yönetiminden (shard atama, node ekleme/çıkarma) sorumludur. CPU/RAM az harcar, disk I/O yapmaz. Stabil olmalıdır.
2.  **Data Nodes:** Veriyi tutar, CRUD yapar, sorguları çalıştırır. Yüksek CPU, RAM ve hızlı SSD ister. "Amele" node'lardır.
3.  **Coordinating (Client) Nodes:** Gelen isteği karşılar, ilgili data node'lara dağıtır (scatter) ve sonuçları toplayıp (gather) birleştirir. Yüksek RAM ister.
4.  **Ingest Nodes:** Logstash gibi çalışır. Veri diske yazılmadan önce "Pipeline" ile işlenir (Grok parse, GeoIP ekleme).

**Pro Tip:** Production cluster'ında en azından Master ve Data node'ları ayırın. Ağır bir sorgu Data node'u kilitlerse, Master node etkilenmesin ve cluster yönetimi devam etsin.

## 10. Güvenlik: X-Pack ve Ötesi

Eskiden ES, "Sadece iç ağda çalışsın, şifreye gerek yok" mantığındaydı. Ransomware saldırıları sonrası bu değişti. Port 9200'ü asla şifresiz (Basic Auth) ve SSL'siz internete açmayın.

*   **RBAC (Role Based Access Control):** Stajyerin sadece `read-only` yetkisi olmalı.
*   **Audit Logging:** Kim, ne zaman, hangi sorguyu attı? (Compliance için şart).
*   **TLS/SSL:** Node'lar arası iletişim (Port 9300) mutlaka şifreli olmalı. Aksi halde bir node arasına giren saldırgan (Man-in-the-Middle) tüm veriyi çalar.


## 11. Karşılaştırma: SQL vs Elasticsearch

"Neden veritabanı yerine Elasticsearch?" sorusuna verilecek teknik cevaplar:

| Kavram | RDBMS (PostgreSQL/MySQL) | Elasticsearch |
| :--- | :--- | :--- |
| **Veri Yapısı** | Tablo (Structured) | Index (Semi-structured JSON) |
| **Transaction** | ACID (Atomic, Consistent...) | Eventual Consistency (BASE) |
| **Hız** | Join'lerde yavaşlayabilir | Denormalize veride çok hızlı |
| **Arama** | `LIKE %term%` (Yavaş, Index Scan) | Inverted Index (Çok Hızlı) |
| **Kullanım** | Ana veri kaynağı (Source of Truth) | Arama ve Analitik motoru |

**Altın Kural:** Asla "Source of Truth" olarak Elasticsearch kullanmayın. Verinin aslı PostgreSQL'de dursun, Elasticsearch'e asenkron olarak (CDC veya Message Queue ile) sync edin. (Bkz: `[RabbitMQ Mesaj Kuyruğu](/backend/rabbitmq-mesaj-kuyrugu-sistemi)`)

## Özetle

Elasticsearch, doğru ellerde inanılmaz güçlü bir analiz motorudur. Yanlış ellerde ise sürekli bakım isteyen bir baş belasıdır. "Schema-less" yalanına kanmayın, mapping'lerinizi sıkı tutun ve alias kullanmadan production'a çıkmayın.

Ve unutmayın: Veri boyutu değil, veri modelleme şekliniz performansı belirler.


