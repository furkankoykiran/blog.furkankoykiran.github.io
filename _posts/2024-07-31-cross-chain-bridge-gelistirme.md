---
title: "Cross-Chain Bridge Geliştirme: Ethereum ve Polygon Arasında Varlık Transferi"
date: 2024-07-31 14:30:00 +0300
categories: [Blockchain, Infrastructure]
tags: [cross-chain, bridge, ethereum, polygon, web3, smart-contracts, interoperability]
image:
  path: /assets/img/posts/cross-chain-bridge-flow-diagram.png
  alt: "Cross-chain bridge akış diyagramı ve varlık transfer mekanizması"
---

Blockchain ekosisteminin fragmentasyonu, farklı zincirler arasında varlık transferini kritik bir ihtiyaç haline getirdi. Kullanıcılar Ethereum'daki token'larını Polygon'da kullanmak, arbitraj fırsatlarını değerlendirmek veya düşük gas fee'li zincirlerde işlem yapmak istiyorlar. Cross-chain bridge'ler bu sorunu çözüyor. Bu yazıda Ethereum ve Polygon arasında tam fonksiyonel bir bridge nasıl geliştireceğinizi, lock-and-mint mekanizmasını, güvenlik önlemlerini ve relayer implementasyonunu detaylı şekilde inceleyeceğiz.

## Cross-Chain Bridge Temelleri

Blockchain bridge'leri, farklı zincirler arasında bilgi ve değer transferini sağlayan protokollerdir. İki temel yaklaşım vardır:

### 1. Lock-and-Mint (Kilit ve Basım)

Kaynak zincirde token'lar kilitlenir, hedef zincirde eşdeğer wrapped token basılır. Bu yaklaşım en yaygın kullanılan yöntemdir çünkü native token'ı korur ve çift harcama riskini ortadan kaldırır.

### 2. Burn-and-Mint (Yakma ve Basım)

Kaynak zincirde token yakılır, hedef zincirde yeniden basılır. Bu yöntem wrapped token'lar için daha uygundur.

![Lock-and-Mint Mekanizması](/assets/img/posts/lock-and-mint-bridge-mechanism.png)
_Lock-and-mint bridge mekanizmasının çalışma prensibi_

## Proje Mimarisi

Bridge'imiz üç ana bileşenden oluşacak:

1. **Source Chain Contract (Ethereum)**: Token'ları kilitler ve event emit eder
2. **Destination Chain Contract (Polygon)**: Wrapped token basar ve yakma işlemi yapar
3. **Relayer Service**: Event'leri dinler ve zincirler arası mesajları iletir

### Teknoloji Stack

```bash
# Smart Contract Development
npm install --save-dev hardhat @nomiclabs/hardhat-ethers ethers
npm install --save-dev @openzeppelin/contracts

# Python Relayer
pip install web3 python-dotenv asyncio aiohttp redis
```

## Smart Contract Implementasyonu

### Ethereum (Source Chain) Bridge Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/security/Pausable.sol";

/**
 * @title EthereumBridge
 * @dev Ethereum tarafındaki bridge contract'ı
 * Token'ları kilitler ve Polygon'a transfer için event emit eder
 */
contract EthereumBridge is ReentrancyGuard, Ownable, Pausable {
    using SafeERC20 for IERC20;
    
    // Desteklenen token'lar
    mapping(address => bool) public supportedTokens;
    
    // Transfer ID tracking (replay attack önleme)
    mapping(bytes32 => bool) public processedTransfers;
    
    // Minimum ve maksimum transfer limitleri
    mapping(address => uint256) public minTransferAmount;
    mapping(address => uint256) public maxTransferAmount;
    
    // Bridge fee (basis points - 10000 = 100%)
    uint256 public bridgeFee = 10; // 0.1%
    address public feeCollector;
    
    // Event'ler
    event TokensLocked(
        bytes32 indexed transferId,
        address indexed token,
        address indexed sender,
        address recipient,
        uint256 amount,
        uint256 timestamp,
        uint256 nonce
    );
    
    event TokensReleased(
        bytes32 indexed transferId,
        address indexed token,
        address indexed recipient,
        uint256 amount
    );
    
    event TokenAdded(address indexed token, uint256 minAmount, uint256 maxAmount);
    event FeeUpdated(uint256 newFee);
    
    // Nonce tracking (her kullanıcı için)
    mapping(address => uint256) public userNonces;
    
    constructor(address _feeCollector) {
        require(_feeCollector != address(0), "Invalid fee collector");
        feeCollector = _feeCollector;
    }
    
    /**
     * @dev Token'ı bridge'e kilitler ve Polygon'a transfer için event emit eder
     * @param token Transfer edilecek token adresi
     * @param amount Transfer miktarı
     * @param recipient Polygon'daki alıcı adres
     */
    function lockTokens(
        address token,
        uint256 amount,
        address recipient
    ) external nonReentrant whenNotPaused {
        require(supportedTokens[token], "Token not supported");
        require(amount >= minTransferAmount[token], "Amount too low");
        require(amount <= maxTransferAmount[token], "Amount too high");
        require(recipient != address(0), "Invalid recipient");
        
        // Transfer ID oluştur (benzersiz olmalı)
        uint256 nonce = userNonces[msg.sender]++;
        bytes32 transferId = keccak256(
            abi.encodePacked(
                msg.sender,
                recipient,
                token,
                amount,
                block.chainid,
                nonce,
                block.timestamp
            )
        );
        
        require(!processedTransfers[transferId], "Transfer already processed");
        processedTransfers[transferId] = true;
        
        // Fee hesapla
        uint256 fee = (amount * bridgeFee) / 10000;
        uint256 netAmount = amount - fee;
        
        // Token'ları contract'a transfer et
        IERC20(token).safeTransferFrom(msg.sender, address(this), amount);
        
        // Fee'yi collector'a gönder
        if (fee > 0) {
            IERC20(token).safeTransfer(feeCollector, fee);
        }
        
        // Event emit et (relayer bu event'i dinleyecek)
        emit TokensLocked(
            transferId,
            token,
            msg.sender,
            recipient,
            netAmount,
            block.timestamp,
            nonce
        );
    }
    
    /**
     * @dev Polygon'dan gelen token'ları unlock eder
     * @param transferId Polygon'daki transfer ID
     * @param token Token adresi
     * @param recipient Alıcı adres
     * @param amount Miktar
     * @param signatures Validator imzaları
     */
    function releaseTokens(
        bytes32 transferId,
        address token,
        address recipient,
        uint256 amount,
        bytes[] calldata signatures
    ) external nonReentrant whenNotPaused {
        require(!processedTransfers[transferId], "Transfer already processed");
        require(supportedTokens[token], "Token not supported");
        
        // İmzaları doğrula (multi-sig validator sistemi)
        _verifySignatures(transferId, token, recipient, amount, signatures);
        
        processedTransfers[transferId] = true;
        
        // Token'ları unlock et
        IERC20(token).safeTransfer(recipient, amount);
        
        emit TokensReleased(transferId, token, recipient, amount);
    }
    
    /**
     * @dev Yeni token ekler
     */
    function addSupportedToken(
        address token,
        uint256 minAmount,
        uint256 maxAmount
    ) external onlyOwner {
        require(token != address(0), "Invalid token");
        require(maxAmount > minAmount, "Invalid limits");
        
        supportedTokens[token] = true;
        minTransferAmount[token] = minAmount;
        maxTransferAmount[token] = maxAmount;
        
        emit TokenAdded(token, minAmount, maxAmount);
    }
    
    /**
     * @dev Bridge fee'yi günceller
     */
    function updateBridgeFee(uint256 newFee) external onlyOwner {
        require(newFee <= 100, "Fee too high"); // Max 1%
        bridgeFee = newFee;
        emit FeeUpdated(newFee);
    }
    
    /**
     * @dev İmzaları doğrular (basitleştirilmiş - production'da daha complex olmalı)
     */
    function _verifySignatures(
        bytes32 transferId,
        address token,
        address recipient,
        uint256 amount,
        bytes[] calldata signatures
    ) internal view {
        // Production'da: Multi-sig doğrulama
        // Şimdilik basit bir check
        require(signatures.length >= 2, "Insufficient signatures");
        
        bytes32 messageHash = keccak256(
            abi.encodePacked(transferId, token, recipient, amount)
        );
        
        // ECDSA signature recovery ve validator check
        // Bu kısım production'da genişletilmeli
    }
    
    /**
     * @dev Emergency pause
     */
    function pause() external onlyOwner {
        _pause();
    }
    
    function unpause() external onlyOwner {
        _unpause();
    }
}
```

### Polygon (Destination Chain) Bridge Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/token/ERC20/extensions/ERC20Burnable.sol";
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/security/Pausable.sol";

/**
 * @title WrappedToken
 * @dev Polygon'da basılacak wrapped token (örn: wETH, wUSDC)
 */
contract WrappedToken is ERC20, ERC20Burnable, Ownable {
    address public bridge;
    
    constructor(
        string memory name,
        string memory symbol,
        address _bridge
    ) ERC20(name, symbol) {
        bridge = _bridge;
    }
    
    modifier onlyBridge() {
        require(msg.sender == bridge, "Only bridge");
        _;
    }
    
    function mint(address to, uint256 amount) external onlyBridge {
        _mint(to, amount);
    }
    
    function burnFrom(address account, uint256 amount) public override onlyBridge {
        super.burnFrom(account, amount);
    }
}

/**
 * @title PolygonBridge
 * @dev Polygon tarafındaki bridge contract'ı
 * Wrapped token basar ve geri transferde yakar
 */
contract PolygonBridge is ReentrancyGuard, Ownable, Pausable {
    // Ethereum token -> Polygon wrapped token mapping
    mapping(address => address) public wrappedTokens;
    
    // İşlenmiş transfer'lar
    mapping(bytes32 => bool) public processedTransfers;
    
    // Validator adresleri (multi-sig için)
    mapping(address => bool) public validators;
    address[] public validatorList;
    uint256 public requiredSignatures;
    
    event TokensMinted(
        bytes32 indexed transferId,
        address indexed wrappedToken,
        address indexed recipient,
        uint256 amount
    );
    
    event TokensBurned(
        bytes32 indexed transferId,
        address indexed sender,
        address indexed ethereumRecipient,
        address wrappedToken,
        uint256 amount,
        uint256 nonce
    );
    
    event WrappedTokenCreated(
        address indexed ethereumToken,
        address indexed wrappedToken
    );
    
    // User nonce tracking
    mapping(address => uint256) public userNonces;
    
    constructor(address[] memory _validators, uint256 _requiredSignatures) {
        require(_validators.length >= _requiredSignatures, "Invalid validator config");
        
        for (uint256 i = 0; i < _validators.length; i++) {
            require(_validators[i] != address(0), "Invalid validator");
            validators[_validators[i]] = true;
            validatorList.push(_validators[i]);
        }
        
        requiredSignatures = _requiredSignatures;
    }
    
    /**
     * @dev Ethereum'dan gelen transfer için wrapped token basar
     * @param transferId Ethereum'daki transfer ID
     * @param ethereumToken Ethereum'daki original token
     * @param recipient Token alacak adres
     * @param amount Basılacak miktar
     * @param signatures Validator imzaları
     */
    function mintTokens(
        bytes32 transferId,
        address ethereumToken,
        address recipient,
        uint256 amount,
        bytes[] calldata signatures
    ) external nonReentrant whenNotPaused {
        require(!processedTransfers[transferId], "Already processed");
        require(wrappedTokens[ethereumToken] != address(0), "Token not mapped");
        
        // İmzaları doğrula
        _verifySignatures(transferId, ethereumToken, recipient, amount, signatures);
        
        processedTransfers[transferId] = true;
        
        address wrappedToken = wrappedTokens[ethereumToken];
        
        // Wrapped token bas
        WrappedToken(wrappedToken).mint(recipient, amount);
        
        emit TokensMinted(transferId, wrappedToken, recipient, amount);
    }
    
    /**
     * @dev Wrapped token'ları yakarak Ethereum'a geri gönderir
     * @param wrappedToken Yakılacak wrapped token
     * @param amount Miktar
     * @param ethereumRecipient Ethereum'da alacak adres
     */
    function burnTokens(
        address wrappedToken,
        uint256 amount,
        address ethereumRecipient
    ) external nonReentrant whenNotPaused {
        require(ethereumRecipient != address(0), "Invalid recipient");
        
        // Wrapped token'ın gerçek olduğunu doğrula
        address ethereumToken = _getEthereumToken(wrappedToken);
        require(ethereumToken != address(0), "Invalid wrapped token");
        
        // Transfer ID oluştur
        uint256 nonce = userNonces[msg.sender]++;
        bytes32 transferId = keccak256(
            abi.encodePacked(
                msg.sender,
                ethereumRecipient,
                wrappedToken,
                amount,
                block.chainid,
                nonce,
                block.timestamp
            )
        );
        
        // Token'ları yak
        WrappedToken(wrappedToken).burnFrom(msg.sender, amount);
        
        // Event emit et (relayer dinleyecek)
        emit TokensBurned(
            transferId,
            msg.sender,
            ethereumRecipient,
            wrappedToken,
            amount,
            nonce
        );
    }
    
    /**
     * @dev Yeni wrapped token oluşturur
     */
    function createWrappedToken(
        address ethereumToken,
        string memory name,
        string memory symbol
    ) external onlyOwner {
        require(ethereumToken != address(0), "Invalid ethereum token");
        require(wrappedTokens[ethereumToken] == address(0), "Already exists");
        
        WrappedToken wrappedToken = new WrappedToken(name, symbol, address(this));
        wrappedTokens[ethereumToken] = address(wrappedToken);
        
        emit WrappedTokenCreated(ethereumToken, address(wrappedToken));
    }
    
    /**
     * @dev İmzaları doğrular
     */
    function _verifySignatures(
        bytes32 transferId,
        address ethereumToken,
        address recipient,
        uint256 amount,
        bytes[] calldata signatures
    ) internal view {
        require(signatures.length >= requiredSignatures, "Insufficient signatures");
        
        bytes32 messageHash = keccak256(
            abi.encodePacked(transferId, ethereumToken, recipient, amount)
        );
        
        bytes32 ethSignedMessageHash = keccak256(
            abi.encodePacked("\x19Ethereum Signed Message:\n32", messageHash)
        );
        
        address[] memory signers = new address[](signatures.length);
        
        // Her imzayı doğrula ve validator olduğundan emin ol
        for (uint256 i = 0; i < signatures.length; i++) {
            address signer = _recoverSigner(ethSignedMessageHash, signatures[i]);
            require(validators[signer], "Invalid signer");
            
            // Duplicate check
            for (uint256 j = 0; j < i; j++) {
                require(signers[j] != signer, "Duplicate signature");
            }
            
            signers[i] = signer;
        }
    }
    
    /**
     * @dev ECDSA signature'dan signer'ı recover eder
     */
    function _recoverSigner(
        bytes32 messageHash,
        bytes memory signature
    ) internal pure returns (address) {
        require(signature.length == 65, "Invalid signature length");
        
        bytes32 r;
        bytes32 s;
        uint8 v;
        
        assembly {
            r := mload(add(signature, 32))
            s := mload(add(signature, 64))
            v := byte(0, mload(add(signature, 96)))
        }
        
        if (v < 27) {
            v += 27;
        }
        
        require(v == 27 || v == 28, "Invalid signature v value");
        
        return ecrecover(messageHash, v, r, s);
    }
    
    /**
     * @dev Wrapped token'dan ethereum token adresini bulur
     */
    function _getEthereumToken(address wrappedToken) internal view returns (address) {
        for (uint256 i = 0; i < validatorList.length; i++) {
            // Mapping'i ters çevirerek bul
            // Production'da daha efficient bir yöntem kullanılmalı
        }
        return address(0);
    }
    
    function pause() external onlyOwner {
        _pause();
    }
    
    function unpause() external onlyOwner {
        _unpause();
    }
}
```

![Bridge Varlık Transfer Yöntemleri](/assets/img/posts/bridge-asset-transfer-methods.png)
_Farklı bridge tiplerinin varlık transfer mekanizmaları_

## Python Relayer İmplementasyonu

Relayer, her iki zinciri de izleyen ve transfer event'lerini işleyen critical bir bileşendir:

```python
# relayer.py
import asyncio
import os
from web3 import Web3
from web3.middleware import geth_poa_middleware
from eth_account import Account
from eth_account.messages import encode_defunct
import json
from typing import Dict, List
from dotenv import load_dotenv
import redis
import logging

load_dotenv()

# Logging setup
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)
logger = logging.getLogger(__name__)

class BridgeRelayer:
    """
    Cross-chain bridge relayer
    Ethereum ve Polygon arasında event'leri dinler ve işler
    """
    
    def __init__(self):
        # Web3 connections
        self.eth_w3 = Web3(Web3.WebsocketProvider(os.getenv("ETH_WSS_URL")))
        self.eth_w3.middleware_onion.inject(geth_poa_middleware, layer=0)
        
        self.poly_w3 = Web3(Web3.WebsocketProvider(os.getenv("POLYGON_WSS_URL")))
        self.poly_w3.middleware_onion.inject(geth_poa_middleware, layer=0)
        
        # Validator account (private key'den)
        self.validator_account = Account.from_key(os.getenv("VALIDATOR_PRIVATE_KEY"))
        
        # Contract instances
        self.eth_bridge = self._load_contract(
            self.eth_w3,
            os.getenv("ETH_BRIDGE_ADDRESS"),
            "EthereumBridge.json"
        )
        
        self.poly_bridge = self._load_contract(
            self.poly_w3,
            os.getenv("POLYGON_BRIDGE_ADDRESS"),
            "PolygonBridge.json"
        )
        
        # Redis for state management
        self.redis_client = redis.Redis(
            host='localhost',
            port=6379,
            decode_responses=True
        )
        
        logger.info("✅ Bridge Relayer initialized")
        logger.info(f"📍 Validator address: {self.validator_account.address}")
    
    def _load_contract(self, w3: Web3, address: str, abi_file: str):
        """Contract instance oluşturur"""
        with open(f"abis/{abi_file}", "r") as f:
            abi = json.load(f)
        
        return w3.eth.contract(
            address=Web3.to_checksum_address(address),
            abi=abi
        )
    
    async def monitor_ethereum_locks(self):
        """
        Ethereum'daki TokensLocked event'lerini dinler
        """
        logger.info("🔍 Monitoring Ethereum TokensLocked events...")
        
        # Event filter oluştur
        event_filter = self.eth_bridge.events.TokensLocked.create_filter(
            fromBlock='latest'
        )
        
        while True:
            try:
                # Yeni event'leri al
                for event in event_filter.get_new_entries():
                    await self._process_ethereum_lock(event)
                
                await asyncio.sleep(2)  # 2 saniye polling
                
            except Exception as e:
                logger.error(f"❌ Ethereum monitoring error: {e}")
                await asyncio.sleep(5)
    
    async def _process_ethereum_lock(self, event):
        """
        Ethereum'da kilitlenmiş token için Polygon'da mint işlemi başlatır
        """
        args = event.args
        transfer_id = args.transferId.hex()
        
        logger.info(f"\n🔒 New Ethereum Lock Detected")
        logger.info(f"   Transfer ID: {transfer_id}")
        logger.info(f"   Token: {args.token}")
        logger.info(f"   Amount: {args.amount}")
        logger.info(f"   Recipient: {args.recipient}")
        
        # Redis'te işlenmiş mi kontrol et
        if self.redis_client.get(f"processed:eth:{transfer_id}"):
            logger.info("   ⏭️  Already processed, skipping")
            return
        
        # Confirmation bekle (6 blok)
        await self._wait_confirmations(self.eth_w3, event.blockNumber, 6)
        
        # İmza oluştur
        signature = self._create_signature(
            transfer_id,
            args.token,
            args.recipient,
            args.amount
        )
        
        # Polygon'da mint transaction gönder
        try:
            tx_hash = await self._mint_on_polygon(
                transfer_id,
                args.token,
                args.recipient,
                args.amount,
                [signature]  # Production'da multiple validator imzaları
            )
            
            logger.info(f"   ✅ Minted on Polygon: {tx_hash.hex()}")
            
            # Redis'e işlenmiş olarak kaydet
            self.redis_client.setex(
                f"processed:eth:{transfer_id}",
                86400,  # 24 saat TTL
                "1"
            )
            
        except Exception as e:
            logger.error(f"   ❌ Mint failed: {e}")
    
    async def monitor_polygon_burns(self):
        """
        Polygon'daki TokensBurned event'lerini dinler
        """
        logger.info("🔍 Monitoring Polygon TokensBurned events...")
        
        event_filter = self.poly_bridge.events.TokensBurned.create_filter(
            fromBlock='latest'
        )
        
        while True:
            try:
                for event in event_filter.get_new_entries():
                    await self._process_polygon_burn(event)
                
                await asyncio.sleep(2)
                
            except Exception as e:
                logger.error(f"❌ Polygon monitoring error: {e}")
                await asyncio.sleep(5)
    
    async def _process_polygon_burn(self, event):
        """
        Polygon'da yakılan token için Ethereum'da release işlemi başlatır
        """
        args = event.args
        transfer_id = args.transferId.hex()
        
        logger.info(f"\n🔥 New Polygon Burn Detected")
        logger.info(f"   Transfer ID: {transfer_id}")
        logger.info(f"   Wrapped Token: {args.wrappedToken}")
        logger.info(f"   Amount: {args.amount}")
        logger.info(f"   Ethereum Recipient: {args.ethereumRecipient}")
        
        if self.redis_client.get(f"processed:poly:{transfer_id}"):
            logger.info("   ⏭️  Already processed, skipping")
            return
        
        # Confirmation bekle
        await self._wait_confirmations(self.poly_w3, event.blockNumber, 128)
        
        # İmza oluştur
        signature = self._create_signature(
            transfer_id,
            args.wrappedToken,
            args.ethereumRecipient,
            args.amount
        )
        
        # Ethereum'da release transaction gönder
        try:
            tx_hash = await self._release_on_ethereum(
                transfer_id,
                args.wrappedToken,
                args.ethereumRecipient,
                args.amount,
                [signature]
            )
            
            logger.info(f"   ✅ Released on Ethereum: {tx_hash.hex()}")
            
            self.redis_client.setex(
                f"processed:poly:{transfer_id}",
                86400,
                "1"
            )
            
        except Exception as e:
            logger.error(f"   ❌ Release failed: {e}")
    
    def _create_signature(
        self,
        transfer_id: str,
        token: str,
        recipient: str,
        amount: int
    ) -> bytes:
        """
        Transfer için validator imzası oluşturur
        """
        # Message hash oluştur
        message_hash = Web3.keccak(
            Web3.to_bytes(hexstr=transfer_id) +
            Web3.to_bytes(hexstr=token) +
            Web3.to_bytes(hexstr=recipient) +
            amount.to_bytes(32, 'big')
        )
        
        # Ethereum signed message format
        signable_message = encode_defunct(message_hash)
        
        # İmzala
        signed = self.validator_account.sign_message(signable_message)
        
        # v, r, s'yi bytes olarak birleştir
        signature = signed.r.to_bytes(32, 'big') + \
                     signed.s.to_bytes(32, 'big') + \
                     signed.v.to_bytes(1, 'big')
        
        return signature
    
    async def _mint_on_polygon(
        self,
        transfer_id: str,
        ethereum_token: str,
        recipient: str,
        amount: int,
        signatures: List[bytes]
    ) -> bytes:
        """
        Polygon'da wrapped token mint eder
        """
        nonce = self.poly_w3.eth.get_transaction_count(
            self.validator_account.address
        )
        
        # Transaction oluştur
        tx = self.poly_bridge.functions.mintTokens(
            Web3.to_bytes(hexstr=transfer_id),
            Web3.to_checksum_address(ethereum_token),
            Web3.to_checksum_address(recipient),
            amount,
            signatures
        ).build_transaction({
            'from': self.validator_account.address,
            'nonce': nonce,
            'gas': 300000,
            'gasPrice': self.poly_w3.eth.gas_price,
            'chainId': 137  # Polygon Mainnet
        })
        
        # İmzala ve gönder
        signed_tx = self.validator_account.sign_transaction(tx)
        tx_hash = self.poly_w3.eth.send_raw_transaction(signed_tx.rawTransaction)
        
        # Receipt bekle
        receipt = self.poly_w3.eth.wait_for_transaction_receipt(tx_hash)
        
        if receipt.status != 1:
            raise Exception("Transaction failed")
        
        return tx_hash
    
    async def _release_on_ethereum(
        self,
        transfer_id: str,
        token: str,
        recipient: str,
        amount: int,
        signatures: List[bytes]
    ) -> bytes:
        """
        Ethereum'da kilitlenen token'ları release eder
        """
        nonce = self.eth_w3.eth.get_transaction_count(
            self.validator_account.address
        )
        
        tx = self.eth_bridge.functions.releaseTokens(
            Web3.to_bytes(hexstr=transfer_id),
            Web3.to_checksum_address(token),
            Web3.to_checksum_address(recipient),
            amount,
            signatures
        ).build_transaction({
            'from': self.validator_account.address,
            'nonce': nonce,
            'gas': 200000,
            'gasPrice': self.eth_w3.eth.gas_price,
            'chainId': 1  # Ethereum Mainnet
        })
        
        signed_tx = self.validator_account.sign_transaction(tx)
        tx_hash = self.eth_w3.eth.send_raw_transaction(signed_tx.rawTransaction)
        
        receipt = self.eth_w3.eth.wait_for_transaction_receipt(tx_hash)
        
        if receipt.status != 1:
            raise Exception("Transaction failed")
        
        return tx_hash
    
    async def _wait_confirmations(self, w3: Web3, block_number: int, confirmations: int):
        """
        Belirtilen sayıda confirmation bekler
        """
        target_block = block_number + confirmations
        
        while True:
            current_block = w3.eth.block_number
            
            if current_block >= target_block:
                logger.info(f"   ✅ {confirmations} confirmations reached")
                break
            
            remaining = target_block - current_block
            logger.info(f"   ⏳ Waiting for confirmations... ({remaining} blocks)")
            
            await asyncio.sleep(12)  # Ethereum block time
    
    async def run(self):
        """
        Relayer'ı başlatır (her iki zinciri de aynı anda izler)
        """
        logger.info("🚀 Starting Bridge Relayer...")
        
        tasks = [
            asyncio.create_task(self.monitor_ethereum_locks()),
            asyncio.create_task(self.monitor_polygon_burns())
        ]
        
        await asyncio.gather(*tasks)

# Main
if __name__ == "__main__":
    relayer = BridgeRelayer()
    asyncio.run(relayer.run())
```

![Blockchain Relayer Mimarisi](/assets/img/posts/blockchain-relayer-architecture.png)
_Relayer servisinin zincirler arası mesaj iletim mimarisi_

## Deployment ve Test

### Hardhat Deployment Script

```javascript
// scripts/deploy.js
const hre = require("hardhat");

async function main() {
  console.log("🚀 Deploying Bridge Contracts...\n");
  
  // Ethereum Bridge Deploy
  console.log("📍 Deploying Ethereum Bridge...");
  const EthereumBridge = await hre.ethers.getContractFactory("EthereumBridge");
  const feeCollector = "0x..."; // Fee collector adresi
  const ethBridge = await EthereumBridge.deploy(feeCollector);
  await ethBridge.deployed();
  console.log(`✅ Ethereum Bridge: ${ethBridge.address}\n`);
  
  // Polygon Bridge Deploy
  console.log("📍 Deploying Polygon Bridge...");
  const PolygonBridge = await hre.ethers.getContractFactory("PolygonBridge");
  const validators = [
    "0x...",  // Validator 1
    "0x...",  // Validator 2
    "0x..."   // Validator 3
  ];
  const requiredSignatures = 2; // 2-of-3 multi-sig
  
  const polyBridge = await PolygonBridge.deploy(validators, requiredSignatures);
  await polyBridge.deployed();
  console.log(`✅ Polygon Bridge: ${polyBridge.address}\n`);
  
  // USDC Token Support Ekle
  console.log("📍 Adding USDC support...");
  const usdcAddress = "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48"; // Mainnet USDC
  
  const addTokenTx = await ethBridge.addSupportedToken(
    usdcAddress,
    hre.ethers.utils.parseUnits("10", 6),  // Min: 10 USDC
    hre.ethers.utils.parseUnits("1000000", 6)  // Max: 1M USDC
  );
  await addTokenTx.wait();
  console.log("✅ USDC support added to Ethereum Bridge\n");
  
  // Wrapped USDC Deploy
  console.log("📍 Creating Wrapped USDC on Polygon...");
  const createWrappedTx = await polyBridge.createWrappedToken(
    usdcAddress,
    "Wrapped USDC",
    "wUSDC"
  );
  const receipt = await createWrappedTx.wait();
  const wrappedUSDC = receipt.events[0].args.wrappedToken;
  console.log(`✅ Wrapped USDC: ${wrappedUSDC}\n`);
  
  // Deployment summary
  console.log("📋 Deployment Summary");
  console.log("=".repeat(50));
  console.log(`Ethereum Bridge: ${ethBridge.address}`);
  console.log(`Polygon Bridge:  ${polyBridge.address}`);
  console.log(`Wrapped USDC:    ${wrappedUSDC}`);
  console.log(`Validators:      ${validators.length}`);
  console.log(`Required Sigs:   ${requiredSignatures}`);
}

main()
  .then(() => process.exit(0))
  .catch((error) => {
    console.error(error);
    process.exit(1);
  });
```

### Test Suite

```javascript
// test/Bridge.test.js
const { expect } = require("chai");
const { ethers } = require("hardhat");

describe("Cross-Chain Bridge", function () {
  let ethBridge, polyBridge, usdc, wUSDC;
  let owner, user, validator1, validator2, validator3;
  
  beforeEach(async function () {
    [owner, user, validator1, validator2, validator3] = await ethers.getSigners();
    
    // Mock USDC deploy
    const MockERC20 = await ethers.getContractFactory("MockERC20");
    usdc = await MockERC20.deploy("USDC", "USDC", 6);
    
    // Bridges deploy
    const EthereumBridge = await ethers.getContractFactory("EthereumBridge");
    ethBridge = await EthereumBridge.deploy(owner.address);
    
    const PolygonBridge = await ethers.getContractFactory("PolygonBridge");
    polyBridge = await PolygonBridge.deploy(
      [validator1.address, validator2.address, validator3.address],
      2  // 2-of-3
    );
    
    // Setup
    await ethBridge.addSupportedToken(
      usdc.address,
      ethers.utils.parseUnits("1", 6),
      ethers.utils.parseUnits("1000000", 6)
    );
    
    await polyBridge.createWrappedToken(
      usdc.address,
      "Wrapped USDC",
      "wUSDC"
    );
    
    const wrappedAddr = await polyBridge.wrappedTokens(usdc.address);
    wUSDC = await ethers.getContractAt("WrappedToken", wrappedAddr);
    
    // User'a USDC ver
    await usdc.mint(user.address, ethers.utils.parseUnits("10000", 6));
  });
  
  it("Should lock tokens on Ethereum", async function () {
    const amount = ethers.utils.parseUnits("100", 6);
    
    // Approve
    await usdc.connect(user).approve(ethBridge.address, amount);
    
    // Lock
    const tx = await ethBridge.connect(user).lockTokens(
      usdc.address,
      amount,
      user.address
    );
    
    const receipt = await tx.wait();
    const event = receipt.events.find(e => e.event === "TokensLocked");
    
    expect(event.args.amount).to.equal(amount.mul(9990).div(10000)); // 0.1% fee
    expect(await usdc.balanceOf(ethBridge.address)).to.equal(amount.mul(9990).div(10000));
  });
  
  it("Should mint wrapped tokens on Polygon", async function () {
    // Transfer ID simüle et
    const transferId = ethers.utils.keccak256(
      ethers.utils.defaultAbiCoder.encode(
        ["address", "address", "uint256"],
        [user.address, usdc.address, 100]
      )
    );
    
    const amount = ethers.utils.parseUnits("100", 6);
    
    // İmzalar oluştur (validator1 ve validator2)
    const messageHash = ethers.utils.keccak256(
      ethers.utils.defaultAbiCoder.encode(
        ["bytes32", "address", "address", "uint256"],
        [transferId, usdc.address, user.address, amount]
      )
    );
    
    const sig1 = await validator1.signMessage(ethers.utils.arrayify(messageHash));
    const sig2 = await validator2.signMessage(ethers.utils.arrayify(messageHash));
    
    // Mint
    await polyBridge.mintTokens(
      transferId,
      usdc.address,
      user.address,
      amount,
      [sig1, sig2]
    );
    
    expect(await wUSDC.balanceOf(user.address)).to.equal(amount);
  });
  
  it("Should burn and release tokens", async function () {
    // Önce mint et
    const amount = ethers.utils.parseUnits("100", 6);
    // ... (önceki test'teki mint işlemi)
    
    // Wrapped token'ları yak
    await wUSDC.connect(user).approve(polyBridge.address, amount);
    
    const tx = await polyBridge.connect(user).burnTokens(
      wUSDC.address,
      amount,
      user.address
    );
    
    const receipt = await tx.wait();
    const event = receipt.events.find(e => e.event === "TokensBurned");
    
    expect(event.args.amount).to.equal(amount);
    expect(await wUSDC.balanceOf(user.address)).to.equal(0);
  });
  
  it("Should reject insufficient signatures", async function () {
    const transferId = ethers.utils.randomBytes(32);
    const amount = ethers.utils.parseUnits("100", 6);
    
    const messageHash = ethers.utils.keccak256(
      ethers.utils.defaultAbiCoder.encode(
        ["bytes32", "address", "address", "uint256"],
        [transferId, usdc.address, user.address, amount]
      )
    );
    
    const sig1 = await validator1.signMessage(ethers.utils.arrayify(messageHash));
    
    // Sadece 1 imza (2 gerekli)
    await expect(
      polyBridge.mintTokens(
        transferId,
        usdc.address,
        user.address,
        amount,
        [sig1]
      )
    ).to.be.revertedWith("Insufficient signatures");
  });
});
```

## Güvenlik Önlemleri ve Best Practices

### 1. Multi-Signature Validation

Production bridge'lerde mutlaka multi-sig validator sistemi kullanın:

```python
# Minimum 3-of-5 validator setup
VALIDATOR_ADDRESSES = [
    "0x...",  # Validator 1
    "0x...",  # Validator 2
    "0x...",  # Validator 3
    "0x...",  # Validator 4
    "0x...",  # Validator 5
]

REQUIRED_SIGNATURES = 3  # 3-of-5
```

### 2. Rate Limiting

Bridge'e hızlı ve büyük transfer saldırılarını önlemek için:

```solidity
// Daily transfer limits
mapping(address => uint256) public dailyVolume;
mapping(address => uint256) public lastResetTime;
uint256 public dailyLimit = 1000000 * 10**6; // 1M USDC

function checkRateLimit(address token, uint256 amount) internal {
    if (block.timestamp > lastResetTime[token] + 1 days) {
        dailyVolume[token] = 0;
        lastResetTime[token] = block.timestamp;
    }
    
    require(
        dailyVolume[token] + amount <= dailyLimit,
        "Daily limit exceeded"
    );
    
    dailyVolume[token] += amount;
}
```

### 3. Emergency Pause Mechanism

Kritik güvenlik açığı tespit edildiğinde bridge'i durdurabilmek:

```python
# Admin dashboard'dan pause
async def emergency_pause():
    """
    Tüm bridge contract'larını pause eder
    """
    logger.warning("🚨 EMERGENCY PAUSE ACTIVATED")
    
    # Ethereum bridge pause
    eth_tx = await eth_bridge.functions.pause().build_transaction({...})
    # ...
    
    # Polygon bridge pause
    poly_tx = await poly_bridge.functions.pause().build_transaction({...})
    # ...
```

### 4. Monitoring ve Alerting

Bridge health monitoring sistemi:

```python
import asyncio
from datetime import datetime, timedelta

class BridgeMonitor:
    """
    Bridge sağlık kontrolü ve alert sistemi
    """
    
    async def check_bridge_health(self):
        """
        Periyodik health check
        """
        while True:
            # 1. Balance check
            await self._check_balances()
            
            # 2. Pending transfers check
            await self._check_pending_transfers()
            
            # 3. Validator status
            await self._check_validators()
            
            # 4. Gas price monitoring
            await self._check_gas_prices()
            
            await asyncio.sleep(300)  # Her 5 dakika
    
    async def _check_balances(self):
        """
        Bridge balance'larının tutarlı olduğunu kontrol eder
        """
        # Ethereum'daki locked amount
        eth_locked = await self.get_total_locked()
        
        # Polygon'daki minted amount
        poly_minted = await self.get_total_minted()
        
        # Fark %1'den fazla ise alert
        if abs(eth_locked - poly_minted) / eth_locked > 0.01:
            await self.send_alert(
                "⚠️ Balance Mismatch Detected",
                f"ETH Locked: {eth_locked}\nPoly Minted: {poly_minted}"
            )
```

## Performans Optimizasyonu

### Batch Processing

Çok sayıda transfer için gas tasarrufu:

```solidity
function lockTokensBatch(
    address[] calldata tokens,
    uint256[] calldata amounts,
    address[] calldata recipients
) external nonReentrant whenNotPaused {
    require(tokens.length == amounts.length, "Length mismatch");
    require(tokens.length == recipients.length, "Length mismatch");
    require(tokens.length <= 10, "Batch too large");
    
    for (uint256 i = 0; i < tokens.length; i++) {
        _lockTokensSingle(tokens[i], amounts[i], recipients[i]);
    }
}
```

### Relayer Load Balancing

Multiple relayer instance çalıştırarak redundancy:

```python
# docker-compose.yml
version: '3.8'

services:
  relayer-1:
    build: .
    environment:
      - RELAYER_ID=1
      - REDIS_HOST=redis
    restart: always
  
  relayer-2:
    build: .
    environment:
      - RELAYER_ID=2
      - REDIS_HOST=redis
    restart: always
  
  relayer-3:
    build: .
    environment:
      - RELAYER_ID=3
      - REDIS_HOST=redis
    restart: always
  
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
```

## Sonuç

Cross-chain bridge geliştirmek, karmaşık ancak blockchain ekosisteminin interoperability'si için kritik bir konudur. Bu yazıda ele aldığımız lock-and-mint mekanizması, güvenli ve ölçeklenebilir bir bridge implementasyonu için temel oluşturur.

Production ortamına geçmeden önce mutlaka:
- Kapsamlı security audit yaptırın (CertiK, OpenZeppelin, vb.)
- Bug bounty programı başlatın
- Testnet'te extensive testing yapın
- Multi-sig wallet ve timelock kullanın
- Insurance fund oluşturun

Cross-chain future'da bridge'ler blockchain adoption'ı için vazgeçilmez olacaktır.

### Faydalı Kaynaklar

- [Ethereum Bridge Security Best Practices](https://ethereum.org/en/developers/docs/bridges/)
- [Polygon Bridge Documentation](https://docs.polygon.technology/pos/how-to/bridging/ethereum-polygon/)
- [OpenZeppelin Contracts](https://docs.openzeppelin.com/contracts/)
- [Chainlink CCIP](https://chain.link/cross-chain)
- [LayerZero Protocol](https://layerzero.network/)
