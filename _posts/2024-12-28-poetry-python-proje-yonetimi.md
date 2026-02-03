---
title: "Python Proje Yönetimi: Poetry, uv ve Modern Standartlar"
description: "requirements.txt devri kapandı. Python 2025 ekosisteminde deterministik paket yönetimi, 'uv' hızı ve lock dosyalarının hayati önemi."
date: "2024-12-28 14:00:00 +0300"
categories: [Python, DevOps, Productivity]
tags: [poetry, uv, python, package-management, dependency-hell]
image:
  path: /assets/img/posts/poetry-dependency-management-workflow.png
  alt: "Dependency Resolution Graph"
---

Python ekosisteminin en büyük günahı "Dependency Hell" (Bağımlılık Cehennemi) idi.
Bir proje `requests==2.25` ister, diğeri `requests==2.28` ister ve global `pip` kurulumunuz patlardı.
Sanal ortamlar (Virtual Environments) bu sorunu bir nebze çözdü ancak `requirements.txt` dosyası yetersiz kaldı.
`requirements.txt`, sadece **doğrudan** bağımlılıklarınızı listelerdi, alt bağımlılıkları (transitive dependencies) değil.
Bir paket güncellendiğinde tüm zincirin kırılma riski vardı.
Sonuç? "Benim makinemde çalışıyor, sunucuda çalışmıyor."
Bu cümle bir Senior Developer için kabul edilemez. Bir sistem mühendisi olarak kariyerimde duyduğum en sinir bozucu bahane budur.
Production ortamı sürprizleri sevmez. Production ortamı sıkıcı olmalıdır. Deterministik olmalıdır. 
Aynı kodu 100 kere deploy etsem, 100'ünde de aynı sonucu almalıyım. Eğer bir deploy çalışıp ikincisi hata veriyorsa, orada mühendislik değil şans faktörü vardır.
Çözüm: **Deterministik Build** ve **Lock Dosyaları**.

![Dependency Resolution](/assets/img/posts/poetry-dependency-management-workflow.png)
*Karmaşık bağımlılık ağacının çözümlenmesi ve versiyon çakışmaları.*

## 1. Neden requirements.txt Yetersiz?

Eski usul `pip freeze > requirements.txt` yaptığınızda, o anki ortamda ne varsa dosyaya yazılır.
Her şeyi `pip install` ile kurdunuz, sanal ortamınızda 50 tane paket var. Hepsini döktünüz.
Ama bu dosya, paketlerin *neden* orada olduğunu söylemez.
`urllib3` paketini siz mi kurdunuz, yoksa `requests` mi getirdi?
`boto3`, `jmespath`'e ihtiyaç duyduğu için mi orada, yoksa siz mi eklediniz?
Gereksiz paketleri temizlemek istediğinizde, "Acaba bunu silersem ne bozulur?" korkusuyla hiçbir şeye dokunamazsınız.

**Poetry** (veya modern alternatifler) iki dosya kullanır:
1.  `pyproject.toml`: Sizin **analitik** istekleriniz. ("Bana FastAPI 0.100 ve üzeri ver"). Burası projenin kalbidir.
2.  `poetry.lock`: O anın **deterministik** fotoğrafı. ("FastAPI 0.109.1, Starlette 0.36.0, Pydantic 2.6.1 kurulacak").

Build sunucusu `poetry install` dediğinde, `lock` dosyasına bakar ve **byte-byte aynı** paketleri kurar. Sürpriz yaşamazsınız.
Kilit dosyası (Lock file), projenizin seyahat sigortasıdır. Onsuz yola çıkılmaz.
Her commit'te bu dosyayı da git'e atmalısınız. `.gitignore`'a eklemek büyük hatadır. Çoğu acemi geliştirici "lock dosyası sürekli conflict yaratıyor" diye onu ignore eder.
Conflict çıkıyorsa, takım içinde iletişim sorunu vardır. Biri paket güncellerken diğeri de güncellediyse bunu konuşmalısınız.

## 2. 2025'in Yıldızı: uv (Rust Tabanlı Hız)

Yıllardır Poetry kullanıyoruz ve seviyoruz. Geliştirici dostu CLI'ı, sanal ortam yönetimi harika.
Ancak Python ile yazıldığı için büyük projelerde "Dependency Resolution" (Bağımlılık Çözümleme) dakikalar sürebiliyor.
Özellikle numpy, pandas, tensorflow gibi C uzantılı dev paketlerle uğraşıyorsanız, Poetry'nin çözücü algoritması CPU'nuzu ısıtabilir.
Astral (Ruff'ın yapımcıları) ekibi, **uv** ile sahneye çıktı.
Rust ile yazılmış, modern, aşırı hızlı bir paket yöneticisi.

Benchmark sonuçları şok edici: Poetry 45sn sürerken, **uv sadece 1.5 saniyede** işlemi bitiriyor. Aradaki fark saniyeler değil, büyüklük mertebesi.


Eğer CI/CD pipeline'ınızda her build için paket kuruyorsanız, `uv` kullanmak deploy sürenizi dakikalarca kısaltabilir.
`uv`, Poetry projelerini (pyproject.toml) direkt destekler. Geçiş yapmak için projenizi yeniden yazmanıza gerek yok.
`uv pip install -r pyproject.toml` komutu, Poetry standartlarını tanır ve ışık hızında kurulum yapar.

Benim tavsiyem: Geliştirme (lokal) ortamında Poetry'nin konforunu kullanın. `poetry add`, `poetry shell` komutları çok rahattır.
CI/CD ortamında (Dockerfile, GitLab Runner) `uv`'nin hızını kullanın. Hibrit bir yapı kurabilirsiniz.
Dockerfile içinde: `RUN pip install uv && uv pip install -r pyproject.toml --system` komutu, imaj build süresini %80 azaltır.

![Poetry vs uv Benchmark](/assets/img/posts/python-packaging-tools-comparison.png)
*Paket yükleme ve çözümleme sürelerinde Rust (uv) vs Python (Poetry) farkı.*

## 3. Dependency Groups: Dev vs Prod

Production imajına `pytest`, `black`, `mypy`, `isort`, `ruff` gibi araçları kurmak güvenlik riskidir ve imajı şişirir.
Neden production sunucusunda test kütüphanesi olsun?
Docker imajınız ne kadar küçükse, atak vektörünüz o kadar azalır ve dağıtım o kadar hızlanır.
Ayrıca, dev paketleri bazen production'da gereksiz çakışmalara yol açabilir.
Poetry ile bağımlılıkları gruplayın:

```toml
[tool.poetry.dependencies]
python = "^3.11"
fastapi = "^0.109.0"
uvicorn = "^0.27.0"
sqlalchemy = "^2.0.0"
pydantic = "^2.5.0"

[tool.poetry.group.dev.dependencies]
pytest = "^8.0.0"
black = "^24.1.0"
mypy = "^1.8.0"
httpx = "^0.26.0" # Test client için
faker = "^24.0.0" # Test datası üretmek için
```

Deploy yaparken (Dockerfile içinde):
```bash
# Sadece ana bağımlılıkları kur, dev grubunu atla
poetry install --without dev --sync --no-interaction --no-ansi
```
`--sync` parametresi çok önemlidir. Bu parametre, "Sadece benim dediğim paketler olacak, fazlasını sil" demektir.
Eğer `poetry.lock` dosyasında olmayan bir paket sanal ortamda varsa, onu **siler**.
Bu da ortamınızın tam olarak lock dosyasıyla eşleşmesini sağlar. Kirli (dirty) ortamları temizler.
Özellikle lokal geliştirmede, bir kütüphaneyi deneyip sildiğinizde, sanal ortamda kalıntısı kalabilir. `--sync` bunu temizler.


## 4. Semantik Versiyonlama: Şapka (^) mı Tilde (~) mi?

`pyproject.toml` dosyasında gördüğünüz o garip işaretler, projenizin kaderini belirler.
*   **Caret (^):** Varsayılandır. "Major sürüm değişmediği sürece güncelle" demektir.
    *   `^1.2.3` -> `>=1.2.3 <2.0.0` (1.9.9'a günceller ama 2.0.0'a geçmez).
    *   **Risk:** Eğer kütüphane geliştiricisi semantik versiyonlamaya (SemVer) uymuyorsa, minor güncellemede bile kodunuz kırılabilir.
*   **Tilde (~):** Daha muhafazakardır. "Sadece son haneyi (Patch) güncelle" demektir.
    *   `~1.2.3` -> `>=1.2.3 <1.3.0` (1.2.9'a günceller, 1.3.0'a geçmez).
    *   **Kullanım:** Çok kritik ve sık bozulan kütüphaneler için (örn: pandas, tensorflow) `~` kullanmak daha güvenlidir.
*   **Pinning (==):** Asla güncellenmez.
    *   `1.2.3` -> Sadece 1.2.3.
    *   **Kullanım:** Sadece geçici workaround'lar için kullanın. Sürekli pinli kalmak teknoloji borcudur.

![Semantic Versioning](/assets/img/posts/poetry-workflow-diagram.png)
*Major.Minor.Patch (X.Y.Z) versiyonlama mantığı ve Tilde/Caret operatörleri.*

## 5. Production İçin Multistage Docker Build

Poetry'yi Docker içinde kullanırken "imaj boyutu" en büyük düşmanımızdır. Poetry'nin kendisi ve cache dosyaları production imajında olmamalıdır.
İşte 1GB'lık imajı 100MB'a düşüren **Multistage Build** tarifi:

```dockerfile
# Stage 1: Builder
FROM python:3.11-slim as builder
WORKDIR /app
RUN pip install poetry
COPY pyproject.toml poetry.lock ./
# Sanal ortam oluşturmadan, direkt requirements.txt üret
RUN poetry export -f requirements.txt --output requirements.txt --without-hashes

# Stage 2: Runner
FROM python:3.11-slim
WORKDIR /app
COPY --from=builder /app/requirements.txt .
# Sadece gerekli paketleri kur, Poetry yok, cache yok
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["python", "main.py"]
```
Bu yöntemle hem Poetry'nin deterministik gücünü kullanırsınız, hem de `requirements.txt` sadeliğinde hafif imajlar üretirsiniz.
Eğer projede C derlemesi gerektiren kütüphaneler varsa, Builder aşamasına `build-essential` ekleyip, Runner aşamasını temiz tutabilirsiniz.

## 6. İleri Seviye Konfigürasyon: pyproject.toml Gücü

Poetry sadece bir paket yükleyici değil, bir proje yöneticisidir. `pyproject.toml` dosyasında neler yapabileceğinize şaşıracaksınız.

**Scripts (Entry Points):**
`python main.py` yazmaktan sıkıldıysanız, projenize bir alias (takma ad) atayabilirsiniz.
```toml
[tool.poetry.scripts]
start = "myapp.main:start_server"
test = "pytest tests/"
```
Artık terminalde sadece `poetry run start` yazmanız yeterli. Bu aynı zamanda paketinizi `pip install` ile kuran kullanıcıların da `start` komutunu kullanabilmesini sağlar.

**Extras (İsteğe Bağlı Bağımlılıklar):**
Kütüphanenizin bir "Core" kısmı bir de "Database" kısmı olabilir. Herkes database sürücülerini kurmak zorunda değil.
```toml
[tool.poetry.dependencies]
psycopg2 = {version = "^2.9", optional = true}

[tool.poetry.extras]
db = ["psycopg2"]
```
Kullanıcı `poetry install --extras "db"` dediğinde `psycopg2` kurulur. Bu, modüler mimariler için harikadır.

**Private Sources (Özel Repolar):**
Şirket içi Nexus veya GitLab PyPI sunucunuz varsa, bunu da burada tanımlarsınız.
Kullanıcıların ekstra `--index-url` parametresi girmesine gerek kalmaz. Her şey dosyanın içindedir.

## 7. Kütüphane Yayınlamak (Publishing)

Eğer açık kaynak bir kütüphane veya şirket içi (private) bir SDK yazıyorsanız, Poetry işi çok kolaylaştırır.
Eskiden `setup.py`, `MANIFEST.in`, `setup.cfg` gibi dosyalarla uğraşırdık. Versiyonu 3 farklı yerde manuel güncellerdik.
`long_description` için README dosyasını okuyan python kodları yazardık.

Poetry ile tek yapmanız gereken:

1.  `poetry version patch`: Versiyonu otomatik artırır (0.1.0 -> 0.1.1).
2.  `poetry build`: `.whl` (Wheel) ve `.tar.gz` (Source) dosyalarını oluşturur. `dist/` klasörüne koyar.
3.  `poetry config pypi-token.pypi <TOKEN>`: PyPI token'ını ekle (Sadece bir kere).
4.  `poetry publish`: Dünyaya açıl.

Private repo için `poetry config repositories.gitlab <URL>` ve `poetry publish -r gitlab` komutları yeterlidir.
Takım içi "utils" kütüphanelerini bu şekilde dağıtmak, Git submodule veya Copy-Paste yapmaktan çok daha profesyonel ve temizdir.
Kütüphanenizi kullanan kişiler `poetry add my-company-utils` diyerek en son (ve test edilmiş) sürüme geçerler.

![CI/CD Pipeline](/assets/img/posts/github-actions-cicd-pipeline-diagram.png)
*GitLab CI üzerinde Poetry ile test, build ve publish adımları.*

## Sonuç

Python proje yönetimi artık "bir sanal ortam kur, pip install yap"tan ibaret değil.
Projenizin yeniden üretilebilirliği (reproducibility) kalitesinin en büyük göstergesidir.
Eğer projenizi 6 ay sonra clone'layıp `poetry install` dediğinizde, "Paket bulunamadı" veya "ImportError" almadan çalışıyorsa, iyi bir iş çıkarmışsınız demektir.
Bu disiplini kazanmak, sizi Junior seviyesinden Senior seviyesine taşıyan önemli adımlardan biridir.
Hız istiyorsanız `uv`'ye şans verin, stabilite ve geniş ekosistem entegrasyonu istiyorsanız `Poetry` ile devam edin.
Ama ne yaparsanız yapın, lütfen `requirements.txt` dosyasını tarihin tozlu raflarına kaldırın.
Modern Python, modern araçlar gerektirir. 2025 yılında manuel paket yönetimi yapmak, at arabasıyla otobana çıkmak gibidir.
Kendi konforunuz ve ekibinizin ruh sağlığı için, bugünden itibaren `poetry init` ile başlayın.
