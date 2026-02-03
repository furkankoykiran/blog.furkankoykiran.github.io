---
title: "Python ile Binance API: Algo-Trading Mimarisi ve Stratejileri"
description: "Sıfırdan production-ready trading bot geliştirme. Async mimari, WebSocket stream yönetimi, risk analizi ve emir yönetim stratejileri."
date: 2024-02-09
categories: [Algorithmic Trading, Python]
tags: [binance-api, python, trading-bot, async, websocket, algo-trading]
image:
  path: /assets/img/posts/python-automated-trading-architecture.png
  alt: "Python Algorithmic Trading Architecture"
---

## Giriş: Milyon Dolarlık API Keys

Trading bot geliştirmek, sadece "Al" ve "Sat" emirleri göndermekten ibaret değildir. Asıl mesele, 7/24 kesintisiz çalışan, hata toleransı yüksek (fault-tolerant) ve milisaniyeler içinde karar verebilen bir **sistem** tasarlamaktır.

Bir Senior Engineer gözüyle, başarılı bir botun %20'si strateji, %80'i ise altyapı ve risk yönetimidir. Bu rehberde, basit scriptlerden öte, kurumsal seviyede bir trading motorunun nasıl kurulacağını inceleyeceğiz.

![AI Stock Trading Visualization](/assets/img/posts/ai-stock-trading-visualization.jpg)

## 1. Mimari: Latency ile Savaş

Binance REST API üzerinden saniyede bir fiyat sorup karar vermek, amatör ligde kalmanıza neden olur. Profesyonel sistemler **WebSocket** üzerinden veri akışı (stream) alır ve asenkron (AsyncIO) yapıda çalışır.

### Senkron vs Asenkron Yaklaşım
Python'un `requests` kütüphanesi bloklayıcıdır (blocking). Fiyat beklerken işlem yapamazsınız. `aiohttp` ve `asyncio` kullanarak, aynı anda binlerce pariteyi dinleyebilir ve emir yönetimini paralel yürütebilirsiniz.

```python
import asyncio
from binance import AsyncClient, BinanceSocketManager

async def main():
    client = await AsyncClient.create(API_KEY, API_SECRET)
    bm = BinanceSocketManager(client)
    
    # Multiplex Stream: BTC, ETH ve BNB fiyatlarını aynı anda dinle
    ts = bm.multiplex_socket(['btcusdt@trade', 'ethusdt@trade', 'bnbusdt@trade'])
    
    async with ts as tscm:
        while True:
            res = await tscm.recv()
            if res:
                await decision_engine(res) # Karar mekanizması
    
    await client.close_connection()

if __name__ == "__main__":
    loop = asyncio.get_event_loop()
    loop.run_until_complete(main())
```

Bu yapı, "Event-Driven" (Olay Güdümlü) mimarinin temelidir. Fiyat değiştiği anda (Event), botunuz tetiklenir.

![Order Book Depth Chart](/assets/img/posts/order-book-depth-chart-visualization.png)

## 2. Akıllı Emir Yönetimi: Taker vs Maker

Yeni başlayanlar genellikle `ORDER_TYPE_MARKET` kullanır. Bu, %0.1 yerine %0.2 komisyon ödemenize ve "Slippage" yemenize neden olur. Profesyonel botlar, Limit Order kullanır ve "Order Chasing" (Fiyat Kovalama) algoritmaları uygular.

```python
async def smart_limit_buy(client, symbol, qty, max_price):
    # En iyi alış fiyatını (Best Bid) al
    ticker = await client.get_order_book_ticker(symbol=symbol)
    best_bid = float(ticker['bidPrice'])
    
    if best_bid < max_price:
        # Best Bid'in bir tık üzerine (one tick above) limit emir gir
        price = best_bid + 0.01 
        order = await client.create_order(
            symbol=symbol,
            side='BUY',
            type='LIMIT',
            timeInForce='GTC',
            quantity=qty,
            price=str(price)
        )
        return order
```

## 3. Risk Yönetimi: Sermayeyi Korumak

Trading'de en önemli kural: "Asla batma". Ne kadar iyi bir stratejiniz olursa olsun, risk yönetiminiz yoksa er ya da geç sıfırlanırsınız.

*   **Sabit Kesir (Fixed Fraction):** Her işlemde sermayenin maks %1-2'sini riske atın.
*   **Kelly Kriteri:** Olasılıksal olarak "en optimimal" pozisyon büyüklüğünü hesaplar (fakat dikkatli kullanılmalı).

```python
def calculate_position_size(capatial, risk_per_trade=0.01, stop_loss_pct=0.05):
    """
    Sermayenin %1'ini riske atarak pozisyon büyüklüğü hesapla
    Örn: 10k sermaye, %1 risk (100$), %5 stop-loss -> 2000$ pozisyon
    """
    risk_amount = capatial * risk_per_trade
    position_size = risk_amount / stop_loss_pct
    return position_size

# 10.000$ Sermaye ile işlem
# risk_amount = 100$
# stop_loss = %5
# position_size = 2000$ (kaldıraçsız)
```

*   **Hard Stop vs Soft Stop:** Emir defterinde bekleyen Stop-Loss (Hard) bazen "Stop Hunting" kurbanı olabilir. Botunuzun hafızasında tuttuğu ve tetiklendiğinde market emri attığı (Soft) stoplar daha güvenlidir ancak ani çöküşlerde risklidir.

## 4. Strateji Doğrulama: Backtesting

Canlı parayı riske atmadan önce stratejinizi geçmiş verilerle test etmelisiniz. `backtrader` kütüphanesi bunun için endüstri standardıdır.

```python
import backtrader as bt

class RSIStrategy(bt.Strategy):
    def next(self):
        if self.rsi < 30 and not self.position:
            self.buy(size=1)
        elif self.rsi > 70 and self.position:
            self.sell(size=1)

cerebro = bt.Cerebro()
cerebro.addstrategy(RSIStrategy)
cerebro.run()
# %100 Python ile local backtesting
```

![Telegram Trading Bot Architecture](/assets/img/posts/telegram-trading-bot-python.png)

## 5. Altyapı ve Production: 7/24 Kesintisizlik

Botunuzu evdeki laptop'ta çalıştırmayın. AWS EC2 (t3.micro) veya DigitalOcean Droplet en ucuz ve güvenli çözümdür.

### Dockerizasyon
Bağımlılık cehenneminden kurtulmak için botunuzu containerize edin:

```dockerfile
FROM python:3.10-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
CMD ["python", "bot.py"]
```

### Logging: Eyes on the Code
`print()` kullanmak yerine structure log kullanın. `loguru` kütüphanesi otomatik rotasyon ve renkli çıktı sağlar.

```python
from loguru import logger

logger.add("logs/trading_{time}.log", rotation="500 MB", level="INFO")

def place_order(symbol, qty):
    logger.info(f"Placing order for {symbol}, Qty: {qty}")
    try:
        # Order logic...
        logger.success("Order filled successfully")
    except Exception as e:
        logger.error(f"Order failed: {e}")
```

## 6. Advanced Strategies: Grid Trading

Basit al-sat stratejileri yatay piyasada para kaybeder. Grid Trading, belirli fiyat aralıklarına kademeli emirler dizerek (Buy Low, Sell High) volatiliteden kar eder.

*   **Alt Limit:** 40,000 USDT
*   **Üst Limit:** 50,000 USDT
*   **Grid Sayısı:** 10
*   **Grid Aralığı:** 1000 USDT

Bu stratejiyi Python ile kodlarken `numpy.linspace(40000, 50000, 10)` fonksiyonu ile grid seviyelerini otomatik hesaplatabilirsiniz.

## 7. Sonuç

Kendi trading botunuzu yazmak, finansal özgürlüğe giden yolda en teknik adımdır. Hazır botların aksine, oyunun kurallarını siz belirlersiniz.

Eğer bu botu bir arayüz ile yönetmek isterseniz [FastAPI ile RESTful API](/backend/python/2024/04/27/python-fastapi-restful-api-gelistirme/) yazımı, verileri analiz etmek için ise [Blockchain Data Analysis](/blockchain/data-science/2024/03/07/blockchain-veri-analizi-python-araclari/) rehberimi inceleyebilirsiniz.
