---
title: "Python ve WebSocket ile Gerçek Zamanlı Trading Sistemleri"
description: "Python asyncio ve WebSocket protokolü kullanarak milisaniye hassasiyetinde trading botu geliştirme rehberi. Latency optimizasyonu ve veri işleme stratejileri."
date: "2024-08-02"
categories: [FinTech, Python]
tags: [python, websocket, trading-bot, asyncio, real-time, algorithmic-trading]
image:
  path: "/assets/img/posts/websocket-real-time-architecture-diagram.png"
  alt: "WebSocket Gerçek Zamanlı Veri Mimarisi"
---

## Giriş: Polling'den Push'a Geçiş - Neden WebSocket?

Finansal piyasalarda "vakit nakittir" sözü, milisaniyeler bazında fiziksel bir gerçekliğe dönüşür. Geleneksel REST API mimarilerinde veriyi almak için sürekli sunucuya sormanız (polling) gerekir. Bu sadece sunucuyu yormakla kalmaz, aynı zamanda verinin oluştuğu an ile sizin ona eriştiğiniz an arasında devasa bir gecikme (latency) yaratır. Profesyonel bir trading botu için 200ms'lik bir gecikme, karlı bir işlemin zarara dönüşmesi için yeterli bir süredir.

WebSocket protokolü, bu darboğazı çift yönlü ve sürekli açık bir bağlantı (full-duplex) sağlayarak çözer. Sunucu, veri değiştiği anda size "push" yapar. Eğer bu verileri kullanarak kullanıcıları bilgilendiren bir arayüz arıyorsanız [Python ile Telegram Trading Bot](/blog-development/python/2024/07/22/python-telegram-trading-bot/) rehberimizdeki asenkron bildirim yapılarını mutlaka incelemelisiniz. Bir senior mühendis gözüyle bakıldığında, WebSocket kullanımı sadece bir hız tercihi değil, sistemin ölçeklenebilirliği ve kaynak verimliliği için bir zorunluluktur.

![Python AsyncIO Mimari](/assets/img/posts/python-asyncio-concurrent-programming.png)

## Python AsyncIO: Yüksek Performanslı Eşzamanlılık

WebSocket dünyasında tek bir bağlantıyla yetinmeyiz; aynı anda onlarca borsayı ve yüzlerce pariteyi izlememiz gerekebilir. Python'ın `asyncio` kütüphanesi, tek bir CPU çekirdeği üzerinde binlerce bağlantıyı "non-blocking" (bloklamayan) şekilde yönetmemizi sağlar.

### Event Loop Mantığı
AsyncIO'nun kalbindeki Event Loop, bir işlem (örneğin ağdan veri beklemek) sırasında CPU'nun boş durmamasını sağlar. Veri beklerken, loop diğer görevlere geçer. Bu, özellikle I/O yoğunluklu trading botları için biçilmiş kaftandır.

```python
import asyncio
import websockets
import json

async def market_stream(symbol):
    uri = f"wss://stream.binance.com:9443/ws/{symbol.lower()}@ticker"
    async with websockets.connect(uri) as websocket:
        while True:
            # Burası asenkron olarak bekler, CPU'yu bloklamaz
            message = await websocket.recv()
            data = json.loads(message)
            print(f"{symbol} Fiyat: {data['c']}")

async def main():
    # Birden fazla pariteyi aynı anda izle
    symbols = ["btcusdt", "ethusdt", "solusdt"]
    tasks = [market_stream(s) for s in symbols]
    await asyncio.gather(*tasks)

if __name__ == "__main__":
    asyncio.run(main())
```

**Senior Notu:** `asyncio.gather` kullanırken bir task'in hata alması durumunda tüm sistemin çökmemesi için `return_exceptions=True` parametresini değerlendirmeli veya her task'ı kendi içinde `try-except` bloklarıyla sarmalamalısınız.

![Order Book Derinlik Analizi](/assets/img/posts/order-book-depth-chart-visualization.png)

## Gerçek Zamanlı Order Book Yönetimi

Sadece ticker (fiyat) verisiyle trading yapmak, karanlıkta araba sürmeye benzer. Piyasayı gerçekten anlamak için "Order Book" (Emir Defteri) verisine ihtiyacınız vardır. WebSocket üzerinden gelen binlerce emir güncellemesini (diff-stream) işleyip lokalde bir "Order Book Snapshot" tutmak, HFT (High-Frequency Trading) sistemlerinin temelidir.

### Veri İşleme Stratejisi
Borsalar genellikle başlangıçta devasa bir snapshot gönderir ve sonrasında sadece değişen fiyat seviyelerini iletir. Bu veriyi işlerken:
1. **İmbalansı Ölçün:** Alıcılar mı yoksa satıcılar mı daha baskın? 
2. **Spread'i İzleyin:** Alış ve satış arasındaki farkın açılması, likiditenin çekildiğinin habercisi olabilir.
3. **Milisaniyelik Güncellemeler:** Binance gibi borsalar 100ms seviyesinde güncelleme sunar. Bu veriyi işleyen fonksiyonun kendisi de milisaniyeler içinde bitmelidir.

```python
from collections import OrderedDict

class OrderBook:
    def __init__(self, symbol):
        self.symbol = symbol
        self.bids = OrderedDict() # Alış emirleri
        self.asks = OrderedDict() # Satış emirleri

    def update(self, bids_data, asks_data):
        # Yeni gelen verilerle lokal defteri güncelle
        for price, quantity in bids_data:
            if float(quantity) == 0:
                self.bids.pop(price, None)
            else:
                self.bids[price] = quantity
        # Aynı işlem asks için de yapılır...
```

**Mühendislik Uyarısı:** Sürekli büyüyen bir sözlük (dictionary) bellek sızıntısına yol açabilir. Sadece en iyi 20-50 fiyat seviyesini tutarak hem CPU hem de bellek kullanımını optimize etmelisiniz.

![Alfasayısal Hız ve HFT Mimarisi](/assets/img/posts/hft-low-latency-trading-architecture.png)

## Latency (Gecikme) Optimizasyonu: Milisaniyelerle Savaş

WebSocket kullanmak tek başına yeterli değildir. Yazılım mimarinizin her katmanında gecikmeyi minimize etmelisiniz. 

### 1. JSON Parsing Darboğazı
Python'ın standart `json` kütüphanesi asenkron döngüler için yavaş kalabilir. `ujson` veya `orjson` gibi C tabanlı kütüphaneler kullanarak mesaj parse süresini %40-60 oranında azaltabilirsiniz.

### 2. DNS ve Connection Reuse
Sürekli yeni bağlantı açmak (TCP Handshake + TLS), trading sistemlerinin düşmanıdır. WebSocket bağlantısını bir kez kurup "Keep-alive" ile canlı tutmak hayati önem taşır.

### 3. İşletim Sistemi Seviyesi Ayarlar
Linux üzerinde çalışan bir bot için:
- **TCP_NODELAY:** Nagle algoritmasını devre dışı bırakarak paketlerin birikmeden anında gönderilmesini sağlar.
- **Taskset:** Python process'ini belirli bir CPU çekirdeğine sabitleyerek (CPU pinning) bağlam değişimini (context switch) minimize edebilirsiniz.

## Güvenilirlik: Kesintisiz Çalışma (High Availability)

WebSocket bağlantıları doğası gereği kırılgandır. İnternet kesintisi, borsa bakımı veya ağ paket kayıpları her an yaşanabilir.

### Otomatik Yeniden Bağlanma (Exponential Backoff)
Bağlantı koptuğunda anında tekrar denemek yerine, her denemede bekleme süresini katlayarak artırmak (2s, 4s, 8s...), borsa sunucuları tarafından banlanmanızı (rate limiting) önler.

```python
async def resilient_stream(uri):
    retry_delay = 1
    while True:
        try:
            async with websockets.connect(uri) as ws:
                retry_delay = 1 # Reset delay
                while True:
                    data = await ws.recv()
                    # Veriyi işle...
        except Exception as e:
            print(f"Hata: {e}. {retry_delay}s içinde tekrar deneniyor...")
            await asyncio.sleep(retry_delay)
            retry_delay *= 2 # Exponential backoff
```

**Pro Tip:** Kritik sistemlerde bağlantı kopmasa bile gelen verideki "sequence number" (sıra numarası) kontrol edilmelidir. Eğer bir numara atlanmışsa, arada veri kaçmış demektir ve sistemin kendini resetleyip güncel snapshot'ı tekrar alması gerekir.

![Sürekli İzleme ve Loglama](/assets/img/posts/uptime-kuma-dashboard.png)

## Loglama ve İzleme (Monitoring)

Gerçek zamanlı bir sistemde neyin neden olduğunu anlamak için yapılandırılmış (structured) loglama hayati önem taşır.

### Merkezi Log Yönetimi
Python'ın standart `logging` modülünü JSON formatında çıktı üretecek şekilde konfigüre etmek, logların merkezi bir sisteme (ELK Stack veya Grafana Loki gibi) aktarılmasını kolaylaştırır. Bir trade neden gerçekleşti veya neden iptal edildi gibi soruların cevabı bu loglarda saklıdır.

```python
import logging
import json

class JsonFormatter(logging.Formatter):
    def format(self, record):
        log_record = {
            "timestamp": self.formatTime(record),
            "level": record.levelname,
            "message": record.getMessage(),
            "symbol": getattr(record, "symbol", "N/A")
        }
        return json.dumps(log_record)
```

## Bulut Yayılımı (Cloud Deployment) ve Güvenlik

Trading botunuzun evdeki bilgisayarınızda çalışması risklidir. Elektrik veya internet kesintisi büyük finansal kayıplara yol açabilir.

- **VPS Seçimi:** Mümkünse borsa sunucularına en yakın veri merkezini (AWS Tokyo veya Dublin gibi) seçmelisiniz. Bu, "network latency"yi 1-2 ms'ye kadar düşürebilir.
- **Dockerization:** Botunuzu Docker konteynerleri içinde çalıştırmak, bağımlılık yönetimini basitleştirir ve farklı ortamlarda (lokal vs prod) aynı davranışın sergilenmesini sağlar.
- **API Key Güvenliği:** API anahtarlarınızı asla kodun içine yazmayın. Ortam değişkenleri (Environment Variables) veya AWS Secrets Manager gibi güvenli kasa servislerini kullanın.

## Teknik Sözlük (Glossary)

- **Full-Duplex:** Verinin her iki yönde de eşzamanlı olarak akabildiği iletişim yöntemi.
- **AsyncIO Selector:** Python'da hangi socket'in veri okumaya hazır olduğunu takip eden alt seviye mekanizma.
- **Coroutine:** Duraklatılabilen ve kaldığı yerden devam edebilen, klasik fonksiyonlardan daha hafif fonksiyon yapıları.
- **Websocket Handshake:** Standart bir HTTP isteği ile başlayıp bağlantının WebSocket'e yükseltilmesi (Upgrade) süreci.
- **Heartbeat (Ping/Pong):** Bağlantının hala aktif olup olmadığını kontrol etmek için gönderilen küçük kontrol paketleri.
- **Order Flow Imbalance:** Alış ve satış emirlerinin hacimsel olarak dengesizleşmesi (genellikle fiyat hareketinin öncü göstergesidir).

## Sonuç: Hız ve Disiplin

Python ve WebSocket ikilisi, modern bir trading botu için en güçlü araçlardan biridir. Ancak unutmayın ki, en hızlı sistem her zaman en karlı sistem değildir. Hızın yanına sağlam bir risk yönetimi, hatasız bir veri işleme mantığı ve sürekli izleme (monitoring) eklemediğiniz sürece hız sadece "daha hızlı likidasyon" anlamına gelebilir.

Bu rehberde kurduğumuz yapı, profesyonel seviyede bir trading altyapısının sadece başlangıcıdır. Bir sonraki adımda, bu verileri kullanarak nasıl makine öğrenmesi modelleri veya karmaşık arbitragemodelleri geliştirebileceğinizi keşfedebilirsiniz. Piyasalar asla uyumaz, botunuzun da uyumayacak şekilde tasarlandığından emin olun.
