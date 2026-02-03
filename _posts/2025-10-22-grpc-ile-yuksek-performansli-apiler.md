---
title: "gRPC ve Protobuf: Yüksek Performanslı API Mimarisi"
description: "REST API'lerin yetersiz kaldığı anlar, HTTP/2 streaming, Protobuf ile binary versiyonlama ve microservice dünyasında gRPC kullanımı."
date: "2025-10-22 14:00:00 +0300"
categories: [Backend, Microservices, API Design, System Architecture]
tags: [grpc, python, protobuf, http2, performance, microservices, backend]
image:
  path: /assets/img/posts/grpc-architecture-workflow.png
  alt: "gRPC Architecture Workflow Diagram"
---

Yıllarca REST API yazdık. JSON aldık, JSON verdik, Swagger dokümanı güncelledik. Endpoint isimleri için (`GET /users/123/orders` mı yoksa `GET /orders?user_id=123` mü?) toplantılarda saatlerce tartıştık.

Ama mikroservis mimarisinin göbeğinde, saniyede 10.000 işlem yapan bir sistem tasarlarken, REST'in aslında ne kadar hantal ve geveze (verbose) bir protokol olduğunu acı bir tecrübeyle anlıyorsunuz.

Bizim ekipte bir gün, iki servis arasındaki basit bir veri transferinin (JSON Serialization/Deserialization) toplam sürenin %60'ını yediğini fark ettik. CPU'lar sadece JSON parse etmek için çalışıyordu. İşte o gün **gRPC (gRPC Remote Procedure Calls)** defterini açtık.

![gRPC Architecture](/assets/img/posts/grpc-architecture-workflow.png)
*İstemci ve Sunucunun, sanki aynı makinedeymiş gibi metot çağırması.*

## 1. REST Neden Yetersiz Kalıyor?

Şunu kabul edelim: **JSON, makineler için değil, insanlar için tasarlandı.**
Okuması kolay ama bilgisayarlar için verimsizdir.

1.  **Metin Tabanlı:** Bir sayı göndermek için `"age": 25` (9 byte) yazıyorsunuz. Binary olsa 1 byte yetecekti.
2.  **Tip Güvensiz:** "age" alanı string mi, int mi? JSON dosyasında bunun garantisi yoktur.
3.  **HTTP/1.1 Yükü:** (gRPC öncesi) Her istek için yeni bağlantı, header'lar, cookie'ler...

Mikroservis A, Mikroservis B'den bir veri istediğinde, B servisi veriyi alıyor, string'e çeviriyor (Serialize), A servisine yolluyor. A servisi string'i alıyor, tekrar objeye çeviriyor (Deserialize). Bu işlem sırasında milyonlarca CPU cycle boşa harcanır.

## 2. Protobuf (Protocol Buffers) Büyüsü

gRPC, veriyi taşımak için metin tabanlı JSON yerine, **binary (ikili)** format olan Protobuf kullanır.

![Protobuf vs JSON](/assets/img/posts/grpc-protobuf-concept.png)
*Protobuf'ın binary yapısı, payload boyutunu %60-80 oranında küçültür.*

### Şema ve Versiyonlama
gRPC'de her şey bir `.proto` dosyasıyla başlar. Bu dosya sizin kontratınızdır (Contract First Design).

```protobuf
syntax = "proto3";

// Versiyonlama için alanlara ID verilir
message UserRequest {
  int32 id = 1;        // Field ID: 1
  string email = 2;    // Field ID: 2
  // string name = 3;  // Alanı silsek bile ID: 3 rezerve kalır!
}
```

> **Senior Notu:** Protobuf'ta alan isimlerini değiştirseniz bile (örn: `email` yerine `mail_address`), ID'ler (`= 2`) aynı kaldığı sürece sistem kırılmaz. JSON API'lerde isim değişikliği (Breaking Change) krizlere yol açarken, gRPC geriye dönük uyumluluk (Backward Compatibility) konusunda muazzamdır.

## 3. İletişim Modelleri: Sadece Request-Response Değil

REST'te elinizde sadece Request-Response vardır. gRPC ise size 4 farklı silah verir:

1.  **Unary RPC:** Klasik metod. Bir istek, bir cevap.
2.  **Server Streaming:** İstemci bir istek atar, sunucu bir video akışı gibi cevap yağdırır.
3.  **Client Streaming:** İstemci (örn: IoT cihazı) sürekli veri gönderir, sunucu işlem bitince tek cevap döner.
4.  **Bidirectional Streaming:** İki taraf da aynı anda konuşur (Chat uygulamaları, oyun sunucuları).

**Python ile Borsa Verisi Akışı (Server Streaming):**

```python
# Server Side
class StockService(stock_pb2_grpc.StockServicer):
    def GetLivePrices(self, request, context):
        stocks = ["AAPL", "GOOGL", "MSFT"]
        while True:
            # Sürekli veri gönder!
            price = get_market_price(request.symbol)
            yield stock_pb2.PriceUpdate(symbol=request.symbol, price=price)
            time.sleep(1)

# Client Side
# Tek bir TCP bağlantısı üzerinden yüzlerce güncelleme alır
for update in stub.GetLivePrices(stock_pb2.StockRequest(symbol="AAPL")):
    print(f"Canlı Fiyat: {update.price}")
```

REST veya Polling ile bunu yapmaya çalışmak, sunucuyu DDoS'lamaktır.

## 4. HTTP/2 ve Load Balancing Sorunsalı

gRPC varsayılan olarak **HTTP/2** üzerinde çalışır. Bu harikadır çünkü **Multiplexing** (tek bağlantıda çok iş) sağlar. Ancak bu durum Load Balancer'lar (LB) için bir kabustur.

Geleneksel L4 Load Balancer'lar (AWS ELB, Nginx varsayılan) bağlantı bazlı dağıtım yapar. gRPC tek bir TCP bağlantısını açık tuttuğu için, LB tüm istekleri **tek bir sunucuya** yönlendirir. Diğer sunucular boşta yatarken, o sunucu çöker.

**Çözüm:**
*   **L7 Load Balancing:** Envoy Proxy, Linkerd veya Istio gibi gRPC protokolünü anlayan (Header'a bakıp isteği dağıtabilen) araçlar kullanmalısınız.
*   **Client-Side Load Balancing:** İstemci, DNS'ten tüm IP'leri alır ve sırayla kendisi dağıtır.

![FastAPI vs gRPC Architecture](/assets/img/posts/fastapi-microservices-architecture.png)

## 5. Performans: Rakamlar Yalan Söylemez

Bir projede, birbirleriyle konuşan 15 mikroservisi REST'ten gRPC'ye geçirdik. Sonuçlar:

*   **Network Trafiği:** %65 azaldı (JSON header'ları ve metin şişkinliği gitti).
*   **CPU Kullanımı:** %40 azaldı (Parse işlemi hafifledi).
*   **Latency (Gecikme):** İç iletişimde 15ms'den 2ms'ye düştü.

![gRPC vs REST Performance](/assets/img/posts/grpc-vs-rest-performance-comparison.png)

## 6. Ne Zaman REST, Ne Zaman gRPC?

gRPC her derde deva sihirli bir değnek değildir.

### gRPC Kullanın:
*   Mikroservisler arası (East-West) iletişimde.
*   Bant genişliğinin kısıtlı olduğu yerlerde (Mobil, IoT).
*   Çok dilli ortamlarda (Örn: Python servisi Go servisini çağırıyorsa).
*   Streaming ihtiyacı varsa.

### REST Kullanın:
*   Doğrudan Web Tarayıcısına (Browser) veri sunuyorsanız (gRPC-Web hala tam olgunlaşmadı).
*   Public API (OpenAPI) sunuyorsanız (Herkes CURL ile test etmek ister).
*   Basit CRUD uygulamalarında.

## 7. Gelişmiş Özellikler: Interceptors ve Error Handling

gRPC'yi production'a alırken iki kritik konu karşınıza çıkar.

### Middleware (Interceptor) Yazmak
Loglama, Auth veya Rate Limiting için her fonksiyona kod yazılmaz. Interceptor kullanın.

```python
class AuthInterceptor(grpc.ServerInterceptor):
    def intercept_service(self, continuation, handler_call_details):
        # Metadata içinden token'ı çek
        metadata = dict(handler_call_details.invocation_metadata)
        token = metadata.get('authorization')
        
        if not valid_token(token):
            # Yetkisiz erişimi burada kes
            def abort(ignored_request, context):
                context.abort(grpc.StatusCode.UNAUTHENTICATED, 'Invalid Token')
            return grpc.unary_unary_rpc_method_handler(abort)
            
        return continuation(handler_call_details)
```

### Zengin Hata Yönetimi (Rich Error Handling)
Sadece "500 Error" dönmek yetmez. Google'ın `google.rpc.Status` modelini kullanın. Hata dönerken yanında "RetryInfo" (3 saniye sonra dene) veya "DebugInfo" (Stack trace) gönderebilirsiniz.

## 8. Karşılaştırma Tablosu: gRPC vs REST

Karar vermenize yardımcı olacak kısa bir özet:

| Özellik | REST (JSON) | gRPC (Protobuf) |
| :--- | :--- | :--- |
| **Payload** | Metin (Büyük) | Binary (Küçük, hızlı) |
| **Protokol** | HTTP/1.1 (Genelde) | HTTP/2 (Default) |
| **Tarayıcı** | %100 Destek | Kısıtlı (gRPC-Web gerekir) |
| **Streaming** | Zor (Long Polling) | Native (Çift yönlü) |
| **Doküman** | OpenAPI (Swagger) | .proto (Contract) |
| **Tip Güvenliği** | Yok (Manual) | Var (Auto-gen) |

## 9. Sıkça Sorulan Sorular

### "Swagger/OpenAPI var mı?"
`grpc-gateway` kullanarak .proto dosyalarından otomatik Swagger üretebilir ve aynı anda hem gRPC hem REST sunabilirsiniz.

### "Versiyonlama nasıl yapılır?"
Package isimlendirmesiyle: `package myapp.v1;` olarak başlarsınız. Kırıcı değişiklik yapacağınız zaman `package myapp.v2;` ile yeni bir .proto dosyası açarsınız. İki versiyon aynı anda çalışabilir.

## 9. Terimler Sözlüğü (Glossary)

gRPC, modern backend mühendisliğinin testeresidir. Keskindir, güçlüdür ama yanlış tutarsanız elinizi keser. Doğru yerde kullanın, sisteminiz uçsun.
