---
title: "Modern Web Mimarisi: Framework'süz Düşünmek"
description: "React/Vue her derde deva mı? 'Islands Architecture' nedir? Vanilla JS ve Web Components ile dependency cehenneminden kaçış."
date: "2025-11-15 11:00:00 +0300"
categories: [Frontend, Web Architecture]
tags: [javascript, performance, architecture, web-components, optimization]
image:
  path: /assets/img/posts/vanilla-javascript-dom-manipulation.jpg
  alt: "Architecture Comparison"
---

Frontend dünyası bir sarkaç gibidir. 2010'da PHP ile sunucuda render ediyorduk. 2020'de React ile tarayıcıda render ettik.
Şimdi ise "Sunucuya geri dönelim ama tarayıcıyı da akıllı kullanalım" (SSR + Hydration) diyoruz.
Bu baş döndürücü hızda değişmeyen tek şey: **Temel Web Standartları.**
Kariyerini bir Framework üzerine kuran geliştirici, o Framework ölünce (Bkz: AngularJS) işsiz kalır.
Standartlara (DOM, Event Loop, CSSOM) hakim olan ise her zaman ayakta kalır.
Bugün, bir framework kullanmadan önce sormanız gereken "Neden?" sorusunu ve native alternatiflerini inceleyeceğiz.

![Vanilla JS Architecture](/assets/img/posts/vanilla-javascript-dom-manipulation.jpg)
*Framework Abstraction Layer yükü vs Native DOM performansı.*

## 1. Bundle Size Sendromu ve Core Web Vitals

Bir "Merhaba Dünya" React uygulamasının 200KB JavaScript indirmesi normal mi? Bence değil.
Kullanıcıların %60'ı mobilden giriyor ve herkesin 5G bağlantısı yok.
Google'ın **Core Web Vitals** metrikleri (LCP, FID, CLS) artık en kritik SEO kriteri.
JavaScript bundle'ınız ne kadar büyükse, "Main Thread" o kadar meşgul olur ve siteniz o kadar geç tepki verir.

**Vanilla JS Avantajı:**
Saf JavaScript ile yazdığınızda `runtime` yoktur. Tarayıcının kendisi runtime'dır.
5KB'lık bir script ile modern bir Slider yapabilecekken, sırf slider için koca bir kütüphane indirmek "Mühendislik Tembelliği"dir.
React'ın "Virtual DOM"u, küçük güncellemelerde native DOM'dan daha yavaş olabilir çünkü ekstra bir "Diffing" işlemi gerektirir (O(n) complexity).

## 2. Islands Architecture (Adacık Mimarisi)

Günümüzün en popüler mimarisi budur (Astro, Fresh).
Sayfanın %90'ı statik HTML (Static) olsun. (Header, Footer, Makale metni).
Sadece etkileşim gereken yerler (Sepet butonu, Yorum alanı) JavaScript ile canlansın (Hydration).

```javascript
// Sadece .interactive class'ı olan elementleri bul ve canlandır
document.querySelectorAll('.interactive').forEach(el => {
    // Dynamic Import ile sadece gerektiğinde kodu indir (Lazy Loading)
    import(`./components/${el.dataset.component}.js`)
        .then(module => module.init(el));
});
```
Bu yaklaşım, TTI (Time to Interactive) süresini milisaniyelere düşürür. Tarayıcı önceliği içeriğe verir.

## 3. Web Components: React'ı Tarayıcıya Gömmek

React'ın Component mantığı harika. Ama React kütüphanesi şart mı?
Tarayıcılar artık **Web Components** (Shadow DOM, Custom Elements) destekliyor. Bu standart, Framework'lerden bağımsızdır.

```javascript
class ProductCard extends HTMLElement {
    constructor() {
        super();
        this.attachShadow({ mode: 'open' }); // Stil izolasyonu (Scoped CSS)
    }
    connectedCallback() {
        this.shadowRoot.innerHTML = `
            <style>
                .card { border: 1px solid #ccc; padding: 10px; border-radius: 8px; }
            </style>
            <div class="card">
                <h3>${this.getAttribute('title')}</h3>
                <p><slot name="description"></slot></p>
            </div>
        `;
    }
}
customElements.define('product-card', ProductCard);
```
HTML'de `<product-card title="Ayakkabı">...</product-card>` yazarsınız ve çalışır. Şirket içi bir UI Kit (Design System) yazıyorsanız, Web Components en mantıklı seçimdir çünkü her framework ile çalışır.

![Modern DOM API](/assets/img/posts/css-flexbox-grid-comparison.jpeg)
*DOM Ağacı ve Render Süreci optimizasyonu.*

## 4. State Yönetimi: Redux Şart mı?

En büyük yalanlardan biri: "Uygulamam çok karmaşık, Redux lazım."
Tarayıcıda zaten bir Event Bus var: **DOM Events**.
Bir component `dispatchEvent(new CustomEvent('cart-updated', { detail: count }))` diyerek olay fırlatır.
Diğeri `window.addEventListener` ile dinler.
Bu "Pub/Sub" yapısı, çoğu orta ölçekli proje için yeterlidir ve 0 byte dependency gerektirir.

### Alternatif: Signals Pattern

React bile (v19 ile) sinyallere (Signals) yaklaşıyor.
Sinyaller, durumu (State) atomik olarak yönetir ve sadece değişen kısmı günceller.
Vue, SolidJS ve Angular bunu yıllardır kullanıyor.
Vanilla JS ile basit bir Signal implementasyonu 10 satırdır:

```javascript
// 1kb'lık bir kütüphane kullanmadan Reactive State
const createSignal = (value) => {
    const subscribers = new Set();
    const read = () => value;
    const write = (newValue) => {
        value = newValue;
        subscribers.forEach(fn => fn(value));
    };
    const effect = (fn) => {
        subscribers.add(fn);
        fn(value);
    };
    return [read, write, effect];
};

const [count, setCount, effect] = createSignal(0);
effect((val) => console.log("Sayaç değişti:", val));
setCount(5); // Konsola "Sayaç değişti: 5" yazar.
```
Bunu DOM ile bağladığınızda, React'ın Virtual DOM diffing maliyetinden tamamen kurtulursunuz. Doğrudan `textContent` güncellenir. O(1) complexity!

## 5. Gelecek Burada: CSS Houdini ve Paint API

Tarayıcının render motoruna (CSS Object Model) müdahale etmeye ne dersiniz?
CSS Houdini ile artık JS yazarak yeni CSS özellikleri yaratabiliyorsunuz.
Örneğin, `border-radius` gibi davranan ama dalgalı kenarlar çizen `paint(wave)` fonksiyonunu kendiniz yazabilirsiniz.
Bu, WebGL kullanmadan yüksek performanslı görsel efektler yaratmanın kapısını açıyor.
Henüz deneysel olsa da, 2026'nın standardı bu olacak.

## 6. Erişilebilirlik (A11y): Yasal Bir Zorunluluk

## 5. Erişilebilirlik (A11y): Yasal Bir Zorunluluk

Modern web sadece "göze hitap etmek" değildir. Sitenizi ekran okuyucu (Screen Reader) kullanan bir görme engelli de kullanabilmelidir.
Framework'ler bazen çıktılarıyla (div soup) bunu bozar.
*   **Semantik HTML:** `div` yerine `header`, `nav`, `main`, `article` kullanın.
*   **ARIA Labels:** İkonlara `aria-label="Kapat"` ekleyin.
*   **Klavye Navigasyonu:** Fare olmadan, sadece `Tab` tuşu ile form doldurulabiliyor mu?
Avrupa ve ABD'de erişilebilir olmayan sitelere ciddi davalar açılmaktadır.

## 6. Performance Profiling: Chrome DevTools

Kodunuzun neden yavaş olduğunu tahmin etmeyin, ölçün.
Chrome DevTools > **Performance** sekmesini açın ve kayıt (Record) alın.
*   **Flame Chart:** JavaScript call stack'in görsel halidir. Hangi fonksiyonun ne kadar sürdüğünü (sarı barlar) görürsünüz.
*   **Layout Thrashing:** Eğer mor barlar görüyorsanız, DOM okuma/yazma işlemlerini ardışık yapıyorsunuz demektir. Batch işlemi yapın. `requestAnimationFrame` kullanın.

![CSS Grid Layout](/assets/img/posts/css-flexbox-grid-comparison.jpeg)
*CSS Grid ve Subgrid ile kompleks dizilimler.*

## 7. Seniör Frontend Mülakat Soruları

**S1: Event Bubbling ve Event Capturing arasındaki fark nedir?**
*Cevap:* Eventler DOM ağacında önce aşağı iner (Capture), sonra yukarı çıkar (Bubble). Delegasyon (event delegation) için Bubbling kullanılır.

**S2: `requestAnimationFrame` neden `setTimeout` yerine tercih edilmelidir?**
*Cevap:* `rAF`, tarayıcının ekran yenileme hızıyla (60hz) senkronize çalışır. `setTimeout` ise bloklanabilir ve frame atlamasına (Jank) sebep olur.

**S3: Critical Rendering Path nedir ve nasıl optimize edilir?**
*Cevap:* Tarayıcının HTML'i alıp ekrana piksel basana kadarki sürecidir. Render-blocking CSS ve JS'lerin ertelenmesi (defer/async) ile hızlandırılır.

## 8. Sık Yapılan Hatalar (Anti-Patterns)

1.  **Div Soup:** Her şey için `<div>` kullanmak. Hem SEO'yu hem erişilebilirliği (A11y) öldürür.
2.  **Megalomaniac Bundles:** `lodash` kütüphanesinin tamamını import etmek (`import _ from 'lodash'`). Sadece `import debounce from 'lodash/debounce'` şeklinde kullanın (Tree Shaking).
3.  **Inline Styles:** JS içinde stil objeleri yaratmak. CSS değişkenleri (CSS Variables) varken buna gerek yok.

## Sonuç

"No-Framework" demek, tekerleği yeniden icat etmek demek değildir. Arabanızın motorunu (Tarayıcı API'leri) tanımak demektir.
Bir gün React projesinde performans sorunu yaşarsanız, sorunu çözecek kişi React dokümantasyonunu ezberleyen değil, tarayıcının Reflow/Repaint mantığını bilen kişi olacaktır.
Mimaride sadelik (Simplicity), ulaşılması en zor karmaşıklık seviyesidir.
Basit tutun, native kalın.

### Sürekli Denetim (Continuous Inspection)

Kod kalitesini manuel kontrol etmek yerine CI pipeline'ına şunları ekleyin:
*   **Lighthouse CI:** Her PR'da performans skorunu ölçer. Skor 90'ın altına düşerse merge'i engeller.
*   **Axe-core:** Erişilebilirlik hatalarını (Kontrast, Label eksikliği) otomatik bulur.
*   **Bundle Analyzer:** "Hangi kütüphane kaç KB yer kaplıyor?" sorusunun cevabını görsel olarak verir.


![Responsive Design](/assets/img/posts/css-variables-theming.png)
*Modern CSS ile her cihaza uyumlu native çözümler.*