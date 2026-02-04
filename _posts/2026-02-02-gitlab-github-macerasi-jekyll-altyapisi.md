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

Bir sabah uyandığımda, yıllardır tüm projelerimi barındırdığım GitHub hesabımın "suspended" (askıya alınmış) olduğunu görmek, bir yazılımcı için kabusların en gerçeğiydi. Sebebi her ne olursa olsun (ki sonradan bir hata olduğu anlaşıldı), o an hızlıca bir B planı devreye sokmam gerekiyordu. Bu kriz, beni GitLab'ın derinliklerine ve altyapımı yeniden düşünmeye itti.

Bu yazıda; profil sitem (`furkankoykiran.github.io`) ve blog sayfamın (`blog.furkankoykiran.github.io`) orijinal hallerini nasıl kurduğumu, GitLab'a geçişte neler keşfettiğimi ve sonunda GitHub'a geri dönerken yanımda getirdiğim dersleri anlatacağım.

![Migration Visual](/assets/img/posts/gitlab_to_github_migration_logic.png)
*GitLab ve GitHub arasındaki veri akışı ve göç süreci.*

## 1. Orijinal Kurulum: Temeller

Portfolyo web sitesi ve blog, bir yazılımcının dijital kimliğidir. Ben bu kimliği Jekyll üzerine inşa ettim.

### Portfolyo (furkankoykiran.github.io)
Ana sitem, sadelik ve hız odaklı bir Jekyll teması üzerine kurulu. 
- **Setup:** `Gemfile` içine `jekyll` ve temel pluginleri ekledikten sonra `bundle install`.
- **Customization:** `_config.yml` üzerinden tüm SEO ve sosyal medya linklerimi yönetiyorum.
- **Workflow:** Sadece Markdown dosyalarını ve HTML partitial'larını düzenlemek yetiyor.
- **Güvenlik & SSL:** GitHub Pages, `furkankoykiran.com.tr` gibi custom domainler için Let's Encrypt aracılığıyla otomatik SSL sağlar. Cloudflare arkasında "Full" SSL modunda çalıştırıyorum.

### Blog (blog.furkankoykiran.github.io)
Blog tarafında ise efsanevi **Chirpy** temasını kullanıyorum.
- **Initial Start:** Chirpy-Starter reposunu forklayarak başladım. Bu, temanın çekirdek dosyalarını bozmadan sadece içeriğe odaklanmanızı sağlıyor.
- **Initialization:** `bash tools/init.sh` komutuyla repoyu temizleyip kendi ayarlarımı yaptım.
- **Kritik İpucu:** Yerel testler için `bundle exec jekyll s` kullanırken, sitenin tam halini görmek için `_config.yml`'deki `url` kısmının lokal domain ile çakışmadığından emin olun.

## 2. GitLab Durağı: Esneklik ve CI/CD Gücü

GitHub kapıları geçici olarak kapandığında, tüm repolarımı GitLab'a taşıdım. GitLab, bir "kod hostingleme" sitesinden çok, uçtan uca bir DevOps platformu gibi hissettiriyor.

### .gitlab-ci.yml vs GitHub Actions
GitHub Actions (`.github/workflows/*.yml`) çok daha "market" odaklıyken, GitLab CI/CD (`.gitlab-ci.yml`) çok daha "entegre" ve esnek.

**Örnek GitLab CI Yapılandırması:**
```yaml
image: ruby:3.2

pages:
  stage: deploy
  script:
    - bundle install
    - bundle exec jekyll build -d public
  artifacts:
    paths:
      - public
  only:
    - master
```

**Örnek GitHub Actions Yapılandırması:**
```yaml
name: Deploy
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-ruby@v1
      - run: bundle install && bundle exec jekyll build
```

| Özellik | GitLab CI/CD | GitHub Actions |
| :--- | :--- | :--- |
| **Entegrasyon** | Built-in, tek parça | Modüler, Marketplace tabanlı |
| **Runner Kontrolü** | Çok esnek (Self-hosted öncelikli) | Kolay ama kısıtlı (Hosted öncelikli) |
| **Syntax** | Merkezi `.gitlab-ci.yml` | Dağınık `.github/workflows/` |
| **Deployment** | Merge Trains, Environment Monitoring | Environment Protection, Deployment Jobs |

![DevOps Pipeline](/assets/img/posts/devops_pipeline_visualization.png)
*Modern bir CI/CD hattının görselleştirilmesi.*

### GitLab Klasöründeki Ekstralar
Migration sırasında GitLab tarafında şu ek dosyalar ortaya çıktı:
- `.gitlab-ci.yml`: Tüm build, test ve deploy süreçlerini yöneten ana kumanda merkezi.
- `public/`: GitLab Pages, GitHub'ın `_site` veya `gh-pages` branch'i yerine kodun içindeki `public` klasörünü yayınlamak ister.
- `artifacts`: GitLab'da build edilen dosyaların pipeline sonrasında saklanma mantığı daha belirgindir.
- **Shared Runners:** GitLab'ın sağladığı ücretsiz runner'lar, GitHub'a göre bazen daha yavaş ama ayarları daha şeffaf.

## 3. GitHub'a Geri Dönüş: Neden?

Hesabım tekrar açıldığında neden GitHub'a döndüm?
1. **Community:** Open source projelerin kalbi hala burada atıyor. PR göndermek veya Issue açmak GitHub'da nefes almak kadar doğal.
2. **GitHub Actions Marketplace:** Bir şey yapmak istediğinizde, muhtemelen birisi onun için bir "Action" yazmıştır. GitLab'da bu süreç genelde manuel script yazmaya dayanıyor.
3. **Integration:** Özellikle VS Code ve Copilot gibi araçlarla olan kusursuz uyumu, geliştirme hızını inanılmaz artırıyor.
4. **Dependabot:** Güvenlik açıklarını tarayıp otomatik PR açma konusunda GitHub hala bir adım önde.

## 4. Teknik Kurulum Rehberi (Sıfırdan)

Eğer kendi sitenizi sıfırdan kurmak istiyorsanız, işte checklist:

### Adım 1: Repo Yapısı
- User site için: `<username>.github.io` (Bu repo özel isimdir, GitHub doğrudan `master` branch'ini yayınlar).
- Project site (blog) için: `blog.<username>.github.io`.

### Adım 2: Chirpy Blog Setup
```bash
# Chirpy Starter'ı clone'layın
git clone https://github.com/cotes2020/chirpy-starter.git yourblog
cd yourblog

# Init script çalıştırın (GitHub dışına kuruyorsanız --no-gh ekleyin)
bash tools/init.sh

# Lokal test
bundle install
bundle exec jekyll s
```

### Adım 3: Disaster Recovery Stratejisi
Eğer bir gün sizin de başınıza platform değişikliği gelirse:
- **Mirroring:** İki tarafta da repo tutun (GitHub ana, GitLab yedek).
- **Generic CI Scriptleri:** CI scriptlerinizi (build komutları gibi) platformdan bağımsız shell scriptlerinde tutun. Böylece hem `.gitlab-ci.yml` hem de `deploy.yml` sadece bu scriptleri çağırmış olur.
- **Backup:** Sadece buluta güvenmeyin. Haftalık `git clone --mirror` ile lokal bir fiziksel yedeğiniz olsun.

## 5. Sonuç ve Dersler

GitLab macerası bana GitLab'ın ne kadar güçlü bir kurumsal araç olduğunu gösterdi. Esnekliği, özellikle kurumsal çapta kendi server'larınızı yönetiyorsanız rakipsiz. Ancak bireysel projeler ve blog yönetimi için GitHub Actions'ın hızı ve GitHub'ın sosyal ekosistemi hala rakipsiz.

En büyük dersim: **Altyapınızı asla tek bir platforma %100 bağımlı kılmayın.** Kodunuz sizin malınızdır, platformlar ise sadece onu dünyaya açan birer araçtır. Yarın GitHub'ın başına bir şey gelse, 5 dakika içinde GitLab pipeline'ımı ayağa kaldırabilecek durumda olmalıyım.

---
**BKZ:** 
- [Git Stratejileri ve Branch Yönetimi](/posts/git-is-akisi-branch-stratejileri/)
- [Modern Web Mimarisi](/posts/modern-web-arayuzu-css-javascript/)
- [SEO ve Performans Optimizasyonu](/posts/veritabani-baglanti-havuzu-optimizasyon/)

![Furkan Köykıran](/assets/img/avatar.png)
*Furkan Köykıran - Senior Software Engineer*
*Not: Bu yazı, 02.02.2026 tarihindeki büyük göç anısına yazılmıştır.*
