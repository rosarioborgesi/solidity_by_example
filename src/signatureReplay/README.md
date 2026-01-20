# 🔁 Signature Replay Attack

**Reference:** https://solidity-by-example.org/hacks/signature-replay/

---

## 📌 Overview

Signing messages **off-chain** and having a contract verify that signature before executing a function is a powerful technique used to:

- **Reduce on-chain transactions** (save gas and improve UX)
- Enable **gas-less transactions** (meta-transactions)

However, if not implemented correctly, signatures can be **replayed** multiple times, leading to severe vulnerabilities.

---

## 🚀 What is a Meta-Transaction?

**Meta-transactions** in Ethereum allow a user to interact with a smart contract **without paying gas themselves**.

Instead:
- A **relayer** submits the transaction and pays the gas
- The **user** only signs a message off-chain

### How Meta-Transactions Work (Step-by-Step)

#### 1. User Signs a Message (Off-Chain)

The message includes:
- Target contract address
- Function name + parameters
- Nonce (for replay protection)
- Chain ID (for cross-chain protection)
- Expiry timestamp (optional)

#### 2. Relayer Submits the Transaction (On-Chain)

The relayer wraps the signed message into a normal Ethereum transaction and pays the gas.

#### 3. Smart Contract Verifies the Signature

The contract checks:
- Signature validity
- Nonce (replay protection)
- That the signer is the intended user

#### 4. Contract Executes Logic "As If" the User Called It

Even though `msg.sender` is the relayer, the contract recovers the **real user** from the signature.

---

## ⚠️ Vulnerability: Signature Replay

**The Problem:** The same signature can be used **multiple times** to execute a function. This can be catastrophic if the signer's intention was to approve a transaction **only once**.

---

## 🔍 Understanding the Vulnerable Contract

The vulnerability is demonstrated in `signatureReplay/SignatureReplay.sol` (the `MultiSigWallet` contract).

### How the Contract Works

- **`deposit()`** allows you to deposit Ether (could use `fallback`/`receive`, but this is explicit)
- **`getTxHash()`** calculates the transaction hash by hashing the recipient `_to` and the `_amount`
- **`_checkSigs()`** validates signatures by:
  1. Calculating the signed message with `ethSignedHash = _txHash.toEthSignedMessageHash()`
  2. Recovering the signer from the message: `signer = ethSignedHash.recover(_sigs[i])`
  3. Verifying that the signer is an owner: `bool valid = signer == owners[i]`
- **`transfer()`** after validating signatures via `_checkSigs()`, sends the amount to the recipient: `_to.call{value: _amount}("")`

---

## 🔐 Deep Dive: Signature Components

### 1. What `bytes[2] memory _sigs` Represents

```solidity
bytes[2] memory _sigs
```

This is an array of **two ECDSA signatures**, one per owner.

Each element:
- Is a `bytes` value
- Contains a standard Ethereum signature: **65 bytes** = `r (32)` + `s (32)` + `v (1)`

**Conceptually:**
- `_sigs[0]` → signature from `owners[0]`
- `_sigs[1]` → signature from `owners[1]`

Each owner signs the **same message hash** off-chain.

---

### 2. What `ethSignedHash` Is

```solidity
bytes32 ethSignedHash = _txHash.toEthSignedMessageHash();
```

This transforms the raw hash into the **Ethereum Signed Message** format:

```solidity
return keccak256(
    abi.encodePacked("\x19Ethereum Signed Message:\n32", hash)
);
```

**Why this matters:**
- Wallets (MetaMask, Ledger, etc.) sign this **prefixed hash**
- Prevents signing raw transaction-like data by mistake
- Standard for `eth_sign` / `personal_sign`

So `ethSignedHash` is **exactly what the owners signed**.

---

### 3. What `recover()` Does

```solidity
address signer = ethSignedHash.recover(_sigs[i]);
```

This line:
1. Takes the message hash (`ethSignedHash`)
2. Takes one signature (`_sigs[i]`)
3. Uses `ecrecover` (via the ECDSA library)
4. **Recovers the Ethereum address** that produced that signature

**Question it answers:** *Given this message and this signature, who signed it?*

---

## 🚨 Critical Vulnerabilities (Educational Code - NOT Production-Safe)

### 1. ⚠️ Replay Attacks

**Problem:** There is **no nonce**.

Once signatures exist, they can be **reused forever** to drain the wallet.

**Example:**
- Owners sign a transfer of 1 ETH
- Attacker replays it **10 times**
- Wallet drained ❌

---

### 2. 🌐 Cross-Chain Replay

The hash does **not include**:
- `chainId`
- `address(this)`

**Result:** Same signatures could be replayed:
- On another chain (e.g., Ethereum → Polygon)
- On another deployed wallet with the same owners

---

### 3. 🔀 `abi.encodePacked` Ambiguity

For `(address, uint256)` this is safe, but as a best practice:
- `abi.encode` is **safer**
- Avoids collision risks in more complex payloads

---

### 4. 📋 Signature Ordering Requirement

Signatures **must be passed in exact owner order**.

If swapped, verification fails.

**Real multisigs:**
- Sort signers by address
- Or allow unordered signatures

---

## ✅ How a Production Version Would Fix This

A **safe version** would hash:

```solidity
keccak256(
    abi.encode(
        address(this),     // prevents cross-contract replay
        block.chainid,     // prevents cross-chain replay
        nonce,             // prevents replay attacks
        _to,
        _amount
    )
);
```

**And also:**
- Increment `nonce` after each transaction
- Allow unordered signatures (sorted by address)
- Support ERC-1271 (contract signatures)
- Use EIP-712 structured data (better UX and security)

**This is exactly what [Safe (formerly Gnosis Safe)](https://safe.global/) does.**

---

## 📝 Important Standards

### EIP-191: Ethereum Signed Message Format

**What it does:** Adds the prefix `\x19Ethereum Signed Message:\n32` to prevent signing raw transaction data.

**What it doesn't do:** Prevent replay attacks.

### EIP-712: Structured Data Signing

Defines a **structured, typed way** to hash and sign data, with **domain separation**.

**To prevent replay attacks, you need at least:**
- `nonce`
- `address(this)` (contract address)
- `block.chainid` (chain ID)

---

## 🛡️ Preventative Techniques

**Solution:** Sign messages with **nonce** and **contract address**.

This has been implemented in `signatureReplay/NoSignatureReplay.sol` (the `MultiSigWallet` contract).

### The Fix

The **only change** is in the `getTxHash()` function, which now hashes:

```solidity
keccak256(abi.encodePacked(address(this), _to, _amount, _nonce));
```

**This includes:**
- `address(this)` → prevents cross-contract replay
- `_to` → recipient
- `_amount` → transfer amount
- `_nonce` → prevents replay attacks

---

## 🧪 Demonstration: Secure Multi-Sig Wallet


Let's test the **secure** implementation!

---

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

### Step 1: Deploy MultiSigWallet

Alice and Bob will be the two owners of the multi-sig wallet.

**Alice's Details (Anvil Account #0):**
- Address: `0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266`
- Private Key: `0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80`

**Bob's Details (Anvil Account #1):**
- Address: `0x70997970C51812dc3A010C7d01b50e0d17dc79C8`
- Private Key: `0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d`

**Deploy the contract:**

```bash
forge create src/signatureReplay/NoSignatureReplay.sol:MultiSigWallet \
  --rpc-url http://127.0.0.1:8545 \
  --private-key 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80 \
  --broadcast \
  --constructor-args [0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266,0x70997970C51812dc3A010C7d01b50e0d17dc79C8]

  # Contract deployed to: 0x5FbDB2315678afecb367f032d93F642f64180aa3
```

---

### Step 2: Alice Deposits 2 ETH into the Multi-Sig Wallet

```bash
cast send 0x5FbDB2315678afecb367f032d93F642f64180aa3 \
  "deposit()" \
  --value 2ether \
  --rpc-url http://127.0.0.1:8545 \
  --private-key 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80
```

**Verify the wallet balance:**

```bash
cast balance 0x5FbDB2315678afecb367f032d93F642f64180aa3 -e
# 2.000000000000000000 ETH
```

✅ The wallet now holds **2 ETH**.

---

### Step 3: Create Signatures for a Transfer

Alice and Bob want to transfer funds to **Eve**.

**Eve's Details (Anvil Account #2):**
- Address: `0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC`
- Private Key: `0x5de4111afa1a4b94908f83103eb1f1706367c2e68ca870fc3fb9a804cdab365a`

**Check Eve's initial balance:**

```bash
cast balance 0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC -e
# 10000.000000000000000000 ETH
```

---

#### 3.1 Get the Transaction Hash from the Contract

```bash
# Define your parameters
TO_ADDRESS="0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC"  # recipient (Eve)
AMOUNT="2000000000000000000"  # 2 ETH in wei
NONCE="0"  # first transaction
WALLET_ADDRESS="0x5FbDB2315678afecb367f032d93F642f64180aa3"

# Get the tx hash from the contract
TX_HASH=$(cast call $WALLET_ADDRESS \
  "getTxHash(address,uint256,uint256)" \
  $TO_ADDRESS $AMOUNT $NONCE \
  --rpc-url http://127.0.0.1:8545)

echo "Transaction Hash: $TX_HASH"
```

**Output:**

```
Transaction Hash: 0xbcbb9550403becd4c577c45c4fead7b98a1af27db2ee5488a9075277e6f3f649
```

---

#### 3.2 Sign the Hash with Both Owners' Private Keys

```bash
# Alice's private key (Anvil account #0)
ALICE_KEY="0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80"

# Bob's private key (Anvil account #1)
BOB_KEY="0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d"

# Sign with Alice (owner[0])
SIG_ALICE=$(cast wallet sign --private-key $ALICE_KEY $TX_HASH)
echo "Alice's signature: $SIG_ALICE"

# Sign with Bob (owner[1])
SIG_BOB=$(cast wallet sign --private-key $BOB_KEY $TX_HASH)
echo "Bob's signature: $SIG_BOB"
```

**Output:**

```
Alice's signature: 0xeda3159a088c59f9d99a82217bfbe044fe863daba5dc53d770179937003057562477394c709ac7e11369626e03352b027faa4b73c481c55798328f3bf3305c931b

Bob's signature: 0xea83e3a1221fbcd0688666ef861b64d7f096a9026a101445a0d5b16bd32618c00d436807db63559876d3d118b36e7b842f08274ddef266f70ad7db5a05da1ceb1b
```

**Note:** `cast wallet sign` automatically applies the Ethereum Signed Message prefix, matching what `.toEthSignedMessageHash()` does in the contract.

---

#### 3.3 Execute the Transfer with Both Signatures

```bash
cast send $WALLET_ADDRESS \
  "transfer(address,uint256,uint256,bytes[2])" \
  $TO_ADDRESS \
  $AMOUNT \
  $NONCE \
  "[$SIG_ALICE,$SIG_BOB]" \
  --rpc-url http://127.0.0.1:8545 \
  --private-key $ALICE_KEY
```

---

### Step 4: Verify the Transfer

**Check Eve's new balance:**

```bash
cast balance 0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC -e
# 10002.000000000000000000 ETH
```

✅ **Eve successfully received 2 ETH!**

---

## 🔑 Key Takeaways

1. **Replay Protection:** By including `nonce` and `address(this)` in the transaction hash, the same signatures **cannot be replayed**.

2. **Off-Chain Signatures:** Both Alice and Bob signed the transaction **off-chain** using `cast wallet sign`. Anyone could submit the transaction on-chain (paying gas), but only the owners' signatures authorize the transfer.

3. **Multi-Sig Security:** The contract requires **both owners** to sign before executing any transfer, providing 2-of-2 multi-signature security.

4. **Production Best Practices:** Real implementations (like Safe) use:
   - `abi.encode` instead of `abi.encodePacked`
   - EIP-712 structured data
   - Chain ID for cross-chain protection
   - Sorted or unordered signature support

---

## 📚 Summary

**The Vulnerability:** Signatures without nonces can be replayed indefinitely.

**The Fix:** Include `nonce`, `address(this)`, and optionally `chainId` in the signed message.

**Real-World Use:** This pattern powers meta-transactions, gasless UX, and secure multi-sig wallets like [Safe](https://safe.global/).