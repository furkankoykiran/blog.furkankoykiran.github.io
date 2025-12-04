---
title: "Smart Contract Testing with Hardhat and Ethers.js"
description: "Comprehensive guide to smart contract testing with Hardhat and Ethers.js. Unit tests, integration testing, gas optimization, and best practices for secure Solidity development."
date: "2024-08-19"
categories:
  - "ethereum"
  - "testing"
tags:
  - "hardhat"
  - "ethers"
  - "smart-contracts"
  - "testing"
  - "solidity"
  - "web3"
  - "development"
image:
  path: "/assets/img/posts/smart-contract-testing-tools.png"
  alt: "Smart Contract Testing Tools and Framework"
---

Testing is crucial in smart contract development. A single bug can lead to millions of dollars in losses, as we've seen in countless DeFi hacks. In this comprehensive guide, we'll explore how to build a robust testing suite using Hardhat and Ethers.js, covering everything from basic unit tests to advanced integration testing and gas optimization.

## Why Testing Matters

Smart contracts are immutable once deployed. Unlike traditional software where you can patch bugs, blockchain code requires extreme care during development. Proper testing:

- **Prevents Financial Loss**: Catch bugs before they cost money
- **Ensures Correctness**: Verify logic works as intended
- **Builds Confidence**: Deploy with certainty
- **Saves Gas**: Optimize before mainnet deployment
- **Enables Refactoring**: Change code safely

> Smart contracts are immutable. A single bug can cost millions. Always test thoroughly before deployment!
{: .prompt-danger }

## Setting Up Hardhat

Hardhat is a development environment for Ethereum that makes it easy to compile, deploy, test, and debug smart contracts.

### Installation

```bash
# Create new project
mkdir my-defi-project
cd my-defi-project
npm init -y

# Install Hardhat
npm install --save-dev hardhat

# Initialize Hardhat
npx hardhat init
# Select: Create a JavaScript project

# Install additional dependencies
npm install --save-dev @nomicfoundation/hardhat-toolbox
npm install --save-dev @nomicfoundation/hardhat-chai-matchers
npm install --save-dev @nomiclabs/hardhat-ethers
npm install --save-dev ethers
npm install --save-dev chai
```

### Project Structure

```
my-defi-project/
├── contracts/
│   └── Token.sol
├── scripts/
│   └── deploy.js
├── test/
│   └── Token.test.js
├── hardhat.config.js
└── package.json
```

### Hardhat Configuration

```javascript
// hardhat.config.js
require("@nomicfoundation/hardhat-toolbox");
require("@nomiclabs/hardhat-ethers");

module.exports = {
  solidity: {
    version: "0.8.19",
    settings: {
      optimizer: {
        enabled: true,
        runs: 200
      }
    }
  },
  networks: {
    hardhat: {
      chainId: 31337,
      mining: {
        auto: true,
        interval: 0
      }
    },
    localhost: {
      url: "http://127.0.0.1:8545"
    },
    goerli: {
      url: process.env.GOERLI_RPC_URL || "",
      accounts: process.env.PRIVATE_KEY ? [process.env.PRIVATE_KEY] : []
    }
  },
  gasReporter: {
    enabled: process.env.REPORT_GAS === "true",
    currency: "USD",
    coinmarketcap: process.env.COINMARKETCAP_API_KEY
  },
  etherscan: {
    apiKey: process.env.ETHERSCAN_API_KEY
  }
};
```

## Sample Smart Contract

Let's create a token contract to test:

```solidity
// contracts/Token.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

contract Token {
    string public name;
    string public symbol;
    uint8 public decimals;
    uint256 public totalSupply;
    
    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;
    
    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);
    
    constructor(string memory _name, string memory _symbol, uint256 _initialSupply) {
        name = _name;
        symbol = _symbol;
        decimals = 18;
        totalSupply = _initialSupply * 10 ** decimals;
        balanceOf[msg.sender] = totalSupply;
        emit Transfer(address(0), msg.sender, totalSupply);
    }
    
    function transfer(address _to, uint256 _value) public returns (bool success) {
        require(_to != address(0), "Invalid address");
        require(balanceOf[msg.sender] >= _value, "Insufficient balance");
        
        balanceOf[msg.sender] -= _value;
        balanceOf[_to] += _value;
        
        emit Transfer(msg.sender, _to, _value);
        return true;
    }
    
    function approve(address _spender, uint256 _value) public returns (bool success) {
        require(_spender != address(0), "Invalid address");
        
        allowance[msg.sender][_spender] = _value;
        emit Approval(msg.sender, _spender, _value);
        return true;
    }
    
    function transferFrom(address _from, address _to, uint256 _value) public returns (bool success) {
        require(_to != address(0), "Invalid address");
        require(balanceOf[_from] >= _value, "Insufficient balance");
        require(allowance[_from][msg.sender] >= _value, "Allowance exceeded");
        
        balanceOf[_from] -= _value;
        balanceOf[_to] += _value;
        allowance[_from][msg.sender] -= _value;
        
        emit Transfer(_from, _to, _value);
        return true;
    }
}
```

## Basic Unit Tests

### Test Structure

```javascript
// test/Token.test.js
const { expect } = require("chai");
const { ethers } = require("hardhat");

describe("Token Contract", function () {
  let Token;
  let token;
  let owner;
  let addr1;
  let addr2;
  let addrs;

  // Deploy contract before each test
  beforeEach(async function () {
    Token = await ethers.getContractFactory("Token");
    [owner, addr1, addr2, ...addrs] = await ethers.getSigners();
    
    token = await Token.deploy("MyToken", "MTK", 1000000);
    await token.deployed();
  });

  describe("Deployment", function () {
    it("Should set the right owner", async function () {
      const ownerBalance = await token.balanceOf(owner.address);
      expect(await token.totalSupply()).to.equal(ownerBalance);
    });

    it("Should assign the total supply of tokens to the owner", async function () {
      const ownerBalance = await token.balanceOf(owner.address);
      const totalSupply = await token.totalSupply();
      expect(totalSupply).to.equal(ownerBalance);
    });

    it("Should have correct name and symbol", async function () {
      expect(await token.name()).to.equal("MyToken");
      expect(await token.symbol()).to.equal("MTK");
    });

    it("Should have 18 decimals", async function () {
      expect(await token.decimals()).to.equal(18);
    });
  });

  describe("Transactions", function () {
    it("Should transfer tokens between accounts", async function () {
      const amount = ethers.utils.parseEther("100");
      
      // Transfer from owner to addr1
      await token.transfer(addr1.address, amount);
      const addr1Balance = await token.balanceOf(addr1.address);
      expect(addr1Balance).to.equal(amount);

      // Transfer from addr1 to addr2
      await token.connect(addr1).transfer(addr2.address, amount);
      const addr2Balance = await token.balanceOf(addr2.address);
      expect(addr2Balance).to.equal(amount);
    });

    it("Should fail if sender doesn't have enough tokens", async function () {
      const initialOwnerBalance = await token.balanceOf(owner.address);
      const overAmount = initialOwnerBalance.add(1);

      await expect(
        token.connect(addr1).transfer(owner.address, overAmount)
      ).to.be.revertedWith("Insufficient balance");

      expect(await token.balanceOf(owner.address)).to.equal(initialOwnerBalance);
    });

    it("Should update balances after transfers", async function () {
      const initialOwnerBalance = await token.balanceOf(owner.address);
      const amount = ethers.utils.parseEther("100");

      await token.transfer(addr1.address, amount);
      await token.transfer(addr2.address, amount);

      const finalOwnerBalance = await token.balanceOf(owner.address);
      expect(finalOwnerBalance).to.equal(initialOwnerBalance.sub(amount.mul(2)));

      const addr1Balance = await token.balanceOf(addr1.address);
      expect(addr1Balance).to.equal(amount);

      const addr2Balance = await token.balanceOf(addr2.address);
      expect(addr2Balance).to.equal(amount);
    });

    it("Should emit Transfer event", async function () {
      const amount = ethers.utils.parseEther("100");
      
      await expect(token.transfer(addr1.address, amount))
        .to.emit(token, "Transfer")
        .withArgs(owner.address, addr1.address, amount);
    });

    it("Should reject transfers to zero address", async function () {
      const amount = ethers.utils.parseEther("100");
      
      await expect(
        token.transfer(ethers.constants.AddressZero, amount)
      ).to.be.revertedWith("Invalid address");
    });
  });

  describe("Allowances", function () {
    it("Should approve tokens for delegated transfer", async function () {
      const amount = ethers.utils.parseEther("100");
      
      await token.approve(addr1.address, amount);
      
      expect(await token.allowance(owner.address, addr1.address)).to.equal(amount);
    });

    it("Should emit Approval event", async function () {
      const amount = ethers.utils.parseEther("100");
      
      await expect(token.approve(addr1.address, amount))
        .to.emit(token, "Approval")
        .withArgs(owner.address, addr1.address, amount);
    });

    it("Should allow transferFrom with sufficient allowance", async function () {
      const amount = ethers.utils.parseEther("100");
      
      await token.approve(addr1.address, amount);
      await token.connect(addr1).transferFrom(owner.address, addr2.address, amount);
      
      expect(await token.balanceOf(addr2.address)).to.equal(amount);
    });

    it("Should fail transferFrom without sufficient allowance", async function () {
      const amount = ethers.utils.parseEther("100");
      
      await expect(
        token.connect(addr1).transferFrom(owner.address, addr2.address, amount)
      ).to.be.revertedWith("Allowance exceeded");
    });

    it("Should decrease allowance after transferFrom", async function () {
      const amount = ethers.utils.parseEther("100");
      const transferAmount = ethers.utils.parseEther("50");
      
      await token.approve(addr1.address, amount);
      await token.connect(addr1).transferFrom(owner.address, addr2.address, transferAmount);
      
      const remainingAllowance = await token.allowance(owner.address, addr1.address);
      expect(remainingAllowance).to.equal(amount.sub(transferAmount));
    });
  });
});
```

## Advanced Testing Patterns

### Testing Time-Dependent Contracts

```solidity
// contracts/TimeLock.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

contract TimeLock {
    mapping(address => uint256) public lockTime;
    mapping(address => uint256) public balances;
    
    uint256 public constant LOCK_DURATION = 7 days;
    
    function deposit() external payable {
        require(msg.value > 0, "Must deposit something");
        balances[msg.sender] += msg.value;
        lockTime[msg.sender] = block.timestamp + LOCK_DURATION;
    }
    
    function withdraw() external {
        require(balances[msg.sender] > 0, "No balance");
        require(block.timestamp >= lockTime[msg.sender], "Still locked");
        
        uint256 amount = balances[msg.sender];
        balances[msg.sender] = 0;
        
        (bool success, ) = msg.sender.call{value: amount}("");
        require(success, "Transfer failed");
    }
}
```

```javascript
// test/TimeLock.test.js
const { expect } = require("chai");
const { ethers } = require("hardhat");
const { time } = require("@nomicfoundation/hardhat-network-helpers");

describe("TimeLock", function () {
  let timeLock;
  let owner;
  let addr1;

  beforeEach(async function () {
    const TimeLock = await ethers.getContractFactory("TimeLock");
    [owner, addr1] = await ethers.getSigners();
    timeLock = await TimeLock.deploy();
  });

  describe("Time-based logic", function () {
    it("Should not allow withdrawal before lock expires", async function () {
      const depositAmount = ethers.utils.parseEther("1");
      
      await timeLock.deposit({ value: depositAmount });
      
      await expect(timeLock.withdraw()).to.be.revertedWith("Still locked");
    });

    it("Should allow withdrawal after lock expires", async function () {
      const depositAmount = ethers.utils.parseEther("1");
      
      await timeLock.deposit({ value: depositAmount });
      
      // Fast forward time by 7 days
      await time.increase(7 * 24 * 60 * 60);
      
      await expect(timeLock.withdraw()).to.not.be.reverted;
      
      expect(await timeLock.balances(owner.address)).to.equal(0);
    });

    it("Should handle multiple deposits correctly", async function () {
      const depositAmount = ethers.utils.parseEther("1");
      
      await timeLock.deposit({ value: depositAmount });
      
      // Fast forward 3 days
      await time.increase(3 * 24 * 60 * 60);
      
      // Second deposit resets lock time
      await timeLock.deposit({ value: depositAmount });
      
      // Fast forward 5 more days (8 days total from first deposit)
      await time.increase(5 * 24 * 60 * 60);
      
      // Should still be locked (only 5 days from second deposit)
      await expect(timeLock.withdraw()).to.be.revertedWith("Still locked");
    });

    it("Should allow withdrawal at exact expiry time", async function () {
      const depositAmount = ethers.utils.parseEther("1");
      
      await timeLock.deposit({ value: depositAmount });
      
      const lockTime = await timeLock.lockTime(owner.address);
      
      // Set time to exact expiry
      await time.increaseTo(lockTime);
      
      await expect(timeLock.withdraw()).to.not.be.reverted;
    });
  });
});
```

### Testing Access Control

```solidity
// contracts/Owned.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

contract Owned {
    address public owner;
    
    event OwnershipTransferred(address indexed previousOwner, address indexed newOwner);
    
    constructor() {
        owner = msg.sender;
    }
    
    modifier onlyOwner() {
        require(msg.sender == owner, "Not owner");
        _;
    }
    
    function transferOwnership(address newOwner) external onlyOwner {
        require(newOwner != address(0), "Invalid address");
        emit OwnershipTransferred(owner, newOwner);
        owner = newOwner;
    }
}
```

```javascript
// test/Owned.test.js
describe("Owned", function () {
  let owned;
  let owner;
  let addr1;
  let addr2;

  beforeEach(async function () {
    const Owned = await ethers.getContractFactory("Owned");
    [owner, addr1, addr2] = await ethers.getSigners();
    owned = await Owned.deploy();
  });

  describe("Access Control", function () {
    it("Should set deployer as owner", async function () {
      expect(await owned.owner()).to.equal(owner.address);
    });

    it("Should allow owner to transfer ownership", async function () {
      await owned.transferOwnership(addr1.address);
      expect(await owned.owner()).to.equal(addr1.address);
    });

    it("Should emit OwnershipTransferred event", async function () {
      await expect(owned.transferOwnership(addr1.address))
        .to.emit(owned, "OwnershipTransferred")
        .withArgs(owner.address, addr1.address);
    });

    it("Should prevent non-owners from transferring ownership", async function () {
      await expect(
        owned.connect(addr1).transferOwnership(addr2.address)
      ).to.be.revertedWith("Not owner");
    });

    it("Should prevent transfer to zero address", async function () {
      await expect(
        owned.transferOwnership(ethers.constants.AddressZero)
      ).to.be.revertedWith("Invalid address");
    });

    it("Should allow new owner to transfer ownership", async function () {
      await owned.transferOwnership(addr1.address);
      await owned.connect(addr1).transferOwnership(addr2.address);
      
      expect(await owned.owner()).to.equal(addr2.address);
    });
  });
});
```

## Testing Contract Interactions

```solidity
// contracts/DEX.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

interface IERC20 {
    function transfer(address to, uint256 amount) external returns (bool);
    function transferFrom(address from, address to, uint256 amount) external returns (bool);
    function balanceOf(address account) external view returns (uint256);
}

contract DEX {
    mapping(address => mapping(address => uint256)) public liquidity;
    
    function addLiquidity(address token, uint256 amount) external {
        IERC20(token).transferFrom(msg.sender, address(this), amount);
        liquidity[token][msg.sender] += amount;
    }
    
    function removeLiquidity(address token, uint256 amount) external {
        require(liquidity[token][msg.sender] >= amount, "Insufficient liquidity");
        liquidity[token][msg.sender] -= amount;
        IERC20(token).transfer(msg.sender, amount);
    }
    
    function swap(address tokenIn, address tokenOut, uint256 amountIn) external {
        require(amountIn > 0, "Invalid amount");
        
        IERC20(tokenIn).transferFrom(msg.sender, address(this), amountIn);
        
        // Simplified swap (1:1 ratio for demo)
        uint256 amountOut = amountIn;
        
        IERC20(tokenOut).transfer(msg.sender, amountOut);
    }
}
```

```javascript
// test/DEX.test.js
describe("DEX", function () {
  let dex;
  let tokenA;
  let tokenB;
  let owner;
  let user;

  beforeEach(async function () {
    const Token = await ethers.getContractFactory("Token");
    const DEX = await ethers.getContractFactory("DEX");
    
    [owner, user] = await ethers.getSigners();
    
    tokenA = await Token.deploy("TokenA", "TKA", 1000000);
    tokenB = await Token.deploy("TokenB", "TKB", 1000000);
    dex = await DEX.deploy();
    
    // Transfer some tokens to user
    await tokenA.transfer(user.address, ethers.utils.parseEther("10000"));
    await tokenB.transfer(user.address, ethers.utils.parseEther("10000"));
  });

  describe("Liquidity Management", function () {
    it("Should add liquidity", async function () {
      const amount = ethers.utils.parseEther("1000");
      
      await tokenA.connect(user).approve(dex.address, amount);
      await dex.connect(user).addLiquidity(tokenA.address, amount);
      
      expect(await dex.liquidity(tokenA.address, user.address)).to.equal(amount);
      expect(await tokenA.balanceOf(dex.address)).to.equal(amount);
    });

    it("Should remove liquidity", async function () {
      const amount = ethers.utils.parseEther("1000");
      
      await tokenA.connect(user).approve(dex.address, amount);
      await dex.connect(user).addLiquidity(tokenA.address, amount);
      
      const initialBalance = await tokenA.balanceOf(user.address);
      
      await dex.connect(user).removeLiquidity(tokenA.address, amount);
      
      expect(await dex.liquidity(tokenA.address, user.address)).to.equal(0);
      expect(await tokenA.balanceOf(user.address)).to.equal(initialBalance.add(amount));
    });

    it("Should fail to remove more liquidity than available", async function () {
      const amount = ethers.utils.parseEther("1000");
      
      await expect(
        dex.connect(user).removeLiquidity(tokenA.address, amount)
      ).to.be.revertedWith("Insufficient liquidity");
    });
  });

  describe("Token Swaps", function () {
    beforeEach(async function () {
      // Add liquidity for both tokens
      const liquidityAmount = ethers.utils.parseEther("5000");
      
      await tokenA.approve(dex.address, liquidityAmount);
      await tokenB.approve(dex.address, liquidityAmount);
      
      await dex.addLiquidity(tokenA.address, liquidityAmount);
      await dex.addLiquidity(tokenB.address, liquidityAmount);
    });

    it("Should swap tokens", async function () {
      const swapAmount = ethers.utils.parseEther("100");
      
      const initialBalanceA = await tokenA.balanceOf(user.address);
      const initialBalanceB = await tokenB.balanceOf(user.address);
      
      await tokenA.connect(user).approve(dex.address, swapAmount);
      await dex.connect(user).swap(tokenA.address, tokenB.address, swapAmount);
      
      expect(await tokenA.balanceOf(user.address)).to.equal(initialBalanceA.sub(swapAmount));
      expect(await tokenB.balanceOf(user.address)).to.equal(initialBalanceB.add(swapAmount));
    });

    it("Should fail swap with zero amount", async function () {
      await expect(
        dex.connect(user).swap(tokenA.address, tokenB.address, 0)
      ).to.be.revertedWith("Invalid amount");
    });
  });
});
```

## Gas Optimization Testing

```javascript
// test/GasOptimization.test.js
describe("Gas Optimization", function () {
  let token;
  let owner;
  let users;

  beforeEach(async function () {
    const Token = await ethers.getContractFactory("Token");
    [owner, ...users] = await ethers.getSigners();
    token = await Token.deploy("GasToken", "GAS", 1000000);
  });

  it("Should measure gas for single transfer", async function () {
    const amount = ethers.utils.parseEther("100");
    
    const tx = await token.transfer(users[0].address, amount);
    const receipt = await tx.wait();
    
    console.log("Gas used for single transfer:", receipt.gasUsed.toString());
    
    expect(receipt.gasUsed).to.be.lt(100000); // Assert gas limit
  });

  it("Should compare gas for multiple transfers", async function () {
    const amount = ethers.utils.parseEther("100");
    
    // Sequential transfers
    const tx1 = await token.transfer(users[0].address, amount);
    const receipt1 = await tx1.wait();
    
    const tx2 = await token.transfer(users[1].address, amount);
    const receipt2 = await tx2.wait();
    
    const totalGas = receipt1.gasUsed.add(receipt2.gasUsed);
    console.log("Total gas for 2 transfers:", totalGas.toString());
    
    // Could compare with batch transfer implementation
  });

  it("Should measure deployment gas", async function () {
    const Token = await ethers.getContractFactory("Token");
    const deployTx = await Token.deploy("TestToken", "TST", 1000000);
    const deployReceipt = await deployTx.deployTransaction.wait();
    
    console.log("Deployment gas:", deployReceipt.gasUsed.toString());
  });
});
```

## Snapshot and Revert Testing

```javascript
const { loadFixture } = require("@nomicfoundation/hardhat-network-helpers");

describe("Using Fixtures", function () {
  async function deployTokenFixture() {
    const Token = await ethers.getContractFactory("Token");
    const [owner, addr1, addr2] = await ethers.getSigners();
    
    const token = await Token.deploy("MyToken", "MTK", 1000000);
    await token.deployed();
    
    // Pre-populate some state
    await token.transfer(addr1.address, ethers.utils.parseEther("1000"));
    
    return { token, owner, addr1, addr2 };
  }

  it("Should use fixture for fast testing", async function () {
    const { token, addr1 } = await loadFixture(deployTokenFixture);
    
    // State is already set up
    expect(await token.balanceOf(addr1.address)).to.equal(
      ethers.utils.parseEther("1000")
    );
  });

  it("Should start with fresh state", async function () {
    const { token, addr1, addr2 } = await loadFixture(deployTokenFixture);
    
    // Each test gets clean state
    const amount = ethers.utils.parseEther("500");
    await token.connect(addr1).transfer(addr2.address, amount);
    
    expect(await token.balanceOf(addr2.address)).to.equal(amount);
  });
});
```

## Coverage Reporting

```bash
# Install coverage plugin
npm install --save-dev solidity-coverage

# Add to hardhat.config.js
require("solidity-coverage");

# Run coverage
npx hardhat coverage
```

Coverage output example:
```
File                |  % Stmts | % Branch |  % Funcs |  % Lines |
--------------------|----------|----------|----------|----------|
 contracts/         |      100 |      100 |      100 |      100 |
  Token.sol         |      100 |      100 |      100 |      100 |
--------------------|----------|----------|----------|----------|
All files           |      100 |      100 |      100 |      100 |
```

## Continuous Integration

```yaml
# .github/workflows/test.yml
name: Smart Contract Tests

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Setup Node.js
      uses: actions/setup-node@v3
      with:
        node-version: '18'
        cache: 'npm'
    
    - name: Install dependencies
      run: npm ci
    
    - name: Run tests
      run: npx hardhat test
    
    - name: Run coverage
      run: npx hardhat coverage
    
    - name: Upload coverage to Codecov
      uses: codecov/codecov-action@v3
      with:
        files: ./coverage/coverage-final.json
```

## Best Practices

### 1. Test Organization

```javascript
describe("Contract Name", function () {
  describe("Function Group 1", function () {
    it("Should do X", async function () {
      // Test X
    });
    
    it("Should fail when Y", async function () {
      // Test failure case
    });
  });
  
  describe("Function Group 2", function () {
    // More tests
  });
});
```

### 2. Use Meaningful Test Names

```javascript
// Good
it("Should transfer tokens from sender to recipient", async function () {});

// Bad
it("Test 1", async function () {});
```

### 3. Test Edge Cases

```javascript
describe("Edge Cases", function () {
  it("Should handle zero amount", async function () {});
  it("Should handle maximum uint256", async function () {});
  it("Should handle zero address", async function () {});
  it("Should handle empty arrays", async function () {});
});
```

### 4. Clean Test Setup

```javascript
beforeEach(async function () {
  // Fresh state for each test
  const Contract = await ethers.getContractFactory("MyContract");
  contract = await Contract.deploy();
});

afterEach(async function () {
  // Cleanup if needed
});
```

## Running Tests

```bash
# Run all tests
npx hardhat test

# Run specific test file
npx hardhat test test/Token.test.js

# Run with gas reporting
REPORT_GAS=true npx hardhat test

# Run with verbose output
npx hardhat test --verbose

# Run specific test by name
npx hardhat test --grep "Should transfer tokens"

# Parallel testing
npx hardhat test --parallel

# Show stack traces
npx hardhat test --show-stack-traces
```

## Debugging Tests

```javascript
// Add console.log in tests
it("Debug test", async function () {
  console.log("Owner address:", owner.address);
  console.log("Balance:", (await token.balanceOf(owner.address)).toString());
});

// Use hardhat console
const { Console } = require("console");

// Inspect transactions
const tx = await token.transfer(addr1.address, amount);
const receipt = await tx.wait();
console.log("Transaction:", receipt);
```

## Conclusion

Comprehensive testing is non-negotiable in smart contract development. With Hardhat and Ethers.js, you have powerful tools to build confidence in your code before it hits mainnet. Remember:

- Test all functions and edge cases
- Measure and optimize gas usage
- Use fixtures for complex setups
- Implement CI/CD for automated testing
- Aim for 100% code coverage
- Test interactions between contracts
- Verify time-dependent logic
- Check access control thoroughly

## Resources

- [Hardhat Documentation](https://hardhat.org/docs)
- [Ethers.js Documentation](https://docs.ethers.org/)
- [Chai Matchers](https://ethereum-waffle.readthedocs.io/en/latest/matchers.html)
- [Hardhat Network Helpers](https://hardhat.org/hardhat-network-helpers)

Happy testing! 🧪
