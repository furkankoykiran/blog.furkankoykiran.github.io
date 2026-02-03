---
title: "Python ile Telegram Trading Botu Geliştirme: Mimariden Yayına"
description: "Python ve Telegram Bot API kullanarak 7/24 çalışan profesyonel trading botu mimarisi. Asenkron programlama, yüksek erişilebilirlik, CCXT entegrasyonu ve güvenli anahtar yönetimi."
date: "2024-07-22"
categories: [Backend, Python]
tags: [python, telegram-bot, trading, crypto, automation, api, infrastructure]
image:
  path: "/assets/img/posts/telegram-trading-bot-python.png"
  alt: "Python ile Telegram Trading Botu"
---

## Giriş: Finansal Araç Üretiminde Telegram'ın Rolü

Piyasaların hiç uyumadığı bir dünyada, bir yatırımcı için en değerli varlık "bilgiye anlık erişim" ve "hızlı aksiyon alma" kabiliyetidir. Geleneksel web arayüzleri veya mobil uygulamalar bazen çok hantal kalabilir. İşte burada Telegram, sadece bir mesajlaşma uygulaması olmaktan çıkıp, güçlü bir "Command-Line Interface" (CLI) ve bildirim terminaline dönüşüyor.

Senior bir geliştirici gözüyle, bir Telegram trading botu geliştirmek sadece `/buy` veya `/sell` komutlarını işlemek değildir. Bu, 7/24 kesintisiz çalışması gereken, asenkron işlemleri (asyncio) yöneten, hata toleransı yüksek ve en önemlisi "güvenliği" birinci önceliğe koyan bir mikroservis tasarlamaktır. Bu yazıda, hobi amaçlı bir bota değil, profesyonel bir trading terminaline giden mimari kararları inceleyeceğiz.

![Telegram Bot Python Mimarisi](/assets/img/posts/telegram-bot-python-architecture.png)

## Mimari Kararlar: Neden Python ve `python-telegram-bot`?

Trading dünyasında Python'ın tartışmasız bir üstünlüğü var. Bunun temel sebebi kütüphane zenginliği (`pandas`, `numpy`, `ccxt`) ve hızlı prototipleme imkanıdır. Ancak Telegram tarafında kütüphane seçimi kritik bir yol ayrımıdır.

### 1. Asenkron Altyapı (Asyncio)
Modern bir trading botu, aynı anda onlarca fiyat alarmını kontrol etmeli, birden fazla kullanıcıdan gelen komutları işlemeli ve eş zamanlı olarak borsa API'lerinden veri çekmelidir. Eğer botunuzun hızı ve gecikmesi (latency) sizin için kritikse, [WebSocket Trading Bot](/blog-development/python/2024/08/02/python-ile-websocket-trading-bot/) mimarimizi de incelemenizi öneririm.

### 2. CCXT Entegrasyonu
Dünya üzerinde yüzlerce kripto para borsası var ve her birinin API yapısı farklıdır. CCXT (CryptoCurrency eXchange Trading) kütüphanesi, bu farklılıkları standart bir formata indirger. Bir senior geliştirici olarak asla borsanın "kendi" SDK'sına bağımlı kalmamalısınız; CCXT sayesinde borsalar arası geçiş yapmak veya arbitraj botları geliştirmek sadece birkaç satır kod değişikliği demektir.

### 3. State Management (Durum Yönetimi)
Kullanıcının bir emir verirken izlediği adımları (symbol seçimi -> miktar girişi -> onay) takip etmek için "ConversationHandler" yapısı hayati önem taşır. RAM üzerinde tutulan geçici durumlar (in-memory state), bot yeniden başladığında kaybolur. Profesyonel bir sistemde bu durumlar Redis veya PostgreSQL gibi kalıcı bir hafızada (persistent storage) tutulmalıdır.

**Mühendislik Notu:** Botunuzu geliştirirken asla "Global State" kullanmayın. Her kullanıcı isteği asenkron bir context içinde işlendiği için, thread-safety ve race condition risklerini her zaman göz önünde bulundurmalısınız.

![Telegram Bot Otomasyon Akışı](/assets/img/posts/telegram-bot-workflow-automation.png)

## Çekirdek Yapı: JobQueue ve Command Handlers

Bir trading botu reaktif (komut bekleyen) değil, proaktif (sürekli izleyen) olmalıdır. Python Telegram Bot kütüphanesinin sunduğu `JobQueue`, sistemin motorudur.

### 1. Periyodik Görevler (JobQueue)
Fiyat alarmları, trailing stop-loss kontrolleri veya saatlik bakiye raporları `job_queue` üzerinden yönetilir. 
```python
async def check_alerts(context: ContextTypes.DEFAULT_TYPE):
    # Tüm aktif alarmları borsadan çekilen fiyatla karşılaştır
    for user_id, symbol in active_alerts:
        price = await exchange.fetch_ticker(symbol)
        if price >= threshold:
            await context.bot.send_message(chat_id=user_id, text=f"Hedef fiyat: {price}")

# Her 30 saniyede bir çalıştır
application.job_queue.run_repeating(check_alerts, interval=30)
```
Bu yaklaşım, ana event loop'u bloklamadan (non-blocking) arka planda binlerce kontrolü yapabilmenizi sağlar.

### 2. Command Handlers ve Reply Markup
Senior bir tasarıma sahip bot, kullanıcıyı "komut ezberlemekten" kurtarmalıdır. `ReplyKeyboardMarkup` ve `InlineKeyboardMarkup` kullanarak "Buton Odaklı" bir arayüz sunmalısınız. Kullanıcı `/start` dediğinde önüne gelen "Fiyatlar", "Bakiyem", "Açık Emirler" gibi butonlar, kullanıcı hatalarını minimize eder.

## Yüksek Erişilebilirlik ve Hata Toleransı

Trading botu bir "mission critical" uygulamadır. Botun çökmesi, bir kullanıcının zararı durduramaması (stop-loss) demektir.

- **Websocket vs Polling:** Başlangıç için `run_polling` yeterlidir ancak profesyonel bir bota geçişte `webhook` kullanmak, ölçeklenebilirlik ve güvenlik (HTTPS) için zorunludur.
- **Circuit Breaker:** Eğer borsa API'si hata veriyorsa, bot sürekli istek gönderip borsa tarafından IP ban yememelidir. Hata sayısı belirli bir eşiğe ulaştığında istekler geçici bir süre durdurulmalıdır.
- **Logging:** Sadece `print()` değil; `logging` modülüyle yapısal (structured) loglar tutulmalıdır. Emrin ne zaman, hangi fiyattan ve hangi hata mesajıyla başarısız olduğu mutlaka bir dosyaya veya log sunucusuna kaydedilmelidir.

![Otomatik Trading Altyapısı](/assets/img/posts/python-automated-trading-architecture.png)

## Güvenlik Stratejisi: API Anahtarlarını Korumak

Trading botları, doğası gereği yüksek finansal değere sahip anahtarlara erişir. Bir güvenlik açığı, tüm sermayenin sıfırlanmasıyla sonuçlanabilir.

### 1. Beyaz Liste (Whitelisting)
Botun sadece belirli Telegram User ID'lerine cevap vermesini sağlamak en temel adımdır. Kodunuzda `ALLOWED_USER_IDS` kontrolü yapmadan hiçbir trading fonksiyonunu tetiklememelisiniz.

### 2. API Key Yetkilendirmesi
Borsadan API key oluştururken "Withdrawal" (Para Çekme) yetkisini mutlaka kapalı tutun. Botun sadece "Spot Trading" ve "Margin Trading" yetkilerine sahip olması, olası bir sızıntıda hasarı sınırlar.

### 3. Anahtar Yönetimi (Secret Management)
API anahtarlarını asla kodun içine gömmeyin. `.env` dosyaları bir başlangıçtır ancak prodüksiyon ortamında HashiCorp Vault veya AWS Secrets Manager gibi profesyonel çözümler kullanılmalıdır. Ayrıca, borsaya sadece botun çalıştığı sunucunun IP adresinden erişim izni (IP Whitelisting) verilmesi, güvenliği bir kademe daha artırır.

## İzleme ve Health Checks (Sağlık Kontrolleri)

Botun "çalışıyor" görünmesi, aslında her şeyin yolunda olduğu anlamına gelmez. API bağlantısı kopmuş olabilir veya borsa rate-limit uygulamış olabilir.

![Uptime Kuma Dashboard](/assets/img/posts/uptime-kuma-dashboard.png)

### İzleme Katmanları
- **Heartbeat:** Botun her dakika "yaşıyorum" bilgisini bir izleme servisine (örn: Healthchecks.io) göndermesi. Eğer mesaj gitmezse, Telegram üzerinden size bildirim gelmesini sağlar.
- **Latency Monitoring:** Borsa API'sine yapılan isteklerin süresini izlemek. Ani gecikme artışları, ağ sorunlarının veya borsa tarafındaki yoğunluğun habercisidir.
- **Visual Dashboards:** Prometheus ve Grafana kullanarak emir başarı oranları, anlık portföy değeri ve relayer performansını görselleştirmek, senior bir mühendislik yaklaşımıdır.

**Kritik Tavsiye:** Botun her zaman "stateless" (durumsuz) kalmasına özen gösterin. Sunucu çöktüğünde veya bot yeniden başladığında, mevcut durumu (açık emirler vb.) otomatik olarak borsadan senkronize etmelidir.

![Python Asyncio Concurrency](/assets/img/posts/python-asyncio-concurrent-programming.png)

## Risk Yönetimi: Stop-Loss ve Take-Profit

Bir trading botunun sizi zengin etmesinden önce, sizi fakirleştirmemesini sağlamalısınız. Senior bir geliştirici, trading mantığının içine "Hard-coded" (sabit kodlanmış) risk limitleri ekler.

### 1. Otomatik Stop-Loss
Emre girildiği anda, eş zamanlı olarak bir stop-loss emri de borsaya iletilmelidir. Eğer OCO (One-Cancels-the-Other) emri destekleniyorsa kullanımı tercih edilmelidir. Botun kendi içindeki bir hata nedeniyle stop-loss'un tetiklenmemesi riskine karşı, bu emirler borsa tarafında (on-exchange) tutulmalıdır.

### 2. Max Position Sizing
Botun tek bir işleme tüm bakiyeyi (all-in) yatırmasını engelleyen limitler olmalıdır. Toplam sermayenin %1 veya %2'sinden fazlasını tek bir işlemde riske atmamak, uzun vadeli hayatta kalmanın anahtarıdır.

## Yayına Alma: Docker ve CI/CD Süreçleri

Botun yerel makinenizde çalışması bir şey ifade etmez. Profesyonel bir bota giden yol Dockerizasyon ve bulut dağıtımından geçer.

### Dockerized Bot
```dockerfile
FROM python:3.10-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["python", "bot.py"]
```
Docker sayesinde botunuzu her ortamda (Local, VPS, Cloud) aynı izolasyonla çalıştırabilirsiniz. `docker-compose` kullanarak yanına bir de `Redis` ekleyebilir ve kalıcı durum yönetimini kolayca çözebilirsiniz.

### CI/CD Pipeline
GitHub Actions veya GitLab CI kullanarak, koda her `push` yaptığınızda testlerin çalışmasını (Pytest) ve başarılıysa botun sunucuda otomatik olarak yeniden başlamasını (Rolling Update) sağlamak, profesyonel bir iş akışıdır.

## Teknik Sözlük (Glossary)

- **CCXT:** Çoklu borsa API'lerini standartlaştıran Python kütüphanesi.
- **Asyncio:** Python'da eş zamanlı (concurrent) programlama kütüphanesi.
- **Polling:** Botun belirli aralıklarla Telegram sunucusuna "yeni mesaj var mı?" diye sorması.
- **Webhook:** Telegram sunucusunun yeni bir mesaj geldiğinde doğrudan sizin sunucunuza bildirim göndermesi.
- **JobQueue:** Belirli aralıklarla çalışması gereken arka plan görevlerini yöneten yapı.
- **API Secret:** API key ile birlikte kullanılan, imza doğrulaması için gereken gizli anahtar.
- **Rate Limit:** Borsanın belirli bir sürede yapabileceğiniz maksimum istek sayısını sınırlaması.
- **Backtesting:** Bir stratejinin geçmiş veriler üzerinde ne kadar başarılı olduğunu test etme süreci.
- **OCO Order:** Bir emrin gerçekleşmesi durumunda diğerinin iptal olduğu akıllı emir türü.
- **HSM (Hardware Security Module):** Kriptografik anahtarları donanımsal olarak koruyan güvenli birim.

## Gelecek Vizyonu: AI ve Duygu Analizi Entegrasyonu

Telegram botunuzu bir adım öteye taşımak için OpenAI API veya HuggingFace modellerini entegre edebilirsiniz. Örneğin; belirli bir coin hakkında Twitter'daki veya haber kanallarındaki duygu analizini (Sentiment Analysis) botunuza bir indikatör olarak eklemek, size piyasada büyük bir avantaj sağlayacaktır.

## Sonuç

Kendi trading botunuzu geliştirmek, sadece finansal bir araç üretmek değil, aynı zamanda sistem tasarımı, güvenlik ve asenkron programlama konularında derinleşmek demektir. Botunuzu yazarken her zaman "en kötü senaryoyu" (borsanın çökmesi, internetin kopması, API sızıntısı) düşünerek defansif bir yaklaşım sergileyin. Küçük adımlarla başlayın, her zaman test ağlarında (testnets) deneyin ve kodun sizi değil, sizin kodu yönettiğinizden emin olun.

## Teknik Karşılaştırma: Polling vs Webhook

Hangi yöntemi ne zaman seçmelisiniz? İşte mühendislik perspektifinden bir kıyaslama:

| Özellik | Long Polling | Webhook |
| :--- | :--- | :--- |
| **Kurulum** | Çok Kolay (Yerel cihazda çalışır) | Orta (SSL/HTTPS ve Public IP gerekir) |
| **Gecikme** | Düşük (Sizin belirlediğiniz interval) | Çok Düşük (Gerçek zamanlı bildirim) |
| **Güvenlik** | Güvenli (Dışarıdan erişim yok) | Kritik (Port açmanız gerekir) |
| **Ölçekleme** | Düzey 1 (Tek process) | Düzey 2 (Load Balancer arkası çalışabilir) |

## Ölçeklenebilirlik: Binlerce Kullanıcıya Hizmet Vermek

Botunuz popülerleştiğinde, tek bir Python process'i yetersiz kalabilir. Senior bir geliştirici, sistemi yatayda nasıl ölçekleyeceğini bilmelidir.

- **Message Broker Kullanımı:** Gelen mesajları doğrudan işlemek yerine bir `RabbitMQ` veya `Redis Queue`'ya atmak, botun arayüzünün (Telegram tarafı) her zaman akıcı kalmasını sağlar.
- **Worker Microservices:** Kuyruğa atılan mesajları bağımsız "worker"lar işler. Eğer trading işlemleri yavaşsa, sadece worker sayısını artırarak sistemi hızlandırabilirsiniz.
- **Database Sharding:** Kullanıcı verilerini ve işlem geçmişini farklı veritabanı parçalarına bölmek, yüksek trafikli anlarda tıkanıklığı önler.

## Yeni Nesil: Telegram Mini Apps (TMA)

Artık sadece metin tabanlı botlarla sınırlı değiliz. Telegram Mini Apps sayesinde, botunuzun içinde React veya Vue ile geliştirilmiş tam teşekküllü bir "Trading Terminal" sunabilirsiniz.

**Senior İpucu:** Kullanıcı deneyimini (UX) artırmak için, karmaşık analizleri Mini App üzerinden görselleştirip, hızlı aksiyonları (Al/Sat) butonlarla yönetmek, modern Web3 uygulamalarının (örn: Unibot, Maestro) izlediği yoldur. Bu, botunuzu basit bir script olmaktan çıkarıp, gerçek bir finansal ürüne dönüştürür.

## İleri Okuma ve Kaynaklar
- Python Telegram Bot (PTB) Documentation
- CCXT Framework Guide
- Binance API Advanced Documentation
- Security Best Practices for Trading Bots
- Telegram Mini Apps (TMA) Developer Guide
