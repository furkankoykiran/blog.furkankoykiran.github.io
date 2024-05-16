---
title: "Yapay Zeka Destekli Trading Stratejileri"
date: "2024-05-16 14:00:00 +0300"
categories: [AI & Trading, Machine Learning]
tags: [machine-learning, ai, trading, python, lstm, neural-networks, fintech, prediction]
image:
  src: /assets/img/posts/ai-trading-algorithm-flow.png
  alt: "AI Trading Algorithm Flow"
---

## Giriş

Finansal piyasalarda yapay zeka ve makine öğrenmesi teknolojileri, son yıllarda devrim niteliğinde değişiklikler yaratmaktadır. Geleneksel teknik analiz ve fundamental analiz yöntemlerinin ötesine geçen AI destekli trading stratejileri, büyük veri setlerini işleyebilme, karmaşık kalıpları tanıyabilme ve piyasa dinamiklerini öğrenebilme yetenekleriyle öne çıkmaktadır.

Bu kapsamlı rehberde, makine öğrenmesi algoritmalarını kullanarak nasıl trading stratejileri geliştirebileceğinizi öğreneceksiniz. LSTM (Long Short-Term Memory) sinir ağları, Random Forest algoritması, sentiment analysis ve feature engineering gibi konuları detaylıca ele alacağız. Ayrıca, geliştirdiğiniz modelleri nasıl test edeceğinizi, optimize edeceğinizi ve production ortamına nasıl deploy edeceğinizi de göreceğiz.

![AI Techniques in Financial Trading](/assets/img/posts/ai-techniques-financial-trading.jpg)
*Şekil 1: Finansal trading'de yapay zeka teknikleri genel görünümü*

## Neden Yapay Zeka ve Trading?

### Geleneksel Yöntemlerin Sınırlamaları

Geleneksel trading stratejileri genellikle şu sınırlamalarla karşılaşır:

1. **Sınırlı Veri İşleme**: İnsan analistler günde yalnızca belirli sayıda varlığı analiz edebilir.
2. **Duygusal Kararlar**: Korku ve açgözlülük gibi duygular kararları olumsuz etkileyebilir.
3. **Statik Kurallar**: Piyasa koşulları değiştiğinde manuel stratejiler hızlı adapte olamaz.
4. **Karmaşık İlişkiler**: Çok değişkenli ilişkileri manuel olarak analiz etmek zordur.

### Yapay Zekanın Avantajları

Machine learning modelleri şu avantajları sunar:

- **Büyük Veri İşleme**: Milyonlarca veri noktasını saniyeler içinde analiz edebilir
- **Objektif Kararlar**: Duygusal faktörlerden etkilenmez
- **Adaptif Öğrenme**: Piyasa koşullarına göre kendini güncelleyebilir
- **Karmaşık Kalıp Tanıma**: Görünmeyen ilişkileri keşfedebilir
- **7/24 Çalışma**: Kesintisiz piyasa takibi ve işlem yapabilir

## Veri Toplama ve Hazırlık

### Gerekli Kütüphanelerin Kurulumu

```bash
# Temel kütüphaneler
pip install pandas numpy matplotlib seaborn

# Veri toplama
pip install yfinance ccxt alpha_vantage

# Machine Learning
pip install scikit-learn tensorflow keras

# Teknik indikatörler
pip install ta-lib pandas-ta

# Model değerlendirme ve optimizasyon
pip install optuna mlflow

# Backtesting
pip install backtrader vectorbt
```

### Veri Toplama

```python
import yfinance as yf
import pandas as pd
import numpy as np
from datetime import datetime, timedelta

# Hisse senedi verisi çekme
def get_stock_data(ticker, start_date, end_date):
    """
    Yahoo Finance'den hisse senedi verisi çek
    
    Parameters:
    ticker (str): Hisse senedi sembolü (örn: 'AAPL', 'BTC-USD')
    start_date (str): Başlangıç tarihi (YYYY-MM-DD)
    end_date (str): Bitiş tarihi (YYYY-MM-DD)
    
    Returns:
    pd.DataFrame: OHLCV verisi
    """
    data = yf.download(ticker, start=start_date, end=end_date)
    return data

# Örnek kullanım
ticker = 'BTC-USD'
end_date = datetime.now()
start_date = end_date - timedelta(days=730)  # 2 yıllık veri

df = get_stock_data(
    ticker,
    start_date.strftime('%Y-%m-%d'),
    end_date.strftime('%Y-%m-%d')
)

print(f"Toplam {len(df)} günlük veri çekildi")
print(df.head())
```

### Kripto Para Verileri

```python
import ccxt

# Binance'den veri çekme
def get_crypto_data(symbol, timeframe='1d', limit=365):
    """
    Binance'den kripto para verisi çek
    
    Parameters:
    symbol (str): Trading çifti (örn: 'BTC/USDT')
    timeframe (str): Zaman dilimi ('1m', '5m', '1h', '1d')
    limit (int): Çekilecek mum sayısı
    
    Returns:
    pd.DataFrame: OHLCV verisi
    """
    exchange = ccxt.binance()
    
    # Veriyi çek
    ohlcv = exchange.fetch_ohlcv(symbol, timeframe, limit=limit)
    
    # DataFrame'e dönüştür
    df = pd.DataFrame(
        ohlcv,
        columns=['timestamp', 'open', 'high', 'low', 'close', 'volume']
    )
    
    # Timestamp'i datetime'a çevir
    df['timestamp'] = pd.to_datetime(df['timestamp'], unit='ms')
    df.set_index('timestamp', inplace=True)
    
    return df

# BTC/USDT verisi çek
btc_data = get_crypto_data('BTC/USDT', timeframe='1h', limit=2000)
print(btc_data.tail())
```

## Feature Engineering

Feature engineering, makine öğrenmesi modelinin performansını belirleyen en kritik adımdır.

![Machine Learning Model Flowchart](/assets/img/posts/machine-learning-model-flowchart.png)
*Şekil 2: Machine learning model geliştirme iş akışı*

### Teknik İndikatörlerin Hesaplanması

```python
import pandas_ta as ta

def add_technical_indicators(df):
    """
    DataFrame'e teknik indikatörler ekle
    
    Parameters:
    df (pd.DataFrame): OHLCV verisi
    
    Returns:
    pd.DataFrame: İndikatörler eklenmiş veri
    """
    # Trend indikatörleri
    df['SMA_20'] = ta.sma(df['close'], length=20)
    df['SMA_50'] = ta.sma(df['close'], length=50)
    df['EMA_12'] = ta.ema(df['close'], length=12)
    df['EMA_26'] = ta.ema(df['close'], length=26)
    
    # MACD
    macd = ta.macd(df['close'])
    df['MACD'] = macd['MACD_12_26_9']
    df['MACD_signal'] = macd['MACDs_12_26_9']
    df['MACD_hist'] = macd['MACDh_12_26_9']
    
    # RSI (Relative Strength Index)
    df['RSI'] = ta.rsi(df['close'], length=14)
    
    # Bollinger Bands
    bbands = ta.bbands(df['close'], length=20)
    df['BB_upper'] = bbands['BBU_20_2.0']
    df['BB_middle'] = bbands['BBM_20_2.0']
    df['BB_lower'] = bbands['BBL_20_2.0']
    
    # ATR (Average True Range)
    df['ATR'] = ta.atr(df['high'], df['low'], df['close'], length=14)
    
    # Stochastic Oscillator
    stoch = ta.stoch(df['high'], df['low'], df['close'])
    df['STOCH_k'] = stoch['STOCHk_14_3_3']
    df['STOCH_d'] = stoch['STOCHd_14_3_3']
    
    # Volume indicators
    df['OBV'] = ta.obv(df['close'], df['volume'])
    df['ADX'] = ta.adx(df['high'], df['low'], df['close'])['ADX_14']
    
    return df

# İndikatörleri ekle
df = add_technical_indicators(df)
print(f"Toplam feature sayısı: {len(df.columns)}")
```

### Özel Feature'lar Oluşturma

```python
def create_custom_features(df):
    """
    Özel feature'lar oluştur
    
    Parameters:
    df (pd.DataFrame): Veri seti
    
    Returns:
    pd.DataFrame: Özel feature'lar eklenmiş veri
    """
    # Fiyat değişim oranları
    df['price_change'] = df['close'].pct_change()
    df['price_change_5'] = df['close'].pct_change(periods=5)
    df['price_change_10'] = df['close'].pct_change(periods=10)
    
    # Volatilite
    df['volatility_5'] = df['close'].rolling(window=5).std()
    df['volatility_20'] = df['close'].rolling(window=20).std()
    
    # High-Low spread
    df['hl_spread'] = (df['high'] - df['low']) / df['close']
    
    # Price momentum
    df['momentum_5'] = df['close'] - df['close'].shift(5)
    df['momentum_10'] = df['close'] - df['close'].shift(10)
    
    # Volume features
    df['volume_change'] = df['volume'].pct_change()
    df['volume_ma_ratio'] = df['volume'] / df['volume'].rolling(window=20).mean()
    
    # Price position relative to range
    df['price_position'] = (df['close'] - df['low']) / (df['high'] - df['low'])
    
    # Gap features
    df['gap'] = df['open'] - df['close'].shift(1)
    df['gap_percent'] = df['gap'] / df['close'].shift(1)
    
    # Trend strength
    df['trend_strength'] = (df['close'] - df['SMA_50']) / df['SMA_50']
    
    return df

df = create_custom_features(df)
```

### Hedef Değişken (Target) Oluşturma

```python
def create_target_variable(df, prediction_days=1, threshold=0.02):
    """
    Hedef değişken oluştur (sınıflandırma için)
    
    Parameters:
    df (pd.DataFrame): Veri seti
    prediction_days (int): Kaç gün sonrasını tahmin edeceğiz
    threshold (float): Alım/satım eşiği (örn: 0.02 = %2)
    
    Returns:
    pd.DataFrame: Hedef değişken eklenmiş veri
    """
    # Gelecekteki fiyat değişimi
    df['future_return'] = df['close'].shift(-prediction_days) / df['close'] - 1
    
    # 3 sınıflı hedef: Buy (1), Hold (0), Sell (-1)
    df['target'] = 0  # Hold
    df.loc[df['future_return'] > threshold, 'target'] = 1  # Buy
    df.loc[df['future_return'] < -threshold, 'target'] = -1  # Sell
    
    # Binary sınıflandırma için (sadece up/down)
    df['target_binary'] = (df['future_return'] > 0).astype(int)
    
    # Regresyon için (fiyat değişim yüzdesi)
    df['target_regression'] = df['future_return']
    
    return df

df = create_target_variable(df, prediction_days=1, threshold=0.015)

# Hedef değişken dağılımı
print("\nHedef değişken dağılımı:")
print(df['target'].value_counts())
print(f"\nBuy signals: {(df['target'] == 1).sum()}")
print(f"Hold signals: {(df['target'] == 0).sum()}")
print(f"Sell signals: {(df['target'] == -1).sum()}")
```

## LSTM ile Fiyat Tahmini

LSTM (Long Short-Term Memory) ağları, zaman serisi tahminleri için özellikle güçlüdür.

### Veriyi LSTM için Hazırlama

```python
from sklearn.preprocessing import MinMaxScaler
import tensorflow as tf
from tensorflow import keras
from tensorflow.keras.models import Sequential
from tensorflow.keras.layers import LSTM, Dense, Dropout
from tensorflow.keras.callbacks import EarlyStopping, ReduceLROnPlateau

def prepare_lstm_data(df, features, target, lookback=60, test_size=0.2):
    """
    LSTM için veriyi hazırla
    
    Parameters:
    df (pd.DataFrame): Veri seti
    features (list): Kullanılacak feature'lar
    target (str): Hedef değişken
    lookback (int): Geçmiş kaç günü kullanacağız
    test_size (float): Test seti oranı
    
    Returns:
    tuple: (X_train, X_test, y_train, y_test, scaler)
    """
    # NaN değerleri temizle
    df = df[features + [target]].dropna()
    
    # Feature'ları ölçeklendir
    scaler = MinMaxScaler(feature_range=(0, 1))
    scaled_data = scaler.fit_transform(df[features])
    
    # Hedef değişkeni al
    target_data = df[target].values
    
    # Sequence'ler oluştur
    X, y = [], []
    for i in range(lookback, len(scaled_data)):
        X.append(scaled_data[i-lookback:i])
        y.append(target_data[i])
    
    X, y = np.array(X), np.array(y)
    
    # Train-test split
    split_idx = int(len(X) * (1 - test_size))
    X_train, X_test = X[:split_idx], X[split_idx:]
    y_train, y_test = y[:split_idx], y[split_idx:]
    
    print(f"Training set: {X_train.shape}")
    print(f"Test set: {X_test.shape}")
    
    return X_train, X_test, y_train, y_test, scaler

# Feature'ları seç
feature_columns = [
    'open', 'high', 'low', 'close', 'volume',
    'SMA_20', 'SMA_50', 'RSI', 'MACD',
    'BB_upper', 'BB_lower', 'ATR',
    'price_change', 'volatility_5'
]

X_train, X_test, y_train, y_test, scaler = prepare_lstm_data(
    df,
    features=feature_columns,
    target='target_binary',
    lookback=60
)
```

### LSTM Modeli Oluşturma

```python
def build_lstm_model(input_shape, units=[128, 64, 32], dropout=0.2):
    """
    LSTM modeli oluştur
    
    Parameters:
    input_shape (tuple): Input shape (timesteps, features)
    units (list): Her LSTM layer'ındaki unit sayısı
    dropout (float): Dropout oranı
    
    Returns:
    keras.Model: Derlenmiş LSTM modeli
    """
    model = Sequential()
    
    # İlk LSTM layer
    model.add(LSTM(
        units=units[0],
        return_sequences=True,
        input_shape=input_shape
    ))
    model.add(Dropout(dropout))
    
    # Orta LSTM layer'lar
    for unit in units[1:-1]:
        model.add(LSTM(units=unit, return_sequences=True))
        model.add(Dropout(dropout))
    
    # Son LSTM layer
    model.add(LSTM(units=units[-1], return_sequences=False))
    model.add(Dropout(dropout))
    
    # Output layer (binary classification)
    model.add(Dense(1, activation='sigmoid'))
    
    # Modeli derle
    model.compile(
        optimizer='adam',
        loss='binary_crossentropy',
        metrics=['accuracy', tf.keras.metrics.AUC()]
    )
    
    return model

# Modeli oluştur
model = build_lstm_model(
    input_shape=(X_train.shape[1], X_train.shape[2]),
    units=[128, 64, 32],
    dropout=0.3
)

model.summary()
```

### Model Eğitimi

```python
# Callbacks
early_stopping = EarlyStopping(
    monitor='val_loss',
    patience=15,
    restore_best_weights=True
)

reduce_lr = ReduceLROnPlateau(
    monitor='val_loss',
    factor=0.5,
    patience=5,
    min_lr=1e-7
)

# Modeli eğit
history = model.fit(
    X_train, y_train,
    validation_data=(X_test, y_test),
    epochs=100,
    batch_size=32,
    callbacks=[early_stopping, reduce_lr],
    verbose=1
)

# Eğitim sonuçlarını görselleştir
import matplotlib.pyplot as plt

plt.figure(figsize=(12, 4))

plt.subplot(1, 2, 1)
plt.plot(history.history['loss'], label='Training Loss')
plt.plot(history.history['val_loss'], label='Validation Loss')
plt.title('Model Loss')
plt.xlabel('Epoch')
plt.ylabel('Loss')
plt.legend()

plt.subplot(1, 2, 2)
plt.plot(history.history['accuracy'], label='Training Accuracy')
plt.plot(history.history['val_accuracy'], label='Validation Accuracy')
plt.title('Model Accuracy')
plt.xlabel('Epoch')
plt.ylabel('Accuracy')
plt.legend()

plt.tight_layout()
plt.savefig('lstm_training_history.png')
plt.show()
```

### Model Değerlendirme

```python
from sklearn.metrics import classification_report, confusion_matrix
import seaborn as sns

# Tahminler
y_pred_proba = model.predict(X_test)
y_pred = (y_pred_proba > 0.5).astype(int).flatten()

# Classification report
print("\nClassification Report:")
print(classification_report(y_test, y_pred))

# Confusion matrix
cm = confusion_matrix(y_test, y_pred)
plt.figure(figsize=(8, 6))
sns.heatmap(cm, annot=True, fmt='d', cmap='Blues')
plt.title('Confusion Matrix')
plt.ylabel('Actual')
plt.xlabel('Predicted')
plt.savefig('confusion_matrix.png')
plt.show()

# Accuracy
accuracy = np.mean(y_pred == y_test)
print(f"\nTest Accuracy: {accuracy:.4f}")
```

## Random Forest ile Feature Importance

Random Forest algoritması, hem güçlü tahminler yapar hem de hangi feature'ların önemli olduğunu gösterir.

![ML Trading Architecture](/assets/img/posts/ml-trading-architecture.jpg)
*Şekil 3: Machine learning trading mimarisi*

### Random Forest Modeli

```python
from sklearn.ensemble import RandomForestClassifier
from sklearn.model_selection import train_test_split, cross_val_score
from sklearn.metrics import accuracy_score, precision_score, recall_score, f1_score

def train_random_forest(df, features, target, test_size=0.2):
    """
    Random Forest modeli eğit
    
    Parameters:
    df (pd.DataFrame): Veri seti
    features (list): Feature column'ları
    target (str): Hedef değişken
    test_size (float): Test seti oranı
    
    Returns:
    tuple: (model, X_test, y_test)
    """
    # Veriyi hazırla
    data = df[features + [target]].dropna()
    X = data[features]
    y = data[target]
    
    # Train-test split
    X_train, X_test, y_train, y_test = train_test_split(
        X, y, test_size=test_size, random_state=42, shuffle=False
    )
    
    # Modeli oluştur ve eğit
    rf_model = RandomForestClassifier(
        n_estimators=200,
        max_depth=15,
        min_samples_split=10,
        min_samples_leaf=5,
        max_features='sqrt',
        random_state=42,
        n_jobs=-1,
        class_weight='balanced'
    )
    
    print("Model eğitiliyor...")
    rf_model.fit(X_train, y_train)
    
    # Tahminler
    y_pred = rf_model.predict(X_test)
    
    # Metrikler
    print("\nModel Performansı:")
    print(f"Accuracy: {accuracy_score(y_test, y_pred):.4f}")
    print(f"Precision: {precision_score(y_test, y_pred, average='weighted'):.4f}")
    print(f"Recall: {recall_score(y_test, y_pred, average='weighted'):.4f}")
    print(f"F1-Score: {f1_score(y_test, y_pred, average='weighted'):.4f}")
    
    # Cross-validation
    cv_scores = cross_val_score(rf_model, X_train, y_train, cv=5)
    print(f"\nCross-validation scores: {cv_scores}")
    print(f"Mean CV score: {cv_scores.mean():.4f} (+/- {cv_scores.std() * 2:.4f})")
    
    return rf_model, X_test, y_test

# Feature'ları seç
rf_features = [
    'open', 'high', 'low', 'close', 'volume',
    'SMA_20', 'SMA_50', 'EMA_12', 'EMA_26',
    'RSI', 'MACD', 'MACD_signal', 'MACD_hist',
    'BB_upper', 'BB_middle', 'BB_lower',
    'ATR', 'STOCH_k', 'STOCH_d', 'ADX',
    'price_change', 'price_change_5', 'price_change_10',
    'volatility_5', 'volatility_20',
    'momentum_5', 'momentum_10',
    'volume_change', 'trend_strength'
]

rf_model, X_test_rf, y_test_rf = train_random_forest(
    df,
    features=rf_features,
    target='target'
)
```

### Feature Importance Analizi

```python
def plot_feature_importance(model, feature_names, top_n=20):
    """
    Feature importance'ı görselleştir
    
    Parameters:
    model: Eğitilmiş model
    feature_names (list): Feature isimleri
    top_n (int): Gösterilecek feature sayısı
    """
    # Feature importance'ı al
    importance = model.feature_importances_
    
    # DataFrame oluştur
    feature_importance_df = pd.DataFrame({
        'feature': feature_names,
        'importance': importance
    }).sort_values('importance', ascending=False)
    
    # Top N feature'ı görselleştir
    plt.figure(figsize=(10, 8))
    plt.barh(
        feature_importance_df['feature'][:top_n][::-1],
        feature_importance_df['importance'][:top_n][::-1]
    )
    plt.xlabel('Importance')
    plt.title(f'Top {top_n} Feature Importance')
    plt.tight_layout()
    plt.savefig('feature_importance.png')
    plt.show()
    
    # En önemli feature'ları yazdır
    print("\nTop 10 Most Important Features:")
    print(feature_importance_df.head(10))
    
    return feature_importance_df

feature_importance_df = plot_feature_importance(
    rf_model,
    rf_features,
    top_n=20
)
```

## Sentiment Analysis ile Piyasa Duygu Analizi

Sosyal medya ve haber verilerinden piyasa duygusunu çıkarmak, AI trading stratejilerinin önemli bir parçasıdır.

### Twitter Sentiment Analysis

```python
from textblob import TextBlob
import tweepy
import re

def clean_tweet(tweet):
    """
    Tweet'i temizle
    """
    # URL'leri kaldır
    tweet = re.sub(r'http\S+', '', tweet)
    # Mention'ları kaldır
    tweet = re.sub(r'@\w+', '', tweet)
    # Hashtag'leri kaldır
    tweet = re.sub(r'#\w+', '', tweet)
    # Özel karakterleri kaldır
    tweet = re.sub(r'[^a-zA-Z\s]', '', tweet)
    return tweet.lower().strip()

def analyze_sentiment(text):
    """
    Metni analiz et ve sentiment score hesapla
    
    Returns:
    dict: Sentiment bilgileri
    """
    analysis = TextBlob(text)
    
    # Polarity: -1 (negative) to 1 (positive)
    polarity = analysis.sentiment.polarity
    
    # Sentiment kategorisi
    if polarity > 0.1:
        sentiment = 'positive'
    elif polarity < -0.1:
        sentiment = 'negative'
    else:
        sentiment = 'neutral'
    
    return {
        'polarity': polarity,
        'sentiment': sentiment,
        'subjectivity': analysis.sentiment.subjectivity
    }

# Örnek tweet analizi
sample_tweets = [
    "Bitcoin is going to the moon! 🚀 Best investment ever!",
    "Major crash incoming. Sell everything now!",
    "Just bought more BTC. HODL for the long term.",
    "Market is uncertain. Be careful with your investments."
]

print("Tweet Sentiment Analysis:")
for tweet in sample_tweets:
    cleaned = clean_tweet(tweet)
    sentiment = analyze_sentiment(cleaned)
    print(f"\nTweet: {tweet}")
    print(f"Sentiment: {sentiment['sentiment']} (polarity: {sentiment['polarity']:.2f})")
```

### Sentiment Feature'ı Ekleme

```python
def add_sentiment_features(df, sentiment_data):
    """
    Veri setine sentiment feature'ları ekle
    
    Parameters:
    df (pd.DataFrame): Ana veri seti
    sentiment_data (pd.DataFrame): Sentiment verileri (tarih ve sentiment score)
    
    Returns:
    pd.DataFrame: Sentiment feature'ları eklenmiş veri
    """
    # Sentiment verilerini birleştir
    df = df.merge(sentiment_data, left_index=True, right_index=True, how='left')
    
    # Sentiment değişim oranı
    df['sentiment_change'] = df['sentiment_score'].diff()
    
    # Sentiment moving average
    df['sentiment_ma_7'] = df['sentiment_score'].rolling(window=7).mean()
    
    # Sentiment momentum
    df['sentiment_momentum'] = df['sentiment_score'] - df['sentiment_ma_7']
    
    # NaN değerleri doldur
    df['sentiment_score'].fillna(0, inplace=True)
    df['sentiment_change'].fillna(0, inplace=True)
    
    return df

# Örnek sentiment verileri oluştur (gerçek uygulamada API'den alınır)
np.random.seed(42)
sentiment_data = pd.DataFrame({
    'sentiment_score': np.random.uniform(-0.5, 0.5, len(df))
}, index=df.index)

df = add_sentiment_features(df, sentiment_data)
```

## Model Optimizasyonu (Hyperparameter Tuning)

Optuna kullanarak model hiperparametrelerini optimize edelim.

![Deep Reinforcement Learning Trading](/assets/img/posts/deep-reinforcement-learning-trading.png)
*Şekil 4: Deep reinforcement learning ile trading sistemi mimarisi*

### Optuna ile Random Forest Optimizasyonu

```python
import optuna
from sklearn.model_selection import cross_val_score

def objective(trial, X, y):
    """
    Optuna objective fonksiyonu
    """
    # Hiperparametre önerileri
    params = {
        'n_estimators': trial.suggest_int('n_estimators', 50, 300),
        'max_depth': trial.suggest_int('max_depth', 5, 30),
        'min_samples_split': trial.suggest_int('min_samples_split', 2, 20),
        'min_samples_leaf': trial.suggest_int('min_samples_leaf', 1, 10),
        'max_features': trial.suggest_categorical('max_features', ['sqrt', 'log2']),
        'random_state': 42,
        'n_jobs': -1
    }
    
    # Modeli oluştur
    model = RandomForestClassifier(**params)
    
    # Cross-validation skoru
    score = cross_val_score(
        model, X, y,
        cv=5,
        scoring='f1_weighted',
        n_jobs=-1
    ).mean()
    
    return score

# Veriyi hazırla
data = df[rf_features + ['target']].dropna()
X = data[rf_features]
y = data['target']

# Optimization study
print("Hiperparametre optimizasyonu başlıyor...")
study = optuna.create_study(direction='maximize')
study.optimize(
    lambda trial: objective(trial, X, y),
    n_trials=50,
    show_progress_bar=True
)

# En iyi parametreler
print("\nEn iyi hiperparametreler:")
print(study.best_params)
print(f"\nEn iyi F1 score: {study.best_value:.4f}")

# Optimize edilmiş model
best_rf_model = RandomForestClassifier(
    **study.best_params,
    random_state=42,
    n_jobs=-1
)
```

### LSTM Hyperparameter Tuning

```python
def create_lstm_model_optuna(trial, input_shape):
    """
    Optuna ile LSTM modeli oluştur
    """
    # Hiperparametreler
    n_layers = trial.suggest_int('n_layers', 1, 4)
    dropout = trial.suggest_float('dropout', 0.1, 0.5)
    learning_rate = trial.suggest_float('learning_rate', 1e-5, 1e-2, log=True)
    
    # Model oluştur
    model = Sequential()
    
    # İlk layer
    first_units = trial.suggest_int('units_0', 32, 256)
    model.add(LSTM(
        units=first_units,
        return_sequences=(n_layers > 1),
        input_shape=input_shape
    ))
    model.add(Dropout(dropout))
    
    # Diğer layer'lar
    for i in range(1, n_layers):
        units = trial.suggest_int(f'units_{i}', 16, 128)
        return_sequences = (i < n_layers - 1)
        model.add(LSTM(units=units, return_sequences=return_sequences))
        model.add(Dropout(dropout))
    
    # Output layer
    model.add(Dense(1, activation='sigmoid'))
    
    # Compile
    optimizer = keras.optimizers.Adam(learning_rate=learning_rate)
    model.compile(
        optimizer=optimizer,
        loss='binary_crossentropy',
        metrics=['accuracy']
    )
    
    return model

def objective_lstm(trial):
    """
    LSTM için objective fonksiyonu
    """
    # Model oluştur
    model = create_lstm_model_optuna(
        trial,
        input_shape=(X_train.shape[1], X_train.shape[2])
    )
    
    # Eğit
    history = model.fit(
        X_train, y_train,
        validation_split=0.2,
        epochs=30,
        batch_size=trial.suggest_int('batch_size', 16, 128),
        verbose=0
    )
    
    # Validation accuracy
    return history.history['val_accuracy'][-1]

# LSTM optimizasyonu
print("\nLSTM hiperparametre optimizasyonu...")
lstm_study = optuna.create_study(direction='maximize')
lstm_study.optimize(objective_lstm, n_trials=20, show_progress_bar=True)

print("\nEn iyi LSTM hiperparametreleri:")
print(lstm_study.best_params)
print(f"En iyi validation accuracy: {lstm_study.best_value:.4f}")
```

## Ensemble Learning

Birden fazla modeli birleştirerek daha güçlü tahminler elde edebiliriz.

```python
from sklearn.ensemble import VotingClassifier
from sklearn.linear_model import LogisticRegression
from sklearn.svm import SVC
import xgboost as xgb

def create_ensemble_model(X_train, y_train):
    """
    Ensemble model oluştur
    
    Returns:
    VotingClassifier: Ensemble modeli
    """
    # Temel modeller
    rf = RandomForestClassifier(
        n_estimators=200,
        max_depth=15,
        random_state=42
    )
    
    xgb_model = xgb.XGBClassifier(
        n_estimators=200,
        max_depth=7,
        learning_rate=0.1,
        random_state=42
    )
    
    lr = LogisticRegression(
        max_iter=1000,
        random_state=42
    )
    
    # Ensemble (soft voting)
    ensemble = VotingClassifier(
        estimators=[
            ('rf', rf),
            ('xgb', xgb_model),
            ('lr', lr)
        ],
        voting='soft',
        n_jobs=-1
    )
    
    print("Ensemble model eğitiliyor...")
    ensemble.fit(X_train, y_train)
    
    return ensemble

# Veriyi hazırla
data = df[rf_features + ['target']].dropna()
X = data[rf_features]
y = data['target']

X_train_ens, X_test_ens, y_train_ens, y_test_ens = train_test_split(
    X, y, test_size=0.2, random_state=42, shuffle=False
)

# Ensemble model
ensemble_model = create_ensemble_model(X_train_ens, y_train_ens)

# Tahmin ve değerlendirme
y_pred_ensemble = ensemble_model.predict(X_test_ens)
ensemble_accuracy = accuracy_score(y_test_ens, y_pred_ensemble)

print(f"\nEnsemble Model Accuracy: {ensemble_accuracy:.4f}")
print("\nClassification Report:")
print(classification_report(y_test_ens, y_pred_ensemble))
```

## Backtesting ve Performans Analizi

Geliştirdiğimiz stratejinin gerçek performansını test edelim.

![AI Stock Trading Visualization](/assets/img/posts/ai-stock-trading-visualization.jpg)
*Şekil 5: AI destekli trading stratejisi görselleştirmesi*

### Basit Backtesting

```python
def backtest_strategy(df, predictions, initial_capital=10000):
    """
    Trading stratejisini backtest et
    
    Parameters:
    df (pd.DataFrame): Fiyat verileri
    predictions (np.array): Model tahminleri (1: Buy, 0: Hold, -1: Sell)
    initial_capital (float): Başlangıç sermayesi
    
    Returns:
    dict: Backtest sonuçları
    """
    capital = initial_capital
    position = 0  # 0: cash, 1: long
    entry_price = 0
    trades = []
    portfolio_value = []
    
    for i in range(len(predictions)):
        current_price = df.iloc[i]['close']
        signal = predictions[i]
        
        # Buy signal
        if signal == 1 and position == 0:
            position = 1
            entry_price = current_price
            shares = capital / current_price
            trades.append({
                'type': 'BUY',
                'price': current_price,
                'date': df.index[i],
                'capital': capital
            })
        
        # Sell signal
        elif signal == -1 and position == 1:
            position = 0
            capital = shares * current_price
            profit = capital - trades[-1]['capital']
            trades.append({
                'type': 'SELL',
                'price': current_price,
                'date': df.index[i],
                'capital': capital,
                'profit': profit,
                'profit_pct': (profit / trades[-1]['capital']) * 100
            })
        
        # Portfolio value
        if position == 1:
            portfolio_value.append(shares * current_price)
        else:
            portfolio_value.append(capital)
    
    # Final sell if still in position
    if position == 1:
        final_price = df.iloc[-1]['close']
        capital = shares * final_price
        profit = capital - trades[-1]['capital']
        trades.append({
            'type': 'SELL',
            'price': final_price,
            'date': df.index[-1],
            'capital': capital,
            'profit': profit,
            'profit_pct': (profit / trades[-1]['capital']) * 100
        })
        portfolio_value.append(capital)
    
    # Metrikleri hesapla
    total_return = ((capital - initial_capital) / initial_capital) * 100
    num_trades = len([t for t in trades if t['type'] == 'SELL'])
    
    if num_trades > 0:
        winning_trades = len([t for t in trades if t['type'] == 'SELL' and t['profit'] > 0])
        win_rate = (winning_trades / num_trades) * 100
        avg_profit = np.mean([t['profit_pct'] for t in trades if t['type'] == 'SELL'])
    else:
        win_rate = 0
        avg_profit = 0
    
    # Buy & Hold stratejisi
    buy_hold_return = ((df.iloc[-1]['close'] - df.iloc[0]['close']) / df.iloc[0]['close']) * 100
    
    results = {
        'initial_capital': initial_capital,
        'final_capital': capital,
        'total_return': total_return,
        'buy_hold_return': buy_hold_return,
        'num_trades': num_trades,
        'win_rate': win_rate,
        'avg_profit_per_trade': avg_profit,
        'trades': trades,
        'portfolio_value': portfolio_value
    }
    
    return results

# Backtest yap
test_data = df.iloc[-len(y_pred_ensemble):].copy()
backtest_results = backtest_strategy(test_data, y_pred_ensemble, initial_capital=10000)

print("\n" + "="*50)
print("BACKTEST SONUÇLARI")
print("="*50)
print(f"Başlangıç Sermayesi: ${backtest_results['initial_capital']:,.2f}")
print(f"Final Sermaye: ${backtest_results['final_capital']:,.2f}")
print(f"Toplam Getiri: {backtest_results['total_return']:.2f}%")
print(f"Buy & Hold Getiri: {backtest_results['buy_hold_return']:.2f}%")
print(f"İşlem Sayısı: {backtest_results['num_trades']}")
print(f"Kazanma Oranı: {backtest_results['win_rate']:.2f}%")
print(f"Ortalama İşlem Karı: {backtest_results['avg_profit_per_trade']:.2f}%")
print("="*50)
```

### Backtest Görselleştirme

```python
def plot_backtest_results(df, portfolio_value, trades):
    """
    Backtest sonuçlarını görselleştir
    """
    fig, (ax1, ax2) = plt.subplots(2, 1, figsize=(15, 10))
    
    # Fiyat ve trade sinyalleri
    ax1.plot(df.index, df['close'], label='Price', alpha=0.7)
    
    # Buy signals
    buy_trades = [t for t in trades if t['type'] == 'BUY']
    if buy_trades:
        buy_dates = [t['date'] for t in buy_trades]
        buy_prices = [t['price'] for t in buy_trades]
        ax1.scatter(buy_dates, buy_prices, marker='^', color='green', 
                   s=100, label='Buy', zorder=5)
    
    # Sell signals
    sell_trades = [t for t in trades if t['type'] == 'SELL']
    if sell_trades:
        sell_dates = [t['date'] for t in sell_trades]
        sell_prices = [t['price'] for t in sell_trades]
        ax1.scatter(sell_dates, sell_prices, marker='v', color='red', 
                   s=100, label='Sell', zindex=5)
    
    ax1.set_title('Trading Signals')
    ax1.set_ylabel('Price')
    ax1.legend()
    ax1.grid(True, alpha=0.3)
    
    # Portfolio değeri
    ax2.plot(df.index[-len(portfolio_value):], portfolio_value, 
            label='Portfolio Value', color='blue', linewidth=2)
    ax2.set_title('Portfolio Value Over Time')
    ax2.set_xlabel('Date')
    ax2.set_ylabel('Portfolio Value ($)')
    ax2.legend()
    ax2.grid(True, alpha=0.3)
    
    plt.tight_layout()
    plt.savefig('backtest_results.png', dpi=300)
    plt.show()

plot_backtest_results(
    test_data,
    backtest_results['portfolio_value'],
    backtest_results['trades']
)
```

## Risk Yönetimi

### Position Sizing

```python
def calculate_position_size(
    capital,
    risk_per_trade=0.02,
    stop_loss_pct=0.05,
    confidence=0.5
):
    """
    Position size hesapla
    
    Parameters:
    capital (float): Toplam sermaye
    risk_per_trade (float): İşlem başına risk (% olarak)
    stop_loss_pct (float): Stop loss yüzdesi
    confidence (float): Model güven skoru (0-1)
    
    Returns:
    float: Position size
    """
    # Kelly Criterion modifikasyonu
    risk_amount = capital * risk_per_trade
    position_size = (risk_amount / stop_loss_pct) * confidence
    
    # Maximum %20 sermaye
    max_position = capital * 0.20
    position_size = min(position_size, max_position)
    
    return position_size

# Örnek kullanım
capital = 10000
confidence_score = 0.75  # Model güven skoru

position_size = calculate_position_size(
    capital=capital,
    risk_per_trade=0.02,
    stop_loss_pct=0.05,
    confidence=confidence_score
)

print(f"Önerilen Position Size: ${position_size:.2f}")
print(f"Sermayenin %{(position_size/capital)*100:.2f}'si")
```

### Stop Loss ve Take Profit

```python
def calculate_stop_loss_take_profit(entry_price, volatility, risk_reward_ratio=2):
    """
    Stop loss ve take profit seviyelerini hesapla
    
    Parameters:
    entry_price (float): Giriş fiyatı
    volatility (float): ATR veya volatilite değeri
    risk_reward_ratio (float): Risk/ödül oranı
    
    Returns:
    dict: Stop loss ve take profit seviyeleri
    """
    # Stop loss (2x ATR)
    stop_loss = entry_price - (2 * volatility)
    
    # Take profit (risk_reward_ratio * risk)
    risk = entry_price - stop_loss
    take_profit = entry_price + (risk * risk_reward_ratio)
    
    return {
        'entry': entry_price,
        'stop_loss': stop_loss,
        'take_profit': take_profit,
        'risk': risk,
        'reward': take_profit - entry_price,
        'risk_reward_ratio': risk_reward_ratio
    }

# Örnek kullanım
entry_price = 50000
atr = 1500

levels = calculate_stop_loss_take_profit(entry_price, atr, risk_reward_ratio=2.5)

print("\nStop Loss & Take Profit Seviyeleri:")
print(f"Giriş: ${levels['entry']:,.2f}")
print(f"Stop Loss: ${levels['stop_loss']:,.2f}")
print(f"Take Profit: ${levels['take_profit']:,.2f}")
print(f"Risk: ${levels['risk']:,.2f}")
print(f"Ödül: ${levels['reward']:,.2f}")
print(f"Risk/Ödül Oranı: 1:{levels['risk_reward_ratio']}")
```

## Model Deployment ve Gerçek Zamanlı Trading

### Model Kaydetme

```python
import joblib
import json

def save_models(models_dict, path='models/'):
    """
    Eğitilmiş modelleri kaydet
    
    Parameters:
    models_dict (dict): Model isimleri ve modeller
    path (str): Kayıt dizini
    """
    import os
    os.makedirs(path, exist_ok=True)
    
    for name, model in models_dict.items():
        if 'keras' in str(type(model)):
            # Keras modelleri
            model.save(f'{path}{name}_model.h5')
            print(f"Keras model kaydedildi: {name}")
        else:
            # Scikit-learn modelleri
            joblib.dump(model, f'{path}{name}_model.pkl')
            print(f"Scikit-learn model kaydedildi: {name}")
    
    # Scaler'ı kaydet
    if 'scaler' in models_dict:
        joblib.dump(models_dict['scaler'], f'{path}scaler.pkl')
        print("Scaler kaydedildi")

# Modelleri kaydet
models_to_save = {
    'lstm': model,
    'random_forest': rf_model,
    'ensemble': ensemble_model,
    'scaler': scaler
}

save_models(models_to_save)
```

### Gerçek Zamanlı Tahmin Sistemi

```python
class TradingBot:
    """
    Gerçek zamanlı AI trading botu
    """
    
    def __init__(self, models_path='models/'):
        """
        Bot'u başlat ve modelleri yükle
        """
        self.lstm_model = keras.models.load_model(f'{models_path}lstm_model.h5')
        self.rf_model = joblib.load(f'{models_path}random_forest_model.pkl')
        self.ensemble_model = joblib.load(f'{models_path}ensemble_model.pkl')
        self.scaler = joblib.load(f'{models_path}scaler.pkl')
        
        self.position = None
        self.capital = 10000
        
        print("Trading bot başlatıldı!")
    
    def fetch_latest_data(self, ticker, lookback=60):
        """
        En son verileri çek ve feature'ları hesapla
        """
        end_date = datetime.now()
        start_date = end_date - timedelta(days=lookback + 30)
        
        df = get_stock_data(
            ticker,
            start_date.strftime('%Y-%m-%d'),
            end_date.strftime('%Y-%m-%d')
        )
        
        # Feature'ları ekle
        df = add_technical_indicators(df)
        df = create_custom_features(df)
        
        return df
    
    def make_prediction(self, df, features):
        """
        Ensemble tahmin yap
        """
        # En son veriyi al
        latest_data = df[features].iloc[-1:].values
        
        # Tahminler
        rf_pred = self.rf_model.predict(latest_data)[0]
        ensemble_pred = self.ensemble_model.predict(latest_data)[0]
        
        # LSTM için veriyi hazırla
        lstm_data = self.scaler.transform(df[features].iloc[-60:].values)
        lstm_data = lstm_data.reshape(1, 60, len(features))
        lstm_pred = (self.lstm_model.predict(lstm_data)[0][0] > 0.5).astype(int)
        
        # Voting
        predictions = [rf_pred, ensemble_pred, lstm_pred]
        final_prediction = max(set(predictions), key=predictions.count)
        
        confidence = predictions.count(final_prediction) / len(predictions)
        
        return final_prediction, confidence
    
    def execute_trade(self, signal, confidence, current_price):
        """
        İşlem gerçekleştir
        """
        if signal == 1 and confidence > 0.66 and self.position is None:
            # BUY
            position_size = calculate_position_size(
                self.capital,
                confidence=confidence
            )
            shares = position_size / current_price
            
            print(f"\n🟢 BUY SIGNAL")
            print(f"Price: ${current_price:,.2f}")
            print(f"Confidence: {confidence:.2%}")
            print(f"Position Size: ${position_size:,.2f}")
            print(f"Shares: {shares:.4f}")
            
            self.position = {
                'entry_price': current_price,
                'shares': shares,
                'entry_time': datetime.now()
            }
        
        elif signal == -1 and self.position is not None:
            # SELL
            profit = (current_price - self.position['entry_price']) * self.position['shares']
            profit_pct = (profit / (self.position['entry_price'] * self.position['shares'])) * 100
            
            print(f"\n🔴 SELL SIGNAL")
            print(f"Price: ${current_price:,.2f}")
            print(f"Profit: ${profit:,.2f} ({profit_pct:.2f}%)")
            
            self.capital += (self.position['entry_price'] * self.position['shares']) + profit
            self.position = None
    
    def run(self, ticker, features, interval=60):
        """
        Bot'u çalıştır
        
        Parameters:
        ticker (str): İşlem yapılacak varlık
        features (list): Kullanılacak feature'lar
        interval (int): Kontrol aralığı (saniye)
        """
        print(f"\nBot çalışıyor: {ticker}")
        print(f"Kontrol aralığı: {interval} saniye")
        print(f"Başlangıç sermayesi: ${self.capital:,.2f}")
        print("-" * 50)
        
        while True:
            try:
                # Veriyi çek
                df = self.fetch_latest_data(ticker)
                current_price = df['close'].iloc[-1]
                
                # Tahmin yap
                signal, confidence = self.make_prediction(df, features)
                
                # İşlem yap
                self.execute_trade(signal, confidence, current_price)
                
                # Bekle
                time.sleep(interval)
                
            except KeyboardInterrupt:
                print("\n\nBot durduruldu")
                break
            except Exception as e:
                print(f"\nHata: {e}")
                time.sleep(interval)

# Bot'u başlat (örnek - gerçek kullanımda dikkatli olun!)
# bot = TradingBot()
# bot.run('BTC-USD', features=rf_features, interval=300)  # 5 dakikada bir kontrol
```

## Best Practices ve Uyarılar

### Yapılması Gerekenler ✅

1. **Kapsamlı Backtesting**: Stratejiyi farklı piyasa koşullarında test edin
2. **Walk-Forward Analysis**: Modeli periyodik olarak yeniden eğitin
3. **Risk Yönetimi**: Her zaman stop loss kullanın
4. **Diversifikasyon**: Tek bir varlığa bağlı kalmayın
5. **Paper Trading**: Gerçek para kullanmadan önce test edin
6. **Model Monitoring**: Model performansını sürekli izleyin

### Yapılmaması Gerekenler ❌

1. **Overfitting**: Geçmiş verilere aşırı uyum sağlamayın
2. **Aşırı Güven**: %100 doğru tahmin yoktur
3. **Leveraj**: Yüksek kaldıraç risklidir
4. **Emotional Trading**: Otomasyona güvenin ama aklınızı da kullanın
5. **Yetersiz Veri**: Az veriyle model eğitmeyin
6. **İhmal**: Sistemi kurup unutmayın

### Yaygın Hatalar

```python
# YANLIŞ: Look-ahead bias
df['target'] = (df['close'].shift(-1) > df['close']).astype(int)
df['ma_20'] = df['close'].rolling(20).mean()
# Bu hatalı çünkü gelecekteki veriyi kullanıyoruz

# DOĞRU: Geçmiş veri kullanımı
df['ma_20'] = df['close'].rolling(20).mean()
df['target'] = (df['close'].shift(-1) > df['close']).astype(int)
# Target oluşturmadan önce feature'ları hesaplayın

# YANLIŞ: Train-test split shuffle
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, shuffle=True  # Zaman serisi için yanlış!
)

# DOĞRU: Zaman sırasını koru
split_idx = int(len(X) * 0.8)
X_train, X_test = X[:split_idx], X[split_idx:]
y_train, y_test = y[:split_idx], y[split_idx:]
```

## Sonuç

Yapay zeka destekli trading stratejileri, finansal piyasalarda önemli avantajlar sağlayabilir ancak risk yönetimi ve dikkatli uygulama gerektirir. Bu rehberde öğrendikleriniz:

1. **Veri Hazırlığı**: Kaliteli veri toplama ve feature engineering'in önemi
2. **Model Geliştirme**: LSTM, Random Forest ve Ensemble öğrenme teknikleri
3. **Sentiment Analysis**: Sosyal medya ve haber verilerinden insight çıkarma
4. **Optimizasyon**: Hyperparameter tuning ile model performansını artırma
5. **Backtesting**: Stratejinizi gerçekçi şekilde test etme
6. **Risk Yönetimi**: Position sizing, stop loss ve take profit stratejileri
7. **Deployment**: Gerçek zamanlı trading bot oluşturma

**Önemli Hatırlatma**: Bu rehber eğitim amaçlıdır. Gerçek para ile trading yapmadan önce:
- Kapsamlı paper trading yapın
- Risk yönetimi stratejinizi netleştirin
- Yasal düzenlemeleri kontrol edin
- Profesyonel danışmanlık alın

Machine learning modelleri güçlüdür ancak %100 doğru tahmin yapamazlar. Her zaman sorumlu trading yapın ve kaybetmeyi göze alamayacağınız parayı riske atmayın.

## Kaynaklar

### Kütüphaneler ve Araçlar
- [TensorFlow/Keras](https://www.tensorflow.org/) - Deep learning framework
- [scikit-learn](https://scikit-learn.org/) - Machine learning kütüphanesi
- [Pandas](https://pandas.pydata.org/) - Veri analizi
- [TA-Lib](https://ta-lib.org/) - Teknik analiz indikatörleri
- [Optuna](https://optuna.org/) - Hyperparameter optimization
- [Backtrader](https://www.backtrader.com/) - Backtesting framework

### Öğrenme Kaynakları
- [Machine Learning for Trading Specialization (Coursera)](https://www.coursera.org/specializations/machine-learning-trading)
- [Quantitative Trading: How to Build Your Own Algorithmic Trading Business](https://www.amazon.com/Quantitative-Trading-Build-Algorithmic-Business/dp/1119800064)
- [Advances in Financial Machine Learning by Marcos Lopez de Prado](https://www.amazon.com/Advances-Financial-Machine-Learning-Marcos/dp/1119482089)

### Topluluklar
- [Quantopian Forum](https://www.quantopian.com/posts) - Quant trading topluluğu
- [r/algotrading](https://www.reddit.com/r/algotrading/) - Reddit topluluğu
- [QuantConnect Community](https://www.quantconnect.com/forum) - Algo trading platformu

### Veri Kaynakları
- [Yahoo Finance](https://finance.yahoo.com/) - Ücretsiz finansal veri
- [Alpha Vantage](https://www.alphavantage.co/) - API hizmeti
- [Quandl](https://www.quandl.com/) - Finansal veri sağlayıcı
- [CryptoCompare](https://www.cryptocompare.com/) - Kripto veri API

Başarılı ve güvenli trading'ler dileriz! 🚀📈
