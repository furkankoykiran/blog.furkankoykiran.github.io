---
title: "CLI-Anything'e Tarayıcı Otomasyon Desteği: MCP Backend Pattern İlk Uygulama"
description: "CLI-Anything projesine tarayıcı otomasyon desteği ekledim. DOMShell MCP server entegrasyonuyla, CLI-Anything'in ilk MCP backend pattern'ini gerçekleştirdim."
date: 2026-03-22 21:00:00 +0300
categories: [Open Source, Development]
tags: [mcp, python, cli-anything, open-source, browser-automation]
image:
  path: /assets/img/posts/2026-03-22-browser-automation-cli/mcp-architecture-diagram.png
  alt: "MCP Architecture"
---

Tarayıcı otomasyonu gerçekten ilginç bir problem. AI ajanlarının web sitelerinde gezinebilmesi, formları doldurabilmesi artık sadece "iyi sahip olmak" değil, zorunluluk. CLI-Anything projesine yaptığım browser support katkısını burada anlatacağım.

![MCP Architecture](/assets/img/posts/2026-03-22-browser-automation-cli/mcp-architecture-diagram.png)
*MCP mimarisi - AI ajanları ile harici servisler arasındaki köprü.*

## Issue #90'dan PR #118'e

Her şey 16 Mart 2026'da [@apireno](https://github.com/apireno)'nin [#90 numaralı issue](https://github.com/HKUDS/CLI-Anything/issues/90)'sını açmasıyla başladı. Ana fikir şuydu: **"CLI wrapper'lar yazılımı ajan-native yapıyor"** ve bunun tarayıcılar için de geçerli olabileceği.

[@apireno](https://github.com/apireno), [DOMShell](https://github.com/apireno/DOMShell) projesinden bahsetti. DOMShell, Chrome'un Accessibility Tree'sini sanal bir dosya sistemine çeviriyor. Ajanlar `ls`, `cd`, `grep`, `click` gibi komutlarla tarayıcıyı kontrol edebiliyor. Benchmark sonuçları ilginçti: screenshot tabanlı yaklaşıma göre %50 daha az API çağrısı.

![Accessibility Tree](/assets/img/posts/2026-03-22-browser-automation-cli/accessibility-tree-view.png)
*Chrome DevTools'taki Accessibility Tree - aslında ekran okuyucular için yapılmış ama ajanlar için de iş görüyor.*

### Accessibility Tree Nedir?

Accessibility Tree, tarayıcının DOM'dan ürettiği ama daha sade bir yapı. Ekran okuyucular için var bu yapı. Öğeleri rollerine göre sınıflandırıyor - button, link, textbox gibi. Ajanlar için de ideal aslında, stabil ve anlaşılır.

DOMShell'in mantığı basitti: **Filesystem primitives** (ls, cd, grep) DOM query'lerden daha verimli. Ajanlar sayfayı gezerken daha az çağrı yapıyor, daha hızlı ulaşıyor hedefe.

21 Mart'ta [@omerarslan0](https://github.com/omerarslan0)'nın önerdiği modeli temel alarak PR'ı oluşturdum. Bu, CLI-Anything'in ilk MCP server backend pattern'idi.

```bash
# Kullanım örneği
cli-anything-browser page open https://example.com
cli-anything-browser fs ls /
cli-anything-browser fs cd /main
cli-anything-browser fs grep "Login"
cli-anything-browser act click /main/button[0]
```

## Mimari

Diğer harness'lar genellikle doğrudan backend API çağrısı yapıyor. Bu sefer durum farklıydı: **MCP Server** entegrasyonu.

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   CLI Commands   │────▶│  browser_cli.py │────▶│   MCP Backend   │
│  (Click groups)  │     │  (CLI entry)    │     │ (domshell_      │
└─────────────────┘     └─────────────────┘     │  backend.py)    │
                                                 └────────┬────────┘
                                                          │
                    ┌─────────────────────────────────────┼────────────┐
                    │                                     │            │
                    ▼                                     ▼            ▼
            ┌───────────────┐                 ┌────────────┐    ┌──────────┐
            │ Spawn npx     │                 │  DOMShell  │    │  Chrome  │
            │ subprocess    │◀──stdio─────────▶│  MCP Server│◀───│ + Ext    │
            └───────────────┘                 └────────────┘    └──────────┘
```

### MCP Backend Pattern

Bu CLI-Anything için yeni bir patterndi. Öncesinde hiçbir harness MCP server kullanmamıştı. Python SDK implementasyonu:

```python
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

server_params = StdioServerParameters(
    command="npx",
    args=["@apireno/domshell"]
)

async with stdio_client(server_params) as (read, write):
    async with ClientSession(read, write) as session:
        await session.initialize()
        result = await session.call_tool("domshell_ls", {"path": "/"})
```

Ana fark şu: MCP server **stateless**. Her komutta yeni subprocess başlıyor. State (URL, working directory, history) CLI tarafında tutuluyor.

### Command Groups

Implementasyon 4 ana group içeriyordu:

- **page**: open, reload, back, forward, info - Sayfa navigasyonu
- **fs**: ls, cd, cat, grep, pwd - Dosya sistemi tarzı gezinme
- **act**: click, type - Etkileşim
- **session**: status, daemon-start, daemon-stop - Oturum yönetimi

### Daemon Mode

Her komutta npx subprocess başlatmak 1-3 saniye kaybettiriyor. **Daemon mode** ile kalıcı bağlantı kuruyorsun. İnteraktif kullanımda fark hissediliyor.

```bash
cli-anything-browser session daemon-start
# Tüm komutlar aynı bağlantıyı kullanır artık
cli-anything-browser fs ls /
cli-anything-browser session daemon-stop
```

## Testler

Üç katmanlı test stratejisi uyguladım. Unit tests 31 taneydi - MCP backend response'ları mock'lanıyor, path resolution ve state management test ediliyordu. Chrome gerekmiyordu.

```bash
pytest cli_anything/browser/tests/test_core.py -v
# Sonuç: 31/31 PASSED
```

E2E tests 10 taneydi. Chrome + DOMShell extension gerekiyordu. Gerçek sayfalarda test ediyordu. DOMShell yoksa testler skip oluyordu, böylece CI/CD patlamıyordu.

```bash
DOMSHELL_E2E=1 pytest cli_anything/browser/tests/test_full_e2e.py -v
```

## Review Süreci

PR'ı gönderdikten sonra [@omerarslan0](https://github.com/omerarslan0) review yaptı. 9 maddelik feedback aldım. En kritikleri:

- **Daemon mode context manager leak** - `_daemon_client_context` global değişkeni ile düzeltildi
- **go_back()/go_forward()** - Local history yerine native MCP tool'ları kullanıldı
- **REPL quoted arguments** - `shlex.split()` ile düzeltildi, tırnaklı argümanlar artık doğru parse ediliyor
- **Security** - `act type` artık typed text'i echo etmiyor, passwordler terminal scrollback'de kalmıyor

Her düzeltme ayrı commit'lerde, her biri review'da belgelendi.

![Browser CLI Interface](/assets/img/posts/2026-03-22-browser-automation-cli/browser-cli-interface.png)
*CLI interface - dosya sistemi komutlarıyla tarayıcı kontrolü.*

## Merge ve Sonuç

22 Mart'ta PR merge edildi. 3,095 satır ekleme, 21 dosya değişti, 15 commit. Bu, CLI-Anything'in ilk MCP backend pattern referans implementasyonu. Gelecekteki MCP entegrasyonları için bir template olacak.

Browser harness CLI-Hub'a eklendi:

```json
{
  "name": "browser",
  "display_name": "Browser",
  "version": "1.0.0",
  "description": "Browser automation via DOMShell MCP server",
  "requires": "Node.js, npx, Chrome + DOMShell extension",
  "homepage": "https://github.com/apireno/DOMShell",
  "install_cmd": "pip install git+https://github.com/HKUDS/CLI-Anything.git#subdirectory=browser/agent-harness",
  "entry_point": "cli-anything-browser",
  "category": "web",
  "contributor": "furkankoykiran"
}
```

## Öğrendiklerim

### MCP Python SDK
`stdio` transport ile subprocess communication, async → sync wrapper dönüşümü, tool calling pattern'leri.

### Accessibility Tree vs DOM
Accessibility Tree daha stabil. DOM query'lerden daha az kırılgan. Agent navigation için ideal.

### Open Source
Issue'dan implementasyona geçiş, review feedback yönetimi, documentation update'leri, test coverage önemi.

### Daemon Mode
Context manager lifecycle, state persistence, event loop limitations - hepsi yeni öğrendiklerim.

## Gelecek

V1'de dışarıda bıraktıklarım: screenshot capture, wait-for-element, form fill helper, headless mode, multi-browser (Firefox, Safari), concurrent MCP operations.

[@apireno](https://github.com/apireno) ayrıca `eval` ve `js` escape hatch'lerini önerdi. Element'in temiz AX representation'ı yoksa fallback olarak kullanılabilecek.

Bu katkı sadece bir browser harness değil, aynı zamanda gelecekteki MCP entegrasyonları için bir referans pattern oluşturdu. `cli-anything-plugin/HARNESS.md`'ye eklenen MCP Backend Pattern bölümü, diğer geliştiricilere rehber olacak.

Açık kaynak dünyasında bu tür bir pattern'ı ilk kez implement etmek ilginçti. Review sürecinde [@omerarslan0](https://github.com/omerarslan0), [@yuh-yang](https://github.com/yuh-yang) ve [@apireno](https://github.com/apireno)'nın feedback'leri katkının kalitesini artırdı.

---

**Referanslar:**
- [Issue #90](https://github.com/HKUDS/CLI-Anything/issues/90)
- [PR #118](https://github.com/HKUDS/CLI-Anything/pull/118)
- [DOMShell](https://github.com/apireno/DOMShell)
- [CLI-Anything](https://github.com/HKUDS/CLI-Anything)
