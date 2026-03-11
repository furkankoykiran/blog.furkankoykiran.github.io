---
title: "Güvenlik Açıkları ve API Geliştirmeleri: 3 Critical Fix"
description: "Şubat ayında yaptığım güvenlik açığı fixleri, API contract geliştirmeleri ve blockchain SDK optimization katkıları."
date: 2026-03-11 17:00:00 +0300
categories: [Open Source, Security, API]
tags: [security, sql-injection, cypher, blockchain, openai, coinbase, api]
image:
  path: /assets/img/2026-03-11/security-api-fixes/banner.png
  alt: "Security and API Fixes Banner"
---

[Önceki blog yazımda](/posts/mcp-ai-contributions/) MCP ekosistemi ve AI platformlarına yaptığım katkıları anlatmıştım. Şimdi, Şubat ayında yoğunlaştığım güvenlik açıkları ve API geliştirmeleri konusuna değineceğim.

Bu dönemde, 3 kritik security fix ve API improvement yaptım: LightRAG'de Cypher injection açığını fixledim, OpenAI Python SDK'da API contract'ını düzelttim ve Coinbase Agentkit'de blockchain decimals sorununu çözdüm.

![SQL Injection Prevention](/assets/img/2026-03-11/security-api-fixes/sql-injection-prevention.png)
*SQL ve Cypher injection açıkları, modern web uygulamalarının en önemli güvenlik tehditleri arasında.*

---

## 1. LightRAG: Cypher Injection Açığı

### Problemi Keşfetmek

LightRAG projesinde, workspace isimleri kullanıcı inputundan doğrudan Cypher query'lerine ekleniyordu. Bu durum, klasik bir **Cypher injection** güvenlik açığı oluşturuyordu.

{% raw %}
```python
# Önceki durum - GÜVENLİK AÇIĞI!
workspace_name = user_input  # "'; DROP TABLE workspace; --"
query = f"MATCH (w:Workspace {{name: '{workspace_name}'}}) RETURN w"
# Bu query, database'e inject edilebilir!
```
{% endraw %}

### Tehlike Analizi

Cypher injection, SQL injection'ın Neo4j graph database versiyonudur. Attack'ler:

1. **Data Exfiltration**: Tüm workspace verilerini çekebilir
2. **Data Deletion**: Workspace'leri silebilir
3. **Authentication Bypass**: Yetkisiz erişim sağlayabilir
4. **DoS Attack**: Database'i çökertebilir

### Çözüm: Workspace Sanitization

[#2713 numaralı PR](https://github.com/HKUDS/LightRAG/pull/2713) ile workspace sanitization ekledim:

{% raw %}
```python
# lightrag/graph_query.py
import re

def sanitize_workspace_name(workspace_name: str) -> str:
    """
    Sanitize workspace name to prevent Cypher injection.

    Removes:
    - Single quotes (')
    - Semicolons (;)
    - Comments (--)
    - Cypher keywords (MATCH, MERGE, DELETE, etc.)

    Args:
        workspace_name: Raw user input

    Returns:
        Sanitized workspace name safe for Cypher queries
    """
    # Remove dangerous characters
    sanitized = re.sub(r"[';\-]", "", workspace_name)

    # Remove Cypher keywords (case-insensitive)
    cypher_keywords = [
        "MATCH", "MERGE", "DELETE", "DROP", "CREATE",
        "SET", "REMOVE", "RETURN", "WHERE", "WITH"
    ]
    for keyword in cypher_keywords:
        sanitized = re.sub(keyword, "", sanitized, flags=re.IGNORECASE)

    # Validate: only alphanumeric and underscore allowed
    if not re.match(r"^[a-zA-Z0-9_]+$", sanitized):
        raise ValueError(f"Invalid workspace name: {workspace_name}")

    return sanitized

# Güvenli kullanım
safe_workspace = sanitize_workspace_name(user_input)
query = f"MATCH (w:Workspace {{name: '{safe_workspace}'}}) RETURN w"
```
{% endraw %}

### Teknik Detaylar

1. **Input Sanitization**: Regex ile dangerous character removal
2. **Keyword Blacklisting**: Cypher keyword'lerini temizleme
3. **Whitelist Validation**: Sadece alphanumeric + underscore izni
4. **Error Handling**: Invalid input için exception fırlatma

### Öğrenilen Dersler

1. **Never Trust User Input**: Kullanıcı inputu her zaman sanitize edilmeli
2. **Defense in Depth**: Birden fazla güvenlik katmanı (char removal + keyword filtering + whitelist)
3. **Graph Database Security**: Neo4j ve Cypher injection best practices
4. **Security Testing**: Malicious input ile test etme (`'; DROP TABLE--`)

### Security Best Practices

{% raw %}
```python
# ❌ YANLIŞ - Direct interpolation
query = f"MATCH (n {{name: '{user_input}'}}) RETURN n"

# ✅ DOĞRU - Parameterized query (Neo4j driver destekliyorsa)
query = "MATCH (n {name: $name}) RETURN n"
params = {"name": user_input}

# ✅ DOĞRU - Sanitization (parameterized query yoksa)
safe_input = sanitize_input(user_input)
query = f"MATCH (n {{name: '{safe_input}'}}) RETURN n"
```
{% endraw %}

---

## 2. OpenAI Python SDK: API Contract Fix

### Problemi Keşfetmek

OpenAI Python SDK'sında `prompt_cache_retention` parametresi için **hyphenated** naming kullanılıyordu, ancak OpenAI API'si **underscore** naming bekliyordu. Bu durum, API contract uyuşmazlığına neden oluyordu.

```python
# Önceki durum - API CONTRACT MISMATCH!
client.chat.completions.create(
    model="gpt-4",
    prompt_cache-retention="auto"  # ❌ Python'da geçersiz!
)

# API beklediği:
# "prompt_cache_retention": "auto"  # ✅ Underscore
```

### Çözüm: Literal Name Fix

[#2893 numaralı PR](https://github.com/openai/openai-python/pull/2893) ile parameter name'i düzelttim:

```python
# openai/resources/chat/completions.py
class CompletionCreateParams(BaseModel):
    # Önceki: prompt-cache-retention: str  # ❌ Invalid Python identifier
    # Sonrası:
    prompt_cache_retention: Optional[str] = Field(
        default=None,
        alias="prompt_cache_retention"  # JSON serialization için
    )
```

### Teknik Detaylar

1. **Python Identifiers**: Python değişken isimleri hyphen içeremez
2. **API Contract Consistency**: SDK ile API arasında naming consistency
3. **Pydantic Aliases**: `alias` parametresi ile JSON serialization control
4. **Backward Compatibility**: Mevcut kodu bozmama

### Öğrenilen Dersler

1. **API Contract Management**: SDK ile API arasında consistency önemlidir
2. **Python Naming Conventions**: `snake_case` kullanımı, `kebab-case` kullanımı
3. **Pydantic Advanced Features**: `alias` ve serialization control
4. **SDK Design**: API wrapper'larında naming decisions

---

## 3. Coinbase Agentkit: Dynamic Token Decimals

### Problemi Keşfetmek

Coinbase Agentkit'te, Morpho protocol için withdraw işlemi sabit **18 decimal** kullanıyordu. Bu durum, USDC (6 decimal) veya USDT (6 decimal) gibi token'lar için doğru çalışmıyordu.

```python
# Önceki durum - FIXED DECIMALS!
amount_wei = amount * 10**18  # ❌ USDC için yanlış!

# USDC: 6 decimals (1 USDC = 1,000,000 base units)
# USDT: 6 decimals
# WBTC: 8 decimals
# ETH: 18 decimals
```

### Çözüm: Dynamic Decimals

[#965 numaralı PR](https://github.com/coinbase/agentkit/pull/965) ile dynamic decimals ekledim:

```python
# coinbase_agentkit/protocols/morpho.py
from web3 import Web3
from typing import Literal

def get_token_decimals(token_address: str) -> int:
    """
    Fetch token decimals from on-chain ERC20 metadata.

    Args:
        token_address: ERC20 token contract address

    Returns:
        Token decimals (6 for USDC, 18 for ETH, etc.)
    """
    # ERC20 `decimals()` function call
    contract = w3.eth.contract(address=token_address, abi=ERC20_ABI)
    return contract.functions.decimals().call()

def withdraw(amount: float, token_address: str) -> str:
    """
    Withdraw tokens with dynamic decimal support.

    Args:
        amount: Token amount (human-readable)
        token_address: ERC20 token contract address

    Returns:
        Amount in wei (base units)
    """
    decimals = get_token_decimals(token_address)
    amount_wei = int(amount * 10**decimals)

    # Morpho withdraw execution
    morpho.withdraw(token_address, amount_wei)

    return str(amount_wei)
```

![OpenAI Python SDK](/assets/img/2026-03-11/security-api-fixes/openai-python-sdk.png)
*OpenAI Python SDK, modern Python practices ve type safety ile geliştirilmiş bir SDK.*

### Teknik Detaylar

1. **ERC20 Standard**: `decimals()` function ile on-chain metadata
2. **Web3.py**: Ethereum smart contract interaction
3. **Precision Handling**: Floating point → integer conversion
4. **Multi-Token Support**: USDC, USDT, WBTC, ETH, vb.

### Öğrenilen Dersler

1. **ERC20 Token Standard**: Her token'ın kendi decimals değeri vardır
2. **Blockchain Development**: Smart contract interaction patterns
3. **Numerical Precision**: Floating point vs integer arithmetic
4. **DeFi Protocols**: Morpho, Aave, Compound gibi lending protokolleri

---

## Blockchain ve DeFi Development

Coinbase Agentkit üzerinde çalışırken öğrendiklerim:

### ERC20 Token Standard

```solidity
// ERC20.sol (Smart Contract)
function decimals() public view returns (uint8) {
    return 6; // USDC için 6, USDT için 6
}
```

### Web3.py Integration

```python
from web3 import Web3

# Contract interaction
contract = w3.eth.contract(
    address=token_address,
    abi=ERC20_ABI
)

# On-chain data fetch
decimals = contract.functions.decimals().call()
symbol = contract.functions.symbol().call()
```

---

## API Contract Management

OpenAI SDK üzerinde çalışırken öğrendiklerim:

### API Versioning

```python
# API version kontrolü
if api_version < "2024-01-01":
    raise ValueError("Please upgrade to latest API version")

# Backward compatibility için deprecated warning
import warnings
warnings.warn(
    "prompt_cache_retention is deprecated, use prompt_cache instead",
    DeprecationWarning
)
```

---

## [Coinbase Blockchain](https://www.coinbase.com/)

![Coinbase Blockchain](/assets/img/2026-03-11/security-api-fixes/coinbase-blockchain.png)
*Coinbase, cryptocurrency trading ve blockchain development için popüler bir platform.*

Coinbase Agentkit, DeFi protocol entegrasyonları için güçlü bir SDK. Bu katkım sırasında öğrendiklerim:

1. **DeFi Lending Protocols**: Morpho, Aave, Compound çalışma prensipleri
2. **Smart Contract Interaction**: Web3.py ile ERC20 token işlemleri
3. **Token Decimals**: Her token'ın farklı decimal değeri vardır
4. **Gas Optimization**: Transaction gas maliyetlerini azaltma

---

## Gateway VPS Deployment

Mission Control projesindeki çalışmam sırasında öğrendim:

1. **VPS Providers**: DigitalOcean, Linode, AWS Lightsail
2. **Gateway Pattern**: API gateway'leri ile request routing
3. **Deployment Strategies**: Blue-green deployment, canary deployment
4. **Monitoring**: Health check'ler ve uptime monitoring

---

## Sonuç: Security ve API Best Practices

Şubat ayında 3 kritik security fix ve API improvement yaptım. Öğrendiklerim:

1. **Input Sanitization**: Kullanıcı inputu her zaman sanitize edilmeli
2. **API Contract Consistency**: SDK ile API arasında naming consistency
3. **Blockchain Development**: ERC20 standard ve Web3.py integration
4. **DeFi Protocols**: Lending protocol'leri ve token decimals

Güvenlik açıkları ve API contract uyuşmazlıkları, production sistemlerde ciddi sorunlara yol açabilir. Open source projelere katkıda bulunarak, bu sorunları erken tespit etmek ve düzeltmek community için değerlidir.

![Gateway VPS Deployment](/assets/img/2026-03-11/security-api-fixes/gateway-vps-deployment.png)
*VPS deployment'larında gateway pattern, esneklik ve scalability sağlar.*

![Furkan Köykıran](/assets/img/avatar.png)
*Furkan Köykıran - Senior Software Engineer*

---

## Diğer Blog Yazılarım

- [Python SDK ve CLI Araçları Geliştirme Deneyimlerim](/posts/python-sdk-cli/)
- [MCP Ekosistemi ve AI Platformlarına Katkılarım](/posts/mcp-ai-contributions/)
