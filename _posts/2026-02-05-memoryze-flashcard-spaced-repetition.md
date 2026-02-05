---
title: "Memoryze: Kendi Sınırlarını Çizdiğin, Özgür Bir Dil Öğrenme Platformu"
description: "Duolingo'nun baykuşundan ve Quizlet'in sınırlamalarından sıkıldım, kendi açık kaynak Flashcard uygulamamı yazdım. React, Supabase ve Framer Motion ile geliştirdiğim Memoryze'ın teknik ve kişisel hikayesi."
date: "2026-02-05 14:00:00 +0300"
categories: [Side Projects, Web Development]
tags: [react, vite, supabase, tailwind, framer-motion, open-source, i18n, spaced-repetition]
image:
  path: /assets/img/posts/memoryze/logo.png
  alt: "Memoryze Logo - Beyin ve Devre Kartı Tasarımı"
---

Dil öğrenme serüveni herkes için benzer başlar. Önce büyük bir hevesle Duolingo indirilir, o yeşil baykuşun "agresif" hatırlatmalarıyla birkaç hafta geçirilir. Sonra kelime ezberlemek için Quizlet veya Anki denenir.

Ben de bu yollardan geçtim. Ama bir noktada hep duvara tosladım:
- **Duolingo:** Fazla oyunlaştırılmış, benim istediğim kelimelere odaklanmıyor.
- **Quizlet:** Güzel ama premium özellikleri olmadan kısıtlı.
- **Anki:** Algoritması harika ama arayüzü 90'lardan kalma.

"Neden hem modern görünen, hem de tamamen benim kontrolümde olan bir araç yok?" diye sorduğum o an, **Memoryze** fikri doğdu.

Ve bugün, bu projeyi sadece kendim için değil, herkes için açık kaynak olarak paylaşıyorum.

[🚀 Uygulamayı Dene](https://furkankoykiran.com.tr/Memoryze) | [💻 GitHub Kodlarını İncele](https://github.com/furkankoykiran/Memoryze)

## Felsefe: "Sadece Öğren, Gerisini Bana Bırak"

Memoryze'ın temel felsefesi **sadelik ve verimlilik**.

Sizi puanlarla, liglerle, reklamlarla boğmak istemedim. Odaklanmanız gereken tek bir şey var: **Kelimeler.**

Arka planda çalışan **SM2 (SuperMemo-2)** algoritması, beyninizin unutma eğrisini (forgetting curve) hesaplıyor.
- Bir kelimeye "Kolay" derseniz, onu günler sonra soruyor.
- "Zorlandım" derseniz, o kelimeyi beyninize kazımak için aynı oturumda defalarca karşınıza çıkarıyor.

Bu, rastgele kart çevirmekten çok daha öte bir şey. Bu, bilimsel bir öğrenme yöntemi.

## Teknolojinin Mutfağı: Modern Stack Seçimi

Bir "Side Project" geliştiriyorsanız, en büyük lüksünüz teknolojiyi seçme özgürlüğüdür. Ben de 2026 standartlarında, geliştirici deneyimi (DX) en yüksek araçları seçtim.

![Memoryze Dashboard](/assets/img/posts/memoryze/logo.png)
*Memoryze, minimalist ve karanlık mod odaklı bir tasarıma sahip.*

### 1. Hızın Adı: React & Vite
Create React App devri bitti. Vite'ın sunduğu nanoraniye hızındaki HMR (Hot Module Replacement) ile geliştirmek büyük keyif. Uygulama, gereksiz yüklerden arındırılmış, SPA (Single Page Application) mimarisinde çalışıyor.

### 2. Backend Olmadan Backend: Supabase
Sunucu yönetmek, veritabanı kurmak, Docker ile uğraşmak... Bunlar ana odağınızı dağıtan şeyler.
PostgreSQL tabanlı **Supabase** ile:
- **Auth:** Google ve Email girişi dakikalar içinde hazır.
- **Database:** İlişkisel veritabanı konforu (Cards -> Decks -> Users).
- **Row Level Security (RLS):** "Benim kartımı sadece ben görebilirim" kuralını veritabanı seviyesinde yazdım. API güvenliğiyle uğraşmaya gerek kalmadı.

### 3. Görsel Büyü: Framer Motion
İyi bir uygulamayı harika olandan ayıran şey detaylardır. Bir butona bastığınızdaki o hafif tepki, sayfa geçişlerindeki yumuşaklık...

Memoryze'da CSS yerine **Framer Motion** kullandım. Özellikle kart çevirme (flip) animasyonu, göründüğünden çok daha karmaşık bir mühendislik içeriyor.

## Teknik Bir Zorluk: 3D Kart Döndürme (The Mirror Effect)

Projenin en can alıcı noktası "Flashcard" deneyimi. Kullanıcı karta dokunduğunda, kartın 3D uzayda dönüp arkasındaki cevabı göstermesi gerekiyor.

İlk başta basit CSS ile başladım:
```css
.card {
  transform: rotateY(180deg);
  backface-visibility: hidden;
}
```

Ama tarayıcılar (özellikle mobil Safari ve bazı Chrome versiyonları) bize acımadı. Kart döndüğünde:
1. Yazılar ters (ayna görüntüsü) çıkıyordu.
2. Arka yüz bazen hiç render olmuyordu (boş ekran).
3. Z-index karışıyor, butonlar çalışmıyordu.

### Çözüm: Hibrit Yaklaşım

CSS'in yetersiz kaldığı yerde JavaScript'in (Framer Motion) gücünü kullandım.

Sadece CSS'e güvenmek yerine, animasyonun her karesini (frame) dinleyen bir yapı kurdum.

```tsx
// Rotation değerini bir MotionValue olarak takip et
const rotateY = useMotionValue(0);

// 90 derece (tam dik olduğu an) kritik eşik
// 0-90 derece arası -> Ön yüz opaklığı 1
// 90-180 derece arası -> Ön yüz opaklığı 0
const frontOpacity = useTransform(rotateY, [89, 90], [1, 0]);
const backOpacity = useTransform(rotateY, [89, 90], [0, 1]);
```

Bu kod ne yapıyor? Kart tam 90 dereceye geldiğinde (kullanıcıya sadece ince bir çizgi olarak göründüğü o milisaniyede), yazılımla ön yüzü yok edip arka yüzü var ediyoruz. Sonuç: Her cihazda kusursuz, pürüzsüz ve hatasız bir dönüş.

## Açık Kaynak ve Gelecek

Memoryze şu an V1 sürümünde. Ama aklımda büyük fikirler var:
- **Topluluk Desteleri:** Hazırladığınız harika bir "İspanyolca 101" destesini tek tıkla paylaşabilmek.
- **Seslendirme (TTS):** Karttaki kelimenin telaffuzunu dinleyebilmek.
- **İstatistikler:** Hangi gün kaç kelime öğrendiğinizi gösteren ısı haritaları (GitHub contribution grafiği gibi).

Ve en güzeli? Kodlar açık. İstediğiniz özelliği siz de ekleyebilirsiniz. PR'lara kapımız her zaman açık (Conflict çıkarsa da çözeriz, [tecrübeliyim](/posts/github-mcp-server-acik-kaynak-katki/)).

## Son Söz

Bir şeyi öğrenmenin en iyi yolu, onu inşa etmektir. Memoryze, benim için hem bir dil öğrenme aracı hem de modern web teknolojilerini denediğim bir oyun alanı oldu.

Umarım sizin de kelime hazinenize (veya GitHub yıldızlarınıza) bir katkısı olur.

Kodla ve kelimelerle kalın! 🚀

---

**BKZ:**
- [GitHub MCP Server'a Katkı Macerası](/posts/github-mcp-server-acik-kaynak-katki/)
- [Modern Web Arayüzü: CSS & JavaScript](/posts/modern-web-arayuzu-css-javascript/)
- [React Context Managers](/posts/python-context-managers-kaynak-yonetimi/)

![Furkan Köykıran](/assets/img/avatar.png)
*Furkan Köykıran - Senior Software Engineer*
