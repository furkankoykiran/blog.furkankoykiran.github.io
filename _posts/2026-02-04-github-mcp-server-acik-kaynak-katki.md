---
title: "GitHub'ın Resmi MCP Server Projesine Katkıda Bulundum: Bir Açık Kaynak Hikayesi"
description: "GitHub MCP Server projesine nasıl katkıda bulundum? Discussion oluşturma özelliğinin eklenmesi, hesap askıya alma sürprizi ve iki ay sonra döndüğümde karşılaştığım 85 dosyalık conflict."
date: "2026-02-04 14:00:00 +0300"
categories: [DevOps, Open Source]
tags: [github, open-source, pull-request, git, mcp, golang]
image:
  path: /assets/img/posts/github-pr-conflict-resolution.png
  alt: "GitHub Pull Request Conflict Resolution"
---

Açık kaynak dünyasına ilk adımımı attığımda, büyük projelere katkı yapmanın ne kadar zor olduğunu düşünürdüm. "Bu kadar büyük bir kod tabanını nasıl anlayacağım?", "Maintainer'lar benim kodum için neden zaman ayırsın?" gibi sorular kafamda dönüp dururdu. Ama işin aslı, bir yerden başlamak lazım.

Bu yazıda, GitHub'ın resmi `github-mcp-server` projesine nasıl katkıda bulunduğumu anlatacağım. Eksik bir özelliği fark etmekten, PR açmaya, hesabımın askıya alınmasına ve iki ay sonra karşılaştığım devasa conflict'leri çözmeye kadar tüm süreci paylaşacağım.

## GitHub MCP Server ile Tanışma

2025'in sonlarına doğru AI asistanları ve bunların dış dünyayla nasıl entegre edildiğini araştırıyordum. Anthropic'in Model Context Protocol (MCP) standardı dikkatimi çekti - AI sistemlerinin güvenli ve standart bir şekilde harici araçlara erişmesini sağlayan açık bir protokol.

![MCP Mimarisi](/assets/img/posts/mcp-architecture-diagram.png)
*Model Context Protocol mimarisi - AI asistanları dış dünyaya bağlayan köprü.*

GitHub da bu trendin farkındaydı ve resmi bir MCP server oluşturmuştu: `github/github-mcp-server`. Bu proje, Copilot gibi AI asistanlarının GitHub API'lerini kullanmasını sağlıyor. Dosya okuma, issue oluşturma, PR yönetimi, branch işlemleri... Neredeyse GitHub'da yapabileceğiniz her şeyi MCP üzerinden gerçekleştirebiliyorsunuz.

Projeyi incelemeye başladım. Go ile yazılmış, GraphQL ve REST API'lerini kullanıyor. Kod yapısı oldukça temiz, toolset'ler mantıklı bir şekilde gruplandırılmış.

## Eksik Bir Parça: CreateDiscussion

Discussions toolset'ini incelerken bir şey dikkatimi çekti. `ListDiscussions` var, `GetDiscussion` var, `ListDiscussionCategories` var... ama `CreateDiscussion` yok.

Yani AI asistanı mevcut discussion'ları okuyabiliyor ama yeni bir discussion oluşturamıyor. Bu bana mantıksız geldi. Eğer bir kullanıcı "bu proje hakkında bir discussion başlat" derse, AI "üzgünüm yapamıyorum" mu diyecek?

Issue tracker'a baktım, açık bir issue yok. O zaman ben açayım dedim.

### Issue #1517: Açık Kaynak'ın Kapısını Çalma

```markdown
## Feature Request: Add CreateDiscussion Tool

### Description
Add a `create_discussion` tool to the discussions toolset.

### Use Cases
- Creating Q&A threads
- Starting community discussions
- Announcing features
...
```

Issue'yu gönderdikten birkaç saat sonra bir maintainer yanıt verdi: "Good idea, would you like to work on this?"

Tabii ki evet.

## İlk PR: #1519

Issue onaylandıktan sonra fork'ladım, yeni bir branch açtım ve çalışmaya başladım. İşin güzel yanı, projedeki diğer tool'lar bana referans oluyordu. `discussions.go` dosyasındaki mevcut fonksiyonları inceledim, pattern'ları anladım.

o zamanki implementasyon şu şekildeydi:

```go
func CreateDiscussion(getGQLClient GetGQLClientFn, t translations.TranslationHelperFunc) (mcp.Tool, mcp.ToolHandlerFor[map[string]any, any]) {
    return mcp.Tool{
        Name:        "create_discussion",
        Description: t("TOOL_CREATE_DISCUSSION_DESCRIPTION", "Create a new discussion..."),
        InputSchema: &jsonschema.Schema{
            // owner, repo, categoryId, title, body
        },
    },
    func(ctx context.Context, req *mcp.CallToolRequest, args map[string]any) (*mcp.CallToolResult, any, error) {
        // GraphQL mutation ile discussion oluşturma
    }
}
```

GraphQL mutation'ı yazdım, input validation'ları ekledim, edge case'leri düşündüm. Organizasyon seviyesinde discussion oluşturma da desteklenmeli miydi? Evet, bu da ekledim.

Testleri yazdım. `githubv4mock` ile GraphQL response'larını mocklamak biraz uğraştırdı ama sonunda düzgün çalışan testlerim oldu.

PR'ı açtım, detaylı bir description yazdım, review istedim. Artık beklemek kalıyordu.

![GitHub MCP Server](/assets/img/posts/github-mcp-server-preview.png)
*GitHub MCP Server projesi*

## Beklenmedik Felaket

PR'm açıldıktan birkaç gün sonra, bir sabah GitHub'a giriş yapmaya çalıştığımda "Your account has been suspended" mesajıyla karşılaştım.

İlk tepkim panikti. Yanlış bir şey mi yaptım? Spam mı attım farkında olmadan? TOS'u mu ihlal ettim?

Sonra öğrendim: Kullandığım WiFi ağından kaynaklı olarak, hesabım kötüye kullanım şüphesiyle askıya alınmıştı. Muhtemelen aynı IP'den şüpheli aktivite tespit edilmişti ve ben de o ağdaydım.

Appeal sürecine girdim. Form doldurdum, bekledim. E-posta yazdım, bekledim. Günler haftaya, haftalar aya dönüştü. İki ay boyunca tüm repolarım, contributiton'larım, açık PR'larım erişilemez durumda kaldı.

Bu süreçte en çok canımı yakan şey, o PR'dı aslında. İlk kez büyük bir projeye katkı yapıyordum ve tam review aşamasındayken her şey donmuştu.

## İki Ay Sonra: Şok

Sonunda hesabım açıldı. İlk iş: PR'ıma bakmak.

Gördüğüm manzara:

```
This branch has conflicts that must be resolved
```

Ama asıl şok, değişikliklerin boyutuydu. `git fetch upstream` yapıp log'a baktığımda...

**85 dosya değişmişti.**

Proje bu iki ayda ciddi bir refactoring geçirmişti. Sadece birkaç conflict değil, tüm mimari değişmişti.

## Yeni Mimari: Ne Değişmişti?

Değişiklikleri incelemeye başladım. Upstream'deki yeni pattern'lar tamamen farklıydı:

### Eski Yaklaşım (Benim PR'ım)
```go
func CreateDiscussion(getGQLClient GetGQLClientFn, t translations.TranslationHelperFunc) (mcp.Tool, mcp.ToolHandlerFor[...]) {
    return mcp.Tool{...}, func(...) {...}
}
```

### Yeni Yaklaşım (Güncel Main)
```go
func CreateDiscussion(t translations.TranslationHelperFunc) inventory.ServerTool {
    return NewTool(
        ToolsetMetadataDiscussions,
        mcp.Tool{...},
        []scopes.Scope{scopes.Repo},
        func(ctx context.Context, deps ToolDependencies, ...) {...},
    )
}
```

Değişiklikler sadece syntax'la sınırlı değildi:

1. **Factory Pattern**: `NewTool()` fonksiyonu eklenmişti
2. **Return Type**: `inventory.ServerTool` diye yeni bir yapı
3. **Dependency Injection**: `ToolDependencies` ile client erişimi
4. **Scope System**: Her tool için gerekli izinler tanımlanıyordu
5. **Metadata**: Her toolset için üst-veri yapıları

85 dosyanın değişme sebebi buydu - tüm tool'lar bu yeni pattern'a migrate edilmişti.

![Git Rebase Workflow](/assets/img/posts/git-rebase-workflow-opensource.jpeg)
*Open source projelerde rebase stratejisi*

## Strateji: Ne Yapmalı?

İki seçenek vardı:

### Seçenek 1: Interactive Rebase
Commit commit ilerleyip her birinde conflict çözmek. Teoride "doğru" yaklaşım, history temiz kalır.

Ama pratikte? 85 dosya conflict ile bu günler sürebilirdi. Her commit'te aynı pattern değişikliklerini tekrar tekrar yapmak, hem zaman kaybı hem de hata riski.

### Seçenek 2: Hard Reset + Yeniden İmplementasyon
Branch'i upstream/main'e reset edip, kendi değişikliklerimi yeni pattern ile sıfırdan yazmak.

Dezavantajı: Eski commit history'si gidecekti.
Avantajı: Temiz, hızlı, hatasız.

**Kararım:** Seçenek 2.

Neden? Çünkü sonuç önemli, yol değil. PR zaten merge edilmemişti, kimse o commit'lerin history'sine bakmayacaktı. Önemli olan çalışan bir CreateDiscussion tool'u sunmaktı.

```bash
git fetch upstream
git reset --hard upstream/main
# Artık temiz bir slate üzerindeyim
```

## Yeniden Yazma Süreci

Sıfırdan başlamak ilk bakışta korkutucu gelebilir. Ama aslında avantajlı yanları da var:

### 1. Güncel Örnekler
Artık elimde yüzlerce güncel tool vardı. Nasıl yapılması gerektiğini görmek için sadece `labels.go` veya `issues.go` açmam yeterliydi.

### 2. Pattern'ı Anlama
İlk PR'da sadece mevcut kodu taklit etmiştim. Şimdi yeni pattern'ı gerçekten anlamam gerekiyordu. `NewTool()` ne yapıyor? `ToolDependencies` neden var? `scopes` sistemi ne sağlıyor?

Bu soruların cevaplarını bulmak, beni daha iyi bir contributor yaptı diyebilirim.

### 3. Daha Temiz Kod
İkinci seferde yazdığım kod, ilkinden daha iyiydi. Edge case'leri daha iyi düşündüm, error handling daha sağlam oldu.

İşte yeni implementasyonun özeti:

```go
func CreateDiscussion(t translations.TranslationHelperFunc) inventory.ServerTool {
    return NewTool(
        ToolsetMetadataDiscussions,
        mcp.Tool{
            Name:        "create_discussion",
            Description: t("TOOL_CREATE_DISCUSSION_DESCRIPTION", "Create a new discussion..."),
            Annotations: &mcp.ToolAnnotations{
                Title: t("TOOL_CREATE_DISCUSSION_USER_TITLE", "Create discussion"),
            },
            InputSchema: &jsonschema.Schema{
                Type: "object",
                Properties: map[string]*jsonschema.Schema{
                    "owner":      {Type: "string", Description: "Repository owner"},
                    "repo":       {Type: "string", Description: "Repository name (optional for org)"},
                    "categoryId": {Type: "string", Description: "Category ID"},
                    "title":      {Type: "string", Description: "Discussion title"},
                    "body":       {Type: "string", Description: "Discussion body"},
                },
                Required: []string{"owner", "categoryId", "title", "body"},
            },
        },
        []scopes.Scope{scopes.Repo},
        func(ctx context.Context, deps ToolDependencies, req *mcp.CallToolRequest, args map[string]any) (*mcp.CallToolResult, any, error) {
            // Implementation using deps.GetGQLClient(ctx)
        },
    )
}
```

Testleri de güncelledim. Yeni test pattern'ları:
- `tool.InputSchema.(*jsonschema.Schema)` ile type assertion
- `githubv4mock.DataResponse` kullanımı
- `toolsnaps.Test` ile snapshot testing

## Verification: Her Şey Yerinde mi?

Değişikliklerimi push etmeden önce kapsamlı doğrulama yaptım:

```bash
# Build
go build ./...
# ✅ Başarılı

# Test
UPDATE_TOOLSNAPS=true go test -v -run Test_CreateDiscussion ./pkg/github/...
# === RUN   Test_CreateDiscussion
# === RUN   Test_CreateDiscussion/successful_discussion_creation
# === RUN   Test_CreateDiscussion/org_level_discussion_(no_repo_specified)
# --- PASS: Test_CreateDiscussion (0.00s)
# ✅ 2/2 test geçti
```

Her şey yeşil. Force push zamanı:

```bash
git push origin feature/add-create-discussion-tool --force
# + e02cd14...e854a95 (forced update)
```

PR güncellendi. Conflict'ler gitti. Maintainer'a ping attım:

> "Hey @SamMorrowDrums 👋 Just wanted to give a heads up - I've rebased this PR on latest main to fix the conflicts..."

## Çıkardığım Dersler

Bu deneyim bana çok şey öğretti. Sadece teknik konular değil, açık kaynak kültürü hakkında da.

### 1. Küçük Başla, Ama Başla
Koca bir projeye bakıp "nereden başlayacağım" diye düşünmek yerine, küçük bir eksiklik bul. Benim için bu, eksik bir tool'du. Senin için bir typo, bir dokümantasyon hatası veya küçük bir bug olabilir.

### 2. Maintainer'lar Meşgul İnsanlar
PR'ın review edilmesi günler, haftalar sürebilir. Bu kişisel değil. Büyük projelerde yüzlerce PR bekliyor olabilir. Sabırlı ol, ama arada kibarca hatırlat.

### 3. Kod Değişir, Adaptasyon Şart
Özellikle aktif projelerde, katkın beklerken kod tabanı değişebilir. Buna hazırlıklı ol. Rebase yapmayı, conflict çözmeyi öğren. Bazen sıfırdan yazmak gerekebilir - bu dünyanın sonu değil.

### 4. Pattern'ları Anla, Körü Körüne Kopyalama
İlk PR'ımda pattern'ları sadece kopyalamıştım. İkinci seferde gerçekten anladım. Bu fark, kodun kalitesine yansıyor.

### 5. Beklenmedik Durumlar Olabilir
Hesap askıya alınması gibi şeyler öngörülmez. Ama önemli olan pes etmemek. İki ay sonra bile dönüp işi bitirmek mümkün.

## Sonuç

Şu an PR #1519 açık ve conflict-free durumda. CI workflow'ları çalışıyor, maintainer review'ı bekliyor. Merge edilirse, dünyadaki herkes `github-mcp-server` kullanırken benim yazdığım `create_discussion` tool'unu kullanabilecek.

Bu his, saatler süren çalışmaya, iki aylık bekleyişe ve 85 dosyalık conflict'e değer.

Eğer siz de açık kaynak'a katkı yapmayı düşünüyorsanız: başlayın. Küçük bir şeyle başlayın. Korkmayın. Ve en önemlisi - pes etmeyin.

---

**BKZ:**
- [Git Stratejileri ve Branch Yönetimi](/posts/git-is-akisi-branch-stratejileri/)
- [GitHub Actions CI/CD Pipeline](/posts/github-actions-cicd-pipeline/)
- [GitLab'dan GitHub'a Dönüş Macerası](/posts/gitlab-github-macerasi-jekyll-altyapisi/)

![Furkan Köykıran](/assets/img/avatar.png)
*Furkan Köykıran - Senior Software Engineer*
