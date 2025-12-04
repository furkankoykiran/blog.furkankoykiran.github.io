---
title: "Python ile Telegram Trading Botu Geliştirme"
description: "Python ve Telegram Bot API kullanarak 7/24 çalışan otomatik trading botu geliştirme rehberi. CCXT entegrasyonu, teknik indikatörler, alarm sistemi ve deployment."
date: "2024-07-22"
categories:
  - "bot-development"
  - "python"
tags:
  - "python"
  - "telegram-bot"
  - "trading"
  - "kripto"
  - "otomasyon"
  - "api"
  - "bot-geliştirme"
  - "telegram-api"
image:
  path: "/assets/img/posts/telegram-trading-bot-python.png"
  alt: "Python ile Telegram Trading Botu"
---

Kripto para piyasaları 7/24 açık olduğundan, manuel olarak takip etmek hem yorucu hem de kaçırılan fırsatlar anlamına gelir. Bu yazıda, Python kullanarak Telegram üzerinden kontrol edebileceğiniz bir trading botu nasıl geliştirilir, adım adım göreceğiz.

## Neden Telegram Botu?

Telegram, trading botları için mükemmel bir platform çünkü:

- **Mobil Erişim**: Botunuzu her yerden kontrol edebilirsiniz
- **Anlık Bildirimler**: Fiyat hareketlerini anında öğrenirsiniz
- **Kolay API**: Telegram Bot API kullanımı oldukça basittir
- **Güvenlik**: End-to-end encryption ile güvenli iletişim
- **Ücretsiz**: Sınırsız mesaj gönderimi

![Telegram Trading Bot](/assets/img/posts/telegram-trading-bot-python.png){: w="700" h="400" .shadow }
_Python ile geliştirilen Telegram trading bot arayüzü_

## Gereksinimler

Başlamadan önce ihtiyacınız olanlar:

```bash
# Python 3.8 veya üzeri
python --version

# Gerekli kütüphaneler
pip install python-telegram-bot==20.6
pip install ccxt  # Kripto exchange API wrapper
pip install pandas numpy
pip install python-dotenv
```
{: .nolineno }

## 1. Telegram Bot Oluşturma

Öncelikle BotFather ile yeni bir bot oluşturalım:

1. Telegram'da [BotFather](https://t.me/botfather) botunu bulun
2. `/newbot` komutunu gönderin
3. Bot adını belirleyin (örn: "MyTradingBot")
4. Username seçin (örn: "my_trading_bot")
5. **Bot Token**'ı kopyalayın (örn: `1234567890:ABCdefGHIjklMNOpqrsTUVwxyz`)

> Token'ı asla paylaşmayın veya GitHub'a yüklemeyin! Token sızdırılması durumunda bot kontrolünü kaybedersiniz.
{: .prompt-danger }

## 2. Proje Yapısı

```
trading_bot/
├── .env                # API anahtarları (git ignore!)
├── config.py           # Konfigürasyon
├── bot.py              # Telegram bot ana dosya
├── trader.py           # Trading logic
├── indicators.py       # Teknik indikatörler
└── requirements.txt    # Dependencies
```
{: .nolineno }

## 3. Temel Bot Yapısı

```python
import os
from dotenv import load_dotenv
from telegram import Update
from telegram.ext import (
    Application,
    CommandHandler,
    ContextTypes,
    MessageHandler,
    filters
)

load_dotenv()

# Bot token'ı .env dosyasından al
BOT_TOKEN = os.getenv('TELEGRAM_BOT_TOKEN')

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Bot başlatma komutu"""
    welcome_message = """
*Trading Bot'a Hoş Geldiniz!*

Kullanılabilir komutlar:
/price BTC - Bitcoin fiyatını gösterir
/alert BTC 45000 - Fiyat alarmı kur
/balance - Portfolio bakiyeniz
/buy BTC 0.001 - Alım emri
/sell BTC 0.001 - Satım emri
/status - Bot durumu
/help - Yardım
    """
    await update.message.reply_text(
        welcome_message,
        parse_mode='Markdown'
    )

async def price(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Kripto fiyatını göster"""
    if not context.args:
        await update.message.reply_text("Kullanım: /price BTC")
        return
    
    symbol = context.args[0].upper()
    
    try:
        # Binance'den fiyat çek (trader.py'dan)
        current_price = get_current_price(symbol)
        
        message = f"""
*{symbol}/USDT Fiyatı*

Fiyat: ${current_price:,.2f}
Son Güncelleme: {datetime.now().strftime('%H:%M:%S')}
        """
        await update.message.reply_text(message, parse_mode='Markdown')
        
    except Exception as e:
        await update.message.reply_text(f"Hata: {str(e)}")

async def help_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Yardım mesajı"""
    help_text = """
*Detaylı Komut Listesi*

*Fiyat Sorguları:*
/price <symbol> - Anlık fiyat
/chart <symbol> - Fiyat grafiği (24h)

*Alarmlar:*
/alert <symbol> <fiyat> - Fiyat alarmı kur
/alerts - Aktif alarmları listele
/delalert <id> - Alarm sil

*Trading:*
/buy <symbol> <miktar> - Alım yap
/sell <symbol> <miktar> - Satım yap
/balance - Portfolio görüntüle
/history - İşlem geçmişi

*Bot Yönetimi:*
/status - Bot durumu
/start - Botu başlat
/stop - Botu durdur
    """
    await update.message.reply_text(help_text, parse_mode='Markdown')

def main():
    """Bot'u başlat"""
    # Application oluştur
    app = Application.builder().token(BOT_TOKEN).build()
    
    # Command handler'ları ekle
    app.add_handler(CommandHandler("start", start))
    app.add_handler(CommandHandler("price", price))
    app.add_handler(CommandHandler("help", help_command))
    
    # Botu başlat
    print("Bot çalışıyor...")
    app.run_polling(allowed_updates=Update.ALL_TYPES)

if __name__ == '__main__':
    main()
```
{: file="bot.py" }

## 4. Exchange Entegrasyonu (CCXT)

```python
import ccxt
from datetime import datetime

class CryptoTrader:
    def __init__(self, api_key=None, api_secret=None):
        """Binance exchange bağlantısı"""
        self.exchange = ccxt.binance({
            'apiKey': api_key,
            'secret': api_secret,
            'enableRateLimit': True,
            'options': {'defaultType': 'spot'}
        })
    
    def get_price(self, symbol='BTC/USDT'):
        """Anlık fiyat getir"""
        ticker = self.exchange.fetch_ticker(symbol)
        return {
            'symbol': symbol,
            'price': ticker['last'],
            'bid': ticker['bid'],
            'ask': ticker['ask'],
            'volume': ticker['volume'],
            'change_24h': ticker['percentage']
        }
    
    def get_balance(self):
        """Hesap bakiyesi"""
        balance = self.exchange.fetch_balance()
        return {
            asset: {
                'free': balance[asset]['free'],
                'used': balance[asset]['used'],
                'total': balance[asset]['total']
            }
            for asset in balance
            if balance[asset]['total'] > 0
        }
    
    def place_order(self, symbol, side, amount, order_type='market', price=None):
        """
        Emir gönder
        
        side: 'buy' veya 'sell'
        order_type: 'market', 'limit'
        """
        try:
            if order_type == 'market':
                order = self.exchange.create_market_order(symbol, side, amount)
            elif order_type == 'limit':
                order = self.exchange.create_limit_order(symbol, side, amount, price)
            
            return {
                'success': True,
                'order_id': order['id'],
                'symbol': order['symbol'],
                'side': order['side'],
                'amount': order['amount'],
                'price': order['price'],
                'status': order['status']
            }
        except Exception as e:
            return {
                'success': False,
                'error': str(e)
            }
    
    def get_ohlcv(self, symbol='BTC/USDT', timeframe='1h', limit=100):
        """OHLCV (mum çubuk) verisi al"""
        ohlcv = self.exchange.fetch_ohlcv(symbol, timeframe, limit=limit)
        df = pd.DataFrame(
            ohlcv,
            columns=['timestamp', 'open', 'high', 'low', 'close', 'volume']
        )
        df['timestamp'] = pd.to_datetime(df['timestamp'], unit='ms')
        return df

# Singleton instance
trader = CryptoTrader()

def get_current_price(symbol):
    """Harici fonksiyon - bot.py'dan çağrılır"""
    data = trader.get_price(f"{symbol}/USDT")
    return data['price']
```
{: file="trader.py" }

## 5. Fiyat Alarm Sistemi

```python
# bot.py'a ekle
import asyncio
from collections import defaultdict

# Global alarm storage
price_alerts = defaultdict(list)  # {user_id: [{symbol, target_price, condition}]}

async def set_alert(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Fiyat alarmı kur"""
    if len(context.args) < 2:
        await update.message.reply_text("Kullanım: /alert BTC 45000")
        return
    
    symbol = context.args[0].upper()
    target_price = float(context.args[1])
    user_id = update.effective_user.id
    
    # Alarm ekle
    alert = {
        'symbol': symbol,
        'target': target_price,
        'created': datetime.now()
    }
    price_alerts[user_id].append(alert)
    
    await update.message.reply_text(
        f"Alarm kuruldu!\n\n"
        f"{symbol}/USDT = ${target_price:,.2f}"
    )

async def check_alerts(context: ContextTypes.DEFAULT_TYPE):
    """Periyodik olarak alarmları kontrol et"""
    for user_id, alerts in price_alerts.items():
        for alert in alerts[:]:  # Copy list to safely remove items
            symbol = alert['symbol']
            target = alert['target']
            
            current_price = get_current_price(symbol)
            
            # Hedef fiyata ulaşıldı mı?
            if current_price >= target:
                message = f"""
*Fiyat Alarmı!*

{symbol}/USDT hedef fiyata ulaştı!
Hedef: ${target:,.2f}
Güncel: ${current_price:,.2f}
                """
                await context.bot.send_message(
                    chat_id=user_id,
                    text=message,
                    parse_mode='Markdown'
                )
                
                # Alarm tetiklendi, sil
                alerts.remove(alert)

# Ana fonksiyona ekle
def main():
    app = Application.builder().token(BOT_TOKEN).build()
    
    # Komutlar...
    app.add_handler(CommandHandler("alert", set_alert))
    
    # Periyodik görevler
    job_queue = app.job_queue
    job_queue.run_repeating(check_alerts, interval=60, first=10)  # Her 60 saniyede
    
    app.run_polling()
```
{: file="bot.py" }

> Alarm sistemi bellekte (RAM) tutuluyor. Kalıcı alarm sistemi için Redis veya SQLite kullanın.
{: .prompt-tip }

## 6. Teknik İndikatörler Entegrasyonu

```python
import pandas as pd
import numpy as np

def calculate_rsi(df, period=14):
    """RSI (Relative Strength Index)"""
    delta = df['close'].diff()
    gain = (delta.where(delta > 0, 0)).rolling(window=period).mean()
    loss = (-delta.where(delta < 0, 0)).rolling(window=period).mean()
    
    rs = gain / loss
    rsi = 100 - (100 / (1 + rs))
    return rsi

def calculate_macd(df, fast=12, slow=26, signal=9):
    """MACD (Moving Average Convergence Divergence)"""
    ema_fast = df['close'].ewm(span=fast).mean()
    ema_slow = df['close'].ewm(span=slow).mean()
    
    macd_line = ema_fast - ema_slow
    signal_line = macd_line.ewm(span=signal).mean()
    histogram = macd_line - signal_line
    
    return macd_line, signal_line, histogram

def calculate_bollinger_bands(df, period=20, std=2):
    """Bollinger Bands"""
    sma = df['close'].rolling(window=period).mean()
    std_dev = df['close'].rolling(window=period).std()
    
    upper_band = sma + (std_dev * std)
    lower_band = sma - (std_dev * std)
    
    return upper_band, sma, lower_band
```
{: file="indicators.py" }

```python
# Bot'a teknik analiz komutu ekle
async def analyze(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Teknik analiz yap"""
    if not context.args:
        await update.message.reply_text("Kullanım: /analyze BTC")
        return
    
    symbol = context.args[0].upper()
    
    # OHLCV verisi al
    df = trader.get_ohlcv(f"{symbol}/USDT", '1h', 100)
    
    # İndikatörler hesapla
    rsi = calculate_rsi(df).iloc[-1]
    macd, signal, hist = calculate_macd(df)
    upper, middle, lower = calculate_bollinger_bands(df)
    
    current_price = df['close'].iloc[-1]
    
    # Sinyal üret
    signals = []
    if rsi < 30:
        signals.append("RSI oversold - AL sinyali")
    elif rsi > 70:
        signals.append("RSI overbought - SAT sinyali")
    
    if macd.iloc[-1] > signal.iloc[-1] and macd.iloc[-2] <= signal.iloc[-2]:
        signals.append("MACD bullish crossover - AL sinyali")
    
    if current_price < lower.iloc[-1]:
        signals.append("Fiyat alt Bollinger Band'de - AL fırsatı")
    
    message = f"""
*{symbol}/USDT Teknik Analiz*

Fiyat: ${current_price:,.2f}

İndikatörler:
RSI (14): {rsi:.2f}
MACD: {macd.iloc[-1]:.2f}
Signal: {signal.iloc[-1]:.2f}

Bollinger Bands:
Upper: ${upper.iloc[-1]:,.2f}
Middle: ${middle.iloc[-1]:,.2f}
Lower: ${lower.iloc[-1]:,.2f}

Sinyaller:
{chr(10).join(signals) if signals else 'Beklemede...'}
    """
    
    await update.message.reply_text(message, parse_mode='Markdown')
```
{: file="bot.py" }

> Teknik indikatörler geçmiş verilere dayalıdır ve gelecek fiyat hareketlerini garanti etmez. Risk yönetimi kullanın.
{: .prompt-warning }

## 7. Güvenlik ve Best Practices

### Environment Variables (.env)

```bash
TELEGRAM_BOT_TOKEN=1234567890:ABCdefGHIjklMNOpqrsTUVwxyz
BINANCE_API_KEY=your_api_key_here
BINANCE_API_SECRET=your_api_secret_here
ALLOWED_USER_IDS=123456789,987654321  # Sadece bu user ID'ler botu kullanabilir
```
{: file=".env" }

> `.env`{: .filepath} dosyasını mutlaka `.gitignore`{: .filepath} dosyasına ekleyin!
{: .prompt-danger }

### Güvenlik Kontrolleri

```python
import os
from dotenv import load_dotenv

load_dotenv()

ALLOWED_USERS = set(map(int, os.getenv('ALLOWED_USER_IDS', '').split(',')))

def is_authorized(user_id):
    """Kullanıcı yetkili mi?"""
    return user_id in ALLOWED_USERS
```
{: file="config.py" }

```python
# bot.py'da kullanım
async def buy(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    
    if not is_authorized(user_id):
        await update.message.reply_text("Bu botu kullanma yetkiniz yok!")
        return
    
    # Trading logic...
```
{: file="bot.py" }

### Rate Limiting

```python
from datetime import datetime, timedelta
from collections import defaultdict

# Rate limiter
user_requests = defaultdict(list)
MAX_REQUESTS_PER_MINUTE = 10

def rate_limit_check(user_id):
    """Kullanıcı çok fazla istek gönderiyor mu?"""
    now = datetime.now()
    user_requests[user_id] = [
        req_time for req_time in user_requests[user_id]
        if now - req_time < timedelta(minutes=1)
    ]
    
    if len(user_requests[user_id]) >= MAX_REQUESTS_PER_MINUTE:
        return False
    
    user_requests[user_id].append(now)
    return True
```

> Rate limiting sayesinde botunuz spam saldırılarından ve aşırı API kullanımından korunur.
{: .prompt-tip }

## 8. Loglama ve Hata Yönetimi

```python
import logging

# Logging yapılandırması
logging.basicConfig(
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    level=logging.INFO,
    handlers=[
        logging.FileHandler('bot.log'),
        logging.StreamHandler()
    ]
)
logger = logging.getLogger(__name__)

# Hata yakalama
async def error_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Global hata yakalayıcı"""
    logger.error(f"Update {update} caused error {context.error}")
    
    if update and update.effective_message:
        await update.effective_message.reply_text(
            "Bir hata oluştu. Lütfen daha sonra tekrar deneyin."
        )

# Main'e ekle
app.add_error_handler(error_handler)
```

## 9. Deployment (VPS/Cloud)

### Systemd Service (Linux)

```ini
[Unit]
Description=Telegram Trading Bot
After=network.target

[Service]
Type=simple
User=ubuntu
WorkingDirectory=/home/ubuntu/trading_bot
ExecStart=/usr/bin/python3 /home/ubuntu/trading_bot/bot.py
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```
{: file="/etc/systemd/system/trading-bot.service" }

```bash
# Servisi etkinleştir
sudo systemctl enable trading-bot
sudo systemctl start trading-bot
sudo systemctl status trading-bot

# Logları görüntüle
journalctl -u trading-bot -f
```
{: .nolineno }

### Docker ile Deployment

```dockerfile
FROM python:3.10-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD ["python", "bot.py"]
```
{: file="Dockerfile" }

```bash
# Build ve çalıştır
docker build -t trading-bot .
docker run -d --name trading-bot --env-file .env trading-bot
```
{: .nolineno }

## 10. Örnek Kullanım Senaryoları

### Senaryo 1: Otomatik Al-Sat Botu

```python
async def auto_trade(context: ContextTypes.DEFAULT_TYPE):
    """RSI stratejisine göre otomatik trade"""
    symbol = 'BTC/USDT'
    df = trader.get_ohlcv(symbol, '15m', 50)
    rsi = calculate_rsi(df).iloc[-1]
    
    if rsi < 30:  # Oversold - AL
        result = trader.place_order(symbol, 'buy', 0.001)
        if result['success']:
            await context.bot.send_message(
                chat_id=ADMIN_USER_ID,
                text=f"Otomatik ALIŞ yapıldı!\nRSI: {rsi:.2f}"
            )
    elif rsi > 70:  # Overbought - SAT
        result = trader.place_order(symbol, 'sell', 0.001)
        if result['success']:
            await context.bot.send_message(
                chat_id=ADMIN_USER_ID,
                text=f"Otomatik SATIŞ yapıldı!\nRSI: {rsi:.2f}"
            )

# Her 15 dakikada bir çalıştır
job_queue.run_repeating(auto_trade, interval=900, first=10)
```

> Otomatik trading kullanırken mutlaka stop-loss ve take-profit limitleri belirleyin!
{: .prompt-warning }

### Senaryo 2: Portfolio Takibi

```python
async def portfolio_update(context: ContextTypes.DEFAULT_TYPE):
    """Günlük portfolio raporu"""
    balance = trader.get_balance()
    
    total_usdt = 0
    assets = []
    
    for asset, data in balance.items():
        if asset == 'USDT':
            total_usdt += data['total']
        else:
            # Her asset'in USDT karşılığını hesapla
            price = get_current_price(asset)
            usdt_value = data['total'] * price
            total_usdt += usdt_value
            
            assets.append(f"{asset}: {data['total']:.4f} (${usdt_value:,.2f})")
    
    message = f"""
*Günlük Portfolio Raporu*

Toplam Değer: ${total_usdt:,.2f}

Varlıklar:
{chr(10).join(assets)}

Tarih: {datetime.now().strftime('%Y-%m-%d %H:%M')}
    """
    
    await context.bot.send_message(
        chat_id=ADMIN_USER_ID,
        text=message,
        parse_mode='Markdown'
    )

# Her gün saat 09:00'da rapor gönder
job_queue.run_daily(portfolio_update, time=datetime.strptime('09:00', '%H:%M').time())
```

## Sonuç

Bu kapsamlı rehberde, sıfırdan başlayarak profesyonel bir Telegram trading botu geliştirmeyi öğrendik:

- Telegram Bot API entegrasyonu  
- CCXT ile exchange bağlantısı  
- Fiyat alarm sistemi  
- Teknik indikatör hesaplamaları  
- Güvenlik ve yetkilendirme  
- Hata yönetimi ve loglama  
- Cloud deployment

> Trading botları finansal risk içerir. Canlı ortamda kullanmadan önce test ortamında (Binance Testnet) deneyin ve küçük miktarlarla başlayın.
{: .prompt-danger }

> Mutlaka stop-loss mekanizmaları ekleyin ve risk yönetimi stratejileri uygulayın.
{: .prompt-warning }

## İleri Seviye Özellikler

Botunuzu daha da geliştirmek için:
- Machine learning modelleri entegrasyonu
- Çoklu exchange desteği
- Backtesting sistemi
- Web dashboard (Flask/FastAPI)
- Veritabanı entegrasyonu (PostgreSQL)
- Grafik oluşturma (matplotlib, plotly)

> Başarılı trading stratejileri sürekli test ve iyileştirme gerektirir. Piyasa koşullarını takip edin ve botunuzu düzenli olarak güncelleyin.
{: .prompt-tip }
