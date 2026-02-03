---
title: "Ethereum NFT Minting Rehberi: ERC-721A ve Gas Optimizasyonu"
description: "Sıfırdan gas-optimized NFT koleksiyonu geliştirme. ERC-721A ile batch minting, IPFS metadata yönetimi ve güvenli whitelist implementasyonu."
date: 2024-01-25
categories: [Smart Contracts, Ethereum]
tags: [nft, solidity, erc721a, ipfs, web3, minting]
image:
  path: /assets/img/posts/nft-project-architecture-complete.png
  alt: "NFT Project Architecture and Workflow"
---

## Giriş: Dijital Varlıkların Mimarisi

NFT (Non-Fungible Token), sadece bir JPEG dosyası değil, programlanabilir dijital mülkiyettir. Bir proje geliştirirken, sadece kontratı deploy etmek yetmez; metadata depolamadan whitelist yönetimine, gas optimizasyonundan ön yüz entegrasyonuna kadar tam bir "stack" kurmanız gerekir.

Bu rehberde, 10.000 parçalık bir koleksiyonu (10k PFP) en düşük gas ücretiyle nasıl deploy edeceğinizi, **Azuki'nin geliştirdiği ERC-721A** standardı üzerinden anlatacağım.

![ERC-721 Standard Diagram](/assets/img/posts/erc721-standard-diagram.png)

## 1. Standartlar Savaşı: ERC-721 vs ERC-721A

Geleneksel OpenZeppelin `ERC721Enumerable` standardı, her mint işleminde storage'ı günceller ve bu da yüksek gas ücretine (örneğin 5 NFT için ~150$) neden olur.

**ERC-721A** ise "Batch Minting" optimizasyonu ile bu maliyeti düşürür. 5 NFT'yi tek seferde mintlemek, neredeyse 1 NFT mintlemek ile aynı maliyete gelir (~20$).

### Neden ERC-721A?
1.  **Gas Tasarrufu:** Birden fazla NFT mintlerken %80'e varan tasarruf.
2.  **Lazy Initialization:** Token sahipliği verisi, transfer edilene kadar boş bırakılır (yani "lazy" yazılır).
3.  **OwnerOf Optimizasyonu:** Bir sonraki dolu slotu arayarak sahibini bulur.

## 2. Metadata ve IPFS: Kalıcı Depolama

NFT'nizin görseli bir AWS sunucusunda duruyorsa, o NFT'ye sahip değilsiniz demektir. Merkeziyetsiz depolama (IPFS) standarttır.

> **Provenance Hash:** Adil bir dağıtım yaptığınızı kanıtlamak için, tüm görsellerin oluşturduğu hash'i (SHA-256) kontrata en baştan yazmalısınız. Bu, metadata'yı sonradan değiştirmediğinizi kanıtlar.
{: .prompt-info }

IPFS'e dosya yüklemek için Pinata veya NFT.Storage kullanabilirsiniz. Klasör yapınız şöyle olmalı:
- `json/1.json` (Metadata içinde `image: ipfs://CID/1.png`)

![Smart Contract Deployment](/assets/img/posts/smart-contract-deployment-nft.png)

## 3. Akıllı Kontrat: Solidity ve ERC-721A

Aşağıda, whitelist desteği olan ve Azuki standartlarını kullanan optimize edilmiş bir kontrat örneği bulunmaktadır:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.4;

import "erc721a/contracts/ERC721A.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/cryptography/MerkleProof.sol";

contract MyNFT is ERC721A, Ownable {
    bytes32 public merkleRoot;
    uint256 public constant MAX_SUPPLY = 10000;
    uint256 public constant PRICE = 0.05 ether;

    constructor() ERC721A("MyNFT", "MNFT") {}

    function mint(uint256 quantity, bytes32[] calldata merkleProof) external payable {
        require(totalSupply() + quantity <= MAX_SUPPLY, "Sold out");
        require(msg.value >= PRICE * quantity, "Yetersiz ETH");
        
        // Whitelist kontrolü (Merkle Tree)
        bytes32 leaf = keccak256(abi.encodePacked(msg.sender));
        require(MerkleProof.verify(merkleProof, merkleRoot, leaf), "Not whitelisted");

        _safeMint(msg.sender, quantity);
    }
    
    function setMerkleRoot(bytes32 _merkleRoot) external onlyOwner {
        merkleRoot = _merkleRoot;
    }
}
```

## 4. Güvenlik: Merkle Tree Whitelist

Binlerce adresi tek tek kontrata array olarak eklemek (`mapping(address => bool)`) binlerce dolar gas ücretine mal olur.

Bunun yerine, tüm whitelist adreslerini off-chain (Python/JS ile) hashleyip tek bir `Root Hash` üretirsiniz. Kontrata sadece bu Root'u kaydedersiniz. Kullanıcılar, mint yaparken kendilerinin o ağaca ait olduğunu kanıtlayan bir `Proof` dizisi sunar. Bu yöntemle whitelist maliyeti sabittir ve neredeyse bedavadır.

![NFT Minting DApp Architecture](/assets/img/posts/nft-minting-ethereum-architecture.png)

## 5. Frontend Entegrasyonu: Mint Butonu

React ve `ethers.js` kullanarak kullanıcıların cüzdanlarını bağlayıp mint yapmalarını sağlayalım.

```javascript
import { ethers } from "ethers";
import MerkleTree from "merkletreejs";
import keccak256 from "keccak256";

async function mintNFT() {
    const provider = new ethers.providers.Web3Provider(window.ethereum);
    const signer = provider.getSigner();
    const address = await signer.getAddress();
    
    // Whitelist Proof Oluşturma (Client-side)
    // NOT: Prodüksiyonda proof'u backend'den (API) çekmek daha güvenlidir.
    const leaf = keccak256(address);
    const proof = tree.getHexProof(leaf);

    const contract = new ethers.Contract(CONTRACT_ADDRESS, ABI, signer);

    try {
        const tx = await contract.mint(1, proof, {
            value: ethers.utils.parseEther("0.05")
        });
        await tx.wait();
        alert("Mint Başarılı!");
    } catch (err) {
        console.error("Mint Hatası:", err);
    }
}
```

## 6. Test ve Doğrulama: Güvenden Önce Kontrol

Akıllı kontratlarda "Ctrl+Z" yoktur. Bu yüzden Hardhat ile kapsamlı test yazmak zorundasınız.

```javascript
const { expect } = require("chai");
const { ethers } = require("hardhat");
const { MerkleTree } = require("merkletreejs");
const keccak256 = require("keccak256");

describe("MyNFT", function () {
  it("Should whitelist mint correctly", async function () {
    const [owner, addr1] = await ethers.getSigners();
    const whitelist = [owner.address, addr1.address];
    const leaves = whitelist.map(addr => keccak256(addr));
    const tree = new MerkleTree(leaves, keccak256, { sort: true });
    
    const MyNFT = await ethers.getContractFactory("MyNFT");
    const nft = await MyNFT.deploy();
    
    // Root'u set et
    const root = tree.getHexRoot();
    await nft.setMerkleRoot(root);
    
    // Proof ile mintle
    const leaf = keccak256(addr1.address);
    const proof = tree.getHexProof(leaf);
    
    await expect(nft.connect(addr1).mint(1, proof, { value: ethers.utils.parseEther("0.05") }))
      .to.emit(nft, "Transfer");
  });
});
```

## 7. Deployment: Mainnet Yolculuğu

Testler bittiğinde, kontratı Ethereum Mainnet'e (veya L2'lere) yüklemek için `hardhat-deploy` kullanın.

```javascript
// scripts/deploy.js
async function main() {
  const MyNFT = await ethers.getContractFactory("MyNFT");
  const nft = await MyNFT.deploy();
  await nft.deployed();

  console.log("NFT deployed to:", nft.address);
  
  // Verify işlemi (Etherscan'de kodun görünmesi için)
  await run("verify:verify", {
    address: nft.address,
    constructorArguments: [],
  });
}
```

> **Gas Price Uyarısı:** Deployment yaparken Gas fiyatının düşük olduğu saatleri (genellikle hafta sonu sabahları) kovalayın. Etherscan Gas Tracker'ı izleyin.
{: .prompt-warning }

## 8. Sonuç

Bir NFT projesi başarısı, topluluk (%50), sanat (%30) ve teknik altyapının (%20) kusursuz uyumuna bağlıdır. Bu rehberde teknik altyapının en sağlam halini (ERC-721A + IPFS + Merkle Tree) kurdunuz.

Eğer bu NFT'leri bir pazar yerinde listelemek isterseniz [DEX Aggregator](/web3/defi/2024/03/29/building-dex-aggregator-web3/) mantığını inceleyebilir, satış gelirlerini takip etmek için [Blockchain Data Analysis](/blockchain/data-science/2024/03/07/blockchain-veri-analizi-python-araclari/) araçlarına göz atabilirsiniz.
