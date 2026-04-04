---
title: "freqtrade-mcp: Bota Claude Üzerinden Sor"
description: "Freqtrade botunu Claude'a bağlayan TypeScript MCP sunucusu. Bakiye, pozisyon, piyasa verisi — hepsi doğal dilde."
date: 2026-04-04 13:00:00 +0300
categories: [MCP, Kripto Ticaret, Açık Kaynak]
tags: [mcp, freqtrade, typescript, claude, kripto]
image:
  path: /assets/img/posts/2026-04-04-freqtrade-mcp/banner.png
  alt: "Freqtrade MCP Sunucusu"
---

Freqtrade botu çalıştırıyorum. Stratejim var, bot işini yapıyor, benim müdahaleye gerek yok. Teorikte.

Pratikte gece 23'te ekrana bakıyorum.

Bir şey kaybetmiş miyim? Yoksa bir fırsat mı kaçtı? Bot kapanmış mıdır? Aynı bakiyeyi görüyorum ama "acaba" duygusu geçmiyor. FreqUI'yı açıyorum, sekmelere tıklıyorum, rakamları görüyorum, kapatıyorum. On beş dakika sonra tekrar açıyorum.

Bu süreci kısaltmak için [@furkankoykiran/freqtrade-mcp](https://github.com/furkankoykiran/freqtrade-mcp)'yi yazdım — Freqtrade'in REST API'sini Claude'a açan, 15 araçlık bir TypeScript MCP sunucusu. Claude Desktop, Cursor, Cline, [Model Context Protocol](https://modelcontextprotocol.io) konuşan her şeyle çalışıyor.

Artık FreqUI açmak yerine Claude'a soruyorum. Cevap geliyor. Geçiyorum.

![Freqtrade FreqUI](/assets/img/posts/2026-04-04-freqtrade-mcp/banner.png)
*Freqtrade — artık açmak zorunda kalmadan da bilgi alabilirsin.*

---

## Neye Yarıyor

### Portföy Durumu

Gece 23'teki sorular genellikle bunlar:

> *"Bu ay net pozitif miyim, yoksa bakmamak mı lazım?"*

> *"Şu an ne kadar USDT açık pozisyonda, ne kadarı boşta bekliyor?"*

> *"Hangi çiftler iyi gidiyor?"*

`get_profit_stats`, `get_balance`, `get_performance` bunları karşılıyor. Claude ham JSON'ı okunaklı hale getiriyor. Sayıları kendin çözmeye çalışmak zorunda kalmıyorsun.

---

### Pozisyonlar

> *"Açık işlemler ne durumda, zararda olan var mı?"*

> *"42 numaralı işlem — açılış tarihi ne, şu anki K/Z ne?"*

> *"Son 10 kapanan işlemi göster."*

Bot aynı anda onlarca pozisyon taşıyabiliyor. Her satırı kendin okumak yerine Claude'a sormak çok daha hızlı. Sonuç doğrudan geliyor.

---

### Piyasa Verisi

> *"BTC/USDT 4 saatlik, son 50 mum."*

> *"ETH/USDT şu an 1 saatlikte nasıl görünüyor?"*

`get_market_data` canlı OHLCV verisini doğrudan borsadan çekiyor. Claude grafiğe bakıp bir yorum yapabilir. Ne yapacağın sana kalmış.

![Freqtrade Backtesting](/assets/img/posts/2026-04-04-freqtrade-mcp/freqtrade-backtesting.png)
*Freqtrade'in analiz ekranı — MCP sunucusu da aynı veriyi anlık çekiyor.*

---

### Liste Yönetimi

> *"LUNA/USDT'yi kara listeye ekle."*

> *"Beyaz listede ne var? BNB/BTC'yi kara listeden çıkar."*

Normalde ya config dosyasını editliyorsun ya da FreqUI'da menüler arasında geziniyorsun. Şimdi tek cümle.

---

### Manuel İşlem

> *"Şu an 100 USDT ETH al."*

> *"BTC/USDT'de 50 USDT ile short aç."*

> *"7 numaralı işlemi kapat."*

`execute_trade` ve `force_exit_trade` strateji beklemeden anında çalışıyor.

> **Önemli:** Gerçek borsa, gerçek para, geri alınmıyor. "Dikkatli ol" burada gerçekten önemli. Onay ekranı yok, borsa "test ediyordum" demeyi tanımıyor.

---

### Bot Yönetimi

> *"Botu durdur, config değiştirecektim."*

> *"Yeniden başlatmadan config'i yenile."*

> *"Bot son ne zaman çalıştı, hâlâ ayakta mı?"*

`get_bot_info` sürüm ve sağlık verisini bir arada veriyor. Botun gerçekten çalışıp çalışmadığını öğrenmek için FreqUI açmana gerek kalmıyor.

---

## Kurulum

Freqtrade `config.json` içinde REST API'yi etkinleştir (varsayılan olarak kapalı gelir), ardından Claude Desktop config'ine ekle:

```json
{
  "mcpServers": {
    "freqtrade": {
      "command": "npx",
      "args": ["-y", "@furkankoykiran/freqtrade-mcp"],
      "env": {
        "FREQTRADE_API_URL": "http://127.0.0.1:8080",
        "FREQTRADE_USERNAME": "Freqtrader",
        "FREQTRADE_PASSWORD": "YourPassword"
      }
    }
  }
}
```

Claude Desktop'ı yeniden başlat. 15 araç hazır, build adımı yok.

---

## Araçlar

| Kategori | Araçlar |
|---|---|
| **Hesap** | `get_profit_stats`, `get_balance`, `get_performance` |
| **İşlemler** | `get_open_trades`, `get_trade`, `get_trade_history` |
| **Piyasa Verisi** | `get_market_data` |
| **Çift Listeleri** | `get_whitelist`, `get_blacklist`, `add_to_blacklist`, `remove_from_blacklist` |
| **İşlem Açma** | `execute_trade`, `force_exit_trade` |
| **Bot Yönetimi** | `start_bot`, `stop_bot`, `reload_config`, `get_bot_info`, `get_locks`, `delete_lock` |

---

## Sonuç

Temelde şunu çözmek istedim: FreqUI'yı açmadan bota bakabileyim. REST API zaten oradaydı, sadece arayüzün arkasına gömülüydü.

Artık Claude'a soruyorum. O söylüyor. Geçiyorum.

Pek tabii "acaba değişmiş mi" duygusu hâlâ geçmiyor, ama en azından sormak kolaylaştı.

Proje açık kaynak ve MIT lisanslı. Freqtrade REST API kurulumu dahil tam kılavuz README'de.

**[github.com/furkankoykiran/freqtrade-mcp](https://github.com/furkankoykiran/freqtrade-mcp)**

---

![Furkan Köykıran](/assets/img/avatar.png)
*Furkan Köykıran - Senior Software Engineer*

---

## Diğer Yazılar

- [CLI-Anything'e Tarayıcı Otomasyonu: İlk MCP Backend Deseni](/posts/cli-anything-browser-automation-contribution-en/)
- [awesome-trending-repos: GitHub Trending için Modern Web Arayüzü](/posts/awesome-trending-repos-web-en/)
- [MCP Ekosistemine Açık Kaynak Katkılarım](/posts/mcp-ai-contributions/)
