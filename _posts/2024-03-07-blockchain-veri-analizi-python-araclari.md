---
title: "Blockchain Veri Analizi: Python Araçları ve On-Chain Stratejiler"
description: "Web3.py, TheGraph ve Dune Analytics kullanarak on-chain veri okuma, event monitoring ve whale tracking teknikleri üzerine kıdemli veri mühendisi rehberi."
date: 2024-03-07
categories: [Blockchain, Data Science]
tags: [python, web3py, datascience, thegraph, dune, on-chain-analysis, mev]
image:
  path: /assets/img/posts/blockchain-data-analytics-architecture.png
  alt: "Blockchain Data Analysis Architecture"
---

## Giriş: Veri Yeni Petroldür, Peki Rafineri Nerede?

Blockchain'in "şeffaf" olduğu söylenir, ancak ham veriye bakmak Matrix kodlarına bakmak gibidir. Bir Senior Data Engineer olarak, başarılı bir Web3 projesini başarısız olandan ayıran temel faktörün **On-Chain Intelligence** olduğunu gördüm.

Sadece fiyat takibi yapmak amatör işidir. Profesyonel analiz, bir balinanın Binance'e 10.000 ETH transfer ettiğini değil, o ETH'lerin hangi DeFi protokolünden çıktığını ve olası satış baskısını (Sell Pressure) hesaplamayı gerektirir. Bu yazıda, Python ekosistemindeki en güçlü araçlarla kendi "veri rafinerimizi" kuracağız.

![TheGraph Protocol Mimarisi](/assets/img/posts/thegraph-protocol-architecture.png)

## 1. Web3.py ile Ham Veri Madenciliği

Her şeyin başladığı yer: RPC Node. Ancak `get_block` çağırıp döngü kurmak, 2017'de kaldı. Profesyonel sistemlerde asenkron mimari (AsyncIO) kullanmak zorundasınız, yoksa Ethereum Mainnet'in hızına yetişemezsiniz.

### Asenkron Event Monitoring
Akıllı kontratların yaydığı (emit) event'leri (örn: `Swap`, `Transfer`) gerçek zamanlı yakalamak için WebSocket bağlantısı şarttır.

```python
import asyncio
from web3 import AsyncWeb3, WebSocketProvider

async def watch_pair_events():
    async with AsyncWeb3(WebSocketProvider("wss://eth-mainnet.g.alchemy.com/v2/KEY")) as w3:
        # Uniswap V2 USDC/ETH Pair
        pair = w3.eth.contract(address="0xB4e16...", abi=PAIR_ABI)
        
        # Filtreleme: Sadece Swap eventleri
        event_filter = await pair.events.Swap.create_filter(fromBlock='latest')
        
        while True:
            for event in await event_filter.get_new_entries():
                handle_swap(event) # Custom logic
            await asyncio.sleep(2)
```
> **Performance Tip:** Event dinlerken `get_all_entries()` yerine `get_new_entries()` kullanın ve block range'i küçük tutun. Aksi takdirde RPC sağlayıcınız sizi banlar.

## 2. Kendi Indexer'ınızı Yazmak: SQLite ve Python

Bazen TheGraph yavaş kalabilir veya istediğiniz özel veriyi içermeyebilir. Bu durumda, hafif (lightweight) bir indexer yazmak en iyi çözümdür. `SQLAlchemy` ve `Web3.py` ikilisi ile milyonlarca transferi yerel veritabanınıza (SQLite/PostgreSQL) kaydedebilirsiniz.

```python
from sqlalchemy import create_engine, Column, String, Float, Integer
from sqlalchemy.orm import declarative_base, sessionmaker

Base = declarative_base()

class Transfer(Base):
    __tablename__ = 'transfers'
    id = Column(Integer, primary_key=True)
    tx_hash = Column(String, index=True)
    from_addr = Column(String, index=True)
    to_addr = Column(String)
    value = Column(Float)
    block_number = Column(Integer)

# 10 satırlık bu model ile milyonlarca transferi sorgulayabilirsiniz
engine = create_engine('sqlite:///blockchain_data.db')
Base.metadata.create_all(engine)
```

Bu yaklaşımın avantajı, karmaşık SQL sorgularını (JOIN, GROUP BY) yerel diskiniz hızında çalıştırabilmenizdir. Örneğin, "Son 1 yılda Uniswap'ta işlem yapan ve cüzdanında en az 1 NFT tutan kullanıcılar" sorusunun cevabını TheGraph ile saatlerce ararken, yerel indexer ile milisaniyede bulursunuz.

![Blockchain Data Analytics Dashboard](/assets/img/posts/blockchain-data-analytics-architecture.png)

## 3. TheGraph: Veri Okyanusunu İndekslemek

Ham blok verisiyle savaşmak yerine, TheGraph (GRT) protokolünün indekslenmiş verilerini kullanmak "Time-to-Insight" sürenizi %90 azaltır. GraphQL kullanarak karmaşık sorguları (örn: "Son 24 saatte en çok hacim yapan ilk 5 Uniswap havuzu") tek request ile alabilirsiniz.

```python
# TheGraph GraphQL Sorgusu (Python)
from gql import gql, Client
# ... (kısaltıldı)
```

## 4. Dune Analytics: SQL Gücü

Python ile uğraşmadan hızlı prototipleme yapmak istiyorsanız Dune Analytics bir numaralı dostunuzdur. SQL bilginizi kullanarak on-chain veriyi sorgulayabilir ve `dune-client` ile bu veriyi Python pipeline'ınıza (örn: Airflow) dahil edebilirsiniz.

![Dune Analytics Dashboard](/assets/img/posts/dune-analytics-dashboard-visualization.png)

## 5. MEV Analizi: Dark Forest'ı İzlemek

Ethereum'un karanlık ormanında (Dark Forest), arbitraj botları ve sandwich saldırıları cirit atar. Bir veri analisti olarak, `Mempool` takibi yaparak bu fırsatları yakalayabilirsiniz.

Normal data analizinden farkı, burada "pending" (bekleyen) işlemleri analiz etmeniz gerekmesidir. Flashbots RPC kullanarak, hangi işlemlerin "bundle" olarak gönderildiğini ve ne kadar "bribe" (rüşvet) ödendiğini analiz edebilirsiniz.

```python
# Pending Transaction Analizi (Basitleştirilmiş)
def analyze_mempool(tx_hash):
    tx = w3.eth.get_transaction(tx_hash)
    
    # Hedef: Uniswap Router V2
    if tx['to'] == UNISWAP_ROUTER:
        decoded_input = decode_input(tx['input'])
        
        # Eğer slippage %2'den yüksekse sandwich saldırısı dene
        if calculate_slippage(decoded_input) > 2.0:
            print(f"🥪 Sandwich Fırsatı: {tx_hash}")
            
            # Flashbots Bundle Oluştur
            bundle = [
                {"signed_transaction": my_frontrun_tx}, # Bizim alım emrimiz
                {"signed_transaction": tx.rawTransaction}, # Kurbanın işlemi
                {"signed_transaction": my_backrun_tx}   # Bizim satış emrimiz
            ]
            
            send_flashbots_bundle(bundle)
```
Bu basit kod parçası bile, doğru optimize edilirse (Rust/Go ile rewrite) karlı bir MEV botunun temelini oluşturabilir. Ancak dikkat: Mainnet'te rekabet çok yüksektir ve yanlış bir hesaplama tüm sermayenizi gas fee olarak yakmanıza neden olabilir.

![Telegram Bot Architecture for Whale Alerts](/assets/img/posts/telegram-bot-python-architecture.png)

## 6. Whale Alert Sistemi: Balinaları Avlamak

Zincir üzerindeki büyük hareketleri takip etmek, piyasa yönü hakkında en net sinyali verir. İşte `python-telegram-bot` kullanarak basit bir alarm sistemi:

```python
import asyncio
from telegram import Bot

TELEGRAM_TOKEN = "YOUR_BOT_TOKEN"
CHAT_ID = "YOUR_CHANNEL_ID"

async def check_whale_movement(tx):
    value_eth = w3.from_wei(tx['value'], 'ether')
    
    # 500 ETH üzeri işlemleri bildir
    if value_eth > 500:
        msg = f"""
🚨 **BALİNA ALARMI** 🚨
-------------------------
Miktar: {value_eth:.2f} ETH
Kimden: `{tx['from']}`
Kime: `{tx['to']}`
TX: [Etherscan Link](https://etherscan.io/tx/{tx['hash'].hex()})
        """
        async with Bot(TELEGRAM_TOKEN) as bot:
            await bot.send_message(chat_id=CHAT_ID, text=msg, parse_mode='Markdown')

# Main loop içinde çağırın
# await check_whale_movement(tx)
```
Bu botu bir AWS Lambda fonksiyonuna veya Raspberry Pi üzerine kurarak 7/24 balina takibi yapabilirsiniz.

## 7. Teknik Sözlük (Glossary)

*   **RPC (Remote Procedure Call):** Blockchain node'u ile iletişim kurmamızı sağlayan protokol.
*   **Indexer:** Blok zincirindeki karmaşık verileri (örn: bir kullanıcının geçmişi) hızlıca sorgulanabilir hale getiren veritabanı yapısı.
*   **MEV (Maximal Extractable Value):** Madencilerin veya botların işlem sırasını değiştirerek elde ettiği ekstra kazanç.
*   **Dark Forest:** Ethereum'un herkese açık ancak tehlikeli (front-running, sandwich attacks) işlem havuzu (mempool).

## 8. Sonuç ve İleri Okuma

Veri analizi, sadece kod yazmak değil, zincir üzerindeki hikayeyi okuyabilmektir. Bu araçları kullanarak kendi alfa sinyallerinizi oluşturabilirsiniz.

> **Pro Tip:** Sadece fiyat verisine odaklanmayın. "On-chain footprints" (zincir üstü ayak izleri) genellikle fiyat hareketinden günler önce sinyal verir. Özellikle yeni mint edilen stabil coin miktarı ve borsa giriş/çıkışları en güvenilir öncü göstergelerdir.
{: .prompt-tip }

Daha derin teknik analiz için [Python ile Algoritmik Trading](/trading/python/2024/08/02/python-ile-websocket-trading-bot/) yazımı okuyabilir veya veriyi nasıl güvenli işleyeceğinizi öğrenmek için [Smart Contract Security](/security/ethereum/2024/03/15/ethereum-smart-contract-security-best-practices/) rehberime göz atabilirsiniz.

Veri analizi projelerinizi veya bulduğunuz ilginç sinyalleri yorumlarda paylaşın, birlikte daha büyük bir veri seti oluşturalım.
