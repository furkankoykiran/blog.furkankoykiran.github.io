---
title: "Blockchain Veri Analizi için Python Araçları"
date: 2024-03-07 09:15:00 +0300
categories: [Blockchain, Analytics]
tags: [python, blockchain, web3py, thegraph, dune-analytics, veri-analizi, on-chain]
image:
  path: /assets/img/posts/blockchain-data-analytics-architecture.png
  alt: "Blockchain veri analizi mimarisi ve Python araçları"
---

Blockchain teknolojisinin gelişmesiyle birlikte, on-chain verilerin analizi modern kripto ve DeFi projelerinin vazgeçilmez bir parçası haline geldi. Ethereum, Polygon ve diğer blokzincirlerdeki işlemleri, akıllı sözleşme olaylarını ve cüzdan hareketlerini analiz etmek için Python ekosisteminde güçlü araçlar mevcut. Bu yazıda Web3.py, TheGraph ve Dune Analytics kullanarak profesyonel blockchain veri analizi yapmanın yollarını inceleyeceğiz.

## Blockchain Veri Analizinin Önemi

On-chain veri analizi, blockchain üzerinde gerçekleşen tüm işlemlerin şeffaf ve değiştirilemez doğasından faydalanarak değerli içgörüler elde etmeyi sağlar. Token transferleri, likidite havuzu hareketleri, NFT satışları ve whale cüzdan aktiviteleri gibi metrikleri izleyerek piyasa trendlerini öngörebilir ve stratejik kararlar alabilirsiniz.

### Kullanım Alanları

Blockchain veri analizi şu alanlarda kritik öneme sahiptir:

- **DeFi Protokol Analizi**: Uniswap, Aave gibi protokollerde TVL, volume ve kullanıcı aktivitesi takibi
- **Token Ekonomi İzleme**: Token dağılımı, holder davranışları ve transfer pattern analizi
- **Whale Tracking**: Büyük cüzdanların hareketlerini takip ederek piyasa sinyalleri yakalama
- **Smart Contract Monitoring**: Kritik olayları gerçek zamanlı izleme ve alarm sistemleri
- **MEV Analizi**: Arbitraj, sandwich attack ve diğer MEV stratejilerinin tespiti

## Web3.py ile Temel On-Chain Veri Okuma

Web3.py, Ethereum ve EVM uyumlu blokzincirlerle etkileşim kurmak için en popüler Python kütüphanesidir. Düşük seviyeli RPC çağrıları yapmak ve akıllı sözleşme olaylarını okumak için temel araçtır.

### Kurulum ve Bağlantı

```python
# Gerekli kütüphanelerin kurulumu
pip install web3 python-dotenv

# Web3.py ile Ethereum node bağlantısı
from web3 import Web3
import os
from dotenv import load_dotenv

load_dotenv()

# Infura veya Alchemy gibi bir RPC endpoint kullanın
RPC_URL = os.getenv("ETHEREUM_RPC_URL")
w3 = Web3(Web3.HTTPProvider(RPC_URL))

# Bağlantı kontrolü
if w3.is_connected():
    print(f"✅ Ethereum ağına bağlanıldı")
    print(f"📊 Son blok numarası: {w3.eth.block_number}")
    print(f"⛽ Gas fiyatı: {w3.eth.gas_price / 10**9} Gwei")
else:
    print("❌ Bağlantı başarısız")
```

### Transaction Data Parsing

Blok verisini okuyup işlemleri analiz etmek, blockchain analizinin temelidir:

```python
from datetime import datetime
from typing import Dict, List

def get_block_transactions(block_number: int) -> List[Dict]:
    """
    Belirtilen bloktaki tüm işlemleri detaylı şekilde getirir
    """
    block = w3.eth.get_block(block_number, full_transactions=True)
    
    transactions = []
    for tx in block.transactions:
        # Transaction detaylarını parse et
        tx_data = {
            'hash': tx['hash'].hex(),
            'from': tx['from'],
            'to': tx['to'],
            'value': w3.from_wei(tx['value'], 'ether'),  # Wei'den ETH'ye çevir
            'gas': tx['gas'],
            'gas_price': w3.from_wei(tx['gasPrice'], 'gwei'),
            'nonce': tx['nonce'],
            'block': block_number,
            'timestamp': datetime.fromtimestamp(block.timestamp)
        }
        
        # Input data varsa (smart contract call) işaretle
        if tx['input'] != '0x':
            tx_data['is_contract_call'] = True
            tx_data['input_length'] = len(tx['input'])
        
        transactions.append(tx_data)
    
    return transactions

# Son 10 bloktaki işlemleri analiz et
latest_block = w3.eth.block_number
for i in range(latest_block - 10, latest_block):
    txs = get_block_transactions(i)
    print(f"Blok {i}: {len(txs)} işlem")
    
    # ETH transfer hacmini hesapla
    total_volume = sum(tx['value'] for tx in txs)
    print(f"  💰 Toplam ETH: {total_volume:.4f}")
```

### Smart Contract Event Monitoring

Akıllı sözleşmelerin emit ettiği event'leri izlemek, DeFi protokollerini analiz etmenin anahtarıdır:

```python
# Uniswap V2 Pair contract ABI (sadece Swap event için)
UNISWAP_PAIR_ABI = [
    {
        "anonymous": False,
        "inputs": [
            {"indexed": True, "name": "sender", "type": "address"},
            {"indexed": False, "name": "amount0In", "type": "uint256"},
            {"indexed": False, "name": "amount1In", "type": "uint256"},
            {"indexed": False, "name": "amount0Out", "type": "uint256"},
            {"indexed": False, "name": "amount1Out", "type": "uint256"},
            {"indexed": True, "name": "to", "type": "address"}
        ],
        "name": "Swap",
        "type": "event"
    }
]

def track_uniswap_swaps(pair_address: str, from_block: int, to_block: int):
    """
    Uniswap pair'inde gerçekleşen swap işlemlerini takip eder
    """
    # Contract instance oluştur
    pair_contract = w3.eth.contract(
        address=Web3.to_checksum_address(pair_address),
        abi=UNISWAP_PAIR_ABI
    )
    
    # Event filter oluştur
    swap_filter = pair_contract.events.Swap.create_filter(
        fromBlock=from_block,
        toBlock=to_block
    )
    
    # Event'leri al ve işle
    swaps = swap_filter.get_all_entries()
    
    print(f"🔄 {len(swaps)} swap işlemi bulundu")
    
    for swap in swaps:
        # Event parametrelerini parse et
        args = swap.args
        tx_hash = swap.transactionHash.hex()
        block = swap.blockNumber
        
        print(f"\n📦 Blok: {block}")
        print(f"🔗 TX: {tx_hash}")
        print(f"👤 Gönderen: {args.sender}")
        print(f"📊 Amount0 In: {args.amount0In}")
        print(f"📊 Amount1 In: {args.amount1In}")
        print(f"📊 Amount0 Out: {args.amount0Out}")
        print(f"📊 Amount1 Out: {args.amount1Out}")
    
    return swaps

# Örnek: USDC/ETH pair'ini izle
USDC_ETH_PAIR = "0xB4e16d0168e52d35CaCD2c6185b44281Ec28C9Dc"
latest = w3.eth.block_number
swaps = track_uniswap_swaps(USDC_ETH_PAIR, latest - 100, latest)
```

![TheGraph Protocol Mimarisi](/assets/img/posts/thegraph-protocol-architecture.png)
_TheGraph protokolü ile blockchain verilerinin indekslenmesi ve sorgulanması_

## TheGraph ile Gelişmiş Veri Sorgulama

Web3.py ile doğrudan RPC çağrıları yapmak performans açısından sınırlıdır. TheGraph, blockchain verilerini indeksleyerek GraphQL ile hızlı sorgulamaya olanak tanır.

### TheGraph Nedir?

TheGraph, blockchain verilerini indeksleyen ve organize eden merkeziyetsiz bir protokoldür. Subgraph'ler, belirli smart contract'ların event'lerini dinleyip veritabanı benzeri bir yapıda saklar. Bu sayede karmaşık sorgular saniyeler içinde çalıştırılabilir.

### Python ile GraphQL Sorguları

```python
# GraphQL client kurulumu
pip install gql[all]

from gql import gql, Client
from gql.transport.aiohttp import AIOHTTPTransport

# Uniswap V3 subgraph endpoint
UNISWAP_V3_SUBGRAPH = "https://api.thegraph.com/subgraphs/name/uniswap/uniswap-v3"

# GraphQL transport ve client oluştur
transport = AIOHTTPTransport(url=UNISWAP_V3_SUBGRAPH)
client = Client(transport=transport, fetch_schema_from_transport=True)

def get_top_pools_by_tvl(limit: int = 10):
    """
    TVL'ye göre en büyük Uniswap V3 pool'larını getirir
    """
    query = gql(f"""
        {% raw %}{{
            pools(
                first: {limit},
                orderBy: totalValueLockedUSD,
                orderDirection: desc
            ) {{
                id
                token0 {{
                    symbol
                    decimals
                }}
                token1 {{
                    symbol
                    decimals
                }}
                totalValueLockedUSD
                volumeUSD
                feeTier
                txCount
            }}
        }}{% endraw %}
    """)
    
    result = client.execute(query)
    return result['pools']

# Top 10 pool'u listele
pools = get_top_pools_by_tvl(10)

print("🏊 Uniswap V3 En Büyük Pool'lar (TVL)")
print("=" * 70)

for i, pool in enumerate(pools, 1):
    symbol_pair = f"{pool['token0']['symbol']}/{pool['token1']['symbol']}"
    tvl = float(pool['totalValueLockedUSD'])
    volume = float(pool['volumeUSD'])
    fee = int(pool['feeTier']) / 10000  # Fee tier'i yüzdeye çevir
    
    print(f"\n{i}. {symbol_pair}")
    print(f"   💰 TVL: ${tvl:,.2f}")
    print(f"   📊 Volume: ${volume:,.2f}")
    print(f"   💸 Fee: {fee}%")
    print(f"   🔢 TX Count: {pool['txCount']}")
```

### Token Holder Analizi

TheGraph ile token holder'ları ve transfer pattern'lerini analiz edebilirsiniz:

```python
def analyze_token_holders(token_address: str, min_balance: float = 1000):
    """
    Belirtilen token için holder analizini yapar
    Not: Token'a özel subgraph gerekir
    """
    query = gql(f"""
        {% raw %}{{
            tokenHolders(
                where: {{
                    token: "{token_address.lower()}",
                    balance_gt: "{int(min_balance * 10**18)}"
                }},
                first: 100,
                orderBy: balance,
                orderDirection: desc
            ) {{
                id
                address
                balance
                transactionCount
            }}
        }}{% endraw %}
    """)
    
    result = client.execute(query)
    holders = result['tokenHolders']
    
    # Holder istatistikleri
    total_holders = len(holders)
    total_balance = sum(int(h['balance']) / 10**18 for h in holders)
    
    print(f"👥 Top {total_holders} Holder Analizi")
    print(f"💰 Toplam Balance: {total_balance:,.2f} token")
    
    # Whale'leri tespit et (top 10 holder)
    whales = holders[:10]
    whale_balance = sum(int(h['balance']) / 10**18 for h in whales)
    whale_percentage = (whale_balance / total_balance) * 100
    
    print(f"\n🐋 Whale Analizi (Top 10)")
    print(f"   Balance: {whale_balance:,.2f} token")
    print(f"   Oran: {whale_percentage:.2f}%")
    
    return holders

# Örnek kullanım (kendi subgraph endpoint'inizi kullanın)
# holders = analyze_token_holders("0x...", min_balance=10000)
```

![TheGraph Subgraph Workflow](/assets/img/posts/thegraph-subgraph-query-workflow.png)
_TheGraph subgraph oluşturma ve veri sorgulama iş akışı_

### Time-Series Veri Analizi

TheGraph'in gücü, zaman serisi verilerini kolayca sorgulamaktır:

```python
from datetime import datetime, timedelta
import pandas as pd

def get_pool_daily_volume(pool_id: str, days: int = 30):
    """
    Belirtilen pool için günlük volume verilerini getirir
    """
    # Unix timestamp hesapla (30 gün öncesi)
    start_timestamp = int((datetime.now() - timedelta(days=days)).timestamp())
    
    query = gql(f"""
        {% raw %}{{
            poolDayDatas(
                where: {{
                    pool: "{pool_id}",
                    date_gt: {start_timestamp}
                }},
                orderBy: date,
                orderDirection: asc
            ) {{
                date
                volumeUSD
                tvlUSD
                feesUSD
                txCount
            }}
        }}{% endraw %}
    """)
    
    result = client.execute(query)
    daily_data = result['poolDayDatas']
    
    # Pandas DataFrame'e çevir
    df = pd.DataFrame(daily_data)
    df['date'] = pd.to_datetime(df['date'], unit='s')
    df['volumeUSD'] = df['volumeUSD'].astype(float)
    df['tvlUSD'] = df['tvlUSD'].astype(float)
    df['feesUSD'] = df['feesUSD'].astype(float)
    
    return df

# Veri analizi ve görselleştirme
# pool_data = get_pool_daily_volume("0xpool_address")
# print(pool_data.describe())
```

## Dune Analytics ile SQL Tabanlı Analiz

Dune Analytics, blockchain verilerini SQL ile sorgulamanıza olanak tanıyan güçlü bir platformdur. Python ile Dune API'sini kullanarak sorgu sonuçlarını alabilir ve kendi analizlerinizi yapabilirsiniz.

![Dune Analytics Dashboard](/assets/img/posts/dune-analytics-dashboard-visualization.png)
_Dune Analytics ile DEX metriklerinin görselleştirilmesi_

### Dune API Kurulumu

```python
# Dune client kurulumu
pip install dune-client

import os
from dune_client.client import DuneClient
from dune_client.query import QueryBase

# API key ile client oluştur (.env dosyasından)
DUNE_API_KEY = os.getenv("DUNE_API_KEY")
dune = DuneClient(DUNE_API_KEY)

# Mevcut bir query'yi çalıştır
def run_dune_query(query_id: int, params: dict = None):
    """
    Dune Analytics query'sini çalıştırır ve sonuçları getirir
    """
    try:
        # Query çalıştır
        query = QueryBase(query_id=query_id, params=params or {})
        results = dune.run_query(query)
        
        return results.result.rows
    except Exception as e:
        print(f"❌ Query hatası: {e}")
        return None

# Örnek: ETH Daily Transactions
query_id = 3236296  # Örnek query ID
results = run_dune_query(query_id)

if results:
    print(f"✅ {len(results)} satır veri alındı")
    for row in results[:5]:  # İlk 5 satır
        print(row)
```

### Custom Dashboard Oluşturma

Dune'dan aldığınız verileri Python ile analiz edip dashboard oluşturabilirsiniz:

```python
import pandas as pd
import plotly.express as px
from datetime import datetime, timedelta

def analyze_dex_volume_comparison():
    """
    Farklı DEX'lerin volume karşılaştırmasını yapar
    """
    # Dune query: DEX günlük volume (önceden oluşturulmuş query)
    query_id = 123456  # Kendi query ID'nizi kullanın
    
    # Son 30 günün verilerini al
    results = run_dune_query(query_id, params={
        "days": 30
    })
    
    if not results:
        return
    
    # DataFrame'e çevir
    df = pd.DataFrame(results)
    df['date'] = pd.to_datetime(df['date'])
    df['volume'] = df['volume'].astype(float)
    
    # DEX'lere göre grupla
    dex_totals = df.groupby('dex_name')['volume'].sum().sort_values(ascending=False)
    
    print("📊 DEX Volume Karşılaştırması (30 Gün)")
    print("=" * 50)
    
    for dex, volume in dex_totals.items():
        print(f"{dex:15} ${volume:,.0f}")
    
    # Görselleştirme (plotly)
    fig = px.line(
        df,
        x='date',
        y='volume',
        color='dex_name',
        title='DEX Volume Trend (30 Gün)',
        labels={'volume': 'Volume (USD)', 'date': 'Tarih'}
    )
    
    fig.write_html('dex_volume_analysis.html')
    print("\n📈 Grafik kaydedildi: dex_volume_analysis.html")

# Analizi çalıştır
# analyze_dex_volume_comparison()
```

### Wallet Activity Tracking

Dune ile belirli cüzdanların aktivitelerini takip edebilirsiniz:

```python
def track_whale_wallets(wallet_addresses: list, min_value_usd: float = 100000):
    """
    Büyük cüzdanların aktivitelerini izler
    """
    # Wallet'ları query parametresi olarak gönder
    wallet_list = "', '".join(wallet_addresses)
    
    # Custom query (Dune'da önceden oluşturulmalı)
    query_id = 789012  # Whale tracking query ID
    
    results = run_dune_query(query_id, params={
        "wallet_list": wallet_list,
        "min_value": min_value_usd
    })
    
    if not results:
        return
    
    print(f"🐋 Whale Aktivite Raporu")
    print(f"📅 Son 24 saat")
    print("=" * 80)
    
    for tx in results:
        print(f"\n⏰ {tx['timestamp']}")
        print(f"👤 Cüzdan: {tx['wallet_address'][:10]}...{tx['wallet_address'][-8:]}")
        print(f"🔄 İşlem: {tx['action']}")
        print(f"💰 Değer: ${tx['value_usd']:,.2f}")
        print(f"🪙 Token: {tx['token_symbol']}")

# Örnek whale adresleri
whale_addresses = [
    "0x...",  # Binance wallet
    "0x...",  # Vitalik wallet
]

# track_whale_wallets(whale_addresses, min_value_usd=500000)
```

## Real-Time Event Monitoring Sistemi

Tüm bu araçları bir araya getirerek gerçek zamanlı monitoring sistemi oluşturabilirsiniz:

```python
import asyncio
from web3 import Web3
from datetime import datetime
import json

class BlockchainMonitor:
    """
    Blockchain event'lerini gerçek zamanlı izleyen sınıf
    """
    def __init__(self, rpc_url: str, contracts: dict):
        self.w3 = Web3(Web3.WebsocketProvider(rpc_url))  # WebSocket kullan
        self.contracts = contracts
        self.is_running = False
    
    async def monitor_new_blocks(self):
        """
        Yeni blokları sürekli izler
        """
        print("🔍 Blok monitoring başlatıldı...")
        
        latest_block = self.w3.eth.block_number
        
        while self.is_running:
            try:
                current_block = self.w3.eth.block_number
                
                # Yeni blok varsa
                if current_block > latest_block:
                    for block_num in range(latest_block + 1, current_block + 1):
                        await self.process_block(block_num)
                    
                    latest_block = current_block
                
                await asyncio.sleep(2)  # 2 saniye bekle
                
            except Exception as e:
                print(f"❌ Hata: {e}")
                await asyncio.sleep(5)
    
    async def process_block(self, block_number: int):
        """
        Bloktaki işlemleri analiz eder
        """
        block = self.w3.eth.get_block(block_number, full_transactions=True)
        
        print(f"\n📦 Yeni Blok: {block_number}")
        print(f"⏰ Zaman: {datetime.fromtimestamp(block.timestamp)}")
        print(f"🔢 İşlem Sayısı: {len(block.transactions)}")
        
        # Her contract için event'leri kontrol et
        for contract_name, contract_info in self.contracts.items():
            await self.check_contract_events(
                contract_name,
                contract_info,
                block_number
            )
    
    async def check_contract_events(self, name: str, info: dict, block: int):
        """
        Belirli contract'ın event'lerini kontrol eder
        """
        contract = self.w3.eth.contract(
            address=info['address'],
            abi=info['abi']
        )
        
        # Her event type için filter oluştur
        for event_name in info['events']:
            event = getattr(contract.events, event_name)
            
            event_filter = event.create_filter(
                fromBlock=block,
                toBlock=block
            )
            
            entries = event_filter.get_all_entries()
            
            if entries:
                print(f"\n🚨 {name} - {event_name}: {len(entries)} event")
                
                for entry in entries:
                    await self.handle_event(name, event_name, entry)
    
    async def handle_event(self, contract: str, event: str, data):
        """
        Event'i işler ve gerekli aksiyonları alır
        """
        # Event tipine göre özel işlemler
        if event == "Swap" and contract == "UniswapV2":
            # Büyük swap'leri logla
            amount0 = data.args.amount0Out
            if amount0 > 10**20:  # 100+ token
                print(f"   🔥 BÜYÜK SWAP TESPİT EDİLDİ!")
                print(f"   TX: {data.transactionHash.hex()}")
                
                # Alert gönder (Telegram, Discord, vb.)
                await self.send_alert(f"Büyük swap: {data.transactionHash.hex()}")
    
    async def send_alert(self, message: str):
        """
        Alert gönderir (Telegram, Discord, email, vb.)
        """
        # Buraya kendi alert sisteminizi entegre edin
        print(f"📬 Alert: {message}")
    
    def start(self):
        """
        Monitoring'i başlatır
        """
        self.is_running = True
        asyncio.run(self.monitor_new_blocks())
    
    def stop(self):
        """
        Monitoring'i durdurur
        """
        self.is_running = False

# Kullanım örneği
"""
contracts_to_monitor = {
    "UniswapV2": {
        "address": "0xB4e16d0168e52d35CaCD2c6185b44281Ec28C9Dc",
        "abi": UNISWAP_PAIR_ABI,
        "events": ["Swap", "Mint", "Burn"]
    }
}

monitor = BlockchainMonitor(
    rpc_url="wss://mainnet.infura.io/ws/v3/YOUR_KEY",
    contracts=contracts_to_monitor
)

# Monitoring'i başlat
# monitor.start()
"""
```

## Best Practices ve Optimizasyon

Blockchain veri analizi yaparken dikkate almanız gereken önemli noktalar:

### 1. RPC Rate Limiting

Public RPC endpoint'leri rate limit'e sahiptir. Production'da mutlaka:
- Paid RPC servisi kullanın (Infura, Alchemy, QuickNode)
- Request cache mekanizması ekleyin
- Exponential backoff retry logic implementasyonu yapın

```python
import time
from functools import wraps

def rate_limited(max_per_second: int = 5):
    """
    RPC çağrılarını rate limit'e uygun şekilde yavaşlatır
    """
    min_interval = 1.0 / max_per_second
    last_called = [0.0]
    
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            elapsed = time.time() - last_called[0]
            wait_time = min_interval - elapsed
            
            if wait_time > 0:
                time.sleep(wait_time)
            
            result = func(*args, **kwargs)
            last_called[0] = time.time()
            return result
        
        return wrapper
    return decorator

@rate_limited(max_per_second=10)
def fetch_block_data(block_number: int):
    return w3.eth.get_block(block_number)
```

### 2. Veri Caching

Aynı verileri tekrar tekrar sorgulamayın:

```python
from functools import lru_cache
import redis

# Redis cache
redis_client = redis.Redis(host='localhost', port=6379, decode_responses=True)

def cached_query(key: str, ttl: int = 3600):
    """
    Redis ile query sonuçlarını cache'ler
    """
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            cache_key = f"{key}:{str(args)}:{str(kwargs)}"
            
            # Cache'de var mı kontrol et
            cached = redis_client.get(cache_key)
            if cached:
                return json.loads(cached)
            
            # Cache'de yok, query çalıştır
            result = func(*args, **kwargs)
            
            # Sonucu cache'le
            redis_client.setex(
                cache_key,
                ttl,
                json.dumps(result)
            )
            
            return result
        
        return wrapper
    return decorator

@cached_query("block_data", ttl=300)
def get_block_with_cache(block_number: int):
    return w3.eth.get_block(block_number)
```

### 3. Batch Processing

Çok sayıda işlem yapacaksanız batch processing kullanın:

```python
from web3 import Web3
from typing import List

def batch_get_transactions(tx_hashes: List[str], batch_size: int = 100):
    """
    Transaction'ları batch olarak getirir
    """
    results = []
    
    for i in range(0, len(tx_hashes), batch_size):
        batch = tx_hashes[i:i + batch_size]
        
        # Batch request oluştur
        batch_results = []
        for tx_hash in batch:
            try:
                tx = w3.eth.get_transaction(tx_hash)
                batch_results.append(tx)
            except Exception as e:
                print(f"❌ TX hata ({tx_hash}): {e}")
        
        results.extend(batch_results)
        
        # Rate limit için bekle
        time.sleep(0.5)
    
    return results
```

### 4. Error Handling

Blockchain data'sı her zaman tutarlı olmayabilir:

```python
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=2, max=10)
)
def safe_web3_call(func, *args, **kwargs):
    """
    Web3 çağrılarını retry logic ile güvenli hale getirir
    """
    try:
        return func(*args, **kwargs)
    except Exception as e:
        print(f"⚠️ Web3 call hatası: {e}")
        raise
```

## Sonuç

Python ekosistemi, blockchain veri analizi için son derece güçlü araçlar sunmaktadır. Web3.py ile düşük seviyeli RPC çağrıları yapabilir, TheGraph ile kompleks sorguları hızlıca çalıştırabilir ve Dune Analytics ile SQL tabanlı analizler yapabilirsiniz.

Bu araçları bir araya getirerek profesyonel monitoring sistemleri, trading bot'ları ve DeFi analiz platformları geliştirebilirsiniz. Önemli olan, doğru aracı doğru iş için kullanmak ve best practice'leri takip etmektir.

### Önerilen Kaynaklar

- [Web3.py Documentation](https://web3py.readthedocs.io/)
- [TheGraph Documentation](https://thegraph.com/docs/)
- [Dune Analytics](https://dune.com/)
- [Etherscan API](https://docs.etherscan.io/)
- [Ethereum JSON-RPC Specification](https://ethereum.org/en/developers/docs/apis/json-rpc/)
