---
title: "GitHub Actions Dakikaları Bitti: Kendi Sunucuma Self-Hosted Runner Kurdum"
description: "Aylık 3.000 dakikalık ücretsiz GitHub Actions kotasını tüketince iki seçenek kalıyor: planı yükselt ya da elimdeki sunucuyu runner yap. İkincisini yaptım."
date: 2026-05-06 12:00:00 +0300
categories: [DevOps, GitHub Actions, CI/CD]
tags: [github-actions, self-hosted-runner, linux, systemd, ci-cd]
image:
  path: /assets/img/posts/2026-05-06-github-actions-self-hosted-runner/banner.png
  alt: "GitHub Actions Self-Hosted Runner Kurulumu"
---

GitHub, private repo'lar için ayda 3.000 ücretsiz Actions dakikası veriyor.

Bu çok gibi görünüyor — ta ki her PR'da, her push'ta, her küçük değişiklikte çok adımlı CI pipeline'ları çalıştırmaya başlayana kadar. O noktada çok değil. O noktada sayacın düşüşünü izleyip "bu ay kaç tane daha commit atabilirim" diye sessizce hesaplıyorsun.

Bariz çözüm: ücretli plana geç. Diğer çözüm: zaten elimde bir sunucu var.

Sunucum vardı. Self-hosted runner olarak kurdum. Nasıl göründüğünü anlattım.

---

## Neden Ödeme Yapmadım?

Güçlü bir prensibim var diyemem. Sunucu zaten çalışıyordu, işler kuyruğa giriyordu, kota sıfırdaydı. En az dirençli yol, elimdeki makineyi GitHub'a göstermekti.

Self-hosted runner'ların bir avantajı da kendi ortamında çalışması: bağımlılıklar zaten kurulu, her run'da "Python'u kur, bağımlılıkları kur, cache'i bekle" yok. İlk run yavaş (Poetry her şeyi sıfırdan kuruyor); sonraki run'lar virtualenv yerinde kaldığı için belirgin şekilde hızlı.

---

## Kurulum Adımları

### 1. Ayrı Bir Kullanıcı Oluştur

Runner'ı root olarak çalıştırma. Bunun için bir sistem kullanıcısı oluştur:

```bash
useradd -r -m -d /opt/github-runner -s /bin/bash \
  -c "GitHub Actions Runner" github-runner
```

`-r` ile sistem hesabı (UID < 1000), `-m` ile home dizini, `-d` ile hedef `/opt/github-runner`. Varsayılan olarak sudo erişimi yok, shell girişi yok. Tam istediğin şey.

### 2. Runner Paketini İndir ve Doğrula

En güncel sürümü [github.com/actions/runner/releases](https://github.com/actions/runner/releases) adresinde bul, ardından:

```bash
cd /opt/github-runner

curl -o actions-runner-linux-x64-2.334.0.tar.gz -L \
  https://github.com/actions/runner/releases/download/v2.334.0/actions-runner-linux-x64-2.334.0.tar.gz

# SHA256 doğrula (hash release body'sinde yazıyor)
echo "048024cd2c848eb6f14d5646d56c13a4def2ae7ee3ad12122bee960c56f3d271  actions-runner-linux-x64-2.334.0.tar.gz" | sha256sum -c

tar xzf actions-runner-linux-x64-2.334.0.tar.gz
```

SHA256 doğrulaması burada önemli. Sunucuna ciddi erişim hakkı tanıyacak bir executable indiriyorsun.

### 3. Kayıt Token'ı Al

Repo → Settings → Actions → Runners → New self-hosted runner. GitHub bir kayıt token'ı gösterecek. Kopyala — yaklaşık bir saatte süresi doluyor.

### 4. Runner'ı Kaydet

```bash
sudo -u github-runner ./config.sh \
  --url https://github.com/kullanici-adi/repo-adi \
  --token KAYIT_TOKENI \
  --name prod-server-01 \
  --labels "self-hosted,linux,prod" \
  --unattended
```

Bunu root olarak değil `github-runner` kullanıcısı olarak çalıştır. `--labels` ile workflow YAML'ında bu runner'ı özellikle hedefleyebilirsin. `--unattended` interaktif soruları atlar.

Çalışınca şunu görürsün:

```
√ Connected to GitHub
√ Runner successfully added
√ Settings Saved
```

### 5. systemd Servisi Olarak Kur

```bash
cd /opt/github-runner
sudo ./svc.sh install github-runner
sudo ./svc.sh start
```

Çalışıp çalışmadığını kontrol et:

```bash
sudo systemctl status actions.runner.*.service
```

`active (running)` görmen gerekiyor. Log'ları da kontrol et:

```bash
sudo journalctl -u actions.runner.*.service -n 50 --no-pager
```

Aradığın satır: `Listening for Jobs`. Bunu görünce runner ayakta ve bekliyor demektir.

![GitHub Actions Runner Kurulumu](/assets/img/posts/2026-05-06-github-actions-self-hosted-runner/runner-setup.png)
*Runner kayıt akışı — token'ı aldıktan sonra oldukça düz.*

---

## Güvenlik Sertleştirmesi

Tamamdır demeden önce şunları yapman iyi olur:

**Dosya izinleri.** Runner dizini yalnızca runner kullanıcısına ait olmalı:

```bash
chown -R github-runner:github-runner /opt/github-runner
chmod 700 /opt/github-runner
```

**Runner kullanıcısının yetkilerini sınırla.** İşlerin gerçekten ihtiyacı olmadıkça sudoers'a ekleme. Gerekiyorsa, tam sudo yerine belirli komutlara izin veren dar kapsamlı bir kural yaz.

**Hassas repo'lar için geçici runner düşün.** Özellikle public repo'larda [ephemeral runner](https://docs.github.com/en/actions/hosting-your-own-runners/managing-self-hosted-runners/about-self-hosted-runners#self-hosted-runner-security) her job'u temiz bir ortamda çalıştırıp otomatik kayıt siler. Private repo, kendi donanımın, persistent runner genellikle iyi — ama trade-off'ların farkında ol.

---

## Workflow Dosyalarını Güncelle

Her workflow YAML'ındaki `runs-on` satırını değiştir:

```yaml
# Önce
jobs:
  test:
    runs-on: ubuntu-latest

# Sonra
jobs:
  test:
    runs-on: [self-hosted, linux, prod]
```

`runs-on`'daki label'lar, kayıt sırasında `--labels` ile verdiğin değerlerle eşleşmeli. Birden fazla runner farklı label'larla (`prod`, `staging` gibi) varsa, tam hedefleme yapabilirsin.

---

## İşe Yaradığı An

PR gönderdim, workflow sayfasını izledim:

```
Running job: Backend Lint (ruff)
```

Azure'da bir yerde dönen GitHub hosted VM'de değil. Karşımda duran makinede. İlk run biraz uzun sürdü — Poetry her şeyi sıfırdan kurdu. Sonrasında run'lar belirgin şekilde hızlandı çünkü virtualenv run'lar arasında kaldı.

---

## Kısa Karşılaştırma

| | GitHub-Hosted | Self-Hosted |
|---|---|---|
| **Maliyet** | Ücretsiz (ayda 3k dakikaya kadar) | Sunucunun elektrik faturası |
| **Bakım** | Yok | Runner güncellemeleri, OS yamaları |
| **Hız** | Tutarlı (plana göre) | İlk run'dan sonra hızlı (yerel bağımlılıklar) |
| **İzolasyon** | Her run'da temiz VM | Run'lar arasında ortak dosya sistemi |
| **Ağ** | GitHub'ın ağı | Kendi ağın (private altyapı için avantajlı) |

Biri evrensel olarak daha iyi değil. Ücretsiz kotanın altındaysan GitHub-hosted en doğru tercih. Kotayı tüketince, atıl sunucu kapasiten varsa self-hosted mantıklı.

---

## Sonuç

Kurulum baştan sona yaklaşık 20 dakika sürüyor: kullanıcı oluştur, runner indir, kaydet, servisi kur, workflow dosyalarını güncelle. Zor kısım kayıt token'ı — hızla süresi dolduğundan, üretmeden önce `config.sh` komutunu hazır bulundur.

Bir kez çalışmaya başlayınca arka plana düşüyor. İşler kuyruğa giriyor, runner alıyor, log'lar akıyor. CI pipeline aynı şekilde çalışıyor — sadece GitHub'ın donanımı yerine seninki üzerinde.

Sayaç sıfırlandı. Yani kotam 3.000'e döndü. Sayaç artık kendi sunucumda.

---

![Furkan Köykıran](/assets/img/avatar.png)
*Furkan Köykıran - Senior Software Engineer*

---

## Diğer Yazılar

- [freqtrade-mcp: Bota Claude Üzerinden Sor](/posts/freqtrade-mcp-server-tr/)
- [CLI-Anything'e Tarayıcı Otomasyonu: İlk MCP Backend Deseni](/posts/cli-anything-browser-automation-katkim/)
- [awesome-trending-repos: GitHub Trending için Modern Web Arayüzü](/posts/awesome-trending-repos-web/)
