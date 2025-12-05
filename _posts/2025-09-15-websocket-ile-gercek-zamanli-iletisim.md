---
title: "WebSocket ile Gerçek Zamanlı İletişim"
description: "FastAPI ile WebSocket kullanarak gerçek zamanlı uygulamalar geliştirme. Bidirectional iletişim, connection management, broadcasting ve production best practices."
date: 2025-09-15 09:00:00 +0300
categories: [Real-Time, Web Development]
tags: [websocket, python, fastapi, real-time, socket-io, bidirectional]
image:
  path: /assets/img/posts/websocket-vs-http-communication.png
  alt: "WebSocket vs HTTP Communication"
---

Modern web uygulamalarında gerçek zamanlı veri akışı giderek daha kritik hale geliyor. Canlı sohbet uygulamaları, anlık bildirimler, borsa takip sistemleri ve çok kullanıcılı oyunlar gibi senaryolar, HTTP'nin istek-yanıt modelinin ötesinde bir iletişim protokolüne ihtiyaç duyar. İşte tam bu noktada WebSocket devreye girer.

## WebSocket Nedir?

WebSocket, istemci ve sunucu arasında tam çift yönlü (full-duplex), kalıcı bir bağlantı sağlayan bir iletişim protokolüdür. 2011 yılında RFC 6455 standardı ile tanımlanmış olup, HTTP'nin kısıtlamalarını aşarak gerçek zamanlı veri akışını mümkün kılar.

### HTTP vs WebSocket

```
HTTP (İstek-Yanıt):
İstemci → [İstek] → Sunucu
İstemci ← [Yanıt] ← Sunucu
(Her veri alışverişi için yeni bağlantı)

WebSocket (Çift Yönlü):
İstemci ↔ [Kalıcı Bağlantı] ↔ Sunucu
(Her iki yön de istediği zaman mesaj gönderebilir)
```

### WebSocket'in Avantajları

- **Düşük gecikme**: Sürekli bağlantı sayesinde header overhead'i yok
- **Çift yönlü iletişim**: Sunucu, istemciye istek beklemeden mesaj gönderebilir
- **Gerçek zamanlı**: Polling veya long-polling'e gerek yok
- **Kaynak verimliliği**: Tek TCP bağlantısı üzerinden çalışır

## FastAPI ile WebSocket Implementasyonu

FastAPI, WebSocket desteği ile birlikte gelir ve kullanımı oldukça kolaydır. Basit bir WebSocket endpoint'i oluşturalım:

```bash
# Gerekli paketleri yükle
pip install fastapi uvicorn websockets python-multipart
```
{: .nolineno }

### Basit WebSocket Sunucusu

```python
from fastapi import FastAPI, WebSocket, WebSocketDisconnect
from fastapi.responses import HTMLResponse
import asyncio

app = FastAPI(title="WebSocket Demo")

# Basit HTML client
html = """
<!DOCTYPE html>
<html>
<head>
    <title>WebSocket Test</title>
</head>
<body>
    <h1>WebSocket Test Client</h1>
    <form action="" onsubmit="sendMessage(event)">
        <input type="text" id="messageText" autocomplete="off"/>
        <button>Gönder</button>
    </form>
    <ul id='messages'></ul>
    
    <script>
        var ws = new WebSocket("ws://localhost:8000/ws");
        
        ws.onmessage = function(event) {
            var messages = document.getElementById('messages');
            var message = document.createElement('li');
            var content = document.createTextNode(event.data);
            message.appendChild(content);
            messages.appendChild(message);
        };
        
        function sendMessage(event) {
            var input = document.getElementById("messageText");
            ws.send(input.value);
            input.value = '';
            event.preventDefault();
        }
    </script>
</body>
</html>
"""

@app.get("/")
async def get():
    """Test client sayfası"""
    return HTMLResponse(html)

@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    """Temel WebSocket endpoint"""
    # Bağlantıyı kabul et
    await websocket.accept()
    
    try:
        while True:
            # İstemciden mesaj al
            data = await websocket.receive_text()
            
            # Geri gönder (echo)
            await websocket.send_text(f"Mesaj alındı: {data}")
            
    except WebSocketDisconnect:
        print("İstemci bağlantıyı kapattı")
```
{: file="simple_websocket.py" }

```bash
# Sunucuyu başlat
uvicorn simple_websocket:app --reload
```
{: .nolineno }

![WebSocket Real-Time Architecture](/assets/img/posts/websocket-realtime-architecture.png){: w="800" h="500" .shadow }
_WebSocket ile gerçek zamanlı mimari_

## Gelişmiş WebSocket Yönetimi

Gerçek dünya uygulamalarında, birden fazla istemciyi yönetmek, oda konsepti oluşturmak ve hata işlemeyi düzgün yapmak gerekir:

```python
from typing import Dict, List, Set
from fastapi import WebSocket, WebSocketDisconnect
import json
import asyncio
from datetime import datetime

class ConnectionManager:
    """WebSocket bağlantılarını yönetir"""
    
    def __init__(self):
        # Tüm aktif bağlantılar
        self.active_connections: List[WebSocket] = []
        
        # Oda bazlı bağlantılar {room_id: {websocket}}
        self.rooms: Dict[str, Set[WebSocket]] = {}
        
        # Kullanıcı bilgileri {websocket: user_data}
        self.user_data: Dict[WebSocket, dict] = {}
    
    async def connect(self, websocket: WebSocket, user_id: str = None):
        """Yeni bağlantıyı kabul et"""
        await websocket.accept()
        self.active_connections.append(websocket)
        
        # Kullanıcı bilgilerini sakla
        self.user_data[websocket] = {
            "user_id": user_id,
            "connected_at": datetime.now(),
            "rooms": set()
        }
        
        print(f"✅ Yeni bağlantı: {user_id} (Toplam: {len(self.active_connections)})")
    
    def disconnect(self, websocket: WebSocket):
        """Bağlantıyı kapat ve temizle"""
        # Aktif listeden çıkar
        if websocket in self.active_connections:
            self.active_connections.remove(websocket)
        
        # Tüm odalardan çıkar
        user_data = self.user_data.get(websocket, {})
        for room_id in user_data.get("rooms", set()):
            self.leave_room(websocket, room_id)
        
        # Kullanıcı verisini temizle
        if websocket in self.user_data:
            user_id = self.user_data[websocket].get("user_id")
            del self.user_data[websocket]
            print(f"❌ Bağlantı kapandı: {user_id} (Kalan: {len(self.active_connections)})")
    
    async def send_personal_message(self, message: str, websocket: WebSocket):
        """Belirli bir istemciye mesaj gönder"""
        try:
            await websocket.send_text(message)
        except Exception as e:
            print(f"Mesaj gönderilemedi: {e}")
            self.disconnect(websocket)
    
    async def send_json(self, data: dict, websocket: WebSocket):
        """JSON mesaj gönder"""
        try:
            await websocket.send_json(data)
        except Exception as e:
            print(f"JSON gönderilemedi: {e}")
            self.disconnect(websocket)
    
    async def broadcast(self, message: str, exclude: WebSocket = None):
        """Tüm bağlı istemcilere mesaj gönder"""
        disconnected = []
        
        for connection in self.active_connections:
            if connection != exclude:
                try:
                    await connection.send_text(message)
                except:
                    disconnected.append(connection)
        
        # Başarısız bağlantıları temizle
        for conn in disconnected:
            self.disconnect(conn)
    
    async def broadcast_json(self, data: dict, exclude: WebSocket = None):
        """Tüm bağlı istemcilere JSON gönder"""
        disconnected = []
        
        for connection in self.active_connections:
            if connection != exclude:
                try:
                    await connection.send_json(data)
                except:
                    disconnected.append(connection)
        
        for conn in disconnected:
            self.disconnect(conn)
    
    def join_room(self, websocket: WebSocket, room_id: str):
        """Kullanıcıyı odaya ekle"""
        if room_id not in self.rooms:
            self.rooms[room_id] = set()
        
        self.rooms[room_id].add(websocket)
        
        if websocket in self.user_data:
            self.user_data[websocket]["rooms"].add(room_id)
        
        print(f"📥 {self.user_data[websocket].get('user_id')} odaya katıldı: {room_id}")
    
    def leave_room(self, websocket: WebSocket, room_id: str):
        """Kullanıcıyı odadan çıkar"""
        if room_id in self.rooms:
            self.rooms[room_id].discard(websocket)
            
            # Boş oda ise sil
            if not self.rooms[room_id]:
                del self.rooms[room_id]
        
        if websocket in self.user_data:
            self.user_data[websocket]["rooms"].discard(room_id)
        
        print(f"📤 Kullanıcı odadan ayrıldı: {room_id}")
    
    async def broadcast_to_room(self, room_id: str, message: dict, exclude: WebSocket = None):
        """Belirli bir odaya mesaj gönder"""
        if room_id not in self.rooms:
            return
        
        disconnected = []
        
        for connection in self.rooms[room_id]:
            if connection != exclude:
                try:
                    await connection.send_json(message)
                except:
                    disconnected.append(connection)
        
        # Başarısız bağlantıları temizle
        for conn in disconnected:
            self.disconnect(conn)
    
    def get_room_members(self, room_id: str) -> List[dict]:
        """Oda üyelerini getir"""
        if room_id not in self.rooms:
            return []
        
        members = []
        for ws in self.rooms[room_id]:
            if ws in self.user_data:
                members.append(self.user_data[ws])
        
        return members
    
    def get_stats(self) -> dict:
        """İstatistikleri getir"""
        return {
            "total_connections": len(self.active_connections),
            "total_rooms": len(self.rooms),
            "rooms": {
                room_id: len(members) 
                for room_id, members in self.rooms.items()
            }
        }

# Singleton instance
manager = ConnectionManager()
```
{: file="connection_manager.py" }

### Sohbet Uygulaması

```python
from fastapi import FastAPI, WebSocket, WebSocketDisconnect, Query
from connection_manager import manager
from datetime import datetime
from typing import Optional
import json

app = FastAPI(title="Real-Time Chat")

@app.websocket("/ws/chat/{room_id}")
async def chat_endpoint(
    websocket: WebSocket, 
    room_id: str,
    user_id: str = Query(...),
    username: str = Query(...)
):
    """Sohbet odası WebSocket endpoint"""
    
    # Bağlantıyı kabul et ve kullanıcıyı odaya ekle
    await manager.connect(websocket, user_id)
    manager.join_room(websocket, room_id)
    
    # Odaya katılma mesajı gönder
    join_message = {
        "type": "user_joined",
        "user_id": user_id,
        "username": username,
        "room_id": room_id,
        "timestamp": datetime.now().isoformat(),
        "message": f"{username} sohbete katıldı"
    }
    await manager.broadcast_to_room(room_id, join_message)
    
    # Oda üyelerini gönder
    members = manager.get_room_members(room_id)
    await manager.send_json({
        "type": "room_members",
        "members": [m.get("user_id") for m in members],
        "count": len(members)
    }, websocket)
    
    try:
        while True:
            # Mesaj al
            data = await websocket.receive_text()
            
            # Mesajı parse et
            try:
                message_data = json.loads(data)
            except json.JSONDecodeError:
                # Düz metin mesaj
                message_data = {"content": data}
            
            # Mesaj tipine göre işle
            msg_type = message_data.get("type", "message")
            
            if msg_type == "message":
                # Sohbet mesajı
                chat_message = {
                    "type": "message",
                    "user_id": user_id,
                    "username": username,
                    "content": message_data.get("content", ""),
                    "timestamp": datetime.now().isoformat(),
                    "room_id": room_id
                }
                await manager.broadcast_to_room(room_id, chat_message)
            
            elif msg_type == "typing":
                # Yazıyor bildirimi
                typing_notification = {
                    "type": "typing",
                    "user_id": user_id,
                    "username": username,
                    "room_id": room_id
                }
                await manager.broadcast_to_room(
                    room_id, 
                    typing_notification, 
                    exclude=websocket
                )
            
            elif msg_type == "ping":
                # Heartbeat
                await manager.send_json({"type": "pong"}, websocket)
    
    except WebSocketDisconnect:
        # Bağlantı kapatıldı
        manager.leave_room(websocket, room_id)
        manager.disconnect(websocket)
        
        # Ayrılma mesajı gönder
        leave_message = {
            "type": "user_left",
            "user_id": user_id,
            "username": username,
            "room_id": room_id,
            "timestamp": datetime.now().isoformat(),
            "message": f"{username} sohbetten ayrıldı"
        }
        await manager.broadcast_to_room(room_id, leave_message)

@app.get("/api/rooms/{room_id}/members")
async def get_room_members(room_id: str):
    """Oda üyelerini getir"""
    members = manager.get_room_members(room_id)
    return {
        "room_id": room_id,
        "members": members,
        "count": len(members)
    }

@app.get("/api/stats")
async def get_stats():
    """Sunucu istatistikleri"""
    return manager.get_stats()
```
{: file="chat_app.py" }

![FastAPI WebSocket Implementation](/assets/img/posts/fastapi-websocket-implementation.png){: w="700" h="400" .shadow }
_FastAPI WebSocket implementasyonu_

## WebSocket Client Implementasyonu

### Python Client

```python
import asyncio
import websockets
import json
from datetime import datetime

class WebSocketClient:
    """WebSocket client sınıfı"""
    
    def __init__(self, url: str, user_id: str, username: str):
        self.url = url
        self.user_id = user_id
        self.username = username
        self.websocket = None
        self.running = False
    
    async def connect(self):
        """Sunucuya bağlan"""
        try:
            self.websocket = await websockets.connect(
                f"{self.url}?user_id={self.user_id}&username={self.username}"
            )
            self.running = True
            print(f"✅ Bağlandı: {self.url}")
        except Exception as e:
            print(f"❌ Bağlantı hatası: {e}")
            raise
    
    async def disconnect(self):
        """Bağlantıyı kapat"""
        self.running = False
        if self.websocket:
            await self.websocket.close()
            print("🔌 Bağlantı kapatıldı")
    
    async def send_message(self, content: str):
        """Mesaj gönder"""
        if not self.websocket:
            return
        
        message = {
            "type": "message",
            "content": content,
            "timestamp": datetime.now().isoformat()
        }
        
        await self.websocket.send(json.dumps(message))
    
    async def send_typing(self):
        """Yazıyor bildirimi gönder"""
        if not self.websocket:
            return
        
        await self.websocket.send(json.dumps({"type": "typing"}))
    
    async def receive_messages(self):
        """Mesajları al (sürekli dinle)"""
        try:
            async for message in self.websocket:
                data = json.loads(message)
                await self.handle_message(data)
        except websockets.exceptions.ConnectionClosed:
            print("⚠️ Bağlantı kapatıldı")
            self.running = False
        except Exception as e:
            print(f"❌ Hata: {e}")
            self.running = False
    
    async def handle_message(self, data: dict):
        """Gelen mesajı işle"""
        msg_type = data.get("type")
        
        if msg_type == "message":
            print(f"[{data.get('username')}]: {data.get('content')}")
        
        elif msg_type == "user_joined":
            print(f"➕ {data.get('message')}")
        
        elif msg_type == "user_left":
            print(f"➖ {data.get('message')}")
        
        elif msg_type == "typing":
            print(f"✍️  {data.get('username')} yazıyor...")
        
        elif msg_type == "room_members":
            print(f"👥 Odada {data.get('count')} kişi var")
    
    async def heartbeat(self):
        """Periyodik ping gönder"""
        while self.running:
            try:
                await self.websocket.send(json.dumps({"type": "ping"}))
                await asyncio.sleep(30)  # Her 30 saniyede bir
            except:
                break

async def main():
    """Test client"""
    client = WebSocketClient(
        url="ws://localhost:8000/ws/chat/general",
        user_id="user123",
        username="TestUser"
    )
    
    await client.connect()
    
    # Paralel görevler
    receive_task = asyncio.create_task(client.receive_messages())
    heartbeat_task = asyncio.create_task(client.heartbeat())
    
    # Mesaj gönderme döngüsü
    try:
        while client.running:
            message = await asyncio.to_thread(input, "Mesaj: ")
            
            if message.lower() == "quit":
                break
            
            await client.send_message(message)
    
    finally:
        await client.disconnect()
        receive_task.cancel()
        heartbeat_task.cancel()

if __name__ == "__main__":
    asyncio.run(main())
```
{: file="websocket_client.py" }

### JavaScript Client

```javascript
// websocket_client.js - Tarayıcı için WebSocket client
class ChatClient {
    constructor(roomId, userId, username) {
        this.roomId = roomId;
        this.userId = userId;
        this.username = username;
        this.ws = null;
        this.reconnectAttempts = 0;
        this.maxReconnectAttempts = 5;
    }
    
    connect() {
        const wsUrl = `ws://localhost:8000/ws/chat/${this.roomId}?user_id=${this.userId}&username=${this.username}`;
        
        this.ws = new WebSocket(wsUrl);
        
        this.ws.onopen = () => {
            console.log('✅ Bağlantı kuruldu');
            this.reconnectAttempts = 0;
            this.startHeartbeat();
        };
        
        this.ws.onmessage = (event) => {
            const data = JSON.parse(event.data);
            this.handleMessage(data);
        };
        
        this.ws.onerror = (error) => {
            console.error('❌ WebSocket hatası:', error);
        };
        
        this.ws.onclose = () => {
            console.log('🔌 Bağlantı kapandı');
            this.stopHeartbeat();
            this.attemptReconnect();
        };
    }
    
    disconnect() {
        if (this.ws) {
            this.ws.close();
            this.ws = null;
        }
    }
    
    sendMessage(content) {
        if (!this.ws || this.ws.readyState !== WebSocket.OPEN) {
            console.error('Bağlantı yok!');
            return;
        }
        
        const message = {
            type: 'message',
            content: content,
            timestamp: new Date().toISOString()
        };
        
        this.ws.send(JSON.stringify(message));
    }
    
    sendTyping() {
        if (!this.ws || this.ws.readyState !== WebSocket.OPEN) {
            return;
        }
        
        this.ws.send(JSON.stringify({ type: 'typing' }));
    }
    
    handleMessage(data) {
        switch (data.type) {
            case 'message':
                this.displayMessage(data);
                break;
            
            case 'user_joined':
                this.displayNotification(data.message, 'join');
                break;
            
            case 'user_left':
                this.displayNotification(data.message, 'leave');
                break;
            
            case 'typing':
                this.showTypingIndicator(data.username);
                break;
            
            case 'room_members':
                this.updateMemberCount(data.count);
                break;
            
            case 'pong':
                // Heartbeat yanıtı
                break;
        }
    }
    
    displayMessage(data) {
        const messagesDiv = document.getElementById('messages');
        const messageEl = document.createElement('div');
        messageEl.className = 'message';
        
        const isOwnMessage = data.user_id === this.userId;
        messageEl.classList.add(isOwnMessage ? 'own' : 'other');
        
        messageEl.innerHTML = `
            <strong>${data.username}</strong>
            <span class="time">${new Date(data.timestamp).toLocaleTimeString()}</span>
            <p>${this.escapeHtml(data.content)}</p>
        `;
        
        messagesDiv.appendChild(messageEl);
        messagesDiv.scrollTop = messagesDiv.scrollHeight;
    }
    
    displayNotification(message, type) {
        const messagesDiv = document.getElementById('messages');
        const notifEl = document.createElement('div');
        notifEl.className = `notification ${type}`;
        notifEl.textContent = message;
        messagesDiv.appendChild(notifEl);
    }
    
    showTypingIndicator(username) {
        const indicator = document.getElementById('typing-indicator');
        indicator.textContent = `${username} yazıyor...`;
        indicator.style.display = 'block';
        
        // 3 saniye sonra gizle
        clearTimeout(this.typingTimeout);
        this.typingTimeout = setTimeout(() => {
            indicator.style.display = 'none';
        }, 3000);
    }
    
    updateMemberCount(count) {
        document.getElementById('member-count').textContent = count;
    }
    
    startHeartbeat() {
        this.heartbeatInterval = setInterval(() => {
            if (this.ws && this.ws.readyState === WebSocket.OPEN) {
                this.ws.send(JSON.stringify({ type: 'ping' }));
            }
        }, 30000); // Her 30 saniyede bir
    }
    
    stopHeartbeat() {
        if (this.heartbeatInterval) {
            clearInterval(this.heartbeatInterval);
        }
    }
    
    attemptReconnect() {
        if (this.reconnectAttempts >= this.maxReconnectAttempts) {
            console.error('Maksimum yeniden bağlanma denemesi aşıldı');
            return;
        }
        
        this.reconnectAttempts++;
        const delay = Math.min(1000 * Math.pow(2, this.reconnectAttempts), 30000);
        
        console.log(`🔄 ${delay}ms sonra yeniden bağlanılıyor... (${this.reconnectAttempts}/${this.maxReconnectAttempts})`);
        
        setTimeout(() => {
            this.connect();
        }, delay);
    }
    
    escapeHtml(text) {
        const div = document.createElement('div');
        div.textContent = text;
        return div.innerHTML;
    }
}

// Kullanım
const client = new ChatClient('general', 'user123', 'TestUser');
client.connect();

// Mesaj gönder
document.getElementById('send-btn').onclick = () => {
    const input = document.getElementById('message-input');
    client.sendMessage(input.value);
    input.value = '';
};

// Yazıyor göstergesi
let typingTimer;
document.getElementById('message-input').oninput = () => {
    clearTimeout(typingTimer);
    client.sendTyping();
    typingTimer = setTimeout(() => {}, 1000);
};
```
{: file="websocket_client.js" }

## Performans ve Ölçeklendirme

### Redis ile Dağıtık WebSocket

Birden fazla sunucu instance'ı çalıştırırken Redis Pub/Sub kullanarak mesajları senkronize edin:

```python
import redis.asyncio as redis
import json
from typing import Optional
from connection_manager import ConnectionManager

class RedisConnectionManager(ConnectionManager):
    """Redis ile dağıtık WebSocket yönetimi"""
    
    def __init__(self, redis_url: str = "redis://localhost"):
        super().__init__()
        self.redis_url = redis_url
        self.redis_client: Optional[redis.Redis] = None
        self.pubsub = None
    
    async def connect_redis(self):
        """Redis'e bağlan"""
        self.redis_client = await redis.from_url(
            self.redis_url,
            encoding="utf-8",
            decode_responses=True
        )
        self.pubsub = self.redis_client.pubsub()
        print("✅ Redis bağlantısı kuruldu")
    
    async def disconnect_redis(self):
        """Redis bağlantısını kapat"""
        if self.pubsub:
            await self.pubsub.close()
        if self.redis_client:
            await self.redis_client.close()
    
    async def publish_message(self, channel: str, message: dict):
        """Redis channel'a mesaj yayınla"""
        if not self.redis_client:
            return
        
        await self.redis_client.publish(
            channel,
            json.dumps(message)
        )
    
    async def subscribe_to_room(self, room_id: str):
        """Odanın channel'ına abone ol"""
        channel = f"room:{room_id}"
        await self.pubsub.subscribe(channel)
        
        # Mesaj dinleme görevi başlat
        async for message in self.pubsub.listen():
            if message["type"] == "message":
                data = json.loads(message["data"])
                await self.broadcast_to_room(room_id, data)
    
    async def broadcast_to_room(self, room_id: str, message: dict, exclude=None):
        """Odaya broadcast (local + Redis)"""
        # Önce local bağlantılara gönder
        await super().broadcast_to_room(room_id, message, exclude)
        
        # Sonra diğer sunuculara Redis üzerinden gönder
        if self.redis_client:
            await self.publish_message(f"room:{room_id}", message)
    
    async def store_message(self, room_id: str, message: dict):
        """Mesajı Redis'te sakla (geçmiş için)"""
        if not self.redis_client:
            return
        
        key = f"messages:{room_id}"
        await self.redis_client.lpush(key, json.dumps(message))
        await self.redis_client.ltrim(key, 0, 99)  # Son 100 mesaj
        await self.redis_client.expire(key, 86400)  # 24 saat
    
    async def get_message_history(self, room_id: str, limit: int = 50):
        """Mesaj geçmişini getir"""
        if not self.redis_client:
            return []
        
        key = f"messages:{room_id}"
        messages = await self.redis_client.lrange(key, 0, limit - 1)
        
        return [json.loads(msg) for msg in messages]
```
{: file="redis_manager.py" }

## Best Practices

### 1. Heartbeat/Ping-Pong

> WebSocket bağlantılarının canlı olup olmadığını kontrol etmek için düzenli ping-pong mekanizması kullanın. Timeout süreleri ile ölü bağlantıları tespit edin.
{: .prompt-tip }

```python
# Bağlantının canlı olduğunu kontrol et
async def websocket_with_heartbeat(websocket: WebSocket):
    await websocket.accept()
    
    async def send_ping():
        while True:
            try:
                await websocket.send_json({"type": "ping"})
                await asyncio.sleep(30)
            except:
                break
    
    ping_task = asyncio.create_task(send_ping())
    
    try:
        while True:
            data = await asyncio.wait_for(
                websocket.receive_text(),
                timeout=60.0  # 60 saniye timeout
            )
            # İşle...
    except asyncio.TimeoutError:
        print("Timeout: İstemci yanıt vermiyor")
    finally:
        ping_task.cancel()
```

### 2. Hata İşleme

```python
# Kapsamlı hata işleme
async def robust_websocket(websocket: WebSocket):
    try:
        await websocket.accept()
        
        while True:
            try:
                data = await websocket.receive_text()
                await process_message(data)
            
            except json.JSONDecodeError:
                await websocket.send_json({
                    "error": "Invalid JSON format"
                })
            
            except ValueError as e:
                await websocket.send_json({
                    "error": f"Validation error: {str(e)}"
                })
    
    except WebSocketDisconnect as e:
        print(f"Bağlantı kapatıldı: {e.code}")
    
    except Exception as e:
        print(f"Beklenmeyen hata: {e}")
    
    finally:
        # Temizlik işlemleri
        await cleanup(websocket)
```

### 3. Mesaj Boyutu Limiti

> Büyük mesajları filtreleyerek DoS saldırılarını önleyin. Makul bir mesaj boyutu limiti belirleyin (örn. 64KB).
{: .prompt-warning }

```python
# Büyük mesajları engelle
MAX_MESSAGE_SIZE = 64 * 1024  # 64KB

async def receive_with_limit(websocket: WebSocket):
    data = await websocket.receive_bytes()
    
    if len(data) > MAX_MESSAGE_SIZE:
        await websocket.send_json({
            "error": "Message too large",
            "max_size": MAX_MESSAGE_SIZE
        })
        return None
    
    return data
```

### 4. Rate Limiting

> Rate limiting uygulamadan production'a çıkmayın! İstemci başına mesaj sayısını sınırlayarak sunucu kaynaklarını koruyun.
{: .prompt-danger }

```python
# Hız sınırlama
from collections import defaultdict
from time import time

class RateLimiter:
    def __init__(self, max_messages: int = 10, window: int = 60):
        self.max_messages = max_messages
        self.window = window
        self.messages = defaultdict(list)
    
    def is_allowed(self, client_id: str) -> bool:
        now = time()
        # Eski mesajları temizle
        self.messages[client_id] = [
            ts for ts in self.messages[client_id]
            if now - ts < self.window
        ]
        
        if len(self.messages[client_id]) >= self.max_messages:
            return False
        
        self.messages[client_id].append(now)
        return True

rate_limiter = RateLimiter(max_messages=20, window=60)

async def rate_limited_websocket(websocket: WebSocket, client_id: str):
    await websocket.accept()
    
    while True:
        data = await websocket.receive_text()
        
        if not rate_limiter.is_allowed(client_id):
            await websocket.send_json({
                "error": "Rate limit exceeded",
                "retry_after": 60
            })
            continue
        
        await process_message(data)
```

## Sonuç

WebSocket, modern web uygulamalarında gerçek zamanlı iletişim için vazgeçilmez bir teknoloji haline gelmiştir. Bu yazıda öğrendiklerinizle:

- WebSocket'in HTTP'ye göre avantajlarını anlayabilirsiniz
- FastAPI ile production-ready WebSocket sunucuları kurabilirsiniz
- Bağlantı yönetimi ve oda konseptini implement edebilirsiniz
- Python ve JavaScript client'ları geliştirebilirsiniz
- Redis ile dağıtık sistemler oluşturabilirsiniz
- Best practice'leri uygulayabilirsiniz

WebSocket kullanırken, heartbeat, hata işleme, rate limiting ve güvenlik gibi konulara özellikle dikkat edin. Doğru implementasyon ile ölçeklenebilir ve güvenilir gerçek zamanlı uygulamalar geliştirebilirsiniz.

## Kaynaklar

- [RFC 6455 - The WebSocket Protocol](https://datatracker.ietf.org/doc/html/rfc6455)
- [FastAPI WebSockets Documentation](https://fastapi.tiangolo.com/advanced/websockets/)
- [MDN WebSocket API](https://developer.mozilla.org/en-US/docs/Web/API/WebSocket)
- [websockets Python Library](https://websockets.readthedocs.io/)
- [Socket.IO Documentation](https://socket.io/docs/)
