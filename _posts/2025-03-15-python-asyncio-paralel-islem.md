---
title: "Python asyncio ile Paralel İşlem: Asenkron Programlama Rehberi"
date: 2025-03-15 10:00:00 +0300
categories: [Python, Async Programming]
tags: [python, asyncio, async-await, coroutines, concurrency, event-loop, parallelism, performance, optimization, aiohttp]
image:
  path: /assets/img/posts/python-asyncio-event-loop.svg
  alt: "Python asyncio Event Loop Architecture"
---

## Giriş

Modern yazılım geliştirmede performans ve ölçeklenebilirlik kritik önem taşır. Python'un `asyncio` modülü, tek bir thread içinde binlerce eşzamanlı işlemi yönetebilme yeteneği sağlar. Bu rehberde, asyncio ile asenkron programlamanın temellerinden ileri seviye tekniklerine kadar tüm detayları öğreneceksiniz.

Asyncio, I/O-bound işlemlerde (network istekleri, dosya okuma/yazma, veritabanı sorguları) CPU'yu boşta bekletmek yerine diğer görevlere geçiş yaparak maksimum verimlilik sağlar.

## Senkron vs Asenkron Programlama

### Senkron Yaklaşım

```python
import time
import requests

def fetch_data(url):
    """Senkron HTTP isteği"""
    response = requests.get(url)
    return response.json()

def main():
    urls = [
        'https://api.github.com/users/python',
        'https://api.github.com/users/django',
        'https://api.github.com/users/flask',
        'https://api.github.com/users/fastapi',
    ]
    
    start = time.time()
    results = []
    
    for url in urls:
        # Her istek diğerini bekler (blocking)
        data = fetch_data(url)
        results.append(data)
    
    elapsed = time.time() - start
    print(f"Toplam süre: {elapsed:.2f} saniye")
    # Çıktı: ~4-6 saniye (her istek ~1-1.5 saniye)
    
    return results

if __name__ == "__main__":
    main()
```

### Asenkron Yaklaşım

```python
import asyncio
import aiohttp
import time

async def fetch_data(session, url):
    """Asenkron HTTP isteği"""
    async with session.get(url) as response:
        return await response.json()

async def main():
    urls = [
        'https://api.github.com/users/python',
        'https://api.github.com/users/django',
        'https://api.github.com/users/flask',
        'https://api.github.com/users/fastapi',
    ]
    
    start = time.time()
    
    async with aiohttp.ClientSession() as session:
        # Tüm istekler eşzamanlı çalışır
        tasks = [fetch_data(session, url) for url in urls]
        results = await asyncio.gather(*tasks)
    
    elapsed = time.time() - start
    print(f"Toplam süre: {elapsed:.2f} saniye")
    # Çıktı: ~1-1.5 saniye (paralel çalışma)
    
    return results

if __name__ == "__main__":
    asyncio.run(main())
```

![Async/Await Concurrency Visualization](/assets/img/posts/async-await-concurrency-visualization.png)

## Event Loop Nedir?

Event loop, asyncio'nun kalbidir. Tüm asenkron görevleri yönetir, bekleyen I/O işlemlerini takip eder ve hazır olan görevleri çalıştırır.

### Event Loop Temel Kavramları

```python
import asyncio

# Event loop'a manuel erişim
loop = asyncio.get_event_loop()

# Python 3.7+ ile önerilen yöntem
asyncio.run(main())  # Otomatik loop yönetimi

# Event loop bilgisi
def show_loop_info():
    loop = asyncio.get_running_loop()
    print(f"Loop running: {loop.is_running()}")
    print(f"Loop closed: {loop.is_closed()}")
    print(f"Debug mode: {loop.get_debug()}")

asyncio.run(show_loop_info())
```

### Event Loop Yaşam Döngüsü

```python
import asyncio

async def task1():
    print("Task 1: Başladı")
    await asyncio.sleep(1)  # I/O simülasyonu
    print("Task 1: Tamamlandı")
    return "Sonuç 1"

async def task2():
    print("Task 2: Başladı")
    await asyncio.sleep(0.5)
    print("Task 2: Tamamlandı")
    return "Sonuç 2"

async def main():
    print("Main başladı")
    
    # Event loop bu görevleri yönetir
    result1, result2 = await asyncio.gather(task1(), task2())
    
    print(f"Sonuçlar: {result1}, {result2}")
    print("Main tamamlandı")

# Çıktı:
# Main başladı
# Task 1: Başladı
# Task 2: Başladı
# Task 2: Tamamlandı
# Task 1: Tamamlandı
# Sonuçlar: Sonuç 1, Sonuç 2
# Main tamamlandı

asyncio.run(main())
```

## Coroutines (Eş Yordamlar)

Coroutine, duraklatılıp devam ettirilebilen fonksiyonlardır. `async def` ile tanımlanır ve `await` ile çağrılır.

### Coroutine Tanımlama

```python
import asyncio

# Basit coroutine
async def hello_world():
    print("Hello")
    await asyncio.sleep(1)  # Event loop'a kontrol verir
    print("World")
    return "Tamamlandı"

# Coroutine çalıştırma
result = asyncio.run(hello_world())
print(result)  # "Tamamlandı"
```

### Coroutine Zincirleme

```python
import asyncio

async def fetch_user(user_id):
    """Kullanıcı bilgisi getir"""
    print(f"Fetching user {user_id}...")
    await asyncio.sleep(1)  # API call simülasyonu
    return {"id": user_id, "name": f"User {user_id}"}

async def fetch_user_posts(user_id):
    """Kullanıcı postlarını getir"""
    print(f"Fetching posts for user {user_id}...")
    await asyncio.sleep(0.5)
    return [
        {"id": 1, "title": "Post 1"},
        {"id": 2, "title": "Post 2"}
    ]

async def get_user_data(user_id):
    """Kullanıcı ve postlarını birlikte getir"""
    # Sıralı çalışma
    user = await fetch_user(user_id)
    posts = await fetch_user_posts(user_id)
    
    user['posts'] = posts
    return user

async def main():
    user_data = await get_user_data(123)
    print(f"User: {user_data}")

asyncio.run(main())
```

![Python Coroutine Execution Flow](/assets/img/posts/python-coroutine-execution-flow.png)

### Paralel Coroutine Çalıştırma

```python
import asyncio

async def get_user_data_parallel(user_id):
    """Kullanıcı ve postlarını paralel getir"""
    # asyncio.gather ile paralel çalışma
    user, posts = await asyncio.gather(
        fetch_user(user_id),
        fetch_user_posts(user_id)
    )
    
    user['posts'] = posts
    return user

async def main():
    # Tek kullanıcı - paralel data fetching
    start = time.time()
    user_data = await get_user_data_parallel(123)
    print(f"Süre: {time.time() - start:.2f}s")  # ~1s (paralel)
    
    # Çok kullanıcı - tümü paralel
    start = time.time()
    users = await asyncio.gather(*[
        get_user_data_parallel(i) for i in range(1, 6)
    ])
    print(f"5 kullanıcı süresi: {time.time() - start:.2f}s")  # ~1s

asyncio.run(main())
```

## async/await Syntax

### await Kullanımı

```python
import asyncio

async def cpu_bound_work():
    """CPU-bound işlem (async'e uygun değil!)"""
    total = sum(i * i for i in range(10**7))
    return total

async def io_bound_work():
    """I/O-bound işlem (async'e uygun)"""
    await asyncio.sleep(1)  # Network/disk I/O
    return "Completed"

async def main():
    # ❌ Yanlış: CPU-bound işlemde await gereksiz
    result1 = await cpu_bound_work()
    
    # ✅ Doğru: I/O-bound işlemde await gerekli
    result2 = await io_bound_work()
    
    # ✅ Doğru: Paralel I/O işlemleri
    results = await asyncio.gather(
        io_bound_work(),
        io_bound_work(),
        io_bound_work()
    )

asyncio.run(main())
```

### Await Olmadan Coroutine

```python
import asyncio

async def my_coroutine():
    await asyncio.sleep(1)
    return "Result"

async def main():
    # ❌ Yanlış: await yok - coroutine çalışmaz
    coro = my_coroutine()
    print(coro)  # <coroutine object my_coroutine at 0x...>
    # RuntimeWarning: coroutine 'my_coroutine' was never awaited
    
    # ✅ Doğru: await ile çalıştır
    result = await my_coroutine()
    print(result)  # "Result"

asyncio.run(main())
```

## Tasks ve Task Yönetimi

Task, event loop tarafından zamanlanmış bir coroutine wrapper'ıdır.

### Task Oluşturma

```python
import asyncio

async def background_task(name, delay):
    """Arka plan görevi"""
    print(f"{name} başladı")
    await asyncio.sleep(delay)
    print(f"{name} tamamlandı")
    return f"{name} sonucu"

async def main():
    # Task oluşturma
    task1 = asyncio.create_task(background_task("Task-1", 2))
    task2 = asyncio.create_task(background_task("Task-2", 1))
    
    print("Tasks oluşturuldu, ana iş devam ediyor...")
    await asyncio.sleep(0.5)
    print("Ana iş tamamlandı")
    
    # Task sonuçlarını bekle
    result1 = await task1
    result2 = await task2
    
    print(f"Sonuçlar: {result1}, {result2}")

asyncio.run(main())
```

![Asyncio Task Queue Processing](/assets/img/posts/asyncio-task-queue-processing.png)

### Task İptali

```python
import asyncio

async def long_running_task():
    """Uzun süren görev"""
    try:
        print("Task başladı")
        await asyncio.sleep(10)
        print("Task tamamlandı")
    except asyncio.CancelledError:
        print("Task iptal edildi!")
        raise  # Exception'ı yeniden fırlat

async def main():
    task = asyncio.create_task(long_running_task())
    
    await asyncio.sleep(2)
    
    # Task'ı iptal et
    task.cancel()
    
    try:
        await task
    except asyncio.CancelledError:
        print("Main: Task iptal edildiğini bildirdi")

asyncio.run(main())
```

### Task Durumu Kontrolü

```python
import asyncio

async def monitored_task(delay):
    await asyncio.sleep(delay)
    return f"Completed after {delay}s"

async def main():
    task = asyncio.create_task(monitored_task(2))
    
    # Task durumu
    print(f"Done: {task.done()}")  # False
    print(f"Cancelled: {task.cancelled()}")  # False
    
    await asyncio.sleep(0.5)
    print(f"Still running, done: {task.done()}")  # False
    
    result = await task
    print(f"Done: {task.done()}")  # True
    print(f"Result: {task.result()}")  # "Completed after 2s"

asyncio.run(main())
```

## asyncio.gather vs asyncio.wait

### asyncio.gather

```python
import asyncio
import random

async def fetch_data(source):
    delay = random.uniform(0.5, 2.0)
    await asyncio.sleep(delay)
    return f"Data from {source}"

async def main_gather():
    # gather: Tüm sonuçları liste olarak döner
    results = await asyncio.gather(
        fetch_data("API-1"),
        fetch_data("API-2"),
        fetch_data("API-3"),
        return_exceptions=True  # Exception yakalama
    )
    
    print("Gather results:", results)
    # ['Data from API-1', 'Data from API-2', 'Data from API-3']

asyncio.run(main_gather())
```

### asyncio.wait

```python
import asyncio

async def main_wait():
    tasks = [
        asyncio.create_task(fetch_data("API-1")),
        asyncio.create_task(fetch_data("API-2")),
        asyncio.create_task(fetch_data("API-3")),
    ]
    
    # wait: done ve pending setleri döner
    done, pending = await asyncio.wait(
        tasks,
        return_when=asyncio.FIRST_COMPLETED
    )
    
    print(f"Tamamlanan: {len(done)}")
    print(f"Bekleyen: {len(pending)}")
    
    for task in done:
        print(f"Result: {task.result()}")
    
    # Kalan görevleri iptal et
    for task in pending:
        task.cancel()

asyncio.run(main_wait())
```

### gather vs wait Karşılaştırma

```python
import asyncio
import time

async def task_with_timeout(name, duration):
    await asyncio.sleep(duration)
    return f"{name} completed"

async def compare_gather_wait():
    # gather: Basit, sonuç listesi
    print("=== GATHER ===")
    start = time.time()
    results = await asyncio.gather(
        task_with_timeout("T1", 1),
        task_with_timeout("T2", 2),
        task_with_timeout("T3", 1.5)
    )
    print(f"Gather sonuçları: {results}")
    print(f"Süre: {time.time() - start:.2f}s\n")
    
    # wait: Esnek, done/pending kontrolü
    print("=== WAIT (ALL_COMPLETED) ===")
    start = time.time()
    tasks = [
        asyncio.create_task(task_with_timeout("T1", 1)),
        asyncio.create_task(task_with_timeout("T2", 2)),
        asyncio.create_task(task_with_timeout("T3", 1.5))
    ]
    done, pending = await asyncio.wait(tasks)
    results = [task.result() for task in done]
    print(f"Wait sonuçları: {results}")
    print(f"Süre: {time.time() - start:.2f}s\n")
    
    # wait: İlk tamamlanana kadar bekle
    print("=== WAIT (FIRST_COMPLETED) ===")
    start = time.time()
    tasks = [
        asyncio.create_task(task_with_timeout("T1", 1)),
        asyncio.create_task(task_with_timeout("T2", 2)),
        asyncio.create_task(task_with_timeout("T3", 1.5))
    ]
    done, pending = await asyncio.wait(
        tasks,
        return_when=asyncio.FIRST_COMPLETED
    )
    first_result = list(done)[0].result()
    print(f"İlk sonuç: {first_result}")
    print(f"Süre: {time.time() - start:.2f}s")
    
    # Kalan görevleri iptal et
    for task in pending:
        task.cancel()

asyncio.run(compare_gather_wait())
```

## Timeout İşlemleri

### asyncio.wait_for

```python
import asyncio

async def slow_operation():
    """Yavaş işlem"""
    await asyncio.sleep(5)
    return "Completed"

async def main():
    try:
        # 2 saniye timeout
        result = await asyncio.wait_for(
            slow_operation(),
            timeout=2.0
        )
        print(result)
    except asyncio.TimeoutError:
        print("İşlem zaman aşımına uğradı!")

asyncio.run(main())
```

### Timeout Context Manager

```python
import asyncio

async def fetch_with_timeout(url, timeout=5):
    """Timeout ile HTTP isteği"""
    try:
        async with asyncio.timeout(timeout):  # Python 3.11+
            # Veya: async with asyncio.wait_for(..., timeout=timeout)
            await asyncio.sleep(2)  # Simulated request
            return f"Data from {url}"
    except asyncio.TimeoutError:
        return f"Timeout: {url}"

async def main():
    result = await fetch_with_timeout("https://example.com", timeout=1)
    print(result)  # "Timeout: https://example.com"

asyncio.run(main())
```

## Asenkron Context Managers

### async with Kullanımı

```python
import asyncio

class AsyncDatabaseConnection:
    """Asenkron veritabanı bağlantısı"""
    
    async def __aenter__(self):
        print("Bağlantı açılıyor...")
        await asyncio.sleep(0.5)  # Connection handshake
        print("Bağlantı açıldı")
        return self
    
    async def __aexit__(self, exc_type, exc_val, exc_tb):
        print("Bağlantı kapatılıyor...")
        await asyncio.sleep(0.2)  # Cleanup
        print("Bağlantı kapatıldı")
    
    async def query(self, sql):
        print(f"Sorgu: {sql}")
        await asyncio.sleep(0.3)
        return [{"id": 1, "name": "Result"}]

async def main():
    async with AsyncDatabaseConnection() as db:
        results = await db.query("SELECT * FROM users")
        print(f"Sonuçlar: {results}")

asyncio.run(main())
```

### Asenkron File Operations

```python
import asyncio
import aiofiles  # pip install aiofiles

async def read_file_async(filename):
    """Asenkron dosya okuma"""
    async with aiofiles.open(filename, 'r') as f:
        contents = await f.read()
        return contents

async def write_file_async(filename, data):
    """Asenkron dosya yazma"""
    async with aiofiles.open(filename, 'w') as f:
        await f.write(data)

async def main():
    # Paralel dosya işlemleri
    await asyncio.gather(
        write_file_async('file1.txt', 'Data 1'),
        write_file_async('file2.txt', 'Data 2'),
        write_file_async('file3.txt', 'Data 3')
    )
    
    # Paralel okuma
    contents = await asyncio.gather(
        read_file_async('file1.txt'),
        read_file_async('file2.txt'),
        read_file_async('file3.txt')
    )
    
    print("Dosya içerikleri:", contents)

asyncio.run(main())
```

## Asenkron Generators ve Iterators

### Asenkron Generator

```python
import asyncio

async def async_range(start, stop):
    """Asenkron range generator"""
    current = start
    while current < stop:
        await asyncio.sleep(0.1)  # Simulated async operation
        yield current
        current += 1

async def main():
    async for num in async_range(0, 5):
        print(f"Number: {num}")

asyncio.run(main())
```

### Asenkron Iterator

```python
import asyncio

class AsyncIterator:
    """Asenkron iterator sınıfı"""
    
    def __init__(self, max_count):
        self.max_count = max_count
        self.current = 0
    
    def __aiter__(self):
        return self
    
    async def __anext__(self):
        if self.current >= self.max_count:
            raise StopAsyncIteration
        
        await asyncio.sleep(0.1)
        result = self.current
        self.current += 1
        return result

async def main():
    async for item in AsyncIterator(5):
        print(f"Item: {item}")

asyncio.run(main())
```

### Asenkron Comprehension

```python
import asyncio

async def fetch_page(page_num):
    await asyncio.sleep(0.1)
    return f"Page {page_num} data"

async def main():
    # Asenkron list comprehension
    pages = [fetch_page(i) async for i in async_range(1, 6)]
    results = await asyncio.gather(*pages)
    print(results)
    
    # Asenkron dict comprehension
    data = {
        i: result 
        async for i, result in enumerate(
            [fetch_page(i) for i in range(1, 6)]
        )
    }

asyncio.run(main())
```

## Queue ile Producer-Consumer Pattern

### Asyncio Queue

```python
import asyncio
import random

async def producer(queue, producer_id):
    """Veri üreten"""
    for i in range(5):
        await asyncio.sleep(random.uniform(0.1, 0.5))
        item = f"P{producer_id}-Item{i}"
        await queue.put(item)
        print(f"Produced: {item}")
    
    print(f"Producer {producer_id} tamamlandı")

async def consumer(queue, consumer_id):
    """Veri tüketen"""
    while True:
        item = await queue.get()
        
        if item is None:  # Poison pill
            break
        
        print(f"Consumer {consumer_id} processing: {item}")
        await asyncio.sleep(random.uniform(0.2, 0.8))
        queue.task_done()
    
    print(f"Consumer {consumer_id} tamamlandı")

async def main():
    queue = asyncio.Queue(maxsize=10)
    
    # Producer'ları başlat
    producers = [
        asyncio.create_task(producer(queue, i))
        for i in range(2)
    ]
    
    # Consumer'ları başlat
    consumers = [
        asyncio.create_task(consumer(queue, i))
        for i in range(3)
    ]
    
    # Producer'ların bitmesini bekle
    await asyncio.gather(*producers)
    
    # Queue'nun boşalmasını bekle
    await queue.join()
    
    # Consumer'ları durdur
    for _ in consumers:
        await queue.put(None)
    
    await asyncio.gather(*consumers)

asyncio.run(main())
```

### Priority Queue

```python
import asyncio
from dataclasses import dataclass, field
from typing import Any

@dataclass(order=True)
class PrioritizedItem:
    priority: int
    item: Any = field(compare=False)

async def priority_worker(queue):
    """Öncelikli görev işleyici"""
    while True:
        prioritized = await queue.get()
        
        if prioritized is None:
            break
        
        print(f"Processing (priority {prioritized.priority}): {prioritized.item}")
        await asyncio.sleep(0.5)
        queue.task_done()

async def main():
    queue = asyncio.PriorityQueue()
    
    # Görevleri ekle (düşük sayı = yüksek öncelik)
    await queue.put(PrioritizedItem(3, "Low priority task"))
    await queue.put(PrioritizedItem(1, "High priority task"))
    await queue.put(PrioritizedItem(2, "Medium priority task"))
    await queue.put(PrioritizedItem(1, "Another high priority"))
    
    # Worker başlat
    worker = asyncio.create_task(priority_worker(queue))
    
    # Queue'nun boşalmasını bekle
    await queue.join()
    
    # Worker'ı durdur
    await queue.put(None)
    await worker

asyncio.run(main())
```

## Semaphore ve Lock

### Semaphore ile Eşzamanlılık Kontrolü

```python
import asyncio

async def limited_resource(semaphore, resource_id):
    """Sınırlı kaynağa erişim"""
    async with semaphore:
        print(f"Resource {resource_id} accessing limited resource")
        await asyncio.sleep(1)
        print(f"Resource {resource_id} released")

async def main():
    # Maksimum 3 eşzamanlı erişim
    semaphore = asyncio.Semaphore(3)
    
    # 10 görev oluştur, ama sadece 3'ü aynı anda çalışır
    tasks = [
        limited_resource(semaphore, i)
        for i in range(10)
    ]
    
    await asyncio.gather(*tasks)

asyncio.run(main())
```

### Lock ile Mutex

```python
import asyncio

class SharedCounter:
    """Paylaşılan sayaç (race condition örneği)"""
    
    def __init__(self):
        self.value = 0
        self.lock = asyncio.Lock()
    
    async def increment_unsafe(self):
        """Thread-unsafe artırma"""
        temp = self.value
        await asyncio.sleep(0.01)  # Context switch simülasyonu
        self.value = temp + 1
    
    async def increment_safe(self):
        """Thread-safe artırma"""
        async with self.lock:
            temp = self.value
            await asyncio.sleep(0.01)
            self.value = temp + 1

async def test_race_condition():
    counter = SharedCounter()
    
    # Race condition testi
    await asyncio.gather(*[
        counter.increment_unsafe() for _ in range(100)
    ])
    print(f"Unsafe counter: {counter.value}")  # < 100 (race condition)
    
    # Lock ile güvenli
    counter.value = 0
    await asyncio.gather(*[
        counter.increment_safe() for _ in range(100)
    ])
    print(f"Safe counter: {counter.value}")  # = 100

asyncio.run(test_race_condition())
```

## Asenkron HTTP İstekleri (aiohttp)

### Temel aiohttp Kullanımı

```python
import asyncio
import aiohttp

async def fetch_url(session, url):
    """Tek URL fetch"""
    async with session.get(url) as response:
        return await response.text()

async def main():
    async with aiohttp.ClientSession() as session:
        html = await fetch_url(session, 'https://example.com')
        print(f"Page length: {len(html)}")

asyncio.run(main())
```

### Paralel HTTP İstekleri

```python
import asyncio
import aiohttp
import time

async def fetch_github_user(session, username):
    """GitHub kullanıcı bilgisi fetch"""
    url = f'https://api.github.com/users/{username}'
    async with session.get(url) as response:
        data = await response.json()
        return {
            'username': username,
            'name': data.get('name'),
            'followers': data.get('followers')
        }

async def main():
    usernames = [
        'torvalds', 'gvanrossum', 'octocat',
        'defunkt', 'pjhyett', 'mojombo'
    ]
    
    start = time.time()
    
    async with aiohttp.ClientSession() as session:
        tasks = [
            fetch_github_user(session, username)
            for username in usernames
        ]
        users = await asyncio.gather(*tasks, return_exceptions=True)
    
    elapsed = time.time() - start
    
    for user in users:
        if isinstance(user, Exception):
            print(f"Error: {user}")
        else:
            print(f"{user['username']}: {user['name']} ({user['followers']} followers)")
    
    print(f"\nTotal time: {elapsed:.2f}s")

asyncio.run(main())
```

### Retry ve Error Handling

```python
import asyncio
import aiohttp
from typing import Optional

async def fetch_with_retry(
    session: aiohttp.ClientSession,
    url: str,
    max_retries: int = 3,
    timeout: float = 10.0
) -> Optional[dict]:
    """Retry mekanizması ile HTTP isteği"""
    
    for attempt in range(max_retries):
        try:
            async with session.get(url, timeout=timeout) as response:
                response.raise_for_status()
                return await response.json()
        
        except asyncio.TimeoutError:
            print(f"Attempt {attempt + 1}: Timeout")
            if attempt == max_retries - 1:
                raise
            await asyncio.sleep(2 ** attempt)  # Exponential backoff
        
        except aiohttp.ClientError as e:
            print(f"Attempt {attempt + 1}: {e}")
            if attempt == max_retries - 1:
                raise
            await asyncio.sleep(2 ** attempt)
    
    return None

async def main():
    async with aiohttp.ClientSession() as session:
        try:
            data = await fetch_with_retry(
                session,
                'https://api.github.com/users/octocat'
            )
            print(f"Success: {data['name']}")
        except Exception as e:
            print(f"Failed after retries: {e}")

asyncio.run(main())
```

## Veritabanı İşlemleri (asyncpg, motor)

### PostgreSQL (asyncpg)

```python
import asyncio
import asyncpg

async def fetch_users():
    """PostgreSQL async sorgu"""
    # Bağlantı havuzu oluştur
    pool = await asyncpg.create_pool(
        host='localhost',
        database='mydb',
        user='user',
        password='password',
        min_size=10,
        max_size=20
    )
    
    async with pool.acquire() as connection:
        # Tek satır
        user = await connection.fetchrow(
            'SELECT * FROM users WHERE id = $1',
            1
        )
        print(f"User: {user}")
        
        # Çoklu satır
        users = await connection.fetch(
            'SELECT * FROM users LIMIT 10'
        )
        print(f"Users: {len(users)}")
    
    await pool.close()

asyncio.run(fetch_users())
```

### MongoDB (motor)

```python
import asyncio
from motor.motor_asyncio import AsyncIOMotorClient

async def fetch_documents():
    """MongoDB async işlemler"""
    client = AsyncIOMotorClient('mongodb://localhost:27017')
    db = client['mydb']
    collection = db['users']
    
    # Tek doküman
    user = await collection.find_one({'username': 'john'})
    print(f"User: {user}")
    
    # Çoklu doküman
    cursor = collection.find({'age': {'$gt': 18}}).limit(10)
    users = await cursor.to_list(length=10)
    print(f"Adult users: {len(users)}")
    
    # Insert
    result = await collection.insert_one({
        'username': 'jane',
        'email': 'jane@example.com',
        'age': 25
    })
    print(f"Inserted ID: {result.inserted_id}")
    
    client.close()

asyncio.run(fetch_documents())
```

## CPU-Bound İşlemler ve asyncio

CPU-bound işlemler için asyncio yeterli değildir. Bunun için `concurrent.futures` kullanılmalıdır.

### ProcessPoolExecutor ile Async

```python
import asyncio
import concurrent.futures
import time

def cpu_intensive_task(n):
    """CPU-bound işlem"""
    return sum(i * i for i in range(n))

async def main():
    loop = asyncio.get_running_loop()
    
    # ProcessPoolExecutor ile CPU-bound işlemler
    with concurrent.futures.ProcessPoolExecutor() as pool:
        start = time.time()
        
        results = await asyncio.gather(*[
            loop.run_in_executor(pool, cpu_intensive_task, 10**7)
            for _ in range(4)
        ])
        
        elapsed = time.time() - start
        print(f"Results: {results}")
        print(f"Time: {elapsed:.2f}s")

asyncio.run(main())
```

### ThreadPoolExecutor ile Blocking I/O

```python
import asyncio
import concurrent.futures
import requests  # Senkron library

def sync_http_request(url):
    """Senkron HTTP isteği"""
    response = requests.get(url)
    return response.json()

async def main():
    loop = asyncio.get_running_loop()
    
    urls = [
        'https://api.github.com/users/python',
        'https://api.github.com/users/django',
        'https://api.github.com/users/flask'
    ]
    
    # ThreadPoolExecutor ile senkron kodu async yap
    with concurrent.futures.ThreadPoolExecutor() as pool:
        results = await asyncio.gather(*[
            loop.run_in_executor(pool, sync_http_request, url)
            for url in urls
        ])
    
    for result in results:
        print(f"{result['name']}: {result['followers']} followers")

asyncio.run(main())
```

## Best Practices ve Anti-Patterns

### ✅ Best Practices

```python
import asyncio

# 1. asyncio.run() kullan (Python 3.7+)
async def main():
    pass

asyncio.run(main())

# 2. Task'ları cancel et
async def cleanup_example():
    task = asyncio.create_task(long_task())
    try:
        await asyncio.wait_for(task, timeout=5)
    except asyncio.TimeoutError:
        task.cancel()
        await task  # Cancel exception'ını yakalamak için await

# 3. Exception handling
async def safe_operation():
    try:
        result = await risky_operation()
    except Exception as e:
        logger.error(f"Error: {e}")
        raise

# 4. Semaphore ile rate limiting
async def rate_limited_requests():
    semaphore = asyncio.Semaphore(10)  # Max 10 concurrent
    async with semaphore:
        await make_request()

# 5. Connection pooling
async with aiohttp.ClientSession() as session:
    # Session tüm isteklerde yeniden kullanılır
    pass
```

### ❌ Anti-Patterns

```python
# ❌ 1. Blocking call'ları await ile çağırmak
async def bad_example():
    await time.sleep(1)  # ❌ Blocking!
    await asyncio.sleep(1)  # ✅ Non-blocking

# ❌ 2. CPU-bound işlemleri asyncio'da çalıştırmak
async def cpu_bound():
    result = sum(range(10**8))  # ❌ Event loop'u bloklar
    return result

# ❌ 3. await olmadan coroutine çağırmak
async def missing_await():
    fetch_data()  # ❌ RuntimeWarning
    await fetch_data()  # ✅

# ❌ 4. Her task için yeni session açmak
async def bad_http():
    async with aiohttp.ClientSession() as session:  # ❌ Her istekte yeni
        await session.get(url)

# ✅ Session'ı paylaş
async with aiohttp.ClientSession() as session:
    await asyncio.gather(*[
        session.get(url) for url in urls
    ])

# ❌ 5. Exception'ları ignore etmek
tasks = [task1(), task2()]
await asyncio.gather(*tasks, return_exceptions=False)  # ❌ İlk hata tümünü iptal eder
await asyncio.gather(*tasks, return_exceptions=True)  # ✅ Tüm sonuçları topla
```

## Performance Monitoring

### Async Task Profiling

```python
import asyncio
import time
from functools import wraps

def async_timed(func):
    """Async fonksiyon süre ölçer"""
    @wraps(func)
    async def wrapper(*args, **kwargs):
        start = time.perf_counter()
        try:
            return await func(*args, **kwargs)
        finally:
            elapsed = time.perf_counter() - start
            print(f"{func.__name__} took {elapsed:.4f}s")
    return wrapper

@async_timed
async def slow_operation():
    await asyncio.sleep(2)
    return "Done"

async def main():
    await slow_operation()  # slow_operation took 2.0000s

asyncio.run(main())
```

### Task Monitoring

```python
import asyncio

async def monitor_tasks():
    """Aktif task'ları izle"""
    while True:
        tasks = asyncio.all_tasks()
        print(f"Active tasks: {len(tasks)}")
        for task in tasks:
            print(f"  - {task.get_name()}: {task._state}")
        await asyncio.sleep(1)

async def main():
    monitor = asyncio.create_task(monitor_tasks(), name="Monitor")
    
    # Diğer görevler
    await asyncio.sleep(5)
    
    monitor.cancel()
    await monitor

asyncio.run(main())
```

## Sonuç

Python asyncio, I/O-bound işlemlerde muazzam performans kazanımları sağlar. Bu rehberde öğrendiğiniz teknikleri kullanarak:

- Web scraping işlemlerini 10x hızlandırabilirsiniz
- API sunucularınız binlerce eşzamanlı bağlantı işleyebilir
- Veritabanı işlemlerini paralel çalıştırabilirsiniz
- Mikroservis mimarisinde yüksek performans elde edebilirsiniz

### Kaynaklar

- [Python asyncio Documentation](https://docs.python.org/3/library/asyncio.html)
- [aiohttp Documentation](https://docs.aiohttp.org/)
- [Real Python: Async IO in Python](https://realpython.com/async-io-python/)
- [asyncpg Documentation](https://magicstack.github.io/asyncpg/)

Asyncio öğrenme eğrisi başta dik olabilir, ancak doğru kullanıldığında Python uygulamalarınızın performansını çarpıcı şekilde artırır! 🚀
