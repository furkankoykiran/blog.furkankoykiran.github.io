---
title: "Keploy'a Katkılarım: Go'da Resource Management ve Open Source Serüveni"
description: "Keploy projesine yaptığım iki bug fix PR'ı, Go'da defer pattern ve HTTP connection pooling hakkında öğrendiklerim."
date: 2026-03-20 18:00:00 +0300
categories: [Açık Kaynak, Go]
tags: [open-source, go, keploy, defer, resource-management]
image:
  path: /assets/img/posts/2026-03-20-keploy/how-keploy-works.png
  alt: "Keploy Architecture Diagram"
---

[awesome-trending-repos](https://github.com/furkankoykiran/awesome-trending-repos) projem ile GitHub trending'deki popüler repoları takip ederken, Go ile yazılmış modern bir API testing aracı olan [Keploy](https://github.com/keploy/keploy) dikkatimi çekti. Kod tabanını incelerken düzeltebileceğim birkaç resource management bug'ı buldum.

![Keploy Architecture](/assets/img/posts/2026-03-20-keploy/how-keploy-works.png)
*Keploy, API, Integration ve E2E testing için otomatik test ve mock oluşturabilen modern bir araç.*

---

## Keploy Nedir?

Keploy, API ve entegrasyon testing için geliştirilmiş açık kaynak bir araç. Mock oluşturma, test generation gibi özellikler sunuyor ve GitHub'da 8k+ yıldızı var. VSCode extension ile entegre çalışabiliyor.

![Keploy VSCode Extension](/assets/img/posts/2026-03-20-keploy/keploy-vscode-extension.png)
*VSCode extension ile Keploy, IDE içinden test ve mock yönetimi sağlıyor.*

Projeyi incelerken Go'da sık yapılan resource management hatalarını fark ettim. İki farklı PR ile bu sorunları çözdüm.

---

## PR #3927: Use-After-Close Bug'ı

### Problemi Keşfetmek

Projenin GitHub issue'larını incelerken, [#3821 numaralı issue](https://github.com/keploy/keploy/issues/3821)'de raporlanan bir sorunu fark ettim. `utils/utils.go` dosyasındaki `isGoBinary` fonksiyonunda, dosya açılıp hemen kapatılıyor, sonra kapatılmış dosya üzerinde section okuma işlemi yapılıyordu.

```go
// HATALI KOD
f, err := elf.Open(filePath)
if err != nil {
    logger.Debug(fmt.Sprintf("failed to open file %s", filePath), zap.Error(err))
    return false
}
if err := f.Close(); err != nil {  // ❌ Hemen kapatılıyor
    LogError(logger, err, "failed to close file", zap.String("file", filePath))
}
// Sonra kapatılmış dosya üzerinde işlem yapılıyor
sections := []string{".go.buildinfo", ".gopclntab"}
for _, section := range sections {
    if sect := f.Section(section); sect != nil {  // ❌ Use-after-close
        fmt.Println(section)
        return true
    }
}
```

Bu, Go'da "use-after-close" olarak bilinen bir hataya yol açar. Dosya kapatıldıktan sonra dosya tanımlayıcısı (file descriptor) artık geçerli değildir.

### Çözüm: Defer Pattern

Go'da bu sorunu çözmek için `defer` pattern'i kullanılır. `defer` ile belirtilen ifade, fonksiyon return olmadan hemen önce çalıştırılır. Bu sayede kaynak, fonksiyon ne şekilde sonlanırsa sonlansın (early return, panic, normal return) garanti olarak kapatılır.

```go
// DÜZELTİLMİŞ KOD
f, err := elf.Open(filePath)
if err != nil {
    logger.Debug(fmt.Sprintf("failed to open file %s", filePath), zap.Error(err))
    return false
}
defer func() {
    if err := f.Close(); err != nil {
        LogError(logger, err, "failed to close file", zap.String("file", filePath))
    }
}()
// Artık dosya açık, section okuma yapılabilir
sections := []string{".go.buildinfo", ".gopclntab"}
for _, section := range sections {
    if sect := f.Section(section); sect != nil {
        return true
    }
}
```

### Code Review Feedback

Copilot code reviewer'dan bir feedback aldım. `defer f.Close()` şeklinde doğrudan kullanımda, Close() tarafından dönen error ignore ediliyordu. Bu da repository'deki diğer yerlerde kullanılan pattern'e uymuyordu.

Feedback'i ele alarak defer'i bir closure içine aldım ve error handling'i ekledim:

```go
defer func() {
    if err := f.Close(); err != nil {
        LogError(logger, err, "failed to close file", zap.String("file", filePath))
    }
}()
```

[PR #3927](https://github.com/keploy/keploy/pull/3927) başarıyla merge edildi.

---

## PR #3932: HTTP Response Body Leak

### Problemi Keşfetmek

Projenin issue tracker'ında, [#3854 numaralı issue](https://github.com/keploy/keploy/issues/3854) ile raporlanan bir sorun tespit ettim. `pkg/platform/http/agent.go` dosyasında `MockOutgoing` ve `UpdateMockParams` fonksiyonlarında HTTP request yapıldıktan sonra response body kapatılmıyordu.

```go
// HATALI KOD
res, err := a.client.Do(req)
if err != nil {
    return fmt.Errorf("failed to send request: %s", err.Error())
}
// Body kapatılmıyor! ❌

var mockResp models.AgentResp
err = json.NewDecoder(res.Body).Decode(&mockResp)
```

Bu durum birkaç soruna yol açar:

1. **Resource Leak**: Açık kalan body'ler memory leak'e neden olur
2. **Connection Pool Exhaustion**: HTTP client'ının connection pool'u tükenir
3. **Port Exhaustion**: Çok sayıda açık connection, sistem portlarını tüketebilir

### Çözüm: Defer Close + Drain Pattern

Go'nun `http` package'i, response body'yi kapatmak için çağıranın sorumlu olduğunu belirtir. Body kapatılmazsa, underlying TCP connection connection pool'a geri dönmez ve yeni bir connection oluşturulması gerekir.

```go
// DÜZELTİLMİŞ KOD
res, err := a.client.Do(req)
if err != nil {
    return fmt.Errorf("failed to send request: %s", err.Error())
}
defer func() {
    io.Copy(io.Discard, res.Body)  // Body'yi EOF'a kadar drain et
    if err := res.Body.Close(); err != nil {
        utils.LogError(a.logger, err, "failed to close response body")
    }
}()

var mockResp models.AgentResp
err = json.NewDecoder(res.Body).Decode(&mockResp)
```

Burada iki önemli işlem yapıyoruz:

1. **`io.Copy(io.Discard, res.Body)`**: Body'yi EOF'a kadar okuyup discard ediyoruz. Bu, connection'ın reuse edilmesini sağlar çünkü JSON decoder tüm body'yi okumamış olabilir.
2. **`res.Body.Close()`**: Body'yi kapatıyoruz ve error'u logluyoruz.

### Neden Drain Ediyoruz?

Copilot reviewer'dan öğrendiğim önemli bir detay: `json.Decoder.Decode()` tüm body'yi okumama garantisi vermez. Eğer body'de okunmamış byte'lar kalırsa ve direkt kapatırsak, HTTP client connection'ı yeniden kullanamaz (keep-alive çalışmaz).

`io.Copy(io.Discard, res.Body)` ile body'yi EOF'a kadar drain ediyoruz, böylece connection pool'a geri dönülebilir.

[PR #3932](https://github.com/keploy/keploy/pull/3932) başarıyla merge edildi.

---

## Go'da Resource Management Best Practices

Bu iki PR ile Go'da resource management hakkında önemli dersler öğrendim:

### 1. Defer Pattern Kullanımı

```go
// Dosya işlemleri
file, err := os.Open("file.txt")
if err != nil {
    return err
}
defer file.Close()

// Mutex kilitleme
mu.Lock()
defer mu.Unlock()

// Database transaction
tx, err := db.Begin()
if err != nil {
    return err
}
defer tx.Rollback()  // Commit olmazsa rollback
```

### 2. Defer'de Error Handling

```go
// ❌ Error ignore edilir
defer file.Close()

// ✅ Error handle edilir
defer func() {
    if err := file.Close(); err != nil {
        log.Printf("close error: %v", err)
    }
}()
```

### 3. HTTP Connection Reuse

```go
// ❌ Connection yeniden kullanılamaz
defer res.Body.Close()

// ✅ Connection pool'a geri döner
defer func() {
    io.Copy(io.Discard, res.Body)
    res.Body.Close()
}()
```

### 4. Defer Execution Order

Defer statements LIFO (Last In, First Out) sırasıyla çalışır:

```go
defer fmt.Println("1")  // Son: "1"
defer fmt.Println("2")  // İkinci: "2"
defer fmt.Println("3")  // İlk: "3"
// Çıktı: 3, 2, 1
```

---

## Öğrendiklerim

Bu açık kaynak katkısı sürecinden öğrendiklerim:

| Konu | Öğrenilen Ders |
|------|----------------|
| **Go Defer Pattern** | Resource cleanup için defer kullanımı, error handling |
| **HTTP Connection Pooling** | Response body drain etmek, connection reuse |
| **Code Review** | Copilot feedback'i ile code kalitesini artırmak |
| **Open Source Communication** | PR description yazma, maintainer ile iletişim |

Maintainer'ların code review'leri çok öğreticiydi. Kod stili, edge case'ler ve error handling hakkında detaylı feedback aldım.

---

## Sonuç

Keploy projesine yaptığım bu iki küçük katkı, bana Go'da resource management konusunda pratik bilgi kazandırdı. Use-after-close ve HTTP response leak gibi subtle bug'lar, production'da büyük sorunlara yol açabilir.

![Keploy Logo](/assets/img/posts/2026-03-20-keploy/keploy-logo.svg)
*Keploy açık kaynak bir proje olarak katkılara açık.*

Açık kaynak dünyasına katkı vermek sadece kod yazmak değil, aynı zamanda community ile etkileşim kurmak ve öğrenmek demek. Her PR, hem projeye hem de kendinize değer katar.

Eğer siz de Go'da resource management hakkında daha fazla bilgi edinmek isterseniz, [Effective Go](https://go.dev/doc/effective_go) ve [Common Mistakes in Go](https://github.com/golang/go/wiki/CommonMistakes) sayfalarını incelemenizi öneriyorum.

---

**Benzer Yazılar:**
- [awesome-trending-repos: GitHub Trending Takip Sistemi](/posts/awesome-trending-repos/)
- [GitHub MCP Server Projesine Katkı](/posts/github-mcp-server-acik-kaynak-katki/)
- [MCP Ekosistemi Katkılarım](/posts/mcp-ai-contributions/)

![Furkan Köykıran](/assets/img/avatar.png)
*Furkan Köykıran - Senior Software Engineer*
