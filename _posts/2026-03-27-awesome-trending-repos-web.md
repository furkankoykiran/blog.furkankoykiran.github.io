---
title: "awesome-trending-repos: GitHub Trending için Modern Web Arayüzü"
description: "React, Vite ve GitHub Pages ile günlük güncellenen trending repos web sitesi"
date: 2026-03-27 18:00:00 +0300
categories: [Web Development, GitHub, React]
tags: [react, vite, github-pages, github-actions, tailwindcss]
image:
  path: /assets/img/posts/2026-03-27-awesome-trending-repos/og-image.png
  alt: "Awesome Trending Repos"
---

Önceki blog yazımda [awesome-trending-repos](https://github.com/furkankoykiran/awesome-trending-repos) projesinden bahsetmiştim. O zamanlar proje sadece README.md dosyasına veri yazıyordu. Artık things changed. Proje artık tam fonksiyonlu bir modern web uygulaması.

![CI/CD Pipeline](/assets/img/posts/github-actions-cicd-pipeline-diagram.png)
*GitHub Actions ile otomatik build ve deploy süreci*

---

## Neden Web Arayüzü?

README.md dosyası işe yarar ama statiktir. Trending projeleri takip etmek için daha interaktif bir deneyim istedim. Kullanıcıların dil bazlı filtreleme yapabilmesi, arama yapabilmesi ve grafiklerle verileri görebilmesi gerekliydi.

Bu yüzden React + Vite ile modern bir single-page application (SPA) geliştirdim.

---

## Tech Stack

**Frontend:**
- React 19 - Son sürüm
- Vite - Hızlı dev server ve build
- Tailwind CSS v4 - Utility-first styling
- Framer Motion - Animasyonlar
- Recharts - Veri görselleştirme

**Backend/Infrastructure:**
- GitHub Actions - Otomasyon
- GitHub Pages - Hosting
- Custom Domain - furkankoykiran.com.tr

---

## Proje Yapısı

Projenin temel dizini şöyle:

```text
awesome-trending-repos/
├── src/
│   ├── App.jsx          # Ana React bileşeni
│   ├── main.jsx         # Entry point
│   └── index.css        # Tailwind + custom styles
├── scripts/
│   └── update.js        # Trending veri scrape script
├── public/
│   ├── data/
│   │   └── trending.json # Frontend veri kaynağı
│   └── logo.png
├── .github/workflows/
│   └── update-trends.yml # CI/CD pipeline
└── vite.config.js       # GitHub Pages config
```

---

## GitHub Pages Entegrasyonu

Vite ile GitHub Pages kullanırken en önemli nokta `base` path konfigürasyonudur. Projeniz bir subdirectory'de çalışacaksa (örn: `github.io/repo-name`), `vite.config.js` dosyasında bunu belirtmelisiniz:

```javascript
// vite.config.js
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import tailwindcss from '@tailwindcss/vite';

export default defineConfig({
  plugins: [react(), tailwindcss()],
  base: '/awesome-trending-repos/', // GitHub Pages subdirectory
});
```

Bu ayar olmadan, production'da asset path'leri bozulur ve 404 hatası alırsınız.

---

## GitHub Actions Workflow

Her gün gece yarısı UTC'de çalışan workflow:

1. **Trending verisi çek** - GitHub Trending sayfasını scrape et
2. **Frontend build** - `npm run build` ile dist/ klasörünü oluştur
3. **Data commit** - Değişiklik varsa README ve data dosyalarını commit et
4. **Deploy** - GitHub Pages'e dağıt

```yaml
# .github/workflows/update-trends.yml
name: Update and Deploy Trends

on:
  schedule:
    - cron: '0 0 * * *'  # Her gün gece yarısı
  workflow_dispatch:

permissions:
  contents: write
  pages: write
  id-token: write

jobs:
  update-and-build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install dependencies
        run: npm ci

      - name: Run update script
        run: npm run update
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - name: Build frontend
        run: npm run build

      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: './dist'

  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: update-and-build
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

![Deployment Flow](/assets/img/posts/cicd-deployment-automation-flow.png)
*Otomatik deployment akışı*

---

## Frontend Özellikleri

### 1. Dil Filtreleme

Kullanıcılar programlama diline göre filtreleme yapabilir:

```javascript
const languages = ['All', ...new Set(repos.map(r => r.language).filter(Boolean))];

const filteredRepos = repos.filter(repo => {
  const matchesLang = filter === 'All' || repo.language === filter;
  const matchesSearch = repo.name.toLowerCase().includes(searchQuery.toLowerCase());
  return matchesLang && matchesSearch;
});
```

### 2. Arama Fonksiyonu

Real-time arama ile repo ismi, owner veya description'da arama yapılır.

### 3. Görsel İstatistikler

Recharts kütüphanesi ile dil dağılımı bar chart ve growth liderleri listesi:

```javascript
<ResponsiveContainer width="100%" height="100%">
  <BarChart data={data?.insights?.topLanguages || []}>
    <CartesianGrid strokeDasharray="8 8" stroke="#ffffff03" />
    <XAxis dataKey="language" stroke="#475569" />
    <YAxis stroke="#475569" />
    <Tooltip contentStyle={{ backgroundColor: 'rgba(15, 23, 42, 0.9)' }} />
    <Bar dataKey="count" radius={[12, 12, 0, 0]}>
      {data?.insights?.topLanguages?.map((entry, index) => (
        <Cell key={index} fill={getLangColor(entry.language)} />
      ))}
    </Bar>
  </BarChart>
</ResponsiveContainer>
```

### 4. Deployment Feed

Her güncelleme için bir feed entry oluşturulur. Kullanıcılar deployment tarihçesini görebilir.

---

## Custom Domain Ayarı

GitHub Pages için custom domain kullanmak istersen:

1. **CNAME dosyası oluştur** - `public/CNAME` içine domain yaz:
   ```
   furkankoykiran.com.tr
   ```

2. **DNS ayarı** - Domain provider'ında:
   ```
   CNAME furkankoykiran.github.io
   ```

3. **HTTPS** - GitHub Pages otomatik olarak Let's Encrypt ile SSL sağlar.

---

## Performans İpuçları

### 1. Asset Optimization

Vite production build otomatik olarak minification ve tree-shaking yapar. Büyük resimler için `public/` klasörü yerine optimized image kullanın.

### 2. Lazy Loading

Büyük listeler için virtual scrolling kullanmayı düşünün. Bu projede 25 repo olduğu için gerekli değil ama scale ederseniz eklenebilir.

### 3. Caching

GitHub Pages otomatik olarak cache headers ekler. Ancak `index.html` için cache-busting kullanın:

```javascript
// vite.config.js
export default defineConfig({
  build: {
    rollupOptions: {
      output: {
        entryFileNames: `assets/[name].[hash].js`,
        chunkFileNames: `assets/[name].[hash].js`,
        assetFileNames: `assets/[name].[hash].[ext]`
      }
    }
  }
});
```

---

## Canlı Demo

Proje şu anda [furkankoykiran.com.tr/awesome-trending-repos](https://furkankoykiran.com.tr/awesome-trending-repos/) adresinde yayında. Her gün otomatik olarak güncelleniyor.

![Project Logo](/assets/img/posts/2026-03-27-awesome-trending-repos/logo.png)
*Proje logosu*

![OG Image](/assets/img/posts/2026-03-27-awesome-trending-repos/og-image.png)
*Open Graph preview görseli*

---

## Sonuç

README.md'den modern web uygulamasına geçiş, projenin kullanılabilirliğini önemli ölçüde artırdı. Kullanıcılar artık sadece statik bir liste yerine interaktif bir deneyim yaşıyor.

GitHub Pages + Vite kombinasyonu, static siteler için harika bir çözüm. Ücretsiz, hızlı ve CI/CD ile tam otomatik.

Proje tamamen açık kaynak. Katkı yapmak isterseniz [GitHub repo](https://github.com/furkankoykiran/awesome-trending-repos)'ya pull request gönderebilirsiniz.

![Furkan Köykıran](/assets/img/avatar.png)
*Furkan Köykıran - Senior Software Engineer*

---

## Diğer Blog Yazılarım

- [Python Proje Yönetimi: Poetry, uv ve Modern Standartlar](/posts/python-proje-yonetimi-poetry-uv/)
- [Traefik v3: Docker Çağının Reverse Proxy Çözümü](/posts/traefik-reverse-proxy-ssl-yonetimi/)
- [MCP Ekosistemi ve AI Platformlarına Açık Kaynak Katkılarım](/posts/mcp-ai-contributions/)
