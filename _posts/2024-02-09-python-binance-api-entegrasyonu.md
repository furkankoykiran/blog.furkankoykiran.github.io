---
title: "Python ile Binance API Entegrasyonu"
description: "Python ile Binance API kullanarak otomatik trading botları geliştirme rehberi. REST API, WebSocket entegrasyonu, order yönetimi ve risk stratejileri."
date: "2024-02-09 13:00:00 +0300"
categories: [Exchange Integration, Python]
tags: [python, binance, api, trading, kripto, exchange, websocket, automation]
image:
  path: /assets/img/posts/python-automated-trading-architecture.png
  alt: "Python ile Binance API Entegrasyonu"
---

## Giriş

Binance, dünyanın en büyük kripto para borsalarından biri olarak günlük milyarlarca dolarlık işlem hacmine sahiptir. Python ile Binance API entegrasyonu, otomatik trading botları geliştirmek, portföy yönetimi yapmak ve gerçek zamanlı piyasa verilerini analiz etmek isteyen geliştiriciler için güçlü bir araçtır.

Bu kapsamlı rehberde, Binance API'sini Python ile nasıl kullanacağınızı sıfırdan öğreneceksiniz. REST API ile temel işlemler yapmayı, WebSocket ile gerçek zamanlı veri akışını dinlemeyi, order placement stratejilerini ve güvenli API key yönetimini detaylıca ele alacağız. Ayrıca, production-ready trading botları geliştirmek için gereken best practice'leri ve error handling mekanizmalarını da göreceğiz.

![AI Trading Architecture](/assets/img/posts/ai-stock-trading-visualization.jpg){: w="700" h="400" .shadow }
_Şekil 1: Kripto trading sistemi mimarisi genel görünümü_

## Binance API'ye Giriş

### API Türleri

Binance üç farklı API türü sunar:

1. **REST API**: HTTP istekleri ile trading, account bilgileri ve market data için kullanılır
2. **WebSocket API**: Gerçek zamanlı veri akışları (fiyatlar, order book, trades)
3. **WebSocket Streams**: Market data ve user data stream'leri

### API Limitleri

Binance, API kullanımında belirli limitler koyar:

- **Request Weight**: Her endpoint'in bir weight değeri var (1-10 arası)
- **Raw Requests**: Dakikada maksimum 1200 raw request
- **Order Limits**: 10 saniyede 100 order, günde 200.000 order
- **WebSocket**: Açık connection sayısı ve subscription limitleri

### Güvenlik Seviyeleri

- **Public Endpoints**: API key gerektirmez (market data)
- **TRADE Endpoints**: API key + SECRET key gerektirir (trading işlemleri)
- **USER_DATA Endpoints**: API key + SECRET key gerektirir (account bilgileri)
- **MARGIN Endpoints**: Margin trading için özel izinler

## Kurulum ve Başlangıç

### Gerekli Kütüphanelerin Kurulumu

```bash
# Binance API wrapper'ı
pip install python-binance

# Alternatif: ccxt (multiple exchange support)
pip install ccxt

# Veri analizi için
pip install pandas numpy

# Async işlemler için
pip install asyncio aiohttp

# WebSocket için
pip install websockets

# Güvenlik için
pip install python-dotenv cryptography

# Logging ve monitoring
pip install loguru
```

### API Key Oluşturma

1. Binance hesabınıza giriş yapın
2. API Management sayfasına gidin
3. "Create API" butonuna tıklayın
4. API key'i güvenli bir yere kaydedin
5. İzinleri yapılandırın:
   - Enable Reading
   - Enable Spot & Margin Trading
   - IP whitelist ekleyin (önerilir)

### Güvenli API Key Yönetimi

> API key'lerinizi asla public repository'ye eklemeyin! .env dosyasını mutlaka .gitignore'a ekleyin.
{: .prompt-warning }

```bash
BINANCE_API_KEY=your_api_key_here
BINANCE_SECRET_KEY=your_secret_key_here
```
{: file=".env" }

```python
# Python'dan okuyun
from dotenv import load_dotenv
import os

load_dotenv()

API_KEY = os.getenv('BINANCE_API_KEY')
SECRET_KEY = os.getenv('BINANCE_SECRET_KEY')
```

```bash
.env
*.key
secrets/
```
{: file=".gitignore" }

## REST API ile Temel İşlemler

### Client Oluşturma

```python
from binance.client import Client
from binance.exceptions import BinanceAPIException
import os
from dotenv import load_dotenv

# Environment variables'ı yükle
load_dotenv()

# Client oluştur
client = Client(
    api_key=os.getenv('BINANCE_API_KEY'),
    api_secret=os.getenv('BINANCE_SECRET_KEY')
)

# Testnet kullanımı (test için)
# client = Client(
#     api_key=os.getenv('TESTNET_API_KEY'),
#     api_secret=os.getenv('TESTNET_SECRET_KEY'),
#     testnet=True
# )

# Server zamanını kontrol et
def check_server_time():
    """
    Binance server zamanını kontrol et
    Local time ile sync olması önemli
    """
    try:
        server_time = client.get_server_time()
        print(f"Server Time: {server_time['serverTime']}")
        return True
    except BinanceAPIException as e:
        print(f"Error: {e}")
        return False

check_server_time()
```
{: file="binance_client.py" }

### Market Data Çekme

```python
import pandas as pd
from datetime import datetime, timedelta

def get_ticker_price(symbol='BTCUSDT'):
    """
    Belirli bir symbol için güncel fiyatı al
    
    Parameters:
    symbol (str): Trading pair (örn: 'BTCUSDT', 'ETHUSDT')
    
    Returns:
    dict: Fiyat bilgileri
    """
    try:
        ticker = client.get_symbol_ticker(symbol=symbol)
        print(f"{symbol} Price: ${float(ticker['price']):,.2f}")
        return ticker
    except BinanceAPIException as e:
        print(f"Error getting ticker: {e}")
        return None

def get_24h_stats(symbol='BTCUSDT'):
    """
    24 saatlik istatistikleri al
    """
    try:
        stats = client.get_ticker(symbol=symbol)
        
        print(f"\n24h Statistics for {symbol}:")
        print(f"High: ${float(stats['highPrice']):,.2f}")
        print(f"Low: ${float(stats['lowPrice']):,.2f}")
        print(f"Volume: {float(stats['volume']):,.2f}")
        print(f"Quote Volume: ${float(stats['quoteVolume']):,.2f}")
        print(f"Price Change: {float(stats['priceChangePercent']):.2f}%")
        
        return stats
    except BinanceAPIException as e:
        print(f"Error: {e}")
        return None

# Kullanım
get_ticker_price('BTCUSDT')
get_24h_stats('BTCUSDT')
```

### Historical Data (Klines) Çekme

```python
def get_historical_klines(
    symbol='BTCUSDT',
    interval='1h',
    lookback='30 days ago UTC'
):
    """
    Historical candlestick data çek
    
    Parameters:
    symbol (str): Trading pair
    interval (str): Zaman dilimi ('1m', '5m', '15m', '1h', '4h', '1d', etc.)
    lookback (str): Ne kadar geriye git
    
    Returns:
    pd.DataFrame: OHLCV verisi
    """
    try:
        # Klines verisi çek
        klines = client.get_historical_klines(
            symbol,
            interval,
            lookback
        )
        
        # DataFrame'e dönüştür
        df = pd.DataFrame(klines, columns=[
            'timestamp', 'open', 'high', 'low', 'close', 'volume',
            'close_time', 'quote_volume', 'trades', 
            'taker_buy_base', 'taker_buy_quote', 'ignore'
        ])
        
        # Veri tiplerini düzenle
        df['timestamp'] = pd.to_datetime(df['timestamp'], unit='ms')
        df['close_time'] = pd.to_datetime(df['close_time'], unit='ms')
        
        numeric_columns = ['open', 'high', 'low', 'close', 'volume']
        df[numeric_columns] = df[numeric_columns].astype(float)
        
        # Sadece gerekli kolonları tut
        df = df[['timestamp', 'open', 'high', 'low', 'close', 'volume']]
        df.set_index('timestamp', inplace=True)
        
        print(f"\nHistorical data çekildi: {len(df)} kayıt")
        print(df.head())
        
        return df
    
    except BinanceAPIException as e:
        print(f"Error: {e}")
        return None

# Örnek kullanım
btc_data = get_historical_klines('BTCUSDT', '1h', '7 days ago UTC')
eth_data = get_historical_klines('ETHUSDT', '15m', '24 hours ago UTC')
```

### Order Book Analizi

```python
def get_order_book(symbol='BTCUSDT', limit=20):
    """
    Order book (emir defteri) verisi al
    
    Parameters:
    symbol (str): Trading pair
    limit (int): Kaç seviye gösterilecek (5, 10, 20, 50, 100, 500, 1000, 5000)
    
    Returns:
    dict: Order book verisi
    """
    try:
        depth = client.get_order_book(symbol=symbol, limit=limit)
        
        # Bid ve ask verilerini DataFrame'e dönüştür
        bids_df = pd.DataFrame(depth['bids'], columns=['price', 'quantity'])
        asks_df = pd.DataFrame(depth['asks'], columns=['price', 'quantity'])
        
        bids_df = bids_df.astype(float)
        asks_df = asks_df.astype(float)
        
        # Kümülatif hacim hesapla
        bids_df['cumulative'] = bids_df['quantity'].cumsum()
        asks_df['cumulative'] = asks_df['quantity'].cumsum()
        
        print(f"\nOrder Book for {symbol}:")
        print("\nTop 5 Bids (Buy Orders):")
        print(bids_df.head())
        print("\nTop 5 Asks (Sell Orders):")
        print(asks_df.head())
        
        # Spread hesapla
        best_bid = bids_df['price'].iloc[0]
        best_ask = asks_df['price'].iloc[0]
        spread = best_ask - best_bid
        spread_percent = (spread / best_bid) * 100
        
        print(f"\nBest Bid: ${best_bid:.2f}")
        print(f"Best Ask: ${best_ask:.2f}")
        print(f"Spread: ${spread:.2f} ({spread_percent:.3f}%)")
        
        return {
            'bids': bids_df,
            'asks': asks_df,
            'spread': spread,
            'spread_percent': spread_percent
        }
    
    except BinanceAPIException as e:
        print(f"Error: {e}")
        return None

# Kullanım
order_book = get_order_book('BTCUSDT', limit=20)
```

## Account Bilgileri ve Bakiye Yönetimi

```python
def get_account_info():
    """
    Account bilgilerini al
    """
    try:
        account = client.get_account()
        
        print("Account Information:")
        print(f"Maker Commission: {account['makerCommission']}")
        print(f"Taker Commission: {account['takerCommission']}")
        print(f"Can Trade: {account['canTrade']}")
        print(f"Can Withdraw: {account['canWithdraw']}")
        
        # Balances'ları filtrele (0'dan büyük olanlar)
        balances = [
            balance for balance in account['balances']
            if float(balance['free']) > 0 or float(balance['locked']) > 0
        ]
        
        print(f"\nAssets with Balance: {len(balances)}")
        
        return account
    
    except BinanceAPIException as e:
        print(f"Error: {e}")
        return None

def get_asset_balance(asset='BTC'):
    """
    Belirli bir asset'in bakiyesini al
    
    Parameters:
    asset (str): Asset sembolü (BTC, ETH, USDT, etc.)
    
    Returns:
    dict: Balance bilgisi
    """
    try:
        balance = client.get_asset_balance(asset=asset)
        
        free = float(balance['free'])
        locked = float(balance['locked'])
        total = free + locked
        
        print(f"\n{asset} Balance:")
        print(f"Free: {free:.8f}")
        print(f"Locked: {locked:.8f}")
        print(f"Total: {total:.8f}")
        
        return balance
    
    except BinanceAPIException as e:
        print(f"Error: {e}")
        return None

def get_all_balances():
    """
    Tüm asset'lerin bakiyesini al ve DataFrame'e dönüştür
    """
    try:
        account = client.get_account()
        balances = account['balances']
        
        # DataFrame'e dönüştür
        df = pd.DataFrame(balances)
        df['free'] = df['free'].astype(float)
        df['locked'] = df['locked'].astype(float)
        df['total'] = df['free'] + df['locked']
        
        # Sadece bakiyesi olan asset'leri göster
        df = df[df['total'] > 0].sort_values('total', ascending=False)
        
        print("\nAll Balances:")
        print(df)
        
        return df
    
    except BinanceAPIException as e:
        print(f"Error: {e}")
        return None

# Kullanım
get_account_info()
get_asset_balance('USDT')
all_balances = get_all_balances()
```

## Order Placement (Emir Verme)

![Python Trading Bot](/assets/img/posts/telegram-trading-bot-python.png){: w="700" h="400" .shadow }
_Şekil 2: Python trading bot mimarisi_

### Market Order

```python
from binance.enums import *

def place_market_order(symbol, side, quantity):
    """
    Market order ver (anlık fiyattan al/sat)
    
    Parameters:
    symbol (str): Trading pair (örn: 'BTCUSDT')
    side (str): 'BUY' veya 'SELL'
    quantity (float): Miktar
    
    Returns:
    dict: Order bilgisi
    """
    try:
        # Market order oluştur
        order = client.create_order(
            symbol=symbol,
            side=side,
            type=ORDER_TYPE_MARKET,
            quantity=quantity
        )
        
        print(f"\nMarket Order Placed:")
        print(f"Symbol: {order['symbol']}")
        print(f"Side: {order['side']}")
        print(f"Quantity: {order['executedQty']}")
        print(f"Status: {order['status']}")
        print(f"Order ID: {order['orderId']}")
        
        # Fill bilgilerini göster
        if 'fills' in order:
            total_cost = sum(float(fill['price']) * float(fill['qty']) 
                           for fill in order['fills'])
            avg_price = total_cost / float(order['executedQty'])
            
            print(f"Average Fill Price: ${avg_price:.2f}")
            print(f"Total Cost: ${total_cost:.2f}")
        
        return order
    
    except BinanceAPIException as e:
        print(f"Order Error: {e}")
        return None
    except Exception as e:
        print(f"Unexpected Error: {e}")
        return None
```

> Gerçek para ile order vermeden önce mutlaka Testnet üzerinde test edin! Yanlış bir order geri alınamaz.
{: .prompt-danger }

### Limit Order

```python
def place_limit_order(symbol, side, quantity, price):
    """
    Limit order ver (belirli fiyattan al/sat)
    
    Parameters:
    symbol (str): Trading pair
    side (str): 'BUY' veya 'SELL'
    quantity (float): Miktar
    price (float): Fiyat
    
    Returns:
    dict: Order bilgisi
    """
    try:
        order = client.create_order(
            symbol=symbol,
            side=side,
            type=ORDER_TYPE_LIMIT,
            timeInForce=TIME_IN_FORCE_GTC,  # Good Till Cancel
            quantity=quantity,
            price=str(price)
        )
        
        print(f"\nLimit Order Placed:")
        print(f"Symbol: {order['symbol']}")
        print(f"Side: {order['side']}")
        print(f"Quantity: {order['origQty']}")
        print(f"Price: ${float(order['price']):,.2f}")
        print(f"Status: {order['status']}")
        print(f"Order ID: {order['orderId']}")
        
        return order
    
    except BinanceAPIException as e:
        print(f"Order Error: {e}")
        return None

# Örnek: BTC'yi 40000 USDT'den almak için limit order
# order = place_limit_order('BTCUSDT', SIDE_BUY, 0.001, 40000)
```

### Stop-Loss Order

```python
def place_stop_loss_order(symbol, side, quantity, stop_price, limit_price=None):
    """
    Stop-loss order ver (belirli fiyata gelince sat)
    
    Parameters:
    symbol (str): Trading pair
    side (str): Genellikle 'SELL'
    quantity (float): Miktar
    stop_price (float): Trigger price
    limit_price (float, optional): Limit price (None ise stop-market)
    
    Returns:
    dict: Order bilgisi
    """
    try:
        if limit_price is None:
            # Stop-Loss Market Order
            order = client.create_order(
                symbol=symbol,
                side=side,
                type=ORDER_TYPE_STOP_LOSS,
                quantity=quantity,
                stopPrice=str(stop_price)
            )
        else:
            # Stop-Loss Limit Order
            order = client.create_order(
                symbol=symbol,
                side=side,
                type=ORDER_TYPE_STOP_LOSS_LIMIT,
                timeInForce=TIME_IN_FORCE_GTC,
                quantity=quantity,
                price=str(limit_price),
                stopPrice=str(stop_price)
            )
        
        print(f"\nStop-Loss Order Placed:")
        print(f"Symbol: {order['symbol']}")
        print(f"Side: {order['side']}")
        print(f"Type: {order['type']}")
        print(f"Quantity: {order['origQty']}")
        print(f"Stop Price: ${float(order['stopPrice']):,.2f}")
        print(f"Order ID: {order['orderId']}")
        
        return order
    
    except BinanceAPIException as e:
        print(f"Order Error: {e}")
        return None

# Örnek: BTC'yi 38000'de sat (stop-loss)
# order = place_stop_loss_order('BTCUSDT', SIDE_SELL, 0.001, 38000)
```

### OCO (One-Cancels-the-Other) Order

```python
def place_oco_order(symbol, side, quantity, price, stop_price, stop_limit_price):
    """
    OCO order ver (take-profit + stop-loss birlikte)
    
    Parameters:
    symbol (str): Trading pair
    side (str): 'SELL' (genellikle)
    quantity (float): Miktar
    price (float): Take-profit price
    stop_price (float): Stop-loss trigger price
    stop_limit_price (float): Stop-loss limit price
    
    Returns:
    dict: Order bilgisi
    """
    try:
        order = client.create_oco_order(
            symbol=symbol,
            side=side,
            quantity=quantity,
            price=str(price),
            stopPrice=str(stop_price),
            stopLimitPrice=str(stop_limit_price),
            stopLimitTimeInForce=TIME_IN_FORCE_GTC
        )
        
        print(f"\nOCO Order Placed:")
        print(f"Symbol: {symbol}")
        print(f"Order List ID: {order['orderListId']}")
        print(f"Take Profit Price: ${price:.2f}")
        print(f"Stop Loss Price: ${stop_price:.2f}")
        
        for report in order['orderReports']:
            print(f"\n  Order ID: {report['orderId']}")
            print(f"  Type: {report['type']}")
            print(f"  Status: {report['status']}")
        
        return order
    
    except BinanceAPIException as e:
        print(f"Order Error: {e}")
        return None

# Örnek: BTC aldıktan sonra 45000'de kâr al veya 38000'de zarar kes
# order = place_oco_order('BTCUSDT', SIDE_SELL, 0.001, 45000, 38000, 37900)
```

## Order Yönetimi

```python
def get_open_orders(symbol=None):
    """
    Açık emirleri listele
    
    Parameters:
    symbol (str, optional): Belirli bir symbol için (None ise hepsi)
    
    Returns:
    list: Açık order'lar
    """
    try:
        if symbol:
            orders = client.get_open_orders(symbol=symbol)
        else:
            orders = client.get_open_orders()
        
        print(f"\nOpen Orders: {len(orders)}")
        
        for order in orders:
            print(f"\nOrder ID: {order['orderId']}")
            print(f"Symbol: {order['symbol']}")
            print(f"Type: {order['type']}")
            print(f"Side: {order['side']}")
            print(f"Price: {order['price']}")
            print(f"Quantity: {order['origQty']}")
            print(f"Filled: {order['executedQty']}")
            print(f"Status: {order['status']}")
        
        return orders
    
    except BinanceAPIException as e:
        print(f"Error: {e}")
        return []

def cancel_order(symbol, order_id):
    """
    Belirli bir order'ı iptal et
    
    Parameters:
    symbol (str): Trading pair
    order_id (int): Order ID
    
    Returns:
    dict: İptal bilgisi
    """
    try:
        result = client.cancel_order(
            symbol=symbol,
            orderId=order_id
        )
        
        print(f"\nOrder Cancelled:")
        print(f"Symbol: {result['symbol']}")
        print(f"Order ID: {result['orderId']}")
        print(f"Status: {result['status']}")
        
        return result
    
    except BinanceAPIException as e:
        print(f"Cancel Error: {e}")
        return None

def cancel_all_orders(symbol):
    """
    Bir symbol için tüm açık order'ları iptal et
    
    Parameters:
    symbol (str): Trading pair
    
    Returns:
    list: İptal edilen order'lar
    """
    try:
        result = client.cancel_open_orders(symbol=symbol)
        
        print(f"\nAll orders cancelled for {symbol}")
        print(f"Cancelled: {len(result)} orders")
        
        return result
    
    except BinanceAPIException as e:
        print(f"Cancel Error: {e}")
        return None

def get_order_history(symbol, limit=10):
    """
    Geçmiş order'ları getir
    
    Parameters:
    symbol (str): Trading pair
    limit (int): Maksimum order sayısı
    
    Returns:
    list: Order history
    """
    try:
        orders = client.get_all_orders(symbol=symbol, limit=limit)
        
        print(f"\nOrder History for {symbol}: {len(orders)} orders")
        
        df = pd.DataFrame(orders)
        if not df.empty:
            df['time'] = pd.to_datetime(df['time'], unit='ms')
            df['price'] = df['price'].astype(float)
            df['executedQty'] = df['executedQty'].astype(float)
            
            print(df[['time', 'side', 'type', 'price', 'executedQty', 'status']].tail())
        
        return orders
    
    except BinanceAPIException as e:
        print(f"Error: {e}")
        return []

# Kullanım
open_orders = get_open_orders('BTCUSDT')
# cancel_order('BTCUSDT', order_id=12345678)
# cancel_all_orders('BTCUSDT')
history = get_order_history('BTCUSDT', limit=20)
```

## WebSocket ile Gerçek Zamanlı Veri

WebSocket, gerçek zamanlı piyasa verilerini almak için en verimli yöntemdir.

> WebSocket kullanarak REST API'ye sürekli request göndermekten kaçının. Bu hem rate limit'lere takılmanızı önler hem de daha düşük gecikme sağlar.
{: .prompt-tip }

### Fiyat Takibi

```python
from binance.streams import ThreadedWebsocketManager
import time

# WebSocket Manager oluştur
twm = ThreadedWebsocketManager(
    api_key=os.getenv('BINANCE_API_KEY'),
    api_secret=os.getenv('BINANCE_SECRET_KEY')
)

# Fiyat güncellemelerini handle et
def handle_price_update(msg):
    """
    WebSocket price update callback
    """
    if msg['e'] == 'error':
        print(f"Error: {msg['m']}")
    else:
        symbol = msg['s']
        price = float(msg['c'])  # Close price
        volume = float(msg['v'])  # Volume
        change = float(msg['P'])  # Price change percent
        
        print(f"{symbol}: ${price:,.2f} | Change: {change:+.2f}% | Volume: {volume:,.2f}")

# WebSocket başlat
twm.start()

# Ticker stream'i başlat
twm.start_symbol_ticker_socket(
    callback=handle_price_update,
    symbol='BTCUSDT'
)

print("WebSocket started. Press Ctrl+C to stop...")

try:
    # Stream'i çalışır durumda tut
    while True:
        time.sleep(1)
except KeyboardInterrupt:
    print("\nStopping WebSocket...")
    twm.stop()
```

### Kline (Candlestick) Stream

```python
def handle_kline_update(msg):
    """
    Candlestick update callback
    """
    if msg['e'] == 'error':
        print(f"Error: {msg['m']}")
        return
    
    kline = msg['k']
    
    print(f"\nKline Update for {kline['s']}:")
    print(f"Interval: {kline['i']}")
    print(f"Open: ${float(kline['o']):,.2f}")
    print(f"High: ${float(kline['h']):,.2f}")
    print(f"Low: ${float(kline['l']):,.2f}")
    print(f"Close: ${float(kline['c']):,.2f}")
    print(f"Volume: {float(kline['v']):,.2f}")
    print(f"Is Closed: {kline['x']}")

# Kline stream başlat
twm.start()
twm.start_kline_socket(
    callback=handle_kline_update,
    symbol='BTCUSDT',
    interval='1m'
)

print("Kline stream started...")
```

### Order Book (Depth) Stream

```python
def handle_depth_update(msg):
    """
    Order book update callback
    """
    if msg['e'] == 'error':
        print(f"Error: {msg['m']}")
        return
    
    bids = msg['b']  # Bid updates
    asks = msg['a']  # Ask updates
    
    print(f"\nOrder Book Update for {msg['s']}:")
    
    if bids:
        print("Bid Updates:")
        for bid in bids[:5]:  # Top 5
            price, quantity = float(bid[0]), float(bid[1])
            print(f"  ${price:,.2f} x {quantity:.4f}")
    
    if asks:
        print("Ask Updates:")
        for ask in asks[:5]:  # Top 5
            price, quantity = float(ask[0]), float(ask[1])
            print(f"  ${price:,.2f} x {quantity:.4f}")

# Depth stream başlat
twm.start()
twm.start_depth_socket(
    callback=handle_depth_update,
    symbol='BTCUSDT'
)

print("Depth stream started...")
```

### Trade Stream

```python
def handle_trade_update(msg):
    """
    Individual trade update callback
    """
    if msg['e'] == 'error':
        print(f"Error: {msg['m']}")
        return
    
    print(f"\nNew Trade for {msg['s']}:")
    print(f"Price: ${float(msg['p']):,.2f}")
    print(f"Quantity: {float(msg['q']):.4f}")
    print(f"Buyer is Maker: {msg['m']}")
    print(f"Trade Time: {pd.to_datetime(msg['T'], unit='ms')}")

# Trade stream başlat
twm.start()
twm.start_trade_socket(
    callback=handle_trade_update,
    symbol='BTCUSDT'
)

print("Trade stream started...")
```

### User Data Stream

```python
def handle_user_update(msg):
    """
    User data stream callback (orders, balances)
    """
    event_type = msg['e']
    
    if event_type == 'executionReport':
        # Order update
        print(f"\nOrder Update:")
        print(f"Symbol: {msg['s']}")
        print(f"Side: {msg['S']}")
        print(f"Type: {msg['o']}")
        print(f"Status: {msg['X']}")
        print(f"Price: {msg['p']}")
        print(f"Quantity: {msg['q']}")
        print(f"Filled: {msg['z']}")
        
    elif event_type == 'outboundAccountPosition':
        # Balance update
        print(f"\nBalance Update:")
        for balance in msg['B']:
            if float(balance['f']) > 0 or float(balance['l']) > 0:
                print(f"{balance['a']}: Free={balance['f']}, Locked={balance['l']}")

# User data stream başlat
twm.start()
twm.start_user_socket(callback=handle_user_update)

print("User data stream started...")
```

## Trading Bot Örneği

Basit bir trading bot implementasyonu:

```python
import time
from loguru import logger

class SimpleTradingBot:
    """
    Basit bir trading bot - SMA crossover stratejisi
    """
    
    def __init__(self, symbol='BTCUSDT', interval='5m', short_period=10, long_period=30):
        self.symbol = symbol
        self.interval = interval
        self.short_period = short_period
        self.long_period = long_period
        self.position = None  # None, 'LONG', 'SHORT'
        self.entry_price = 0
        
        logger.add("trading_bot.log", rotation="1 day")
        logger.info(f"Bot initialized for {symbol}")
        
    def get_sma(self, period):
        """
        SMA (Simple Moving Average) hesapla
        """
        klines = client.get_klines(
            symbol=self.symbol,
            interval=self.interval,
            limit=period
        )
        
        closes = [float(k[4]) for k in klines]
        sma = sum(closes) / len(closes)
        
        return sma
    
    def check_signal(self):
        """
        Trading sinyali kontrol et
        """
        short_sma = self.get_sma(self.short_period)
        long_sma = self.get_sma(self.long_period)
        current_price = float(client.get_symbol_ticker(symbol=self.symbol)['price'])
        
        logger.info(f"Price: ${current_price:.2f} | Short SMA: ${short_sma:.2f} | Long SMA: ${long_sma:.2f}")
        
        # Bullish crossover (AL)
        if short_sma > long_sma and self.position is None:
            return 'BUY', current_price
        
        # Bearish crossover (SAT)
        elif short_sma < long_sma and self.position == 'LONG':
            return 'SELL', current_price
        
        return None, current_price
    
    def execute_trade(self, signal, price):
        """
        Trading işlemini gerçekleştir
        """
        try:
            if signal == 'BUY':
                # UYARI: Bu örnek kodda miktar sabit, gerçek kullanımda hesaplayın
                quantity = 0.001
                
                logger.info(f"BUY signal at ${price:.2f}")
                # order = place_market_order(self.symbol, SIDE_BUY, quantity)
                
                self.position = 'LONG'
                self.entry_price = price
                
                logger.info(f"Position opened: LONG at ${price:.2f}")
                
            elif signal == 'SELL' and self.position == 'LONG':
                quantity = 0.001
                
                profit = (price - self.entry_price) / self.entry_price * 100
                logger.info(f"SELL signal at ${price:.2f} | Profit: {profit:.2f}%")
                
                # order = place_market_order(self.symbol, SIDE_SELL, quantity)
                
                self.position = None
                self.entry_price = 0
                
                logger.info(f"Position closed at ${price:.2f}")
                
        except Exception as e:
            logger.error(f"Trade execution error: {e}")
    
    def run(self, check_interval=60):
        """
        Bot'u çalıştır
        
        Parameters:
        check_interval (int): Kontrol aralığı (saniye)
        """
        logger.info(f"Bot started. Checking every {check_interval} seconds...")
        
        try:
            while True:
                signal, price = self.check_signal()
                
                if signal:
                    self.execute_trade(signal, price)
                
                time.sleep(check_interval)
                
        except KeyboardInterrupt:
            logger.info("Bot stopped by user")
        except Exception as e:
            logger.error(f"Bot error: {e}")
```

> Herhangi bir trading botunu production'a almadan önce en az 1 ay backtest ve paper trading yapın. Risk yönetimi kurallarını mutlaka uygulayın.
{: .prompt-warning }

## Risk Yönetimi

```python
class RiskManager:
    """
    Risk yönetimi için yardımcı sınıf
    """
    
    def __init__(self, max_position_size_usd=1000, max_loss_percent=2):
        self.max_position_size_usd = max_position_size_usd
        self.max_loss_percent = max_loss_percent
    
    def calculate_position_size(self, entry_price, stop_loss_price, account_balance):
        """
        Position size hesapla (risk bazlı)
        
        Parameters:
        entry_price (float): Giriş fiyatı
        stop_loss_price (float): Stop loss fiyatı
        account_balance (float): Hesap bakiyesi (USDT)
        
        Returns:
        float: Position size
        """
        # Risk miktarı (USDT)
        risk_amount = account_balance * (self.max_loss_percent / 100)
        
        # Fiyat farkı (risk per unit)
        price_difference = abs(entry_price - stop_loss_price)
        
        # Position size
        position_size = risk_amount / price_difference
        
        # Maximum position size kontrolü
        max_quantity = self.max_position_size_usd / entry_price
        position_size = min(position_size, max_quantity)
        
        return position_size
    
    def check_daily_loss_limit(self, trades_today):
        """
        Günlük zarar limitini kontrol et
        
        Parameters:
        trades_today (list): Bugünkü işlemler
        
        Returns:
        bool: Trading devam edilebilir mi?
        """
        total_pnl = sum(trade['pnl'] for trade in trades_today)
        
        # Günlük maksimum zarar: %5
        max_daily_loss = 0.05
        
        if total_pnl < -max_daily_loss:
            logger.warning(f"Daily loss limit reached: {total_pnl:.2%}")
            return False
        
        return True
    
    def calculate_stop_loss(self, entry_price, side, atr, multiplier=2):
        """
        ATR bazlı stop loss hesapla
        
        Parameters:
        entry_price (float): Giriş fiyatı
        side (str): 'BUY' veya 'SELL'
        atr (float): Average True Range
        multiplier (float): ATR çarpanı
        
        Returns:
        float: Stop loss fiyatı
        """
        stop_distance = atr * multiplier
        
        if side == SIDE_BUY:
            stop_loss = entry_price - stop_distance
        else:
            stop_loss = entry_price + stop_distance
        
        return stop_loss
    
    def calculate_take_profit(self, entry_price, stop_loss, risk_reward_ratio=2):
        """
        Risk/reward ratio bazlı take profit hesapla
        
        Parameters:
        entry_price (float): Giriş fiyatı
        stop_loss (float): Stop loss fiyatı
        risk_reward_ratio (float): Risk/reward oranı
        
        Returns:
        float: Take profit fiyatı
        """
        risk = abs(entry_price - stop_loss)
        reward = risk * risk_reward_ratio
        
        if entry_price > stop_loss:  # Long position
            take_profit = entry_price + reward
        else:  # Short position
            take_profit = entry_price - reward
        
        return take_profit

# Kullanım
risk_manager = RiskManager(max_position_size_usd=1000, max_loss_percent=2)

# Example: BTC long position
entry = 40000
stop_loss = 39000
account_balance = 10000  # USDT

position_size = risk_manager.calculate_position_size(entry, stop_loss, account_balance)
print(f"Position Size: {position_size:.4f} BTC")

take_profit = risk_manager.calculate_take_profit(entry, stop_loss, risk_reward_ratio=2.5)
print(f"Take Profit: ${take_profit:.2f}")
```

## Error Handling ve Best Practices

```python
from binance.exceptions import (
    BinanceAPIException,
    BinanceOrderException,
    BinanceRequestException
)
import requests
from tenacity import retry, stop_after_attempt, wait_exponential

class BinanceTrader:
    """
    Gelişmiş error handling ile Binance trader
    """
    
    def __init__(self, api_key, secret_key):
        self.client = Client(api_key, secret_key)
        
    @retry(
        stop=stop_after_attempt(3),
        wait=wait_exponential(multiplier=1, min=4, max=10)
    )
    def safe_api_call(self, func, *args, **kwargs):
        """
        Retry logic ile güvenli API çağrısı
        """
        try:
            return func(*args, **kwargs)
        except BinanceAPIException as e:
            logger.error(f"Binance API Error: {e.status_code} - {e.message}")
            
            # Rate limit hatası
            if e.status_code == 429:
                logger.warning("Rate limit exceeded, waiting...")
                time.sleep(60)
                raise  # Retry için
            
            # Invalid order
            elif e.status_code == 400:
                logger.error("Invalid order parameters")
                return None
            
            # Insufficient balance
            elif 'insufficient' in str(e).lower():
                logger.error("Insufficient balance for order")
                return None
            
            else:
                raise
                
        except BinanceRequestException as e:
            logger.error(f"Request Error: {e}")
            raise  # Retry için
            
        except requests.exceptions.RequestException as e:
            logger.error(f"Network Error: {e}")
            raise  # Retry için
            
        except Exception as e:
            logger.error(f"Unexpected Error: {e}")
            return None
    
    def validate_order_params(self, symbol, quantity, price=None):
        """
        Order parametrelerini valide et
        """
        try:
            # Symbol bilgisini al
            info = self.client.get_symbol_info(symbol)
            
            if not info:
                logger.error(f"Invalid symbol: {symbol}")
                return False
            
            # Filters kontrolü
            for f in info['filters']:
                # LOT_SIZE kontrolü
                if f['filterType'] == 'LOT_SIZE':
                    min_qty = float(f['minQty'])
                    max_qty = float(f['maxQty'])
                    step_size = float(f['stepSize'])
                    
                    if quantity < min_qty:
                        logger.error(f"Quantity {quantity} below minimum {min_qty}")
                        return False
                    
                    if quantity > max_qty:
                        logger.error(f"Quantity {quantity} above maximum {max_qty}")
                        return False
                
                # PRICE_FILTER kontrolü
                if f['filterType'] == 'PRICE_FILTER' and price:
                    min_price = float(f['minPrice'])
                    max_price = float(f['maxPrice'])
                    tick_size = float(f['tickSize'])
                    
                    if price < min_price or price > max_price:
                        logger.error(f"Price {price} out of range")
                        return False
            
            return True
            
        except Exception as e:
            logger.error(f"Validation error: {e}")
            return False
    
    def place_order_safe(self, symbol, side, quantity, order_type='MARKET', price=None):
        """
        Güvenli order placement
        """
        # Parametreleri valide et
        if not self.validate_order_params(symbol, quantity, price):
            return None
        
        # Order'ı oluştur
        order_params = {
            'symbol': symbol,
            'side': side,
            'type': order_type,
            'quantity': quantity
        }
        
        if order_type == ORDER_TYPE_LIMIT and price:
            order_params['timeInForce'] = TIME_IN_FORCE_GTC
            order_params['price'] = str(price)
        
        # API çağrısı yap
        order = self.safe_api_call(
            self.client.create_order,
            **order_params
        )
        
        if order:
            logger.info(f"Order placed successfully: {order['orderId']}")
        
        return order

# Kullanım
trader = BinanceTrader(
    api_key=os.getenv('BINANCE_API_KEY'),
    secret_key=os.getenv('BINANCE_SECRET_KEY')
)

# order = trader.place_order_safe('BTCUSDT', SIDE_BUY, 0.001)
```

## Monitoring ve Logging

```python
from loguru import logger
import sys

# Logging konfigürasyonu
logger.remove()  # Default handler'ı kaldır

# Console logger
logger.add(
    sys.stdout,
    format="<green>{time:YYYY-MM-DD HH:mm:ss}</green> | <level>{level: <8}</level> | <cyan>{name}</cyan>:<cyan>{function}</cyan> - <level>{message}</level>",
    level="INFO"
)

# File logger
logger.add(
    "logs/trading_{time:YYYY-MM-DD}.log",
    rotation="00:00",  # Her gün yeni dosya
    retention="30 days",  # 30 gün sakla
    compression="zip",  # Eski logları sıkıştır
    format="{time:YYYY-MM-DD HH:mm:ss} | {level: <8} | {name}:{function} - {message}",
    level="DEBUG"
)

# Error logger
logger.add(
    "logs/errors_{time:YYYY-MM-DD}.log",
    rotation="00:00",
    retention="90 days",
    compression="zip",
    format="{time:YYYY-MM-DD HH:mm:ss} | {level: <8} | {name}:{function} - {message}",
    level="ERROR"
)

# Performance monitoring
class PerformanceMonitor:
    """
    Trading performansını takip et
    """
    
    def __init__(self):
        self.trades = []
        self.daily_pnl = {}
    
    def log_trade(self, trade_data):
        """
        İşlemi kaydet
        """
        self.trades.append(trade_data)
        
        date = trade_data['timestamp'].date()
        if date not in self.daily_pnl:
            self.daily_pnl[date] = []
        
        self.daily_pnl[date].append(trade_data['pnl'])
        
        logger.info(f"Trade logged: {trade_data['symbol']} | PnL: {trade_data['pnl']:.2f}")
    
    def get_statistics(self):
        """
        İstatistikleri hesapla
        """
        if not self.trades:
            return None
        
        df = pd.DataFrame(self.trades)
        
        stats = {
            'total_trades': len(df),
            'winning_trades': len(df[df['pnl'] > 0]),
            'losing_trades': len(df[df['pnl'] < 0]),
            'win_rate': (len(df[df['pnl'] > 0]) / len(df)) * 100,
            'total_pnl': df['pnl'].sum(),
            'avg_win': df[df['pnl'] > 0]['pnl'].mean() if len(df[df['pnl'] > 0]) > 0 else 0,
            'avg_loss': df[df['pnl'] < 0]['pnl'].mean() if len(df[df['pnl'] < 0]) > 0 else 0,
            'largest_win': df['pnl'].max(),
            'largest_loss': df['pnl'].min(),
            'profit_factor': abs(df[df['pnl'] > 0]['pnl'].sum() / df[df['pnl'] < 0]['pnl'].sum()) if len(df[df['pnl'] < 0]) > 0 else 0
        }
        
        return stats
    
    def print_report(self):
        """
        Rapor yazdır
        """
        stats = self.get_statistics()
        
        if not stats:
            logger.info("No trades to report")
            return
        
        logger.info("\n" + "="*50)
        logger.info("TRADING PERFORMANCE REPORT")
        logger.info("="*50)
        logger.info(f"Total Trades: {stats['total_trades']}")
        logger.info(f"Winning Trades: {stats['winning_trades']}")
        logger.info(f"Losing Trades: {stats['losing_trades']}")
        logger.info(f"Win Rate: {stats['win_rate']:.2f}%")
        logger.info(f"Total PnL: ${stats['total_pnl']:.2f}")
        logger.info(f"Average Win: ${stats['avg_win']:.2f}")
        logger.info(f"Average Loss: ${stats['avg_loss']:.2f}")
        logger.info(f"Largest Win: ${stats['largest_win']:.2f}")
        logger.info(f"Largest Loss: ${stats['largest_loss']:.2f}")
        logger.info(f"Profit Factor: {stats['profit_factor']:.2f}")
        logger.info("="*50 + "\n")

# Kullanım
monitor = PerformanceMonitor()

# İşlem kaydı
# trade = {
#     'timestamp': pd.Timestamp.now(),
#     'symbol': 'BTCUSDT',
#     'side': 'BUY',
#     'entry_price': 40000,
#     'exit_price': 41000,
#     'quantity': 0.1,
#     'pnl': 100
# }
# monitor.log_trade(trade)
# monitor.print_report()
```

## Sonuç

> Bu rehberde öğrendiğiniz teknikler sadece Binance için değil, CCXT kütüphanesi ile 100+ farklı exchange için de kullanılabilir.
{: .prompt-info }

Python ile Binance API entegrasyonu, kripto trading dünyasında güçlü otomasyonlar kurmanızı sağlar. Bu rehberde öğrendikleriniz:

1. **REST API**: Market data çekme, order placement ve account yönetimi
2. **WebSocket**: Gerçek zamanlı veri akışları ve event handling
3. **Trading Stratejileri**: Basit SMA crossover bot implementasyonu
4. **Risk Yönetimi**: Position sizing, stop loss ve take profit hesaplamaları
5. **Error Handling**: Güvenli API çağrıları ve retry logic
6. **Monitoring**: Performance tracking ve logging best practices

**Önemli Hatırlatmalar:**

- API key'lerinizi asla paylaşmayın ve git'e eklemeyin
- Test işlemlerinde Testnet kullanın
- Risk yönetimi kurallarına uyun
- Rate limit'lere dikkat edin
- Production'da comprehensive logging kullanın
- Her zaman stop loss kullanın

Trading bot geliştirmek teknik bilgi gerektirir ancak risk yönetimi daha da önemlidir. Küçük başlayın, test edin ve yavaş yavaş ölçeklendirin.

## Kaynaklar

### Resmi Dokümantasyon
- [Binance API Documentation](https://binance-docs.github.io/apidocs/spot/en/)
- [python-binance Documentation](https://python-binance.readthedocs.io/)
- [Binance API GitHub](https://github.com/binance/binance-spot-api-docs)
- [CCXT Documentation](https://docs.ccxt.com/)

### Kütüphaneler
- [python-binance](https://github.com/sammchardy/python-binance) - Binance API wrapper
- [ccxt](https://github.com/ccxt/ccxt) - Multi-exchange library
- [pandas-ta](https://github.com/twopirllc/pandas-ta) - Technical analysis
- [loguru](https://github.com/Delgan/loguru) - Logging

### Öğrenme Kaynakları
- [Binance Academy](https://academy.binance.com/) - Trading eğitimleri
- [QuantConnect](https://www.quantconnect.com/) - Algorithmic trading platform
- [Backtrader](https://www.backtrader.com/) - Backtesting framework

### Topluluklar
- [r/algotrading](https://www.reddit.com/r/algotrading/)
- [Binance API Telegram](https://t.me/binance_api_english)
- [Python Trading Discord](https://discord.gg/python-trading)

Başarılı ve güvenli trading'ler! 🚀📈
