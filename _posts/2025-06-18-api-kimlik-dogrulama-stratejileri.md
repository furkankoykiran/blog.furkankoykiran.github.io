---
layout: post
title: "API Güvenliği Manifestosu: JWT'nin Ötesinde Zero-Trust"
date: 2025-06-18 10:00:00 +0300
categories: [Backend, Security]
description: "JWT'nin 'alg:none' tuzağı, PASETO devrimi, API Key Hashing ve mTLS ile Zero Trust mimarisi üzerine senior notlar."
image: assets/img/posts/api-security-best-practices.jpg
---

Bir backend geliştiricisi için en korkulu rüya, veritabanının silinmesi değil, kullanıcı verisinin sızdırılmasıdır. Veritabanını yedekten dönersiniz ama itibarınızı geri getiremezsiniz.

Sektörde "Token aldım, header'a koydum, bitti" sığlığından kurtulup, **Zero Trust** (Kimseye Güvenme) prensibini konuşmanın vakti geldi.

## 1. API Key: En Büyük Yalan

Pek çok developer, API Key'leri veritabanında "text" (plaintext) olarak saklar.
**Bu, şifreleri text olarak saklamakla aynı suçtur.**

Bir saldırgan veritabanınızı dump ederse, tüm müşterilerinizin API Key'lerini ele geçirir.
**Doğrusu:**
1.  API Key'i oluşturun: `sk_live_xyz...`
2.  Kullanıcıya **sadece bir kez** gösterin.
3.  Veritabanına `SHA-256` veya `Argon2` hash'ini kaydedin.
4.  Kullanıcı istek attığında, gelen key'i hash'leyip veritabanındaki hash ile karşılaştırın.

Github ve Stripe bunu yıllardır yapıyor. Siz de yapın.

## 2. JWT vs PASETO: Kral Öldü mü?

JWT (JSON Web Token) standardı o kadar esnektir ki, bu esneklik güvenlik açığı doğurur.
Ünlü **"alg: none"** saldırısını duydunuz mu? Hacker, token header'ındaki algoritmayı `none` yapar, imzayı siler ve backend bunu geçerli sayar.

**Yeni Standart: PASETO (Platform-Agnostic Security Tokens)**
PASETO, geliştiriciye "algoritma seçme" şansı tanımaz.
*   **V4 Public:** RSA yerine modern Ed25519 kullanır.
*   **V4 Local:** Simetrik şifreleme (Payload şifrelidir, okunamaz).

```python
# JWT'de algoritma kafa karıştırır
jwt.encode(payload, key, algorithm="HS256")

# PASETO'da hata yapma şansınız yoktur
paseto.create(key, payload, purpose="local")
```
2025 yılında sıfırdan proje yazıyorsanız, JWT yerine PASETO kullanmayı ciddi ciddi düşünün.

## 3. Zero Trust: mTLS (Mutual TLS)

Mikroservis mimarisinde Servis A, Servis B'yi çağırırken genelde kimlik sormaz (VPN içindeyiz ya!).
Peki ya saldırgan container içine sızarsa? Tüm servislerinize erişir.

**Çözüm: mTLS**
Sadece istemci sunucuyu değil, sunucu da istemciyi doğrular. Her servisin kendi SSL sertifikası vardır. Google'ın kendi iç altyapısında (BeyondCorp) kullandığı temel prensip budur.

Python (FastAPI) tarafında değil, bunu **Service Mesh** (Istio/Linkerd) veya **Reverse Proxy** (Nginx/Traefik) katmanında çözmek en performanslısıdır.

## 4. OAuth2 ve OpenID Connect: Yetki Karmaşası

Kavramları karıştırmayın:
*   **OAuth2:** Yetkilendirme (Authorization) protokolüdür. "Bu uygulamanın fotoğraflarıma erişim izni var mı?" sorusuna cevap verir.
*   **OIDC (OpenID Connect):** Kimlik Doğrulama (Authentication) protokolüdür. "Bu kullanıcı kim?" sorusuna cevap verir.

Kendi OAuth2 sunucunuzu yazmaya kalkmayın. Bu, kriptografi kütüphanesini kendiniz yazmaya benzer. Ory Hydra, Keycloak veya Auth0 kullanın.

## 5. Session vs Token: Ebedi Savaş

Stateless (Token) mimari her zaman en iyisi değildir.

| Özellik | Session (Redis) | JWT/PASETO |
| :--- | :--- | :--- |
| **İptal (Revoke)** | Anında (Redis'ten sil) | İmkansız (Süre bitene kadar geçerli) |
| **Boyut** | Küçük (32 byte ID) | Büyük (Payload arttıkça şişer) |
| **Kullanım** | Klasik Web App | Mobile App, SPA, Mikroservis |

**Senaryo:** Kullanıcının şifresi çalındı.
*   **Session:** Tüm oturumları tek tuşla kapatırsınız.
*   **JWT:** Access Token süresi (örn: 15 dk) dolana kadar hacker içeridedir. "Blacklist" tutacaksanız, JWT'nin stateless olmasının ne anlamı kaldı?

## 6. Güvenlik Duvarları: Rate Limiting ve CORS

**CORS (Cross-Origin Resource Sharing):**
`Access-Control-Allow-Origin: *` yapmak development tembelliğidir, production intiharıdır. Sadece izinli domainleri (örn: `https://app.domain.com`) whitelist'e ekleyin.

**Rate Limiting:**
Brute-force saldırılarını ve DDoS'u engellemenin en ucuz yoludur. Token Bucket algoritması kullanın. Nginx veya Redis üzerinde yapın.

## 7. Python Örneği: Secure Middleware

FastAPI ile gelen her isteğin `Authorization` header'ını kontrol eden basit ama etkili bir middleware:

```python
from fastapi import Request, HTTPException
from jose import jwt, JWTError

async def auth_middleware(request: Request, call_next):
    if request.url.path in ["/login", "/docs"]:
        return await call_next(request)

    token = request.headers.get("Authorization")
    if not token or not token.startswith("Bearer "):
        raise HTTPException(status_code=401, detail="Token missing")

    try:
        # Verify signature & expiration
        payload = jwt.decode(token.split()[1], SECRET_KEY, algorithms=["HS256"])
        request.state.user = payload # User'ı request'e ekle
    except JWTError:
        raise HTTPException(status_code=403, detail="Invalid token")

    response = await call_next(request)
    return response
```


## 9. Replay Attack Önlemleri: "JTI" ve "Nonce"

Bir saldırgan, kullanıcının gönderdiği geçerli bir API isteğini (Örn: "Para Gönder") dinleyip, aynı isteği tekrar tekrar sunucuya gönderirse ne olur?

Buna **Replay Attack** denir. Önlemenin yolları:

1.  **JTI (JWT ID):** Her token'a benzersiz bir ID verin. Sunucuda (Redis) kullanılan ID'leri kısa bir süre (exp süresi kadar) saklayın. Aynı JTI ile ikinci istek gelirse reddedin.
2.  **Timestamp Kontrolü:** İstek payload'una `timestamp` ekleyin. Sunucu zamanından (türbülans payı hariç) 5 dakikadan daha eski istekleri kabul etmeyin.
3.  **Signature:** Bu timestamp'i de imzalayın ki saldırgan değiştiremesin.

## 10. Pratik mTLS Kurulumu

mTLS teoride güzeldir ama pratikte "Sertifika Cehennemi"dir. İşte minimal bir OpenSSL reçetesi:

**1. CA (Certificate Authority) Oluştur:**
```bash
openssl req -x509 -sha256 -newkey rsa:4096 -keyout ca.key -out ca.crt -days 365
```

**2. Server Sertifikası:**
```bash
openssl req -new -newkey rsa:4096 -keyout server.key -out server.csr
openssl x509 -req -CA ca.crt -CAkey ca.key -in server.csr -out server.crt
```

**3. Client Sertifikası:**
Aynı işlemi client (istemci) için yapın.

**4. Nginx Config:**
```nginx
server {
    listen 443 ssl;
    ssl_certificate server.crt;
    ssl_certificate_key server.key;
    
    # Client doğrulama aktif!
    ssl_client_certificate ca.crt;
    ssl_verify_client on; 
}
```
Artık client sertifikası olmayan kimse (hackerlar dahil) Nginx'e "Hello" bile diyemez. Bağlantı TCP seviyesinde reddedilir.

## 11. Audit Logging: "Kim Yaptı?"

Güvenlik sadece engellemek değil, izlemektir. Bir veri sızıntısı olduğunda "Hangi user, hangi IP'den, saat kaçta, hangi endpoint'e erişti?" sorusuna cevap veremiyorsanız, yandınız.

Basit bir `access.log` yetmez. Yapısal (Structured) log gerekir:

```json
{
  "timestamp": "2025-06-18T10:22:00Z",
  "user_id": "12345",
  "action": "DELETE_PRODUCT",
  "resource_id": "987",
  "ip_address": "203.0.113.1",
  "status": "SUCCESS",
  "user_agent": "Mozilla/5.0..."
}
```
Bu logları asla uygulamanın olduğu sunucuda tutmayın. Anında **Log Aggregator**'a (Elasticsearch, CloudWatch, Datadog) gönderin. Hacker sunucuya girerse ilk iş yerel logları siler.


## Özetle

Güvenlik bir ürün değil, bir süreçtir. "Authentication" kapıdan geçiş iznidir, "Authorization" ise hangi odalara girebileceğinin iznidir. İkisini karıştırmayın, kütüphanelerin varsayılan ayarlarına güvenmeyin ve asla, ama asla kendi kripto algoritmanızı yazmayın.
