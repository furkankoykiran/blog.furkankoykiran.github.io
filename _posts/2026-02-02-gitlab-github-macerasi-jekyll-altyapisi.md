---
title: "GitLab'dan GitHub'a Dönüş: Bir Altyapı Migration Macerası"
description: "GitHub hesabımın askıya alınmasıyla başlayan GitLab yolculuğu, CI/CD esnekliği ve iki dev platformun karşılaştırmalı analizi."
date: "2026-02-02 10:00:00 +0300"
categories: [DevOps, Architecture]
tags: [gitlab, github, cicd, jekyll, migration, infrastructure]
image:
  path: /assets/img/posts/gitlab_to_github_migration_logic.png
  alt: "GitLab to GitHub Migration Path"
---

Bir sabah uyandığımda, GitHub hesabımın "suspended" olduğunu görmek benim için en büyük kabustu. Yılların emeği, açık kaynak projelerim ve blog altyapım bir anda erişilemez hale gelmişti. Sebebi sonradan basit bir hata olarak anlaşılsa da, o an yaşadığım panik beni hızlıca bir B planı yapmaya yöneltti.

Bu kriz, beni GitLab'ın sularına ve altyapımı "platform-agnostic" (platformdan bağımsız) hale getirme fikrine itti.

Bu yazıda; profil sitem (`https://furkankoykiran.com.tr`) ve blog sayfamın (`https://blog.furkankoykiran.com.tr`) orijinal kurulumlarını, GitLab'a zorunlu göç sürecinde keşfettiklerimi ve GitHub'a dönerken cebimde getirdiğim teknik dersleri anlatacağım.

![Migration Visual](/assets/img/posts/gitlab_to_github_migration_logic.png)
*GitLab ve GitHub arasındaki veri akışı ve göç süreci.*

## 1. Orijinal Kurulum: Temeller

Portfolyo web sitesi ve blog, bir yazılımcının dijital kimliğidir. Ben bu kimliği Jekyll ve modern statik site prensipleri üzerine inşa ettim.

### Portfolyo (furkankoykiran.com.tr)
Ana sitem, sadelik ve hız odaklı bir Jekyll teması üzerine kurulu. 
- **Setup:** `Gemfile` içine `jekyll` ve temel pluginleri ekledikten sonra `bundle install`.
- **Customization:** `_config.yml` üzerinden tüm SEO ve sosyal medya linklerimi yönetiyorum.
- **Workflow:** Sadece Markdown dosyalarını ve HTML partitial'larını düzenlemek yetiyor.
- **Güvenlik & SSL:** Cloudflare arkasında "Full" SSL modunda çalıştırıyorum. GitHub Pages ile Cloudflare arasındaki bu uyum, hem güvenlik hem de CDN performansı açısından kritik.

### Blog (blog.furkankoykiran.com.tr)
Blog tarafında ise **Chirpy** temasını kullanıyorum.
- **Initial Start:** Chirpy-Starter reposunu forklayarak başladım. Bu, çekirdek dosyaları bozmadan içerik üretmeyi kolaylaştırıyor.
- **Initialization:** `bash tools/init.sh` komutuyla repoyu temizleyip kendi ayarlarımı yaptım.
- **Kritik İpucu:** Yerel testler için `bundle exec jekyll s` kullanırken, `_config.yml`'deki `url` ayarının lokal domain ile çakışmadığından emin olmak gerekiyor. Aksi takdirde asset yükleme hataları (404) kaçınılmaz oluyor.

## 2. GitLab Durağı: Esneklik ve CI/CD Gücü

GitHub kapıları geçici olarak kapandığında, tüm repolarımı GitLab'a taşıdım. GitLab, sadece bir kod deposu değil, uçtan uca bir DevOps platformu.

### .gitlab-ci.yml vs GitHub Actions
GitHub Actions (`.github/workflows/*.yml`) ile GitLab CI/CD (`.gitlab-ci.yml`) arasındaki fark, sadece syntax değil, felsefe farkı.

**Örnek GitLab CI Yapılandırması:**
GitLab'da cache mekanizması ve job tanımları daha "deklaratif" hissettiriyor.
```yaml
image: ruby:3.2

# Cache, build sürelerini ciddi oranda düşürüyor
cache:
  paths:
    - vendor/

pages:
  stage: deploy
  script:
    - bundle config set --local path 'vendor'
    - bundle install
    - bundle exec jekyll build -d public
  artifacts:
    paths:
      - public
  only:
    - master
```

**Örnek GitHub Actions Yapılandırması:**
GitHub Actions ise Marketplace sayesinde inanılmaz modüler.
```yaml
name: Deploy
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-ruby@v1
        with:
          ruby-version: '3.2'
          bundler-cache: true # Caching burada tek satırla hallediliyor
      - run: bundle exec jekyll build
```

### Derinlemesine Karşılaştırma
Bu süreçte iki platformu da production ortamında deneyimleme şansım oldu. İşte "kağıt üzerinde" yazmayan, ancak sahada karşınıza çıkan farklar:

| Özellik | GitLab CI/CD | GitHub Actions |
| :--- | :--- | :--- |
| **Öğrenme Eğrisi** | Daha dik. `.gitlab-ci.yml` çok güçlü ama verbose olabiliyor. İlk kurulumda dokümantasyonla sıkı fıkı olmanız şart. | Daha düz. `workflow_dispatch` gibi özellikler çok pratik. "Action" mantığı legolarla oynamak gibi. |
| **Caching** | `cache:` keyword'ü ile manuel path belirtmek gerekiyor. Distributed cache yapısı çok sağlam ama yapılandırması dikkat ister. | `actions/cache` veya `setup-` action'ları ile neredeyse zero-config çalışıyor. Ruby, Node.js gibi dillerde dependency cache otomatik. |
| **Runner Yapısı** | Enterprise dünyasında, kendi runner'larınızı yönetmek GitLab'da çocuk oyuncağı. Docker, K8s entegrasyonu built-in. | Hosted runner'lar çok hızlı ve pratik. Self-hosted runner desteği var ama security yönetimi (özellikle public repolarda) dikkat istiyor. |
| **Marketplace** | Yok denecek kadar az. Genelde script yazmanız bekleniyor. Tekerleği yeniden icat etmeniz gerekebilir. | Devasa bir pazar. Neredeyse her şey için hazır bir Action var. S3 upload'dan Slack bildirimine kadar her şey hazır. |

![DevOps Pipeline](/assets/img/posts/devops_pipeline_visualization.png)
*Modern bir CI/CD hattının görselleştirilmesi.*

### 2.5. Derinlemesine Teknik: Cache Stratejileri
Her iki platformda da build sürelerini düşürmenin anahtarı "caching" mekanizmasını doğru kullanmaktan geçiyor. Ancak yaklaşımlar farklı:

**GitLab Cache Mantığı:**
GitLab, cache'i "key" bazlı yönetir. `Gemfile.lock` dosyasının hash'ini key olarak kullanmak yaygın bir pratiktir. Eğer lock dosyası değişmezse, Ruby gem'lerini tekrar indirmek yerine cache'den çeker.
```yaml
cache:
  key:
    files:
      - Gemfile.lock
  paths:
    - vendor/bundle
```
Bu yöntem, özellikle büyük monorepo'larda veya bağımlılığı çok olan projelerde build süresini dakikalardan saniyelere indirebilir. Ancak cache'in "push" ve "pull" politikalarını (`policy: pull-push`) doğru ayarlamazsanız, gereksiz yere cache upload süresiyle savaşırsınız.

**GitHub Cache Mantığı:**
GitHub Actions'ta `actions/cache@v3` action'ı bu işi daha soyut bir katmanda yapar. 
```yaml
- uses: actions/cache@v3
  with:
    path: vendor/bundle
    key: ${{ runner.os }}-gems-${{ hashFiles('**/Gemfile.lock') }}
    restore-keys: |
      ${{ runner.os }}-gems-
```
GitHub'ın en büyük avantajı, `setup-ruby` veya `setup-node` gibi resmi action'ların içinde "dependency caching" özelliğinin artık built-in (dahili) olarak gelmesidir. Sadece `cache: 'bundler'` diyerek tüm bu konfigürasyonu atlayabilirsiniz.

### GitLab Klasöründeki Ekstralar
Migration sırasında GitLab tarafında şu ek dosyalar ortaya çıktı:
- `.gitlab-ci.yml`: Tüm build, test ve deploy süreçlerini yöneten ana kumanda merkezi. GitHub'daki `.github/workflows` klasörüne denk gelir ama tek dosyada yönetim bazen karmaşaya yol açabilir.
- `public/`: GitLab Pages, GitHub'ın `_site` klasörü yerine, web server'ın sunacağı dosyaları bu klasörde bekliyor.
- `artifacts`: GitLab'da pipeline sonrası oluşan dosyaların saklanma mantığı (expiration, download) çok daha görünür ve yönetilebilir. Hangi job'un hangi dosyayı ürettiğini UI üzerinden kolayca takip edebilirsiniz.

## 3. GitHub'a Geri Dönüş: Neden?

Hesabım tekrar açıldığında neden GitHub'a döndüm?
1. **Community:** Açık kaynak projelerin kalbi hala burada atıyor. Bir kütüphanede hata bulduğumda PR göndermek, Issue açmak GitHub'da çok daha doğal bir akış.
2. **Ekosistem:** VS Code, Copilot ve diğer geliştirici araçlarıyla olan entegrasyonu, geliştirme hızımı ciddi oranda artırıyor. Editörden çıkmadan PR review yapmak paha biçilemez.
3. **Dependabot:** Güvenlik açıklarını tarayıp otomatik PR açması, özellikle bakım maliyetlerini düşürmek için harika bir özellik. GitLab'ın da benzer özellikleri var (Dependency Scanning) ama GitHub'ın entegrasyonu daha "developer-friendly".

## 4. Teknik Kurulum Rehberi (Sıfırdan)

Eğer kendi sitenizi sıfırdan kurmak istiyorsanız, işte checklist:

### Adım 1: Repo Yapısı
- User site için: `<username>.github.io` (Bu repo özel isimdir, GitHub doğrudan `master` veya `main` branch'ini yayınlar).
- Project site (blog) için: `blog.<username>.github.io` (veya istediğiniz herhangi bir isim). Reponun "public" olması, GitHub Pages'in ücretsiz sürümünü kullanabilmeniz için genelde şarttır.

### Adım 2: Chirpy Blog Setup
```bash
# Chirpy Starter'ı clone'layın
git clone https://github.com/cotes2020/chirpy-starter.git yourblog
cd yourblog

# Init script çalıştırın (GitHub dışına kuruyorsanız --no-gh ekleyin)
# Bu script gereksiz dosyaları temizler ve repoyu size özel hale getirir.
bash tools/init.sh

# Lokal test
# Gemfile.lock oluşması için önce install
bundle install
# Server'ı ayağa kaldırın
bundle exec jekyll s
```

### Adım 3: Disaster Recovery Stratejisi
Bu tecrübeden çıkardığım en önemli ders, felaket senaryolarına hazırlıklı olmak. Sadece "yedek almak" yetmiyor, o yedeği "geri yükleyebileceğinizi" (restore) test etmeniz gerekiyor:

- **Mirroring:** Kodunuzun bir kopyasını her zaman başka bir platformda (GitLab, Bitbucket) tutun. GitHub'da `git push --mirror` komutuyla tüm branch ve tag'leri aktarabilirsiniz.
- **Generic CI Scriptleri:** Build ve test komutlarınızı (örneğin `./scripts/build.sh`) shell scriptlerinde tutun. Böylece CI aracınız değişse bile (GitLab -> GitHub -> Jenkins), sadece bu scripti çağıran config dosyasını değiştirmeniz yeterli olur. "Vendor Lock-in"den kaçınmanın en temiz yolu budur.
- **Backup:** Sadece buluta güvenmeyin. Haftalık `git clone --mirror` ile lokal bir fiziksel yedeğiniz olsun. 3-2-1 yedekleme kuralını (3 kopya, 2 farklı medya, 1 of-site) kodlarınız için de uygulayın.
- **DNS Kontrolü:** Domain yönetiminin (DNS) kod hosting sağlayıcısından bağımsız (örneğin Cloudflare) olması, siteyi başka bir sunucuya yönlendirmenizi saniyeler içinde yapmanızı sağlar.

## 5. Sonuç ve Dersler

GitLab macerası bana GitLab'ın ne kadar güçlü, özellikle "on-premise" ve kurumsal ihtiyaçlar için ne kadar yetkin bir araç olduğunu gösterdi. Esnekliği ve pipeline üzerindeki kontrol hissi muazzam. Ancak bireysel projeler, hız ve topluluk etkileşimi söz konusu olduğunda GitHub Actions ve GitHub Pages ikilisi hala çok güçlü.

En büyük çıkarımım şu: **Kodunuz sizin malınızdır, platformlar ise sadece onu dünyaya açan birer araçtır.** Yarın GitHub veya GitLab'ın başına bir şey gelse, 5 dakika içinde diğer platformda pipeline'ımı ayağa kaldırabilecek esneklikte kalmalıyım. Bu yüzden altyapıyı "platform-agnostic" tasarlamak, bir mühendisin en önemli sorumluluklarından biri.

---
**BKZ:** 
- [Git Stratejileri ve Branch Yönetimi](/posts/git-is-akisi-branch-stratejileri/)
- [Modern Web Mimarisi](/posts/modern-web-arayuzu-css-javascript/)
- [SEO ve Performans Optimizasyonu](/posts/veritabani-baglanti-havuzu-optimizasyon/)

![Furkan Köykıran](/assets/img/avatar.png)
*Furkan Köykıran - Senior Software Engineer*
