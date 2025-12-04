---
title: "Gas Optimization Techniques for Solidity Smart Contracts"
description: "Advanced gas optimization techniques for Solidity smart contracts. Storage layout, assembly optimization, EVM opcodes, and best practices for cost-effective blockchain development."
date: "2024-09-25 16:00:00 +0300"
categories: [Smart Contracts, Solidity]
tags: [solidity, ethereum, gas-optimization, evm, smart-contracts, blockchain, opcodes, storage]
image:
  path: /assets/img/posts/evm-gas-opcodes-how-it-works.png
  alt: "Gas Optimization Techniques for Solidity"
---

## Introduction

Gas optimization is one of the most critical skills for Solidity developers. Every operation in an Ethereum smart contract costs gas, and inefficient code can make your contracts prohibitively expensive to use. In a world where users pay for every interaction with your contract, optimizing gas consumption isn't just about performance—it's about usability and adoption.

This comprehensive guide explores advanced gas optimization techniques for Solidity smart contracts. We'll dive deep into the EVM's inner workings, understand how different operations consume gas, and learn practical strategies to minimize costs. Whether you're building DeFi protocols, NFT marketplaces, or DAOs, these techniques will help you create efficient, cost-effective smart contracts.

From storage layout optimization to advanced assembly techniques, we'll cover everything you need to know to write production-grade, gas-efficient Solidity code. By the end of this guide, you'll have a complete toolkit for analyzing and optimizing your smart contracts' gas consumption.

![EVM Architecture](/assets/img/posts/evm-architecture-execution-diagram.png){: w="800" h="500" .shadow }
_EVM architecture and execution flow_

## Understanding Gas in Ethereum

### What is Gas?

Gas is the unit of measurement for computational work on the Ethereum blockchain. Every operation executed by the EVM (Ethereum Virtual Machine) has an associated gas cost:

- **Storage operations**: Most expensive (5,000-20,000 gas)
- **Computation**: Moderate cost (3-8 gas for arithmetic)
- **Memory operations**: Cheaper (3 gas plus memory expansion)
- **Reading state**: 100-2,100 gas depending on access

Gas prices fluctuate based on network demand, making optimization even more crucial during high-traffic periods.

### Gas Cost Components

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract GasCostExample {
    // Storage variable - EXPENSIVE
    uint256 public storageVar; // 20,000 gas for SSTORE (from zero)
    
    function demonstrateGasCosts() public {
        // Memory variable - CHEAP
        uint256 memoryVar = 10; // ~3 gas
        
        // Reading storage - MODERATE
        uint256 value = storageVar; // 2,100 gas (cold access)
        
        // Writing storage - EXPENSIVE
        storageVar = value + memoryVar; // 5,000 gas (warm) or 20,000 (cold)
        
        // Computation - CHEAP
        uint256 result = memoryVar * 2; // ~5 gas
    }
}
```

### Key Gas Metrics

- **Transaction Base Cost**: 21,000 gas (minimum)
- **Contract Creation**: 32,000 gas base + deployment costs
- **SLOAD (cold)**: 2,100 gas
- **SLOAD (warm)**: 100 gas
- **SSTORE (from zero)**: 20,000 gas
- **SSTORE (to zero)**: 15,000 gas refund
- **SSTORE (change)**: 5,000 gas (warm)

![EVM Opcodes Table](/assets/img/posts/evm-opcodes-gas-cost-table.jpg)
*Figure 2: EVM opcodes and their gas costs*

## Storage Optimization

Storage is by far the most expensive resource in smart contracts. Let's explore advanced optimization techniques.

### Packing Variables

```solidity
// BAD: Uses 3 storage slots (3 * 20,000 = 60,000 gas)
contract Unoptimized {
    uint256 a; // slot 0
    uint256 b; // slot 1
    uint256 c; // slot 2
}

// GOOD: Uses 1 storage slot (20,000 gas)
contract Optimized {
    uint128 a; // slot 0 (first 16 bytes)
    uint128 b; // slot 0 (last 16 bytes)
    uint256 c; // slot 1
}

// BEST: Strategic packing based on usage
contract HighlyOptimized {
    // Frequently accessed together - pack in same slot
    uint64 timestamp;   // slot 0
    uint64 userId;      // slot 0
    uint64 amount;      // slot 0
    uint64 flags;       // slot 0
    
    // Large value needs its own slot
    uint256 largeValue; // slot 1
    
    // Booleans are cheap to pack
    bool isActive;      // slot 2
    bool isVerified;    // slot 2
    bool isPremium;     // slot 2
    address owner;      // slot 2 (20 bytes)
}
```

### Strategic Variable Ordering

```solidity
// BAD: Wastes storage slots
contract BadOrdering {
    uint8 small1;    // slot 0 (1 byte used, 31 wasted)
    uint256 big1;    // slot 1 (32 bytes)
    uint8 small2;    // slot 2 (1 byte used, 31 wasted)
    uint256 big2;    // slot 3 (32 bytes)
    // Total: 4 slots = 80,000 gas
}

// GOOD: Optimal ordering
contract GoodOrdering {
    uint256 big1;    // slot 0
    uint256 big2;    // slot 1
    uint8 small1;    // slot 2 (first byte)
    uint8 small2;    // slot 2 (second byte)
    // Total: 3 slots = 60,000 gas (25% savings)
}

// BEST: Group by access patterns
contract BestOrdering {
    // Hot path: frequently accessed together
    address owner;        // slot 0
    uint96 balance;       // slot 0
    
    // Config: rarely changed together
    uint128 minAmount;    // slot 1
    uint128 maxAmount;    // slot 1
    
    // Timestamps: updated together
    uint128 createdAt;    // slot 2
    uint128 updatedAt;    // slot 2
}
```

### Constants and Immutables

```solidity
contract ConstantOptimization {
    // BAD: Storage variable (2,100 gas per read)
    uint256 public fee = 100;
    
    // GOOD: Constant (compiled into bytecode, ~3 gas)
    uint256 public constant FEE = 100;
    
    // BETTER: Immutable (set in constructor, ~100 gas)
    uint256 public immutable deployTime;
    address public immutable deployer;
    
    constructor() {
        deployTime = block.timestamp;
        deployer = msg.sender;
    }
    
    // Constants are free to access in expressions
    function calculateFee(uint256 amount) public pure returns (uint256) {
        return amount * FEE / 10000; // FEE costs 0 gas
    }
}
```

### Mappings vs Arrays

```solidity
contract CollectionOptimization {
    // Mappings: O(1) access, no iteration
    mapping(address => uint256) public balances;
    
    // Arrays: O(n) iteration, expensive to grow
    address[] public users;
    
    // BEST: Combine both for different use cases
    mapping(address => uint256) public userBalances;
    mapping(address => bool) public isUser;
    address[] public userList; // Only when iteration needed
    
    function addUser(address user, uint256 balance) public {
        require(!isUser[user], "Already exists");
        
        userBalances[user] = balance;
        isUser[user] = true;
        
        // Only add to array if iteration is needed
        if (needsIteration()) {
            userList.push(user);
        }
    }
    
    // Avoid iterating large arrays in transactions
    function getTotalBalance() public view returns (uint256) {
        uint256 total;
        // BAD: Unbounded loop can hit gas limit
        for (uint256 i = 0; i < userList.length; i++) {
            total += userBalances[userList[i]];
        }
        return total;
    }
    
    // BETTER: Use off-chain indexing or events
    function needsIteration() private pure returns (bool) {
        return false; // Prefer off-chain processing
    }
}
```

## Memory vs Storage vs Calldata

Understanding data location is crucial for gas optimization.

### Data Location Comparison

```solidity
contract DataLocationOptimization {
    struct User {
        string name;
        uint256 age;
        address wallet;
    }
    
    User[] public users;
    
    // BAD: Copies storage to memory (expensive)
    function updateUserBad(uint256 index, string memory newName) public {
        User memory user = users[index]; // COPY to memory
        user.name = newName;
        users[index] = user; // WRITE back to storage
        // Cost: read storage → copy to memory → write to storage
    }
    
    // GOOD: Direct storage pointer (cheap)
    function updateUserGood(uint256 index, string memory newName) public {
        User storage user = users[index]; // POINTER (no copy)
        user.name = newName; // Direct write to storage
        // Cost: write to storage only
    }
    
    // BEST: Use calldata for read-only parameters
    function processUser(string calldata name) public pure returns (bytes32) {
        // calldata: cheaper than memory, can't be modified
        return keccak256(bytes(name));
    }
    
    // Calldata vs Memory for arrays
    function sumArray(uint256[] calldata numbers) public pure returns (uint256) {
        uint256 sum;
        for (uint256 i = 0; i < numbers.length; i++) {
            sum += numbers[i]; // Read from calldata (cheap)
        }
        return sum;
    }
    
    function processArray(uint256[] memory numbers) public pure returns (uint256[] memory) {
        // Use memory when you need to modify the array
        for (uint256 i = 0; i < numbers.length; i++) {
            numbers[i] *= 2; // Modify in memory
        }
        return numbers;
    }
}
```

### String and Bytes Optimization

```solidity
contract StringOptimization {
    // BAD: string (expensive, dynamic allocation)
    string public name;
    
    // GOOD: bytes32 for short strings (fixed size)
    bytes32 public shortName;
    
    // BETTER: Custom encoding for fixed formats
    bytes32 public encodedData;
    
    function setName(string calldata _name) public {
        require(bytes(_name).length <= 32, "Too long");
        name = _name; // Expensive storage operation
    }
    
    function setShortName(string calldata _name) public {
        require(bytes(_name).length <= 32, "Too long");
        shortName = bytes32(bytes(_name)); // Single slot write
    }
    
    // Encode multiple values into bytes32
    function encodeData(uint128 id, uint64 timestamp, uint64 amount) public {
        encodedData = bytes32(
            (uint256(id) << 128) | 
            (uint256(timestamp) << 64) | 
            uint256(amount)
        );
    }
    
    function decodeData() public view returns (uint128 id, uint64 timestamp, uint64 amount) {
        uint256 data = uint256(encodedData);
        id = uint128(data >> 128);
        timestamp = uint64(data >> 64);
        amount = uint64(data);
    }
    
    // Use events instead of storing strings
    event DataStored(string indexed key, string value);
    
    function storeInEvent(string calldata key, string calldata value) public {
        emit DataStored(key, value); // Much cheaper than storage
    }
}
```

## Function Optimization

![Opcode Execution](/assets/img/posts/evm-opcode-execution-flow.png)
*Figure 3: EVM opcode execution flow*

### External vs Public Functions

```solidity
contract FunctionOptimization {
    uint256 public value;
    
    // PUBLIC: Can be called internally and externally
    // Costs more gas due to memory copying
    function publicFunction(uint256[] memory data) public {
        // data is copied to memory
        value = data[0];
    }
    
    // EXTERNAL: Only callable from outside
    // Cheaper - data stays in calldata
    function externalFunction(uint256[] calldata data) external {
        // data stays in calldata (no copy)
        value = data[0];
    }
    
    // If you need both, use this pattern
    function internalLogic(uint256[] calldata data) private {
        value = data[0];
    }
    
    function publicWrapper(uint256[] memory data) public {
        // Convert memory to calldata (not possible directly)
        // Better to just use internal logic differently
        value = data[0];
    }
    
    function externalWrapper(uint256[] calldata data) external {
        internalLogic(data); // Pass calldata directly
    }
}
```

### Function Modifiers

```solidity
contract ModifierOptimization {
    address public owner;
    
    // BAD: Modifier with code
    modifier onlyOwnerBad() {
        require(msg.sender == owner, "Not owner");
        _;
    }
    
    // GOOD: Internal function
    function _onlyOwner() private view {
        require(msg.sender == owner, "Not owner");
    }
    
    // Usage comparison
    function withModifier() public onlyOwnerBad {
        // Modifier code is inlined, increasing bytecode size
    }
    
    function withFunction() public {
        _onlyOwner(); // Function call is more gas efficient
        // Function logic
    }
    
    // BEST: Short-circuit with custom errors
    error NotOwner();
    
    function _checkOwner() private view {
        if (msg.sender != owner) revert NotOwner();
    }
    
    function optimized() public {
        _checkOwner();
        // Function logic
    }
}
```

### Custom Errors vs Require Strings

```solidity
contract ErrorOptimization {
    // BAD: String error messages (expensive)
    function badError(uint256 amount) public pure {
        require(amount > 100, "Amount must be greater than 100");
        // String stored in contract bytecode: ~50 bytes
    }
    
    // GOOD: Custom errors (cheap)
    error InsufficientAmount(uint256 provided, uint256 required);
    
    function goodError(uint256 amount) public pure {
        if (amount <= 100) {
            revert InsufficientAmount(amount, 100);
        }
        // Custom error: ~22 bytes, much cheaper
    }
    
    // Comparison of different error methods
    error Unauthorized();
    error InvalidAmount();
    error TransferFailed();
    
    function compareErrors(uint256 amount) public view {
        // Method 1: require with string (EXPENSIVE)
        // require(msg.sender == address(this), "Unauthorized access denied");
        
        // Method 2: require without string (CHEAPER)
        // require(msg.sender == address(this));
        
        // Method 3: custom error (CHEAPEST)
        if (msg.sender != address(this)) revert Unauthorized();
        
        // Method 4: assert for invariants (CHEAPEST, no error data)
        assert(amount != 0); // Only for internal errors
    }
}
```

### Short-Circuiting

```solidity
contract ShortCircuitOptimization {
    mapping(address => bool) public isWhitelisted;
    mapping(address => uint256) public userTier;
    
    // BAD: Expensive check first
    function badAccess() public view returns (bool) {
        return userTier[msg.sender] >= 3 && isWhitelisted[msg.sender];
        // Always reads userTier (2,100 gas) even if not whitelisted
    }
    
    // GOOD: Cheap check first
    function goodAccess() public view returns (bool) {
        return isWhitelisted[msg.sender] && userTier[msg.sender] >= 3;
        // Exits early if not whitelisted, saves gas
    }
    
    // BETTER: Most likely to fail first
    function optimizedAccess() public view returns (bool) {
        // Assume most users aren't whitelisted
        if (!isWhitelisted[msg.sender]) return false;
        
        // Only check tier for whitelisted users
        if (userTier[msg.sender] < 3) return false;
        
        return true;
    }
    
    // Order operations by probability of failure
    function validateTransaction(
        address sender,
        uint256 amount,
        bytes calldata signature
    ) public view returns (bool) {
        // 1. Cheapest check (amount > 0)
        if (amount == 0) return false;
        
        // 2. Cheap storage check (whitelisting)
        if (!isWhitelisted[sender]) return false;
        
        // 3. Moderate storage check (tier)
        if (userTier[sender] < 2) return false;
        
        // 4. Most expensive check last (signature)
        return verifySignature(sender, amount, signature);
    }
    
    function verifySignature(
        address sender,
        uint256 amount,
        bytes calldata signature
    ) private pure returns (bool) {
        // Expensive cryptographic operation
        return signature.length > 0; // Simplified
    }
}
```

## Loop Optimization

Loops can quickly consume gas and even hit the gas limit.

### Loop Best Practices

```solidity
contract LoopOptimization {
    uint256[] public numbers;
    mapping(address => uint256) public balances;
    address[] public users;
    
    // BAD: Multiple storage reads in loop
    function badLoop() public view returns (uint256) {
        uint256 sum;
        for (uint256 i = 0; i < numbers.length; i++) {
            sum += numbers[i]; // SLOAD every iteration
        }
        return sum;
    }
    
    // GOOD: Cache array length
    function goodLoop() public view returns (uint256) {
        uint256 sum;
        uint256 length = numbers.length; // Cache length
        for (uint256 i = 0; i < length; i++) {
            sum += numbers[i];
        }
        return sum;
    }
    
    // BETTER: Use unchecked for counter
    function betterLoop() public view returns (uint256) {
        uint256 sum;
        uint256 length = numbers.length;
        for (uint256 i = 0; i < length;) {
            sum += numbers[i];
            unchecked { ++i; } // No overflow check, saves ~30-40 gas per iteration
        }
        return sum;
    }
    
    // BEST: Process in batches
    function batchProcess(uint256 start, uint256 end) public view returns (uint256) {
        require(end <= numbers.length, "Out of bounds");
        require(end - start <= 100, "Batch too large"); // Limit batch size
        
        uint256 sum;
        for (uint256 i = start; i < end;) {
            sum += numbers[i];
            unchecked { ++i; }
        }
        return sum;
    }
    
    // Avoid unbounded loops in transactions
    function distributeRewards() public {
        uint256 length = users.length;
        require(length <= 50, "Too many users, use batch distribution");
        
        uint256 rewardPerUser = 100;
        for (uint256 i = 0; i < length;) {
            balances[users[i]] += rewardPerUser;
            unchecked { ++i; }
        }
    }
    
    // Use pagination pattern
    uint256 public lastProcessedIndex;
    
    function processInChunks(uint256 chunkSize) public {
        uint256 length = users.length;
        uint256 endIndex = lastProcessedIndex + chunkSize;
        
        if (endIndex > length) {
            endIndex = length;
        }
        
        for (uint256 i = lastProcessedIndex; i < endIndex;) {
            // Process user
            balances[users[i]] += 50;
            unchecked { ++i; }
        }
        
        lastProcessedIndex = endIndex;
        
        // Reset when done
        if (lastProcessedIndex >= length) {
            lastProcessedIndex = 0;
        }
    }
}
```

### Loop Unrolling

```solidity
contract LoopUnrolling {
    // BAD: Small fixed loop
    function badSmallLoop(uint256[4] memory values) public pure returns (uint256) {
        uint256 sum;
        for (uint256 i = 0; i < 4; i++) {
            sum += values[i];
        }
        return sum;
    }
    
    // GOOD: Unrolled loop (saves ~200 gas)
    function goodUnrolled(uint256[4] memory values) public pure returns (uint256) {
        return values[0] + values[1] + values[2] + values[3];
    }
    
    // Unrolling with processing
    function processArray(uint256[8] memory data) public pure returns (uint256) {
        uint256 result;
        
        // Unroll by 4
        result += data[0] * 2;
        result += data[1] * 2;
        result += data[2] * 2;
        result += data[3] * 2;
        result += data[4] * 2;
        result += data[5] * 2;
        result += data[6] * 2;
        result += data[7] * 2;
        
        return result;
    }
}
```

## Advanced Optimization Techniques

### Bit Manipulation

```solidity
contract BitManipulation {
    // Pack multiple booleans into one uint256
    uint256 private flags;
    
    // Set bit at position
    function setFlag(uint8 position) public {
        flags |= (1 << position);
    }
    
    // Clear bit at position
    function clearFlag(uint8 position) public {
        flags &= ~(1 << position);
    }
    
    // Check bit at position
    function hasFlag(uint8 position) public view returns (bool) {
        return (flags & (1 << position)) != 0;
    }
    
    // Multiple flags in one operation
    function setMultipleFlags(uint8[] calldata positions) public {
        uint256 newFlags;
        for (uint256 i = 0; i < positions.length;) {
            newFlags |= (1 << positions[i]);
            unchecked { ++i; }
        }
        flags |= newFlags; // Single SSTORE
    }
    
    // Bit manipulation for efficient math
    function multiplyBy8(uint256 value) public pure returns (uint256) {
        return value << 3; // Cheaper than value * 8
    }
    
    function divideBy4(uint256 value) public pure returns (uint256) {
        return value >> 2; // Cheaper than value / 4
    }
    
    function isEven(uint256 value) public pure returns (bool) {
        return (value & 1) == 0; // Cheaper than value % 2 == 0
    }
    
    // Pack multiple values
    struct PackedData {
        uint256 packed; // 256 bits total
    }
    
    function packData(
        uint64 timestamp,   // 64 bits
        uint32 userId,      // 32 bits
        uint16 category,    // 16 bits
        uint8 status,       // 8 bits
        uint8 flags         // 8 bits
    ) public pure returns (uint256) {
        return uint256(timestamp) << 192 |
               uint256(userId) << 160 |
               uint256(category) << 144 |
               uint256(status) << 136 |
               uint256(flags) << 128;
    }
    
    function unpackData(uint256 packed) public pure returns (
        uint64 timestamp,
        uint32 userId,
        uint16 category,
        uint8 status,
        uint8 flags
    ) {
        timestamp = uint64(packed >> 192);
        userId = uint32(packed >> 160);
        category = uint16(packed >> 144);
        status = uint8(packed >> 136);
        flags = uint8(packed >> 128);
    }
}
```

### Assembly Optimization

```solidity
contract AssemblyOptimization {
    // Efficient storage access
    function getSlot(uint256 slot) public view returns (uint256 value) {
        assembly {
            value := sload(slot)
        }
    }
    
    function setSlot(uint256 slot, uint256 value) public {
        assembly {
            sstore(slot, value)
        }
    }
    
    // Efficient memory operations
    function copyBytes(bytes calldata data) public pure returns (bytes memory) {
        bytes memory result;
        assembly {
            // Allocate memory
            result := mload(0x40)
            let length := data.length
            
            // Store length
            mstore(result, length)
            
            // Copy data
            calldatacopy(add(result, 0x20), data.offset, length)
            
            // Update free memory pointer
            mstore(0x40, add(add(result, 0x20), length))
        }
        return result;
    }
    
    // Efficient address validation
    function isContract(address account) public view returns (bool) {
        uint256 size;
        assembly {
            size := extcodesize(account)
        }
        return size > 0;
    }
    
    // Efficient return data
    function efficientReturn(uint256 value) public pure {
        assembly {
            mstore(0x00, value)
            return(0x00, 0x20)
        }
    }
    
    // Custom revert
    error CustomError(uint256 value);
    
    function efficientRevert(uint256 value) public pure {
        assembly {
            // Store error selector
            mstore(0x00, 0x8d6ea8be) // CustomError(uint256) selector
            mstore(0x04, value)
            revert(0x00, 0x24)
        }
    }
    
    // Efficient keccak256
    function hashTwo(uint256 a, uint256 b) public pure returns (bytes32 hash) {
        assembly {
            mstore(0x00, a)
            mstore(0x20, b)
            hash := keccak256(0x00, 0x40)
        }
    }
}
```

### Proxy Pattern Gas Optimization

```solidity
contract OptimizedProxy {
    // Use immutable for implementation address when possible
    address public immutable implementation;
    
    constructor(address _implementation) {
        implementation = _implementation;
    }
    
    // Optimized fallback
    fallback() external payable {
        address impl = implementation;
        assembly {
            // Copy calldata to memory
            calldatacopy(0, 0, calldatasize())
            
            // Delegate call
            let result := delegatecall(gas(), impl, 0, calldatasize(), 0, 0)
            
            // Copy return data
            returndatacopy(0, 0, returndatasize())
            
            // Return or revert
            switch result
            case 0 { revert(0, returndatasize()) }
            default { return(0, returndatasize()) }
        }
    }
    
    receive() external payable {}
}
```

## Testing and Measurement

### Gas Profiling

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract GasProfiler {
    event GasUsed(string operation, uint256 gasUsed);
    
    function profileOperation(string memory operation) internal {
        uint256 gasBefore = gasleft();
        
        // Perform operation
        
        uint256 gasAfter = gasleft();
        emit GasUsed(operation, gasBefore - gasAfter);
    }
    
    // Compare different implementations
    mapping(address => uint256) public balances1;
    mapping(address => uint256) public balances2;
    
    function testMethod1() public {
        uint256 gas1 = gasleft();
        balances1[msg.sender] += 100;
        uint256 gas2 = gasleft();
        emit GasUsed("Method 1", gas1 - gas2);
    }
    
    function testMethod2() public {
        uint256 gas1 = gasleft();
        unchecked {
            balances2[msg.sender] += 100;
        }
        uint256 gas2 = gasleft();
        emit GasUsed("Method 2", gas1 - gas2);
    }
}
```

### Hardhat Gas Reporter Configuration

```javascript
// hardhat.config.js
require("hardhat-gas-reporter");

module.exports = {
  gasReporter: {
    enabled: true,
    currency: 'USD',
    gasPrice: 21,
    coinmarketcap: process.env.COINMARKETCAP_API_KEY,
    outputFile: 'gas-report.txt',
    noColors: true,
    excludeContracts: ['Migrations']
  }
};
```

### Gas Comparison Tests

```javascript
// test/gas-optimization.test.js
const { expect } = require("chai");
const { ethers } = require("hardhat");

describe("Gas Optimization Tests", function() {
  it("Should use less gas with optimized version", async function() {
    const Unoptimized = await ethers.getContractFactory("Unoptimized");
    const Optimized = await ethers.getContractFactory("Optimized");
    
    const unoptimized = await Unoptimized.deploy();
    const optimized = await Optimized.deploy();
    
    // Test unoptimized
    const tx1 = await unoptimized.operation();
    const receipt1 = await tx1.wait();
    const gas1 = receipt1.gasUsed;
    
    // Test optimized
    const tx2 = await optimized.operation();
    const receipt2 = await tx2.wait();
    const gas2 = receipt2.gasUsed();
    
    console.log(`Unoptimized gas: ${gas1}`);
    console.log(`Optimized gas: ${gas2}`);
    console.log(`Savings: ${gas1 - gas2} (${((gas1 - gas2) / gas1 * 100).toFixed(2)}%)`);
    
    expect(gas2).to.be.lt(gas1);
  });
  
  it("Should compare storage patterns", async function() {
    const contract = await ethers.deployContract("StorageTest");
    
    // Test packed storage
    const tx1 = await contract.setPacked(100, 200, 300);
    const receipt1 = await tx1.wait();
    
    // Test unpacked storage
    const tx2 = await contract.setUnpacked(100, 200, 300);
    const receipt2 = await tx2.wait();
    
    console.log(`Packed: ${receipt1.gasUsed}`);
    console.log(`Unpacked: ${receipt2.gasUsed}`);
  });
});
```

## Real-World Examples

### ERC20 Token Optimization

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract OptimizedERC20 {
    // Storage layout optimized
    string public name;
    string public symbol;
    uint8 public immutable decimals;
    uint256 public totalSupply;
    
    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;
    
    // Events
    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);
    
    // Custom errors (cheaper than require strings)
    error InsufficientBalance();
    error InsufficientAllowance();
    error InvalidAddress();
    
    constructor(string memory _name, string memory _symbol, uint8 _decimals) {
        name = _name;
        symbol = _symbol;
        decimals = _decimals;
    }
    
    function transfer(address to, uint256 amount) external returns (bool) {
        // Short-circuit checks
        if (to == address(0)) revert InvalidAddress();
        
        // Cache storage reads
        uint256 senderBalance = balanceOf[msg.sender];
        if (senderBalance < amount) revert InsufficientBalance();
        
        // Unchecked math when safe
        unchecked {
            balanceOf[msg.sender] = senderBalance - amount;
            balanceOf[to] += amount; // Overflow not possible with fixed supply
        }
        
        emit Transfer(msg.sender, to, amount);
        return true;
    }
    
    function approve(address spender, uint256 amount) external returns (bool) {
        allowance[msg.sender][spender] = amount;
        emit Approval(msg.sender, spender, amount);
        return true;
    }
    
    function transferFrom(
        address from,
        address to,
        uint256 amount
    ) external returns (bool) {
        if (to == address(0)) revert InvalidAddress();
        
        // Cache storage reads
        uint256 senderBalance = balanceOf[from];
        uint256 currentAllowance = allowance[from][msg.sender];
        
        if (senderBalance < amount) revert InsufficientBalance();
        if (currentAllowance < amount) revert InsufficientAllowance();
        
        unchecked {
            balanceOf[from] = senderBalance - amount;
            balanceOf[to] += amount;
            allowance[from][msg.sender] = currentAllowance - amount;
        }
        
        emit Transfer(from, to, amount);
        return true;
    }
    
    // Batch operations save gas for multiple transfers
    function batchTransfer(
        address[] calldata recipients,
        uint256[] calldata amounts
    ) external {
        uint256 length = recipients.length;
        require(length == amounts.length, "Length mismatch");
        require(length <= 100, "Batch too large");
        
        uint256 senderBalance = balanceOf[msg.sender];
        uint256 totalAmount;
        
        // Calculate total first
        for (uint256 i = 0; i < length;) {
            totalAmount += amounts[i];
            unchecked { ++i; }
        }
        
        if (senderBalance < totalAmount) revert InsufficientBalance();
        
        // Update sender balance once
        unchecked {
            balanceOf[msg.sender] = senderBalance - totalAmount;
        }
        
        // Update recipients
        for (uint256 i = 0; i < length;) {
            address recipient = recipients[i];
            uint256 amount = amounts[i];
            
            if (recipient == address(0)) revert InvalidAddress();
            
            unchecked {
                balanceOf[recipient] += amount;
            }
            
            emit Transfer(msg.sender, recipient, amount);
            unchecked { ++i; }
        }
    }
}
```

### NFT Minting Optimization

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract OptimizedNFT {
    // Packed storage
    struct TokenData {
        address owner;      // 20 bytes
        uint48 timestamp;   // 6 bytes
        uint48 tokenId;     // 6 bytes (supports 281 trillion tokens)
    }
    
    mapping(uint256 => TokenData) private _tokens;
    mapping(address => uint256) public balanceOf;
    uint256 public totalSupply;
    
    // Events
    event Transfer(address indexed from, address indexed to, uint256 indexed tokenId);
    
    // Errors
    error TokenNotFound();
    error Unauthorized();
    error InvalidAddress();
    
    // Optimized minting
    function mint(address to) external returns (uint256) {
        if (to == address(0)) revert InvalidAddress();
        
        uint256 tokenId;
        unchecked {
            tokenId = ++totalSupply;
            balanceOf[to]++;
        }
        
        _tokens[tokenId] = TokenData({
            owner: to,
            timestamp: uint48(block.timestamp),
            tokenId: uint48(tokenId)
        });
        
        emit Transfer(address(0), to, tokenId);
        return tokenId;
    }
    
    // Batch minting
    function batchMint(address to, uint256 quantity) external {
        if (to == address(0)) revert InvalidAddress();
        require(quantity <= 20, "Batch too large");
        
        uint256 startId = totalSupply;
        
        unchecked {
            totalSupply += quantity;
            balanceOf[to] += quantity;
        }
        
        for (uint256 i = 0; i < quantity;) {
            uint256 tokenId;
            unchecked {
                tokenId = startId + i + 1;
            }
            
            _tokens[tokenId] = TokenData({
                owner: to,
                timestamp: uint48(block.timestamp),
                tokenId: uint48(tokenId)
            });
            
            emit Transfer(address(0), to, tokenId);
            unchecked { ++i; }
        }
    }
    
    function ownerOf(uint256 tokenId) external view returns (address) {
        address owner = _tokens[tokenId].owner;
        if (owner == address(0)) revert TokenNotFound();
        return owner;
    }
    
    function transfer(address to, uint256 tokenId) external {
        if (to == address(0)) revert InvalidAddress();
        
        TokenData storage token = _tokens[tokenId];
        if (token.owner != msg.sender) revert Unauthorized();
        
        unchecked {
            balanceOf[msg.sender]--;
            balanceOf[to]++;
        }
        
        token.owner = to;
        token.timestamp = uint48(block.timestamp);
        
        emit Transfer(msg.sender, to, tokenId);
    }
}
```

## Best Practices Checklist

### Storage Optimization
- ✅ Pack variables into 32-byte slots
- ✅ Order variables by size and access patterns
- ✅ Use constants and immutables when possible
- ✅ Prefer mappings over arrays for lookups
- ✅ Delete unused storage (get gas refund)

### Function Optimization
- ✅ Use `external` over `public` when possible
- ✅ Use `calldata` instead of `memory` for read-only parameters
- ✅ Use custom errors instead of require strings
- ✅ Short-circuit conditionals (cheap checks first)
- ✅ Cache storage reads in local variables

### Loop Optimization
- ✅ Cache array length before loops
- ✅ Use `unchecked` for counter increments
- ✅ Avoid unbounded loops in transactions
- ✅ Implement batch processing for large datasets
- ✅ Consider loop unrolling for small fixed loops

### Advanced Techniques
- ✅ Use bit manipulation for flags
- ✅ Pack multiple values into single storage slots
- ✅ Consider assembly for critical paths
- ✅ Batch operations when possible
- ✅ Use events instead of storage for historical data

### Testing
- ✅ Measure gas before and after optimizations
- ✅ Use hardhat-gas-reporter
- ✅ Test edge cases for gas limits
- ✅ Profile different implementations
- ✅ Document gas savings

## Conclusion

Gas optimization is both an art and a science. The techniques covered in this guide can dramatically reduce your smart contract's gas consumption, making it more accessible and economical for users.

**Key Takeaways:**

1. **Storage is expensive**: Optimize storage layout, pack variables, and minimize writes
2. **Think in opcodes**: Understand the EVM's cost model to make informed decisions
3. **Test everything**: Always measure gas usage before and after optimizations
4. **Balance readability**: Don't sacrifice code clarity for minor gas savings
5. **Stay updated**: Gas costs and best practices evolve with network upgrades

Remember that premature optimization can harm code quality. Start with clear, correct code, then optimize hot paths based on actual usage patterns. Use profiling tools to identify bottlenecks before optimizing blindly.

As Ethereum continues to evolve with Layer 2 solutions and potential protocol upgrades, gas optimization remains crucial. The principles and techniques you've learned here will serve you well regardless of future changes to the network.

## Resources

### Official Documentation
- [Ethereum Yellow Paper](https://ethereum.github.io/yellowpaper/paper.pdf) - EVM specification
- [Solidity Documentation](https://docs.soliditylang.org/) - Language reference
- [EVM Opcodes](https://www.evm.codes/) - Complete opcode reference with gas costs

### Tools
- [Hardhat Gas Reporter](https://github.com/cgewecke/hardhat-gas-reporter) - Gas usage metrics
- [eth-gas-reporter](https://github.com/cgewecke/eth-gas-reporter) - Truffle gas reporter
- [Tenderly](https://tenderly.co/) - Transaction simulation and debugging
- [Remix IDE](https://remix.ethereum.org/) - In-browser development with gas estimation

### Analysis Tools
- [Slither](https://github.com/crytic/slither) - Static analysis with gas optimization checks
- [sol2uml](https://github.com/naddison36/sol2uml) - Visualize storage layout
- [Foundry](https://book.getfoundry.sh/) - Fast testing with gas snapshots

### Learning Resources
- [RareSkills Gas Optimization](https://www.rareskills.io/post/gas-optimization) - Advanced techniques
- [Solidity Gas Optimization Tips](https://github.com/mds1/gas-optimizations) - Community collection
- [OpenZeppelin Contracts](https://github.com/OpenZeppelin/openzeppelin-contracts) - Production-grade optimized contracts

### Communities
- [r/ethdev](https://www.reddit.com/r/ethdev/) - Ethereum developers
- [Ethereum Stack Exchange](https://ethereum.stackexchange.com/) - Q&A
- [Discord: Ethereum R&D](https://discord.gg/ethereum-org) - Direct access to core devs

Happy optimizing! ⛽🚀
