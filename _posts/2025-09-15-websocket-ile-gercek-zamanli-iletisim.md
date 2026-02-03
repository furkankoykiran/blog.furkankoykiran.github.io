---
title: "WebSocket ve Gerçek Zamanlı Sistemler: Ölçeklenebilir Mimari Sanatı"
description: "HTTP Polling'in ölümü, FastAPI ve Redis Pub/Sub ile ölçeklenebilir WebSocket mimarisi, Sticky Sessions ve Production zorlukları."
date: "2025-09-15 15:30:00 +0300"
categories: [Backend, Realtime, System Design, Scalability]
tags: [websocket, python, fastapi, redis, scaling, socketio]
image:
  path: /assets/img/posts/websocket-realtime-architecture.png
  alt: "WebSocket Real-Time Architecture Diagram"
---

Bir chat uygulaması, borsa takip ekranı veya canlı maç skoru sitesi yapıyorsanız, HTTP protokolü size yetmez. Neden mi? Çünkü HTTP, doğası gereği "Soru-Cevap" (Request-Response) mantığıyla çalışır. İstemci sormadan, sunucu cevap veremez.

Eskiden **Polling** yapardık. JavaScript ile her 3 saniyede bir sunucuya "Yeni mesaj var mı?" diye sorardık.
*   "Yok."
*   "Yok."
*   "Yok."
*   "Var!"

Bu, sunucuyu kendi elimizle DDoS'lamaktan farksızdır. Gereksiz trafik, yüksek CPU kullanımı ve gecikme (latency) demektir. Pili sömürür, kullanıcıyı üzer.

Çözüm: **WebSocket Protocol**.
Tek bir TCP bağlantısı açılır ve o bağlantı (kopana kadar) açık kalır. Tıpkı bir telefon görüşmesi gibi, iki taraf da istediği an konuşabilir. Bu "Full Duplex" iletişimdir.

![WebSocket vs HTTP](/assets/img/posts/websocket-vs-http-communication.png)
*HTTP'nin kesikli yapısına karşı WebSocket'in sürekli akışı.*

## 1. FastAPI ile Native WebSocket Mimarisi

Python dünyasında Django Channels biraz konfigürasyon yüküyle gelir. FastAPI ise WebSocket konusunda çok daha hafiftir ve "Modern Python" (async/await) ile harika çalışır.

İşte basit ama güçlü bir `ConnectionManager` sınıfı:

```python
from fastapi import FastAPI, WebSocket, WebSocketDisconnect
from typing import List

class ConnectionManager:
    def __init__(self):
        # Aktif soketleri RAM'de tutuyoruz
        self.active_connections: List[WebSocket] = []

    async def connect(self, websocket: WebSocket):
        await websocket.accept()
        self.active_connections.append(websocket)

    def disconnect(self, websocket: WebSocket):
        self.active_connections.remove(websocket)

    async def broadcast(self, message: str):
        # Bağlı olan herkese mesajı ilet (Fan-out)
        for connection in self.active_connections:
            await connection.send_text(message)

manager = ConnectionManager()
app = FastAPI()

@app.websocket("/ws/{client_id}")
async def websocket_endpoint(websocket: WebSocket, client_id: int):
    await manager.connect(websocket)
    try:
        while True:
            # Burası Loop'a girer ve bağlantı kopana kadar devam eder
            data = await websocket.receive_text()
            await manager.broadcast(f"Client #{client_id} dedi ki: {data}")
    except WebSocketDisconnect:
        manager.disconnect(websocket)
```

Bu kodla 100 kişi sohbet edebilir. Peki ya 100.000 kişi?

## 2. Ölçeklenme Sorunu: "Scaling Out"

WebSocket **Stateful** (Durumlu) bir protokoldür. Canlı bağlantıyı hafızada (RAM) tutar.

Senaryo:
*   **Sunucu A:** Ahmet bağlı.
*   **Sunucu B:** Mehmet bağlı.
*   Ahmet mesaj attığında, Sunucu A bunu bilir ama Sunucu B'nin haberi olmaz. Mehmet mesajı göremez.

Load Balancer arkasında birden fazla sunucunuz varsa, sunucular arası haberleşme (Inter-Process Communication) şarttır.

### Redis Pub/Sub ile Mesajlaşma
Redis burada "Message Broker" olarak devreye girer.

1.  Ahmet mesaj atar -> Sunucu A mesajı alır.
2.  Sunucu A, mesajı Redis'in `chat_channel` kanalına **Publish** eder.
3.  Sisteme dahil olan TÜM sunucular (A, B, C...) Redis'in `chat_channel` kanalına **Subscribe** olmuştur.
4.  Sunucu B, Redis'ten mesajı alır ve kendisine bağlı olan Mehmet'e iletir.

![Redis Pub/Sub Architecture](/assets/img/posts/redis-distributed-rate-limiter.png)
*(Görsel temsilidir: Redis, dağıtık nodelar arasındaki sinir sistemidir)*

## 3. Reliability: Bağlantı Koptu Mu?

WebSocket bağlantıları kırılgandır. Kullanıcı tünele girer, Wi-Fi kopar, IP değişir.
Daha kötüsü **Zombi Bağlantılar**dır. İstemci kopmuştur ama TCP FIN paketi sunucuya ulaşmamıştır. Sunucu hala istemcinin bağlı olduğunu sanır ve boşluğa mesaj atmaya çalışır (ve hata alır).

**Çözüm: Heartbeat (Kalp Atışı)**
*   Sunucu her 30 saniyede bir `PING` mesajı atar.
*   İstemci `PONG` ile cevap vermek zorundadır.
*   Cevap gelmezse, sunucu bağlantıyı `force close` yapar ve kaynakları temizler.

## 4. Güvenlik: WebSocket Handshake Auth

WebSocket bağlantısı kurulduktan sonra (Connected), header gönderemezsiniz. Bu yüzden Authentication (Kimlik Doğrulama) işlemi **Handshake** sırasında yapılmalıdır.

```python
async def get_current_user(token: str):
    # Token doğrulama mantığı...
    pass

@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket, token: str = Query(...)):
    # URL'den gelen token'ı doğrula: wss://api.com/ws?token=JWT_TOKEN
    user = await get_current_user(token)
    if not user:
        await websocket.close(code=4001) # Yetkisiz
        return
    
    await manager.connect(websocket)
    # ...
```

> **Senior Notu:** HTTP header'daki `Authorization: Bearer` yapısı WebSocket handshake sırasında tarayıcı JavaScript API'si tarafından desteklenmez (genelde). Bu yüzden token'ı query parametresi (URL) veya ilk mesaj (Message) olarak göndermek yaygın bir pratiktir.

## 5. Load Balancing: Sticky Sessions

Eğer Socket.IO gibi "HTTP Long Polling fallback" kullanan kütüphanelerle çalışıyorsanız, bir kullanıcının handshake süresince AYNI sunucuya gitmesi gerekir.
Nginx ayarlarında `ip_hash` kullanarak **Sticky Session** açmanız gerekir. Yoksa handshake isteği Sunucu 1'e, onay isteği Sunucu 2'ye gider ve bağlantı kurulamaz.

Ancak saf (Native) WebSocket kullanıyorsanız Sticky Session şart değildir, çünkü handshake tek bir HTTP isteğidir ve sonrasında TCP bağlantısı upgrade edilir.

## 6. Sıkça Sorulan Sorular (SSS)

### "WebSocket mi, SSE mi?"
*   **SSE (Server Sent Events):** Sadece sunucudan istemciye (Tek yönlü) veri akar. Bildirimler, haber akışları için mükemmeldir. HTTP tabanlıdır, firewall dostudur.
*   **WebSocket:** İki yönlüdür (Full Duplex). Chat, oyun, borsa al-sat işlemleri için şarttır.

### "Kaç bağlantı kaldırır?"
Modern bir sunucu (örneğin Uvicorn/Node.js) tek çekirdekte 50.000+ "idle" bağlantıyı tutabilir. Darboğaz CPU değil, RAM kullanımı ve Linux'un dosya limitleridir (`ulimit -n`).

## 9. Terimler Sözlüğü (Glossary)

*   **Handshake:** HTTP'den WebSocket protokolüne geçiş (Upgrade) anlaşması.
*   **Full Duplex:** Aynı anda hem veri gönderme hem de alma yeteneği.
*   **Backpressure:** İstemci veriyi işleyemeyecek kadar yavaşsa, sunucunun veri gönderim hızını yavaşlatması mekanizması.

### Client-Side Reconnection (JavaScript)

Sunucunuz ne kadar sağlam olursa olsun, bağlantı kopacaktır. İstemci tarafında (Frontend) mutlaka "Exponential Backoff" ile yeniden bağlanma mantığı kurmalısınız.

```javascript
// Basit bir Reconnection Mantığı
let socket;
function connect() {
  socket = new WebSocket("ws://api.com/ws");
  
  socket.onclose = function(e) {
    console.log('Bağlantı koptu! 3 saniye sonra tekrar deneniyor...');
    setTimeout(function() {
      connect();
    }, 3000); // 3 saniye bekle ve tekrar dene
  };
  
  socket.onerror = function(err) {
    console.error('Socket hatası:', err.message);
    socket.close();
  }
}

connect();
```

Bu kod, bağlantı her koptuğunda sonsuz döngüde tekrar bağlanmaya çalışır. Production'da bekleme süresini katlayarak artırmak (1sn, 2sn, 4sn, 8sn...) sunucuyu korumak için daha iyidir.

## Son Söz

Gerçek zamanlı uygulamalar (Real-Time Apps) kullanıcı deneyimini başka bir seviyeye taşır. "F5'e basıp sayfayı yenileme" devri kapandı. Doğru mimari (Redis + WebSocket + Async Worker) ile milyonlarca kullanıcıya anlık veri akıtabilirsiniz.
