---
title: "pytest ile İleri Seviye Test Teknikleri"
date: "2025-10-10 09:00:00 +0300"
categories: [Python, Testing]
tags: [pytest, testing, tdd, fixtures, mocking, python, test-automation, ci-cd, coverage, unit-testing]
image:
  src: /assets/img/posts/pytest-unittest-comparison-flowchart.jpg
  alt: "pytest vs unittest Test Framework Karşılaştırması"
---

Python dünyasında test yazmak söz konusu olduğunda pytest, geliştiricilerin en çok tercih ettiği framework'lerden biridir. Basit sözdizimi, güçlü fixture sistemi ve kapsamlı plugin ekosistemi ile pytest, unit testlerden entegrasyon testlerine kadar geniş bir yelpazede kullanılabilir. Bu yazıda, pytest'in ileri seviye özelliklerini ve best practice'leri ele alacağız.

## pytest Neden Bu Kadar Popüler?

pytest'in popülaritesi birkaç önemli avantajdan kaynaklanıyor:

- **Basit Sözdizimi**: Standart `assert` ifadeleri kullanabilirsiniz, özel assertion metodlarına gerek yok
- **Fixture Sistemi**: Test setup/teardown işlemlerini modüler ve yeniden kullanılabilir şekilde organize eder
- **Parametrize**: Aynı testi farklı girdi değerleriyle çalıştırmayı kolaylaştırır
- **Plugin Ekosistemi**: 800'den fazla plugin ile her türlü test senaryosuna çözüm sunar
- **Detaylı Hata Mesajları**: Test başarısız olduğunda ne olduğunu tam olarak anlamanızı sağlar

![pytest Fixtures ve Test Organizasyonu](/assets/img/posts/pytest-fixtures-setup-pattern.png)

## pytest Kurulumu ve İlk Testler

pytest'i kurmak çok basit:

```bash
# pytest kurulumu
pip install pytest

# Coverage plugin ile birlikte kurulum
pip install pytest pytest-cov

# Async test desteği için
pip install pytest-asyncio

# Mock ve patching için
pip install pytest-mock
```

Basit bir test örneği:

```python
# test_calculator.py
def add(a, b):
    """İki sayıyı toplar"""
    return a + b

def test_add_positive_numbers():
    """Pozitif sayıların toplamını test eder"""
    result = add(2, 3)
    assert result == 5

def test_add_negative_numbers():
    """Negatif sayıların toplamını test eder"""
    result = add(-1, -1)
    assert result == -2

def test_add_mixed_numbers():
    """Pozitif ve negatif sayıların toplamını test eder"""
    result = add(5, -3)
    assert result == 2
```

Testleri çalıştırma:

```bash
# Tüm testleri çalıştır
pytest

# Verbose mod ile detaylı çıktı
pytest -v

# Belirli bir dosyayı test et
pytest test_calculator.py

# Belirli bir test fonksiyonunu çalıştır
pytest test_calculator.py::test_add_positive_numbers
```

## Fixtures: Test Setup ve Teardown

Fixtures, pytest'in en güçlü özelliklerinden biridir. Test setup/teardown işlemlerini temiz ve yeniden kullanılabilir bir şekilde organize ederler.

### Temel Fixture Kullanımı

```python
import pytest
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

# conftest.py - Tüm testler tarafından kullanılabilir
@pytest.fixture
def database_connection():
    """Veritabanı bağlantısı fixture'ı"""
    # Setup: Veritabanı bağlantısı oluştur
    engine = create_engine('sqlite:///:memory:')
    Session = sessionmaker(bind=engine)
    session = Session()
    
    # Fixture'ı test fonksiyonuna ver
    yield session
    
    # Teardown: Bağlantıyı kapat
    session.close()
    engine.dispose()

@pytest.fixture
def sample_user():
    """Test için örnek kullanıcı verisi"""
    return {
        'id': 1,
        'username': 'testuser',
        'email': 'test@example.com',
        'is_active': True
    }

# test_user.py
def test_user_creation(database_connection, sample_user):
    """Kullanıcı oluşturma testi"""
    # database_connection ve sample_user fixture'ları otomatik inject edilir
    from models import User
    
    user = User(**sample_user)
    database_connection.add(user)
    database_connection.commit()
    
    # Veritabanından kullanıcıyı oku
    saved_user = database_connection.query(User).filter_by(
        username=sample_user['username']
    ).first()
    
    assert saved_user is not None
    assert saved_user.username == sample_user['username']
    assert saved_user.email == sample_user['email']
```

### Fixture Scope'ları

Fixture'lar farklı scope'larda tanımlanabilir:

![pytest Fixture Scope Sıralaması](/assets/img/posts/pytest-fixtures-scope-order.svg)

```python
import pytest
import redis
from fastapi.testclient import TestClient

@pytest.fixture(scope="function")
def function_fixture():
    """Her test fonksiyonu için yeni instance"""
    print("Setup: function scope")
    yield "function_data"
    print("Teardown: function scope")

@pytest.fixture(scope="class")
def class_fixture():
    """Her test class'ı için bir kez"""
    print("Setup: class scope")
    yield "class_data"
    print("Teardown: class scope")

@pytest.fixture(scope="module")
def module_fixture():
    """Her test modülü için bir kez"""
    print("Setup: module scope")
    yield "module_data"
    print("Teardown: module scope")

@pytest.fixture(scope="session")
def redis_connection():
    """Tüm test session'ı boyunca bir kez"""
    print("Setup: Connecting to Redis...")
    client = redis.Redis(host='localhost', port=6379, db=0)
    
    # Redis'in çalıştığını kontrol et
    client.ping()
    
    yield client
    
    print("Teardown: Closing Redis connection...")
    client.flushdb()  # Test verilerini temizle
    client.close()

# Test kullanımı
def test_redis_set(redis_connection):
    """Redis SET komutu testi"""
    redis_connection.set('test_key', 'test_value')
    value = redis_connection.get('test_key')
    assert value == b'test_value'

def test_redis_get(redis_connection):
    """Redis GET komutu testi - aynı connection'ı kullanır"""
    # Önceki testin verisi temizlendi
    value = redis_connection.get('nonexistent_key')
    assert value is None
```

### Parametrize Edilmiş Fixtures

```python
import pytest

@pytest.fixture(params=['sqlite', 'postgresql', 'mysql'])
def database_engine(request):
    """Farklı veritabanı motorları için fixture"""
    db_type = request.param
    
    if db_type == 'sqlite':
        engine = create_engine('sqlite:///:memory:')
    elif db_type == 'postgresql':
        engine = create_engine('postgresql://user:pass@localhost/testdb')
    elif db_type == 'mysql':
        engine = create_engine('mysql://user:pass@localhost/testdb')
    
    yield engine
    engine.dispose()

def test_database_operations(database_engine):
    """Tüm veritabanı motorlarıyla test edilir"""
    # Bu test 3 kez çalıştırılır (sqlite, postgresql, mysql)
    connection = database_engine.connect()
    result = connection.execute("SELECT 1")
    assert result.fetchone()[0] == 1
```

## Parametrize: Çoklu Test Senaryoları

`@pytest.mark.parametrize` decorator'ı ile aynı testi farklı parametrelerle çalıştırabilirsiniz:

```python
import pytest

@pytest.mark.parametrize("input_value,expected", [
    (2, 4),      # 2^2 = 4
    (3, 9),      # 3^2 = 9
    (4, 16),     # 4^2 = 16
    (0, 0),      # 0^2 = 0
    (-2, 4),     # (-2)^2 = 4
])
def test_square(input_value, expected):
    """Kare alma fonksiyonu testi"""
    assert input_value ** 2 == expected

# Çoklu parametre kombinasyonları
@pytest.mark.parametrize("base", [2, 3, 5])
@pytest.mark.parametrize("exponent", [2, 3])
def test_power(base, exponent):
    """Üs alma testi - 6 kombinasyon (2x3)"""
    result = base ** exponent
    assert isinstance(result, int)
    assert result > 0

# İsimlendirilmiş parametreler ile daha okunabilir testler
@pytest.mark.parametrize("test_input,expected", [
    pytest.param({"email": "test@example.com"}, True, id="valid_email"),
    pytest.param({"email": "invalid"}, False, id="invalid_email"),
    pytest.param({"email": ""}, False, id="empty_email"),
    pytest.param({}, False, id="missing_email"),
])
def test_email_validation(test_input, expected):
    """Email validasyon testi"""
    from validators import validate_email
    result = validate_email(test_input.get('email', ''))
    assert result == expected
```

## Mocking ve Patching

Test ortamında dış bağımlılıkları simüle etmek için mocking kullanılır:

```python
import pytest
from unittest.mock import Mock, patch, MagicMock
import requests

# Test edilecek kod
def get_user_data(user_id):
    """API'den kullanıcı verisi çeker"""
    response = requests.get(f'https://api.example.com/users/{user_id}')
    response.raise_for_status()
    return response.json()

def send_notification(user_id, message):
    """Kullanıcıya bildirim gönderir"""
    user = get_user_data(user_id)
    # Email servisi çağrısı
    send_email(user['email'], message)
    return True

# Mock kullanımı
@patch('requests.get')
def test_get_user_data(mock_get):
    """API çağrısını mock'layarak test et"""
    # Mock response oluştur
    mock_response = Mock()
    mock_response.json.return_value = {
        'id': 1,
        'name': 'Test User',
        'email': 'test@example.com'
    }
    mock_response.status_code = 200
    mock_get.return_value = mock_response
    
    # Test et
    result = get_user_data(1)
    
    # Assertions
    assert result['name'] == 'Test User'
    assert result['email'] == 'test@example.com'
    mock_get.assert_called_once_with('https://api.example.com/users/1')

# Çoklu mock kullanımı
@patch('my_module.send_email')
@patch('my_module.get_user_data')
def test_send_notification(mock_get_user, mock_send_email):
    """İki farklı fonksiyonu mock'la"""
    # get_user_data mock'ı
    mock_get_user.return_value = {
        'id': 1,
        'email': 'test@example.com'
    }
    
    # send_email mock'ı
    mock_send_email.return_value = True
    
    # Test et
    result = send_notification(1, "Hello!")
    
    # Assertions
    assert result is True
    mock_get_user.assert_called_once_with(1)
    mock_send_email.assert_called_once_with('test@example.com', "Hello!")
```

### pytest-mock Plugin ile Daha Kolay Mocking

```python
import pytest

# pytest-mock plugin fixture'ını kullan
def test_with_mocker(mocker):
    """mocker fixture ile daha temiz syntax"""
    # Fonksiyonu patch'le
    mock_get = mocker.patch('requests.get')
    
    # Mock return value ayarla
    mock_get.return_value.json.return_value = {'status': 'success'}
    mock_get.return_value.status_code = 200
    
    # Test et
    result = get_user_data(1)
    assert result['status'] == 'success'

# Spy kullanımı - gerçek fonksiyonu çağırır ama takip eder
def test_spy_example(mocker):
    """Spy ile fonksiyon çağrılarını izle"""
    import my_module
    
    spy = mocker.spy(my_module, 'internal_function')
    
    # Gerçek fonksiyonu çağır
    my_module.public_function()
    
    # internal_function'ın çağrıldığını doğrula
    spy.assert_called_once()
    spy.assert_called_with(expected_arg='value')
```

## Async Test Yazma

Modern Python uygulamaları genellikle async/await kullanır. pytest-asyncio plugin ile async fonksiyonları test edebilirsiniz:

```python
import pytest
import asyncio
import aiohttp

# Test edilecek async kod
async def fetch_data(url):
    """Async HTTP request"""
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as response:
            return await response.json()

async def process_multiple_requests(urls):
    """Birden fazla async request paralel olarak"""
    tasks = [fetch_data(url) for url in urls]
    results = await asyncio.gather(*tasks)
    return results

# Async test
@pytest.mark.asyncio
async def test_fetch_data(mocker):
    """Async fonksiyonu test et"""
    # aiohttp.ClientSession'ı mock'la
    mock_response = mocker.AsyncMock()
    mock_response.json.return_value = {'data': 'test'}
    mock_response.__aenter__.return_value = mock_response
    
    mock_session = mocker.MagicMock()
    mock_session.get.return_value = mock_response
    mock_session.__aenter__.return_value = mock_session
    
    mocker.patch('aiohttp.ClientSession', return_value=mock_session)
    
    # Test et
    result = await fetch_data('https://api.example.com/data')
    assert result['data'] == 'test'

@pytest.mark.asyncio
async def test_parallel_requests(mocker):
    """Paralel async requestleri test et"""
    mock_response = mocker.AsyncMock()
    mock_response.json.return_value = {'status': 'ok'}
    mock_response.__aenter__.return_value = mock_response
    
    mock_session = mocker.MagicMock()
    mock_session.get.return_value = mock_response
    mock_session.__aenter__.return_value = mock_session
    
    mocker.patch('aiohttp.ClientSession', return_value=mock_session)
    
    urls = ['https://api1.com', 'https://api2.com', 'https://api3.com']
    results = await process_multiple_requests(urls)
    
    assert len(results) == 3
    assert all(r['status'] == 'ok' for r in results)

# Async fixture
@pytest.fixture
async def async_database():
    """Async veritabanı fixture'ı"""
    from motor.motor_asyncio import AsyncIOMotorClient
    
    # Setup
    client = AsyncIOMotorClient('mongodb://localhost:27017')
    db = client['test_database']
    
    yield db
    
    # Teardown
    await client.drop_database('test_database')
    client.close()

@pytest.mark.asyncio
async def test_async_database_operations(async_database):
    """Async veritabanı işlemlerini test et"""
    collection = async_database['users']
    
    # Insert
    result = await collection.insert_one({'name': 'Test User'})
    assert result.inserted_id is not None
    
    # Find
    user = await collection.find_one({'name': 'Test User'})
    assert user['name'] == 'Test User'
```

## Test Coverage Analizi

Test coverage, kodunuzun ne kadarının testlerle kapsandığını gösterir:

```bash
# Coverage raporu oluştur
pytest --cov=my_package tests/

# HTML rapor oluştur
pytest --cov=my_package --cov-report=html tests/

# Eksik satırları göster
pytest --cov=my_package --cov-report=term-missing tests/

# Belirli bir coverage oranı altında hata ver
pytest --cov=my_package --cov-fail-under=80 tests/
```

pytest.ini ile coverage ayarları:

```ini
[tool:pytest]
testpaths = tests
python_files = test_*.py
python_classes = Test*
python_functions = test_*

# Coverage ayarları
addopts = 
    --cov=my_package
    --cov-report=html
    --cov-report=term-missing
    --cov-fail-under=80
    -v

# Asyncio ayarları
asyncio_mode = auto

# Test timeout (saniye)
timeout = 300

# Parallel test execution
# -n auto: CPU sayısı kadar worker
markers =
    slow: marks tests as slow (deselect with '-m "not slow"')
    integration: marks tests as integration tests
    unit: marks tests as unit tests
```

## Custom Markers ve Test Organizasyonu

Testleri kategorize etmek için custom marker'lar kullanabilirsiniz:

```python
import pytest

# Marker tanımları
@pytest.mark.slow
def test_slow_operation():
    """Yavaş çalışan test"""
    import time
    time.sleep(2)
    assert True

@pytest.mark.integration
def test_database_integration():
    """Veritabanı entegrasyon testi"""
    # Gerçek veritabanı bağlantısı gerektirir
    pass

@pytest.mark.unit
def test_pure_function():
    """Hızlı unit test"""
    assert 1 + 1 == 2

# Özel marker ile parametre geçme
@pytest.mark.timeout(5)
def test_with_timeout():
    """5 saniye timeout'u olan test"""
    pass

# Marker kombinasyonları
@pytest.mark.slow
@pytest.mark.integration
def test_slow_integration():
    """Hem yavaş hem entegrasyon testi"""
    pass
```

Marker'ları kullanarak test seçimi:

```bash
# Sadece unit testleri çalıştır
pytest -m unit

# Slow testleri hariç tut
pytest -m "not slow"

# Integration testlerini çalıştır
pytest -m integration

# Unit ve integration, slow hariç
pytest -m "unit or integration and not slow"
```

## Pytest Plugins ve İleri Seviye Özellikler

### pytest-xdist: Parallel Test Execution

```bash
# Kurulum
pip install pytest-xdist

# Tüm CPU core'ları kullan
pytest -n auto

# Belirli sayıda worker
pytest -n 4

# Her test dosyasını farklı worker'da çalıştır
pytest --dist loadfile
```

### pytest-benchmark: Performance Testing

```python
import pytest

def fibonacci(n):
    """Fibonacci hesaplama"""
    if n < 2:
        return n
    return fibonacci(n-1) + fibonacci(n-2)

def test_fibonacci_performance(benchmark):
    """Fibonacci performans testi"""
    result = benchmark(fibonacci, 10)
    assert result == 55

# Benchmark karşılaştırma
def test_compare_implementations(benchmark):
    """Farklı implementasyonları karşılaştır"""
    def iterative_fib(n):
        a, b = 0, 1
        for _ in range(n):
            a, b = b, a + b
        return a
    
    result = benchmark(iterative_fib, 10)
    assert result == 55
```

### pytest-timeout: Test Timeout

```python
import pytest

@pytest.mark.timeout(5)
def test_with_timeout():
    """5 saniye içinde bitmeli"""
    import time
    time.sleep(2)  # OK
    assert True

@pytest.mark.timeout(1)
def test_timeout_failure():
    """Timeout aşımı - fail olacak"""
    import time
    time.sleep(2)  # FAIL - 1 saniye timeout
```

### pytest-django: Django Test Integration

```python
import pytest
from django.contrib.auth.models import User

@pytest.mark.django_db
def test_user_create():
    """Django model testi"""
    user = User.objects.create_user(
        username='testuser',
        email='test@example.com',
        password='testpass123'
    )
    assert user.username == 'testuser'
    assert user.email == 'test@example.com'

@pytest.mark.django_db
class TestUserModel:
    """Django model test class"""
    
    def test_user_str(self):
        user = User.objects.create_user(username='test')
        assert str(user) == 'test'
    
    def test_user_email_unique(self):
        User.objects.create_user(username='user1', email='test@example.com')
        
        with pytest.raises(Exception):
            User.objects.create_user(username='user2', email='test@example.com')
```

## FastAPI Test Örneği

FastAPI uygulamalarını test etmek için kapsamlı bir örnek:

```python
import pytest
from fastapi import FastAPI, HTTPException, Depends
from fastapi.testclient import TestClient
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker, Session
from pydantic import BaseModel

# FastAPI app
app = FastAPI()

# Models
class UserCreate(BaseModel):
    username: str
    email: str

class UserResponse(BaseModel):
    id: int
    username: str
    email: str

# Database dependency
def get_db():
    """Veritabanı session dependency"""
    pass  # Gerçek implementasyon

# Routes
@app.post("/users/", response_model=UserResponse, status_code=201)
async def create_user(user: UserCreate, db: Session = Depends(get_db)):
    """Kullanıcı oluştur"""
    # Kullanıcıyı veritabanına ekle
    return user

@app.get("/users/{user_id}", response_model=UserResponse)
async def get_user(user_id: int, db: Session = Depends(get_db)):
    """Kullanıcı getir"""
    # Kullanıcıyı veritabanından çek
    if user_id == 999:
        raise HTTPException(status_code=404, detail="User not found")
    return {"id": user_id, "username": "test", "email": "test@example.com"}

# Test fixtures
@pytest.fixture
def test_db():
    """Test veritabanı fixture'ı"""
    engine = create_engine('sqlite:///:memory:')
    TestingSessionLocal = sessionmaker(bind=engine)
    
    # Tabloları oluştur
    # Base.metadata.create_all(bind=engine)
    
    yield TestingSessionLocal()
    
    engine.dispose()

@pytest.fixture
def client(test_db):
    """TestClient fixture'ı"""
    # Database dependency'yi override et
    def override_get_db():
        yield test_db
    
    app.dependency_overrides[get_db] = override_get_db
    
    with TestClient(app) as client:
        yield client
    
    app.dependency_overrides.clear()

# Tests
def test_create_user(client):
    """Kullanıcı oluşturma endpoint testi"""
    response = client.post(
        "/users/",
        json={"username": "testuser", "email": "test@example.com"}
    )
    
    assert response.status_code == 201
    data = response.json()
    assert data["username"] == "testuser"
    assert data["email"] == "test@example.com"
    assert "id" in data

def test_get_user(client):
    """Kullanıcı getirme endpoint testi"""
    response = client.get("/users/1")
    
    assert response.status_code == 200
    data = response.json()
    assert data["id"] == 1

def test_get_user_not_found(client):
    """Kullanıcı bulunamadı testi"""
    response = client.get("/users/999")
    
    assert response.status_code == 404
    assert response.json()["detail"] == "User not found"

@pytest.mark.parametrize("username,email,expected_status", [
    ("valid", "valid@example.com", 201),
    ("", "test@example.com", 422),  # Invalid username
    ("test", "invalid-email", 422),  # Invalid email
    ("test", "", 422),  # Empty email
])
def test_create_user_validation(client, username, email, expected_status):
    """Kullanıcı validasyon testi"""
    response = client.post(
        "/users/",
        json={"username": username, "email": email}
    )
    assert response.status_code == expected_status
```

## Test Best Practices

### 1. Test İsimlendirme

```python
# İyi isimlendirme - ne test edildiği açık
def test_user_cannot_login_with_wrong_password():
    pass

def test_order_total_includes_tax():
    pass

def test_email_validation_rejects_invalid_format():
    pass

# Kötü isimlendirme - belirsiz
def test_user():
    pass

def test_1():
    pass

def test_function():
    pass
```

### 2. AAA Pattern (Arrange-Act-Assert)

```python
def test_shopping_cart_total():
    # Arrange - Test verilerini hazırla
    cart = ShoppingCart()
    product1 = Product(name="Book", price=10.0)
    product2 = Product(name="Pen", price=2.0)
    
    # Act - Test edilecek aksiyonu gerçekleştir
    cart.add_item(product1)
    cart.add_item(product2)
    total = cart.calculate_total()
    
    # Assert - Sonucu doğrula
    assert total == 12.0
```

### 3. Test Isolation

```python
# Her test bağımsız olmalı
@pytest.fixture(autouse=True)
def reset_database(test_db):
    """Her testten önce veritabanını temizle"""
    test_db.query(User).delete()
    test_db.commit()
    yield

def test_user_creation_1():
    # Bu test diğer testleri etkilememeli
    user = create_user("user1")
    assert user.username == "user1"

def test_user_creation_2():
    # user1 verisi yok - test isolation sağlandı
    user = create_user("user2")
    assert user.username == "user2"
```

### 4. Test Data Builders

```python
class UserBuilder:
    """Test için kullanıcı oluşturma builder'ı"""
    
    def __init__(self):
        self.username = "testuser"
        self.email = "test@example.com"
        self.is_active = True
        self.is_admin = False
    
    def with_username(self, username):
        self.username = username
        return self
    
    def with_email(self, email):
        self.email = email
        return self
    
    def as_admin(self):
        self.is_admin = True
        return self
    
    def inactive(self):
        self.is_active = False
        return self
    
    def build(self):
        return User(
            username=self.username,
            email=self.email,
            is_active=self.is_active,
            is_admin=self.is_admin
        )

# Kullanım
def test_admin_user():
    user = UserBuilder().with_username("admin").as_admin().build()
    assert user.is_admin is True

def test_inactive_user():
    user = UserBuilder().inactive().build()
    assert user.is_active is False
```

## CI/CD Entegrasyonu

GitHub Actions ile pytest entegrasyonu:

```yaml
# .github/workflows/test.yml
name: Tests

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: [3.8, 3.9, "3.10", "3.11"]
    
    services:
      postgres:
        image: postgres:14
        env:
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: test_db
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
      
      redis:
        image: redis:7
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 6379:6379
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Set up Python ${{ matrix.python-version }}
      uses: actions/setup-python@v4
      with:
        python-version: ${{ matrix.python-version }}
    
    - name: Install dependencies
      run: |
        python -m pip install --upgrade pip
        pip install pytest pytest-cov pytest-asyncio pytest-mock
        pip install -r requirements.txt
    
    - name: Run tests with coverage
      env:
        DATABASE_URL: postgresql://postgres:postgres@localhost:5432/test_db
        REDIS_URL: redis://localhost:6379
      run: |
        pytest --cov=./src --cov-report=xml --cov-report=term-missing -v
    
    - name: Upload coverage to Codecov
      uses: codecov/codecov-action@v3
      with:
        file: ./coverage.xml
        fail_ci_if_error: true
```

## Sonuç

pytest, Python ekosistemindeki en güçlü ve esnek test framework'lerinden biridir. Basit syntax'ı ile başlaması kolay, ancak fixture sistemi, parametrize, mocking ve plugin ekosistemi ile karmaşık test senaryolarını da kolayca handle edebilirsiniz.

Bu yazıda ele aldığımız konular:
- pytest'in temel özellikleri ve avantajları
- Fixture sistemi ve scope yönetimi
- Parametrize ile çoklu test senaryoları
- Mocking ve patching teknikleri
- Async test yazma
- Test coverage analizi
- Custom marker'lar ve test organizasyonu
- Popüler pytest pluginleri
- FastAPI ve Django test entegrasyonları
- Test best practice'leri
- CI/CD pipeline entegrasyonu

Başarılı bir test stratejisi, kod kalitesini artırır, refactoring'i güvenli hale getirir ve deployment confidence'ı sağlar. pytest ile bu stratejinizi oluşturabilir ve sürdürülebilir bir test suite geliştirebilirsiniz.

**Kaynaklar:**
- [pytest Documentation](https://docs.pytest.org/)
- [pytest Plugins](https://docs.pytest.org/en/latest/reference/plugin_list.html)
- [Real Python pytest Tutorial](https://realpython.com/pytest-python-testing/)
- [FastAPI Testing Guide](https://fastapi.tiangolo.com/tutorial/testing/)
