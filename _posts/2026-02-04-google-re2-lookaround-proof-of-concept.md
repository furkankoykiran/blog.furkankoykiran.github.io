---
title: "Google'ın Regex Kütüphanesine Yeni Özellik Eklemek: Bir Açık Kaynak Hikayesi"
description: "Google'ın RE2 projesine nasıl katkıda bulundum? İsviçre'den bir üniversiteyle aynı problemi çözerken tanıştım. İmkansız denilen bir şeyi mümkün kılma hikayesi."
date: "2026-02-04 23:00:00 +0300"
categories: [DevOps, Open Source]
tags: [re2, regex, open-source, c++, google, algorithm, nfa, lookaround]
image:
  path: /assets/img/posts/re2-lookaround-implementation.png
  alt: "RE2 Lookaround Implementation Architecture"
---

Yazılım dünyasında "bu yapılamaz" denilen şeyler genellikle "henüz kimse yapmadı" anlamına gelir. Bu yazıda, Google'ın yaygın kullanılan bir kütüphanesine "imkansız" denilen bir özelliği ekleme girişimimi anlatacağım. Ve yolda İsviçre'den bir üniversiteyle tanıştım.

## Regex Nedir ve Neden Önemli?

Regex (Regular Expression), metin içinde arama ve eşleştirme yapmak için kullanılan bir kalıp dilidir. Günlük hayattan örnekler:

- E-posta adresinin geçerli olup olmadığını kontrol etmek
- Log dosyalarında hata mesajlarını bulmak
- "Şifre en az 8 karakter ve bir rakam içermeli" kuralını doğrulamak
- Bir metindeki tüm telefon numaralarını maskelemek

Örneğin, bir müşteri veri tabanında telefon numarası aramak istiyorsunuz:
```
0532-123-4567 veya (532) 123 45 67 veya +90 532 1234567
```

Normal bir arama ile bu üç formatı da bulmak için üç ayrı sorgu yazmanız gerekir. Regex ile tek satırda halledersiniz. Bu yüzden neredeyse her programlama dili ve araç regex desteği sunar.

![RE2 Kullanım Alanları](/assets/img/posts/re2-usage-ecosystem.png)
*Regex kütüphaneleri farklı sistemlerde yaygın olarak kullanılıyor*

## Google'ın RE2 Projesi

Regex motorları arasında Google'ın RE2 kütüphanesi özel bir yere sahip. Nedenini anlamak için önce bir güvenlik açığından bahsetmem gerekiyor.

### ReDoS Saldırısı Nedir?

Normal regex motorları bazı kalıplarla çok yavaşlayabilir. Basit bir örnek: `(a+)+$` kalıbını `aaaaaaaaaaaaaaaaaaaaaX` metninde aramak.

Bu işlem saniyeler, dakikalar, hatta saatler sürebilir. Kötü niyetli bir kullanıcı, özellikle hazırlanmış bir kalıpla web sitenizi çökertebilir. Buna **ReDoS (Regex Denial of Service)** saldırısı deniyor.

2019'da Stack Overflow bu saldırıyla 30 dakika çökmüştü. Cloudflare da benzer bir sorun yaşamıştı.

### RE2'nin Çözümü

RE2 bu sorunu kökünden çözüyor: Her zaman tahmin edilebilir sürede çalışıyor. Metin ne kadar uzun olursa olsun, işlem süresi orantılı artıyor, katlanarak değil.

Bu garantiyi sağlamak için RE2 bazı özelliklerden vazgeçiyor. Ama karşılığında:
- Google Sheets'te regex kullanabilirsiniz
- Prometheus sorguları güvenle çalışır
- Kullanıcı girdisi içeren sistemlerde güvenle regex kullanılabilir

## "Lookaround" Özelliği ve Eksikliği

İşte vazgeçilen özelliklerden biri: **Lookaround**.

### Ne İşe Yarıyor?

Lookaround, "şuna bak ama eşleştirmeye dahil etme" demek. Pratik örnekler:

**1. Şifre doğrulama:**
"Şifrede en az bir rakam olmalı" kuralını kontrol etmek istiyorsunuz, ama rakamın kendisini değil, şifrenin geçerli olup olmadığını öğrenmek istiyorsunuz.

**2. Loglarda filtre:**
"ERROR" içeren satırları bulmak istiyorsunuz, ama sadece önünde "DEBUG" yazmıyorsa.

**3. Veri temizleme:**
Bir metindeki "password" kelimelerini maskelemek istiyorsunuz, ama "old_password" gibi değişken isimlerini değil.

```
(?<!old_)password
```

Bu kalıp, `new_password` içindeki "password"ü bulur, ama `old_password` içindekini bulmaz.

### Issue #156: 61 Thumbs Up, 0 Solution

Projenin issue tracker'ını incelerken Issue #156'ya rastladım. 2017'de açılmış, 61 kişi "+1" vermiş, ama kapatılmış. 

Maintainer açıkça belirtmiş: "RE2 does not support constructs that require backtracking."

Yani "güvenlik garantimizi bozmadan bu özelliği ekleyemeyiz."

![Akademik Araştırma](/assets/img/posts/regex-academic-paper-reference.jpg)
*Akademik araştırmalar pratik problemlere çözüm sunabiliyor*

## "İmkansız" Denen Çözümü Aramak

Ben meraklandım: Gerçekten imkansız mı, yoksa henüz kimse denemedi mi?

Akademik makaleler araştırmaya başladım. Aylarca okuduktan sonra 2024'te yayınlanan bir makale dikkatimi çekti: JavaScript regex'lerinin güvenli bir şekilde nasıl çalıştırılabileceğini gösteriyordu.

Temel fikir şuydu: Lookaround için geri sarma (backtracking) zorunlu değil. Doğru algoritmayla, güvenlik garantilerini koruyarak lookaround desteklenebilir.

Bu akademik bir kanıttı. Şimdi pratik kısmı geliyordu.

## Fork ve Deneysel Geliştirme

RE2'nin kaynak kodunu kendi hesabıma kopyaladım (fork) ve çalışmaya başladım. C++ ile yazılmış, karmaşık bir kod tabanı. Ama adım adım anlamaya başladım.

Hedefim basitti: Lookaround özelliğini eklemek, ama güvenlik garantilerini bozmamak.

### Yaklaşımım

1. Her lookaround için ayrı bir mini-program oluştur
2. Bu mini-programları ana aramayla birlikte çalıştır
3. Geriye bakma mesafesini sınırla (255 karakter)

255 karakter sınırı neden? Çünkü pratikte çoğu kullanım senaryosunu kapsıyor ve sistemi güvenli tutuyor.

Birkaç hafta sonra çalışan bir prototipim vardı:

```cpp
// "test" kelimesini bul, önünde rakam yoksa
RE2 re("(?<!\\d)test");
RE2::PartialMatch("hello test", re);  // ✓ bulur
RE2::PartialMatch("123test", re);     // ✗ bulmaz

// "world" kelimesini bul, önünde "hello " varsa
RE2 re2("(?<=hello )world");
RE2::PartialMatch("hello world", re2);   // ✓ bulur
RE2::PartialMatch("goodbye world", re2); // ✗ bulmaz
```

![Test Sonuçları](/assets/img/posts/re2-lookaround-test-results.png)
*Testlerin başarıyla geçmesi her zaman güzel bir his*

## Issue Açtım, İlginç Bir Şey Oldu

Çalışan prototipimi toplulukla paylaşmak için Issue #585'i açtım. Detaylı bir açıklama yazdım: Ne yaptım, nasıl yaptım, hangi kısıtlamalar var.

Birkaç hafta sonra beklemediğim bir yorum geldi. İsviçre'deki EPFL üniversitesinden bir araştırmacı yazdı:

> "İlginç bir şeyle paylaşmak istiyorum - biz de aynı problemi çözdük, ama sınırsız lookbehind'ı destekleyen bir yöntemle."

Meğer aynı problemi, aynı dönemde, farklı yaklaşımlarla çözmeye çalışan iki grup varmış! Bir tarafta ben, diğer tarafta İsviçre'nin en prestijli üniversitelerinden birinin araştırma ekibi.

![Karşılaştırma](/assets/img/posts/epfl-re2-comparison-diagram.png)
*Farklı algoritma yaklaşımlarının karşılaştırması*

## İki Yaklaşımın Karşılaştırması

| Özellik | Benim Yaklaşımım | EPFL Yaklaşımı |
|---------|-----------------|----------------|
| Geriye bakma limiti | 255 karakter | Sınırsız |
| İleriye bakma | ✅ Destekliyor | ❌ Henüz yok |
| Kod değişikliği | ~280 satır | ~275 satır |
| Yaklaşım | Pragmatik | Akademik |

İkisi de güvenlik garantilerini koruyor. İkisi de teorik olarak doğru. Sadece farklı öncelikler.

## Bu Deneyimden Ne Öğrendim?

### 1. "İmkansız" Çoğu Zaman "Zor" Demek

Yıllardır "yapılamaz" denilen bir şey, doğru yaklaşımla mümkün olabilir. 61 kişi istemesine rağmen 7 yıldır eklenmeyen bir özellik, birkaç haftalık çalışmayla prototip haline geldi.

### 2. Dünyanın Bir Köşesinde Birisi Aynı Şeyi Düşünüyor

Issue açmak, sadece çözüm paylaşmak değil. Aynı problem üzerinde çalışan insanlarla tanışmak için de bir fırsat. Ben İstanbul'da, onlar İsviçre'de - ama aynı soruna farklı çözümler ürettik.

### 3. Sonuç Her Zaman "Merge" Değil

PR'ım henüz merge edilmedi. Belki hiç edilmeyecek. Ama bu yolculukta kazandığım deneyimler kalıcı:
- Karmaşık bir kod tabanını analiz etme
- Akademik makale okuma ve pratiğe çevirme
- Açık kaynak toplulukla iletişim kurma

![Açık Kaynak Katkı Süreci](/assets/img/posts/opensource-contribution-diagram.png)
*Açık kaynak katkı süreci*

### 4. Küçük Adımlarla Başlayın

İlk açık kaynak katkınız Google'ın kritik altyapısını değiştirmek olmak zorunda değil. Bir dokümantasyon hatası düzeltmek, küçük bir bug fix göndermek veya test eklemek - hepsi değerli katkılar.

## Sonuç

Issue #585 hala açık. Tartışma devam ediyor. Belki bir gün merge edilir, belki edilmez. Ama "imkansız" denilen bir şeyin aslında yapılabilir olduğunu gösterdik.

Eğer siz de bir projeye bakıp "bu özellik neden yok?" diye soruyorsanız, belki cevap sizin elinizde olabilir.

---

**Kaynaklar:**
- [Issue #585](https://github.com/google/re2/issues/585) - Benim proof of concept önerim
- [Issue #156](https://github.com/google/re2/issues/156) - Orijinal özellik isteği (61 👍)
- [EPFL Blog](https://systemf.epfl.ch/blog/re2-lookbehinds/) - İsviçre ekibinin çözümü

**BKZ:**
- [GitHub MCP Server'a Katkı Macerası](/posts/github-mcp-server-acik-kaynak-katki/)
- [Git Stratejileri ve Branch Yönetimi](/posts/git-is-akisi-branch-stratejileri/)
- [GitHub Actions CI/CD Pipeline](/posts/github-actions-cicd-pipeline/)

![Furkan Köykıran](/assets/img/avatar.png)
*Furkan Köykıran - Senior Software Engineer*
