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

### 🔢 Arithmetic Overflow and Underflow
**Location:** `src/arithmeticOverflowAndUnderflow/`

Demonstrates integer overflow/underflow vulnerabilities in Solidity < 0.8.0. Shows how unchecked arithmetic can bypass security mechanisms like time locks, allowing immediate fund withdrawal.

- **Contracts:** `TimeLock.sol`, `Attack.sol`
- **Guide:** [README](src/arithmeticOverflowAndUnderflow/README.md)
- **Key Learning:** Why Solidity 0.8.0+ is crucial for security, understanding SafeMath, and the importance of checking arithmetic operations in legacy contracts

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

### 💣 Self Destruct Vulnerability
**Location:** `src/selfDestruct/`

Demonstrates how `selfdestruct` can be weaponized to force-send ETH to any contract, breaking game logic and business rules that rely on `address(this).balance` for critical decisions.

- **Contracts:** `EtherGame.sol`, `Attack.sol`
- **Guide:** [README](src/selfDestruct/README.md)
- **Key Learning:** Never rely on `address(this).balance` for critical logic; use internal accounting variables instead to prevent manipulation via forced ETH transfers

### 🎭 Hiding Malicious Code
**Location:** `src/hideMaliciousCode/`

Demonstrates how attackers can hide malicious code by exploiting Solidity's unchecked type casting. Any address can be cast to any contract type, allowing malicious contracts to masquerade as legitimate ones.

- **Contracts:** `Foo.sol`, `Bar.sol`, `Mal.sol`
- **Guide:** [README](src/hideMaliciousCode/README.md)
- **Key Learning:** Solidity doesn't verify type casts at runtime; prefer creating contracts with `new` instead of accepting external addresses, and always make external contract addresses public for verification

### 🍯 Honeypot Trap
**Location:** `src/honeyPot/`

Demonstrates a honeypot - a deliberately vulnerable-looking contract that's actually a trap for attackers. The contract appears to have a reentrancy vulnerability, baiting hackers to exploit it, but secretly prevents the attack from succeeding and captures their funds.

- **Contracts:** `Bank.sol`, `HoneyPot.sol`, `Attack.sol`
- **Guide:** [README](src/honeyPot/README.md)
- **Key Learning:** Understanding how type casting is unchecked, deployment parameters are hidden, and inner revert reasons are masked; demonstrating why thorough verification of external contracts is crucial before interaction

### 🏃 Front Running
**Location:** `src/frontRunning/`

Demonstrates how attackers can watch the mempool and front-run profitable transactions by submitting their own transaction with higher gas fees. Also shows the **commit-reveal scheme**, a cryptographic defense that prevents front-running by binding commitments to the sender's address.

- **Contracts:** `FrontRunning.sol` (vulnerable), `CommitReveal.sol` (secure)
- **Guide:** [README](src/frontRunning/README.md)
- **Key Learning:** Understanding mempool visibility, front-running attacks, and how commit-reveal schemes (two-phase submissions with hashed commitments) protect against transaction reordering attacks

### 🔁 Signature Replay Attack
**Location:** `src/signatureReplay/`

Demonstrates how signatures without proper replay protection can be reused multiple times to drain funds. Shows the vulnerability in off-chain signature schemes and meta-transactions, then demonstrates the secure implementation using nonce-based replay protection.

- **Contracts:** `SignatureReplay.sol` (vulnerable), `NoSignatureReplay.sol` (secure), `ECDSA.sol` (signature verification library)
- **Guide:** [README](src/signatureReplay/README.md)
- **Key Learning:** Understanding ECDSA signature recovery, the importance of nonces and contract address binding in signed messages, EIP-191 vs EIP-712, and how to implement secure multi-sig wallets with off-chain signatures for meta-transactions

### 📏 Bypass Contract Size Check
**Location:** `src/bypassContractSizeCheck/`

Demonstrates how the `extcodesize` check can be bypassed during contract construction. Contracts attempting to restrict access to EOAs only can be fooled because `extcodesize` returns 0 during a contract's constructor execution, before bytecode is stored on-chain.

- **Contracts:** `Target.sol`, `FailedAttack.sol`, `Hack.sol`
- **Guide:** [README](src/bypassContractSizeCheck/README.md)
- **Key Learning:** Understanding that `extcodesize` returns 0 during contract construction, creating a timing window where contracts can masquerade as EOAs; demonstrating why address-type-based access control is unreliable

### 🔄 Deploy Different Contracts at Same Address
**Location:** `src/deployDifferentContractsAtTheSameAddress/`

Demonstrates a sophisticated attack combining CREATE2 and selfdestruct to deploy different contracts at the same address. An attacker can deploy a benign contract, get it approved by a DAO, destroy it, then redeploy malicious code at the same address to bypass the original approval.

- **Contracts:** `DAO.sol`, `Proposal.sol`, `Attack.sol`, `DeployerDeployer.sol`, `Deployer.sol`
- **Guide:** [README](src/deployDifferentContractsAtTheSameAddress/README.md)
- **Key Learning:** Understanding CREATE2 deterministic deployment, how selfdestruct enables address reuse, the dangers of approving addresses instead of code hashes, and why delegatecall to untrusted addresses is extremely dangerous

## 🔗 Resources

- [Solidity by Example](https://solidity-by-example.org) - Original tutorial source
- [Foundry Book](https://book.getfoundry.sh/) - Foundry documentation
- [Solidity Documentation](https://docs.soliditylang.org/) - Official Solidity docs

## 📝 License

Educational purposes - following examples from Solidity by Example.
