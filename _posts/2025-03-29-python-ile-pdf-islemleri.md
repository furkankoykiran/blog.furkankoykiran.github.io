---
title: "Python ile PDF İşlemleri: Oluşturma, Düzenleme ve Otomasyon"
date: 2025-03-29 10:00:00 +0300
categories: [Python, PDF Processing]
tags: [python, pdf, pypdf2, reportlab, pdfplumber, pdf-generation, pdf-extraction, automation, document-processing, fpdf]
image:
  path: /assets/img/posts/python-pdf-reportlab-generation.png
  alt: "Python PDF Generation with ReportLab"
---

## Giriş

PDF (Portable Document Format), belge paylaşımında evrensel standart haline gelmiştir. Python ekosistemi, PDF'lerle çalışmak için güçlü kütüphaneler sunar: oluşturma, okuma, düzenleme, birleştirme ve çok daha fazlası.

Bu rehberde, Python ile PDF işlemlerinin tüm yönlerini öğreneceksiniz: Sıfırdan PDF oluşturma, mevcut PDF'leri okuma ve düzenleme, form doldurma, watermark ekleme ve daha fazlası.

## PDF Kütüphaneleri Genel Bakış

### Popüler Kütüphaneler

| Kütüphane | Amaç | Güçlü Yönleri |
|-----------|------|---------------|
| **ReportLab** | PDF oluşturma | Profesyonel raporlar, grafikler, tablolar |
| **PyPDF2** | PDF manipülasyonu | Birleştirme, bölme, sayfa çıkarma |
| **pdfplumber** | PDF okuma | Tablo ve metin çıkarma |
| **pypdf** | PDF işleme | PyPDF2'nin modern fork'u |
| **FPDF** | Basit PDF oluşturma | Hızlı, kolay kullanım |
| **PDFMiner** | Metin çıkarma | Gelişmiş metin analizi |
| **borb** | Full-featured | Okuma ve yazma |

### Kurulum

```bash
# PDF oluşturma
pip install reportlab fpdf2

# PDF okuma ve manipülasyon
pip install pypdf PyPDF2 pdfplumber

# Metin çıkarma
pip install pdfminer.six pdfplumber

# Form işleme
pip install pypdf pdfrw

# Gelişmiş işlemler
pip install pikepdf borb

# Tüm araçlar
pip install reportlab pypdf pdfplumber pikepdf Pillow
```

## PyPDF2/pypdf ile PDF Okuma

### Temel PDF Okuma

```python
from pypdf import PdfReader

def read_pdf(file_path):
    """PDF dosyasını oku"""
    # PDF'i aç
    reader = PdfReader(file_path)
    
    # Metadata
    metadata = reader.metadata
    print(f"Başlık: {metadata.title}")
    print(f"Yazar: {metadata.author}")
    print(f"Oluşturma tarihi: {metadata.creation_date}")
    print(f"Sayfa sayısı: {len(reader.pages)}")
    
    # Tüm sayfaları oku
    for i, page in enumerate(reader.pages):
        text = page.extract_text()
        print(f"\n--- Sayfa {i+1} ---")
        print(text)
    
    return reader

# Kullanım
reader = read_pdf("document.pdf")
```

### Belirli Sayfaları Okuma

```python
from pypdf import PdfReader

def extract_specific_pages(file_path, page_numbers):
    """Belirli sayfaları çıkar"""
    reader = PdfReader(file_path)
    
    extracted_text = {}
    
    for page_num in page_numbers:
        if page_num < len(reader.pages):
            page = reader.pages[page_num]
            text = page.extract_text()
            extracted_text[page_num] = text
        else:
            print(f"Sayfa {page_num} bulunamadı")
    
    return extracted_text

# İlk 3 sayfayı çıkar
texts = extract_specific_pages("document.pdf", [0, 1, 2])

for page_num, text in texts.items():
    print(f"\n=== Sayfa {page_num + 1} ===")
    print(text[:500])  # İlk 500 karakter
```

![PDF Text Extraction with Python](/assets/img/posts/pdf-text-extraction-python.png)

### PDF Şifreleme ve Şifre Çözme

```python
from pypdf import PdfReader, PdfWriter

def encrypt_pdf(input_path, output_path, password):
    """PDF'i şifrele"""
    reader = PdfReader(input_path)
    writer = PdfWriter()
    
    # Tüm sayfaları kopyala
    for page in reader.pages:
        writer.add_page(page)
    
    # Şifrele
    writer.encrypt(user_password=password, owner_password=password)
    
    # Kaydet
    with open(output_path, "wb") as output_file:
        writer.write(output_file)
    
    print(f"PDF şifrelendi: {output_path}")

def decrypt_pdf(input_path, output_path, password):
    """Şifreli PDF'i oku"""
    reader = PdfReader(input_path)
    
    # Şifre kontrolü
    if reader.is_encrypted:
        reader.decrypt(password)
    
    writer = PdfWriter()
    
    for page in reader.pages:
        writer.add_page(page)
    
    with open(output_path, "wb") as output_file:
        writer.write(output_file)
    
    print(f"PDF şifresi çözüldü: {output_path}")

# Kullanım
encrypt_pdf("document.pdf", "encrypted.pdf", "mypassword123")
decrypt_pdf("encrypted.pdf", "decrypted.pdf", "mypassword123")
```

## PDF Birleştirme ve Bölme

### PDF Birleştirme

```python
from pypdf import PdfMerger, PdfReader
import os

def merge_pdfs(pdf_list, output_path):
    """Birden fazla PDF'i birleştir"""
    merger = PdfMerger()
    
    for pdf in pdf_list:
        if os.path.exists(pdf):
            print(f"Ekleniyor: {pdf}")
            merger.append(pdf)
        else:
            print(f"Dosya bulunamadı: {pdf}")
    
    # Kaydet
    merger.write(output_path)
    merger.close()
    
    print(f"Birleştirilmiş PDF: {output_path}")

# Kullanım
pdf_files = ["doc1.pdf", "doc2.pdf", "doc3.pdf"]
merge_pdfs(pdf_files, "combined.pdf")
```

### Gelişmiş Birleştirme

```python
from pypdf import PdfMerger

def advanced_merge(merge_config, output_path):
    """
    Gelişmiş birleştirme - sayfa aralıkları ve bookmark'lar ile
    
    merge_config = [
        {"file": "doc1.pdf", "pages": (0, 5), "bookmark": "Bölüm 1"},
        {"file": "doc2.pdf", "pages": (2, 10), "bookmark": "Bölüm 2"},
        {"file": "doc3.pdf", "pages": None, "bookmark": "Bölüm 3"},
    ]
    """
    merger = PdfMerger()
    
    for config in merge_config:
        file_path = config["file"]
        pages = config.get("pages")
        bookmark = config.get("bookmark")
        
        if pages:
            # Belirli sayfa aralığı
            merger.append(file_path, pages=pages, import_outline=False)
        else:
            # Tüm sayfalar
            merger.append(file_path)
        
        # Bookmark ekle
        if bookmark:
            merger.add_outline_item(bookmark, len(merger.pages) - 1)
    
    merger.write(output_path)
    merger.close()
    
    print(f"Gelişmiş birleştirme tamamlandı: {output_path}")

# Kullanım
config = [
    {"file": "intro.pdf", "pages": None, "bookmark": "Giriş"},
    {"file": "chapter1.pdf", "pages": (0, 10), "bookmark": "Bölüm 1"},
    {"file": "chapter2.pdf", "pages": (5, 20), "bookmark": "Bölüm 2"},
]

advanced_merge(config, "book.pdf")
```

### PDF Bölme

```python
from pypdf import PdfReader, PdfWriter

def split_pdf(input_path, output_folder):
    """PDF'i sayfa sayfa ayır"""
    reader = PdfReader(input_path)
    
    os.makedirs(output_folder, exist_ok=True)
    
    for i, page in enumerate(reader.pages):
        writer = PdfWriter()
        writer.add_page(page)
        
        output_path = os.path.join(output_folder, f"page_{i+1}.pdf")
        
        with open(output_path, "wb") as output_file:
            writer.write(output_file)
        
        print(f"Oluşturuldu: {output_path}")

def split_by_range(input_path, output_path, start_page, end_page):
    """Belirli sayfa aralığını ayır"""
    reader = PdfReader(input_path)
    writer = PdfWriter()
    
    # Sayfa aralığını kontrol et
    start = max(0, start_page)
    end = min(len(reader.pages), end_page)
    
    for i in range(start, end):
        writer.add_page(reader.pages[i])
    
    with open(output_path, "wb") as output_file:
        writer.write(output_file)
    
    print(f"Sayfa {start+1}-{end} çıkarıldı: {output_path}")

# Kullanım
split_pdf("document.pdf", "pages/")
split_by_range("document.pdf", "extract.pdf", 5, 15)
```

![PDF Merge Split Watermark Operations](/assets/img/posts/pdf-merge-split-watermark.png)

## Watermark ve Overlay

### Watermark Ekleme

```python
from pypdf import PdfReader, PdfWriter
from reportlab.pdfgen import canvas
from reportlab.lib.pagesizes import letter
import io

def create_watermark(text, output_path=None):
    """Watermark PDF oluştur"""
    packet = io.BytesIO()
    can = canvas.Canvas(packet, pagesize=letter)
    
    # Watermark metni
    can.setFont("Helvetica", 60)
    can.setFillColorRGB(0.5, 0.5, 0.5, alpha=0.3)  # Yarı şeffaf gri
    can.saveState()
    can.translate(300, 400)
    can.rotate(45)
    can.drawCentredString(0, 0, text)
    can.restoreState()
    can.save()
    
    packet.seek(0)
    
    if output_path:
        with open(output_path, "wb") as f:
            f.write(packet.getvalue())
    
    return PdfReader(packet)

def add_watermark(input_path, output_path, watermark_text):
    """PDF'e watermark ekle"""
    # Ana PDF
    reader = PdfReader(input_path)
    writer = PdfWriter()
    
    # Watermark oluştur
    watermark = create_watermark(watermark_text)
    watermark_page = watermark.pages[0]
    
    # Her sayfaya watermark ekle
    for page in reader.pages:
        page.merge_page(watermark_page)
        writer.add_page(page)
    
    # Kaydet
    with open(output_path, "wb") as output_file:
        writer.write(output_file)
    
    print(f"Watermark eklendi: {output_path}")

# Kullanım
add_watermark("document.pdf", "watermarked.pdf", "CONFIDENTIAL")
```

### Image Watermark

```python
from pypdf import PdfReader, PdfWriter
from reportlab.pdfgen import canvas
from reportlab.lib.pagesizes import letter
from PIL import Image
import io

def add_image_watermark(input_path, output_path, image_path, opacity=0.3):
    """Resim watermark ekle"""
    reader = PdfReader(input_path)
    
    # Watermark oluştur
    packet = io.BytesIO()
    can = canvas.Canvas(packet, pagesize=letter)
    
    # Resmi ekle
    can.setFillAlpha(opacity)
    can.drawImage(image_path, 200, 300, width=200, height=200, 
                  preserveAspectRatio=True, mask='auto')
    can.save()
    
    packet.seek(0)
    watermark = PdfReader(packet)
    watermark_page = watermark.pages[0]
    
    # Her sayfaya ekle
    writer = PdfWriter()
    for page in reader.pages:
        page.merge_page(watermark_page)
        writer.add_page(page)
    
    with open(output_path, "wb") as output_file:
        writer.write(output_file)
    
    print(f"Resim watermark eklendi: {output_path}")

# Kullanım
add_image_watermark("document.pdf", "branded.pdf", "logo.png", opacity=0.2)
```

## ReportLab ile PDF Oluşturma

### Basit PDF Oluşturma

```python
from reportlab.pdfgen import canvas
from reportlab.lib.pagesizes import letter, A4
from reportlab.lib.units import inch

def create_simple_pdf(output_path):
    """Basit PDF oluştur"""
    c = canvas.Canvas(output_path, pagesize=letter)
    width, height = letter
    
    # Başlık
    c.setFont("Helvetica-Bold", 24)
    c.drawString(1*inch, height - 1*inch, "Python PDF Raporu")
    
    # Alt başlık
    c.setFont("Helvetica", 14)
    c.drawString(1*inch, height - 1.5*inch, "ReportLab ile oluşturuldu")
    
    # Çizgi
    c.line(1*inch, height - 1.7*inch, width - 1*inch, height - 1.7*inch)
    
    # Metin paragrafı
    c.setFont("Helvetica", 12)
    text_object = c.beginText(1*inch, height - 2.5*inch)
    text_object.textLines("""
    Bu bir örnek PDF belgesidir.
    Python ve ReportLab kullanılarak oluşturulmuştur.
    
    PDF oluşturma işlemleri:
    - Metin ekleme
    - Şekil çizme
    - Resim ekleme
    - Tablo oluşturma
    """)
    c.drawText(text_object)
    
    # Dikdörtgen
    c.setStrokeColorRGB(0, 0, 1)
    c.setFillColorRGB(0.8, 0.8, 1)
    c.rect(1*inch, 2*inch, 3*inch, 1*inch, fill=1)
    
    # Daire
    c.setFillColorRGB(1, 0.8, 0.8)
    c.circle(5*inch, 2.5*inch, 0.5*inch, fill=1)
    
    c.save()
    print(f"PDF oluşturuldu: {output_path}")

# Kullanım
create_simple_pdf("simple_report.pdf")
```

### Çok Sayfalı PDF

```python
from reportlab.pdfgen import canvas
from reportlab.lib.pagesizes import letter
from reportlab.lib.units import inch

def create_multipage_pdf(output_path, num_pages=5):
    """Çok sayfalı PDF oluştur"""
    c = canvas.Canvas(output_path, pagesize=letter)
    width, height = letter
    
    for page_num in range(1, num_pages + 1):
        # Başlık
        c.setFont("Helvetica-Bold", 18)
        c.drawString(1*inch, height - 1*inch, f"Sayfa {page_num}")
        
        # İçerik
        c.setFont("Helvetica", 12)
        y_position = height - 2*inch
        
        for line_num in range(1, 20):
            text = f"Sayfa {page_num}, Satır {line_num}: Örnek metin içeriği"
            c.drawString(1*inch, y_position, text)
            y_position -= 0.3*inch
        
        # Sayfa numarası (footer)
        c.setFont("Helvetica", 10)
        c.drawCentredString(width/2, 0.5*inch, f"- {page_num} -")
        
        # Yeni sayfa
        if page_num < num_pages:
            c.showPage()
    
    c.save()
    print(f"Çok sayfalı PDF oluşturuldu: {output_path}")

# Kullanım
create_multipage_pdf("multipage_report.pdf", num_pages=10)
```

### Resim Ekleme

```python
from reportlab.pdfgen import canvas
from reportlab.lib.pagesizes import letter
from reportlab.lib.units import inch
from PIL import Image

def create_pdf_with_images(output_path, image_paths):
    """Resimlerle PDF oluştur"""
    c = canvas.Canvas(output_path, pagesize=letter)
    width, height = letter
    
    y_position = height - 1*inch
    
    for i, image_path in enumerate(image_paths):
        # Başlık
        c.setFont("Helvetica-Bold", 14)
        c.drawString(1*inch, y_position, f"Resim {i+1}")
        y_position -= 0.5*inch
        
        # Resim boyutlarını al
        img = Image.open(image_path)
        img_width, img_height = img.size
        
        # Boyutlandır (max 4 inch genişlik)
        max_width = 4*inch
        aspect_ratio = img_height / img_width
        display_width = min(max_width, img_width)
        display_height = display_width * aspect_ratio
        
        # Sayfaya sığmazsa yeni sayfa
        if y_position - display_height < 1*inch:
            c.showPage()
            y_position = height - 1*inch
        
        # Resmi ekle
        c.drawImage(image_path, 1*inch, y_position - display_height,
                   width=display_width, height=display_height)
        
        y_position -= (display_height + 0.5*inch)
    
    c.save()
    print(f"Resimli PDF oluşturuldu: {output_path}")

# Kullanım
images = ["chart1.png", "graph2.png", "photo3.jpg"]
create_pdf_with_images("image_report.pdf", images)
```

### Tablo Oluşturma

```python
from reportlab.lib import colors
from reportlab.lib.pagesizes import letter
from reportlab.platypus import SimpleDocTemplate, Table, TableStyle, Paragraph
from reportlab.lib.styles import getSampleStyleSheet
from reportlab.lib.units import inch

def create_pdf_with_table(output_path):
    """Tablo içeren PDF oluştur"""
    doc = SimpleDocTemplate(output_path, pagesize=letter)
    elements = []
    
    # Stil
    styles = getSampleStyleSheet()
    
    # Başlık
    title = Paragraph("<b>Satış Raporu</b>", styles['Heading1'])
    elements.append(title)
    
    # Tablo verisi
    data = [
        ['Ürün', 'Adet', 'Fiyat', 'Toplam'],
        ['Laptop', '5', '$1000', '$5000'],
        ['Mouse', '20', '$25', '$500'],
        ['Keyboard', '15', '$50', '$750'],
        ['Monitor', '8', '$300', '$2400'],
        ['', '', 'Genel Toplam:', '$8650']
    ]
    
    # Tablo oluştur
    table = Table(data, colWidths=[2*inch, 1*inch, 1*inch, 1.5*inch])
    
    # Tablo stili
    table.setStyle(TableStyle([
        # Başlık satırı
        ('BACKGROUND', (0, 0), (-1, 0), colors.grey),
        ('TEXTCOLOR', (0, 0), (-1, 0), colors.whitesmoke),
        ('ALIGN', (0, 0), (-1, 0), 'CENTER'),
        ('FONTNAME', (0, 0), (-1, 0), 'Helvetica-Bold'),
        ('FONTSIZE', (0, 0), (-1, 0), 12),
        ('BOTTOMPADDING', (0, 0), (-1, 0), 12),
        
        # Data satırları
        ('BACKGROUND', (0, 1), (-1, -2), colors.beige),
        ('TEXTCOLOR', (0, 1), (-1, -1), colors.black),
        ('ALIGN', (1, 1), (-1, -1), 'RIGHT'),
        ('FONTNAME', (0, 1), (-1, -1), 'Helvetica'),
        ('FONTSIZE', (0, 1), (-1, -1), 10),
        ('GRID', (0, 0), (-1, -2), 1, colors.black),
        
        # Toplam satırı
        ('BACKGROUND', (0, -1), (-1, -1), colors.lightblue),
        ('FONTNAME', (0, -1), (-1, -1), 'Helvetica-Bold'),
        ('ALIGN', (0, -1), (-1, -1), 'RIGHT'),
    ]))
    
    elements.append(table)
    
    # PDF oluştur
    doc.build(elements)
    print(f"Tablolu PDF oluşturuldu: {output_path}")

# Kullanım
create_pdf_with_table("sales_report.pdf")
```

## PDF Form İşlemleri

![PDF Form Filling Automation](/assets/img/posts/pdf-form-filling-automation.png)

### Form Alanlarını Okuma

```python
from pypdf import PdfReader

def read_form_fields(pdf_path):
    """PDF form alanlarını oku"""
    reader = PdfReader(pdf_path)
    
    fields = reader.get_fields()
    
    if not fields:
        print("Bu PDF'de form alanı bulunamadı")
        return None
    
    print(f"Toplam {len(fields)} form alanı bulundu:\n")
    
    for field_name, field_info in fields.items():
        field_type = field_info.get('/FT', 'Unknown')
        field_value = field_info.get('/V', '')
        
        print(f"Alan: {field_name}")
        print(f"  Tip: {field_type}")
        print(f"  Değer: {field_value}")
        print()
    
    return fields

# Kullanım
fields = read_form_fields("form_template.pdf")
```

### Form Doldurma

```python
from pypdf import PdfReader, PdfWriter

def fill_pdf_form(input_path, output_path, field_data):
    """
    PDF formunu doldur
    
    field_data = {
        'name': 'John Doe',
        'email': 'john@example.com',
        'phone': '+1234567890'
    }
    """
    reader = PdfReader(input_path)
    writer = PdfWriter()
    
    # Sayfaları kopyala
    for page in reader.pages:
        writer.add_page(page)
    
    # Form alanlarını doldur
    writer.update_page_form_field_values(
        writer.pages[0], 
        field_data
    )
    
    # Kaydet
    with open(output_path, "wb") as output_file:
        writer.write(output_file)
    
    print(f"Form dolduruldu: {output_path}")

# Kullanım
form_data = {
    'full_name': 'Jane Smith',
    'email': 'jane.smith@example.com',
    'phone': '+1-555-0123',
    'address': '123 Main St, City, State 12345',
    'date': '2025-03-29'
}

fill_pdf_form("application_form.pdf", "filled_form.pdf", form_data)
```

### Toplu Form Doldurma

```python
from pypdf import PdfReader, PdfWriter
import pandas as pd

def bulk_fill_forms(template_path, data_csv, output_folder):
    """CSV'den veri okuyarak toplu form doldur"""
    # CSV oku
    df = pd.read_csv(data_csv)
    
    os.makedirs(output_folder, exist_ok=True)
    
    for index, row in df.iterrows():
        reader = PdfReader(template_path)
        writer = PdfWriter()
        
        # Sayfaları kopyala
        for page in reader.pages:
            writer.add_page(page)
        
        # Form doldur
        form_data = row.to_dict()
        writer.update_page_form_field_values(
            writer.pages[0],
            form_data
        )
        
        # Kaydet
        output_path = os.path.join(
            output_folder,
            f"form_{index+1}_{row['name'].replace(' ', '_')}.pdf"
        )
        
        with open(output_path, "wb") as output_file:
            writer.write(output_file)
        
        print(f"Oluşturuldu: {output_path}")

# Kullanım
# CSV format: name,email,phone,address
bulk_fill_forms("template.pdf", "applicants.csv", "filled_forms/")
```

## pdfplumber ile Gelişmiş Okuma

### Tablo Çıkarma

```python
import pdfplumber
import pandas as pd

def extract_tables_from_pdf(pdf_path):
    """PDF'den tabloları çıkar"""
    with pdfplumber.open(pdf_path) as pdf:
        all_tables = []
        
        for page_num, page in enumerate(pdf.pages):
            print(f"Sayfa {page_num + 1} işleniyor...")
            
            tables = page.extract_tables()
            
            for table_num, table in enumerate(tables):
                if table:
                    # DataFrame'e dönüştür
                    df = pd.DataFrame(table[1:], columns=table[0])
                    all_tables.append({
                        'page': page_num + 1,
                        'table': table_num + 1,
                        'data': df
                    })
                    
                    print(f"  Tablo {table_num + 1} bulundu:")
                    print(df.head())
                    print()
        
        return all_tables

# Kullanım
tables = extract_tables_from_pdf("financial_report.pdf")

# İlk tabloyu CSV'ye kaydet
if tables:
    tables[0]['data'].to_csv("extracted_table.csv", index=False)
```

### Gelişmiş Metin Çıkarma

```python
import pdfplumber
import re

def extract_structured_text(pdf_path):
    """Yapılandırılmış metin çıkarma"""
    with pdfplumber.open(pdf_path) as pdf:
        full_text = ""
        
        for page in pdf.pages:
            # Metin çıkar
            text = page.extract_text()
            full_text += text + "\n"
            
            # Sayfa bilgisi
            print(f"Sayfa boyutları: {page.width} x {page.height}")
            print(f"Sayfa numarası: {page.page_number}")
            
            # Karakterler
            chars = page.chars
            print(f"Karakter sayısı: {len(chars)}")
            
            # Fontlar
            fonts = set([char['fontname'] for char in chars])
            print(f"Kullanılan fontlar: {fonts}")
            print()
        
        return full_text

def extract_emails_and_urls(pdf_path):
    """E-posta ve URL'leri çıkar"""
    with pdfplumber.open(pdf_path) as pdf:
        full_text = ""
        for page in pdf.pages:
            full_text += page.extract_text() + "\n"
    
    # Regex ile e-posta ve URL bul
    emails = re.findall(r'\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b', full_text)
    urls = re.findall(r'http[s]?://(?:[a-zA-Z]|[0-9]|[$-_@.&+]|[!*\\(\\),]|(?:%[0-9a-fA-F][0-9a-fA-F]))+', full_text)
    
    return {
        'emails': list(set(emails)),
        'urls': list(set(urls))
    }

# Kullanım
text = extract_structured_text("document.pdf")
contacts = extract_emails_and_urls("document.pdf")

print("E-postalar:", contacts['emails'])
print("URL'ler:", contacts['urls'])
```

### Sayfa Görsellerini Çıkarma

```python
import pdfplumber
from PIL import Image
import io

def extract_images_from_pdf(pdf_path, output_folder):
    """PDF'den resimleri çıkar"""
    os.makedirs(output_folder, exist_ok=True)
    
    with pdfplumber.open(pdf_path) as pdf:
        image_count = 0
        
        for page_num, page in enumerate(pdf.pages):
            # Sayfayı resme çevir
            im = page.to_image(resolution=200)
            
            # Sayfa resmini kaydet
            page_image_path = os.path.join(
                output_folder,
                f"page_{page_num + 1}.png"
            )
            im.save(page_image_path)
            print(f"Sayfa resmi kaydedildi: {page_image_path}")
            
            # PDF içindeki görselleri çıkar
            for img_num, img in enumerate(page.images):
                image_count += 1
                print(f"Resim {image_count} bulundu: {img}")

# Kullanım
extract_images_from_pdf("document.pdf", "extracted_images/")
```

## Performans ve Optimizasyon

### Büyük PDF'leri İşleme

```python
from pypdf import PdfReader
import multiprocessing as mp

def process_page(args):
    """Tek bir sayfayı işle (paralel çalışma için)"""
    pdf_path, page_num = args
    
    reader = PdfReader(pdf_path)
    page = reader.pages[page_num]
    text = page.extract_text()
    
    return {
        'page': page_num,
        'text': text,
        'word_count': len(text.split())
    }

def process_large_pdf_parallel(pdf_path, num_workers=4):
    """Büyük PDF'i paralel işle"""
    reader = PdfReader(pdf_path)
    num_pages = len(reader.pages)
    
    # Her sayfa için argüman hazırla
    args = [(pdf_path, i) for i in range(num_pages)]
    
    # Paralel işleme
    with mp.Pool(processes=num_workers) as pool:
        results = pool.map(process_page, args)
    
    return results

# Kullanım
results = process_large_pdf_parallel("large_document.pdf", num_workers=8)

total_words = sum(r['word_count'] for r in results)
print(f"Toplam kelime sayısı: {total_words}")
```

### Batch PDF İşleme

```python
import os
from pypdf import PdfReader, PdfWriter
from concurrent.futures import ThreadPoolExecutor, as_completed

def process_single_pdf(file_path, operation):
    """Tek bir PDF'i işle"""
    try:
        if operation == "compress":
            # Sıkıştırma işlemi
            reader = PdfReader(file_path)
            writer = PdfWriter()
            
            for page in reader.pages:
                page.compress_content_streams()
                writer.add_page(page)
            
            output_path = file_path.replace(".pdf", "_compressed.pdf")
            with open(output_path, "wb") as f:
                writer.write(f)
            
            return f"Compressed: {file_path}"
        
        elif operation == "extract_text":
            reader = PdfReader(file_path)
            text = ""
            for page in reader.pages:
                text += page.extract_text()
            
            txt_path = file_path.replace(".pdf", ".txt")
            with open(txt_path, "w", encoding="utf-8") as f:
                f.write(text)
            
            return f"Text extracted: {file_path}"
    
    except Exception as e:
        return f"Error processing {file_path}: {str(e)}"

def batch_process_pdfs(folder_path, operation, max_workers=4):
    """Klasördeki tüm PDF'leri işle"""
    pdf_files = [
        os.path.join(folder_path, f)
        for f in os.listdir(folder_path)
        if f.endswith('.pdf')
    ]
    
    print(f"Toplam {len(pdf_files)} PDF bulundu")
    
    with ThreadPoolExecutor(max_workers=max_workers) as executor:
        futures = {
            executor.submit(process_single_pdf, pdf, operation): pdf
            for pdf in pdf_files
        }
        
        for future in as_completed(futures):
            result = future.result()
            print(result)

# Kullanım
batch_process_pdfs("pdf_folder/", operation="compress", max_workers=8)
batch_process_pdfs("pdf_folder/", operation="extract_text", max_workers=8)
```

## PDF Otomasyon Örnekleri

### Fatura Oluşturma

```python
from reportlab.lib.pagesizes import letter
from reportlab.lib import colors
from reportlab.platypus import SimpleDocTemplate, Table, TableStyle, Paragraph, Spacer
from reportlab.lib.styles import getSampleStyleSheet, ParagraphStyle
from reportlab.lib.units import inch
from datetime import datetime

def create_invoice(invoice_data, output_path):
    """
    Profesyonel fatura oluştur
    
    invoice_data = {
        'invoice_number': 'INV-2025-001',
        'date': '2025-03-29',
        'customer': {...},
        'items': [...],
        'subtotal': 1000,
        'tax': 100,
        'total': 1100
    }
    """
    doc = SimpleDocTemplate(output_path, pagesize=letter)
    elements = []
    styles = getSampleStyleSheet()
    
    # Başlık
    title_style = ParagraphStyle(
        'CustomTitle',
        parent=styles['Heading1'],
        fontSize=24,
        textColor=colors.HexColor('#2C3E50'),
        spaceAfter=30
    )
    title = Paragraph("<b>FATURA</b>", title_style)
    elements.append(title)
    
    # Fatura bilgileri
    info_data = [
        ['Fatura No:', invoice_data['invoice_number']],
        ['Tarih:', invoice_data['date']],
        ['Müşteri:', invoice_data['customer']['name']],
        ['Adres:', invoice_data['customer']['address']],
    ]
    
    info_table = Table(info_data, colWidths=[1.5*inch, 4*inch])
    info_table.setStyle(TableStyle([
        ('FONTNAME', (0, 0), (-1, -1), 'Helvetica'),
        ('FONTSIZE', (0, 0), (-1, -1), 10),
        ('TEXTCOLOR', (0, 0), (0, -1), colors.grey),
        ('ALIGN', (0, 0), (0, -1), 'RIGHT'),
    ]))
    elements.append(info_table)
    elements.append(Spacer(1, 0.5*inch))
    
    # Ürün tablosu
    item_data = [['Ürün', 'Miktar', 'Birim Fiyat', 'Toplam']]
    
    for item in invoice_data['items']:
        item_data.append([
            item['name'],
            str(item['quantity']),
            f"${item['price']:.2f}",
            f"${item['total']:.2f}"
        ])
    
    # Toplamlar
    item_data.append(['', '', 'Ara Toplam:', f"${invoice_data['subtotal']:.2f}"])
    item_data.append(['', '', 'KDV (%10):', f"${invoice_data['tax']:.2f}"])
    item_data.append(['', '', 'Genel Toplam:', f"${invoice_data['total']:.2f}"])
    
    item_table = Table(item_data, colWidths=[3*inch, 1*inch, 1.5*inch, 1.5*inch])
    item_table.setStyle(TableStyle([
        # Başlık
        ('BACKGROUND', (0, 0), (-1, 0), colors.HexColor('#3498DB')),
        ('TEXTCOLOR', (0, 0), (-1, 0), colors.whitesmoke),
        ('ALIGN', (0, 0), (-1, 0), 'CENTER'),
        ('FONTNAME', (0, 0), (-1, 0), 'Helvetica-Bold'),
        
        # Ürün satırları
        ('FONTNAME', (0, 1), (-1, -4), 'Helvetica'),
        ('ALIGN', (1, 1), (-1, -1), 'RIGHT'),
        ('GRID', (0, 0), (-1, -4), 0.5, colors.grey),
        
        # Toplam satırları
        ('FONTNAME', (2, -3), (-1, -1), 'Helvetica-Bold'),
        ('LINEABOVE', (2, -3), (-1, -3), 1, colors.black),
        ('LINEABOVE', (2, -1), (-1, -1), 2, colors.black),
        ('BACKGROUND', (2, -1), (-1, -1), colors.HexColor('#ECF0F1')),
    ]))
    
    elements.append(item_table)
    elements.append(Spacer(1, 0.5*inch))
    
    # Footer
    footer = Paragraph(
        "<i>Ödemenizi banka hesabımıza yapabilirsiniz.<br/>Teşekkür ederiz!</i>",
        styles['Normal']
    )
    elements.append(footer)
    
    # PDF oluştur
    doc.build(elements)
    print(f"Fatura oluşturuldu: {output_path}")

# Kullanım
invoice = {
    'invoice_number': 'INV-2025-001',
    'date': datetime.now().strftime('%Y-%m-%d'),
    'customer': {
        'name': 'ABC Şirketi',
        'address': 'İstanbul, Türkiye'
    },
    'items': [
        {'name': 'Web Development', 'quantity': 40, 'price': 50, 'total': 2000},
        {'name': 'Consulting', 'quantity': 10, 'price': 100, 'total': 1000},
        {'name': 'Hosting (1 year)', 'quantity': 1, 'price': 500, 'total': 500},
    ],
    'subtotal': 3500,
    'tax': 350,
    'total': 3850
}

create_invoice(invoice, "invoice_001.pdf")
```

### Rapor Otomasyonu

```python
from reportlab.lib.pagesizes import letter
from reportlab.platypus import SimpleDocTemplate, Paragraph, Spacer, PageBreak
from reportlab.platypus import Table, TableStyle, Image
from reportlab.lib.styles import getSampleStyleSheet, ParagraphStyle
from reportlab.lib import colors
from reportlab.lib.units import inch
import matplotlib.pyplot as plt
import pandas as pd

def generate_chart(data, output_path):
    """Grafik oluştur"""
    plt.figure(figsize=(8, 5))
    plt.plot(data['x'], data['y'], marker='o')
    plt.title('Aylık Satış Grafiği')
    plt.xlabel('Ay')
    plt.ylabel('Satış ($)')
    plt.grid(True, alpha=0.3)
    plt.savefig(output_path, dpi=150, bbox_inches='tight')
    plt.close()

def create_monthly_report(data, output_path):
    """Aylık rapor oluştur"""
    doc = SimpleDocTemplate(output_path, pagesize=letter)
    elements = []
    styles = getSampleStyleSheet()
    
    # Başlık
    title_style = ParagraphStyle(
        'ReportTitle',
        parent=styles['Heading1'],
        fontSize=20,
        textColor=colors.HexColor('#1A5490'),
        spaceAfter=20
    )
    
    title = Paragraph("<b>Aylık Satış Raporu</b>", title_style)
    elements.append(title)
    elements.append(Spacer(1, 0.3*inch))
    
    # Özet
    summary = Paragraph(
        f"<b>Rapor Dönemi:</b> {data['period']}<br/>"
        f"<b>Toplam Satış:</b> ${data['total_sales']:,.2f}<br/>"
        f"<b>Önceki Aya Göre:</b> {data['change']:+.1f}%",
        styles['Normal']
    )
    elements.append(summary)
    elements.append(Spacer(1, 0.5*inch))
    
    # Grafik oluştur ve ekle
    chart_path = "temp_chart.png"
    generate_chart(data['chart_data'], chart_path)
    
    chart = Image(chart_path, width=5*inch, height=3*inch)
    elements.append(chart)
    elements.append(Spacer(1, 0.5*inch))
    
    # Tablo
    table_data = [['Kategori', 'Satış', 'Yüzde']]
    for item in data['categories']:
        table_data.append([
            item['name'],
            f"${item['sales']:,.2f}",
            f"{item['percentage']:.1f}%"
        ])
    
    table = Table(table_data, colWidths=[2*inch, 1.5*inch, 1*inch])
    table.setStyle(TableStyle([
        ('BACKGROUND', (0, 0), (-1, 0), colors.HexColor('#3498DB')),
        ('TEXTCOLOR', (0, 0), (-1, 0), colors.whitesmoke),
        ('ALIGN', (1, 1), (-1, -1), 'RIGHT'),
        ('FONTNAME', (0, 0), (-1, 0), 'Helvetica-Bold'),
        ('GRID', (0, 0), (-1, -1), 0.5, colors.grey),
    ]))
    
    elements.append(table)
    
    # PDF oluştur
    doc.build(elements)
    
    # Geçici dosyayı temizle
    os.remove(chart_path)
    
    print(f"Rapor oluşturuldu: {output_path}")

# Kullanım
report_data = {
    'period': 'Mart 2025',
    'total_sales': 45000,
    'change': 12.5,
    'chart_data': {
        'x': ['Oca', 'Şub', 'Mar', 'Nis', 'May'],
        'y': [30000, 35000, 40000, 45000, 42000]
    },
    'categories': [
        {'name': 'Elektronik', 'sales': 20000, 'percentage': 44.4},
        {'name': 'Giyim', 'sales': 15000, 'percentage': 33.3},
        {'name': 'Kitap', 'sales': 10000, 'percentage': 22.2},
    ]
}

create_monthly_report(report_data, "monthly_report_march_2025.pdf")
```

## Sonuç

Python ile PDF işlemleri, belgelerinizi tamamen otomatikleştirebileceğiniz güçlü bir yetenektir. Bu rehberde öğrendiğiniz tekniklerle:

- Profesyonel raporlar ve faturalar oluşturabilirsiniz
- Binlerce PDF'i otomatik olarak işleyebilirsiniz
- Form doldurma süreçlerini otomatikleştirebilirsiniz
- PDF'lerden veri çıkarabilir ve analiz edebilirsiniz

### Kaynaklar

- [ReportLab Documentation](https://www.reportlab.com/docs/)
- [pypdf Documentation](https://pypdf.readthedocs.io/)
- [pdfplumber Documentation](https://github.com/jsvine/pdfplumber)
- [PyPDF2 Tutorial](https://realpython.com/pdf-python/)

PDF otomasyonu ile iş süreçlerinizi hızlandırın ve verimliliğinizi artırın! 🚀
