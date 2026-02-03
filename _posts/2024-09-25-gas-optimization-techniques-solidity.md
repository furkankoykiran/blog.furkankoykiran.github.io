---
title: "Solidity Akıllı Kontratlarında Gas Optimizasyonu Teknikleri"
description: "Solidity akıllı kontratları için ileri seviye gas optimizasyonu teknikleri. Storage layout, assembly optimizasyonu, EVM opcode'ları ve maliyet etkin blockchain geliştirme."
date: "2024-09-25 16:00:00 +0300"
categories: [Smart Contracts, Solidity]
tags: [solidity, ethereum, gas-optimization, evm, smart-contracts, blockchain, opcodes, storage]
image:
  path: /assets/img/posts/evm-gas-opcodes-how-it-works.png
  alt: "Gas Optimization Techniques for Solidity"
---

## Giriş: Neden Gas Optimizasyonu?

Ethereum ve EVM tabanlı ağlarda akıllı kontrat geliştiren bir mühendis için **Gas**, sadece bir maliyet birimi değil, aynı zamanda yazılımınızın "verimlilik skoru"dur. Bir web uygulamasında CPU veya RAM kullanımını optimize etmek genellikle saniyeler bazında hız kazandırırken, Solidity'de yapılan tek bir verimsiz işlem, son kullanıcının ödeyeceği faturayı katlayabilir veya kontratın tamamını kullanılamaz hale getirebilir.

Bir senior mühendis olarak, gas optimizasyonunu "erken optimizasyon şeytanı" (premature optimization) olarak görmüyorum. Aksine, Solidity dünyasında bu bir mimari gerekliliktir. Kodunuz ne kadar güvenli olursa olsun, eğer bir transfer işlemi 50 dolar tutuyorsa, o kontrat "ölü" doğmuş demektir. 

Bu maliyetleri [Ethereum 2.0 Staking]({% post_url 2024-10-29-understanding-ethereum-2-staking %}) gibi büyük ölçekli mekanizmalarda nasıl yönetildiğini anlamak, vizyonunuzu genişletecektir. Bu rehberde, EVM'in (Ethereum Virtual Machine) derinliklerine inecek, opcode seviyesinde nasıl tasarruf edebileceğimizi ve gerçek hayat senaryolarında hangi "hacetlerin" bizi kurtaracağını inceleyeceğiz.

![EVM Gas Opcode Mantığı](/assets/img/posts/evm-gas-opcodes-how-it-works.png)

## Ethereum'da Gas Mekanizmasını Anlamak

Gas, ağdaki hesaplama gücünün ölçü birimidir. Karmaşıklık arttıkça gas maliyeti de artar. Ancak her işlem aynı maliyette değildir. EVM tarafında işlemler kabaca dört ana kategoriye ayrılır:

1.  **Storage İşlemleri (En Pahalı)**: Veri yazma (SSTORE) ve okuma (SLOAD). 5.000 ile 20.000 gas arası maliyet çıkarabilir.
2.  **Hesaplama (Ucuz)**: Toplama, çarpma gibi aritmetik işlemler. Genellikle 3-10 gas arasıdır.
3.  **Memory İşlemleri (Dinamik)**: Geçici veri saklama. Ucuz başlar ancak memory genişledikçe maliyet katlanarak artar.
4.  **Harici Çağrılar (Orta/Pahalı)**: Diğer kontratlarla konuşmak.

**Senior Notu:** Gas fiyatları (Gwei) ağ yoğunluğuna göre değişse de, işlemin gas miktarı sabittir. Bu yüzden biz "fiyatı" değil, "miktarı" optimize etmekle sorumluyuz.

### Gas Maliyet Bileşenleri ve EVM Opcodes

EVM, yazdığınız Solidity kodunu düşük seviyeli opcode'lara dönüştürür. Örneğin, `uint256 a = b + c;` kodu arka planda `ADD` opcode'unu çalıştırır.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract GasMechanics {
    // Storage değişkeni - PAHALI
    uint256 public total; // İlk yazma 20,000 gas, güncelleme 5,000 gas

    function calc(uint256 x, uint256 y) public {
        // Memory değişkeni - UCUZ
        uint256 result = x + y; // ADD işlemi ~3 gas
        
        // Storage okuma - MODERATE
        uint256 currentTotal = total; // SLOAD ~100-2,100 gas
        
        // Storage yazma
        total = currentTotal + result;
    }
}
```

Burada dikkat edilmesi gereken nokta, verinin "sıcak" (warm) veya "soğuk" (cold) olmasıdır. EIP-2929 sonrası, aynı işlem içinde erişilen veriler "warm" sayılır ve okuma maliyeti 2.100 gas'tan 100 gas'a düşer. Bu yüzden, bir storage değişkenini loop içinde defalarca okumak yerine, loop başında bir memory değişkenine kopyalamak (caching) hayat kurtarır.

![EVM Mimari Şeması](/assets/img/posts/evm-architecture-execution-diagram.png)

## Storage Optimizasyonu: En Büyük Tasarruf Alanı

Storage, Ethereum'un en kıymetli kaynağıdır. Bir veri parçasını blockchain üzerinde sonsuza dek tutmanın maliyeti yüksektir. Ancak akıllıca bir tasarımla bu maliyeti %50-70 oranında azaltabiliriz.

### Variable Packing (Değişken Paketleme)

Solidity'de her storage slotu 32 byte (256 bit) genişliğindedir. Eğer değişkenleriniz bu slotu tam doldurmazsa, EVM yan yana gelen küçük değişkenleri (uint8, uint128, bool vb.) aynı slota paketleyebilir.

```solidity
// KÖTÜ: 3 slot kaplar (60,000 gas)
contract Unoptimized {
    uint256 a; // Slot 0
    uint128 b; // Slot 1
    uint128 c; // Slot 2 (Paketlenebilirdi ama 256'nın arasına girdi)
}

// İYİ: 2 slot kaplar (40,000 gas)
contract Optimized {
    uint128 b; // Slot 0 (16 byte)
    uint128 c; // Slot 0 (16 byte) - b ile aynı slota girdi
    uint256 a; // Slot 1 (32 byte)
}
```

**Kritik Kural:** Değişkenlerinizi boyutlarına göre ardışık tanımlayın. Ancak dikkat; bu teknik sadece *Storage* için geçerlidir. *Memory* veya *Stack* seviyesinde her şey her zaman 32 byte yer kaplar, bu yüzden memory'de küçük tipler kullanmak bazen ekstra opcode (masking) maliyeti getirebilir.

### Constant ve Immutable Kullanımı

Eğer bir değişkenin değeri kontrat deploy edildikten sonra hiç değişmeyecekse, onu asla normal bir değişken olarak tanımlamayın.

- **`constant`**: Değer compile sırasında kodun içine gömülür. Okunması neredeyse bedavadır (~3 gas).
- **`immutable`**: Değer constructor sırasında belirlenir ve bytecode içinde sabitlenir. Normal değişken okumadan (~2100 gas) çok daha ucuzdur (~100 gas).

```solidity
contract Constants {
    uint256 public constant TOTAL_SUPPLY = 1_000_000; // Bytecode'da
    address public immutable OWNER; // Constructor'da set edilir

    constructor() {
        OWNER = msg.sender;
    }
}
```

Bu basit dokunuş, özellikle sık okunan "admin" veya "fee" adresleri için kontratınızın ömür boyu harcayacağı gas miktarını ciddi oranda azaltır.

![EVM Opcode Akışı](/assets/img/posts/evm-opcode-execution-flow.png)

## Hafıza Yönetimi: Memory mi, Calldata mı?

Fonksiyon parametreleri için veri lokasyonu seçimi, özellikle büyük array veya struct'lar ile çalışırken kritik önem taşır. Çoğu geliştirici alışkanlık olarak `memory` anahtar kelimesini kullanır, ancak bu her zaman en iyisi değildir.

### Calldata Kullanımının Gücü

`calldata`, fonksiyonun çağrıldığı "input data" alanını temsil eder ve salt okunur (read-only) bir alandır.

- **`memory`**: Veriyi input alanından kopyalar ve RAM üzerinde yeni bir alan açar (Kopyalama maliyeti + Memory genişleme maliyeti).
- **`calldata`**: Veriyi kopyalamaz, doğrudan input alanından okur (Sıfır kopyalama maliyeti).

Eğer fonksiyon içindeki parametreyi değiştirmeyecekseniz (ki çoğu zaman durum budur), mutlaka `calldata` kullanın. Bu, büyük byte dizilerinde (bytes) veya array'lerde binlerce gas tasarrufu sağlayabilir.

## Hataları Yönetmek: Revert, Require ve Custom Errors

Geleneksel Solidity'de (`0.8.4` öncesi) hata mesajları string olarak saklanırdı: `require(balance > 0, "Yetersiz bakiye");`. Bu string'lerin her biri kontratın bytecode boyutunu artırır ve bir hata fırlatıldığında bu string'in gönderilmesi ciddi gas tüketir.

### Custom Errors (Özel Hatalar)

Yeni versiyonlarda gelen `error` tanımı, bir hash üzerinden çalışır.

```solidity
// KÖTÜ: String maliyeti yüksek
require(x > 10, "Girdi 10'dan buyuk olmali");

// İYİ: Custom error
error InputTooSmall(uint256 provided, uint256 required);

if (x <= 10) {
    revert InputTooSmall(x, 10);
}
```

**Mühendislik Avantajı:** Custom error'lar sadece gas dostu değildir, aynı zamanda parametre alabildikleri için off-chain debugging süreçlerini (örneğin Etherscan üzerinde hatayı görmek) çok daha profesyonel hale getirir.

## Mantıksal Optimizasyonlar: Short-Circuiting

Mantıksal operasyonlarda (`&&` ve `||`) Solidity, sonucun kesinleştiği andan itibaren diğer kontrolleri bırakır. Buna "short-circuiting" denir.

```solidity
// Eğer isWhitelisted false ise, checkBalance asla okunmaz (Gas tasarrufu)
if (isWhitelisted(user) && checkBalance(user) > 100) {
    // ...
}
```

**Strateji:** `&&` kullanırken en ucuz ve en çok "false" dönecek ihtimali olan kontrolü en başa koyun. `||` kullanırken ise en ucuz ve en çok "true" dönecek olanı başa alın. Bu sayede gereksiz fonksiyon çağrılarından ve storage okumalarından kaçınmış olursunuz.

![EVM Opcode Gas Tablosu](/assets/img/posts/evm-opcodes-gas-cost-table.jpg)

## İleri Seviye: Assembly (Yul) ile Optimizasyon

Bazen Solidity'nin yüksek seviyeli abstraction'ları beklediğimiz performansı veremez. Bu noktada "Yul" adı verilen, Solidity'nin içine gömülebilen düşük seviyeli dili (Inline Assembly) kullanabiliriz.

**Neden Assembly?**
Solidity compiler'ı kodu optimize etmeye çalışsa da bazen gereksiz "safety checks" ekler. Örneğin bir array'in sınırlarını kontrol etmek için ekstra opcode'lar çalıştırır. Eğer bu kontrollerin gereksiz olduğundan eminseniz, Assembly ile bu maliyeti bypass edebilirsiniz.

```solidity
function isContract(address account) public view returns (bool) {
    uint256 size;
    assembly {
        // Kontratın bytecode boyutunu kontrol eder
        size := extcodesize(account)
    }
    return size > 0;
}
```

Ancak unutmayın: "Büyük güç, büyük sorumluluk getirir." Assembly kullanmak kontratınızı hata yapmaya çok daha açık hale getirir ve readable (okunabilir) olmaktan uzaklaştırır. Sadece performansın kritik olduğu "hot-path" fonksiyonlarda kullanılmalıdır.

## Döngülerde (Loops) Dikkat Edilmesi Gerekenler

Döngüler, Solidity'de "gas limit reached" hatasının en yaygın sebebidir. 

1.  **Dışarıda Cache'leyin:** `for (uint i=0; i < arr.length; i++)` yazmak yerine, length değerini bir memory değişkenine sabitleyin. Her iterasyonda storage'dan length okumaktan kurtulursunuz.
2.  **`unchecked` kullanımı:** Döngü sayacının (`i++`) overflow yapmayacağından eminseniz, `unchecked { i++; }` bloğunu kullanın. Bu, her artırımda yapılan matematiksel kontrolü kapatarak iterasyon başına ~30-40 gas tasarrufu sağlar.

## Teknik Sözlük (Glossary)

- **EVM (Ethereum Virtual Machine):** Ethereum akıllı kontratlarının çalışma ortamı olan sanal makine.
- **Opcode:** EVM'in anladığı en temel işlem komutları (ADD, SLOAD, SSTORE vb.).
- **SSTORE / SLOAD:** State değişkenlerini storage'a yazma ve storage'dan okuma işlemleri.
- **Warm / Cold Access:** EIP-2929 ile gelen, veriye daha önce erişilip erişilmediğine göre değişen maliyet sistemi.
- **Yul:** Solidity ile opcode'lar arasında köprü kuran düşük seviyeli programlama dili.

## Son Mühendislik Notları: Dengeyi Bulmak

Gas optimizasyonu, güvenlik ve okunabilirlik arasında hassas bir dengedir. Kodunuzu sırf 1.000 gas tasarruf etmek için anlaşılamaz hale getirmeyin. "Optimize edilmiş karmaşık kod", yarın bir bug çıktığında size çok daha pahalıya mal olabilir.

Önceliğiniz her zaman mimari temizlik olmalı, ardından storage paketleme ve `calldata` gibi maliyetsiz optimizasyonları yapmalı, Assembly gibi "ağır silahları" ise en sona bırakmalısınız. Unutmayın ki en iyi optimizasyon, hiç yapılmayan (gereksiz olan) işlemdir.

Kendi projelerimde, özellikle DeFi protokollerinde bu teknikleri uygulayarak kullanıcı maliyetlerini %30'a varan oranlarda düşürebiliyoruz. Bu, platformun rekabet gücünü doğrudan artıran bir mühendislik başarısıdır. Terminal başında geçirdiğiniz her dakika, kullanıcılarınızın cebinde kalan birer ethereuma dönüşebilir.
