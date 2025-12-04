---
title: "Web3.py ile Ethereum Blockchain Etkileşimi"
description: "Python Web3.py kütüphanesi ile Ethereum blockchain geliştirme rehberi. Wallet bağlantısı, transaction gönderme, smart contract etkileşimi ve DApp development."
date: "2024-05-30"
categories:
  - "web3-development"
  - "blockchain"
tags:
  - "python"
  - "web3py"
  - "ethereum"
  - "blockchain"
  - "smart-contracts"
  - "dapp"
  - "development"
image:
  path: "/assets/img/posts/web3py-ethereum-development.png"
  alt: "Web3.py ile Ethereum Blockchain Development"
---

Python programlama dilinin güçlü ekosistemi ve Ethereum blockchain'inin devrim niteliğindeki teknolojisi bir araya geldiğinde, geliştiriciler için sınırsız olanaklar doğuyor. Web3.py kütüphanesi, bu iki dünyayı birleştiren köprü görevi görüyor ve Python geliştiricilerinin Ethereum blockchain ile kolayca etkileşime geçmesini sağlıyor.

Bu kapsamlı rehberde, Web3.py kütüphanesini kullanarak Ethereum blockchain ile nasıl etkileşime geçeceğinizi, wallet bağlantısı nasıl yapacağınızı, transaction'lar nasıl göndereceğinizi ve smart contract'larla nasıl çalışacağınızı öğreneceksiniz.

## Web3.py Nedir ve Neden Kullanmalıyız?

Web3.py, Ethereum blockchain ile etkileşim kurmak için tasarlanmış bir Python kütüphanesidir. Ethereum Foundation tarafından resmi olarak desteklenen bu kütüphane, blockchain üzerinde okuma ve yazma işlemleri yapmamızı sağlar.

### Web3.py'nin Temel Avantajları

- **Python Ekosistemi**: Python'un zengin kütüphane ekosisteminden yararlanma
- **Güçlü Dokümantasyon**: Kapsamlı ve sürekli güncellenen dökümantasyon
- **Asenkron Destek**: Async/await ile yüksek performanslı uygulamalar
- **Geniş Topluluk**: Aktif geliştirici topluluğu ve bol kaynak
- **Çoklu Provider Desteği**: HTTP, WebSocket, IPC bağlantı seçenekleri

## Kurulum ve Temel Yapılandırma

Web3.py ile çalışmaya başlamak için öncelikle gerekli kütüphaneleri kurmamız gerekiyor.

### Kütüphane Kurulumu

```bash
# Web3.py kurulumu
pip install web3

# Geliştirme için ek araçlar
pip install python-dotenv
pip install eth-account
pip install eth-utils
```
{: .nolineno }

> Web3.py 6.x sürümünü kullanmanızı öneririz. Eski sürümlerle API farklılıkları olabilir.
{: .prompt-tip }

### İlk Bağlantı Kurulumu

Ethereum node'una bağlanmak için birkaç farklı yöntem kullanabiliriz. En yaygın yöntemler Infura, Alchemy gibi servis sağlayıcılar veya yerel bir node kullanmaktır.

```python
from web3 import Web3
import os
from dotenv import load_dotenv

# Çevre değişkenlerini yükle
load_dotenv()

# Infura üzerinden bağlantı
INFURA_URL = f"https://mainnet.infura.io/v3/{os.getenv('INFURA_PROJECT_ID')}"
w3 = Web3(Web3.HTTPProvider(INFURA_URL))

# Bağlantıyı kontrol et
if w3.is_connected():
    print("✅ Ethereum node'una başarıyla bağlanıldı!")
    print(f"📊 Mevcut block numarası: {w3.eth.block_number}")
else:
    print("❌ Bağlantı başarısız!")
```
{: file="connect_ethereum.py" }

> Infura veya Alchemy API key'lerinizi `.env`{: .filepath} dosyasında saklayın ve `.gitignore`{: .filepath}'a ekleyin.
{: .prompt-danger }

![Web3 Developer Workflow](/assets/img/posts/web3py-developer-workflow.png){: w="800" h="500" .shadow }
_Web3.py ile geliştirme sürecinin genel akışı_

### Provider Türleri ve Kullanım Senaryoları

```python
from web3 import Web3

# 1. HTTP Provider (En yaygın kullanım)
http_provider = Web3(Web3.HTTPProvider('https://mainnet.infura.io/v3/YOUR_KEY'))

# 2. WebSocket Provider (Gerçek zamanlı veri için)
ws_provider = Web3(Web3.WebsocketProvider('wss://mainnet.infura.io/ws/v3/YOUR_KEY'))

# 3. IPC Provider (Yerel node için en hızlı)
ipc_provider = Web3(Web3.IPCProvider('/path/to/geth.ipc'))

# 4. Testnet bağlantısı (Geliştirme için)
goerli_provider = Web3(Web3.HTTPProvider('https://goerli.infura.io/v3/YOUR_KEY'))

# Provider değiştirme
def switch_network(network='mainnet'):
    """Network'ü dinamik olarak değiştir"""
    networks = {
        'mainnet': 'https://mainnet.infura.io/v3/YOUR_KEY',
        'goerli': 'https://goerli.infura.io/v3/YOUR_KEY',
        'sepolia': 'https://sepolia.infura.io/v3/YOUR_KEY',
        'polygon': 'https://polygon-rpc.com'
    }
    
    if network in networks:
        w3 = Web3(Web3.HTTPProvider(networks[network]))
        if w3.is_connected():
            print(f"✅ {network.upper()} ağına bağlanıldı!")
            return w3
    return None
```

## Ethereum Temel Kavramları

Ethereum ile çalışmadan önce temel kavramları anlamak önemli.

### Account (Hesap) Yapısı

```python
from eth_account import Account
import secrets

# Yeni bir wallet oluşturma
def create_new_wallet():
    """Yeni bir Ethereum wallet oluştur"""
    # Güvenli rastgele private key oluştur
    private_key = "0x" + secrets.token_hex(32)
    
    # Account nesnesini oluştur
    account = Account.from_key(private_key)
    
    wallet_info = {
        'address': account.address,
        'private_key': private_key,
        'public_key': account._key_obj.public_key.to_hex()
    }
    
    print(f"🆕 Yeni Wallet Oluşturuldu!")
    print(f"📍 Adres: {wallet_info['address']}")
    print(f"🔑 Private Key: {wallet_info['private_key']}")
    print(f"⚠️  DİKKAT: Private key'i güvenli bir yerde saklayın!")
    
    return wallet_info

# Mevcut private key'den wallet yükleme
def load_wallet(private_key):
    """Mevcut bir wallet'ı yükle"""
    try:
        account = Account.from_key(private_key)
        print(f"✅ Wallet yüklendi: {account.address}")
        return account
    except Exception as e:
        print(f"❌ Wallet yükleme hatası: {e}")
        return None
```

### Balance (Bakiye) Sorgulama

```python
def get_balance(w3, address):
    """Bir adresin ETH bakiyesini sorgula"""
    try:
        # Wei cinsinden bakiye
        balance_wei = w3.eth.get_balance(address)
        
        # ETH'ye çevir
        balance_eth = w3.from_wei(balance_wei, 'ether')
        
        print(f"💰 Adres: {address}")
        print(f"💵 Bakiye: {balance_eth} ETH")
        print(f"📊 Wei: {balance_wei}")
        
        return {
            'eth': float(balance_eth),
            'wei': balance_wei
        }
    except Exception as e:
        print(f"❌ Bakiye sorgulama hatası: {e}")
        return None

# Örnek kullanım
vitalik_address = "0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045"
balance = get_balance(w3, vitalik_address)
```

### Wei, Gwei ve Ether Dönüşümleri

Ethereum'da değerler genellikle Wei cinsinden tutulur. 1 Ether = 10^18 Wei'dir.

```python
from web3 import Web3

# Dönüşüm fonksiyonları
def wei_conversions():
    """Wei, Gwei ve ETH arası dönüşümler"""
    
    # 1 ETH = 10^18 Wei
    one_eth_in_wei = Web3.to_wei(1, 'ether')
    print(f"1 ETH = {one_eth_in_wei} Wei")
    
    # Wei'den ETH'ye
    wei_amount = 1500000000000000000  # 1.5 ETH
    eth_amount = Web3.from_wei(wei_amount, 'ether')
    print(f"{wei_amount} Wei = {eth_amount} ETH")
    
    # Gwei (Gas price için kullanılır)
    gas_price_gwei = 50
    gas_price_wei = Web3.to_wei(gas_price_gwei, 'gwei')
    print(f"{gas_price_gwei} Gwei = {gas_price_wei} Wei")
    
    # Tüm birimler
    units = ['wei', 'kwei', 'mwei', 'gwei', 'szabo', 'finney', 'ether']
    amount = 1
    
    print("\n📊 Ethereum Birimleri:")
    for unit in units:
        wei_value = Web3.to_wei(amount, unit)
        print(f"{amount} {unit.ljust(10)} = {wei_value:,} Wei")

wei_conversions()
```

## Transaction (İşlem) Gönderme

Ethereum üzerinde ETH transfer etmek için transaction oluşturup imzalamamız gerekiyor.

### Basit ETH Transferi

```python
from web3 import Web3
from eth_account import Account

def send_eth_transaction(w3, from_private_key, to_address, amount_eth):
    """
    ETH transfer işlemi gerçekleştir
    
    Args:
        w3: Web3 instance
        from_private_key: Gönderen wallet'ın private key'i
        to_address: Alıcı adres
        amount_eth: Gönderilecek miktar (ETH cinsinden)
    """
    try:
        # Account'u yükle
        account = Account.from_key(from_private_key)
        from_address = account.address
        
        # Nonce değeri (account'tan gönderilen transaction sayısı)
        nonce = w3.eth.get_transaction_count(from_address)
        
        # Gas fiyatını al
        gas_price = w3.eth.gas_price
        
        # Transaction dict'i oluştur
        transaction = {
            'nonce': nonce,
            'to': to_address,
            'value': w3.to_wei(amount_eth, 'ether'),
            'gas': 21000,  # Standart ETH transferi için
            'gasPrice': gas_price,
            'chainId': w3.eth.chain_id
        }
        
        # Transaction'ı imzala
        signed_txn = w3.eth.account.sign_transaction(transaction, from_private_key)
        
        # Transaction'ı gönder
        tx_hash = w3.eth.send_raw_transaction(signed_txn.rawTransaction)
        
        print(f"✅ Transaction gönderildi!")
        print(f"📝 TX Hash: {tx_hash.hex()}")
        print(f"🔗 Etherscan: https://etherscan.io/tx/{tx_hash.hex()}")
        
        # Transaction'ın onaylanmasını bekle
        print("⏳ Transaction onayı bekleniyor...")
        tx_receipt = w3.eth.wait_for_transaction_receipt(tx_hash)
        
        if tx_receipt['status'] == 1:
            print(f"✅ Transaction başarılı! Block: {tx_receipt['blockNumber']}")
        else:
            print("❌ Transaction başarısız!")
        
        return tx_hash.hex()
        
    except Exception as e:
        print(f"❌ Transaction hatası: {e}")
        return None

# Örnek kullanım (TESTNET'te deneyin!)
# tx_hash = send_eth_transaction(
#     w3=goerli_w3,
#     from_private_key="YOUR_PRIVATE_KEY",
#     to_address="0xRecipientAddress",
#     amount_eth=0.01
# )
```

### Gas Yönetimi ve Optimizasyonu

```python
def estimate_transaction_cost(w3, transaction):
    """Transaction maliyetini tahmin et"""
    try:
        # Gas limitini tahmin et
        gas_estimate = w3.eth.estimate_gas(transaction)
        
        # Mevcut gas fiyatı
        gas_price = w3.eth.gas_price
        
        # Toplam maliyet (Wei cinsinden)
        total_cost_wei = gas_estimate * gas_price
        
        # ETH'ye çevir
        total_cost_eth = w3.from_wei(total_cost_wei, 'ether')
        
        print(f"⛽ Gas Tahmin Raporu:")
        print(f"  • Gas Limit: {gas_estimate:,}")
        print(f"  • Gas Price: {w3.from_wei(gas_price, 'gwei')} Gwei")
        print(f"  • Toplam Maliyet: {total_cost_eth} ETH")
        print(f"  • USD Karşılığı: ${float(total_cost_eth) * get_eth_price():.2f}")
        
        return {
            'gas_limit': gas_estimate,
            'gas_price': gas_price,
            'total_cost_eth': float(total_cost_eth)
        }
    except Exception as e:
        print(f"❌ Gas tahmin hatası: {e}")
        return None

def get_eth_price():
    """ETH fiyatını al (basitleştirilmiş)"""
    # Gerçek uygulamada bir API'den çekin
    return 2000  # USD

# EIP-1559 Transaction (Modern yöntem)
def send_eip1559_transaction(w3, from_private_key, to_address, amount_eth):
    """EIP-1559 standardı ile transaction gönder"""
    account = Account.from_key(from_private_key)
    
    # Base fee ve priority fee
    latest_block = w3.eth.get_block('latest')
    base_fee = latest_block['baseFeePerGas']
    max_priority_fee = w3.to_wei(2, 'gwei')  # Tip for miners
    max_fee = base_fee * 2 + max_priority_fee
    
    transaction = {
        'nonce': w3.eth.get_transaction_count(account.address),
        'to': to_address,
        'value': w3.to_wei(amount_eth, 'ether'),
        'gas': 21000,
        'maxFeePerGas': max_fee,
        'maxPriorityFeePerGas': max_priority_fee,
        'chainId': w3.eth.chain_id,
        'type': 2  # EIP-1559 transaction tipi
    }
    
    signed_txn = w3.eth.account.sign_transaction(transaction, from_private_key)
    tx_hash = w3.eth.send_raw_transaction(signed_txn.rawTransaction)
    
    return tx_hash.hex()
```
{: file="send_transaction.py" }

> Gas fee'leri daima kontrol edin! Yüksek network congestion'da işlemler çok pahalı olabilir.
{: .prompt-warning }

![Python Web3 Blockchain](/assets/img/posts/python-web3-blockchain.png){: w="800" h="500" .shadow }
_Python ve Web3 teknolojilerinin entegrasyonu_

## Smart Contract Etkileşimi

Smart contract'lar Ethereum'un en güçlü özelliklerinden biri. Web3.py ile contract'ları okuyabilir ve yazabiliriz.

### Contract Yükleme ve Okuma

```python
import json

# ERC-20 Token Contract örneği
ERC20_ABI = json.loads('''[
    {
        "constant": true,
        "inputs": [{"name": "_owner", "type": "address"}],
        "name": "balanceOf",
        "outputs": [{"name": "balance", "type": "uint256"}],
        "type": "function"
    },
    {
        "constant": true,
        "inputs": [],
        "name": "totalSupply",
        "outputs": [{"name": "", "type": "uint256"}],
        "type": "function"
    },
    {
        "constant": true,
        "inputs": [],
        "name": "name",
        "outputs": [{"name": "", "type": "string"}],
        "type": "function"
    },
    {
        "constant": true,
        "inputs": [],
        "name": "symbol",
        "outputs": [{"name": "", "type": "string"}],
        "type": "function"
    },
    {
        "constant": false,
        "inputs": [
            {"name": "_to", "type": "address"},
            {"name": "_value", "type": "uint256"}
        ],
        "name": "transfer",
        "outputs": [{"name": "", "type": "bool"}],
        "type": "function"
    }
]''')

def load_contract(w3, contract_address, abi):
    """Smart contract yükle"""
    try:
        # Checksum address (büyük/küçük harf kontrolü)
        checksum_address = w3.to_checksum_address(contract_address)
        
        # Contract instance oluştur
        contract = w3.eth.contract(address=checksum_address, abi=abi)
        
        print(f"✅ Contract yüklendi: {checksum_address}")
        return contract
    except Exception as e:
        print(f"❌ Contract yükleme hatası: {e}")
        return None

# USDT contract örneği
USDT_ADDRESS = "0xdAC17F958D2ee523a2206206994597C13D831ec7"
usdt_contract = load_contract(w3, USDT_ADDRESS, ERC20_ABI)
```

### Contract Okuma İşlemleri (View Functions)

```python
def read_erc20_info(w3, contract_address):
    """ERC-20 token bilgilerini oku"""
    contract = load_contract(w3, contract_address, ERC20_ABI)
    
    if contract:
        try:
            # Token bilgileri
            name = contract.functions.name().call()
            symbol = contract.functions.symbol().call()
            total_supply = contract.functions.totalSupply().call()
            decimals = contract.functions.decimals().call() if hasattr(contract.functions, 'decimals') else 18
            
            # Formatlanmış supply
            formatted_supply = total_supply / (10 ** decimals)
            
            print(f"\n📊 Token Bilgileri:")
            print(f"  • İsim: {name}")
            print(f"  • Sembol: {symbol}")
            print(f"  • Decimals: {decimals}")
            print(f"  • Total Supply: {formatted_supply:,.2f} {symbol}")
            
            return {
                'name': name,
                'symbol': symbol,
                'decimals': decimals,
                'total_supply': formatted_supply
            }
        except Exception as e:
            print(f"❌ Token bilgileri okuma hatası: {e}")
    
    return None

def check_token_balance(w3, contract_address, wallet_address):
    """Bir wallet'ın token bakiyesini kontrol et"""
    contract = load_contract(w3, contract_address, ERC20_ABI)
    
    if contract:
        try:
            balance = contract.functions.balanceOf(wallet_address).call()
            decimals = contract.functions.decimals().call() if hasattr(contract.functions, 'decimals') else 18
            symbol = contract.functions.symbol().call()
            
            formatted_balance = balance / (10 ** decimals)
            
            print(f"💰 Token Bakiyesi:")
            print(f"  • Adres: {wallet_address}")
            print(f"  • Bakiye: {formatted_balance:,.4f} {symbol}")
            
            return formatted_balance
        except Exception as e:
            print(f"❌ Bakiye okuma hatası: {e}")
    
    return 0

# Örnek kullanım
read_erc20_info(w3, USDT_ADDRESS)
```

### Contract Yazma İşlemleri (Transaction Functions)

```python
def send_erc20_token(w3, contract_address, from_private_key, to_address, amount):
    """ERC-20 token transfer et"""
    try:
        # Account ve contract yükle
        account = Account.from_key(from_private_key)
        contract = load_contract(w3, contract_address, ERC20_ABI)
        
        if not contract:
            return None
        
        # Token decimals'ı al
        decimals = contract.functions.decimals().call() if hasattr(contract.functions, 'decimals') else 18
        
        # Amount'u wei formatına çevir
        amount_wei = int(amount * (10 ** decimals))
        
        # Transaction oluştur
        nonce = w3.eth.get_transaction_count(account.address)
        
        # Contract function çağrısı
        transaction = contract.functions.transfer(
            to_address,
            amount_wei
        ).build_transaction({
            'nonce': nonce,
            'gas': 100000,  # ERC-20 transfer için
            'gasPrice': w3.eth.gas_price,
            'chainId': w3.eth.chain_id
        })
        
        # İmzala ve gönder
        signed_txn = w3.eth.account.sign_transaction(transaction, from_private_key)
        tx_hash = w3.eth.send_raw_transaction(signed_txn.rawTransaction)
        
        print(f"✅ Token transfer gönderildi!")
        print(f"📝 TX Hash: {tx_hash.hex()}")
        
        # Onay bekle
        receipt = w3.eth.wait_for_transaction_receipt(tx_hash)
        
        if receipt['status'] == 1:
            print(f"✅ Transfer başarılı!")
            return tx_hash.hex()
        else:
            print(f"❌ Transfer başarısız!")
            return None
            
    except Exception as e:
        print(f"❌ Token transfer hatası: {e}")
        return None
```

### Event Listening (Olay Dinleme)

```python
def listen_to_transfer_events(w3, contract_address, from_block='latest'):
    """Transfer eventlerini dinle"""
    contract = load_contract(w3, contract_address, ERC20_ABI)
    
    if contract:
        try:
            # Transfer event filtresi oluştur
            event_filter = contract.events.Transfer.create_filter(
                fromBlock=from_block
            )
            
            print(f"👂 Transfer eventleri dinleniyor...")
            print(f"⏸️  Durdurmak için Ctrl+C basın\n")
            
            while True:
                for event in event_filter.get_new_entries():
                    from_addr = event['args']['from']
                    to_addr = event['args']['to']
                    value = event['args']['value']
                    
                    print(f"🔔 Yeni Transfer!")
                    print(f"  • From: {from_addr}")
                    print(f"  • To: {to_addr}")
                    print(f"  • Amount: {value}")
                    print(f"  • TX: {event['transactionHash'].hex()}\n")
                
                # 2 saniye bekle
                import time
                time.sleep(2)
                
        except KeyboardInterrupt:
            print("\n✋ Event listening durduruldu.")
        except Exception as e:
            print(f"❌ Event listening hatası: {e}")
```
{: file="event_listener.py" }

![Web3 Application Architecture](/assets/img/posts/web3-application-architecture.png){: w="800" h="500" .shadow }
_Web3 uygulamalarının mimari yapısı ve bileşenleri_

## İleri Seviye Teknikler

### Batch İstekler ile Performans Optimizasyonu

```python
from web3 import Web3
from concurrent.futures import ThreadPoolExecutor

def batch_balance_check(w3, addresses):
    """Birden fazla adresin bakiyesini paralel olarak kontrol et"""
    
    def get_single_balance(address):
        try:
            balance = w3.eth.get_balance(address)
            return {
                'address': address,
                'balance_eth': float(w3.from_wei(balance, 'ether')),
                'balance_wei': balance
            }
        except Exception as e:
            return {'address': address, 'error': str(e)}
    
    # Paralel işlem
    with ThreadPoolExecutor(max_workers=10) as executor:
        results = list(executor.map(get_single_balance, addresses))
    
    # Sonuçları göster
    print(f"\n📊 Batch Balance Sonuçları ({len(addresses)} adres):")
    for result in results:
        if 'error' not in result:
            print(f"  • {result['address']}: {result['balance_eth']:.4f} ETH")
        else:
            print(f"  • {result['address']}: ❌ {result['error']}")
    
    return results

# Örnek kullanım
addresses = [
    "0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045",  # Vitalik
    "0xAb5801a7D398351b8bE11C439e05C5B3259aeC9B",  # Example
    "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb"   # Example
]

batch_results = batch_balance_check(w3, addresses)
```

### Transaction Monitoring (İzleme) Sistemi

```python
import time
from datetime import datetime

def monitor_pending_transactions(w3, target_address):
    """Bir adresin pending transaction'larını izle"""
    
    print(f"🔍 {target_address} adresi izleniyor...")
    print(f"⏸️  Durdurmak için Ctrl+C\n")
    
    # Pending transaction filter
    pending_filter = w3.eth.filter('pending')
    
    try:
        while True:
            for tx_hash in pending_filter.get_new_entries():
                try:
                    tx = w3.eth.get_transaction(tx_hash)
                    
                    # Hedef adres kontrolü
                    if tx and (tx['from'].lower() == target_address.lower() or 
                              (tx['to'] and tx['to'].lower() == target_address.lower())):
                        
                        value_eth = w3.from_wei(tx['value'], 'ether')
                        timestamp = datetime.now().strftime('%H:%M:%S')
                        
                        print(f"[{timestamp}] 🆕 Yeni Transaction!")
                        print(f"  • Hash: {tx_hash.hex()}")
                        print(f"  • From: {tx['from']}")
                        print(f"  • To: {tx['to']}")
                        print(f"  • Value: {value_eth} ETH")
                        print(f"  • Gas Price: {w3.from_wei(tx['gasPrice'], 'gwei')} Gwei\n")
                        
                except Exception as e:
                    continue
            
            time.sleep(1)
            
    except KeyboardInterrupt:
        print("\n✋ Monitoring durduruldu.")
```

### Custom Smart Contract Deployment

```python
def deploy_contract(w3, from_private_key, bytecode, abi, constructor_args=None):
    """Smart contract deploy et"""
    try:
        account = Account.from_key(from_private_key)
        
        # Contract instance oluştur
        Contract = w3.eth.contract(abi=abi, bytecode=bytecode)
        
        # Constructor transaction oluştur
        if constructor_args:
            transaction = Contract.constructor(*constructor_args).build_transaction({
                'from': account.address,
                'nonce': w3.eth.get_transaction_count(account.address),
                'gas': 3000000,
                'gasPrice': w3.eth.gas_price,
                'chainId': w3.eth.chain_id
            })
        else:
            transaction = Contract.constructor().build_transaction({
                'from': account.address,
                'nonce': w3.eth.get_transaction_count(account.address),
                'gas': 3000000,
                'gasPrice': w3.eth.gas_price,
                'chainId': w3.eth.chain_id
            })
        
        # İmzala ve gönder
        signed_txn = w3.eth.account.sign_transaction(transaction, from_private_key)
        tx_hash = w3.eth.send_raw_transaction(signed_txn.rawTransaction)
        
        print(f"📤 Contract deployment gönderildi: {tx_hash.hex()}")
        
        # Receipt bekle
        receipt = w3.eth.wait_for_transaction_receipt(tx_hash)
        
        if receipt['status'] == 1:
            contract_address = receipt['contractAddress']
            print(f"✅ Contract başarıyla deploy edildi!")
            print(f"📍 Contract Address: {contract_address}")
            return contract_address
        else:
            print("❌ Deployment başarısız!")
            return None
            
    except Exception as e:
        print(f"❌ Deployment hatası: {e}")
        return None
```

## Best Practices ve Güvenlik

### Private Key Güvenliği

```python
import os
from cryptography.fernet import Fernet
import keyring

class SecureWallet:
    """Güvenli wallet yönetimi"""
    
    def __init__(self):
        # Şifreleme anahtarı oluştur veya yükle
        self.key = self._get_or_create_key()
        self.cipher = Fernet(self.key)
    
    def _get_or_create_key(self):
        """Şifreleme anahtarını al veya oluştur"""
        key = keyring.get_password("web3py_wallet", "encryption_key")
        if not key:
            key = Fernet.generate_key().decode()
            keyring.set_password("web3py_wallet", "encryption_key", key)
        return key.encode() if isinstance(key, str) else key
    
    def encrypt_private_key(self, private_key):
        """Private key'i şifrele"""
        encrypted = self.cipher.encrypt(private_key.encode())
        return encrypted.decode()
    
    def decrypt_private_key(self, encrypted_key):
        """Şifrelenmiş private key'i çöz"""
        decrypted = self.cipher.decrypt(encrypted_key.encode())
        return decrypted.decode()
    
    def save_wallet(self, name, private_key):
        """Wallet'ı güvenli şekilde kaydet"""
        encrypted = self.encrypt_private_key(private_key)
        keyring.set_password("web3py_wallet", name, encrypted)
        print(f"✅ Wallet '{name}' güvenli şekilde kaydedildi")
    
    def load_wallet(self, name):
        """Wallet'ı güvenli şekilde yükle"""
        encrypted = keyring.get_password("web3py_wallet", name)
        if encrypted:
            private_key = self.decrypt_private_key(encrypted)
            print(f"✅ Wallet '{name}' yüklendi")
            return private_key
        else:
            print(f"❌ Wallet '{name}' bulunamadı")
            return None

# Kullanım
secure_wallet = SecureWallet()
# secure_wallet.save_wallet("my_mainnet_wallet", "0xYOUR_PRIVATE_KEY")
# private_key = secure_wallet.load_wallet("my_mainnet_wallet")
```

### Error Handling ve Retry Mekanizması

```python
import time
from functools import wraps

def retry_on_failure(max_retries=3, delay=1, backoff=2):
    """Transaction başarısızlıklarında retry decorator"""
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            retries = 0
            current_delay = delay
            
            while retries < max_retries:
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    retries += 1
                    if retries >= max_retries:
                        print(f"❌ {max_retries} deneme sonrası başarısız: {e}")
                        raise
                    
                    print(f"⚠️  Hata oluştu (Deneme {retries}/{max_retries}): {e}")
                    print(f"⏳ {current_delay} saniye sonra tekrar denenecek...")
                    time.sleep(current_delay)
                    current_delay *= backoff
            
        return wrapper
    return decorator

@retry_on_failure(max_retries=3, delay=2)
def safe_send_transaction(w3, transaction, private_key):
    """Güvenli transaction gönderimi"""
    signed_txn = w3.eth.account.sign_transaction(transaction, private_key)
    tx_hash = w3.eth.send_raw_transaction(signed_txn.rawTransaction)
    receipt = w3.eth.wait_for_transaction_receipt(tx_hash, timeout=120)
    
    if receipt['status'] != 1:
        raise Exception("Transaction failed on-chain")
    
    return tx_hash.hex()
```

### Gas Price Optimizasyonu

```python
def get_optimal_gas_price(w3, speed='standard'):
    """Optimum gas price hesapla"""
    try:
        # Mevcut gas price
        current_gas = w3.eth.gas_price
        current_gwei = w3.from_wei(current_gas, 'gwei')
        
        # Speed'e göre ayarla
        multipliers = {
            'slow': 0.8,      # %20 daha ucuz (yavaş)
            'standard': 1.0,   # Normal
            'fast': 1.2,       # %20 daha hızlı
            'rapid': 1.5       # %50 daha hızlı
        }
        
        multiplier = multipliers.get(speed, 1.0)
        optimal_gwei = float(current_gwei) * multiplier
        optimal_wei = w3.to_wei(optimal_gwei, 'gwei')
        
        print(f"⛽ Gas Price Önerisi ({speed}):")
        print(f"  • Mevcut: {current_gwei:.2f} Gwei")
        print(f"  • Önerilen: {optimal_gwei:.2f} Gwei")
        print(f"  • Fark: {(multiplier - 1) * 100:+.0f}%")
        
        return optimal_wei
        
    except Exception as e:
        print(f"❌ Gas price hatası: {e}")
        return w3.eth.gas_price

# Kullanım
gas_price = get_optimal_gas_price(w3, speed='fast')
```

## Sık Yapılan Hatalar ve Çözümleri

### 1. Insufficient Funds Hatası

```python
def check_sufficient_balance(w3, from_address, amount_eth, gas_limit, gas_price):
    """İşlem için yeterli bakiye var mı kontrol et"""
    balance = w3.eth.get_balance(from_address)
    
    # Transaction maliyeti
    tx_value = w3.to_wei(amount_eth, 'ether')
    gas_cost = gas_limit * gas_price
    total_required = tx_value + gas_cost
    
    if balance < total_required:
        balance_eth = w3.from_wei(balance, 'ether')
        required_eth = w3.from_wei(total_required, 'ether')
        shortfall_eth = w3.from_wei(total_required - balance, 'ether')
        
        print(f"❌ Yetersiz Bakiye!")
        print(f"  • Mevcut: {balance_eth} ETH")
        print(f"  • Gerekli: {required_eth} ETH")
        print(f"  • Eksik: {shortfall_eth} ETH")
        return False
    
    print(f"✅ Yeterli bakiye mevcut")
    return True
```

### 2. Nonce Sorunları

```python
def get_safe_nonce(w3, address, pending=True):
    """Güvenli nonce değeri al"""
    if pending:
        # Pending transaction'ları dahil et
        nonce = w3.eth.get_transaction_count(address, 'pending')
    else:
        # Sadece onaylanmış transaction'lar
        nonce = w3.eth.get_transaction_count(address, 'latest')
    
    print(f"🔢 Nonce değeri: {nonce}")
    return nonce
```

### 3. Gas Limit Yetersizliği

```python
def estimate_with_buffer(w3, transaction, buffer_percent=20):
    """Gas limit'i güvenlik payı ile tahmin et"""
    estimated = w3.eth.estimate_gas(transaction)
    buffer = int(estimated * (buffer_percent / 100))
    safe_limit = estimated + buffer
    
    print(f"⛽ Gas Tahmini:")
    print(f"  • Tahmin: {estimated:,}")
    print(f"  • Buffer: {buffer:,} (%{buffer_percent})")
    print(f"  • Güvenli Limit: {safe_limit:,}")
    
    return safe_limit
```

## Sonuç ve Öneriler

Web3.py, Python geliştiricileri için Ethereum ekosisteminin kapılarını açan güçlü bir araçtır. Bu rehberde öğrendiklerimizi özetleyelim:

### Temel Kazanımlar

✅ **Web3.py Kurulumu ve Konfigürasyonu**: Farklı provider türleri ve network bağlantıları  
✅ **Wallet Yönetimi**: Account oluşturma, private key güvenliği  
✅ **Transaction İşlemleri**: ETH transferi, gas optimizasyonu, EIP-1559  
✅ **Smart Contract Etkileşimi**: Contract okuma/yazma, event listening  
✅ **İleri Seviye Teknikler**: Batch işlemler, monitoring, deployment  
✅ **Güvenlik**: Best practices, error handling, secure coding  

### Geliştiriciler İçin Öneriler

1. **Test Ortamı Kullanın**: Mainnet'te işlem yapmadan önce mutlaka Goerli veya Sepolia testnet'te test edin
2. **Gas Yönetimi**: Gas limit ve gas price ayarlamalarını doğru yapın
3. **Error Handling**: Tüm blockchain işlemlerinde kapsamlı hata yönetimi uygulayın
4. **Private Key Güvenliği**: Private key'leri asla kod içinde saklamayın, environment variables veya güvenli key management sistemleri kullanın
5. **Rate Limiting**: Provider'ınızın rate limit'lerini göz önünde bulundurun
6. **Monitoring**: Production ortamında transaction'ları ve contract event'lerini sürekli izleyin
7. **Documentation**: Web3.py ve Ethereum dokümantasyonlarını düzenli takip edin

### Daha Fazla Öğrenin

- 📚 [Web3.py Resmi Dokümantasyon](https://web3py.readthedocs.io/)
- 🔗 [Ethereum Developer Resources](https://ethereum.org/en/developers/)
- 🎓 [Solidity by Example](https://solidity-by-example.org/)
- 🛠️ [OpenZeppelin Contracts](https://docs.openzeppelin.com/contracts/)

Web3.py ile Ethereum blockchain üzerinde geliştirme yapmak artık çok daha kolay! Bu bilgilerle kendi dApp'lerinizi, trading botlarınızı veya blockchain entegrasyonlarınızı geliştirebilirsiniz. Önemli olan pratik yapmak ve sürekli öğrenmeye devam etmek.

Happy coding! 🚀✨
