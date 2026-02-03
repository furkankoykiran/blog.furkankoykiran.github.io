---
title: "Python ve Web3.py ile Ethereum Blockchain Etkileşimi: Üst Düzey Geliştirme Rehberi"
description: "Web3.py kütüphanesini kullanarak Ethereum blockchain üzerinde profesyonel uygulama geliştirme. Provider mimarisi, asenkron yapılar ve akıllı kontrat entegrasyonu."
date: "2024-05-30"
categories: [Web3-Development, Python]
tags: [web3py, ethereum, blockchain, python, smart-contracts, dApp, infrastructure]
image:
  path: "/assets/img/posts/web3py-ethereum-development.png"
  alt: "Python Web3.py Geliştirme Eko-sistemi"
---

## Giriş: Neden Python ile Web3?

Blockchain dünyasında JavaScript (özellikle ethers.js ve web3.js) hakimiyeti olsa da, veri bilimi, yapay zeka ve sofistike backend sistemleri söz konusu olduğunda Python'ın yeri sarsılmazdır. Bir senior yazılım mühendisi için Web3.py, sadece bir kütüphane değil; Ethereum ekosistemini Python'ın zengin kütüphane havuzuna (Pandas, Scikit-learn, FastAPI) bağlayan stratejik bir köprüdür.

Bu yazıda, Web3.py kütüphanesini bir "script" yazmanın ötesine geçerek, prodüksiyon seviyesinde bir altyapı olarak nasıl kurgulayacağımızı, provider seçim stratejilerini ve asenkron mimarilerin önemini inceleyeceğiz.

![Python Web3 Blockchain Entegrasyonu](/assets/img/posts/python-web3-blockchain.png)

## Provider Mimarisi: Bağlantı Stratejinizi Seçmek

Ethereum ağındaki veriye ulaşmak veya işlem göndermek için bir "node" ile konuşmanız gerekir. Web3.py, bu iletişimi `Provider` sınıfları üzerinden yönetir. Ancak her provider her senaryo için uygun değildir.

### 1. HTTPProvider: Standart ve Güvenilir
REST tabanlı uygulamalar ve tek seferlik sorgular (bakiye kontrolü, kontrat durumu okuma) için en stabil yoldur. Alchemy veya Infura gibi servislerle kusursuz çalışır.
- **Kullanım:** "Request-Response" döngüsü gerektiren senaryolar.

### 2. WebsocketProvider: Gerçek Zamanlı Veri Akışı
Eğer bir "trading bot" veya "live indexer" geliştiriyorsanız, HTTP polling (sürekli sorma) yerine WebSocket kullanmalısınız. Yeni bloklar veya belirli event'ler oluştuğu anda size "push" yapılır.
- **Senior İpucu:** WebSocket bağlantıları kopmaya meyillidir. Prodüksiyon ortamında mutlaka "reconnection" (yeniden bağlanma) mantığı kurulmalıdır.

### 3. IPCProvider: Yerel Maksimum Performans
Eğer uygulamayı koşturduğunuz sunucuda bir Geth veya Nethermind node'u çalışıyorsa, en hızlı iletişim yolu Unix Domain Sockets (IPC) kullanmaktır. Ağ gecikmesini (latency) sıfıra indirir.

![Web3 Uygulama Mimarisi](/assets/img/posts/web3-application-architecture.png)

## Asenkron Programlama: AsyncHTTPProvider'ın Gücü

Modern Web3 uygulamaları artık "synchronous" (senkron) yapılarla ölçeklenemeyecek kadar karmaşık. Web3.py v6 ile birlikte gelen tam asenkron destek, yüzlerce kontrat sorgusunu aynı anda (concurrently) yapabilmenizi sağlar.

```python
from web3 import AsyncWeb3

async def monitor_network():
    # Asenkron provider başlatma
    w3 = AsyncWeb3(AsyncWeb3.AsyncHTTPProvider('https://mainnet.infura.io/v3/YOUR_ID'))
    
    # Blok numarasını beklemeden alırken diğer işlere devam edebilirsiniz
    block = await w3.eth.block_number
    print(f"Güncel Blok: {block}")

# Bir senior olarak, 'blocking' (engelleyici) kodlardan kaçınmalısınız.
```

Asenkron yapı, özellikle ağ gecikmelerinin (network latency) yüksek olduğu blockchain dünyasında uygulamanızın "donmamasını" sağlar. `AsyncWeb3` kullanımı, botlarınızın veya API'lerinizin throughput (iş çıkarma kapasitesi) değerini 10 katına çıkarabilir.

![Web3 Geliştirici İş Akışı](/assets/img/posts/web3py-developer-workflow.png)

## Hesap Yönetimi: Private Key'leri Korumak

Bir dApps geliştirirken yapılan en büyük amatör hata, private key'leri kodun içine "hardcode" olarak gömmektir. Senior seviyesinde bir yaklaşım, `python-dotenv` kullanarak çevre değişkenlerini yönetmeyi veya AWS Secrets Manager / HashiCorp Vault gibi araçları entegre etmeyi gerektirir.

```python
import os
from eth_account import Account
from dotenv import load_dotenv

load_dotenv()

# Private key'i güvenli şekilde yükle
private_key = os.getenv('ETHEREUM_PRIVATE_KEY')
account = Account.from_key(private_key)

print(f"Cüzdan Adresi: {account.address}")
```

### Nonce Yönetimi
Ethereum'da her işlemin bir `nonce` (sayısal sıra) değeri vardır. Eğer eş zamanlı olarak birden fazla işlem gönderiyorsanız (örneğin bir arbitraj botunda), her işlemin nonce değerini manuel olarak takip etmek ve artırmak zorundasınız. Aksi takdirde "Replacement transaction underpriced" hatalarıyla karşılaşırsınız.

## Transaction Yaşam Döngüsü: EIP-1559 ve Modern Gas Yönetimi

Artık sadece `gasPrice` göndermek yeterli değil. London Hard Fork sonrasında hayatımıza giren EIP-1559 ile birlikte `maxFeePerGas` ve `maxPriorityFeePerGas` kavramlarını kullanmalısınız.

**Senior Analizi:** Gaz fiyatları volatilite gösterdiği için dinamik bir tahminleyici kullanmak şarttır. `w3.eth.max_priority_fee` değerini baz alarak, ağın o anki yoğunluğuna göre "tip" belirlemek işleminizin hızını doğrudan etkiler.

```python
def prepare_transaction(to_address, amount_eth, w3):
    nonce = w3.eth.get_transaction_count(account.address)
    
    # Dinamik gas tahmini
    max_priority_fee = w3.eth.max_priority_fee
    base_fee = w3.eth.get_block('latest')['baseFeePerGas']
    max_fee = (2 * base_fee) + max_priority_fee

    tx = {
        'type': 2, # EIP-1559
        'nonce': nonce,
        'to': to_address,
        'value': w3.to_wei(amount_eth, 'ether'),
        'gas': 21000,
        'maxFeePerGas': max_fee,
        'maxPriorityFeePerGas': max_priority_fee,
        'chainId': w3.eth.chain_id
    }
    return tx
```

![EVM Execusion Diyagramı](/assets/img/posts/evm-architecture-execution-diagram.png)

## Mühendislik Notu: Wait for Receipt ve Confirmation
İşlemi göndermek yetmez; işlemin blok içerisine girdiğinden ve belirli bir "confirmation" sayısına ulaştığından emin olmalısınız. Web3.py'deki `wait_for_transaction_receipt` fonksiyonu, işleminiz onaylanana kadar execution'ı bekletir.

![Python AsyncIO Mimari](/assets/img/posts/python-asyncio-concurrent-programming.png)

## Akıllı Kontratlarla Etkileşim: ABI ve Checksum

Ethereum üzerindeki bir akıllı kontratla konuşmak için iki şeye ihtiyacınız vardır: Kontratın adresi ve ABI (Application Binary Interface) dosyası. ABI, kontratın hangi fonksiyonlara sahip olduğunu ve bu fonksiyonların hangi parametreleri aldığını belirten bir JSON haritasıdır.

### Checksum Address Neden Önemli?
Ethereum adresleri büyük/küçük harf duyarsızdır ancak yanlış yazımları önlemek için bir "checksum" (doğrulama) mekanizması kullanılır. Web3.py, adreslerin her zaman `to_checksum_address()` fonksiyonuyla doğrulanmış formatta olmasını bekler.

```python
import json

# ERC-20 standart ABI'sinden bir parça
abi = json.loads('[{"constant":true,"inputs":[{"name":"_owner","type":"address"}],"name":"balanceOf","outputs":[{"name":"balance","type":"uint256"}],"type":"function"}]')
usdt_address = w3.to_checksum_address('0xdAC17F958D2ee523a2206206994597C13D831ec7')

contract = w3.eth.contract(address=usdt_address, abi=abi)

# Bakiye sorgulama (Read-only call)
balance = contract.functions.balanceOf(account.address).call()
print(f"USDT Bakiyesi: {balance / 10**6}") # USDT 6 decimal'dir.
```

## Event Listening: Zincirdeki Hareketleri Takip Etmek

Bir senior mühendis için blockchain verisini çekmenin en verimli yolu `Event`leri (Olaylar) dinlemektir. Akıllı kontratlar, önemli işlemler gerçekleştiğinde (transfer, satış, oy verme vb.) loglar yayınlar.

Web3.py üzerinden `create_filter` veya `get_logs` kullanarak bu geçmişe ulaşabilir veya `WebSocketProvider` ile canlı olarak dinleyebilirsiniz.

**Performans Uyarısı:** Çok geniş bir blok aralığında (örn: 0'dan son bloğa) event sorgusu yapmak, RPC sağlayıcınızın sizi bloklamasına veya timeout almanıza neden olur. Sorgularınızı küçük blok parçalarına (chunks) bölerek yapmalısınız.

```python
# Örnek: Son 100 bloğu tara
event_filter = contract.events.Transfer.create_filter(fromBlock=block - 100)
events = event_filter.get_all_entries()
![Blockchain Veri Analizi Altyapısı](/assets/img/posts/blockchain-data-analytics-architecture.png)

## Middleware: Web3.py'nin Kontrol Merkezi

Web3.py'nin en esnek özelliklerinden biri `Middleware` (Ara Yazılım) katmanıdır. İstek gönderilmeden önce veya yanıt alındıktan sonra araya girerek veriyi manipüle etmenize olanak tanır.

**Senior Kullanımı:** Örneğin, PoA (Proof of Authority) ağlarında (Polygon Mumbai veya Goerli gibi) çalışırken `geth_poa_middleware` kullanmanız zorunludur. Aksi takdirde blok başlıklarındaki ekstra veriler Web3.py tarafından anlaşılamaz ve hata verir.

```python
from web3.middleware import geth_poa_middleware

# PoA ağları için middleware ekleme
w3.middleware_onion.inject(geth_poa_middleware, layer=0)
```

Ayrıca kendi özel middleware'lerinizi yazarak; otomatik gas artırma, logging veya hata yakalama (retry logic) mekanizmaları kurabilirsiniz.

## Test ve Geliştirme: Forking ve Simulation

Gerçek ağda işlem yapmadan önce kodunuzu test etmeniz hayati önem taşır. `eth-tester` kütüphanesi yerel bir blockchain simülasyonu sunar ancak asıl "senior" yöntem **Mainnet Forking**'dir.

Hardhat veya Anvil kullanarak Ethereum ana ağını yerel bilgisayarınıza klonlayabilir, devasa likiditeye sahip kontratlarla (Uniswap gibi) sanki gerçekmiş gibi ama gas ücreti ödemeden etkileşime geçebilirsiniz.

```python
# Yerel fork'a bağlanma örneği
w3 = Web3(Web3.HTTPProvider('http://127.0.0.1:8545'))
```

### Teknik Sözlük (Glossary)

- **Provider:** Blockchain node'u ile iletişimi sağlayan arayüz.
- **ABI (Application Binary Interface):** Akıllı kontrat fonksiyonlarının haritası.
- **Nonce:** Bir adresten gönderilen işlem sırasını belirten benzersiz sayı.
- **Checksum:** Adres yazım hatalarını önleyen büyük/küçük harf doğrulaması.
- **EIP-1559:** Gaz ücretlerini daha öngörülebilir kılan modern Ethereum standartı.
- **Middleware:** İstek ve yanıt döngüsünde çalışan özelleştirilebilir katmanlar.
- **Wait for Receipt:** İşlemin blok içerisine girmesini bekleme süreci.
- **Contract Call vs. Transact:** Call, zincirde değişiklik yapmaz ve ücretsizdir. Transact ise değişiklik yapar ve gas gerektirir.

## Sonuç: Geleceğin Web3 Altyapısı

Python ve Web3.py ikilisi, sadece blockchain etkileşimi için değil, aynı zamanda veriyi işleyip anlamlı içgörüler üreten sistemler için en güçlü araç setidir. Bir mühendis olarak, kodunuzda her zaman asenkron yapıları tercih etmeli, güvenliği (private key yönetimi) en baştan kurgulamalı ve gas optimizasyonunu bir alışkanlık haline getirmelisiniz.

Blockchain dünyası hızla gelişiyor; ZK-Rolluplar, L2 çözümleri ve yeni standartlar her gün karşımıza çıkıyor. Eğer bu altyapıyı kullanarak ölçeklenebilir ağlarda uygulama geliştirmek isterseniz [Polygon Network Mimarisi](/blockchain/infrastructure/2024/06/05/polygon-network-architecture-deep-dive/) rehberimizi, gerçek zamanlı veri akışını ticari stratejilere dönüştürmek isterseniz de [Python ile Otomatik Trading Sistemleri](/trading/python/2024/09/12/building-automated-trading-systems-python/) yazımızı inceleyebilirsiniz. Web3.py, bu değişen dünyada Python geliştiricilerinin en sadık dostu olmaya devam edecektir.

## İleri Okuma ve Kaynaklar
- [Web3.py Resmi Dokümantasyonu](https://web3py.readthedocs.io/)
- [Ethereum Developer Portal](https://ethereum.org/developers/)
- [Etherscan API ve Araçları](https://etherscan.io/apis)
- [Python AsyncIO Temelleri](https://docs.python.org/3/library/asyncio.html)
