---
title: "Python ile Görüntü İşleme ve Computer Vision: Pillow ve OpenCV Kullanımı"
date: 2025-06-28 09:00:00 +0300
categories: [Image Processing, Computer Vision]
tags: [python, pillow, image-processing, opencv, manipulation, automation]
image:
  path: /assets/img/posts/python-image-processing-pillow.jpg
  alt: "Python Pillow ile Görüntü İşleme"
---

Görüntü işleme, modern yazılım geliştirmenin vazgeçilmez bir parçası haline gelmiştir. Web uygulamalarından mobil uygulamalara, makine öğrenmesinden veri görselleştirmeye kadar birçok alanda görüntü manipülasyonuna ihtiyaç duyarız. Python ekosistemi, Pillow ve OpenCV gibi güçlü kütüphaneler sayesinde bu ihtiyaçları karşılamak için mükemmel araçlar sunar.

## Görüntü İşleme Nedir?

Görüntü işleme, dijital görüntüler üzerinde çeşitli operasyonlar gerçekleştirerek onları dönüştürme, analiz etme veya iyileştirme sürecidir. Bu süreç, basit yeniden boyutlandırmadan karmaşık makine öğrenmesi uygulamalarına kadar geniş bir yelpaze kapsar.

### Görüntü İşleme Kullanım Alanları

- **Web Uygulamaları**: Kullanıcı profil fotoğrafları, ürün görselleri
- **E-Ticaret**: Ürün görsellerinin optimize edilmesi, filigran ekleme
- **Sosyal Medya**: Filtreler, efektler, otomatik kırpma
- **Makine Öğrenmesi**: Veri ön işleme, augmentation
- **Belge İşleme**: OCR, form tanıma, QR kod okuma
- **Medikal Görüntüleme**: Röntgen, MR analizi

## Pillow ve OpenCV Karşılaştırması

![Python Görüntü İşleme Araçları](/assets/img/posts/image-processing-tools-python.png)
_Python ekosistemindeki popüler görüntü işleme araçları_

### Pillow (PIL Fork)

**Avantajları:**
- Basit ve kullanımı kolay API
- Hızlı temel operasyonlar
- Geniş format desteği (JPEG, PNG, GIF, BMP, TIFF vb.)
- Hafif ve bağımlılık az
- Web uygulamaları için ideal

**Dezavantajları:**
- Gelişmiş görüntü işleme özellikleri sınırlı
- Computer vision algoritmaları yok
- Video işleme desteği yok

### OpenCV

**Avantajları:**
- Gelişmiş computer vision algoritmaları
- Yüz tanıma, nesne tespiti
- Video işleme desteği
- Yüksek performans (C++ backend)
- Makine öğrenmesi entegrasyonu

**Dezavantajları:**
- Daha karmaşık API
- Büyük kütüphane boyutu
- Basit işlemler için overkill olabilir

## Pillow ile Temel Görüntü İşleme

### Kurulum ve İlk Adımlar

```bash
# Pillow kurulumu
pip install Pillow

# İsteğe bağlı: İlave format desteği için
pip install pillow-heif  # HEIF/HEIC format
pip install pillow-avif  # AVIF format
```

### Görüntü Yükleme ve Kaydetme

```python
from PIL import Image
import os

class ImageProcessor:
    """Temel görüntü işleme sınıfı"""
    
    def __init__(self, image_path: str):
        """
        Görüntü yükleme
        
        Args:
            image_path: Görüntü dosya yolu
        """
        self.image = Image.open(image_path)
        self.original = self.image.copy()  # Orijinali sakla
        
        # Görüntü bilgileri
        print(f"Format: {self.image.format}")
        print(f"Boyut: {self.image.size}")  # (width, height)
        print(f"Mod: {self.image.mode}")  # RGB, RGBA, L (grayscale)
    
    def save(self, output_path: str, quality: int = 85, optimize: bool = True):
        """
        Görüntüyü kaydetme
        
        Args:
            output_path: Çıktı dosya yolu
            quality: JPEG kalitesi (1-95)
            optimize: Dosya boyutu optimizasyonu
        """
        # Format otomatik tespit (uzantıdan)
        ext = os.path.splitext(output_path)[1].lower()
        
        # JPEG için özel parametreler
        if ext in ['.jpg', '.jpeg']:
            self.image.save(
                output_path,
                'JPEG',
                quality=quality,
                optimize=optimize,
                progressive=True  # Progressive JPEG
            )
        # PNG için optimizasyon
        elif ext == '.png':
            self.image.save(
                output_path,
                'PNG',
                optimize=optimize,
                compress_level=9  # Maksimum sıkıştırma
            )
        else:
            self.image.save(output_path, optimize=optimize)
        
        print(f"Saved: {output_path}")
        
        # Dosya boyutu bilgisi
        file_size = os.path.getsize(output_path) / 1024  # KB
        print(f"File size: {file_size:.2f} KB")
    
    def reset(self):
        """Orijinal görüntüye geri dön"""
        self.image = self.original.copy()


# Kullanım örneği
processor = ImageProcessor('input.jpg')
processor.save('output.jpg', quality=90)
```

### Görüntü Boyutlandırma

![Görüntü Resize, Crop ve Filter İşlemleri](/assets/img/posts/image-resize-crop-filters.jpg)
_Görüntü üzerinde temel dönüşüm operasyonları_

```python
from PIL import Image
from typing import Tuple, Optional

def resize_image(
    image: Image.Image,
    size: Tuple[int, int],
    maintain_aspect: bool = True,
    resample: int = Image.Resampling.LANCZOS
) -> Image.Image:
    """
    Görüntü boyutlandırma
    
    Args:
        image: PIL Image objesi
        size: Hedef boyut (width, height)
        maintain_aspect: En-boy oranını koru
        resample: Resampling algoritması
    
    Returns:
        Boyutlandırılmış görüntü
    """
    if maintain_aspect:
        # En-boy oranını koruyarak boyutlandır
        image.thumbnail(size, resample)
        return image
    else:
        # Tam boyuta zorlama (distortion olabilir)
        return image.resize(size, resample)


def resize_to_width(image: Image.Image, width: int) -> Image.Image:
    """Genişliği belirle, yüksekliği oranla hesapla"""
    aspect_ratio = image.height / image.width
    height = int(width * aspect_ratio)
    return image.resize((width, height), Image.Resampling.LANCZOS)


def resize_to_height(image: Image.Image, height: int) -> Image.Image:
    """Yüksekliği belirle, genişliği oranla hesapla"""
    aspect_ratio = image.width / image.height
    width = int(height * aspect_ratio)
    return image.resize((width, height), Image.Resampling.LANCZOS)


def smart_resize(
    image: Image.Image,
    max_width: int,
    max_height: int
) -> Image.Image:
    """
    Maksimum boyutları aşmadan akıllı boyutlandırma
    """
    width, height = image.size
    
    # Eğer görüntü zaten küçükse, dokunma
    if width <= max_width and height <= max_height:
        return image
    
    # En-boy oranını koruyarak boyutlandır
    width_ratio = max_width / width
    height_ratio = max_height / height
    
    # En küçük oranı kullan (her iki boyut da sınırlar içinde kalır)
    ratio = min(width_ratio, height_ratio)
    
    new_width = int(width * ratio)
    new_height = int(height * ratio)
    
    return image.resize((new_width, new_height), Image.Resampling.LANCZOS)


# Kullanım örnekleri
img = Image.open('photo.jpg')

# Basit boyutlandırma
resized = resize_image(img, (800, 600), maintain_aspect=True)

# Genişlik bazlı
width_based = resize_to_width(img, 1200)

# Maksimum boyut sınırı ile
smart = smart_resize(img, max_width=1920, max_height=1080)

# Thumbnail oluşturma (orijinali değiştirir!)
img_copy = img.copy()
img_copy.thumbnail((200, 200), Image.Resampling.LANCZOS)
```

### Görüntü Kırpma (Cropping)

```python
from PIL import Image
from typing import Tuple

def crop_center(image: Image.Image, crop_width: int, crop_height: int) -> Image.Image:
    """
    Merkezi kırpma
    
    Args:
        image: PIL Image objesi
        crop_width: Kırpma genişliği
        crop_height: Kırpma yüksekliği
    
    Returns:
        Kırpılmış görüntü
    """
    width, height = image.size
    
    # Merkez koordinatları
    left = (width - crop_width) // 2
    top = (height - crop_height) // 2
    right = left + crop_width
    bottom = top + crop_height
    
    # Kırpma (left, top, right, bottom)
    return image.crop((left, top, right, bottom))


def crop_to_aspect_ratio(
    image: Image.Image,
    aspect_ratio: float
) -> Image.Image:
    """
    Belirli bir en-boy oranına kırpma
    
    Args:
        image: PIL Image objesi
        aspect_ratio: Hedef en-boy oranı (width/height)
                      Örn: 16/9 = 1.778, 4/3 = 1.333, 1/1 = 1.0
    """
    width, height = image.size
    current_ratio = width / height
    
    if current_ratio > aspect_ratio:
        # Görüntü çok geniş, kenarlardan kes
        new_width = int(height * aspect_ratio)
        left = (width - new_width) // 2
        return image.crop((left, 0, left + new_width, height))
    else:
        # Görüntü çok yüksek, üst/alttan kes
        new_height = int(width / aspect_ratio)
        top = (height - new_height) // 2
        return image.crop((0, top, width, top + new_height))


def smart_crop_face_detection(image_path: str, output_size: Tuple[int, int]) -> Image.Image:
    """
    Yüz tespiti ile akıllı kırpma (OpenCV gerektirir)
    
    Not: Bu fonksiyon Pillow ve OpenCV'yi birlikte kullanır
    """
    import cv2
    import numpy as np
    
    # Görüntüyü yükle
    img = Image.open(image_path)
    cv_image = cv2.cvtColor(np.array(img), cv2.COLOR_RGB2BGR)
    
    # Yüz tespiti için Haar Cascade
    face_cascade = cv2.CascadeClassifier(
        cv2.data.haarcascades + 'haarcascade_frontalface_default.xml'
    )
    
    # Gri tonlamaya çevir (yüz tespiti için)
    gray = cv2.cvtColor(cv_image, cv2.COLOR_BGR2GRAY)
    
    # Yüzleri tespit et
    faces = face_cascade.detectMultiScale(gray, 1.1, 4)
    
    if len(faces) > 0:
        # İlk yüzü merkez al
        x, y, w, h = faces[0]
        
        # Yüz etrafında kırpma alanı hesapla
        center_x = x + w // 2
        center_y = y + h // 2
        
        crop_w, crop_h = output_size
        left = max(0, center_x - crop_w // 2)
        top = max(0, center_y - crop_h // 2)
        right = min(img.width, left + crop_w)
        bottom = min(img.height, top + crop_h)
        
        return img.crop((left, top, right, bottom))
    else:
        # Yüz bulunamazsa merkezi kırp
        return crop_center(img, *output_size)


# Kullanım örnekleri
img = Image.open('landscape.jpg')

# Merkezi kırpma
cropped = crop_center(img, 800, 600)

# 16:9 en-boy oranına kırpma
wide_screen = crop_to_aspect_ratio(img, 16/9)

# Kare kırpma (1:1)
square = crop_to_aspect_ratio(img, 1.0)

# Yüz tespiti ile kırpma
face_cropped = smart_crop_face_detection('portrait.jpg', (400, 400))
```

### Görüntü Filtreleri ve Efektler

```python
from PIL import Image, ImageFilter, ImageEnhance
from typing import Optional

class ImageEffects:
    """Görüntü efektleri ve filtreleri"""
    
    @staticmethod
    def apply_blur(image: Image.Image, radius: int = 2) -> Image.Image:
        """Bulanıklaştırma efekti"""
        return image.filter(ImageFilter.GaussianBlur(radius))
    
    @staticmethod
    def apply_sharpen(image: Image.Image, factor: float = 1.5) -> Image.Image:
        """Keskinleştirme efekti"""
        enhancer = ImageEnhance.Sharpness(image)
        return enhancer.enhance(factor)
    
    @staticmethod
    def apply_edge_enhance(image: Image.Image) -> Image.Image:
        """Kenar belirginleştirme"""
        return image.filter(ImageFilter.EDGE_ENHANCE_MORE)
    
    @staticmethod
    def apply_emboss(image: Image.Image) -> Image.Image:
        """Kabartma efekti"""
        return image.filter(ImageFilter.EMBOSS)
    
    @staticmethod
    def adjust_brightness(image: Image.Image, factor: float = 1.2) -> Image.Image:
        """
        Parlaklık ayarlama
        
        Args:
            factor: 1.0 = orijinal, <1.0 = koyulaştır, >1.0 = aydınlat
        """
        enhancer = ImageEnhance.Brightness(image)
        return enhancer.enhance(factor)
    
    @staticmethod
    def adjust_contrast(image: Image.Image, factor: float = 1.2) -> Image.Image:
        """
        Kontrast ayarlama
        
        Args:
            factor: 1.0 = orijinal, <1.0 = azalt, >1.0 = arttır
        """
        enhancer = ImageEnhance.Contrast(image)
        return enhancer.enhance(factor)
    
    @staticmethod
    def adjust_saturation(image: Image.Image, factor: float = 1.3) -> Image.Image:
        """
        Renk doygunluğu ayarlama
        
        Args:
            factor: 1.0 = orijinal, 0.0 = siyah-beyaz, >1.0 = daha canlı
        """
        enhancer = ImageEnhance.Color(image)
        return enhancer.enhance(factor)
    
    @staticmethod
    def convert_to_grayscale(image: Image.Image) -> Image.Image:
        """Siyah-beyaz dönüşümü"""
        return image.convert('L')
    
    @staticmethod
    def apply_sepia(image: Image.Image) -> Image.Image:
        """Sepia (nostaljik) efekti"""
        # RGB moduna çevir
        img = image.convert('RGB')
        pixels = img.load()
        
        for i in range(img.width):
            for j in range(img.height):
                r, g, b = pixels[i, j]
                
                # Sepia formülü
                tr = int(0.393 * r + 0.769 * g + 0.189 * b)
                tg = int(0.349 * r + 0.686 * g + 0.168 * b)
                tb = int(0.272 * r + 0.534 * g + 0.131 * b)
                
                # 255 sınırını aşma kontrolü
                pixels[i, j] = (min(tr, 255), min(tg, 255), min(tb, 255))
        
        return img
    
    @staticmethod
    def create_thumbnail_with_border(
        image: Image.Image,
        size: tuple,
        border_size: int = 5,
        border_color: str = 'white'
    ) -> Image.Image:
        """Kenarlıklı thumbnail oluşturma"""
        # Thumbnail oluştur
        img = image.copy()
        img.thumbnail(size, Image.Resampling.LANCZOS)
        
        # Kenarlık ekle
        bordered = Image.new(
            'RGB',
            (img.width + border_size * 2, img.height + border_size * 2),
            border_color
        )
        bordered.paste(img, (border_size, border_size))
        
        return bordered


# Kullanım örnekleri
img = Image.open('photo.jpg')

# Filtreler
blurred = ImageEffects.apply_blur(img, radius=5)
sharpened = ImageEffects.apply_sharpen(img, factor=2.0)
embossed = ImageEffects.apply_emboss(img)

# Renk ayarlamaları
brightened = ImageEffects.adjust_brightness(img, 1.3)
high_contrast = ImageEffects.adjust_contrast(img, 1.5)
vibrant = ImageEffects.adjust_saturation(img, 1.4)

# Dönüşümler
grayscale = ImageEffects.convert_to_grayscale(img)
sepia = ImageEffects.apply_sepia(img)

# Kenarlıklı thumbnail
thumbnail = ImageEffects.create_thumbnail_with_border(img, (300, 300))
```

### Filigran (Watermark) Ekleme

```python
from PIL import Image, ImageDraw, ImageFont
from typing import Tuple, Optional

def add_text_watermark(
    image: Image.Image,
    text: str,
    position: str = 'bottom-right',
    font_size: int = 36,
    opacity: int = 128,
    margin: int = 20
) -> Image.Image:
    """
    Metin filigranı ekleme
    
    Args:
        image: PIL Image objesi
        text: Filigran metni
        position: Konum ('bottom-right', 'bottom-left', 'top-right', 'top-left', 'center')
        font_size: Font boyutu
        opacity: Opaklık (0-255)
        margin: Kenar boşluğu
    """
    # RGBA moduna çevir (transparanlık için)
    img = image.convert('RGBA')
    
    # Transparent katman oluştur
    txt_layer = Image.new('RGBA', img.size, (255, 255, 255, 0))
    draw = ImageDraw.Draw(txt_layer)
    
    # Font yükleme (varsayılan font kullanımı)
    try:
        font = ImageFont.truetype('/usr/share/fonts/truetype/dejavu/DejaVuSans-Bold.ttf', font_size)
    except:
        font = ImageFont.load_default()
    
    # Metin boyutunu hesapla
    bbox = draw.textbbox((0, 0), text, font=font)
    text_width = bbox[2] - bbox[0]
    text_height = bbox[3] - bbox[1]
    
    # Pozisyon hesaplama
    if position == 'bottom-right':
        x = img.width - text_width - margin
        y = img.height - text_height - margin
    elif position == 'bottom-left':
        x = margin
        y = img.height - text_height - margin
    elif position == 'top-right':
        x = img.width - text_width - margin
        y = margin
    elif position == 'top-left':
        x = margin
        y = margin
    elif position == 'center':
        x = (img.width - text_width) // 2
        y = (img.height - text_height) // 2
    else:
        x, y = margin, margin
    
    # Metni çiz (beyaz renk, ayarlı opacity)
    draw.text((x, y), text, fill=(255, 255, 255, opacity), font=font)
    
    # Katmanları birleştir
    watermarked = Image.alpha_composite(img, txt_layer)
    
    # RGB'ye geri dön
    return watermarked.convert('RGB')


def add_image_watermark(
    image: Image.Image,
    watermark_path: str,
    position: str = 'bottom-right',
    scale: float = 0.1,
    opacity: int = 128,
    margin: int = 20
) -> Image.Image:
    """
    Görüntü filigranı ekleme
    
    Args:
        image: Ana görüntü
        watermark_path: Filigran görüntü yolu
        position: Konum
        scale: Filigran boyutu (ana görüntünün yüzdesi)
        opacity: Opaklık
        margin: Kenar boşluğu
    """
    # Ana görüntüyü RGBA'ya çevir
    base = image.convert('RGBA')
    
    # Filigran yükle
    watermark = Image.open(watermark_path).convert('RGBA')
    
    # Filigran boyutunu ayarla
    wm_width = int(base.width * scale)
    wm_height = int(watermark.height * (wm_width / watermark.width))
    watermark = watermark.resize((wm_width, wm_height), Image.Resampling.LANCZOS)
    
    # Opacity ayarla
    alpha = watermark.split()[3]
    alpha = alpha.point(lambda p: int(p * (opacity / 255)))
    watermark.putalpha(alpha)
    
    # Pozisyon hesapla
    if position == 'bottom-right':
        x = base.width - watermark.width - margin
        y = base.height - watermark.height - margin
    elif position == 'bottom-left':
        x = margin
        y = base.height - watermark.height - margin
    elif position == 'top-right':
        x = base.width - watermark.width - margin
        y = margin
    elif position == 'top-left':
        x = margin
        y = margin
    elif position == 'center':
        x = (base.width - watermark.width) // 2
        y = (base.height - watermark.height) // 2
    else:
        x, y = margin, margin
    
    # Filigranı yapıştır
    base.paste(watermark, (x, y), watermark)
    
    return base.convert('RGB')


# Kullanım örnekleri
img = Image.open('photo.jpg')

# Metin filigranı
watermarked_text = add_text_watermark(
    img,
    text='© 2025 My Company',
    position='bottom-right',
    font_size=40,
    opacity=150
)

# Görüntü filigranı
watermarked_image = add_image_watermark(
    img,
    watermark_path='logo.png',
    position='bottom-right',
    scale=0.15,
    opacity=180
)
```

## OpenCV ile Gelişmiş Görüntü İşleme

![Python Görüntü Manipülasyon Teknikleri](/assets/img/posts/python-image-manipulation.png)
_OpenCV ile gelişmiş görüntü işleme teknikleri_

### OpenCV Kurulumu ve Temel Kullanım

```bash
# OpenCV kurulumu
pip install opencv-python

# Ekstra modüller (opsiyonel)
pip install opencv-contrib-python

# NumPy (gerekli)
pip install numpy
```

```python
import cv2
import numpy as np
from typing import Tuple

class OpenCVProcessor:
    """OpenCV ile görüntü işleme"""
    
    def __init__(self, image_path: str):
        """Görüntü yükleme"""
        # BGR formatında yükler (not RGB!)
        self.image = cv2.imread(image_path)
        
        if self.image is None:
            raise ValueError(f"Could not load image: {image_path}")
        
        print(f"Shape: {self.image.shape}")  # (height, width, channels)
        print(f"Size: {self.image.size}")    # total pixels
        print(f"Dtype: {self.image.dtype}")  # uint8
    
    def show(self, window_name: str = 'Image'):
        """Görüntüyü göster"""
        cv2.imshow(window_name, self.image)
        cv2.waitKey(0)
        cv2.destroyAllWindows()
    
    def save(self, output_path: str, quality: int = 95):
        """Görüntüyü kaydet"""
        if output_path.endswith('.jpg') or output_path.endswith('.jpeg'):
            cv2.imwrite(
                output_path,
                self.image,
                [cv2.IMWRITE_JPEG_QUALITY, quality]
            )
        elif output_path.endswith('.png'):
            cv2.imwrite(
                output_path,
                self.image,
                [cv2.IMWRITE_PNG_COMPRESSION, 9]
            )
        else:
            cv2.imwrite(output_path, self.image)
        
        print(f"Saved: {output_path}")
    
    def to_rgb(self):
        """BGR'den RGB'ye dönüşüm"""
        self.image = cv2.cvtColor(self.image, cv2.COLOR_BGR2RGB)
        return self
    
    def to_grayscale(self):
        """Gri tonlamaya dönüşüm"""
        self.image = cv2.cvtColor(self.image, cv2.COLOR_BGR2GRAY)
        return self


# Kullanım
processor = OpenCVProcessor('photo.jpg')
processor.to_grayscale().save('gray.jpg')
```

### Renk Uzayı Dönüşümleri

```python
import cv2
import numpy as np

def convert_color_space(image: np.ndarray, conversion: str) -> np.ndarray:
    """
    Renk uzayı dönüşümleri
    
    Args:
        image: NumPy array (OpenCV formatı)
        conversion: 'rgb', 'hsv', 'lab', 'gray', 'yuv'
    """
    conversions = {
        'rgb': cv2.COLOR_BGR2RGB,
        'hsv': cv2.COLOR_BGR2HSV,
        'lab': cv2.COLOR_BGR2LAB,
        'gray': cv2.COLOR_BGR2GRAY,
        'yuv': cv2.COLOR_BGR2YUV
    }
    
    if conversion not in conversions:
        raise ValueError(f"Unknown conversion: {conversion}")
    
    return cv2.cvtColor(image, conversions[conversion])


def adjust_color_temperature(image: np.ndarray, temperature: float) -> np.ndarray:
    """
    Renk sıcaklığı ayarlama
    
    Args:
        temperature: -1.0 (soğuk/mavi) ile 1.0 (sıcak/sarı) arası
    """
    # LAB renk uzayına çevir
    lab = cv2.cvtColor(image, cv2.COLOR_BGR2LAB)
    l, a, b = cv2.split(lab)
    
    # B kanalını ayarla (mavi-sarı)
    b = b.astype(np.float32)
    b += temperature * 50
    b = np.clip(b, 0, 255).astype(np.uint8)
    
    # Kanallları birleştir ve BGR'ye dön
    lab = cv2.merge([l, a, b])
    return cv2.cvtColor(lab, cv2.COLOR_LAB2BGR)


def auto_white_balance(image: np.ndarray) -> np.ndarray:
    """Otomatik beyaz dengesi"""
    result = cv2.cvtColor(image, cv2.COLOR_BGR2LAB)
    avg_a = np.average(result[:, :, 1])
    avg_b = np.average(result[:, :, 2])
    
    result[:, :, 1] = result[:, :, 1] - ((avg_a - 128) * (result[:, :, 0] / 255.0) * 1.1)
    result[:, :, 2] = result[:, :, 2] - ((avg_b - 128) * (result[:, :, 0] / 255.0) * 1.1)
    
    return cv2.cvtColor(result, cv2.COLOR_LAB2BGR)


# Kullanım
img = cv2.imread('photo.jpg')

# HSV'ye çevir
hsv = convert_color_space(img, 'hsv')

# Sıcak ton
warm = adjust_color_temperature(img, 0.5)

# Otomatik beyaz dengesi
balanced = auto_white_balance(img)
```

### Kenar Tespiti ve Kontur Bulma

```python
import cv2
import numpy as np
from typing import List, Tuple

def detect_edges_canny(
    image: np.ndarray,
    threshold1: int = 100,
    threshold2: int = 200
) -> np.ndarray:
    """
    Canny kenar tespiti
    
    Args:
        threshold1: Alt eşik
        threshold2: Üst eşik
    """
    # Gri tonlamaya çevir
    gray = cv2.cvtColor(image, cv2.COLOR_BGR2GRAY)
    
    # Gürültü azaltma
    blurred = cv2.GaussianBlur(gray, (5, 5), 0)
    
    # Canny kenar tespiti
    edges = cv2.Canny(blurred, threshold1, threshold2)
    
    return edges


def find_contours(image: np.ndarray) -> List[np.ndarray]:
    """
    Kontur bulma
    
    Returns:
        Kontur listesi
    """
    # Gri tonlama ve eşikleme
    gray = cv2.cvtColor(image, cv2.COLOR_BGR2GRAY)
    _, binary = cv2.threshold(gray, 127, 255, cv2.THRESH_BINARY)
    
    # Konturları bul
    contours, hierarchy = cv2.findContours(
        binary,
        cv2.RETR_EXTERNAL,
        cv2.CHAIN_APPROX_SIMPLE
    )
    
    return contours


def draw_contours(
    image: np.ndarray,
    contours: List[np.ndarray],
    color: Tuple[int, int, int] = (0, 255, 0),
    thickness: int = 2
) -> np.ndarray:
    """Konturları çiz"""
    result = image.copy()
    cv2.drawContours(result, contours, -1, color, thickness)
    return result


def detect_shapes(image: np.ndarray) -> dict:
    """
    Temel şekil tespiti
    
    Returns:
        Tespit edilen şekillerin sayıları
    """
    contours = find_contours(image)
    shapes = {'triangle': 0, 'rectangle': 0, 'circle': 0, 'other': 0}
    
    for contour in contours:
        # Konturu yaklaşık poligona dönüştür
        perimeter = cv2.arcLength(contour, True)
        approx = cv2.approxPolyDP(contour, 0.04 * perimeter, True)
        
        # Köşe sayısına göre şekil belirle
        vertices = len(approx)
        
        if vertices == 3:
            shapes['triangle'] += 1
        elif vertices == 4:
            shapes['rectangle'] += 1
        elif vertices > 4:
            shapes['circle'] += 1
        else:
            shapes['other'] += 1
    
    return shapes


# Kullanım
img = cv2.imread('objects.jpg')

# Kenar tespiti
edges = detect_edges_canny(img)

# Kontur bulma ve çizme
contours = find_contours(img)
with_contours = draw_contours(img, contours)

# Şekil tespiti
shapes = detect_shapes(img)
print(f"Detected shapes: {shapes}")
```

### Yüz Tespiti

```python
import cv2
import numpy as np
from typing import List, Tuple

def detect_faces(
    image_path: str,
    scaleFactor: float = 1.1,
    minNeighbors: int = 5
) -> Tuple[np.ndarray, List[Tuple[int, int, int, int]]]:
    """
    Haar Cascade ile yüz tespiti
    
    Args:
        image_path: Görüntü yolu
        scaleFactor: Ölçeklendirme faktörü
        minNeighbors: Minimum komşu sayısı
    
    Returns:
        (görüntü, yüz koordinatları listesi)
    """
    # Görüntüyü yükle
    img = cv2.imread(image_path)
    gray = cv2.cvtColor(img, cv2.COLOR_BGR2GRAY)
    
    # Haar Cascade yükle
    face_cascade = cv2.CascadeClassifier(
        cv2.data.haarcascades + 'haarcascade_frontalface_default.xml'
    )
    
    # Yüzleri tespit et
    faces = face_cascade.detectMultiScale(
        gray,
        scaleFactor=scaleFactor,
        minNeighbors=minNeighbors,
        minSize=(30, 30)
    )
    
    return img, faces


def draw_face_boxes(
    image: np.ndarray,
    faces: List[Tuple[int, int, int, int]],
    color: Tuple[int, int, int] = (0, 255, 0),
    thickness: int = 2
) -> np.ndarray:
    """Yüz etrafına kutu çiz"""
    result = image.copy()
    
    for (x, y, w, h) in faces:
        cv2.rectangle(result, (x, y), (x+w, y+h), color, thickness)
        
        # Yüz numarası ekle
        cv2.putText(
            result,
            'Face',
            (x, y-10),
            cv2.FONT_HERSHEY_SIMPLEX,
            0.5,
            color,
            thickness
        )
    
    return result


def blur_faces(
    image: np.ndarray,
    faces: List[Tuple[int, int, int, int]],
    blur_amount: int = 25
) -> np.ndarray:
    """Yüzleri bulanıklaştır (gizlilik için)"""
    result = image.copy()
    
    for (x, y, w, h) in faces:
        # Yüz bölgesini al
        face_region = result[y:y+h, x:x+w]
        
        # Bulanıklaştır
        blurred_face = cv2.GaussianBlur(face_region, (blur_amount, blur_amount), 30)
        
        # Geri yerleştir
        result[y:y+h, x:x+w] = blurred_face
    
    return result


def extract_faces(
    image: np.ndarray,
    faces: List[Tuple[int, int, int, int]],
    padding: int = 20
) -> List[np.ndarray]:
    """Tespit edilen yüzleri ayrı görüntüler olarak çıkar"""
    face_images = []
    
    for (x, y, w, h) in faces:
        # Padding ekle
        x_start = max(0, x - padding)
        y_start = max(0, y - padding)
        x_end = min(image.shape[1], x + w + padding)
        y_end = min(image.shape[0], y + h + padding)
        
        # Yüz bölgesini kes
        face_img = image[y_start:y_end, x_start:x_end]
        face_images.append(face_img)
    
    return face_images


# Kullanım örnekleri
img, faces = detect_faces('group_photo.jpg')
print(f"Detected {len(faces)} faces")

# Yüz etrafına kutu çiz
with_boxes = draw_face_boxes(img, faces)
cv2.imwrite('faces_detected.jpg', with_boxes)

# Yüzleri bulanıklaştır
blurred = blur_faces(img, faces, blur_amount=35)
cv2.imwrite('faces_blurred.jpg', blurred)

# Yüzleri çıkar ve kaydet
face_images = extract_faces(img, faces)
for i, face in enumerate(face_images):
    cv2.imwrite(f'face_{i}.jpg', face)
```

## Toplu Görüntü İşleme

Web uygulamalarında ve otomasyonlarda sıkça ihtiyaç duyulan toplu işleme örnekleri:

```python
import os
from pathlib import Path
from PIL import Image
from typing import List, Callable
from concurrent.futures import ThreadPoolExecutor, as_completed
from tqdm import tqdm

class BatchImageProcessor:
    """Toplu görüntü işleme sınıfı"""
    
    def __init__(self, input_dir: str, output_dir: str):
        self.input_dir = Path(input_dir)
        self.output_dir = Path(output_dir)
        
        # Çıktı dizinini oluştur
        self.output_dir.mkdir(parents=True, exist_ok=True)
        
        # Desteklenen formatlar
        self.supported_formats = {'.jpg', '.jpeg', '.png', '.gif', '.bmp', '.tiff'}
    
    def get_image_files(self) -> List[Path]:
        """Dizindeki tüm görüntü dosyalarını bul"""
        image_files = []
        
        for file_path in self.input_dir.rglob('*'):
            if file_path.suffix.lower() in self.supported_formats:
                image_files.append(file_path)
        
        return image_files
    
    def process_single(
        self,
        file_path: Path,
        process_func: Callable,
        **kwargs
    ) -> bool:
        """Tek bir görüntüyü işle"""
        try:
            # Görüntüyü yükle
            img = Image.open(file_path)
            
            # İşleme fonksiyonunu uygula
            processed = process_func(img, **kwargs)
            
            # Çıktı yolunu hesapla (yapıyı koru)
            relative_path = file_path.relative_to(self.input_dir)
            output_path = self.output_dir / relative_path
            
            # Çıktı dizinini oluştur
            output_path.parent.mkdir(parents=True, exist_ok=True)
            
            # Kaydet
            processed.save(output_path, quality=90, optimize=True)
            
            return True
            
        except Exception as e:
            print(f"Error processing {file_path}: {e}")
            return False
    
    def process_batch(
        self,
        process_func: Callable,
        max_workers: int = 4,
        **kwargs
    ) -> dict:
        """
        Toplu işleme (paralel)
        
        Args:
            process_func: İşleme fonksiyonu (image -> processed_image)
            max_workers: Maksimum thread sayısı
            **kwargs: İşleme fonksiyonuna geçilecek parametreler
        
        Returns:
            İşlem istatistikleri
        """
        image_files = self.get_image_files()
        total = len(image_files)
        
        print(f"Found {total} images to process")
        
        success_count = 0
        failed_count = 0
        
        # ThreadPoolExecutor ile paralel işleme
        with ThreadPoolExecutor(max_workers=max_workers) as executor:
            # Future'ları başlat
            futures = {
                executor.submit(self.process_single, file_path, process_func, **kwargs): file_path
                for file_path in image_files
            }
            
            # Progress bar ile takip et
            with tqdm(total=total, desc="Processing") as pbar:
                for future in as_completed(futures):
                    if future.result():
                        success_count += 1
                    else:
                        failed_count += 1
                    pbar.update(1)
        
        return {
            'total': total,
            'success': success_count,
            'failed': failed_count
        }


# İşleme fonksiyonları tanımla
def resize_for_web(image: Image.Image, max_width: int = 1200) -> Image.Image:
    """Web için optimize boyutlandırma"""
    if image.width > max_width:
        aspect = image.height / image.width
        new_height = int(max_width * aspect)
        return image.resize((max_width, new_height), Image.Resampling.LANCZOS)
    return image


def create_thumbnail(image: Image.Image, size: tuple = (300, 300)) -> Image.Image:
    """Thumbnail oluştur"""
    img = image.copy()
    img.thumbnail(size, Image.Resampling.LANCZOS)
    return img


def add_company_watermark(image: Image.Image, opacity: int = 128) -> Image.Image:
    """Şirket filigranı ekle"""
    from PIL import ImageDraw, ImageFont
    
    img = image.convert('RGBA')
    txt_layer = Image.new('RGBA', img.size, (255, 255, 255, 0))
    draw = ImageDraw.Draw(txt_layer)
    
    text = "© 2025 Company"
    font_size = int(img.width * 0.03)
    
    try:
        font = ImageFont.truetype('/usr/share/fonts/truetype/dejavu/DejaVuSans-Bold.ttf', font_size)
    except:
        font = ImageFont.load_default()
    
    bbox = draw.textbbox((0, 0), text, font=font)
    text_width = bbox[2] - bbox[0]
    text_height = bbox[3] - bbox[1]
    
    x = img.width - text_width - 20
    y = img.height - text_height - 20
    
    draw.text((x, y), text, fill=(255, 255, 255, opacity), font=font)
    
    watermarked = Image.alpha_composite(img, txt_layer)
    return watermarked.convert('RGB')


# Kullanım örnekleri

# Web için boyutlandırma
processor = BatchImageProcessor('raw_photos/', 'web_optimized/')
stats = processor.process_batch(resize_for_web, max_width=1920, max_workers=8)
print(f"Processed: {stats['success']}/{stats['total']}")

# Thumbnail oluşturma
thumb_processor = BatchImageProcessor('products/', 'thumbnails/')
thumb_processor.process_batch(create_thumbnail, size=(400, 400))

# Filigran ekleme
watermark_processor = BatchImageProcessor('originals/', 'watermarked/')
watermark_processor.process_batch(add_company_watermark, opacity=150)
```

## Best Practices ve Performans İpuçları

### 1. Bellek Yönetimi

```python
from PIL import Image
import gc

def process_large_images(image_paths: list):
    """Büyük görüntülerle çalışırken bellek yönetimi"""
    
    for path in image_paths:
        # Görüntüyü yükle
        img = Image.open(path)
        
        # İşlemleri yap
        processed = img.resize((800, 600), Image.Resampling.LANCZOS)
        processed.save(f'processed_{path}')
        
        # Bellekten temizle
        img.close()
        processed.close()
        
        # Garbage collection zorla (büyük dosyalar için)
        gc.collect()
```

### 2. Lazy Loading

```python
from PIL import Image

# ❌ YANLIŞ: Tüm görüntü belleğe yüklenir
img = Image.open('huge_image.jpg')
width, height = img.size

# ✅ DOĞRU: Sadece metadata okunur
with Image.open('huge_image.jpg') as img:
    width, height = img.size
    format = img.format
    # Görüntü verisi henüz yüklenmedi
```

### 3. Format Seçimi

```python
def optimize_image_format(image: Image.Image, has_transparency: bool = False):
    """Uygun format seçimi"""
    
    if has_transparency:
        # Transparanlık varsa PNG kullan
        image.save('output.png', 'PNG', optimize=True, compress_level=9)
    else:
        # Transparanlık yoksa JPEG daha iyi
        image.save('output.jpg', 'JPEG', quality=85, optimize=True)
```

### 4. Thumbnail Strategy

```python
from PIL import Image

# ❌ YANLIŞ: Büyük görüntüyü yükleyip küçült
img = Image.open('huge.jpg')  # 20MB
img.thumbnail((200, 200))

# ✅ DOĞRU: Draft mode kullan
img = Image.open('huge.jpg')
img.draft('RGB', (200, 200))  # Hızlı ve hafif
img.thumbnail((200, 200))
```

## Sonuç

Python ile görüntü işleme, Pillow ve OpenCV kütüphaneleri sayesinde hem basit hem de gelişmiş seviyede mümkündür. Pillow, web uygulamaları ve temel manipülasyonlar için ideal bir seçimken, OpenCV computer vision ve gelişmiş analiz gerektiren projelerde tercih edilmelidir.

Bu yazıda ele aldığımız konular:

1. **Pillow Temel İşlemler**: Yükleme, kaydetme, boyutlandırma, kırpma
2. **Görüntü Filtreleri**: Blur, sharpen, brightness, contrast ayarları
3. **Filigran Ekleme**: Metin ve görüntü watermark'ları
4. **OpenCV İleri Seviye**: Renk uzayları, kenar tespiti, yüz tanıma
5. **Toplu İşleme**: Paralel işleme, optimizasyon stratejileri
6. **Best Practices**: Bellek yönetimi, format seçimi, performans

Görüntü işleme projeleri geliştirirken dikkat edilmesi gereken en önemli noktalar performans, bellek kullanımı ve dosya formatı seçimidir. Production ortamında mutlaka error handling, logging ve monitoring implementasyonu yapın.

## Kaynaklar

- [Pillow Documentation](https://pillow.readthedocs.io/)
- [OpenCV Python Tutorial](https://docs.opencv.org/master/d6/d00/tutorial_py_root.html)
- [Real Python - Image Processing](https://realpython.com/image-processing-with-the-python-pillow-library/)
- [OpenCV Face Detection](https://docs.opencv.org/master/db/d28/tutorial_cascade_classifier.html)
- [NumPy for Image Processing](https://numpy.org/doc/stable/user/quickstart.html)
