---
layout: post
title: "Python ile Görüntü İşleme: Pillow vs OpenCV Production Savaşları"
date: 2025-06-28 11:30:00 +0300
categories: [Backend, Python]
description: "Pillow ve OpenCV arasındaki gerçek farklar, Thread-Safety tuzakları, Decompression Bomb saldırıları ve Async işleme mimarisi."
image: assets/img/posts/python-image-manipulation.png
---

Görüntü işleme, hobi projelerinde "eğlenceli", production ortamlarında ise "sunucu katili" (CPU/RAM killer) bir iştir.

Pek çok geliştirici `pip install Pillow` der, gelen resmi `resize()` eder ve iş bitti sanır. Ta ki siteleri "Decompression Bomb" saldırısıyla çökene veya Gunicorn worker'ları kilitlenene kadar.

Bu yazıda kütüphanelerin API'lerini değil, production'da hayatta kalma stratejilerini konuşacağız.

## 1. Pillow (PIL): Ekmek Bıçağı

Pillow, Python'ın de-facto görüntü işleme kütüphanesidir. Amacınız basit manipülasyonlarsa (kırpma, format değiştirme, watermark ekleme) en doğru tercihtir.

Ancak Pillow'un varsayılan ayarları "güvenlik" değil "kullanılabilirlik" odaklıdır.

### Güvenlik Riski: Decompression Bomb (Zip Bomb)
Kötü niyetli bir kullanıcı, 40.000 x 40.000 piksellik simsiyah bir resmi (RAM'de 4GB+ yer kaplar) sıkıştırıp 50KB'lık bir PNG haline getirebilir. Sunucunuz bunu açmaya çalıştığında RAM anında tükenir (OOM Kill).

**Nasıl Korunursunuz?**
Pillow varsayılan olarak bir limit koyar ama bunu bilerek yönetmelisiniz:

```python
from PIL import Image

# Varsayılan limit: 178 MP. Bunu ihtiyacınıza göre ayarlayın (örn: 50 MP)
Image.MAX_IMAGE_PIXELS = 50_000_000

try:
    with Image.open("uploaded_file.jpg") as img:
        img.verify() # Dosya gerçekten resim mi? (Sadece header okur)
        
        # Resmi yeniden açın (verify cursor'ı bozar) ve işleyin
        with Image.open("uploaded_file.jpg") as valid_img:
            valid_img.thumbnail((800, 800))
            valid_img.save("processed.webp", "WEBP", quality=85)
except Image.DecompressionBombError:
    print("Saldırı girişimi engellendi!")
```

## 2. OpenCV: Lazer Neşteri

OpenCV bir Python kütüphanesi değil, C++ ile yazılmış devasa bir framework'ün Python bağlantısıdır (binding). Görüntüleri `numpy` array (matris) olarak tutar.

*   **Hız:** C++ backend'i sayesinde Pillow'dan kat kat hızlıdır (özellikle karmaşık filtrelerde).
*   **Kullanım:** Bilgisayarlı Görüntüleme (Computer Vision), Yüz Tanıma, Hareket Algılama için zorunludur.

**Thread Safety Uyarısı:**
OpenCV'nin bazı fonksiyonları ve GUI bileşenleri thread-safe değildir. Flask veya Django altında çalıştırıyorsanız ve `cv2` objelerini global tutuyorsanız, garip "Segmentation Fault" hataları alabilirsiniz. Her request için yeni obje oluşturmak en güvenlisidir.

## 3. Mimari Kararı: Senkron mu Asenkron mu?

Bir kullanıcının profil fotoğrafı yüklediğini düşünelim.
**Asla** resmi HTTP Request içinde (View/Controller katmanında) işlemeyin.
1.  Resim 5MB olabilir, işlemesi 2 saniye sürebilir.
2.  Python GIL (Global Interpreter Lock) yüzünden o 2 saniye boyunca o worker başka kimseye cevap veremez.
3.  10 kişi aynı anda resim yüklerse siteniz kitlenir.

**Doğrusu:**
1.  Resmi `tmp` veya S3'e kaydedin.
2.  Message Broker'a (RabbitMQ/Redis) bir görev bırakın: `{"path": "s3://bucket/raw/image.jpg"}`
3.  HTTP cevabını hemen dönün: "Resminiz işleniyor."
4.  Arka plandaki Celery worker resmi alır, işler ve günceller.

(Bkz: `[Celery ve ARQ ile Asenkron Görev Kuyruğu](/backend/celery-arq-asenkron-gorev-kuyrugu)`)

## 4. Karşılaştırma: Hangisini Seçmeli?

| Özellik | Pillow (PIL) | OpenCV (cv2) |
| :--- | :--- | :--- |
| **Öğrenme Eğrisi** | Düşük (Pythonic) | Yüksek (Matematiksel) |
| **Performans** | Orta | Çok Yüksek |
| **Veri Yapısı** | `Image` Object | `numpy.ndarray` |
| **En İyi Kullanım** | Web (Resize, Crop, Format) | AI, Analiz, Video İşleme |
| **Dosya Boyutu** | Düşük (Sadece Python kodu) | Yüksek (Binary dependency) |


## 5. Gerçek Hayat Problemi: "iPhone Fotosu Neden Yan Duruyor?"

Bir e-ticaret sitesi yaptınız, kullanıcı mobilden ürün yükledi. Telefondaki galeride düzgün duran fotoğraf, sitenizde 90 derece yan yatmış. Neden?

Çünkü modern telefonlar fotoğrafı çevirmez, sadece EXIF metadatasında "Orientation: 6 (Rotate 90 CW)" etiketi basar. Browserlar bunu okur ve düzeltir ama `Image.open()` saf haliyle okumaz.

**Çözüm: `ImageOps.exif_transpose`**

```python
from PIL import Image, ImageOps

def safe_open(path):
    with Image.open(path) as img:
        # EXIF rotasyonunu fiziksel piksel değişimine çevir
        img = ImageOps.exif_transpose(img)
        img.save("corrected.jpg")
```
Bu tek satır, müşteri hizmetlerine gelen "Fotoğraflarım yamuk çıkıyor" şikayetlerini %100 bitirir.

## 6. Dockerize Ederken Çıkan O Hata: `libGL.so.1`

OpenCV'yi localde kurdunuz, çalıştı. Docker'a attınız ve patladı:
`ImportError: libGL.so.1: cannot open shared object file: No such file or directory`

Çünkü `opencv-python` paketi, arka planda OpenGL kütüphanelerine ihtiyaç duyar ve `python:3.9-slim` gibi minimal imajlarda bunlar yoktur.

**Çözüm 1 (Kötü):** Dev fat imajlar kullanmak.
**Çözüm 2 (İyi):** `opencv-python-headless` kullanmak.

Eğer sunucuda GUI (pencere) açmayacaksanız (ki sunucuda ekran yoktur), `requirements.txt` dosyanıza normal opencv yerine şunu yazın:

```text
opencv-python-headless==4.8.0.74
```
Bu paket GUI bağımlılıklarından arındırılmıştır, boyutu daha küçüktür ve Docker'da sorunsuz çalışır.

## 7. Görüntü İşleme Pipeline Mimarisi

Sadece kod yazmak yetmez, sistemi tasarlamalısınız. İşte ölçeklenebilir bir mimari örneği:

1.  **Ingestion:** API Gateway dosyayı alır, sadece header kontrolü yapar (Magic Bytes).
2.  **Storage:** Dosya ham haliyle (`raw/`) S3'e yüklenir.
3.  **Queue:** S3'e yükleme event'i (S3 Event Notifications veya manuel publish) kuyruğa düşer.
4.  **Worker:** GPU destekli worker (OpenCV/CUDA) veya CPU worker (Pillow) resmi kuyruktan alır.
5.  **Processing:** Resize, Watermark, EXIF Rotate, NSFW Check işlemlerini yapar.
6.  **Final Storage:** İşlenmiş dosyayı (`processed/`) S3'e yazar ve veritabanını günceller.
7.  **CDN:** Kullanıcı sadece `processed/` klasörünü CloudFront üzerinden görür.

Bu yapıda API sunucunuz (Django/FastAPI) asla resim işleme yükü altına girmez. 10MB resim yüklemek CPU'yu %1 bile etkilemez.


## 8. WebP Devrimi: Trafiği Yarıya İndirin

Yıl 2025. Artık JPEG veya PNG kullanmak için çok geçerli bir sebebiniz olmalı.
WebP formatı, JPEG kalitesini %30 daha düşük dosya boyutunda sunar. Pillow ile dönüşüm çok basittir:

```python
img.save("image.webp", "WEBP", quality=80, optimize=True)
```
Trafik maliyetinizi düşürün, SEO puanınızı (Core Web Vitals) artırın. Eğer "Eski browserlar desteklemiyor" diye endişeleniyorsanız, endişelenmeyin. WebP desteği %97 oranında.

## 9. Bonus: "Bu Resmi Daha Önce Görmüştüm" (Perceptual Hashing)

Milyonlarca resim olan bir sistemde, kullanıcılar aynı "kedi fotoğrafını" farklı isimlerle (kedi.jpg, my_cat.png) yükleyebilir. MD5 hash işe yaramaz çünkü bir piksel değişse MD5 değişir.

Burada **pHash (Perceptual Hash)** devreye girer. Resmin "görsel parmak izini" çıkarır. Resim küçültülse de, siyah beyaz yapılsa da parmak izi aynı kalır.

```python
import imagehash
from PIL import Image

hash1 = imagehash.phash(Image.open('kedi.jpg'))
hash2 = imagehash.phash(Image.open('kedi_resize.png'))

if hash1 - hash2 < 5: # Hamming Distance
    print("Bunlar aynı resim!")
```
Bu yöntemle veritabanı şişkinliğini %20 oranında azaltabilirsiniz.

## 10. Son Kontrol: Production Checklist

Canlıya çıkmadan önce şunları kontrol edin:

*   [ ] **Limitler Koyuldu mu?** `MAX_IMAGE_PIXELS` ayarlı mı? Dosya boyutu (Upload Size Limit) Nginx ve Code tarafında sınırlı mı?
*   [ ] **Format Temizliği:** Kullanıcı `.php` dosyasını `.jpg` olarak yüklemeye çalışabilir. Sadece uzantıya değil, Magic Byte'lara bakın (`python-magic`).
*   [ ] **Metadata Temizliği:** EXIF verisini (GPS koordinatları!) siliyor musunuz? KVKK/GDPR için bu zorunludur.
*   [ ] **Asenkron İşleme:** Resim küçültme işlemi API thread'ini kilitliyor mu? (Cevap 'Evet' ise Celery'ye taşıyın).
*   [ ] **Storage:** İşlenmiş dosyalar S3'te public, ham dosyalar private bucket'ta mı?

## Özetle

*   Basit web işleri için **Pillow** kullanın ama `MAX_IMAGE_PIXELS` ayarını unutmayın.
*   Analiz ve hız için **OpenCV** kullanın ama bellek yönetimine dikkat edin.
*   Her iki durumda da işlemi **Asenkron Kuyruklara (Celery)** devredin.

Kullanıcının 10MB'lık fotoğraf yükleyip sunucuyu kilitlemesi, kullanıcının suçu değil, sizin mimari hatanızdır.

