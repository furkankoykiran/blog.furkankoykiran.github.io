---
title: "Ethereum Smart Contract Güvenliği: Best Practices ve Denetim Süreçleri"
description: "Reentrancy saldırılarını önleme, güvenli Access Control, Overflow koruması ve profesyonel Smart Contract Audit teknikleri üzerine kıdemli mühendis rehberi."
date: "2024-03-15"
categories: [Blockchain, Security]
tags: [ethereum, smart-contracts, security, solidity, audit, reentrancy, defi]
image:
  path: "/assets/img/posts/ethereum-smart-contract-security.png"
  alt: "Ethereum Smart Contract Security Audit"
---

## Giriş: Kod Kanundur, Ama Hatalar Pahalıdır

Web2 dünyasında bir hata yaptığınızda "rollback" yaparsınız. Web3'te ise 600 milyon dolarlık bir hack (Poly Network) ile tarih kitaplarına geçersiniz. Smart Contract güvenliği, "olsa güzel olur" değil, "olmazsa olmaz" bir gerekliliktir.

Bir Senior Blockchain Engineer olarak yüzlerce kontrat inceledim. Gördüğüm en büyük hata, güvenliğin proje sonunda yapılan bir "checklist" olarak görülmesidir. Oysa güvenlik, mimari tasarımın bir parçası olmalıdır. Bugün, DAO Hack'ten bu yana en çok can yakan zafiyetleri ve bunlardan nasıl korunacağımızı, teorik değil **pratik** kod örnekleriyle inceleyeceğiz.

![Ethereum Security Vulnerabilities](/assets/img/posts/web3-smart-contract-architecture.png)

## 1. Reentrancy: En Klasik Düşman

DAO hack'inin başrol oyuncusu Reentrancy, bir kontratın dışarıya yaptığı çağrının (external call), çağrı tamamlanmadan geri dönerek (re-enter) fonksiyonu tekrar tetiklemesiyle oluşur.

### Yanlış Desen
```solidity
// Hatalı Kod: Bakiye düşmeden para gönderiliyor
mapping(address => uint) public balances;

function withdraw() public {
    uint amount = balances[msg.sender];
    require(amount > 0);
    
    // Zafiyet: State update (bakiye sıfırlama) işlemden SONRA yapılıyor
    (bool sent, ) = msg.sender.call{value: amount}("");
    require(sent, "Transfer failed");
    
    balances[msg.sender] = 0;
}
```

Saldırgan, `receive()` fonksiyonunda tekrar `withdraw()` çağırarak bakiyesi sıfırlanmadan havuzu boşaltabilir.

### Doğru Desen: Checks-Effects-Interactions
Her zaman önce kontrolleri yapın, sonra durumu güncelleyin, en son dış etkileşime girin.

```solidity
function withdrawSafe() public nonReentrant {
    uint amount = balances[msg.sender];
    require(amount > 0); // Check
    
    balances[msg.sender] = 0; // Effect (Önce bakiye sıfırla)
    
    (bool sent, ) = msg.sender.call{value: amount}(""); // Interaction
    require(sent, "Transfer failed");
}
```
Özellikle OpenZeppelin `ReentrancyGuard` kullanmak, bu kontrolü `nonReentrant` modifier'ı ile standartlaştırır.

![Smart Contract Access Control](/assets/img/posts/api-security-best-practices.jpg)

## 2. Access Control: Tek Admin Yetmez

Sıklıkla `onlyOwner` kullanıyoruz ama karmaşık protokollerde bu yetersiz kalır. Bir kontratın admininin private key'i çalınırsa ne olur? Tüm protokol çöker. "Multi-signature" cüzdanlar (Gnosis Safe) ve "Role-Based" yetkilendirme standarttır.

```solidity
// OpenZeppelin AccessControl Kullanımı
bytes32 public constant MINTER_ROLE = keccak256("MINTER_ROLE");

constructor() {
    _grantRole(DEFAULT_ADMIN_ROLE, msg.sender);
    _grantRole(MINTER_ROLE, msg.sender);
}

function mint(address to, uint256 amount) public onlyRole(MINTER_ROLE) {
    _mint(to, amount);
}
```
Bu sayede, mint yetkisini bir adrese, pause yetkisini başka bir adrese verebilir ve riski dağıtabilirsiniz.

## 3. Integer Overflow/Underflow

Solidity 0.8.0 öncesinde `SafeMath` kütüphanesi zorunluydu. Örneğin `uint8(255) + 1` işlemi sonucu `0` dönerdi ve bu felaketlere (token basma bugları) yol açardı. Solidity 0.8+ ile bu kontroller dilin içine gömüldü (built-in).

```solidity
function uncheckedMath() public pure {
    // Gas tasarrufu için manuel check kapatılabilir
    unchecked {
        uint256 i = 0;
        i--; // Underflow -> 2^256 - 1
    }
}
```
> **Senior Tip:** Döngü sayaçlarında (`for (uint i=0; i<length; ++i)`) taşma ihtimali yoksa `unchecked` bloğu kullanmak gas maliyetini önemli ölçüde düşürür.

## 4. Randomness: Blokhash Tahmin Edilebilir

Kumar veya piyango uygulamalarında `block.timestamp` veya `blockhash` kullanmak intihardır. Miner'lar bloğun zaman damgasını veya hash'ini manipüle edebilir. Gerçek rastgelelik için **Chainlink VRF (Verifiable Random Function)** kullanmalısınız.

```solidity
// Yanlış: Miner manipülasyonuna açık
uint256 random = uint256(keccak256(abi.encodePacked(block.timestamp, msg.sender)));
```

## 5. Delegatecall ve Storage Collision: Tehlikeli Güç

Proxy pattern'lerin temel taşı olan `delegatecall`, bir kontratın başka bir kontratın kodunu **kendi storage bağlamında** çalıştırmasını sağlar. Bu çok güçlüdür ancak "Storage Collision" riskini doğurur.

Eğer Proxy kontratınızdaki değişkenlerin sırası ile Logic kontratınızdaki değişkenlerin sırası birebir eşleşmezse, logic kontratı yanlış slot'u güncelleyerek Proxy'nin `owner` değişkenini veya kritik mapping'lerini bozabilir.

### Hatalı Implementasyon
```solidity
// Proxy Storage
address implementation;
address owner; // Slot 1

// Logic Storage
address owner; // Slot 0 - Çakışma!
address implementation;
```

Bu hatayı önlemek için **EIP-1967** (Unstructured Storage) standardını ve OpenZeppelin'in `Upgradeable` kütüphanelerini kullanmak şarttır. Manuel proxy yazmaya çalışmak, çoğu zaman felaketle sonuçlanır.

![Smart Contract Audit Tools](/assets/img/posts/web3py-developer-workflow.png)

## 6. Denetim (Audit) Süreci ve Araçlar

Kodunuzu yazdıktan sonra "Mainnet"e deploy etmeden önce mutlaka statik analiz araçlarından geçirmelisiniz.

- **Slither:** Python tabanlı bu araç, kodunuzdaki yaygın hataları (reentrancy, uninitialized variables) saniyeler içinde bulur. CI/CD pipeline'ınıza mutlaka ekleyin.
- **Echidna:** Fuzzing (rastgele veri) testi yaparak, kodunuzun beklenmedik inputlar karşısında nasıl davrandığını simüle eder.
- **Foundry:** Modern Solidity geliştiricilerinin favorisi. `forge test` ile Solidity içinde unit test yazabilirsiniz.

### Pre-Deployment Checklist
1.  **Unit Tests:** %100 Branch Coverage hedefleyin.
2.  **Static Analysis:** Slither raporundaki tüm "High" ve "Medium" uyarıları çözün.
3.  **Testnet:** Kontratı Sepolia veya Goerli'de en az 1 hafta test edin.
4.  **Professional Audit:** Eğer kullanıcılardan para toplayacaksanız (DeFi, ICO), CertiK veya Hacken gibi firmalardan denetim alın.

## 7. Cross-Chain Bridge Riskleri

Ronin ve Poly Network hack'lerinin gösterdiği gibi, en zayıf halka genellikle bridge kontratlarıdır.

- **Validator Güvenliği:** Doğrulayıcıların private key'leri güvende mi? (Ronin hack: 5/9 validatör ele geçirildi)
- **Mesaj İmzalama:** Zincirler arası mesajlar "Replay Attack"a karşı korumalı mı? (Nonce kontrolü şart)
- **Likidite Havuzu Yetkileri:** Bridge admini tüm havuzu boşaltabilir mi?

Eğer kendi köprünüzü yazacaksanız, [Cross-Chain Bridge Geliştirme](/blockchain/interoperability/2024/07/31/cross-chain-bridge-gelistirme/) yazımda anlattığım "Lock & Mint" mekanizmasındaki imza doğrulama adımlarına özellikle dikkat etmelisiniz.

## 8. Incident Response Plan: Hata Olduğunda Ne Yapmalı?

En iyi güvenlik önlemleri bile %100 garanti sağlamaz. Bir saldırı olduğunda "Pausable" kontratlarınız (devre kesici) cankurtaran simidinizdir.

1.  **izleme:** Tenderly gibi araçlarla şüpheli işlemleri anında tespit edin.
2.  **Müdahale:** Multisig cüzdanınızla protokolü `pause()` moduna alın.
3.  **İletişim:** Kullanıcıları Twitter ve Discord üzerinden şeffaf şekilde bilgilendirin.
4.  **Bounty:** Saldırgana "Whitehat" olması için teklif götürün (genelde %10 ödül verilir).

## Teknik Sözlük (Glossary)

- **Reentrancy:** Bir fonksiyonun dışarıdan manipüle edilerek tekrar tekrar çağrılması.
- **Checks-Effects-Interactions:** Güvenli kod yazım deseni; önce kontrol, sonra işlem, en son dış çağrı.
- **Fuzzing:** Yazılıma rastgele ve geçersiz veriler göndererek hataları bulma yöntemi.
- **Access Control:** Kimin hangi fonksiyonu çağırabileceğini belirleyen yetkilendirme sistemi.
- **Honeypot:** Saldırganları tuzağa düşürmek için bilerek savunmasız bırakılmış gibi görünen kontratlar.

## Sonuç: Paranoyak Olun

Blokzincirde "Hotfix" yoktur. Deploy ettiğiniz an kodunuz sonsuza kadar oradadır. Bu yüzden güvenlik konusunda paranoyak olmalısınız.

Güvenli kontratlar geliştirdikten sonra, bu sağlam altyapıyı [Polygon Ağı Mimarisi](/blockchain/scaling/2024/06/05/polygon-network-architecture-deep-dive/) üzerinde düşük maliyetle çalıştırabilir veya [Flash Loan Arbitraj Botu](/defi/smart-contracts/2024/04/11/building-flash-loan-arbitrage-bot/) yazarken bu güvenlik prensiplerini uygulayabilirsiniz.
