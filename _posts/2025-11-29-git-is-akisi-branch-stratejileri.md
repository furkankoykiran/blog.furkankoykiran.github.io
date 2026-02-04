---
title: "Git Stratejileri: Takım Çalışması ve Trunk Based Development"
description: "GitFlow vs Trunk Based Development karşılaştırması. Büyük takımlarda Merge Hell'den kurtulma, CI/CD süreçleri ve Git Bisect ile hata ayıklama teknikleri."
date: "2025-11-29 10:00:00 +0300"
categories: [Engineering Culture, DevOps]
tags: [git, methodology, ci-cd, teamwork, trunk-based]
image:
  path: /assets/img/posts/gitflow-branching-model.png
  alt: "GitFlow vs Trunk Based Development"
---

Bir Cuma akşamı, tüm ekip "Deploy" butonuna basmak için bekliyor.
Ama o da ne? 3 farklı `feature` branch'ini `develop` branch'ine birleştirmeye çalışırken yüzlerce "Merge Conflict" çıkıyor.
Kodlar birbirine girmiş, kimin neyi sildiği belli değil.
Tanıdık geldi mi? Buna "Merge Hell" (Birleştirme Cehennemi) diyoruz.
Yazılım geliştirmek sadece kod yazmak değildir; o kodu binlerce satırlık bir havuzda, onlarca kişiyle birlikte yönetebilme sanatıdır.

Bu yazıda, GitFlow'dan neden vazgeçtiğimizi ve Silikon Vadisi standardı olan **Trunk Based Development**'a neden geçtiğimizi anlatacağım.

![GitFlow Diagram](/assets/img/posts/gitflow-branching-model.png)
*Geleneksel GitFlow: Uzun ömürlü feature ve develop branchleri.*

## 1. GitFlow: Güvenli Ama Hantal

2010'larda popüler olan bu model, yazılımı "Fabrika Üretimi" gibi görür.
Bir **Develop** branch'iniz vardır. Herkes buraya merge eder. Ayda bir **Release** branch açılır.
**Avantajı:** Production (Master) çok stabildir.
**Dezavantajı:** Branchler çok uzun yaşar (Long-lived branches).
Siz 2 hafta boyunca `feature-x` üzerinde çalışırken, iş arkadaşınız `feature-y` üzerinde çalışır ve `develop` branch'inin yapısını tamamen değiştirir.
İkiniz birleşmeye kalktığınızda kaos çıkar. Modern CI/CD dünyasında bu model hantaldır.

## 2. Trunk Based Development: Hızın Formülü

Google, Netflix gibi devlerin kullandığı yöntemdir.
Tek kural: **Herkes Ana Branch'e (Trunk/Main) kod atar.**
Feature branch diye bir şey yoktur (varsa da ömrü en fazla 24 saattir).
"Ama yarım kalan kod ne olacak?" -> **Feature Flags**.
Kod Main branch'e girer ama bir `if (FEATURE_X_ENABLED)` bloğu içindedir. Production'a gitse bile kullanıcı görmez.

**Neden Üstün?**
1.  **Sürekli Entegrasyon (CI):** Kodlar günde 50 kere birleşir. Çatışmalar küçük olur ve anında çözülür.
2.  **Hız:** "Release günü" stresi yoktur. Kod hazırsa deploy edilir.
3.  **Refactoring:** Kod tabanı sürekli günceldir.

## 3. High-Velocity Teams: Merge Queue ve Monorepo

Ekip 100 kişiye çıktığında TBD bile yetersiz kalabilir.
Aynı anda 20 kişi `main` branch'e merge etmeye çalışırsa, CI pipeline'ları tıkanır.
GitHub'ın **Merge Queue** özelliği burada devreye girer.
Siz "Merge" dediğinizde kod hemen main'e girmez, bir kuyruğa girer.
GitHub arka planda geçici bir branch yaratır, sizin kodunuzu o anki main ile birleştirir, testleri çalıştırır ve geçerse main'e atar.
Sıradaki diğer kodu da sanki sizin kodunuz varmış gibi test eder.
Böylece "Broken Master" sorunu matematiksel olarak imkansızlaşır.

### Monorepo vs Polyrepo: Büyük Dilenma

Google, Meta ve Uber neden tüm kodlarını tek bir devasa repoda (Monorepo) tutuyor?

*   **Polyrepo (Her mikroservis ayrı repo):**
    *   *Avantaj:* Bağımsız deploy, temiz tarihçe.
    *   *Dezavantaj:* Ortak kütüphane güncellemesi tam bir kâbus. `utils` kütüphanesinde bir bug fix yaptığınızda, onu kullanan 50 servisi tek tek güncellemeniz gerekir.

*   **Monorepo (Tek Dev Depo):**
    *   *Avantaj:* Tek commit ile (`feat: update user model`) hem backend'i, hem frontend'i hem de mobile app'i güncelleyebilirsiniz. "Atomic Commit" gücü.
    *   *Dezavantaj:* Tooling şarttır (Nx, Turborepo, Bazel). Yoksa `güt pull` yapmak 10 dakika sürer.

Senior bir mühendis olarak, ekibin büyüklüğüne göre bu kararı vermelisiniz. 5 kişilik ekipte Monorepo gereksiz karmaşıklıktır (Overengineering). 100 kişilik ekipte Polyrepo ise iletişim krizidir.

![Trunk Based Development](/assets/img/posts/git-merge-rebase-comparison.png)
*TBD: Kısa ömürlü branchler ve feature flag kullanımı.*

## 3. Otomasyon: Commitlint ve Husky

Disiplin, insanlara değil araçlara bırakılmalıdır.
"Commit mesajın kurallara uymuyor" diye code review reddetmek zaman kaybıdır.
Bunu **Husky** ve **Commitlint** ile otomatize edin.

```bash
# .pre-commit-config.yaml örneği
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.4.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-added-large-files # Büyük dosyaları engelle
```

Developer `git commit` dediği anda bu hooklar çalışır. Eğer kodda style hatası varsa veya commit mesajı "fix: bug" yerine "bug fixed" ise (standarda uymuyorsa) commit'i reddeder.

## 4. Debugging Masterclass: Git Bisect

Elinizde 1000 commitlik bir proje var ve bir "Bug" oluştu. Ama bug'ın ne zaman girdiğini bilmiyorsunuz.
Tek tek 1000 commit'i gezmek yerine **Binary Search** yapın.

```bash
# 1. Bisect başlat
git bisect start

# 2. Şu anki halin bozuk olduğunu söyle
git bisect bad

# 3. Bildiğiniz en son çalışan commit'i işaretleyin (örn: v1.0 tag'i)
git bisect good v1.0

# Git şimdi tam ortadaki commit'e geçer. Test edin.
# Çalışıyorsa 'git bisect good', bozuksa 'git bisect bad' deyin.
# 10 adımda hatayı çıkaran commit'i bulursunuz.
```

Bu özellik, "Dün çalışıyordu bugün çalışmıyor" durumlarında hayat kurtarır.

![Merge Conflict Resolution](/assets/img/posts/git-cherry-pick-diagram.jpg)
*Conflict yönetimi: Paniğe kapılmadan "Incoming" ve "Current" değişiklikleri birleştirmek.*

## 5. Rebase vs Merge: Kirli Tarihçe Sorunu

Bir PR merge edilirken "Merge Commit" mi atılmalı yoksa "Rebase and Merge" mi yapılmalı?
GitFlow "Merge Commit" sever, tarihçede "Burada birleşti" izi kalır.
Trunk Based ise "Rebase" sever, doğrusal (Linear) bir tarihçe ister.
Benim tercihim: **Squash and Merge**.
Feature branch'indeki 50 tane "typo fix", "wip" commit'ini tek bir temiz commit haline getirip main'e ekler.

## 6. Semantik Versiyonlama

Git loglarına baktığınızda `fix`, `update`, `bug` yazan 50 tane commit görüyorsanız, o projede disiplin yoktur.
**Conventional Commits** standardını zorunlu kılın:
-   `feat(auth): add google sso login` -> Minor versiyon artırır (1.1.0 -> 1.2.0)
-   `fix(payment): resolve timeout` -> Patch versiyon artırır (1.1.0 -> 1.1.1)

Bu disiplin sayesinde `Semantic Release` gibi araçlar, commit mesajlarına bakıp otomatik Changelog dosyası oluşturabilir ve versiyon numarasını basabilir.

## 7. Sık Yapılan Hatalar (Anti-Patterns)

1.  **Dangling Commits:** `git reset --hard` ile sildiğiniz commitlerin kaybolduğunu sanmayın. `git reflog` komutu, başvurduğunuz (HEAD'in uğradığı) tüm noktaları tutar. Yanlışlıkla sildiğiniz kodları buradan kurtarabilirsiniz.
2.  **Repo Şişkinliği:** `.git` klasörü 2GB olmuşsa muhtemelen birisi yanlışlıkla `.sql` dump veya `.mp4` video commitlemiştir. Tarihçeden büyük dosyayı silmek için `git-filter-repo` (eski BFG Repo-Cleaner) kullanın.
3.  **Secret Leak:** AWS Key veya DB şifresini yanlışlıkla commitleyip Github'a pushlamak. Botlar bunu 3 saniyede bulur. Çözüm: `git-secrets` veya `trufflehog` ile push öncesi tarama yapın.
4.  **Force Push Master:** Production'ı bozmanın en hızlı yolu. Master branch'i mutlaka korumalı (Protected Branch) olmalı ve sadece PR ile kod girmelidir.

## Sık Sorulan Sorular (SSS)

**S: Junior geliştirici direkt Master'a yazabilir mi?**
C: TBD'de evet, ama iyi bir Code Review (PR) sürecinden geçmelidir. Pair programming ile risk azaltılabilir.

**S: Hotfix nasıl çıkılır?**
C: TBD kullanıyorsanız, `fix` commit'ini main'e atıp hemen deploy edersiniz. GitFlow'da `hotfix` branch'i açıp hem Master hem Develop'a merge etmeniz gerekir.

**S: Büyük bir özellik (Epic) aylarca sürerse ne olacak?**
C: Feature Flag kullanın. Kod production'da olsun ama kapalı olsun. Branch'i aylarca açık tutmak felakettir.

## Terimler Sözlüğü (Glossary)

*   **Repository (Repo):** Kodun saklandığı depo.
*   **Rebase:** Bir dalın başlangıcını (Base) değiştirmek. Tarihçeyi temiz tutar.
*   **PR/MR:** Kod inceleme talebi.
*   **Cherry-pick:** Başka bir branch'ten sadece belirli bir commit'i çalmak.
*   **Stash:** Kirli çalışma alanını (Worktree) geçici olarak hafızaya almak.

## Sonuç

Eğer bir SaaS (Web Servisi) geliştiriyorsanız ve günde birden fazla deploy hedefiniz varsa **Trunk Based Development** tek yoldur.
Korkutucu gelebilir ama Feature Flag'lere güvenin.
Her stratejinin bir maliyeti vardır. Önemli olan maliyeti (Merge conflict mi, Test yazmak mı?) nerede ödeyeceğinizi seçmektir.
Takımınızın kültürü, Git stratejinizi belirler.

## Bonus: Git Verimlilik Araçları

Terminalinizi hızlandırmak için şunları kurun:
1.  **`git-delta`:** Diff çıktılarını renklendirir ve okunaklı yapar. (Syntax highlighting for git).
2.  **`lazygit`:** Terminal içinde görsel arayüz (TUI) sağlar. Hızlıca stage/unstage yapmak için harikadır.
3.  **`oh-my-zsh` git plugin:** Branch ismini ve durumunu prompt'ta gösterir.


![Git Branching Strategies](/assets/img/posts/gitflow-branching-model.png)
*Stratejilerin Görsel Karşılaştırması.*