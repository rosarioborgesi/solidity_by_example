# Solidity by Example - Foundry Implementation

This repository contains code examples from [Solidity by Example](https://solidity-by-example.org) implemented and tested using **Foundry**.

## 📋 About

[Solidity by Example](https://solidity-by-example.org) provides excellent educational content for learning Solidity. This repository recreates those examples using Foundry, a blazing-fast Ethereum development framework written in Rust.

## 🛠️ Prerequisites

- [Foundry](https://book.getfoundry.sh/getting-started/installation) installed
- Basic knowledge of Solidity and smart contracts

## 🚀 Getting Started

```bash
# Clone the repository
git clone <repository-url>
cd foundry_sscd_prep

# Install dependencies
forge install

# Run tests
forge test

# Start local blockchain (Anvil)
anvil
```

## 📚 Examples

### 🔓 Accessing Private Data Hack
**Location:** `src/accessingPrivateDataHack/`

Demonstrates that "private" variables in Solidity are not truly private. All blockchain data is publicly readable through direct storage access.

- **Contract:** `Vault.sol`
- **Guide:** [README](src/accessingPrivateDataHack/README.md)
- **Key Learning:** Understanding Solidity storage layout and how to read private data using `cast storage`

### ⚔️ Denial of Service Attack
**Location:** `src/denialOfService/`

Demonstrates how a malicious contract can permanently lock a legitimate contract by refusing to accept Ether refunds, rendering the entire system unusable.

- **Contracts:** `KingOfEther.sol`, `Attack.sol`
- **Guide:** [README](src/denialOfService/README.md)
- **Key Learning:** The importance of the "pull over push" pattern and avoiding automatic fund transfers that can fail

## 🔗 Resources

- [Solidity by Example](https://solidity-by-example.org) - Original tutorial source
- [Foundry Book](https://book.getfoundry.sh/) - Foundry documentation
- [Solidity Documentation](https://docs.soliditylang.org/) - Official Solidity docs

## 📝 License

Educational purposes - following examples from Solidity by Example.
