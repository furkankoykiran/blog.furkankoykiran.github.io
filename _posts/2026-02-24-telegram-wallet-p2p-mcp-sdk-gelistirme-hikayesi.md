---
title: "Telegram Wallet P2P API İle Kendi Trading Altyapınızı Kurun: Python, Node.js ve MCP Destekli Resmi Olmayan Ekosistem"
description: "Telegram'ın duyurduğu yeni Wallet P2P API'sini kullanarak kantitatif analiz, arbitraj takibi ve algoritmik ticaret için gerekli altyapıyı nasıl kurduğumun detaylı teknik analizidir. Claude gibi AI asistanları için entegre MCP sunucusu detaylarını içerir."
date: "2026-02-24 19:30:00 +0300"
categories: [Open Source, SDK, TypeScript, Python, Architecture]
tags: [telegram, wallet, p2p, algoritmik-ticaret, arbitraj, sdk, python, typescript, mcp, açık-kaynak]
---

![Telegram Wallet P2P API](/assets/img/telegram_wallet_p2p_api.png)

Kripto para piyasalarının en heyecan verici ve bir o kadar da karmaşık alanlarından biri, hiç şüphesiz eşler arası (P2P - Peer to Peer) ticarettir. Borsaların sunduğu anlık tahtaların aksine, P2P piyasaları yerel ödeme yöntemleri, anlık satıcı/alıcı dinamikleri ve bankacılık saatleri gibi sayısız fiziksel değişkenden etkilenir. Bu değişkenler, piyasada sıklıkla ciddi "arbitraj" (fiyat farkından kâr etme) fırsatları doğurur.

Yakın zamanda, devasa bir kullanıcı kitlesine sahip olan Telegram'ın kendi cüzdan çözümü (Wallet), geliştirici topluluğunda büyük yankı uyandıran sessiz bir adım attı.

> 📊 **P2P API for analytics and monitoring**
> You can now access P2P Market data via an API without manually checking offers in the app.
> Available data includes: active listings, buy and sell prices, fiat currencies, crypto assets, maker ratings, payment methods.

Bu kısa bildirim, Telegram gibi 900 milyondan fazla aktif kullanıcısı olan bir platformun içindeki kapalı bir ekonominin anahtarlarını geliştiricilere sunuyordu. Artık pazar verilerini manuel olarak uygulamayı açarak takip etmeye, sayısız ekran kaydırarak manuel fiyat hesaplaması yapmaya gerek kalmamıştı.

Ancak, bu fırsatın henüz ham bir maden olduğunu da belirtmek gerek: **API şu an için tamamen Read-Only (Sadece Okuma) modunda.** Yani doğrudan bot üzerinden otomatik alım-satım emirleri (Write) gönderemiyorsunuz. 

Peki, sadece veri okuyabildiğimiz bir API için neden bu kadar kapsamlı bir altyapı geliştirilmeli? Cevap tamamen "kantitatif analiz (quantitative analysis)" ve "hazırlık" prensiplerinde yatıyor. Yazılımsal ticarette (algorithmic trading) asıl değer, veriyi toplayıp anlamlandırabilmekten geçer. Fiat paritelerindeki makasları izlemek, likidite durumunu saat bazlı analiz etmek ve gelecekte "Write" (Yazma/İşlem yapma) özelliği açıldığında anında aksiyon alabilecek sinyalizasyon altyapısını bugünden hazır etmek şarttır.

Bu stratejik vizyon doğrultusunda, ham uç noktaları (endpoints) tüketmek yerine, tüm süreci standardize edecek, **hem Python hem de Node.js için resmi olmayan, type-safe (tip güvenli) ve üretime hazır (production-grade) bütünsellik taşıyan bir SDK ekosistemi** tasarlamaya karar verdim. Üstelik projeyi sadece geleneksel yazılım dili kalıplarında bırakmadım; günümüzün yeni normali olan **Model Context Protocol (MCP)** ile donatarak doğrudan yapay zekanın emrine sundum.

Bu detaylı yazıda, projenin ardındaki mühendislik kararlarını, kripto API'leri ile çalışırken dikkat edilmesi gereken veri kayıplarını, asenkron işlemleri ve yapay zeka entegrasyonunu ele alacağız.

## Bölüm 1: Mimari Kararlar - Neden Monorepo?

Ciddi bir yazılım altyapısı kurarken, sadece kod yazmak yeterli değildir; projeyi nasıl yönettiğiniz de projenin ömrünü belirler. Birbirini tamamlayan ancak farklı ekosistemlerde (Python/pip, Node.js/npm) çalışan paketlerim olacaktı. 

Bunları ayrı depolarda (repository) tutmak yerine **Monorepo** mimarisini tercih ettim. Dizin yapısını şu şekilde kurguladım:

1.  `packages/python`: Veri bilimi, makine öğrenmesi modelleri için geçmiş veri toplayıcıları ve scriptler için Python paketini barındırır.
2.  `packages/node`: Telegram Mini App'leri, Next.js tabanlı arayüzler ve gerçek zamanlı Node backend'leri için TypeScript SDK'sını içerir.
3.  `packages/mcp`: Yapay zeka asistanlarına sistemin anahtarını veren Model Context Protocol sunucusudur.

Monorepo'nun en büyük avantajı, Telegram API'sinde yaşanacak ufacık bir model değişikliğinde, tüm paketlerin tek bir "Pull Request" ile senkronize bir şekilde güncellenebilmesidir. Dağınık versiyonlar ve "node'da çalışıyor ama python patlıyor" gibi kâbus senaryolarının önüne geçer.

## Bölüm 2: Node.js (TypeScript) SDK - Kripto ve Hassasiyet (Precision)

Finansal veriler söz konusu olduğunda, özellikle kripto para alanında, klasik web geliştirmedeki alışkanlıklar başa bela olabilir. JavaScript'in meşhur "floating point" (ondalıklı sayı) problemleri herkesin malumudur. `0.1 + 0.2 = 0.30000000000000004` esprisi finansal bir sistemde kritik kayıplar demektir.

### Zod ile Çalışma Zamanı (Runtime) Koruması

Dışarıdan (API'den) gelen veriye körü körüne güvenmek bir mühendislik hatasıdır. Telegram API'sinden gelen fiyat, ID, bakiye gibi kritik verilerin asla JS `Number` objesi olarak işlem görmemesi, `String` formatında alınması hayati önem taşır.

Bu nedenle TypeScript'te tanımlamalar yaparken, sadece build-time değil, runtime güvencesi de sunan **Zod** kütüphanesini mimarinin kalbine yerleştirdim.

```typescript
import { z } from 'zod';

// API'den dönen kritik P2P ilan nesnesini güvenceye alıyoruz
const OnlineItemSchema = z.object({
  id: z.string(), // ID'ler asla int bazlı sınırlarla kısıtlanmamalı (BigInt kayıpları)
  number: z.string(),
  userId: z.number(),
  nickname: z.string(),
  cryptoCurrency: z.string(),
  fiatCurrency: z.string(),
  side: z.enum(['BUY', 'SELL']), // Enum ile hatalı parametrelerin önüne geçiş
  price: z.string(), // KESİNLİKLE string kalmalı. Hesaplamalar Decimal/BigNumber kütüphaneleriyle yapılmalı.
  availableAmount: z.string(),
  minAmount: z.string(),
  maxAmount: z.string(),
  isOnline: z.boolean(),
  isAutoAccept: z.boolean()
});

export type OnlineItem = z.infer<typeof OnlineItemSchema>;
```

### Kurulumsuz İstemci: Native Fetch

Ayrıca Node.js ekosisteminin ağır kütüphane yığınlarından (node_modules cehennemi) kaçınmak projenin temel ilkelerinden biriydi. Bu sebeple Axios veya benzeri bir `request` kütüphanesi yerine, Node.js 18 serisi ile standartlaşan native `fetch` API tercih edildi. Böylece kütüphanenin izi (footprint) minimuma indirildi ve dışa bağımlılıkların getirebileceği olası güvenlik zaafiyetleri kapı dışında bırakıldı.

Node.js SDK ile bir fiyat makası (spread) analizi botunun iskeletini yazmak son derece kolaydır:

```typescript
import { TelegramP2PClient } from 'telegram-wallet-p2p';

const client = new TelegramP2PClient('SİZİN_API_ANAHTARINIZ'); // DevTools'dan alınan auth key

async function analyzeSpread() {
  try {
    const buyOrders = await client.getOnlineItems({
      cryptoCurrency: 'USDT', fiatCurrency: 'TRY', side: 'BUY', pageSize: 1
    });
    
    const sellOrders = await client.getOnlineItems({
      cryptoCurrency: 'USDT', fiatCurrency: 'TRY', side: 'SELL', pageSize: 1
    });

    const highestBid = parseFloat(buyOrders[0]?.price);
    const lowestAsk = parseFloat(sellOrders[0]?.price);
    const spread = lowestAsk - highestBid;

    console.log(`Piyasa Makası (Spread): ${spread.toFixed(2)} TRY`);
  } catch (error) {
    console.error("Analiz edilirken bağlantı hatası oluştu:", error);
  }
}

analyzeSpread();
```

## Bölüm 3: Python SDK - Asenkron Yüksek Frekanslı Veri Toplama

P2P tarafında manuel olarak arayüzde gezen bir insanın görebileceği limitli ilan sayısının ötesine geçmek için "veri madenciliği" (data mining) mantığıyla çalışmanız gerekir. 10 farklı itibari para birimi (Fiat Currency) karşılığında USDT/TON/BTC ilanlarını sürekli ve duraksamadan çekecek bir analiz altyapısı kurmak Python cephesinin görevidir.

Bu görevi senkron çalışan requests vb. kütüphaneler yerine tamamen asenkron tabanlı olan **aiohttp** ile donatılmış bir altyapı üstlendi. Asenkron programlama, ağ isteklerinde geçen I/O bekleme sürelerini adeta sıfıra indirerek botun paralel işlem yapmasını (Concurrency) sağlar.

### Pydantic ve Veri Standardizasyonu

TypeScript'te Zod'un gördüğü kritik validasyon görevini Python'da elbette **Pydantic V2** göğüslüyor. Pydantic, Python'da veriyi sadece doğrulamaz, aynı zamanda muazzam derecede hızlı bir şekilde nesneye dönüştürür (deserialize).

Örnek bir yüksek-frekans (high-frequency) veri çekme modeli şöyle görünür:

```python
import asyncio
from typing import List, Optional
from pydantic import BaseModel
from telegram_wallet_p2p import P2PClient

# Model tanımlaması
class OfferAnalytics(BaseModel):
    nickname: str
    price: float
    methods: List[str]

async def collect_market_data(currency: str, client: P2PClient):
    print(f"[{currency}] piyasası taranıyor...")
    # asyncio.gather ile tek seferde çift yönlü derinlik analizi (Order Book Depth)
    tasks = [
        client.get_online_items("USDT", currency, "BUY"),
        client.get_online_items("USDT", currency, "SELL")
    ]
    buy_orders, sell_orders = await asyncio.gather(*tasks)
    return currency, len(buy_orders), len(sell_orders)

async def main():
    fiats = ["TRY", "RUB", "EUR", "USD"]
    async with P2PClient(api_key="API_KEY_BURAYA") as client:
        # 4 Farklı fiat piyasasını aynı anda tarıyoruz
        tasks = [collect_market_data(f, client) for f in fiats]
        results = await asyncio.gather(*tasks)
        
        for currency, buys, sells in results:
            print(f"Birim: {currency} | Alış İlanı: {buys} | Satış İlanı: {sells}")

if __name__ == "__main__":
    asyncio.run(main())
```

Yukarıdaki script, piyasadaki sığ (shallow) ve derin (deep) fiat para birimlerini milisaniyeler içerisinde haritalandırabilir. Bu veriyi bir zaman serisi veritabanına (TimescaleDB, InfluxDB vb.) yazarak geçmişe dönük arbitraj analizleri rahatlıkla çalıştırılabilir.

## Bölüm 4: MCP (Model Context Protocol) - AI Asistanlarınızı Trading Desk'ine Bağlayın

Yeni nesil yazılım süreçlerinde "Kütüphanemi yazdım, kullanacak olan alsın kodlasın" demek vizyonsuz bir yaklaşımdır. Claude (Anthropic), Cursor ve diğer LLM'ler standart araçlar haline geldiğine göre, yazdığımız altyapıyı onların da kullanabileceği bir protokole oturtmamız gerekir. 

Node.js SDK üzerine kurduğum **`packages/mcp`** modülü tam da bu işe yarar. Sizin yerinize Telegram cüzdan kimlik doğrulamasını yapar ve uygulamanızın sunduğu fonksiyonları bir "tool" (araç) olarak LLM'lerin emrine sunar.

AI'a verdiğiniz şu komutu hayal edin:
*"Claude, şu an Telegram P2P'de, Ziraat Bankası (Ziraat) ödeme yöntemi kabul eden, limiti en az 5000 TL olan en iyi 3 satıcıyı benim için listele ve onlardan alım yaparsam ortalama fiyatın ne olacağını söyle."*

Normal şartlarda bu sorunun cevabı için saatler sürecek Python scriptleri yazmanız gerekir. Ancak projedeki MCP sunucusunu LLM istemcinize tanımladığınız an, Claude arkaplanda SDK'yı çağıracak, veriyi kendi okuyacak, filtreleme matematiğini kendi yapacak ve size sadece altın değerindeki raporu sunacaktır. İnsan ile karmaşık veri arasındaki yazılımsal bariyerler tamamen ortadan kalkmıştır. Araçlarınızı yapay zeka standartlarında sunmak artık bir lüks değil zorunluluktur.

## Bölüm 5: Sayfalandırma (Pagination) ve Rate-Limiting Gerçekleri

Finansal altyapılar inşa ederken karşılaşılan en büyük handikap sayfa sınırlarıdır. Telegram API'si devasa veri çekimlerini önlemek için katı sayfa limitleri (page size) koymuştur ve saniyede belirli bir eşiğin üzerinde istek attığınızda sistem sizi `HTTP 429 Too Many Requests` status koduyla belirli bir süre "spam" listesine alır (Rate Limit).

Bunu aşmak için projenin derinliklerinde gelişmiş bekleme ve yineleme algoritmaları kurmak şarttır. Her sayfa arasında stratejik bir uyku döngüsü (sleep loop) eklemek veya hata alındığında katlanarak bekleyen (Exponential Backoff) bir retry fonksiyonu kullanmak, bu projenin gelecekteki Write modüllerinde de en çok dikkat edeceği yapılardan biridir. 

Örneğin, yüzlerce emri barındıran derin bir tahtayı (Order Book) kazımak (scrape) için basit bir TypeScript iteratörü kurgulayalım:

```typescript
import { TelegramP2PClient, OnlineItem } from 'telegram-wallet-p2p';

const client = new TelegramP2PClient('SİZİN_API_ANAHTARINIZ');
const sleep = (ms: number) => new Promise(resolve => setTimeout(resolve, ms));

async function fetchAllActiveOrders(crypto: string, fiat: string): Promise<OnlineItem[]> {
  let allOrders: OnlineItem[] = [];
  let currentPage = 1;
  let hasMore = true;

  console.log(`[${crypto}/${fiat}] Piyasası taranmaya başlıyor...`);

  while (hasMore) {
    try {
      const pageData = await client.getOnlineItems({
        cryptoCurrency: crypto,
        fiatCurrency: fiat,
        side: 'BUY',
        page: currentPage,
        pageSize: 50 // Maksimum sayfa limiti
      });

      if (pageData.length === 0) {
        hasMore = false;
        break;
      }

      allOrders = [...allOrders, ...pageData];
      console.log(`Sayfa ${currentPage} çekildi. (Ara Toplam: ${allOrders.length})`);
      
      currentPage++;
      // Rate-Limit'e takılmamak için her sayfa arası 500ms nazik bekleme (Graceful Sleep)
      await sleep(500); 
    } catch (error: any) {
      if (error?.message?.includes('429')) {
        console.warn("API Kotası aşıldı! 5 saniye bekleniyor (Exponential Backoff başlatılıyor...)");
        await sleep(5000); // 429 yakalandığında bekle ve aynı sayfayı tekrar dene
        continue; 
      }
      throw error; // Diğer kritik hataları yukarı fırlat
    }
  }

  return allOrders;
}
```

Yukarıdaki gibi, sadece veriyi çekmek yerine "güvenli veri çekimi" algoritmaları kurmak, Read-Only modda yüzlerce ürün çağırırken sistemin stabilitesini koruyan en hayati kararlardan biridir. Özellikle botunuz bir Raspberry Pi üzerinde 7/24 çalışıyorsa, bağlantı koptuğunda "process"in çökmemesi kritik değer taşır.


## Gelecek Vizyonu ve Yol Haritası (Roadmap)

Bir analiz aracı oluşturmak güzel, fakat nihai hedef tamamen siber-organik bir ticaret sistemine evrilmek. Ekosistem için yakın gelecekte çizdiğim yol haritası şöyle şekilleniyor:

1. **Trade/Order (Write) API'nin Entegrasyonu:** Okuma yetkisinin yanına alım/satım (create order) yetkilerinin de ileride tam anlamıyla modüllere dâhil edilmesi. Böylece sinyalin üretildiği anda webhook üzerinden anında aksiyon alınması.
2. **WebSocket Desteği (Event-Driven Architecture):** Kur veya sipariş defteri (Order Book) güncellemelerinin REST üzerinden sürekli poll (sorulması) edilmesi yerine, sunucu tabanlı stream akışlarıyla sadece değişim anında yakalanması.
3. **Önbellek (Caching) Katmanı:** Çok sorgulanan USDT paritelerinde veritabanı veya Redis üzerinde otomatik mühletli (TTL) önbellek (Cache) mekanizması ile API kotasını koruma kalkanları eklentileri (plugins).

## Açık Kaynak Topluluğuna Çağrı

Böyle dinamik ve hızla şekil değiştiren kripto ortamlarında kapalı yapılar üretmek mantıksızdır. Tüm bu ekosistemi (Node.js, Python, MCP) açık kaynak (MIT License) felsefesiyle bir araya getirerek GitHub üzerinde dileyen herkesin kullanımına ve erişimine açtım.

Ticaret stratejilerini geliştiren, "veri altındır" mottosuyla piyasaları analiz etmeyi seven, ya da sadece modern TypeScript ve Python kodlamalarını MCP entegrasyonuyla canlı örneklerde incelemek isteyen her geliştirici projeyi forklayabilir.

👉 **[GitHub Telegram Wallet P2P SDK](https://github.com/furkankoykiran/telegram-wallet-p2p-sdk)**

Eğer siz de bu altyapıyı kendi botlarında test etmek veya projeye `Pull Request` ile katkı sağlamak istiyorsanız repoya mutlaka uğrayın. Birlikte, Telegram cüzdan mimarisinde çok daha devasa yapılar inşa edebiliriz. Katkıda bulunurken "Issue" açmaktan çekinmeyin ve en azından projeyi desteklemek için Repoya basitçe bir yıldız ⭐️ (Star) bırakmayı unutmayın.

Unutmayın; büyük finans algoritmalarının temelindeki en büyük sır, sağlam tuğlalar kullanmaktır. Biz şu anda o tuğlaları hep birlikte üretiyoruz.

Geliştirdiğiniz trading mekanizmalarında başarılar! Her yeni algoritmanızın kârlı, kodunuzun bug-free olması dileğiyle... Sağlıcakla kalın.

---

**Daha Fazla API ve Yapay Zeka Entegrasyonunu Okumak İsteyenler İçin:**
- [DevTo-MCP: DEV Community API'si ile AI Asistanları Arasında Güvenli Bir Köprü](/posts/devto-mcp-server-gelistirme-ve-yayinlama/)
- [OmniWire-MCP: AI Modelleri İçin Haber Köprüsü](/posts/omniwire-mcp-ai-news-server/)

![Furkan Köykıran](/assets/img/avatar.png)
*Furkan Köykıran - Senior Software Engineer*
