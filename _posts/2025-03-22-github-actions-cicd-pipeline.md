---
layout: post
title: "GitHub Actions Enterprise: OIDC, Reusable Workflows ve Maliyet Kültürü"
date: 2025-03-22 15:00:00 +0300
categories: [DevOps, CI/CD, Security]
description: "50 repoyu tek tek yönetmekten bıktınız mı? OIDC ile users credential-less deploy, Reusable Workflows ile merkezi yönetim ve maliyet düşürme teknikleri."
image: assets/img/posts/github-actions-cicd-pipeline-diagram.png
---

"Merhaba Dünya" seviyesinde bir GitHub Actions pipeline'ı yazmak 5 dakika sürer. Ama 50 mikroservislik bir organizasyonu yönetiyorsanız, o basit `.yml` dosyaları bir süre sonra yönetilemez bir kaosa (YAML Hell) dönüşür.

Bu yazıda, hobi projelerinden çıkıp **Enterprise DevOps** dünyasına gireceğiz. Konumuz: Güvenlik, Ölçeklenebilirlik ve Para.

## 1. Copy-Paste Kültürüne Son: Reusable Workflows

Her mikroservisinizde `docker build` ve `deploy` adımlarını kopyalayıp yapıştırıyorsanız, yanlış yapıyorsunuz. Docker versiyonunu güncellemeniz gerektiğinde 50 repo gezmeniz gerekir.

Çözüm: **Reusable Workflows**.
Merkezi bir repo (`infra-workflows`) açın ve `build.yml` oluşturun:

```yaml
# infra-workflows/.github/workflows/docker-build.yml
name: Reusable Docker Build
on:
  workflow_call:
    inputs:
      image_name:
        required: true
        type: string
    secrets:
      registry_password:
        required: true

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build and Push
        run: |
          docker build -t ${{ inputs.image_name }} .
          # ...
```

Artık servislerinizde sadece şunu çağırırsınız:

```yaml
# service-a/.github/workflows/main.yml
jobs:
  call-build:
    uses:my-org/infra-workflows/.github/workflows/docker-build.yml@v1
    with:
      image_name: "service-a"
    secrets: inherit # Org secretlarını otomatik aktar
```
Böylece pipeline mantığını tek bir yerden yönetirsiniz.

## 2. AWS Key'lerini Çöpe Atın: OIDC (OpenID Connect)

Eğer Github Secret'larınızda `AWS_ACCESS_KEY_ID` varsa, büyük bir güvenlik riski taşıyorsunuz. O key sızarsa hackerlar AWS hesabınızı boşaltır. Key rotasyonu yapmak ise tam bir işkencedir.

GitHub Actions ve AWS artık **OIDC** ile birbirine "Passwordless" bağlanabilir.

1.  AWS'de bir **Role** oluşturun ve GitHub'ın OIDC provider'ına güvenmesini söyleyin.
2.  Rolün Trust Policy'sinde `sub:repo:my-org/my-repo:ref:refs/heads/main` diyerek sadece bu reponun bu branch'ine izin verin.

```yaml
permissions:
  id-token: write # Bu satır şart!
  contents: read

steps:
  - name: Configure AWS Credentials
    uses: aws-actions/configure-aws-credentials@v4
    with:
      role-to-assume: arn:aws:iam::1234567890:role/GitHubDeployRole
      aws-region: eu-central-1
```
Artık statik key yok. GitHub, AWS'den 1 saatlik geçici token (STS) alır. %100 güvenli, baş ağrısız.

## 3. Maliyet Optimizasyonu: Parayı Sokağa Atmayın

GitHub Actions bedava değildir (limitler aşılınca). Dakikası para yazar.

**1. Concurrency Groups:**
Bir PR'a commit attınız, build başladı. 10 saniye sonra bir commit daha attınız. İlk build hala çalışıyor ama sonucu artık çöp.
Bunu engelleyin:

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```
Bu ayar, yeni commit gelince eskini iptal eder.

**2. Timeout Tanımlayın:**
Bazen bir step (örn: `pip install`) sunucu hatası yüzünden takılır. Default timeout 6 saattir! Kredi kartınız yanar.

```yaml
steps:
  - run: npm install
    timeout-minutes: 5 # 5 dakikada bitmezse öldür
```

**3. Cache Kullanın:**
Node_modules veya pip cache'ini her seferinde indirmeyin. `actions/setup-node` artık build-in caching destekliyor:

```yaml
- uses: actions/setup-node@v4
  with:
    node-version: 18
    cache: 'npm'
```

## 4. Self-Hosted Runners: Ne Zaman?

GitHub'ın bulut sunucuları (Runners) bazen yavaş kalabilir veya VPC içindeki veritabanınıza erişemez.
Bu durumda EC2 veya Kubernetes üzerine kendi runner'ınızı kurabilirsiniz (`actions/runner-controller`).

| Özellik | GitHub Hosted | Self-Hosted |
| :--- | :--- | :--- |
| **Bakım** | Sıfır | Yüksek (OS update, disk temizliği) |
| **Güvenlik** | Ephemeral (Her iş sonrası silinir) | Persistent (Dikkatli olunmalı) |
| **Maliyet** | Dakika başı | Sabit sunucu maliyeti |

**Uyarı:** Public repolarda asla Self-Hosted runner kullanmayın. Birisi PR açıp `minning-crypto.sh` çalıştırabilir!


## 5. Yönetişim: Branch Protection ve Environments

Pipeline yazdınız ama junior geliştirici `main` branch'e direk force-push attı. Pipeline'ın ne anlamı kaldı?

GitHub Settings altından **Branch Protection Rules** koymalısınız:
1.  **Require status checks to pass:** "Test" ve "Lint" jobları yeşil olmadan Merge butonu aktif olmasın.
2.  **Require pull request reviews:** En az 1 senior onayı şart koşun.

**Environments ve Manuel Onay:**
Production deployment'ı tamamen otomatize etmek korkutucu olabilir. GitHub **Environments** özelliği ile "Production" ortamına deploy çıkmadan önce "Müdür Onayı" isteyebilirsiniz.

```yaml
jobs:
  deploy-prod:
    environment: production # GitHub UI'da tanımlı ortam
    runs-on: ubuntu-latest
    steps: ...
```
Pipeline bu adıma gelince durur, Slack'ten bildirim atar, yetkili kişi "Approve" butonuna basınca devam eder.

## 6. Temiz Kod: Composite Actions

Reusable Workflows (büyük süreçler) ile Composite Actions (küçük adımlar) karıştırılır.
Eğer 5-6 adımlık bir scriptiniz varsa (örn: "Setup Java + Maven + Settings.xml"), bunu repo içinde gizleyin.

`.github/actions/setup-java-maven/action.yml`:
```yaml
name: 'Setup Java & Maven'
description: 'Standard Java setup'
runs:
  using: "composite"
  steps:
    - uses: actions/setup-java@v4
      with:
        java-version: '17'
    - run: mvn -B -s settings.xml
      shell: bash
```

Ana workflow tertemiz olur:
```yaml
steps:
  - uses: ./.github/actions/setup-java-maven
```
Bu, pipeline kodunuzu "Spagetti" olmaktan kurtarır.

