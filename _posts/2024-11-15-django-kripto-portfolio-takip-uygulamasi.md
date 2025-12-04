---
title: "Django ile Kripto Portfolio Takip Uygulaması Geliştirme"
description: "Django ile tam özellikli kripto portfolio takip uygulaması. REST API, Celery ile asenkron işlemler, Redis cache ve gerçek zamanlı fiyat takibi."
date: 2024-11-15 09:30:00 +0300
categories: [Web Development, Django]
tags: [python, django, kripto, portfolio, web-app, rest-api, database]
image:
  path: /assets/img/posts/crypto-portfolio-tracker-dashboard.jpg
  alt: "Kripto Portfolio Takip Dashboard Arayüzü"
---

Kripto para yatırımlarınızı takip etmek, kar/zarar hesaplamalarını yapmak ve portföyünüzü yönetmek için profesyonel bir web uygulaması geliştirmek istiyorsanız Django tam size göre. Bu yazıda, gerçek zamanlı fiyat güncellemeleri, kullanıcı kimlik doğrulama ve detaylı portfolio analitiği içeren kapsamlı bir kripto takip uygulaması oluşturacağız.

## Django Neden Portfolio Uygulamaları İçin İdeal?

Django, Python tabanlı güçlü bir web framework'üdür ve finansal uygulamalar için birçok avantaj sunar:

- **ORM (Object-Relational Mapping)**: Veritabanı işlemlerini Python koduna dönüştürür
- **Built-in Authentication**: Kullanıcı yönetimi hazır gelir
- **Admin Panel**: Veritabanı yönetimi için otomatik arayüz
- **Security Features**: CSRF, XSS, SQL injection koruması
- **REST Framework**: API geliştirmek için güçlü araçlar
- **Scalability**: Büyük ölçekli uygulamalar için uygun mimari

![Django MVT Architecture](/assets/img/posts/django-mvt-architecture-diagram.jpg)
_Django'nun Model-View-Template mimarisi, temiz ve bakımı kolay kod yazmayı sağlar_

## Proje Yapısı ve Kurulum

### Django Projesini Oluşturma

Öncelikle gerekli paketleri kuralım:

```bash
# Sanal ortam oluşturma
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

# Django ve gerekli paketleri kurma
pip install django djangorestframework
pip install python-decouple requests
pip install django-cors-headers celery redis
pip install psycopg2-binary  # PostgreSQL için

# Proje oluşturma
django-admin startproject crypto_portfolio
cd crypto_portfolio

# Uygulama oluşturma
python manage.py startapp portfolio
python manage.py startapp accounts
```

### Proje Ayarları (settings.py)

```python
# crypto_portfolio/settings.py
from decouple import config
from pathlib import Path

BASE_DIR = Path(__file__).resolve().parent.parent

# Güvenlik ayarları
SECRET_KEY = config('SECRET_KEY', default='django-insecure-key-for-development')
DEBUG = config('DEBUG', default=True, cast=bool)
ALLOWED_HOSTS = config('ALLOWED_HOSTS', default='localhost,127.0.0.1').split(',')

# Uygulama kayıtları
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    
    # Third party apps
    'rest_framework',
    'corsheaders',
    
    # Local apps
    'accounts',
    'portfolio',
]

MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'corsheaders.middleware.CorsMiddleware',  # CORS için
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
]

# Database - PostgreSQL kullanımı (production için önerilen)
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': config('DB_NAME', default='crypto_portfolio'),
        'USER': config('DB_USER', default='postgres'),
        'PASSWORD': config('DB_PASSWORD', default='password'),
        'HOST': config('DB_HOST', default='localhost'),
        'PORT': config('DB_PORT', default='5432'),
    }
}

# REST Framework ayarları
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.SessionAuthentication',
        'rest_framework.authentication.TokenAuthentication',
    ],
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticated',
    ],
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
    'PAGE_SIZE': 20,
}

# Celery ayarları (arka plan görevleri için)
CELERY_BROKER_URL = config('REDIS_URL', default='redis://localhost:6379/0')
CELERY_RESULT_BACKEND = CELERY_BROKER_URL
CELERY_ACCEPT_CONTENT = ['json']
CELERY_TASK_SERIALIZER = 'json'
CELERY_RESULT_SERIALIZER = 'json'
```

## Database Modelleri

Portfolio uygulaması için ihtiyacımız olan ana modeller:

![Django Database Models](/assets/img/posts/django-database-models-erd.png)
_Database modelleri arasındaki ilişkiler ve foreign key bağlantıları_

### Portfolio Modelleri (portfolio/models.py)

```python
from django.db import models
from django.contrib.auth.models import User
from django.core.validators import MinValueValidator
from decimal import Decimal

class Cryptocurrency(models.Model):
    """
    Kripto para bilgilerini tutan model.
    CoinGecko veya CoinMarketCap API'den çekilecek.
    """
    symbol = models.CharField(max_length=10, unique=True)  # BTC, ETH, vb.
    name = models.CharField(max_length=100)  # Bitcoin, Ethereum
    coingecko_id = models.CharField(max_length=50, unique=True)
    logo_url = models.URLField(blank=True, null=True)
    
    # Fiyat bilgileri (cache için)
    current_price = models.DecimalField(
        max_digits=20, 
        decimal_places=8, 
        default=0
    )
    price_change_24h = models.DecimalField(
        max_digits=10, 
        decimal_places=2, 
        default=0
    )
    market_cap = models.BigIntegerField(default=0)
    last_updated = models.DateTimeField(auto_now=True)
    
    class Meta:
        ordering = ['-market_cap']
        verbose_name_plural = 'Cryptocurrencies'
    
    def __str__(self):
        return f"{self.symbol} - {self.name}"


class Portfolio(models.Model):
    """
    Kullanıcının portfolio'sunu temsil eder.
    Her kullanıcı birden fazla portfolio oluşturabilir.
    """
    user = models.ForeignKey(User, on_delete=models.CASCADE, related_name='portfolios')
    name = models.CharField(max_length=100)
    description = models.TextField(blank=True)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    is_active = models.BooleanField(default=True)
    
    class Meta:
        ordering = ['-created_at']
        unique_together = ['user', 'name']
    
    def __str__(self):
        return f"{self.user.username} - {self.name}"
    
    @property
    def total_value(self):
        """Portfolio'nun toplam güncel değerini hesaplar"""
        total = Decimal('0')
        for holding in self.holdings.all():
            total += holding.current_value
        return total
    
    @property
    def total_profit_loss(self):
        """Toplam kar/zarar hesapla"""
        total_current = self.total_value
        total_cost = sum(h.total_cost for h in self.holdings.all())
        return total_current - total_cost
    
    @property
    def profit_loss_percentage(self):
        """Kar/zarar yüzdesi"""
        total_cost = sum(h.total_cost for h in self.holdings.all())
        if total_cost == 0:
            return 0
        return ((self.total_value - total_cost) / total_cost) * 100


class Holding(models.Model):
    """
    Portfolio içindeki bir kripto para pozisyonunu temsil eder.
    """
    portfolio = models.ForeignKey(
        Portfolio, 
        on_delete=models.CASCADE, 
        related_name='holdings'
    )
    cryptocurrency = models.ForeignKey(
        Cryptocurrency, 
        on_delete=models.CASCADE
    )
    quantity = models.DecimalField(
        max_digits=20, 
        decimal_places=8,
        validators=[MinValueValidator(Decimal('0'))]
    )
    average_buy_price = models.DecimalField(
        max_digits=20, 
        decimal_places=8,
        validators=[MinValueValidator(Decimal('0'))]
    )
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    
    class Meta:
        unique_together = ['portfolio', 'cryptocurrency']
        ordering = ['-created_at']
    
    def __str__(self):
        return f"{self.portfolio.name} - {self.cryptocurrency.symbol}"
    
    @property
    def total_cost(self):
        """Toplam maliyet hesapla"""
        return self.quantity * self.average_buy_price
    
    @property
    def current_value(self):
        """Güncel değer hesapla"""
        return self.quantity * self.cryptocurrency.current_price
    
    @property
    def profit_loss(self):
        """Kar/zarar hesapla"""
        return self.current_value - self.total_cost
    
    @property
    def profit_loss_percentage(self):
        """Kar/zarar yüzdesi"""
        if self.total_cost == 0:
            return 0
        return ((self.current_value - self.total_cost) / self.total_cost) * 100


class Transaction(models.Model):
    """
    Alım/satım işlemlerini kaydeder.
    """
    TRANSACTION_TYPES = (
        ('BUY', 'Alım'),
        ('SELL', 'Satım'),
    )
    
    portfolio = models.ForeignKey(
        Portfolio, 
        on_delete=models.CASCADE, 
        related_name='transactions'
    )
    cryptocurrency = models.ForeignKey(Cryptocurrency, on_delete=models.CASCADE)
    transaction_type = models.CharField(max_length=4, choices=TRANSACTION_TYPES)
    quantity = models.DecimalField(
        max_digits=20, 
        decimal_places=8,
        validators=[MinValueValidator(Decimal('0'))]
    )
    price_per_unit = models.DecimalField(
        max_digits=20, 
        decimal_places=8,
        validators=[MinValueValidator(Decimal('0'))]
    )
    fee = models.DecimalField(
        max_digits=20, 
        decimal_places=8, 
        default=0
    )
    notes = models.TextField(blank=True)
    transaction_date = models.DateTimeField()
    created_at = models.DateTimeField(auto_now_add=True)
    
    class Meta:
        ordering = ['-transaction_date']
    
    def __str__(self):
        return f"{self.transaction_type} - {self.cryptocurrency.symbol}"
    
    @property
    def total_amount(self):
        """İşlem tutarı + komisyon"""
        return (self.quantity * self.price_per_unit) + self.fee
    
    def save(self, *args, **kwargs):
        """
        Transaction kaydedildiğinde holding'i güncelle.
        """
        super().save(*args, **kwargs)
        self.update_holding()
    
    def update_holding(self):
        """
        Bu transaction'a göre holding'i güncelle.
        Alım ise holding'e ekle, satım ise çıkar.
        """
        holding, created = Holding.objects.get_or_create(
            portfolio=self.portfolio,
            cryptocurrency=self.cryptocurrency,
            defaults={'quantity': 0, 'average_buy_price': 0}
        )
        
        if self.transaction_type == 'BUY':
            # Ortalama alış fiyatını güncelle
            total_cost = holding.total_cost + (self.quantity * self.price_per_unit)
            total_quantity = holding.quantity + self.quantity
            holding.average_buy_price = total_cost / total_quantity if total_quantity > 0 else 0
            holding.quantity = total_quantity
        
        elif self.transaction_type == 'SELL':
            # Satış işlemi - miktar azalt
            holding.quantity -= self.quantity
            if holding.quantity < 0:
                holding.quantity = 0
        
        holding.save()


class PriceAlert(models.Model):
    """
    Kullanıcıların fiyat alarmları için model.
    """
    ALERT_TYPES = (
        ('ABOVE', 'Üzerine Çıkınca'),
        ('BELOW', 'Altına Düşünce'),
    )
    
    user = models.ForeignKey(User, on_delete=models.CASCADE, related_name='price_alerts')
    cryptocurrency = models.ForeignKey(Cryptocurrency, on_delete=models.CASCADE)
    alert_type = models.CharField(max_length=5, choices=ALERT_TYPES)
    target_price = models.DecimalField(
        max_digits=20, 
        decimal_places=8,
        validators=[MinValueValidator(Decimal('0'))]
    )
    is_active = models.BooleanField(default=True)
    triggered = models.BooleanField(default=False)
    created_at = models.DateTimeField(auto_now_add=True)
    triggered_at = models.DateTimeField(blank=True, null=True)
    
    class Meta:
        ordering = ['-created_at']
    
    def __str__(self):
        return f"{self.cryptocurrency.symbol} - {self.alert_type} - ${self.target_price}"
```

### Migrations Oluşturma ve Uygulama

```bash
# Migration dosyalarını oluştur
python manage.py makemigrations

# Veritabanına uygula
python manage.py migrate

# Superuser oluştur (admin panel için)
python manage.py createsuperuser
```

## REST API Geliştirme

Django REST Framework kullanarak güçlü bir API oluşturalım:

![Django REST API Architecture](/assets/img/posts/django-rest-api-architecture.png)
_Django REST Framework ile API geliştirme mimarisi_

### Serializers (portfolio/serializers.py)

```python
from rest_framework import serializers
from django.contrib.auth.models import User
from .models import (
    Cryptocurrency, Portfolio, Holding, 
    Transaction, PriceAlert
)

class CryptocurrencySerializer(serializers.ModelSerializer):
    """Cryptocurrency model için serializer"""
    
    price_change_percentage = serializers.SerializerMethodField()
    
    class Meta:
        model = Cryptocurrency
        fields = [
            'id', 'symbol', 'name', 'coingecko_id', 
            'logo_url', 'current_price', 'price_change_24h',
            'price_change_percentage', 'market_cap', 'last_updated'
        ]
        read_only_fields = ['last_updated']
    
    def get_price_change_percentage(self, obj):
        """24 saatlik değişim yüzdesi"""
        if obj.current_price > 0:
            return float(obj.price_change_24h)
        return 0


class HoldingSerializer(serializers.ModelSerializer):
    """Holding model için serializer"""
    
    cryptocurrency = CryptocurrencySerializer(read_only=True)
    cryptocurrency_id = serializers.PrimaryKeyRelatedField(
        queryset=Cryptocurrency.objects.all(),
        source='cryptocurrency',
        write_only=True
    )
    total_cost = serializers.DecimalField(
        max_digits=20, 
        decimal_places=2, 
        read_only=True
    )
    current_value = serializers.DecimalField(
        max_digits=20, 
        decimal_places=2, 
        read_only=True
    )
    profit_loss = serializers.DecimalField(
        max_digits=20, 
        decimal_places=2, 
        read_only=True
    )
    profit_loss_percentage = serializers.DecimalField(
        max_digits=10, 
        decimal_places=2, 
        read_only=True
    )
    
    class Meta:
        model = Holding
        fields = [
            'id', 'portfolio', 'cryptocurrency', 'cryptocurrency_id',
            'quantity', 'average_buy_price', 'total_cost',
            'current_value', 'profit_loss', 'profit_loss_percentage',
            'created_at', 'updated_at'
        ]
        read_only_fields = ['created_at', 'updated_at']


class PortfolioSerializer(serializers.ModelSerializer):
    """Portfolio model için serializer"""
    
    holdings = HoldingSerializer(many=True, read_only=True)
    total_value = serializers.DecimalField(
        max_digits=20, 
        decimal_places=2, 
        read_only=True
    )
    total_profit_loss = serializers.DecimalField(
        max_digits=20, 
        decimal_places=2, 
        read_only=True
    )
    profit_loss_percentage = serializers.DecimalField(
        max_digits=10, 
        decimal_places=2, 
        read_only=True
    )
    holdings_count = serializers.SerializerMethodField()
    
    class Meta:
        model = Portfolio
        fields = [
            'id', 'user', 'name', 'description',
            'total_value', 'total_profit_loss', 'profit_loss_percentage',
            'holdings', 'holdings_count', 'is_active',
            'created_at', 'updated_at'
        ]
        read_only_fields = ['user', 'created_at', 'updated_at']
    
    def get_holdings_count(self, obj):
        """Portfolio içindeki holding sayısı"""
        return obj.holdings.count()
    
    def create(self, validated_data):
        """Portfolio oluştururken user'ı otomatik ata"""
        validated_data['user'] = self.context['request'].user
        return super().create(validated_data)


class TransactionSerializer(serializers.ModelSerializer):
    """Transaction model için serializer"""
    
    cryptocurrency = CryptocurrencySerializer(read_only=True)
    cryptocurrency_id = serializers.PrimaryKeyRelatedField(
        queryset=Cryptocurrency.objects.all(),
        source='cryptocurrency',
        write_only=True
    )
    total_amount = serializers.DecimalField(
        max_digits=20, 
        decimal_places=2, 
        read_only=True
    )
    
    class Meta:
        model = Transaction
        fields = [
            'id', 'portfolio', 'cryptocurrency', 'cryptocurrency_id',
            'transaction_type', 'quantity', 'price_per_unit',
            'fee', 'total_amount', 'notes', 'transaction_date',
            'created_at'
        ]
        read_only_fields = ['created_at']
    
    def validate(self, data):
        """
        Satış işleminde yeterli miktarın olup olmadığını kontrol et.
        """
        if data.get('transaction_type') == 'SELL':
            portfolio = data.get('portfolio')
            crypto = data.get('cryptocurrency')
            quantity = data.get('quantity')
            
            try:
                holding = Holding.objects.get(
                    portfolio=portfolio,
                    cryptocurrency=crypto
                )
                if holding.quantity < quantity:
                    raise serializers.ValidationError(
                        f"Yetersiz bakiye. Mevcut miktar: {holding.quantity}"
                    )
            except Holding.DoesNotExist:
                raise serializers.ValidationError(
                    "Bu kripto para için holding bulunamadı."
                )
        
        return data


class PriceAlertSerializer(serializers.ModelSerializer):
    """PriceAlert model için serializer"""
    
    cryptocurrency = CryptocurrencySerializer(read_only=True)
    cryptocurrency_id = serializers.PrimaryKeyRelatedField(
        queryset=Cryptocurrency.objects.all(),
        source='cryptocurrency',
        write_only=True
    )
    
    class Meta:
        model = PriceAlert
        fields = [
            'id', 'user', 'cryptocurrency', 'cryptocurrency_id',
            'alert_type', 'target_price', 'is_active',
            'triggered', 'created_at', 'triggered_at'
        ]
        read_only_fields = ['user', 'triggered', 'created_at', 'triggered_at']
```

### Views (portfolio/views.py)

```python
from rest_framework import viewsets, status, filters
from rest_framework.decorators import action
from rest_framework.response import Response
from rest_framework.permissions import IsAuthenticated
from django.db.models import Q, Sum, F
from django_filters.rest_framework import DjangoFilterBackend

from .models import (
    Cryptocurrency, Portfolio, Holding, 
    Transaction, PriceAlert
)
from .serializers import (
    CryptocurrencySerializer, PortfolioSerializer,
    HoldingSerializer, TransactionSerializer,
    PriceAlertSerializer
)

class CryptocurrencyViewSet(viewsets.ReadOnlyModelViewSet):
    """
    Cryptocurrency verileri için ViewSet.
    Sadece okuma izni var (liste ve detay).
    """
    queryset = Cryptocurrency.objects.all()
    serializer_class = CryptocurrencySerializer
    filter_backends = [filters.SearchFilter, filters.OrderingFilter]
    search_fields = ['symbol', 'name']
    ordering_fields = ['market_cap', 'current_price', 'price_change_24h']
    ordering = ['-market_cap']
    
    @action(detail=False, methods=['get'])
    def top_gainers(self, request):
        """24 saatte en çok yükselen coinler"""
        cryptos = self.queryset.order_by('-price_change_24h')[:10]
        serializer = self.get_serializer(cryptos, many=True)
        return Response(serializer.data)
    
    @action(detail=False, methods=['get'])
    def top_losers(self, request):
        """24 saatte en çok düşen coinler"""
        cryptos = self.queryset.order_by('price_change_24h')[:10]
        serializer = self.get_serializer(cryptos, many=True)
        return Response(serializer.data)


class PortfolioViewSet(viewsets.ModelViewSet):
    """
    Portfolio CRUD işlemleri için ViewSet.
    Kullanıcı sadece kendi portfolio'larını görebilir.
    """
    serializer_class = PortfolioSerializer
    permission_classes = [IsAuthenticated]
    filter_backends = [DjangoFilterBackend, filters.OrderingFilter]
    filterset_fields = ['is_active']
    ordering_fields = ['created_at', 'name']
    ordering = ['-created_at']
    
    def get_queryset(self):
        """Kullanıcının kendi portfolio'ları"""
        return Portfolio.objects.filter(
            user=self.request.user
        ).prefetch_related('holdings__cryptocurrency')
    
    @action(detail=True, methods=['get'])
    def summary(self, request, pk=None):
        """
        Portfolio özet bilgileri.
        Toplam değer, kar/zarar, coin dağılımı vb.
        """
        portfolio = self.get_object()
        
        # Coin bazında dağılım
        holdings_distribution = []
        total_value = portfolio.total_value
        
        for holding in portfolio.holdings.all():
            percentage = (holding.current_value / total_value * 100) if total_value > 0 else 0
            holdings_distribution.append({
                'symbol': holding.cryptocurrency.symbol,
                'name': holding.cryptocurrency.name,
                'quantity': float(holding.quantity),
                'current_value': float(holding.current_value),
                'percentage': float(percentage),
                'profit_loss': float(holding.profit_loss),
                'profit_loss_percentage': float(holding.profit_loss_percentage)
            })
        
        # Son işlemler
        recent_transactions = Transaction.objects.filter(
            portfolio=portfolio
        ).order_by('-transaction_date')[:5]
        
        summary_data = {
            'portfolio': PortfolioSerializer(portfolio).data,
            'distribution': holdings_distribution,
            'recent_transactions': TransactionSerializer(
                recent_transactions, 
                many=True
            ).data,
            'total_holdings': portfolio.holdings.count(),
            'total_transactions': portfolio.transactions.count()
        }
        
        return Response(summary_data)
    
    @action(detail=True, methods=['get'])
    def performance(self, request, pk=None):
        """
        Portfolio performans metrikleri.
        """
        portfolio = self.get_object()
        
        # En karlı ve zararlı pozisyonlar
        best_performer = portfolio.holdings.order_by('-profit_loss_percentage').first()
        worst_performer = portfolio.holdings.order_by('profit_loss_percentage').first()
        
        performance_data = {
            'total_value': float(portfolio.total_value),
            'total_profit_loss': float(portfolio.total_profit_loss),
            'profit_loss_percentage': float(portfolio.profit_loss_percentage),
            'best_performer': HoldingSerializer(best_performer).data if best_performer else None,
            'worst_performer': HoldingSerializer(worst_performer).data if worst_performer else None,
        }
        
        return Response(performance_data)


class HoldingViewSet(viewsets.ModelViewSet):
    """
    Holding CRUD işlemleri için ViewSet.
    """
    serializer_class = HoldingSerializer
    permission_classes = [IsAuthenticated]
    
    def get_queryset(self):
        """Kullanıcının portfolio'larındaki holding'ler"""
        return Holding.objects.filter(
            portfolio__user=self.request.user
        ).select_related('cryptocurrency', 'portfolio')


class TransactionViewSet(viewsets.ModelViewSet):
    """
    Transaction CRUD işlemleri için ViewSet.
    """
    serializer_class = TransactionSerializer
    permission_classes = [IsAuthenticated]
    filter_backends = [DjangoFilterBackend, filters.OrderingFilter]
    filterset_fields = ['transaction_type', 'cryptocurrency']
    ordering_fields = ['transaction_date', 'created_at']
    ordering = ['-transaction_date']
    
    def get_queryset(self):
        """Kullanıcının portfolio'larındaki işlemler"""
        return Transaction.objects.filter(
            portfolio__user=self.request.user
        ).select_related('cryptocurrency', 'portfolio')


class PriceAlertViewSet(viewsets.ModelViewSet):
    """
    PriceAlert CRUD işlemleri için ViewSet.
    """
    serializer_class = PriceAlertSerializer
    permission_classes = [IsAuthenticated]
    filter_backends = [DjangoFilterBackend]
    filterset_fields = ['is_active', 'triggered', 'alert_type']
    
    def get_queryset(self):
        """Kullanıcının alarm'ları"""
        return PriceAlert.objects.filter(
            user=self.request.user
        ).select_related('cryptocurrency')
    
    def perform_create(self, serializer):
        """Alarm oluştururken user'ı otomatik ata"""
        serializer.save(user=self.request.user)
```

### URL Yapılandırması (portfolio/urls.py)

```python
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from .views import (
    CryptocurrencyViewSet, PortfolioViewSet,
    HoldingViewSet, TransactionViewSet,
    PriceAlertViewSet
)

router = DefaultRouter()
router.register(r'cryptocurrencies', CryptocurrencyViewSet, basename='cryptocurrency')
router.register(r'portfolios', PortfolioViewSet, basename='portfolio')
router.register(r'holdings', HoldingViewSet, basename='holding')
router.register(r'transactions', TransactionViewSet, basename='transaction')
router.register(r'price-alerts', PriceAlertViewSet, basename='price-alert')

urlpatterns = [
    path('', include(router.urls)),
]
```

### Ana URL Yapılandırması (crypto_portfolio/urls.py)

```python
from django.contrib import admin
from django.urls import path, include
from rest_framework.authtoken import views as auth_views

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/', include('portfolio.urls')),
    path('api/auth/', include('rest_framework.urls')),
    path('api/token-auth/', auth_views.obtain_auth_token),
]
```

## Gerçek Zamanlı Fiyat Güncellemeleri

CoinGecko API kullanarak kripto fiyatlarını güncelleyen servis:

### Price Update Service (portfolio/services.py)

```python
import requests
from decimal import Decimal
from django.utils import timezone
from .models import Cryptocurrency, PriceAlert
import logging

logger = logging.getLogger(__name__)

class CryptoPriceService:
    """
    CoinGecko API'den kripto fiyatlarını çeken servis.
    """
    BASE_URL = "https://api.coingecko.com/api/v3"
    
    @classmethod
    def update_all_prices(cls):
        """
        Tüm kripto paraların fiyatlarını güncelle.
        """
        cryptocurrencies = Cryptocurrency.objects.all()
        coin_ids = ','.join([c.coingecko_id for c in cryptocurrencies])
        
        try:
            url = f"{cls.BASE_URL}/simple/price"
            params = {
                'ids': coin_ids,
                'vs_currencies': 'usd',
                'include_24hr_change': 'true',
                'include_market_cap': 'true'
            }
            
            response = requests.get(url, params=params, timeout=10)
            response.raise_for_status()
            data = response.json()
            
            updated_count = 0
            for crypto in cryptocurrencies:
                if crypto.coingecko_id in data:
                    coin_data = data[crypto.coingecko_id]
                    
                    old_price = crypto.current_price
                    crypto.current_price = Decimal(str(coin_data.get('usd', 0)))
                    crypto.price_change_24h = Decimal(
                        str(coin_data.get('usd_24h_change', 0))
                    )
                    crypto.market_cap = coin_data.get('usd_market_cap', 0)
                    crypto.last_updated = timezone.now()
                    crypto.save()
                    
                    # Fiyat alarmlarını kontrol et
                    cls.check_price_alerts(crypto, old_price)
                    
                    updated_count += 1
            
            logger.info(f"{updated_count} cryptocurrency updated successfully")
            return updated_count
            
        except requests.RequestException as e:
            logger.error(f"Error updating prices: {str(e)}")
            return 0
    
    @classmethod
    def update_single_price(cls, cryptocurrency):
        """
        Tek bir kripto paranın fiyatını güncelle.
        """
        try:
            url = f"{cls.BASE_URL}/simple/price"
            params = {
                'ids': cryptocurrency.coingecko_id,
                'vs_currencies': 'usd',
                'include_24hr_change': 'true',
                'include_market_cap': 'true'
            }
            
            response = requests.get(url, params=params, timeout=10)
            response.raise_for_status()
            data = response.json()
            
            if cryptocurrency.coingecko_id in data:
                coin_data = data[cryptocurrency.coingecko_id]
                
                old_price = cryptocurrency.current_price
                cryptocurrency.current_price = Decimal(str(coin_data.get('usd', 0)))
                cryptocurrency.price_change_24h = Decimal(
                    str(coin_data.get('usd_24h_change', 0))
                )
                cryptocurrency.market_cap = coin_data.get('usd_market_cap', 0)
                cryptocurrency.last_updated = timezone.now()
                cryptocurrency.save()
                
                cls.check_price_alerts(cryptocurrency, old_price)
                
                logger.info(f"Updated {cryptocurrency.symbol}: ${cryptocurrency.current_price}")
                return True
            
        except requests.RequestException as e:
            logger.error(f"Error updating {cryptocurrency.symbol}: {str(e)}")
            return False
    
    @classmethod
    def check_price_alerts(cls, cryptocurrency, old_price):
        """
        Fiyat alarmlarını kontrol et ve tetiklenen alarmları işaretle.
        """
        current_price = cryptocurrency.current_price
        
        # Aktif alarmları getir
        alerts = PriceAlert.objects.filter(
            cryptocurrency=cryptocurrency,
            is_active=True,
            triggered=False
        )
        
        for alert in alerts:
            triggered = False
            
            if alert.alert_type == 'ABOVE' and current_price >= alert.target_price:
                triggered = True
            elif alert.alert_type == 'BELOW' and current_price <= alert.target_price:
                triggered = True
            
            if triggered:
                alert.triggered = True
                alert.triggered_at = timezone.now()
                alert.save()
                
                # Burada email veya push notification gönderilebilir
                logger.info(
                    f"Price alert triggered for {alert.user.username}: "
                    f"{cryptocurrency.symbol} {alert.alert_type} ${alert.target_price}"
                )
    
    @classmethod
    def get_coin_list(cls):
        """
        CoinGecko'dan coin listesini çek ve veritabanına kaydet.
        """
        try:
            url = f"{cls.BASE_URL}/coins/list"
            response = requests.get(url, timeout=10)
            response.raise_for_status()
            coins = response.json()
            
            # İlk 100 coini ekle (örnek için)
            for coin_data in coins[:100]:
                Cryptocurrency.objects.get_or_create(
                    coingecko_id=coin_data['id'],
                    defaults={
                        'symbol': coin_data['symbol'].upper(),
                        'name': coin_data['name']
                    }
                )
            
            logger.info(f"Added/updated {len(coins[:100])} cryptocurrencies")
            return True
            
        except requests.RequestException as e:
            logger.error(f"Error fetching coin list: {str(e)}")
            return False
```

### Celery Tasks (portfolio/tasks.py)

```python
from celery import shared_task
from .services import CryptoPriceService
import logging

logger = logging.getLogger(__name__)

@shared_task
def update_crypto_prices():
    """
    Tüm kripto fiyatlarını güncelle.
    Periyodik olarak çalışacak (örn: her 5 dakikada).
    """
    logger.info("Starting crypto price update task")
    updated_count = CryptoPriceService.update_all_prices()
    logger.info(f"Crypto price update completed: {updated_count} updated")
    return updated_count

@shared_task
def sync_coingecko_list():
    """
    CoinGecko'dan coin listesini senkronize et.
    Günde bir kez çalışmalı.
    """
    logger.info("Starting CoinGecko list sync")
    result = CryptoPriceService.get_coin_list()
    logger.info(f"CoinGecko list sync completed: {result}")
    return result
```

### Celery Konfigürasyonu (crypto_portfolio/celery.py)

```python
import os
from celery import Celery
from celery.schedules import crontab

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'crypto_portfolio.settings')

app = Celery('crypto_portfolio')
app.config_from_object('django.conf:settings', namespace='CELERY')
app.autodiscover_tasks()

# Periyodik görevler
app.conf.beat_schedule = {
    'update-crypto-prices-every-5-minutes': {
        'task': 'portfolio.tasks.update_crypto_prices',
        'schedule': 300.0,  # 5 dakika
    },
    'sync-coingecko-list-daily': {
        'task': 'portfolio.tasks.sync_coingecko_list',
        'schedule': crontab(hour=0, minute=0),  # Her gece saat 00:00
    },
}
```

## Admin Panel Özelleştirme

Django Admin'i portfolio yönetimi için özelleştirelim:

```python
# portfolio/admin.py
from django.contrib import admin
from django.utils.html import format_html
from .models import (
    Cryptocurrency, Portfolio, Holding,
    Transaction, PriceAlert
)

@admin.register(Cryptocurrency)
class CryptocurrencyAdmin(admin.ModelAdmin):
    list_display = [
        'symbol', 'name', 'current_price_display',
        'price_change_display', 'market_cap_display',
        'last_updated'
    ]
    list_filter = ['last_updated']
    search_fields = ['symbol', 'name', 'coingecko_id']
    readonly_fields = ['last_updated']
    
    def current_price_display(self, obj):
        return f"${obj.current_price:,.2f}"
    current_price_display.short_description = 'Current Price'
    
    def price_change_display(self, obj):
        color = 'green' if obj.price_change_24h >= 0 else 'red'
        return format_html(
            '<span style="color: {};">{:.2f}%</span>',
            color, obj.price_change_24h
        )
    price_change_display.short_description = '24h Change'
    
    def market_cap_display(self, obj):
        return f"${obj.market_cap:,}"
    market_cap_display.short_description = 'Market Cap'


class HoldingInline(admin.TabularInline):
    model = Holding
    extra = 0
    readonly_fields = ['total_cost', 'current_value', 'profit_loss']
    
    def total_cost(self, obj):
        return f"${obj.total_cost:,.2f}"
    
    def current_value(self, obj):
        return f"${obj.current_value:,.2f}"
    
    def profit_loss(self, obj):
        color = 'green' if obj.profit_loss >= 0 else 'red'
        return format_html(
            '<span style="color: {};">${:,.2f}</span>',
            color, obj.profit_loss
        )


@admin.register(Portfolio)
class PortfolioAdmin(admin.ModelAdmin):
    list_display = [
        'name', 'user', 'total_value_display',
        'profit_loss_display', 'holdings_count',
        'is_active', 'created_at'
    ]
    list_filter = ['is_active', 'created_at', 'user']
    search_fields = ['name', 'user__username']
    readonly_fields = ['created_at', 'updated_at']
    inlines = [HoldingInline]
    
    def total_value_display(self, obj):
        return f"${obj.total_value:,.2f}"
    total_value_display.short_description = 'Total Value'
    
    def profit_loss_display(self, obj):
        pl = obj.total_profit_loss
        color = 'green' if pl >= 0 else 'red'
        return format_html(
            '<span style="color: {};">${:,.2f} ({:.2f}%)</span>',
            color, pl, obj.profit_loss_percentage
        )
    profit_loss_display.short_description = 'Profit/Loss'
    
    def holdings_count(self, obj):
        return obj.holdings.count()
    holdings_count.short_description = 'Holdings'


@admin.register(Transaction)
class TransactionAdmin(admin.ModelAdmin):
    list_display = [
        'transaction_date', 'portfolio', 'transaction_type',
        'cryptocurrency', 'quantity', 'price_per_unit',
        'total_amount_display'
    ]
    list_filter = ['transaction_type', 'transaction_date', 'cryptocurrency']
    search_fields = ['portfolio__name', 'cryptocurrency__symbol']
    date_hierarchy = 'transaction_date'
    
    def total_amount_display(self, obj):
        return f"${obj.total_amount:,.2f}"
    total_amount_display.short_description = 'Total Amount'


@admin.register(PriceAlert)
class PriceAlertAdmin(admin.ModelAdmin):
    list_display = [
        'user', 'cryptocurrency', 'alert_type',
        'target_price', 'is_active', 'triggered',
        'created_at'
    ]
    list_filter = ['alert_type', 'is_active', 'triggered', 'created_at']
    search_fields = ['user__username', 'cryptocurrency__symbol']
    readonly_fields = ['triggered_at']
```

## Uygulamayı Çalıştırma

### Development Sunucusu

```bash
# Django sunucusunu başlat
python manage.py runserver

# Celery worker başlat (ayrı terminal)
celery -A crypto_portfolio worker --loglevel=info

# Celery beat başlat (periyodik görevler için, ayrı terminal)
celery -A crypto_portfolio beat --loglevel=info
```

### API Kullanım Örnekleri

```bash
# Token alma
curl -X POST http://localhost:8000/api/token-auth/ \
  -H "Content-Type: application/json" \
  -d '{"username": "user", "password": "pass"}'

# Portfolio oluşturma
curl -X POST http://localhost:8000/api/portfolios/ \
  -H "Authorization: Token YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "Ana Portfolio", "description": "İlk portfoliom"}'

# Transaction ekleme (alım)
curl -X POST http://localhost:8000/api/transactions/ \
  -H "Authorization: Token YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "portfolio": 1,
    "cryptocurrency_id": 1,
    "transaction_type": "BUY",
    "quantity": "0.5",
    "price_per_unit": "50000",
    "transaction_date": "2024-11-15T10:00:00Z"
  }'

# Portfolio özeti görüntüleme
curl http://localhost:8000/api/portfolios/1/summary/ \
  -H "Authorization: Token YOUR_TOKEN"
```

## Production İçin Best Practices

### Güvenlik Ayarları

```python
# production settings
DEBUG = False
ALLOWED_HOSTS = ['yourdomain.com']

# HTTPS zorunlu
SECURE_SSL_REDIRECT = True
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True

# CORS ayarları
CORS_ALLOWED_ORIGINS = [
    "https://yourdomain.com",
]

# Rate limiting
REST_FRAMEWORK['DEFAULT_THROTTLE_CLASSES'] = [
    'rest_framework.throttling.AnonRateThrottle',
    'rest_framework.throttling.UserRateThrottle'
]
REST_FRAMEWORK['DEFAULT_THROTTLE_RATES'] = {
    'anon': '100/day',
    'user': '1000/day'
}
```

### Database Optimization

```python
# portfolio/views.py içinde
class PortfolioViewSet(viewsets.ModelViewSet):
    def get_queryset(self):
        return Portfolio.objects.filter(
            user=self.request.user
        ).select_related(
            'user'
        ).prefetch_related(
            'holdings__cryptocurrency',
            'transactions__cryptocurrency'
        )
```

### Caching Stratejisi

```python
from django.core.cache import cache
from django.views.decorators.cache import cache_page

class CryptocurrencyViewSet(viewsets.ReadOnlyModelViewSet):
    @cache_page(60 * 5)  # 5 dakika cache
    def list(self, request):
        cache_key = 'crypto_list'
        data = cache.get(cache_key)
        
        if not data:
            queryset = self.get_queryset()
            serializer = self.get_serializer(queryset, many=True)
            data = serializer.data
            cache.set(cache_key, data, 300)  # 5 dakika
        
        return Response(data)
```

## Sonuç

Bu yazıda Django kullanarak profesyonel bir kripto portfolio takip uygulaması geliştirdik. Uygulamanın temel özellikleri:

- **User Authentication**: Güvenli kullanıcı yönetimi
- **Portfolio Management**: Çoklu portfolio desteği
- **Real-time Prices**: CoinGecko API entegrasyonu
- **Transaction Tracking**: Alım/satım işlemlerini kaydetme
- **Profit/Loss Calculation**: Otomatik kar/zarar hesaplama
- **Price Alerts**: Kullanıcı tanımlı fiyat alarmları
- **REST API**: Mobil ve web frontend için API
- **Admin Panel**: Kolay yönetim arayüzü
- **Background Tasks**: Celery ile arka plan görevleri

### Geliştirme Önerileri

Uygulamayı daha da geliştirebilirsiniz:

1. **Frontend Ekleyin**: React, Vue.js veya Angular ile modern UI
2. **Grafik ve Analitik**: Chart.js ile portfolio performans grafikleri
3. **Çoklu Para Birimi**: USD, EUR, TRY desteği
4. **Exchange API Entegrasyonu**: Binance, Coinbase gibi borsalardan otomatik import
5. **Tax Reporting**: Vergi raporlama özellikleri
6. **Mobile App**: React Native veya Flutter ile mobil uygulama
7. **WebSocket**: Gerçek zamanlı fiyat güncellemeleri
8. **Two-Factor Authentication**: Ekstra güvenlik katmanı

Django'nun güçlü yapısı sayesinde tüm bu özellikleri kolayca ekleyebilir ve ölçeklenebilir bir uygulama geliştirebilirsiniz.

## Kaynaklar

- [Django Documentation](https://docs.djangoproject.com/)
- [Django REST Framework](https://www.django-rest-framework.org/)
- [CoinGecko API](https://www.coingecko.com/en/api)
- [Celery Documentation](https://docs.celeryproject.org/)
- [PostgreSQL Documentation](https://www.postgresql.org/docs/)
