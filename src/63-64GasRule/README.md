# ⛽ 63/64 Gas Rule Vulnerability

**Reference:** https://solidity-by-example.org/hacks/63-64-gas-rule/

---

## 📌 Overview

This demonstrates a vulnerability in contracts that attempt to measure gas consumption across external calls and refund users based on that measurement. The root cause is the **EIP-150 63/64 gas forwarding rule**, which causes gas measurements across contract boundaries to be inaccurate, leading to overpayment and potential fund drainage.

**The key:** When contract A calls contract B, the EVM only forwards **63/64** of the remaining gas, keeping **1/64** in the caller. Naive gas accounting that doesn't account for this will overestimate gas usage and overpay refunds.

---

## 🔍 Understanding the 63/64 Gas Rule

### What is the 63/64 Rule?

When contract **A** makes an **external call** to another contract **B**, the EVM does **not** forward all remaining gas.

Instead, it forwards **at most**:

- **(63/64) × (gas remaining right before the call)**

and it **keeps 1/64** in the caller (A).

This is an **EIP-150** behavior designed to prevent certain "gas griefing" and call-depth issues.

### Gas Distribution Example

If contract A has `g*` gas right before calling B:

- **B receives:** `63/64 * g*`
- **A keeps:** `1/64 * g*`

This means **1/64 of the gas is withheld** without being spent—it's simply reserved in the caller's context.

---

## 🐛 The Vulnerable Pattern

### What the Contract Tries to Do

**Contract A:**

```solidity
uint256 gasStart = gasleft();
B(b).g(msg.sender, gasStart);
```

`gasStart` captures "how much gas A had at some point before the external call."

**Contract B:**

```solidity
uint256 gasNow = gasleft();
uint256 gasUsed = gasStart - gasNow;
receiver.call{value: gasUsed}("");
```

**The idea:** Refund the caller an amount of ETH proportional to gas consumed.

---

## 💥 The Bug: Incorrect Gas Calculation

### The Problem

**Key issue:** `gasStart` was measured in **A**, but `gasNow` is measured in **B**.

Because of the 63/64 rule, when you jump from A → B, you "lose" up to **1/64 of g*** without it being spent inside B.

That missing gas stayed behind in A. **It wasn't used by B**, but the naive subtraction `gasStart - gasNow` treats it like it was "spent", causing an **overestimation of gas used**.

---

### The Math Behind the Bug

Let's define:

- `g0` = `gasleft()` in A (the `gasStart` value)
- `g*` = actual gas remaining in A **right before** calling B
- B receives `63/64 * g*`
- `g1` = `gasleft()` in B (the `gasNow` value)

**Real gas used between measurements:**

```
realGasUsed = g0 - g1 - (1/64)*g*
```

**But your vulnerable code refunds:**

```
refund = g0 - g1
```

**So it overpays by:**

```
overpay = (1/64)*g*
```

**The exploit:** An attacker can make `g*` arbitrarily large by calling A with a huge gas limit, making the overpayment enormous.

---

## 💸 How the Exploit Works

### Attack Scenario

If contract B holds ETH (it has a `receive()` function and has been funded):

1. **Attacker calls `A.f(b)` with a very high gas limit** (e.g., 9,000,000,000,000,000,000 gas)
2. **The overpayment term `(1/64)*g*` becomes massive**
3. **Contract B sends `gasUsed` wei to the receiver** (the attacker)
4. **Repeat multiple times to drain B's ETH balance**

### Why It Works

- The higher the gas limit provided to the initial call, the larger `g*` becomes
- The overpayment `(1/64)*g*` scales proportionally
- Each call drains more ETH than the actual gas consumed
- Over repeated calls, this can completely drain the contract's ETH balance

---

## 🛡️ The Fix: Accounting for the Missing Gas

### Vulnerable Code

```solidity
uint256 gasUsed = gasStart - gasNow;
```

This is **wrong** because it doesn't account for the withheld **1/64** gas portion.

---

### Secure Code

```solidity
uint256 gasUsed = gasStart - (gasNow / 63) - gasNow;
```

### Why Does This Work?

The term `gasNow / 63` approximates the missing `(1/64)*g*` portion.

**The reasoning:**

1. In contract B, you only know `gasNow` (≈ `g1`)
2. You want to estimate the missing `(1/64)*g*`, but you don't know `g*` exactly
3. However, you do know:
   - B received at most `63/64 * g*`
   - `gasNow = g1` is **≤** the gas B started with

4. From `g1 <= 63/64 * g*`, you can derive an upper/lower bound
5. The algebra shows you can safely subtract about `g1/63` (which is `gasNow / 63`) to account for the withheld 1/64

**The approach:**

- Compute the naive refund: `g0 - g1`
- **Reduce** it by an estimate of the "withheld gas portion": `gasNow / 63`
- Result: You don't overpay

---

## 🎯 Key Takeaways

1. **The 63/64 rule is fundamental** - EIP-150 always withholds 1/64 of gas in external calls

2. **Cross-contract gas measurements are dangerous** - Never assume `gasStart (in A) - gasNow (in B)` equals actual gas spent

3. **Gas-based refunds are tricky** - The withheld gas is not consumed but affects measurements across call boundaries

4. **Attackers control gas limits** - High gas limits amplify the overpayment, making drainage attacks feasible

5. **Account for EIP-150** - Always compensate for the 63/64 forwarding rule when measuring gas across calls

---

## 🛡️ Prevention

### Option 1: Correct Gas Accounting

Use the corrected formula that accounts for the withheld gas:

```solidity
uint256 gasUsed = gasStart - (gasNow / 63) - gasNow;
```

---

### Option 2: Measure Gas Within Same Call Frame

Avoid measuring gas across external calls. Measure within the same contract:

```solidity
function operation() external {
    uint256 gasStart = gasleft();
    // ... do work ...
    uint256 gasUsed = gasStart - gasleft();
    // refund based on gasUsed
}
```

---

### Option 3: Fixed Fee Model

Instead of dynamic gas-based refunds, use fixed fees or a verified accounting model:

```solidity
function operation() external payable {
    require(msg.value >= FIXED_FEE, "Insufficient fee");
    // ... do work ...
    // refund excess
    payable(msg.sender).transfer(msg.value - FIXED_FEE);
}
```

---

### Option 4: Avoid Gas-Based Refunds Entirely

The safest approach: don't tie refunds to gas consumption at all. Use alternative payment models that don't rely on gas measurements.

---

## 📚 Summary

**The Vulnerability:** Contracts that measure gas consumption across external calls will miscalculate due to the EIP-150 63/64 gas forwarding rule. The EVM withholds 1/64 of gas in the caller, but naive gas accounting treats this withheld gas as "consumed", leading to overpayment.

**The Attack:** Call the vulnerable contract with an extremely high gas limit. The overpayment `(1/64)*g*` becomes massive, allowing repeated drainage of the contract's ETH balance.

**The Root Cause:** Failing to account for the EIP-150 rule that only forwards 63/64 of gas to external calls, keeping 1/64 in the caller.

**The Fix:** Adjust the gas calculation to subtract the estimated withheld portion: `gasStart - (gasNow / 63) - gasNow`, or avoid cross-contract gas measurements entirely.

**The Lesson:** Never implement gas-based refunds without accounting for the 63/64 rule. Measuring gas across contract boundaries requires careful consideration of EIP-150 gas forwarding semantics!
