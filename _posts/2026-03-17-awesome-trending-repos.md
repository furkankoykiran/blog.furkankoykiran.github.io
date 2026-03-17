---
title: "awesome-trending-repos: GitHub Trending'i Otomatik Takip Etme"
description: "GitHub Actions ile günlük trending projeleri otomatik takip eden sistem"
date: 2026-03-17 17:00:00 +0300
categories: [GitHub, Otomasyon]
tags: [github-actions, automation, nodejs, trending]
image:
  path: /assets/img/posts/2026-03-18-awesome-trending-repos/banner.jpg
  alt: "Awesome Trending Repos"
---

GitHub'ta her gün yüzlerce yeni proje trending listesine giriyor. Ancak bu projeleri takip etmek, hangilerinin gerçekten önemli olduğunu belirlemek zaman alıcı bir iş. Bu sorunu çözmek için [awesome-trending-repos](https://github.com/furkankoykiran/awesome-trending-repos) projesini geliştirdim.

![GitHub Actions](/assets/img/posts/2026-03-18-awesome-trending-repos/github-actions.png)
*GitHub Actions ile otomatik günlük güncellemeler*

---

## Proje Nedir?

awesome-trending-repos, GitHub'ın trending sayfasındaki projeleri otomatik olarak takip eden, günlük olarak güncellenen bir liste. Proje her gün gece yarısı UTC'de çalışıyor ve trending reposu analiz edip sonuçları README.md dosyasına yazıyor.

**Temel özellikler:**
- 🔄 Günlük otomatik güncellemeler
- 📊 Geçmiş karşılaştırmaları ve sıralama değişiklikleri
- 📈 ASCII grafikleri ve trend görselleştirmeleri
- 🏆 En hızlı büyüyen repolar (rising stars)
- 💾 7 günlük geçmiş veri saklama

---

## Nasıl Çalışır?

Proje, GitHub Actions kullanarak otomatik olarak çalışıyor. Her gün gece yarısı (UTC) bir workflow tetikleniyor, GitHub trending sayfasını scrape ediyor, verileri analiz ediyor ve sonuçları README.md dosyasına yazıyor.

**Veri toplama süreci:**
1. GitHub trending sayfasını scrape etme (Cheerio ile)
2. GitHub Search API'den ek veri alma (fallback olarak)
3. Projeleri dil kategorilerine göre ayırma
4. Geçmiş verilerle karşılaştırma yapma
5. README.md'yi otomatik güncelleme

![GitHub Actions](/assets/img/posts/2026-03-18-awesome-trending-repos/github-actions.png)
*CI/CD pipeline ile otomatik workflow yönetimi*

---

## Özellikler

### Günlük Otomasyon

GitHub Actions ile her gün otomatik olarak çalışıyor. Manuel hiçbir işlem gerektirmiyor. Sistem her gün gece yarısı UTC'de trending verilerini çekip README.md'yi güncelliyor.

### Geçmiş Takibi

Proje, son 7 günün verilerini saklıyor. Bu sayede bir projenin trending listesindeki konumu nasıl değişmiş görebiliyorsunuz. Yeni giren projeler, yükselenler ve düşenler hakkında detaylı bilgi veriliyor.

### Dil Bazlı Sıralama

Trending projeleri programlama diline göre kategorize ediyor. JavaScript, Python, TypeScript, Go, Rust gibi popüler dillerdeki projeleri ayrı ayrı takip edebiliyorsunuz.

### Görselleştirme

ASCII grafikleri ile projelerin yükselişini ve düşüşünü görsel olarak takip edebiliyorsunuz. Hangi projelerin kaç yıldız aldığını, hangi hızla büyüdüklerini grafiklerden görebiliyorsunuz.

---

## Teknik Detaylar

Proje Node.js ile yazıldı ve ES Modules kullanıyor. GitHub Actions ile otomatikleştirildi ve Octokit library'i ile GitHub API'ye entegre edildi.

**Kullanılan teknolojiler:**
- Node.js 20+
- GitHub Actions (cron tabanlı scheduling)
- Octokit (GitHub API client)
- Cheerio (web scraping)
- Axios (HTTP requests)

---

## Sonuç

awesome-trending-repos projesi ile GitHub'taki trending projeleri otomatik olarak takip edebiliyorsunuz. Her gün güncellenen liste ile en yeni ve en popüler projeleri kaçırmıyorsunuz.

Proje hem kişisel kullanım için hem de developer community'si için değerli bir kaynak oluşturuyor. Otomasyon sayesinde zaman kazandırıyor ve trendleri takip etmeyi kolaylaştırıyor.

![Furkan Köykıran](/assets/img/avatar.png)
*Furkan Köykıran - Senior Software Engineer*

---

## Diğer Blog Yazılarım

- [MCP Ekosistemi ve AI Platformlarına Açık Kaynak Katkılarım](/posts/mcp-ai-contributions/)
- [Dev.to MCP Server Geliştirme ve Yayınlama](/posts/devto-mcp-server-gelistirme-ve-yayinlama/)
