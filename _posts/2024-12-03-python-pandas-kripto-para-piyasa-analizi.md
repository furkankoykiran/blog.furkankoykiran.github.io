---
title: "Python Pandas ile Kripto Para Piyasa Analizi"
description: "Pandas ile kripto para piyasa analizi. Veri toplama, temizleme, teknik indikatörler, görselleştirme ve trading stratejisi geliştirme."
date: "2024-12-03"
categories:
  - "python"
  - "veri-analizi"
tags:
  - "pandas"
  - "kripto"
  - "analiz"
  - "trading"
  - "veri-bilimi"
  - "matplotlib"
  - "teknik-analiz"
image:
  path: "/assets/img/posts/pandas-crypto-analysis.png"
  alt: "Pandas ile Kripto Para Veri Analizi"
---

Kripto para piyasalarında başarılı olmak için veri analizi vazgeçilmezdir. Bu kapsamlı rehberde, Python'un güçlü veri analiz kütüphanesi Pandas kullanarak kripto para verilerini nasıl toplayacağınızı, temizleyeceğinizi, analiz edeceğinizi ve görselleştireceğinizi öğreneceksiniz.

## Pandas'a Giriş

Pandas, Python'da veri manipülasyonu ve analizi için en popüler kütüphanedir. Kripto piyasa verileri gibi zaman serisi verileriyle çalışmak için idealdir.

### Kurulum

```bash
# Gerekli kütüphaneleri yükleyin
pip install pandas numpy matplotlib seaborn
pip install yfinance ccxt ta-lib requests
pip install plotly jupyter

# TA-Lib için sistem bağımlılıkları (Linux)
sudo apt-get install build-essential
pip install TA-Lib
```

### Temel Veri Yapıları

```python
import pandas as pd
import numpy as np
from datetime import datetime, timedelta

# Series - Tek boyutlu veri
prices = pd.Series([45000, 46000, 45500, 47000], 
                   index=['2024-01-01', '2024-01-02', '2024-01-03', '2024-01-04'])
print(prices)

# DataFrame - İki boyutlu veri
data = {
    'date': ['2024-01-01', '2024-01-02', '2024-01-03'],
    'open': [45000, 46000, 45500],
    'high': [46500, 47000, 46800],
    'low': [44800, 45500, 45000],
    'close': [46000, 45500, 46200],
    'volume': [1000000, 1200000, 950000]
}
df = pd.DataFrame(data)
print(df)
```

## Kripto Veri Kaynakları

### 1. Binance API ile Veri Çekme

```python
import ccxt
import pandas as pd
from datetime import datetime

def fetch_binance_data(symbol='BTC/USDT', timeframe='1h', limit=1000):
    """
    Binance'den OHLCV verisi çeker
    """
    exchange = ccxt.binance({
        'enableRateLimit': True,
        'options': {'defaultType': 'future'}
    })
    
    try:
        # OHLCV verisi çek
        ohlcv = exchange.fetch_ohlcv(symbol, timeframe, limit=limit)
        
        # DataFrame'e dönüştür
        df = pd.DataFrame(ohlcv, columns=['timestamp', 'open', 'high', 'low', 'close', 'volume'])
        
        # Timestamp'i datetime'a çevir
        df['timestamp'] = pd.to_datetime(df['timestamp'], unit='ms')
        df.set_index('timestamp', inplace=True)
        
        # Veri tiplerini düzelt
        for col in ['open', 'high', 'low', 'close', 'volume']:
            df[col] = pd.to_numeric(df[col], errors='coerce')
        
        return df
        
    except Exception as e:
        print(f"Hata: {e}")
        return None

# Kullanım
btc_data = fetch_binance_data('BTC/USDT', '1h', 720)  # Son 30 gün
print(btc_data.head())
print(f"\nVeri şekli: {btc_data.shape}")
print(f"Eksik veri: {btc_data.isnull().sum().sum()}")
```

### 2. CoinGecko API ile Veri

```python
import requests
import pandas as pd

def fetch_coingecko_data(coin_id='bitcoin', days=30, vs_currency='usd'):
    """
    CoinGecko'dan fiyat verisi çeker
    """
    url = f"https://api.coingecko.com/api/v3/coins/{coin_id}/market_chart"
    params = {
        'vs_currency': vs_currency,
        'days': days,
        'interval': 'daily'
    }
    
    response = requests.get(url, params=params)
    data = response.json()
    
    # Fiyat verilerini DataFrame'e dönüştür
    prices = pd.DataFrame(data['prices'], columns=['timestamp', 'price'])
    volumes = pd.DataFrame(data['total_volumes'], columns=['timestamp', 'volume'])
    market_caps = pd.DataFrame(data['market_caps'], columns=['timestamp', 'market_cap'])
    
    # Birleştir
    df = prices.merge(volumes, on='timestamp').merge(market_caps, on='timestamp')
    df['timestamp'] = pd.to_datetime(df['timestamp'], unit='ms')
    df.set_index('timestamp', inplace=True)
    
    return df

# Kullanım
btc_market = fetch_coingecko_data('bitcoin', 90)
eth_market = fetch_coingecko_data('ethereum', 90)

print("Bitcoin Market Data:")
print(btc_market.tail())
```

### 3. Çoklu Exchange Karşılaştırma

```python
def fetch_multi_exchange_data(symbol='BTC/USDT', timeframe='1h', limit=100):
    """
    Birden fazla exchange'den veri çeker ve karşılaştırır
    """
    exchanges = ['binance', 'kraken', 'coinbase']
    dfs = {}
    
    for exchange_name in exchanges:
        try:
            exchange = getattr(ccxt, exchange_name)()
            ohlcv = exchange.fetch_ohlcv(symbol, timeframe, limit=limit)
            df = pd.DataFrame(ohlcv, columns=['timestamp', 'open', 'high', 'low', 'close', 'volume'])
            df['timestamp'] = pd.to_datetime(df['timestamp'], unit='ms')
            df.set_index('timestamp', inplace=True)
            dfs[exchange_name] = df
        except Exception as e:
            print(f"{exchange_name} hatası: {e}")
    
    return dfs

# Karşılaştırma
exchange_data = fetch_multi_exchange_data()

for exchange, df in exchange_data.items():
    print(f"\n{exchange.upper()} - Son fiyat: ${df['close'].iloc[-1]:.2f}")
```

## Veri Temizleme ve Hazırlama

### Eksik Veri Yönetimi

```python
def clean_crypto_data(df):
    """
    Kripto veri setini temizler
    """
    # Kopyasını oluştur
    df_clean = df.copy()
    
    # Eksik verileri kontrol et
    print("Eksik veriler:")
    print(df_clean.isnull().sum())
    
    # Eksik verileri forward fill ile doldur
    df_clean.fillna(method='ffill', inplace=True)
    
    # Hala eksik varsa backward fill
    df_clean.fillna(method='bfill', inplace=True)
    
    # Aykırı değerleri tespit et (IQR yöntemi)
    Q1 = df_clean['close'].quantile(0.25)
    Q3 = df_clean['close'].quantile(0.75)
    IQR = Q3 - Q1
    
    lower_bound = Q1 - 3 * IQR
    upper_bound = Q3 + 3 * IQR
    
    # Aykırı değerleri işaretle
    outliers = (df_clean['close'] < lower_bound) | (df_clean['close'] > upper_bound)
    print(f"\nAykırı değer sayısı: {outliers.sum()}")
    
    # Aykırı değerleri median ile değiştir
    df_clean.loc[outliers, 'close'] = df_clean['close'].median()
    
    # Duplike satırları kaldır
    df_clean = df_clean[~df_clean.index.duplicated(keep='first')]
    
    return df_clean

# Kullanım
btc_clean = clean_crypto_data(btc_data)
print(f"\nTemizlenmiş veri şekli: {btc_clean.shape}")
```

### Veri Resampling

```python
def resample_data(df, freq='1D'):
    """
    Veriyi farklı zaman dilimlerine resamples eder
    """
    resampled = df.resample(freq).agg({
        'open': 'first',
        'high': 'max',
        'low': 'min',
        'close': 'last',
        'volume': 'sum'
    })
    
    return resampled.dropna()

# Saatlik veriyi günlüğe çevir
btc_daily = resample_data(btc_clean, '1D')
print("Günlük veri:")
print(btc_daily.head())

# Haftalık veri
btc_weekly = resample_data(btc_clean, '1W')
print("\nHaftalık veri:")
print(btc_weekly.head())
```

## Temel İstatistiksel Analiz

### Tanımlayıcı İstatistikler

```python
def descriptive_statistics(df):
    """
    Temel istatistikleri hesaplar
    """
    stats = {
        'Ortalama Fiyat': df['close'].mean(),
        'Medyan Fiyat': df['close'].median(),
        'Standart Sapma': df['close'].std(),
        'Minimum': df['close'].min(),
        'Maksimum': df['close'].max(),
        'Değişim Katsayısı': (df['close'].std() / df['close'].mean()) * 100,
        'Çarpıklık': df['close'].skew(),
        'Basıklık': df['close'].kurtosis()
    }
    
    return pd.Series(stats)

# Analiz
btc_stats = descriptive_statistics(btc_daily)
print("Bitcoin İstatistikleri:")
print(btc_stats)

# Getiri hesaplama
btc_daily['returns'] = btc_daily['close'].pct_change()
btc_daily['log_returns'] = np.log(btc_daily['close'] / btc_daily['close'].shift(1))

print(f"\nOrtalama Günlük Getiri: {btc_daily['returns'].mean():.4%}")
print(f"Volatilite (std): {btc_daily['returns'].std():.4%}")
print(f"Sharpe Ratio (annualized): {(btc_daily['returns'].mean() / btc_daily['returns'].std()) * np.sqrt(365):.2f}")
```

### Korelasyon Analizi

```python
def correlation_analysis(symbols, days=90):
    """
    Farklı kripto paralar arası korelasyon analizi
    """
    dfs = []
    
    for symbol in symbols:
        df = fetch_coingecko_data(symbol, days)
        dfs.append(df['price'].rename(symbol))
    
    # Tüm verileri birleştir
    combined = pd.concat(dfs, axis=1)
    
    # Getiri hesapla
    returns = combined.pct_change().dropna()
    
    # Korelasyon matrisi
    correlation = returns.corr()
    
    return correlation, returns

# Analiz
crypto_symbols = ['bitcoin', 'ethereum', 'binancecoin', 'cardano', 'solana']
corr_matrix, returns = correlation_analysis(crypto_symbols, 180)

print("Korelasyon Matrisi:")
print(corr_matrix)

# Görselleştirme
import seaborn as sns
import matplotlib.pyplot as plt

plt.figure(figsize=(10, 8))
sns.heatmap(corr_matrix, annot=True, cmap='coolwarm', center=0, 
            square=True, linewidths=1, fmt='.2f')
plt.title('Kripto Para Korelasyon Matrisi')
plt.tight_layout()
plt.savefig('crypto_correlation.png', dpi=300, bbox_inches='tight')
plt.show()
```

## Teknik Analiz İndikatörleri

### Hareketli Ortalamalar

```python
def calculate_moving_averages(df):
    """
    Çeşitli hareketli ortalamalar hesaplar
    """
    df = df.copy()
    
    # Basit Hareketli Ortalama (SMA)
    df['SMA_20'] = df['close'].rolling(window=20).mean()
    df['SMA_50'] = df['close'].rolling(window=50).mean()
    df['SMA_200'] = df['close'].rolling(window=200).mean()
    
    # Üssel Hareketli Ortalama (EMA)
    df['EMA_12'] = df['close'].ewm(span=12, adjust=False).mean()
    df['EMA_26'] = df['close'].ewm(span=26, adjust=False).mean()
    
    # MACD
    df['MACD'] = df['EMA_12'] - df['EMA_26']
    df['MACD_Signal'] = df['MACD'].ewm(span=9, adjust=False).mean()
    df['MACD_Histogram'] = df['MACD'] - df['MACD_Signal']
    
    # Golden Cross / Death Cross sinyalleri
    df['Golden_Cross'] = (df['SMA_50'] > df['SMA_200']) & (df['SMA_50'].shift(1) <= df['SMA_200'].shift(1))
    df['Death_Cross'] = (df['SMA_50'] < df['SMA_200']) & (df['SMA_50'].shift(1) >= df['SMA_200'].shift(1))
    
    return df

# Hesaplama
btc_with_ma = calculate_moving_averages(btc_daily)

# Golden/Death Cross'ları bul
golden_crosses = btc_with_ma[btc_with_ma['Golden_Cross']].index
death_crosses = btc_with_ma[btc_with_ma['Death_Cross']].index

print(f"Golden Cross sayısı: {len(golden_crosses)}")
print(f"Death Cross sayısı: {len(death_crosses)}")

if len(golden_crosses) > 0:
    print(f"\nSon Golden Cross: {golden_crosses[-1]}")
if len(death_crosses) > 0:
    print(f"Son Death Cross: {death_crosses[-1]}")
```

### RSI (Relative Strength Index)

```python
def calculate_rsi(df, period=14):
    """
    RSI indikatörünü hesaplar
    """
    df = df.copy()
    
    # Fiyat değişimlerini hesapla
    delta = df['close'].diff()
    
    # Kazanç ve kayıpları ayır
    gain = delta.where(delta > 0, 0)
    loss = -delta.where(delta < 0, 0)
    
    # Ortalama kazanç ve kayıp
    avg_gain = gain.rolling(window=period).mean()
    avg_loss = loss.rolling(window=period).mean()
    
    # RS ve RSI
    rs = avg_gain / avg_loss
    df['RSI'] = 100 - (100 / (1 + rs))
    
    # Aşırı alım/satım bölgeleri
    df['RSI_Overbought'] = df['RSI'] > 70
    df['RSI_Oversold'] = df['RSI'] < 30
    
    return df

# Hesaplama
btc_with_rsi = calculate_rsi(btc_with_ma)

# Aşırı alım/satım anlarını bul
overbought = btc_with_rsi[btc_with_rsi['RSI_Overbought']]
oversold = btc_with_rsi[btc_with_rsi['RSI_Oversold']]

print(f"Aşırı alım durumu sayısı: {len(overbought)}")
print(f"Aşırı satım durumu sayısı: {len(oversold)}")

# Son değerler
print(f"\nGüncel RSI: {btc_with_rsi['RSI'].iloc[-1]:.2f}")
```

### Bollinger Bands

```python
def calculate_bollinger_bands(df, period=20, std_dev=2):
    """
    Bollinger Bantlarını hesaplar
    """
    df = df.copy()
    
    # Orta bant (SMA)
    df['BB_Middle'] = df['close'].rolling(window=period).mean()
    
    # Standart sapma
    rolling_std = df['close'].rolling(window=period).std()
    
    # Üst ve alt bantlar
    df['BB_Upper'] = df['BB_Middle'] + (rolling_std * std_dev)
    df['BB_Lower'] = df['BB_Middle'] - (rolling_std * std_dev)
    
    # Bant genişliği
    df['BB_Width'] = df['BB_Upper'] - df['BB_Lower']
    df['BB_Width_Pct'] = (df['BB_Width'] / df['BB_Middle']) * 100
    
    # Fiyatın bantlara göre pozisyonu
    df['BB_Position'] = (df['close'] - df['BB_Lower']) / (df['BB_Upper'] - df['BB_Lower'])
    
    # Sinyaller
    df['BB_Squeeze'] = df['BB_Width_Pct'] < df['BB_Width_Pct'].rolling(100).quantile(0.1)
    df['BB_Upper_Touch'] = df['close'] >= df['BB_Upper']
    df['BB_Lower_Touch'] = df['close'] <= df['BB_Lower']
    
    return df

# Hesaplama
btc_with_bb = calculate_bollinger_bands(btc_with_rsi)

print("Bollinger Bands Özeti:")
print(f"Güncel Pozisyon: {btc_with_bb['BB_Position'].iloc[-1]:.2%}")
print(f"Bant Genişliği: {btc_with_bb['BB_Width_Pct'].iloc[-1]:.2f}%")
print(f"Squeeze durumu: {btc_with_bb['BB_Squeeze'].iloc[-1]}")
```

### Volume Analysis

```python
def analyze_volume(df):
    """
    Volume analizi yapar
    """
    df = df.copy()
    
    # Volume hareketli ortalamaları
    df['Volume_MA_20'] = df['volume'].rolling(window=20).mean()
    df['Volume_Ratio'] = df['volume'] / df['Volume_MA_20']
    
    # Volume trend
    df['Volume_Trend'] = df['volume'].rolling(window=10).apply(
        lambda x: 1 if x.iloc[-1] > x.iloc[0] else -1
    )
    
    # On-Balance Volume (OBV)
    df['OBV'] = (np.sign(df['close'].diff()) * df['volume']).fillna(0).cumsum()
    df['OBV_MA'] = df['OBV'].rolling(window=20).mean()
    
    # Volume spike detection
    df['Volume_Spike'] = df['Volume_Ratio'] > 2.0
    
    # Price-Volume correlation
    df['PV_Correlation'] = df['close'].rolling(window=20).corr(df['volume'])
    
    return df

# Analiz
btc_with_volume = analyze_volume(btc_with_bb)

# Volume spike'ları bul
volume_spikes = btc_with_volume[btc_with_volume['Volume_Spike']]
print(f"Volume spike sayısı: {len(volume_spikes)}")

if len(volume_spikes) > 0:
    print("\nSon 5 Volume Spike:")
    print(volume_spikes[['close', 'volume', 'Volume_Ratio']].tail())
```

## Gelişmiş Analiz Teknikleri

### Volatilite Analizi

```python
def volatility_analysis(df, windows=[7, 14, 30, 60]):
    """
    Farklı zaman dilimlerinde volatilite analizi
    """
    df = df.copy()
    
    # Getiri hesapla (eğer yoksa)
    if 'returns' not in df.columns:
        df['returns'] = df['close'].pct_change()
    
    # Farklı periyotlarda volatilite
    for window in windows:
        col_name = f'Volatility_{window}d'
        df[col_name] = df['returns'].rolling(window=window).std() * np.sqrt(365)
    
    # Parkinson volatilitesi (high-low range kullanarak)
    df['Parkinson_Vol'] = np.sqrt(
        (1 / (4 * np.log(2))) * 
        np.log(df['high'] / df['low']) ** 2
    ).rolling(window=30).mean() * np.sqrt(365)
    
    # Garman-Klass volatilitesi
    df['GK_Vol'] = np.sqrt(
        0.5 * np.log(df['high'] / df['low']) ** 2 -
        (2 * np.log(2) - 1) * np.log(df['close'] / df['open']) ** 2
    ).rolling(window=30).mean() * np.sqrt(365)
    
    return df

# Analiz
btc_with_vol = volatility_analysis(btc_with_volume)

print("Volatilite Özeti:")
for col in ['Volatility_7d', 'Volatility_30d', 'Volatility_60d']:
    print(f"{col}: {btc_with_vol[col].iloc[-1]:.2%}")
```

### Trend Tespiti

```python
def detect_trends(df, window=50):
    """
    Trend yönünü ve gücünü tespit eder
    """
    df = df.copy()
    
    # Lineer regresyon eğimi
    def calculate_slope(series):
        x = np.arange(len(series))
        slope = np.polyfit(x, series, 1)[0]
        return slope
    
    df['Trend_Slope'] = df['close'].rolling(window=window).apply(calculate_slope, raw=False)
    
    # Trend gücü (R-squared)
    def calculate_r_squared(series):
        x = np.arange(len(series))
        slope, intercept = np.polyfit(x, series, 1)
        y_pred = slope * x + intercept
        ss_res = np.sum((series - y_pred) ** 2)
        ss_tot = np.sum((series - series.mean()) ** 2)
        return 1 - (ss_res / ss_tot) if ss_tot != 0 else 0
    
    df['Trend_Strength'] = df['close'].rolling(window=window).apply(calculate_r_squared, raw=False)
    
    # Trend sınıflandırması
    def classify_trend(row):
        if pd.isna(row['Trend_Slope']) or pd.isna(row['Trend_Strength']):
            return 'Unknown'
        
        if row['Trend_Strength'] < 0.5:
            return 'Sideways'
        elif row['Trend_Slope'] > 0:
            return 'Uptrend' if row['Trend_Strength'] > 0.7 else 'Weak Uptrend'
        else:
            return 'Downtrend' if row['Trend_Strength'] > 0.7 else 'Weak Downtrend'
    
    df['Trend'] = df.apply(classify_trend, axis=1)
    
    return df

# Analiz
btc_with_trend = detect_trends(btc_with_vol)

print(f"Güncel Trend: {btc_with_trend['Trend'].iloc[-1]}")
print(f"Trend Gücü: {btc_with_trend['Trend_Strength'].iloc[-1]:.2%}")
print(f"Trend Eğimi: {btc_with_trend['Trend_Slope'].iloc[-1]:.2f}")

# Trend dağılımı
print("\nTrend Dağılımı:")
print(btc_with_trend['Trend'].value_counts())
```

### Support ve Resistance Seviyeleri

```python
def find_support_resistance(df, window=20, num_levels=5):
    """
    Destek ve direnç seviyelerini bulur
    """
    df = df.copy()
    
    # Local maxima ve minima
    df['Local_Max'] = df['high'] == df['high'].rolling(window=window, center=True).max()
    df['Local_Min'] = df['low'] == df['low'].rolling(window=window, center=True).min()
    
    # Direnç seviyeleri
    resistance_levels = df[df['Local_Max']]['high'].nlargest(num_levels).values
    
    # Destek seviyeleri
    support_levels = df[df['Local_Min']]['low'].nsmallest(num_levels).values
    
    return {
        'resistance': sorted(resistance_levels, reverse=True),
        'support': sorted(support_levels),
        'current_price': df['close'].iloc[-1]
    }

# Seviyeleri bul
levels = find_support_resistance(btc_with_trend, window=30, num_levels=5)

print("Direnç Seviyeleri:")
for i, level in enumerate(levels['resistance'], 1):
    distance = ((level / levels['current_price']) - 1) * 100
    print(f"R{i}: ${level:,.2f} ({distance:+.2f}%)")

print("\nDestek Seviyeleri:")
for i, level in enumerate(levels['support'], 1):
    distance = ((level / levels['current_price']) - 1) * 100
    print(f"S{i}: ${level:,.2f} ({distance:+.2f}%)")

print(f"\nGüncel Fiyat: ${levels['current_price']:,.2f}")
```

## Veri Görselleştirme

### Fiyat ve Volume Grafiği

```python
import matplotlib.pyplot as plt
import matplotlib.dates as mdates

def plot_price_volume(df, title="Bitcoin Fiyat ve Volume"):
    """
    Fiyat ve volume grafiği çizer
    """
    fig, (ax1, ax2) = plt.subplots(2, 1, figsize=(14, 8), 
                                     gridspec_kw={'height_ratios': [3, 1]})
    
    # Fiyat grafiği
    ax1.plot(df.index, df['close'], label='Close', color='#1f77b4', linewidth=2)
    
    if 'SMA_20' in df.columns:
        ax1.plot(df.index, df['SMA_20'], label='SMA 20', 
                color='orange', linewidth=1.5, alpha=0.7)
    if 'SMA_50' in df.columns:
        ax1.plot(df.index, df['SMA_50'], label='SMA 50', 
                color='red', linewidth=1.5, alpha=0.7)
    
    ax1.set_title(title, fontsize=16, fontweight='bold')
    ax1.set_ylabel('Fiyat (USD)', fontsize=12)
    ax1.legend(loc='upper left')
    ax1.grid(True, alpha=0.3)
    
    # Volume grafiği
    colors = ['green' if df['close'].iloc[i] >= df['open'].iloc[i] 
              else 'red' for i in range(len(df))]
    ax2.bar(df.index, df['volume'], color=colors, alpha=0.5, width=0.8)
    
    if 'Volume_MA_20' in df.columns:
        ax2.plot(df.index, df['Volume_MA_20'], 
                color='blue', linewidth=2, label='Volume MA 20')
        ax2.legend(loc='upper left')
    
    ax2.set_ylabel('Volume', fontsize=12)
    ax2.set_xlabel('Tarih', fontsize=12)
    ax2.grid(True, alpha=0.3)
    
    # Tarih formatı
    ax1.xaxis.set_major_formatter(mdates.DateFormatter('%Y-%m-%d'))
    ax2.xaxis.set_major_formatter(mdates.DateFormatter('%Y-%m-%d'))
    
    plt.xticks(rotation=45)
    plt.tight_layout()
    plt.savefig('price_volume.png', dpi=300, bbox_inches='tight')
    plt.show()

# Görselleştir
plot_price_volume(btc_with_trend.tail(90))
```

### Teknik İndikatör Grafiği

```python
def plot_technical_indicators(df, title="Teknik İndikatörler"):
    """
    RSI, MACD ve Bollinger Bands grafiği
    """
    fig, (ax1, ax2, ax3) = plt.subplots(3, 1, figsize=(14, 10), 
                                          gridspec_kw={'height_ratios': [2, 1, 1]})
    
    # Fiyat ve Bollinger Bands
    ax1.plot(df.index, df['close'], label='Close', color='blue', linewidth=2)
    
    if 'BB_Upper' in df.columns:
        ax1.plot(df.index, df['BB_Upper'], 'r--', label='BB Upper', alpha=0.7)
        ax1.plot(df.index, df['BB_Middle'], 'g--', label='BB Middle', alpha=0.7)
        ax1.plot(df.index, df['BB_Lower'], 'r--', label='BB Lower', alpha=0.7)
        ax1.fill_between(df.index, df['BB_Lower'], df['BB_Upper'], alpha=0.1)
    
    ax1.set_title(title, fontsize=16, fontweight='bold')
    ax1.set_ylabel('Fiyat (USD)', fontsize=12)
    ax1.legend(loc='upper left')
    ax1.grid(True, alpha=0.3)
    
    # RSI
    if 'RSI' in df.columns:
        ax2.plot(df.index, df['RSI'], label='RSI', color='purple', linewidth=2)
        ax2.axhline(y=70, color='r', linestyle='--', alpha=0.5, label='Overbought')
        ax2.axhline(y=30, color='g', linestyle='--', alpha=0.5, label='Oversold')
        ax2.fill_between(df.index, 30, 70, alpha=0.1)
        ax2.set_ylabel('RSI', fontsize=12)
        ax2.set_ylim(0, 100)
        ax2.legend(loc='upper left')
        ax2.grid(True, alpha=0.3)
    
    # MACD
    if 'MACD' in df.columns:
        ax3.plot(df.index, df['MACD'], label='MACD', color='blue', linewidth=2)
        ax3.plot(df.index, df['MACD_Signal'], label='Signal', color='red', linewidth=2)
        ax3.bar(df.index, df['MACD_Histogram'], label='Histogram', 
               color='gray', alpha=0.3)
        ax3.axhline(y=0, color='black', linestyle='-', alpha=0.3)
        ax3.set_ylabel('MACD', fontsize=12)
        ax3.set_xlabel('Tarih', fontsize=12)
        ax3.legend(loc='upper left')
        ax3.grid(True, alpha=0.3)
    
    plt.xticks(rotation=45)
    plt.tight_layout()
    plt.savefig('technical_indicators.png', dpi=300, bbox_inches='tight')
    plt.show()

# Görselleştir
plot_technical_indicators(btc_with_trend.tail(90))
```

### Candlestick Chart

```python
import plotly.graph_objects as go

def plot_candlestick(df, title="Bitcoin Candlestick Chart"):
    """
    İnteraktif candlestick grafiği oluşturur
    """
    fig = go.Figure(data=[go.Candlestick(
        x=df.index,
        open=df['open'],
        high=df['high'],
        low=df['low'],
        close=df['close'],
        name='OHLC'
    )])
    
    # Hareketli ortalamaları ekle
    if 'SMA_20' in df.columns:
        fig.add_trace(go.Scatter(
            x=df.index, y=df['SMA_20'],
            mode='lines', name='SMA 20',
            line=dict(color='orange', width=1)
        ))
    
    if 'SMA_50' in df.columns:
        fig.add_trace(go.Scatter(
            x=df.index, y=df['SMA_50'],
            mode='lines', name='SMA 50',
            line=dict(color='red', width=1)
        ))
    
    # Layout
    fig.update_layout(
        title=title,
        yaxis_title='Fiyat (USD)',
        xaxis_title='Tarih',
        template='plotly_dark',
        height=600,
        xaxis_rangeslider_visible=False
    )
    
    fig.write_html('candlestick.html')
    fig.show()

# Görselleştir
plot_candlestick(btc_with_trend.tail(90))
```

## Performans Metrikleri

### Backtest Fonksiyonları

```python
def simple_ma_strategy_backtest(df, short_window=20, long_window=50, initial_capital=10000):
    """
    Basit hareketli ortalama stratejisi backtest
    """
    df = df.copy()
    
    # Sinyaller
    df['Short_MA'] = df['close'].rolling(window=short_window).mean()
    df['Long_MA'] = df['close'].rolling(window=long_window).mean()
    
    df['Signal'] = 0
    df['Signal'][short_window:] = np.where(
        df['Short_MA'][short_window:] > df['Long_MA'][short_window:], 1, 0
    )
    df['Position'] = df['Signal'].diff()
    
    # Alım satım simülasyonu
    capital = initial_capital
    position = 0
    trades = []
    
    for i in range(len(df)):
        if df['Position'].iloc[i] == 1:  # Al sinyali
            position = capital / df['close'].iloc[i]
            trades.append({
                'date': df.index[i],
                'type': 'BUY',
                'price': df['close'].iloc[i],
                'amount': position,
                'value': capital
            })
            
        elif df['Position'].iloc[i] == -1 and position > 0:  # Sat sinyali
            capital = position * df['close'].iloc[i]
            trades.append({
                'date': df.index[i],
                'type': 'SELL',
                'price': df['close'].iloc[i],
                'amount': position,
                'value': capital
            })
            position = 0
    
    # Son pozisyonu kapat
    if position > 0:
        capital = position * df['close'].iloc[-1]
    
    # Performans metrikleri
    total_return = (capital - initial_capital) / initial_capital
    trades_df = pd.DataFrame(trades)
    
    results = {
        'Initial Capital': initial_capital,
        'Final Capital': capital,
        'Total Return': total_return,
        'Total Return %': total_return * 100,
        'Number of Trades': len(trades),
        'Buy and Hold Return': ((df['close'].iloc[-1] - df['close'].iloc[0]) / df['close'].iloc[0])
    }
    
    return results, trades_df

# Backtest
results, trades = simple_ma_strategy_backtest(btc_daily, 20, 50, 10000)

print("Backtest Sonuçları:")
for key, value in results.items():
    if isinstance(value, float):
        print(f"{key}: {value:.2f}")
    else:
        print(f"{key}: {value}")

print("\nİşlemler:")
print(trades)
```

## Sonuç

Pandas ile kripto para analizi yapmak, güçlü veri manipülasyon yetenekleri sayesinde oldukça etkilidir. Bu rehberde öğrendikleriniz:

- **Veri Toplama**: Çeşitli API'lerden veri çekme
- **Veri Temizleme**: Eksik veri ve aykırı değer yönetimi
- **İstatistiksel Analiz**: Tanımlayıcı istatistikler ve korelasyon
- **Teknik Analiz**: RSI, MACD, Bollinger Bands, hareketli ortalamalar
- **Görselleştirme**: Matplotlib ve Plotly ile profesyonel grafikler
- **Backtest**: Strateji performans ölçümü

## En İyi Uygulamalar

1. **Veri Kalitesi**: Her zaman veriyi temizleyin ve doğrulayın
2. **Birden Fazla Kaynak**: Tek bir API'ye bağımlı kalmayın
3. **Hata Yönetimi**: API çağrılarında try-except kullanın
4. **Performans**: Büyük veri setlerinde vectorize işlemler kullanın
5. **Dokümantasyon**: Kodunuzu ve analizinizi belgeleyin
6. **Versiyon Kontrolü**: Git ile değişiklikleri takip edin

## Kaynaklar

- [Pandas Documentation](https://pandas.pydata.org/docs/)
- [CCXT Documentation](https://docs.ccxt.com/)
- [TA-Lib Documentation](https://ta-lib.org/)
- [Plotly Documentation](https://plotly.com/python/)

Başarılı analizler! 📊
