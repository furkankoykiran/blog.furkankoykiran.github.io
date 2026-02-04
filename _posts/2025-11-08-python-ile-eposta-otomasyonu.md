---
title: "Python E-Posta Otomasyonu: Deliverability ve SMTP Mühendisliği"
description: "Python ile profesyonel mail gönderimi. Spam filtrelerini (Spam Filters) aşma, SPF/DKIM ayarları, Deliverability ve SMTP protokolü detayları."
date: "2025-11-08 12:00:00 +0300"
categories: [Backend, Automation]
tags: [python, email, deliverability, security, smtp]
image:
  path: /assets/img/posts/email-automation-python-architecture.png
  alt: "Email Delivery Infrastructure"
---

Bir backend geliştirici için "Mail gönderme" görevi genelde basife alınır.
`smtplib` ile 5 satır kod yazılır ve bitti sanılır.
Ta ki pazarlama departmanı kapınızı çalana kadar: "Müşterilere attığımız faturalar Spam klasörüne düşüyor!"
E-posta, 1970'lerden kalma eski bir protokoldür (SMTP) ama üzerindeki güvenlik katmanları (Spam Filtreleri) yapay zeka ile yönetilen bir savaş alanıdır.
Bu yazıda, Python ile sadece mail atmayı değil, o mailin yerine ulaşmasını sağlayan "Deliverability" (Teslim Edilebilirlik) mühendisliğini konuşacağız.
Bir mailin "Sent" (Gönderildi) olması ile "Delivered" (Teslim Edildi) olması arasındaki farkı anlamak, profesyonelliğin ilk adımıdır.

![SMTP Flow](/assets/img/posts/email-automation-python-architecture.png)
*SMTP Handshake ve Mail Transfer Agent (MTA) yolculuğu.*

## 1. Spam Filtrelerini Anlamak: SPF, DKIM, DMARC

Gmail, Hotmail gibi sağlayıcılar, gelen mailin kimliğini doğrulamak için 3 DNS kaydına bakar. Bunlar opsiyonel değildir, zorunludur.

-   **SPF (Sender Policy Framework):** "Benim adıma sadece şu IP'ler mail atabilir" listesi.
-   **DKIM (DomainKeys Identified Mail):** Mailin içeriğine atılan kriptografik imza. Mailin yolda değiştirilmediğini kanıtlar.
-   **DMARC:** "Eğer SPF veya DKIM başarısız olursa ne yapayım?" kuralı. `p=reject` derseniz, sahte mailleri direkt reddeder. Bu, domain itibarınızı (Reputation) korur.

## 2. Multi-Part MIME: Spam Filtrelerinin Aşkı

Sadece HTML gönderirseniz, spam skoru yersiniz.
Profesyonel bir mail, **MIME-Multipart** formatında olmalıdır. Hem HTML hem de Plain Text versiyonu içermelidir.
Eski cihazlar (Apple Watch gibi) veya güvenlik nedeniyle HTML'i engelleyen kurumsal sunucular Plain Text'i gösterir.

```python
from email.mime.multipart import MIMEMultipart
from email.mime.text import MIMEText

msg = MIMEMultipart('alternative')
msg['Subject'] = "Hoşgeldin"
msg['From'] = "me@test.com"
msg['To'] = "you@test.com"

# İki versiyonu da ekle
part1 = MIMEText("Merhaba, linke tıkla: https://site.com", 'plain')
part2 = MIMEText("<h1>Merhaba</h1><a href='https://site.com'>Tıkla</a>", 'html')

msg.attach(part1)
msg.attach(part2)
# Sunucu en uygun olanı seçip gösterir
```

## 3. Inbound Parse: E-postayı API Olarak Okumak

Hep mail atmaktan bahsettik. Peki ya kullanıcı cevap verirse?
SMTP sunucusu kurup POP3 ile o maili okumaya çalışmak işkencedir.
SendGrid veya Mailgun'ın **Inbound Parse Webhook** özelliği, gelen maili (Subject, Body, Attachments) JSON'a çevirip sizin vereceğiniz bir URL'e (`POST /api/save-reply`) atar.
Böylece e-posta sistemini bir Chat uygulaması gibi kurgulayabilirsiniz.
Otomatik destek talebi (Ticket) açmak veya CRM'e işlemek için bu yöntem standarttır.

## 4. IP Stratejisi: Shared vs Dedicated

Mail hacminiz arttıkça şu kararı vermelisiniz:

*   **Shared IP (Paylaşımlı):** API sağlayıcısının havuzundaki diğer müşterilerle aynı IP'yi kullanırsınız.
    *   *Avantaj:* Ucuzdur. IP zaten "ısınmıştır".
    *   *Dezavantaj:* Havuzdaki başka bir müşteri spam yaparsa, sizin de mailleriniz spam'e düşebilir (Noisy Neighbor).

*   **Dedicated IP (Özel):** IP sadece size aittir.
    *   *Avantaj:* İtibar (Reputation) tamamen sizin elinizdedir.
    *   *Dezavantaj:* Pahalıdır ($30-$50/ay). Sıfırdan ısıtma (Warming) yapmanız gerekir.

**Kural:** Aylık 100.000 mailin altındaysanız Shared IP'de kalın. Üstüne çıkarsanız Dedicated IP şart.

## 5. Python `smtplib` vs Transactional API

Kendi sunucunuzdan (Postfix/Sendmail) mail atmak, 2024 yılında "Rus Ruleti" oynamaktır.
IP adresiniz bir kez karalisteye (Blacklist) girerse, oradan çıkmak aylar sürer.
Profesyonel çözüm: **Transactional Email API** kullanmaktır (AWS SES, SendGrid, Mailgun).
Bu servisler IP itibarını kendileri yönetir.

Python ile entegrasyonu `requests` kadar basittir ve SMTP'den çok daha hızlıdır. Ayrıca `Message-ID` takibi yapmanızı saplar.

## 6. HTML Template Yönetimi (Jinja2)

HTML mail tasarımı, Web tasarımına benzemez. Modern CSS'lerin çoğu (Flexbox, Grid) Gmail'de çalışmaz.
1990'ların teknolojisiyle, `<table>` kullanarak (Table-based layout) tasarım yapmanız gerekir.
Kodun içine HTML gömmek yerine **Jinja2** şablon motorunu kullanın.

`mjml` aracını kullanmanızı şiddetle tavsiye ederim. Sizi tablo cehenneminden kurtarır ve responsive mail çıktısı üretir.

![HTML Email Template](/assets/img/posts/jinja2-email-template-rendering.jpg)
*Jinja2 ve MJML ile dinamik içerik üretimi.*

## 7. Feedback Loops (FBL) ve List Hygiene

Mail gönderdiğinizde, alıcı "Spam Olarak İşaretle" butonuna basarsa ne olur?
Eğer **Feedback Loop (FBL)** kaydınız varsa (Gmail Postmaster Tools), Google size bir rapor yollar: "Şu kullanıcı senden şikayetçi oldu."
Siz de o kullanıcıyı veritabanından hemen **unsubscribe** etmelisiniz.
Eğer şikayet edenlere mail atmaya devam ederseniz, Google tüm domaininizi blocklar.

### List Hygiene (Liste Temizliği)

Veritabanınızda %10 oranında çalışmayan mail (Hard Bounce) varsa, %90 çalışan mail de riske girer.
Çünkü sunucular "Bu adam listesini temizlemiyor, demek ki spammer" diye düşünür.
Otomatik bir script yazarak:
1.  Son 6 aydır maili açmayanları (Inactive) ayrı bir segmente alın.
2.  Hard Bounce olanları silin (Soft delete değil, Hard delete).
3.  Rol hesaplarını (`admin@`, `support@`, `info@`) listenizden çıkarın. Bunlar genelde kişisel hesap değildir.

## 8. IP Isıtma (IP Warming) ve Rate Limiting

Elinizde 100.000 kişilik bir mail listesi var. Scripti yazdınız, tüm gücünüzle "Gönder" tuşuna bastınız.
Sonuç: **BLOCKLENDINIZ.**
Gmail, tanımadığı bir IP'den aniden binlerce mail gelirse bunu Saldırı sayar.
IP'nizi "Isıtmanız" gerekir.
1. Gün: 50 mail.
2. Gün: 100 mail.
Her gün %20 artırarak güven kazanırsınız. Python scriptinizde `time.sleep()` veya Celery Rate Limit kullanın.

## 9. Yardımcı Kütüphaneler

`smtplib` ve `email` modülleri "low-level"dır ve kullanımı zordur.
Production ortamında şu kütüphaneleri kullanmak işinizi kolaylaştırır:

*   **Envelopes:** İnsanlar için mail kütüphanesi.
    ```python
    from envelopes import Envelope
    envelope = Envelope(
        from_addr=(u'me@example.com', u'From Me'),
        to_addr=(u'world@example.com', u'To World'),
        subject=u'Merhaba Dünya',
        text_body=u'Bu bir test mailidir.'
    )
    envelope.send('smtp.googlemail.com', login='user', password='password', tls=True)
    ```
*   **Marrow Mailer:** Yüksek performanslı ve asenkron gönderim desteği sunar.
*   **Django Anymail:** Django kullanıyorsanız, Mailgun/SendGrid backendlerini tek bir ayarla değiştirmenizi sağlar.

## 10. Güvenlik: TLS ve STARTTLS

Mail trafiği, internet üzerinde düz metin (Plain Text) olarak akmamalıdır.
Şifreleme standartlarını bilmek zorundasınız:

*   **SSL (Port 465):** Bağlantı kurulduğu an şifrelenir. Eskidir ama hala güvenlidir.
*   **STARTTLS (Port 587):** Bağlantı önce güvensiz başlar, sonra `ehlo` komutu ile "Hadi şifreleyelim" denir. Modern standart budur.
*   **Port 25:** Genelde ISP'ler tarafından bloklanır (Spam çıkışını engellemek için). Asla kullanmayın.

## 11. Senior Backend Mülakat Soruları

**S1: SMTP ve API ile mail göndermek arasındaki temel performans farkı nedir?**
*Cevap:* SMTP Stateful'dur, çok sayıda el sıkışma (Handshake) gerektirir ve yavaştır. API (HTTP) Stateless'tır, connection pool ile çok daha hızlıdır.

**S2: "Grey Listing" nedir?**
*Cevap:* Sunucu, tanımadığı göndericiye "Geçici Hata" (4xx) döner. Gerçek sunucular tekrar dener (Retry), spam botları denemez.

**S3: Transactional vs Marketing Mail neden ayrılmalıdır?**
*Cevap:* Promosyon mailleri (Marketing) spam riski taşır. Şifre sıfırlama (Transactional) maillerinin bu riskten etkilenmemesi için farklı IP/Subdomain kullanılmalıdır.

![Spam Filters](/assets/img/posts/email-automation-python-architecture.png)
*Spam Score hesaplama mekanizması: İçerik, IP İtibarı ve Kullanıcı Tepkisi.*

## Sık Yapılan Hatalar (Anti-Patterns)

1.  **"No Reply" Adresi Kullanmak:** `noreply@sirket.com` kullanmak, müşteriye "Senin cevabın umurumda değil" demektir. Etkileşimi düşürür ve spam skorunu artırır.
2.  **Resim Ağırlıklı Mail:** Mailin %50'den fazlası resimse, filtreler bunu şüpheli bulur. Metin/Resim oranını 60/40 tutun. Sadece resimden oluşan ("Image-only") mailler direkt Spam'dir.
3.  **Unsubscribe Linki Koymamak:** Yasal bir suçtur (GDPR/KVKK) ve alıcıyı "Spam Olarak İşaretle" butonuna basmaya iter. Kullanıcıyı listeden çıkarmak, spam işaretlenmekten bin kat iyidir.
4.  **Hızlı Link Kısaltıcı:** `bit.ly` veya `goo.gl` gibi genel servisleri mail içinde kullanırsanız, spam klasörüne düşmeniz garantidir. Kendi domaininizi (örneğin `link.sirket.com`) kullanın.
5.  **Subject Line Hileleri:** "RE: Fatura" gibi yalan başlıklar (aslında fatura değilse) kısa vadede açılma oranını artırır ama uzun vadede domain itibarınızı yok eder.

## Terimler Sözlüğü (Glossary)

*   **SMTP (Simple Mail Transfer Protocol):** E-posta gönderme protokolü (Giden kutusu).
*   **IMAP (Internet Message Access Protocol):** E-posta okuma protokolü (Sunucuda saklar).
*   **POP3:** Eski okuma protokolü (Cihaza indirir ve sunucudan siler).
*   **MTA (Mail Transfer Agent):** Posta sunucusu yazılımı (Postfix, Sendmail).
*   **MUA (Mail User Agent):** E-posta istemcisi (Outlook, Thunderbird).
*   **Bounce:** Teslim edilemeyen ve geri dönen e-posta.
*   **Soft Bounce:** Geçici hata (Sunucu meşgul, kutu dolu).
*   **Hard Bounce:** Kalıcı hata (Adres yok). Listeden hemen silinmeli.
*   **Blacklist (RBL):** Spam gönderdiği için engellenen IP listesi.
*   **Rate Limiting:** Birim zamanda gönderilen mail sayısı limiti.
*   **Greylisting:** Tanımadığı sunucuyu "Biraz bekle sonra gel" diye bekletme tekniği.
*   **Header:** Mailin üst bilgisi (Kimden, Kime, Tarih, Message-ID).

## Sonuç

E-posta göndermek bir kodlama işi değil, bir **Protokol ve İtibar Yönetimi** işidir.
Kodunuz sadece buzdağının görünen kısmıdır.
Müşterinize "Maili attım" demek yetmez, "Mail ulaştı" diyebilmek ve bunu loglardan kanıtlamak gerekir (Observability).
Eğer mailleriniz spam'e düşüyorsa kodu değil, DNS kayıtlarınızı ve içerik kalitenizi inceleyin.

![Email Protocols](/assets/img/posts/python-smtp-email-workflow.png)
*Protokol detayları.*