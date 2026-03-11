---
title: "MCP Ekosistemi ve AI Platformlarına Açık Kaynak Katkılarım"
description: "Ocak ayından itibaren Model Context Protocol (MCP) ekosistemi ve AI platformlarına yaptığım katkılar, technical solutions ve öğrenilen dersler."
date: 2026-03-11 16:00:00 +0300
categories: [Open Source, MCP, AI]
tags: [mcp, playwright, github, lobehub, ai, open-source, contribution]
image:
  path: /assets/img/2026-03-11/mcp-ai-contributions/banner.png
  alt: "MCP and AI Contributions Banner"
---

Bu blog yazısında, Model Context Protocol (MCP) ekosistemi ve AI platformlarına yaptığım katkılardan bahsedeceğim.

Bu dönemde, Playwright'de Unicode sanitization sorununu çözdüm, GitHub MCP server'ında security validation ekledim ve LobeHub platformunda onboarding crash fix'i yaptım.

![MCP Protocol](/assets/img/2026-03-11/mcp-ai-contributions/mcp-protocol.png)
*Model Context Protocol (MCP), AI asistanları ve araçlar arasında standart bir iletişim katmanı sağlıyor.*

---

## MCP Ekosistemi Nedir?

Model Context Protocol (MCP), AI asistanları ile harici araçlar arasında güvenli ve verimli iletişim kurmak için geliştirilmiş açık bir standart. Claude Code, Cursor, Windsurf gibi AI kod editörleri, bu protokolü kullanarak extensibility (genişletilebilirlik) sağlıyor.

Geçen yıl yaptığım [DevTo-MCP](https://github.com/furkankoykiran/DevTo-MCP) ve [OmniWire-MCP](https://github.com/furkankoykiran/OmniWire-MCP) projelerinde, MCP'nin pratik kullanım senaryolarını deneyimlemiştim. Bu dönemde ise, bu ekosisteme doğrudan katkıda bulunmaya odaklandım.

---

## 1. Playwright: MCP Unicode Sanitization

### Problemi Keşfetmek

`playwright-mcp` projesinde bir issue'ya rastladım. Bir başka katkıda bulunan (frankhommers), [#1447 numaralı issue](https://github.com/microsoft/playwright-mcp/issues/1447)'de MCP responses'larında malformed Unicode (bozuk Unicode) karakterler nedeniyle JSON serialization hataları yaşandığını bildirmişti. Sonra frankhommers, [PR #1448](https://github.com/microsoft/playwright-mcp/pull/1448) ile bu sorunu çözmeye çalışmıştı, ancak bu PR transport layer'da (cli.js) çözüm sunuyordu ve kapatılmıştı.

Sorun, web sayfalarındaki "lone surrogate" karakterlerinin (örneğin yarım kalmış emoji parçaları) JSON'a serialize edilirken hata oluşturmasıydı.

```javascript
// Hata veren durum
const text = "Hello\uD800"; // Lone high surrogate
JSON.stringify({ content: text }); // Error!
```

### Çözüm: Core Layer Sanitization

Playwright'in core katmanında (`response.ts`) zaten `redactText()` fonksiyonu ile metin işleme yapıldığını fark ettim. Buraya `sanitizeUnicode()` fonksiyonu ekleyerek sorunu kökten çözmeye karar verdim.

```typescript
// packages/playwright-core/src/tools/response.ts
function sanitizeUnicode(text: string): string {
  // Node 20+ için native API
  if (typeof String.prototype.toWellFormed === 'function') {
    return text.toWellFormed();
  }

  // Node 18 için fallback
  return text.replace(
    /[\uD800-\uDDBF](?![\uDC00-\uDFFF])|(?:[^\uD800-\uDBFF])^[\uDC00-\uDFFF]/g,
    '\uFFFD' // Replacement character
  );
}
```

### PR'ın Durumu

[11 Mart'ta açtığım #39625 numaralı PR](https://github.com/microsoft/playwright/pull/39625) şu anda açık durumda. Core katmanında yapılan değişiklik olduğu için review süreci daha uzun sürüyor.

![Playwright Testing](/assets/img/2026-03-11/mcp-ai-contributions/playwright-testing.png)
*Playwright, modern web testing ve automation için kullanılan güçlü bir araç.*

### Öğrenilen Dersler

1. **Transport vs Core Layer**: Çözümün yerini doğru seçmek kritik. CLI'da geçici çözüm yapmak yerine, core'da kalıcı çözüm bulmak daha doğru.

2. **Native API vs Polyfill**: `String.prototype.toWellFormed()` gibi modern JavaScript API'lerini kullanırken, Node 18 gibi older versiyonlar için fallback sunmak gerekiyor.

3. **Test Coverage**: Unicode testleri yazmak, edge case'leri yakalamak için kritik. Lone high/low surrogates, valid surrogate pairs (emoji, CJK), mixed content gibi tüm durumları test ettim.

---

## 2. GitHub MCP Server: Discussions API ve Read-Only Mode

### Problemi Keşfetmek

GitHub MCP server'ında iki farklı eksiklik tespit ettim. Birincisi, discussions oluşturmak için bir tool yoktu. İkincisi, `--read-only` flag'i HTTP mode'da çalışmıyordu - write tools (create_branch, create_pull_request, merge_pull_request) read-only mode'da bile erişilebiliyordu.

### Çözüm: Create Discussion Tool ve Read-Only Fix

[#1519 numaralı PR](https://github.com/github/github-mcp-server/pull/1519) ile create_discussion tool ekledim:

```python
# pkg/github/discussions.go
func CreateDiscussion(ctx context.Context, client *github.Client, input DiscussionInput) (*github.Discussion, error) {
    // GraphQL mutation ile discussion oluşturma
    mutation := `
        mutation {
            createDiscussion(input: {repositoryId: $repoId, categoryId: $catId, title: $title, body: $body}) {
                discussion { title, url, number }
            }
        }
    `
    // ...
}
```

[#2200 numaralı PR](https://github.com/github/github-mcp-server/pull/2200) ile read-only flag fix'i yaptım:

```go
// pkg/http/server.go
type ServerConfig struct {
    ReadOnly bool `json:"read_only"`
}

// withGlobalReadonly middleware ile write tools'u block'la
if config.ReadOnly {
    // create_branch, create_pull_request, merge_pull_request gizle
}
```

### Öğrenilen Dersler

1. **GraphQL Mutation**: GitHub Discussions API'si GraphQL üzerinden çalışıyor.

2. **Middleware Pattern**: Read-only mode enforcement için middleware pattern kullanımı.

3. **Tool Discovery**: MCP server'larında tools/list endpoint'i ile hangi tool'ların available olduğunu belirtmek.

---

## 3. LobeHub: Onboarding Crash Fix

### Problemleri Keşfetmek

LobeHub organizasyonunda iki farklı sorun tespit ettim:

1. **High Priority Regression**: ModelSelect onboarding crash ([#12817](https://github.com/lobehub/lobehub/issues/12817))
2. **Feature Request**: 4 MCP server'ını marketplace'e ekleme talebi ([#12805](https://github.com/lobehub/lobehub/issues/12805))

### Çözüm: Crash Regression Fix

[#12818 numaralı PR](https://github.com/lobehub/lobehub/pull/12818) ile onboarding crash'ini fix ettim:

```typescript
// packages/ui/model-select/components/onboarding.tsx
// Önceki: Null check eksikti
const selectedModel = models.find(m => m.id === modelId);

// Sonrası: Null check eklendi
const selectedModel = models.find(m => m.id === modelId);
if (!selectedModel) {
  console.warn(`Model not found: ${modelId}`);
  return <FallbackComponent />;
}
```

![GitHub Contribution](/assets/img/2026-03-11/mcp-ai-contributions/github-contribution.png)
*GitHub açık kaynak katkıları, kod kalitesini ve community growth'yi artırıyor.*

### Issue #12817: High Priority Regression

[#12817 numaralı issue](https://github.com/lobehub/lobehub/issues/12817) ile bu critical regression'ı rapor ettim. React Error #185, null check eksikliğinden kaynaklanıyordu.

### Issue #12805: MCP Marketplace Submission

[#12805 numaralı issue](https://github.com/lobehub/lobehub/issues/12805) ile 4 MCP server'ını (DevTo-MCP, OmniWire-MCP, NotebookLM-MCP, Whop-MCP) LobeHub marketplace'e eklenmesini talep ettim.

### Öğrenilen Dersler

1. **Regression Testing**: Yeni features'lar var olan functionality'ı bozabilir
2. **Null Safety**: TypeScript'te null check importance
3. **Error Boundaries**: React error boundary pattern kullanımı
4. **Community Engagement**: Open source projelerde feature request süreçleri

---

## MCP Server Development Best Practices

Bu projelerde çalışırken öğrendiklerim:

### MCP Server Mimarisı

```typescript
// Standart bir MCP server yapısı
const server: Server = {
  name: "my-mcp-server",
  version: "1.0.0",

  // Tools (kullanılabilir fonksiyonlar)
  tools: [
    {
      name: "get_data",
      description: "Fetch data from API",
      inputSchema: {
        type: "object",
        properties: {
          id: { type: "string" }
        }
      }
    }
  ],

  // Resources (dosya veya veri kaynakları)
  resources: [
    {
      uri: "file:///config.json",
      name: "Configuration"
    }
  ]
};
```

### Security Considerations

1. **Authenticated User Validation**: MCP server'larında user authentication kontrolü
2. **Input Sanitization**: Kullanıcı inputu sanitization
3. **Access Control**: Private vs public resource access control
4. **Rate Limiting**: API rate limiting implementation

---

## LobeHub Platformu

![LobeHub Platform](/assets/img/2026-03-11/mcp-ai-contributions/lobehub-platform.png)
*LobeHub, AI uygulamaları için modern ve user-friendly bir platform.*

LobeHub, yapay zeka uygulamaları için modern bir platform. Şubat ayında bir PR ve iki issue ile katkıda bulundum:

### LobeHub Architecture

Bu projeler, modern React, TypeScript ve monorepo mimarisi kullanıyor. Katkılarım sırasında öğrendiklerim:

1. **Monorepo Management**: pnpm workspace kullanımı
2. **TypeScript Project References**: Büyük projelerde type management
3. **Module Resolution**: `exports` field kullanımı
4. **Build Optimization**: Incremental compilation

---

## Unicode Handling in JavaScript

Playwright projesinde öğrendiklerim:

### Lone Surrogates

```javascript
// Lone high surrogate (0xD800-0xDBFF)
const loneHigh = "Hello\uD800"; // ❌ Invalid

// Lone low surrogate (0xDC00-0xDFFF)
const loneLow = "Hello\uDC00"; // ❌ Invalid

// Valid surrogate pair
const validPair = "Hello\uD83D\uDE00"; // ✅ Emoji (😀)
```

### String.prototype.toWellFormed()

```javascript
// Modern JavaScript (Node 20+)
const malformed = "Hello\uD800World";
const wellFormed = malformed.toWellFormed(); // "Hello�World"

// Fallback for older versions
function toWellFormedPolyfill(str) {
  return str.replace(
    /[\uD800-\uDDBF](?![\uDC00-\uDFFF])|(?:[^\uD800-\uDBFF])^[\uDC00-\uDFFF]/g,
    '\uFFFD'
  );
}
```

---

## Sonuç: Açık Kaynak Katkılarının Önemi

Ocak ayından itibaren 3 farklı projeye katkıda bulunarak, hem teknik bilgilerimi geliştirdim hem de açık kaynak topluluğuna değer kattım. Öğrendiklerim:

1. **Transport vs Core Layer**: Çözümün yerini doğru seçmek
2. **Authenticated User Validation**: MCP security'i
3. **Regression Testing**: Yeni features'lar var olan functionality'ı bozabilir
4. **Unicode Handling**: JavaScript'te lone surrogate problemi
5. **Open Source Communication**: PR description'ları, review süreçleri

Açık kaynak katkılarının sadece kod yazmak değil, aynı zamanda community ile iletişim kurmak, documentation okumak ve issue'ları analiz etmek gerektiğini öğrendim.

Eğer siz de MCP ekosistemine katkıda bulunmak isterseniz, GitHub'daki `modelcontextprotocol` organizasyonunu incelemenizi öneriyorum.

![Furkan Köykıran](/assets/img/avatar.png)
*Furkan Köykıran - Senior Software Engineer*

---

## Diğer Blog Yazılarım

- [Güvenlik Açıkları ve API Geliştirmeleri: 3 Critical Fix](/posts/security-api-fixes/)
- [Python SDK ve CLI Araçları Geliştirme Deneyimlerim](/posts/python-sdk-cli/)
