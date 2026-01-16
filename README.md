# Solidity Security Vulnerabilities - Foundry Implementation

This repository contains **security vulnerability examples** (hacks) from [Solidity by Example](https://solidity-by-example.org/hacks/), implemented and tested using **Foundry**.

## 📋 About

[Solidity by Example](https://solidity-by-example.org/hacks/) provides educational content on common smart contract vulnerabilities. This repository recreates those vulnerability demonstrations using Foundry, allowing you to learn how these attacks work and how to prevent them in a hands-on environment.


## 🛠️ Prerequisites

- [Foundry](https://book.getfoundry.sh/getting-started/installation) installed
- Basic knowledge of Solidity and smart contracts

## 📚 Examples

### 🔓 Accessing Private Data Hack
**Location:** `src/accessingPrivateData/`

Demonstrates that "private" variables in Solidity are not truly private. All blockchain data is publicly readable through direct storage access.

- **Contract:** `Vault.sol`
- **Guide:** [README](src/accessingPrivateData/README.md)
- **Key Learning:** Understanding Solidity storage layout and how to read private data using `cast storage`

### 🔄 Reentrancy Attack
**Location:** `src/reentrancy/`

Demonstrates the infamous reentrancy vulnerability used in the 2016 DAO hack. Shows how malicious contracts can recursively call back into vulnerable contracts before state updates complete, draining funds.

- **Contracts:** `EtherStore.sol`, `Attack.sol`
- **Guide:** [README](src/reentrancy/README.md)
- **Key Learning:** The Checks-Effects-Interactions pattern, reentrancy guards, and why state should be updated before external calls

### ⚔️ Denial of Service Attack
**Location:** `src/denialOfService/`

Demonstrates how a malicious contract can permanently lock a legitimate contract by refusing to accept Ether refunds, rendering the entire system unusable.

- **Contracts:** `KingOfEther.sol`, `Attack.sol`
- **Guide:** [README](src/denialOfService/README.md)
- **Key Learning:** The importance of the "pull over push" pattern and avoiding automatic fund transfers that can fail

### 🎭 Delegatecall Vulnerability
**Location:** `src/delegateCall/`

Demonstrates how mismatched storage layouts between contracts can be exploited to hijack ownership when using delegatecall, allowing an attacker to take control of a contract.

- **Contracts:** `Lib.sol`, `HackMe.sol`, `Attack.sol`
- **Guide:** [README](src/delegateCall/README.md)
- **Key Learning:** Understanding delegatecall context, storage layout matching, and the dangers of delegatecalling to mutable addresses

### 🎣 Phishing with tx.origin
**Location:** `src/phishingWithTxOrigin/`

Demonstrates how using `tx.origin` for authentication creates a phishing vulnerability where attackers can trick users into calling malicious contracts that drain their wallets.

- **Contracts:** `Wallet.sol`, `Attack.sol`
- **Guide:** [README](src/phishingWithTxOrigin/README.md)
- **Key Learning:** The critical difference between `tx.origin` and `msg.sender`, and why `tx.origin` should never be used for authentication

## 🔗 Resources

- [Solidity by Example](https://solidity-by-example.org) - Original tutorial source
- [Foundry Book](https://book.getfoundry.sh/) - Foundry documentation
- [Solidity Documentation](https://docs.soliditylang.org/) - Official Solidity docs

## 📝 License

Educational purposes - following examples from Solidity by Example.
