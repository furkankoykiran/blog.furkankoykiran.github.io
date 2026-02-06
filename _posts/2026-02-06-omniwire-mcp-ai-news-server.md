---
title: "OmniWire-MCP: AI Modelleri İçin Güvenli, Hızlı ve Standartları Belirleyen Haber Köprüsü"
description: "Model Context Protocol (MCP) kullanarak geliştirdiğim, Sentinel mimarisine sahip, hata toleranslı ve açık kaynak dostu yeni nesil haber sunucusu OmniWire-MCP'nin hikayesi."
date: "2026-02-06 20:00:00 +0300"
categories: [Open Source, MCP, TypeScript]
tags: [mcp, nodejs, typescript, github-actions, circuit-breaker, open-source, npm]
image:
  path: /assets/img/posts/omniwire-mcp/typescript_node.png
  alt: "OmniWire-MCP Teknoloji Yığını"
---

Yapay zeka modellerinin (LLM'lerin) en büyük sorunu nedir? Halüsinasyon? Belki. Ama bence daha kritik bir sorun var: **Güncel bağlam (context) eksikliği.**

Bir model ne kadar zeki olursa olsun, dünyada şu an ne olup bittiğini bilemez. Ona bir haber kaynağı vermeniz gerekir. İşte tam bu noktada, "context is king" (bağlam kraldır) felsefesiyle yola çıktığım yeni projem **OmniWire-MCP** devreye giriyor.

Bu yazıda, sadece "bir proje yaptım" demek istemiyorum. Bir fikirden yola çıkarak; dağıtık sistem mimarisi tasarlamayı, production-grade (üretime hazır) standartları oturtmayı ve GitHub Actions ile boğuşarak kurduğum CI/CD hattını en ince detayına kadar anlatacağım.

Ve tabii ki, o meşhur NPM yayınlama (publish) hatalarını nasıl çözdüğümden de bahsedeceğim.

## OmniWire-MCP Nedir?

OmniWire-MCP, **Model Context Protocol (MCP)** standardı üzerine inşa edilmiş, hataya dayanıklı (fault-tolerant) bir haber ve içerik toplama sunucusudur.

Kısaca; RSS, Atom, JSON veya HTML formatındaki herhangi bir kaynağı alır, normalize eder ve yapay zeka modellerinin anlayabileceği, temiz bir formatta sunar. Claude, Gemini veya ChatGPT gibi modeller, bu sunucu sayesinde "Şu an teknoloji dünyasında neler oluyor?" sorusuna gerçek zamanlı verilerle cevap verebilir.

Projeyi geliştirirken Node.js ve TypeScript ikilisini tercih ettim. Tip güvenliği (type safety) ve modern JavaScript özellikleri, bu karmaşıklıkta bir proje için olmazsa olmazdı. Özellikle "Discriminated Unions" ve "Generic" yapıları, farklı haber kaynaklarını tek bir potada eritmeyi inanılmaz kolaylaştırdı.

## Mimari Kararlar 1: Sentinel Asla Uyumaz

Bir haber sunucusunun en büyük kabusu nedir? Kaynakların çökmesi.

Diyelim ki popüler bir teknoloji sitesinin RSS servisi yanıt vermiyor. Eğer sunucunuz bu duruma hazırlıklı değilse, o kaynaktan veri beklerken (pending state) kilitlenir. Node.js'in meşhur Event Loop'u bloklanmasa bile, bekleyen Promise'ler bellek şişmesine (memory leak) yol açabilir. Kilitlenen bir sunucu, diğer sağlıklı kaynakları da sunamaz hale gelir. Zincirleme bir reaksiyonla tüm sistem çöker.

İşte bu sorunu çözmek için **Sentinel** adını verdiğim bir servis geliştirdim. Sentinel, mikroservis mimarilerinde Netflix Hystrix ile popülerleşen **Circuit Breaker (Devre Kesici)** tasarım desenini uyguluyor.

![Circuit Breaker Mimarisi](/assets/img/posts/omniwire-mcp/sentinel_circuit_breaker.png)
*Sentinel Servisi'nin kalbindeki Circuit Breaker mantığı.*

### Kodun İçinden: Circuit Breaker Nasıl Çalışıyor?

`src/services/sentinel/circuit-breaker.ts` dosyamda, bu mantığı yöneten çekirdek yapı var. Sistemin bir kaynağa "istek atıp atmayacağına" karar veren `canExecute` metoduna yakından bakalım:

```typescript
/**
 * Check if request should be allowed
 */
public canExecute(): boolean {
    const currentState = this.getState();

    // 1. Kapalıysa (Her şey yolunda) -> İzin ver
    // Devre kapalı demek, akım geçiyor demek. Yani her şey normal.
    if (currentState === CircuitState.CLOSED) {
        return true;
    }

    // 2. Yarı Açıksa -> Tek bir deneme isteğine izin ver
    // Bu, "kanarya testi" gibidir. Sistem düzeldi mi diye bakıyoruz.
    if (currentState === CircuitState.HALF_OPEN) {
        return true;
    }

    // 3. Açıksa -> Recovery süresi dolduysa Yarı Açık moda geç
    // Hata eşiği aşıldı, devre açıldı. Belirli bir süre (örn. 60sn) bekledik mi?
    if (this.shouldAttemptRecovery()) {
        this.transitionTo(CircuitState.HALF_OPEN);
        return true;
    }

    // Aksi halde -> Reddet
    // Kaynak hala "cezalı". İsteği doğrudan reddet, hiç network'e çıkma.
    return false;
}
```

Bu kontrol mekanizması, sistemin %99.9 erişilebilirlik (uptime) sağlamasının anahtarı. OmniWire, 100 kaynaktan 20'si çökse bile, kalan 80 kaynağı milisaniyeler içinde sunmaya devam eder. Kullanıcı (veya AI) asla bekletilmez. Hatalı kaynak için anında "Service Unavailable" benzeri bir internal sinyal üretilir ve AI'a "Bu kaynak şu an erişilemiyor, diğerlerine bakıyorum" bilgisi verilir.

Sentinel servisi ayrıca her başarısızlıkta akıllıca davranır. Hata sayısı belirlenen eşiği (varsayılan: 3) geçerse devreyi açar ve "Soğuma Süresi" (Recovery Timeout) boyunca o kaynağı nadasa bırakır. Bu süre dinamik olarak da ayarlanabilir, böylece her kaynak için farklı tolerans seviyeleri belirlenebilir.

## Mimari Kararlar 2: Evrensel Ayrıştırıcı (Universal Parser)

İnternet dünyasında standartlar güzeldir, ama herkesin uyması şartıyla. Haber kaynaklarında ise tam tersi bir kaos hakim:
- Kimi RSS 2.0 kullanır (standart XML).
- Kimi Atom 1.0 tercih eder (biraz daha modern XML).
- Kimi özel bir JSON API sunar.
- Kimi ise hiçbir şey sunmaz, HTML kazımanız (scrape) gerekir.

OmniWire'da bu kaosu yönetmek için **Universal Parser** adında bir yapı kurdum. Bu yapı, `Strategy Pattern` benzeri bir yaklaşımla çalışıyor ancak biraz daha "zeki". Gelen içeriğin tipini sadece `Content-Type` başlığına (header) bakarak değil, içeriğin ilk baytlarını koklayarak (sniffing) algılıyor.

`src/services/parser/index.ts` dosyasındaki şu koda dikkat edin:

```typescript
/**
 * Detect content type from content and headers
 */
public detectContentType(content: string, contentTypeHeader?: string): ContentType {
    // 1. Önce Header'a bak
    if (contentTypeHeader) {
        const lower = contentTypeHeader.toLowerCase();
        if (lower.includes('json')) return ContentType.JSON;
        // ... XML kontrolleri ...
    }

    // 2. Header yoksa veya belirsizse (octet-stream), içeriği incele
    const trimmed = content.trim();

    // XML tabanlı mı? (RSS veya Atom)
    if (trimmed.startsWith('<?xml') || trimmed.startsWith('<rss') || trimmed.startsWith('<feed')) {
        return trimmed.includes('<feed') ? ContentType.ATOM : ContentType.RSS;
    }

    // JSON mı?
    if (trimmed.startsWith('[') || trimmed.startsWith('{')) {
        return ContentType.JSON;
    }

    // HTML mi?
    if (trimmed.toLowerCase().startsWith('<!doctype') || trimmed.toLowerCase().startsWith('<html')) {
        return ContentType.HTML;
    }

    return ContentType.UNKNOWN;
}
```

Bu yöntem sayesinde, bir kaynak sunucusu yanlış header gönderse bile (ki çok sık oluyor, XML verisine `text/html` diyenler var), OmniWire doğru adaptörü devreye sokabiliyor.

## TypeScript ile Veri Güvenliği: `NewsItemSchema`

Böylesine kaotik bir veri girişini standart bir çıktıya dönüştürmek büyük risk taşır. Ya veride `title` eksikse? Ya `link` geçerli bir URL değilse? Runtime'da patlamamak için, çalışma zamanı doğrulama kütüphanesi olan **Zod** kullandım.

`src/services/parser/types.ts` dosyasındaki şema, sistemin "Single Source of Truth" (Tek Gerçeklik Kaynağı) görevini görür:

```typescript
export const NewsItemSchema = z.object({
    id: z.string(),
    title: z.string(),
    link: z.string().url(), // Geçersiz URL girerse anında yakalar
    description: z.string().optional(),
    publishedAt: z.date().optional(),
    categories: z.array(z.string()).default([]),
    // ...
});

export type NewsItem = z.infer<typeof NewsItemSchema>;
```

Bu yapı sayesinde, ayrıştırıcı (parser) katmanından çıkan her veri paketinin (news item) formatından %100 emin olabiliyorum. Eğer dış kaynaktan bozuk bir veri gelirse, Zod bunu yakalıyor ve sistemi kirletmeden logluyor. TypeScript'in `infer` gücü sayesinde de, ekstra bir `interface` yazmama gerek kalmadan otomatik tip tanımlarına sahip oluyorum.

## Proje Standartları: Bir "Hobi Projesi"nden Fazlası

OmniWire-MCP'yi geliştirirken kendime bir söz verdim: Bu proje, herhangi bir kurumsal projeyle yarışacak kalitede olmalıydı. "Side project" olması, kodun kirli veya dökümantasyonun eksik olacağı anlamına gelmemeliydi.

Bu yüzden repoya ilk commit'i atmadan önce **Open Source Standartları**nı belirledim:

1.  **Rulesetler:** GitHub repository ayarlarında `main` branch'ini korumaya aldım. Pull Request (PR) açılmadan, CI testleri geçmeden ve onay alınmadan kimse (ben dahil!) ana koda doğrudan commit atamaz. Bu disiplin, projenin her an "deploy edilebilir" durumda kalmasını sağlıyor.
2.  **Conventional Commits:** Git geçmişini temiz tutmak için standart mesaj formatı (feat, fix, chore, docs) zorunlu hale getirdim. Bu sadece estetik değil; otomatik versiyonlama araçlarının (semantic release) çalışması için şart.
3.  **Otomatik Temizlik (Stale Workflow):** Açık kaynak projelerin kaderidir; unutulmuş Issue'lar ve PR'lar. `stale.yml` workflow'u ile 30 gün işlem görmeyen kayıtları otomatik olarak işaretliyor ve kapatıyorum. Bu, backlog'un her zaman taze kalmasını sağlıyor.

![Open Source Topluluk](/assets/img/posts/omniwire-mcp/open_source_community.png)

Ayrıca topluluk dosyalarını (Health Files) eksiksiz hazırladım:
- **`CONTRIBUTING.md`**: Projeye nasıl katkı sağlanacağını anlatan rehber.
- **`CODE_OF_CONDUCT.md`**: Topluluk kuralları.
- **`SECURITY.md`**: Güvenlik bildirim süreçleri.

Bu dosyalar süs olsun diye orada değil. Bir gün dünyanın bir ucundan bir geliştirici projeye katkı sağlamak isterse, neyi nasıl yapacağını adım adım bilmeli ve kendini topluluğun bir parçası hissetmeli.

## Workflow Cehennemi: NPM Publish ve OIDC

Ve tabii ki, her yazılımcının "initiation" (kabul) töreni: CI/CD problemleri.

Amacım şuydu: GitHub'da bir `release` oluşturduğumda, workflow otomatik çalışsın, testleri yapsın, build alsın ve paketi NPM'e göndersin. Böylece `npx omniwire-mcp` diyerek herkes kullanabilsin.

![GitHub Actions Workflow](/assets/img/posts/omniwire-mcp/github_actions_workflow.png)
*Otomasyonun kalbi: CI/CD Pipeline'ımız.*

Ancak NPM'in modern **Trusted Publishing (OIDC)** sistemini kullanmak isterken ciddi bir duvara tosladım.

Sürekli `404 Not Found` hatası alıyordum. Paket NPM'de yok değildi, yetkim de vardı. Ama GitHub Actions, NPM'e kendini kanıtlayamıyordu. Normalde bir `NPM_TOKEN` alır ve secret olarak eklersiniz. Ama OIDC (OpenID Connect) ile şifresiz, token'sız, tamamen güvene dayalı bir köprü kurmak istedim.

Sorunun kaynağı? NPM'deki "Provenance" ayarları ile GitHub'daki "Environment" ayarlarının uyuşmamasıydı. NPM, "Ben bu paketi sadece `production` ortamından kabul ederim" diyordu, ama benim workflow'um ortam belirtmeden çalışıyordu.

### Çözüm: Environment Kullanımı

GitHub Actions dosyamda (`npm-publish.yml`), `environment: production` satırını eklemeyi atlamıştım.

```yaml
jobs:
  publish-npm:
    needs: build
    runs-on: ubuntu-latest
    environment: production  # <-- Beni günlerce uğraştıran eksik parça
    permissions:
      contents: read
      id-token: write      # OIDC doğrulaması için şart
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v3
        with:
          node-version: 20
          registry-url: 'https://registry.npmjs.org'
      # ... kurulum ve build adımları ...
      - run: npm publish --access public
```

Bu küçük satırı ekledikten sonra o yeşil tıkı görmek... Paha biçilemezdi. Şu an `v1.0.3` sürümü yayında ve sistem saat gibi işliyor. Artık tek yapmam gereken GitHub'da "Create Release" butonuna basmak. Gerisini robotlar hallediyor.

## Kurulum ve Kullanım

OmniWire-MCP'yi denemek isterseniz, bilgisayarınıza bir şey kurmanıza bile gerek yok. Eğer Node.js yüklüyse (ki 2026'da muhtemelen yüklüdür), terminale şunu yazmanız yeterli:

```bash
npx omniwire-mcp
```

Bu komut, sunucuyu indirir ve varsayılan ayarlarla başlatır. Eğer kendi RSS kaynaklarınızı eklemek isterseniz, bir JSON dosyası göstererek ayar yapabilirsiniz:

```bash
RSS_FEEDS="https://benim-config-dosyam.json" npx omniwire-mcp
```

Veya daha izole bir ortam isterseniz Docker ile tek satırda ayağa kaldırabilirsiniz:

```bash
docker run -i --rm omniwire-mcp
```

Projenin Docker imajı, Alpine Linux tabanlı minimal bir imaj kullanılarak oluşturulduğu için boyutu oldukça küçüktür ve deployment süreçlerinde hız kazandırır.

## Gelecek Planları

OmniWire-MCP şu an sağlam bir temel üzerinde duruyor. Ama aklımda daha birçok fikir var:

1.  **Vektör Veritabanı Entegrasyonu:** Şu an haberleri anlık çekip veriyorum. Amaç, bu haberleri bir vektör veritabanına (örn. Pinecone veya yerel ChromaDB) gömmek (embedding). Böylece AI'a "Geçen ay Apple hakkında çıkan olumsuz haberler nelerdi?" diye sorduğunuzda, anlamsal arama yapabilecek.
2.  **Akıllı Filtreleme:** LLM'ler her haberi okumak zorunda değil. Önceliklendirme katmanı ekleyerek, sadece yüksek önem derecesine sahip haberleri sunmak.
3.  **Çoklu Platform:** Sadece RSS değil; Slack, Discord veya Telegram kanallarını da birer "haber kaynağı" olarak eklemek.
4.  **Admin UI:** Konsol tabanlı yönetim yerine, kaynakları görsel olarak yönetebileceğimiz basit bir web arayüzü eklemek.

![Typescript ve Node.js](/assets/img/posts/omniwire-mcp/typescript_node.png)

## Son Söz

OmniWire-MCP, benim için sadece bir haber sunucusu değil, aynı zamanda modern bir yazılım projesinin nasıl yönetilmesi gerektiğine dair bir laboratuvar oldu.

- **Typescript** ve **Node.js** ile sağlam, tip güvenli bir temel attım.
- **Sentinel** ile dağıtık sistemlerde hata yönetimini (resilience) deneyimledim.
- **Zod** ile çalışma zamanı tip güvenliği sağladım.
- **GitHub Actions** ile CI/CD kaslarımı güçlendirdim ve OIDC dünyasına daldım.
- Ve en önemlisi, **MCP** gibi yeni ve heyecan verici bir standardın parçası oldum.

Umarım bu yazı, kendi projelerinizde karşılaşacağınız benzer sorunlar (özellikle o 404 hataları!) için size bir rehber olur.

Eğer siz de bu yolculuğa katılmak, bir satır kodla destek olmak veya sadece bir yıldız (⭐️) bırakmak isterseniz, GitHub reposuna beklerim:

[👉 GitHub: furkankoykiran/OmniWire-MCP](https://github.com/furkankoykiran/OmniWire-MCP)

Kodla ve merakla kalın.

---

**BKZ:**
- [GitHub MCP Server'a Katkı Macerası](/posts/github-mcp-server-acik-kaynak-katki/)
- [GitHub Actions CI/CD Pipeline](/posts/github-actions-cicd-pipeline/)
- [Modern Web Arayüzü: CSS & JavaScript](/posts/modern-web-arayuzu-css-javascript/)

![Furkan Köykıran](/assets/img/avatar.png)
*Furkan Köykıran - Senior Software Engineer*
