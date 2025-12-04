---
title: "Python ile WebSocket Trading Bot: Real-Time Kripto Trading Sistemi"
description: "Python asyncio ve WebSocket ile milisaniye hızında gerçek zamanlı kripto trading botu geliştirme. Order book analizi, latency optimizasyonu ve HFT stratejileri."
date: "2024-08-02 10:00:00 +0300"
categories: [High-Frequency Trading, Bot Development]
tags: [python, websocket, trading-bot, real-time, hft, asyncio, kripto, automation]
image:
  path: /assets/img/posts/websocket-real-time-architecture-diagram.png
  alt: "WebSocket Real-Time Data Stream Architecture"
---

## Giriş

Modern kripto para piyasalarında başarılı olmak için milisaniye seviyesinde hızlı karar verme ve işlem gerçekleştirme yeteneği kritik öneme sahiptir. Geleneksel REST API tabanlı sistemler, sürekli polling gerektirdiği için hem yavaş hem de kaynak tüketimi açısından verimsizdir. WebSocket protokolü, sürekli açık bağlantı üzerinden gerçek zamanlı, çift yönlü iletişim sağlayarak bu soruna ideal bir çözüm sunar.

Bu yazıda, Python kullanarak profesyonel bir WebSocket tabanlı trading bot nasıl geliştirileceğini, asyncio ile nasıl yüksek performanslı concurrent programlama yapılacağını, order book'ları gerçek zamanlı nasıl takip edeceğinizi ve latency'yi nasıl optimize edeceğinizi detaylı bir şekilde öğreneceksiniz.

### Bu Yazıda Öğrenecekleriniz

- WebSocket protokolünün temel çalışma prensibi ve avantajları
- Python asyncio ile asenkron programlama temelleri
- Binance, Bybit gibi exchange'lerin WebSocket API'leri ile entegrasyon
- Order book verilerini gerçek zamanlı işleme ve analiz
- Yüksek frekanslı trading stratejileri için latency optimizasyonu
- Error handling ve reconnection stratejileri
- Production-ready bot mimarisi ve best practices

## WebSocket Protokolü Nedir?

WebSocket, HTTP handshake ile başlayan ancak sonrasında sürekli açık kalan, full-duplex (çift yönlü) bir iletişim protokolüdür. REST API'den temel farkları:

### REST API vs WebSocket

**REST API (Polling):**
```python
import time
import requests

# Her saniye fiyat kontrolü - Verimsiz!
while True:
    response = requests.get('https://api.exchange.com/ticker/BTCUSDT')
    price = response.json()['price']
    print(f"BTC Price: {price}")
    time.sleep(1)  # 1 saniye gecikme
```

**WebSocket (Real-time):**
```python
import asyncio
import websockets
import json

# Sürekli bağlantı - Anında veri!
async def price_stream():
    uri = "wss://stream.exchange.com/ws/btcusdt@ticker"
    async with websockets.connect(uri) as websocket:
        while True:
            data = await websocket.recv()
            ticker = json.loads(data)
            print(f"BTC Price: {ticker['p']}")  # Gecikme ~1-5ms
```

### WebSocket Avantajları

1. **Düşük Latency**: REST'te ~100-500ms, WebSocket'te ~1-10ms
2. **Daha Az Overhead**: Her istek için HTTP header gönderilmez
3. **Server Push**: Exchange verisi değiştiğinde anında gönderir
4. **Kaynak Verimliliği**: Sürekli bağlantı açıp kapatma maliyeti yok

![Python AsyncIO Concurrent Programming](/assets/img/posts/python-asyncio-concurrent-programming.png)
*Şekil 1: Python AsyncIO ile concurrent task yönetimi - Event loop birden fazla WebSocket bağlantısını paralel yönetir*

## Python AsyncIO ile Asenkron Programlama

WebSocket ile çalışmak için asyncio'nun temellerini anlamak şarttır. Asyncio, tek bir thread içinde binlerce concurrent işlem yapmanızı sağlar.

### Event Loop ve Coroutines

```python
import asyncio

# Coroutine - async def ile tanımlanır
async def fetch_price(symbol):
    print(f"Fetching {symbol}...")
    await asyncio.sleep(1)  # Non-blocking sleep
    return f"{symbol}: $50000"

async def main():
    # Birden fazla coroutine'i paralel çalıştır
    tasks = [
        fetch_price("BTC"),
        fetch_price("ETH"),
        fetch_price("SOL")
    ]
    
    # gather() tüm task'leri paralel çalıştırır
    results = await asyncio.gather(*tasks)
    for result in results:
        print(result)

# Event loop başlat
asyncio.run(main())
```

**Çıktı:**
```
Fetching BTC...
Fetching ETH...
Fetching SOL...
BTC: $50000
ETH: $50000
SOL: $50000
# Toplam süre: ~1 saniye (sıralı olsaydı 3 saniye)
```

### Async Context Managers

```python
import aiohttp
import asyncio

async def fetch_url(url):
    # aiohttp async HTTP client
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as response:
            return await response.json()

async def main():
    urls = [
        "https://api.binance.com/api/v3/ticker/24hr?symbol=BTCUSDT",
        "https://api.binance.com/api/v3/ticker/24hr?symbol=ETHUSDT",
    ]
    
    tasks = [fetch_url(url) for url in urls]
    results = await asyncio.gather(*tasks)
    
    for result in results:
        print(f"{result['symbol']}: ${result['lastPrice']}")

asyncio.run(main())
```

## Binance WebSocket API Entegrasyonu

Binance, en popüler kripto exchange'lerden biri olup güçlü WebSocket API'si sunar.

### Temel WebSocket Bağlantısı

```python
import asyncio
import websockets
import json
from datetime import datetime

async def binance_ticker_stream(symbol):
    """
    Binance ticker stream - fiyat güncellemelerini gerçek zamanlı alır
    """
    # WebSocket endpoint (lowercase sembol gerekli)
    uri = f"wss://stream.binance.com:9443/ws/{symbol.lower()}@ticker"
    
    try:
        async with websockets.connect(uri) as websocket:
            print(f"Connected to {symbol} ticker stream")
            
            while True:
                # Veri gelene kadar bekle (non-blocking)
                message = await websocket.recv()
                data = json.loads(message)
                
                # İlgili verileri parse et
                current_price = float(data['c'])  # Current price
                high_24h = float(data['h'])       # 24h high
                low_24h = float(data['l'])        # 24h low
                volume = float(data['v'])         # 24h volume
                timestamp = datetime.fromtimestamp(data['E'] / 1000)
                
                print(f"[{timestamp.strftime('%H:%M:%S')}] {symbol}")
                print(f"  Price: ${current_price:,.2f}")
                print(f"  24h Range: ${low_24h:,.2f} - ${high_24h:,.2f}")
                print(f"  Volume: {volume:,.2f}")
                print("-" * 50)
                
    except websockets.exceptions.ConnectionClosed:
        print(f"Connection closed for {symbol}")
    except Exception as e:
        print(f"Error: {e}")

# Çalıştır
asyncio.run(binance_ticker_stream("BTCUSDT"))
```
{: file="binance_ticker.py" }

### Çoklu Stream Yönetimi

```python
import asyncio
import websockets
import json

class BinanceMultiStream:
    """
    Birden fazla sembolü aynı anda izleyen WebSocket manager
    """
    def __init__(self, symbols):
        self.symbols = symbols
        self.prices = {}  # Symbol -> price mapping
        
    async def handle_stream(self, symbol):
        """Her sembol için ayrı stream handler"""
        uri = f"wss://stream.binance.com:9443/ws/{symbol.lower()}@ticker"
        
        while True:  # Auto-reconnect loop
            try:
                async with websockets.connect(uri) as websocket:
                    print(f"Connected: {symbol}")
                    
                    while True:
                        message = await websocket.recv()
                        data = json.loads(message)
                        
                        # Price'ı güncelle
                        self.prices[symbol] = {
                            'price': float(data['c']),
                            'change_24h': float(data['P']),  # % change
                            'timestamp': data['E']
                        }
                        
            except Exception as e:
                print(f"{symbol} disconnected: {e}")
                print("Reconnecting in 5 seconds...")
                await asyncio.sleep(5)
    
    async def print_dashboard(self):
        """Fiyatları dashboard olarak göster"""
        while True:
            await asyncio.sleep(2)  # Her 2 saniyede güncelle
            
            print("\n" + "="*60)
            print(f"{'Symbol':<12} {'Price':>15} {'24h Change':>15}")
            print("="*60)
            
            for symbol, data in sorted(self.prices.items()):
                price = data['price']
                change = data['change_24h']
                change_color = "🟢" if change > 0 else "🔴"
                
                print(f"{symbol:<12} ${price:>14,.2f} {change_color} {change:>8.2f}%")
    
    async def run(self):
        """Tüm stream'leri ve dashboard'u başlat"""
        tasks = []
        
        # Her sembol için stream task'i oluştur
        for symbol in self.symbols:
            task = asyncio.create_task(self.handle_stream(symbol))
            tasks.append(task)
        
        # Dashboard task'i ekle
        dashboard_task = asyncio.create_task(self.print_dashboard())
        tasks.append(dashboard_task)
        
        # Tüm task'leri paralel çalıştır
        await asyncio.gather(*tasks)

# Kullanım
async def main():
    symbols = ["BTCUSDT", "ETHUSDT", "SOLUSDT", "BNBUSDT", "ADAUSDT"]
    streamer = BinanceMultiStream(symbols)
    await streamer.run()

asyncio.run(main())
```

![Order Book Depth Chart Visualization](/assets/img/posts/order-book-depth-chart-visualization.png)
*Şekil 2: Order Book Depth Chart - Bid/ask spread ve likidite seviyelerini görselleştirme*

## Order Book Stream ve Real-Time Analiz

Order book (emir defteri), bir piyasanın likiditesini ve derinliğini gösterir. Real-time order book tracking, HFT stratejileri için kritiktir.

### Order Book Yapısı

```python
import asyncio
import websockets
import json
from collections import OrderedDict
from decimal import Decimal

class OrderBook:
    """
    Gerçek zamanlı order book yönetimi
    """
    def __init__(self, symbol, max_depth=20):
        self.symbol = symbol
        self.max_depth = max_depth
        
        # OrderedDict kullanarak fiyat seviyelerini sıralı tut
        self.bids = OrderedDict()  # {price: quantity}
        self.asks = OrderedDict()
        
        self.last_update_id = 0
        
    def update(self, bids, asks, update_id):
        """
        Order book güncelleme
        bids/asks: [[price, quantity], ...]
        """
        self.last_update_id = update_id
        
        # Bid updates
        for price_str, qty_str in bids:
            price = Decimal(price_str)
            qty = Decimal(qty_str)
            
            if qty == 0:
                # Quantity 0 ise seviyeyi kaldır
                self.bids.pop(price, None)
            else:
                self.bids[price] = qty
        
        # Ask updates
        for price_str, qty_str in asks:
            price = Decimal(price_str)
            qty = Decimal(qty_str)
            
            if qty == 0:
                self.asks.pop(price, None)
            else:
                self.asks[price] = qty
        
        # Bid'leri fiyata göre sırala (yüksekten düşüğe)
        self.bids = OrderedDict(
            sorted(self.bids.items(), key=lambda x: x[0], reverse=True)[:self.max_depth]
        )
        
        # Ask'leri fiyata göre sırala (düşükten yükseğe)
        self.asks = OrderedDict(
            sorted(self.asks.items(), key=lambda x: x[0])[:self.max_depth]
        )
    
    def get_best_bid(self):
        """En yüksek alış fiyatı"""
        if self.bids:
            return next(iter(self.bids.items()))
        return None, None
    
    def get_best_ask(self):
        """En düşük satış fiyatı"""
        if self.asks:
            return next(iter(self.asks.items()))
        return None, None
    
    def get_spread(self):
        """Bid-ask spread hesapla"""
        best_bid, _ = self.get_best_bid()
        best_ask, _ = self.get_best_ask()
        
        if best_bid and best_ask:
            spread = best_ask - best_bid
            spread_pct = (spread / best_ask) * 100
            return float(spread), float(spread_pct)
        return None, None
    
    def get_mid_price(self):
        """Orta fiyat (bid + ask) / 2"""
        best_bid, _ = self.get_best_bid()
        best_ask, _ = self.get_best_ask()
        
        if best_bid and best_ask:
            return float((best_bid + best_ask) / 2)
        return None
    
    def calculate_depth(self, side, price_range_pct=1.0):
        """
        Belirli fiyat aralığındaki toplam likidite
        side: 'bid' veya 'ask'
        price_range_pct: % kaç aralıkta derinlik hesaplansın
        """
        mid_price = self.get_mid_price()
        if not mid_price:
            return 0
        
        orders = self.bids if side == 'bid' else self.asks
        total_qty = Decimal(0)
        
        for price, qty in orders.items():
            # Fiyat aralığı kontrolü
            price_diff_pct = abs(float(price) - mid_price) / mid_price * 100
            
            if price_diff_pct <= price_range_pct:
                total_qty += qty
        
        return float(total_qty)
    
    def display(self):
        """Order book'u terminal'de göster"""
        print(f"\n{'='*70}")
        print(f"Order Book: {self.symbol} (Update ID: {self.last_update_id})")
        print(f"{'='*70}")
        
        # Spread bilgisi
        spread, spread_pct = self.get_spread()
        mid_price = self.get_mid_price()
        
        if spread:
            print(f"Mid Price: ${mid_price:,.4f} | Spread: ${spread:.4f} ({spread_pct:.3f}%)")
            print(f"{'-'*70}")
        
        # Ask tarafı (ters sırada göster - yüksek fiyatlar üstte)
        print(f"{'ASKS (Sell Orders)':^70}")
        print(f"{'Price':>20} | {'Quantity':>15} | {'Total':>15}")
        print(f"{'-'*70}")
        
        asks_list = list(reversed(list(self.asks.items())[:10]))
        for price, qty in asks_list:
            total = float(price) * float(qty)
            print(f"${float(price):>19,.4f} | {float(qty):>15,.4f} | ${total:>14,.2f}")
        
        print(f"\n{'─'*70}\n")
        
        # Bid tarafı
        print(f"{'BIDS (Buy Orders)':^70}")
        print(f"{'Price':>20} | {'Quantity':>15} | {'Total':>15}")
        print(f"{'-'*70}")
        
        for price, qty in list(self.bids.items())[:10]:
            total = float(price) * float(qty)
            print(f"${float(price):>19,.4f} | {float(qty):>15,.4f} | ${total:>14,.2f}")
        
        # Derinlik analizi
        bid_depth = self.calculate_depth('bid', 0.5)
        ask_depth = self.calculate_depth('ask', 0.5)
        
        print(f"\n{'Depth Analysis (±0.5%):':^70}")
        print(f"Bid Depth: {bid_depth:,.2f} | Ask Depth: {ask_depth:,.2f}")
        print(f"{'='*70}\n")

async def binance_orderbook_stream(symbol):
    """Binance order book stream"""
    uri = f"wss://stream.binance.com:9443/ws/{symbol.lower()}@depth20@100ms"
    
    order_book = OrderBook(symbol, max_depth=20)
    
    async with websockets.connect(uri) as websocket:
        print(f"✅ Connected to {symbol} order book stream")
        
        while True:
            message = await websocket.recv()
            data = json.loads(message)
            
            # Order book'u güncelle
            order_book.update(
                bids=data['bids'],
                asks=data['asks'],
                update_id=data['lastUpdateId']
            )
            
            # Her güncellemeyi göster (100ms'de bir)
            order_book.display()
            
            await asyncio.sleep(1)  # Display throttle (çok hızlı scroll'u önle)

# Çalıştır
asyncio.run(binance_orderbook_stream("BTCUSDT"))
```

## Trading Bot Mimarisi

Şimdi tüm bileşenleri bir araya getirerek production-ready bir trading bot oluşturalım.

### Modüler Bot Yapısı

```python
import asyncio
import websockets
import json
import logging
from datetime import datetime
from decimal import Decimal
from typing import Dict, List, Callable
from dataclasses import dataclass

# Logging yapılandırması
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)
logger = logging.getLogger(__name__)

@dataclass
class Trade:
    """Trade sinyali"""
    symbol: str
    side: str  # 'BUY' or 'SELL'
    price: float
    quantity: float
    timestamp: datetime
    reason: str

class TradingStrategy:
    """
    Base trading strategy class
    """
    def __init__(self, name: str):
        self.name = name
        
    async def analyze(self, order_book, ticker_data) -> Trade:
        """
        Strateji analizi - override edilmeli
        Returns: Trade object ya da None
        """
        raise NotImplementedError

class SpreadStrategy(TradingStrategy):
    """
    Spread-based strategy: Spread belirli bir eşiğin altına düştüğünde al
    """
    def __init__(self, spread_threshold_pct=0.05):
        super().__init__("SpreadStrategy")
        self.spread_threshold_pct = spread_threshold_pct
        
    async def analyze(self, order_book, ticker_data) -> Trade:
        """Spread analizi yap"""
        spread, spread_pct = order_book.get_spread()
        
        if spread_pct and spread_pct < self.spread_threshold_pct:
            # Dar spread - likidite yüksek, iyi al fırsatı
            best_ask, ask_qty = order_book.get_best_ask()
            
            if best_ask and ask_qty:
                logger.info(f"🎯 {self.name}: Narrow spread detected ({spread_pct:.3f}%)")
                
                return Trade(
                    symbol=order_book.symbol,
                    side='BUY',
                    price=float(best_ask),
                    quantity=min(float(ask_qty), 0.01),  # Max 0.01 BTC
                    timestamp=datetime.now(),
                    reason=f"Spread {spread_pct:.3f}% < {self.spread_threshold_pct}%"
                )
        
        return None

class ImbalanceStrategy(TradingStrategy):
    """
    Order book imbalance strategy:
    Bid side çok güçlüyse fiyat yükselir (BUY)
    Ask side çok güçlüyse fiyat düşer (SELL)
    """
    def __init__(self, imbalance_threshold=2.0):
        super().__init__("ImbalanceStrategy")
        self.imbalance_threshold = imbalance_threshold
        
    async def analyze(self, order_book, ticker_data) -> Trade:
        """Order book dengesizliğini analiz et"""
        bid_depth = order_book.calculate_depth('bid', 0.5)
        ask_depth = order_book.calculate_depth('ask', 0.5)
        
        if ask_depth == 0:
            return None
        
        # Bid/Ask ratio
        imbalance_ratio = bid_depth / ask_depth
        
        best_bid, bid_qty = order_book.get_best_bid()
        best_ask, ask_qty = order_book.get_best_ask()
        
        if imbalance_ratio > self.imbalance_threshold:
            # Çok fazla bid pressure - Fiyat yükselecek
            logger.info(f"🎯 {self.name}: Strong bid pressure (ratio: {imbalance_ratio:.2f})")
            
            return Trade(
                symbol=order_book.symbol,
                side='BUY',
                price=float(best_ask),
                quantity=0.001,
                timestamp=datetime.now(),
                reason=f"Bid imbalance {imbalance_ratio:.2f}x"
            )
        
        elif imbalance_ratio < (1 / self.imbalance_threshold):
            # Çok fazla ask pressure - Fiyat düşecek
            logger.info(f"🎯 {self.name}: Strong ask pressure (ratio: {imbalance_ratio:.2f})")
            
            return Trade(
                symbol=order_book.symbol,
                side='SELL',
                price=float(best_bid),
                quantity=0.001,
                timestamp=datetime.now(),
                reason=f"Ask imbalance {imbalance_ratio:.2f}x"
            )
        
        return None

class WebSocketTradingBot:
    """
    Ana trading bot class'ı
    """
    def __init__(self, symbol: str, strategies: List[TradingStrategy]):
        self.symbol = symbol
        self.strategies = strategies
        self.order_book = OrderBook(symbol)
        self.ticker_data = {}
        
        # Performance metrics
        self.message_count = 0
        self.start_time = datetime.now()
        self.trades_signaled = []
        
    async def handle_orderbook_stream(self):
        """Order book stream handler"""
        uri = f"wss://stream.binance.com:9443/ws/{self.symbol.lower()}@depth20@100ms"
        
        while True:
            try:
                async with websockets.connect(uri) as websocket:
                    logger.info(f"✅ Connected to {self.symbol} order book stream")
                    
                    while True:
                        message = await websocket.recv()
                        data = json.loads(message)
                        
                        # Order book güncelle
                        self.order_book.update(
                            bids=data['bids'],
                            asks=data['asks'],
                            update_id=data['lastUpdateId']
                        )
                        
                        self.message_count += 1
                        
                        # Stratejileri çalıştır
                        await self.run_strategies()
                        
            except Exception as e:
                logger.error(f"❌ Order book stream error: {e}")
                logger.info("🔄 Reconnecting in 5 seconds...")
                await asyncio.sleep(5)
    
    async def handle_ticker_stream(self):
        """Ticker stream handler - fiyat ve volume bilgisi"""
        uri = f"wss://stream.binance.com:9443/ws/{self.symbol.lower()}@ticker"
        
        while True:
            try:
                async with websockets.connect(uri) as websocket:
                    logger.info(f"✅ Connected to {self.symbol} ticker stream")
                    
                    while True:
                        message = await websocket.recv()
                        data = json.loads(message)
                        
                        # Ticker data güncelle
                        self.ticker_data = {
                            'price': float(data['c']),
                            'volume_24h': float(data['v']),
                            'high_24h': float(data['h']),
                            'low_24h': float(data['l']),
                            'change_24h_pct': float(data['P'])
                        }
                        
            except Exception as e:
                logger.error(f"❌ Ticker stream error: {e}")
                logger.info("🔄 Reconnecting in 5 seconds...")
                await asyncio.sleep(5)
    
    async def run_strategies(self):
        """Tüm stratejileri paralel çalıştır"""
        tasks = []
        
        for strategy in self.strategies:
            task = strategy.analyze(self.order_book, self.ticker_data)
            tasks.append(task)
        
        # Tüm stratejileri paralel çalıştır
        results = await asyncio.gather(*tasks)
        
        # Trade sinyallerini işle
        for trade in results:
            if trade:
                await self.execute_trade(trade)
    
    async def execute_trade(self, trade: Trade):
        """
        Trade execution (simülasyon)
        Production'da buraya Binance API order placement gelir
        """
        logger.info(f"\n{'='*70}")
        logger.info(f"🚀 TRADE SIGNAL GENERATED")
        logger.info(f"{'='*70}")
        logger.info(f"Symbol: {trade.symbol}")
        logger.info(f"Side: {trade.side}")
        logger.info(f"Price: ${trade.price:,.4f}")
        logger.info(f"Quantity: {trade.quantity}")
        logger.info(f"Reason: {trade.reason}")
        logger.info(f"Timestamp: {trade.timestamp}")
        logger.info(f"{'='*70}\n")
        
        self.trades_signaled.append(trade)
        
        # Gerçek execution için:
        # await self.binance_api.create_order(
        #     symbol=trade.symbol,
        #     side=trade.side,
        #     type='LIMIT',
        #     price=trade.price,
        #     quantity=trade.quantity
        # )
    
    async def performance_monitor(self):
        """Performance monitoring task"""
        while True:
            await asyncio.sleep(30)  # Her 30 saniyede rapor
            
            uptime = (datetime.now() - self.start_time).total_seconds()
            messages_per_sec = self.message_count / uptime if uptime > 0 else 0
            
            logger.info(f"\n{'─'*70}")
            logger.info(f"📊 PERFORMANCE METRICS")
            logger.info(f"{'─'*70}")
            logger.info(f"Uptime: {uptime:.0f}s")
            logger.info(f"Messages Processed: {self.message_count}")
            logger.info(f"Throughput: {messages_per_sec:.2f} msg/sec")
            logger.info(f"Trades Signaled: {len(self.trades_signaled)}")
            logger.info(f"{'─'*70}\n")
    
    async def run(self):
        """Bot'u başlat"""
        logger.info(f"🤖 Starting WebSocket Trading Bot for {self.symbol}")
        logger.info(f"📈 Strategies: {[s.name for s in self.strategies]}")
        
        # Tüm task'leri paralel çalıştır
        await asyncio.gather(
            self.handle_orderbook_stream(),
            self.handle_ticker_stream(),
            self.performance_monitor()
        )

# Bot'u çalıştır
async def main():
    # Stratejileri tanımla
    strategies = [
        SpreadStrategy(spread_threshold_pct=0.05),
        ImbalanceStrategy(imbalance_threshold=2.5)
    ]
    
    # Bot'u oluştur ve çalıştır
    bot = WebSocketTradingBot("BTCUSDT", strategies)
    await bot.run()

if __name__ == "__main__":
    try:
        asyncio.run(main())
    except KeyboardInterrupt:
        logger.info("\n👋 Bot stopped by user")
```

![HFT Low Latency Trading Architecture](/assets/img/posts/hft-low-latency-trading-architecture.png)
*Şekil 3: High-Frequency Trading Architecture - Ultra-low latency için optimize edilmiş sistem mimarisi*

## Latency Optimizasyonu

HFT'de her milisaniye önemlidir. İşte latency'yi minimize etmek için kritik teknikler:

### 1. Connection Pooling ve Reuse

```python
import asyncio
import websockets
from typing import Dict

class WebSocketPool:
    """
    WebSocket connection pool - connection reuse
    """
    def __init__(self):
        self.connections: Dict[str, websockets.WebSocketClientProtocol] = {}
        self.locks: Dict[str, asyncio.Lock] = {}
    
    async def get_connection(self, uri: str):
        """Get or create connection"""
        if uri not in self.connections:
            self.locks[uri] = asyncio.Lock()
            
            async with self.locks[uri]:
                if uri not in self.connections:
                    self.connections[uri] = await websockets.connect(
                        uri,
                        ping_interval=20,  # Keep-alive
                        ping_timeout=10,
                        close_timeout=5
                    )
        
        return self.connections[uri]
    
    async def close_all(self):
        """Tüm bağlantıları kapat"""
        for conn in self.connections.values():
            await conn.close()

# Global pool
ws_pool = WebSocketPool()

async def optimized_stream(symbol):
    uri = f"wss://stream.binance.com:9443/ws/{symbol.lower()}@ticker"
    
    # Pool'dan connection al
    websocket = await ws_pool.get_connection(uri)
    
    while True:
        message = await websocket.recv()
        # Process message...
```

### 2. Message Batching

```python
import asyncio
from collections import deque

class MessageBatcher:
    """
    Message'ları batch'leyerek işleme - throughput artışı
    """
    def __init__(self, batch_size=100, batch_timeout=0.1):
        self.batch_size = batch_size
        self.batch_timeout = batch_timeout
        self.queue = deque()
        
    async def add_message(self, message):
        """Queue'ya message ekle"""
        self.queue.append(message)
        
        # Batch size'a ulaşıldıysa hemen işle
        if len(self.queue) >= self.batch_size:
            await self.process_batch()
    
    async def process_batch(self):
        """Batch'i işle"""
        if not self.queue:
            return
        
        batch = []
        while self.queue and len(batch) < self.batch_size:
            batch.append(self.queue.popleft())
        
        # Batch processing (vectorized operations)
        # Bu örnekte sadece sayıyoruz
        print(f"Processed batch of {len(batch)} messages")
        
        # Gerçek uygulamada:
        # - DataFrame'e çevir
        # - Vectorized hesaplamalar yap
        # - Bulk database insert
    
    async def batch_worker(self):
        """Background worker - timeout'a göre batch işle"""
        while True:
            await asyncio.sleep(self.batch_timeout)
            await self.process_batch()

# Kullanım
batcher = MessageBatcher(batch_size=100, batch_timeout=0.1)

async def main():
    # Worker'ı başlat
    asyncio.create_task(batcher.batch_worker())
    
    # Message'ları gönder
    for i in range(1000):
        await batcher.add_message(f"Message {i}")
        await asyncio.sleep(0.001)  # 1ms delay

asyncio.run(main())
```

### 3. CPU Affinity ve Process Priority

```python
import os
import psutil

def optimize_process():
    """
    Process'i optimize et - Linux/Unix için
    """
    try:
        process = psutil.Process(os.getpid())
        
        # High priority ayarla
        if os.name == 'posix':  # Linux/Mac
            os.nice(-10)  # Negative = higher priority
        
        # CPU affinity - belirli core'lara pin'le
        # Core 0-3'ü kullan (isolated cores ideal)
        process.cpu_affinity([0, 1, 2, 3])
        
        print(f"✅ Process optimized: PID {os.getpid()}")
        print(f"   Priority: {process.nice()}")
        print(f"   CPU Affinity: {process.cpu_affinity()}")
        
    except Exception as e:
        print(f"⚠️ Could not optimize process: {e}")

# Bot başlangıcında çağır
optimize_process()
```

### 4. uvloop - Faster Event Loop

```python
# uvloop: libuv tabanlı, asyncio'dan %2-4x daha hızlı
# pip install uvloop

import asyncio
import uvloop

# uvloop'u default event loop olarak ayarla
asyncio.set_event_loop_policy(uvloop.EventLoopPolicy())

async def main():
    # Artık tüm async code uvloop kullanıyor
    # Latency: ~10-50% azalma
    pass

asyncio.run(main())
```

## Error Handling ve Resilience

Production sistemlerde robust error handling şarttır.

### Exponential Backoff ile Reconnection

```python
import asyncio
import websockets
import random

async def resilient_websocket_stream(uri, max_retries=float('inf')):
    """
    Exponential backoff ile auto-reconnect
    """
    retry_count = 0
    base_delay = 1  # 1 saniye
    max_delay = 60  # Max 60 saniye
    
    while retry_count < max_retries:
        try:
            async with websockets.connect(uri) as websocket:
                logger.info(f"✅ Connected (attempt {retry_count + 1})")
                retry_count = 0  # Başarılı bağlantı - counter'ı sıfırla
                
                while True:
                    try:
                        message = await asyncio.wait_for(
                            websocket.recv(),
                            timeout=30.0  # 30s timeout
                        )
                        
                        # Process message
                        yield message
                        
                    except asyncio.TimeoutError:
                        # Ping gönder - connection alive mı?
                        pong = await websocket.ping()
                        await asyncio.wait_for(pong, timeout=10)
                        logger.debug("🏓 Ping successful")
                        
        except (websockets.exceptions.ConnectionClosed,
                websockets.exceptions.WebSocketException,
                asyncio.TimeoutError) as e:
            
            retry_count += 1
            
            # Exponential backoff hesapla
            delay = min(base_delay * (2 ** retry_count) + random.uniform(0, 1), max_delay)
            
            logger.warning(f"⚠️ Connection lost: {e}")
            logger.info(f"🔄 Retry {retry_count} in {delay:.1f}s...")
            
            await asyncio.sleep(delay)
        
        except Exception as e:
            logger.error(f"❌ Unexpected error: {e}")
            raise

# Kullanım
async def main():
    uri = "wss://stream.binance.com:9443/ws/btcusdt@ticker"
    
    async for message in resilient_websocket_stream(uri):
        print(f"Received: {message[:50]}...")

asyncio.run(main())
```

### Circuit Breaker Pattern

```python
from enum import Enum
from datetime import datetime, timedelta

class CircuitState(Enum):
    CLOSED = "closed"    # Normal operation
    OPEN = "open"        # Failures detected, blocking requests
    HALF_OPEN = "half_open"  # Testing if service recovered

class CircuitBreaker:
    """
    Circuit breaker pattern - sürekli fail eden servisleri koru
    """
    def __init__(self, failure_threshold=5, timeout=60, success_threshold=2):
        self.failure_threshold = failure_threshold
        self.timeout = timeout  # seconds
        self.success_threshold = success_threshold
        
        self.failure_count = 0
        self.success_count = 0
        self.last_failure_time = None
        self.state = CircuitState.CLOSED
    
    async def call(self, func, *args, **kwargs):
        """Protected function call"""
        
        if self.state == CircuitState.OPEN:
            # Circuit açık - timeout geçti mi?
            if self.last_failure_time:
                elapsed = (datetime.now() - self.last_failure_time).total_seconds()
                
                if elapsed > self.timeout:
                    logger.info("🔄 Circuit breaker: HALF_OPEN (testing recovery)")
                    self.state = CircuitState.HALF_OPEN
                else:
                    raise Exception(f"Circuit breaker OPEN - retry in {self.timeout - elapsed:.0f}s")
        
        try:
            # Function'ı çağır
            result = await func(*args, **kwargs)
            
            # Success
            self.on_success()
            return result
            
        except Exception as e:
            # Failure
            self.on_failure()
            raise e
    
    def on_success(self):
        """Başarılı çağrı"""
        self.failure_count = 0
        
        if self.state == CircuitState.HALF_OPEN:
            self.success_count += 1
            
            if self.success_count >= self.success_threshold:
                logger.info("✅ Circuit breaker: CLOSED (service recovered)")
                self.state = CircuitState.CLOSED
                self.success_count = 0
    
    def on_failure(self):
        """Başarısız çağrı"""
        self.failure_count += 1
        self.last_failure_time = datetime.now()
        
        if self.failure_count >= self.failure_threshold:
            logger.warning(f"⚠️ Circuit breaker: OPEN (failure threshold reached)")
            self.state = CircuitState.OPEN

# Kullanım
circuit_breaker = CircuitBreaker(failure_threshold=3, timeout=30)

async def unreliable_api_call():
    """Bazen fail eden API call"""
    import random
    if random.random() < 0.3:  # %30 fail
        raise Exception("API Error")
    return "Success"

async def main():
    for i in range(20):
        try:
            result = await circuit_breaker.call(unreliable_api_call)
            print(f"✅ Call {i}: {result}")
        except Exception as e:
            print(f"❌ Call {i}: {e}")
        
        await asyncio.sleep(1)

asyncio.run(main())
```

## Production Deployment Best Practices

### 1. Docker Container

```dockerfile
# Dockerfile
FROM python:3.11-slim

WORKDIR /app

# Dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Bot kodu
COPY . .

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
  CMD python healthcheck.py || exit 1

# Run
CMD ["python", "trading_bot.py"]
```

**requirements.txt:**
```
websockets==12.0
aiohttp==3.9.1
uvloop==0.19.0
python-binance==1.0.19
pandas==2.1.4
numpy==1.26.2
```

### 2. Environment Configuration

```python
# config.py
import os
from dataclasses import dataclass

@dataclass
class Config:
    # Exchange
    EXCHANGE: str = os.getenv("EXCHANGE", "binance")
    SYMBOLS: list = os.getenv("SYMBOLS", "BTCUSDT,ETHUSDT").split(",")
    
    # Trading
    MAX_POSITION_SIZE: float = float(os.getenv("MAX_POSITION_SIZE", "0.1"))
    SPREAD_THRESHOLD: float = float(os.getenv("SPREAD_THRESHOLD", "0.05"))
    
    # API Keys (production'da secrets manager kullan!)
    API_KEY: str = os.getenv("BINANCE_API_KEY", "")
    API_SECRET: str = os.getenv("BINANCE_API_SECRET", "")
    
    # Monitoring
    LOG_LEVEL: str = os.getenv("LOG_LEVEL", "INFO")
    METRICS_PORT: int = int(os.getenv("METRICS_PORT", "9090"))

config = Config()
```

### 3. Monitoring ve Alerting

```python
from prometheus_client import Counter, Histogram, Gauge, start_http_server
import time

# Prometheus metrics
messages_received = Counter('websocket_messages_received_total', 'Total messages received')
trades_executed = Counter('trades_executed_total', 'Total trades executed', ['side'])
latency = Histogram('message_processing_latency_seconds', 'Message processing latency')
connection_status = Gauge('websocket_connection_status', 'WebSocket connection status', ['symbol'])

class MonitoredTradingBot(WebSocketTradingBot):
    """Monitoring ile enhanced bot"""
    
    async def handle_orderbook_stream(self):
        """Monitored order book stream"""
        uri = f"wss://stream.binance.com:9443/ws/{self.symbol.lower()}@depth20@100ms"
        
        while True:
            try:
                async with websockets.connect(uri) as websocket:
                    connection_status.labels(symbol=self.symbol).set(1)
                    
                    while True:
                        start_time = time.time()
                        
                        message = await websocket.recv()
                        data = json.loads(message)
                        
                        # Update order book
                        self.order_book.update(
                            bids=data['bids'],
                            asks=data['asks'],
                            update_id=data['lastUpdateId']
                        )
                        
                        # Metrics
                        messages_received.inc()
                        latency.observe(time.time() - start_time)
                        
                        await self.run_strategies()
                        
            except Exception as e:
                connection_status.labels(symbol=self.symbol).set(0)
                logger.error(f"Stream error: {e}")
                await asyncio.sleep(5)
    
    async def execute_trade(self, trade: Trade):
        """Monitored trade execution"""
        await super().execute_trade(trade)
        
        # Metric
        trades_executed.labels(side=trade.side).inc()

# Prometheus HTTP server başlat
start_http_server(9090)
```

## Gerçek Dünya Örneği: Market Making Bot

Market making stratejisi uygulayan tam bir bot örneği:

```python
import asyncio
import websockets
import json
from decimal import Decimal

class MarketMakingBot:
    """
    Simple market making bot:
    - Bid ve ask tarafına limit order koy
    - Spread'den profit yap
    """
    def __init__(self, symbol, spread_pct=0.1, order_size=0.001):
        self.symbol = symbol
        self.spread_pct = spread_pct  # %0.1 spread
        self.order_size = order_size
        
        self.mid_price = None
        self.our_bid_price = None
        self.our_ask_price = None
        
    async def handle_ticker_stream(self):
        """Fiyat güncellemelerini takip et"""
        uri = f"wss://stream.binance.com:9443/ws/{self.symbol.lower()}@ticker"
        
        async with websockets.connect(uri) as websocket:
            while True:
                message = await websocket.recv()
                data = json.loads(message)
                
                # Bid ve ask'ten mid price hesapla
                bid = Decimal(data['b'])
                ask = Decimal(data['a'])
                self.mid_price = (bid + ask) / 2
                
                # Quote'larımızı güncelle
                await self.update_quotes()
    
    async def update_quotes(self):
        """Bid ve ask quote'larımızı güncelle"""
        if not self.mid_price:
            return
        
        spread_offset = self.mid_price * Decimal(str(self.spread_pct / 100))
        
        # Bizim fiyatlarımız
        new_bid_price = self.mid_price - spread_offset
        new_ask_price = self.mid_price + spread_offset
        
        # Fiyat değiştiyse order'ları güncelle
        if new_bid_price != self.our_bid_price or new_ask_price != self.our_ask_price:
            self.our_bid_price = new_bid_price
            self.our_ask_price = new_ask_price
            
            print(f"\n{'='*60}")
            print(f"📊 Market Making Quotes Updated")
            print(f"{'='*60}")
            print(f"Mid Price: ${self.mid_price:.2f}")
            print(f"Our Bid:   ${self.our_bid_price:.2f} (quantity: {self.order_size})")
            print(f"Our Ask:   ${self.our_ask_price:.2f} (quantity: {self.order_size})")
            print(f"Spread:    {self.spread_pct}% (${spread_offset:.2f})")
            print(f"{'='*60}\n")
            
            # Production'da buraya order placement gelir:
            # await self.cancel_all_orders()
            # await self.place_order('BUY', self.our_bid_price, self.order_size)
            # await self.place_order('SELL', self.our_ask_price, self.order_size)
    
    async def run(self):
        """Bot'u çalıştır"""
        await self.handle_ticker_stream()

# Çalıştır
async def main():
    bot = MarketMakingBot("BTCUSDT", spread_pct=0.1, order_size=0.001)
    await bot.run()

asyncio.run(main())
```

## Sonuç

Bu yazıda Python ile WebSocket tabanlı gerçek zamanlı trading bot geliştirmenin tüm detaylarını ele aldık:

### Öğrendiklerimiz

1. **WebSocket Temelleri**: REST API'ye göre 100x daha düşük latency
2. **AsyncIO Mastery**: Event loop, coroutines, concurrent programming
3. **Exchange Integration**: Binance WebSocket API'leri ile real-time veri
4. **Order Book Analysis**: Bid/ask spread, depth, imbalance stratejileri
5. **Latency Optimization**: uvloop, connection pooling, CPU affinity
6. **Production Readiness**: Error handling, monitoring, deployment

### Önemli Noktalar

- **Test Et**: Canlıya geçmeden önce testnet'te kapsamlı test yap
- **Risk Yönetimi**: Position size limitleri, stop-loss mekanizmaları ekle
- **Monitoring**: Prometheus, Grafana ile sürekli izle
- **Backtesting**: Geçmiş verilerde stratejiyi test et
- **Latency Kritik**: HFT'de her milisaniye fark yaratır

### İleri Seviye Konular

Bu yazının devamında şunları öğrenebilirsiniz:

- Multi-exchange arbitrage botları
- Machine learning entegrasyonu
- Options ve futures trading
- Portfolio optimization
- Risk management sistemleri

### Kaynaklar

- [Binance WebSocket API Documentation](https://binance-docs.github.io/apidocs/spot/en/#websocket-market-streams)
- [Python AsyncIO Documentation](https://docs.python.org/3/library/asyncio.html)
- [uvloop - Ultra fast asyncio event loop](https://github.com/MagicStack/uvloop)
- [websockets library](https://websockets.readthedocs.io/)

Happy trading! 🚀📈
