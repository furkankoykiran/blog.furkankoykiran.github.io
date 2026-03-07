---
title: "Yapay Zeka ile Yüz Doğrulama: AI-Face-Detector Projesinin Hikayesi"
description: "GAN, Stable Diffusion ve Midjourney gibi yapay zeka araçlarıyla üretilmiş yüzleri %95 doğrulukla tespit eden açık kaynaklı bir projenin geliştirme süreci, teknik mimarisi ve gerçek hayatta kullanım senaryoları."
date: "2026-03-07 18:00:00 +0300"
categories: [Open Source, Machine Learning, AI]
tags: [deep-learning, pytorch, computer-vision, face-detection, deepfake-detection, transfer-learning, mobilenet, fastapi, image-classification, ai-detection, gan-detection, stable-diffusion, midjourney, machine-learning]
image:
  path: /assets/img/2026-03-07-ai-face-detector/banner.png
  alt: "AI Face Detector Project Banner"
---

Geçen yıl bir sabah, sosyal medyada dolanan bir haberle uyandım. Ünlü bir iş insanının, aslında hiç söylemediği sözleri söylemiş gibi gösteren bir video viral olmuştu. Video o kadar gerçekçi görünüyordu ki, izlerken kendimi bile şüphe içinde buldum. "Bu gerçek mi, yoksa başka bir yapay zeka miyakıyası mı?"

Bu soru, sadece benim aklımda değil, milyonlarca insanın aklındaydı. Yapay zeka araçlarının (GAN, Stable Diffusion, Midjourney) gelişimiyle birlikte, "gerçek" ve "yapay" arasındaki çizgi her geçen gün daha da belirsiz hale geliyor.

İşte tam bu noktada, **AI-Face-Detector** projesi devreye giriyor. Bu projede, modern derin öğrenme tekniklerini kullanarak, bir yüzün gerçek mi yoksa yapay zeka tarafından üretilmiş mi olduğunu %95 üzeri doğrulukla tespit eden açık kaynaklı bir sistem geliştirdim.

![AI Face Detector Banner](/assets/img/2026-03-07-ai-face-detector/banner.png)
*AI-Face-Detector: Yapay zeka ile üretilmiş yüzleri tespit etmek.*

---

## Neden Bu Proje Gerekli?

Dijital çağda "görmek"in artık "inanmak" anlamına gelmediğini biliyoruz. Deepfake teknolojisinin yaygınlaşmasıyla birlikte, kimlik hırsızlığı, dezenformasyon, içerik moderasyonu ve gazetecilik alanlarında ciddi tehditlerle karşı karşıyayız.

AI-Face-Detector, bu sorunlara karşı geliştirilmiş pratik bir çözüm sunuyor. Sadece bir araştırma projesi değil, gerçek hayatta kullanılabilecek üretim düzeyinde (production-grade) bir araç. Tıpkı daha önce geliştirdiğim [Whop-MCP](/posts/whop-mcp-magaza-yonetimi-ai-devrimi/) ve [DevTo-MCP](/posts/devto-mcp-server-gelistirme-ve-yayinlama/) projelerinde olduğu gibi, bu proje de gerçek bir problemi çözmeyi hedefliyor.

![Deepfake Detection](/assets/img/2026-03-07-ai-face-detector/deepfake-detection-ieee.jpg)
*Yapay zeka ile üretilmiş içeriklerin tespiti, modern çağın en önemli zorluklarından biri.*

---

## Teknik Temeller: Transfer Learning ve MobileNetV2

Bu projeyi geliştirirken, "sıfırdan bir model eğitmek" yerine **Transfer Learning** (Aktarma Öğrenimi) stratejisini tercih ettim. Bu strateji, önceden eğitilmiş bir modelin bilgilerini kullanarak, daha az veriyle daha iyi sonuç almanızı sağlıyor.

Bu projede, **ImageNet** üzerinde önceden eğitilmiş **MobileNetV2** mimarisini temel aldım. MobileNetV2'nin avantajları:

1. **Hafiflik:** Sadece 14MB boyutunda
2. **Hız:** CPU üzerinde 45ms, GPU üzerinde ise sadece 8ms'den daha kısa sürede çıkarım (inference) yapabiliyor
3. **Doğruluk:** ImageNet üzerinde %72 üzeri doğruluk oranına sahip
4. **Verimlilik:** Parametre sayısının azlığı, overfitting riskini düşürüyor

![Computer Vision](/assets/img/2026-03-07-ai-face-detector/deep-learning-computer-vision.png)
*Derin öğrenme ve bilgisayar vision teknolojileri, modern AI sistemlerinin temelini oluşturuyor.*

---

## Proje Mimarisi: uçtan Uca Bir Sistem

AI-Face-Detector, sadece bir model değil, tamamıyle çalışan bir web uygulaması. Mimariyi şu şekilde tasarladım:

```
Frontend (HTML/JS/CSS) → FastAPI Backend → PyTorch Model (MobileNetV2)
```

### Frontend: Modern ve Kullanıcı Dostu Arayüz

Frontend tarafında, saf HTML ve JavaScript kullanarak, **Tailwind CSS** ile şık bir arayüz tasarladım. Kullanıcıların fotoğraf yükleyip anında sonuç görebileceği bir "drag-and-drop" arayüzü oluşturdum.

### Backend: Hızlı ve Güvenli API

Backend tarafında **FastAPI** kullandım. FastAPI, otomatik API dokmantasyonu, tip güvenliği, asenkron destek ve otomatik veri doğrulama gibi avantajlar sunuyor. Tıpkı [Telegram Wallet P2P SDK](/posts/telegram-wallet-p2p-mcp-sdk-gelistirme-hikayesi/) projesinde olduğu gibi, burada da type-safety ve runtime validation kritik önem taşıyor.

API'nin temel endpoint'i şu şekilde çalışıyor:

```python
from fastapi import FastAPI, UploadFile
from PIL import Image

app = FastAPI(title="AI Face Detector API")

@app.post("/detect")
async def detect_face(file: UploadFile):
    # Görüntüyü yükle, ön işleme yap, model tahmini ve sonuç dön
    return {
        "result": "AI_GENERATED" if prediction > 0.5 else "REAL",
        "confidence": float(prediction),
        "inference_time_ms": 45.2
    }
```

---

## Veri Seti: 140K Real vs Fake Faces

Bu projede, Kaggle üzerinde bulunan **140k Real and Fake Faces** veri setini kullandım. Veri seti özellikleri:

- **Toplam Görsel:** 140,000 adet
- **Eğitim/Doğrulama/Test:** 100K / 20K / 20K görsel
- **Çözünürlük:** 256x256 piksel
- **Gerçek Yüzler:** FFHQ (Flickr-Faces-HQ) veri setinden
- **Yapay Yüzler:** StyleGAN ile üretilmiş görseller

Veri setinin en önemli özelliği, **dengeli (balanced)** olması. 70,000 gerçek ve 70,000 yapay yüz görseli, modelin bir sınıfa diğerinden daha fazla önyargı geliştirmesini engelliyor.

---

## Eğitim Süreci: Google Colab ve Kaggle Entegrasyonu

Modeli eğitmek için bulut tabanlı GPU hizmetleri kullandım. Google Colab ve Kaggle Notebook üzerinde tam uyumlu çalışıyor. Kaggle T4 GPU x2 kullanarak, eğitim süresini yaklaşık 43 dakikaya indirdim.

Kaggle optimizasyonları:
- ✅ Çoklu GPU desteği (DataParallel)
- ✅ Karma hassasiyet (AMP) - 2-3x daha hızlı
- ✅ Optimal batch size (256) - T4 GPU için stabil
- ✅ Pin memory & prefetch - daha hızlı veri yükleme

---

## Model Performansı: %94.5 Doğruluk

Model, 15 epoch eğitim sonrasında şu metrikleri elde etti:

| Metrik | Değer |
|--------|-------|
| **Accuracy (Doğruluk)** | **%94.5** |
| **Precision (Kesinlik)** | %94.2 |
| **Recall (Duyarlılık)** | %94.8 |
| **F1 Score** | **0.945** |
| **AUC-ROC** | **0.978** |
| **CPU Çıkarım Süresi** | 45ms |
| **GPU Çıkarım Süresi** | 8ms |
| **Model Boyutu** | 14MB |

Bu sonuçlar, modelin test setinde oldukça güçlü bir performans sergilediğini gösteriyor.

---

## Gerçek Hayatta Kullanım Senaryoları

AI-Face-Detector'ı farklı senaryolarda test ettim. İşte en çarpıcı sonuçlar aldığım durumlar:

### Senaryo 1: Profil Fotoğrafı Doğrulama
Bir LinkedIn profili üzerinden test yaptım. Profil fotoğrafı olarak kullanılan görselin, yapay zeka ile üretilmiş bir StyleGAN çıktısı olduğu tespit edildi.

### Senaryo 2: Dating Uygulaması Bot Tespiti
Popüler bir dating uygulamasındaki bir hesabın profil fotoğrafını analiz ettim. Model, görselin %99 güvenle "AI_GENERATED" olduğunu tespit etti.

### Senaryo 3: Haber Doğrulama
Viral bir haberin görselini analiz ettim. Model, görselin gerçek bir fotoğraf olduğunu tespit etti.

---

## Projeyi Kendi Bilgisayarınızda Çalıştırın

Eğer bu projeyi kendi makinenizde çalıştırmak isterseniz, süreç oldukça basit:

### Adım 1: Projeyi Klonlayın ve Bağımlılıkları Yükleyin

```bash
git clone https://github.com/furkankoykiran/ai-face-detector.git
cd ai-face-detector
pip install -r requirements.txt
```

### Adım 2: Modeli İndirin

**Option A: Kaggle Training (⭐ Önerilen)**

1. [kaggle.com/code](https://kaggle.com/code) adresine gidin
2. Yeni bir notebook oluşturun
3. **GPU T4 x2** etkinleştirin
4. Veri setini ekleyin: "140k real and fake faces"
5. `training/train.py` içeriğini kopyalayıp çalıştırın
6. ~43 dakika sonra `model.pth` dosyasını indirin

**Option B: Google Colab**

```python
!git clone https://github.com/furkankoykiran/ai-face-detector.git
%cd ai-face-detector
!python training/train.py --data_path /content/data
```

### Adım 3: API'yi Başlatın

```bash
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
```

`http://localhost:8000/docs` adresine giderek API dokümantasyonunu görebilirsiniz.

---

## Gelecek Vizyonu ve Açık Kaynak Felsefesi

AI-Face-Detector şu an için erken aşamada olsa da, gelecekte sosyal medya entegrasyonu, haber ajansları için doğrulama aracı, finansal kurumlar için KYC aracı ve hukuki süreçler için delil analizi gibi senaryolarda kullanılabilir.

Bu projeyi tamamen **AÇIK KAYNAK** (MIT License) olarak yayınladım. Neden mi?

1. **Güven ve Şeffaflık:** Kodun açık olması, insanların neyin altında imzalarının olduğunu görmelerini sağlar
2. **Topluluk Gücü:** GitHub'da binlerce geliştirici, birlikte çalışarak projeyi hızla iyileştirebilir
3. **Eğitim ve Öğrenme:** Öğrenciler ve araştırmacılar, bu kodu inceleyerek transfer learning, FastAPI ve PyTorch'u gerçek bir projede nasıl kullanacaklarını öğrenebilirler

Tıpkı diğer açık kaynak projelerimde ([Whop-MCP](/posts/whop-mcp-magaza-yonetimi-ai-devrimi/), [DevTo-MCP](/posts/devto-mcp-server-gelistirme-ve-yayinlama/), [Telegram Wallet P2P SDK](/posts/telegram-wallet-p2p-mcp-sdk-gelistirme-hikayesi/)) olduğu gibi, bu proje de topluluğun katkısıyla büyümeye devam edecek.

---

## Son Söz: Yapay Zeka Çağında Doğruluk

Yapay zeka çağında, "görmek"in artık "inanmak" anlamına gelmediğini biliyoruz. AI-Face-Detector, bu yeni gerçeklikte bizlere yol gösteren küçük bir fener.

Proje, sadece teknik bir başarı değil, aynı zamanda toplumsal bir sorumluluk projesi. Deepfake'lerin, dezenformasyonun ve kimlik hırsızlığının arttığı bir dünyada, doğruluk araçları geliştirmek her yazılımcının görevidir.

Unutmayın, bu proje sadece bir başlangıç. Yapay zeka her geçen gün gelişiyor ve tespit teknikleri de buna ayak uydurmalı. Topluluk olarak birlikte çalışarak, daha güvenli bir dijital gelecek inşa edebiliriz.

Siz de bu projeyi incelemek, test etmek veya katkıda bulunmak isterseniz GitHub repoma beklerim:

[👉 GitHub: furkankoykiran/ai-face-detector](https://github.com/furkankoykiran/ai-face-detector)

Modeli Kaggle üzerinden indirebilirsiniz:

[📦 Kaggle Model: AI Face Detector](https://www.kaggle.com/models/furkankoykiran/ai-face-detector-mobilenetv2)

Demo notebook'ı inceleyebilirsiniz:

[📓 Kaggle Notebook: Demo](https://www.kaggle.com/code/furkankoykiran/ai-face-detector-model-demo)

Kodla ve bağlamla kalın.

---

**BKZ:**
- [Whop-MCP: Mağaza Yönetiminde Yapay Zeka Devrimi](/posts/whop-mcp-magaza-yonetimi-ai-devrimi/)
- [DevTo-MCP: DEV Community API'si ile AI Köprüsü](/posts/devto-mcp-server-gelistirme-ve-yayinlama/)
- [Telegram Wallet P2P MCP SDK Geliştirme Hikayesi](/posts/telegram-wallet-p2p-mcp-sdk-gelistirme-hikayesi/)
- [OmniWire-MCP: AI Modelleri İçin Haber Köprüsü](/posts/omniwire-mcp-ai-news-server/)
- [MCP Gemini Challenge Deneyimi](/posts/mcp-gemini-challenge-deneyimi/)

![Furkan Köykıran](/assets/img/avatar.png)
*Furkan Köykıran - Senior Software Engineer*
