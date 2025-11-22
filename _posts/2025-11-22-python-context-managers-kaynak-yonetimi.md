---
title: "Python Context Managers: İleri Seviye Kaynak Yönetimi ve Best Practices"
date: "2025-11-22 11:00:00 +0300"
categories: [Python, Software Development]
tags: [python, context-managers, resource-management, contextlib, with-statement, clean-code, best-practices, design-patterns]
image:
  src: /assets/img/posts/python-context-manager-with-statement.jpg
  alt: "Python Context Manager ve With Statement"
---

Python'da kaynak yönetimi, özellikle dosya işlemleri, veritabanı bağlantıları, network soketleri gibi dış kaynaklarla çalışırken kritik öneme sahiptir. Context managers, bu kaynakların güvenli ve otomatik olarak yönetilmesini sağlayan güçlü bir Python özelliğidir.

## Context Manager Nedir?

Context manager, `with` statement ile birlikte kullanılan ve kaynakların otomatik olarak açılıp kapatılmasını sağlayan Python objesidir. En bilinen örneği dosya işlemleridir.

**Temel Kullanım:**

```python
# Context manager ile dosya işlemi
with open('dosya.txt', 'r') as f:
    content = f.read()
    print(content)
# Dosya otomatik olarak kapatıldı

# Context manager olmadan (eski yöntem)
f = open('dosya.txt', 'r')
try:
    content = f.read()
    print(content)
finally:
    f.close()  # Manuel kapatma gerekli
```

Context manager kullanmanın avantajları:
- Otomatik kaynak temizleme
- Exception handling garantisi
- Daha temiz ve okunabilir kod
- Resource leak'lerden korunma

![Python Context Manager ve With Statement](/assets/img/posts/python-context-manager-with-statement.jpg)

## Context Manager Protocol

Context manager olmak için bir sınıfın `__enter__` ve `__exit__` methodlarını implement etmesi gerekir.

```python
class FileManager:
    def __init__(self, filename, mode):
        self.filename = filename
        self.mode = mode
        self.file = None
    
    def __enter__(self):
        """
        with bloğuna girildiğinde çağrılır.
        Return edilen değer 'as' keyword'ünden sonra atanır.
        """
        print(f"Opening {self.filename}")
        self.file = open(self.filename, self.mode)
        return self.file
    
    def __exit__(self, exc_type, exc_value, exc_traceback):
        """
        with bloğundan çıkıldığında çağrılır.
        
        Args:
            exc_type: Exception tipi (hata yoksa None)
            exc_value: Exception değeri (hata yoksa None)
            exc_traceback: Traceback objesi (hata yoksa None)
        
        Returns:
            True: Exception'ı suppress et
            False/None: Exception'ı propagate et
        """
        print(f"Closing {self.filename}")
        if self.file:
            self.file.close()
        
        # Exception handling
        if exc_type is not None:
            print(f"Exception occurred: {exc_type.__name__}: {exc_value}")
            # False dönerek exception'ı yukarı fırlat
            return False
        
        return True

# Kullanım
with FileManager('test.txt', 'w') as f:
    f.write('Hello, Context Manager!')
    # Exception test
    # raise ValueError("Test error")

# Output:
# Opening test.txt
# Closing test.txt
```

## Database Connection Manager

Veritabanı bağlantıları için context manager örneği:

```python
import psycopg2
from typing import Optional

class DatabaseConnection:
    def __init__(self, host: str, database: str, user: str, password: str):
        self.host = host
        self.database = database
        self.user = user
        self.password = password
        self.connection: Optional[psycopg2.extensions.connection] = None
        self.cursor: Optional[psycopg2.extensions.cursor] = None
    
    def __enter__(self):
        """Veritabanı bağlantısı ve cursor oluştur"""
        self.connection = psycopg2.connect(
            host=self.host,
            database=self.database,
            user=self.user,
            password=self.password
        )
        self.cursor = self.connection.cursor()
        return self.cursor
    
    def __exit__(self, exc_type, exc_value, exc_traceback):
        """
        Bağlantıyı kapat. 
        Exception yoksa commit, varsa rollback yap.
        """
        if exc_type is None:
            # Başarılı, commit yap
            self.connection.commit()
        else:
            # Hata oluştu, rollback yap
            self.connection.rollback()
            print(f"Transaction rolled back due to: {exc_value}")
        
        # Kaynakları temizle
        if self.cursor:
            self.cursor.close()
        if self.connection:
            self.connection.close()
        
        # Exception'ı propagate et
        return False

# Kullanım
try:
    with DatabaseConnection('localhost', 'mydb', 'user', 'pass') as cursor:
        cursor.execute("INSERT INTO users (name, email) VALUES (%s, %s)", 
                      ('John Doe', 'john@example.com'))
        cursor.execute("SELECT * FROM users WHERE name = %s", ('John Doe',))
        result = cursor.fetchall()
        print(result)
        # Hata olursa otomatik rollback
        # raise Exception("Something went wrong")
except Exception as e:
    print(f"Error: {e}")

# Bağlantı otomatik olarak kapatıldı ve commit edildi
```

## contextlib ile Context Manager Oluşturma

`contextlib` modülü, context manager oluşturmayı kolaylaştıran decorator'lar sağlar.

![Contextlib Decorator](/assets/img/posts/python-contextlib-decorator.png)

### @contextmanager Decorator

```python
from contextlib import contextmanager
import time

@contextmanager
def timer(name: str):
    """Kod bloğunun çalışma süresini ölçen context manager"""
    start_time = time.time()
    print(f"[{name}] Starting...")
    
    try:
        yield  # with bloğuna geç
    finally:
        # with bloğu bittikten sonra burası çalışır
        end_time = time.time()
        duration = end_time - start_time
        print(f"[{name}] Finished in {duration:.4f} seconds")

# Kullanım
with timer("Database Query"):
    time.sleep(2)  # Simüle edilen işlem
    print("Executing query...")

# Output:
# [Database Query] Starting...
# Executing query...
# [Database Query] Finished in 2.0001 seconds
```

### İç İçe Context Managers

```python
@contextmanager
def logging_context(name: str):
    """Log mesajları ile context manager"""
    print(f">>> Entering {name}")
    try:
        yield
    except Exception as e:
        print(f"!!! Exception in {name}: {e}")
        raise
    finally:
        print(f"<<< Exiting {name}")

# İç içe kullanım
with logging_context("Outer"):
    print("In outer context")
    
    with logging_context("Inner"):
        print("In inner context")
        # raise ValueError("Test error")
    
    print("Back to outer context")

# Output:
# >>> Entering Outer
# In outer context
# >>> Entering Inner
# In inner context
# <<< Exiting Inner
# Back to outer context
# <<< Exiting Outer
```

## Gerçek Dünya Örnekleri

### API Rate Limiter

```python
from contextlib import contextmanager
import time
from threading import Lock

class RateLimiter:
    def __init__(self, max_calls: int, time_frame: float):
        """
        Args:
            max_calls: Time frame içinde izin verilen maksimum çağrı sayısı
            time_frame: Zaman dilimi (saniye)
        """
        self.max_calls = max_calls
        self.time_frame = time_frame
        self.calls = []
        self.lock = Lock()
    
    @contextmanager
    def limit(self):
        """Rate limiting context manager"""
        with self.lock:
            now = time.time()
            
            # Eski çağrıları temizle
            self.calls = [call_time for call_time in self.calls 
                         if now - call_time < self.time_frame]
            
            # Limit kontrolü
            if len(self.calls) >= self.max_calls:
                # Bekle
                sleep_time = self.time_frame - (now - self.calls[0])
                print(f"Rate limit reached. Waiting {sleep_time:.2f}s...")
                time.sleep(sleep_time)
                self.calls = []
            
            # Çağrıyı kaydet
            self.calls.append(time.time())
        
        yield

# Kullanım
rate_limiter = RateLimiter(max_calls=5, time_frame=10.0)

def make_api_call(i):
    with rate_limiter.limit():
        print(f"Making API call {i} at {time.time():.2f}")
        # API çağrısı simülasyonu
        time.sleep(0.5)

# 10 çağrı yap (5'ten sonra bekleyecek)
for i in range(10):
    make_api_call(i)
```

![Custom Context Managers](/assets/img/posts/custom-context-managers-python.png)

### Temporary Directory Manager

```python
import os
import tempfile
import shutil
from contextlib import contextmanager

@contextmanager
def temporary_directory(suffix: str = '', prefix: str = 'tmp'):
    """
    Geçici dizin oluşturur ve işlem bitince siler.
    """
    temp_dir = tempfile.mkdtemp(suffix=suffix, prefix=prefix)
    print(f"Created temporary directory: {temp_dir}")
    
    try:
        yield temp_dir
    finally:
        # Dizini ve içeriğini sil
        shutil.rmtree(temp_dir)
        print(f"Cleaned up temporary directory: {temp_dir}")

# Kullanım
with temporary_directory(prefix='myapp_') as temp_dir:
    # Geçici dosyalar oluştur
    file_path = os.path.join(temp_dir, 'test.txt')
    with open(file_path, 'w') as f:
        f.write('Temporary data')
    
    print(f"Working with: {file_path}")
    print(f"File exists: {os.path.exists(file_path)}")

# Dizin otomatik olarak silindi
```

### Configuration Override Manager

```python
from contextlib import contextmanager
from typing import Dict, Any
import copy

class Config:
    """Global configuration singleton"""
    _instance = None
    _config = {
        'debug': False,
        'api_url': 'https://api.example.com',
        'timeout': 30,
        'retry_count': 3
    }
    
    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance
    
    def get(self, key: str, default=None):
        return self._config.get(key, default)
    
    def set(self, key: str, value: Any):
        self._config[key] = value
    
    def update(self, updates: Dict[str, Any]):
        self._config.update(updates)
    
    @contextmanager
    def override(self, **overrides):
        """
        Geçici olarak config değerlerini override et.
        Context'ten çıkınca eski değerlere dön.
        """
        # Mevcut değerleri kaydet
        original_values = {
            key: self._config.get(key) 
            for key in overrides.keys()
        }
        
        try:
            # Override değerlerini uygula
            self.update(overrides)
            yield self
        finally:
            # Eski değerlere dön
            self.update(original_values)

# Kullanım
config = Config()

print(f"Original debug: {config.get('debug')}")  # False

with config.override(debug=True, timeout=60):
    print(f"Overridden debug: {config.get('debug')}")  # True
    print(f"Overridden timeout: {config.get('timeout')}")  # 60
    
    # Test kodları çalıştır
    # ...

print(f"Restored debug: {config.get('debug')}")  # False
print(f"Restored timeout: {config.get('timeout')}")  # 30
```

### HTTP Session Manager

```python
from contextlib import contextmanager
import requests
from typing import Optional

class SessionManager:
    def __init__(self, base_url: str, timeout: int = 30):
        self.base_url = base_url
        self.timeout = timeout
        self.session: Optional[requests.Session] = None
    
    @contextmanager
    def get_session(self, headers: dict = None):
        """
        HTTP session context manager.
        Connection pooling ve automatic retry sağlar.
        """
        session = requests.Session()
        
        # Default headers
        session.headers.update({
            'User-Agent': 'MyApp/1.0',
            'Accept': 'application/json'
        })
        
        # Custom headers
        if headers:
            session.headers.update(headers)
        
        # Retry adapter
        from requests.adapters import HTTPAdapter
        from urllib3.util.retry import Retry
        
        retry_strategy = Retry(
            total=3,
            backoff_factor=1,
            status_forcelist=[429, 500, 502, 503, 504]
        )
        adapter = HTTPAdapter(max_retries=retry_strategy)
        session.mount("http://", adapter)
        session.mount("https://", adapter)
        
        try:
            yield session
        finally:
            session.close()
    
    def fetch(self, endpoint: str, method: str = 'GET', **kwargs):
        """Session ile HTTP request yap"""
        with self.get_session() as session:
            url = f"{self.base_url}/{endpoint.lstrip('/')}"
            response = session.request(
                method, 
                url, 
                timeout=self.timeout,
                **kwargs
            )
            response.raise_for_status()
            return response.json()

# Kullanım
api = SessionManager('https://api.github.com')

# Tekli request
try:
    data = api.fetch('users/octocat')
    print(data['name'])
except requests.exceptions.RequestException as e:
    print(f"Error: {e}")

# Çoklu request (aynı session)
with api.get_session() as session:
    # Connection pooling sayesinde hızlı
    for username in ['octocat', 'torvalds', 'gvanrossum']:
        response = session.get(f'{api.base_url}/users/{username}')
        if response.ok:
            data = response.json()
            print(f"{data['name']}: {data['public_repos']} repos")
```

## contextlib Utilities

### suppress: Exception'ları Bastırma

```python
from contextlib import suppress
import os

# Exception handling olmadan
try:
    os.remove('nonexistent.txt')
except FileNotFoundError:
    pass

# suppress ile daha temiz
with suppress(FileNotFoundError):
    os.remove('nonexistent.txt')

# Çoklu exception
with suppress(FileNotFoundError, PermissionError):
    os.remove('protected_file.txt')
```

### redirect_stdout ve redirect_stderr

```python
from contextlib import redirect_stdout, redirect_stderr
import sys
import io

# stdout'u dosyaya yönlendir
with open('output.txt', 'w') as f:
    with redirect_stdout(f):
        print('Bu dosyaya yazılacak')
        print('Terminal'de görünmeyecek')

# stdout'u string buffer'a yönlendir
output = io.StringIO()
with redirect_stdout(output):
    print('Captured output')
    help(str.upper)

captured = output.getvalue()
print(f"Captured {len(captured)} characters")

# stderr yönlendirme
error_log = io.StringIO()
with redirect_stderr(error_log):
    print("This is an error", file=sys.stderr)

print(error_log.getvalue())
```

### ExitStack: Dinamik Context Manager'lar

```python
from contextlib import ExitStack
import os

def process_files(filenames):
    """Birden fazla dosyayı aynı anda aç"""
    with ExitStack() as stack:
        # Dinamik sayıda dosya aç
        files = [
            stack.enter_context(open(fname, 'r'))
            for fname in filenames
        ]
        
        # Tüm dosyalarla çalış
        for f in files:
            content = f.read()
            print(f"{f.name}: {len(content)} bytes")
        
        # ExitStack otomatik olarak tüm dosyaları kapatır

# Kullanım
file_list = ['file1.txt', 'file2.txt', 'file3.txt']
# Önce dosyaları oluştur
for fname in file_list:
    with open(fname, 'w') as f:
        f.write(f'Content of {fname}')

process_files(file_list)

# Cleanup
for fname in file_list:
    os.remove(fname)
```

### Callback ile ExitStack

```python
from contextlib import ExitStack

def setup_resources():
    """Callback'lerle kaynak yönetimi"""
    stack = ExitStack()
    
    # Kaynak ayır
    resource1 = "Database Connection"
    print(f"Opening {resource1}")
    
    # Cleanup callback kaydet
    stack.callback(lambda: print(f"Closing {resource1}"))
    
    # Başka kaynak
    resource2 = "File Handle"
    print(f"Opening {resource2}")
    stack.callback(lambda: print(f"Closing {resource2}"))
    
    return stack

# Kullanım
with setup_resources():
    print("Working with resources...")
    # İşlemler yap

# Output:
# Opening Database Connection
# Opening File Handle
# Working with resources...
# Closing File Handle
# Closing Database Connection
```

## Async Context Managers

Asenkron programlama için async context manager'lar:

```python
import asyncio
from contextlib import asynccontextmanager

class AsyncDatabaseConnection:
    def __init__(self, connection_string: str):
        self.connection_string = connection_string
        self.connection = None
    
    async def __aenter__(self):
        """Async enter method"""
        print("Connecting to database...")
        await asyncio.sleep(1)  # Simüle edilen bağlantı
        self.connection = f"Connection to {self.connection_string}"
        return self.connection
    
    async def __aexit__(self, exc_type, exc_value, exc_traceback):
        """Async exit method"""
        print("Closing database connection...")
        await asyncio.sleep(0.5)  # Simüle edilen kapatma
        self.connection = None
        return False

# asynccontextmanager decorator ile
@asynccontextmanager
async def async_timer(name: str):
    """Async işlemler için timer"""
    import time
    start = time.time()
    print(f"[{name}] Started")
    
    try:
        yield
    finally:
        duration = time.time() - start
        print(f"[{name}] Completed in {duration:.2f}s")

# Kullanım
async def main():
    # Async context manager
    async with AsyncDatabaseConnection("postgresql://localhost") as conn:
        print(f"Using {conn}")
        await asyncio.sleep(1)
    
    # Async timer
    async with async_timer("API Call"):
        await asyncio.sleep(2)
        print("Making API call...")

# Çalıştır
asyncio.run(main())
```

## Best Practices

### 1. Her Zaman Context Manager Kullan

```python
# ❌ Kötü: Manuel kaynak yönetimi
file = open('data.txt', 'w')
file.write('data')
file.close()  # Hata olursa çalışmaz!

# ✅ İyi: Context manager
with open('data.txt', 'w') as file:
    file.write('data')
# Otomatik kapatma garantisi
```

### 2. Custom Exception Handling

```python
class TransactionContext:
    def __init__(self, db):
        self.db = db
    
    def __enter__(self):
        self.db.begin_transaction()
        return self.db
    
    def __exit__(self, exc_type, exc_value, exc_traceback):
        if exc_type is None:
            self.db.commit()
        else:
            self.db.rollback()
            # Specific exception handling
            if exc_type == ValueError:
                print("ValueError occurred, rolling back")
                return True  # Suppress exception
        return False  # Propagate other exceptions
```

### 3. Reusable Context Managers

```python
from functools import wraps

def retry_context(max_attempts=3):
    """Decorator for retrying context manager operations"""
    @contextmanager
    def wrapper():
        last_exception = None
        for attempt in range(max_attempts):
            try:
                yield attempt
                break
            except Exception as e:
                last_exception = e
                print(f"Attempt {attempt + 1} failed: {e}")
                if attempt == max_attempts - 1:
                    raise last_exception
    
    return wrapper()

# Kullanım
with retry_context(max_attempts=3):
    # Başarısız olabilecek işlem
    import random
    if random.random() < 0.7:
        raise ConnectionError("Connection failed")
    print("Success!")
```

## Performans İpuçları

```python
import time
from contextlib import contextmanager

@contextmanager
def measure_performance(operation_name: str):
    """Performans ölçümü için context manager"""
    import tracemalloc
    
    # Memory tracking başlat
    tracemalloc.start()
    start_time = time.perf_counter()
    start_memory = tracemalloc.get_traced_memory()[0]
    
    try:
        yield
    finally:
        end_time = time.perf_counter()
        end_memory = tracemalloc.get_traced_memory()[0]
        tracemalloc.stop()
        
        duration = end_time - start_time
        memory_used = (end_memory - start_memory) / 1024 / 1024  # MB
        
        print(f"\n{operation_name} Performance:")
        print(f"  Time: {duration:.4f} seconds")
        print(f"  Memory: {memory_used:.2f} MB")

# Kullanım
with measure_performance("List Comprehension"):
    result = [i**2 for i in range(1000000)]

with measure_performance("Generator Expression"):
    result = list(i**2 for i in range(1000000))
```

## Sonuç

Context managers, Python'da kaynak yönetiminin temel taşlarından biridir. `with` statement ve context manager protocol kullanarak:

- **Otomatik kaynak temizleme** garantisi sağlarsınız
- **Exception-safe** kod yazarsınız
- **Daha okunabilir** ve maintainable kod üretirsiniz
- **Resource leak'lerden** korunursunuz

**Önemli noktalar:**
- `__enter__` ve `__exit__` methodlarını doğru implement edin
- `contextlib.contextmanager` decorator'ı basit durumlar için kullanın
- Async operasyonlar için `async with` ve `__aenter__`/`__aexit__` kullanın
- `ExitStack` dinamik kaynak yönetimi için idealdir
- Her dış kaynak erişiminde context manager kullanmayı düşünün

Context managers, clean code prensiplerine uygun, güvenli ve profesyonel Python kodu yazmanın vazgeçilmez aracıdır.
