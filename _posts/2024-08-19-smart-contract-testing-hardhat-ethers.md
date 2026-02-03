---
title: "Hardhat ve Ethers.js ile Akıllı Kontrat Test Otomasyonu"
description: "Akıllı kontrat geliştirmede test süreçlerinin önemi. Hardhat ve Ethers.js kullanarak unit testler, entegrasyon testleri ve güvenli Solidity geliştirme pratikleri."
date: "2024-08-19"
categories: [Blockchain, Testing]
tags: [hardhat, ethers, solidity, testing, blockchain, web3, smart-contracts]
image:
  path: "/assets/img/posts/smart-contract-testing-tools.png"
  alt: "Hardhat ve Ethers.js Test Araçları"
---

## Giriş: Blockchain Dünyasında "Geri Al" Tuşu Yoktur

Geleneksel web geliştirmede bir bug çıktığında genellikle bir "hotfix" yayınlar, sunucuyu restart eder ve yolunuza devam edersiniz. Ancak akıllı kontrat dünyasında işler böyle yürümüyor. Kontrat bir kez deploy edildikten sonra (eğer karmaşık bir proxy mimariniz yoksa) kodunuz taşa kazınmış demektir. Milyonlarca dolarlık DeFi "hack" olaylarının çoğunun arkasında, aslında basit bir test senaryosuyla yakalanabilecek mantık hataları yatıyor.

Bir senior blockchain mühendisi için test yazmak bir tercih değil, bir hayatta kalma mekanizmasıdır. Kodunuzun sadece "çalışması" yetmez; ağ üzerindeki en kötü niyetli aktörlere karşı "direnmesi" de gerekir. Özellikle [Cross-chain Bridge Mimarisi](/blockchain/infrastructure/2024/07/31/cross-chain-bridge-gelistirme/) gibi yüksek riskli projelerde veya [Solana Rust Geliştirme](/blockchain/backend/2024/07/08/solana-program-development-rust/) süreçlerinde test otomasyonu, projenin başarısı ile felaketi arasındaki ince çizgidir.

![Akıllı Kontrat Mimari ve Güvenlik](/assets/img/posts/ethereum-smart-contract-security.png)

## Araç Setimiz: Hardhat, Ethers ve Chai Üçlüsü

Modern bir test altyapısı kurarken tekerleği yeniden icat etmiyoruz. Sektör standardı haline gelmiş araçların sunduğu ekosistemden faydalanmak, hem hız hem de güvenlik sağlar.

### 1. Hardhat: Geliştirme Ortamı
Hardhat sadece bir test koşucu (test runner) değil, aynı zamanda size lokal bir Ethereum ağı, stack trace'ler ve gelişmiş hata ayıklama yetenekleri sunan bir orkestra şefidir. `npx hardhat node` komutuyla ayağa kalkan bu yerel ağ, kontratlarımızın anlık davranışlarını izlemek için en iyi laboratuvarımızdır.

### 2. Ethers.js: Blockchain Etkileşim Katmanı
Kontratlarımızla JavaScript/TypeScript üzerinden konuşmamızı sağlayan köprüdür. Web3.js'e göre daha hafif, modüler ve güvenli yapısı sayesinde senior geliştiriciler arasında daha popülerdir. Provider, Signer ve Contract abstraction'ları ile temiz bir kod yazmamıza olanak tanır.

### 3. Chai ve Mocha: Test İskeleti
Javascript dünyasının klasik ikilisi. Ancak blockchain dünyasında Hardhat, Chai'yi `@nomicfoundation/hardhat-chai-matchers` eklentisiyle güçlendirir. Bu sayede "Bu transaction şu hatayla revert olmalı" veya "Bu adres şu kadar ETH kazanmalı" gibi blockchain'e özgü asenkron ifadeleri kolayca test edebiliriz.

**Mühendislik Notu:** Testlerinizi TypeScript ile yazmanızı şiddetle öneririm. `typechain` eklentisi sayesinde kontratlarınızın ABI'larından gelen tip tanımlamaları, IDE tarafında size muazzam bir oto-tamamlama ve hata yakalama kabiliyeti sunar.

![Akıllı Kontrat Geliştirme Akışı](/assets/img/posts/web3py-developer-workflow.png)

## Kurulum ve Proje Yapılandırması

Hardhat ile sıfırdan bir proje başlatmak oldukça basittir ancak profesyonel bir yapı için bazı ek eklentilere ihtiyaç duyarız.

```bash
# Proje dizinini oluştur ve başlat
mkdir hardhat-testing && cd hardhat-testing
npm init -y

# Hardhat ve gerekli toolbox'ı kur
npm install --save-dev hardhat @nomicfoundation/hardhat-toolbox

# Projeyi interaktif olarak başlat
npx hardhat init
# "Create a JavaScript project" seçeneğini işaretleyin
```

Burada en kritik dosyamız `hardhat.config.js` dosyasıdır. Özellikle `gasReporter` ve `solidity-coverage` eklentilerini buraya dahil etmek, yazdığımız testlerin kalitesini ölçmemiz için hayati önem taşır.

## Birim Testler (Unit Testing): İzole ve Hızlı

Birim testlerin ana amacı, kontrat içindeki her bir fonksiyonun kendi başına doğru çalıştığından emin olmaktadır. Bir senior mühendisin birim test yazarken izlediği en önemli kural **izolasyondur**.

### Örnek Senaryo: Basit Bir Vault Kontratı
Diyelim ki kullanıcıların ETH yatırıp 7 gün sonra çekebildiği bir `TimeLock` kontratımız var. Bu kontrat üzerinde şu soruların cevabını aramalıyız:
1. Para yatırılabiliyor mu? `balanceOf` güncelleniyor mu?
2. 7 gün dolmadan çekilmeye çalışılırsa kontrat doğru hatayı (`revertedWith`) veriyor mu?
3. Zaman dolduğunda transfer gerçekleşiyor mu?

```javascript
const { expect } = require("chai");
const { time } = require("@nomicfoundation/hardhat-network-helpers");

describe("Vault Denemeleri", function () {
  it("7 günde önce çekim yapılamamalı", async function () {
    const [owner] = await ethers.getSigners();
    const Vault = await ethers.getContractFactory("Vault");
    const vault = await Vault.deploy();

    await vault.deposit({ value: ethers.parseEther("1.0") });
    
    // Çekim yapmayı dene ve hata almayı bekle
    await expect(vault.withdraw()).to.be.revertedWith("Henuz kilitli");
  });
});
```

**Senior İpucu:** `beforeEach` içinde sürekli kontrat deploy etmek bazen testleri yavaşlatabilir. Hardhat'in `loadFixture` özelliğini kullanarak ağın o anki durumunun bir snapshot'ını alabilir ve testler arasında bu snapshot'lar üzerinden çok daha hızlı ilerleyebilirsiniz.

![Akıllı Kontrat Mimarisi](/assets/img/posts/web3-smart-contract-architecture.png)

## Entegrasyon Testleri ve Mainnet Forking

Birim testler temel mantığı doğrularken, entegrasyon testleri sistemin bütünüyle olan ilişkisini test eder. Gerçek bir DeFi projesinde genellikle kendi kontratlarınızın Uniswap, Aave veya Chainlink gibi harici protokollerle konuşması gerekir. 

Bu noktada Hardhat'in en güçlü özelliklerinden biri olan **Mainnet Forking** devreye girer. Bu özellik sayesinde, gerçek Ethereum ana ağının belirli bir bloktaki kopyasını lokalinizde çalıştırabilirsiniz.

### Neden Mainnet Forking?
1. **Gerçek Protokoller:** Uniswap havuzlarını veya Chainlink oracle'larını "mock"lamak (sahtesini yapmak) yerine, gerçek likidite ve veriyle test yaparsınız.
2. **Gerçek Bakiyeler:** "Whale" (balina) hesapları impersonate ederek büyük işlemlerin kontratınız üzerindeki etkisini görebilirsiniz.
3. **Senaryo Simülasyonu:** Mevcut bir exploit senaryosunu veya karmaşık bir likvidasyon akışını birebir simüle edebilirsiniz.

```javascript
// hardhat.config.js içinde forking yapılandırması
networks: {
  hardhat: {
    forking: {
      url: "https://eth-mainnet.g.alchemy.com/v2/YOUR_API_KEY",
      blockNumber: 18000000
    }
  }
}
```

Integration test yazarken, kontratlar arası "race condition" (yarış durumu) ihtimallerine ve asenkron çağrıların sırasına çok dikkat etmelisiniz. Özellikle `reentrancy` saldırılarını yakalamak için entegrasyon seviyesinde statik analiz araçlarıyla desteklenmiş testler hayat kurtarıcı olabilir.

## Zaman Yolculuğu (Time Travel) ve Madencilik Kontrolü

Blockchain testlerinde en büyük sıkıntılardan biri, zamanla yarışan kontratları test etmektir. Bir stake kontratında "1 yıl sonraki ödülleri" test etmek için 1 yıl bekleyemeyiz. Hardhat Network bize zamanı manipüle etme gücü verir.

- `time.increase(duration)`: Zamanı istediğiniz kadar ileri sarar.
- `time.setNextBlockTimestamp(timestamp)`: Bir sonraki bloğun vaktini ayarlar.
- `mine(count)`: Belirli sayıda boş blok kazarak blok numarasına dayalı mantıkları tetikleyin.

Bu metodlar, özellikle vesting (kilitli token) ve governance (yönetişim) oylama süreçlerini test ederken senior geliştiricilerin en yakın dostlarıdır.

![Web3 Uygulama Mimarisi](/assets/img/posts/web3-application-architecture.png)

## Verimlilik Analizi: Gas Reporting ve Coverage

Bir test suite'inin kalitesi sadece "geçip geçmediği" ile değil, neleri test ettiği ve ne kadar maliyetli olduğu ile de ölçülür.

### 1. Hardhat Gas Reporter
Akıllı kontratlarda performans, doğrudan ödenen gas miktarıdır. `hardhat-gas-reporter` eklentisi, testleriniz sonucunda her bir fonksiyonun ortalama ne kadar gas tükettiğini size tablo halinde sunar. Bu, bir refactoring sonrasında gas maliyetlerinin artıp artmadığını görmek için kritiktir.

### 2. Solidity Coverage
Koda ne kadar hakimiz? `solidity-coverage` eklentisi, testlerinizin kontratınızdaki tüm satırları ve dallanmaları (branches) kapsayıp kapsamadığını gösterir. Senior seviyesindeki bir projede hedefimiz genelde %100 kapsama değil, kritik iş mantığının (business logic) %100 kapsanmasıdır.

## CI/CD Entegrasyonu: Güvenli Deployment

Testlerin sadece lokalde çalışması yetmez. GitHub Actions veya GitLab CI gibi platformlarda her "pull request" sonrasında testlerin otomatik koşması gerekir.

```yaml
# GitHub Actions Örneği
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - run: npm install
      - run: npx hardhat test
```

## Teknik Sözlük (Glossary)

- **ABI (Application Binary Interface):** Kontratın dış dünya ile nasıl konuşacağını tanımlayan JSON şeması.
- **Provider:** Blockchain ağına bağlantı sağlayan soyutlama katmanı (örn: Alchemy, Infura).
- **Signer:** Bir hesabı temsil eden ve transaction'ları imzalama yetkisi olan obje.
- **Waffle Matchers:** Chai testleri için akıllı kontratlara özel (revert, emit vb.) eklenen yardımcılar.
- **Impersonate Account:** Lokal ağda, private key'i bizde olmayan bir adresten işlem yapma yetisi (Mainnet forking ile kullanılır).
- **Hardhat network helpers:** Zaman manipülasyonu ve blok kontrolü gibi ağ seviyesindeki işlemler için kullanılan kütüphane.

## Son Mühendislik Notları: Testin Ötesi

Testing bir varış noktası değil, bir süreçtir. Makul bir test suite'i, kodunuzun bildiğiniz senaryolarda doğru çalıştığını kanıtlar. Ancak bir senior mühendis olarak şunu unutmamalısınız: "Testler sadece hataların varlığını kanıtlayabilir, yokluğunu değil."

Bu yüzden testlerinizin yanına mutlaka:
- **Formal Verification:** Matematiksel ispat yöntemleri.
- **Slither/Mythril:** Statik analiz araçları.
- **External Audit:** Bağımsız güvenlik denetimleri.

gibi ek katmanlar eklemelisiniz. Ethereum ekosistemi hızla evriliyor ve Hardhat/Ethers ikilisi şu anki en güçlü silahlarımız. Bu silahları doğru kullanmak, sadece güvenli kod yazmanızı değil, aynı zamanda mülakatlarda ve profesyonel projelerde fark yaratmanızı sağlar.

## Gelecek Notu
Fonksiyonel testlerin ötesinde, `fuzzing` (Rastgele veriyle test etme) ve `invariant testing` (Echidna veya Foundry ile) gibi ileri seviye tekniklere de göz atmanızı öneririm. Bir gün o koda milyonlarca dolar emanet edildiğinde, "Keşke daha fazla test yazsaydım" dememek için bugünden elinizi korkak alıştırmayın.
