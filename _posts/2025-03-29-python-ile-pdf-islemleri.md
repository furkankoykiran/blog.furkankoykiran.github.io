---
layout: post
title: "Python ile Kurumsal PDF Mimarisi: HTML'den PDF'e, OCR ve E-İmza"
date: 2025-03-29 13:00:00 +0300
categories: [Backend, Python, Automation]
description: "Piksel piksel koordinat hesaplamayı bırakın. WeasyPrint ile raporlama, pyHanko ile E-İmza ve Tesseract ile OCR süreçlerini derinlemesine inceliyoruz."
image: assets/img/posts/python-pdf-reportlab-generation.png
---

Bir backend geliştiricisiyseniz, kariyerinizin bir noktasında mutlaka o cümleyi duyarsınız: **"Bu raporu PDF olarak indirebilir miyiz?"**

İlk başta kolay görünür. "Bir kütüphane kurarım, hallederim" dersiniz. Sonra kendinizi ReportLab ile koordinat hesaplarken veya Docker'da font family tanıtmaya çalışırken bulursunuz.

Bu yazıda, amatör PDF scriptlerini değil, günde milyonlarca fatura ve sözleşme üreten **Enterprise PDF Pipeline** mimarisini konuşacağız.

## 1. Tasarımın Kurtarıcısı: WeasyPrint

Eskiden `ReportLab` ile "canvas.drawLine(100, 200)" diye kod yazardık. Bir satır kaysa, tüm sayfa düzeni bozulurdu.
Modern dünyada standart bellidir: **HTML + CSS = PDF.**

**WeasyPrint**, web tarayıcısı gibi davranır. Jinja2 ile dinamik HTML üretirsiniz, WeasyPrint bunu vektörel PDF'e çevirir.

```python
from weasyprint import HTML, CSS
from jinja2 import Environment, FileSystemLoader

# Jinja2 ortamı
env = Environment(loader=FileSystemLoader('.'))
template = env.get_template('invoice.html')

# Veriyi HTML'e bas
html_out = template.render(
    company="Tech Corp",
    total=5000,
    items=[{"name": "Server", "price": 4000}, {"name": "SLA", "price": 1000}]
)

# PDF'e çevir
HTML(string=html_out).write_pdf("invoice.pdf", stylesheets=[CSS('style.css')])
```
**Avantajı:** Fron-end geliştiricinize "bana fatura için HTML/CSS ver" diyebilirsiniz. (ReportLab kodunu kimse yazmak istemez).

## 2. Docker ve Font Kabusu

Kodunuz localde (Mac/Windows) harika çalışır. Docker'a (Linux Alpine/Slim) attığınızda iki sorunla karşılaşırsınız:
1.  **Dependency Hell:** WeasyPrint, `Pango` ve `Cairo` kütüphanelerine muhtaçtır.
2.  **Kutu Kutu Fontlar:** Sunucuda `Arial` veya `Times New Roman` yoktur. Türkçe karakterler kare (□) çıkar.

**Doğru Dockerfile:**
```dockerfile
FROM python:3.11-slim

# 1. Sistem bağımlılıkları (Debian için)
RUN apt-get update && apt-get install -y \
    libpango-1.0-0 \
    libpangoft2-1.0-0 \
    libharfbuzz-subset0 \
    libjpeg-dev \
    libopenjp2-7-dev \
    libxcb1 \
    fontconfig

# 2. Fontları kopyala ve cache'i yenile
COPY ./fonts /usr/share/fonts/truetype/custom
RUN fc-cache -f -v
```
Fontları projenizin içine koyun (`assets/fonts/`), sistem fontlarına güvenmeyin.

## 3. Dijital İmza (E-İmza) ve pyHanko

Kurumsal PDF'lerin hukuki geçerliliği olması için **imzalanması** gerekir.
Ekrana JPEG imza yapıştırmak **imza değildir**. Kriptografik imza (PAdES) gerekir. Bunun için `pyHanko` kullanıyoruz.

```python
from pyhanko.sign import signers, fields
from pyhanko.pdf_utils.incremental_writer import IncrementalPdfFileWriter

# PKCS#12 (.p12) sertifikası ile imzalama
signer = signers.P12Signer(
    pfx_file='sirket_sertifikasi.p12',
    passphrase=b'gizli_sifre'
)

with open('sozlesme.pdf', 'rb') as inf:
    w = IncrementalPdfFileWriter(inf)
    fields.append_signature_field(
        w, sig_field_spec=fields.SigFieldSpec(sig_field_name='Signature1')
    )
    
    with open('sozlesme_imzali.pdf', 'wb') as outf:
        signers.sign_pdf(
            w, signers.PdfSignatureMetadata(field_name='Signature1'),
            signer=signer, output=outf,
        )
```
Bu işlem sonrası Adobe Reader'da "Signed and all signatures are valid" yeşil tikini görürsünüz. Hukuki olarak bağlayıcıdır.

## 4. PDF/A: Geleceğe Mektup

Müşteriniz "Faturayı 10 yıl saklamamız lazım" diyorsa, standart PDF yetmez. **PDF/A (Archival)** standardı gerekir.
PDF/A, dış kaynaklara (internet üzerindeki fontlar, görseller) referans vermeyi yasaklar. Her şey dosyanın içine gömülmelidir (Embedded).

WeasyPrint bunu destekler:
```python
HTML(string=html).write_pdf(
    "archive.pdf",
    font_config=font_config,
    pdf_variant="pdf/a-3b" # ISO standardı
)
```

## 5. OCR: Taranmış Belgeleri Okumak

Kullanıcı sisteme fotoğrafı çekilmiş bir kimlik veya fatura yüklediğinde, içindeki metni nasıl alırsınız?
Burada **Tesseract** devreye girer. Python wrapper'ı `pytesseract`'tir.

**Dikkat:** Tesseract sihirbaz değildir. Görüntü işleme (Preprocessing) şarttır.
1.  Grayscale'e çevir.
2.  Threshold (Binarization) uygula.
3.  Noise Reduction yap.
(Bkz: Bir önceki `Pillow ve OpenCV` yazımız).

## 6. Performans: Asla Senkron Yapmayın!

PDF üretmek CPU intensive (işlemciyi sömüren) bir iştir. 10 sayfalık bir raporu render etmek 2-3 saniye sürebilir.
Bu işlemi HTTP Request/Response döngüsü içinde yaparsanız sunucunuz kilitlenir.

**Mimari:**
1.  Kullanıcı "Rapor Al" butonuna basar.
2.  Backend `celery_task.delay(user_id)` der ve hemen "Raporunuz hazırlanıyor, bitince bildireceğiz" döner.
3.  Celery worker PDF'i üretir, S3'e yükler.
4.  User'a WebSocket veya Email ile indirme linki gider.


## 7. Şablon Yönetimi: Jinja2 Inheritance

Binlerce çeşit raporunuz olabilir (Fatura, İrsaliye, Teklif). Hepsinin header'ı, footer'ı, fontu aynıdır. Her HTML'e bunları kopyalamak ameleliktir.

Jinja2'nin `extends` özelliği hayat kurtarır:

**layout.html:**
```html
{% raw %}
<!DOCTYPE html>
<html>
<head>
    <style>
        @page { size: A4; margin: 2cm; @bottom-center { content: counter(page); } }
        body { font-family: 'Roboto'; }
    </style>
</head>
<body>
    <header> <img src="logo.png"> </header>
    <main> {% block content %}{% endblock %} </main>
    <footer> <p>Mersis No: 123456</p> </footer>
</body>
</html>
{% endraw %}
```

**fatura.html:**
```html
{% raw %}
{% extends "layout.html" %}

{% block content %}
    <h1>Fatura Detayı</h1>
    <!-- Sadece içeriğe odaklan -->
    <p>Toplam: {{ total }} TL</p>
{% endblock %}
{% endraw %}
```
Tasarımcı "Logo değişti" dediğinde 50 dosyayı değil, tek dosyayı değiştirirsiniz.

## 8. Güvenlik: Şifreleme ve Watermark

Bazen PDF'in açılması için şifre istersiniz (Örn: E-Ekstre). WeasyPrint bunu yerleşik yapmaz, `pypdf` ile post-process yaparız.

```python
from pypdf import PdfReader, PdfWriter

reader = PdfReader("fatura.pdf")
writer = PdfWriter()

for page in reader.pages:
    writer.add_page(page)

# AES-128 şifreleme
writer.encrypt("kullanici_sifresi_123", algorithm="AES-256")

with open("fatura_sifreli.pdf", "wb") as f:
    writer.write(f)
```
Bu sayede dosya sızsa bile, şifreyi bilmeyen (TCKN veya GSM son 4 hane gibi) içeriği göremez.

## 9. Hukuki Arşivleme: PDF/A-3 ve ZUGFeRD

Sadece görüntüyü (PDF) değil, veriyi (XML) de dosyanın içine gömmek ister misiniz? 
Almanya ve Avrupa'da standart olan **ZUGFeRD**, faturanın fiziksel görüntüsünün içine, makinenin okuyabileceği XML'i ekler. PDF/A-3 standardı buna izin verir.

Böylece muhasebeci PDF'e bakar, muhasebe yazılımı ise içindeki XML'i parse eder. OCR'a gerek kalmaz.

