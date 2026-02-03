---
title: "Pytest ile Modern Test Mimarisi: Assert True'nun Ötesi"
description: "Unittest boilerplate kodlarından kurtulun. Fixture scope yönetimi, Mocking stratejileri ve Hypothesis ile property-based testing rehberi."
date: "2025-10-10 12:00:00 +0300"
categories: [Testing, Quality Assurance, Python, DevOps]
tags: [python, pytest, tdd, automation, mocking, ci-cd]
image:
  path: /assets/img/posts/pytest-fixtures-setup-pattern.png
  alt: "Pytest Fixtures Setup Pattern Diagram"
---

Yazılım geliştirme sürecinde "Test yazmak" genelde angarya olarak görülür. Kariyerime başladığımda ben de "Kod çalışıyor işte, neden test yazayım?" diyenlerdendim. Ancak production ortamında, gece yarısı gelen bir "hotfix"in başka bir yeri patlattığını görünce anladım: **Test yazmak, kodunuzun değil, akıl sağlığınızın sigortasıdır.**

Python'ın standart `unittest` kütüphanesi, Java'nın JUnit'inden devşirme olduğu için Python'ın ruhuna (Pythonic) pek uymaz. Class'lar oluşturmak, `self.assertEqual` yazmak... Bu hamallık, geliştiriciyi test yazmaktan soğutur. İşte **Pytest** burada devreye girer.

![Pytest vs Unittest Comparison](/assets/img/posts/pytest-unittest-comparison-flowchart.jpg)
*Neden Pytest? Daha az boilerplate, daha okunabilir hata mesajları.*

## 1. Fixture'lar: Test Ortamının İskeleti

Unittest'te `setUp()` ve `tearDown()` metodları içinde kaybolurdunuz. Pytest'te ise **Fixture**'lar var. Bunlar, testlerinizin ihtiyaç duyduğu "bağlamı" (veritabanı bağlantısı, API login token, geçici dosya) sağlayan fonksiyonlardır.

### Scope Yönetimi ve `conftest.py`
En kritik konu Scope yönetimidir. Her test fonksiyonu için veritabanını baştan oluşturursanız testleriniz dakikalar sürer.

`conftest.py` dosyasında tanımlanan fixture'lar tüm projede geçerlidir:

```python
# conftest.py
import pytest
from sqlalchemy import create_engine

# Scope='session': Test süreci boyunca SADECE BİR KEZ çalışır.
@pytest.fixture(scope="session")
def db_engine():
    print("\n--- DB Engine Başlatılıyor ---")
    engine = create_engine("postgresql://user:pass@localhost:5432/testdb")
    yield engine
    print("\n--- DB Engine Kapatılıyor ---")
    engine.dispose()

# Scope='function': Her test fonksiyonu için AYRI çalışır.
@pytest.fixture(scope="function")
def db_session(db_engine):
    connection = db_engine.connect()
    transaction = connection.begin() # İzolasyon için transaction başlat
    session = Session(bind=connection)
    
    yield session # Test burada koşar
    
    # Test bitince Rollback yap -> Veritabanı tertemiz kalır
    session.close()
    transaction.rollback()
    connection.close()
```

![Pytest Fixture Scope Order](/assets/img/posts/pytest-fixtures-scope-order.svg)
*Fixture Hiyerarşisi: Session -> Module -> Class -> Function*

## 2. Mocking: Dış Dünyayı İzole Etmek

Test yazmanın altın kuralı: **Unit Test, internete çıkmaz.** Stripe API'sine gidip gerçekten para çekmeye çalışıyorsa o bir "Integration Test"tir (veya felakettir).

`unittest.mock` güçlüdür ama karmaşıktır. `pytest-mock` eklentisi bu işi çok basitleştirir.

```python
# service.py
def process_payment(amount):
    resp = requests.post("https://api.stripe.com/charge", json={"amount": amount})
    return resp.json()["status"]

# test_service.py
def test_payment_integration(mocker):
    # 'requests.post' çağrısını yakala ve sahte cevap döndür
    mock_post = mocker.patch("service.requests.post")
    mock_post.return_value.json.return_value = {"status": "success"}
    
    result = process_payment(100)
    
    assert result == "success"
    # Çağrının doğru parametrelerle yapıldığını doğrula
    mock_post.assert_called_once_with(
        "https://api.stripe.com/charge", 
        json={"amount": 100}
    )
```

## 3. Parametrize Testler (DDT)

Validation fonksiyonlarını test ederken "Copy-Paste" yapmayın. `pytest.mark.parametrize` kullanın.

```python
@pytest.mark.parametrize("email, expected", [
    ("test@example.com", True),
    ("user+tag@gmail.com", True),
    ("plainaddress", False),       # @ yok
    ("@example.com", False),       # username yok
    ("user@.com.my", False),       # domain yok
])
def test_email_validator(email, expected):
    """Tek fonksiyonla 5 farklı senaryoyu test ediyoruz."""
    assert is_valid_email(email) is expected
```

## 4. Senior Seviye: Property Based Testing (Hypothesis)

Junior developer, aklına gelen 3-5 senaryoyu test eder. Senior developer ise **Hypothesis** kütüphanesini kullanarak sisteme rastgele veri pompalar ve "kırılma noktalarını" bulur.

```python
from hypothesis import given, strategies as st

# Fonksiyona rastgele integer'lar gönderir: -2147483648'den +sonsuza kadar.
# Sizin aklınıza gelmeyecek '0', '-1', 'çok büyük sayı' gibi durumları dener.
@given(st.integers())
def test_add_numbers_property(x):
    # Özellik: Bir sayıya 0 eklenirse sonuç değişmemelidir.
    assert add(x, 0) == x
```

## 5. Production Checklist: Kalite Kontrol

Kodunuzu merge etmeden önce CI/CD pipeline'ında şunlar olmalıdır:

*   [ ] **Coverage (Kapsama) Raporu:** `%80+` coverage hedefleyin. `pytest-cov` eklentisi ile `pytest --cov=src` komutunu çalıştırın.
*   [ ] **Yavaş Testleri Bulma:** `pytest --durations=5` komutu ile sistemdeki en yavaş 5 testi bulun. 50ms süren veritabanı testi normaldir ama 5sn süren testte bir sorun (muhtemelen sleep veya dış ağ isteği) vardır.
*   [ ] **Flaky Testler:** Bazen geçip bazen kalan testler, test olmayanlardan daha tehlikelidir. `@pytest.mark.flaky(reruns=3)` ile geçici çözüm bulun ama mutlaka kök nedeni (Race Condition vb.) çözün.

![CI/CD Pipeline Flow](/assets/img/posts/cicd-deployment-automation-flow.png)

## 6. Terimler Sözlüğü (Glossary)

*   **Fixture:** Testin çalışması için gereken ön hazırlık (Setup) ve temizlik (Teardown) yapıları.
*   **Mock:** Gerçek bir objenin davranışını taklit eden sahte obje.
*   **Spy:** Gerçek objeyi çağıran ama kaç kere çağrıldığını, hangi parametrelerle çağrıldığını kaydeden yapı.
*   **Flaky Test:** Ortama, zamana veya şansa bağlı olarak bazen geçip bazen kalan kararsız test.

## 6. Snapshot Testing: UI ve Büyük Data Testi

Bazen `assert` ile kontrol edemeyeceğiniz kadar büyük çıktılar olur (örn: 50 satırlık bir HTML veya karmaşık bir JSON). Satır satır kontrol etmek hamallıktır.
`pytest-snapshot` eklentisi burada devreye girer.

```python
def test_homepage_html(snapshot):
    html = render_homepage()
    # İlk çalıştırmada bu HTML'i dosyaya kaydeder.
    # Sonraki çalıştırmalarda, gelen HTML ile dosyayı karşılaştırır.
    # Tek bir karakter değişse testi patlatır.
    snapshot.assert_match(html, 'homepage.html')
```

Bu yöntemle, yanlışlıkla yaptığınız küçük UI değişikliklerini hemen yakalarsınız. Özellikle API responseları için harikadır.

## 7. Yapılmaması Gerekenler (Anti-Patterns)

Junior geliştiricilerin sık yaptığı hatalar:

*   **Test içinde Logic Kurmak:** `if x: assert y` gibi mantıklar kurmayın. Test dümdüz (linear) olmalıdır. Eğer if varsa, o testin kendisi de test edilmelidir!
*   **Birbirine Bağlı Testler:** `test_step_1` çalışmadan `test_step_2` çalışmıyorsa tasarımınız yanlıştır. Her test izole olmalı ve rastgele sırada çalışabilmelidir.
*   **Sleep Kullanmak:** `time.sleep(5)` yaziyorsanız o test yanlıştır. Polling veya Event beklemeyi (await) kullanın. Sleep test süresini gereksiz uzatır.

## 8. Mutlaka Kullanılması Gereken Eklentiler

Pytest ekosistemi çok geniştir. İşte benim "olmazsa olmaz" listem:

*   **pytest-xdist:** Testleri paralel (`-n auto`) çalıştırır. CI süresini yarıya indirir.
*   **pytest-sugar:** Terminal çıktısını güzelleştirir, anlık ilerleme çubuğu ekler.
*   **pytest-cov:** Kapsama oranı raporu üretir.
*   **mutmut:** Mutasyon testi yapar (Kodunuzu bilerek bozar ve testlerin bunu yakalayıp yakalamadığını kontrol eder).

## 9. Terimler Sözlüğü (Glossary)

"Test yazmaya vaktim yok" en büyük yalandır. Bug fix yapmak, test yazmaktan 10 kat daha uzun sürer. Pytest size temiz, hızlı ve zevkli bir test ortamı sunar.

Unutmayın: Test edilmemiş kod, bozuk koddur. Sadece henüz ne zaman bozulacağını bilmiyorsunuzdur.
