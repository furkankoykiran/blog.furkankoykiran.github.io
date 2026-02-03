---
title: "Solana Program Geliştirme: Rust ve Anchor ile Üst Düzey Mimari"
description: "Solana ağında yüksek performanslı akıllı kontratlar (programlar) geliştirme rehberi. Rust dili, Anchor frameworkü, hesap modeli ve güvenlik en iyi uygulamaları."
date: "2024-07-08"
categories: [Blockchain, Backend]
tags: [solana, rust, anchor, blockchain, web3, smart-contracts, programming]
image:
  path: "/assets/img/posts/anchor-framework-components.png"
  alt: "Solana Anchor Framework Mimarisi"
---

## Giriş: Neden Solana ve Neden Şimdi?

Blockchain dünyasında hız ve ölçeklenebilirlik denilince akla gelen ilk isim olan Solana, saniyede 65.000'den fazla işlemi (TPS) bir saniyenin altındaki kesinleşme (finality) süresiyle işleyebiliyor. Ancak bu performansın arkasında, Ethereum'un EVM yapısından tamamen farklı, radikal bir mimari yatıyor.

Bir senior mühendis için Solana'da geliştirme yapmak, sadece yeni bir dil öğrenmek değil, "hesap tabanlı" (account-based) ama "durumsuz" (stateless) bir mantıkla düşünmeye başlamaktır. Eğer daha önce [Hardhat ve Ethers.js ile Ethereum Testleri](/blockchain/infrastructure/2024/08/19/smart-contract-testing-hardhat-ethers/) geliştirdiyseniz, Solana'nın test ve geliştirme iş akışındaki radikal farkları daha iyi görebilirsiniz.

![Solana Full Stack Geliştirme Akışı](/assets/img/posts/solana-rust-anchor-full-stack.png)

## Solana Mimarisinin Temelleri: Sealevel ve Paralel İşleme

Solana'yı diğer ağlardan ayıran en büyük fark, işlemlerin seri yerine paralel olarak yürütülebilmesidir. Sealevel runtime, hangi işlemlerin birbiriyle çakışmadığını (farklı hesapları kullandığını) analiz ederek aynı anda binlerce işlemi işleyebilir.

### 1. Hesap Modeli (Account Model)
Solana'da her şey bir hesaptır. Kodlar ("Program" olarak adlandırılır), veriler ve bakiyeler ayrı hesaplarda tutulur.
- **Programlar:** Durum tutmazlar (Stateless). Sadece kod içerirler.
- **Data Accounts:** Verilerin saklandığı hesaplardır. Programlar bu hesaplara okuma/yazma yapar.
- **Signers:** İşlemi imzalayan ve yetki veren hesaplar.

### 2. Proof of History (PoH)
Geleneksel blokzincirlerin aksine, Solana işlemleri zaman damgasıyla işaretleyerek validator'lerin blokları doğrulamak için sürekli iletişim kurma ihtiyacını ortadan kaldırır. Bu, ağın bir nevi "kendi saatine" sahip olması demektir.

![Akıllı Kontrat Yapısı](/assets/img/posts/smart-contract-structure-diagram.jpg)

## Anchor Framework: Boilerplate'den Kurtulun

Saf Rust ile Solana programı yazmak (Native Rust), tonlarca güvenlik kontrolü ve manuel veri serileştirme gerektirir. Anchor framework'ü, bu süreçleri makrolar aracılığıyla soyutlayarak geliştiricinin "iş mantığına" odaklanmasını sağlar.

- **Güvenlik Otomasyonu:** Reentrancy, ownership ve account validation kontrollerini otomatik yapar.
- **IDL Üretimi:** Akıllı kontratın frontend tarafında kolayca kullanılabilmesi için otomatik olarak JSON şeması (IDL) üretir.
- **Type Safety:** Rust'ın tip sistemini sonuna kadar kullanarak çalışma zamanı hatalarını derleme zamanına çeker.

**Mühendislik Notu:** Bir projeye başlarken native Rust mı yoksa Anchor mı kullanacağınıza karar vermek kritiktir. Çok düşük seviyeli (low-level) optimizasyonlar gerekmiyorsa, Anchor her zaman daha güvenli ve hızlı bir tercihtir.

![Anchor Bileşenleri](/assets/img/posts/what-is-anchor-solana.png)

## Program Geliştirme Akışı: Kurulumdan Kodlamaya

Solana'da geliştirme yapmak için güçlü bir araç seti gerekir. Sadece Rust bilmek yeterli değildir; ağın kendine has "CLI" ve "Test" araçlarına hakim olmalısınız.

### 1. Ortam Kurulumu
Geliştiricilerin en çok zaman kaybettiği yer genellikle versiyon uyumsuzluklarıdır. `solana-cli`, `anchor-cli` ve `rustc` versiyonlarınızın birbiriyle uyumlu olduğundan emin olmak için mutlaka `avm` (Anchor Version Manager) kullanmalısınız.

### 2. PDAs (Program Derived Addresses)
Solana'nın en güçlü ama anlaşılması en zor özelliklerinden biri PDA'dır. PDA'lar, bir program ID ve çeşitli "seed"lerin (tohumların) birleşimiyle üretilen, özel anahtarı (private key) olmayan adreslerdir.
- **Güvenlik:** Sadece o adresi türeten program o hesap üzerinde işlem yapabilir.
- **Determinizm:** Belirli bir kullanıcı ve belirli bir amaç için her zaman aynı adres üretilir. Bu, veritabanı benzeri bir indeksleme imkanı sunar.

## Kod Üzerinden Önemli Kavramlar

Anchor'da bir kontrat yazarken üç temel yapı bloğu vardır: `#[program]`, `#[derive(Accounts)]` ve `#[account]`.

```rust
#[program]
pub mod my_contract {
    pub fn update_data(ctx: Context<UpdateData>, new_val: u64) -> Result<()> {
        let account = &mut ctx.accounts.data_account;
        account.value = new_val;
        Ok(())
    }
}

#[derive(Accounts)]
pub struct UpdateData<'info> {
    #[account(mut, has_one = authority)]
    pub data_account: Account<'info, DataAccount>,
    pub authority: Signer<'info>,
}
```
Yukarıdaki örnekte `has_one = authority` kısıtı, senior bir geliştiricinin asla unutmaması gereken bir "Access Control" (Erişim Kontrolü) mekanizmasıdır. Bu, hesabın sadece asıl sahibi tarafından güncellenebilmesini sağlar.

![Solana Güvenlik ve Test Akışı](/assets/img/posts/smart-contract-structure-diagram.jpg)

## Güvenlik: Solana Dünyasında Hata Payı Yoktur

Blockchain üzerinde yapılan bir hata, geri döndürülemez finansal kayıplara neden olabilir. Solana özelinde dikkat edilmesi gereken kritik noktalar şunlardır:

### 1. Account Validation (Hesap Doğrulama)
Hemen hemen her Solana programı, gelen hesapların gerçekten beklenen hesaplar olduğunu doğrulamalıdır. Anchor bunu `#[account(...)]` makrosuyla kolaylaştırsa da, mantıksal hatalardan kaçınmak için manuel kontroller de gerekebilir.

### 2. Overflow ve Underflow
Rust dili varsayılan olarak "saturating" veya "checked" aritmetik sunar. Solana programlarında bakiye hesaplarken mutlaka `.checked_add()` veya `.checked_sub()` metodlarını kullanmalısınız.

### 3. Reentrancy
Solana'da reentrancy (yeniden giriş) saldırıları, Ethereum'daki kadar yaygın değildir çünkü programlar asenkron değildir. Ancak, Cross-Program Invocations (CPI) yaparken yine de dikkatli olunmalı ve "Checks-Effects-Interactions" prensibi takip edilmelidir.

## Test Stratejileri: Mocha ve Chai İş Başında

Anchor, `anchor test` komutuyla çalışan kapsamlı bir test altyapısı sunar. Senior bir geliştirici, kodunun en az %90'ını testlerle kapsamalıdır (Code Coverage).

- **Unit Tests:** Rust içindeki fonksiyonların basit mantığını test eden hızlı testler.
- **Integration Tests:** TypeScript kullanarak gerçek bir yerel validator (solana-test-validator) üzerinde yapılan uçtan uca testler.
- **Fuzz Testing:** Programınıza rastgele veriler göndererek beklenmedik çöküşleri (panic) tespit eden ileri düzey test teknikleri.

## Performans Optimizasyonu: Compute Units

Solana'da bir işlemin maliyeti sadece gas değil, aynı zamanda harcanan işlemci gücüdür (Compute Units).
- **CU Limitleri:** Her işlemin bir işlemci limiti vardır. Gereksiz döngülerden ve ağır matematiksel işlemlerden kaçınmalısınız.
- **Heap vs Stack:** Stack bellek limitleri dar olduğu için büyük veri yapılarını dikkatli yönetmelisiniz.

**Pro-Tip:** `msg!()` makroları geliştirme aşamasında hayat kurtarıcıdır ancak prodüksiyon ortamında fazla kullanımı Compute Unit tüketimini artırabilir. Gereksiz loglardan kaçının.

![Solana Geliştirme Araçları](/assets/img/posts/anchor-project-initialization.png)

## Yayınlama (Deployment): Devnet'ten Mainnet'e Yolculuk

Kodunuz yerel testlerden geçtiyse, artık dünyaya açılma vakti gelmiştir. Ancak Solana'da canlıya çıkmak, diğer ağlara göre daha maliyetlidir (Rent).

### 1. Devnet Testleri
Her zaman önce `devnet` üzerinde deneme yapın. `solana airdrop` komutuyla ücretsiz SOL alabilir ve programınızın gerçek ağ koşullarında nasıl davrandığını görebilirsiniz.

### 2. Rent (Kira) Mekanizması
Solana'da bir hesap açmak için bakiye ödemeniz gerekir. Eğer hesabın içinde yeterli SOL varsa (Rent-exempt bakiye), kira ödemezsiniz. Senior bir mimar olarak, kullanıcılarınızın veya protokolün bu kira maliyetlerini nasıl karşılayacağını önceden planlamalısınız.

### 3. Upgrade Authority (Güncelleme Yetkisi)
Solana programları varsayılan olarak güncellenebilir (upgradeable). Ancak protokolün tam güven kazanması için bu yetkinin bir "Multi-sig" (örn: Realms) veya "DAO"ya devredilmesi veya tamamen kapatılması (immutable) önerilir.

## Teknik Sözlük (Glossary)

- **Account:** Solana'daki tüm veri depolama birimlerinin genel adı.
- **Lamport:** SOL'un en küçük birimi (1 SOL = 10^9 Lamport).
- **PDA (Program Derived Address):** Özel anahtarı olmayan, program kontrollü adresler.
- **CPI (Cross-Program Invocation):** Bir programdan diğerine yapılan çağrı.
- **IDL (Interface Definition Language):** Programın yapısını tanımlayan JSON dosyası.
- **Sealevel:** İşlemleri paralel işleyen Solana runtime motoru.
- **Rent:** Bir hesabın ağda saklanması için ödenen maliyet.
- **Signer:** İşlemi imzalayan ve yetkilendiren hesap.
- **Compute Unit (CU):** İşlemin harcadığı CPU kaynağının ölçü birimi.
- **Anchor:** Solana geliştirme iş akışını basitleştiren framework.
- **DEX (Decentralized Exchange):** Merkeziyetsiz borsa yapılarını temsil eden protokoller.
- **Oracle:** Zincir dışı verileri Solana programlarına taşıyan servisler (örn: Pyth).
- **Slot:** Solana'nın blok süresine tekabül eden zaman dilimi (~400ms).

## İleri Seviye: Solana'da Likidite ve DEX Entegrasyonu

Solana'nın en popüler kullanım alanı DeFi (Merkeziyetsiz Finans) ekosistemidir. Bir program geliştirirken, Serum veya Raydium gibi devasa likidite havuzlarına nasıl entegre olacağınızı bilmek sizi bir adım öne çıkarır.

- **OpenBook/Serum Integration:** Kendi programınız üzerinden doğrudan borsa emirleri (orderbook) verebilirsiniz.
- **Atomic Swaps:** Tek bir Solana transaction'ı içinde birden fazla işlemi atomik olarak (ya hep ya hiç) gerçekleştirebilirsiniz. Bu, karmaşık finansal işlemleri güvenli kılar.

## Frontend Entegrasyonu: @solana/web3.js

Harika bir program yazdınız ama kullanıcılar buna nasıl erişecek? Solana ekosisteminde frontend tarafı genellikle `@solana/web3.js` ve `@coral-xyz/anchor` paketleri üzerine kurulur.

- **Wallet Adapter:** Kullanıcıların Phantom veya Solflare gibi cüzdanlarını uygulamanıza bağlamasını sağlar.
- **Connection Object:** RPC sunucularına bağlanarak ağdaki verileri çekmenize ve işlem göndermenize yarar.
- **IDL Parsing:** Anchor'ın ürettiği JSON dosyasını frontend tarafında import ederek, program fonksiyonlarını sanki yerel bir Javascript kütüphanesiymiş gibi çağırabilirsiniz.

**Senior İpucu:** Frontend tarafında "transaction confirmation" beklerken her zaman `processed`, `confirmed` ve `finalized` seviyelerini doğru yönetin. Kullanıcıya işleminin "alındığı" bilgisini hemen verip, "kesinleşme" bilgisini arka planda takip etmek en iyi UX pratiğidir.

## Gelecek Vizyonu: Firedancer ve Ötesi

Solana laboratuvarlarında heyecan verici gelişmeler yaşanıyor. Jump Crypto tarafından geliştirilen **Firedancer** isimli yeni validator istemcisi, ağın performansını teorik olarak saniyede 1 milyon işleme çıkarabilir. Bu, Solana'nın sadece en hızlı blockchain değil, küresel finans sisteminin ana omurgası olma yolundaki en büyük adımıdır.

## Sonuç

Solana program geliştirmek, Rust'ın disipliniyle Solana'nın hızını birleştiren benzersiz bir deneyimdir. Anchor framework'ü bu süreci insani seviyelere çekse de, arka plandaki hesap modelini ve performans limitlerini anlamak sizi orta seviye bir yazılımcıdan bir blockchain mimarına dönüştürecektir. Kodunuzu yazarken her zaman güvenlik kısıtlarını (constraints) en başa koyun ve optimizasyonu bir alışkanlık haline getirin.

## İleri Okuma ve Kaynaklar
- Solana Documentation (Official)
- Anchor Framework Book
- Solana Cookbook (Practical Examples)
- Helius Blog (Advanced Engineering Posts)
- Solanafm - Solana Explorer & Analytics Tools
