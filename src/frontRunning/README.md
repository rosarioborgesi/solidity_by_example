# Front Running

**Reference:** [Solidity by Example - Front Running](https://solidity-by-example.org/hacks/front-running/)

## 🎯 Vulnerability

Transactions take time before they are mined, sitting in the **mempool** where they are visible to everyone. An attacker can:
1. Watch the transaction pool for profitable transactions
2. Copy the transaction data
3. Submit their own transaction with a **higher gas price**
4. Get their transaction mined **first**, front-running the original

This allows attackers to reorder transactions to their advantage, stealing rewards or manipulating outcomes.

**Example Contract:** `FrontRunning.sol` demonstrates this vulnerability.

---

## 🛡️ Defense: Commit-Reveal Scheme

A **commit-reveal scheme** is a cryptographic pattern that prevents front-running by separating transaction submission into two phases:

### How It Works

1. **Commit Phase:** Users submit a **hash** of their answer + a secret
   - The actual answer is hidden
   - Front-runners can copy the hash, but it won't help them

2. **Reveal Phase:** Users reveal their actual answer + secret
   - The contract verifies: `keccak256(msg.sender + answer + secret) == committedHash`
   - Front-runners fail because their `msg.sender` is different

**Key Insight:** The hash includes `msg.sender`, so copying the commit hash is useless for attackers!

**Example Contract:** `CommitReveal.sol` demonstrates this defense.

**Further Reading:**
- [Exploring Commit-Reveal Schemes on Ethereum](https://medium.com/swlh/exploring-commit-reveal-schemes-on-ethereum-c4ff5a777db8)
- [Submarine Sends](https://libsubmarine.org/)

---

## 🧪 Demonstration: Commit-Reveal Protection

The following walkthrough shows how the commit-reveal scheme protects Bob's answer from being front-run by Eve.

### Step 1: Alice Deploys the Contract with 10 ETH Reward

Alice creates a `SecuredFindThisHash` contract with 10 ETH as the reward for finding the correct answer.

**Alice's Account:**
- Address: `0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266`
- Private Key: `0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80`

```bash
forge create src/frontRunning/CommitReveal.sol:SecuredFindThisHash \
  --rpc-url http://127.0.0.1:8545 \
  --value 10ether \
  --private-key 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80 \
  --broadcast
```

**Output:**
```
Contract deployed to: 0x5FbDB2315678afecb367f032d93F642f64180aa3
```

### Step 2: Bob Finds the Correct Answer

Bob discovers that "Ethereum" hashes to the target value in the contract.

**Bob's Account:**
- Address: `0x70997970C51812dc3A010C7d01b50e0d17dc79C8`
- Private Key: `0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d`

```bash
cast keccak "Ethereum"
```

**Output:**
```
0x564ccaf7594d66b1eaaea24fe01f0585bf52ee70852af4eac0cc4b04711cd0e2
```

✅ This matches the `hash` variable in the `SecuredFindThisHash` contract!

---

### Step 3: Bob Creates His Commit Hash

Bob creates a commitment hash using: `keccak256(address + solution + secret)`

**Components:**
- **Address:** `0x70997970c51812dc3a010c7d01b50e0d17dc79c8` (Bob's address in lowercase)
- **Solution:** `"Ethereum"` (the answer)
- **Secret:** `"mysecret"` (Bob's private password, prevents others from copying his commit)

```bash
# Set variables
address="70997970c51812dc3a010c7d01b50e0d17dc79c8"
solution='Ethereum'
secret='mysecret'

# Step 2: Convert strings to hex
solution_hex=$(cast --from-utf8 $solution)
secret_hex=$(cast --from-utf8 $secret)

# Step 3: Concatenate and hash
hash=$(cast keccak "0x${address}${solution_hex:2}${secret_hex:2}")
```

**Output:**
```
0x90d39e05f4161bf65aaa7a43665e73fe087df2a5c4d83555e9d1c9fef2c4c167
```

This hash commits Bob to his answer without revealing it!

---

### Step 4: Bob Commits His Hash (Commit Phase)

Bob submits his commit hash to the contract. The actual answer remains hidden.

```bash
cast send 0x5FbDB2315678afecb367f032d93F642f64180aa3 \
  "commitSolution(bytes32)" $hash \
  --gas-price 15gwei \
  --rpc-url http://127.0.0.1:8545 \
  --private-key 0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d
```

**✅ Transaction successful!** Bob's commitment is now on-chain.

---

### Step 5: Verify Bob's Commitment

Let's check that Bob's commitment was stored correctly:

```bash
cast call 0x5FbDB2315678afecb367f032d93F642f64180aa3 \
  "getMySolution()" \
  --rpc-url http://127.0.0.1:8545 \
  --private-key 0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d
```

**Output:**
```
0x90d39e05f4161bf65aaa7a43665e73fe087df2a5c4d83555e9d1c9fef2c4c16700000000000000000000000000000000000000000000000000000000696e31090000000000000000000000000000000000000000000000000000000000000000
```

This returns: `(bytes32 solutionHash, uint256 commitTime, bool revealed)`

Check Bob's balance before revealing:

```bash
cast balance 0x70997970C51812dc3A010C7d01b50e0d17dc79C8 -e
```

**Output:**
```
9999.999937528408641606 ETH
```

---

### Step 6: Bob Reveals His Solution (Reveal Phase)

Now Bob reveals his actual answer and secret. The contract will verify:
1. `keccak256(msg.sender + solution + secret)` matches Bob's committed hash
2. `keccak256(solution)` matches the target hash

```bash
cast send 0x5FbDB2315678afecb367f032d93F642f64180aa3 \
  "revealSolution(string,string)" $solution $secret \
  --rpc-url http://127.0.0.1:8545 \
  --private-key 0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d
```

**✅ Transaction successful!** Bob wins the 10 ETH reward.

---

### Step 7: Verify Bob Won

Check Bob's balance after revealing:

```bash
cast balance 0x70997970C51812dc3A010C7d01b50e0d17dc79C8 -e
```

**Output:**
```
10009.999869864568830766 ETH
```

**Bob gained ~10 ETH!** 🎉 (minus gas fees)

---

## 🛡️ Why Eve (The Attacker) Cannot Front-Run This

Let's consider what happens if Eve tries to front-run Bob:

### Scenario 1: Eve Copies Bob's Commit

**What Eve sees in the mempool:**
```
commitSolution(0x90d39e05f4161bf65aaa7a43665e73fe087df2a5c4d83555e9d1c9fef2c4c167)
```

**Eve's attempt:**
1. Eve copies Bob's commit hash
2. Eve submits `commitSolution(0x90d39...c167)` with higher gas
3. ✅ Eve's transaction succeeds and gets mined first

**But:** Eve has only committed to a hash she doesn't understand. She doesn't know the `solution` or `secret`!

---

### Scenario 2: Eve Tries to Copy Bob's Reveal

**What Eve sees in Bob's reveal transaction:**
```
revealSolution("Ethereum", "mysecret")
```

**Eve's attempt:**
1. Eve sees Bob's reveal in the mempool
2. Eve copies the exact same parameters: `revealSolution("Ethereum", "mysecret")`
3. Eve submits with higher gas to front-run Bob
4. ❌ Eve's transaction **REVERTS** with "Hash doesn't match"

**Why Eve fails:**

The contract checks:
```solidity
keccak256(abi.encodePacked(msg.sender, _solution, _secret)) == commit.solutionHash
```

- **Bob's commit hash:** `keccak256(Bob's address + "Ethereum" + "mysecret")`
- **Eve's reveal hash:** `keccak256(Eve's address + "Ethereum" + "mysecret")` ❌ Different!

Since `msg.sender` is different, Eve's reveal doesn't match her commit!

---

## 🎓 Key Takeaways

1. **Commit-Reveal prevents front-running** by binding commitments to the sender's address
2. **Two-phase submission** separates the "what" (commit) from the "reveal" (proof)
3. **Secrets add unpredictability** - attackers can't precompute commit hashes
4. **Front-runners are trapped** - they can copy commits but can't successfully reveal

This pattern is used in:
- On-chain voting systems
- Sealed-bid auctions
- Fair random number generation
- Any system requiring hidden input


