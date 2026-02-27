---
title: "Yapay Zeka ile Makale Yazıp, Hedefe Ulaşmak: DevTo-MCP & Gemini Hackathon Deneyimi"
description: "Kendi geliştirdiğim DevTo-MCP sunucusunu kullanarak, Google Gemini ile hazırladığım MLH Hackathon makalesini komut satırından hiç çıkmadan DEV Community'de nasıl yayınladığımın meta hikayesi."
date: "2026-02-27 15:20:00 +0300"
categories: [Open Source, AI, Hackathon]
tags: [gemini, devto, mcp, mlh, typescript]
image:
  path: /assets/img/2026-02-27-mcp-submission/gemini-logo.png
  alt: "Google Gemini Logo"
---

Eğer düzenli takip ediyorsanız, geçtiğimiz günlerde [DevTo-MCP: DEV Community API'si ile AI Asistanları Arasında Güvenli Bir Köprü](/posts/devto-mcp-server-gelistirme-ve-yayinlama/) başlıklı yazımda, **Forem API**'sini Model Context Protocol (MCP) kullanarak yapay zeka araçlarına nasıl entegre ettiğimi uzun uzun anlatmıştım.

Peki bu aracı sırf "yapabiliyorum" demek için mi yazdım? Tabii ki hayır. Bugün o projenin gerçek hayattaki ilk büyük sınavından, hatta biraz "rüyalar içinde rüya" tadındaki bir deneyimden bahsedeceğim.

Gündemimizde **Major League Hacking (MLH)** ve DEV Community'nin ortaklaşa düzenlediği o meşhur **"Built with Google Gemini: Writing Challenge"** yarışması vardı.

![DEV Community Logo](/assets/img/2026-02-27-mcp-submission/dev-logo.png)
*DEV Community - Yeni evimiz*

## Bir Meydan Okuma Başlıyor

Madem asistanıma (benim durumumda Google Gemini odaklı bir ajana) DEV ortamında araştırma yapma ve makale yayınlama yeteneği kazandırmıştım, neden bu yeteneği bizzat yarışmanın kendisine katılmak için kullanmayayım ki?

Kod editörümden hiç çıkmadan asistanıma şu basit talimatı verdim:
*"Benim için MLH'nin Built with Google Gemini yarışmasını araştır, önceden bu blogda anlattığım DevTo-MCP yazısını oku ve bu süreci anlatan, yarışma şablonuna uygun 150+ satırlık bir İngilizce makale hazırla. Taslağı bitirince de doğrudan DEV.to hesabımda yayınla."*

Kulağa bilimkurgu gibi geliyor değil mi? Ama değil.

## Yapay Zeka Nasıl "Ajan" Oldu?

Eskiden yapay zekalar sadece bizim verdiğimiz metinlere cevap dönen akıllı papağanlar gibiydi. Şimdi ise MCP (Model Context Protocol) sayesinde elleri, kolları olan birer mühendise dönüştüler. 

Asistanım komutu aldıktan sonra şu adımları tamamen tek başına, benim ekranı izlediğim süre zarfında halletti:
1. Gidip MLH yarışmasının detaylarını, son katılım tarihlerini ve Markdown şablonunu okudu.
2. Oturup benim kişisel blogumdaki DevTo-MCP mimarisini anlattığım yazıyı satır satır taradı. (Zod ile yaptığım runtime validasyonunu bile hesaba katmış, şaşırdım).
3. Hem teknik detayları hem de bir "maker" hikayesini içeren, yarışmanın konseptine birebir uyan şahane bir makale hazırladı.
4. Yazının sonuna "P.S. As a special proof-of-concept, this very article was drafted, formatted, and published autonomously to DEV.to by my AI Assistant (Gemini) using the DevTo-MCP server I built!" notunu sıkıştırdı. Harika bir dokunuş.
5. Son olarak, benim kendi açık kaynak projem olan **DevTo-MCP**'yi kullanarak DEV API'sine bağlandı ve makaleyi yayınladı.

Evet yanlış duymadınız, kendi yazdığım aracı kullanarak yarışmaya benim adıma başvurdu.

## mcp:google-image-search: İşin Görünen Yüzü

Şu an okuduğunuz bu Türkçe blog yazısı ve içerisindeki görseller de aslında aynı otonom sürecin bir parçası. 

Yazıyı hazırlamadan önce asistanıma *"Bu olanları anlatan bir blog yazısı hazırla ve `mcp:google-image-search` ile konuyla alakalı yüksek kaliteli, arkaplanı şeffaf ve telifsiz görseller bulup klasöre indir"* komutunu verdim. 

Aradaki DEV, Gemini ve hemen aşağıda göreceğiniz Raspberry Pi AI Kit görselleri tamamen asistanımın otonom olarak arayıp bulduğu ve blog repo'ma indirdiği gerçek dosyalar.

![Raspberry Pi AI Kit](/assets/img/2026-02-27-mcp-submission/raspberry-pi-ai-kit.jpg)
*Belki de yarışmayı kazanırsak bu cihazla yepyeni projeler inşa ederiz, kim bilir?*

## Son Söz

Eskiden bir yarışmaya katılmak veya bir sistemi entegre etmek saatlerimizi, belki günlerimizi alırdı. Şimdi ise sağlam bir altyapı (MCP + API yetenekleri) kurduktan sonra fikirleri hayata geçirmek dakikalar sürüyor. 

DevTo-MCP'yi yazarken asıl amacım tam olarak buydu: "Zaman alan manuel işleri otomatize edip, odaklanmamız gereken yaratıcı kısma (hikayeye) daha fazla zaman ayırmak." Sanırım MLH yarışmasına otonom olarak katılarak bu amacın kanıtını da sunmuş olduk.

Yarışma sonucu ne olursa olsun, bir fikrin koddan pratiğe ve en nihayetinde tamamen yapay zeka tarafından yönetilen bir akışa dönüşmesini izlemek muazzamdı. 

Siz de bu aracı incelemek veya projeye katkıda bulunmak isterseniz GitHub repoma göz atabilirsiniz.

[👉 GitHub: furkankoykiran/DevTo-MCP](https://github.com/furkankoykiran/DevTo-MCP)

Kodla ve bağlamla kalın.

---

**BKZ:**
- [DevTo-MCP: DEV Community API'si ile AI Asistanları Arasında Güvenli Bir Köprü](/posts/devto-mcp-server-gelistirme-ve-yayinlama/)
- [OmniWire-MCP: AI Modelleri İçin Haber Köprüsü](/posts/omniwire-mcp-ai-news-server/)
- [GitHub MCP Server'a Katkı Macerası](/posts/github-mcp-server-acik-kaynak-katki/)

![Furkan Köykıran](/assets/img/avatar.png)
*Furkan Köykıran - Senior Software Engineer*
