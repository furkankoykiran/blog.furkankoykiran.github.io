---
title: "Aave ve Flash Loan ile Arbitraj Botu: Sıfırdan Production'a"
description: "Aave Flash Loan kullanarak teminatsız kredi çekme, Uniswap ve SushiSwap arasında arbitraj yapma ve Solidity ile bot geliştirme rehberi."
date: "2024-04-11"
categories: [DeFi, Smart-Contracts]
tags: [flash-loans, arbitrage, aave, solidity, ethereum, mev, defi]
image:
  path: "/assets/img/posts/flash-loan-arbitrage-simplified-diagram.png"
  alt: "Flash Loan Arbitraj Mekanizması"
---

## Giriş: DeFi'nin Kara Büyüsü "Flash Loans"

Geleneksel finansta milyon dolarlık bir kredi çekmek için ipotek, kefil ve aylar süren bürokrasi gerekir. DeFi'de ise aynı parayı çekmek 13 saniye (bir Ethereum bloğu) sürer ve teminat gerekmez. Tek şart: **Parayı aynı işlem (transaction) içinde geri ödemek.**

Bir senior smart contract mühendisi için Flash Loan, sadece bir kredi aracı değil; sermaye verimliliğini maksimize eden kompleks bir "atomik" işlemdir. Bu rehberde, Aave kullanarak nasıl sermayesiz arbitraj yapabileceğinizi ve MEV (Miner Extractable Value) dünyasındaki rekabeti inceleyeceğiz.

![Flash Loan Transaction Workflow](/assets/img/posts/flash-loan-transaction-workflow.png)

## Flash Loan Mimarisi: "Atomik" İşlem Nedir?

Flash Loan'ın sihiri "Atomicity" (Bölünemezlik) ilkesinde yatar. İşlemlerinizin hepsi ya gerçekleşir ya da hiçbiri gerçekleşmemiş sayılır (revert).

1.  **Borrow:** Aave havuzundan 1.000.000 USDT çek.
2.  **Action:** Bu parayı Uniswap'ta ETH'ye çevir, SushiSwap'ta tekrar USDT'ye sat.
3.  **Repay:** Aave'ye 1.000.900 USDT (ana para + %0.09 fee) geri öde.
4.  **Profit:** Kalan farkı (örneğin 500 USDT) cüzdanına at.

Eğer 3. adımda parayı geri ödeyemezseniz, Ethereum Sanal Makinesi (EVM) tüm işlemi iptal eder. Sanki hiç kredi çekmemişsiniz gibi, sadece işlem ücretini (gas) ödersiniz.

## Solidity ile Arbitraj Kontratı

Aave protokolü ile konuşmak için `IFlashLoanReceiver` arayüzünü implement etmeniz gerekir. Kontratınızın kalbi `executeOperation` fonksiyonudur; Aave parayı gönderdiğinde bu fonksiyonu tetikler.

```solidity
// Senior Notu: Prodüksiyonda OpenZeppelin kütüphanelerini statik import edin.
pragma solidity ^0.8.10;

import { FlashLoanSimpleReceiverBase } from "@aave/core-v3/contracts/flashloan/base/FlashLoanSimpleReceiverBase.sol";
import { IPoolAddressesProvider } from "@aave/core-v3/contracts/interfaces/IPoolAddressesProvider.sol";

contract ArbitrageBot is FlashLoanSimpleReceiverBase {
    
    constructor(address _addressProvider) 
        FlashLoanSimpleReceiverBase(IPoolAddressesProvider(_addressProvider)) {}

    function executeOperation(
        address asset, 
        uint256 amount, 
        uint256 premium, 
        address initiator, 
        bytes calldata params
    ) external override returns (bool) {
        
        // 1. Arbitraj Fonksiyonunu Çağır (Uniswap -> SushiSwap)
        uint256 profit = _makeArbitrage(asset, amount);
        
        // 2. Krediyi Geri Öde (Ana Para + Premium)
        uint256 amountOwed = amount + premium;
        IERC20(asset).approve(address(POOL), amountOwed);
        
        return true;
    }
}
```

![Flash Loan Attack Analysis Framework](/assets/img/posts/flash-loan-attack-analysis-framework.png)

## Arbitraj Mantığı: Fırsatı Yakalamak

Kontrat sadece parayı alır, çevirir ve geri öder. Asıl zeka, "Nerede fırsat var?" sorusunun cevabındadır. Dex'ler (Merkeziyetsiz Borsalar) arasındaki fiyat farkları genellikle "Price Impact" veya ani volatilite sırasında oluşur.

**Senior Stratejisi:** İki DEX arasındaki fiyat farkını hesaplarken sadece spot fiyatı değil, "Effective Price"ı (işlem hacmi sonrası oluşacak fiyatı) baz almalısınız. Aksi takdirde "slippage" yüzünden kar beklerken zarar edebilirsiniz.

```javascript
// Off-chain monitoring örneği (Node.js)
const checkArbitrage = async () => {
    const uniPrice = await uniswap.getAmountsOut(amountIn, [tokenA, tokenB]);
    const sushiPrice = await sushiswap.getAmountsOut(amountIn, [tokenA, tokenB]);
    
    const profit = sushiPrice[1].sub(uniPrice[1]);
    
    // Gas maliyeti ve Flash Loan fee (%0.09) düşüldükten sonra kar var mı?
    if (profit.gt(gasCost.add(flashLoanFee))) {
        console.log("Fırsat Bulundu! İşlem başlatılıyor...");
        await arbitrageBot.executeOperation(...);
    }
}
```

## Off-Chain İzleme: Fırsat Avcısı Botlar

Akıllı kontratlar kendi kendine tetiklenemez. Onları dışarıdan bir "EOA" (Externally Owned Account) tetiklemelidir. Bu nedenle Go, Rust veya Node.js ile yazılmış, mempool'u (bekleyen işlemler havuzu) saniyenin altında hızlarla tarayan botlara ihtiyacınız vardır.

Bu botlar, büyük bir satış işlemini (whale sell) gördüğü anda fiyatın düşeceğini öngörerek aynı blok içerisinde işlem yapmaya çalışır.

![Cross-Exchange Arbitrage Strategy](/assets/img/posts/cross-exchange-arbitrage-strategy.png)

## MEV ve Dark Forest: Rekabetin Karanlık Yüzü

Ethereum mempool'u, herkesin görebildiği bir savaş alanıdır (Dark Forest). Eğer karlı bir işlem gönderirseniz, MEV (Miner Extractable Value) botları bunu görür, kopyalar ve daha yüksek gas ücreti ödeyerek (front-running) sizin işleminizin önüne geçer. Sonuç? Onlar parayı kazanır, siz ise sadece gas ücreti ödeyip başarısız işlemle kalırsınız.

### Flashbots: Güvenli Liman

Bu "predatory" (yırtıcı) botlardan korunmanın yolu işlemlerinizi public mempool yerine, doğrudan "block builder"lara göndermektir. **Flashbots**, işlemlerinizi gizli bir kanaldan ileterek front-running riskini ortadan kaldırır. Ayrıca, işleminiz başarısız olursa (revert), blok içerisine dahil edilmez ve gas ücreti ödemezsiniz.

## Güvenlik: Reentrancy ve Access Control

Flash Loan kontratları, saldırganların hedefi olabilir. `executeOperation` fonksiyonunuzun sadece Aave Lending Pool tarafından çağrıldığından emin olmalısınız.

```solidity
require(msg.sender == address(POOL), "Unauthorized caller");
require(initiator == address(this), "FlashLoan must be initiated by this contract");
```

Ayrıca, `nonReentrant` modifier'ını kullanmak, olası reentrancy saldırılarına karşı ek bir katman sağlar.

![Flash Loan Arbitrage Simplified](/assets/img/posts/flash-loan-arbitrage-simplified-diagram.png)

## Maliyet Analizi: Gas vs Kar

Bir arbitraj işleminin karlı olması için şu formülün pozitif olması gerekir:

`Net Kar = (Satış Geliri - Alış Maliyeti) - (Flash Loan Fee + Execution Gas Cost + Bribe)`

Eğer bu denklem negatif çıkarsa, işlem yapmamalısınız. Flashbots kullanıyorsanız, `Bribe` (madenciye rüşvet) miktarını dinamik olarak ayarlayarak işleminizi garanti altına alabilirsiniz.

## Geliştirme Ortamı: Mainnet Forking

Flash Loan'ları test ağlarında (Goerli, Sepolia) denemek zordur çünkü DEX'lerde yeterli likidite yoktur. Bu yüzden "Mainnet Forking" kullanmalısınız. Hardhat veya Foundry ile ana ağın anlık bir kopyasını (snapshot) yerel bilgisayarınıza indirip, gerçek likidite üzerinde risksiz test yapabilirsiniz.

```bash
# Hardhat ile Mainnet Forking Başlatma
npx hardhat node --fork https://eth-mainnet.alchemyapi.io/v2/YOUR_KEY
```

Bu sayede, cebinizden 1 kuruş çıkmadan milyon dolarlık arbitraj senaryolarını simüle edebilir, Aave ve Uniswap kontratlarının gerçek durumlarıyla etkileşime girebilirsiniz. Bu özellik, özellikle DeFi projelerinde riskli olan "Test in Prod" mantığının güvenli ve profesyonel bir alternatifidir. İşte profesyonel bir `hardhat.config.js` ayarı:

```javascript
// hardhat.config.js
module.exports = {
  networks: {
    hardhat: {
      forking: {
        url: "https://eth-mainnet.alchemyapi.io/v2/KEY",
        blockNumber: 14390000 // Belirli bir blokta dondurmak test tutarlılığı sağlar
      }
    }
  }
}
```

**Risk Uyarısı:** Flash Loan işlemleri finansal olarak risksiz görünse de (para kaybedemezsiniz, sadece gas ödersiniz), akıllı kontratınızdaki mantık hataları saldırıya açık olabilir. Kontratınızda asla "withdraw" fonksiyonunu korumasız bırakmayın; aksi takdirde arbitraj karını çekerken botunuz boşaltılabilir.

**Senior Pro Tip:** Flash Loan işlemlerinde gas maliyeti yüksektir. İşlem başarısız olsa bile gas ödersiniz. Bu yüzden `checkProfitability` fonksiyonunu mutlaka off-chain çağırın ve on-chain işlem sırasında `gasLimit` değerini optimize edin. Gereksiz gas harcaması, arbitraj karınızı eritebilir.

## Teknik Sözlük (Glossary)

- **Atomic Transaction:** Tamamı ya başarılı olan ya da tamamen başarısız sayılan işlem bütünü.
- **MEV (Maximal Extractable Value):** Madencilerin veya botların işlem sırasını değiştirerek elde ettiği ekstra kar.
- **Slippage:** Beklenen fiyat ile işlemin gerçekleştiği fiyat arasındaki fark (kayma).
- **Front-running:** Bir işlemi görüp, daha yüksek gas vererek önüne geçme saldırısı.
- **EOA (Externally Owned Account):** Private key ile kontrol edilen, akıllı kontrat olmayan standart cüzdan adresi.
- **Arbitraj:** Fiyat farklarından risksiz kar elde etme stratejisi.
- **Liquidity Pool:** DEX'lerde token takası için kilitlenen fon havuzu.

## Sonuç: DeFi Lego Blokları

Flash Loan, DeFi'nin "Money Lego" kavramının en güçlü örneğidir. Sermayesi olmayan bir geliştiricinin, milyar dolarlık fonlarla işlem yapmasına olanak tanır. Ancak unutmayın, büyük güç büyük sorumluluk (ve güvenlik riski) getirir.

Botunuzu canlıya almadan önce kontratınızı test etmek için [Smart Contract Test Stratejileri](/smart-contracts/testing/2024/08/19/smart-contract-testing-hardhat-ethers/) yazımızı inceleyebilir, düşük işlem ücretleri ve yüksek hız için arbitraj stratejilerinizi [Polygon Ağı](/blockchain/infrastructure/2024/06/05/polygon-network-architecture-deep-dive/) üzerinde kurgulayabilirsiniz.
