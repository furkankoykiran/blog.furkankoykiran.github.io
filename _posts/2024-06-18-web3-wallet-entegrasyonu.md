---
title: "Web3 Wallet Entegrasyonu: MetaMask ve WalletConnect ile DApp Geliştirme"
description: "MetaMask ve WalletConnect ile Web3 wallet entegrasyonu rehberi. React, Vue ve vanilla JavaScript örnekleriyle modern DApp geliştirme, güvenlik ve best practices."
date: "2024-06-18 14:00:00 +0300"
categories: [Web3, Frontend]
tags: [web3, metamask, walletconnect, ethereum, dapp, javascript, wallet, blockchain]
image:
  path: /assets/img/posts/web3-application-architecture.png
  alt: "Web3 Wallet Entegrasyonu"
---

## Giriş

Web3 wallet entegrasyonu, modern blockchain uygulamalarının en kritik bileşenlerinden biridir. Kullanıcıların blockchain ile etkileşime girmesi, işlem yapması ve dijital varlıklarını yönetmesi için cüzdan bağlantısı şarttır. MetaMask, WalletConnect, Coinbase Wallet gibi çözümler, kullanıcı deneyimini kolaylaştırırken güvenli bir şekilde blockchain'e erişim sağlar.

Bu kapsamlı rehberde, Web3 wallet entegrasyonunu sıfırdan öğreneceksiniz. MetaMask ile temel bağlantıdan, WalletConnect ile çoklu cüzdan desteğine, işlem imzalamadan kullanıcı yetkilendirmesine kadar her şeyi detaylı kod örnekleriyle ele alacağız. React, Vue ve vanilla JavaScript ile implementasyon örnekleri göreceğiz.

DApp (Decentralized Application) geliştirmek isteyenler için bu rehber, production-ready wallet entegrasyonu oluşturmak için gereken tüm bilgiyi sağlayacak. Güvenlik best practice'leri, error handling ve kullanıcı deneyimi optimizasyonlarını da öğreneceksiniz.

![Web3 Architecture](/assets/img/posts/web3-smart-contract-architecture.png){: w="700" h="400" .shadow }
_Web3 uygulama mimarisi ve wallet bağlantı akışı_

## Web3 Wallet'lara Giriş

### Web3 Wallet Nedir?

Web3 wallet'lar, blockchain ağlarına erişim sağlayan ve dijital varlıkları yöneten yazılımlardır:

- **Browser Extension**: MetaMask, Coinbase Wallet (en popüler)
- **Mobile Wallet**: Trust Wallet, Rainbow, Argent
- **Hardware Wallet**: Ledger, Trezor (en güvenli)
- **Multi-Chain**: WalletConnect (çoklu cüzdan desteği)

### Wallet Türleri

1. **Custodial Wallet**: Private key'ler servis sağlayıcıda (Coinbase, Binance)
2. **Non-Custodial**: Kullanıcı kendi key'lerini yönetir (MetaMask, Trust Wallet)
3. **Multi-Sig**: Çoklu imza gerektiren (Gnosis Safe)
4. **Smart Contract Wallet**: Programlanabilir (Argent, Safe)

> Non-custodial wallet'lar, kullanıcılara tam kontrol sağlar ancak private key kaybı durumunda fonlar kurtarılamaz.
{: .prompt-info }

### Web3 Provider'lar

```javascript
// Browser'da Web3 provider'ı algılama
if (typeof window.ethereum !== 'undefined') {
  console.log('MetaMask is installed!');
  // window.ethereum = EIP-1193 provider
  
  // Provider bilgilerini kontrol et
  console.log('Provider:', window.ethereum.isMetaMask ? 'MetaMask' : 'Other');
  console.log('Chain ID:', await window.ethereum.request({ method: 'eth_chainId' }));
} else {
  console.log('Please install MetaMask!');
  // Kullanıcıyı MetaMask kurulumuna yönlendir
  window.location.href = 'https://metamask.io/download/';
}

// Birden fazla provider kontrolü
const detectProviders = () => {
  const providers = [];
  
  if (window.ethereum) {
    // MetaMask
    if (window.ethereum.isMetaMask) {
      providers.push({ name: 'MetaMask', provider: window.ethereum });
    }
    
    // Coinbase Wallet
    if (window.ethereum.isCoinbaseWallet) {
      providers.push({ name: 'Coinbase', provider: window.ethereum });
    }
    
    // Trust Wallet
    if (window.ethereum.isTrust) {
      providers.push({ name: 'Trust Wallet', provider: window.ethereum });
    }
  }
  
  // WalletConnect
  if (window.WalletConnect) {
    providers.push({ name: 'WalletConnect', provider: null });
  }
  
  return providers;
};

console.log('Available wallets:', detectProviders());
```
{: file="detectProviders.js" }

## MetaMask Entegrasyonu

> MetaMask kurulumunu kontrol etmeden önce, kullanıcıya anlaşılır bir mesaj gösterin ve kurulum linkini sağlayın.
{: .prompt-tip }

### Temel Bağlantı

```javascript
// Modern Web3 bağlantısı (ethers.js v6)
import { ethers } from 'ethers';

class WalletConnector {
  constructor() {
    this.provider = null;
    this.signer = null;
    this.address = null;
    this.chainId = null;
  }
  
  // MetaMask'i algıla ve bağlan
  async connect() {
    try {
      // Provider kontrolü
      if (!window.ethereum) {
        throw new Error('MetaMask is not installed');
      }
      
      // Hesap erişimi iste
      const accounts = await window.ethereum.request({
        method: 'eth_requestAccounts'
      });
      
      if (accounts.length === 0) {
        throw new Error('No accounts found');
      }
      
      // Provider ve signer oluştur
      this.provider = new ethers.BrowserProvider(window.ethereum);
      this.signer = await this.provider.getSigner();
      this.address = accounts[0];
      
      // Chain ID al
      const network = await this.provider.getNetwork();
      this.chainId = network.chainId;
      
      console.log('Connected:', {
        address: this.address,
        chainId: this.chainId
      });
      
      // Event listener'ları ayarla
      this.setupEventListeners();
      
      return {
        address: this.address,
        chainId: this.chainId
      };
      
    } catch (error) {
      console.error('Connection error:', error);
      throw error;
    }
  }
  
  // Bağlantıyı kes
  async disconnect() {
    this.provider = null;
    this.signer = null;
    this.address = null;
    this.chainId = null;
    
    console.log('Disconnected');
  }
  
  // Event listener'lar
  setupEventListeners() {
    // Hesap değişikliği
    window.ethereum.on('accountsChanged', (accounts) => {
      if (accounts.length === 0) {
        // Kullanıcı disconnect etti
        this.disconnect();
      } else {
        // Yeni hesaba geç
        this.address = accounts[0];
        console.log('Account changed:', this.address);
        window.location.reload(); // Sayfayı yenile
      }
    });
    
    // Ağ değişikliği
    window.ethereum.on('chainChanged', (chainId) => {
      console.log('Chain changed:', chainId);
      window.location.reload(); // Sayfayı yenile
    });
    
    // Bağlantı kopması
    window.ethereum.on('disconnect', (error) => {
      console.log('Disconnected:', error);
      this.disconnect();
    });
  }
  
  // Bakiye sorgulama
  async getBalance() {
    if (!this.provider || !this.address) {
      throw new Error('Not connected');
    }
    
    const balance = await this.provider.getBalance(this.address);
    return ethers.formatEther(balance);
  }
  
  // İşlem gönderme
  async sendTransaction(to, amount) {
    if (!this.signer) {
      throw new Error('Not connected');
    }
    
    try {
      const tx = await this.signer.sendTransaction({
        to: to,
        value: ethers.parseEther(amount)
      });
      
      console.log('Transaction sent:', tx.hash);
      
      // İşlemin onaylanmasını bekle
      const receipt = await tx.wait();
      console.log('Transaction confirmed:', receipt);
      
      return receipt;
      
    } catch (error) {
      console.error('Transaction error:', error);
      throw error;
    }
  }
}

// Kullanım
const wallet = new WalletConnector();

// Bağlan
await wallet.connect();

// Bakiye kontrol et
const balance = await wallet.getBalance();
console.log('Balance:', balance, 'ETH');

// İşlem gönder
// await wallet.sendTransaction('0x...', '0.01');
```
{: file="WalletConnector.js" }

> Ağ ve hesap değişikliklerinde sayfayı yenilemek yerine, state yönetimi ile dinamik güncelleme yapmanızı öneririz.
{: .prompt-tip }

### Ağ Yönetimi

> Kullanıcı yanlış ağda ise, otomatik ağ değiştirme önerisi sunun.
{: .prompt-info }

```javascript
class NetworkManager {
  // Desteklenen ağlar
  static NETWORKS = {
    ethereum: {
      chainId: '0x1',
      chainName: 'Ethereum Mainnet',
      nativeCurrency: {
        name: 'Ether',
        symbol: 'ETH',
        decimals: 18
      },
      rpcUrls: ['https://mainnet.infura.io/v3/'],
      blockExplorerUrls: ['https://etherscan.io']
    },
    polygon: {
      chainId: '0x89',
      chainName: 'Polygon Mainnet',
      nativeCurrency: {
        name: 'MATIC',
        symbol: 'MATIC',
        decimals: 18
      },
      rpcUrls: ['https://polygon-rpc.com/'],
      blockExplorerUrls: ['https://polygonscan.com/']
    },
    bsc: {
      chainId: '0x38',
      chainName: 'BNB Smart Chain',
      nativeCurrency: {
        name: 'BNB',
        symbol: 'BNB',
        decimals: 18
      },
      rpcUrls: ['https://bsc-dataseed.binance.org/'],
      blockExplorerUrls: ['https://bscscan.com']
    },
    arbitrum: {
      chainId: '0xa4b1',
      chainName: 'Arbitrum One',
      nativeCurrency: {
        name: 'Ether',
        symbol: 'ETH',
        decimals: 18
      },
      rpcUrls: ['https://arb1.arbitrum.io/rpc'],
      blockExplorerUrls: ['https://arbiscan.io']
    }
  };
  
  // Mevcut ağı kontrol et
  static async getCurrentChain() {
    const chainId = await window.ethereum.request({ 
      method: 'eth_chainId' 
    });
    return chainId;
  }
  
  // Ağ değiştir
  static async switchNetwork(networkKey) {
    const network = this.NETWORKS[networkKey];
    
    if (!network) {
      throw new Error('Network not supported');
    }
    
    try {
      // Önce ağı değiştirmeyi dene
      await window.ethereum.request({
        method: 'wallet_switchEthereumChain',
        params: [{ chainId: network.chainId }]
      });
      
      console.log(`Switched to ${network.chainName}`);
      
    } catch (switchError) {
      // Ağ eklenmemişse, ekle
      if (switchError.code === 4902) {
        try {
          await window.ethereum.request({
            method: 'wallet_addEthereumChain',
            params: [network]
          });
          
          console.log(`Added and switched to ${network.chainName}`);
          
        } catch (addError) {
          console.error('Error adding network:', addError);
          throw addError;
        }
      } else {
        console.error('Error switching network:', switchError);
        throw switchError;
      }
    }
  }
  
  // Ağın doğru olup olmadığını kontrol et
  static async ensureNetwork(requiredNetwork) {
    const currentChain = await this.getCurrentChain();
    const required = this.NETWORKS[requiredNetwork];
    
    if (currentChain !== required.chainId) {
      console.log(`Wrong network. Switching to ${required.chainName}...`);
      await this.switchNetwork(requiredNetwork);
    }
  }
}

// Kullanım
// Polygon'a geç
await NetworkManager.switchNetwork('polygon');

// Ethereum'da olduğundan emin ol
await NetworkManager.ensureNetwork('ethereum');
```
{: file="NetworkManager.js" }

> Her ağın farklı RPC endpoint'leri vardır. Production'da Infura, Alchemy gibi güvenilir sağlayıcılar kullanın.
{: .prompt-tip }

### Smart Contract Etkileşimi

> Gas tahmini yaparken %20-30 buffer ekleyin. Ağ yoğunluğuna göre gas fiyatları değişebilir.
{: .prompt-warning }

```javascript
class ContractInteraction {
  constructor(contractAddress, abi, signer) {
    this.contract = new ethers.Contract(contractAddress, abi, signer);
  }
  
  // Token bakiyesi (ERC-20)
  async getTokenBalance(address) {
    try {
      const balance = await this.contract.balanceOf(address);
      const decimals = await this.contract.decimals();
      return ethers.formatUnits(balance, decimals);
    } catch (error) {
      console.error('Error getting balance:', error);
      throw error;
    }
  }
  
  // Token transfer (ERC-20)
  async transferToken(to, amount) {
    try {
      // Decimal'i al
      const decimals = await this.contract.decimals();
      const value = ethers.parseUnits(amount, decimals);
      
      // Gas tahmini
      const gasEstimate = await this.contract.transfer.estimateGas(to, value);
      
      // İşlemi gönder
      const tx = await this.contract.transfer(to, value, {
        gasLimit: gasEstimate * 120n / 100n // %20 buffer
      });
      
      console.log('Transfer initiated:', tx.hash);
      
      // Onay bekle
      const receipt = await tx.wait();
      console.log('Transfer confirmed:', receipt);
      
      return receipt;
      
    } catch (error) {
      console.error('Transfer error:', error);
      throw error;
    }
  }
  
  // Approval (ERC-20)
  async approve(spender, amount) {
    try {
      const decimals = await this.contract.decimals();
      const value = ethers.parseUnits(amount, decimals);
      
      const tx = await this.contract.approve(spender, value);
      await tx.wait();
      
      console.log('Approval confirmed');
      return true;
      
    } catch (error) {
      console.error('Approval error:', error);
      throw error;
    }
  }
  
  // NFT mint
  async mintNFT(to, tokenURI) {
    try {
      // Gas estimate
      const gasEstimate = await this.contract.mint.estimateGas(to, tokenURI);
      
      // Mint transaction
      const tx = await this.contract.mint(to, tokenURI, {
        gasLimit: gasEstimate * 120n / 100n
      });
      
      console.log('Minting NFT:', tx.hash);
      
      const receipt = await tx.wait();
      
      // Event'ten token ID'yi çıkar
      const event = receipt.logs.find(
        log => log.fragment && log.fragment.name === 'Transfer'
      );
      
      const tokenId = event ? event.args[2] : null;
      
      console.log('NFT minted, token ID:', tokenId);
      
      return { receipt, tokenId };
      
    } catch (error) {
      console.error('Mint error:', error);
      throw error;
    }
  }
  
  // Read-only fonksiyon çağrısı
  async callReadFunction(functionName, ...args) {
    try {
      const result = await this.contract[functionName](...args);
      return result;
    } catch (error) {
      console.error(`Error calling ${functionName}:`, error);
      throw error;
    }
  }
  
  // Write fonksiyon çağrısı
  async callWriteFunction(functionName, options = {}, ...args) {
    try {
      // Gas estimate
      const gasEstimate = await this.contract[functionName].estimateGas(...args);
      
      // İşlem gönder
      const tx = await this.contract[functionName](...args, {
        gasLimit: gasEstimate * 120n / 100n,
        ...options
      });
      
      console.log(`${functionName} called:`, tx.hash);
      
      const receipt = await tx.wait();
      console.log(`${functionName} confirmed`);
      
      return receipt;
      
    } catch (error) {
      console.error(`Error calling ${functionName}:`, error);
      throw error;
    }
  }
}

// ERC-20 Token ABI (sadece gerekli fonksiyonlar)
const ERC20_ABI = [
  'function balanceOf(address) view returns (uint256)',
  'function decimals() view returns (uint8)',
  'function transfer(address to, uint256 amount) returns (bool)',
  'function approve(address spender, uint256 amount) returns (bool)',
  'function allowance(address owner, address spender) view returns (uint256)'
];

// Kullanım
const wallet = new WalletConnector();
await wallet.connect();

const tokenContract = new ContractInteraction(
  '0x...', // Token address
  ERC20_ABI,
  wallet.signer
);

const balance = await tokenContract.getTokenBalance(wallet.address);
console.log('Token balance:', balance);
```
{: file="contractUsage.js" }

> Smart contract fonksiyonlarını çağırmadan önce mutlaka gas tahmini yapın ve kullanıcıya işlem maliyetini gösterin.
{: .prompt-warning }

## WalletConnect Entegrasyonu

![Web3 Development](/assets/img/posts/web3py-ethereum-development.png){: w="700" h="400" .shadow }
_Web3 geliştirme workflow ve bağlantı katmanları_

> WalletConnect, mobile wallet'lar için QR kod tabanlı bağlantı sağlar. WalletConnect Cloud'dan ücretsiz Project ID alın.
{: .prompt-info }

### WalletConnect v2 Setup

```javascript
// WalletConnect v2 kurulumu
import { EthereumProvider } from '@walletconnect/ethereum-provider';
import { ethers } from 'ethers';

class WalletConnectManager {
  constructor() {
    this.provider = null;
    this.signer = null;
    this.address = null;
  }
  
  async initialize() {
    try {
      // WalletConnect provider oluştur
      this.provider = await EthereumProvider.init({
        projectId: 'YOUR_PROJECT_ID', // WalletConnect Cloud'dan al
        chains: [1], // Ethereum Mainnet
        optionalChains: [137, 56, 42161], // Polygon, BSC, Arbitrum
        showQrModal: true,
        qrModalOptions: {
          themeMode: 'dark',
          themeVariables: {
            '--wcm-z-index': '1000'
          }
        },
        metadata: {
          name: 'My DApp',
          description: 'My DApp Description',
          url: 'https://mydapp.com',
          icons: ['https://mydapp.com/icon.png']
        }
      });
      
      // Event listener'ları ayarla
      this.setupEventListeners();
      
      console.log('WalletConnect initialized');
      
    } catch (error) {
      console.error('WalletConnect init error:', error);
      throw error;
    }
  }
  
  async connect() {
    try {
      if (!this.provider) {
        await this.initialize();
      }
      
      // Bağlantıyı aç (QR kod modal gösterir)
      await this.provider.enable();
      
      // Ethers provider oluştur
      const ethersProvider = new ethers.BrowserProvider(this.provider);
      this.signer = await ethersProvider.getSigner();
      this.address = await this.signer.getAddress();
      
      console.log('WalletConnect connected:', this.address);
      
      return {
        address: this.address,
        chainId: this.provider.chainId
      };
      
    } catch (error) {
      console.error('WalletConnect connection error:', error);
      throw error;
    }
  }
  
  async disconnect() {
    try {
      if (this.provider) {
        await this.provider.disconnect();
      }
      
      this.provider = null;
      this.signer = null;
      this.address = null;
      
      console.log('WalletConnect disconnected');
      
    } catch (error) {
      console.error('Disconnect error:', error);
      throw error;
    }
  }
  
  setupEventListeners() {
    this.provider.on('accountsChanged', (accounts) => {
      console.log('Accounts changed:', accounts);
      if (accounts.length > 0) {
        this.address = accounts[0];
      } else {
        this.disconnect();
      }
    });
    
    this.provider.on('chainChanged', (chainId) => {
      console.log('Chain changed:', chainId);
    });
    
    this.provider.on('disconnect', () => {
      console.log('Provider disconnected');
      this.disconnect();
    });
  }
  
  async switchChain(chainId) {
    try {
      await this.provider.request({
        method: 'wallet_switchEthereumChain',
        params: [{ chainId: `0x${chainId.toString(16)}` }]
      });
    } catch (error) {
      console.error('Switch chain error:', error);
      throw error;
    }
  }
}

// Kullanım
const wcManager = new WalletConnectManager();
await wcManager.connect();
```
{: file="WalletConnectManager.js" }

### Multi-Wallet Desteği

```javascript
// Birden fazla wallet seçeneği sunan manager
class MultiWalletManager {
  constructor() {
    this.currentProvider = null;
    this.walletType = null;
  }
  
  // MetaMask ile bağlan
  async connectMetaMask() {
    if (!window.ethereum) {
      throw new Error('MetaMask not installed');
    }
    
    const accounts = await window.ethereum.request({
      method: 'eth_requestAccounts'
    });
    
    this.currentProvider = new ethers.BrowserProvider(window.ethereum);
    this.walletType = 'metamask';
    
    return {
      address: accounts[0],
      provider: this.currentProvider,
      type: this.walletType
    };
  }
  
  // WalletConnect ile bağlan
  async connectWalletConnect() {
    const wcManager = new WalletConnectManager();
    const result = await wcManager.connect();
    
    this.currentProvider = wcManager.provider;
    this.walletType = 'walletconnect';
    
    return {
      address: result.address,
      provider: this.currentProvider,
      type: this.walletType
    };
  }
  
  // Coinbase Wallet ile bağlan
  async connectCoinbase() {
    if (!window.ethereum?.isCoinbaseWallet) {
      throw new Error('Coinbase Wallet not installed');
    }
    
    const accounts = await window.ethereum.request({
      method: 'eth_requestAccounts'
    });
    
    this.currentProvider = new ethers.BrowserProvider(window.ethereum);
    this.walletType = 'coinbase';
    
    return {
      address: accounts[0],
      provider: this.currentProvider,
      type: this.walletType
    };
  }
  
  // Wallet seçimi modal
  async showWalletModal() {
    return new Promise((resolve, reject) => {
      // Modal HTML'i oluştur
      const modal = document.createElement('div');
      modal.className = 'wallet-modal';
      modal.innerHTML = `
        <div class="wallet-modal-content">
          <h2>Connect Wallet</h2>
          <button class="wallet-option" data-wallet="metamask">
            <img src="/icons/metamask.svg" alt="MetaMask">
            MetaMask
          </button>
          <button class="wallet-option" data-wallet="walletconnect">
            <img src="/icons/walletconnect.svg" alt="WalletConnect">
            WalletConnect
          </button>
          <button class="wallet-option" data-wallet="coinbase">
            <img src="/icons/coinbase.svg" alt="Coinbase">
            Coinbase Wallet
          </button>
          <button class="close-modal">Close</button>
        </div>
      `;
      
      document.body.appendChild(modal);
      
      // Event listener'lar
      modal.querySelectorAll('.wallet-option').forEach(button => {
        button.addEventListener('click', async () => {
          const walletType = button.dataset.wallet;
          document.body.removeChild(modal);
          
          try {
            let result;
            switch (walletType) {
              case 'metamask':
                result = await this.connectMetaMask();
                break;
              case 'walletconnect':
                result = await this.connectWalletConnect();
                break;
              case 'coinbase':
                result = await this.connectCoinbase();
                break;
            }
            resolve(result);
          } catch (error) {
            reject(error);
          }
        });
      });
      
      modal.querySelector('.close-modal').addEventListener('click', () => {
        document.body.removeChild(modal);
        reject(new Error('User cancelled'));
      });
    });
  }
}

// CSS (örnek)
const walletModalCSS = `
.wallet-modal {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.8);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
}

.wallet-modal-content {
  background: white;
  padding: 2rem;
  border-radius: 12px;
  max-width: 400px;
  width: 90%;
}

.wallet-option {
  width: 100%;
  padding: 1rem;
  margin: 0.5rem 0;
  display: flex;
  align-items: center;
  gap: 1rem;
  border: 2px solid #e0e0e0;
  border-radius: 8px;
  background: white;
  cursor: pointer;
  transition: all 0.2s;
}

.wallet-option:hover {
  border-color: #4CAF50;
  background: #f5f5f5;
}

.wallet-option img {
  width: 32px;
  height: 32px;
}
`;

// Kullanım
const multiWallet = new MultiWalletManager();
const connection = await multiWallet.showWalletModal();
console.log('Connected with:', connection.type);
```
{: file="MultiWalletManager.js" }

> Kullanıcılara birden fazla wallet seçeneği sunmak, DApp erişilebilirliğini artırır. Web3Modal gibi hazır çözümler de kullanabilirsiniz.
{: .prompt-tip }

## React Component Implementasyonu

```jsx
// React ile wallet bağlantısı
import { createContext, useContext, useState, useEffect } from 'react';
import { ethers } from 'ethers';

// Wallet Context
const WalletContext = createContext();

export const useWallet = () => {
  const context = useContext(WalletContext);
  if (!context) {
    throw new Error('useWallet must be used within WalletProvider');
  }
  return context;
};

// Wallet Provider
export const WalletProvider = ({ children }) => {
  const [account, setAccount] = useState(null);
  const [chainId, setChainId] = useState(null);
  const [balance, setBalance] = useState('0');
  const [provider, setProvider] = useState(null);
  const [signer, setSigner] = useState(null);
  const [isConnecting, setIsConnecting] = useState(false);
  const [error, setError] = useState(null);
  
  // Wallet'ı bağla
  const connect = async () => {
    try {
      setIsConnecting(true);
      setError(null);
      
      if (!window.ethereum) {
        throw new Error('MetaMask not installed');
      }
      
      // Hesap erişimi iste
      const accounts = await window.ethereum.request({
        method: 'eth_requestAccounts'
      });
      
      if (accounts.length === 0) {
        throw new Error('No accounts found');
      }
      
      // Provider ve signer ayarla
      const web3Provider = new ethers.BrowserProvider(window.ethereum);
      const web3Signer = await web3Provider.getSigner();
      const network = await web3Provider.getNetwork();
      
      setProvider(web3Provider);
      setSigner(web3Signer);
      setAccount(accounts[0]);
      setChainId(Number(network.chainId));
      
      // Bakiye al
      await updateBalance(accounts[0], web3Provider);
      
    } catch (err) {
      console.error('Connection error:', err);
      setError(err.message);
    } finally {
      setIsConnecting(false);
    }
  };
  
  // Bağlantıyı kes
  const disconnect = () => {
    setAccount(null);
    setChainId(null);
    setBalance('0');
    setProvider(null);
    setSigner(null);
    setError(null);
  };
  
  // Bakiye güncelle
  const updateBalance = async (address, web3Provider) => {
    try {
      const bal = await web3Provider.getBalance(address);
      setBalance(ethers.formatEther(bal));
    } catch (err) {
      console.error('Balance error:', err);
    }
  };
  
  // Event listener'ları ayarla
  useEffect(() => {
    if (!window.ethereum) return;
    
    const handleAccountsChanged = (accounts) => {
      if (accounts.length === 0) {
        disconnect();
      } else if (accounts[0] !== account) {
        setAccount(accounts[0]);
        if (provider) {
          updateBalance(accounts[0], provider);
        }
      }
    };
    
    const handleChainChanged = (newChainId) => {
      setChainId(parseInt(newChainId, 16));
      window.location.reload();
    };
    
    const handleDisconnect = () => {
      disconnect();
    };
    
    window.ethereum.on('accountsChanged', handleAccountsChanged);
    window.ethereum.on('chainChanged', handleChainChanged);
    window.ethereum.on('disconnect', handleDisconnect);
    
    return () => {
      window.ethereum.removeListener('accountsChanged', handleAccountsChanged);
      window.ethereum.removeListener('chainChanged', handleChainChanged);
      window.ethereum.removeListener('disconnect', handleDisconnect);
    };
  }, [account, provider]);
  
  // Sayfa yüklendiğinde bağlantıyı kontrol et
  useEffect(() => {
    const checkConnection = async () => {
      if (window.ethereum) {
        const accounts = await window.ethereum.request({
          method: 'eth_accounts'
        });
        
        if (accounts.length > 0) {
          await connect();
        }
      }
    };
    
    checkConnection();
  }, []);
  
  const value = {
    account,
    chainId,
    balance,
    provider,
    signer,
    isConnecting,
    error,
    connect,
    disconnect,
    updateBalance: () => updateBalance(account, provider)
  };
  
  return (
    <WalletContext.Provider value={value}>
      {children}
    </WalletContext.Provider>
  );
};

// Wallet Button Component
export const WalletButton = () => {
  const { account, balance, isConnecting, connect, disconnect } = useWallet();
  
  const formatAddress = (address) => {
    return `${address.substring(0, 6)}...${address.substring(38)}`;
  };
  
  if (account) {
    return (
      <div className="wallet-info">
        <div className="balance">{parseFloat(balance).toFixed(4)} ETH</div>
        <button className="wallet-address" onClick={disconnect}>
          {formatAddress(account)}
        </button>
      </div>
    );
  }
  
  return (
    <button 
      className="connect-button" 
      onClick={connect}
      disabled={isConnecting}
    >
      {isConnecting ? 'Connecting...' : 'Connect Wallet'}
    </button>
  );
};

// Network Switcher Component
export const NetworkSwitcher = () => {
  const { chainId, provider } = useWallet();
  const [switching, setSwitching] = useState(false);
  
  const networks = {
    1: { name: 'Ethereum', color: '#627EEA' },
    137: { name: 'Polygon', color: '#8247E5' },
    56: { name: 'BSC', color: '#F3BA2F' },
    42161: { name: 'Arbitrum', color: '#28A0F0' }
  };
  
  const switchNetwork = async (targetChainId) => {
    if (!provider) return;
    
    try {
      setSwitching(true);
      
      await window.ethereum.request({
        method: 'wallet_switchEthereumChain',
        params: [{ chainId: `0x${targetChainId.toString(16)}` }]
      });
      
    } catch (err) {
      console.error('Switch network error:', err);
    } finally {
      setSwitching(false);
    }
  };
  
  return (
    <div className="network-switcher">
      <div className="current-network" style={ {% raw %}{{ background: networks[chainId]?.color }}{% endraw %} }>
        {networks[chainId]?.name || `Chain ${chainId}`}
      </div>
      
      <div className="network-options">
        {Object.entries(networks).map(([id, network]) => (
          <button
            key={id}
            onClick={() => switchNetwork(parseInt(id))}
            disabled={switching || chainId === parseInt(id)}
            style={ {% raw %}{{ borderColor: network.color }}{% endraw %} }
          >
            {network.name}
          </button>
        ))}
      </div>
    </div>
  );
};

// App.jsx
function App() {
  return (
    <WalletProvider>
      <div className="app">
        <header>
          <h1>My DApp</h1>
          <div className="header-actions">
            <NetworkSwitcher />
            <WalletButton />
          </div>
        </header>
        
        <main>
          <YourDAppContent />
        </main>
      </div>
    </WalletProvider>
  );
}
```
{: file="App.jsx" }

> React Context API ile global wallet state yönetimi yaparak, tüm componentlerde wallet bilgilerine erişebilirsiniz.
{: .prompt-tip }

## Mesaj İmzalama ve Doğrulama

![Python Web3](/assets/img/posts/python-web3-blockchain.png)
*Şekil 3: Web3 blockchain etkileşim katmanları*

```javascript
class MessageSigning {
  constructor(signer) {
    this.signer = signer;
  }
  
  // Basit mesaj imzalama
  async signMessage(message) {
    try {
      const signature = await this.signer.signMessage(message);
      console.log('Message signed:', signature);
      return signature;
    } catch (error) {
      console.error('Signing error:', error);
      throw error;
    }
  }
  
  // Mesaj imzasını doğrulama
  static verifyMessage(message, signature) {
    try {
      const recoveredAddress = ethers.verifyMessage(message, signature);
      console.log('Signer address:', recoveredAddress);
      return recoveredAddress;
    } catch (error) {
      console.error('Verification error:', error);
      throw error;
    }
  }
  
  // Structured data imzalama (EIP-712)
  async signTypedData(domain, types, value) {
    try {
      // EIP-712 typed data signature
      const signature = await this.signer.signTypedData(domain, types, value);
      console.log('Typed data signed:', signature);
      return signature;
    } catch (error) {
      console.error('Typed data signing error:', error);
      throw error;
    }
  }
  
  // Login için imza oluşturma
  async signLoginMessage(nonce) {
    const message = `Sign this message to authenticate with nonce: ${nonce}`;
    return await this.signMessage(message);
  }
  
  // Transaction için imza
  async signTransaction(tx) {
    try {
      const signedTx = await this.signer.signTransaction(tx);
      return signedTx;
    } catch (error) {
      console.error('Transaction signing error:', error);
      throw error;
    }
  }
}

// EIP-712 örneği (NFT permit)
const domain = {
  name: 'MyNFT',
  version: '1',
  chainId: 1,
  verifyingContract: '0x...'
};

const types = {
  Permit: [
    { name: 'owner', type: 'address' },
    { name: 'spender', type: 'address' },
    { name: 'tokenId', type: 'uint256' },
    { name: 'nonce', type: 'uint256' },
    { name: 'deadline', type: 'uint256' }
  ]
};

const value = {
  owner: '0x...',
  spender: '0x...',
  tokenId: 1,
  nonce: 0,
  deadline: Math.floor(Date.now() / 1000) + 3600
};

// Kullanım
const wallet = new WalletConnector();
await wallet.connect();

const messageSigning = new MessageSigning(wallet.signer);

// Basit mesaj imzala
const signature = await messageSigning.signMessage('Hello Web3!');

// İmzayı doğrula
const signer = MessageSigning.verifyMessage('Hello Web3!', signature);
console.log('Verified signer:', signer);

// EIP-712 imzala
const typedSignature = await messageSigning.signTypedData(domain, types, value);
```
{: file="messageSigning.js" }

> Mesaj imzalama, kimlik doğrulama için güvenli bir yöntemdir ancak kullanıcıya imzaladığı mesajı açıkça gösterin.
{: .prompt-warning }

## Güvenlik Best Practices

> Web3 güvenliği kritiktir! Private key'ler asla frontend'de saklanmamalı ve tüm işlemler HTTPS üzerinden yapılmalıdır.
{: .prompt-danger }

```javascript
class SecureWalletManager {
  constructor() {
    this.maxRetries = 3;
    this.retryDelay = 1000;
  }
  
  // Güvenli bağlantı kontrolü
  async secureConnect() {
    // HTTPS kontrolü
    if (window.location.protocol !== 'https:' && 
        window.location.hostname !== 'localhost') {
      throw new Error('HTTPS required for wallet connection');
    }
    
    // Provider kontrolü
    if (!window.ethereum) {
      throw new Error('No wallet provider found');
    }
    
    // Phishing kontrolü
    if (this.detectPhishing()) {
      throw new Error('Potential phishing attempt detected');
    }
    
    return await this.connectWithRetry();
  }
  
  // Phishing algılama
  detectPhishing() {
    // Şüpheli URL pattern'leri
    const suspiciousPatterns = [
      /metamask.*\.com(?!$)/,
      /.*-metamask\.com/,
      /.*metamask-.*\.com/
    ];
    
    const currentUrl = window.location.href.toLowerCase();
    
    return suspiciousPatterns.some(pattern => pattern.test(currentUrl));
  }
  
  // Retry mekanizması ile bağlantı
  async connectWithRetry() {
    for (let i = 0; i < this.maxRetries; i++) {
      try {
        const accounts = await window.ethereum.request({
          method: 'eth_requestAccounts'
        });
        
        return accounts[0];
        
      } catch (error) {
        if (error.code === 4001) {
          // Kullanıcı reddetti
          throw error;
        }
        
        if (i === this.maxRetries - 1) {
          throw error;
        }
        
        // Bekle ve tekrar dene
        await new Promise(resolve => setTimeout(resolve, this.retryDelay));
      }
    }
  }
  
  // İşlem güvenlik kontrolleri
  async secureTransaction(tx) {
    // Gas price kontrolü (sandwich attack önleme)
    const gasPrice = await this.getGasPrice();
    if (tx.gasPrice && tx.gasPrice > gasPrice * 1.5) {
      console.warn('Gas price too high, possible attack');
    }
    
    // Değer kontrolü
    if (tx.value) {
      const balance = await this.getBalance(tx.from);
      if (tx.value > balance) {
        throw new Error('Insufficient balance');
      }
    }
    
    // Contract adresi kontrolü
    if (tx.to) {
      const isContract = await this.isContractAddress(tx.to);
      if (isContract) {
        console.info('Interacting with smart contract');
      }
    }
    
    return true;
  }
  
  // Rate limiting
  rateLimit = {
    requests: [],
    maxRequests: 10,
    windowMs: 60000 // 1 minute
  };
  
  checkRateLimit() {
    const now = Date.now();
    
    // Eski istekleri temizle
    this.rateLimit.requests = this.rateLimit.requests.filter(
      time => now - time < this.rateLimit.windowMs
    );
    
    // Limit kontrolü
    if (this.rateLimit.requests.length >= this.rateLimit.maxRequests) {
      throw new Error('Rate limit exceeded');
    }
    
    // Yeni istek ekle
    this.rateLimit.requests.push(now);
  }
  
  // Input sanitization
  sanitizeAddress(address) {
    // Address formatını kontrol et
    if (!/^0x[a-fA-F0-9]{40}$/.test(address)) {
      throw new Error('Invalid address format');
    }
    
    // Checksum kontrolü
    try {
      return ethers.getAddress(address);
    } catch {
      throw new Error('Invalid address checksum');
    }
  }
  
  // Contract adres kontrolü
  async isContractAddress(address) {
    const code = await window.ethereum.request({
      method: 'eth_getCode',
      params: [address, 'latest']
    });
    
    return code !== '0x';
  }
  
  async getGasPrice() {
    return await window.ethereum.request({
      method: 'eth_gasPrice'
    });
  }
  
  async getBalance(address) {
    const balance = await window.ethereum.request({
      method: 'eth_getBalance',
      params: [address, 'latest']
    });
    
    return BigInt(balance);
  }
}

// Kullanım
const secureWallet = new SecureWalletManager();

try {
  // Güvenli bağlantı
  const account = await secureWallet.secureConnect();
  console.log('Securely connected:', account);
  
  // İşlem öncesi kontrol
  const tx = {
    from: account,
    to: secureWallet.sanitizeAddress('0x...'),
    value: ethers.parseEther('0.1')
  };
  
  await secureWallet.secureTransaction(tx);
  
} catch (error) {
  console.error('Security error:', error);
}
```
{: file="SecureWalletManager.js" }

## Error Handling ve User Experience

> Kullanıcı dostu error mesajları gösterin! Teknik hataları kullanıcının anlayabileceği dile çevirin.
{: .prompt-tip }

```javascript
class WalletErrorHandler {
  static ERROR_MESSAGES = {
    // User errors
    4001: 'İşlem kullanıcı tarafından reddedildi',
    4100: 'İstenen method desteklenmiyor',
    4200: 'Provider izni yok',
    4900: 'Provider bağlı değil',
    4901: 'Provider bağlantısı kesildi',
    
    // RPC errors
    '-32700': 'JSON parse hatası',
    '-32600': 'Geçersiz istek',
    '-32601': 'Method bulunamadı',
    '-32602': 'Geçersiz parametreler',
    '-32603': 'Internal hata',
    
    // Custom errors
    'NETWORK_ERROR': 'Ağ bağlantı hatası',
    'INSUFFICIENT_FUNDS': 'Yetersiz bakiye',
    'GAS_LIMIT': 'Gas limiti aşıldı',
    'NONCE_TOO_LOW': 'Nonce çok düşük',
    'REPLACEMENT_UNDERPRICED': 'Gas fiyatı çok düşük'
  };
  
  static handleError(error) {
    console.error('Wallet error:', error);
    
    let userMessage = 'Bir hata oluştu';
    let errorType = 'unknown';
    
    // Error code kontrolü
    if (error.code) {
      userMessage = this.ERROR_MESSAGES[error.code] || error.message;
      errorType = error.code;
    }
    
    // Error message pattern matching
    if (error.message) {
      if (error.message.includes('insufficient funds')) {
        userMessage = this.ERROR_MESSAGES.INSUFFICIENT_FUNDS;
        errorType = 'INSUFFICIENT_FUNDS';
      } else if (error.message.includes('gas')) {
        userMessage = this.ERROR_MESSAGES.GAS_LIMIT;
        errorType = 'GAS_LIMIT';
      } else if (error.message.includes('nonce')) {
        userMessage = this.ERROR_MESSAGES.NONCE_TOO_LOW;
        errorType = 'NONCE_TOO_LOW';
      }
    }
    
    // User notification
    this.showNotification(userMessage, 'error');
    
    return {
      type: errorType,
      message: userMessage,
      originalError: error
    };
  }
  
  static showNotification(message, type = 'info') {
    // Toast notification göster
    const toast = document.createElement('div');
    toast.className = `toast toast-${type}`;
    toast.textContent = message;
    
    document.body.appendChild(toast);
    
    setTimeout(() => {
      toast.classList.add('show');
    }, 100);
    
    setTimeout(() => {
      toast.classList.remove('show');
      setTimeout(() => {
        document.body.removeChild(toast);
      }, 300);
    }, 3000);
  }
  
  // Loading state management
  static async withLoading(operation, loadingMessage = 'İşlem yapılıyor...') {
    const loader = this.showLoader(loadingMessage);
    
    try {
      const result = await operation();
      this.hideLoader(loader);
      this.showNotification('İşlem başarılı!', 'success');
      return result;
    } catch (error) {
      this.hideLoader(loader);
      this.handleError(error);
      throw error;
    }
  }
  
  static showLoader(message) {
    const loader = document.createElement('div');
    loader.className = 'loader-overlay';
    loader.innerHTML = `
      <div class="loader-content">
        <div class="spinner"></div>
        <p>${message}</p>
      </div>
    `;
    
    document.body.appendChild(loader);
    return loader;
  }
  
  static hideLoader(loader) {
    if (loader && loader.parentNode) {
      loader.parentNode.removeChild(loader);
    }
  }
}

// Kullanım
async function transferTokens(to, amount) {
  try {
    const result = await WalletErrorHandler.withLoading(
      async () => {
        // İşlem
        const tx = await contract.transfer(to, amount);
        await tx.wait();
        return tx;
      },
      'Token transfer ediliyor...'
    );
    
    console.log('Transfer successful:', result);
    
  } catch (error) {
    // Error otomatik handle edildi
    console.log('Transfer failed');
  }
}
```
{: file="WalletErrorHandler.js" }

## Sonuç

Web3 wallet entegrasyonu, modern DApp geliştirmenin temel taşıdır. Bu rehberde öğrendikleriniz:

1. **MetaMask Entegrasyonu**: Temel bağlantı, ağ yönetimi ve işlem gönderme
2. **WalletConnect**: Çoklu cüzdan desteği ve mobil uyumluluk
3. **Smart Contract Etkileşimi**: Token transfer, NFT minting ve contract çağrıları
4. **React Integration**: Context API ile state management ve component yapısı
5. **Güvenlik**: Phishing koruması, rate limiting ve input validation
6. **User Experience**: Error handling, loading states ve bildirimler

**Best Practices:**

- ✅ Her zaman HTTPS kullanın
- ✅ User input'larını validate edin
- ✅ Error handling implement edin
- ✅ Loading states gösterin
- ✅ Network değişikliklerini handle edin
- ✅ Rate limiting uygulayın
- ✅ Gas estimation yapın
- ✅ Transaction confirmation bekleyin

Web3 ecosystem sürekli gelişiyor. Yeni cüzdan çözümleri, improved security patterns ve better UX paradigms ortaya çıkıyor. Bu rehberdeki temel prensipleri öğrenerek, herhangi bir Web3 projesine güvenle entegrasyon yapabilirsiniz.

## Kaynaklar

### Resmi Dokümantasyonlar
- [MetaMask Developer Docs](https://docs.metamask.io/)
- [WalletConnect Documentation](https://docs.walletconnect.com/)
- [Ethers.js Documentation](https://docs.ethers.org/)
- [Web3.js Documentation](https://web3js.readthedocs.io/)

### Kütüphaneler
- [ethers.js](https://github.com/ethers-io/ethers.js/) - Ethereum library
- [WalletConnect v2](https://github.com/WalletConnect/walletconnect-monorepo) - Multi-wallet support
- [wagmi](https://wagmi.sh/) - React hooks for Ethereum
- [web3modal](https://github.com/WalletConnect/web3modal) - Multi-provider modal

### Best Practices
- [EIP-1193](https://eips.ethereum.org/EIPS/eip-1193) - Ethereum Provider API
- [EIP-712](https://eips.ethereum.org/EIPS/eip-712) - Typed structured data hashing
- [MetaMask Security Best Practices](https://docs.metamask.io/guide/common-terms.html#security-best-practices)

### Öğrenme Kaynakları
- [useWeb3](https://www.useweb3.xyz/) - Web3 development resources
- [LearnWeb3](https://learnweb3.io/) - Comprehensive Web3 courses
- [Ethereum.org Developer Portal](https://ethereum.org/en/developers/)

### Topluluklar
- [r/ethdev](https://www.reddit.com/r/ethdev/)
- [Ethereum Stack Exchange](https://ethereum.stackexchange.com/)
- [WalletConnect Discord](https://discord.gg/walletconnect)

Başarılı DApp geliştirmeler!
