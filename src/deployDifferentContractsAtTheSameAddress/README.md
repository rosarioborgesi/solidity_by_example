# 🔄 Deploy Different Contracts at the Same Address

**Reference:** https://solidity-by-example.org/hacks/deploy-different-contracts-same-address/

---

## 📌 Overview

This demonstrates a sophisticated attack where an attacker can:
1. Deploy a **legitimate contract** at a specific address
2. Get that contract **approved** by a DAO
3. **Destroy** the legitimate contract using `selfdestruct`
4. **Re-deploy a malicious contract** at the **same address**
5. Execute the malicious contract using the previous approval

**The key:** Using `CREATE2` to deploy contracts at predictable addresses, then exploiting `selfdestruct` to clear the address and redeploy different code at the same location.

---

## 🎯 The Attack Strategy (Simple Explanation)

Imagine this scenario:

### The Setup
- **Alice** owns a DAO that controls important operations
- The DAO approves specific contract addresses to execute code via `delegatecall`
- **Attacker** wants to take control of the DAO

### The Trick
1. **Attacker deploys a "good" contract** at address `0xABC...` (a harmless `Proposal`)
2. **Alice approves** address `0xABC...` because it looks safe
3. **Attacker destroys the contract** at `0xABC...` using `selfdestruct`
4. **Attacker re-deploys a "bad" contract** at the **same address** `0xABC...` (a malicious `Attack`)
5. **Attacker executes** the approved address → now runs the malicious code!

Alice approved a specific address, but the **code at that address changed**!

---

## 🔍 How It Works Technically

### Key Concepts

#### 1. **CREATE2 - Deterministic Deployment**
- Normal contract deployment uses `CREATE` → address depends on the deployer's nonce (unpredictable)
- `CREATE2` → address depends on: `sender`, `salt`, and `bytecode` (predictable)
- Formula: `address = keccak256(0xff, sender, salt, keccak256(bytecode))`

**Why this matters:** If you know these inputs, you can predict the deployment address **before** deploying!

#### 2. **selfdestruct - Removing Contracts**
- `selfdestruct` destroys a contract and removes its code from the blockchain
- After `selfdestruct`, the address becomes "empty" again
- You can then deploy a **new contract** at that address

#### 3. **Redeploying at the Same Address**
- Use the **same** `sender`, `salt`, and `bytecode` → deploy at the **same address**
- But wait! You can use a **different factory contract** (like `Deployer`) that creates different contracts
- The `Deployer` address is predictable (via CREATE2), and it can create different contracts each time

---

## 🎭 Step-by-Step Attack Flow

Let me break down each step in simple terms:

### **Step 0: Alice Deploys DAO** (The Victim)

```
Alice → Deploys DAO
```

Alice creates a DAO that will approve and execute proposals via `delegatecall`.

**Key Code:**
```solidity
function execute(uint256 proposalId) external payable {
    Proposal storage proposal = proposals[proposalId];
    require(proposal.approved, "not approved");
    
    // This delegatecall runs in DAO's context!
    (bool ok,) = proposal.target.delegatecall(
        abi.encodeWithSignature("executeProposal()")
    );
}
```

---

### **Step 1: Attacker Deploys DeployerDeployer** (Setup)

```
Attacker → Deploys DeployerDeployer
```

This is a factory that uses **CREATE2** to deploy `Deployer` contracts at predictable addresses.

---

### **Step 2: Attacker Calls DeployerDeployer.deploy()** (Deploy Factory)

```
Attacker → DeployerDeployer.deploy() → Creates Deployer at address 0xDEF...
```

The `Deployer` is created using **CREATE2** with a specific `salt`:

```solidity
bytes32 salt = keccak256(abi.encode(uint256(123)));
address addr = address(new Deployer{salt: salt}());
```

**Important:** The `Deployer` address (`0xDEF...`) is now **predictable** and **reproducible**.

---

### **Step 3: Attacker Calls Deployer.deployProposal()** (Deploy "Good" Contract)

```
Attacker → Deployer.deployProposal() → Creates Proposal at address 0xABC...
```

The `Deployer` creates a harmless `Proposal` contract:

```solidity
function deployProposal() external {
    address addr = address(new Proposal());  // Regular CREATE
}
```

The `Proposal` contract is benign:
```solidity
function executeProposal() external {
    emit Log("Executed code approved by DAO");
}
```

---

### **Step 4: Alice Approves the Proposal** (The Trap is Set)

```
Alice → DAO.approve(0xABC...)
```

Alice reviews the `Proposal` contract at `0xABC...`, sees it's safe, and approves it in the DAO.

**The DAO now trusts address `0xABC...`**

---

### **Step 5: Attacker Deletes Proposal and Deployer** (Clean the Slate)

```
Attacker → Proposal.emergencyStop() → selfdestruct (0xABC... is now empty)
Attacker → Deployer.kill() → selfdestruct (0xDEF... is now empty)
```

Both contracts are destroyed. Their addresses are now empty!

**Critical:** The DAO's approval for address `0xABC...` still exists!

---

### **Step 6: Attacker Re-deploys Deployer** (Resurrection)

```
Attacker → DeployerDeployer.deploy() → Creates Deployer at address 0xDEF... (SAME ADDRESS!)
```

Because we use **the same CREATE2 parameters** (same salt, same bytecode), the `Deployer` is recreated at **exactly the same address** `0xDEF...`.

---

### **Step 7: Attacker Calls Deployer.deployAttack()** (Deploy "Bad" Contract)

```
Attacker → Deployer.deployAttack() → Creates Attack at address 0xABC... (SAME ADDRESS!)
```

Now the `Deployer` creates a **malicious** `Attack` contract:

```solidity
function deployAttack() external {
    address addr = address(new Attack());  // Regular CREATE
}
```

**The magic:** Because the `Deployer` is at the same address and uses the same nonce, the `Attack` contract is deployed at **the same address** `0xABC...` where the `Proposal` used to be!

The `Attack` contract is malicious:
```solidity
function executeProposal() external {
    owner = msg.sender;  // Hijack ownership via delegatecall context!
}
```

---

### **Step 8: Attacker Calls DAO.execute()** (Execute the Attack)

```
Attacker → DAO.execute(proposalId)
```

The DAO executes the "approved" proposal at address `0xABC...`:
- The DAO thinks it's executing the safe `Proposal` contract
- But `0xABC...` now contains the malicious `Attack` contract!
- Via `delegatecall`, the `Attack` code runs **in the DAO's context**

**Result:** The `owner` variable in the DAO's storage is overwritten!

---

### **Step 9: Verify - Attacker Now Owns the DAO** (Success!)

```
Check DAO.owner → Returns Attacker's address
```

The attacker has successfully hijacked the DAO's ownership! 🚨

---

## 🧩 Why This Works

### 1. **CREATE2 Predictability**
The `Deployer` address is deterministic, so it can be recreated at the same address.

### 2. **selfdestruct Resets Addresses**
After `selfdestruct`, an address becomes empty and can receive a new contract.

### 3. **Nonce Resets**
When a contract is destroyed and recreated, its nonce resets. Using `CREATE` from the same deployer address with the same nonce produces the same deployed contract address.

### 4. **delegatecall Context**
The malicious code runs in the DAO's storage context, allowing it to modify the DAO's state variables (like `owner`).

### 5. **Approval Bound to Address, Not Code**
The DAO approved an **address**, not specific **code**. Once approved, any code at that address can be executed.

---

## 🎯 Key Takeaways

1. **Approving addresses is dangerous** - The code at an address can change after `selfdestruct` and redeployment

2. **CREATE2 + selfdestruct = Address Reuse** - Contracts can be destroyed and different code deployed at the same address

3. **delegatecall is extremely powerful** - It executes external code in your contract's storage context

4. **Don't trust addresses alone** - Consider checking code hashes or using other verification methods

5. **This attack requires multiple steps** - But it's technically feasible on-chain

---

## 🛡️ Prevention

### Option 1: Don't Use `selfdestruct` (EIP-6780)
As of the Cancun upgrade (EIP-6780), `selfdestruct` no longer deletes code except in the same transaction as contract creation. This significantly limits this attack vector.

### Option 2: Verify Code Hash
Instead of just approving an address, also store and verify the code hash:

```solidity
bytes32 codeHash = address(proposal).codehash;
// Store and verify this hash doesn't change
```

### Option 3: Avoid delegatecall to External Contracts
`delegatecall` to untrusted addresses is inherently dangerous. Consider alternative patterns.

### Option 4: Use a Registry Pattern
Maintain a registry of verified contract implementations rather than allowing arbitrary addresses.

---

## 📚 Summary

**The Vulnerability:** Contracts can be destroyed with `selfdestruct` and then different code can be deployed at the same address using CREATE2, bypassing address-based approvals.

**The Attack:** Deploy a benign contract → get approval → destroy it → redeploy malicious code at same address → execute with existing approval.

**The Lesson:** Never trust an address alone. Addresses can have their code changed. Always verify what code you're calling, especially with `delegatecall`.
