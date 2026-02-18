---
title: "DevTo-MCP: DEV Community API'si ile AI Asistanları Arasında Güvenli Bir Köprü"
description: "Forem (DEV Community) API'sini kullanarak geliştirdiğim, TypeScript ve Zod tabanlı, modern CI/CD süreçlerine sahip açık kaynak MCP sunucusu DevTo-MCP'nin geliştirme hikayesi."
date: "2026-02-18 16:00:00 +0300"
categories: [Open Source, MCP, TypeScript]
tags: [mcp, devto, forem, typescript, github-actions, npm, open-source, ai, claude]
image:
  path: https://miro.medium.com/v2/1*wQ5bdLDQ0wceeANG2i2YzA.png
  alt: "DevTo-MCP Architecture"
---

Yazılım geliştiricilerin evi neresidir? GitHub? Stack Overflow? Bence son yıllarda bu sorunun cevabı giderek **DEV Community (dev.to)** ve onun altyapısını oluşturan **Forem** ekosistemi olmaya başladı.

Sabah kahvenizi içerken okuduğunuz o "React Hooks Best Practices" makalesi, takıldığınız bir bug için bulduğunuz çözüm veya kariyer tavsiyeleri için başvurduğunuz ilk duraklardan biri burası. Teknik makaleler, tartışmalar ve samimi bilgi paylaşımı için harika bir platform.

Peki, günümüzün kaçınılmaz gerçeği olan ve kod yazarken sürekli yanımızda duran AI asistanlarımız (Claude, Cursor, vs.) neden bu bilgi havuzuna erişemesin?

- Neden Claude'a "Şu an DEV.to'da 'Rust' hakkında popüler olan makaleleri listele" diyemeyeyim?
- Neden taslağını oluşturduğum bir makaleyi, editörden çıkmadan tek komutla "Bunu taslak olarak DEV'e gönder" diyemeyeyim?
- Neden takip ettiğim yazarların son gönderilerini AI benim için özetleyemesin?

İşte bu sorulardan yola çıkarak **DevTo-MCP** projesine başladım. Amacım basitti: DEV Community (Forem) API'si ile AI modelleri arasında güvenli, standartlara uygun ve "type-safe" bir köprü kurmak.

Bu yazıda, sadece "bir proje yaptım" demekle kalmayıp; **Model Context Protocol (MCP)**'ün ne olduğunu, projenin teknik mimarisini, karşılaştığım ilginç API sorunlarını ve bir açık kaynak projesini profesyonelce yönetmenin inceliklerini (CI/CD, npm release süreçleri) detaylıca anlatacağım.

## Bölüm 1: Model Context Protocol (MCP) Nedir?

Önce biraz teoriden bahsedelim, çünkü MCP'yi anlamadan bu projenin değerini anlamak zor.

Eskiden (yani 6 ay önce), bir AI modeline dış dünyadan veri vermek istiyorsanız, o modele özel "Function Calling" veya "Plugin" yapısını kullanmanız gerekirdi. OpenAI için ayrı, Anthropic için ayrı, Google için ayrı kod yazardınız. Bu sürdürülebilir değildi.

**MCP (Model Context Protocol)**, bu karmaşaya bir son vermek için Anthropic tarafından tasarlanan açık bir standarttır. Mantığı, USB girişine benzer:

- **Host (AI Asistan):** Bilgisayarınız (Claude Desktop, Cursor, vs.)
- **Client (MCP Server):** USB cihazınız (DevTo-MCP)
- **Protokol:** JSON-RPC 2.0 (İletişim dili)

MCP Server, AI asistanına "Benim yeteneklerim (Tools) şunlardır, şu kaynakları (Resources) okuyabilirim" der. AI asistanı da ihtiyaç duyduğunda bu yetenekleri çağırır. Hepsi bu.

DevTo-MCP de tam olarak bunu yapar. Forem API'sinin karmaşık endpoints dünyasını, AI'ın anlayabileceği basit fonksiyonlara dönüştürür.

## Bölüm 2: Teknik Mimarı ve TypeScript Gücü

Bir MCP sunucusu yazıyorsanız, en kritik konu **Veri Güvenliği** ve **Tip Güvenliği (Type Safety)**'dir. AI modelleri halüsinasyon görebilir, yanlış parametreler üretebilir. Sunucunuzun savunma hattı sağlam olmalıdır.

Bu yüzden projeyi **TypeScript** ve **Node.js** üzerine inşa ettim ve veri doğrulama için **Zod** kütüphanesini kullandım.

### Mimari Genel Bakış

Proje yapısı şu şekilde kurgulandı:

```
src/
├── services/       # Forem API ile konuşan katman
│   └── api-client.ts
├── tools/          # MCP araçlarının tanımlandığı yer
│   ├── articles.ts
│   ├── comments.ts
│   └── ...
├── types/          # TypeScript arayüzleri
└── server.ts       # Ana sunucu dosyası
```

### Zod ile "Runtime" Koruması

Dış bir API ile çalışıyorsanız (burada Forem API), dokümantasyona asla %100 güvenemezsiniz. "Bu alan kesinlikle sayı döner" denilen yer, bir gün `null` veya string dönebilir. Bu da JavaScript tabanlı sunucunuzu anında çökertir (crash).

Bunu engellemek için API'den gelen veriyi de, AI'dan gelen parametreleri de Zod şemalarından geçiriyorum.

Örneğin, bir makale oluşturma (create_article) aracı için tanımladığım şema:

```typescript
server.tool(
    "create_article",
    "Create a new article on DEV Community.",
    {
        title: z.string().describe("Title of the article"),
        body_markdown: z.string().optional().describe("Article content in markdown"),
        published: z.boolean().optional().describe("Whether to publish immediately"),
        tags: z.array(z.string()).max(4).optional().describe("List of tags (max 4)"),
        canonical_url: z.string().url().optional(), // Geçersiz URL girilirse Zod yakalar
        organization_id: z.number().int().optional()
    },
    async (params) => {
        // ... implementation
    }
);
```

Burada `.url()` veya `.max(4)` gibi kısıtlamalar, kodun daha veritabanına veya API'ye gitmeden doğrulanmasını sağlar. Eğer AI yanlışlıkla 5 tane etiket gönderirse, Zod devreye girer ve "En fazla 4 etiket gönderebilirsin" hatasını fırlatır. Bu, gereksiz API çağrılarını ve olası banlanmaları engeller.

### Hata Toleranslı API İstemcisi

`src/services/api-client.ts` dosyasında, basit bir `fetch` wrapper'ı yerine, kendini iyileştirebilen akıllı bir HTTP istemcisi yazdım.

Özellikleri:
1.  **Rate Limit Takibi:** `x-ratelimit-remaining` header'ını okur ve limit dolmak üzereyse işlemi yavaşlatır.
2.  **Retry Mekanizması:** 429 (Too Many Requests) veya 5xx (Server Error) hatalarında otomatik olarak "Exponential Backoff" (üstel bekleme) stratejisiyle tekrar dener.
3.  **Tip Güvenli İstekler:** Generic yapısı sayesinde `client.get<Article[]>(...)` dediğimde dönen verinin tipinden emin olurum.

```typescript
// Retry ve Backoff mantığının basitleştirilmiş hali
private async request<T>(...): Promise<T> {
    for (let attempt = 0; attempt <= MAX_RETRIES; attempt++) {
        try {
            const response = await fetch(url, options);
            if (response.ok) return await response.json();
            
            // Hata 429 veya 500 ise bekle ve tekrar dene
            if (this.isRetryable(response.status)) {
                const delay = this.getRetryDelay(attempt);
                await this.sleep(delay);
                continue;
            }
            // ...
        } catch (error) {
             // Network hatası ise tekrar dene
        }
    }
}
```

Bu yapı, ağ dalgalanmalarında veya Forem API'sinin anlık kesintilerinde sunucunun ayakta kalmasını sağlar.

## Bölüm 3: Geliştirme Günlükleri - Karşılaşılan Zorluklar

Kod yazmak kolaydır, asıl zorluk beklenmedik durumları yönetmektir. DevTo-MCP geliştirirken de ilginç engellerle karşılaştım.

### 1. "Disable Reactions" Kararı

Projeye başlarken, kullanıcıların makalelere tepki (Like, Unicorn, Fire vb.) vermesini sağlayan bir araç eklemek istiyordum. Bu, interaktif bir deneyim için şarttı.

Ancak API'yi test ederken tutarsızlıklar fark ettim. Bazen tepki başarılı dönüyor ama arayüze yansımıyordu, bazen 401 yetki hatası alıyordum (API key doğru olmasına rağmen). MCP sunucuları, AI için "güvenilir" araçlar olmalıdır. Eğer AI bir işlem yaptığını sanıp yapamazsa, bu zincirleme hatalara yol açar.

Bu yüzden zor bir karar alarak, `src/server.ts` dosyasında şu satırı yorum satırına aldım:

```typescript
// registerReactionTools(server, client); // Disabled due to API 401 issues
```

Bazen "hayır" demek, kötü çalışan bir özellikten iyidir. Yazılım mühendisliğinde buna "Feature Cutting" denir ve projenin sağlığı için gereklidir.

### 2. Kayıp Makaleler (Drafts) ve 404 Yönetimi

Kullanıcı "benim X başlıklı taslağımı getir" dediğinde, sistem `get_article_by_id` aracını kullanır. Ancak Forem API'si, taslak makaleler için public endpoint'ten 404 döner.

AI asistanı 404 alınca "Böyle bir makale yok" diyecek ve pes edecekti. Oysa makale var, sadece taslak olduğu için o endpoint'te yok.

Bunu çözmek için `get_article_by_id` fonksiyonuna bir "Fallback" (yedek plan) mekanizması ekledim:

```typescript
} catch (error: unknown) {
    // Eğer 404 alırsak ve hata bir "bulunamadı" ise...
    if (is404) {
        try {
            // Kullanıcının TÜM makalelerini (taslaklar dahil) çek
            const allArticles = await client.get<Article[]>("/articles/me/all");
            // İçinde aradığımız ID var mı bak
            const article = allArticles.find((a) => a.id === params.id);
            if (article) return { content: JSON.stringify(article) };
        } catch (fallbackError) {
             // Yine bulamadık, yapacak bir şey yok.
        }
    }
    // ...
}
```

Bu küçük dokunuş, kullanıcı deneyimini (UX) inanılmaz iyileştirdi. Artık AI, taslaklarınız üzerinde de çalışabiliyor.

## Bölüm 4: CI/CD ve Yayınlama Maceraları

Açık kaynak bir proje sadece koddan ibaret değildir. Onun nasıl paketlendiği, nasıl test edildiği ve nasıl dağıtıldığı da önemlidir.

### "Trusted Publishing" ve OIDC

NPM paketlerini yayınlamak için eskiden bir `NPM_TOKEN` oluşturur ve bunu GitHub Secrets'a eklerdik. Bu token çalınırsa, birisi sizin adınıza zararlı kod yayınlayabilirdi.

DevTo-MCP'de, modern **OpenID Connect (OIDC)** yöntemini, yani NPM'in "Trusted Publishing" özelliğini kullandım. Bu yöntemde şifre yok, token yok. GitHub Actions, NPM'e "Ben furkankoykiran/DevTo-MCP reposunun release workflow'uyum, işte kanıtım" diyor ve NPM buna güvenerek (trust) yayın izni veriyor.

Ancak burada küçük bir detay beni günlerce uğraştırdı: **Environment**.

NPM tarafında bu güven ilişkişini "production" ortamı için kurmuştum. Ancak GitHub Actions dosyamda (`npm-publish.yml`) `environment: production` satırını eklemeyi unutmuştum. Sonuç? Sürekli 404 hataları.

```yaml
jobs:
  publish:
    runs-on: ubuntu-latest
    environment: production  # <-- İşte o sihirli satır!
    permissions:
      id-token: write      # OIDC için zorunlu
```

Bu satırı eklediğim an, her şey saat gibi çalışmaya başladı.

### V1.0.3 Neden Çıkmadı?

Projeyi yayınladıktan sonra küçük bir hata fark ettim ve düzelttim. PR açtım, merge ettim. Beklentim otomatik olarak `v1.0.3` sürümünün NPM'e gitmesiydi. Ama gitmedi.

GitHub Actions loglarını incelediğimde workflow'un hiç tetiklenmediğini gördüm. Sebebi basitti: `npm-publish.yml` dosyasındaki tetikleyici kuralı:

```yaml
on:
  release:
    types: [published]
```

Workflow sadece "Release yayınlandığında" çalışıyordu, kod merge edildiğinde değil. Bu aslında güvenli bir yaklaşımdı ama benim o anki beklentimle uyuşmadı. Ayrıca `package.json` dosyasında versiyonu manuel olarak `1.0.3` yapmayı unutmuştum.

Dersi aldık: Otomasyon harikadır, ama insan hatasını (versiyon bumplamayı unutmak) tamamen kapsamaz.

## Bölüm 5: Kurulum ve Kullanım Rehberi

Peki bu sunucuyu siz nasıl kullanacaksınız? İşte Claude Desktop için adım adım rehber.

### Gereksinimler
- Node.js (v18 veya üzeri)
- Bir DEV Community API Key (Ayarlar -> Extensions bölümünden alabilirsiniz)

### Claude Desktop Yapılandırması

Bilgisayarınızdaki Claude yapılandırma dosyasını açın:
- macOS: `~/Library/Application Support/Claude/claude_desktop_config.json`
- Windows: `%APPDATA%\Claude\claude_desktop_config.json`

Ve şu ayarı ekleyin:

```json
{
  "mcpServers": {
    "devto": {
      "command": "npx",
      "args": [
        "-y",
        "devto-mcp"
      ],
      "env": {
        "DEVTO_API_KEY": "SİZİN_API_ANAHTARINIZ"
      }
    }
  }
}
```

Kaydedip Claude'u yeniden başlatın. Artık sağ taraftaki "ataç" ikonuna tıkladığınızda `devto` araçlarını göreceksiniz.

### Neler Yapabilirsiniz?

1.  **Makale Okuma:** "Bana bu haftanın en popüler TypeScript makalelerini özetle."
2.  **Taslak Oluşturma:** "Rust ve WebAssembly hakkında giriş seviyesinde bir blog yazısı taslağı hazırla ve DEV.to hesabımda taslak olarak oluştur."
3.  **Yorum Analizi:** "Son yazdığım makaleye gelen yorumları listele ve cevaplanması gereken soruları bana söyle."

## Son Söz

DevTo-MCP benim için sadece bir araç geliştirmek değil, aynı zamanda API tasarım kalıpları, hata yönetimi ve modern CI/CD süreçleri üzerine bir laboratuvar çalışması oldu.

Açık kaynak dünyasında, kodunuzun başkaları tarafından kullanılabilmesi için sadece "çalışması" yetmez; **güvenilir**, **belgelenmiş** ve **sürdürülebilir** olması gerekir. Bu projede bu standartları yakalamaya çalıştım.

Siz de projeyi incelemek, kodlarına bakmak (belki o kapattığım reaction tool'unu düzeltmek?) veya sadece bir yıldız (⭐️) bırakmak isterseniz repoya beklerim:

[👉 GitHub: furkankoykiran/DevTo-MCP](https://github.com/furkankoykiran/DevTo-MCP)

Kodla ve bağlamla kalın.

---

**BKZ:**
- [OmniWire-MCP: AI Modelleri İçin Haber Köprüsü](/posts/omniwire-mcp-ai-news-server/)
- [GitHub MCP Server'a Katkı Macerası](/posts/github-mcp-server-acik-kaynak-katki/)
- [GitHub Actions CI/CD Pipeline](/posts/github-actions-cicd-pipeline/)

![Furkan Köykıran](/assets/img/avatar.png)
*Furkan Köykıran - Senior Software Engineer*
