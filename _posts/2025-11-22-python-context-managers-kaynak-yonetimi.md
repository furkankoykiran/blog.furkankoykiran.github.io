---
title: "Python Context Managers: Kaynak Yönetiminde Ustalık"
description: "Dosya açıp kapatmanın ötesine geçin. Database Transaction'ları, Kilitler (Locks) ve Asenkron süreçleri 'with' bloğuyla yönetmek."
date: "2025-11-22 15:00:00 +0300"
categories: [Backend, Python]
tags: [python, architecture, clean-code, asyncio, optimization]
image:
  path: /assets/img/posts/python-context-manager-with-statement.jpg
  alt: "Python Context Manager Mechanics"
---

Python'da `with open('file.txt')` kalıbını herkes bilir.
Ama çoğu geliştirici, bu yapının (Context Manager) gücünü sadece dosya işlemleriyle sınırlı sanır.
Oysa bu, Python'un en güçlü tasarım desenlerinden (Design Pattern) biridir.
C++ dünyasındaki "Kaynak Edinme Başlangıçta Yapılır" (RAII) prensibinin Pythoncasıdır.
Veritabanı transactionları, Thread kilitleri, Ağ bağlantıları... "Başlangıcı ve sonu belli olan" her şey için kendi Context Manager'larınızı yazmalısınız.
Manuel olarak `close()` çağırmak insani bir hatadır ve unutulmaya mahkumdur. `with` bloku, hata olsa bile temizliği garanti eder.

## 1. Class Tabanlı Yönetim (`__enter__` & `__exit__`)

En temel (ve esnek) yöntem sınıf yazmaktır.
Buradaki asıl sihir `__exit__` metodunun içindeki hata yönetimidir.
Hata olsa bile çalışması garanti edilen bir `finally` bloğu gibi davranır.

```python
import sqlite3

class Transaction:
    def __init__(self, db_name):
        self.conn = sqlite3.connect(db_name)

    def __enter__(self):
        print("Transaction Başlıyor...")
        return self.conn.cursor()

    def __exit__(self, exc_type, exc_val, exc_tb):
        if exc_type:
            # Hata varsa Rollback yap
            print(f"Hata oluştu: {exc_val}. Geri alınıyor...")
            self.conn.rollback()
            # True dönersek hatayı yutar (Suppress), False dönersek fırlatır
            return False 
        
        # Hata yoksa Commitle
        print("İşlem Başarılı. Kaydediliyor...")
        self.conn.commit()
        self.conn.close()
```
Bu sayede iş mantığınızın içine `try-except-commit-rollback` spagettisi karışmaz. Kodunuz, hata yönetimi mantığından tamamen ayrışmış (Decoupled) olur.

![Context Manager Flow](/assets/img/posts/python-context-manager-with-statement.jpg)
*Execution Flow: Exception Handling mekanizması.*

## 2. Decorator ile Basitleştirme (`contextlib`)

Her şey için sınıf yazmak yorucudur.
Python `contextlib` modülü ile bir generator fonksiyonunu Context Manager'a çevirebilirsiniz. `yield` anahtar kelimesi, `__enter__` ve `__exit__` arasındaki sınırı çizer.

```python
from contextlib import contextmanager
import time
import os

@contextmanager
def timer(etiket):
    baslangic = time.time()
    try:
        yield # Burası with bloğunun içindeki kodun çalıştığı yer
    finally:
        bitis = time.time()
        print(f"[{etiket}] Geçen Süre: {bitis - baslangic:.4f} sn")

# Kullanımı: Çok temiz ve net
with timer("Veri İşleme"):
    yap_agir_islem()
```

## 3. Dinamik Yönetim: `contextlib.ExitStack`

Bazen kaç tane dosyayı veya kaynağı yöneteceğinizi çalışma zamanına (runtime) kadar bilemezsiniz.
Örneğin, dinamik bir dosya listesini açmak istediğinizde iç içe `with` blokları yazamazsınız.
Burada `ExitStack` hayat kurtarır.

```python
from contextlib import ExitStack

filenames = ["file1.txt", "file2.txt", "file3.txt"]

with ExitStack() as stack:
    # Tüm dosyaları dinamik olarak aç ve stack'e ekle
    files = [stack.enter_context(open(name)) for name in filenames]
    
    # Hepsiyle işlem yap
    for f in files:
        f.write("Log verisi")
        
# Bloktan çıkınca hepsi otomatik kapanır!
```
Bu yöntem özellikle plugin sistemlerinde veya bilinmeyen sayıda kaynağın (DB connections, Sockets) yönetilmesinde paha biçilemezdir.

## 4. Asenkron Context Managers (`async with`)

Modern Python (FastAPI döneminde) her şey asenkron.
Veritabanı havuzundan (Connection Pool) bağlantı almak bloklayan bir işlemdir.
Bunu `await` ile beklemezseniz Event Loop kilitlenir.

```python
import aiohttp

async def fetch_data(url):
    # Doğru (Pythonic Way)
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as response:
            return await response.text()
```
Bağlantı sızıntıları (Connection Leaks), yüksek trafikli asenkron sistemlerin bir numaralı katilidir. `async with` bunun sigortasıdır.

![Resource Management](/assets/img/posts/database-connection-pool-architecture.png)
*Kaynak Yaşam Döngüsü: Acquire -> Utilize -> Release.*

## 5. İleri Seviye: Reentrant Locks ve Dosya Kilitleri

Çoklu Thread veya Process çalışan ortamlarda "Race Condition" kaçınılmazdır.
İki process aynı dosyaya aynı anda yazmaya çalışırsa veri bozulur.
Context Manager ile bir "Kilit" (Lock) mekanizması kurmak çok temizdir.

```python
from filelock import FileLock

# 'data.txt.lock' dosyası varsa bekle, yoksa oluştur ve gir
lock = FileLock("data.txt.lock")

with lock:
    # Bu blok aynı anda sadece tek bir process tarafından çalıştırılabilir
    with open("data.txt", "a") as f:
        f.write("Güvenli veri\n")
```

![Custom Context Manager](/assets/img/posts/custom-context-managers-python.png)
*Kendi context manager'ınızı tasarlamak.*

## 6. Sık Yapılan Hatalar (Anti-Patterns)

1.  **Hatayı Yutmak (Swallowing Exceptions):**
    `__exit__` metodunda `return True` dönerseniz, `with` bloğunda oluşan hatayı sessizce yok edersiniz.
    ```python
    def __exit__(...):
        return True # YANLIŞ! Hata olduğundan haberdar olmalısınız.
    ```
    Sadece loglayıp `False` dönün veya `raise` ile fırlatın.

2.  **`__enter__` İçinde Ağır İş Yapmak:**
    `with` satırı mümkün olduğunca hızlı çalışmalıdır. Eğer `__enter__` içinde uzun süren bir API çağrısı yaparsanız, kodun akışı bloke olur. Bu tür işleri bloğun içine saklayın.

3.  **Context Manager Olmayanları Zorlamak:**
    Her şey `with` ile kullanılmak zorunda değildir. Sadece "Kurulum" (Setup) ve "Temizlik" (Teardown) aşamaları belirgin olan işlemler için kullanın.

## 7. `contextlib.suppress`: Estetik Hata Yönetimi

Bazen bir hata oluşursa sadece yoksaymak istersiniz. `try-except pass` bloğu Python dünyasında "Code Smell" olarak kabul edilir.

```python
from contextlib import suppress
import os

# Zarif ve okunaklı
with suppress(FileNotFoundError):
    os.remove('gereksiz_dosya.tmp')
```
Bu, kodun niyetini (Intent) çok daha net belli eder: "Bu hata önemli değil, devam et."

## Sonuç

Senior developer olmanın yolu, dili sadece "çalıştırmak" değil, onun "deyimlerini" (Idioms) kullanmaktan geçer.
Context Manager kullanmak, "Ben kaynak yönetimini ciddiye alıyorum, arkamda çöp bırakmıyorum" demektir.
Kodunuzda `try ... finally` blokları tekrar etmeye başladığında durun.
"Bunu bir Context Manager içine alabilir miyim?" diye düşünün.
Cevap %90 evettir ve kodunuz %100 daha temiz, daha güvenli ve daha bakımı kolay olacaktır.

![Python GIL](/assets/img/posts/async-await-concurrency-visualization.png)
*GIL ve Thread Safety bağlamında Context Manager rolü.*