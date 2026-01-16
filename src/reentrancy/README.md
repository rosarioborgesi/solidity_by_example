## Reentrancy Attack

**Reference:** https://solidity-by-example.org/hacks/re-entrancy/

This demonstrates the infamous **reentrancy vulnerability** - one of the most dangerous smart contract exploits. This attack was used in the 2016 DAO hack, which resulted in the loss of 3.6 million ETH and led to the Ethereum hard fork.

### 🎯 The Vulnerability

**What is Reentrancy?**

Let's say contract A calls contract B. Reentrancy exploit allows B to call back into A before A finishes execution.

**The Danger:**
```
Contract A calls Contract B
    ↓
Contract B calls back into Contract A (reenters!)
    ↓
Contract A's state is still in an inconsistent state
    ↓
Contract B can exploit this to steal funds
```

In this attack, the vulnerable pattern is:
1. Send ETH to the caller
2. **THEN** update the balance ← Too late! Attacker can reenter before this

---

### 🚀 Setup

**Start Anvil (local blockchain)**

```bash
anvil
```

---

### 💰 Step 1: Deploy EtherStore

Alice deploys a simple wallet contract that allows deposits and withdrawals.

**Alice's Details:**
- Address: `0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266`
- Private Key: `0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80`

```bash
forge create src/reentrancy/Reentrancy.sol:EtherStore \
  --rpc-url http://127.0.0.1:8545 \
  --private-key 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80 \
  --broadcast

  # Contract deployed to: 0x5FbDB2315678afecb367f032d93F642f64180aa3
```

---

### 👥 Step 2: Alice and Bob Deposit Funds

For the attack to be profitable, the EtherStore needs to have funds from other users.

#### Alice Deposits 1 ETH

```bash
cast send 0x5FbDB2315678afecb367f032d93F642f64180aa3 \
  "deposit()" \
  --value 1ether \
  --rpc-url http://127.0.0.1:8545 \
  --private-key 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80
```

**Verify contract balance:**

```bash
cast balance 0x5FbDB2315678afecb367f032d93F642f64180aa3 -e
# 1.000000000000000000 ✓
```

#### Bob Deposits 1 ETH

**Bob's Details:**
- Address: `0x70997970C51812dc3A010C7d01b50e0d17dc79C8`
- Private Key: `0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d`

```bash
cast send 0x5FbDB2315678afecb367f032d93F642f64180aa3 \
  "deposit()" \
  --value 1ether \
  --rpc-url http://127.0.0.1:8545 \
  --private-key 0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d
```

**Verify contract balance:**

```bash
cast balance 0x5FbDB2315678afecb367f032d93F642f64180aa3 -e
# 2.000000000000000000 ✓
```

💡 **EtherStore now holds 2 ETH** from Alice and Bob - perfect target for the attack!

---

### 😈 Step 3: Eve Deploys the Attack Contract

**Eve's Details:**
- Address: `0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC`
- Private Key: `0x5de4111afa1a4b94908f83103eb1f1706367c2e68ca870fc3fb9a804cdab365a`

```bash
forge create src/reentrancy/Reentrancy.sol:Attack \
  --rpc-url http://127.0.0.1:8545 \
  --private-key 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80 \
  --broadcast \
  --constructor-args 0x5FbDB2315678afecb367f032d93F642f64180aa3

  # Contract deployed to: 0x9fE46736679d2D9a65F0992F2272dE9f3c7fa6e0
```

---

### ⚔️ Step 4: Execute the Reentrancy Attack

**Verify Attack contract balance (before):**

```bash
cast balance 0x9fE46736679d2D9a65F0992F2272dE9f3c7fa6e0 -e
# 0.000000000000000000
```

**Launch the attack with 1 ETH:**

```bash
cast send 0x9fE46736679d2D9a65F0992F2272dE9f3c7fa6e0 \
  "attack()" \
  --value 1ether \
  --rpc-url http://127.0.0.1:8545 \
  --private-key 0x5de4111afa1a4b94908f83103eb1f1706367c2e68ca870fc3fb9a804cdab365a
```

**Verify Attack contract balance (after):**

```bash
cast balance 0x9fE46736679d2D9a65F0992F2272dE9f3c7fa6e0 -e
# 3.000000000000000000 🎯
```

**Attack successful!** Eve stole 2 ETH from Alice and Bob, plus got back her 1 ETH investment = **3 ETH total!**

**Verify EtherStore is drained:**

```bash
cast balance 0x5FbDB2315678afecb367f032d93F642f64180aa3 -e
# 0.000000000000000000
# Completely drained! ❌
```

---

### 🔍 How the Attack Works

#### The Vulnerable Code

```solidity
function withdraw() public {
    uint256 bal = balances[msg.sender];
    require(bal > 0);

    // ⚠️ DANGER: Sends ETH BEFORE updating balance
    (bool sent,) = msg.sender.call{value: bal}("");
    require(sent, "Failed to send Ether");

    // ❌ TOO LATE: Balance updated after external call
    balances[msg.sender] = 0;
}
```

#### The Attack Flow

**Step-by-step execution:**

1. **Eve calls `Attack.attack()` with 1 ETH**
   - Attack deposits 1 ETH to EtherStore
   - EtherStore balance: 3 ETH (Alice's 1 + Bob's 1 + Eve's 1)

2. **Attack calls `EtherStore.withdraw()`**
   - EtherStore checks: `balances[Attack] = 1 ETH` ✓
   - EtherStore sends 1 ETH to Attack contract
   
3. **Attack's `fallback()` function is triggered** (receives ETH)
   - Checks: `EtherStore.balance >= 1 ETH`? YES (2 ETH left!)
   - **Reenters:** Calls `EtherStore.withdraw()` again!
   
4. **Second `withdraw()` call executes**
   - Checks: `balances[Attack] = 1 ETH`? ✓ **STILL 1 ETH!** (wasn't updated yet!)
   - Sends another 1 ETH to Attack
   
5. **Attack's `fallback()` triggers again**
   - Checks: `EtherStore.balance >= 1 ETH`? YES (1 ETH left!)
   - **Reenters again:** Calls `withdraw()` a third time!
   
6. **Third `withdraw()` call executes**
   - Checks: `balances[Attack] = 1 ETH`? ✓ **STILL 1 ETH!**
   - Sends final 1 ETH to Attack
   
7. **Attack's `fallback()` triggers**
   - Checks: `EtherStore.balance >= 1 ETH`? NO (0 ETH left)
   - Stops reentering

8. **Finally, all the calls unwind**
   - First `withdraw()` sets `balances[Attack] = 0`
   - Second `withdraw()` sets `balances[Attack] = 0` (already 0)
   - Third `withdraw()` sets `balances[Attack] = 0` (already 0)

**Result:** Attack contract withdrew 3 ETH but only had a balance of 1 ETH recorded!

#### Visual Call Stack

```
Attack.attack()
├─> EtherStore.deposit() [Attack balance = 1 ETH]
└─> EtherStore.withdraw() [1st call]
    ├─> send 1 ETH to Attack
    └─> Attack.fallback()
        └─> EtherStore.withdraw() [2nd call - REENTRY!]
            ├─> send 1 ETH to Attack
            └─> Attack.fallback()
                └─> EtherStore.withdraw() [3rd call - REENTRY!]
                    ├─> send 1 ETH to Attack
                    └─> Attack.fallback()
                        └─> [stops - no more ETH]
                    └─> balances[Attack] = 0 [Too late!]
                └─> balances[Attack] = 0
            └─> balances[Attack] = 0
```

---

### 🛡️ Prevention Techniques

#### Solution 1: Checks-Effects-Interactions Pattern

**Update state BEFORE external calls:**

```solidity
function withdraw() public {
    // Checks
    uint256 bal = balances[msg.sender];
    require(bal > 0);

    // Effects
    balances[msg.sender] = 0;

    // Interactions
    (bool sent,) = msg.sender.call{value: bal}("");
    require(sent, "Failed to send Ether");
}
```

#### Solution 2: Reentrancy Guard

Use a mutex lock to prevent reentrant calls:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

contract ReEntrancyGuard {
    bool internal locked;

    modifier noReentrant() {
        require(!locked, "No re-entrancy");
        locked = true;
        _;
        locked = false;
    }
}
```

**Usage:**

```solidity
contract EtherStore is ReEntrancyGuard {
    mapping(address => uint256) public balances;

    function withdraw() public noReentrant {  // ✅ Protected
        uint256 bal = balances[msg.sender];
        require(bal > 0);

        (bool sent,) = msg.sender.call{value: bal}("");
        require(sent, "Failed to send Ether");

        balances[msg.sender] = 0;
    }
}
```

#### Solution 3: Use OpenZeppelin's ReentrancyGuard

```solidity
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

contract EtherStore is ReentrancyGuard {
    mapping(address => uint256) public balances;

    function withdraw() public nonReentrant {  // ✅ Protected
        uint256 bal = balances[msg.sender];
        require(bal > 0);

        (bool sent,) = msg.sender.call{value: bal}("");
        require(sent, "Failed to send Ether");

        balances[msg.sender] = 0;
    }
}
```

---

### 📋 Best Practices

1. **Always follow Checks-Effects-Interactions pattern:**
   - Checks: Validate conditions
   - Effects: Update state variables
   - Interactions: Call external contracts

2. **Use reentrancy guards** on functions that:
   - Transfer ETH
   - Call external contracts
   - Are public/external


4. **Use established libraries:**
   - OpenZeppelin's `ReentrancyGuard`
   - Well-audited and battle-tested

### 🎯 Key Takeaways

- ✅ Reentrancy is one of the most dangerous vulnerabilities
- ✅ Always update state before external calls
- ✅ Use reentrancy guards for defense in depth
- ✅ The DAO hack (2016) stole 3.6M ETH using this exact vulnerability
- ✅ Never trust external contracts - they can call back into your code!

**Remember:** If you're making external calls, assume they're malicious until proven otherwise!
