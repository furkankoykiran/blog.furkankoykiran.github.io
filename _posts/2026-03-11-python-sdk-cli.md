---
title: "Python SDK ve CLI Araçları Geliştirme Deneyimlerim"
description: "Mart ayında açık kaynak projelere yaptığım Python SDK geliştirme, CLI tool creation ve deployment optimizasyonu katkıları."
date: 2026-03-11 18:00:00 +0300
categories: [Open Source, Python, CLI]
tags: [python, sdk, cli, typescript, deployment, openai, coinbase, notebooklm]
image:
  path: /assets/img/2026-03-11/python-sdk-cli/banner.png
  alt: "Python SDK and CLI Development Banner"
---

[Önceki blog yazımda](/posts/security-api-fixes/) güvenlik açıkları ve API geliştirmeleri konularını ele almıştım. Şimdi, yoğunlaştığım Python SDK geliştirme, CLI araçları oluşturma ve deployment optimizasyonu konusuna değineceğim.

![Python SDK Development](/assets/img/2026-03-11/python-sdk-cli/python-sdk-development.png)
*Modern Python SDK geliştirme, type safety ve async/await pattern'leri gerektiriyor.*

---

## 1. Jules SDK: Session Deletion API

### Problemi Keşfetmek

Google Labs'in Jules SDK'sında, AI asistan oturumlarını sonlandırma (terminate) işlevi yoktu. Kullanıcılar, tamamlanan oturumları manuel olarak temizlemek zorunda kalıyordu.

### Çözüm: Session Management

[#184 numaralı PR](https://github.com/google-labs-code/jules-sdk/pull/184) ile session deletion API'si ekledim:

```python
# jules_sdk/core/client.py
class JulesClient:
    async def delete_session(self, session_id: str) -> bool:
        """
        Delete a completed session and free up resources.

        Args:
            session_id: The session to delete

        Returns:
            True if deletion was successful
        """
        response = await self._client.delete(f"/sessions/{session_id}")
        return response.status_code == 204
```

### Teknik Detaylar

1. **Async/Await Pattern**: Modern Python async/await kullanımı
2. **Type Hints**: Full type annotation ile type safety
3. **Error Handling**: Graceful error handling ile resilience
4. **Documentation**: Docstring'ler ile API documentation

### Öğrenilen Dersler

1. **API Design**: Delete operations için `204 No Content` status code kullanımı
2. **Resource Management**: Oturumları temizlemek için cleanup mekanizması
3. **Backward Compatibility**: Yeni API'nin mevcut kodu bozmaması

---

## 2. NotebookLM-Py: Metadata Export

### Problemi Keşfetmek

NotebookLM Python wrapper'ında, notebook metadata'sını export etmek için bir mekanizma yoktu. Kullanıcılar notebook başlığı, yazar, oluşturma tarihi gibi bilgilere erişemiyordu.

### Çözüm: Metadata Property

[#174 numaralı PR](https://github.com/teng-lin/notebooklm-py/pull/174) ile metadata export özelliği ekledim:

```python
# notebooklm/client.py
class Notebook:
    @property
    def metadata(self) -> NotebookMetadata:
        """
        Export notebook metadata including title, author, and creation date.

        Returns:
            NotebookMetadata object with structured metadata
        """
        return NotebookMetadata(
            title=self._data.get("title"),
            author=self._data.get("author"),
            created_at=self._data.get("created_at"),
            updated_at=self._data.get("updated_at")
        )
```

### Teknik Detaylar

1. **Property Decorator**: `@property` kullanımı ile attribute-like access
2. **Data Class**: `NotebookMetadata` data class ile structured data
3. **Lazy Loading**: Metadata sadece istendiğinde yükleniyor
4. **Validation**: Type validation ile data integrity

### Öğrenilen Dersler

1. **Python Properties**: `@property` decorator kullanımı
2. **Data Classes**: Structured data için `@dataclass` kullanımı
3. **API Ergonomics**: Kullanıcı dostu API tasarımı

---

## 3. CLI-Agent: Health Check Command

### Problemi Keşfetmek

CLI-Agent projesinde, birden fazla AI provider (OpenAI, Anthropic, Google, vb.) entegrasyonu vardı, ancak sistem sağlık durumunu kontrol etmek için bir komut yoktu.

### Çözüm: Health Check Command

[#7 numaralı PR](https://github.com/amranu/cli-agent/pull/7) ile `/health` komutu ekledim:

```python
# cli_agent/commands/health.py
@click.command()
@click.option("--provider", "-p", multiple=True, help="Specific providers to check")
def health(provider: tuple[str, ...]):
    """
    Check health status of AI providers.

    Supports 6 providers: openai, anthropic, google, groq, mistral, together
    """
    providers_to_check = provider or get_all_providers()

    click.echo("🏥 Health Check Results:")
    click.echo("-" * 50)

    for prov in providers_to_check:
        status = check_provider_health(prov)
        icon = "✅" if status.healthy else "❌"
        click.echo(f"{icon} {prov}: {status.message}")

        if status.latency_ms:
            click.echo(f"   ⏱️  Latency: {status.latency_ms}ms")
```

![CLI Tools Development](/assets/img/2026-03-11/python-sdk-cli/cli-tools-development.png)
*CLI araçları geliştirirken, user experience ve error handling kritik önem taşıyor.*

### Teknik Detaylar

1. **Click Framework**: Python CLI geliştirme için Click kullanımı
2. **Multiple Providers**: 6 farklı provider için health check
3. **Latency Measurement**: Her provider için response time ölçümü
4. **Visual Feedback**: Emoji ve color coding ile kullanıcı deneyimi

### Issue #6: Feature Request

Ayrıca [#6 numaralı issue](https://github.com/amranu/cli-agent/issues/6) ile bu feature'ı talep etmiştim.

### Öğrenilen Dersler

1. **Click Framework**: `@click.command()` ve `@click.option()` kullanımı
2. **Provider Abstraction**: Birden fazla provider için unified interface
3. **Health Check Pattern**: Timeout ve retry mekanizmaları
4. **UX Design**: CLI output'u için visual feedback importance

---

## 4. Mission Control: Gateway Optional Mode

### Problemi Keşfetmek

Builderz Labs'in Mission Control projesinde, VPS deployment'larında gateway zorunlu olarak gerekiyordu. Bu durum, bazı deployment senaryolarında esneklik sağlamıyordu.

### Çözüm: Optional Gateway

[#271 numaralı PR](https://github.com/builderz-labs/mission-control/pull/271) ile gateway'i optional hale getirdim:

```typescript
// src/deployment/vps.ts
interface VPSDeploymentOptions {
    gateway?: {
        enabled: boolean;
        host?: string;
        port?: number;
    };
}

async function deployToVPS(options: VPSDeploymentOptions) {
    if (options.gateway?.enabled !== false) {
        // Gateway kullan
        await setupGateway(options.gateway);
    }

    // Doğrudan deployment
    await deployApplication(options);
}
```

### Issue #289: Feature Request

[#289 numaralı issue](https://github.com/builderz-labs/mission-control/issues/289) ile bu feature'ı önerdim.

### Teknik Detaylar

1. **Optional Pattern**: `gateway?.enabled !== false` ile default behavior
2. **TypeScript Interface**: Strict typing ile configuration safety
3. **Backward Compatibility**: Mevcut deployment'ları bozmama
4. **Documentation**: README güncellemesi ile usage examples

### Öğrenilen Dersler

1. **Optional Parameters**: TypeScript'te optional chaining kullanımı
2. **Configuration Management**: Deployment configuration best practices
3. **Feature Flags**: Gateway enable/disable için feature flag pattern
4. **VPS Deployment**: Cloud deployment stratejileri

---

## Teknik Derinlik: Python SDK Development

Bu projelerde çalışırken, modern Python SDK geliştirme hakkında derinlemesine bilgi edindim.

### Type Safety

```python
from typing import Optional, List
from dataclasses import dataclass
from datetime import datetime

@dataclass
class NotebookMetadata:
    title: str
    author: Optional[str] = None
    created_at: Optional[datetime] = None
```

### Async/Await

```python
import asyncio
import httpx

async def fetch_data() -> dict:
    async with httpx.AsyncClient() as client:
        response = await client.get("https://api.example.com/data")
        return response.json()
```

### Error Handling

```python
from typing import Union
from httpx import HTTPError

class APIError(Exception):
    pass

async def safe_api_call() -> Union[dict, None]:
    try:
        return await fetch_data()
    except HTTPError as e:
        logger.error(f"API call failed: {e}")
        return None
```

---

## CLI Development Best Practices

CLI araçları geliştirirken öğrendiklerim:

1. **User Experience**: Clear error messages ve helpful output
2. **Command Structure**: Logical command hierarchy (e.g., `cli agent health`)
3. **Color Coding**: ANSI colors veya emoji ile visual feedback
4. **Progress Indicators**: Long-running tasks için progress bars
5. **Shell Completion**: Bash/zsh completion scripts

---

## Deployment Optimizasyonu

Mission Control projesinde öğrendiklerim:

1. **Gateway Pattern**: API gateway'leri ile request routing
2. **VPS Deployment**: Virtual Private Server deployment stratejileri
3. **Configuration Management**: Environment variables ve config files
4. **Health Checks**: Deployment sonrası health check mekanizmaları

---

## Sonuç: Modern Python Development

4 farklı projeye katkıda bulunarak, modern Python development ve CLI araçları geliştirme konusunda önemli bilgiler edindim:

1. **Type Safety**: Type hints ve data classes kullanımı
2. **Async Programming**: Async/await pattern'leri
3. **API Design**: User-friendly API tasarımı
4. **CLI Frameworks**: Click ve Typer gibi CLI framework'leri
5. **Deployment**: VPS deployment ve gateway optimizasyonu

Açık kaynak projelere katkıda bulunmak, sadece kod yazmak değil, aynı zamanda community'ye değer katmak ve yeni teknolojileri öğrenmek için harika bir yol.

![NotebookLM API](/assets/img/2026-03-11/python-sdk-cli/notebooklm-api.png)
*NotebookLM API'si, AI-powered notebook yönetimi için güçlü bir platform.*

Eğer siz de Python SDK veya CLI araçları geliştiriyorsanız, type safety, async programming ve user experience konularına odaklanmanızı öneriyorum. Bu, projelerinizin kalitesini ve kullanılabilirliğini önemli ölçüde artıracaktır.

![Furkan Köykıran](/assets/img/avatar.png)
*Furkan Köykıran - Senior Software Engineer*

---

## Diğer Blog Yazılarım

- [Güvenlik Açıkları ve API Geliştirmeleri: 3 Critical Fix](/posts/security-api-fixes/)
- [MCP Ekosistemi ve AI Platformlarına Katkılarım](/posts/mcp-ai-contributions/)
